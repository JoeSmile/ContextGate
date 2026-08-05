# Task 40 · Wave 4 — S1–S5 + 挂起等批 + resume

> **状态:** 待实现  
> **依赖:** Wave 3 Runner；Wave 1 OrgScope + evaluator  
> **挂接:** J0.6；§9.4；§10 S1–S5 / §10.4；§11.3；B4  
> **红线:** S* 未满足前 **不上** 挂起路径（无权只失败）  
> **验证分层:** API + pytest；待办 UI = W5b

---

## 40.40 — S1–S5 闸门清单落地（代码断言）

**Files:**
- Create: `backend/core/workflow/security_gates.py`
- Test: `tests/test_security_gates.py`

| ID | 代码必须强制 |
|----|----------------|
| S1 | `acting_user` 只来自已校验上下文；禁 body 覆盖；机侧预留 created_by（本 Wave 人侧） |
| S2 | 仅 `requestable=True` 可申请；跨租户/密钥类 capability 黑名单不可申请 |
| S3 | 节点执行前后 OrgScope；查询面已在 W1 |
| S4 | delegation/caps 只是上限；浏览器不发 connector/delegation |
| **S5** | approve / 路由资格 **实时** `resolve_org_scope`；禁止用 TTL 快照放行 |

- [ ] **Step 1:** 负向单测每条至少 1 个（含：撤权后旧 scope 缓存不得 approve）
- [ ] **Step 2:** `assert_hang_wait_allowed(...)`；失败则 runner 走 fail 不 suspend
- [ ] **Step 3:** Commit `feat(workflow): S1-S5 security gates`

---

## 40.41 — 二次鉴权扩展：可申请 → suspend（W3 已 fail-closed）

**Files:**
- Modify: `backend/core/workflow/runner.py`
- Test: `tests/test_runner_secondary_auth.py`

> W3（40.35）已真实接入 evaluator：**无权 → failed**。本任务**不重做**鉴权核心，只扩展分支。

每节点（在 W3 闸之上）:
1. 有权 → 执行（不变）  
2. 无权且不可申请 → 节点失败，run failed（不变，回归保留）  
3. 无权且 `requestable` 且 S* gates OK → 进入 40.42 suspend  
4. 无权且 `requestable` 但 gates 未过 → 仍 failed（不上挂起）

- [ ] **Step 1:** 测：requestable+gates → 走 suspend 路径（可先 mock 到 40.42）；非 requestable → 仍 failed
- [ ] **Step 2:** 改 runner 分支，禁止削弱 W3 fail-closed
- [ ] **Step 3:** Commit `feat(workflow): secondary auth requestable branch to suspend`

---

## 40.42 — Suspend + permission_request

**Files:**
- Migration: `alembic/versions/012_hang_wait.py`
- Modify: runner
- Test: `tests/test_run_suspend.py`

```text
run.status = suspended
request(run_id, node_id, applicant_user_id, needed_perm, org_unit_id, status=pending)
UNIQUE(run_id, node_id)   -- §10.4 #12；重复申请 → 409 或返回已有行
```

- [ ] **Step 1:** 测可申请节点 → suspended + 请求行；双写同 run+node → 唯一约束生效
- [ ] **Step 2:** 实现；**禁止**先执行再补申请
- [ ] **Step 3:** Commit `feat(workflow): suspend run and unique permission request`

---

## 40.43 — 通知路由 + 挂起超时升级

**Files:**
- Create: `backend/core/workflow/notify.py`
- Create: 超时扫描入口（最小：周期 job 或 `GET` 懒检查 / 启动时 hook；写死一种：**进程内 interval 扫描** 或 API 读 inbox 时懒升级）
- Test: `tests/test_hang_notify_routing.py` · `tests/test_hang_timeout_escalate.py`

规则（写死）：
1. 解析申请人主部门  
2. 找该部门 `business_role=dept_manager` → inbox  
3. 找不到 → 升级 `tenant_admin`，audit `escalate_no_dept_manager`  
4. **超时（B4）：** 配置 `HANG_ESCALATE_AFTER`（默认 **24h**）；超时仍 `pending` → 自动 escalate 到 `tenant_admin`（若尚未）、audit `escalate_timeout`；请求可打标 `escalated_at`（**不**另造 `suspended_long` 状态，除非需要 UI 文案——MVP 用字段即可）  
5. **无主动推送**（邮件/IM Later）；§1.4 待办 = 用户打开「待我批」可点

- [ ] **Step 1:** 路由两种单测 + 超时升级单测
- [ ] **Step 2:** 实现 inbox + 超时
- [ ] **Step 3:** Commit `feat(workflow): hang notify and timeout escalate`

---

## 40.44 — Approve / Reject + resume

**Files:**
- Create: `backend/routers/workflow_approvals.py`
- Modify: runner `resume_run(run_id)`
- Test: `tests/test_run_resume.py` · `tests/test_approve_concurrency.py`

Endpoints:
- `GET /api/workflow-approvals/inbox` — 待我批（**实时**业务角色，非 TTL 快照）
- `POST /api/workflow-approvals/{id}/approve`
- `POST /api/workflow-approvals/{id}/reject`

**Approve / Reject（§10.4 #5–8, #11）+ 挂权模型（已决）:**
1. **实时**校验审批人仍为该部门 `dept_manager`（或升级路径的 `tenant_admin`）；撤权 → **403**。  
2. `UPDATE … WHERE status='pending'`；`rowcount=0` → **409**。  
3. approve 与 reject 互斥。  
4. `resume_run`：`suspended→running` **CAS**；与重试并发时仅一 executor。  
5. **挂权（写死）：** approve 成功后，为该 run **签发短命 delegation**（§9.2/9.4）：
   - `credential_kind = delegation`
   - `acting_user_id` = 原申请人（run 主体，不变）
   - `caps` = **仅该挂起节点**所需 `spec.permission`（最小增量；可多值则只含本节点集合）
   - 绑 `run_id` + `node_id`；TTL 分钟级；**不进浏览器**
   - 二次鉴权：delegation.`caps` 是**上限**，仍过 §9.4 闸；本节点凭此 cap 视为可调一次（或至节点 succeeded）
   - **不**改 `user_app_perms` / 平台角色（可收回 = TTL 到期或 run 终态作废 delegation）
   - reject：**不**签发 delegation  
6. **已 succeeded 节点跳过**（40.34）。  
7. **后续未成功节点必须重新二次鉴权**；若下一节点仍缺权且无对应 cap → 再 failed / 再 suspend（按 requestable）。批一次 **不**等于整链放行。

- [ ] **Step 1:** 端到端：挂起 → 批 → 签发 delegation(caps=节点 perm) → 该节点过 → succeeded（或续跑）
- [ ] **Step 2:** 断言：approve **不**写入 `user_app_perms`；delegation TTL/绑 run；双 approve CAS
- [ ] **Step 3:** resume 后下一节点无 cap → 再 failed/suspend
- [ ] **Step 4:** 成员不可批他部；撤权审批人 403
- [ ] **Step 5:** Commit `feat(workflow): approve issues scoped delegation then resume`

---

## 40.45 — 审计字段加固（挂起剧本）

**Files:**
- Modify: audit 调用点
- Test: `tests/test_hang_audit.py`

剧本可查：申请人、审批人、credential_kind、run_id、node_id、escalate 原因。

- [ ] **Step 1:** 测导出/查询含上述字段
- [ ] **Step 2:** Commit `test(audit): hang-wait audit trail`

---

## 40.46 — Wave 4 收口

```bash
uv run pytest tests/test_security_gates.py tests/test_runner_secondary_auth.py \
  tests/test_run_suspend.py tests/test_hang_notify_routing.py tests/test_run_resume.py \
  tests/test_hang_audit.py -q
```

- [ ] S* 负向全绿  
- [ ] 金线 API：缺权挂起 → dept_manager 批 → **delegation caps** → resume 完成  
- [ ] Code review（挂权模型已决，无 Important 挂权项）

---

## Wave 4 Done 标准

- [ ] J0.6 API 可证伪  
- [ ] 通知路由与 §11.3 一致  
- [ ] 无「gates 未过仍 suspend」
