# Task 23: 补齐 v1.2 测试覆盖缺口

> **状态:待执行(Cursor)**
> **基线:feat/task-21-v1-2-enterprise @ bafd36c(或 merge 后的 main);验收:make verify + make check + pytest 全绿**
> **每个 Subtask 完成后 git commit,Signed-off-by: Joe**
> **背景:Task 21/22 Code Review Important #2(拍板 2A,2026-08-01)— CONTRIBUTING 要求新代码有测;22.02 已覆盖 registry / `_pick_variant` / cost_summary 参数拼装,下列模块仍无单测。**

## 23.01 多模态 extractors + file_sanitizer

> **现状:** `backend/modules/rag/extractors/audio.py` / `image.py` 与 `file_sanitizer` 扩展无测;依赖缺失路径靠手测。

**方案:**
- `tests/test_multimodal_extractors.py`:monkeypatch 掉 faster-whisper / paddleocr import → 断言 `MultimodalDependencyError` / `ErrorCode.RAG_DEP_MISSING`;文件不存在 → `FILE_NOT_FOUND`
- `tests/test_file_sanitizer.py`(或并入上文件):audio/image 扩展名 + MIME 校验通过/拒绝;超大文件 `FILE_001`

**修改文件:** `tests/test_multimodal_extractors.py`(新建)、可选 `tests/test_file_sanitizer.py`
**验证:** `uv run pytest tests/test_multimodal_extractors.py -q`;`make check`

## 23.02 LangFuse 路径采样

> **现状:** `backend/observability/sampling.py` 无测;短/长路径采样与幂等决策靠约定。

**方案:**
- `tests/test_sampling.py`:monkeypatch `random.random` + Settings 采样率 → 断言 `should_sample` 短路径 0/1 边界、长路径 rate=1 必真、同请求二次调用幂等(`_sample_decided`);`is_short_path` 集合断言;`reset_sampling_state` 后可再掷

**修改文件:** `tests/test_sampling.py`(新建)
**验证:** `uv run pytest tests/test_sampling.py -q`

## 23.03 A/B hooks + `/api/ab` 路由

> **现状:** `experiment_hook` / `conversion_hook` 与 `backend/routers/ab.py` 无测;conversion 静默失败与「无 experiment 不写」无断言。

**方案:**
- `tests/test_ab_hooks.py`:桩 `assign_variant` / `record_event` → experiment_hook 写入 `ab_*` 并调 exposure;conversion_hook 在无 id / 无 response / 有完整字段三种路径;record_event 抛错不抬异常
- `tests/test_ab_router.py`(FastAPI TestClient + 桩 auth/DB):创建实验、list、stats 形状;非法 groups/weights → AB_001

**修改文件:** `tests/test_ab_hooks.py`(新建)、`tests/test_ab_router.py`(新建)
**验证:** `uv run pytest tests/test_ab*.py -q`;`make check`

---

## 验收标准(Task 23 全部)

- [ ] 23.01 multimodal + sanitizer 测绿
- [ ] 23.02 sampling 测绿
- [ ] 23.03 hooks + ab router 测绿
- [ ] `make verify` / `make check` / pytest 全绿(计数相对 33 上升)

## Cursor 会踩的坑

1. **勿真装 paddleocr/whisper:** 用 import 失败或模块级 monkeypatch,CI 无 GPU/大依赖
2. **ab router 鉴权:** 复用 `tests/test_auth.py` 的 fixture 模式,勿绕过 `require_permission`
3. **conversion_hook 与 SSE:** 图内 stream_mode 早退时 response 为空是预期;SSE 补记已在 `pipeline/router.py`(bafd36c),hooks 单测只测节点本身即可
