# Architecture

## Overview

ContextGate is an enterprise LLM gateway built on:

- FastAPI (API layer)
- LangGraph (pipeline orchestration)
- pgvector (vector storage)
- LangFuse (observability)

## Data Flow

See [COMPLIANCE.md](COMPLIANCE.md) for data flow diagram.

## Component Diagram

```
[Client] → [nginx] → [FastAPI] → [LangGraph Pipeline]
    ↑                                  ↓
    |                           [pgvector DB]
    |                                  ↓
    |                           [LangFuse]
    └────←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←←
```

## Pipeline

```
auth_check → load_memory → rate_limiter → cache_check
  ├─ hit → END
  └─ miss → guardrails_input → analyze_parallel → build_context
            → model_router
              ├─ short path → execute skill → END
              └─ long path → llm_generate → guardrails_output
                            → write_memory + audit → END
```

## Key Design Decisions

1. TypedDict over Pydantic for PipelineState — avoids serialization overhead
2. Harness pattern for all external calls — unified observability
3. AES-256-GCM for key encryption — authenticated encryption
4. Fire-and-forget audit logging — doesn't block main request
5. Quality gate scopes mypy to the ContextGate main path; legacy modules are retired incrementally
