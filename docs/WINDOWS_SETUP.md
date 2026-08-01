# Windows 部署指南

面向本机开发 / 工作站（建议 32GB RAM；本地 vLLM 7B 量化需 NVIDIA GPU，如 4070S 16GB）。

## 方案对比

| 方案 | 适用 | 说明 |
|------|------|------|
| **WSL2 + Docker Desktop**（推荐） | 日常开发 | Linux 体验最佳；GPU 需在 Docker Desktop 开启 WSL2 集成 |
| **原生 PowerShell + Docker Desktop** | 不想进 WSL | 使用本仓库 `scripts/*.ps1`；路径与 macOS/`make` 命令等价（`uv` 统一） |
| **原生 + 远程 Postgres** | 无 Docker | 在 `config.env` 填 `DATABASE_URL` 指向已有库 |

## 快速开始（原生 PowerShell）

```powershell
# 1. 安装 uv（若尚未安装）
irm https://astral.sh/uv/install.ps1 | iex

# 2. 初始化依赖 + Postgres
pwsh -NoProfile -File scripts/setup_windows.ps1

# 3. 种子数据
uv run python scripts/seed_api_keys.py
uv run python scripts/seed_pgvector.py

# 4. 启动 API
pwsh -NoProfile -File scripts/run_windows.ps1
```

浏览器：`http://localhost:8000/docs` · Playground：`http://localhost:8000/playground/`

## GPU / 本地模型

1. 安装 NVIDIA 驱动 +（可选）CUDA。
2. 用 vLLM / Ollama 暴露 OpenAI 兼容口，例如 `http://127.0.0.1:8001/v1`。
3. 在 `config.env` 注册：

```env
MODEL_REGISTRY_JSON=[{"name":"local-7b","provider":"vllm","base_url":"http://127.0.0.1:8001/v1","tier":"best","cost_per_1k":0,"max_tokens":2048,"api_key_ref":""}]
```

## 多模态（Whisper + PaddleOCR）

```powershell
uv sync --extra multimodal
```

未安装时 `POST /api/rag/upload` 对音频/图片返回 `501 RAG_001`。

## WSL2 备注

在 WSL2 内可直接使用与 Linux/macOS 相同的 `make up` / `make run`。Docker Desktop 设置中勾选 **Use the WSL 2 based engine**，并为发行版开启 GPU 支持后再跑 vLLM。
