# Tasks 队列

> **当前主队列:** Task **40** — 试点 B · **链 A**（人侧金线 → 7A）  
> **并行/旁路:** Task 39（管线早预处理，不挡 40）  
> **历史:** `tasks/archive/`  
> **设计事实源:** `docs/superpowers/specs/2026-08-05-enterprise-pilot-b-gaps-design.md`  
> **批次计划:** `docs/superpowers/plans/2026-08-05-pilot-b-master-plan.md`

## 当前状态

| ID | 标题 | 状态 |
|----|------|------|
| **40** | 试点 B · 链 A（JWT → OrgScope → Runner → 挂起 → `/app`/`/admin` → 7A） | **拆分中 / 待实现** |
| 39 | 管线早预处理 | 设计已拍板，待实现 |
| **41** | 记忆/提示词/缓存 一体设计（设计稿：LangFuse Prompt + langMem 优先，假名化/同权限缓存后置） | **设计已拍板，待实现** |
| 30.29 | 产品 FE | 见 `tasks/30/` |

**链 B（Wave 2B / 7B）:** 7A 验收前 **不拆**。

## 新任务怎么写

1. 在 `tasks/` 建 `NN-short-title.md`（或 `NN/` 子任务目录）。  
2. 文首写：**状态 / 依赖 / 非目标 / 挂接 J\***。  
3. 每个可独立验收的切片一节：Files · Steps（checkbox）· 验证命令 · Commit 提示。  
4. 改完更新本 README 状态表；归档时移入 `tasks/archive/`。  
5. Conventional Commits + `Signed-off-by: Joe`。  
6. 实现后按 `.cursor/rules/post-impl-code-review.mdc` 做 review；Important 等用户拍板。

## Task 40 入口

→ [`40-pilot-b-chain-a.md`](40-pilot-b-chain-a.md) · 子任务 [`40/`](40/)
