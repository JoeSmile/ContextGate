# Task 40 · Wave 0 — 凭证契约脚手架（不回填、不收紧）

> **状态:** 待实现  
> **依赖:** 无（链 A 起点）  
> **阻塞下游:** W1 可读 `credential_kind`；W2A 加 JWT；**禁止**在本 Wave 回填 `credential_type` 或让非 machine key 401  
> **挂接:** §9.6 M1；master D2/D7  
> **验证分层:** 迁移可升 + 全量既有 auth 测试仍绿

---

## 40.01 — Alembic：加列（可空）

**Files:**
- Create: `alembic/versions/007_credential_scaffold.py`
- Modify: `backend/database/pgvector_session.py` → `ApiKey`
- Modify: `backend/database/models.py`（若 ApiKey/审计相关 ORM 有重复定义则同步）

**列（`api_keys`）:**

| 列 | 类型 | 默认 | 说明 |
|----|------|------|------|
| `credential_type` | `VARCHAR(32)` NULL | NULL | 目标枚举 `machine`；本 Wave **不回填** |
| `created_by_user_id` | `VARCHAR(100)` NULL | NULL | 与现有 `created_by` 并存；本 Wave 不强制；2B 再统一语义 |

**说明:** 现有 `created_by` 保留不动，避免破坏 admin CRUD。文档注释：`created_by_user_id` 为 D8-a 正式字段，2B 强制非空。

- [ ] **Step 1:** 写迁移 `upgrade`/`downgrade`（只 ADD COLUMN，无 UPDATE）
- [ ] **Step 2:** `uv run alembic upgrade head` 在本地 PG 通过
- [ ] **Step 3:** ORM 加同名可空字段
- [ ] **Step 4:** Commit `chore(auth): add nullable credential_type columns (wave0)`

**验证:**
```bash
uv run alembic current
uv run pytest tests/test_auth.py tests/test_auth_password.py -q
```
Expected: PASS（行为不变）

---

## 40.02 — `TenantContext` 扩展 + 构建点预留

**Files:**
- Modify: `backend/core/auth/models.py`
- Modify: `backend/core/auth/api_key_auth.py`（`verify_api_key` 填新字段；**不**按 type 拒绝）
- Test: `tests/test_auth_tenant_context.py`（新建）或扩 `tests/test_auth.py`

**Produces（下游契约）:**
```python
@dataclass
class TenantContext:
    tenant_id: str
    user_id: str
    role: str
    extra_permissions: list[str]
    is_cross_tenant: bool
    # NEW — Wave 0
    credential_kind: str = "api_key"   # 过渡值；2A→ human_session；机→ machine_key；链内→ delegation
    key_id: str | None = None          # api_keys.id 或稳定对外 id
    acting_user_id: str | None = None  # 默认 = user_id；机侧 2B 改为 created_by
```

- [ ] **Step 1:** 单测：现有 key 登录后 `credential_kind == "api_key"`，`acting_user_id == user_id`
- [ ] **Step 2:** 跑测确认红（字段缺失）
- [ ] **Step 3:** 扩展 dataclass + `verify_api_key` 赋值；**禁止** `if credential_type != machine: raise`
- [ ] **Step 4:** 绿后 commit `feat(auth): TenantContext credential_kind scaffold`

**验证:**
```bash
uv run pytest tests/test_auth.py tests/test_auth_tenant_context.py -q
```

---

## 40.03 — 审计 / request 上下文字段预留

**Files:**
- Modify: `backend/core/audit.py`（`log_audit` / sync 写允许可选 `credential_kind`, `key_id`, `run_id`, `node_id`）
- Modify: 若 `audit_logs` 表无列 → 同批迁移 `007` 或 `008_audit_run_fields.py`：`credential_kind`, `run_id`, `node_id` 可空
- Test: `tests/test_audit_fields.py`（新建，写一条带新字段的 audit 可读回）

- [ ] **Step 1:** 迁移可空列（无回填）
- [ ] **Step 2:** `log_audit(..., credential_kind=None, run_id=None, node_id=None)` 签名兼容旧调用
- [ ] **Step 3:** 单测旧调用仍通 + 新字段可落库
- [ ] **Step 4:** Commit `feat(audit): optional credential_kind/run_id/node_id`

---

## 40.04 — Wave 0 收口文档 + 迁移编排说明

**Files:**
- Modify: `docs/CACHE.md` 或新建短注 `docs/CREDENTIAL_MIGRATION.md`（推荐短注，避免污染 CACHE）
- Modify: `learning/04a-auth-rbac.md` 加一句「Wave0：列已加、语义未切」
- Modify: `tasks/40-pilot-b-chain-a.md` 勾 W0 完成

**文档必须写清:**
1. 兼容窗仍开；存量 key 行为不变  
2. 回填 + 关窗 = **Wave 2B only**  
3. `credential_kind` 枚举目标：`human_session` | `machine_key` | `delegation` | 过渡 `api_key`

- [ ] **Step 1:** 写 `docs/CREDENTIAL_MIGRATION.md`
- [ ] **Step 2:** learning 一句同步
- [ ] **Step 3:** Commit `docs(auth): wave0 credential migration notes`

---

## Wave 0 Done 标准

- [ ] 迁移可反复 upgrade/downgrade  
- [ ] `pytest tests/test_auth*.py` 全绿  
- [ ] 全仓 **无**「非 machine → 401」新逻辑  
- [ ] Code review（Critical/Minor 自修；Important 列给用户）
