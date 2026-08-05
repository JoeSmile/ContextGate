# 试点 B master plan — Review 调整单

> 对象:[`2026-08-05-pilot-b-master-plan.md`](2026-08-05-pilot-b-master-plan.md)  
> Spec:[`../specs/2026-08-05-enterprise-pilot-b-gaps-design.md`](../specs/2026-08-05-enterprise-pilot-b-gaps-design.md)  
> 日期:2026-08-05 · 状态:**已吸收（含第二轮 feedback P0–P2）**  
> 拍板摘要：Coze∈链A且不挡7A；D8-a=`created_by` + **machine run acting_user=设置人**；2B=链B且仅7A后开；J0.5 拆 a/b。

---

## 0. 结论先行

1. **骨架认可**:批次依赖图、S1–S4 硬闸、测得见约束方向正确。
2. **重大调整:链 A(人侧)先行**——JWT → 工作台「运行」→ Runner → 挂起；机器侧(链 B)=2B 后置。
3. **第二轮 feedback**:J0.5 验收点空转、acting_user 未写死、IR/幂等只在已删 plan、M2 未拆、通知路由打架、定时/J2 Must 与后置矛盾、§4 占位、plan 缺失 —— **已全部回写 spec + 恢复 master plan**。

## 吸收对照（含第二轮）

| 项 | 落点 |
|----|------|
| P0-1 J0.5 旧 key 401 测不到 | Spec **J0.5a/b**；7A 只验 a；7B 验 b |
| P0-2 machine acting_user | Spec §9.2/9.3：**acting_user = created_by**；审计 key_id+设置人 |
| P0-3 IR 快照 + 写幂等 | Spec **T1** + §9.6 + master Wave 3 |
| P1-4 §9.6 M2 | 拆 **M2a / M2b** |
| P1-5 通知路由 | §9.4 / B4 / S1 统一 **§11.3** |
| P1-6 定时 vs §1.4 | §1.4 仅按钮；**J1.5=Later 链B** |
| P1-7 S3 查询面 | S3 补 RAG/memory/audit 导出 |
| P1-8 J2 Must vs 不挡 7A | J2=**Must（7A 后第一批）** |
| P2-9 D8-a 正文 | §9.2/9.3 字段清单 |
| P2-10 master plan 缺失 | **已恢复** `plans/2026-08-05-pilot-b-master-plan.md` |
| P2-11 §4 待补 | **已补全**；拆 tasks 强制前置 |
| **Tasks feedback-1** | RAG 写入打标 + NULL 策略 | Task **40.14**（files/rag 上传清单） |
| **Tasks feedback-2** | W3 二次鉴权非空钩子 | Task **40.35** AC；W4 **40.41** 只扩展 suspend |
| **Tasks feedback-3** | Memory hook 空转 | **40.15** 部门维 N/A；隔离测不回退 |
| **Tasks feedback-4** | login 是否建 key | **40.22** 写死：login/register **零** api_keys INSERT |
| **Concurrency** | A–E 13 项 | spec **§10.4 / T9 / S5** + concurrency.md；#4 双击降 Optional |
| **Gold-line fix** | 挂起不可达 | **D10**：保存=可见 / 运行=可调；40.31/51/70 对齐 |
| **MVP hygiene** | cancelled / dept_operator / params / DELETE / timeout / resume re-auth / pagination / audit params | 已写入 Task 40 对应 AC |

## 仍待拍板

无。暂不拆 tasks；等主人一声从 Wave 0 起。

---

## 下方为原始 review 正文（归档保留）

### 1. A 链 / B 链定义

| | 链 A(人)· 先做 | 链 B(机)· 后置 |
|--|------------------|------------------|
| 凭证 | 登录 → JWT · `Authorization: Bearer` | X-API-Key(仅 machine) |
| Depends | `verify_session` → TenantContext(human_session) | `verify_api_key` → TenantContext(machine_key) |
| 入口 | Chat(模糊)∥ 工作台「运行」(确定) | run API / 定时 / webhook → **直连 Runner** |
| 禁止 | 浏览器长期持 cg_ | 机器进 Chat DAG |
| J0.5 | **a** JWT 可用 | **b** 旧 key 401 |

### 2. 为什么 A 链先行

- **化解测试基建崩塌**:原 Wave 2 把"login 改发 JWT"与"verify_api_key 收紧仅 machine"绑在一起 → 现有测试全体 401。拆开(2A/2B)后:2A 期间不收紧。
- **产品面先被证明**:人侧工作台 + 审批是国企真正摸到的界面。
- **验收叙事干净**:7A 验 J0.5a；7B 验 J0.5b。

### 3–7 原文要点

详见本文件 git 历史；执行以 **spec + master plan 现行正文** 为准。
