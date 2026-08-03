# Redis 缓存 Key 规范（Task 35.04 / 32.64）

统一形状：`<域>:<名>:<租户>:<键>`（实现见 `backend.core.redis_tools.cache_key`）。

| 域 | 用途 | 典型 key |
|----|------|----------|
| `rag` | RAG L1 答案 / L2 embedding | `rag:a:{epoch}:{tid}:{qhash}` · `rag:e:{model}:{thash}` · `rag:epoch:{tid}` · `rag:lock:*` |
| `chat` | 对话 / PerformanceOptimizer | `chat:v:{epoch}:{tid}:{logical}` · `chat:epoch:{tid}` · `chat:lock:{digest}` |
| `ctx` | 能力/上下文缓存 | 预留 `ctx:*` |
| `rl` | 限流 | `rl:cap:*` · `rl:rag:req:{tid}:{minute}` |
| `mem` | 记忆热缓存 | 预留 `mem:*`（持久化真源仍为 PG，见 Task 34） |

## 契约

1. **静默降级**：redis 不可用 → miss / 跳过写，业务不得 500。
2. **单飞**：`SET key NX EX`；未抢到锁则短等再读，超时可重复回源。
3. **epoch 失效**：写路径 `INCR <域>:epoch:<tid>`，读路径把 epoch 编进 value key；旧键靠 TTL 自然过期。
4. **滑动 TTL**：命中时 `EXPIRE` 续期（RAG L1 另有最大年龄封顶）。

## 失效入口

| 事件 | 动作 |
|------|------|
| RAG 上传/重置知识 | `bump_epoch(tenant)`（`backend.modules.rag.cache`） |
| 用户遗忘权 / 清 chat 缓存 | `CacheManager.bump_epoch(tenant)` |
| 运维清缓存 | `POST /performance/cache/clear`（`*` → SCAN 并 bump **全部** `chat:epoch`/`chat:v` 租户） |
