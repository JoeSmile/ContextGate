# Langfuse v4 Upgrade Implementation Plan

> **For agentic workers:** Execute task-by-task. Spec: `docs/superpowers/specs/2026-08-07-langfuse-v4-upgrade-design.md`

**Goal:** Replace EOL Langfuse v2 (image + SDK) with OSS v4 stack + Python SDK v4, preserving observe/prompt/sampling APIs.

**Architecture:** Dedicated Langfuse sidecars (web/worker/CH/minio/pg/redis) in compose; thin adapter in `backend/observability/` over SDK v4 `observe`/`get_client`.

**Tech Stack:** `langfuse/langfuse:4`, `langfuse-worker:4`, ClickHouse, MinIO, Python `langfuse>=4.7,<5`

**Global Constraints:** Local wipe OK; isolate Langfuse Redis/PG from business; keep UI `:3001` and seed keys; Phase 2 scale docs already in spec (no impl).

---

### Task 1: Compose v4 stack (local + prod)

**Files:** `docker-compose.local.yml`, `docker-compose.prod.yml`, optionally drop langfuse init from business postgres

- [x] Replace `langfuse:2` with web+worker+clickhouse+minio+langfuse-postgres+langfuse-redis
- [x] Point app `LANGFUSE_HOST` / `LANGFUSE_BASE_URL` at `http://langfuse-web:3000`
- [x] Keep host port `3001`, init seed keys

### Task 2: SDK bump

**Files:** `pyproject.toml`, `uv.lock`

- [x] `uv add "langfuse>=4.7.0,<5"` → 4.14.3

### Task 3: Observability adapter

**Files:** `backend/observability/langfuse_client.py`, `decorators.py`, callers if needed

- [x] v4 client init (`LANGFUSE_BASE_URL` + host fallback)
- [x] `observe` from `langfuse`; `langfuse_context` shim via `get_client()`
- [x] flush/discard adapted; graph parent kwargs → `langfuse_trace_id`
- [x] tests/conftest.py quiet OTLP unless `LANGFUSE_IN_TESTS=1`

### Task 4: Docs + env

**Files:** `config.env.example`, `examples/qa/LANGFUSE.md`, design status, nginx upstream

- [x] Document wipe / new services / pitfalls pointer

### Task 5: Verify

- [x] ruff + pytest (prompt/memory/ab) — 40 passed
- [ ] Optional: compose up smoke if Docker available（需 Joe wipe 后 `docker compose -f docker-compose.local.yml up -d`）
