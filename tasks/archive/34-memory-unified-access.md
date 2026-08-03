# Task 34: Memory/context 统一存取层（V1.x · 原 32.63）

> **状态: 已完成(2026-08-03)。** 设计拍板见 `tasks/32-governance-deepening-roadmap.md` §32.63。
> **范围:** V1.x 结构债——补统一存取层，不合并四套职责；不引入 V2.0 治理新能力。
> **依赖:** Task 31✓、Task 30✓。

## 子任务

| # | 内容 | AC 摘要 | 状态 |
|---|------|---------|------|
| 34.01 | 引用审计 + 僵尸表代码停用（memory_items / user_profiles 仅 ORM 保留） | grep 无业务读写；附录 | ✓ |
| 34.02 | `backend/core/memory_service.py`：`write()` / `read()` 骨架 + hot/warm/cold 视图 | 单测 mock DB | ✓ |
| 34.03 | pipeline `write_memory` / `load_memory` 改走 UnifiedMemoryService | 行为不变回归 | ✓ |
| 34.04 | MemoryHub 改视图（持久化到底层表，禁纯内存真源） | agent 重启不丢冒烟 | ✓ |
| 34.05 | cold_memories 触发式摘要（规则起步）+ token 预算组装 | 单测；头+尾取样 | ✓ |
| 34.06 | 衰减迁入 + 删除/遗忘权入口 + system-role 隔离标记 | 护栏漂移检测接线 | ✓ |
| 34.07 | 文档 / README / 归档 | 本文件 | ✓ |

**不做本轮:** 向量 1536→768 全量重嵌；LLM 摘要默认开启。

## 交付要点

| 能力 | 落点 |
|------|------|
| 统一存取 | `backend/core/memory_service.py` → `UnifiedMemoryService` |
| 管线 | `load_memory` / `write_memory` / `build_context` |
| Agent 视图 | `PersistentScopedStore` → `user_memories` key=`hub:<scope>:<path>` |
| Cold 摘要 | 消息数达阈值且为整数倍 → 规则摘要；取样头 K+尾 K |
| 衰减 | `decay_score(0.9^days)`；warm 低权重过滤、cold 按权重取 |
| 遗忘权 | `DELETE /memory/users/{uid}/memories`（tenant_admin/super_admin + 调用方 tenant 作用域）；chat 脱敏 |
| 角色不漂 | `MEMORY_ISOLATION_HEADER` + `check_role_drift`；PromptComposer 数值收敛、emoji 可配 |

## 硬约束（Joe）

记忆拼 prompt 保持 system role；隔离标记 + `check_role_drift`；租户 role 只允许收敛。

## Important 拍板

- Task 35 1A/2A — 先做 34，缓存 35.04 后置
- Agent 调用链透传 `tenant_id` → MemoryHub
- 34.05：长会话摘要 **头+尾** 取样（非仅 ASC limit）
- 34.06：1A 单删不级联；2A 遗忘权鉴权；3B 风格数值夹紧、emoji 可读配置

## 附录 A: 34.01 审计（2026-08-03）

| 轨道 | 路径 | 处置 |
|------|------|------|
| 真源(热) | `chat_messages` | 统一层 write_turn / read；遗忘权脱敏不删 |
| 真源(温) | `user_memories` | 画像 key=`profile:v1`；含 hub 持久化 |
| 冷 | `cold_memories` | 34.05 触发写入 |
| Agent 视图 | MemoryHub | 34.04 持久化视图 |
| 旁路 | EnhancedMemoryManager | user_memories；decay 委托 core |
| 僵尸 | MemoryItem / UserProfileDB | 仅 ORM 保留 |
