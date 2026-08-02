# ContextGate — 改造任务状态

> **v1.0 全部完成(2026-08-01)** · **v1.1/v1.2 完成** · **v1.2 follow-up 完成**
> 19 个 Task · 89 个 Subtask · 8 个 Batch · 1 个独立批次(情绪子系统拆除),全部实现并验证通过。

## 活动任务

| 文件 | 内容 | 状态 |
|------|------|------|
| `tasks/25-evidence-fix-batch.md` | 证据包小修批:EVID 05 注入变体 / 06 PII 顺序 / 07 init-sample / 10 await / 12 admin key / 04 cache 降级 / 11 文档 + 回归单测 | 待执行 |
| `tasks/26-llm-provider-unify.md` | EVID-08:LLM 依赖路径(RAG ask/Agent/Eval)统一走 mock/record/replay 抽象 | ✅ 已拍板(方案 A)→ 待执行 |

## 完成情况

| 批次 | 内容 | 完成提交(节选) |
|------|------|----------------|
| Batch 1 | Rebranding + pgvector 迁移 | `1c2d9ac`, `e3b2548` |
| Batch 2 | 认证 + RBAC0 + 请求签名 | `1eee7d6` |
| Batch 3 | 多租户 + 审计 + 健康检查 | `11dffd9`, `28ad90f` |
| Batch 4 | LangGraph 管线(10 节点) | `59d5d16` |
| Batch 5a | 可观测 + 缓存 + 护栏 + 上传 + 断路器 | `fd3c496` |
| Batch 5b | 成本治理 + 模型路由 + Skill | `c58205f` |
| Batch 6 | 依赖锁定 + Seed + Mock | `5b43a25` |
| Batch 7 | Docker + CI/CD + 生产部署 | `d8bf458` |
| Batch 8 | Ownership + LLM Key 治理 | `116047f`, `070b16c` |
| Batch 10 | 情绪子系统拆除 | `5bfeadd` → `9d646ce`(8 commits) |
| Task 19 | 性能瓶颈消除 | `9d5f493` |
| Task 20 | v1.1 加固与打磨 | `feat/task-20-v1-1-hardening` |
| Task 21 | v1.2 企业级增强 | `feat/task-21-v1-2-enterprise` |
| Task 22 | v1.2 收尾(测/策略/conversion) | `feat/task-21-v1-2-enterprise` |
| Task 23 | v1.2 测试覆盖补齐 | `feat/task-21-v1-2-enterprise` |
| SSE 系列 | 04.11 / 07.07e / 09.04(从 Task 02 延期) | `backend/pipeline/router.py` `/chat/streaming` 已实测:200 + 206 个 SSE 事件(2026-08-01) |

## 遗留验收(全部满足)

- `make verify` — 品牌 10 词门禁全绿(含负向测试)
- `make check` — ruff + mypy,102 files
- `pytest` — 70 passed
- `scripts/verify_schema.py` — 24 张表与 ORM 一致
- `scripts/audit_consistency.py` — 7 维度全绿

## 历史文档

完成批次的详细计划、代码骨架、验收标准在 `tasks/archive/`,仅供追溯,不再执行。

## 新任务怎么写

新工作按以下格式在 `tasks/` 下新建编号文件(如 `20-xxx.md`):

```markdown
# Task 20: 标题
## Subtask 20.01: 标题
> 现状: ...
**方案:** ...
**修改文件:** ...
**验证:** ...
```

完成后把文件移入 `tasks/archive/` 并更新本 README 的完成表。
