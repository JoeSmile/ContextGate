# Task 40 · Wave 5b — `/admin` 管理台

> **状态:** 待实现  
> **依赖:** Wave 1 org API；Wave 3/4；Wave 2A  
> **挂接:** §12 `/admin`；J0.2 审计面；J1.3 步骤只读预览  
> **非目标:** 画布编辑器；治理大盘 KPI

---

## 40.55 — `/admin` 壳与导航

**Files:**
- Create: `frontend/src/pages/admin/layout.tsx`
- Modify: `frontend/src/router.tsx`

导航最小：概览 · 组织 · 流程 · 权限/挂起 · 审计 ·（集成 Key 可灰显待 2B）

- [ ] **Step 1:** 路由 + 角色门禁（tenant_admin / auditor / super_admin）
- [ ] **Step 2:** Commit `feat(fe): /admin shell nav`

---

## 40.56 — 组织树 CRUD UI

**Files:**
- Create: `frontend/src/pages/admin/OrgTree.tsx`
- Create: `frontend/src/api/org.ts`

- [ ] **Step 1:** 树展示 + 建部门 + 调 membership / 业务角色
- [ ] **Step 2:** Commit `feat(fe): admin org tree UI`

---

## 40.57 — 流程步骤列表 + 只读预览 + 发布

**Files:**
- Create: `frontend/src/pages/admin/WorkflowAdmin.tsx`

**§10.4 #2:** 发布调 publish API；若 409（版本冲突）提示刷新；**不可**在 UI 上直接改 published——须「基于此版开草稿」。

- [ ] **Step 1:** 列表草稿/已发布；点开步骤只读预览（非画布）
- [ ] **Step 2:** 发布按钮；409 提示；fork-draft 入口
- [ ] **Step 3:** Commit `feat(fe): admin workflow preview publish and fork`

---

## 40.58 — 挂起治理 + 审计导出入口

**Files:**
- Create: `frontend/src/pages/admin/HangInbox.tsx`
- Modify: 现有 audit 面板迁入 `/admin/audit` 或包一层

- [ ] **Step 1:** admin 可见升级上来的挂起；可批
- [ ] **Step 2:** auditor 导出含 acting_user / credential_kind / run_id
- [ ] **Step 3:** Commit `feat(fe): admin hang inbox and audit export entry`

---

## Wave 5b Done 标准

- [ ] admin 可建组织、发流程、处理升级挂起、导审计  
- [ ] 发布 409 / fork-draft 可用  
- [ ] 无画布编辑  
- [ ] 组织/流程/挂起/审计页 **空态可行动**
