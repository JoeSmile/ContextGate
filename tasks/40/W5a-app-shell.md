# Task 40 · Wave 5a — `/app` 业务壳（工作台 + 运行）

> **状态:** 待实现  
> **依赖:** Wave 2A（Bearer）；Wave 3（run API）；Wave 4（挂起状态可展示；审批入口可链到 5b）  
> **挂接:** §12；J1.3 预览下限；J1.4；J5.3  
> **非目标:** 自由画布；定时；Coze 面板（W6）

---

## 40.50 — 路由壳：`/app` 与登录分流

**Files:**
- Modify: `frontend/src/router.tsx`
- Create: `frontend/src/pages/app/layout.tsx`（或 `shells/AppWorkbench.tsx`）
- Modify: `frontend/src/pages/login.tsx` — `user` → `/app`；`tenant_admin`/`auditor` → `/admin`（可进 `/app` 测流）

- [ ] **Step 1:** 路由表加 `/app/*`，旧 `/panels/*` 保留或重定向策略写清
- [ ] **Step 2:** RequireAuth 认 Bearer
- [ ] **Step 3:** Commit `feat(fe): /app shell routes and login home`

---

## 40.51 — 工作台：模板 + 表单建链

**Files:**
- Create: `frontend/src/pages/app/Workbench.tsx`
- Create: `frontend/src/api/workflows.ts`
- Create: 最小模板（含：**可见但缺 permission、requestable=true** 的节点，供金线挂起）

**目录语义（D10）:**
- 列表：**可见** capability；**不可见** 不展示  
- **可见但当前不可调** 的可展示，角标「运行时可能需审批」（勿当成保存拦截）  
- 表单字段来自 capability **param_spec**；必填/类型前端校验 + 依赖服务端 40.30/31

- [ ] **Step 1:** 选模板 → 按 param_spec 填参 → `POST /api/workflows` 草稿
- [ ] **Step 2:** 不可见不出现；可见可存（含缺权 requestable 节点）
- [ ] **Step 3:** Commit `feat(fe): workbench template+form visible-save`
- [ ] **空态:** 无模板/无草稿时有可行动文案（去申请权限 / 联系 admin）

---

## 40.52 — 运行按钮 + 结果 / 历史

**Files:**
- Create: `frontend/src/pages/app/RunHistory.tsx`
- Modify: Workbench

- [ ] **Step 1:** 「运行」→ `POST .../runs` → 轮询/展示 status + 节点日志
- [ ] **Step 2:** suspended 态展示「待审批」+ 链到说明（审批在 `/admin` 或同壳「待我批」若角色允许）
- [ ] **Step 3:** Commit `feat(fe): run button and history on /app`
- [ ] **Step 4 (Optional · §10.4 #4):** 按钮防抖 / 429 提示 — 锦上添花，不挡 5a Done

---

## 40.53 — 「待我批 / 我发起的挂起」分栏（业务经理）

**Files:**
- Create: `frontend/src/pages/app/Approvals.tsx`
- Create: `frontend/src/api/workflowApprovals.ts`

`dept_manager` 在 `/app` 可批本部门；与 §12 一致。

- [ ] **Step 1:** inbox 列表 + approve/reject
- [ ] **Step 2:** 「我发起的」只读列表
- [ ] **Step 3:** Commit `feat(fe): app approvals inbox for dept_manager`

---

## Wave 5a Done 标准

- [ ] 有权用户可在 `/app`：**建草稿 → 运行 → 看历史**  
- [ ] 可见但缺权的 requestable 链可保存（为挂起金线铺路）  
- [ ] 挂起态可见  
- [ ] 不用四槽 API Key 登录主路径  
- [ ] 工作台 / 历史 / 待办 **空态可行动**（各至少一句 CTA）
