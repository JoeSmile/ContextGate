# Task 40: 试点 B · 链 A（人侧金线 → 7A）

> **状态:** 已细拆，待实现（按 Wave 顺序；可多 agent 同 Wave 内并行子项）  
> **规格:** [`docs/superpowers/specs/2026-08-05-enterprise-pilot-b-gaps-design.md`](../docs/superpowers/specs/2026-08-05-enterprise-pilot-b-gaps-design.md)  
> **计划:** [`docs/superpowers/plans/2026-08-05-pilot-b-master-plan.md`](../docs/superpowers/plans/2026-08-05-pilot-b-master-plan.md)  
> **原则:** 质量与安全 > 工期；先链 A 后链 B；**7A 前不拆 / 不做 2B**

## 目标（一句话）

打通：**登录 JWT（J0.5a）→ OrgScope + 求值器 → Workflow Runner（IR 快照 + 写幂等）→ S\*后挂起（dept_manager）→ `/app`+`/admin` 可点 → 7A 金线剧本**。

## 明确不做（本 Task）

- Wave **2B / 7B**（旧 key 401、回填关窗）  
- **J1.5 定时**（链 B）  
- 业务假数据 seed、真钉钉写回、自由画布  
- 把 JWT 与收紧 `verify_api_key` 绑在同一步  

## 依赖图

```text
40.0x W0 ──► 40.1x W1 ──► 40.2x W2A ──► 40.3x W3 ──► 40.4x W4
                                                         │
                     40.5a / 40.5b ◄──────────────────────┘
                         │
                     40.7x W7A ★
                         │
              40.6x W6（可交错；不挡 7A）
```

`40.5c`（`/dev`）可与 5a/5b 并行。Task 39 不阻塞本队列。

## 子任务索引

| ID | Wave | 文件 | 测得见 / 挂接 |
|----|------|------|----------------|
| 40.01–40.04 | **0** | [`40/W0-credential-scaffold.md`](40/W0-credential-scaffold.md) | 列可空；旧路径全绿；契约预留 |
| 40.10–40.18 | **1** | [`40/W1-orgscope.md`](40/W1-orgscope.md) | J0.7；S3（RAG 写+读；memory 部门维 N/A）；求值器 |
| 40.20–40.26 | **2A** | [`40/W2A-jwt.md`](40/W2A-jwt.md) | **J0.5a**；login **不**建 key；不收紧 api_key |
| 40.30–40.38 | **3** | [`40/W3-runner.md`](40/W3-runner.md) | T1；J1.1/1.4 API；**无权→failed** |
| 40.40–40.46 | **4** | [`40/W4-hang-wait.md`](40/W4-hang-wait.md) | J0.6；S1–S4；§11.3 |
| 40.50–40.53 | **5a** | [`40/W5a-app-shell.md`](40/W5a-app-shell.md) | `/app` 工作台运行 |
| 40.55–40.58 | **5b** | [`40/W5b-admin-shell.md`](40/W5b-admin-shell.md) | `/admin` 组织/预览/待办 |
| 40.59 | **5c** | [`40/W5c-dev-shell.md`](40/W5c-dev-shell.md) | `/dev` 可后置并行 |
| 40.70–40.72 | **7A** | [`40/W7A-gold-line.md`](40/W7A-gold-line.md) | 人侧金线；**不验 J0.5b** |
| 40.60–40.63 | **6** | [`40/W6-coze.md`](40/W6-coze.md) | J2；**不挡 7A** |

## 全局约束（每个子任务默认继承）

1. Runner / 契约字段预留 `credential_kind`，实现不写死「仅 JWT」。  
2. 测得见分层：W3/W4 = API+pytest；UI 金线 = W5+W7A。  
3. 结构 seed 允许；业务域假数据 seed 禁止。  
4. 挂起上线前必须 S1–S5 可测。  
5. 通知路由：本部门 `dept_manager` → 找不到再 `tenant_admin`（审计原因）。  
6. **并发与一致性**（§10.4）：乐观锁 / run CAS / 审批实时鉴权 — 见 [`docs/superpowers/plans/2026-08-05-pilot-b-concurrency.md`](../docs/superpowers/plans/2026-08-05-pilot-b-concurrency.md)；**链 B 不得另起一套状态机**。  
7. Commit: Conventional + `Signed-off-by: Joe`；实现后 code review。

### Alembic 编号（固定 · 防多 agent 撞车）

| Wave | 编号 | 文件意图 |
|------|------|----------|
| W0 | **007** | `007_credential_scaffold.py`（含 audit 可选列则可同文件） |
| W1 | **008** | `008_org_b.py` |
| W1 | **009** | `009_rag_org_unit.py`（文档/chunk `org_unit_id`） |
| W3 | **010** | `010_workflows.py` |
| W3 | **011** | `011_workflow_runs.py` |
| W4 | **012** | `012_hang_wait.py`（permission_request + UNIQUE） |

禁止任务正文再写 `00N` 占位；若需加列，在对应 Wave 号上 append revision，勿抢下一 Wave 号。

## 建议开工顺序

先做 **40.01**（W0 迁移 007），做完一个 Wave 再开下一 Wave；同 Wave 内按文件内编号顺序。
