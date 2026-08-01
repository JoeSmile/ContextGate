# Task 21: v1.2 — 企业级增强

> **状态:✅ 完成(Cursor),依赖 v1.1(20)完成**
> **基线:main @ 9a4b90b;验收:make verify + make check + pytest 全绿(audit_consistency 太重:逐模块起 uv 子进程 1min+,不纳入 subtask 验收,批次收尾由 Hermes 统一跑)**
> **每个 Subtask 完成后 git commit,Signed-off-by: Joe**

## 21.01 ModelRegistry — 多模型统一路由

> **现状:** 模型选择散落在 `backend/pipeline/nodes/model_router.py`(MODEL_CHEAP/GOOD/BEST env)+ `backend/modules/llm/harness.py`(LLM_BASE_URL 单端点)。OpenAI 兼容接口可配,但无模型注册表、无按租户/按场景的模型策略。

**方案:**
- `backend/core/model_registry.py`(新建):模型注册表,每条 = {name, provider, base_url, api_key_ref, capability(chat/embedding/vision), cost_per_1k, max_tokens, enabled}
- 数据源:env + `llm_api_keys` 表(复用 Task 18 的 KeyManager 加密取 key),env 示例见 config.env.example
- `model_router` 节点改为查注册表:意图 → 档位 → 模型选择(短路径 skill 不走模型、长路径按 COST 策略)
- 本地模型:OpenAI 兼容(vLLM/ollama)直接注册,`base_url=http://localhost:8001/v1` 这类
- **实现备注(Task 22.01):** 同 tier 取 `cost_per_1k` 最低(cheapest-in-tier)。tenant budget / 配额语义(硬限/软限、超额拒绝还是降级、超额审计留痕)移 v2.0,待产品决策,不在本版本赶。

**修改文件:** `backend/core/model_registry.py`(新建)、`backend/pipeline/nodes/model_router.py`、`backend/modules/llm/harness.py`、`config.py`、`config.env.example`
**验证:** 注册 2 个模型(deepseek + 本地 mock),`/chat` 长路径命中配置模型;`make check`

## 21.02 多模态提取管线(whisper + PaddleOCR)

> **现状:** RAG 只支持文本/PDF(`/api/rag/upload/pdf` + pypdf)。无音频/图片提取。Windows 迁移后(32G RAM/4070S 16G)可本地跑 whisper + PaddleOCR。

**方案:**
- `backend/modules/rag/extractors/`(新建):`audio.py`(faster-whisper 或 openai-whisper,分片转写 + 时间戳)、`image.py`(PaddleOCR 或 paddleocr,中文优先)
- 上传端点扩展:`/api/rag/upload` 支持 audio(wav/mp3/m4a)/image(png/jpg)MIME 校验(复用 `backend/core/file_sanitizer.py`)
- 提取结果入 knowledge_chunks(带 source_type 列区分 text/audio/image)
- 依赖做成 optional:pyproject `[project.optional-dependencies] multimodal = [...]`,未装则端点返回明确错误码(参考 errors.py 风格)

**修改文件:** `backend/modules/rag/extractors/`(新建)、`backend/modules/rag/routers/rag_router.py`、`backend/modules/rag/services/rag_service.py`、`pyproject.toml`、`alembic/versions/002_multimodal.py`(如加列)
**验证:** 上传 mp3 + 图片 → chunks 生成 → `/api/rag/ask` 命中;`make check`

## 21.03 Windows 部署支持

> **现状:** 脚本全部是 sh/bash/macOS 路径。用户迁移 Windows 32G/4070S(本机跑 vLLM 量化 7B + 多模态)。

**方案:**
- `scripts/setup_windows.ps1`:uv 安装、`uv sync`、Docker Desktop(postgres+pgvector)、config.env 生成引导
- `scripts/run_windows.ps1`(或复用 `uv run uvicorn` 一行):启动服务 + vLLM 本地端点说明
- 文档:`docs/WINDOWS_SETUP.md`(WSL2 推荐方案 vs 原生 PowerShell,GPU 直通说明)

**修改文件:** `scripts/setup_windows.ps1`(新建)、`docs/WINDOWS_SETUP.md`(新建)
**验证:** 文档走查;PowerShell 语法 `pwsh -NoProfile -File scripts/setup_windows.ps1`(本机无 Windows 则语法审查 + 用户实机验证)

## 21.04 A/B 测试框架(从 v2.0 提前)

> **现状:** `backend/routers/evaluation.py` 有 `compare-prompts`(同请求多 prompt 对比),无流量级 A/B。

**方案:**
- `ab_experiments` 表 + `backend/core/ab/`(新建):experiment(名称/分流比例/变体 A/B 配置)/assignment(用户-实验-变体,确定性 hash 分流)/event(曝光/转化)
- `/api/ab/*` 管理端点 + pipeline 内 `experiment_hook`(在 build_context 后按分流注入变体配置)
- 指标:LangFuse trace 加 experiment 标签,评估落库
- **实现备注(Task 22.04):** conversion 由管线 `conversion_hook` 在短/长路径自动落库(不再依赖手动 `/api/ab/events`);exposure 仍由 `experiment_hook` 写入。

**修改文件:** `backend/core/ab/`(新建)、`alembic/versions/003_ab_experiments.py`(新建)、`backend/routers/admin.py`、`backend/pipeline/graph.py`
**验证:** 100 次请求分流比例 ≈ 配置;变体 B 的 prompt 生效;`make check`

## 21.05 成本治理看板

> **现状:** `backend/core/cost_manager.py` 记 token/成本,无聚合查询端点,无看板。

**方案:**
- `backend/core/cost_manager.py` 加聚合:按租户/模型/时间窗口的 token+成本(按日/按小时)
- `GET /api/admin/cost-summary?tenant_id=&from_ts=&to_ts=&granularity=day|hour` 端点(admin 权限)。`from`/`to` 是 Python 保留字,故用 `*_ts`,时间戳语义明确。
- `examples/admin.html` 加成本 Tab(配合 20.06)

**修改文件:** `backend/core/cost_manager.py`、`backend/routers/admin.py`、`examples/admin.html`
**验证:** 造几笔调用 → 聚合数字正确;`make check`

## 21.06 LangFuse span 级深度可观测

> **现状:** `backend/observability/` 已有 @observe 装饰器 + trace 链路。缺:节点级 span 明细、输入输出采样、审计联动。

**方案:**
- pipeline 每个节点加 span(observability/decorators.py 扩展,或 LangGraph `astream_events` 捕获)
- 采样策略:长路径全量、短路径 10%(可配置)
- 审计联动:audit_logs 加 trace_id 关联(已有?确认后补)

**修改文件:** `backend/observability/`、`backend/pipeline/`、`backend/routers/audit.py`
**验证:** LangFuse UI(http://localhost:3001)看到节点级 span 树;`make check`

---

## 验收标准(v1.2 全部)

- [x] 21.01 模型注册表 + 按策略路由
- [x] 21.02 音频/图片提取入知识库
- [x] 21.03 Windows 安装文档 + 脚本
- [x] 21.04 A/B 分流 + 指标
- [x] 21.05 成本聚合端点 + 看板 Tab
- [x] 21.06 span 级可观测
- [x] `make verify` / `make check` / pytest 全绿（audit_consistency 批次收尾由 Hermes 跑,不阻塞 subtask）
