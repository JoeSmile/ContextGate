# ContextGate — 开发指引

> **开始改造前，先读 `tasks/batch-README.md` 了解批次计划。**
> **执行入口: `tasks/batch-*.md`，按编号顺序执行。**
> **详细参考: `tasks/00-*.md` ~ `tasks/17-*.md`，验收时对照。**

## 项目

**ContextGate** — The Intelligent Gateway for LLM Context Management.
企业级 LLM 前置处理管线，支持认证、多租户、安全护栏、可观测、模型路由、缓存。

- 包管理: `uv` (不用 pip)
- Python: 3.11+
- API 框架: FastAPI
- 管线引擎: LangGraph StateGraph
- 存储: PostgreSQL + pgvector
- 可观测: LangFuse
- 代码风格: Ruff + mypy

## 常用命令

```bash
uv sync              # 安装依赖
uv lock              # 锁定依赖
uv run python ...    # 运行脚本
uv run pytest ...    # 跑测试
uv run ruff check .  # lint
uv run mypy backend/ # 类型检查
```

## 架构

管线节点 (LangGraph DAG):
```
auth_check → load_memory → rate_limiter → cache_check
  ├─ hit → END
  └─ miss → guardrails_input → analyze_parallel → build_context
            → model_router
              ├─ short path → execute skill → END
              └─ long path → llm_generate → guardrails_output
                            → write_memory + audit → END
```

## 权限

4 种角色: super_admin(跨租户), auditor(跨租户只读审计), tenant_admin, user
认证: X-API-Key Header → SHA256 → api_keys 表
权限: `@require_permission("chat:write")` Depends

## 目录约定

- `backend/pipeline/nodes/` — LangGraph 节点，每节点一个文件
- `backend/core/auth/` — 认证 + 权限
- `backend/core/guardrails/` — 安全护栏
- `backend/skills/builtin/` — Skill 自动发现
- `backend/observability/` — LangFuse
- `tasks/` — 改造计划 Subtask

## 提交规范

Conventional Commits: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`
每个 commit 加 `Signed-off-by: Joe`
