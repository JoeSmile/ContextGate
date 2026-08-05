# 试点 B 主路径 — 批次级实现计划

> **For agentic workers:** 批次级 master plan。确认后再拆 `tasks/*`。  
> **状态: 已锁定** · **链 A tasks 已拆** → [`tasks/40-pilot-b-chain-a.md`](../../../tasks/40-pilot-b-chain-a.md)  
> Spec 事实源: [`../specs/2026-08-05-enterprise-pilot-b-gaps-design.md`](../specs/2026-08-05-enterprise-pilot-b-gaps-design.md)  
> Review: [`2026-08-05-pilot-b-master-plan-review.md`](2026-08-05-pilot-b-master-plan-review.md)

**Goal:** 先打通 **链 A**（人 JWT → Chat∥工作台 → Runner → RAG/连接器 → 挂起 → 审计）；**7A 验收后**再开 **链 B**（= Wave 2B）。质量与安全优先于工期。

**Architecture:** OrgScope（含查询面）· 契约双轨实现分阶段 · Runner + **IR 快照** + **写节点幂等** · S1–S4 后挂起 · Coze∈链 A 且不挡 7A。

## Global Constraints

- 先链 A 后链 B；Runner 不写死仅 JWT；`credential_kind` Wave 0 预留
- S1–S4 未满足不上挂起
- 定时/webhook = 链 B 后置（**非 7A Must**）
- 测得见分层：Wave 3/4 = API+单测；UI 金线 = 5a/5b+7A
- 结构 seed 允许；业务假数据 seed 禁止
- **拆 tasks 强制前置：** spec **§4 五层索引已补全**（见 spec）

## 链 A / 链 B

| | 链 A · 先 | 链 B · 后（=2B 起） |
|--|-----------|---------------------|
| 凭证 | JWT | machine Key（收紧+回填） |
| 入口 | Chat∥工作台运行 | run API / 定时 / webhook |
| 完成 | **7A** | **7B** |
| J0.5 | **J0.5a** JWT 可用（7A Must） | **J0.5b** 人带旧 key→401（2B/7B Must） |

**关键路径 A:** `0 → 1 → 2A → 3 → 4 → 5a → 5b → 7A`（6 可交错不挡 7A；5c 可并行）  
**路径 B:** **仅 7A 后** → `2B → 7B`

```text
W0 列先加不回填 → W1 OrgScope+求值器 → W2A JWT 纯增量
  → W3 Runner(IR快照+幂等) → W4 挂起(S*) → W5a/5b → W7A ★
  → W2B 回填关窗+D8-a → W7B
W6 Coze ∈ 链A，不挡 7A
```

## Wave 摘要

| Wave | 要点 |
|------|------|
| 0 | credential_* 列**不回填**；TenantContext；迁移编排说明 |
| 1 | 组织 B；OrgScope；权限求值器；**RAG 写入打标+查询过滤**；memory 部门维 N/A |
| 2A | JWT；不收紧 api_key；存量 cg_ 全通路 |
| 3 | Runner；**run.ir_snapshot**；写节点幂等；**二次鉴权真实接入（无权→failed+审计）**；API 验收 |
| 4 | S*+挂起；通知 **dept_manager→升级 tenant_admin** |
| 5a/5b/5c | /app · /admin · /dev |
| 6 | Coze 导入给人跑；**不挡 7A**；J2 = 链A加测/7A后第一批 |
| 7A | 人侧金线；验 **J0.5a** 非 J0.5b |
| 2B | =链B；回填+关窗；**acting_user=created_by**；created_by 强制 |
| 7B | 机侧金线；验 **J0.5b** 旧 key 401 |

## 决策闸（锁定）

| ID | 结论 |
|----|------|
| D1 | 短 TTL + org/run 快照防 N+1 — **仅列表/展示**；审批与二次鉴权 **实时查**（§10.4 #11 / S5） |
| D2 | A 不收紧；2B 回填+重签发+关窗 |
| D3 | Coze 整单拒绝 |
| D4 | 第一批 `member` / `dept_manager`；`dept_operator` Later |
| D5 | 顺序 MVP + IR 快照 + 写节点幂等 |
| D6 | Task 39 不阻塞 7A |
| D7 | 先 A 后 B；定时后置；2B 仅 7A 后 |
| D8-a | machine key 强制 `created_by_user_id`；**machine run 的 `acting_user_id` = 该设置人**；审计记 key_id+设置人；禁匿名 key；禁客户端自报冒充 |
| D9 | Coze∈链 A、不挡 7A；J2 Must 标注「7A 后第一批 / 加测」 |
| **D10** | **保存=可见 / 运行=可调**：挂起语义 (a)；金线可达 |
| **D11** | approve 挂权 = **短命 delegation**（`caps`=该节点 permission；不改 user_app_perms；§9.2/9.4） |

## 状态

计划已锁定。链 A 细拆见 **Task 40**（`tasks/40/W0`…`W7A`；W6 不挡 7A）。  
**链 B（2B/7B）仍不拆**，等 7A Done。开工从 **40.01**。
