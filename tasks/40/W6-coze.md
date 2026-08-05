# Task 40 · Wave 6 — Coze 导入（链 A 加测，不挡 7A）

> **状态:** 待实现 · **可与 W5 交错 · 不阻塞 W7A**  
> **依赖:** Wave 3 Runner（同一 IR）  
> **挂接:** J2.1–J2.3；D3 整单拒绝；D9  
> **优先级:** 7A 后第一批也可；若人力够可提前加测

---

## 40.60 — Coze 导出物解析器

**Files:**
- Create: `backend/core/workflow/coze_import.py`
- Test: `tests/test_coze_import.py` + fixtures `tests/fixtures/coze/*.json`

- [ ] **Step 1:** 支持的最小节点集写死；不支持 → **整单拒绝**（D3），错误列表可读
- [ ] **Step 2:** 映射到 `WorkflowIR`
- [ ] **Step 3:** Commit `feat(workflow): Coze export parser with hard fail`

---

## 40.61 — 导入落权校验

**Files:**
- Modify: coze_import / workflows service
- Test: `tests/test_coze_acl.py`

- [ ] **Step 1:** 无权 capability → 标红不可发布（可存 `validation_errors`）
- [ ] **Step 2:** Commit `feat(workflow): Coze import ACL validation`

---

## 40.62 — 导入 API + Admin UI 入口

**Files:**
- Modify: `backend/routers/workflows.py` — `POST /api/workflows/import/coze`
- Create: `frontend/src/pages/admin/CozeImport.tsx`

- [ ] **Step 1:** API 测
- [ ] **Step 2:** Admin「导入 Coze」面板
- [ ] **Step 3:** Commit `feat(fe): Coze import panel`

---

## 40.63 — 导入流按钮执行 = 同源 Runner

**Files:**
- Test: `tests/test_coze_run_same_engine.py`

- [ ] **Step 1:** 导入成功 → publish → run → 与自研链同一 `workflow_runs` 历史
- [ ] **Step 2:** Commit `test(workflow): Coze-imported run shares runner history`

---

## Wave 6 Done 标准

- [ ] J2.1–2.3 可证伪  
- [ ] **未完成也不阻挡** 勾选 W7A（父任务索引已标注）
