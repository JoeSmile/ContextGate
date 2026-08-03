# ContextGate — 开发指引

> **任务状态:见 `tasks/README.md`。** Task 30 阶段 1 / 30b / 31 已归档；当前 V1.x 队列见 README。
> **历史批次与验收标准:`tasks/archive/`(仅供追溯)。**
> **新任务:`tasks/README.md` 的"新任务怎么写"。**

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
uv run ruff check backend/ scripts/  # lint（与 CI 一致）
uv run mypy          # 类型检查（pyproject files= 主路径门禁）
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
- `backend/core/memory_service.py` — 统一记忆存取（hot/warm/cold；Task 34）
- `backend/core/guardrails/` — 安全护栏
- `backend/skills/builtin/` — Skill 自动发现
- `backend/observability/` — LangFuse
- `tasks/` — 改造计划 Subtask

## 提交规范

Conventional Commits: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`
每个 commit 加 `Signed-off-by: Joe`

## 实现后 Code Review 工作流

详见 `.cursor/rules/post-impl-code-review.mdc`：

1. 做完代码 → 跑计划内验证 → **立刻 code review**
2. **Important** 错误：列出来给用户审核，不擅自改
3. Critical / Minor：直接改正
4. Important 未决前，不进入下一个 Batch
