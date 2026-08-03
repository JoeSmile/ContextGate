# Task 35: 缓存语义统一（V1.x · 原 32.64）

> **状态: 执行中(35.01–35.03 已落地骨架)。** 设计拍板见 `tasks/32-governance-deepening-roadmap.md` §32.64（2026-08-03）。
> **范围:** 删 ICacheService 僵尸；抽 `redis_tools`；RAG 模式为模板；不引入 cachetools。
> **依赖:** Task 31✓。

## 子任务

| # | 内容 | 状态 |
|---|------|------|
| 35.01 | 删除 `ICacheService` + `CacheServiceFactory` + `get_cache_service` | ✓ |
| 35.02 | 新建 `backend/core/redis_tools.py`（sync+async 惰性连接、静默降级） | ✓ |
| 35.03 | RAG `cache.py` / PerformanceOptimizer 改用 redis_tools | ✓ |
| 35.04 | key 前缀规范文档 + CacheManager 单飞/epoch（可选增量） | **后置**(2026-08-03 拍板 2A：先 Task 34) |
| 35.05 | 单测：redis 挂降级、单飞、epoch | ✓（redis_tools + 既有 rag_cache） |
| 35.06 | 文档归档 | 待 |

## Important 拍板(2026-08-03)

1. **A** — `PerformanceOptimizer.close()` 只清实例引用，不关共享 async Redis
2. **A** — 先 Task 34；35.04 CacheManager 单飞/epoch 后置

## 验收

```bash
uv run ruff check backend/ scripts/
LLM_MOCK=true uv run pytest tests/ -q --tb=short
# 无 ICacheService 符号残留（业务代码）
```
