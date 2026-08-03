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

## 二、V1.x 收尾尾巴(新增 2026-08-04,2026-08-04 修订:实事求是,不造演示数据)

> **数据原则(2026-08-04, Joe 拍板):** 全部对接真实数据,**不需要 demo 数据**;
> 除非功能是 env 可控(如 LEAF_STUB_MODE / LLM_PROVIDER=replay)才允许降级。
> **实事求是: 没有就是没有,用提醒告诉 user,不塞假数据。**

| # | 内容 | 状态 |
|---|------|------|
| 35.07 | **journeys 实测 + 修 bug**(真实数据): 用测试 FE + 4 份角色旅程剧本(01-user / 02-tenant-admin / 03-auditor / 04-super-admin)实测;别扭点进 MANUAL_TEST §13,修 V1.x bug。**一律真实数据**: RAG 有真实文档就查真实文档;LLM 走真实调用(record/openai);**不预置 demo 数据、不造 fixture 美化输出**。缺失场景(如知识库为空、无历史会话)→ 界面给"空态提示/提醒",不是塞样例数据 | 待 |
| 35.08 | **EVID-03 fixture 策略修订**(原"重录干净 fixture"→ 改为实事求是): replay fixture 保留历史残留的**标注问题**修掉(如明确 fixture 是测试产物),但**不为了 demo 截图好看而重录美化数据**;真实数据由 35.07 的 record 采集,天然干净。env 可控降级(LLM_PROVIDER=replay / LEAF_STUB_MODE)仅作离线开发用,文档注明"replay 输出为录制回放,非真实响应" | 待 |
| 35.09 | 归档: 35 全绿 → 本文件入 archive,tasks/README 完成表更新,V2.0(32)解除冻结标记 | 待 |

## 验收(全绿才算 V1.x 完成)

```bash
uv run ruff check backend/ scripts/
LLM_MOCK=true uv run pytest tests/ -q --tb=short
cd frontend && npm run test && npm run build
# 无 ICacheService 符号残留(业务代码)  — 35.06 已核实
# journeys 4 份剧本用真实数据实测通过,别扭点已修或入 §13
# 无 demo 数据造假: 空场景以提示/空态呈现,不是塞样例
# replay/LEAF_STUB_MODE 仅 env 可控降级,文档注明"回放非真实"
```

**Important(2026-08-03 35.04):** 1A 清缓存 `*` bump 全部租户 epoch；2B write_turn 不 bump（短 TTL + 遗忘/清缓存失效）。
