# Task 40 · Wave 5c — `/dev` 壳（可并行，不挡 7A）

> **状态:** 待实现（可与 5a/5b 并行）  
> **依赖:** Wave 2A 建议完成  
> **挂接:** §12 `/dev`  
> **非目标:** 产品用户走 `/dev`

---

## 40.59 — Dev 壳：角色切换 + 旧面板入口

**Files:**
- Create: `frontend/src/pages/dev/layout.tsx`
- Modify: `frontend/src/router.tsx`
- Modify: `frontend/src/components/role/RoleSwitcher.tsx`（仅 `/dev` 挂载）

- [ ] **Step 1:** `/dev` 挂载现有 `/panels/*` 或重定向
- [ ] **Step 2:** 四槽 key / 角色切换仅此处
- [ ] **Step 3:** Commit `feat(fe): /dev shell with role switcher`

## Done

- [ ] 产品壳不出现四槽 Key；开发调试仍可用
