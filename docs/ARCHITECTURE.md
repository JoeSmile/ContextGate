# ContextGate Architecture

ContextGate is an enterprise LLM context gateway: auth, multi-tenant isolation, guardrails, observability, model routing, and caching sit in front of model calls.

## Pipeline (LangGraph DAG)

```
auth_check → load_memory → rate_limiter → cache_check
 ├─ hit → END
 └─ miss → guardrails_input → analyze_parallel → build_context
 → model_router
 ├─ short path → execute skill → END
 └─ long path → llm_generate → guardrails_output
 → write_memory + audit → END
```

Nodes live under `backend/pipeline/nodes/` (Batch 4+). Until the StateGraph lands, the FastAPI app still drives the legacy chat path.

## Storage

| Concern | Store |
|--------|--------|
| Relational + vectors | PostgreSQL + **pgvector** |
| Embeddings | OpenAI-compatible API, or deterministic hash fallback (dev only) |
| Secrets / API keys | `api_keys` table (Batch 2+) |

Default local URL:

`postgresql://contextgate:contextgate_local@localhost:5432/contextgate`

## Dual ORM (temporary)

| Import | Use for |
|--------|---------|
| `from backend.database.pgvector_session import ChatMessage, Base` | New vector / session code |
| `from backend.database import VectorChatMessage, VectorBase` | Same, via package aliases |
| `from backend.database import ChatMessage, Base` | **Legacy** `DatabaseManager` ORM only |

Prefer `Vector*` or `pgvector_session` for new work. Package-level `ChatMessage` stays on legacy to avoid breaking existing callers.

## Auth & roles (Batch 2+)

Roles: `super_admin`, `auditor`, `tenant_admin`, `user`.  
Auth: `X-API-Key` → SHA256 → `api_keys`.  
Permissions: `@require_permission("chat:write")`.

## Package layout

- `backend/pipeline/` — LangGraph nodes
- `backend/core/auth/` — auth + RBAC
- `backend/core/guardrails/` — safety
- `backend/database/` — SQLAlchemy + pgvector
- `backend/observability/` — LangFuse
- `backend/skills/builtin/` — skill discovery

## License

Apache 2.0 (see README). Full `LICENSE` / `NOTICE` templates land in Batch 8 (project ownership).
