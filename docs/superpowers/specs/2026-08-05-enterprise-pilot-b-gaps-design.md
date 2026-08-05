# 部门试点 B — 企业「直接用」缺口清单（讨论稿）

> 状态:**已锁定（含 2026-08-05 feedback）** — §4 已补全；实现排期见 master plan。  
> 日期:2026-08-05  
> 定位档位:**B 部门试点**（非集团采购 C）  
> 关联:`docs/ROADMAP.md`、`docs/strategy/AI_MIDDLE_PLATFORM.md`、`learning/04a-auth-rbac.md`、[`../plans/2026-08-05-pilot-b-master-plan.md`](../plans/2026-08-05-pilot-b-master-plan.md)  
> 用途:缺口事实源 + 已签核设计（§9–§12）；排期以 master plan 为准。  
> 原则:**质量与安全优先于工期**（组织与 UX 按全面档设计，不为赶工降级到扁平部门）。

---

## 1. 已拍板约束

| 项 | 结论 |
|----|------|
| 档位 | 部门试点上线：日常可用、可运维、基本看板；不是等保正式包 |
| 优先级 | **质量与安全 > 工期**；组织与 UX 按全面档落地，不靠「先扁平再凑合」 |
| UI | 同仓三壳（`/app` `/admin` `/dev`）；业务模板+表单；管理台步骤+只读预览；画布后置；可借开源组件 |
| 组织 | **B 档**（§11）：部门树 + 兼岗 + 平台角色∥业务角色分离 + 唯一 `OrgScope` 门面 |
| 硬门槛 | 安全 / 审计 / 租户隔离 / **稳定** — 不过关不算能试点 |
| 安全红线 | **S1–S4**（§10.2）+ 组织隔离走 `OrgScope`；红线未满足时 **不上挂起等批** |
| 凭证 | **人机拆分**（§9）：人 → JWT；机 → `X-API-Key`；delegation + 连接器 key；二次鉴权 / 挂起等批 |
| 模型 | 与 **微调小模型** 一体推（本地/私有化优先；开发可用 OpenAI 兼容端点） |
| 数据 | 缺口按 **能力抽象**（不绑钉钉/某 OA）；开发用本地 DB / Dev API Server + 像真样例数据 |
| 清单结构 | **2+1 混合**：旅程主表 + 五层 L0～L4 交叉索引 |
| 测得见原则 | 后台有能力、FE/journey 点不到 → **一律算缺口** |

### 1.1 产品优先级（相对原 ROADMAP 上调）

相对 `docs/ROADMAP.md`「阶段一/二不重仓画布」：**本试点讨论稿将 Workflow 编排提到产品主线**。与 ROADMAP 冲突处，以本文为准讨论，合并进 ROADMAP 前需再拍板。

| 优先级 | 内容 |
|--------|------|
| **P0 硬门槛** | 安全 / 审计 / 租户隔离 / 稳定（可复测） |
| **P0 产品主线** | 用户按自身权限 **可视化编排** workflow，并能 **按钮执行**（定时 = 链 B / 试点后） |
| **P0′ 减压阀** | 自研编排 UI 压力大 → **Coze 导入∈链 A**（给人测，**不挡 7A**） |
| **P1** | 业务 Chat：总结卡（考勤/支出/客户等）+ 人工审核卡（如批准请假） |
| **P2** | 其余洞（五层交叉索引扫出） |

### 1.2 Coze 兼容策略

- **长期:** A + B 都要  
  - **A** = 导入/同步 Coze 流程定义 → **在我们侧**按权限执行（试点先 **按钮**；定时属链 B），调用过治理  
  - **B** = 编排与执行仍在 Coze，我们做 LLM/数据调用的治理网关  
- **第一期:** 先做 **A**（导入 → 我方执行）

### 1.3 Chat 形态（两阶段）

- **试点：** Chat 为 `/app` **二级入口**；确定动作以工作台按钮 / Workflow 为主（执行走 Runner，不进 Chat DAG）  
- **后期：** Chat 可升为「办公桌面」（总结卡 / 审批卡 / 对话内触发已发布链）；**执行仍调 Workflow Runner**，不是改回万事走 `/chat` 管线  
- 业务总结与人工审核卡仍属 P1 能力清单（J3）

### 1.4 试点成功口径（7A 人侧金线）

- 有权用户能在 UI 编一条链 → **按钮跑通** → 审计能看到（**定时不进 7A**，见 J1.5）  
- 护栏 / 鉴权 / 租户隔离（含 OrgScope）在上述路径可演示且可复测  
- **J0.5a** 可证：登录发 JWT、产品壳用人会话跑通；**不要求**此时验「人带旧 key→401」（那是 **J0.5b / 7B**）  
- Coze 导入（J2）= 链 A **加测 / 7A 后第一批**，**不挡** 7A 验收  
- 小模型可作为默认生成端点切换并可见于观测  
- 业务 Chat 卡可后置于 P1，但仍列入清单  
- **不要求本档 / 7A:** 等保正式证书、全员无限自由画布、真钉钉写回、假大屏数据、定时 cron、旧 key 401

---

## 2. 结构说明

```text
主表 J*  — 按用户旅程列缺口（下文 §3）
附表 L*  — 五层交叉索引（§4，已补全；拆 tasks 前置）
元规则   — 每个 Must 必须有：面板或对话卡 + examples/qa/journeys 剧本
```

图例:

- **Must** — 不过关不算对应验收点（注明 7A / 7B / 后置）  
- **Should** — 体验差但可口头补  
- **Later** — 试点后 / 链 B  
- **Must（7A 后第一批）** — 链 A 能力，不挡 7A，但须尽快补齐

---

## 3. 旅程缺口主表

### J0 — 硬门槛（贯穿）

| ID | 缺口 | 优先级 | 现状粗判 | 测得见 |
|----|------|--------|----------|--------|
| J0.1 | 租户隔离 + 能力级权限在「编排/执行/审批」全路径可证伪 | Must | 有 RBAC/能力权限；编排/定时路径未贯通 | Admin + Capability + 跨租户 journey |
| J0.2 | 每次执行/审批进 `audit_logs`，auditor 可导出 | Must | chat/invoke 有审计；编排保存/定时跑/业务审批卡未对齐 | Audit 面板 + 导出 |
| J0.3 | 护栏覆盖：导入 Coze 节点若调 LLM/外发，须过治理入口 | Must | 主管线有护栏；Hub/外部路径深浅不一 | 负向用例可见 |
| J0.4 | 稳定：健康检查、失败可诊断、关键路径不静默丢 | Must | `/health`、LangFuse 有；缺「跑挂了去哪看」产品面 | Performance/错误态面板 |
| J0.5a | **人侧 JWT 可用**（链 A / Wave 2A）：登录发 JWT；产品壳用人会话；浏览器产品路径不再依赖长期 `cg_` | Must（**7A**） | 密码登录仍只发 `cg_` | 登录拿 JWT；工作台/Chat 用 Bearer 跑通 |
| J0.5b | **旧人侧 key 拒绝**（链 B / Wave 2B）：`verify_api_key` 仅 machine；人带旧 `cg_` → **401** | Must（**7B**，仅 7A 后） | 兼容窗仍开；2A 不收紧 | 人带旧 key 调人侧/机侧入口最终 401；机侧 machine Key 通 |
| J0.5 | （母项）人机凭证拆分总述 = J0.5a + J0.5b；**7A 只验 a，不验 b** | — | 见上 | 拆子项验收，禁止绑死成一步 |
| J0.6 | 链内/出站二次鉴权 + 可申请 **挂起等批**（approve 后 resume）；须先满足 S1–S4 | Must | `/permissions` 雏形；无 run/node 挂起续跑 | 无权挂起 → **本部门 dept_manager**（找不到再升级 tenant_admin）批 → 续跑 |
| J0.7 | 组织 B：部门树 + 兼岗 + 平台角色∥业务角色；全部列表/Runner 走 `OrgScope` | Must | 仅扁平四角色；无部门 | 跨部门拒绝可证伪；经理批本部门可证伪 |

### J1 — 权限内 Workflow 编排 → 执行（P0 主线）

| ID | 缺口 | 优先级 | 现状粗判 | 测得见 |
|----|------|--------|----------|--------|
| J1.1 | 编排数据模型：用户/部门级 workflow（节点=已授权 capability、边/顺序、参数、版本） | Must | Hub registry + chain 雏形；`kind=workflow` 偏预留；无用户自建一等公民 | 编排面板 CRUD |
| J1.2 | 权限过滤节点：**可见 vs 可调分离** — UI/保存只要求 **可见**；`spec.permission` **可调**仅运行时二次鉴权（不足且 requestable → 挂起） | Must | invoke 有闸；编排保存校验无 | 可见即可存；运行缺权可挂起（金线） |
| J1.3 | 可视化编排 UI（可开源）：至少步骤/图预览；自由拖拽可后置 | Must（预览）/ Should（拖拽） | 测试 FE 无编排台 | 保存 → 预览 → 运行 |
| J1.4 | 按钮执行 + 结果可见（状态/日志/产出） | Must（**7A**） | invoke 有；「跑我的 workflow」入口弱 | 工作台/编排页「运行」 |
| J1.5 | 定时执行：cron/间隔、时区、运行历史、失败可见 | **Later（链 B）** | `scheduler_service` 偏提醒，非 workflow cron；**非 7A 验收项** | 定时配置 + 历史列表（7B 或试点后） |
| J1.6 | 发布/审批闸（跨权或生产链） | Should | `/permissions` 是权限申请，非 workflow 发布 | Admin 待批 |
| J1.7 | chat ↔ Hub 桥接（意图触发已发布链） | Should | 两路断开 | Chat 触发链 + Audit |

### J2 — Coze 兼容（第一期 A；长期 +B）

| ID | 缺口 | 优先级 | 现状粗判 | 测得见 |
|----|------|--------|----------|--------|
| J2.1 | Coze 导出物 → 内部 workflow IR 导入器（不支持则整单拒绝） | Must（**7A 后第一批 / 链 A 加测**；**不挡 7A**） | `external_app` 有 Dify 风格客户端味；无 Coze→IR | 「导入 Coze」面板 |
| J2.2 | 导入落权：映射本租户 capability/模型；无权节点标红不可发布 | Must（同上） | 无 | 同面板校验结果 |
| J2.3 | 导入流走我方执行引擎（与 J1.4 同源历史与审计；定时同源属链 B） | Must（同上） | 无统一用户 workflow runner | 导入后 **按钮**运行 = 同套历史 |
| J2.4 | 长期 B：Coze 内执行、仅网关治理 | Later | 竞品叙事有 | — |

### J3 — 业务 Chat 卡（P1）

| ID | 缺口 | 优先级 | 现状粗判 | 测得见 |
|----|------|--------|----------|--------|
| J3.1 | Chat 结构化响应（文本 + `cards[]`） | Must（做 P1 时） | `approval_request_id` 有苗头；无通用 card 协议 | Chat 渲染卡 |
| J3.2 | 业务读汇总抽象 + Dev DB/假 API + 像真数据（样板≥1） | Must（≥1）/ Should（三类） | RAG 有；业务实体连接器无 | 「本月考勤总结」→ Summary 卡 |
| J3.3 | 人工审核写动作 + 权限 + 审计 + Approval 卡（样板≥1） | Must（做 P1 时） | Admin 权限审批 ≠ 业务单据审批 | 经理批准/拒绝可查 Audit |
| J3.4 | 开源 Chat/卡片壳接入测试 FE | Should | 自研 panel，偏纯文本 | 同 Chat 面板 |

### J4 — 微调小模型一体推

| ID | 缺口 | 优先级 | 现状粗判 | 测得见 |
|----|------|--------|----------|--------|
| J4.1 | ModelRegistry「租户默认小模型」产品化挂载 | Must | Registry/Harness 有；缺向导 | 切换后 Chat/WF 生效 |
| J4.2 | 编排节点/Chat 继承该模型；成本与 trace 可区分 | Must | 部分有；产品面弱 | Performance/LangFuse 按模型 |
| J4.3 | 模型故障可诊断（含流式 failover 对齐） | Should | 非流 failover 有；流式弱 | 错误态可见 |

### J5 — 「测得见」元缺口

| ID | 缺口 | 优先级 | 现状粗判 | 测得见 |
|----|------|--------|----------|--------|
| J5.1 | 每个 Must ↔ 面板或对话卡 + journey 剧本 | Must | 多能力仅 OpenAPI/curl | journeys 打勾清单 |
| J5.2 | Dev 数据面：本地 DB / Dev API + seed 像真域数据 | Must | seed 偏 key/RAG；无考勤/支出/请假域 | 一键 seed 后 UI 有数 |
| J5.3 | 测试 FE 必须可点：编排/导入/运行历史/审批/审计 | Must | 面板散、无编排入口 | AppShell 导航可见 |

### 明确不进本档 Must

- 等保正式测评交付包  
- 全员无限自由画布（无权限节点限制）  
- 真钉钉/企微写回生产  
- 再造平行第二套执行引擎  
- 用假数据撑大屏或编排 Demo  

---

## 4. 五层交叉索引（已补全 · 拆 tasks 强制前置）

> 防止只盯旅程漏治理。**未补全前不得拆 `tasks/*`。**

| 层 | 已有（粗） | 挂接的缺口 / 设计 |
|----|------------|-------------------|
| L0 接入 | chat / API / invoke；缺工作台与编排入口 | **J0.5a**（7A）· **J0.5b**（7B）· J1.4 · J3.* · J5.3 · §9.5 路由 · §12 三壳 |
| L1 编排 | Hub + chain 雏形；缺用户 workflow + runner | J0.6 · J1.1–J1.4 · J1.6–J1.7 · **J1.5=Later 链B** · J2.*（不挡7A）· **T1** IR快照+写节点幂等 · §9.4 |
| L2 数据 | RAG；缺业务实体连接器 / Dev 假源 | J0.5* · J0.6 · J3.2 · J5.2 · 出站连接器 · **RAG 写标+查询 OrgScope**；memory 部门维 N/A |
| L3 治理 | RBAC / audit / 护栏 / 权限审批 | J0.* · J1.2 · J1.6 · J3.3 · §10 S1–S4 · §11 OrgScope · 挂起路由 dept_manager→tenant_admin |
| L4 横切 | LangFuse / Harness / 缓存 / key | J0.4 · J0.5a/b · J4.* · `credential_type` · machine `created_by` · 审计字段 |

---

## 5. 与现有 ROADMAP 的张力（讨论点）

| ROADMAP 原表述 | 本文讨论稿 |
|----------------|------------|
| 阶段一/二不做全员拖拽；阶段三表单+预览再受限画布 | Workflow 编排升为 P0；可视化至少步骤/图预览为 Must，拖拽 Should |
| 第一仗：chat 桥接 + 按钮 + OA 样板 + 指标 | 第一仗叙事改为：**编排+执行(+Coze A)** 置顶；业务卡与 OA 抽象为 P1 |
| Coze 未写入 ROADMAP | 写入 P0′：第一期导入我方执行 |

**待探讨:** 是否回写 `docs/ROADMAP.md` / `AI_MIDDLE_PLATFORM.md`，以及「表单编排 vs 一上来就画布 vs 只做 Coze 导入」的第一期切片。

---

## 6. 待继续探讨的问题（开放）

1. ~~第一期可视化下限~~ → **已决**：业务模板+表单；管理台步骤+只读预览；画布后置（§12）  
2. Coze 导出格式范围：支持哪些节点类型？不支持时失败策略（整单拒 vs 标红不可发布）——须 **写死一种**  
3. ~~定时执行基础设施~~ → **已决后置链 B**（J1.5=Later）；进程内/Redis/外部 cron 实现时再定  
4. 业务 Chat 样板选哪一个先做（请假审批 vs 考勤总结）？  
5. ~~测试 FE vs 工作台~~ → **已决**：同仓 `/app` `/admin` `/dev`（§12）  
6. 本文与 ROADMAP 冲突：改 ROADMAP，还是本文仅作「试点加急附录」？  
7. ~~人机凭证 / 挂起~~ → **已决，§9**（含 J0.5a/b 拆分）  
8. JWT 是否上 refresh？（§9 倾向短 access + 重登）  
9. 兼容窗里程碑？（默认：关窗 = Wave 2B / 7B）  
10. ~~组织 A vs B~~ → **已决 B，§11**；质量安全优先于工期  
11. ~~业务角色与挂起路由~~ → **已决**：第一批 **`member` / `dept_manager`**；`dept_operator` **Later**（未定义权限前不进求值器）；挂起先本部门 `dept_manager`，找不到再升级 `tenant_admin` 并审计（§11.3）  
12. ~~编排保存 vs 运行鉴权~~ → **已决 (a)**：保存=**可见**；运行=**可调**/二次鉴权（§9.4）；否则金线挂不起  

---

## 7. 建议下一步

1. Master plan（已恢复并锁定）→ [`docs/superpowers/plans/2026-08-05-pilot-b-master-plan.md`](../plans/2026-08-05-pilot-b-master-plan.md)  
   Review 吸收对照 → [`../plans/2026-08-05-pilot-b-master-plan-review.md`](../plans/2026-08-05-pilot-b-master-plan-review.md)  
2. ~~确认 master plan / §4~~ → **已锁定 + §4 已补全**  
3. ~~拆 tasks~~ → **Task 40 链 A 已细拆**（`tasks/40-pilot-b-chain-a.md`）；**7A 前不拆 2B**  
4. 回写 ROADMAP（可选，与本文冲突处以本文为准） 

---

## 8. 修订记录

| 日期 | 变更 |
|------|------|
| 2026-08-05 | 初稿：B 档 + 2+1 结构 + Workflow P0 + Coze C(先 A) + 业务卡 P1 + 旅程主表；§4 占位 |
| 2026-08-05 | 并入 **§9 凭证拆分**（签核） |
| 2026-08-05 | §9 澄清：机侧不进 Chat，直连 Workflow Runner |
| 2026-08-05 | 并入 **§10**（签核）：T*/B*/S1–S4；附录 C |
| 2026-08-05 | 并入 **§11 组织 B** + **§12 前端 UX**（签核）：质量安全优先于工期；三壳；Chat 两阶段；J0.7 |
| 2026-08-05 | **Feedback 吸收**：J0.5→a/b；machine `acting_user=created_by`；T1 写死 IR 快照+写幂等；M2→M2a/M2b；挂起通知统一 §11.3；J1.5/定时后置；S3 覆盖查询面；J2 不挡 7A；§4 补全；恢复 master plan |
| 2026-08-05 | **Tasks feedback**：RAG 写入打标+NULL 策略；W3 二次鉴权 fail-closed；memory 部门维 N/A；login 禁建 api_keys |
| 2026-08-05 | **并发 §10.4 / T9 / S5**：乐观锁、run CAS、审批实时鉴权、申请 UNIQUE；Task 40 AC + [`../plans/2026-08-05-pilot-b-concurrency.md`](../plans/2026-08-05-pilot-b-concurrency.md) |
| 2026-08-05 | **金线闭合 D10**：保存=可见/运行=可调；cancelled 移出 MVP；dept_operator Later；param_spec；DELETE draft；挂起超时升级；resume 重鉴权；runs 分页；audit 禁 params 明文 |
| 2026-08-05 | **D11**：approve 挂权 = 短命 delegation（caps=节点 permission），不改长期权限表 |

---

## 9. 凭证拆分设计（已签核 · 2026-08-05）

> Brainstorm 结论：方案 **双 Depends 并列**；边界 **人点运行用 JWT，链内/出站分票**；身份 **混合**；过渡 **语义重标 `credential_type`**；不足可申请则 **挂起等批**。  
> 实现排期见 master plan；本文为设计事实源。

### 9.1 问题现状

- 人机共用 `X-API-Key`；密码登录（Task 38）只是「取 `cg_` 的门」，FE 把长期 key 当会话用。  
- 与「人会话短命可吊销 / 机器凭证可轮换可收窄」的治理叙事冲突。  
- ROADMAP「架构深化」已点名 `credential_type`，此前无设计正文。

### 9.2 身份模型（§1）

统一下游仍是 `TenantContext`；变的是进法与 `credential_kind`。  
**执行面分流（重要）：** Chat 管线只服务**人侧模糊需求**；机器与「确定动作」一律进 **Workflow Runner**（自研编排 / Coze 导入 IR / 后期可视化），**不进** `_run_chat_pipeline`。

```text
人（浏览器 / 产品 FE）
  登录 → JWT 会话 · Authorization: Bearer
  verify_session → TenantContext
       credential_kind = human_session
       acting_user_id  = sub
  然后分流：
       · Chat / SSE     → LangGraph chat DAG（模糊）
       · 工作台「运行」 → Workflow Runner（确定动作，与机侧同源）

机（外部系统 / 定时 / 集成）
  X-API-Key: cg_…（仅 credential_type=machine）
  verify_api_key → TenantContext
       credential_kind = machine_key
       acting_user_id  = **api_keys.created_by_user_id**（设置人；写死，禁客户端自报）
       key_id          = 本把 machine key
  → **直接** Workflow Runner / capability 节点执行
  → **不进** Chat 管线
  → 审计同时记：`key_id` + `acting_user_id`（=设置人）+ `credential_kind`

链内子调用（Runner 内 Hub / agent）
  运行时签发短命 delegation（不进浏览器）
       credential_kind = delegation
       acting_user_id  = 触发本次 run 的主体
                         · 人启：JWT `sub`
                         · 机启：**该 machine key 的设置人**（同上，不另发明主体）
       caps ⊆ 可见/可调集合（上限，非放行本身）

出站 OA / 财务等
  连接器专用 machine key（服务端持有）
  审计: connector_key_id + acting_user_id（仍为上述 run 主体）
```

**硬规则**

1. 人主路径不发、不长期持有 `cg_`；机器 Key 只在管理台创建/轮换，**强制绑定设置人 `created_by_user_id`**（禁匿名 key）。  
2. `verify_api_key` 拒绝非 `machine`：**仅 Wave 2B / J0.5b** 收紧为 401；Wave 2A 兼容窗仍可通旧 key（见 §9.6）。  
3. 审计能回答：谁点的 / 机侧谁设的这把 key + 哪类票 + 出站哪把连接器 key。  
4. Delegation ≠ 出站凭证；连接器 key ≠ 浏览器会话。  
5. **Machine run 的 `acting_user` = key 的设置人**；二次鉴权与 OrgScope 相对该用户，不相对「匿名机器」。

### 9.3 令牌形态（§2）

| 种类 | 传递 | 要点 |
|------|------|------|
| 人会话 JWT | `Authorization: Bearer` | claims：`sub`/`tid`/`role`/`jti`/`exp`；第一刀 HS256 + 短 access，可不做 refresh |
| 机器 Key | `X-API-Key` | 只存 SHA256；`credential_type=machine`；**必填 `created_by_user_id`（设置人）**；存量在 2B 回填或作废重发 |
| Delegation | 进程内 / 内部头 | 分钟级、绑单次 run；含 `acting_user_id` + `caps`；**挂起 approve 后签发**：`caps`=该节点 permission（D11） |
| 连接器 Key | 仅服务端 | 独立 machine 凭证库；同样可追溯创建者 |

登录：`POST /api/auth/login|register` **改发 JWT**（Wave **2A**），不再给人轮换长期 `cg_`。  
审计字段最低集（机侧 run）：`key_id` · `created_by_user_id` / `acting_user_id` · `tenant_id` · `credential_kind` · `run_id`。

### 9.4 二次鉴权 + 挂起等批（§2.5）

对内子调用与出站 **不是有票就能跑**：

```text
票有效？ → 二次鉴权（相对 acting_user）
  · 有权 → 执行
  · 无权且不可申请 → 驳回（节点/链失败）
  · 无权但可申请 → 不执行；run.status=suspended
                    写 permission_request(run_id, node_id, …)
                    通知按 §11.3：本部门 dept_manager → 找不到才升级 tenant_admin（审计原因）
                    approve → **签发短命 delegation**
                         （caps=该节点 spec.permission；acting_user=申请人；绑 run/node；不进浏览器）
                         → resume 自该 node_id；二次鉴权仍走本闸，caps 仅上限
                    reject → 不签发 delegation；节点失败 + 审计原因
```

| 调用 | 校验对象 | 相对谁 |
|------|----------|--------|
| 对内 Hub/agent | capability `spec.permission` + 租户可见性 | `acting_user` |
| 出站连接器 | 连接器/对象·行级范围 | 同上；connector key 只证明通道 |

Delegation 的 `caps` 是上限；真正放行仍过 RBAC / `user_app_perms`。  
不可申请的敏感能力（跨租户、密钥管理等）只驳回、不进申请队列。  
机器 Key / delegation **不能**绕过 acting_user 二次鉴权。  
**安全优先：** 挂起等批上线前必须满足 §10 **S1–S4**；否则该路径只允许「无权 → 失败」，不允许申请提权。

### 9.5 路由挂载（§3）— 方案：双 Depends 并列

| 入口 | Depends | 说明 |
|------|---------|------|
| 人侧 Chat / SSE | `verify_session` + 权限工厂 | **仅人**；模糊需求走 LangGraph chat DAG |
| 人侧工作台/编排「运行」 | `verify_session` | 进 **Workflow Runner**（与机侧同源执行面） |
| 机侧（外部 run/webhook/定时） | `verify_api_key`（仅 machine） | **直连 Workflow Runner**；**禁止**当 Chat 会话用 |
| 过渡双接受 | 优先 Bearer，否则 machine；打 metrics | 兼容窗结束后人侧只留 JWT |
| Runner 内部节点 | delegation + `RunContext` | 每步 §9.4 |
| 出站 | 服务端 connector key + acting_user | 同 §9.4 |

`require_permission` 底层依赖「已解析的 `TenantContext`」，不写死只从 API Key 来。  
Hub 动态权限保留；二次鉴权 + 挂起成为 **Workflow Runner** 节点必经。  
编排定义来源（自研 / Coze 导入 / 后期画布）只影响 IR，**不**再分叉第二套执行引擎。

### 9.6 迁移步骤与阶段衔接（已拆 2A / 2B）

| 步 | 内容 | Wave / 挂接 |
|----|------|-------------|
| M1 | 表字段 `credential_type` / `created_by_user_id`（可空先加列）；上下文 `credential_kind`；request 增 run/node | Wave 0；**不回填、不收紧** |
| **M2a** | `verify_session`；登录/注册 **发 JWT**；人侧 FE 切 Bearer | Wave **2A** · **J0.5a**；**不**收紧 `verify_api_key` |
| M3 | 路由分类；过渡双接受（优先 Bearer，否则旧 key 仍可通并打 metrics） | 工作台/编排前；仍属链 A |
| M4 | 二次鉴权 + 挂起等批 + resume（通知按 §11.3） | Wave 4；须先 S1–S4 |
| M5 | 连接器密钥不出浏览器 | 与 L2 出站 |
| **M2b / M6** | 存量回填或重发 machine；`created_by` 强制；**收紧** `verify_api_key`（非 machine→401）；关兼容窗；改文档 | Wave **2B**（**仅 7A 后**）· **J0.5b** / 7B |

- **禁止**把「发 JWT」和「收紧 api_key」绑在同一步实现（旧 M2 已废弃）。  
- **7A** 只验收 M1 + M2a + M3（+ Runner/挂起/UI 金线）；**不**验收旧 key 401。  
- Runner 设计约束（亦见 §10 T1）：run 启动时固化 **IR 快照**；写节点幂等键 `(run_id, node_id)`（或等价）。

**本设计明确不做：** 完整 IdP/SSO（可后接 RS256）；第一刀复杂 refresh；delegation 暴露给浏览器；平行第二套执行引擎。

### 9.7 旅程挂接

- **J0.5a** ← §9.2–9.3、9.5–9.6（M1、**M2a**、M3）— **7A Must**  
- **J0.5b** ← §9.6（**M2b/M6**）+ machine `created_by` / acting_user — **7B Must**  
- **J0.6** ← §9.4、9.6（M4）；**须先满足 §10 S1–S4**；通知路由 = §11.3  
- 执行/编排路径（J1.2/J1.4）依赖 J0.5a/J0.6；定时 J1.5 = 链 B  
- 技术/业务加强与全景债 → **§10**；排期 → master plan

---

## 10. 技术 / 业务要加强（已签核 · 2026-08-05）

> 用途：**D = 试点排期 + 架构风险**；时间窗 = **试点前为主 + 附录 C 全景**。  
> 组织：双栏风险登记；与 J* / §9 互链。面试抽讲见 `learning/00-interview-map.md`。

### 10.1 试点前 · 技术

| ID | 风险 / 缺口 | 为何要紧 | 缓解（方向） | 阻塞试点? | 挂接 |
|----|-------------|----------|--------------|-----------|------|
| T1 | **没有一等公民 Workflow Runner** | 图上机/人「运行」都依赖它；现在只有 chat DAG + Hub invoke | 最小 Runner：**IR + run 状态机 + 历史**；**run 启动固化 `ir_snapshot`**（定义后改不影响在飞 run）；**写节点幂等键 `(run_id, node_id)`**（重试不双写）；Coze 导入映射同一 IR | **是** | J1.1/1.4, J2.3；J1.5=Later |
| T2 | **人机凭证仍单钥匙** | 浏览器长期持 `cg_`；无法证伪人/机分轨 | **分阶段**：M2a=JWT（J0.5a）；M2b=收紧+`created_by`+acting_user=设置人（J0.5b） | **是**（7A 至少 J0.5a） | J0.5a/b, §9 |
| T3 | **二次鉴权 / 挂起未贯通 Runner** | 「权限内编排」不可证伪 | 节点强制 acting_user 鉴权；可申请 → suspended + resume（**先过 S1–S4**） | **是** | J0.6, J1.2, S* |
| T4 | **Chat ↔ Runner 边界不清** | 机器误打 `/chat` → 成本/审计乱 | Chat 仅 session；run API 显式；FE 禁机走 Chat | **是** | §9.5 |
| T5 | **Hub/Runner 护栏深浅不一** | 导入/外发可能绕过主管线护栏 | LLM/外发必须过统一治理入口，禁裸 SDK | **是** | J0.3 |
| T6 | **「跑挂了去哪看」弱** | 试点运维失败 | run_id 贯穿 audit + LangFuse；运行历史/错误态 | **是** | J0.4, J5.3 |
| T7 | **Coze→IR 语义缺口** | 不支持节点 silently 降级 = 假成功 | **整单拒绝**（不支持节点不 silent 降级；落权标红另属 J2.2） | **是**（若走 Coze A；不挡 7A） | J2.1/2.2 |
| T8 | **早预处理 / exact cache 债** | 拖可信度与成本故事 | Task 39 并行；未对齐前勿吹命中率 | **否** | Task 39 |
| **T9** | **编排/Run/审批并发与一致性** | 丢更新、双 run、双批、快照越权窗 → 链 B 必返工 | 见 **§10.4**；乐观锁 + run CAS + 审批实时鉴权 + 申请 UNIQUE | **是**（链 A 内做） | Task 40；[`../plans/2026-08-05-pilot-b-concurrency.md`](../plans/2026-08-05-pilot-b-concurrency.md) |

**技术侧一句话：** 试点缺的是 **Runner + 凭证分轨 + 节点鉴权（安全红线内）+ 入口纪律 + 护栏/可观测对齐 + 并发正确性**；Chat DAG 只服务模糊需求。

### 10.2 试点前 · 业务 + 安全红线

| ID | 风险 / 缺口 | 为何要紧 | 缓解（方向） | 阻塞试点? | 挂接 |
|----|-------------|----------|--------------|-----------|------|
| B1 | **谁编谁跑权责不清** | 国企推不动 | 有权可编草稿；跨权/生产链须 admin 发布；跑时 acting_user 鉴权 | **是** | J1.2, J1.6 |
| B2 | **缺可演示办公样板链** | 空 Runner 无感觉 | 样板≥1 + Dev 假数据 + seed | **是** | J5.2, J3.* 可缩 |
| B3 | **Coze vs 自研对外说法** | 被问「是不是套 Coze」 | 对外：治理+我方执行；Coze=导入源；第一期只做 A | **是** | §1.2 |
| B4 | **挂起等批 SLA 未定义** | 挂住无人批 | 通知 §11.3；**超时 → 自动升级 tenant_admin**（复用 escalate）+ 可观测时长；待办可点；**无主动推送**（试点只 inbox）— **且先满足 S*** | **是**（若启用挂起） | J0.6；40.43 |
| B5 | **Chat / Workflow 产品边界糊** | 凡事丢 Chat、成本失控 | Chat=问/总结/轻卡；确定动作进工作台/WF | **是** | T4 |
| B6 | **审计演示剧本缺失** | auditor 卖点演不出 | 跑链 → audit 含 acting_user + credential_kind → 导出 | **是** | J0.2 |
| B7 | **小模型切换未产品化** | 私有化叙事弱 | 租户默认模型可切换 + 观测可区分 | Should | J4.* |
| B8 | **测得见：能力只在 curl** | 验收人不是开发 | Must ↔ 面板/卡 + journey | **是** | J5.* |

#### 安全红线（一票否决 — 要安全，还是要安全）

> 权限申请、权限与**数据安全**是硬门槛。红线未满足时：**不上挂起等批**，无权节点直接失败。

| ID | 红线 | 必须做到 | 阻塞? |
|----|------|----------|-------|
| **S1** | 权限申请不可伪造/冒领 | 申请人身份只来自已校验会话/机凭证，禁客户端乱填 `acting_user`；机侧 `acting_user`=key.`created_by`；approve 按 §11.3（本部门 `dept_manager`，升级路径才到 `tenant_admin`）；每步进 `audit_logs` | **是** |
| **S2** | 挂起等批不可成提权后门 | 仅 `requestable` 可申请；跨租户/密钥等不可申请；approve 后只发 **短命 delegation（caps=该节点 permission）**，**不**改长期 `user_app_perms`；后续节点仍逐节鉴权 | **是** |
| **S3** | 数据隔离不可破 | Runner/出站全程 `tenant_id` + **`OrgScope`**；**RAG 写入打标（主部门）+ 查询过滤**（存量 `org_unit_id` NULL 仅治理角色可见）；**Memory：本期无部门共享类型 → 部门维 OrgScope 过滤 N/A，跨用户/跨租户隔离不回退**；审计导出经 OrgScope；跨租户/跨部门默认拒绝；连接器行级跟 acting_user；禁客户端自报部门扩权 | **是** |
| **S4** | 票与人不可混用绕权 | delegation / machine key **不能**绕过 acting_user 鉴权；浏览器永不持有连接器密钥或 delegation | **是** |
| **S5** | 审批资格不可吃陈旧快照 | approve / 挂起路由资格 **实时**查业务角色与 OrgScope；D1 快照不得放行 | **是** |

**业务侧一句话：** 权责 + 样板 + 话术 + Chat/WF 边界 + 审计可演；其中 **S1–S5 优先于「好用的挂起」**。

### 10.4 并发与一致性（链 A Must · 避免链 B 返工）

> 详细对照表与取舍：[`../plans/2026-08-05-pilot-b-concurrency.md`](../plans/2026-08-05-pilot-b-concurrency.md)。实现 AC 写在 Task 40。

| # | 场景 | 策略（写死） | Task |
|---|------|--------------|------|
| 1 | 双 PATCH 同一草稿 | `base_version` 乐观锁；不匹配 → **409** | 40.31 |
| 2 | 编辑 vs 发布 | 发布原子固化 IR + `version+1`；**published 冻结**；再改开新 draft | 40.31 / 40.57 |
| 3 | 双 publish | 同 version CAS → 第二 **409** | 40.31 |
| 4 | 双击运行 | FE 防抖 + 1s 窗口 — **Optional / 不挡 7A**（锦上添花） | 40.36 / 40.52 |
| 5 | 同 run 双 execute/resume | `status` 原子迁移或 `FOR UPDATE`；仅一 worker 进入 `running` | 40.35 / 40.44 |
| 6 | 双 approve | `UPDATE WHERE status=pending`；第二 no-op/409 | 40.44 |
| 7 | approve vs reject | 单终态；另一返回已处理 | 40.44 |
| 8 | 批时角色已变 | **approve 时刻实时**校验业务角色 | 40.44；**S5** |
| 9 | 移/删部门 | 子树 path 原子更新；有依赖则拒或软删 | 40.10 / 40.11 |
| 10 | 换主+授角色 | **单事务** | 40.11 |
| 11 | org 快照 TTL | 快照**仅列表/展示**；安全判定**永远实时查**（修订 D1） | 40.12 / 40.44 |
| 12 | 重复挂起申请 | `UNIQUE(run_id, node_id)` | 40.42 |
| 13 | Alembic 撞号 | Wave 固定迁移编号，禁止并行自编 | Task 40 全局 |

**D1 修订:** 短 TTL org/run 快照用于**展示与减 N+1**；**不得**用于 approve / 二次鉴权放行 / 挂起路由资格。

### 10.3 附录 C · 目标架构全景债

| ID | 类型 | 债 | 阻塞试点? |
|----|------|-----|-----------|
| C1 | 技术 | 完整 refresh / IdP·SSO（RS256） | 否 |
| C2 | 技术 | 受限拖拽画布 | 否（步骤/预览即可） |
| C3 | 技术 | Prompt 版本回滚、SSE 续传、流式 failover 对齐 | 否 |
| C4 | 技术 | 凭证 M2b/M6 关窗（J0.5b） | **否挡 7A**：7A 验 M2a；关窗在 2B/7B |
| C5 | 技术 | 数据不出域套件 / 等保正式包 | 否（B 档不要求） |
| C6 | 业务 | 真钉钉写回、全量 OA | 否 |
| C7 | 业务 | 治理大盘 / 业务 KPI | 否（run 历史+audit 即可） |
| C8 | 业务 | Chat 多业务卡铺开 | 否（P1） |
| C9 | 安全 | 完整轮换/jti 黑名单/渗透包 | **部分**：S1–S4+审计要；渗透可后置 |

**附录一句话：** 远景可以慢；**S1–S4 + OrgScope + Runner + 分轨入口** 不能慢。

---

## 11. 组织与角色模型 B（已签核 · 2026-08-05）

> 原则：**全面 + 安全**；不为赶工采用扁平-only。IdP 同步仍属后置（预留 `external_org_id`）。

### 11.1 两套角色分离

| 种类 | 职责 | 例子 |
|------|------|------|
| **平台角色 `platform_role`** | 进哪个壳、治理能力（现有 4 个） | `user` / `tenant_admin` / `auditor` / `super_admin` |
| **业务角色 `business_role`** | 部门内职责、审批路由、数据范围 | 第一批：`member` / `dept_manager`；**`dept_operator` = Later**（权限未定前勿授） |

禁止用「人人升 tenant_admin」模拟部门经理。  
例：`platform_role=user` + `business_role=dept_manager@财务部` → 默认 `/app`，可批本部门挂起。

### 11.2 组织对象

```text
tenant
  └── org_unit（树：parent_id + path，便于子树查询）
        └── membership（user ↔ org_unit，is_primary，business_roles[]）

user（display_name，employee_no，…）— 经 membership 挂到一个或多个 org_unit
```

- 允许兼职；**主部门**用于顶栏展示与默认审批路由。  
- 所有列表 / Runner / 出站过滤经唯一门面 **`OrgScope`**（禁止散落 `department_id ==`）。

### 11.3 权限叠加顺序

```text
放行 =
  平台权限包(platform_role)
  ∪ extra_permissions（显式加挂，须审计）
  ∪ 业务角色权限（仅在其 org_scope 内）

且必须同时满足 OrgScope（租户 → 部门子树 → 行级）
```

**硬规则（叠 S1–S4）**

1. 客户端不可自填部门 / 业务角色 / `acting_user`。  
2. 跨部门默认拒绝；显式授权或平台高权除外。  
3. 挂起路由：按业务角色（如本部门 `dept_manager`）→ 找不到则升级 `tenant_admin` 并审计原因。  
4. 敏感能力（跨租户、密钥、连接器写）不可申请。  
5. 不把业务角色塞进单一 `api_keys.role` 字符串凑合——拆表/字段。

### 11.4 与壳的关系

| platform_role | 默认壳 | 组织相关 UI |
|---------------|--------|-------------|
| user | `/app` | 顶栏显示名·主部门；兼职只读 |
| tenant_admin | `/admin` | 组织树 CRUD、调岗、业务角色授予（全审计） |
| auditor | `/admin` 审计向 | 跨部门只读审计；跨租户仅 auditor/super_admin |
| super_admin | `/admin` | 全开 |

### 11.5 本档明确不做

- 完整 IdP/AD 自动同步（预留外部 ID）  
- 编制/HC、薪资权等 HR 深水区  

挂接：**J0.7**；安全 **S3**；UX **§12**。

---

## 12. 前端 UX（已签核 · 2026-08-05）

> 同仓 Vite App 三壳；业务编排 = 模板+表单；管理台 = 步骤列表+只读预览；画布后置。  
> Chat：**试点二级**；**后期**可升办公桌面（执行仍 Runner）。

### 12.1 信息架构与导航

```text
/login · /register  → JWT（§9）
/app/*   业务工作台（默认 user）
/admin/* 管理台（默认 tenant_admin / auditor / super_admin）
/dev/*   原 QA 面板（journeys / 安全负向；不进对外叙事）
```

| 壳 | 主导航（试点） | 默认首页 |
|----|----------------|----------|
| `/app` | 工作台 · 我的流程 · 运行历史 · 待办 · Chat（二级） | 大按钮样板 + 待办 + 最近运行 |
| `/admin` | 概览 · 组织 · 流程 · Coze 导入 · 权限/挂起 · 审计 · 集成 Key · 模型 | 待批/失败 run/导入健康/越权拒绝 |
| `/dev` | 现有各面板 | 测试控制台 |

`tenant_admin` 可进 `/app` 以便全套测流。产品壳不用四槽 API Key；`/dev` 可保留角色切换。

### 12.2 关键旅程页面

**`/app`**

| 页面 | 要点 |
|------|------|
| 工作台 | 大按钮；部门范围可见；无权禁用+原因；待办分「待我批 / 我发起的挂起」 |
| 我的流程 | 模板+表单；节点仅 OrgScope+权限内；保存服务端再校验；默认主部门 |
| 运行 / 历史 | 确认页展示身份与部门范围；结果含节点日志；历史过滤走 OrgScope |
| 待办 | 仅路由到本人的单；批/驳写原因；S1–S4 |
| Chat | 二级；后期卡片触发 Runner |

**`/admin`**

| 页面 | 要点 |
|------|------|
| 组织 | 树 CRUD、兼岗、业务角色；调岗/授角审计 |
| 流程 | 步骤列表 + 只读预览；发布闸；无权节点标红 |
| Coze 导入 | 校验落权；失败策略写死 |
| 权限/挂起 | 治理申请 vs 链内挂起分流 |
| 审计 | acting_user + org + credential_kind；导出 |
| 集成 Key / 模型 | 人机凭证分离；明文一次性 |

**金线（测得见）**

```text
admin 建部门树与经理角色
  → user 模板表单存部门草稿 → admin 发布
  → user 工作台运行 → 缺权挂起 → dept_manager 待办批过
  → resume 完成 → auditor 导出全程
```

### 12.3 好用原则

| 原则 | 做法 |
|------|------|
| 一屏一主任务 | `/app` 低密度；`/admin` 可密，危险操作分步确认 |
| 空态可行动 | 说明原因 + 下一步（找 admin / 去模板） |
| 错误可行动 | 缺权说明是否可申请、找谁；禁止模糊 Failed |
| 挂起可理解 | 状态、审批角色/部门、「待我批」与「我发起的」分开 |
| 安全可见 | 越权有审计；文案不泄露其他租户/部门存在性 |
| 表单先于画布 | 业务 A；管理台 B；拖拽后置 |
| Chat 两阶段 | 试点二级；后期办公桌面仍调 Runner |
| OrgScope 唯一 | UI 不可自报部门扩权 |

**UX 明确不做：** 假大屏、无权限自由画布、产品壳四槽 Key、业务壳堆满 QA 面板。
