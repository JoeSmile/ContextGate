# ContextGate — 改造任务状态

> **v1.0 全部完成(2026-08-01)** · **v1.1/v1.2 完成** · **v1.2 follow-up 完成**
> **Task 30 阶段 1(30.01–30.28) 已归档(2026-08-03)** · 19 个历史 Task 见 `archive/`

## 活动任务

| 文件 | 内容 | 状态 |
|------|------|------|
| ~~30b~~ → [archive/30b-leaf-real-execution.md](archive/30b-leaf-real-execution.md) | 叶子能力真实执行(替换 stub) | **已完成(2026-08-03)** |
| [30/30.29-product-fe.md](30/30.29-product-fe.md) | 产品 FE 蓝图 | **不在本轮**(实测后启动) |
| ~~31~~ → [archive/31-agent-orphan-consolidation.md](archive/31-agent-orphan-consolidation.md) | Agent 孤儿副本收口（结构债 1a） | **已完成(2026-08-03)** |
| [32-governance-deepening-roadmap.md](32-governance-deepening-roadmap.md) | 治理做透改造总纲（V2.0 新能力） | **冻结待命** |
| ~~33~~ → [archive/33-chat-consolidation.md](archive/33-chat-consolidation.md) | Chat 旁路收口（V1.x 结构债） | **已完成(2026-08-03)** |
| [34-memory-unified-access.md](34-memory-unified-access.md) | Memory 统一存取层（原 32.63） | 待执行 |
| [35-cache-unified.md](35-cache-unified.md) | 缓存语义统一（原 32.64） | 执行中(35.01–35.03) |

## 执行顺序(2026-08-04 修订:V1.x 做扎实,冻结 V2.0 新能力)

> **范围决策(2026-08-04, Joe 拍板):** 先把 V1.x 做扎实,**不引入新功能**。
> 分界线: **修"代码已经烂了"的 = V1.x 做;造"客户要付费的" = V2.0 冻结。**
> - V1.x 做: Task 30 收尾✓ / 30b✓ / Task 31✓ / **Task 34 记忆 + Task 35 缓存**/
>   Task 33✓ / journeys 实测修 bug / EVID-03 fixture 重录
> - V2.0 冻结(设计已落盘 32 文件,不动工): 预算三级语义/两本账/审批放行、
>   合规 Excel/脱敏/留痕、命中率指标、护栏配置化、不出域、样板间、30.29 产品 FE

| 序 | 任务 | 依赖 | 说明 |
|----|------|------|------|
| ✓ | **Task 30 阶段 1**(30.01-30.28) | — | 已归档 → `archive/30-capability-hub-frontend.md` + `archive/30/` |
| ✓ | **Task 30b**(叶子真实执行) | — | 已归档 → `archive/30b-leaf-real-execution.md` |
| ✓ | **Task 31**(Agent 孤儿删除) | — | 已归档 → `archive/31-agent-orphan-consolidation.md` |
| 3 | **Task 34**(记忆统一层) | 31✓ | → `34-memory-unified-access.md` |
| 4 | **Task 35**(缓存统一) | 31✓ | → `35-cache-unified.md`（35.01–03 进行中） |
| ✓ | **Task 33**(Chat 旁路收口) | — | 已归档 → `archive/33-chat-consolidation.md` |
| 6 | **journeys 实测 + 修 bug** | 任意时 | 4 角色旅程;别扭点进 MANUAL_TEST §13 |
| 7 | **EVID-03 fixture 重录** | — | replay fixture 脏数据重录 |
| ⏸ | **Task 32 V2.0 部分(冻结待命)** | V1.x 全绿 + 证据包 | 预算/报表/护栏配置/命中率/不出域/样板间 |
| ⏸ | **Task 30.29 产品 FE** | 测试 FE 实测后 | 能力市场/工作台/治理中心 |

**并行规则:** 30b 与 31 可并行;记忆统一层严格在 31 后;缓存统一与记忆层可并行;journeys 实测随时可跑。

## 完成情况

| 批次 | 内容 | 完成提交(节选) |
|------|------|----------------|
| Batch 1–8 / Task 19–29 | 见历史 | `tasks/archive/` |
| **Task 30 阶段 1** | Capability Hub + 测试 FE(30.01–30.28) | `feat/task-30-capability-hub` |
| **Task 31** | Agent 孤儿副本收口 | `feat/task-30-capability-hub` |
| **Task 30b** | 叶子能力真实执行 | `feat/task-30-capability-hub` |
| **Task 33** | Chat 旁路收口 | `feat/task-30-capability-hub` |

## 遗留验收(全部满足)

- `make verify` — 品牌 10 词门禁全绿(含负向测试)
- `make check` — ruff + mypy
- `pytest` — 全绿(含 capability)
- `cd frontend && npm run test && npm run build`

## 历史文档

完成批次的详细计划、代码骨架、验收标准在 `tasks/archive/`,仅供追溯,不再执行。
Task 30 子任务: `tasks/archive/30/`。

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
