# Task 04: LangGraph 管线重构

> ⚠️ **最大 Task，10 个 Subtask。**
> **前置依赖:** `tasks/02-auth-rbac.md`, `tasks/03-tenant-audit.md`
> **完成后:** `tasks/05-langfuse.md`, `tasks/06-cache.md`, `tasks/07-cost-modelrouter-skill.md` 可并行
> PipelineState 是 **TypedDict**，不是 Pydantic。
> 每个节点签名 `(state: PipelineState) -> PipelineState`。
> 并行用 `asyncio.gather`，不是 ThreadPool。
> FastAPI → LangGraph 通过 `pipeline/router.py` 转接。

## Subtask 04.01: PipelineState 定义

**文件:** `backend/pipeline/state.py`
```python
class PipelineState(TypedDict):
    tenant_id: str
    user_id: str
    session_id: str
    user_context: dict                     # {tenant_id, user_id, permissions, role} — 供二级权限校验
    message: str
    raw_input: str
    hot_memory: list[dict]
    warm_memory: dict[str, str]
    emotion: Optional[str]
    emotion_intensity: float
    intent: Optional[str]
    intent_confidence: float
    entities: dict[str, str]
    fingerprint: Optional[str]
    cache_hit: bool
    cache_value: Optional[str]
    pii_redacted: bool
    prompt_injection_detected: bool
    guardrails_passed: bool
    selected_model: str
    estimated_cost: float
    llm_tools: list[dict]
    response: str
    finish_reason: str  # skill_executed | llm_generated | cache_hit | blocked | AUTH_002 | PENDING_APPROVAL
    approval_request_id: Optional[str]    # 人工介入审批单号
    trace_id: str
    total_tokens: int
    total_cost: float
    pipeline_latency_ms: float
    error_code: Optional[str]
    langfuse_span: Optional[Any]
```

## Subtask 04.02: 节点 — auth_check（注入 user_context）

**文件:** `backend/pipeline/nodes/auth_check.py`
调用 Task 02 的 `verify_api_key`，将 tenant + permissions 注入 state 的 `user_context` 字段。

```python
from backend.core.auth.api_key_auth import verify_api_key

async def auth_check(state: PipelineState) -> PipelineState:
    # verify_api_key 是 FastAPI Depends，pipeline 内手动构造
    # 实际开发中由 router.py 的 Depends 预先校验，节点直接拿 tenant
    state["user_context"] = {
        "tenant_id": state["tenant_id"],
        "user_id": state["user_id"],
        "permissions": [],  # 由 Task 02 的 verify_api_key 填充
        "role": "",
    }
    return state
```

## Subtask 04.03: 节点 — load_memory + rate_limiter

**文件:**
- `backend/pipeline/nodes/load_memory.py` — 查 pgvector 加载 L1+L2
- `backend/pipeline/nodes/rate_limiter.py` — 桶令牌检查

## Subtask 04.04: 节点 — cache_check + guardrails_input

**文件:**
- `backend/pipeline/nodes/cache_check.py` — 精确+指纹缓存查询
- `backend/pipeline/nodes/guardrails_input.py` — 具体逻辑在 Task 09

## Subtask 04.05: 节点 — analyze_parallel

**文件:** `backend/pipeline/nodes/analyze_parallel.py`
```python
async def analyze_parallel(state: PipelineState) -> PipelineState:
    emotion_task = _analyze_emotion(state["message"])
    intent_task = _analyze_intent(state["message"])
    emotion_result, intent_result = await asyncio.gather(emotion_task, intent_task)
    state["emotion"] = emotion_result.get("emotion")
    state["emotion_intensity"] = emotion_result.get("intensity", 5.0)
    state["intent"] = intent_result.get("intent")
    state["intent_confidence"] = intent_result.get("confidence", 0.0)
    state["entities"] = intent_result.get("entities", {})
    return state
```

## Subtask 04.06: 节点 — build_context + model_router

**文件:**
- `backend/pipeline/nodes/build_context.py` — 组装上下文
- `backend/pipeline/nodes/model_router.py` — 双路径（Task 07）

## Subtask 04.07: 节点 — llm_generate + guardrails_output + write_memory

**文件:**
- `backend/pipeline/nodes/llm_generate.py` — LLM 调用
- `backend/pipeline/nodes/guardrails_output.py` — 输出审查
- `backend/pipeline/nodes/write_memory.py` — 写回记忆+缓存+审计

## Subtask 04.08: 图组装 + 条件边

**文件:** `backend/pipeline/graph.py`
```python
from langgraph.graph import StateGraph, END

builder = StateGraph(PipelineState)
builder.add_node("auth_check", auth_check)
builder.add_node("load_memory", load_memory)
# ... 所有节点

# 条件边: 缓存命中 → END
builder.add_conditional_edges("cache_check", should_skip_to_end, {...})
# 条件边: model_router 短路径 → END
builder.add_conditional_edges("model_router", route_short_or_long, {...})

compiled_graph = builder.compile()
```

## Subtask 04.09: FastAPI 转接层（注入 user_context）

**文件:** `backend/pipeline/router.py`
```python
@router.post("/chat")
@observe(name="chat.pipeline")
async def chat_pipeline(
    request: ChatRequest,
    background_tasks: BackgroundTasks,
    tenant: TenantContext = Depends(require_permission("chat:write")),
):
    initial = PipelineState(
        tenant_id=tenant.tenant_id,
        user_id=tenant.user_id,
        user_context={
            "tenant_id": tenant.tenant_id,
            "user_id": tenant.user_id,
            "permissions": tenant.extra_permissions,
            "role": tenant.role,
        },
        ...
    )
    final = await compiled_graph.ainvoke(initial)
    log_audit(background_tasks, ...)
    return ChatResponse(response=final["response"], ...)
```

## Subtask 04.10: 老代码退役

- `backend/services/chat_service.py` 加文件头 `# DEPRECATED: 请使用 pipeline/router.py`
- 新 `POST /chat` 路由走 pipeline，旧路由不动但不可达

## 验证

```bash
uv run python -c "
from backend.pipeline.graph import compiled_graph
print(f'✅ {len(compiled_graph.nodes)} 个节点编译成功')
"
```
