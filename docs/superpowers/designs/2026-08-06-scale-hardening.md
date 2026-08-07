# 高并发加固 + 记忆层增强(2026-08-06 讨论记录)

> 日期:2026-08-06 · 状态:**已拍板(记录)** · 关联:master **D17** · Task 32(预算 Redis 桶)· Task 40/41
> 来源讨论:「NexusAI 怎么没有一个 queue 管理用户 API?用户量大怎么搞」(08-06 晚)+「开源 packages 选型(mem0/Zep/Prompt 管理)」(08-06 下午)
> 原则:同步请求不排队,准入控制削峰;队列只长在异步任务层。

---

## 1. 现状核实:不是没管理,是用了「准入控制」而不是队列(08-06 代码核实)

| # | 机制 | 位置 | 作用 |
|---|------|------|------|
| 1 | 租户级 TokenBucket 限流 | `backend/core/rate_limiter.py`(10 req/s、burst 20) | 削峰靠「快速拒绝」,不是排队 |
| 2 | 断路器 + fallback | `backend/core/circuit_breaker.py` | 上游 LLM 挂了不拖垮自己 |
| 3 | 配额/预算引擎 | Task 32 设计(单次/日/月三窗口 Redis 桶) | 超限拒绝 |
| 4 | 缓存四层 | exact/模板 + RAG L1/L2 + 记忆热缓存 | 减上游命中 |
| 5 | dual-path 短路径 | intent 命中直接 skill 执行(50-200ms,零 LLM) | 简单请求根本不占 LLM |
| 6 | 边缘 nginx | `deploy/nginx.conf` | 入口层 |

**为什么同步 API 不放队列:** 同步请求的用户在等响应,队列 = 排队等待 = 直接加延迟。
正确的削峰是限流(快速 429)+ 并发上限 + 扩容,不是排队。FastAPI/uvicorn 异步事件循环
本身能扛几千并发连接 —— 瓶颈从来不在连接数,在 LLM 上游和数据库。

## 2. 队列的正确位置:异步任务层

用户量大时,「慢活」不能占着请求线程:

- **workflow Runner (Task 40)**:用户点「运行」→ HTTP 立即返回 run_id → 后台队列 + worker 执行 → 前端轮询状态(W3 预留的「任务队列 / 多 worker」,扩展阶梯见 40.36)
- **记忆整合**(Task 41 设计):Redis 队列 + worker,写路径异步
- **链 B 定时任务**:cron → 队列 → worker

这些设计里都已埋点,只是未实现 —— 这就是「队列」的正确位置。

## 3. 用户量大,按这个顺序搞(四件事)

| 优先级 | 事项 | 现状 → 目标 | 为什么 |
|--------|------|------------|--------|
| **P0** | 限流迁 Redis | 进程内存 TokenBucket → Task 32 的 Redis 桶 | 单进程没事;gunicorn 多 worker 后每 worker 一个桶 → 10/s × N worker,租户限额形同虚设 |
| **P0** | LLM 全局并发信号量 | 代码里无 semaphore → per-model + per-tenant 并发上限(如每模型 50 并发),超了排队等信号量或快速拒绝 | 量大时所有请求同时打到 LLM 上游 = 触发上游限流/熔断 |
| P1 | workflow 队列化 | 长任务进 Redis 队列 + worker,HTTP 只收 run_id 返回 | 慢活不占请求线程;链 A 排期里已有,优先级中 |
| P2 | 边缘 + 扩容 | nginx 限流/负载均衡 → 多实例(无状态:状态在 PG/Redis,实例可随便加)→ K8s | 缓存每实例一份无妨,Redis 共享 |

**诚实评估:** 现量级(个人学习 + 演示)下准入控制完全够;真到「用户量大」,
最先爆的不是队列缺失,是**进程内限流失效**(多 worker 后)和 **无 LLM 并发上限**(上游被打爆)。
这两件是 P0。

## 4. 记忆层/上下文增强方向(08-06 下午评估结论)

### 4.1 开源包逐家结论

| 包 | 结论 | 理由 |
|----|------|------|
| **LangGraph Store (PostgresStore + semantic search)** | **★ 唯一建议「直接用」** | 落在已有 PostgreSQL + pgvector,零新基础设施;与 LangGraph DAG 原生同构,天然被 LangFuse CallbackHandler 追踪(档位 B:同 trace、可审计);可作 cold/warm 层检索底座 |
| mem0 | 借鉴思路,不引入 | 内部自调 LLM 做抽取 = 黑盒,违反 harness 约束(EVID-08:LLM 依赖必须走 LLM_PROVIDER);开源版拿不到 README 的 benchmark(托管平台专属优化);对「全链路可控」诉求不可审计 |
| Zep / Graphiti | 当 benchmark 参考,不引入 | 需 Neo4j + LLM 抽取,部署面大;Zep Cloud 闭源;pgvector + 时间戳列 + 关系表覆盖 80% 场景 |
| Letta / Cognee | 不碰 | agent runtime / graph-native 管线,与网关定位冲突,重且绕 LLM |
| PromptLayer / Helicone / Agenta | 一个不加 | LangFuse 内置已有:版本化、get_prompt(label)、Prompt Experiments、promptfoo 集成;多一个平台 = 多一个数据面 + 合规审计面 |

### 4.2 mem0 v3 算法思路(抄进 memory_service,自研、走 harness)

- single-pass **ADD-only** 抽取(一次调用、只增不删,天然留痕)
- **entity linking**(实体跨记忆关联,检索增强)
- **multi-signal retrieval**(语义 + BM25 + 实体,多路融合)
- **temporal reasoning**(时间感知排序)

现有 hot/warm/cold 三层 + pgvector,加这些是增量改造,不是重写。

### 4.3 LangFuse 对接质量两档

- 档位 A:span 能进 LangFuse,但是孤立 span(独立进程,无 trace 上下文)—— mem0 + OpenLIT 默认形态,需手工传播 trace_id 才连到对话
- 档位 B:进同一条 trace,和 DAG 节点并排,点开能看到「记忆检索了什么、抽取调了几次 LLM、花了多少 token」—— 审计要的「可溯源」;LangGraph Store 原生就是这档

### 4.4 补评测面:promptfoo

离线 eval + red teaming(注入红队、输出护栏回归脚本化),补 P0/P1 安全护栏的自动化测试面。
注意定位是「评测工具」不是「prompt 平台」,和 4.1 的结论不冲突。

## 5. 落点

| 项 | 落点 | 优先级 |
|----|------|--------|
| 限流迁 Redis(多 worker 后失效) | Task 32 的 Redis 桶设计直接落地 | P0 |
| LLM 全局并发信号量(per-model + per-tenant) | 独立小任务书(挂 Task 41 旁或独立编号) | P0 |
| workflow 队列化(Redis 队列 + worker,run_id 即返) | 链 A 排期(W3 40.36 扩展阶梯) | P1 |
| 边缘 nginx 限流/负载均衡 + 多实例扩容 | 后置 | P2 |
| LangGraph PostgresStore 语义检索立项评估 | 记忆层(Task 41 延伸) | 中 |
| mem0 v3 算法思路(ADD-only / entity linking / 多信号 / 时间排序)自研增强 | memory_service 增量改造,走 harness 保审计 | 中 |
| LangFuse prompt 管理(版本化/get_prompt/Experiments) | **已落地**:Slice 1(d9d9d9d,prompt_service) | ✅ 完成 |
| 零 LLM 记忆抽取 | **已落地**:Slice 2a(d9d9d9d,extractor) | ✅ 完成 |
| promptfoo 接入(评测/red-team) | 评测面,待立项 | 低 |
