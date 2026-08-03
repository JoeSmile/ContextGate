# Task 30: 能力中枢 + 统一前端(Capability Hub / Agent OS 底座)

> **状态: 待执行(Cursor)。29 个子任务,每个独立文件、独立 AC、独立 commit。**
> **执行顺序: 30.01 → 30.28(阶段 1),30.29(阶段 2)不在本轮。**

## 决策记录(2026-08-02, Joe 拍板)

1. **统一前端**: 全新 React 前端,只对接 ContextGate 一个后端(替换 ai-platform 前端,不再双前端)。
   ai-platform 的 NestJS 后端暂作上游 provider(方案 B),不并入;datagateway/GraphSpec 类型是其资产,本轮不动。
2. **一切皆能力(Capability)**: 模型 / 工作流 / 数据源 / 外部应用 / Agent 统一为 Capability,
   注册进 capability registry,统一 invoke 通道,治理(审计/护栏/成本/配额)在 invoke 层一次实现。
   Dify/Coze 应用 = capability(kind=external_app, provider=dify/coze),网关代理转发,现有企业应用零改动。
3. **多 Agent(本轮含 Agent 调 Agent)**: Agent = 组合型 Capability(角色+记忆+能力集),
   递归抽象使 Agent 可调 Agent,治理层看到的是两条能力调用记录,成本/审计/护栏全程不脱管。
4. **自研 workflow studio 后置**(对标 Dify 的 react-flow 画布工作量大),导航预留入口 + GraphSpec 类型地基先建。
5. 前端技术栈(用户指定): React 19 + Vite 8(脚手架 create-vite 当前默认;原记 Vite 6,2026-08-03 拍板接受 8) + TypeScript + zustand(persist 认证) + @tanstack/react-query + Tailwind v4 + Radix UI(shadcn 风格)。
   SSE 手写 fetch + ReadableStream 解析器(不用 EventSource——POST + 自定义头;不用 fetch-event-source——后端无 Last-Event-ID)。
6. **首轮交付 = 测试 FE**: 先做统一测试前端,替代散装 HTML QA 页(playground/admin/rag/agent 等 7 个互不相通页面)
   ——灵魂功能是**角色切换器**(4 角色 key 一键切换 + 角色徽章 + 403 高亮),让"user 看到什么 / admin 看到什么"的权限边界一眼可见。
   产品 FE(能力市场 / 工作台 / 治理中心)是**扩展阶段**,等测试 FE 实测出别扭点再定义——先做狗粮,再用狗粮验证产品。
7. **粒度原则**: 每个子任务 = 一个文件 + 自带 AC + 一次 commit + 一次快速验收(1-2 个文件,独立验证命令)。
   禁止大坨交付——Cursor 每做完一个立即 commit,review 粒度小。
8. **Review 卡点**: 每完成约 **4–5 个子任务**做一次 code review(Standards + Spec)。
   Critical/Minor 当场修;Important 列给 Joe 拍板后再进下一批。卡点建议: 30.05 / 30.10 / 30.15 / 30.20 / 30.25 / 30.28。

## 子任务索引(29 个)

| # | 文件 | 阶段 | 依赖 | 内容 |
|---|------|------|------|------|
| 30.01 | [30.01-capability-models.md](30/30.01-capability-models.md) | 1a 后端 | 无 | Capability 模型 + 错误码 |
| 30.02 | [30.02-capability-registry.md](30/30.02-capability-registry.md) | 1a | 30.01 | Registry + 005 migration |
| 30.03 | [30.03-capability-config.md](30/30.03-capability-config.md) | 1a | 30.02 | config.env + 凭证接入 |
| 30.04 | [30.04-capability-invoke.md](30/30.04-capability-invoke.md) | 1a | 30.02 | invoke 核心分发 + 成本幂等 |
| 30.05 | [30.05-capability-governance.md](30/30.05-capability-governance.md) | 1a | 30.04 | 治理强制(护栏/限流/配额) |
| 30.06 | [30.06-capability-router.md](30/30.06-capability-router.md) | 1a | 30.04, 30.05 | router + LangFuse + 挂载 |
| 30.07 | [30.07-external-connectors.md](30/30.07-external-connectors.md) | 1a | 30.03 | Dify/Coze 连接器 |
| 30.08 | [30.08-frontend-scaffold.md](30/30.08-frontend-scaffold.md) | 1b 前端基建 | 无 | Vite 脚手架 + shadcn |
| 30.09 | [30.09-ux-tokens.md](30/30.09-ux-tokens.md) | 1b | 30.08 | UX token 落地 |
| 30.10 | [30.10-vite-proxy.md](30/30.10-vite-proxy.md) | 1b | 30.08 | proxy + 目录骨架 |
| 30.11 | [30.11-http-auth.md](30/30.11-http-auth.md) | 1b | 30.10 | http 客户端 + authStore(4 槽位) |
| 30.12 | [30.12-sse-hooks.md](30/30.12-sse-hooks.md) | 1c 前端核心 | 30.11 | useSSEStream + useChatStream |
| 30.13 | [30.13-app-shell.md](30/30.13-app-shell.md) | 1c | 30.11 | AppShell + 路由 + 登录页 |
| 30.14 | [30.14-role-switcher.md](30/30.14-role-switcher.md) | 1c | 30.11, 30.13 | 角色切换器(灵魂) |
| 30.15 | [30.15-panel-components.md](30/30.15-panel-components.md) | 1c | 30.09, 30.14 | RequestPanel + SSEPanel |
| 30.16 | [30.16-panel-chat.md](30/30.16-panel-chat.md) | 1d 面板 | 30.12, 30.15 | Chat 面板(SSE) |
| 30.17 | [30.17-panel-rag.md](30/30.17-panel-rag.md) | 1d | 30.15 | RAG 面板 |
| 30.18 | [30.18-panel-admin.md](30/30.18-panel-admin.md) | 1d | 30.15 | Admin 面板 |
| 30.19 | [30.19-panel-audit.md](30/30.19-panel-audit.md) | 1d | 30.15 | Audit 面板 |
| 30.20 | [30.20-panel-agent.md](30/30.20-panel-agent.md) | 1d | 30.15 | Agent 面板 |
| 30.21 | [30.21-panel-eval.md](30/30.21-panel-eval.md) | 1d | 30.15 | Eval 面板 |
| 30.22 | [30.22-panel-performance.md](30/30.22-panel-performance.md) | 1d | 30.15 | 性能面板 |
| 30.23 | [30.23-panel-capabilities.md](30/30.23-panel-capabilities.md) | 1d | 30.15 | Capabilities 面板 |
| 30.24 | [30.24-agent-gateway.md](30/30.24-agent-gateway.md) | 1e Agent | 30.02, 30.04 | Agent 门面 + seed |
| 30.25 | [30.25-agent-nesting-ui.md](30/30.25-agent-nesting-ui.md) | 1e | 30.20, 30.24 | Agent 嵌套链标注 |
| 30.26 | [30.26-frontend-tests.md](30/30.26-frontend-tests.md) | 1f 收尾 | 30.12-30.15 | 前端单测 |
| 30.27 | [30.27-backend-tests.md](30/30.27-backend-tests.md) | 1f | 30.01-30.07 | 后端单测 |
| 30.28 | [30.28-readme.md](30/30.28-readme.md) | 1f | 全部 | README + 过渡说明 |
| 30.29 | [30.29-product-fe.md](30/30.29-product-fe.md) | 2 扩展 | 阶段 1 实测后 | 产品 FE(不在本轮) |

## 架构总纲(前后端分层)

```
工作台(用户入口): 能力市场 / 气泡调用 / Agent 选择
  │
Agent 层(组合能力): Agent = 角色 + 记忆 + 能力集;Agent 可调 Agent
  │
Capability 层(原子能力): model / datasource / tool / workflow / external_app
  │
治理层(OS 内核): 认证 X-API-Key · RBAC0 · 审计 · 成本 · 配额 · 护栏 · LangFuse trace
```

前端目录: `frontend/`(项目根新建,独立 package.json,与后端 uv 隔离)。

**测试 FE 与产品 FE 的关键差异(实现时不可混淆):**
- 测试 FE: 同一界面 + 角色切换器(4 key 一键切换)+ 403 高亮 —— 权限边界可视化
- 产品 FE: 按角色裁剪界面(user 看不到管理区)—— 权限边界隐藏化
- 测试 FE 的角色切换器在产品 FE 中保留为"调试模式"(admin 可开),不删除

## UX 设计规范(硬性要求,对齐 Dify × 阿里云控制台)

> 基准来源: 阿里云 Console Design(aliyun.github.io/alibabacloud-console-design,Wind 组件库)的
> 控制台布局/密度/色彩体系 + Dify 的对话/卡片/徽章现代感。用户拍板: 别搞默认模板脸。

**布局骨架(所有页面统一):**
```
┌────────────────────────────────────────────────────┐
│ Topbar: logo · 环境徽章(dev/test/demo) · 全局搜索 · 用户菜单 │
├──────────┬─────────────────────────────────────────┤
│ 侧边栏    │ 内容区(白底,浅灰页面底 #f5f7fa)          │
│ 演示区     │  · 面包屑                               │
│  能力市场  │  · 页面标题区(标题+副标题+右侧主操作按钮) │
│  工作台    │  · 内容卡片(圆角 8px,细边框)            │
│ 治理区     │                                        │
│  审计/成本 │                                        │
│ 管理区     │                                        │
│  管理台/…  │                                        │
└──────────┴─────────────────────────────────────────┘
```

**设计 token(写进 tailwind 主题 / CSS 变量,全站引用,禁止散写硬编码色值):**
- 主色: 云蓝 `#1677ff`(hover `#4096ff` / active `#0958d9`),与 Dify 蓝同族
- 背景: 页面 `#f5f7fa`,内容卡 `#ffffff`,边框 `#e5e6eb`
- 文字: 主 `#1f2329` / 次 `#646a73` / 弱 `#6b7280`(无障碍 P1,原 #8f959e 不达 AA)
- 圆角: 卡片/按钮 8px,徽章 999px
- 阴影: 卡片轻阴影 `0 1px 2px rgba(0,0,0,.04)`,浮层 `0 6px 16px rgba(0,0,0,.12)`
- 字号: 页面标题 20/600 · 卡片标题 14/600 · 正文 14 · 表格/辅助 12
- 密度: 表格紧凑(行高 40px 内),控制台感;数字用 `tabular-nums`
- 字体: 系统栈 `-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif`
- 状态色: 成功 `#00b42a` / 警告 `#ff7d00` / 错误 `#f53f3f`(阿里云语义色)

**无障碍最小清单(WCAG 2.1 AA):**
- 正文/次文字对比度 ≥4.5:1;12px 辅助文字与背景对比 ≥4.5:1(在 #fff 上)
- 全部交互元素键盘可达 + focus 可见环(2px 云蓝 outline)
- 表单控件带 aria-label / 关联 label;图标按钮加 aria-label
- 表格用 `<th scope>` 语义列头;toast 区域 aria-live=polite
- 徽章/状态点不只靠颜色传达(附文字标签),红绿色盲可辨

**组件基调(shadcn/ui Radix 基座 + 上述 token,不许默认紫/蓝渐变脸):**
- 侧边栏: 浅色(#fff 或 #f7f8fa),分组标题 12px 弱化,选中项云蓝浅底+蓝字
- 按钮: 主按钮实心云蓝;次按钮白底细边框;危险按钮红
- 表格: 白底、细分割线、hover 浅灰、状态列用徽章(success/warning/error 三色)
- 卡片: 带 kind 徽章 + lucide 图标;hover 抬升阴影
- 对话: Dify 风格气泡 + 打字机光标 + 流式 token 平滑追加
- 空态/加载: 骨架屏(灰块 shimmer),空态图标+文案+引导按钮,禁止白屏裸奔
- 反馈: 操作成功 toast(右下角),失败 error banner(带错误码,如 AUTH_001)

**页面级对齐参考(实现前各自打开看一眼):**
- 能力市场 → Dify"应用列表"页;工作台 → Dify 对话 webapp;治理/管理页 → 阿里云控制台资源列表;登录页 → 阿里云居中卡片式

## ⚠️ Cursor Pitfalls(每个子任务实现前必读)

1. **EventSource 不可用**: /chat/streaming 是 POST + X-API-Key,EventSource 只支持 GET 且无法带自定义头。必须 fetch + ReadableStream。也不要引入 @microsoft/fetch-event-source(后端无 Last-Event-ID,重连逻辑无用)。
2. **双格式响应**: 同一 invoke 端点,短路径返回 `application/json`(无 SSE 帧),长路径返回 `text/event-stream`。useSSEStream 必须先读 Content-Type 再决定解析路径。
3. **心跳是注释行**: `: ping` 是 SSE 注释(冒号开头),解析器必须跳过,不能当 data 解析,更不能 JSON.parse 它。
4. **短路径降采样**: LangFuse 短路径采样率 0.1(默认),trace 缺失不是 bug,别改采样率。
5. **replay/mock 是假流式**: LLM_PROVIDER=replay 时 token 是逐字切片,演示 OK;要真流式用 `make demo`(LLM_PROVIDER=openai)。
6. **key 只明文一次**: 管理台创建 API key 后必须立即展示(仅此一次),别在列表页二次查询明文(后端不存明文)。
7. **CORS**: dev 用 Vite proxy(30.10),不要依赖后端 FRONTEND_ORIGINS;生产再考虑 FastAPI 挂静态文件或 nginx 反代。
8. **Agent 递归防环**: Agent 调 Agent 深度上限 3,Agent 能力集引用自身时启动校验拒绝。
9. **别动已验证引擎**: harness / RBAC0 / 审计 / 断路器 / key 治理是 116 个测试守护的资产,capability 层只在其上"加",不"改"。
10. **LangFuse 分层(接缝 1)**: `_inject_langfuse_parent` 在 pipeline/router.py:181(pipeline 层)——**core/capability 禁止 import pipeline**(分层倒置),根 trace 注入放 routers/capability.py(30.06),core 只做纯分发(30.04)。
11. **外部应用必须显式过治理(接缝 3,安全关键)**: kind=external_app / agent 绕过 LangGraph 管线,护栏/限流/配额必须在 invoke 层显式补(30.05),注册时校验,否则"网关代理"= 治理旁路。
12. **成本别双计费(接缝 4)**: harness 内部已 record_consumption(harness/llm.py:94/198),invoke 层对 kind=model 不重复记(30.04);只对 external_app/agent 记账,审计带 cost_source。
13. **别建双 Agent 世界(接缝 2,验收 P1-2)**: 真实实现是 `backend/agent/`(V2 Runtime,经 routers/agent.py 挂载,agent_service.py:9 import 它);`backend/modules/agent/` 是未挂载孤儿副本——AgentRuntime 门面包装 agent_service(30.24),modules/agent 本轮不动,文档注明"未挂载,勿引用"。
14. **UX 基准是硬性要求,不是可选**: 所有页面必须走 shadcn 骨架组件 + UX 规范 token,禁止散写色值/内联样式;对照 Dify/阿里云参考页自查(布局/密度/空态/反馈四件套)。
15. **别用默认模板脸**: Vite 默认样式、shadcn 默认紫色主题、浏览器默认按钮/输入框样式都是"难看"来源——初始化后第一件事就是套 token。
16. **测试 FE 不是产品 FE**: 本轮是测试 FE——同一界面 + 角色切换器 + 403 高亮,不是"角色裁剪界面"。角色裁剪是 30.29 扩展阶段的事。
17. **key 四槽位不要互覆盖**: authStore 里 user/tenant_admin/auditor/super_admin 四个 key 槽位独立存储,切换只换激活槽位,不清其他槽位。
18. **一次 commit 一个子任务**: 按 30.01→30.28 顺序逐个做,每个做完跑该文件 AC + commit,在 tasks/README.md 勾选,不要攒批。

## 评审记录(三轮 + 两轮策略调整)

> 2026-08-02 第一轮(代码审查 bg_232516): 4 个接缝——① 分层倒置(30.04/30.06)② 双 Agent 抽象(30.24)
> ③ 非 model 治理旁路(30.05)④ 成本双计费(30.04)。均已写入对应子任务与 Pitfall。
>
> 2026-08-02 第二轮(五角色评审): P0 key 存储 sessionStorage(30.11)、用户/能力管理页(30.29);
> P1 配额模型(30.03/30.05)、Agent 流式化(30.24)、角色落地页(30.29)、审批页(30.29)、无障碍(UX 规范)、流式批渲染(30.29)。
>
> 2026-08-02 第三轮(独立验收 deleg_76e7c601,15 条断言全核实): P1-1 005 migration(30.02)、
> P1-2 双 Agent 现状(30.24);P2 注入机制(30.24)、PATCH role 端点(30.29)、last_used_at 粒度(30.29)、
> 分钟桶限流(30.05)、/health 前缀(30.10/30.11)、users 表(30.29)、AgentSpec 存储(30.24)、成本数据源(30.29)。
>
> 2026-08-02 第四次(交付策略, Joe 拍板): 首轮 = 测试 FE,产品 FE 后置为 30.29。
>
> 2026-08-02 第五次(粒度, Joe 拍板): **拆为 29 个子任务,每个独立文件 + AC + commit**。本文件为总纲索引,
> 具体内容在 tasks/30/*.md。Cursor 按 30.01→30.28 顺序逐个完成,每步跑独立验证 + 勾 AC。
>
> 2026-08-03(收尾评审 4B, Joe 拍板 **B**): 测试 FE **接受各面板手写 Card 现状**,关闭
> 「RequestPanel 统一骨架」要求——不阻塞 30.28 归档。产品 FE(30.29)若需统一壳再另开
> `PanelShell` / 扩展 RequestPanel,不在本轮强行套单表单组件。

## 验收清单(全绿才归档)

**阶段 1(本轮,30.01-30.28 全部完成):**
- [ ] 30.01-30.28 每个子任务 AC 全绿,tasks/README.md 完成表逐个勾选
- [ ] `make check`(ruff + mypy)通过
- [ ] `uv run pytest` 全量通过(含 test_capability.py)
- [ ] `cd frontend && npm run test && npm run build` 通过
- [ ] 测试 FE: 角色切换器 4 槽位一键切换 + 403 高亮;8 面板全可调
- [ ] journeys 联动: 01-user 任务 1.2(403→申请)、02-tenant-admin 任务 2.2(审批)用测试 FE 走通
- [ ] 后端: invoke 通道 + 005 migration + Agent 调 Agent 嵌套链(审计三条记录)
- [ ] UX/无障碍/安全自查(见各子任务 AC)

**阶段 2(30.29,不在本轮):**
- [ ] 能力市场 / 工作台 / 治理中心 / 角色裁剪 / 用户管理 / 能力管理 / 审批页;journeys 4 角色端到端

- [ ] 本文件移入 `tasks/archive/`,更新 `tasks/README.md` 完成表

## Backlog(本轮不做,防止丢失)

- [ ] demo 三连剧本(Agent 嵌套链 / 审计溯源 / 成本看板)→ docs/DEMO_SCRIPT.md
- [ ] 会话持久化(工作台会话列表/恢复)
- [ ] 成本分摊报表深化(部门/项目维度 + 环比 + 导出)
- [ ] 预算告警通知(80% 预警 / 100% 熔断)
- [ ] API key 轮换提醒 + 过期自动停用
- [ ] i18n(文案收敛 locale 文件)
- [ ] 深色模式(token 层预留 dark 变量)
- [ ] 响应式断点(1366/1920 + 侧边栏折叠)
- [ ] 审计保留策略(留存天数 + 归档)
- [ ] 生产部署形态(FastAPI 静态 / nginx 反代 + Docker)
