# Complete Project Documentation — Italian Agronomist AI Chatbot

This document is a full technical and functional walkthrough of the system — built end-to-end as an independent project. It covers every component, every design decision, and the reasoning behind each choice.

---

## Table of Contents

1. [Project Origin & Purpose](#1-project-origin--purpose)
2. [What the System Does — User Perspective](#2-what-the-system-does--user-perspective)
3. [System Architecture](#3-system-architecture)
4. [Service 1 — Main Web Application](#4-service-1--main-web-application)
5. [Service 2 — RAG Service](#5-service-2--rag-service)
6. [Service 3 — Database Service](#6-service-3--database-service)
7. [Service 4 — WhatsApp Service](#7-service-4--whatsapp-service)
8. [Database Schema — All 9 Tables](#8-database-schema--all-9-tables)
9. [AI Design — The Agronomist Persona](#9-ai-design--the-agronomist-persona)
10. [Knowledge Base Builder Engine](#10-knowledge-base-builder-engine)
11. [GDPR Compliance System](#11-gdpr-compliance-system)
12. [Image Analysis Pipeline](#12-image-analysis-pipeline)
13. [Security Architecture](#13-security-architecture)
14. [Cloud Infrastructure](#14-cloud-infrastructure)
15. [Key Engineering Decisions](#15-key-engineering-decisions)
16. [Full Tech Stack](#16-full-tech-stack)
17. [Contact & Collaboration](#17-contact--collaboration)

---

## 1. Project Origin & Purpose

This project was built to solve a real problem: **Italian small-scale farmers — especially vineyard and orchard growers — have no affordable, accessible, expert agronomic guidance at the moment they need it.**

Agricultural consultants are expensive and not always available. Government resources (SIAN, CAA) are bureaucratic and hard to navigate. Most farmers rely on word-of-mouth or trial and error when dealing with crop diseases, pest attacks, or regulatory compliance.

The goal was to build an AI assistant that:
- Speaks **Italian naturally** — not translated, not robotic
- Understands the **local context** — commune, crop type, current season, farming method
- Is available **24/7** on the tools farmers already use — **WhatsApp** is ubiquitous in rural Italy
- Handles **real agronomic scenarios** — disease diagnosis, treatment planning, product dosage, field records
- Is **GDPR-compliant** from day one — required for any Italian user-facing product

The system was designed, built, deployed, and tested as a fully working product on Google Cloud Platform.

---

## 2. What the System Does — User Perspective

### Web User Journey

1. User visits the web application
2. Enters their email — account is created automatically on first visit
3. Completes onboarding: name, location (comune), main crops, farming method (organic/conventional/biodynamic)
4. Accepts GDPR terms, privacy policy, and optionally marketing consent — each tracked independently
5. Enters the chat interface and starts conversing with the AI agronomist
6. The AI responds in Italian, personalised to their specific context
7. Conversation history is persisted across sessions — they can return and continue where they left off
8. Multiple conversations can be created, each with an auto-generated title

### WhatsApp User Journey

1. User sends any message to the system's WhatsApp number (via Twilio)
2. If they are a new user, they receive a welcome message in Italian with a **secure one-time onboarding link**
3. The link opens a web form where they complete their profile (name, location, crops, farming method, GDPR consent)
4. The token is single-use and expires in 24 hours
5. Once onboarding is complete, they return to WhatsApp and can start chatting immediately
6. The AI processes their message and responds directly in WhatsApp — usually within seconds
7. They can send **photos of diseased plants** — the AI analyses the image and provides a visual diagnosis
8. Conversation history is maintained across all WhatsApp sessions

### A Realistic Conversation Example

> **Farmer:** "Ciao, la previsione dice pioggia nei prossimi giorni. Il Sangiovese è in fase di pre-fioritura. Cosa mi consigli?"
>
> **AI:** "Ciao! Capisco bene l'ansia, maggio a Montepulciano è un mese meraviglioso, ma per chi lavora in vigna è sempre una lotteria con queste piogge improvvise. In questa fase pre-fiorale, che è estremamente delicata, dobbiamo anticipare il problema. Ti suggerisco una strategia su tre fronti: Romeo, che agisce come un 'vaccino' per la vite stimolando le difese naturali; Coprantol Hi Bio 2.0, un idrossido di rame che crea una barriera di contatto contro la peronospora; e Tiovit Jet, uno zolfo micronizzato per tenere a bada l'oidio..."

This is the tone and depth the system delivers — practical, personalised, grounded in the user's actual location and crop phase.

---

## 3. System Architecture

### Overview

The system is built as **4 independent microservices** deployed on Google Cloud Run. Each service is a separate Docker container with its own codebase, dependencies, Dockerfile, and Cloud Build configuration.

```
User (Web Browser)             User (WhatsApp)
        │                             │
        ▼                             ▼
┌───────────────────┐     ┌───────────────────────┐
│   Main Web App    │     │   WhatsApp Service     │
│   Python / Flask  │     │   Python / Flask       │
│                   │     │                        │
│  - Web UI         │     │  - Twilio webhook      │
│  - Email auth     │     │  - Sig verification    │
│  - Session mgmt   │     │  - Async processing    │
│  - Orchestration  │     │  - Image pipeline      │
└────────┬──────────┘     └──────────┬─────────────┘
         │                           │
         └─────────────┬─────────────┘
                       │  Internal HTTP calls
            ┌──────────┴──────────┐
            │                     │
            ▼                     ▼
   ┌─────────────────┐   ┌──────────────────────┐
   │   RAG Service   │   │      DB Service       │
   │   Python/Flask  │   │   Python / Flask      │
   │                 │   │                       │
   │  ChromaDB       │   │  SQLAlchemy ORM       │
   │  GPT-4o         │   │  PostgreSQL 15        │
   │  LangFuse       │   │  REST API (9 tables)  │
   └────────┬────────┘   └──────────┬────────────┘
            │                       │
            ▼                       ▼
      Google Cloud            Google Cloud SQL
      Storage (GCS)           PostgreSQL 15
      ChromaDB backup         Managed DB
      Image storage
```

### Why Microservices?

Three concrete reasons drove this choice over a monolith:

**1. Independent resource scaling.** The RAG service is the most resource-intensive component — it runs ChromaDB in-process and makes OpenAI API calls that can take several seconds. It needs 2 CPU cores and 2GB RAM. The WhatsApp service is lightweight — it receives webhooks and delegates work. Separating them means each container is right-sized. A monolith would need to be sized for the most demanding component at all times.

**2. Cost-to-zero on Cloud Run.** Cloud Run containers scale to zero when idle. An Italian agricultural assistant has seasonal traffic — heavy in spring and summer (growing season), quiet in winter. With 4 separate services, each one independently scales to zero during idle periods. A monolith would need to keep its most expensive instance running.

**3. Fault isolation.** If the RAG service hits an OpenAI rate limit or has a slow cold start, the web app can still respond with a graceful degradation message. The WhatsApp service can still receive and queue messages. Services don't share failure modes.

---

## 4. Service 1 — Main Web Application

**Directory:** `app/`
**Framework:** Flask (Python 3.11)
**Responsibilities:** Web UI, user authentication, session management, orchestration

### Authentication

The system uses **email-based authentication** without passwords. A user enters their email:
- If they exist in the database, they are logged in
- If they don't exist, an account is created automatically and they are sent to onboarding

This choice was deliberate for the target audience — Italian farmers are not comfortable managing passwords. Email is sufficient for the use case.

### Custom Server-Side Session Management

This was one of the more complex engineering decisions. Cloud Run is stateless — requests can hit different container instances. Standard Flask sessions use signed cookies, which work across instances, but storing user state in cookies has security implications and size limits.

The solution was a **fully custom session implementation**:

- A `DatabaseSession` class (subclass of Python's `dict`) tracks modifications via overridden `__setitem__`, `__delitem__`, `update`, `pop`, and `clear` methods
- On each request, a `before_request` hook reads the `viridia_session_id` cookie, calls the DB service to load the session data, and hydrates the `DatabaseSession` object
- On each response, an `after_request` hook checks if the session was modified — if so, it writes the updated data back to the DB
- New sessions are created with a UUID, stored in the DB, and set as a secure HTTPS-only cookie
- Sessions expire after 24 hours — enforced at the DB level

This gives the security of server-side sessions with the statelessness required by Cloud Run.

### Orchestration

The web app does not contain AI logic directly. It acts as an orchestrator:
- Receives user messages from the chat interface
- Calls the **DB service** to retrieve conversation history
- Calls the **RAG service** with the message, history, and user context
- Receives the AI response from the RAG service
- Calls the **DB service** to persist the chat log
- Returns the response to the browser

The only OpenAI call made directly by the web app is for **conversation title generation** — a lightweight GPT-4o-mini call to generate a short, descriptive title for a new conversation based on its first message.

### Web UI

- Responsive design with dark mode support
- Real-time chat interface (streaming-style feel)
- Conversation sidebar — list of past conversations, ability to switch between them
- Onboarding form — multi-step flow collecting name, location (comune), crops, farming method
- Legal acceptance step — three separate consent checkboxes (terms, privacy, marketing), each independently tracked

---

## 5. Service 2 — RAG Service

**Directory:** `rag-service/`
**Framework:** Flask (Python 3.11)
**Responsibilities:** Vector search, prompt assembly, OpenAI GPT-4o calls, LangFuse tracking

### What RAG Is and Why It's Used Here

RAG — Retrieval-Augmented Generation — is the technique of retrieving relevant documents from a knowledge base and injecting them into the prompt before asking the LLM to generate a response.

Without RAG, GPT-4o would answer from its training data — which includes general agricultural knowledge but not Italian-specific regulations, local product registrations, regional crop advice, or up-to-date treatment protocols.

With RAG, the system retrieves the most relevant chunks from a curated knowledge base of Italian agricultural documents before every query. The LLM then generates a response grounded in that specific, verified content rather than hallucinating from general knowledge.

### ChromaDB — The Vector Store

ChromaDB is an open-source vector database that runs **in-process** — it's a Python library, not a separate service. This was a deliberate choice over hosted solutions like Pinecone:

- No additional service to manage or pay for
- No network round-trip for vector search — sub-millisecond retrieval
- The knowledge base (Italian agricultural documents for a specific region) is small enough that in-process is perfectly fast
- ChromaDB data is a directory of files — easy to backup to GCS and sync on startup

**Cold start syncing:** When a RAG service container starts cold, it downloads the full ChromaDB directory from Google Cloud Storage before accepting any requests. This adds 10-30 seconds to cold starts but ensures every container instance has identical, up-to-date knowledge.

### The Prompt Architecture

The system prompt is the core IP of this project. It defines the AI persona, its constraints, its response style, and its boundaries. It uses template variables that are filled at runtime:

```
{{username}}         → e.g., "Mario Rossi"
{{municipality}}     → e.g., "Montepulciano (SI)"
{{crops}}            → e.g., "Vite Sangiovese, olivo"
{{type_of_management}} → e.g., "Biologico"
{{dd/mm/yyyy}}       → e.g., "25/05/2024"
```

Every single response is personalised with the user's real context. The AI doesn't give generic advice — it gives advice specific to Montepulciano in late May for an organic Sangiovese grower.

A **separate image analysis prompt** is used when the user sends a photo — it instructs the model to analyse the visual content specifically for plant disease symptoms, pests, or nutrient deficiencies.

### Model Configuration

```json
{
  "model_name": "gpt-4o",
  "max_tokens": 3000,
  "temperature": 0.2,
  "presence_penalty": 0.3,
  "frequency_penalty": 0.6
}
```

- **Temperature 0.2** — low randomness for consistent, factual agronomic advice. The AI should not "get creative" with pesticide recommendations.
- **Presence and frequency penalties** — discourage repetition of phrases across a long conversation. Without these, the model tends to repeat its opening acknowledgement formula.
- **3000 max tokens** — allows detailed multi-step treatment plans without truncation.

### LangFuse Integration

Every conversation turn is tracked in LangFuse — an open-source LLM observability platform. This captures:
- Full prompt sent to OpenAI (including system prompt, RAG context, history)
- Model response
- Token usage (prompt tokens + completion tokens)
- Latency
- User identifier

This enabled post-hoc analysis of hallucinations, response quality, and cost per conversation.

---

## 6. Service 3 — Database Service

**Directory:** `db-service/`
**Framework:** Flask (Python 3.11) + SQLAlchemy ORM
**Database:** PostgreSQL 15 on Google Cloud SQL
**Responsibilities:** All database operations, exposed as a REST API

### Why a Separate DB Service?

Having a dedicated database microservice — rather than each service connecting directly to PostgreSQL — provides several benefits:

- **Single point of schema control** — only one service knows the database schema. Other services call HTTP endpoints, not raw SQL.
- **Connection pooling in one place** — PostgreSQL has a hard limit on concurrent connections. A single service manages the pool for all other services.
- **Easier to swap the database** — the web app and WhatsApp service don't know what database engine is running. Only the DB service does.
- **Cleaner separation** — no ORM models duplicated across services, no risk of schema drift between services.

### Route Blueprints

The DB service is organised into three Flask blueprints:

- **`whatsapp_routes`** — endpoints for all WhatsApp user operations (create user, get user, update profile, image quota management, conversation CRUD)
- **`onboarding_routes`** — endpoints for onboarding token creation, validation, and completion
- **`migration_routes`** — utility endpoints for running database migrations during deployment

### Key Endpoints

```
GET  /health                              → Health check (tests DB connection)
GET  /api/db/status                       → Stats (user count, conversation count, message count)

POST /api/db/users                        → Create web user
GET  /api/db/users/<email>                → Get web user
PUT  /api/db/users/<email>/location       → Update user location

POST /api/db/conversations                → Create conversation
GET  /api/db/conversations/<email>        → List user's conversations
GET  /api/db/conversations/<email>/active → Get active conversation

POST /api/db/chat-logs                    → Save a chat message pair
GET  /api/db/chat-logs/<conversation_id>  → Get conversation history

POST /api/db/sessions                     → Create a new session
GET  /api/db/sessions/<session_id>        → Load a session
PUT  /api/db/sessions/<session_id>        → Update session data
DELETE /api/db/sessions/<session_id>      → Delete session (logout)
```

---

## 7. Service 4 — WhatsApp Service

**Directory:** `whatsapp-service/`
**Framework:** Flask (Python 3.11)
**Responsibilities:** Twilio webhook handling, async message processing, image pipeline

### Webhook Security

Every incoming request from Twilio is verified using **Twilio's webhook signature**. Twilio signs each webhook request with an HMAC-SHA1 signature using your auth token. The service validates this signature before processing any message. Any request that fails signature verification is rejected with HTTP 403.

This prevents malicious actors from sending fake WhatsApp messages to the endpoint.

### Async Processing Pattern

Twilio's webhook has a **15-second hard timeout**. If the endpoint doesn't return HTTP 200 within 15 seconds, Twilio considers the delivery failed and may retry.

A complete message processing cycle involves:
1. Database lookup (user profile + conversation history)
2. ChromaDB vector search (usually fast, but varies)
3. OpenAI GPT-4o API call (3-15 seconds)
4. Database write (chat log + query log)
5. Twilio outbound message send

This easily exceeds 15 seconds for complex queries.

**Solution:** The webhook handler validates the request, extracts the message data, spawns a background daemon thread, and immediately returns HTTP 200 to Twilio. The background thread then runs the full processing pipeline and sends the AI response via Twilio's outbound API when ready.

```python
# Webhook handler (simplified)
def webhook():
    verify_signature(request)          # Validate Twilio signature
    extract_message_data(request)      # Parse phone number, message, media
    spawn_background_thread(process)   # Non-blocking
    return Response(status=200)        # Immediate response to Twilio

# Background thread
def process(phone_number, message, webhook_data):
    user = get_or_create_user(phone_number)
    check_onboarding(user)
    handle_images_if_present(webhook_data)
    history = get_conversation_history(user)
    response = call_rag_service(message, history, user.context)
    send_whatsapp_message(phone_number, response)
    save_to_database(user, message, response)
```

If the background thread crashes, the error handler attempts to send the user a polite fallback message in Italian.

### Message Formatting

WhatsApp has a specific set of text formatting rules — bold, italic, bullet points — that differ from standard markdown. A dedicated `message_formatter.py` module handles converting AI responses to WhatsApp-compatible formatting, and splitting long responses across multiple messages (WhatsApp has a per-message character limit).

### WhatsApp Onboarding Flow

New WhatsApp users cannot be onboarded directly over WhatsApp — collecting structured information (name, location, crops, GDPR consent) over a text conversation is unreliable and poor UX.

Instead, the system uses a **web-based onboarding flow with a secure token**:

1. New user messages the WhatsApp number
2. System detects `onboarding_completed = False` for this phone number
3. System generates a UUID token, stores it in `onboarding_tokens` with a 24-hour expiry
4. Sends the user a WhatsApp message: *"Benvenuto! Per iniziare, completa il tuo profilo qui: https://app.../onboarding?token=UUID"*
5. User opens the link on their phone, fills in the web form, accepts GDPR terms
6. Token is marked as `completed` — it cannot be used again
7. WhatsApp user record is updated with the profile data
8. User returns to WhatsApp and the AI now has their full context

---

## 8. Database Schema — All 9 Tables

### Web Domain Tables

#### `users`
| Column | Type | Notes |
|---|---|---|
| `id` | Integer PK | Auto-increment |
| `email` | String(255) | Unique, indexed |
| `password_hash` | String(255) | Nullable — not required for basic auth |
| `name` | String(255) | Collected during onboarding |
| `location` | String(255) | Free-text location |
| `comune` | String(255) | Italian municipality — used for AI personalisation |
| `farming_type` | String(255) | Biologico / Convenzionale / Biodinamico |
| `onboarding_completed` | Boolean | Gates access to chat |
| `onboarding_completed_at` | DateTime | Timestamped |
| `terms_accepted` | Boolean | GDPR |
| `terms_accepted_at` | DateTime | GDPR |
| `privacy_accepted` | Boolean | GDPR |
| `privacy_accepted_at` | DateTime | GDPR |
| `marketing_accepted` | Boolean | GDPR — optional |
| `marketing_accepted_at` | DateTime | GDPR |
| `legal_acceptance_ip` | String(50) | GDPR audit |
| `legal_acceptance_user_agent` | Text | GDPR audit |
| `created_at` | DateTime | Server default |

#### `conversations`
| Column | Type | Notes |
|---|---|---|
| `id` | Integer PK | |
| `user_email` | String FK → users | Indexed |
| `title` | String(500) | Auto-generated by GPT-4o-mini |
| `message_count` | Integer | Updated on each message |
| `is_active` | Boolean | Only one active conversation per user |
| `created_at` | DateTime | |

#### `chat_logs`
| Column | Type | Notes |
|---|---|---|
| `id` | Integer PK | |
| `user_email` | String FK → users | Indexed |
| `conversation_id` | Integer FK → conversations | Indexed |
| `user_message` | Text | |
| `ai_response` | Text | |
| `timestamp` | DateTime | |

#### `sessions`
| Column | Type | Notes |
|---|---|---|
| `id` | String(255) PK | UUID |
| `user_email` | String FK → users | Indexed |
| `data` | Text | JSON-encoded session dict |
| `created_at` | DateTime | |
| `expires_at` | DateTime | 24hr TTL, indexed |
| `last_accessed` | DateTime | Updated on every load |

#### `query_logs`
| Column | Type | Notes |
|---|---|---|
| `id` | Integer PK | |
| `query` | Text | The user's message |
| `mode` | String(20) | `web` or `whatsapp` |
| `system_prompt` | Text | Full system prompt with context filled in |
| `context_extracted` | Text | ChromaDB chunks retrieved |
| `response` | Text | AI response |
| `user_identifier` | String(255) | Email (web) or phone number (WhatsApp) |
| `conversation_id` | Integer | |
| `retrieval_time_ms` | Integer | ChromaDB search duration |
| `llm_time_ms` | Integer | OpenAI call duration |
| `total_time_ms` | Integer | End-to-end duration |
| `tokens_used` | Integer | Total tokens consumed |
| `timestamp` | DateTime | Indexed |

#### `onboarding_tokens`
| Column | Type | Notes |
|---|---|---|
| `id` | Integer PK | |
| `token` | String(36) | UUID, unique, indexed |
| `user_identifier` | String(255) | Email or phone number |
| `user_type` | String(20) | `web` or `whatsapp` |
| `status` | String(20) | `pending` / `completed` / `expired` |
| `expires_at` | DateTime | 24hr from creation, indexed |
| `completed_at` | DateTime | Set on successful use |
| `ip_address` | String(45) | GDPR audit |
| `user_agent` | Text | GDPR audit |

#### `consent_logs`
| Column | Type | Notes |
|---|---|---|
| `id` | Integer PK | |
| `user_identifier` | String(255) | Email or phone |
| `user_type` | String(20) | `web` or `whatsapp` |
| `consent_type` | String(50) | `terms_of_use` / `privacy_policy` / `marketing` |
| `action` | String(20) | `accepted` or `revoked` |
| `method` | String(50) | `checkbox` / `button` / `api` |
| `timestamp` | DateTime | Indexed |
| `ip_address` | String(45) | |
| `user_agent` | Text | |
| `consent_version` | String(20) | Document version |
| `consent_metadata` | Text | JSON — additional context |

### WhatsApp Domain Tables

#### `whatsapp_users`
| Column | Type | Notes |
|---|---|---|
| `id` | Integer PK | |
| `phone_number` | String(20) | Unique, indexed — primary identifier |
| `name` | String(255) | From onboarding |
| `location` | String(255) | |
| `email` | String(255) | Optional, collected during onboarding |
| `comune` | String(255) | For AI personalisation |
| `farming_type` | String(255) | |
| `onboarding_completed` | Boolean | Indexed — checked on every message |
| `onboarding_completed_at` | DateTime | |
| `terms_accepted` | Boolean | GDPR |
| `privacy_accepted` | Boolean | GDPR |
| `marketing_accepted` | Boolean | GDPR — optional |
| `media_urls` | JSONB | Array of image URLs stored in GCS |
| `daily_image_count` | Integer | Resets daily |
| `last_image_date` | DateTime | Used to detect day rollover |
| `created_at` | DateTime | |
| `last_active` | DateTime | Updated on every message |

#### `whatsapp_conversations`
Mirrors the web `conversations` table — linked to `whatsapp_users` by phone number instead of email.

---

## 9. AI Design — The Agronomist Persona

### The Core Problem with General-Purpose LLMs for This Use Case

A vanilla GPT-4o prompt like *"You are an agricultural assistant. Answer farming questions."* produces responses that are:
- Generic — not specific to Italian regulations, Italian products, or Italian climate
- Sometimes in English — requiring extra prompting
- Structured as bullet lists — not how a human expert speaks
- Overconfident — the model will invent pesticide registration numbers if asked
- Out of scope — it will answer questions about human medicine, law, or animal husbandry

The system prompt is the solution to all of these problems.

### Persona Design

The AI is designed as a **neighbour-expert**: someone with deep professional knowledge who speaks casually, understands the farmer's reality, and gives practical advice — not textbook answers.

Key persona traits:
- **Italian-only** — the system prompt explicitly requires all responses in Italian, with no exceptions
- **Warm and local** — uses natural Italian phrasing, acknowledges the farmer's specific situation ("Capisco bene l'ansia, maggio a Montepulciano...")
- **Practical** — focuses on "how-to" rather than theory. "Do this" rather than "research suggests..."
- **Narrative** — responds in paragraphs, not bullet lists, unless the advice has 3+ sequential steps that are hard to follow in prose

### Hard Boundaries

The system prompt enforces strict constraints:
- **Human health** → redirects to GP ("Io sono un agronomo, non un medico!")
- **Veterinary** → redirects to vet
- **Tax/legal/accounting** → redirects to CAA or commercialista
- **Live market prices** → directs to ISMEA or Chamber of Commerce
- **Future weather** → discusses seasonal patterns only, never makes forecasts
- **Regulatory citations** → never invents law numbers; always says "verifica su SIAN..." rather than claiming to have checked
- **Product recommendations** → only when explicitly asked; focuses on active substance, not brand name; always includes PPE reminder

### Context Personalisation

Every response is personalised via template variables filled at runtime:

```
User: Mario Rossi
Location: Montepulciano (SI), Tuscany
Main crops: Vite Sangiovese, Vite Trebbiano
Farming method: Biologico
Current date: 25/05/2024
```

This context is injected into the system prompt for every single API call. The AI knows it's talking to Mario in Montepulciano in late May — it can reference the local microclimate, the Sangiovese flowering phase, and the relevant organic treatment windows without the user having to explain any of it.

### Response Style

The prompt specifies a **narrative arc** for every response:
1. **Warm-up** — acknowledge the user's specific situation, weave in their context
2. **The fix** — explain what's happening and what to do about it, in plain Italian
3. **Parting wisdom** — one high-value closing tip (timing trick, cost-saving, watch-out)

This makes responses feel like advice from a person, not output from a machine.

---

## 10. Knowledge Base Builder Engine

The ChromaDB vector database that powers every AI response is not static — it is compiled by a **separate, standalone Knowledge Base Builder Engine** that can rebuild or extend the knowledge base independently of the chatbot codebase.

### Why a Separate Engine?

Separating the knowledge base builder from the chatbot achieves two things:

1. **Domain experts can maintain the knowledge** — someone with agricultural expertise can write or update YAML documents without touching Python or the chatbot infrastructure
2. **Knowledge updates don't require code deployments** — rebuild the database, upload to GCS, and the RAG service picks it up on the next container start

### The Knowledge Library — 12 Domains

The knowledge base covers the full scope of Italian agronomy across 12 hierarchical sections:

| Section | Coverage |
|---|---|
| A | Climate, soil, ecology |
| B | Botany, plant physiology, agricultural genetics |
| C | Agronomy and territory — techniques and planning |
| D | Herbaceous vegetable production (field crops, horticulture) |
| E | Viticulture, fruit growing, arboreal crops |
| F | General and special forestry |
| **G** | **Crop adversities and plant disease defence** ← core for diagnosis |
| H | Animal husbandry and livestock |
| I | Agri-food industries (enology, dairy, olive oil, HACCP) |
| L | Rural engineering and agricultural mechanization |
| M | Agricultural economics, policy, land law |
| N | Mathematics, statistics, agricultural modelling |

The directory hierarchy itself encodes metadata — `domain/subdomain/topic/document.txt` — and the engine extracts `hierarchical_code` from the path so every ChromaDB chunk knows its precise location in the knowledge taxonomy.

### YAML Document Format

Documents are written in a **custom YAML schema** that encodes semantic boundaries directly in the structure:

```yaml
Peronospora_della_vite:
  definizione: "La peronospora (Plasmopara viticola) è una malattia crittogamica..."
  sintomi:
    foglie: "Macchie oleose sulla pagina superiore..."
    grappoli: "Disseccamento nei casi gravi..."
  condizioni_favorevoli: "Temperature 18-25°C con umidità >80%..."
  strategie_di_difesa:
    biologica: ["Rame (idrossido)", "Fosfonati"]
    convenzionale: ["Mancozeb", "Metalaxyl"]
```

### Semantic Chunking

The engine chunks by **YAML subsection boundaries**, not by character count. Each subsection (`sintomi`, `strategie_di_difesa`, etc.) becomes a separate, self-contained chunk with its parent context preserved. This produces significantly better retrieval quality than `RecursiveCharacterTextSplitter` — a farmer's question about downy mildew symptoms retrieves the complete `sintomi` block, not a fragment cut at an arbitrary character limit.

### Metadata Enrichment

Every chunk stored in ChromaDB carries:
- **Hierarchical path metadata** — domain, subdomain, topic, hierarchical_code
- **Section classification** — definition / table / classification / process / mixed
- **Auto-detected topic tags** — 7 categories (difesa_fitosanitaria, gestione_suolo, gestione_acqua, coltivazione, gestione_agronomica, economia_politica, meccanizzazione)
- **Quality score** — 0.0 for placeholder sections, up to 1.7 for dense, structured content
- **Word count and oversized flag**

### Embedding Model

**OpenAI `text-embedding-3-large`** — 3072-dimensional vectors, OpenAI's highest-quality embedding model. The same model is used at both build time and query time — essential for consistent similarity scoring. Documents are processed in batches of 100 to respect OpenAI API token limits.

See [KNOWLEDGE_BASE_ENGINE.md](KNOWLEDGE_BASE_ENGINE.md) for the complete technical documentation.

---

## 11. GDPR Compliance System

GDPR compliance was built as a first-class feature — not added after the fact.

### What Must Be Compliant

Under GDPR (and the Italian implementation via D.Lgs. 196/2003 as amended), the system must:
- Obtain **explicit, informed, separate consent** for each processing purpose
- Record **when, how, from where, and in what version** consent was given
- Allow users to **revoke consent** at any time
- Maintain an **immutable audit trail** of all consent events

### Implementation

**Three separate consent types**, each independently tracked:
- `terms_of_use` — required to use the service
- `privacy_policy` — required to use the service
- `marketing` — optional, separate checkbox

**`consent_logs` is append-only.** Consent is never deleted or updated — a new row is always inserted. If a user accepts, then later revokes, there are two rows: one `accepted` and one `revoked`. The current state is derived from the latest record. This is what GDPR Article 7(1) requires — the ability to demonstrate that consent was obtained.

**What's recorded per consent event:**
- Consent type and action (accepted/revoked)
- Method (checkbox / button / API)
- Timestamp
- IP address
- User-agent string (browser/device)
- Consent document version (so you can prove what the user actually agreed to)

**Onboarding tokens** add an extra layer for WhatsApp: the token is single-use, expires in 24 hours, and the IP address and user-agent of the person who completed the form are recorded — confirming it was the actual user, not someone who intercepted the link.

---

## 12. Image Analysis Pipeline

Farmers frequently need to diagnose crop problems from visual inspection. A major capability of the system is the ability for WhatsApp users to send a photo of a diseased plant and receive an AI diagnosis.

### Full Pipeline

```
User sends image via WhatsApp
          │
          ▼
Twilio webhook includes MediaUrl0 + MediaContentType0 + NumMedia
          │
          ▼
WhatsApp service checks daily image quota (DB lookup)
          │
          ├── Quota exceeded → send polite limit message in Italian, stop
          │
          ▼
Download image bytes from Twilio
(requires Basic Auth with Twilio credentials)
          │
          ▼
Upload image to Google Cloud Storage
bucket: viridia-whatsapp-images
path: {phone_number}/{date}/{timestamp}.{ext}
          │
          ▼
Generate signed URL for the GCS object
(time-limited access URL — GPT-4o needs to fetch the image)
          │
          ▼
Call RAG service with:
- Text message (may be empty — user sent only a photo)
- GCS signed URL
- User context
- Conversation history
          │
          ▼
RAG service uses image_prompt (separate from text prompt)
GPT-4o vision enabled — model receives both text and image URL
          │
          ▼
AI analyses image: identifies disease, pest, deficiency
Generates detailed diagnosis in Italian
          │
          ▼
Response sent via Twilio to WhatsApp user
DB log updated: chat_log + query_log + daily_image_count incremented
```

### Quota System

Each WhatsApp user has a `daily_image_count` and `last_image_date` field. On each image message:
- If `last_image_date` is not today → reset `daily_image_count` to 0, set `last_image_date` to today
- If `daily_image_count` >= daily limit → reject with a polite message
- Otherwise → process image and increment counter

This prevents abuse while giving genuine users ample capacity for daily farm inspections.

---

## 13. Security Architecture

| Layer | Mechanism |
|---|---|
| **Incoming WhatsApp messages** | Twilio HMAC-SHA1 webhook signature verification on every request |
| **Onboarding links** | UUID one-time tokens, 24hr expiry, single-use enforcement |
| **Web sessions** | Server-side DB-backed sessions — UUID cookie, no session data on client |
| **API credentials** | Google Secret Manager — injected at runtime, never in code or images |
| **Transport** | HTTPS enforced across all Cloud Run endpoints |
| **Database access** | SQLAlchemy ORM throughout — no raw SQL, no injection surface |
| **Inter-service communication** | Internal HTTP between Cloud Run services (not exposed publicly) |
| **GDPR** | Consent audit trail, IP logging, 24hr token expiry, append-only consent log |

---

## 14. Cloud Infrastructure

### Google Cloud Run

Each of the 4 services runs as an independent Cloud Run service:

| Service | CPU | Memory | Min Instances | Max Instances |
|---|---|---|---|---|
| Web App | 1 | 1Gi | 0 | 10 |
| RAG Service | 2 | 2Gi | 0 | 5 |
| DB Service | 1 | 1Gi | 0 | 5 |
| WhatsApp Service | 1 | 1Gi | 0 | 10 |

All services scale to zero when idle — cost-efficient for seasonal traffic patterns.

### Google Cloud SQL

PostgreSQL 15 managed instance:
- Automatic daily backups
- Point-in-time recovery
- Private IP connection from Cloud Run (no public exposure)
- Connection via Cloud SQL Auth Proxy

### Google Cloud Storage

Two buckets:
- **ChromaDB backup bucket** — synced to RAG service containers on cold start
- **`viridia-whatsapp-images`** — stores all images sent by WhatsApp users

### Google Cloud Build

Each service has a `cloudbuild.yaml` that:
1. Builds the Docker image
2. Pushes to Google Container Registry
3. Deploys to Cloud Run with updated environment variables

### Google Secret Manager

All sensitive values managed as secrets:
- `OPENAI_API_KEY`
- `TWILIO_ACCOUNT_SID`
- `TWILIO_AUTH_TOKEN`
- `DB_CONNECTION_STRING`
- `SECRET_KEY` (Flask session signing)
- `CHROMA_GCS_BUCKET`

---

## 15. Key Engineering Decisions

### Decision 1: ChromaDB in-process vs. hosted vector DB
**Chose:** ChromaDB (in-process)
**Why:** The knowledge base is a fixed, moderate-sized collection of Italian agricultural documents. In-process retrieval is faster (no network), cheaper (no external service), and simpler (it's just a directory). The trade-off is that each RAG container instance carries its own copy of the database — acceptable given the read-only, GCS-synced nature of the data.

### Decision 2: Server-side sessions vs. signed cookie sessions
**Chose:** Custom server-side DB-backed sessions
**Why:** Cloud Run's statelessness means any container instance might serve any request. Signed cookies work across instances, but storing real user state client-side has security implications. DB-backed sessions give full server-side control, instant invalidation capability, and no size limits. The trade-off is an additional DB round-trip per request — acceptable given the DB service is internal and fast.

### Decision 3: Async WhatsApp processing via background threads
**Chose:** Daemon threads spawned from the webhook handler
**Why:** Twilio's 15-second webhook timeout cannot accommodate a full RAG + GPT-4o cycle. The background thread approach was chosen over a task queue (Celery, Cloud Tasks) because it's simpler to deploy (no additional infrastructure), adequate for the expected message volume, and Cloud Run's per-instance request concurrency handles multiple simultaneous messages.

### Decision 4: Separate user tables for web vs. WhatsApp
**Chose:** `users` and `whatsapp_users` as separate tables
**Why:** The two user types have different primary identifiers (email vs. phone), different onboarding flows, and different data fields (e.g., WhatsApp needs daily image quota tracking; web needs password hash field). A single table with nullable fields for each type would be messy and create ambiguity in queries. Separate tables are cleaner and allow the two domains to evolve independently.

### Decision 5: Append-only consent logs
**Chose:** Never update or delete consent records
**Why:** GDPR Article 7(1) requires the ability to demonstrate that consent was obtained at the time of processing. An update would destroy the audit trail. An append-only log preserves the full history — you can always see exactly what a user consented to, when, from what IP, and in what version of the document.

---

## 16. Full Tech Stack

### Backend
- **Python 3.11** — all 4 services
- **Flask** — web framework
- **SQLAlchemy** — ORM for PostgreSQL
- **Gunicorn** — WSGI server for production

### AI / ML
- **OpenAI GPT-4o** — main LLM (chat + vision)
- **OpenAI GPT-4o-mini** — conversation title generation (lightweight, cost-efficient)
- **ChromaDB** — in-process vector store for RAG
- **LangFuse** — LLM observability, conversation tracking, token analytics

### Database
- **PostgreSQL 15** — primary database (Google Cloud SQL)

### Cloud Infrastructure (GCP)
- **Cloud Run** — serverless container hosting (4 services)
- **Cloud SQL** — managed PostgreSQL
- **Cloud Storage (GCS)** — ChromaDB backup + WhatsApp image storage
- **Secret Manager** — credentials management
- **Cloud Build** — CI/CD pipeline (build → push → deploy)
- **Container Registry** — Docker image storage

### Messaging
- **Twilio WhatsApp API** — WhatsApp message send/receive

### DevOps
- **Docker** — containerisation (one Dockerfile per service)
- **dotenv** — local environment variable management

---

## 17. Contact & Collaboration

This project was built end-to-end as an independent project — including architecture design, AI prompt engineering, cloud infrastructure, WhatsApp integration, GDPR compliance system, and full deployment on GCP.

### Open to

- **Technical collaboration** — extending this to other crops, regions (Spain, France, Greece), or other languages
- **Licensing** — deploying a tailored version for a specific agricultural business, cooperative, or agri-tech company
- **Consulting** — building similar RAG + WhatsApp AI systems for other domains or industries
- **Acquisition / IP transfer** — full handover of source code, knowledge base, system prompts, infrastructure config, and documentation

### What a Full Transfer Includes

| Asset | Description |
|---|---|
| Source code | All 4 microservices (Python / Flask) |
| System prompts | AI persona definition and domain constraints |
| ChromaDB knowledge base | Italian agricultural documents, regulations, product data |
| Database schema + migrations | All 9 tables, fully documented |
| Cloud infrastructure config | Dockerfiles, Cloud Build configs, GCP setup scripts |
| Deployment guide | Step-by-step guide for GCP deployment |
| GDPR compliance system | Consent tracking, audit logs, onboarding token flow |

### Contact

**Email:** giduthuri.jsd2@gmail.com
**LinkedIn:** [Deepak Giduthuri](https://www.linkedin.com/in/deepak-giduthuri-a88883188/)
**Book a call:** [calendly.com/js-deepakgiduthuri](https://calendly.com/js-deepakgiduthuri)

Response time: within a few business days. Or skip the back-and-forth and book a call directly.
