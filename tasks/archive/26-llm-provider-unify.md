# Task 26: LLM 依赖路径统一 mock/replay 抽象 [EVID-08]

> **状态:✅ 完成(Cursor,方案 A)**
> **基线:main @ af46833;验收:make verify + make check + pytest 82 passed**
> **来源:docs/EVIDENCE_PACK.md EVID-08 — 内容计划「离线可复现」的硬前提。**
> **交付:** `backend/core/harness/llm_client.py`(`get_llm_client`);RAG / AgentCore / EvaluationEngine 改走工厂;`tests/test_llm_client_factory.py`

## 背景

**问题:** `/chat` 已走 LLM_PROVIDER=mock|record|replay|openai 抽象(离线确定性),但三条业务路径绕过它直接读 LLM_API_KEY:
- `RAGService.ask`(rag_service.py:211)— 「RAG 需要可用的 LLM,请在 config.env 中配置 LLM_API_KEY 与 LLM_BASE_URL」
- Agent 服务(agent/chat 返回「抱歉,我遇到了一些问题」)— 内部 LLM 调用失败被吞
- `EvaluationEngine.evaluate_response`(evaluation.py)— 「评估引擎未配置API_KEY」

后果:replay 模式下 RAG/Agent/评测全部不可用 → 证据包、内容截图、CI 离线演示全被卡。

## 26.01 方案(已拍板:方案 A)

**方案 A(拍板确认):统一 LLM 客户端工厂**
- 在 `backend/core/harness/` 加一个 `get_llm_client()` 工厂:按 `LLM_PROVIDER` 返回 mock/record/replay/openai 实现(复用 harness.py 现有 provider 逻辑,不新造)
- RAGService / AgentService / EvaluationEngine 三处把直接 LLM 调用替换为工厂调用
- 现有 `LLM_PROVIDER=replay` 的 fixture 体系自动覆盖三条路径(未命中降级 mock,与 /chat 一致)

**方案 B(最小):只接 mock 兜底**
- 仅在 LLM_PROVIDER=mock 时返回确定性响应,record/replay 不接
- 成本低,但 replay 确定性/回放能力对 RAG/Agent/评测仍缺失,内容截图依然不稳

**推荐 A。** 依据:工厂抽取是复用而非重构(harness 已有四种 provider),改动面可控;一次性解决「离线可复现」,是内容计划与 CI 的前提。**(2026-08-02 Joe 拍板:复用 harness,方案 A 确认。)**

## 26.02 实施(方案 A 落地后执行)

**修改文件:**
- `backend/core/harness/`(工厂,新建 `llm_client.py` 或并入现有 harness)
- `backend/modules/rag/services/rag_service.py`(ask/create_qa_chain 改走工厂)
- `backend/services/agent_service.py`(内部 LLM 调用改走工厂)
- `backend/evaluation/` 或 `backend/routers/evaluation.py` 引用的 EvaluationEngine(改走工厂)

**验证(核心标准,离线可复现):**
```bash
make run   # APP_ENV=dev, LLM_PROVIDER=replay,无 LLM_API_KEY
# RAG: 先 init/sample(依赖 25.03),再 ask → 200 确定性响应
# Agent: /agent/chat → 200 非「抱歉」
# Eval: /evaluation/evaluate → 200 有分数(非「未配置API_KEY」)
uv run pytest && make check
```

**不做:** 不引入 LangChain 之外的 LLM 框架;不改 /chat 主路径(已工作);不做 provider 热切换。

## 验收标准(Task 26 全部)

- [x] 26.01 方案拍板(A/B)
- [x] 26.02 三条路径离线可跑,确定性(同请求两次结果一致)
- [x] 证据包 §2 EVID-08 复测通过(pytest 工厂覆盖;live curl 可在 `make run` + replay 下复验)
- [x] `make verify` / `make check` / pytest 全绿

## Cursor 会踩的坑

1. **工厂放对层:** 放 `backend/core/harness/`,不要在三个业务模块里各自实现 mock 分支(会变成三份漂移)
2. **fixture 命名:** 沿用 (model + messages) 哈希命名,新路径首次 record 时落盘,之后 replay 零成本 — 证据包阶段我会先 record 一轮真实 RAG/Agent/Eval fixture,再切 replay
3. **EvaluationEngine 的「评估进度」日志:** 改工厂后保持原有日志/落库行为不变,别顺手重构
4. **Agent 的「抱歉」错误被吞:** 修好后应能区分真实错误与 mock 响应,别把错误吞掉的逻辑带到新工厂里
