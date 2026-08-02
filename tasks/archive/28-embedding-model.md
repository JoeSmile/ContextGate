# Task 28: 真实 Embedding 接入(text-embedding-v3,768 维)

> **状态:✅ 完成(Cursor)**
> **验收:make verify + make check + pytest 105 passed;人工 §6 [T28] 需起服后实测**
> **决策记录(2026-08-02, Joe 已拍板):** 用 DashScope `text-embedding-v3` + `dimensions=768`。
> 依据: QWEN_API_KEY 现成、费用按 token 计费(维度不影响价格)、768 是中文检索甜点(MTEB 768 vs 1024 差 <1pt)、
> 存储列是 Vector(1536) + `embed_text` 自动补零 → 768/1024 都不用改 schema(补零对 cosine 零影响)。
> 目的: 人工 QA §6 RAG 需要真实语义检索,当前配置指向 DeepSeek(无 /embeddings 端点),检索要么 404 要么哈希兜底(无语义),6.3/6.4/6.5/6.6 无法产出验收证据。

## Subtask 28.01: embedding 端点解析(registry 优先 + env 兜底)

> 现状: `backend/database/embeddings.py:32-53` 的 `embed_text()` 读 `LLM_API_KEY` + `LLM_BASE_URL`(=DeepSeek),
> 即使用 `EMBEDDING_MODEL` env 也只会拿 deepseek key 打 dashscope → 401;失败回退哈希向量。
> 另外 `backend/core/model_registry.py:17` 的 `ModelSpec.capability` 已预留 `embedding` 取值但从未被使用。

**方案:** 让 embedding 走模型注册表治理,与 Task 21/26 的 spec-aware 模式一致:

1. `backend/core/model_registry.py` 新增 `select_embedding_model() -> ModelSpec`:
   - 从 registry 里筛 `capability == "embedding" and enabled`,同档取 `cost_per_1k` 最低
   - 无注册项时兜底构造默认 spec: `name = os.getenv("EMBEDDING_MODEL", "text-embedding-v3")`,
     `base_url = os.getenv("EMBEDDING_BASE_URL", "https://dashscope.aliyuncs.com/compatible-mode/v1")`,
     `api_key_ref = "QWEN_API_KEY"`, `capability = "embedding"`
2. `backend/database/embeddings.py` 的 `embed_text()` 改为:
   - `spec = select_embedding_model()`
   - `api_key = os.getenv("EMBEDDING_API_KEY") or os.getenv(spec.api_key_ref) or os.getenv("QWEN_API_KEY") or os.getenv("LLM_API_KEY")`
   - `base_url = spec.base_url or os.getenv("EMBEDDING_BASE_URL") or os.getenv("LLM_BASE_URL")`
   - `dims = int(os.getenv("EMBEDDING_DIMENSIONS", "768"))`
   - 调用 `client.embeddings.create(model=spec.name, input=text[:8000], dimensions=dims)`;若 API 报 dimensions
     参数不支持(将来本地 vLLM 等),去掉 `dimensions` 重试一次;仍失败才走哈希兜底(保留现有 `_hash_embed`)
   - 输出保持补零/截断到 `EMBED_DIM = 1536`(存储列维度,不动)

**修改文件:** `backend/core/model_registry.py`, `backend/database/embeddings.py`

## Subtask 28.02: 清理误导性死代码 + 默认值更新

> 现状: `backend/modules/rag/core/knowledge_base.py:51-68` 初始化 LangChain `OpenAIEmbeddings`
> (硬编码 `text-embedding-v1`,base_url 指向 deepseek),但实际存储/检索全部走 `vector_ops.add_knowledge/search_knowledge`
> → `embed_text()`,`self.embeddings` 赋值后从未被读取(死配置,且文案误导: 让人以为用了真实 embedding)。
> `backend/core/config.py:84` 与 `backend/modules/rag/models/rag_models.py:117` 默认 `text-embedding-v1`(已下线型号)。

**方案:**
1. 删除 `knowledge_base.py:51-68` 的 OpenAIEmbeddings 初始化块及其导入(`langchain_compat.py` 引用一并清理,
   先 `grep -n "self.embeddings" backend/modules/rag/core/knowledge_base.py` 确认无读取点再删)
2. `get_stats()` 的 `"embedding_model"` 从固定字符串 `"api-or-hash"` 改为动态值:
   `select_embedding_model().name + ("(hash)" if 未配 key 走哈希 else "")` — 便于 QA 确认实际生效模型
3. 默认值 `text-embedding-v1` → `text-embedding-v3`(config.py:84, rag_models.py:117)

**修改文件:** `backend/modules/rag/core/knowledge_base.py`, `backend/core/config.py`, `backend/modules/rag/models/rag_models.py`, `backend/modules/rag/core/langchain_compat.py`(按清理结果)

## Subtask 28.03: 配置与文档

**方案:**
1. `config.env` 新增(嵌在 QWEN 相关区):
   ```
   EMBEDDING_MODEL=text-embedding-v3
   EMBEDDING_DIMENSIONS=768
   EMBEDDING_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
   # EMBEDDING_API_KEY=  # 可选;不设则用 QWEN_API_KEY
   ```
2. `config.env` 的 MODEL_REGISTRY_JSON 增加 embedding 条目(追加到**第一行** qwen-turbo 那个 JSON 数组里):
   `{"name":"text-embedding-v3","provider":"qwen","base_url":"https://dashscope.aliyuncs.com/compatible-mode/v1","api_key_ref":"QWEN_API_KEY","capability":"embedding","tier":"cheap","cost_per_1k":0.0001,"max_tokens":0}`
3. **顺带修 Minor 缺陷:** config.env 里 MODEL_REGISTRY_JSON 有两行(qwen-turbo 行 + local-7b 行)。
   dotenv 加载 override=False → **第一行生效,local-7b 行是死的**。合并为一行(两个条目都在数组里),
   或删掉死行。改前先 `git show HEAD:config.env | grep MODEL_REGISTRY_JSON` 核对原文。
4. `config.env.example` 同步新键(带注释说明用途与 key 来源)
5. `docs/MANUAL_TEST.md`:
   - §0 env 检查清单加三行(EMBEDDING_MODEL / EMBEDDING_DIMENSIONS / EMBEDDING_BASE_URL,期望值如上)
   - §6 加 [T28] 标记: 6.1-6.4 前置条件改为"Task 28 落地后为真实语义检索"
   - §13 缺陷记录表加一行: GAP-02 `MODEL_REGISTRY_JSON` 双行,local-7b 死行(Minor,本任务内修)

**修改文件:** `config.env`, `config.env.example`, `docs/MANUAL_TEST.md`

## Subtask 28.04: 测试(不许碰真网络)

> 现有测试无一直接覆盖 embedding 路径(`grep -rln "embed_text" tests backend/tests` 为空),无冲突。

**方案:** 新增 `tests/test_embedding_config.py`:
1. `select_embedding_model()`: monkeypatch `MODEL_REGISTRY_JSON` 含 embedding 条目 → 返回该 spec
   (capability/name/base_url/api_key_ref 断言);无条目 → 返回 env 兜底默认 spec(text-embedding-v3 / dashscope / QWEN_API_KEY)
2. `embed_text()` 真实调用路径: monkeypatch openai 客户端 → 断言传入 `model=text-embedding-v3`、`dimensions=768`,
   输出长度 == 1536(768 补零);dimensions 报错 → 无 dimensions 重试
3. 哈希兜底: 清空 EMBEDDING_*/QWEN_API_KEY/LLM_API_KEY/LLM_BASE_URL → 输出 1536 维、同文同向量(确定性)
4. 所有用例 monkeypatch,不发起真实 HTTP

**修改文件:** `tests/test_embedding_config.py`(新建)

---

## 验证(自动化,实现后必跑)

```bash
make verify && make check && uv run pytest        # 96+ passed,ruff+mypy 全绿
```

## 验证(人工, [T28] — 需 dev server 起后跑,证据留档)

```bash
KEY=<seed_key>; BASE=localhost:8000/api/rag
curl -s -X POST $BASE/init/sample -H "X-API-Key: $KEY"                 # 6.1 chunks > 0
curl -s -X POST $BASE/upload/pdf -H "X-API-Key: $KEY" -F "file=@docs/COMPLIANCE.md"  # 6.2
curl -s "$BASE/search" -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"query":"信息安全管理制度","top_k":5}' | python3 -m json.tool    # 6.3 相关性真实
curl -s $BASE/ask -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"question":"如何查询公司的信息安全管理制度"}' | python3 -m json.tool    # 6.4 有引用
```

- `GET /api/rag/status` 的 embedding_model 字段显示 `text-embedding-v3`(非 api-or-hash)
- psql 佐证维度: `SELECT vector_dims(embedding) FROM knowledge_chunks LIMIT 1;` → 1536
- 语义抽查: 问「信息安全管理制度」top-1 必须是 COMPLIANCE.md 相关 chunk(哈希向量做不到,此条是硬验收)
- 6.5/6.6 HyDE / ReRank 开关对比数据留档(`RAG_HYDE_ENABLED` / `RAG_RERANK_ENABLED` 各跑一轮 top-1 记录)

> 注意: embedding 不走 harness(与 chat 路径不同),QA 实测时会真实调用 dashscope,
> 费用约 0.0005 元/千 token,可忽略;测试代码必须 monkeypatch 禁止真网络。

## 不在本 Task 范围(记录待决)

- embedding 是否纳入 harness(mock/record/replay)——EVID-08 教训覆盖的是 chat 路径;embedding 纳入需先定
  record/replay 的 fixture 格式,单独决策
- 本地 embedding 模型(bge 系列)——等 Windows 4070S + vLLM 落地,届时 vLLM /v1/embeddings 是 OpenAI 兼容格式,本 Task 路径原样复用
- 向量列改 768 省存储——收益太小,留 v2.0
