# 缓存语义统一（Task 35 / 原 32.64）

> **状态: 代码侧完成(35.01–35.06, 2026-08-03)。** V1.x 收尾(journeys / EVID-03)见 `tasks/35-cache-unified.md` §二。

## 目标态

| 之前 | 之后 |
|------|------|
| 僵尸 `ICacheService` / Factory（无实现） | **已删除**（业务代码零残留） |
| RAG / PerformanceOptimizer 各自连 redis | 统一 `backend/core/redis_tools.py`（sync + async，惰性连接，静默降级） |
| CacheManager 裸 get/set | RAG 模板：单飞锁 + epoch + 滑动 TTL |
| key 命名分散 | 域前缀规范（下表 + `cache_key()`） |

不引入 cachetools/cachelib（进程内 LRU 非本问题；淘汰交 redis maxmemory）。

## 模块落点

| 模块 | 路径 |
|------|------|
| 公共客户端 | `backend/core/redis_tools.py` |
| RAG L1/L2 | `backend/modules/rag/cache.py` |
| Chat / 优化器 | `backend/services/performance_optimizer.py` → `CacheManager` |
| Key 规范 | 本文档下列表；辅助 `redis_tools.cache_key` / `CACHE_KEY_DOMAINS` |

## Key 规范

统一形状：`<域>:<名>:<租户>:<键>`。

| 域 | 用途 | 典型 key |
|----|------|----------|
| `rag` | RAG L1 答案 / L2 embedding | `rag:a:{epoch}:{tid}:{qhash}` · `rag:e:{model}:{thash}` · `rag:epoch:{tid}` · `rag:lock:*` |
| `chat` | 对话 / PerformanceOptimizer | `chat:v:{epoch}:{tid}:{logical}` · `chat:epoch:{tid}` · `chat:lock:{digest}` |
| `ctx` | 能力/上下文缓存 | 预留 `ctx:*` |
| `rl` | 限流 | `rl:cap:*` · `rl:rag:req:{tid}:{minute}` |
| `mem` | 记忆热缓存 | 预留 `mem:*`（真源 PG，见 Task 34） |

## 契约

1. **静默降级**：redis 不可用 → miss / 跳过写，业务不得 500。
2. **单飞**：`SET key NX EX`；未抢到锁则短等再读，超时可重复回源。
3. **epoch 失效**：写路径 `INCR <域>:epoch:<tid>`，读路径把 epoch 编进 value key；旧键靠 TTL 自然过期。
4. **滑动 TTL**：命中时 `EXPIRE` 续期（RAG L1 另有最大年龄封顶）。

## 失效入口

| 事件 | 动作 |
|------|------|
| RAG 上传/重置知识 | `bump_epoch(tenant)`（`backend.modules.rag.cache`） |
| 用户遗忘权 | `CacheManager.bump_epoch(tenant)`（同租户 chat） |
| 运维清缓存 | `POST /performance/cache/clear`（`*` → SCAN 并 bump **全部** chat 租户 epoch） |
| 普通 `write_turn` | **不** bump（短 TTL 足够；拍板 2B） |

## 验证

```bash
uv run ruff check backend/ scripts/
LLM_MOCK=true uv run pytest tests/test_redis_tools.py tests/test_cache_manager.py tests/test_rag_cache.py -q
# 业务代码无 ICacheService / CacheServiceFactory / get_cache_service
```

人工：`docs/MANUAL_TEST.md` §10.4–10.9；RAG 专项 §6.8–6.16。

## Important 拍板摘要

- close() 只清实例引用，共享客户端由 `redis_tools` 管理
- 35.04 先于 journeys；1A 清缓存 bump 全租户；2B write_turn 不 bump
