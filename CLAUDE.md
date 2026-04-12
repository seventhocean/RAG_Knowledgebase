# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WinterAI is a RAG (Retrieval-Augmented Generation) chatbot application with:
- **Backend**: FastAPI + LangChain/LangGraph + PostgreSQL + Redis + Milvus
- **Frontend**: Vue 3 (CDN) + marked + highlight.js, served as static files
- **Core features**: Hybrid search (dense + BM25 sparse), Auto-merging retrieval, query rewriting (Step-Back/HyDE), JWT authentication with RBAC

## Quick Start

### Prerequisites
- Python 3.12+, Docker/Docker Compose
- Package manager: `uv` (recommended) or `pip`

### Setup
```bash
# Install dependencies
uv sync

# Start infrastructure (PostgreSQL, Redis, Milvus)
docker compose up -d

# Configure environment
cp .env.example .env  # Edit with your API keys

# Run backend
uv run uvicorn backend.app:app --host 0.0.0.0 --port 8000 --reload
```

Access: Frontend at `http://127.0.0.1:8000/`, API docs at `/docs`

## Key Commands

```bash
# Development
uv run uvicorn backend.app:app --reload

# Testing
uv run python test_embedding.py
uv run python test_langsmith_eval.py

# Docker
docker compose up -d           # Start infrastructure
docker compose logs -f standalone  # View Milvus logs
docker compose ps              # Check status
```

## Architecture

### RAG Pipeline (LangGraph)

The RAG system is a state machine in `rag_pipeline.py`:

```
retrieve_initial → grade_documents → [rewrite_question → retrieve_expanded] → END
                              ↓ (if relevant)
                          generate_answer
```

- **retrieve_initial**: L3 leaf retrieval via hybrid search (dense + sparse + RRF)
- **grade_documents**: LLM binary relevance scoring (`yes`/`no`)
- **rewrite_question**: Step-Back / HyDE / Complex strategy selection
- **retrieve_expanded**: Secondary retrieval with rewritten query

### Streaming Architecture

```
POST /chat/stream → StreamingResponse(text/event-stream)
    ├── _agent_worker (background task): agent.astream() → output_queue
    └── main loop: await output_queue.get() → yield SSE
         ▲
         └── RAG steps: emit_rag_step() → loop.call_soon_threadsafe() → put_nowait()
```

Key insight: `asyncio.Queue` unifies content tokens and RAG steps. The `_RagStepProxy` wraps `emit_rag_step()` calls to push directly to the queue, so retrieval steps stream in real-time even while the agent is blocked waiting for tool results.

### Document Processing Pipeline

1. **Upload**: `POST /documents/upload` → PDF/Word/Excel parsing
2. **3-level chunking**: L1 (large) → L2 (medium) → L3 (small leaf chunks)
3. **Storage**:
   - L1/L2 → PostgreSQL `ParentChunk` table (DocStore)
   - L3 → Milvus (dense + sparse vectors)
4. **BM25 persistence**: `data/bm25_state.json` tracks `vocab`, `doc_freq`, `total_docs`, `sum_token_len`

### Backend Module Responsibilities

| Module | Responsibility |
|--------|---------------|
| `api.py` | REST endpoints, JWT auth decorators, document CRUD |
| `agent.py` | LangChain agent, conversation storage (PostgreSQL + Redis), session summarization |
| `rag_pipeline.py` | LangGraph state machine: retrieve → grade → rewrite → expand |
| `rag_utils.py` | `retrieve_documents()`, `step_back_expand()`, `generate_hypothetical_document()` |
| `tools.py` | `search_knowledge_base`, `get_current_weather`, `emit_rag_step()` |
| `embedding.py` | `EmbeddingService`: dense (BAAI/bge-m3) + BM25 sparse with incremental update |
| `milvus_client.py` | `MilvusManager`: hybrid search, pagination (`query_all` bypasses 16384 limit) |
| `milvus_writer.py` | Vector writing with BM25 `increment_add` before insert |
| `parent_chunk_store.py` | PostgreSQL + Redis CRUD for L1/L2 parent chunks |
| `document_loader.py` | PDF/Word/Excel loading, 3-level sliding window chunking |

### Authentication & RBAC

- **JWT tokens**: HS256, configurable expiry (default 1440 min)
- **Password hashing**: PBKDF2-SHA256 (310,000 rounds)
- **Roles**: `admin` (document management) vs `user` (chat/sessions only)
- **Admin registration**: Requires `ADMIN_INVITE_CODE`

### Frontend Architecture

Vue 3 reactive app with custom SSE parser:
- **ReadableStream**: Manual `\n\n` splitting for SSE events
- **Thinking state machine**: Single bubble transitions `thinking → RAG steps → streaming`
- **AbortController**: Client-side cancel triggers `GeneratorExit` → `agent_task.cancel()`

## Environment Variables

Required in `.env`:
- `ARK_API_KEY`, `MODEL`, `BASE_URL` (LLM endpoint)
- `EMBEDDING_MODEL=BAAI/bge-m3`, `EMBEDDING_DEVICE=cpu`, `DENSE_EMBEDDING_DIM=1024`
- `MILVUS_HOST=127.0.0.1`, `MILVUS_PORT=19530`
- `DATABASE_URL=postgresql+psycopg2://postgres:postgres@127.0.0.1:5432/langchain_app`
- `REDIS_URL=redis://127.0.0.1:6379/0`
- `JWT_SECRET_KEY`, `ADMIN_INVITE_CODE`

Optional:
- `RERANK_MODEL`, `RERANK_BINDING_HOST`, `RERANK_API_KEY` (Jina reranker)
- `BM25_STATE_PATH` (default: `data/bm25_state.json`)
- `AMAP_WEATHER_API`, `AMAP_API_KEY` (weather tool)
- `GRADE_MODEL` (LLM for relevance grading, defaults to `gpt-4.1`)

## Key Implementation Details

### BM25 State Management
- **File**: `data/bm25_state.json` (gitignored)
- **Structure**: `{version, total_docs, sum_token_len, vocab, doc_freq}`
- **Incremental update**: `increment_add_documents()` on upload, `increment_remove_documents()` on delete
- **Vocab immutability**: Token indices never reclaimed to avoid dimension mismatch with existing Milvus vectors

### Milvus Pagination
- Single `query()` limited to `QUERY_MAX_LIMIT=16384` (server-side constraint)
- `query_all()` loops with offset to fetch all matching documents (used before deletion to sync BM25 stats)

### Caching Strategy (Redis)
- `chat_messages:{user}:{session}`: Message list cache
- `chat_sessions:{user}`: Session list cache  
- `parent_chunk:{chunk_id}`: Parent chunk content cache
- Cache invalidation on write/delete

### Hybrid Search Fusion
- Dense: `AnnSearchRequest` with HNSW index (IP metric)
- Sparse: `AnnSearchRequest` with SPARSE_INVERTED_INDEX
- Fusion: `RRFRanker(k=60)` for parameter-free ranking combination

## Common Development Tasks

### Adding a New Tool
1. Define in `tools.py` with `@tool` decorator
2. Register in `agent.py` → `create_agent_instance(tools=[...])`
3. Update system prompt if needed

### Modifying RAG Behavior
- Edit `rag_pipeline.py` node functions or graph structure
- Adjust retrieval params in `rag_utils.py::retrieve_documents()`
- Tuning: `top_k`, `candidate_k`, rerank thresholds

### Changing Chunking Strategy
- Modify `document_loader.py::DocumentLoader.chunk_document()`
- Update `ParentChunk` schema if adding metadata fields

## Notes
- `data/` is gitignored; BM25 state file lives there
- `volumes/` for Docker data is also gitignored
- Milvus `query_all()` uses pagination to bypass server-side limit
- Frontend thinking state machine: single bubble transitions from "thinking" → "RAG steps" → "streaming answer"
- The `_RAG_STEP_LOOP` global in `tools.py` captures the asyncio loop from the main thread to enable cross-thread `call_soon_threadsafe()` dispatch
