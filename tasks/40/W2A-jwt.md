# Task 40 · Wave 2A — 人侧 JWT（纯增量，不收紧）

> **状态:** 待实现  
> **依赖:** Wave 0（建议 Wave 1 也完成，便于 FE 联调组织；硬依赖仅 W0）  
> **挂接:** **J0.5a**；§9.6 **M2a**；master D2  
> **红线:** **禁止** 修改 `verify_api_key` 拒绝非 machine；存量 `cg_` 全通路必须仍绿  
> **非目标:** J0.5b / 关窗 / refresh token 完整体系

---

## 40.20 — JWT 签发与校验模块

**Files:**
- Create: `backend/core/auth/jwt_session.py`
- Modify: `backend/core/config.py` 或 settings：`JWT_SECRET`, `JWT_TTL_SECONDS`（短 access，默认 3600）
- Test: `tests/test_jwt_session.py`

**Produces:**
```python
def issue_access_token(*, sub: str, tid: str, role: str) -> str: ...
def verify_access_token(token: str) -> dict:  # claims: sub, tid, role, jti, exp
```

算法第一刀：**HS256**。claims 不含权限大包（权限运行时求值）。

- [ ] **Step 1:** 单测签发→校验；过期拒绝；篡改拒绝
- [ ] **Step 2:** 实现
- [ ] **Step 3:** Commit `feat(auth): HS256 access token issue/verify`

---

## 40.21 — `verify_session` Depends

**Files:**
- Create: `backend/core/auth/session_auth.py`
- Modify: `backend/core/auth/models.py`（`credential_kind="human_session"`）
- Test: `tests/test_verify_session.py`

**Produces:**
```python
async def verify_session(authorization: str | None = Header(None)) -> TenantContext:
    # Bearer <jwt> → TenantContext(
    #   credential_kind="human_session",
    #   acting_user_id=sub, user_id=sub, tenant_id=tid, role=role,
    #   extra_permissions=从 user_app_perms 加载, key_id=None)
```

- [ ] **Step 1:** 无头 / 坏 token → 401 单测
- [ ] **Step 2:** 实现；加载 `user_app_perms` 与现网一致
- [ ] **Step 3:** Commit `feat(auth): verify_session Depends`

---

## 40.22 — 登录/注册改发 JWT（**禁止**再建 api_keys）

**Files:**
- Modify: `backend/routers/auth.py`（去掉 `_insert_api_key` / `_rotate_user_keys` 在 login/register 路径的调用）
- Test: `tests/test_auth_password.py`

**行为（写死）:**
1. `POST /api/auth/login|register` 响应**仅**：`access_token`, `token_type="bearer"`, `expires_in`, 用户摘要（`user_id`/`role`/`tenant_id`/…）。  
2. **响应不含** `api_key` / `cg_`。  
3. **login / register 事务内禁止 INSERT/轮换 `api_keys` 行**——零例外。  
4. Key 来源只剩：`scripts/seed_api_keys.py`（测/开发）、管理台发 machine key（**Wave 2B**）。W2A 期间测试继续用 seed 的 key 走 `verify_api_key` 兼容窗。

- [ ] **Step 1:** 改测：登录后 DB **无**新 `api_keys` 行；响应有 `access_token`、无 `api_key`
- [ ] **Step 2:** 实现 login/register；删除会话用途的 key 创建代码路径
- [ ] **Step 3:** Commit `feat(auth): login/register issue JWT only, no api_keys`

---

## 40.23 — 双接受辅助（人侧路由过渡）

**Files:**
- Create: `backend/core/auth/dual_auth.py`
- Modify: 选 1～2 条人侧路由试点挂载（建议 `backend/pipeline/router.py` 的 `/chat` + 后续 runner 路由占位）
- Test: `tests/test_dual_auth.py`

**Produces:**
```python
async def verify_human_or_legacy_key(...) -> TenantContext:
    # 1) 有 Bearer → verify_session
    # 2) 否则 X-API-Key → verify_api_key（仍不按 type 拒绝）
    # metrics/log: auth_path=bearer|api_key
```

- [ ] **Step 1:** 单测两种入口都得到 TenantContext
- [ ] **Step 2:** `/chat` 改用 dual（或 `verify_session` + 保留旧测用 key 的 dual）
- [ ] **Step 3:** Commit `feat(auth): dual accept bearer or api key on human routes`

**禁止:** 在 `verify_api_key` 内加 machine-only。

---

## 40.24 — 权限工厂不绑死 api_key

**Files:**
- Modify: `backend/core/auth/permissions.py`
- Test: 扩既有 permission 测；加 Bearer 路径冒烟

今日 `require_permission` 底层 `Depends(verify_api_key)`。改为：
- `require_permission` 接受「已解析 TenantContext」或内部改依赖 `verify_human_or_legacy_key`（人侧路由）
- Capability Hub 保持现有动态闸（可继续 api_key；W2A 不强制改完所有路由）

- [ ] **Step 1:** 梳理调用点；人侧路由切 dual/session
- [ ] **Step 2:** 全量 `pytest tests/test_auth*.py tests/test_capability*.py -q` 仍绿（key 路径）
- [ ] **Step 3:** Commit `refactor(auth): permission depends accept session context`

---

## 40.25 — FE：Bearer 会话

**Files:**
- Modify: `frontend/src/stores/authStore.ts`
- Modify: `frontend/src/api/http.ts`
- Modify: `frontend/src/api/auth.ts`
- Modify: `frontend/src/pages/login.tsx` / `register.tsx`
- Test: 既有 FE 测（若有）+ 手动 checklist

行为：
- 登录存 `access_token`（sessionStorage）  
- `apiFetch`：优先 `Authorization: Bearer`；**RoleSwitcher 四槽长期 key** 可保留给 `/dev`（W5c），产品壳登录路径不用  
- 401 → 清 token → `/login`

- [ ] **Step 1:** login 存 token
- [ ] **Step 2:** http 客户端挂 Bearer
- [ ] **Step 3:** Chat 面板用登录用户跑通一轮
- [ ] **Step 4:** Commit `feat(fe): bearer session for product login`

---

## 40.26 — J0.5a 验收包（明确不验 0.5b）

**Files:**
- Create: `examples/qa/journeys/j05a_jwt_session.md`（或 `tests/test_j05a_acceptance.py`）

**Must 断言:**
1. 密码登录返回 JWT，**且** DB 无新 `api_keys` 行  
2. Bearer 调人侧 API 200  
3. 响应/产品路径无长期 `cg_`  

**Explicitly NOT in this pack:**
- 人带旧 key → 401（那是 J0.5b / 7B）

- [ ] **Step 1:** 自动化或脚本化上述 3 条
- [ ] **Step 2:** 跑全量 auth + 抽样业务测确认 **旧 X-API-Key 仍通**
- [ ] **Step 3:** Code review；勾 W2A

**回归命令:**
```bash
uv run pytest tests/test_auth.py tests/test_auth_password.py tests/test_jwt_session.py tests/test_verify_session.py tests/test_dual_auth.py -q
# 抽样：旧 key 路径不得大面积 401
uv run pytest tests/test_capability.py tests/test_auth_scope.py -q
```

---

## Wave 2A Done 标准

- [ ] J0.5a 三断言通过  
- [ ] **零**「非 machine → 401」代码  
- [ ] 旧 key 主测绿  
- [ ] FE 登录走 Bearer
