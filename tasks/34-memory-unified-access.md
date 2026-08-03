# Task 34: Memory/context 统一存取层（V1.x · 原 32.63）

> **状态: 执行中(34.01–34.04✓)。** 设计拍板见 `tasks/32-governance-deepening-roadmap.md` §32.63（2026-08-03）。
> **范围:** V1.x 结构债——补统一存取层，不合并四套职责；不引入 V2.0 治理新能力。
> **依赖:** Task 31✓（agent 路径干净）、Task 30✓。

## 子任务

| # | 内容 | AC 摘要 | 状态 |
|---|------|---------|------|
| 34.01 | 引用审计 + 僵尸表代码停用（memory_items / user_profiles 仅 ORM 保留） | grep 无业务读写；附录 | ✓ |
| 34.02 | `backend/core/memory_service.py`：`write()` / `read()` 骨架 + hot/warm/cold 视图 | 单测 mock DB | ✓ |
| 34.03 | pipeline `write_memory` / `load_memory` 改走 MemoryService | 行为不变回归 | ✓ |
| 34.04 | MemoryHub 改视图（持久化到底层表，禁纯内存真源） | agent 重启不丢冒烟 | ✓ |
| 34.05 | cold_memories 触发式摘要（规则起步）+ token 预算组装 | 单测 | 待 |
| 34.06 | 衰减迁入 + 删除/遗忘权入口 + system-role 隔离标记 | 护栏漂移检测接线 | 待 |
| 34.07 | 文档 / README / 归档 | | 待 |

**不做本轮:** 向量 1536→768 全量重嵌（可单开子任务）；LLM 摘要默认开启。

## 硬约束（Joe）

记忆拼 prompt 保持 system role；隔离标记 + `check_role_drift`；租户 role 只允许收敛。

## 附录 A: 34.01 审计（2026-08-03）

| 轨道 | 路径 | 处置 |
|------|------|------|
| 真源(热) | `chat_messages` ← pipeline `write_memory` / `load_memory` | 保留；34.03 收敛到统一层 |
| 真源(温) | `user_memories` ← `services/memory_service.py`、`vector_ops`、`load_memory` | 保留；画像 key=`profile:v1` |
| 冷 | `cold_memories` ORM 在 `pgvector_session`，业务未接线 | 34.05 启用 |
| Agent 视图 | `backend/agent/memory_hub.py`（InMemory） | 34.04 改持久化视图 |
| 旁路 | `EnhancedMemoryManager` / `UserProfileBuilder`（仅 enhanced-chat） | 34.01 已改走 user_memories |
| 僵尸 | `MemoryItem` / `UserProfileDB`（`database/models.py`） | **仅 ORM 保留**；业务零读写 |

**Important 关联:** Task 35 拍板 1A/2A — 先做 34，缓存 35.04 后置。
