# Task 36: Redis 共享客户端生命周期修复(V1.x · 35 收尾发现)

> **状态: 已完成(2026-08-03)。** 来源: Task 35 代码审查(redis_tools 生命周期缺陷)。
> **范围:** 修 redis_tools 共享客户端的关闭/恢复问题;不引入新功能。
> **交付:** `0485a00` — close shared Redis clients and retry after TTL。

## 子任务

| # | 内容 | 状态 |
|---|------|------|
| 36.01 | lifespan shutdown 调 `close_async_redis()` | ✓ |
| 36.02 | `close_sync_redis()` 对称入口 | ✓ |
| 36.03 | 失败标志 TTL（`RETRY_AFTER_SEC=30`）+ 单测 | ✓ |

## AC

- [x] `close_async_redis` 接线 `backend/app.py`
- [x] `close_sync_redis()` 存在
- [x] 单测：降级 + TTL 后重试（`tests/test_redis_tools.py`）
- [x] `LLM_MOCK=true uv run pytest tests/ -q` 全绿
- [x] 已 commit

## 落点

- `backend/core/redis_tools.py` — TTL 失败闩、`close_sync_redis`、`close_async_redis` 清失败标志
- `backend/app.py` — shutdown 关共享 async 池
