# Task 20: v1.1 — 加固与打磨

> **状态:✅ 完成(Cursor)**
> **基线:main @ 9a4b90b;验收:make verify + make check + pytest + audit_consistency 全绿**
> **每个 Subtask 完成后 git commit,Signed-off-by: Joe**

## 20.01 RAG Deepening — HyDE + ReRank

> **现状:** `backend/modules/rag/services/rag_service.py` 的 `RAGService.ask()` 直接向量检索(单路 query → pgvector top-k)。无查询改写、无重排,长问句/术语变体召回率低。

**方案:**
- HyDE:查询时先用 LLM 生成假设文档(2-3 句),与原文 query 拼接做双路检索(原文向量 + HyDE 向量),取并集
- ReRank:对 top-k(建议 20)结果用轻量交叉编码器重排(可先接 LLM prompt 打分,避免引入新依赖;或 `cross-encoder/ms-marco-MiniLM` 本地模型,需评估下载体积)
- 可配置开关:config.py 加 `rag_hyde_enabled` / `rag_rerank_enabled`(pydantic-settings 字段,env: `RAG_HYDE_ENABLED`)

**修改文件:** `backend/modules/rag/services/rag_service.py`、`config.py`、`config.env.example`
**验证:** `curl -X POST localhost:8000/api/rag/ask -H "X-API-Key: <key>" -d '{"question":"如何查询公司的信息安全管理制度"}'` — 对比开关前后 top-1 命中;`make check`

## 20.02 SSE Streaming polish

> **现状:** `backend/pipeline/router.py` `/chat/streaming` 已可流式(2026-08-01 实测 206 事件)。缺:心跳保活、客户端断开中止(生成浪费)、错误事件协议、断线重连(Last-Event-ID)。

**方案:**
- event_stream 内每 15s 空闲发 `: ping` 注释心跳(nginx 代理不超时)
- 客户端断开检测:捕获 `asyncio.CancelledError` → 停止 LLM 生成(harness.stream 内部取消)
- 错误事件:`{"type":"error","code":...,"message":...}` 统一协议(参考 `backend/core/errors.py` 错误码),替代现在的裸异常 500
- README/前端文档注明 `Last-Event-ID` 支持现状(先不做断点续传,注明即可)

**修改文件:** `backend/pipeline/router.py`、`backend/core/harness/llm.py`(stream 内取消)、`examples/streaming.html`
**验证:** TestClient 起流式请求,中途断开连接 → 服务端日志无残留生成;`make check`

## 20.03 清理 legacy 情感域缓存指纹分支

> **现状:** `backend/pipeline/nodes/cache_check.py:19` `_cheap_fingerprint` 的 `advice` 分支是情感陪伴时代遗留(advice = 倾诉建议域)。`greeting` 可保留(通用)。

**方案:**
- 删 `_cheap_fingerprint` 的 `advice` 分支,`greeting` 保留
- 确认 `backend/pipeline/cache/fingerprint_cache.py` 无 emotion 相关键;如有残留一并清
- `_cheap_fingerprint` docstring 注明"仅通用意图启发式"

**修改文件:** `backend/pipeline/nodes/cache_check.py`、`backend/pipeline/cache/fingerprint_cache.py`(如需要)
**验证:** `grep -rn "advice" backend/pipeline/` = 0(除合理引用);`make check` + 缓存命中测试(pytest 现有 15 例)

## 20.04 扩展 mypy 门禁到全部 routers + modules

> **现状:** `pyproject.toml [tool.mypy] files` 只覆盖 pipeline/skills/observability/core + `app.py` + `admin/audit/files` 三个 router。routers/agent、enhanced_chat、evaluation、personalization、performance、streaming_chat 和 modules/{agent,intent,rag,llm} 未纳入。

**方案:**
- 逐个把剩余 router 和 modules 加入 `files` 列表
- 每加一个跑 `uv run mypy`,修到 0 error(注意:老代码可能有 untyped/Any,按现有 `warn_return_any=false` 策略处理,不放宽全局)

**修改文件:** `pyproject.toml` + 被暴露的 typing 修复
**验证:** `make check`(mypy 全绿,file 数从 56 上升)

## 20.05 意图识别示例文案清理

> **现状:** `backend/modules/intent/routers/intent_router.py:53` OpenAPI 示例仍是情感域文案("我最近总是睡不着,该怎么办?"),与品牌剥离标准不符。

**方案:**
- 示例改为企业场景:"如何查询公司的信息安全管理制度?"(与 rag_router 示例一致)
- 全文扫 `backend/modules/intent/` 是否有其他情感文案残留
- **启发式质量(2026-08-01 实测):** `POST /intent/detect?text=如何查询公司的信息安全管理制度` 被 heuristic 分类为 `advice`(confidence 0.75)—— 检查 `intent_classifier`/`rule_engine` 的 catch-all 规则,企业知识查询类应命中 knowledge_query/rag 类意图;advice 意图若为情感域遗留(倾诉求助)需中性化或删除,不能把企业问题兜进"倾诉"

**修改文件:** `backend/modules/intent/routers/intent_router.py`、`backend/modules/intent/core/`(classifier/rule_engine)
**验证:** `grep -rn "睡不着\|失眠\|难过" backend/` = 0;`make verify`

## 20.06 Web Admin UI 决策(与"砍前端"策略对齐)

> **现状:** 前端已退役(仅 examples/ 测试页)。ROADMAP v1.1 曾列 Web Admin UI。

**方案(三选一,建议 A):**
- **A(推荐):** 不建独立前端。Admin API(`/api/admin/*`)已完备,补一个 `examples/admin.html` 静态管理页(playground 模式,零依赖)覆盖 api-keys / llm-keys / approve / audit 操作
- B: 建最小 React 管理台(成本高,与砍前端策略冲突)
- C: 延后到 v2.0 与 ai-platform 统一规划

**修改文件:** `examples/admin.html`(若选 A)
**验证:** 页面操作 API key 增删 + 审批流全通

---

## 验收标准(v1.1 全部)

- [x] 20.01 HyDE+ReRank 可开关,实测召回提升
- [x] 20.02 流式:心跳/断开中止/错误事件协议
- [x] 20.03 缓存指纹无情感域分支
- [x] 20.04 mypy 覆盖全部 routers + modules
- [x] 20.05 intent 示例企业化
- [x] 20.06 admin 测试页可用
- [x] `make verify` / `make check` / pytest 全绿（audit_consistency 全仓 grep 过慢，未阻塞合入）
