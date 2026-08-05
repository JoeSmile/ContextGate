# Task 40 · Wave 3 — Workflow Runner（IR 快照 + 写节点幂等）

> **状态:** 待实现  
> **依赖:** Wave 1（OrgScope）；Wave 2A（人侧入口）。契约预留机侧 `credential_kind`，本 Wave **不**接 machine run API。  
> **挂接:** §10 **T1**；J1.1 / J1.4（API）；master D5  
> **验证分层:** **仅 API + pytest**（UI = W5a）  
> **非目标:** 定时 cron；Coze 导入；挂起 resume（W4）；画布

---

## 40.30 — Workflow 定义 + IR 模型

**Files:**
- Create: `alembic/versions/010_workflows.py`
- Create: `backend/core/workflow/ir.py`
- Create: `backend/core/workflow/models.py`（ORM）
- Test: `tests/test_workflow_ir.py`

**IR（顺序 MVP）:**
```python
class ParamSpec(BaseModel):  # 最小声明；亦可从 capability.spec 读取合并
    name: str
    type: Literal["string", "number", "boolean", "enum"] = "string"
    required: bool = False
    enum_values: list[str] | None = None

class WorkflowNode(BaseModel):
    node_id: str
    capability_id: str
    params: dict[str, Any] = {}
    # 保存时用 capability 的 param spec（或节点覆盖）做类型/必填校验
    requestable: bool = False  # W4：运行缺权时可申请挂起

class WorkflowIR(BaseModel):
    version: str
    nodes: list[WorkflowNode]  # 有序
```

表：`workflows(id, tenant_id, org_unit_id, name, status draft|published|archived, ir_json, created_by, updated_at, version INT NOT NULL DEFAULT 1)`  
- `version` = 乐观锁计数（§10.4 #1–3）  
- `published` 行 **冻结** IR；再编辑须 `POST .../fork-draft` 或等价开新 draft（40.31）  
- **params：** 保存/发布前按 capability `param_spec`（Hub spec 或内置表）校验；缺 spec 的 capability → 仅允许空 params 或拒绝带参（写死：**无 spec 则 params 必须为空对象**）

- [ ] **Step 1:** pydantic + param 校验单测（必填缺失 → 拒；无 spec 带参 → 拒）
- [ ] **Step 2:** 迁移 + ORM；capability 侧最小 `param_spec` 字段或并行 JSON 配置
- [ ] **Step 3:** Commit `feat(workflow): IR model with param spec`

---

## 40.31 — Workflow CRUD API（可见保存 + 乐观锁 + 删草稿）

**Files:**
- Create: `backend/core/workflow/service.py`
- Create: `backend/routers/workflows.py`
- Modify: `backend/app.py`
- Test: `tests/test_workflow_crud.py` · `tests/test_workflow_optimistic_lock.py` · `tests/test_workflow_visible_vs_permitted.py`

**语义写死（D10 / 金线闭合 (a)）:**

| 时机 | 校验 | 失败 |
|------|------|------|
| UI 目录 / 保存 / 发布 | capability 对 acting_user **可见**（租户+目录可见性） | 不可见 → 400/403 |
| 保存 / 发布 | params 符合 param_spec | 400 |
| **运行时节点** | **可调** = `evaluate_permission(spec.permission)`（§9.4） | 无权+requestable+S* → suspend；否则 failed |

**禁止**在保存时要求「可见且可调」——否则挂起金线不可达。

Endpoints:
- `POST /api/workflows` — 草稿；节点须 **可见**；params 校验
- `GET /api/workflows` — OrgScope 过滤
- `GET /api/workflows/{id}`
- `PATCH /api/workflows/{id}` — **仅 draft**；`base_version` 乐观锁 → 409
- `POST /api/workflows/{id}/publish` — 仅 draft；原子发布；第二发 409；published 不可 PATCH
- `POST /api/workflows/{id}/fork-draft` — 从 published 开新 draft
- `DELETE /api/workflows/{id}` — **仅 draft**；published 或已有 run → **409**

Depends：`verify_session` 或 dual（人侧）。

**AC（§10.4 #1–3 · 乐观锁硬门槛 + D10）:**
- 双 PATCH → 409；双 publish → 409；published 不可 PATCH  
- **可见但不可调** 的 requestable 节点 **可以保存/发布**  
- 不可见节点保存 → 400/403  
- 删 draft 成功；删 published / 有 run → 409  

- [ ] **Step 1:** 可见可存 / 不可见拒 / 乐观锁 / DELETE 单测
- [ ] **Step 2:** 实现
- [ ] **Step 3:** Commit `feat(workflow): CRUD visible-save optimistic-lock delete-draft`

---

## 40.32 — Run 表 + 状态机

**Files:**
- Create: `alembic/versions/011_workflow_runs.py`
- Create: `backend/core/workflow/run_state.py`
- Test: `tests/test_run_state_machine.py`

```text
workflow_runs(
  id, tenant_id, workflow_id, org_unit_id,
  status,              -- pending|running|succeeded|failed|suspended(W4)
                       -- **MVP 不含 cancelled**（取消 = Later；勿实现半套）
  ir_snapshot JSONB,   -- 启动时固化（40.33）
  acting_user_id,
  credential_kind,
  error_code, error_message,
  created_at, updated_at, finished_at
)
workflow_run_nodes(
  id, run_id, node_id,
  status, attempt, idempotency_key UNIQUE,
  output_json, error_message,
  started_at, finished_at
)
```

合法迁移（MVP）：`pending→running→succeeded|failed`；W4 加 `running↔suspended`。  
**不**含 `cancelled`（有定义无实现会误导；需要时另开 task：`POST /runs/{id}/cancel`）。

- [ ] **Step 1:** 状态机单测非法迁移抛错
- [ ] **Step 2:** ORM + 迁移
- [ ] **Step 3:** Commit `feat(workflow): run tables and state machine`

---

## 40.33 — 启动时固化 `ir_snapshot`

**Files:**
- Modify: `backend/core/workflow/runner.py`（新建入口 `start_run`）
- Test: `tests/test_ir_snapshot.py`

规则：
- `start_run` 深拷贝当前 workflow.ir → `run.ir_snapshot`
- 之后改 workflow 定义 **不影响** 在飞 / 历史 run
- 执行只读 `ir_snapshot`

- [ ] **Step 1:** 测：start 后改 workflow，再执行仍用旧 IR
- [ ] **Step 2:** 实现
- [ ] **Step 3:** Commit `feat(workflow): freeze ir_snapshot on run start`

---

## 40.34 — 写节点幂等键 `(run_id, node_id)`

**Files:**
- Modify: `backend/core/workflow/runner.py`
- Test: `tests/test_write_node_idempotency.py`

规则：
- 每个节点执行前：`idempotency_key = f"{run_id}:{node_id}"`（或 DB UNIQUE）
- 若节点已 `succeeded`，重试 **跳过副作用**，返回缓存 output
- 适用于调用 Hub `invoke` / 出站写（本 Wave 至少对「标记为 write」或所有节点统一幂等落库）

- [ ] **Step 1:** 测：同一 node 执行两次，副作用计数器 +1 一次
- [ ] **Step 2:** 实现
- [ ] **Step 3:** Commit `feat(workflow): write-node idempotency by run_id+node_id`

---

## 40.35 — Runner 顺序执行 + **真实二次鉴权**（无权→failed）

**Files:**
- Create: `backend/core/workflow/runner.py`（完整）
- Modify: `backend/core/capability/invoke.py`（只读调用，不改语义）
- Test: `tests/test_runner_execute.py`
- Test: `tests/test_runner_secondary_auth_fail.py`（可与 execute 合并）

```python
async def start_and_run(*, workflow_id, tenant: TenantContext, org_scope: OrgScope) -> Run: ...
async def execute_run(run_id: str) -> Run: ...
```

**二次鉴权（写死 — 与 master Wave 3 验收对齐）:**
1. 每节点 **真实**调用权限求值器 + capability 可见性（W1 `evaluate_permission`），相对 `acting_user`。  
2. **无权 → 节点失败，`run.status=failed`，audit 记原因**（含 `needed_perm` / capability_id）。  
3. **本 Wave 不进入 `suspended`**——可申请/挂起分支留给 W4；W3 视一切无权为失败（即使节点将来会标 `requestable`）。  
4. 「占位」**仅**指：不实现 suspend/resume/通知；**不是**指鉴权空函数。

**Run 级并发（§10.4 #5）:**
- 进入执行：`UPDATE workflow_runs SET status='running' WHERE id=? AND status IN ('pending')`（W4 另加 `suspended`→`running`）；`rowcount!=1` → 拒绝第二 executor（409 或静默 no-op，写死 **409**）。  
- 节点幂等 **不**替代本条。

**AC（必须有测）:**
- 低权限用户跑含无权节点的已发布链 → `failed` + 审计原因可读  
- 有权两节点链 → `succeeded`  
- **双 execute 同 run → 仅一成功进入 running**  
- LLM/外发仍走 Hub/治理，禁裸 SDK  
- `credential_kind` / `acting_user_id` 从 TenantContext 写入 run

- [ ] **Step 1:** 单测：低权 → failed + audit；有权 → succeeded；双 execute CAS
- [ ] **Step 2:** 实现顺序执行 + **真实** evaluator 闸 + run CAS + 节点日志
- [ ] **Step 3:** Commit `feat(workflow): sequential runner with real secondary auth (fail-closed)`

---

## 40.36 — Run API：启动 / 查询 / 历史

**Files:**
- Create: `backend/routers/workflow_runs.py`
- Test: `tests/test_workflow_runs_api.py`  
  - Optional: `tests/test_run_create_throttle.py`

Endpoints:
- `POST /api/workflows/{id}/runs` — 按钮执行（人侧）
- `GET /api/runs/{run_id}`
- `GET /api/runs` — OrgScope 列表；**必含分页** `limit`（默认 20，max 100）+ `offset` 或 `cursor`（写死一种：**limit+offset**）
- `GET /api/runs/{run_id}/nodes`

**Optional（§10.4 #4 · 锦上添花，不挡 Done）:** 同 `(acting_user_id, workflow_id)` **1s** 窗口第二次创建 → 429。

**AC:** 列表无分页参数时用默认 limit；超 max → 400。

- [ ] **Step 1:** API 测跑通一条两节点链 + 分页
- [ ] **Step 2:** 实现
- [ ] **Step 3:** Commit `feat(workflow): run start/history APIs with pagination`
- [ ] **Step 4 (Optional):** 创建限流单测 + 实现

---

## 40.37 — 审计挂 run_id（params 不进明文）

**Files:**
- Modify: runner 内 `log_audit` 调用
- Test: `tests/test_runner_audit.py`

每 run / 每节点至少一条 audit：`acting_user_id`, `credential_kind`, `run_id`, `node_id`。

**敏感参数（写死）:**
- `params` / IR 快照内容 **不**以明文写入 `audit_logs`（可记 param **键名**列表或 hash）。  
- 节点 output 默认不进 audit 全文；需要时只记截断摘要。  
- 链 B 出站密钥同规则。

- [ ] **Step 1:** 测 audit 可按 run_id 查；断言 params 值不出现在 audit 行
- [ ] **Step 2:** 实现
- [ ] **Step 3:** Commit `feat(audit): run_id/node_id without plaintext params`

---

## 40.38 — Wave 3 收口

```bash
uv run pytest tests/test_workflow*.py tests/test_runner*.py tests/test_ir_snapshot.py tests/test_write_node_idempotency.py -q
uv run ruff check backend/core/workflow backend/routers/workflows.py backend/routers/workflow_runs.py
```

- [ ] API 可演示：建草稿 → 发布 → run → 历史  
- [ ] **低权跑无权节点 → failed + 审计**（40.35 AC）  
- [ ] IR 快照 + 幂等单测绿  
- [ ] **无** UI（W5）、**无** suspend（W4）、**无** machine 入口  
- [ ] Code review

---

## Wave 3 Done 标准

- [ ] T1 最小 Runner 可 API 验收  
- [ ] 二次鉴权 **fail-closed** 已接入（非空钩子）  
- [ ] `ir_snapshot` + `(run_id,node_id)` 幂等有测  
- [ ] OrgScope 过滤 run 列表
