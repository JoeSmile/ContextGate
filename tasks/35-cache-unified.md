# Task 35: 缓存语义统一 + V1.x 收尾(V1.x 最后一站)

> **状态: 执行中 — 缓存统一(35.01–35.06)✓；收尾尾巴(35.07–35.09)待。**
> **设计拍板:** `tasks/32-governance-deepening-roadmap.md` §32.64；**规范文档:** [`docs/CACHE.md`](../docs/CACHE.md)。
> **范围:** ① 缓存统一(35.01-35.06)② **V1.x 收尾尾巴**(35.07-35.09)——journeys 实测 + EVID-03 + 归档。
> **依赖:** Task 31✓ / Task 34✓。
> **本文件是 V1.x 的最后一站:** 35 全绿 = V1.x 结构债清零,之后才能解锁 V2.0(32 冻结待命)。

## 一、缓存语义统一(原 32.64)

| # | 内容 | 状态 |
|---|------|------|
| 35.01 | 删除 `ICacheService` + `CacheServiceFactory` + `get_cache_service` | ✓ |
| 35.02 | 新建 `backend/core/redis_tools.py`(sync+async 惰性连接、静默降级) | ✓ |
| 35.03 | RAG `cache.py` / PerformanceOptimizer 改用 redis_tools | ✓ |
| 35.04 | key 前缀规范文档 + CacheManager 单飞/epoch(可选增量) | ✓ |
| 35.05 | 单测:redis 挂降级、单飞、epoch（含 CacheManager） | ✓ |
| 35.06 | 缓存统一文档/归档 | ✓ → `docs/CACHE.md`；整 Task 入 archive 留 35.09 |

### 交付要点(35.01–35.06)

| 能力 | 落点 |
|------|------|
| 公共 Redis | `backend/core/redis_tools.py` |
| RAG | `backend/modules/rag/cache.py`（L1/L2/单飞/epoch） |
| Chat | `CacheManager`（`chat:v` / `chat:epoch` / `chat:lock`） |
| 文档 | `docs/CACHE.md`；人工项 `MANUAL_TEST` §10.4–10.9 |
| 单测 | `tests/test_redis_tools.py` · `tests/test_cache_manager.py` · `tests/test_rag_cache.py` |

## 二、V1.x 收尾尾巴(新增 2026-08-04)

| # | 内容 | 状态 |
|---|------|------|
| 35.07 | **journeys 实测 + 修 bug**: 用测试 FE + 4 份角色旅程剧本(01-user / 02-tenant-admin / 03-auditor / 04-super-admin)实测,别扭点进 MANUAL_TEST §13,修 V1.x bug | 待 |
| 35.08 | **EVID-03 fixture 重录**: replay fixture 含历史测试残留,重录干净(RAG/Agent/Eval 各 record 一轮真实数据),demo 截图不穿帮 | 待 |
| 35.09 | 归档: 35 全绿 → 本文件入 archive,tasks/README 完成表更新,V2.0(32)解除冻结标记 | 待 |

## 验收(全绿才算 V1.x 完成)

```bash
uv run ruff check backend/ scripts/
LLM_MOCK=true uv run pytest tests/ -q --tb=short
cd frontend && npm run test && npm run build
# 无 ICacheService 符号残留(业务代码)  — 35.06 已核实
# journeys 4 份剧本实测通过,别扭点已修或入 §13
# EVID-03 重录完成,demo 截图干净
```

**Important(2026-08-03 35.04):** 1A 清缓存 `*` bump 全部租户 epoch；2B write_turn 不 bump（短 TTL + 遗忘/清缓存失效）。
