# NexusAI = 企业 AI 中台(定位与五层架构)

> 更新:2026-08-05。本文档固化「NexusAI = 企业 AI 中台」的拍板结论:一句话定位、用户与场景、
> 五层架构(含与代码的「已有/待建」映射)、阶段划分、面试可讲的六个点、风险与原则。
> 配套:`docs/strategy/README.md`(战略档案总览)、`docs/ARCHITECTURE.md`(技术架构)、`learning/01-architecture.md`(学习文档)。

## 1. 一句话定位

**NexusAI = 企业 AI 中台。**

企业都有业务中台、数据中台,缺的是 AI 中台。NexusAI 补上这一层:**统一入口调企业 AI 能力,
能力由 IT 编排,数据接内部 OA/RAG,治理平台兜底。**

> 与战略档案的关系:治理层是战略本质,AI 中台是市场定位,两者兼容——
> 前者回答「我们是什么」,后者回答「客户怎么理解我们」。详见 `docs/strategy/README.md` 的「一句话定位」。

## 2. 用户与场景

(拍板事实,2026-08-05)

| 维度 | 结论 |
|------|------|
| 目标客户 | 国有大型企业集团,决策人是 CIO / 信息中心主任 |
| 使用者 | 各部门业务人员(user)用平台干活(第一阶段不自己编排);IT/admin 管平台、管权限、管数据接入 |
| 角色模型 | 4 角色已在代码中编码:super_admin(跨租户)/ auditor(跨租户只读审计)/ tenant_admin / user |
| 角色含义 | user = 业务,admin = 管理 |
| 数据对接 | 企业内部 OA + RAG 知识库(后续财务系统) |
| 平台兜底 | 权限 / 数据安全 / 审计由平台兜底 |

要点:

- 第一阶段的编排权在 IT:业务用户消费能力,不生产编排。
- 审计角色是国企的硬需求:跨租户只读,面向合规与监察。

## 3. 五层架构

五层从上到下:L0 接入层 → L1 能力编排层 → L2 数据连接层 → L3 治理层 → L4 横切层。
每层写:定位一句话、构成、「已有/待建」代码映射、设计点。

### L0 接入层——三种入口,一个引擎

**定位:** 用户和业务系统怎么进到平台。

构成:

- **chat** — 自然语言入口,意图识别驱动,LLM 成本高,处理模糊需求
- **按钮 + 工作台** — 确定动作,直连 capability chain,$0
- **API** — 业务系统接入,X-API-Key / HMAC

代码映射:

| 已有 | 待建 |
|------|------|
| chat 管线(`backend/pipeline/`)、API key 认证(`backend/core/auth/api_key_auth.py`) | 工作台按钮页(纯 FE:按钮 → 直连 invoke) |

设计点:

- 入口是 UX 的事,引擎是架构的事,两者解耦。
- 同一个 capability chain,chat 和 button 都能触发,成本不同:chat 走 LLM(模糊需求),button 零成本直连。

### L1 能力编排层——中台的芯

**定位:** 企业 AI 能力的注册、编排与执行,是中台的芯。

构成:

- **能力注册表** — 目录 + 权限绑定(已有)
- **capability chain** — IT 编排的固定流程,rag-ask → contract-query → vendor-risk 已有雏形
- **执行引擎** — LangGraph DAG,支持分支/条件/审计节点(引擎已有)
- **双路径** — confidence ≥ 0.85 直连 skill 执行($0 / 约 50ms),否则走 LLM 管线(已有)

代码映射:

| 已有 | 待建 |
|------|------|
| 能力注册表、能力链雏形(30.24 递归链)、双路径(`backend/pipeline/nodes/model_router.py`,intent → skill 直连)、LangGraph 执行引擎(`backend/pipeline/graph.py`) | chat → capability chain 桥接(让 skill 能包装能力链;IT 配链 + 注册意图,chat 喊话调用) |

设计点:

- 第一阶段编排权在 IT 不在业务用户——灵活性让给可控性。
- 自助编排留到 V2,只加 UI,不动引擎。

### L2 数据连接层——为什么叫「中台」

**定位:** 平台连接企业数据,「中台」一词的由来。

构成:

- 数据源:OA / RAG 知识库 / 财务系统(后续)
- 连接器 + 租户/角色/行级权限隔离
- 数据不出域(私有化部署)

代码映射:

| 已有 | 待建 |
|------|------|
| RAG 知识库(`backend/modules/rag/`)、多租户隔离(`backend/core/tenant.py`) | OA 连接器、财务连接器(硬) |

设计点:

- 数据在墙内,AI 在墙内,治理在中间。

### L3 治理层——和「套壳 chat」的分水岭

**定位:** 权限、审计、预算、护栏、审批,是平台区别于「套壳 chat」的分水岭。

构成:

- **RBAC 4 角色 + 能力级动态权限** — 已有
- **全量审计 + 合规导出** — 已有(audit_logs)
- **预算配额** — 已有(Task 32)
- **安全护栏** — 注入 / PII / 输出审查 / 断路器,已有
- **审批流** — 权限申请 → 审批已有雏形(`backend/routers/admin.py` 的 `/permissions/request` → `/pending-requests` → `/approve`);将来加 workflow 审批节点

代码映射:

| 已有 | 待建 |
|------|------|
| 4 角色 RBAC(`backend/core/auth/`)、能力级动态权限、审计(`backend/core/audit.py`)、预算配额(Task 32)、护栏(`backend/core/guardrails/`)、审批流雏形(`backend/routers/admin.py`) | workflow 审批节点(将来,配合 V2 自助编排) |

设计点:

- 治理是卖点不是功能——「你的数据敢不敢给它」。

### L4 横切层

**定位:** 跨层公共能力,横切在每一层之下。

构成:

- LangFuse 可观测
- 模型路由 + harness(mock / record / replay / openai)
- 缓存
- AES-256-GCM 密钥治理(LLM_KEY_MASTER_KEY)

代码映射:

| 已有 | 待建 |
|------|------|
| LangFuse(`backend/observability/`)、harness(`backend/core/harness/`)、缓存(`backend/core/redis_tools.py`)、密钥治理(`backend/core/key_manager.py`) | — |

## 4. 与现有代码的对账(诚实版)

| 板块 | 已有 | 待建 |
|------|------|------|
| 能力注册表 | 已有(能力目录 + 权限绑定) | — |
| capability chain | 已有雏形(30.24 递归链;rag-ask → contract-query → vendor-risk) | — |
| 双路径 | 已有(`model_router.py`:intent → skill 直连;confidence ≥ 0.85 走 skill,$0) | — |
| RBAC | 已有(4 角色:super_admin / auditor / tenant_admin / user) | — |
| 审计 | 已有(audit_logs 全量) | — |
| 预算 | 已有(Task 32) | — |
| 护栏 | 已有(注入 / PII / 输出审查 / 断路器) | — |
| 可观测 | 已有(LangFuse) | — |
| harness | 已有(mock / record / replay / openai) | — |
| 缓存 | 已有 | — |
| chat → 能力链桥接 | — | 待建(小):skill 包装能力链,IT 配链 + 注册意图,chat 喊话调用。现状:chat 管线走 intent → skill 直连,但 skill 注册表只有 greeting;能力链走 `/api/capabilities/{id}/invoke`,两条路是断的 |
| 工作台按钮页 | — | 待建(纯 FE):按钮 → 直连 invoke |
| OA / 财务连接器 | — | 待建(硬):数据源接入 + 行级权限 |

结论:**第一仗现有代码吃掉 80%**,待建主要是三块:chat → 能力链桥接、工作台按钮页、OA/财务连接器。

## 5. 阶段划分

### 第一仗(现在,2026-08-05 拍板)

- 不做自定义 workflow 编排器。
- IT 先配 capability chain;业务用户通过 chat 调用,或工作台按钮调用。
- 现有代码吃掉 80%。

### V2 目标

- 自助编排 UI:**模板 + 表单式编排优先**(业务人员选模板、填参数、排顺序,零学习成本),拖拽式可视化画布后置。
- **审批节点是国企 workflow 的灵魂**:对账报告要领导批、采购申请要流程批。
- 样板链:「拉数据 → RAG 查制度 → LLM 计算 → 人工审批 → 出报告」。

## 6. 面试可讲的六个点

1. **双路径成本工程** —「80% 的简单请求直连 skill 不走 LLM——架构按成本设计,不是按 demo 设计」
2. **多入口统一引擎** —「chat / button / API 三入口打同一个引擎,入口解耦于引擎」
3. **编排权和治理权分离** —「业务用户不碰编排(第一阶段),IT 配链、平台管权限审计;将来开放自助编排,引擎零改动」
4. **和 Dify 的分工** —「Dify 做应用编排,我们做治理——应用可以跑在 Dify,数据必须过我们」
5. **国企准入证** —「私有化 + 数据不出域 + 全量审计,云厂商做不到(数据在他们那),这是合规不是功能」
6. **中台关系** —「不是再造一个中台,是给企业已有的业务/数据中台加 AI 治理层」

## 7. 风险与原则

- **workflow 平台本身是红海**(钉钉宜搭 / 简道云 / 明道云 / Dify 全在做),不要在 workflow 功能上竞争,功能越简单越好。
- **竞争点永远是那一句「你的数据敢不敢给它」**——治理层越厚越好;workflow 板块是壳,治理是芯。
- **数据实事求是**:全部真实数据,不造 demo 数据;没有就是没有,空态/提醒告知用户。

## 8. 关联文档

- `docs/strategy/README.md` — 战略档案总览(治理层定位 + 文件清单)
- `docs/strategy/MOAT.md` — 护城河与生存率分析
- `docs/strategy/MARKET_FACTS.md` — 市场与商业化事实
- `docs/ARCHITECTURE.md` — 技术架构(部署层 / 应用层)
- `learning/01-architecture.md` — 学习文档:架构图 + 中台定位与五层架构
- `learning/13-capability-agent.md` — 能力注册表与 Agent 详解

---

更新:2026-08-05
