# Knowledge Base Builder Engine

The RAG knowledge base that powers the AI agronomist's responses is not hardcoded — it is built by a **separate, standalone Knowledge Base Builder Engine**. This engine processes a curated library of Italian agricultural documents and compiles them into a ChromaDB vector database that can be deployed to the chatbot's RAG service.

This means the knowledge base can be **extended, updated, or rebuilt at any time** without touching the chatbot code — you simply update the documents and re-run the builder.

---

## What It Does

The engine takes a structured library of Italian agricultural knowledge documents (written in a custom YAML schema), processes them into semantically meaningful chunks, generates vector embeddings for each chunk using OpenAI's `text-embedding-3-large` model, and writes the result into a ChromaDB vector database ready to be deployed to the RAG service.

```
KnowledgeBaseDocuments/
  (12 domain sections, hierarchically organised)
            │
            ▼
   YAML Loader (yaml_loader.py)
   - Parse YAML structure
   - Extract hierarchical metadata
   - Semantic chunking by subsection
            │
            ▼
   Ingestion Pipeline (ingestion.py)
   - Quality scoring & filtering
   - Topic classification
   - OpenAI text-embedding-3-large
   - Batch processing (100 docs/batch)
            │
            ▼
   ChromaDB Vector Database
   db/chroma_db_tuscany/
            │
            ▼
   Ready for upload to GCS →
   RAG service syncs on startup
```

---

## Knowledge Base Structure — 12 Domains

The knowledge base is organised into 12 academic sections covering the complete scope of Italian agronomy. Each section is a directory, each sub-topic is a subdirectory, and each document is a YAML-structured `.txt` file.

| Section | Domain |
|---|---|
| **A** | Climate, soil, and ecology |
| **B** | Botany, plant physiology, agricultural genetics |
| **C** | Agronomy and territory — techniques, technologies, planning |
| **D** | Herbaceous vegetable production (field crops, horticulture, floriculture) |
| **E** | Shrub and tree production — viticulture, fruit growing, ornamental |
| **F** | General and special forestry |
| **G** | Crop adversities and plant disease defence *(core section for disease diagnosis)* |
| **H** | Animal husbandry and livestock management |
| **I** | Agri-food industries — enology, dairy, olive oil, cereals, HACCP |
| **L** | Rural engineering and agricultural mechanization |
| **M** | Agricultural economics, policy, land valuation, and professional law |
| **N** | Appendix — mathematics, statistics, agricultural modelling |

The hierarchy within each section goes 3-4 levels deep:

```
A_Clima_suolo_ecologia/
  └── A.1_Meteorologia/
        └── 1_Meteo_e_clima/
              └── 1.1_Atmosfera.txt   ← YAML document
```

The directory path itself becomes metadata — the engine extracts `domain`, `subdomain`, `topic`, and `hierarchical_code` directly from the file path, so every chunk in ChromaDB knows exactly where in the agronomic knowledge hierarchy it came from.

---

## How the YAML Document Format Works

Agricultural knowledge is written in a **custom YAML schema** that preserves the hierarchical structure of the content. This is a deliberate choice over plain `.txt` or `.pdf` — YAML allows semantic boundaries to be encoded directly in the document structure, enabling the engine to chunk by meaning rather than by arbitrary character count.

Example document structure:
```yaml
Peronospora_della_vite:
  definizione:
    "La peronospora della vite (Plasmopara viticola) è una malattia
     crittogamica causata da un oomicete. Colpisce tutte le parti verdi..."

  sintomi:
    foglie: "Macchie oleose sulla pagina superiore, muffa bianca su quella inferiore..."
    grappoli: "Disseccamento e avvizzimento nei casi gravi..."
    germogli: "Imbrunimento e collasso dei nuovi getti..."

  condizioni_favorevoli:
    "Temperature tra 18-25°C con umidità relativa superiore all'80%..."

  strategie_di_difesa:
    biologica:
      - "Rame (idrossido, ossicloruro) — barriera di contatto preventiva"
      - "Fosfonati — stimolatori delle difese naturali della pianta"
    convenzionale:
      - "Mancozeb, folpet — funghicidi di contatto"
      - "Metalaxyl, dimethomorph — funghicidi sistemici"
```

The engine processes each top-level key as a section and each nested key as a subsection, creating one chunk per subsection — so `sintomi`, `condizioni_favorevoli`, and `strategie_di_difesa` each become separate, independently retrievable chunks with full metadata.

---

## Semantic Chunking — The Core Design Decision

Most RAG systems use `RecursiveCharacterTextSplitter` — they split documents at character limits regardless of meaning, often cutting mid-sentence or mid-concept.

This engine chunks **by YAML subsection boundaries instead**. Each chunk is a complete, self-contained piece of knowledge:

- The chunk boundary is always at a semantic boundary (a subsection heading)
- Each chunk includes its parent section as context header
- Chunks are never split mid-concept
- Oversized chunks (exceeding 1500 words) are flagged but kept intact rather than split arbitrarily

This produces much higher-quality retrieval — when a farmer asks about downy mildew symptoms, the retriever returns the complete `sintomi` subsection, not a fragment that starts mid-sentence from a character-limit split.

---

## Metadata Enrichment

Each chunk stored in ChromaDB carries rich metadata that enables filtered and scored retrieval:

| Metadata Field | Source | Example |
|---|---|---|
| `domain` | Directory path | `G_avversità_difesa_delle_colture` |
| `subdomain` | Directory path | `G.1_Parte_generale` |
| `topic` | Directory path | `1_Funghi_patogeni` |
| `hierarchical_code` | Directory path | `G.1/1/1.2` |
| `section` | YAML key | `Peronospora_della_vite` |
| `subsection` | YAML nested key | `strategie_di_difesa` |
| `section_type` | Auto-classified | `definition` / `table` / `classification` / `process` / `mixed` |
| `topics` | Keyword detection | `difesa_fitosanitaria, gestione_acqua` |
| `primary_topic` | Keyword detection | `difesa_fitosanitaria` |
| `quality_score` | Content analysis | `1.5` (high) / `0.0` (placeholder) |
| `is_placeholder` | Content analysis | `false` |
| `word_count` | Computed | `342` |
| `content_type` | Parse result | `yaml_structured` / `plaintext` |

### Quality Scoring

Every chunk receives a quality score before ingestion:
- Placeholder sections (`"sezione da completare"`) receive score `0.0` and are flagged — they can be filtered out at retrieval time
- Base score: `1.0`
- +0.3 for content > 500 words
- +0.2 for content > 1000 words
- +0.2 for structured content (lists, numbered items, specific terminology)

### Topic Classification

7 agricultural topic categories are auto-detected from content keywords:

| Topic | Keywords Detected |
|---|---|
| `difesa_fitosanitaria` | malattia, peronospora, oidio, trattamento, funghi, insetti, afidi... |
| `gestione_suolo` | suolo, fertilità, pH, sostanza organica, azoto, fosforo... |
| `gestione_acqua` | irrigazione, drenaggio, stress idrico, evapotraspirazione... |
| `coltivazione` | semina, cultivar, varietà, ciclo, raccolta, resa... |
| `gestione_agronomica` | potatura, sarchiatura, rotazione, lavorazione, densità... |
| `economia_politica` | PAC, PSR, normativa, certificazione, mercato... |
| `meccanizzazione` | trattore, aratro, seminatrice, meccanizzazione... |

---

## Embedding Model

**OpenAI `text-embedding-3-large`** — OpenAI's highest-quality text embedding model.

- 3072-dimensional embedding vectors
- Significantly better semantic similarity than `text-embedding-ada-002`
- Batch processing at 100 documents per API call to respect OpenAI token limits
- The same embedding model is used at both build time (ingestion) and query time (retrieval) — this is critical for consistent similarity scoring

---

## The Retriever

A `TuscanyRetriever` class wraps the ChromaDB instance for querying. It:
- Loads the persisted ChromaDB from disk
- Runs similarity search with configurable `k` (number of top results)
- Returns combined context text with `---` separators between chunks
- Logs full metadata for each retrieved chunk (source, hierarchy, section type, domain, content preview) — useful for debugging retrieval quality

---

## Running the Engine

```bash
# Install dependencies
pip install -r requirements_ingestion.txt

# Set your OpenAI API key in config/.env
# OPENAI_API_KEY=sk-...

# Build the full ChromaDB from scratch
python -c "from agronomist.ingestion import build_vector_db; build_vector_db(rebuild=True)"

# Test retrieval
python -c "
from agronomist.retriever import TuscanyRetriever
r = TuscanyRetriever()
print(r.retrieve_as_text('Come trattare la peronospora della vite in biologico?', k=5))
"
```

After building, the `db/chroma_db_tuscany/` directory is uploaded to Google Cloud Storage and the RAG service syncs it automatically on next cold start.

---

## Adding or Updating Knowledge

To add new content to the knowledge base:

1. Create a new `.txt` file in the appropriate section directory under `KnowledgeBaseDocuments/`
2. Write the content in YAML format following the existing document schema
3. Run `build_vector_db(rebuild=True)` to rebuild ChromaDB
4. Upload the new `db/chroma_db_tuscany/` to GCS
5. The RAG service picks it up on next container start — no code changes needed

This separation means the knowledge base can be maintained and grown independently of the chatbot codebase, by someone with agricultural domain expertise and no software engineering background — they only need to write YAML documents.

---

## Tech Stack

- **Python 3.11**
- **LangChain** — document processing and ChromaDB integration
- **ChromaDB** — vector store
- **OpenAI `text-embedding-3-large`** — embedding model
- **PyYAML** — YAML document parsing
- **Google Cloud Storage** — output destination for deployment
