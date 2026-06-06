# DashNoteSystem — Definitive AI Evolution Blueprint
## Final Architecture Plan (Pre-Code Reference)

> Philosophy: We build the AI **product layer**, not AI inference infrastructure.
> We own orchestration, memory, retrieval, workflows, automation, and tenancy.
> LLM inference and embeddings come from hosted providers.

---

## Repository Structure (Final)

```
dashnote/
│
├── src/                          # HTTP control plane — stays lean
│   ├── auth/
│   ├── notes/
│   ├── files/
│   ├── workspaces/
│   ├── membership/
│   ├── core/
│   │   ├── database/
│   │   ├── redis/
│   │   ├── security/
│   │   ├── storage/
│   │   └── health.py
│   └── main.py
│
├── ai/                           # AI orchestration brain — no HTTP
│   ├── providers/                # LLM provider abstraction
│   ├── embeddings/               # embedding provider abstraction
│   ├── retrieval/                # Qdrant wrapper + hybrid search
│   ├── workflows/                # LangGraph graphs
│   ├── prompts/                  # all prompt templates
│   ├── orchestration/            # AI gateway — routing, retries, streaming
│   ├── memory/                   # conversation context assembly
│   ├── tools/                    # agent tools (search_notes, create_note...)
│   ├── services/                 # chat_service, rag_service
│   └── schemas/                  # AI-specific Pydantic models
│
├── worker/                       # Execution muscle — heavy lifting
│   ├── ingestion/                # text extraction, chunking
│   ├── indexing/                 # Qdrant upsert, deletion
│   ├── automation/               # event-driven AI tasks
│   ├── tasks.py                  # ARQ task registry
│   └── main.py                   # ARQ WorkerSettings
│
├── shared/                       # Absolute source of truth — imported by all
│   ├── events/                   # domain event definitions
│   ├── contracts/                # inter-layer interface contracts
│   └── schemas/                  # shared Pydantic models
│
├── tests/
│   ├── src/
│   ├── ai/
│   ├── worker/
│   └── shared/
│
├── docker-compose.yml
├── pyproject.toml                # all 4 roots as installable packages
├── Dockerfile.api                # lean — no torch, no transformers
├── Dockerfile.worker             # heavier — parsing libs here only
└── nginx/
```

---

## Dependency Direction Law (Non-Negotiable)

```
shared/   ←  imported by everyone, imports NOTHING
ai/       ←  imports from shared/ only
worker/   ←  imports from shared/ and ai/ only
src/      ←  imports from shared/, calls ai/ via service interfaces only
```

**Circular import = architecture violation. No exceptions.**

---

## pyproject.toml Package Setup

```toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.backends.legacy:build"

[tool.setuptools.packages.find]
where = ["."]
include = ["src*", "ai*", "worker*", "shared*"]

[tool.pytest.ini_options]
pythonpath = ["src", "ai", "worker", "shared"]
asyncio_mode = "auto"
addopts = "--import-mode=importlib"
```

---

## Environment Variables (Complete Final List)

```env
# ── Existing ──────────────────────────────────────────
DATABASE_URL=postgresql+asyncpg://...
REDIS_URL=redis://localhost:6379
JWT_SECRET=...
REDIS_ENABLED=true

# ── AI Providers ──────────────────────────────────────
OPENAI_API_KEY=sk-...
LLM_MODEL=gpt-4.1-mini
LLM_TEMPERATURE=0.0
LLM_MAX_TOKENS=2048

# ── Embeddings ────────────────────────────────────────
EMBEDDING_PROVIDER=openai          # "openai" only in Phase 2
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIMENSION=1536
EMBEDDING_BATCH_SIZE=32
EMBEDDING_MAX_RETRIES=3

# ── Chunking ──────────────────────────────────────────
CHUNK_SIZE=1000
CHUNK_OVERLAP=150
CHUNK_MIN_LENGTH=50

# ── Qdrant Cloud ──────────────────────────────────────
QDRANT_URL=https://your-cluster.qdrant.io
QDRANT_API_KEY=...
QDRANT_NOTES_COLLECTION=notes_chunks
QDRANT_FILES_COLLECTION=files_chunks
QDRANT_TIMEOUT=30

# ── Neo4j (Phase 9 — leave blank until then) ──────────
NEO4J_URI=
NEO4J_USER=neo4j
NEO4J_PASSWORD=

# ── LangSmith ─────────────────────────────────────────
LANGSMITH_API_KEY=...
LANGSMITH_PROJECT=dashnote
LANGSMITH_TRACING_ENABLED=false    # true from Phase 5 onwards

# ── ARQ Worker ────────────────────────────────────────
ARQ_REDIS_URL=redis://localhost:6379
WORKER_MAX_JOBS=5                  # align with OpenAI tier TPM
WORKER_JOB_TIMEOUT=300

# ── Cost Controls ─────────────────────────────────────
EMBEDDING_CACHE_ENABLED=true
EMBEDDING_CACHE_TTL=86400          # 24h — chunk vectors rarely change
TOKEN_BUDGET_PER_REQUEST=8000
```

---

## Docker Compose (Final Services)

```yaml
services:
  nginx:
    image: nginx:alpine
    ports: ["80:80"]
    depends_on: [api]

  api:                             # LEAN — no ML libs
    build:
      context: .
      dockerfile: Dockerfile.api
    ports: ["8000:8000"]
    env_file: .env
    depends_on: [db, redis, qdrant]

  worker:                          # HEAVIER — parsing, indexing
    build:
      context: .
      dockerfile: Dockerfile.worker
    command: python -m arq worker.main.WorkerSettings
    env_file: .env
    depends_on: [db, redis, qdrant]
    deploy:
      replicas: 1                  # scale independently later

  db:
    image: postgres:16-alpine
    volumes: [pgdata:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine

  qdrant:
    image: qdrant/qdrant:latest
    ports: ["6333:6333"]
    volumes: [qdrant_data:/qdrant/storage]
    # For production: replace with Qdrant Cloud URL in .env

  migrate:
    build:
      context: .
      dockerfile: Dockerfile.api
    command: alembic upgrade head
    env_file: .env
    depends_on: [db]

volumes:
  pgdata:
  qdrant_data:
```

---

## Dockerfile Strategy

### Dockerfile.api (Lean)
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.api.txt .
RUN pip install --no-cache-dir -r requirements.api.txt
# NO: torch, transformers, unstructured, pypdf
COPY src/ ./src/
COPY ai/ ./ai/
COPY shared/ ./shared/
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Dockerfile.worker (Full)
```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN apt-get update && apt-get install -y libmagic1 && rm -rf /var/lib/apt/lists/*
COPY requirements.worker.txt .
RUN pip install --no-cache-dir -r requirements.worker.txt
# YES: pypdf, unstructured, python-docx, markdown, beautifulsoup4
COPY worker/ ./worker/
COPY ai/ ./ai/
COPY shared/ ./shared/
CMD ["python", "-m", "arq", "worker.main.WorkerSettings"]
```

---

## Phase-by-Phase Implementation Plan

---

### PHASE 1 — Clean Foundation
**Goal:** Repo structure, packages, config. Zero AI features yet.

#### Step 1.1 — Create Directory Structure
Create all directories listed in repo structure above.
Add `__init__.py` to every package folder.
`shared/` is the first thing to get actual code.

#### Step 1.2 — shared/events/ — Domain Events
```python
# shared/events/definitions.py
class NoteCreatedEvent(BaseModel):
    note_id: str
    workspace_id: str
    created_by: str
    is_private: bool
    title: str
    content: str
    timestamp: datetime

class NoteUpdatedEvent(BaseModel): ...
class FileUploadedEvent(BaseModel): ...
class FileDeletedEvent(BaseModel): ...
class WorkspaceUpdatedEvent(BaseModel): ...
```
These are the contracts. API emits them. Worker consumes them. They never import from `src/` or `worker/`.

#### Step 1.3 — shared/contracts/ — Inter-Layer Contracts
```python
# shared/contracts/indexing.py
class IndexingRequest(BaseModel):
    note_id: str
    workspace_id: str
    created_by: str
    is_private: bool
    title: str
    content: str
    operation: Literal["upsert", "delete"]

class IndexingResult(BaseModel):
    note_id: str
    chunks_indexed: int
    success: bool
    error: str | None = None
```

#### Step 1.4 — Extend Settings (src/config/settings.py)
Add all env var groups from the list above.
Add computed property:
```python
@property
def ai_enabled(self) -> bool:
    return bool(self.OPENAI_API_KEY) or self.EMBEDDING_PROVIDER == "local"
```
Fail fast: validators raise on boot if required keys missing.

#### Step 1.5 — Split Requirements Files
```
requirements.api.txt     — fastapi, sqlalchemy, pydantic, openai, qdrant-client,
                           langchain-core, langgraph, arq, tenacity, structlog

requirements.worker.txt  — everything in api.txt PLUS:
                           pypdf, python-docx, unstructured, markdown,
                           beautifulsoup4, aiofiles
```

#### Step 1.6 — pyproject.toml
Set up as shown above. All 4 roots as installable packages.
This prevents all circular import issues before they start.

---

### PHASE 2 — Embedding Pipeline (Worker-Side)
**Goal:** Notes get embedded in background after creation. API never blocks.

#### Step 2.1 — ARQ Setup (worker/main.py)
```python
from arq import cron
from arq.connections import RedisSettings

class WorkerSettings:
    functions = [embed_note_task, delete_note_vectors_task]
    redis_settings = RedisSettings.from_dsn(settings.ARQ_REDIS_URL)
    max_jobs = settings.WORKER_MAX_JOBS        # rate limit guard
    job_timeout = settings.WORKER_JOB_TIMEOUT
    on_startup = startup
    on_shutdown = shutdown
```

#### Step 2.2 — Embedding Provider (ai/embeddings/)
Files:
- `base.py` — abstract BaseEmbeddingProvider, EmbeddedChunk, EmbeddingProviderError
- `openai_provider.py` — AsyncOpenAI, tenacity retry on RateLimitError
- `factory.py` — singleton factory, FastAPI Depends

Key rules:
- Retry: `@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=2, max=30))`
- Only retry: RateLimitError, APITimeoutError, APIConnectionError
- Never retry: AuthenticationError, BadRequestError

#### Step 2.3 — Text Chunker (ai/embeddings/chunker.py)
- RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)
- Deterministic chunk_id: `str(uuid5(NAMESPACE_URL, f"{note_id}:{index}"))`
- Idempotent: same note always produces same chunk IDs → safe Qdrant upserts

#### Step 2.4 — Embedding Cache (ai/embeddings/cache.py)
```python
async def get_cached_vector(chunk_hash: str, redis) -> list[float] | None:
    key = f"embed:v1:{chunk_hash}"
    cached = await redis.get(key)
    return json.loads(cached) if cached else None

async def cache_vector(chunk_hash: str, vector: list[float], redis, ttl: int):
    await redis.setex(f"embed:v1:{chunk_hash}", ttl, json.dumps(vector))
```
Cache key = `sha256(chunk_text)`. Cache hit = skip OpenAI call entirely.
TTL = 24h (EMBEDDING_CACHE_TTL setting).

#### Step 2.5 — Ingestion Worker Task (worker/ingestion/tasks.py)
```python
async def embed_note_task(ctx, *, request: IndexingRequest):
    # 1. chunk
    # 2. for each chunk: check embedding cache first
    # 3. embed uncached chunks in batch (EMBEDDING_BATCH_SIZE)
    # 4. cache new vectors
    # 5. store EmbeddedChunk objects → passed to Phase 3 indexer
    # 6. log result
```
ARQ handles retry at task level. Tenacity handles retry at OpenAI call level.
Two layers of protection against rate limits.

#### Step 2.6 — API Emits Events (src/notes/router.py)
```python
# After note saved to DB — non-blocking
if settings.ai_enabled:
    await arq_pool.enqueue_job(
        "embed_note_task",
        request=IndexingRequest(
            note_id=str(note.id),
            workspace_id=str(ctx.workspace_id),
            ...
            operation="upsert",
        )
    )
```
API does NOT import anything from `worker/`. It only enqueues via ARQ pool.

---

### PHASE 3 — Qdrant Vector Memory
**Goal:** Embedded chunks stored and searchable with full tenant isolation.

#### Step 3.1 — Collection Strategy (Single Collection)
```
Collection: notes_chunks
Payload schema:
{
  "workspace_id": "uuid",      ← KEYWORD index — mandatory filter
  "note_id": "uuid",           ← KEYWORD index
  "created_by": "uuid",        ← KEYWORD index
  "visibility": "public|private", ← KEYWORD index
  "chunk_index": 0             ← INTEGER index
}
```

#### Step 3.2 — Payload Indexes (Non-Negotiable)
```python
# Must run on collection init — not optional
for field, schema in [
    ("workspace_id", "keyword"),
    ("note_id", "keyword"),
    ("created_by", "keyword"),
    ("visibility", "keyword"),
]:
    await client.create_payload_index(
        collection_name=collection,
        field_name=field,
        field_schema=schema,
    )
```
Without these: every filter = full collection scan = O(n) latency at scale.

#### Step 3.3 — Hybrid Search Setup (Dense + Sparse from Day One)
```
Dense vector:  text-embedding-3-small (1536d) — semantic similarity
Sparse vector: BM25 (Qdrant native) — keyword/exact term matching
Combined:      Reciprocal Rank Fusion (RRF)
```
Why BGE-M3 is deferred: BGE-M3 is 560MB model, requires GPU or significant CPU.
BM25 sparse via Qdrant native sparse vectors is zero infrastructure overhead.
Upgrade path: swap BM25 for BGE-M3 sparse encoder later without re-indexing dense.

#### Step 3.4 — Tenant-Safe Vector Wrapper (ai/retrieval/qdrant_wrapper.py)
```python
class WorkspaceVectorSearch:
    """
    The ONLY interface to Qdrant. Raw qdrant_client never called directly.
    workspace_id is injected from RequestContext — cannot be bypassed.
    """
    def __init__(self, client: AsyncQdrantClient, ctx: RequestContext): ...

    async def search(
        self,
        query_vector: list[float],
        query_sparse: SparseVector,
        *,
        limit: int = 10,
        visibility_filter: Literal["all", "public", "own"] = "all",
    ) -> list[ScoredPoint]:
        # workspace_id ALWAYS injected as must filter
        # visibility ALWAYS enforced per RBAC rules
        # No raw search() ever exposed
```

#### Step 3.5 — RBAC Filter Logic (ai/retrieval/filters.py)
```python
def build_rbac_filter(ctx: RequestContext) -> Filter:
    if ctx.role in ("owner", "admin"):
        # See all in workspace
        return Filter(must=[workspace_must(ctx)])
    else:
        # Member: own notes OR public notes
        return Filter(
            must=[workspace_must(ctx)],
            should=[
                public_condition(),
                own_condition(ctx),
            ],
            minimum_should_match=1,
        )
```
Mirrors `src/notes/permissions.py` exactly. Single source of logic.

#### Step 3.6 — Tenant Sparse Vector Isolation Note
Qdrant BM25 IDF is computed globally across collection.
Mitigation: `workspace_id` must filter runs BEFORE scoring.
Documents outside tenant never reach scoring stage.
IDF cross-contamination is minimal at normal SaaS scale.
Document limitation. Revisit if reaching 10M+ chunks.

#### Step 3.7 — Indexing Worker Task (worker/indexing/tasks.py)
```python
async def index_chunks_task(ctx, *, embedded_chunks: list[EmbeddedChunk]):
    # Always upsert — never insert
    # Idempotent: same chunk_id = overwrite, not duplicate
    await qdrant.upsert(collection=..., points=[...])

async def delete_note_vectors_task(ctx, *, note_id: str, workspace_id: str):
    await qdrant.delete(
        collection=...,
        points_selector=FilterSelector(
            filter=Filter(must=[
                FieldCondition(key="note_id", match=MatchValue(value=note_id)),
                FieldCondition(key="workspace_id", match=MatchValue(value=workspace_id)),
            ])
        )
    )
```
Always upsert. Idempotency guaranteed by deterministic chunk_ids.

---

### PHASE 4 — AI Orchestration Layer
**Goal:** Provider-independent AI gateway. Structured outputs. Prompt isolation.

#### Step 4.1 — AI Gateway (ai/orchestration/ai_gateway.py)
Single class that all AI features call. Never call OpenAI directly from routers.
```python
class AIGateway:
    async def chat(self, messages, system_prompt) -> ChatResult
    async def structured(self, prompt, response_model: type[BaseModel]) -> BaseModel
    async def stream(self, messages, system_prompt) -> AsyncIterator[str]
    async def embed(self, texts: list[str]) -> list[list[float]]
```

#### Step 4.2 — Structured Outputs (Critical)
```python
# Use LangChain .with_structured_output() or Instructor
async def structured(self, prompt: str, response_model: type[T]) -> T:
    llm = self._llm.with_structured_output(response_model)
    return await llm.ainvoke(prompt)
```
Every automation, citation, tagging, and agent tool uses structured outputs.
Never parse raw LLM strings with regex.

#### Step 4.3 — Prompt Templates (ai/prompts/)
```
ai/prompts/
├── rag.py           — RAG answer with citations
├── summarize.py     — note/workspace summarization
├── chat.py          — conversational workspace assistant
├── extract.py       — entity/tag extraction (Phase 9)
└── base.py          — shared template utilities
```
No prompt strings anywhere in routers or services.

#### Step 4.4 — LangSmith Tracing (Enable Now)
```python
import langsmith
# Wrap all AIGateway calls with LangSmith tracing
# Even if LANGSMITH_TRACING_ENABLED=false in dev, wiring is ready
```
Wire tracing in Phase 4. Turn it on in Phase 5 when real LLM calls start.

---

### PHASE 5 — First Real AI Feature: Workspace Chat + RAG
**Goal:** Users can ask questions about their notes. Citations included.

#### Step 5.1 — POST /ai/chat Endpoint (src/ai_routes/chat.py)
```python
@router.post("/ai/chat")
async def chat(body: ChatRequest, ctx: RequestContext = Depends(get_current_context)):
    # Freeze security context BEFORE opening stream generator
    workspace_id = str(ctx.workspace_id)
    user_id = str(ctx.user_id)
    role = ctx.role

    async def generate():
        # Only frozen primitives inside — never ctx reference
        async for chunk in rag_service.stream_answer(
            workspace_id=workspace_id,
            user_id=user_id,
            role=role,
            question=body.message,
            thread_id=body.thread_id,
        ):
            yield f"data: {json.dumps({'content': chunk})}\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```
RequestContext validated BEFORE stream opens. workspace_id frozen. Never passed as object.

#### Step 5.2 — RAG Pipeline (ai/services/rag_service.py)
```
user question
  → embed query (OpenAI)
  → hybrid search (dense + sparse + RBAC filter)
  → rerank top-k chunks by relevance score
  → context assembly (chunks + recent messages + workspace metadata)
  → token budget check (stay under TOKEN_BUDGET_PER_REQUEST)
  → inject into prompt template
  → stream LLM response
  → attach citation map to response
```

#### Step 5.3 — Citation Map (ai/schemas/citations.py)
```python
class Citation(BaseModel):
    note_id: str
    chunk_id: str
    title: str
    relevance_score: float
    char_start: int
    char_end: int

class RAGResponse(BaseModel):
    answer: str
    citations: list[Citation]
    thread_id: str
```
Every RAG response includes source references. Enterprise-grade from day one.

---

### PHASE 6 — Memory + Conversations
**Goal:** Persistent conversations with LangGraph state management.

#### Step 6.1 — Product Tables (PostgreSQL migrations)
```sql
CREATE TABLE ai_threads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    created_by UUID NOT NULL REFERENCES users(id),
    title TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE ai_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thread_id UUID NOT NULL REFERENCES ai_threads(id),
    role TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system', 'tool')),
    content TEXT NOT NULL,
    citations JSONB DEFAULT '[]',
    token_count INTEGER,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_ai_messages_thread ON ai_messages(thread_id, created_at DESC);
CREATE INDEX idx_ai_threads_workspace ON ai_threads(workspace_id, updated_at DESC);
```

#### Step 6.2 — LangGraph Checkpointing (Separate from Product Tables)
```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

checkpointer = AsyncPostgresSaver.from_conn_string(settings.DATABASE_URL)
await checkpointer.setup()  # creates langgraph internal tables

# Link by thread_id only — product tables own UX, checkpointer owns graph state
graph = workflow.compile(checkpointer=checkpointer)
result = await graph.ainvoke(state, config={"configurable": {"thread_id": str(thread.id)}})
```
Two separate concerns. Linked only by `thread_id` string.

#### Step 6.3 — Context Assembly (ai/memory/context_builder.py)
```python
class ContextBuilder:
    async def build(
        self,
        ctx: RequestContext,
        question: str,
        thread_id: str,
    ) -> AssembledContext:
        recent_messages = await self._load_recent_messages(thread_id, n=10)
        retrieved_chunks = await self._hybrid_search(question, ctx)
        workspace_meta = await self._load_workspace_meta(ctx.workspace_id)
        return AssembledContext(
            messages=recent_messages,
            chunks=retrieved_chunks,
            workspace=workspace_meta,
            token_budget_remaining=self._calculate_budget(recent_messages, retrieved_chunks),
        )
```

---

### PHASE 7 — LangGraph Workflows
**Goal:** Single-agent workflows. Tool calling. Durable state.

#### Step 7.1 — Workspace Assistant Graph (ai/workflows/workspace_assistant.py)
```python
# Nodes
async def retrieve_node(state): ...    # hybrid search → chunks
async def reason_node(state): ...      # LLM decides if tool call needed
async def tool_node(state): ...        # execute tool calls
async def respond_node(state): ...     # final LLM response + citations

# Edges
graph.add_edge("retrieve", "reason")
graph.add_conditional_edges("reason", route_to_tool_or_respond)
graph.add_edge("tool", "reason")       # reason again after tool
graph.add_edge("respond", END)
```

#### Step 7.2 — Agent Tools (ai/tools/)
```python
# Tools call services — never repositories directly
@tool
async def search_notes(query: str, ctx: RequestContext) -> list[NoteSearchResult]:
    return await rag_service.search(query, ctx)

@tool
async def create_note(title: str, content: str, ctx: RequestContext) -> NoteResult:
    return await note_service.create(title, content, ctx)

@tool
async def summarize_workspace(ctx: RequestContext) -> WorkspaceSummary:
    return await summary_service.summarize_workspace(ctx)
```
Tools → services → repositories. Never tools → repositories directly.

#### Step 7.3 — Tool Rate Limiting
LangGraph tool calls inherit ARQ rate limiting for background operations.
Synchronous tool calls (search, read) go through API with existing rate limits.

---

### PHASE 8 — Event-Driven Automation
**Goal:** File uploads, note creation trigger intelligent background workflows.

#### Step 8.1 — Event Bus (shared/events/bus.py)
```python
async def emit_event(event: BaseEvent, arq_pool: ArqRedis):
    task_map = {
        NoteCreatedEvent: "handle_note_created",
        FileUploadedEvent: "handle_file_uploaded",
    }
    task_name = task_map.get(type(event))
    if task_name:
        await arq_pool.enqueue_job(task_name, event=event.model_dump())
```

#### Step 8.2 — Automation Tasks (worker/automation/tasks.py)
```python
async def handle_file_uploaded(ctx, *, event: dict):
    # 1. extract text (pypdf, python-docx — this is why worker container is heavier)
    # 2. chunk text
    # 3. embed chunks (with cache)
    # 4. index to Qdrant
    # 5. structured output: auto-generate summary + tags
    # 6. store summary in DB
```

#### Step 8.3 — Human Approval Gate
For destructive or external operations:
```python
class AutomationDecision(BaseModel):
    action: str
    requires_approval: bool
    confidence: float
    reasoning: str

# If requires_approval: store pending action, notify user via API
# If confidence > 0.95 and not destructive: execute directly
```

---

### PHASE 9 — GraphRAG with Neo4j (Optional — Only If Needed)
**Goal:** Relationship intelligence on top of existing retrieval.

**Trigger criteria (do NOT start before):**
- Retrieval working well in production
- Real users generating relationship-rich content
- Specific feature request requiring relationship traversal

#### Step 9.1 — Entity Extraction (Added to Ingestion Pipeline)
```python
# During ingestion (worker), after embedding:
entities = await ai_gateway.structured(
    prompt=extract_entities_prompt(chunk_text),
    response_model=EntityExtractionResult,
)
# Write entities + relationships to Neo4j
# Write chunks to Qdrant (already happening)
```

#### Step 9.2 — Graph Schema
```cypher
(User)-[:OWNS]->(Note)
(Note)-[:REFERENCES]->(Note)
(Note)-[:ATTACHED_TO]->(File)
(Workspace)-[:CONTAINS]->(Notebook)
(Note)-[:MENTIONS]->(Entity)
(Entity)-[:RELATED_TO]->(Entity)
```

#### Step 9.3 — Combined Retrieval (GraphRAG)
```
user question
  → Qdrant hybrid search → semantic chunks
  → Neo4j graph traversal → related notes + entities
  → merge + deduplicate context
  → LLM answer with richer context
```

---

### PHASE 10 — Multi-Agent System (Late Stage)
**Trigger:** Stable retrieval + stable workflows + real product need proven.

#### Agents
- `WorkspaceAgent` — general queries, note operations
- `ResearchAgent` — deep retrieval, cross-note synthesis
- `AutomationAgent` — event-driven workflow execution
- `KnowledgeAgent` — graph traversal, relationship discovery (after Phase 9)

#### Supervisor Pattern (ai/workflows/supervisor.py)
```python
# LangGraph supervisor routes between specialized agents
async def route(state) -> Literal["workspace", "research", "automation"]:
    classification = await ai_gateway.structured(
        prompt=classify_intent_prompt(state["question"]),
        response_model=IntentClassification,
    )
    return classification.agent  # cheap model for routing
```
Cheap model for routing. Expensive model for reasoning. Cost control.

---

### PHASE 11 — Observability
**Goal:** Full production visibility into AI behavior and costs.

#### Step 11.1 — LangSmith (Already Wired from Phase 4)
Enable `LANGSMITH_TRACING_ENABLED=true` in production.
Traces: token usage, latency, tool calls, retrieval quality per workspace.

#### Step 11.2 — Structured Logging (Every AI Operation)
```python
logger.info("rag_request", extra={
    "request_id": request_id,
    "workspace_id": workspace_id,
    "thread_id": thread_id,
    "question_tokens": token_count,
    "retrieved_chunks": len(chunks),
    "latency_ms": latency,
})
```

#### Step 11.3 — AI Cost Tracking
```python
class TokenUsageEvent(BaseModel):
    workspace_id: str
    operation: str              # "rag_chat", "auto_summarize", "embed"
    prompt_tokens: int
    completion_tokens: int
    model: str
    cost_usd: float             # calculated from model pricing
    timestamp: datetime
# Store in PostgreSQL ai_usage table
# Aggregate per workspace for billing/quota enforcement
```

---

### PHASE 12 — Production Scaling
**Goal:** Cost controls, caching, horizontal scaling.

#### Step 12.1 — Embedding Cache Review
Redis cache hit rate monitoring. Tune TTL based on note edit frequency.

#### Step 12.2 — AI Response Cache
```python
# Cache deterministic RAG answers
cache_key = f"rag:{workspace_id}:{hash(question)}:{retrieval_gen_counter}"
# Invalidate on note write (same generation counter pattern as existing cache)
```

#### Step 12.3 — Model Routing
```
Classification/routing:  gpt-4o-mini  (fast, cheap)
RAG responses:           gpt-4.1-mini  (balanced)
Complex reasoning:       gpt-4.1       (expensive, rare)
```

#### Step 12.4 — Worker Horizontal Scaling
```yaml
# docker-compose.yml — scale workers independently
worker:
  deploy:
    replicas: 3  # each worker respects WORKER_MAX_JOBS
```
Each worker instance: max_jobs=5 → 3 workers = 15 concurrent embedding jobs max.
Stay within OpenAI TPM limits per tier.

---

## Critical Rules Summary (Never Violate)

| Rule | Reason |
|---|---|
| API container has no torch/transformers | Image size + event loop blocking |
| shared/ imports nothing from other layers | Circular import prevention |
| workspace_id always injected from RequestContext | Tenant security |
| workspace_id filter always `must` in Qdrant | Cross-tenant vector leakage |
| Always upsert, never insert to Qdrant | Idempotency on retry |
| Freeze ctx before SSE generator | Streaming security |
| Tools call services, not repositories | Architectural boundary |
| ARQ max_jobs aligned to OpenAI TPM tier | Rate limit protection |
| Embedding cache keyed by sha256(chunk_text) | Cost control |
| Structured outputs everywhere, no regex parsing | Reliability |

---

## What Each Phase Delivers to Users

| Phase | User-Visible Feature |
|---|---|
| 1 | Nothing yet — foundation only |
| 2 | Nothing yet — background indexing begins silently |
| 3 | Nothing yet — search index ready |
| 4 | Nothing yet — AI gateway ready |
| **5** | **POST /ai/chat — ask questions about your notes** |
| **5** | **Citations in every answer** |
| **5** | **Streaming responses** |
| **6** | **Conversation history — continue previous chats** |
| **7** | **Agent creates/updates notes on your behalf** |
| **7** | **Workspace summarization** |
| **8** | **Auto-summarize on file upload** |
| **8** | **Auto-tagging on note creation** |
| 9 | Related notes discovery |
| 10 | Multi-step research workflows |
| 11 | Usage dashboard (cost per workspace) |
| 12 | Faster responses (caching) |
```
