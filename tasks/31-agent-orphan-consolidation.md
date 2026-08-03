# Task 31: Agent 孤儿副本收口（结构债 Batch 1a）

> **状态: 31.01–31.02 完成；31.03/31.04 收尾中。**
> **拍板(2026-08-03, Joe):**
> 1. 方案 **A**（薄兼容、删孤儿；真源 = `backend/agent/`）
> 2. **本轮只做 Agent 孤儿删除（1a）**；Chat 旁路（1b）与文档/QA 收口（1c）另开 Task 32+
> **依赖:** 无硬依赖。与 Task 30.24（Agent 门面）对齐：门面只包 `backend.agent`，本任务清除「勿引用」的孤儿树。
> **不做:** Chat 三路径合并、SkillRegistry 双份、Capability Hub、改 Agent 内部算法。

---

## 0. 背景与证据

仓库存在两套近乎拷贝的 Agent 实现：

| 轨道 | 路径 | 状态 |
|------|------|------|
| **真源（保留）** | `backend/agent/*` | 经 `backend/services/agent_service.py` → `backend/routers/agent.py` 挂载为 `/agent/*`；Task 30.24 门面也包这里 |
| **孤儿（删除）** | `backend/modules/agent/core/`、`routers/`、`services/` | **未被 `app.py` 挂载**；与真源 Jaccard≈1.0 的重复实现 |
| **共享（保留）** | `backend/modules/agent/protocol/`（尤其 `mcp.py`） | 真源 `planner/reflector/tool_caller/agent_core` **正在 import**，禁止删 |

图谱/共变佐证：`backend/agent/reflector.py` ↔ `modules/.../reflector.py` 共变更耦合 1.0；SIMILAR_TO 成对函数约 14 对、平均 Jaccard ≈ 0.997。

`backend/modules/agent/models/`：仅被 `modules/agent` 包内引用，运行时无外部 import → **随孤儿删除**（若执行前 grep 发现新引用则改为迁到 `backend/agent/models` 或保留，见 31.01）。

---

## 1. 目标态

```
backend/agent/                  ← 唯一 Agent 实现（V2 Runtime + Skills）
backend/services/agent_service.py
backend/routers/agent.py        ← /agent/* 唯一路由
backend/modules/agent/
  protocol/                     ← 仅保留 MCP 协议（真源依赖）
  __init__.py                   ← 只 re-export protocol 公开面（或极薄说明）
```

删除后禁止再出现第二份 `AgentCore` / `MemoryHub` / `Planner` / `Reflector` / `ToolCaller` 实现文件树。

---

## 2. 子任务索引

| # | 内容 | 依赖 | 预计 |
|---|------|------|------|
| 31.01 | 引用审计（硬门禁，未绿不准删） | 无 | 0.5 commit |
| 31.02 | 删除孤儿树 + 改写 `modules/agent/__init__.py` | 31.01 | 1 commit |
| 31.03 | 文档对齐（README / Task 30.24 注记 / AGENTS 若需） | 31.02 | 1 commit |
| 31.04 | 验收冒烟 + 回归 | 31.02, 31.03 | 含在 31.02/03 或独立 commit |

每子任务：独立 AC；Conventional Commits + `Signed-off-by: Joe`。

---

## Subtask 31.01: 引用审计（删前门禁）

> 现状: 口头上「未挂载」，必须以 grep/import 证明无运行时依赖后再删。
> **结论(2026-08-03):** A 类为空，可进 31.02。见附录 A。

**方案:**

1. 全仓检索（含 tests/scripts/examples/tasks 仅作清单，**删代码以 runtime import 为准**）:
   ```bash
   rg -n "modules\.agent\.(core|routers|services)|modules/agent/core|modules/agent/routers|modules/agent/services" \
     --glob '*.py' --glob '*.md' --glob '*.sh'
   rg -n "from backend\.modules\.agent|import backend\.modules\.agent" --glob '*.py'
   rg -n "AgentAction|AgentRequest|AgentResponse|agent_models" --glob '*.py'
   ```
2. 分类结果:
   - **A 类（阻断删除）:** 任何非孤儿树内的 `.py` import → 先改 import 到 `backend.agent` / `protocol`，再进 31.02
   - **B 类（可删）:** 仅孤儿树自引用、或仅文档/archive 提及
   - **C 类（保留）:** `modules.agent.protocol*` 及真源对其的 import
3. 确认 `app.py` 只挂载 `backend.routers.agent`，无 `modules.agent.routers`

**修改文件:** 无（只出审计结论；若有 A 类则本子任务含最小 import 改写）

## AC

- [x] 审计表写入本文件「附录 A」或 PR/commit 说明：A/B/C 三类路径列表
- [x] A 类为空，或已全部改写且 `rg` 无残留运行时 import
- [x] `protocol` 引用列表完整（至少 planner/reflector/tool_caller/agent_core）

---

## Subtask 31.02: 删除孤儿树 + 包入口收口

> 现状: `modules/agent/__init__.py` 仍从 `core/routers/services/models` 导出，import 包即拖入孤儿。

**方案:**

1. **删除目录/文件（整树）:**
   - `backend/modules/agent/core/`（含 `agent/` 下全部实现与 tools 副本、README）
   - `backend/modules/agent/routers/`
   - `backend/modules/agent/services/`
   - `backend/modules/agent/models/`（31.01 确认无外部引用后）
2. **保留:**
   - `backend/modules/agent/protocol/`（及 `protocol/__init__.py`）
3. **重写** `backend/modules/agent/__init__.py`:
   - 不再导出 `AgentCore` / `AgentService` / `agent_router` / models
   - 可选：re-export protocol 稳定符号；或模块 docstring 写明「仅 protocol；Agent 实现见 `backend.agent`」
4. **禁止** 把孤儿改成对 `backend.agent` 的薄 re-export 后长期双路径并存——本任务目标是删副本，不是再留一层壳（`backend.agent` 已是公开 API）

**修改文件:** 删上述树；改 `backend/modules/agent/__init__.py`；若 `protocol/__init__.py` 需补公开导出可小改

## AC

- [x] `backend/modules/agent/core|routers|services|models` 目录不存在
- [x] `backend/modules/agent/protocol/` 仍在，且真源可 import MCP 类型
- [x] `uv run python -c "from backend.agent import get_agent_core, AgentCore; print(get_agent_core)"` 成功
- [x] `uv run python -c "from backend.modules.agent.protocol.mcp import MCPContext"` 成功
- [x] `uv run python -c "from backend.modules.agent import AgentCore"` **失败**（或不再导出——预期 ImpError / 无该名）
- [x] `uv run ruff check backend/ scripts/` 通过
- [x] 已 commit: `refactor: remove orphan modules/agent duplicate of backend.agent`

---

## Subtask 31.03: 文档对齐

> 现状: `backend/agent/README.md`、`modules/.../README`、Task 30.24、archive 批次仍描述双轨。

**方案:**

1. `backend/agent/README.md` — 明确「唯一实现入口」；示例 import 保持 `from backend.agent import ...`
2. 删除已随 31.02 消失的 `modules/agent/core/agent/README.md`（随树删即可）
3. `tasks/30/30.24-agent-gateway.md` — 将「未挂载,勿引用」更新为「Task 31 已删除孤儿；仅保留 protocol」
4. 若 `AGENTS.md` / `docs/ARCHITECTURE.md` 提及 `modules/agent` 双轨，改一句指向 `backend.agent` + `modules.agent.protocol`
5. **不改** `tasks/archive/*` 历史文档（只读追溯）

**修改文件:** 上述活跃文档；禁止借机大改 Task 30 其它子任务

## AC

- [x] 活跃文档不再教用户 `from backend.modules.agent.core...` 取 AgentCore
- [x] 30.24 注记与仓库现状一致（活跃代码注释已更新；`tasks/archive/30/30.24` 历史原文不改）
- [ ] 已 commit: `docs: align agent entrypoint after orphan removal`

---

## Subtask 31.04: 验收冒烟

**验证命令（全绿才算 Task 31 完成）:**

```bash
# 孤儿树不存在
test ! -d backend/modules/agent/core
test ! -d backend/modules/agent/routers
test ! -d backend/modules/agent/services
test ! -d backend/modules/agent/models
test -d backend/modules/agent/protocol

# 真源 + protocol
uv run python -c "from backend.agent import get_agent_core; from backend.services.agent_service import get_agent_service; print('ok')"
uv run python -c "from backend.modules.agent.protocol.mcp import MCPContext, MCPProtocol; print('ok')"

# 路由仍挂真源 agent
uv run python -c "from backend.app import app; print(any(getattr(r,'path',None)=='/agent/chat' or '/agent' in str(getattr(r,'path','')) for r in app.routes))"

uv run ruff check backend/ scripts/
LLM_MOCK=true uv run pytest tests/ -q --tb=short
```

## AC

- [ ] 上列命令全绿
- [ ] `rg -n "modules\.agent\.core|modules/agent/core" --glob '*.py'` 零命中（业务代码）

---

## 3. 风险与回滚

| 风险 | 缓解 |
|------|------|
| 漏网 import 导致启动失败 | 31.01 硬门禁；31.02 后立刻 app import |
| 误删 `protocol` | AC 显式 `test -d protocol` + MCP import |
| Task 30 执行中有人又引用 modules | 30.24 已写勿引用；31 完成后更新 30.24；Capability 门面只包 `backend.agent` |
| models 将来需要 | 真源若需 Pydantic 模型，新建 `backend/agent/models` 或复用 routers 内联模型，不恢复孤儿树 |

回滚: `git revert` 删除 commit（整树删除单 commit 易 revert）。

---

## 4. 后续任务（不在本 Task）

| 编号 | 内容 | 说明 |
|------|------|------|
| **Task 32** | 治理做透改造总纲 | `tasks/32-governance-deepening-roadmap.md`（预算/报表/护栏/样板等） |
| **Task 33（建议）** | Chat 旁路收口（原 Batch 1b）+ 文档/QA（1c） | `/streaming` → pipeline；旁路 deprecate；主入口 = `/chat` + `/agent` |
| Task 30 | Capability Hub | 可与 31 并行；30.24 依赖「只有一套 Agent」——31 完成后更干净 |

---

## 附录 A: 引用审计结果（31.01，2026-08-03）

| 类别 | 路径/符号 | 处理 |
|------|-----------|------|
| A | _(无)_ — 业务 `.py` 无对 `modules.agent.(core\|routers\|services\|models)` 的 import；`tests/`/`scripts/` 亦无 | 不阻断 |
| B | `backend/modules/agent/core/**`（agent_core/memory_hub/planner/reflector/tool_caller/tools） | 删 |
| B | `backend/modules/agent/routers/**`、`services/**`、`models/**` | 删 |
| B | `backend/modules/agent/__init__.py` 当前导出 AgentCore/AgentService/agent_router/models | 31.02 重写 |
| B | `tasks/archive/*` 历史双轨提及（batch-10、25-evidence 等） | 保留只读，不改 |
| C | `backend/modules/agent/protocol/mcp.py` ← `backend/agent/{planner,reflector,tool_caller,agent_core}.py` | 保留 |
| C | `backend/modules/agent/protocol/__init__.py` 公开 MCP 符号 | 保留 |
| — | `app.py` 挂载 `backend.routers.agent` + `backend.routers.agents`；无 `modules.agent.routers` | 确认 |
| — | `AgentAction`/`AgentRequest`/`AgentResponse` 仅 modules 包内定义与自导出，无外部 runtime 消费 | 随 models 删 |

---

## 决策摘要

- **真源:** `backend/agent`（不是 modules）
- **删:** modules 下 core / routers / services / models
- **留:** modules 下 protocol
- **范围:** 仅 1a；Chat 多路径 → Task 32+
