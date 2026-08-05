# Task 40 · Wave 1 — 组织 B + OrgScope + 权限求值器

> **状态:** 待实现  
> **依赖:** Wave 0 Done  
> **挂接:** J0.7；§11；S3（含 RAG/memory/audit 查询面）；master D1/D4  
> **验证分层:** API + 负向单测（跨部门拒绝）；UI 后置 W5b  
> **非目标:** 业务假数据 seed；挂起等批（W4）

---

## 40.10 — Schema：部门树 + 兼岗

**Files:**
- Create: `alembic/versions/008_org_b.py`
- Create: `backend/core/org/models.py`（或 `backend/database/org_models.py`，与仓库习惯一致优先挂现有 Base）
- Modify: `backend/database/pgvector_session.py` / `models.py` 注册表

**表（最小）:**

```text
org_units(
  id, tenant_id, parent_id NULL, name, path,  -- path 如 /root/finance/
  deleted_at NULL,   -- 软删；有依赖时用
  created_at
)
org_memberships(
  id, tenant_id, user_id, org_unit_id,
  is_primary bool,
  business_roles JSON,   -- ["member"] | ["dept_manager"] | …
  created_at,
  UNIQUE(tenant_id, user_id, org_unit_id)
)
```

业务角色枚举第一批：**`member` | `dept_manager`**（D4）。  
`dept_operator` **不进第一批**（权限未定义；求值器勿留空角色）。

**并发 AC（§10.4 #9）:** schema 支持软删；path 反规范化由 40.11 原子维护。

- [ ] **Step 1:** 迁移 + ORM
- [ ] **Step 2:** 空表 upgrade 通过
- [ ] **Step 3:** Commit `feat(org): org_units and memberships schema`

---

## 40.11 — 组织 CRUD API（tenant_admin）

**Files:**
- Create: `backend/core/org/service.py`
- Create: `backend/routers/org.py`
- Modify: `backend/app.py` 挂载 `/api/org/...`
- Test: `tests/test_org_api.py`

**Endpoints（最小）:**
- `POST /api/org/units` — 建部门（需 `tenant_admin`）
- `PATCH /api/org/units/{id}/move` — 移父；**同事务**更新整棵子树 `path`
- `DELETE /api/org/units/{id}` — 若有 members / 已发布 workflow / 在飞 run → **409** 或软删（写死一种：有依赖 → 软删+禁用新建挂靠）
- `GET /api/org/units/tree` — 本租户树（可排除已软删）
- `POST /api/org/memberships` — 调岗/兼岗/设主部门/授业务角色（**换主+授角色单事务**，§10.4 #10）
- `GET /api/org/memberships/me` — 当前用户 memberships

全部 `Depends`：过渡期仍可用现有 `verify_api_key` / 后续 W2A 的 session；**不要**写死只 JWT。

**AC:**
- 移父后子节点 path 前缀正确（单测）
- 有在飞 run 的部门不可硬删
- membership 变更单请求内原子提交

- [ ] **Step 1:** 失败单测（无 admin → 403；有依赖删除 → 409/软删）
- [ ] **Step 2:** 实现 service + router（含 path 重写事务）
- [ ] **Step 3:** 绿；commit `feat(org): tenant_admin org CRUD API`

---

## 40.12 — `OrgScope` 唯一门面

**Files:**
- Create: `backend/core/org/scope.py`
- Test: `tests/test_org_scope.py`

**Produces:**
```python
@dataclass(frozen=True)
class OrgScope:
    tenant_id: str
    user_id: str
    primary_org_unit_id: str | None
    org_unit_ids: frozenset[str]       # 兼岗全部
    subtree_paths: frozenset[str]      # 经理可见子树 path 前缀
    business_roles: frozenset[str]

def resolve_org_scope(conn, *, tenant_id: str, user_id: str) -> OrgScope: ...
def visible_org_filter(scope: OrgScope) -> ...:  # 供 SQL/ORM 过滤的辅助
def assert_org_access(scope: OrgScope, resource_org_unit_id: str | None) -> None: ...
```

规则：
- `tenant_admin` / 跨租户角色：本租户（或跨租户策略保持现状）可见范围按平台角色，**仍经门面**，禁止散落 `if role ==`
- `dept_manager`：主/兼岗部门及其 **子树**
- `member`：仅自身兼岗部门（不含随意扩子树）
- 客户端 **不可** 传 `org_unit_id` 扩权；query 参数只作筛选且再经 `assert_org_access`
- **§10.4 #11 / S5:** 可选短 TTL 缓存仅用于**列表展示**；`assert_org_access` / 审批资格路径必须 `resolve_org_scope` **实时**（或 TTL=0）

- [ ] **Step 1:** 纯单测 fixture 树（财务/人事）覆盖经理可见/成员拒绝
- [ ] **Step 2:** 实现 `resolve_org_scope` + assert；文档注明缓存不得进安全判定
- [ ] **Step 3:** Commit `feat(org): OrgScope facade`

---

## 40.13 — 权限求值器（平台 ∪ extra ∪ 业务）

**Files:**
- Create: `backend/core/auth/evaluator.py`
- Modify: `backend/core/auth/models.py`（可选：`TenantContext.has_permission` 委托 evaluator）
- Test: `tests/test_permission_evaluator.py`

**Produces:**
```python
def evaluate_permission(
    *,
    platform_role: str,
    extra_permissions: list[str],
    business_roles: Iterable[str],
    needed: str,
    org_scope: OrgScope | None,
) -> bool: ...
```

- 平台包 = 现有 `ROLES`
- `extra_permissions` = `user_app_perms`
- 业务角色表驱动（第一批 **仅**）:
  - `dept_manager` → `workflow:approve_dept`（仅 org_scope 内）
  - `member` → 无审批权（运行权跟平台 `chat:*` / 显式 extra）
- **`dept_operator`：第一批不实现**；求值器遇到未知业务角色 → 忽略（不当作提权）
- 禁止「升 tenant_admin 冒充经理」
- **可见 vs 可调：** 本求值器管 **可调**；目录可见性另函数（既有 `capability_visible_to`）

- [ ] **Step 1:** 表驱动单测（member 无批权；dept_manager 有批权且仅 scope 内）
- [ ] **Step 2:** 实现并让 `has_permission` 可选用
- [ ] **Step 3:** Commit `feat(auth): permission evaluator with business roles`

---

## 40.14 — RAG：org 标签**写入面** + OrgScope **查询面**

> **坑：** 只滤查询、不给文档打标 → 新文档 `org_unit_id=NULL` → 过滤后全空或全漏。写入与查询必须同 Wave 交付。

**Files:**
- Migration: 文档/知识块表加可空 `org_unit_id`（**009_rag_org_unit.py**；chunk 继承文档标签）
- Modify: `backend/routers/files.py`（上传入库路径）
- Modify: `backend/modules/rag/routers/rag_router.py`（上传 / ingest / list / ask）
- Modify: `backend/modules/rag/services/rag_service.py`（写入打标 + 检索过滤）
- Modify: 任何其它 RAG 入库入口（capability `_invoke_rag` 写路径若存在一并打标）
- Test: `tests/test_rag_org_scope.py`

**写入规则（写死）:**
1. 上传/入库时绑定 `org_unit_id =` 上传者 **主部门**（`OrgScope.primary_org_unit_id`）；无主部门 → **拒绝上传**（400），禁止静默 NULL。  
2. **禁止**客户端自报 `org_unit_id` 扩权；若 body 带该字段，忽略或仅当与主部门一致时接受。  
3. Chunk / 向量 metadata 必须携带同一 `org_unit_id`。

**存量 NULL 策略（写死）:**
- `org_unit_id IS NULL` 的历史文档：**仅** `tenant_admin` / `auditor` / `super_admin` 在查询面可见。  
- 普通 `user` / `member` / `dept_manager`：**不可见** NULL 文档（避免「全可见」漏权）。  
- 本 Wave **不做**批量回填；回填属运维/后续任务。

**查询规则:**
- list/ask 一律 `resolve_org_scope` + 可见集合（本部门/子树按角色）∪（平台治理角色对 NULL 的例外）。

- [ ] **Step 1:** 迁移加 `org_unit_id`；单测：上传后文档带主部门标签
- [ ] **Step 2:** 负向测：财务 member 看不到人事文档；member 看不到 NULL 存量；tenant_admin 看得到 NULL
- [ ] **Step 3:** list/ask 过滤实现与写入同批合并
- [ ] **Step 4:** Commit `feat(rag): org tag on ingest + OrgScope on query`

---

## 40.15 — Memory 查询面（本期无部门共享 → OrgScope 过滤 N/A）

**Files:**
- Modify: `backend/routers/memory.py` / `backend/core/memory_service.py`（仅加固既有隔离，**不**假装部门过滤）
- Create/Modify: `docs/` 或 `learning/` 一句 + 本任务注释：「部门共享记忆」未立项前，S3 对 memory **部门维**标 N/A
- Test: `tests/test_memory_isolation.py`（命名勿叫 org_scope 假装已做）

**写死（本期）:**
1. 现状 memory **没有**「部门共享」资源类型 → **不做** `org_unit_id` 过滤，**禁止**只留 hook 声称满足 S3。  
2. S3 对本期的承诺收窄为：**跨用户 / 跨租户隔离不回退**（既有 `assert_user_access` / `tenant_id` 边界有测）。  
3. 真·部门维 OrgScope 过滤：等「部门共享记忆」类型立项后单开任务；届时写入面+查询面一起做（吸取 40.14 教训）。

- [ ] **Step 1:** 负向测：用户 A 列不出用户 B 的记忆；跨租户拒（不回退）
- [ ] **Step 2:** 文档标注 memory 部门维 = N/A（本期）
- [ ] **Step 3:** Commit `test(memory): harden user/tenant isolation; org filter N/A`

---

## 40.16 — OrgScope 接入 Audit 导出/查询

**Files:**
- Modify: `backend/routers/audit.py`
- Test: `tests/test_audit_org_scope.py`

规则：
- `auditor` / `super_admin`：保持跨租户能力，但部门过滤仍走门面参数
- `tenant_admin`：本租户
- 普通 `user`：仅本人相关 audit（若今日已如此则加固；不可因 OrgScope 放大）

- [ ] **Step 1:** 负向测：member 导不出他部经理审批记录
- [ ] **Step 2:** 实现
- [ ] **Step 3:** Commit `feat(audit): OrgScope on query/export`

---

## 40.17 — 结构 seed（仅组织，无业务假域）

**Files:**
- Create: `scripts/seed_org_structure.py`
- Modify: `scripts/seed_api_keys.py` 或 README 说明调用顺序

Seed 内容（acme 示例）：
- 根部门 + 财务 + 人事
- 用户：`user`+member@财务；`manager`+dept_manager@财务；`tenant_admin`
- **不** seed `dept_operator`；**不** seed 考勤/报销单据

- [ ] **Step 1:** 脚本幂等可跑
- [ ] **Step 2:** 文档写清「结构 seed ≠ 业务 seed」
- [ ] **Step 3:** Commit `chore(seed): org structure seed for pilot B`

---

## 40.18 — Wave 1 负向包收口

**Files:**
- Test: `tests/test_org_negative_pack.py`（汇总跨部门拒绝）

- [ ] **Step 1:** journey：经理可见子树 / 成员拒 / 跨租户拒；**RAG** 含「写入打标 + 跨部不可见 + NULL 仅 admin」
- [ ] **Step 2:** `uv run pytest tests/test_org*.py tests/test_rag_org_scope.py tests/test_memory_isolation.py tests/test_audit_org_scope.py -q`
- [ ] **Step 3:** Code review；勾父任务 W1

---

## Wave 1 Done 标准

- [ ] 组织 CRUD + OrgScope 单测绿  
- [ ] **RAG：** 写入打标 + 查询过滤 + NULL 策略均有测（缺一不可）  
- [ ] **Memory：** 跨用户/跨租户隔离测绿；部门维明确 **N/A**（禁止 hook 充数）  
- [ ] **Audit：** 至少一条 OrgScope 负向测  
- [ ] 无客户端自报部门扩权路径  
- [ ] 无挂起等批代码（留给 W4）
