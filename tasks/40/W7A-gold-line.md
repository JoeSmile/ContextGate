# Task 40 · Wave 7A — 人侧金线验收

> **状态:** 待实现  
> **依赖:** W0–W4 + W5a + W5b（W5c/W6 可选）  
> **挂接:** §1.4；master 7A；验 **J0.5a**，**不验 J0.5b**；**D10 挂起语义 (a)**  
> **非目标:** 旧 key 401；定时；机侧 run；主动推送通知

---

## 40.70 — Journey 剧本（可重复执行）

**Files:**
- Create: `examples/qa/journeys/pilot_b_7a_gold_line.md`
- Create: `examples/qa/journeys/pilot_b_7a_gold_line.sh`（或 pytest e2e）

**挂起何以可达（D10）:**  
保存/UI 只要求 capability **可见**；节点标 `requestable`；user **没有**对应 `spec.permission` → 运行时二次鉴权 → **suspended**。  
禁止用「保存时可见可调」或「UI 藏掉缺权节点」堵死本剧本。

**剧本（必须逐步可勾）：**

```text
1. seed 组织结构（40.17）
2. tenant_admin 登录（JWT）→ /admin 建部门树 + 授 dept_manager
3. 准备「可见但 user 不可调」且 requestable 的 capability（seed/配置）
4. user 登录 → /app 选含该节点的模板 → 填参（param_spec）→ 保存草稿（须成功）
5. admin 发布
6. user 运行 → 缺权节点 → suspended（非 failed）
7. dept_manager 在 /app「待我批」批准 → resume
8. resume 后后续节点重新二次鉴权 → 全通过则 succeeded
9. auditor 导出：acting_user + credential_kind=human_session + run_id；params 无明文
10. 确认：登录发 JWT；Bearer 全程；产品壳无长期 cg_ 依赖
```

**禁止出现在本剧本：**
- 人带旧 key 期望 401  
- 定时触发  
- machine X-API-Key 主路径  
- 靠「发布后撤权」才挂起（那是备选 b，本金线不用）

- [ ] **Step 1:** 写 Markdown 剧本 + 自动化/半自动脚本
- [ ] **Step 2:** 本地跑通，证据入 `examples/qa/journeys/evidence/7a/`
- [ ] **Step 3:** Commit `test(qa): pilot B 7A gold-line journey`

---

## 40.71 — 验收清单（对照表）

| 项 | 期望 | 结果 |
|----|------|------|
| J0.5a | JWT 登录 + Bearer 跑通；login 不建 api_keys | ☐ |
| J0.7 | 跨部门拒绝可证伪 | ☐ |
| J1.1/1.4 | 编链 + 按钮跑 + 历史（分页） | ☐ |
| J1.2 / D10 | 可见可存；运行缺权可挂起 | ☐ |
| J0.6 | 挂起 → dept_manager → resume → 后续节点重鉴权 | ☐ |
| S1–S5 | 负向测绿 | ☐ |
| 乐观锁 | 双 PATCH → 409 | ☐ |
| J0.2 | audit 可导出；params 非明文 | ☐ |
| 通知 | inbox 可点；**无**主动推送（已知） | ☐ |
| J1.5 定时 | **不在范围** | N/A |
| J0.5b | **不在范围** | N/A |
| J2 Coze | 不挡；未做则 skip | ☐/skip |

- [ ] **Step 1:** 跑完勾表
- [ ] **Step 2:** 缺口只允许 W6 skip；其余 Must 必须过

---

## 40.72 — 收口与交接链 B

**Files:**
- Modify: `tasks/README.md` — Task 40 链 A 标 **7A Done**
- Modify: `docs/superpowers/plans/2026-08-05-pilot-b-master-plan.md` — 可开 2B
- Modify: `learning/00-interview-map.md`（可选）

- [ ] **Step 1:** 文档状态翻转
- [ ] **Step 2:** Code review 总览（无 Important 未决）
- [ ] **Step 3:** **停止**。等用户一声再拆 Wave 2B / 7B

---

## Wave 7A Done 标准

- [ ] §1.4 + D10 挂起金线真实跑通  
- [ ] J0.5a 满足且未冒充验 J0.5b  
- [ ] 证据目录可复查
