# Task 29: RAG 查询缓存(L1 答案缓存 + L2 embedding 缓存)

> **决策记录(2026-08-02, Joe 拍板):** 两级缓存。L1 = 归一化 query → 完整响应(防同 query 爆刷,
> 合法重复零成本);L2 = 归一化 query → embedding 向量(防重复调 dashscope)。存储走 redis-stack
> (已入 local compose,TTL/淘汰/锁原生),不混 postgres cache_entries(那是 chat 管线语义)。
> 前置: Task 28 已落地真实 embedding(text-embedding-v3)。

## Subtask 29.01: 归一化 + key 设计 + redis 客户端封装

> 现状: `backend/modules/rag/` 无任何缓存;`backend/database/embeddings.py` 每次 embed_text 都调 API;
> redis 只有 performance_optimizer 的惰性连接模式可参考。

**方案:**
1. 新建 `backend/modules/rag/cache.py`:
   - `normalize(text) -> str`: **轻量无损归一化**——`unicodedata.normalize("NFKC")`(全角→半角)、
     lower()、折叠连续空白、strip。**不做同义词替换**(文档 chunk 是原文向量,改写 query 会错位检索空间)。
     归一化后文本同时作为 L1 key 输入和 L2 的 embed 输入,保持一致。
   - `get_redis()`: 惰性 redis 客户端(参考 `performance_optimizer.py` 的 `_ensure_redis` 模式),
     **redis 不可用时静默降级**(跳过缓存直接计算,绝不因缓存挂掉而 500)。
   - key 约定:
     - L1: `rag:a:{epoch}:{tenant}:{norm_hash}` → JSON `{answer, sources, created_at, cache_version}`
     - L2: `rag:e:{model}:{norm_hash}` → struct 打包的 768 float(~3KB,不用 JSON)
     - epoch: `rag:epoch:{tenant}`(整数,知识库写入时 INCR,整批失效,不用 KEYS/SCAN)
     - 单飞锁: `rag:lock:{key}`(SET NX EX 10s)
2. 配置项(config.env + config.env.example):
   ```
   RAG_CACHE_ENABLED=true
   RAG_CACHE_TTL_ANSWER=3600        # L1,滑动续期,上限 4h
   RAG_CACHE_TTL_EMBED=86400        # L2
   RAG_RATE_LIMIT_REQ=60            # 请求限流: 次/分钟/租户
   RAG_RATE_LIMIT_MISS=10           # miss 限流: 次/分钟/租户(防爆刷正主)
   ```

**修改文件:** `backend/modules/rag/cache.py`(新建), `config.env`, `config.env.example`

## Subtask 29.02: L2 embedding 缓存(放进 embed_text 内部,全局受益)

> 现状: embed_text 每次调用都打 dashscope;RAG query、memory 写入、chat 记忆检索共用它,同文本重复调用。

**方案:**
1. `backend/database/embeddings.py` 的 `embed_text()` 开头加 L2 查询:
   - key = `rag:e:{spec.name}:{normalize(text) 的 sha256 前 16 位}`
   - 命中 → 解包返回(仍走 `_pad_or_trim` 补零到 1536)
   - miss → 调 API → 成功后写入(struct 打包,TTL 86400,滑动续期)
   - redis 异常一律吞掉走原逻辑(哈希兜底链不变)
2. 注意: L2 是模型级全局缓存(同文本同向量,不依赖租户),key 带模型名自动版本化
   (换 text-embedding-v4 后旧 key 自然失效,无需清理)
3. 指标: `cache_hits/cache_misses` 加 `cache_type="rag_embed"` 标签
   (backend/core/metrics.py,参考现有 exact/template 用法)

**修改文件:** `backend/database/embeddings.py`, `backend/core/metrics.py`

## Subtask 29.03: L1 答案缓存(rag_service,含单飞/滑动 TTL/epoch 失效)

> 现状: `rag_service.ask()` 每次全链路(embed → 检索 → LLM);`/ask/context` 带会话历史,
> L1 不适用(答案依赖历史),只吃 L2。

**方案:**
1. `rag_service.ask()` 流程改为:
   ```
   norm = normalize(question)
   epoch = GET rag:epoch:{tenant}(缺省 0)
   key = rag:a:{epoch}:{tenant}:{sha256(norm)[:16]}
   L1 命中 → 审计(cache_hit=true) → 返回 JSON + "cache_hit": true(命中即续 TTL,上限 4h)
   miss → 限流检查(miss 桶) → 单飞锁 SET NX → 未获锁: 短轮询缓存(≤500ms),仍无则直接算
   → embed(L2 已挡重复) → 检索 → LLM → 写 L1(带 cache_version) → 审计(cache_hit=false, cost)
   ```
2. 响应加字段: `cache_hit: bool`;`GET /api/rag/status` 增加 `cache: {hit, miss, hit_ratio, l1_entries, l2_entries}`
3. 失效挂点: `rag_router.py` 的 `/upload`、`/upload/pdf`、`/init/sample`、`/init/knowledge-base`、`/reset`
   成功后 `INCR rag:epoch:{tenant}`(O(1),不扫前缀)
4. `/search`: 只走 L2 + 检索,不做 L1(检索结果缓存优先级低,留 v2.0)

**修改文件:** `backend/modules/rag/services/rag_service.py`, `backend/modules/rag/routers/rag_router.py`

## Subtask 29.04: RAG 限流 + 审计

> 现状: RAG 端点无限流(现有 TokenBucket 是进程内存,多 worker 不准);RAG ask 不写审计
> (chat 管线有 audit,`backend/core/audit.py:log_audit` 可复用)。

**方案:**
1. redis 滑动窗口限流(INCR + EXPIRE,多 worker 安全):
   - 请求桶: `rl:rag:req:{tenant}:{minute}` 阈值 RAG_RATE_LIMIT_REQ
   - miss 桶: `rl:rag:miss:{tenant}:{minute}` 阈值 RAG_RATE_LIMIT_MISS,在 L1 miss 后消费
   - 超限 → `ContextGateException("RATE_001", "rate_limited")`(与 chat 管线错误码一致)
   - 挂载: rag_router 依赖(全端点请求桶;/ask 额外 miss 桶)
2. 审计: `rag_service.ask()` 命中与 miss 都写 `log_audit`:
   `action="rag.ask", tenant_id, user_id, cache_hit, cost_usd, trace_id, question(norm 截断)`
   (命中 cost_usd=0——合规故事: 缓存命中零成本可溯源)

**修改文件:** `backend/modules/rag/cache.py`, `backend/modules/rag/routers/rag_router.py`, `backend/modules/rag/services/rag_service.py`

## Subtask 29.05: 测试(无真网络)

新建 `tests/test_rag_cache.py`(monkeypatch redis + openai + LLM,不碰真服务):
1. normalize: 全角→半角、大小写、连续空白折叠、中文不变
2. L2: 同文本两次 embed → 第二次不打 API(mock client 计数 == 1);不同模型 key 隔离
3. L1: ask 两次 → 第二次 LLM 调用计数 == 1 + 响应 cache_hit=true
4. epoch: upload 后旧 key 失效(同 query 重新计算)
5. 单飞: 并发同 query(threads)→ 仅一次 LLM 调用
6. 限流: 超 miss 阈值 → RATE_001
7. redis 不可用 → 静默降级,请求照常成功
8. PII 问题(含身份证号)→ 不写缓存(守卫跳过)

**修改文件:** `tests/test_rag_cache.py`(新建)

---

## 验证(自动化)

```bash
make verify && make check && uv run pytest    # 107+ 新增,ruff+mypy 全绿
```

## 验证(人工, [T29] — dev server + redis-stack 起着)

```bash
KEY=<user key>; BASE=localhost:8000/api/rag
# 同 query 爆 20 次
for i in $(seq 1 20); do curl -s $BASE/ask -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"question":"如何查询公司的信息安全管理制度"}' | python3 -c "import sys,json; d=json.load(sys.stdin)['data']; print(d.get('cache_hit'), round(d.get('latency_ms',0)))"; done
```
- 第 1 次 cache_hit=false,之后全 true;延迟显著下降
- LangFuse: 20 次里只有 1 个 embedding span + 1 个 LLM span
- `GET /api/rag/status` 的 cache.hit_ratio 上升;`redis-cli keys 'rag:*'` 可见 L1/L2 条目
- 爆刷: `seq 1 100 | xargs -P 10 -I{} curl ...` 快速打 → 触发 RATE_001(结构化错误码)
- 知识库变更: upload 一个文件后,同 query 立即 cache_hit=false(epoch 失效生效)
- PII: 问含身份证号的问题 → redis 无新 L1 条目

## 实现取舍记录(2026-08-02 code review 后补)

- **审计 cache_hit 无独立字段**: `audit_logs` 表无 cache_hit 列,实现以 `input_text` 前缀
  `cache_hit=1|` 溯源(见 rag_service._audit)。正式字段(加列迁移)留 v2.0。
- **RAG 端点认证**: 实现时为全部 9 个端点补了 `chat:write` 认证(原大部分裸奔,属安全改进);
  语义上 kb 端点更贴切的是 `kb:read/kb:write`,角色表内 user 均具备,功能无差异,权限细化留后续。

## 不在本 Task 范围(记录待决)

- 进程内 L1(LRU)兜底 redis——多 worker 脏读风险,等单进程压测有需要再做
- 同义词归一化词典(L1 专用)——维护成本高,先靠无损归一化
- top-k 检索结果缓存——L1/L2 落地后再看收益
- 每租户缓存配额——redis maxmemory + allkeys-lru 全局淘汰已够
- /ask/context 的 L1(答案依赖会话历史,key 需含历史 hash)——复杂度高,暂只吃 L2
