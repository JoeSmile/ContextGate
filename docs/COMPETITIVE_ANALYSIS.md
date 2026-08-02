# ContextGate 竞品分析与定位

> 更新:2026-08-01。用途:定位校准 + 售前材料 + 技术内容素材。
> 核心结论:**ContextGate 是 LLM 治理层,不是业务应用层** — 执行层随便换(Dify/Coze/NocoBase/自研 workflow),治理永远在网关。

## 1. 定位宣言:治理层 vs 执行层

AI 应用技术栈分两层,ContextGate 站在下面那层:

```
业务应用层(执行层)          Dify / Coze / NocoBase / Flowise / 自研 workflow
                                    │ 调用 LLM
LLM 治理层(网关层)     ──►  ContextGate: 认证 / 护栏 / 审计 / 成本 / 路由 / 观测
                                    │
LLM 供应商                  DeepSeek / 豆包 / 智谱 / vLLM 本地 / Dify·Coze 的模型端点
```

执行层负责"编排与业务":搭对话、搭工作流、搭业务系统。
治理层负责"边界与证据":谁在用 LLM、用了什么、花多少钱、输出安不安全、出问题能不能溯源。

**给 CIO 的话术(一页版):** 编排工具随便换,审计/合规/成本/护栏永远在网关这一层。今天用 Dify,明天换自研,后天接 NocoBase — ContextGate 不动,治理证据链不断。

## 2. 竞品全景(按分层,不按知名度)

| 产品 | 分层 | 定位 | 与我们的关系 |
|------|------|------|--------------|
| Dify | 执行层 | AI Agent + Workflow 编排(事实标准) | 互补:可当执行层;其模型端点可注册进 ModelRegistry |
| Coze(扣子) | 执行层 | 字节 Bot 平台(国内最佳参考) | 互补:同上;OpenAI 兼容端点 |
| NocoBase | 执行层 | AI+无代码业务系统平台(23k★,Apache-2.0) | 互补:业务应用层,见 §3 拆解 |
| Flowise | 执行层 | 可视化 LLM 流程编排(LangChain) | 互补:同上 |
| LangFuse | 观测层 | LLM trace/span 可观测 | 已集成(自托管 3001),是我们观测能力的底座 |
| LiteLLM / Portkey | 网关层(API 级) | 统一 API / 负载均衡 / 成本代理 | 同类但更薄:API 转发层,无管线、无护栏纵深、无业务状态 |
| **ContextGate** | **治理层(管线级)** | **LangGraph DAG + 全链路治理** | **本产品** |

**与 LiteLLM/Portkey 这类 API 网关的差别(诚实版):** 它们是"请求转发层" — 统一端点、密钥轮换、基础成本统计。ContextGate 是"管线治理层" — LangGraph DAG(意图分流→RAG→护栏→路由→生成→审计)、Skill 直执行($0 短路径)、多租户加密 key、审批流、prompt 注入/PII/输出护栏、A/B 实验、成本聚合看板。API 网关解决"调得到",我们解决"调得安全、调得明白、调得起账"。

## 3. NocoBase 深度拆解(2026-08-01 调研)

### 3.1 归属纠正

**NocoBase 不是豆包/字节的产品。** 独立开源项目(Apache-2.0,GitHub 23k★,杭州团队),多年低代码平台 + 2025-2026 新增 AI 能力。豆包只是其可接入的 LLM 供应商之一(国产平台标配),不是母公司。其官方 AI 文档收录了 Hermes Agent / Claude Code / Codex / OpenCode / 腾讯 WorkBuddy 作为外部协作 Agent — 我们的 Hermes 也在列,可当谈资。

### 3.2 它是什么

AI+无代码业务系统平台:数据建模驱动、所见即所得界面、插件架构、业务工作流、细粒度权限(字段级)、内置"AI 员工"(带权限+审计,进业务流程执行任务)。目标用户:交付团队/集成商给企业搭 CRM/订单/工单/HR 等业务系统。

### 3.3 表面重合面(为什么看着"高度重合")

| 维度 | NocoBase | ContextGate |
|------|----------|-------------|
| 权限 | 字段级 RBAC + AI 员工权限 | RBAC0 四角色 + app 权限 + skill 二级权限 |
| 审计 | 数据变更/流程触发审计 | 全链路 LLM 调用审计(含 token/成本/trace_id/key 版本) |
| 工作流 | 业务流程引擎 | LangGraph 管线 |
| AI 能力 | AI 员工 | Skill 注册表 + 意图路由 |
| 部署 | 私有化 | 私有化 |

### 3.4 实质差异(分层不同,护城河不重叠)

NocoBase **没有**的,正是 ContextGate 的护城河:

- prompt 注入防护、PII 脱敏、输出护栏、断路器(安全护栏四层)
- LLM 成本治理(按租户/模型/时间窗聚合 + 看板)
- 模型注册与策略路由(ModelRegistry:意图/预算/租户 → 模型)
- LangFuse 级 LLM 可观测(trace 树/span/采样)
- 多租户加密 LLM Key 管理(AES-256-GCM,版本轮换)
- A/B 实验(分流/转化落库/指标)
- 双路径管线(Skill 短路径 50-200ms $0 vs LLM 长路径)

它的"审计"是业务数据审计,不是 LLM 调用审计。国企要的"每一次 AI 调用可审计可溯源",它给不了 — 那是网关层的活。

### 3.5 互补接法(推荐,也是内容卖点)

```
NocoBase 业务系统(CRM/工单…)
   └─ AI 员工 / 工作流 HTTP 节点 ──► ContextGate /chat
                                        ├─ 认证 / 护栏 / 审计 / 成本
                                        └─ ModelRegistry ──► 豆包 / DeepSeek / 本地 vLLM
```

同法适用于 Dify/Coze:它们当执行层,我们当治理层。

## 4. 三条防御线(护城河,按优先级)

1. **治理纵深** — 安全护栏全链路:输入(注入/PII)→ 管线(限流/审批/skill 二级权限)→ 输出(敏感词/角色漂移/长度)。这是 P0/P1/P2 分级审计过的纵深,不是单点防护。
2. **可观测与合规** — LangFuse 全链路 trace + audit_logs 全量留痕(含 token/成本/key 版本/trace_id)+ 成本聚合。审计交付物直接可打印给合规部门。
3. **模型无关** — ModelRegistry 把任何 OpenAI 兼容端点注册为 provider:DeepSeek/豆包/智谱/Dify/Coze/本地 vLLM。不绑定任何供应商,客户选型自由。

## 5. 战略结论

- **不做的事:** 不拼无代码业务系统(NocoBase 的地基)、不拼工作流拖拽编辑器(Dify 的护城河)。那不是我们的山。
- **做的事:** 守住治理层,把"执行层随便换"变成卖点 — 反而吃下所有执行层的流量。
- **内容角度(技术博主素材):** 「为什么你的 AI 业务系统需要一个 LLM 网关」「69 元情感课 vs 开源企业级网关:差在哪」 — 每篇文章的论据都指向 §1 的分层图和 §4 的三条防御线。

## 附:后续动作

- [ ] v2.0 前:证据包(MANUAL_TEST 全路线实测)验证 §4 每条防御线都有数字
- [ ] 与 Dify/Coze/NocoBase 的互操作验证(Dify/Coze 模型端点注册 + NocoBase HTTP 节点调 /chat)可列入 v2.0 或单独 PoC
- [ ] 竞品动态跟踪:NocoBase AI 员工能力演进、Dify 企业版合规卖点,纳入 Phase 0 例行检查
