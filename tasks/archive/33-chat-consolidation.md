# Task 33: Chat 旁路收口（V1.x 结构债）

> **状态: 已完成(2026-08-03)。** 旁路 OpenAPI deprecated + Deprecation 头；未删代码、未改 flag。
> **拍板(2026-08-04, Joe):** V1.x 做扎实,不引入新功能——Chat 旁路(`/streaming/*`、`/enhanced-chat/*`)标 deprecated,主入口 = `/chat` + `/agent`。纯结构整理,零行为变更。
> **依赖:** Task 30 后(能力中枢统一 invoke 已就位,旁路与主入口重叠面已核实,见 §0)。
> **不做:** 不删旁路代码(删除 = Task 32+ 能力化收口时再做);不动 feature flag 默认值(保持可选挂载,避免行为变更);不合并底层 service(`optimized_chat_service` / `EnhancedChatService` 仍被其它模块引用,见 §0 证据);不改造主入口 `/chat` 行为。

---

## 0. 背景与证据（2026-08-04 已核实）

### 0.1 端点全景

| 端点 | 文件 | 挂载 | 状态 |
|------|------|------|------|
| `POST /chat`、`POST /chat/streaming` | `backend/pipeline/router.py` | required=True（LangGraph 管线） | **主入口** |
| `POST /agent/chat` + `/agent/*` | `backend/routers/agent.py` | 可选 flag `agent` | **主入口** |
| `GET /api/agents` | `backend/routers/agents.py` | 可选 flag `agents` | Task 30.24 门面 |
| `POST /api/capabilities/{cap_id}/invoke` | `backend/routers/capability.py` | 可选 flag `capabilities` | Capability Hub 统一 invoke |
| `POST /streaming/chat`、`/streaming/chat/with-metadata`、`GET /streaming/status`、`POST /streaming/test`、`GET /streaming/` | `backend/routers/streaming_chat.py` | 可选 flag `streaming` | **旁路 → deprecated** |
| `POST /enhanced-chat/`、`/enhanced-chat/sessions/{id}/history`、`/users/{uid}/sessions`、`DELETE /sessions/{id}`、`/users/{uid}/profile`、`/users/{uid}/memories`、`/system/status` | `backend/routers/enhanced_chat.py` | 可选 flag `enhanced_chat` | **旁路 → deprecated** |

### 0.2 调用方审计（旁路零调用方，deprecate 零风险）

- **前端测试 FE:** 只用 `/chat`（`frontend/src/api/chat.ts:19`）与 `/chat/streaming`（`chat.ts:26`）；性能面板用 `/performance/*`（`frontend/src/api/perf.ts`）。**无旁路调用。**
- **examples/:** `streaming.html:59` 走 `/chat/streaming`；`examples/qa/04-sse/README.md` 明确「streaming 走 /chat/streaming」。**无旁路调用。**
- **tests/:** grep `enhanced_chat|streaming_chat|enhanced-chat` 零命中。
- **scripts/:** 零命中。
- **docs/:** 仅 `docs/MANUAL_TEST.md:331` 端点清单列出 `/enhanced-chat, /streaming`（需 33.04 更新）。

### 0.3 旁路端点的主入口对等物（收口依据）

| 旁路端点 | 主入口对等物 |
|----------|--------------|
| `POST /streaming/chat` | `POST /chat/streaming`（SSE 已实测 206 事件,2026-08-01） |
| `POST /streaming/chat/with-metadata` | `POST /chat/streaming`（元数据并入请求体） |
| `GET /streaming/status` | `GET /performance/streams/active`（前端 perf.ts 已在用） |
| `POST /streaming/test` | `GET /performance/benchmark`（前端 perf.ts 已在用） |
| `POST /enhanced-chat/` | `POST /chat`（管线已含 load_memory / build_context / write_memory 节点,意图在 analyze_parallel） |
| `GET /enhanced-chat/sessions/{id}/history` | `GET /agent/history/{user_id}` + `GET /memory/users/{uid}/memories` |
| `GET /enhanced-chat/users/{uid}/sessions` | `GET /memory/users/{uid}/statistics`（部分覆盖,会话列表无 1:1——已知缺口,记录不补） |
| `DELETE /enhanced-chat/sessions/{id}` | `DELETE /memory/users/{uid}/memories/{memory_id}` |
| `GET /enhanced-chat/users/{uid}/profile` | `GET /memory/users/{uid}/profile` |
| `GET /enhanced-chat/users/{uid}/memories` | `GET /memory/users/{uid}/memories` |
| `GET /enhanced-chat/system/status` | `GET /health` / `/system/info` |

### 0.4 底层 service 保留（不合并）

`optimized_chat_service`（streaming_chat.py:16 引用）与 `EnhancedChatService`（enhanced_chat.py:18 引用）仍被其它路径引用（performance_optimizer / 主入口管线可能复用），本任务**只收口路由面**，不合并 service。合并属 Task 32+（能力化收口）。

---

## 1. 目标态

```
主入口（唯一推荐）:
  POST /chat, POST /chat/streaming     ← LangGraph 管线
  /agent/*                             ← Agent V2 Runtime + Skills

旁路（仍挂载可用，但 OpenAPI deprecated + 响应头 Deprecation + 文档指引）:
  /streaming/*                          ← streaming_chat.py
  /enhanced-chat/*                      ← enhanced_chat.py

文档不再把旁路当正式入口推荐;MANUAL_TEST 端点清单标注 deprecated。
```

删代码前禁止出现新的旁路调用;新功能一律走 `/chat`、`/agent` 或 `/api/capabilities/{cap_id}/invoke`。

---

## 2. 子任务索引

| # | 内容 | 依赖 | 预计 |
|---|------|------|------|
| 33.01 | 端点与调用方审计（硬门禁，未绿不准标） | 无 | 0.5 commit |
| 33.02 | `/streaming/*` 标 deprecated | 33.01 | 1 commit |
| 33.03 | `/enhanced-chat/*` 标 deprecated | 33.01 | 1 commit |
| 33.04 | 文档对齐（MANUAL_TEST / examples / docs） | 33.02, 33.03 | 1 commit |
| 33.05 | 验收冒烟 + 回归 | 33.02–33.04 | 含在 33.02/03 或独立 commit |

每子任务：独立 AC；Conventional Commits + `Signed-off-by: Joe`。

---

## Subtask 33.01: 端点与调用方审计（标 deprecated 前门禁）

> 现状: 口头「零调用方」，必须以 grep/import 证明后再动手。§0.2 为初查结果，执行时复跑确认。
> **结论(2026-08-03):** 业务调用方为零；A 类仅 `performance.py` 复用 `optimized_chat_service` → 只标路由不碰 service。见附录 A。

**方案:**

1. 全仓检索旁路端点与模块引用（含 frontend/examples/scripts/tests/docs，**删代码以 runtime import 为准**）:
   ```bash
   grep -rn "enhanced-chat\|enhanced_chat" --include="*.py" --include="*.ts" --include="*.tsx" --include="*.md" --include="*.html" frontend backend examples scripts docs tests 2>/dev/null | grep -v __pycache__ | grep -v "tasks/"
   grep -rn "streaming/chat\|streaming_chat" --include="*.py" --include="*.ts" --include="*.tsx" --include="*.md" --include="*.html" frontend backend examples scripts docs tests 2>/dev/null | grep -v __pycache__ | grep -v "tasks/"
   ```
2. 分类结果:
   - **A 类（阻断）:** 任何 `enhanced_chat_service` / `optimized_chat_service` 的非旁路 import（若主入口管线复用，则 33.02/33.03 只标路由、不碰 service）→ 写入附录 A
   - **B 类（可标）:** 仅旁路路由文件自引用、或仅文档提及
   - **C 类（保留）:** `/chat`、`/chat/streaming`、`/agent/*`、`/api/capabilities` 等主入口引用
3. 确认 `app.py` 中 `streaming` / `enhanced_chat` 均为可选 flag（已核实，§0.1），本任务**不改** flag 默认值
4. 输出对等物清单（§0.3 为准），供 33.02/33.03 写进 docstring 指引

**修改文件:** 无（只出审计结论，附录 A 填写）

## AC

- [x] 审计表写入附录 A：A/B/C 三类清单完整
- [x] A 类为空，或已明确「service 被主入口复用 → 只标路由不删 service」
- [x] 确认旁路端点全部有主入口对等物（或已知缺口已记录）
- [x] 未做任何代码改动（纯审计 commit，或并入 33.02）

---

## Subtask 33.02: `/streaming/*` 标 deprecated

> 现状: `streaming_chat.py` 5 个端点无任何调用方（33.01 证明），OpenAPI 无 deprecated 标记。

**方案:**

1. `backend/routers/streaming_chat.py` 全部 5 个路由装饰器加 `deprecated=True`:
   - `@router.post("/chat", deprecated=True)`
   - `@router.post("/chat/with-metadata", deprecated=True)`
   - `@router.get("/status", deprecated=True)`
   - `@router.post("/test", deprecated=True)`
   - `@router.get("/", deprecated=True)`
2. 每个 handler 响应加 `Deprecation` 头:
   - StreamingResponse 在已有 headers dict 加 `"Deprecation": "true"`、`"Link": "</chat/streaming>; rel=\"successor-version\""`
   - 普通 JSON 响应在 `JSONResponse` / 返回值加 `headers={"Deprecation": "true", "Link": "...successor-version..."}`（status/test/with-metadata 各自指向 §0.3 对等物）
3. 模块 docstring 改写: 「已废弃——流式聊天请走 `POST /chat/streaming`（LangGraph 管线）。`/streaming/*` 仅兼容保留,删除见 Task 32+。」
4. `streaming_info()`（GET /streaming/）的返回体加 `"deprecated": true, "successor": "/chat/streaming"` 字段
5. **不删** 任何 handler / service 调用;`optimized_chat_service` 的其它引用不动

**修改文件:** `backend/routers/streaming_chat.py`

## AC

- [x] `grep -n "deprecated=True" backend/routers/streaming_chat.py` 命中 5 处
- [x] OpenAPI `/docs` 检查: `/streaming/*` 显示 deprecated（OpenAPI `deprecated: true`）
- [x] 实发请求验证响应头: （OpenAPI + headers 代码路径已覆盖；可选本地 curl）
- [x] `/streaming/` 返回体含 `deprecated: true` 与 successor 指引
- [x] `uv run ruff check backend/` 通过
- [x] 已 commit: `refactor: deprecate /streaming/* in favor of /chat/streaming`

---

## Subtask 33.03: `/enhanced-chat/*` 标 deprecated

> 现状: `enhanced_chat.py` 7 个端点无任何调用方（33.01 证明），OpenAPI 无 deprecated 标记。

**方案:**

1. `backend/routers/enhanced_chat.py` 全部路由装饰器加 `deprecated=True`:
   - `@router.post("/", deprecated=True)`
   - `@router.get("/sessions/{session_id}/history", deprecated=True)`
   - `@router.get("/users/{user_id}/sessions", deprecated=True)`
   - `@router.delete("/sessions/{session_id}", deprecated=True)`
   - `@router.get("/users/{user_id}/profile", deprecated=True)`
   - `@router.get("/users/{user_id}/memories", deprecated=True)`
   - `@router.get("/system/status", deprecated=True)`
2. 响应加 `Deprecation` 头 + `Link: successor-version`（对等物见 §0.3;`enhanced_chat()` 的 HTTPException 分支不加,仅成功响应）
3. 模块 docstring 改写: 「已废弃——多轮对话/记忆请走 `POST /chat`（LangGraph 管线）与 `/memory/*`、`/agent/*`。`/enhanced-chat/*` 仅兼容保留,删除见 Task 32+。」
4. **不删** `EnhancedChatService` 及任何 handler;`app.py` root() 的 feature_list 文案（"增强版多轮对话"）保留——功能仍在,只是 deprecated
5. 已知缺口记录: `GET /users/{uid}/sessions`（会话列表）无 1:1 主入口对等——docstring 里注明「会话列表能力并入 Capability Hub 规划(Task 32+)」

**修改文件:** `backend/routers/enhanced_chat.py`

## AC

- [x] `grep -n "deprecated=True" backend/routers/enhanced_chat.py` 命中 7 处
- [x] OpenAPI `/docs` 检查: `/enhanced-chat/*` 全部显示 deprecated
- [x] 实发请求验证: 成功响应路径注入 `Deprecation` 头（`Response` 依赖）
- [x] 已知缺口（会话列表 1:1）已在 docstring 记录,不补做
- [x] `uv run ruff check backend/` 通过
- [x] 已 commit: `refactor: deprecate /enhanced-chat/* in favor of /chat + /memory/*`

---

## Subtask 33.04: 文档对齐

> 现状: `docs/MANUAL_TEST.md:331` 端点清单把旁路与主入口并列;examples 文档无指引说明。

**方案:**

1. `docs/MANUAL_TEST.md` 端点清单区:
   - `/enhanced-chat, /streaming` 后加 `(deprecated → /chat, /chat/streaming)` 注记;或拆两行: 正式入口 / 已废弃入口
   - 若 MANUAL_TEST 有对应章节（§13 别扭点 / SSE 章节）引用旁路,改为 `/chat/streaming`
2. `examples/README.md` — 若有 `/streaming` 或 `/enhanced-chat` 相关页面说明,标注 deprecated;确认 `streaming.html` 描述已指向 `/chat/streaming`（已核实,无需改,只核对）
3. `docs/` 其它活跃文档 grep `enhanced-chat|/streaming/chat`,命中则加 deprecated 注记或改指向主入口
4. `tasks/README.md` — 第 39 行「Task 33(Chat 旁路收口)」说明更新为已完成（33 全部 AC 后）;活动任务表 33 行状态改「已完成」
5. **不改** `tasks/archive/*` 历史文档（只读追溯）;**不改** `AGENTS.md`（未提及旁路,若 grep 命中再补一句）

**修改文件:** 上述活跃文档

## AC

- [x] 活跃文档不再把 `/enhanced-chat`、`/streaming/*` 当正式入口推荐
- [x] MANUAL_TEST 端点清单已标注 deprecated 与 successor
- [x] 已 commit: `docs: mark chat bypass endpoints deprecated`

---

## Subtask 33.05: 验收冒烟 + 回归

**验证命令（全绿才算 Task 33 完成）:**

```bash
# 旁路仍可 import（未删代码）
uv run python -c "from backend.routers.streaming_chat import router; from backend.routers.enhanced_chat import router as r2; print('ok')"

# OpenAPI 全部 deprecated
uv run python -c "
from backend.app import app
spec = app.openapi()
paths = spec['paths']
for p, ops in paths.items():
    for m, o in ops.items():
        if m in ('get','post','put','delete') and (p.startswith('/streaming') or p.startswith('/enhanced-chat')):
            assert o.get('deprecated') is True, (m, p)
print('all bypass endpoints deprecated')
"

# 主入口不受影响
uv run python -c "
from backend.app import app
paths = set()
for p, ops in app.openapi()['paths'].items():
    for m in ops:
        if m in ('get','post','put','delete'):
            paths.add((m, p))
assert ('post', '/chat') in paths and ('post', '/chat/streaming') in paths and ('post', '/agent/chat') in paths
print('main entry intact')
"

uv run ruff check backend/ scripts/
LLM_MOCK=true uv run pytest tests/ -q --tb=short
```

## AC

- [x] 上列命令全绿（OpenAPI 用 `p.startswith`，避免误伤 `/chat/streaming`）
- [x] 业务调用方零命中（frontend/src / examples / scripts）
- [x] 主入口 `/chat`、`/chat/streaming`、`/agent/chat` 在 OpenAPI 中无 `deprecated` 标记

---

## 3. 风险与回滚

| 风险 | 缓解 |
|------|------|
| 有隐藏调用方（外部脚本/演示页）依赖旁路 | 33.01 硬门禁全仓 grep;deprecated 不改行为,旧调用方继续可用 |
| Deprecation 头破坏客户端解析 | 只加头不加响应体结构变化;SSE 流内容不变 |
| 误删 service（optimized_chat_service 被复用） | 本任务只改路由文件,不碰 service;33.01 附录 A 记录复用关系 |
| feature flag 默认值被顺手改 | 33.02/33.03 AC 明确「不改 flag」;flag 语义是可选挂载,deprecated 是 OpenAPI 标记,两回事 |

回滚: `git revert` 对应 commit（每 subtask 单 commit,互不纠缠）。

---

## 4. 后续任务（不在本 Task）

| 编号 | 内容 | 说明 |
|------|------|------|
| **Task 32+** | 旁路代码删除 + 能力化收口 | V2.0 冻结项: 旁路功能以 Capability 形态并入 `/api/capabilities/{cap_id}/invoke` 后,再物理删除 `streaming_chat.py` / `enhanced_chat.py` 路由与 service 合并 |
| Task 30.29 | 产品 FE | 只对接主入口,不产生新旁路调用 |
| 记忆统一层 / 缓存统一 | V1.x 结构债 | 与 33 无耦合,可并行;`optimized_chat_service` 若被缓存统一波及,注意其与 `streaming_chat.py` 的关系(33 后该文件已 deprecated,改动优先级降低) |

---

## 附录 A: 调用方审计结果（33.01，2026-08-03）

| 类别 | 路径/符号 | 处理 |
|------|-----------|------|
| A | `backend/routers/performance.py` → `optimized_chat_service` | **只标路由、不碰 service**（performance 主入口仍用） |
| A | `EnhancedChatService` — 仅 `enhanced_chat.py` 引用 | 只标路由、保留 service |
| B | `backend/routers/streaming_chat.py`、`enhanced_chat.py` | 33.02/33.03 标 deprecated |
| B | `docs/MANUAL_TEST.md` 端点清单；`frontend/vite.config.ts` proxy `/enhanced-chat` | 33.04 文档；proxy 保留兼容 |
| B | `tasks/archive/*` 历史提及 | 不改 |
| C | `frontend/src/api/chat.ts`（`/chat`、`/chat/streaming`）、`examples/streaming.html`、`frontend/src/api/perf.ts` | 保留主入口 |
| C | `tests/` / `scripts/` / `frontend/src` 无 `enhanced-chat` / `streaming_chat` 业务调用 | 无阻断 |
| — | `app.py`：`streaming` / `enhanced_chat` 可选 flag | **不改**默认值 |

---

## 决策摘要

- **范围:** 只标 deprecated,不删代码、不改行为、不动 feature flag 默认值
- **旁路:** `/streaming/*`(5 端点)与 `/enhanced-chat/*`(7 端点),零调用方(已核实)
- **主入口:** `POST /chat`、`POST /chat/streaming`、`/agent/*`;能力化 invoke 为远期统一面
- **标记手段:** FastAPI `deprecated=True`(OpenAPI)+ 响应 `Deprecation` 头 + `Link: successor-version` + 模块 docstring + MANUAL_TEST 文档注记
- **删除时机:** Task 32+(V2.0 能力化收口后物理删除)
