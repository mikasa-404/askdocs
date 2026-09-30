# askdocs: requirements and plan

> **Single source of truth** for what we're building, why, and in what order.
> If a decision changes, update this file in the same commit as the code change.
> Progress is tracked with the checkboxes in [Milestones](#6-milestones).

---

## 1. Vision

**One-liner:** Upload your docs, then ask questions and get answers that cite their sources, streamed back live.

**Who it's for:** Developers and small teams whose knowledge is spread across Markdown files, READMEs, runbooks and notes, and who want to ask "how do we rotate the DB password?" instead of grepping.

**Why this project (learning goals):**
- Practice idiomatic Go end to end: HTTP, concurrency, I/O, `context`, errors, testing.
- Build a real SQL-backed service (Postgres + pgvector) with migrations and transactions.
- Build and operate a background job queue with a worker pool.
- Ship to production: Docker, AWS ECS Fargate, RDS, CI.
- Come out with a project that's easy to explain in an interview and has depth under the hood.

---

## 2. Scope

### In scope (MVP, milestones 0–6)
- Collections (named groups of documents)
- Upload `.md` and `.txt` documents
- Asynchronous ingestion: chunk → embed → store, with a status for each document
- Ask a question against a collection, get an answer with citations, streamed over SSE
- Local LLM via Ollama

### Later (milestones 7–10)
- A second LLM provider (Gemini) behind the same interfaces
- API-key auth, rate limits, per-key token budgets
- Docker image, CI, AWS deployment
- Stretch: PDF ingestion, importing a GitHub repo or URL, a CLI client, a minimal web UI, conversation history

### Out of scope
- Training or fine-tuning models
- Multi-tenant billing
- Real-time collaborative editing of documents

---

## 3. Functional requirements

IDs are stable, so reference them in commits and PRs (for example `feat: FR-3.2 chunk overlap`).

### FR-1 Collections
- **FR-1.1** Create, list and delete collections.
- **FR-1.2** Deleting a collection deletes its documents and chunks (cascade).

### FR-2 Document upload
- **FR-2.1** Upload a `.md` or `.txt` file to a collection (multipart/form-data).
- **FR-2.2** Reject files over a configurable size limit, and unsupported types, with a clear 4xx error.
- **FR-2.3** Uploading returns immediately with a document ID and status `pending`. Ingestion is asynchronous.
- **FR-2.4** Store a content hash per document. Re-uploading identical content does not re-embed it.
- **FR-2.5** List documents in a collection with their status; delete a document (and its chunks).

### FR-3 Ingestion pipeline
- **FR-3.1** Stages: parse → chunk → embed → store.
- **FR-3.2** Chunking: a fixed size with overlap, both configurable. Chunks must never split a multi-byte UTF-8 character.
- **FR-3.3** Embed chunks in batches with bounded concurrency, so we never exceed the provider's rate limit.
- **FR-3.4** Each chunk stores: text, position/offset in the source, embedding, **embedding model name** and dimension.
- **FR-3.5** Document status moves `pending → processing → ready | failed`; `failed` stores the error message.
- **FR-3.6** A document is `ready` only when all its chunks are stored (transactional, so no half-ingested documents).

### FR-4 Job queue
- **FR-4.1** Ingestion jobs are stored in Postgres and claimed with `SELECT … FOR UPDATE SKIP LOCKED`.
- **FR-4.2** A worker pool of configurable size processes jobs concurrently.
- **FR-4.3** Failed jobs retry with exponential backoff up to a maximum number of attempts, then are marked dead.
- **FR-4.4** Jobs held by a worker that crashed are recovered after a visibility timeout (at-least-once delivery).
- **FR-4.5** Workers stop cleanly on shutdown: finish or release in-flight jobs.

### FR-5 Question answering
- **FR-5.1** Ask a question against a collection.
- **FR-5.2** Embed the question with the **same model** used for the collection's chunks.
- **FR-5.3** Retrieve the top-k most similar chunks (cosine distance, pgvector); k is configurable.
- **FR-5.4** Build a prompt from the retrieved chunks and instruct the model to answer only from the context and say when it doesn't know.
- **FR-5.5** Stream the answer token by token via Server-Sent Events.
- **FR-5.6** Return citations: the document and chunk each answer relied on.
- **FR-5.7** If the client disconnects, cancel the upstream LLM call (context cancellation).

### FR-6 LLM providers
- **FR-6.1** The app depends on two small interfaces, one for embedding and one for chat/generation, not on any specific provider.
- **FR-6.2** Ollama implementation (local development).
- **FR-6.3** Gemini implementation (production candidate).
- **FR-6.4** The provider is selected by config (for example `LLM_PROVIDER=ollama|gemini`); no code changes needed.
- **FR-6.5** Provider errors are mapped to app-level errors (rate limited, unavailable, bad request).

### FR-7 Auth and limits (later)
- **FR-7.1** API-key auth; keys are stored **hashed**, never in plaintext.
- **FR-7.2** Rate limit per API key.
- **FR-7.3** Daily token budget per API key, stored in Postgres; exceeding it returns 429.

### FR-8 Operations
- **FR-8.1** `GET /healthz` (process is alive) and `GET /readyz` (DB and provider are reachable).
- **FR-8.2** Structured JSON logs (`log/slog`) with a request ID on every line.
- **FR-8.3** All configuration comes from environment variables, with validation at startup.

---

## 4. Non-functional requirements

| Area | Requirement |
|---|---|
| Reliability | Graceful shutdown on SIGTERM (ECS sends it before stopping a task) |
| Timeouts | Every outbound call (DB, LLM) has a timeout or deadline via `context` |
| Safety | Upload size limits; request body limits; no secrets in the repo or logs |
| Concurrency | Code passes `go test -race ./...`; no goroutine leaks |
| Data integrity | Ingestion is transactional; migrations are versioned and reversible |
| Portability | Runs identically with `docker compose up` locally and on ECS |
| Cost | Embeddings cached by content hash; answer length capped; provider spending limit set |
| Performance | Targets to be set once we have real measurements; no numbers invented up front |

---

## 5. Architecture

```
            ┌──────────────── askdocs (one Go binary, two roles) ────────────────┐
 client ──▶ │  HTTP API  ──▶  enqueue job ──▶ ┌────────────┐                     │
  (curl,    │     │                           │  Postgres  │◀── worker pool:     │
   CLI,     │     │  ask question             │  + pgvector│    parse→chunk→     │
   UI)      │     ▼                           │  jobs,docs,│    embed→store      │
            │  retrieve top-k ◀───────────────│  chunks    │                     │
            │     │                           └────────────┘                     │
            │     ▼                                                              │
            │  LLM (Ollama / Gemini) ──▶ stream tokens back over SSE             │
            └────────────────────────────────────────────────────────────────────┘
```

- **One binary, two roles** (API server and workers) to start. Splitting them later is a config/flag change; that's a good interview discussion point.
- **Postgres is the only stateful dependency**: data, vectors and the job queue. No Redis or Kafka, on purpose.

### Planned data model (high level; exact schema is designed in Milestone 2)
- `collections`: id, name, created_at
- `documents`: id, collection_id, filename, content_hash, status, error, created_at
- `chunks`: id, document_id, position, text, embedding (vector), embedding_model, created_at
- `jobs`: id, type, payload, status, attempts, run_at, locked_until, last_error

### Planned API (refined as we build)
| Method | Path | Purpose |
|---|---|---|
| POST | `/collections` | Create collection |
| GET | `/collections` | List collections |
| DELETE | `/collections/{id}` | Delete collection |
| POST | `/collections/{id}/documents` | Upload document (async) |
| GET | `/collections/{id}/documents` | List documents and their status |
| DELETE | `/documents/{id}` | Delete document |
| POST | `/collections/{id}/ask` | Ask a question (SSE stream) |
| GET | `/healthz`, `/readyz` | Health checks |

### Tech stack and decisions
| Concern | Choice | Why |
|---|---|---|
| Language | Go 1.27 | Learning goal |
| HTTP | stdlib `net/http` + `ServeMux` method/path patterns | Learn the fundamentals before adding frameworks |
| DB driver | `pgx` | The de facto Postgres driver; supports pools and `LISTEN/NOTIFY` |
| SQL | Hand-written SQL first; consider `sqlc` later | Learn SQL properly, then remove boilerplate |
| Vectors | `pgvector` extension | Keeps everything in one database |
| Migrations | *Open:* `golang-migrate` vs `goose` | Decide in Milestone 2 |
| Logging | `log/slog` | Standard library, structured |
| LLM (local) | Ollama: `nomic-embed-text`, `llama3.2:3b` | Free; runs on an M2 with 16 GB |
| LLM (prod) | Gemini (candidate) | Low or free cost at hobby scale; confirm pricing when we get there |
| Packaging | Docker multi-stage build, docker-compose | Small image; one-command local setup |
| Deploy | AWS ECR + ECS Fargate + RDS Postgres | Managed; relevant for interviews |
| CI | GitHub Actions | vet, test with race detector, build image |

---

## 6. Milestones

Each milestone ends with: code review, `go vet ./...`, `go test -race ./...` passing, and 2–3 interview questions.

- [ ] **M0 Setup and first LLM call**
  - Talk to Ollama with `curl` and understand the embed response. *(Task 0.1)*
  - Call the embed endpoint from Go and print the vector length.
  - *Concepts:* `net/http` client, JSON struct tags, `encoding/json`, error handling.
- [ ] **M1 Server skeleton**: `cmd/` entry point, config from env, `slog`, `/healthz`, graceful shutdown.
  - *Concepts:* `http.Server`, `signal.NotifyContext`, `context`, package layout, middleware.
- [ ] **M2 Database**: docker-compose with Postgres + pgvector, migrations, connection pool, collections CRUD (FR-1).
  - *Concepts:* `pgx` pools, transactions, SQL, `errors.Is`, repository pattern via interfaces.
- [ ] **M3 Ingestion (synchronous first)**: upload, chunk, embed, store (FR-2, FR-3).
  - *Concepts:* multipart, `io.Reader`, runes vs bytes, `errgroup` with limits, batching.
- [ ] **M4 Question answering**: embed the question, run a top-k vector search, build the prompt, return a non-streamed answer with citations (FR-5.1–5.4, 5.6).
  - *Concepts:* vector search SQL, prompt construction, HTTP client timeouts.
- [ ] **M5 Streaming**: SSE answers and cancellation when the client disconnects (FR-5.5, 5.7).
  - *Concepts:* `http.Flusher`, `bufio.Scanner`, goroutines + channels, `context` propagation.
- [ ] **M6 Job queue**: make ingestion asynchronous with a Postgres queue and worker pool (FR-4).
  - *Concepts:* `SKIP LOCKED`, worker pools, backoff, at-least-once delivery, graceful worker shutdown.
- [ ] **M7 Provider interfaces + Gemini** (FR-6).
  - *Concepts:* interface design, adapters, error mapping, config-driven wiring.
- [ ] **M8 Tests, Docker, CI**: table tests, `httptest`, fakes, integration tests against real Postgres; multi-stage Dockerfile; GitHub Actions.
- [ ] **M9 Deploy to AWS**: ECR, ECS Fargate, RDS, secrets management, ALB health checks, logs.
- [ ] **M10 Hardening**: API keys, rate limits, token budgets (FR-7), `/readyz`, cost controls.

---

## 7. How we work

- **The user writes the code.** Claude assigns one task at a time with a goal, hints, the concepts involved and a way to verify it. Claude writes implementation code only when told "write it".
- **When stuck:** a hint or concept first; a small snippet only if asked again.
- **After each task:** Claude runs `go vet ./...` and `go test -race ./...`, reviews the code like a senior Go engineer (idiomatic issues, races, leaks, error handling), and suggests improvements.
- **After each feature:** 2–3 interview-style questions.
- **This file is updated** whenever a decision is made or a milestone is completed.

---

## 8. Open questions

- Migration tool: `golang-migrate` or `goose`? (decide in M2)
- UI: API + CLI only, or a minimal server-rendered web page (`html/template`, maybe HTMX)? (decide before M9)
- Production LLM provider and pricing: confirm Gemini's current free tier and limits before M7.
- Chunking strategy beyond fixed size (Markdown-heading-aware?) (revisit after M4 with real results)

---

## 9. Decision log

| Date | Decision | Reason |
|---|---|---|
| 2026-09-30 | Build a RAG "chat with your docs" app | Covers the most Go concepts; easy to explain in interviews |
| 2026-09-30 | Name: `askdocs`, module `github.com/mikasa-404/askdocs` | Chosen by the user |
| 2026-09-30 | Postgres + pgvector as the only datastore | One dependency for data, vectors and the job queue |
| 2026-09-30 | Ollama locally, swappable provider in production | Free development; avoids GPU hosting costs in the cloud |
| 2026-09-30 | Deploy target: ECS Fargate + RDS | Managed, interview-relevant |
