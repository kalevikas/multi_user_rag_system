# Multi-User RAG System - Complete Project Documentation (Interview Guide)

## 1) Project Overview

This project is a **multi-user (multi-tenant) Retrieval-Augmented Generation (RAG) system** where each user/company gets an isolated knowledge base in Qdrant and can:

- Upload documents (`pdf`, `xlsx`, `xls`, `csv`, `docx`, `doc`, `txt`, `md`)
- Ingest web pages (scraping)
- Ingest REST API data (JSON)
- Ask questions and get grounded answers with cited sources
- Use streaming answers (token-by-token)

Main value proposition:

- **Data isolation per user/company**
- **Hybrid retrieval** (dense vectors + BM25 sparse search)
- **Cross-encoder reranking**
- **Context-only anti-hallucination prompting**

---

## 2) Tech Stack

- **Language:** Python
- **UIs:** Streamlit (`streamlit_app.py`), Flask (`web_app.py`)
- **LLM:** OpenAI (`gpt-4o-mini` by default)
- **Embeddings:** `BAAI/bge-large-en-v1.5` (default 1024-d)
- **Vector DB:** Qdrant (local Docker or Qdrant Cloud)
- **Sparse Search:** BM25 (`rank-bm25`)
- **Reranking:** `cross-encoder/ms-marco-MiniLM-L-6-v2`
- **Document model/orchestration:** LangChain core objects
- **Web scraping:** `requests` + BeautifulSoup (+ Selenium advanced mode)

---

## 3) Repository Structure (Important Files)

- `streamlit_app.py`: modern chat-style multi-user UI with persistent conversations
- `web_app.py`: Flask API/UI for multi-company ingestion + querying
- `src/rag_pipeline.py`: central orchestrator (ingest, retrieve, answer, stream)
- `src/company_manager.py`: user/company to Qdrant collection mapping
- `src/multi_format_processor.py`: file/url/api loaders -> `Document` objects
- `src/chunking.py`: chunking logic (runtime uses recursive chunking via `HybridChunker`)
- `src/embeddings.py`: embedding generation and query encoding
- `src/vector_store.py`: Qdrant create/upsert/search/filter/delete
- `src/hybrid_retriever.py`: dense + BM25 fusion (RRF)
- `src/reranker.py`: rerank top candidates with cross-encoder
- `src/llm_handler.py`: OpenAI wrapper + strict RAG prompt templates
- `src/chat_memory.py`: in-memory conversation memory for multi-turn context
- `config/config.yaml`: central configuration
- `data/companies.json`: tenant registry
- `data/conversations/*.json`: per-user conversation persistence (Streamlit)

---

## 4) Multi-User Design (Core Interview Topic)

### How user isolation is implemented

Each user/company is assigned a dedicated Qdrant collection:

- Display name: `"ABC Corp"`
- Slugged collection: `"company_abc_corp"`

This mapping is created and stored by `CompanyManager` in `data/companies.json`.

### Runtime flow

- UI selects/creates a user/company
- `get_pipeline(company)` in `src/rag_pipeline.py` returns a cached `CompanyRAGPipeline`
- Pipeline initializes with the specific collection from the company record
- All ingested chunks and all retrieval happen only in that collection

### What this gives you

- Logical tenant separation at vector index level
- Simpler operational model than shared-index filtering
- Easy deletion of one tenant (`delete_collection` + remove registry record)

### Important caveat

- This is **tenant scoping**, not true identity security
- No robust auth/session/JWT/RBAC is enforced in active routes

---

## 5) End-to-End Data Flow

## 5.1 Ingestion Flow

1. Data source loaded as `Document` list (`multi_format_processor`)
2. Pipeline adds tenant/source metadata (`company`, `source_type`, etc.)
3. Chunking via recursive splitter (`HybridChunker -> RecursiveChunker`)
4. Embeddings generated (`EmbeddingManager`)
5. Upsert into Qdrant with payload (`QdrantVectorStore.add_documents`)
6. BM25 index refreshed in retriever (`retriever.update_documents`)
7. Company stats updated (`doc_count`, source types)

## 5.2 Query Flow

1. User asks question
2. Query embedding generated
3. Dense vector search in Qdrant
4. Sparse BM25 retrieval
5. Fusion via RRF (reciprocal rank fusion)
6. Cross-encoder reranking
7. Top chunks -> context block
8. Prompt template enforces context-only answering
9. LLM generates final answer (normal or streaming mode)
10. Sources returned and shown in UI

## 5.3 Fallback Flow (anti-hallucination)

If no relevant chunks found, system returns a fixed "Not Found in Documents" response and avoids fabrication.

---

## 6) Ingestion Sources and Processing Logic

## 6.1 Files

Handled in `src/multi_format_processor.py`:

- PDF: PyMuPDF first, fallback to `pypdf`
- Excel/CSV: parsed into summary + row-level documents
- Word: paragraph and table extraction
- Text/Markdown: plain text ingestion

## 6.2 URL Scraping

- Light mode: `requests` + BeautifulSoup text extraction
- Advanced mode: Selenium crawler in `src/web_scraper.py`
- Optional depth/follow-links behavior in Flask flow

## 6.3 REST API Ingestion

- Generic endpoint call with method/headers/params/body
- Optional nested extraction by `json_path`
- JSON normalized into textual `Document` records

---

## 7) Retrieval and Ranking Internals

## 7.1 Embeddings

- Default model: `BAAI/bge-large-en-v1.5` (1024 dimensions)
- BGE-specific instruction prefixes are used for:
  - document embedding
  - query embedding
- Normalized embeddings are used for cosine-friendly similarity

## 7.2 Vector Search

`QdrantVectorStore.search` supports:

- Top-k
- Score threshold
- Metadata filtering (Qdrant filter conditions)

Payload stores text + metadata + source info for reconstruction/citations.

## 7.3 Sparse Search (BM25)

`BM25Retriever` tokenizes chunk text and scores lexical overlap.

Use case:

- Handles exact keywords/acronyms that dense retrieval may miss.

## 7.4 Fusion

`HybridRetriever._fusion_scores` uses RRF:

- Dense and sparse ranked lists are fused by reciprocal rank
- Avoids problematic score-scale normalization
- Produces final merged candidate set

## 7.5 Reranking

`RerankerPipeline`:

- Retrieve larger set (`initial_k`)
- Score query-document pairs with cross-encoder
- Return best `final_k`

Result: higher precision in final context sent to LLM.

---

## 8) Prompting Strategy and Hallucination Control

`src/llm_handler.py` defines a strict system prompt:

- Use only reference documents
- If insufficient context, return exact fallback
- Never invent facts
- Keep structured, source-aware answers

Pipeline-level fallback in `src/rag_pipeline.py` additionally handles empty retrieval.

---

## 9) Conversation Memory Model

Two memory layers exist:

- **In-process memory** (`ConversationMemory`): last N turns in RAM (used for prompt context)
- **Persisted history** (`streamlit_app.py`): JSON files per user in `data/conversations`

When switching conversation in UI, memory is repopulated from saved messages.

---

## 10) APIs (Flask)

Major endpoints in `web_app.py`:

- `GET /api/companies` - list tenants
- `POST /api/companies` - create/get tenant
- `DELETE /api/companies/<company_name>` - delete tenant and collection
- `POST /upload` - ingest file into tenant index
- `POST /scrape` - ingest URL content
- `POST /ingest-api` - ingest REST API data
- `POST /ask` - synchronous answer
- `POST /ask-stream` - SSE streaming answer
- `POST /clear-memory` - clear conversation memory
- `GET /collection-info` - Qdrant collection stats

---

## 11) Configuration and Environment

Primary config: `config/config.yaml`

Critical keys:

- `embedding.model_name`
- `vectorstore.vector_size`
- `vectorstore.host/port/use_cloud/cloud_url/api_key`
- `llm.model`
- `chunking.recursive.chunk_size/chunk_overlap`

Environment variables:

- `OPENAI_API_KEY` (required)
- `QDRANT_URL`, `QDRANT_API_KEY` (for cloud mode)

Key constraint: embedding dimensionality and Qdrant vector size must match.

---

## 12) How to Run

## 12.1 Local (Streamlit)

1. Install dependencies: `pip install -r requirements.txt`
2. Start Qdrant (or configure cloud)
3. Set `OPENAI_API_KEY`
4. Run: `streamlit run streamlit_app.py`

## 12.2 Local (Flask)

1. Same setup
2. Run: `python web_app.py`
3. Open `http://localhost:8080`

---

## 13) Architecture Strengths (Use in Interview)

- Clear tenant isolation model (one collection per tenant)
- Strong retrieval quality stack: dense + sparse + reranking
- Unified ingestion for files, websites, APIs
- Practical production features:
  - streaming responses
  - source display
  - persistent chat history
  - cloud deployment path
- Config-driven and modular component boundaries

---

## 14) Problems, Limitations, and Risks

## 14.1 Security and access control

- No robust authentication/authorization in active app paths
- Tenant selection in UI can be misused without identity enforcement
- Interview framing: "Designed for controlled/internal environments; next phase includes JWT/RBAC and audit trails."

## 14.2 Data consistency and lifecycle

- File deletion and vector deletion are not always strongly coupled in every flow
- Some UI warnings indicate storage/index mismatch risks

## 14.3 Pipeline state

- In-memory pipeline pool may grow in long-running processes for many tenants
- No explicit eviction strategy for pipeline instances/models

## 14.4 BM25 scope limitation

- BM25 index updates from newly ingested chunks in current process context
- Not a durable sparse index persisted independently across restarts

## 14.5 Operational overhead

- Embedding and reranking models are compute-heavy
- First-load latency can be high due to model download/init

## 14.6 Codebase quality debt

- Legacy entrypoints (`v1.py`, `oneurls.py`) can confuse maintainers
- `chunking.py` includes duplicate imports and unused chunker classes at runtime
- Some docs are partially stale vs current implementation
- Minimal automated tests for core RAG correctness

---

## 15) Performance and Scaling Considerations

## 15.1 Current performance levers

- Chunk size/overlap tuning
- `top_k` and reranking candidate sizes
- Embedding model size (large/base/small tradeoff)
- Batch sizes for embedding

## 15.2 Scaling roadmap

- Add async/background ingestion workers (Celery/RQ)
- Add queue-based retries and dead-letter handling
- Introduce connection pooling and health checks
- Add caching layer for common queries/embeddings
- Add horizontal scaling with stateless API workers
- Add observability (latency histograms, retrieval hit rate, token cost)

---

## 16) Suggested Production Hardening Plan

1. Implement auth (JWT/OAuth2), tenant-bound authorization checks on every endpoint
2. Add rate limiting and abuse controls
3. Add strict input validation and size limits per ingestion source
4. Add idempotent ingestion keys and document versioning
5. Add audit logging for ingest/query/delete events
6. Add integration tests for full ingest->query path
7. Add regression benchmarks (answer quality + latency + cost)

---

## 17) Testing Status and Gaps

Current:

- Limited test coverage (scraper-focused script)

Missing and recommended:

- Unit tests for chunking/retriever/fusion/reranker
- API contract tests for all Flask endpoints
- Tenant-isolation tests (cross-tenant data leakage checks)
- Failure-mode tests (Qdrant down, OpenAI timeout, bad files)
- Load and concurrency tests

---

## 18) High-Quality Interview Q&A Bank

## Q1: Why one collection per user/company?

Answer:
It simplifies tenant isolation and operational deletion. Each tenant maps to one Qdrant collection, so retrieval is naturally scoped and accidental cross-tenant recall risk is reduced compared to a shared collection with complex filters.

## Q2: Why hybrid retrieval?

Answer:
Dense retrieval captures semantics; BM25 captures exact keyword overlap. Combining both via RRF improves recall and robustness, especially for technical terms, abbreviations, and exact entity names.

## Q3: Why rerank after retrieval?

Answer:
Initial retrieval maximizes recall. Cross-encoder reranking improves precision by jointly scoring query-document pairs, so only the most relevant chunks are given to the LLM.

## Q4: How do you control hallucinations?

Answer:
The system prompt enforces context-only responses, and the pipeline explicitly returns a fixed fallback when relevant context is absent. This dual guardrail minimizes fabricated answers.

## Q5: How do you handle multi-turn context?

Answer:
A bounded in-memory conversation history is included in prompt construction. Streamlit additionally persists chats to disk per user and reloads memory when a conversation is reopened.

## Q6: Biggest current weakness?

Answer:
Security/auth is the main gap. Tenant separation exists at data layer, but strong identity and authorization controls are still needed for production-grade multi-user deployment.

## Q7: How would you scale this to enterprise level?

Answer:
Introduce auth/RBAC, async ingestion workers, robust observability, durable sparse indexing strategy, model/pipeline resource pooling, and comprehensive CI test suites with quality and latency SLOs.

## Q8: What failures did you plan for?

Answer:
Loader-level extraction failures, empty retrieval fallback, cloud/local Qdrant switching, and streaming error handling are addressed. Next improvements include retries/circuit breakers and stronger partial-failure recovery.

## Q9: How do you maintain source traceability?

Answer:
Every chunk stores metadata in Qdrant payload (`source`, `file_name`, `api_url`, etc.). Response assembly converts these into user-friendly source labels shown with answers.

## Q10: Why this embedding model?

Answer:
`bge-large-en-v1.5` gives strong retrieval quality and aligns with cosine similarity workflows. The system is config-driven, so we can switch to lighter models for cost/latency constraints.

---

## 19) Interview Storyline (2-Minute Pitch)

"I built a multi-user RAG platform where each user has an isolated Qdrant vector index. The ingestion pipeline supports documents, web pages, and API data, normalizes everything into chunked embeddings, and stores rich metadata for traceability. During query time, I combine dense vector search with BM25 sparse search, fuse candidates with RRF, and rerank them using a cross-encoder before sending context to GPT-4o-mini. I added strict context-only prompting and a deterministic fallback to control hallucinations. The system has both Streamlit and Flask interfaces, supports streaming answers, and persists per-user conversations. I also identified production gaps such as authentication, broader testing, and operational hardening, and documented a concrete roadmap to close them."

---

## 20) Quick Revision Checklist (Before Interview)

- Explain full ingest -> retrieve -> answer path from memory
- Explain why dense+sparse+rereank beats single retrieval
- Explain tenant isolation mechanism and its security gap
- Explain fallback/hallucination control strategy
- Explain key config knobs and vector dimension matching
- Explain top 3 improvements for production readiness

---

## 21) Final Notes

This project is a strong practical RAG system with clear modular architecture and good retrieval quality design. For interview confidence, focus on:

- **system design reasoning**
- **tradeoffs and limitations**
- **clear next-step improvements**

If you can explain those with this document, you can answer both implementation and architecture-level interview questions effectively.

