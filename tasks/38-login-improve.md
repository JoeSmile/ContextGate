# Task 38: 登录改造 — 账号密码注册/登录(测试 FE)

> **状态:** 待执行(Cursor 实现)
> **动机(2026-08-04, Joe):** "还有谁家的 Login 页面是用 key 登录的"——测试 FE 登录页
> 目前只有 X-API-Key 粘贴框(`frontend/src/pages/login.tsx`),产品体验不合理。
> **范围:** 测试 FE 的登录体验改造(V1.x 收尾);产品 FE(30.29)冻结中,账号体系后端先行落地,30.29 直接复用。
> **前置:** Task 35(V1.x 收尾)不阻塞本任务,可并行。

## 设计决策(已拍板,A/B 见依据)

| # | 决策点 | 定案 | 依据 |
|---|--------|------|------|
| D1 | 范围 | 只改**测试 FE** 登录页 + 后端账号体系;不动产品 FE(30.29) | 30.29 在 V2.0 冻结名单;测试 FE 是 V1.x 交付物,登录页体验缺陷属于"修烂代码" |
| D2 | 登录凭证 | **密码登录 → 后端签发/复用 cg_ API key**,FE 存入现有 sessionStorage 槽位 | 备选 B(JWT + 全路由改认证)爆炸半径大,违反"禁大坨";`api_keys` 表 + `api_key_auth.py` 是唯一认证通道,密码只是"取 key 的门",下游零改动 |
| D3 | 密码存储 | bcrypt(`uv add bcrypt`,cost=12),禁明文;不用 stdlib pbkdf2 | 业界约定,安全审计/CIO 验收友好 |
| D4 | 注册开放 | `APP_ENV∈{dev,test,demo}` 开放注册;prod 下注册端点返回 403(预留 admin 邀请制) | 测试 FE 是 dev 工具,开放注册合理;企业场景注册必须受控 |
| D5 | 注册字段 | username(唯一,小写归一)/ password(≥8)/ confirm_password(FE 校验)/ display_name(可选)/ role(仅 dev 显示,默认 user);tenant 固定 `acme` | 测试 FE 语境;4 角色注册即可替代手工配 key |
| D8 | users 表(2026-08-04 已落地) | **扩展既有 users 表**(001 迁移创建,memory_hub 在用),不新建:006_users 迁移加 `password_hash`(bcrypt)/`display_name`/`tenant_id`/`role` + username 唯一索引;`models.py User` 已加对应列;`seed_api_keys.py` 已 upsert 5 个测试账号(密码统一 `123456`) | 否则与 memory_hub 的 User 模型/001 表冲突;账号体系列直接长在既有表上 |
| D6 | 防爆破(P0) | 登录失败 + 注册探测计数:同 username 各 5 次/5 分钟 → 429(复用 `redis_tools`,降级内存 dict) | 安全 P0,账号体系一出生就要带限流 |
| D7 | 审计 | register/login 事件写 `audit_logs`(action=`auth.register`/`auth.login`,复用 `backend/core/audit.py:log_audit`) | 审计联动是项目红线,账号事件必须留痕 |

**关键机制:** 注册/登录成功后返回 `cg_...` key(明文仅一次),FE 按账号 role 存入对应槽位
(如注册 role=auditor → 存入 auditor 槽位并切换),RoleSwitcher / 面板 / 权限矩阵全部原样工作。

## Subtask 38.01: 后端账号体系

> **现状:** users 表已存在(001 迁移创建,`models.py User`,memory_hub 在用);**006_users 迁移已落地**
> (password_hash/display_name/tenant_id/role + username 唯一索引);`models.py User` 已加对应列;
> **seed 已同步 5 个测试账号**(alice/bob/admin/auditor1/admin_acme,密码统一 `123456`);
> bcrypt 依赖已装。**本 subtask 只剩 auth 路由,不再动表结构。**

**方案:**
1. 新建 `backend/core/auth/password.py`: `hash_password(pw) -> str`(bcrypt, cost=12)、
   `verify_password(pw, hash) -> bool`。
2. 新建 `backend/routers/auth.py`(无全局 auth Depends,自身校验):
   - `POST /api/auth/register` {username, password, display_name?, role?} → 409(重名)/ 422(弱密码)/
     403(prod)/ 200 {api_key, role, tenant_id}(创建 users 行 + api_keys 行,tenant=acme,created_by='register')。
     明文 key 仅此一次返回;`log_audit(auth.register)`。
   - `POST /api/auth/login` {username, password} → 401(密码错)/ 429(5 次/5min)/ 200 {api_key, role, tenant_id}。
     登录成功**轮换**该用户 active key(停用旧 key → 签发新 `cg_` 明文返回一次;
     因 api_keys 仅存 SHA256,无法真正「复用」明文),`log_audit(auth.login)`。
   - 失败/注册计数:redis(有)→ 内存 dict(降级);登录键 `auth:fail:{username}`,
     注册键 `auth:reg:{username}`,各 5 次/5 分钟窗口。
   - ⚠️ users 表身份列:user_id 与 username 并存,seed 里两者相同;登录按 `username` 查,
     `memory_hub` 仍按 `user_id` 查(兼容)。
3. `backend/app.py` 挂载:`_lazy_include(app, "backend.routers.auth", "router", label="账号认证", required=True)`。

**修改文件:** `backend/core/auth/password.py`(新) · `backend/routers/auth.py`(新) · `backend/app.py`
(表结构/模型/seed 已由 2026-08-04 先行落地,勿重复建表)

**AC(自带):**
- [ ] 注册成功返回 cg_ key(仅一次),DB 中 password_hash 为 bcrypt 密文(非明文)
- [ ] 重复 username 注册 → 409;密码 <8 位 → 422;prod env 注册 → 403
- [ ] 登录成功轮换并返回新 cg_ key;错误密码 → 401;同 username 错 5 次 → 429
- [ ] 同 username 注册探测 5 次/5min → 429
- [ ] `audit_logs` 出现 `auth.register` / `auth.login` 记录

**验证:**
```bash
uv run ruff check backend/ scripts/
uv run mypy
uv run pytest tests/test_auth_password.py -q --tb=short
# 手动(curl): register → login → 用返回 key 打 /api/rag/status 应 200
```

## Subtask 38.02: 测试 FE 登录页改造

> **现状:** `frontend/src/pages/login.tsx` 仅 key 粘贴框(角色槽位 + /health 探活);
> `authStore.loginWithKey` 探活后写入槽位。

**方案:**
1. 新建 `frontend/src/api/auth.ts`: `registerAccount(...)` / `loginAccount(...)`。
2. `frontend/src/pages/login.tsx` 改双 tab:**「密码登录」**(username/password + 登录 + 去注册链接)/
   「Key 登录」(现有槽位流程原样保留,QA 角色切换仍是核心能力)。
3. 新建 `frontend/src/pages/register.tsx`: username / password / confirm_password / display_name /
   role 下拉(仅 dev 环境显示;prod 隐藏强制 user);前端校验两次密码一致 + 长度 ≥8;
   成功后自动登录(把返回 key 经 `authStore.setKey(role, key)` 存入对应槽位并切到该角色)→ 跳 `/panels/chat`。
4. `frontend/src/router.tsx` 加 `/register` 路由(公开,不 RequireAuth)。
5. `authStore.ts` 加 `loginWithPassword(username, password)`: 调 login → setKey(role, key) +
   switchRole(role) + 跳转;401/429 错误文案展示。

**修改文件:** `frontend/src/api/auth.ts`(新) · `frontend/src/pages/login.tsx` ·
`frontend/src/pages/register.tsx`(新) · `frontend/src/router.tsx` · `frontend/src/stores/authStore.ts`

**AC(自带):**
- [ ] /login 双 tab 可切换;Key 登录路径行为与现在完全一致(4 槽位互不覆盖)
- [ ] 注册表单校验:两次密码不一致/长度不足 → 阻止提交并提示
- [ ] 注册成功自动登录进入 Chat 面板,右上角角色徽章 = 注册所选 role
- [ ] 密码登录成功 → 面板可访问;错误密码 → 红字提示;429 → 提示"尝试过多,请稍后再试"
- [ ] `npm run test` 现有 RoleSwitcher/login 相关测试不回归

**验证:**
```bash
cd frontend && npm run test && npm run build
# 手动: 注册 auditor 账号 → 自动登录 → Audit 面板可读;Key 登录切 user 槽位照常
```

## Subtask 38.03: 测试与文档

**方案:**
1. 新建 `tests/test_auth_password.py`: 注册/登录/重名 409/弱密码 422/错密 401/限流 429/
   密码密文落库/审计联动(复用现有 test 基建,LLM_MOCK=true)。
2. `docs/MANUAL_TEST.md` 更新:
   - §0: 登录方式补「密码注册/登录」路径(注册 → 自动登录);
   - §1 冒烟: 补 1.9 注册+登录冒烟项;
   - §2 权限矩阵: 注明注册所得 key 等价于对应角色 seed key。
3. `tasks/README.md` 活动任务表加 Task 38 行(状态: 执行中)。

**修改文件:** `tests/test_auth_password.py`(新) · `docs/MANUAL_TEST.md` · `tasks/README.md`

**AC(自带):**
- [ ] 新单测全绿;`make verify && make check && uv run pytest` 回归全绿
- [ ] MANUAL_TEST 已含密码登录路径步骤
- [ ] 一次 commit(Conventional Commits, Signed-off-by: Joe)

**验证:**
```bash
make verify && make check && uv run pytest -q --tb=short
cd frontend && npm run test && npm run build
```

## 验收(全绿才算完成)

```bash
uv run ruff check backend/ scripts/
uv run mypy
uv run pytest -q --tb=short
cd frontend && npm run test && npm run build
# 手动: 注册 4 角色账号各一 → 密码登录逐角色走通 4 面板 → 错误密码 5 次 → 429
# audit_logs 含 auth.register / auth.login;DB 无明文密码
```

## 明确不做(本轮)

- 不发 JWT / 不改现有路由认证(D2 已定:密码是取 key 的门)
- 不做产品 FE(30.29)登录页——其账号体系直接复用本任务后端
- 不做 admin 邀请制/邮箱验证/找回密码——预留 prod 403 闸门,后续 V2.0
