# askdocs
A "chat with your docs" RAG service in Go. Module: `github.com/mikasa-404/askdocs`.

- Upload docs → split into chunks → embed → store in Postgres with pgvector → answer questions with cited sources, streamed back over SSE.
- LLM: Ollama locally (`nomic-embed-text` for embeddings, `llama3.2:3b` for chat). The providers sit behind interfaces so a paid API such as Gemini can replace Ollama in production.
- Ingestion runs asynchronously on a small Postgres-backed job queue with a worker pool.
- Docker + docker-compose locally; deploy to AWS ECS Fargate + RDS Postgres.

**`docs/REQUIREMENTS.md` is the single source of truth** for features, milestones, decisions and progress. Read it at the start of each session to find the current milestone; update it (checkboxes, decision log, open questions) whenever a decision is made or a milestone is finished.

# How to help me
I'm learning Go through this project and preparing for backend interviews.
- Don't write implementation code unless I explicitly say "write it".
- When I'm stuck, give a hint or point to the concept first, then a small snippet only if I ask again.
- Review my code like a senior Go engineer: idiomatic issues, races, leaks, error handling.
- After each feature, ask me 2-3 interview-style questions about what I just built.
- Always run `go vet` and `go test -race ./...` after changes.
