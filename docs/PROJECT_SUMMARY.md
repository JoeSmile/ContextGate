# NexusAI 项目全景总结(有什么 · 要做什么)

> 更新:2026-08-05。用途:个人对项目的完整认知地图——不给人跑,给自己看。
> 一句话:企业 AI 中台——统一入口调企业 AI 能力,能力由 IT 编排,数据接 OA/RAG,治理平台兜底。

---

## 1. 技术栈

| 层 | 选型 |
|----|------|
| 语言/包管理 | Python 3.11 · uv |
| API 框架 | FastAPI + SQLAlchemy |
| 管线引擎 | LangGraph StateGraph(自封装 langgraph_compat) |
| 存储 | PostgreSQL + pgvector · Redis(限流/预算,静默降级) |
| 可观测 | LangFuse + audit_logs + Prometheus metrics |
| 前端 | React 19 + Vite + TS + Zustand + Tailwind(测试 FE) |
| 部署 | Docker Compose(本地/prod)· Dockerfile · GitHub Actions |

## 2. 有什么(完整模块清单)

### 2.1 认证与账号(backend/core/auth/ + routers/auth.py)
- `api_key_auth.py` — X-API-Key 认证:SHA256 → api_keys 表 → TenantContext(tenant/user/role/permissions)
- `signature_auth.py` — HMAC-SHA256 请求签名 + 防重放(±5min,nonce 去重),AWS SigV4 思路
- `password.py` — bcrypt(cost=12)
- `permissions.py` / `scope.py` / `models.py` — RBAC 4 角色 + 能力级动态权限
- `routers/auth.py` — Task 38 账号体系:register/login、限流 5 次/5min(redis 桶,降级内存)、登录轮换 key、审计联动

### 2.2 管线(backend/pipeline/)
- 13 节点 LangGraph DAG:auth_check→load_memory→rate_limiter→cache_check→guardrails_input→analyze_parallel→build_context→experiment_hook→model_router→llm_generate→guardrails_output→write_memory→conversion_hook
- 3 分支:cache 命中→END / 护栏拦截→END / model_router 双路径(skill 直连 or LLM)
- `state.py` — PipelineState 裸 dict;`graph.py` — 图定义(分支全在图里);`router.py` — 路由适配

### 2.3 记忆(backend/core/memory_service.py)
- 三层:hot(chat_messages 最近 5 条)/ warm(user_memories 画像 kv)/ cold(cold_memories 会话摘要,pgvector)
- 超阈值自动归档摘要(maybe_cold_summarize)

### 2.4 能力中枢(backend/core/capability/)
- `registry.py` — 能力注册表(目录 + 权限绑定 + 租户可见性 + resolve_credential)
- `invoke.py` — 统一 invoke(短路径 JSON / 长路径 SSE),权限闸门
- `agents.py` — Agent 门面(30.24):AgentSpec + 递归调用(深度 3),vendor-risk→contract-query→rag-ask 链
- `connectors/external_app.py` — Dify/Coze 连接器(SSE 透传 + 断路器 + 成本估算 + mock fixture)
- `governance.py` — 能力治理

### 2.5 RAG(backend/modules/rag/)
- pgvector 语义检索;上传(文件硬化)/ 分块 / 问答 / 搜索 / 状态;/api/rag 全套

### 2.6 模型层与 harness(backend/core/ + modules/llm/)
- `harness/` — LLM_PROVIDER(mock/record/replay/openai)核心抽象,禁绕过
- `key_manager.py` — AES-256-GCM 加密 llm_api_keys(LLM_KEY_MASTER_KEY)
- `key_repository.py` — 按租户+provider 查询、LRU、失败冷却
- `key_failover.py` / `key_health.py` — failover + 周期健康检查(Task 27)
- `model_registry.py` — 模型注册、意图→模型选择

### 2.7 治理层
- `cost_manager.py` + Task 32 设计 — 三窗口限额(单次/日/月)、软限告警、硬限+审批两本账
- `guardrails/` — 输入(注入检测/PII 脱敏)、输出(密钥/角色漂移拦截)
- `audit.py` — 全量审计 + 合规导出(脱敏)
- `circuit_breaker.py` — LLM/上游断路器
- `rate_limiter.py` — 令牌桶
- `file_sanitizer.py` — 上传 MIME + UUID 改名

### 2.8 可观测与基建(backend/core/ + observability/)
- LangFuse trace(@observe 装饰器)、metrics、health、ab(A/B 实验)
- `redis_tools.py` — 共享 Redis(静默降级)、`tenant.py`、`factories.py`、`fallback.py`

### 2.9 前端(frontend/,测试 FE)
- 8 面板:chat / admin / audit / agent / capabilities / eval / rag / performance
- 角色切换器(4 槽位)、密码/Key 双登录(Task 38)、apiFetch 统一 401/403 处理

### 2.10 工程与数据
- alembic 6 个迁移;seed(api_keys / capabilities / pgvector);docker-compose local+prod;Makefile
- skill 注册表(builtin 仅 greeting)

## 3. 已完成能力(验证状态)

- pytest 195 全绿 · ruff/mypy 门禁通过
- 双路径执行(意图≥0.85 → skill 直连 $0/50-200ms,否则 LLM)
- 密码注册/登录 + 限流 + 审计(Task 38)
- RAG 全链路 / 三层记忆 / 能力链编排 / Dify-Coze 连接器 / 预算设计(Task 32)
- LangFuse 全链路 / 审计 / 护栏 / 断路器 / key failover / HMAC 签名

## 4. 要做什么(知识清单,时间问题,不紧急)

### A. V1.x 收尾(修'代码已烂')
1. **缓存写侧缺失** — cache_check 只读不写,精确缓存形同虚设(cache_entries 写入方只有 files/personalization)
2. **实体抽取空壳** — analyze_parallel 的 entities 恒为空,假并行(gather 单任务)
3. **消息归一化缺失** — 精确缓存哈希吃原始字符串,"你好"vs"你好 " 必 miss
4. **Chat 旁路** — enhanced_chat / streaming_chat 职责与管线重叠
5. **Agent 孤儿** — backend/agent/(V2 Runtime)与 capability kind=agent 双轨

### B. 新定位第一仗(IT 配能力链 + chat/按钮调用)
6. **chat→能力链桥接** — skill 包装 capability invoke,chat 喊话调链(现两条路是断的)
7. **工作台按钮页** — FE 按钮 → 直连 invoke,业务用户一键执行
8. **OA/财务数据连接器** — 硬骨头,导入+定时同步起步

### C. 架构深化(面试谈资级)
9. **凭证拆分** — api_keys 一表两用(机器凭证 vs 人类会话),登录轮换一锅端
10. **统一上游管理面** — LLM(harness)与外部平台(capability connector)两套体系合并
11. **Prompt 版本管理** — 现在只有 A/B prefix 注入
12. **任务中断恢复** — SSE 断开即中止,无断点续传
13. **多 Agent 并发调度** — 现在只有线性链式协作
14. **节点独立超时** — 现在单节点无超时,坏节点拖全链

## 5. 简历/面试弹药

- 数字:195 tests · 13 节点 DAG · 50-200ms/$0 双路径 · 4 角色 RBAC · AES-256-GCM 密钥治理
- 决策故事:为什么 LangGraph(StateGraph vs 裸编排)、为什么双路径(按成本设计)、为什么 key 只存哈希、为什么三层记忆、为什么能力中枢
- 对标:Agent 平台岗(多 Agent 编排/记忆/双路径)、网关治理岗(管线/治理层/密钥治理)——场景是壳,平台能力是芯
