# Architecture Deep Dive

This document covers the key technical decisions made in building this system — not just what was built, but why.

---

## Why 4 Microservices?

The system was designed from the start for **Google Cloud Run**, which runs stateless containers that scale to zero. A monolith would work, but a microservice split gave three concrete advantages here:

1. **Independent scaling** — The RAG service is the most resource-intensive (ChromaDB + LLM calls). It needs 2 CPU / 2Gi RAM. The WhatsApp service and web app are lightweight. Splitting them means each service gets exactly the resources it needs, and scales based on its own traffic.

2. **Cost efficiency** — Cloud Run charges per request and scales to zero when idle. An Italian agricultural assistant has very uneven traffic (busy in spring/summer growing season, quiet in winter). Zero-scaling on 4 services saves significant cost vs. keeping a monolith always-on.

3. **Fault isolation** — If the RAG service has a cold-start delay or an OpenAI rate limit hit, the web app and WhatsApp service keep running. Users see a degraded response, not a full outage.

---

## The Session Problem on Cloud Run

Cloud Run is **stateless** — each request may hit a different container instance. Standard Flask sessions (stored in signed cookies) work fine, but storing any real user state in cookies has security implications.

**Decision:** Server-side sessions stored in PostgreSQL via the DB service.

Every request carries a `viridia_session_id` cookie (a UUID). The web app reads this cookie, calls the DB service to load the session data, hydrates a `DatabaseSession` object (a subclass of `dict`), and uses it throughout the request. On response, if the session was modified, it writes back to the DB.

This means:
- Session data never touches the client
- Sessions survive container restarts and scale-out
- Sessions can be invalidated server-side instantly (for logout or security events)
- The 24-hour TTL is enforced at the DB level, not the cookie level

---

## The WhatsApp Async Processing Pattern

Twilio's webhook has a **15-second hard timeout**. If your endpoint doesn't respond in 15 seconds, Twilio marks the message as failed and may retry.

A full message processing cycle involves:
- DB lookup (user + conversation history)
- ChromaDB vector search
- OpenAI GPT-4o API call (can take 5-15 seconds alone)
- DB write (chat log + query log)
- Twilio send (outbound message)

This easily exceeds 15 seconds.

**Decision:** Return HTTP 200 to Twilio immediately, process everything in a background daemon thread.

```
Twilio webhook arrives
       │
       ▼
Webhook handler validates signature
       │
       ▼
Spawns background thread → returns 200 to Twilio instantly
       │
       (background thread)
       ▼
Full processing pipeline runs
       │
       ▼
Twilio send API called with AI response
```

The thread is a Python `threading.Thread(daemon=True)` — daemon threads die if the main process exits, which is acceptable since Cloud Run manages graceful shutdown. If the background thread crashes, an error handler attempts to send the user a fallback error message in Italian.

---

## The RAG Pipeline

**ChromaDB** stores vector embeddings of Italian agricultural documents — regulations, crop treatment guides, product data sheets, seasonal advice.

On every query:
1. The user's message is embedded and used to search ChromaDB for the top-K most relevant document chunks
2. Those chunks are injected into the prompt as context
3. The full prompt (`system prompt + user context + RAG context + conversation history + user message`) is sent to GPT-4o
4. GPT-4o generates a response grounded in the retrieved documents

**Cold start behaviour:** ChromaDB data is stored in Google Cloud Storage. When a RAG service container starts cold, it syncs the full ChromaDB directory from GCS before serving any requests. This adds ~10-30 seconds to cold starts but ensures the container always has up-to-date knowledge.

**Why ChromaDB over Pinecone/Weaviate?** ChromaDB is file-based and runs in-process — no external service to manage, no additional cost, no network round-trip for vector search. For a knowledge base of this size (agricultural documents for a specific region), in-process is faster and cheaper.

---

## Two Separate User Domains

Web users and WhatsApp users are stored in completely separate database tables (`users` vs `whatsapp_users`), with separate conversation tables.

**Why not a single `users` table with a nullable `phone_number`?**

- Web users are identified by email. WhatsApp users are identified by phone number. Neither is guaranteed to have the other.
- The onboarding flows are entirely different (web form vs. one-time token link sent via WhatsApp).
- A WhatsApp user who later creates a web account should have a clean merge path — not a pre-existing half-populated record.
- GDPR consent is tracked separately because the method of consent collection differs between channels.

---

## GDPR Compliance Design

The system was built for Italian users, so GDPR compliance was a first-class requirement, not an afterthought.

**What's tracked:**
- Terms of use, privacy policy, and marketing consent — each tracked independently
- Timestamp, IP address, user-agent, and consent version stored per consent event
- Consent can be revoked — revocation is also logged (the `consent_logs` table tracks the full history, not just current state)
- A separate `onboarding_tokens` table handles the WhatsApp onboarding flow — one-time tokens expire in 24 hours and are marked `completed` once used, preventing reuse

**`consent_logs` is append-only by design** — you never update or delete a consent record, you only add new ones. This gives a full audit trail of every consent event for every user, which is what GDPR Article 7 requires.

---

## QueryLog — Observability at the DB Level

Every RAG query is logged to the `query_logs` table with:
- The full query text
- The system prompt used (with user context filled in)
- The context chunks retrieved from ChromaDB
- The AI response
- Timing metrics: `retrieval_time_ms`, `llm_time_ms`, `total_time_ms`
- Token count
- User identifier and conversation ID

This allows post-hoc analysis of:
- Which queries retrieved poor context (hallucination source tracing)
- LLM latency trends over time
- Token consumption per user / per conversation
- Prompt effectiveness across different user contexts

LangFuse is used in parallel for real-time conversation monitoring and alerting.

---

## Image Handling

When a WhatsApp user sends a photo:

1. Twilio webhook includes `MediaUrl0`, `MediaContentType0`, `NumMedia`
2. The WhatsApp service downloads the image from Twilio using basic auth (Twilio requires credentials to access media URLs)
3. The image is uploaded to a dedicated GCS bucket (`viridia-whatsapp-images`) with a path keyed by phone number and timestamp
4. A **signed URL** is generated for the GCS object (time-limited access)
5. The signed URL is passed to the RAG service alongside the text message
6. GPT-4o receives the signed URL in a vision-enabled message and analyses the image

**Daily quota enforcement:** Each `whatsapp_users` record has `daily_image_count` and `last_image_date`. On each image message, the handler checks if `last_image_date` is today — if not, the counter resets. If the daily limit is reached, the user receives a polite message in Italian explaining the limit.

---

## Deployment

Each service has its own `Dockerfile` and `cloudbuild.yaml`. Cloud Build handles container builds and pushes to Google Container Registry. Cloud Run pulls from GCR on deploy.

Secrets (OpenAI API key, Twilio credentials, DB connection string, Flask secret key) are managed via **Google Secret Manager** and injected into Cloud Run as environment variables at deploy time — nothing sensitive is ever in the container image or source code.
