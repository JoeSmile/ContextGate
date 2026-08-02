# Task 27: LLM API Key 故障转移(多 key 池 + 429/401 自动切换)

> **状态:待执行(Cursor)**
> **基线:main @ f071463;验收:make verify + make check + pytest 全绿(audit_consistency 批次收尾 Hermes 跑)**
> **来源:Joe P0 清单「key auto-failover」+ 2026-08-02 设计讨论。存储层已就绪(llm_api_keys 本就多行多版本),本任务补齐:候选链 + 失败切换 + 冷却 + 健康摘除。**

## 27.01 存储:冷却字段 + 候选链

> **现状:** `llm_api_keys` 表无冷却字段;`LLMKeyRepository.get_key()`(key_repository.py:76)只取最新 active 单 key(`ORDER BY key_version DESC LIMIT 1`)。

**方案:**
- `alembic/versions/004_key_failover.py`(新建):`llm_api_keys` 加两列
  - `last_failed_at timestamptz NULL`(最近失败时间,冷却依据)
  - `consecutive_failures int NOT NULL DEFAULT 0`(连续失败计数,摘除依据)
- `LLMKeyRepository` 新增:
  - `async def get_key_chain(tenant_id, provider, limit=3) -> list[LLMKey]`:`is_active=true` + 未过期 + `NOT (last_failed_at > now() - :cooldown)`,按 `key_version DESC` 排序,取 limit
  - `async def mark_key_failed(key_id) -> None`:`consecutive_failures+1`,`last_failed_at=now()`;若 `consecutive_failures >= KEY_MAX_CONSECUTIVE_FAILURES` → `is_active=false`(自动摘除)
  - `async def clear_key_failure(key_id) -> None`:成功调用后归零
  - 原 `get_key()` 改为 `get_key_chain()` 的薄封装(取第一个),保持调用方不破
- `config.py` Settings + `config.env.example`:`KEY_COOLDOWN_SECONDS=60`、`KEY_MAX_CONSECUTIVE_FAILURES=3`

**修改文件:** `alembic/versions/004_key_failover.py`(新建)、`backend/core/key_repository.py`、`config.py`、`config.env.example`
**验证:** `make db-init` 后 `\d llm_api_keys` 见新列;get_key_chain 返回排序链;mark_key_failed 3 次后 is_active=false

## 27.02 调用层:429/401 自动切 key

> **现状:** harness/llm_client 拿到单 key 直接调,429/401 直接抛错;断路器管 provider 整体,不切 key。

**方案:**
- 定义调用层错误分类:429(限流)/ 401(鉴权失效)→ **切 key**;5xx/超时 → **不切**(走既有断路器)
- `backend/core/harness/llm_client.py`(Task 26 的工厂,文本生成功那里):失败分类后
  - 429/401 → `mark_key_failed(key_id)` → 取链上下一个 key 重试(最多链长,≤3 次)→ 全挂则抛原错误
  - 成功 → `clear_key_failure(key_id)` + 审计记 `llm_key_failover`(provider / from_key_id / to_key_id / reason)
- 注意:短路径(skill)不走 LLM,不涉及;RAG/Agent/Eval 走工厂,自动继承

**修改文件:** `backend/core/harness/llm_client.py`(或对应调用处)、`backend/core/key_repository.py`(mark/clear 已含)
**验证:** 见 27.04 单测;真实场景:两个同 provider key,第一个 mock 429 → 第二次调用用第二个 key 成功

## 27.03 健康检查扩展:自动摘除

> **现状:** `key_health.py` KeyHealthChecker 周期 verify 所有 key,只记录 last_verified_ok,不摘除。

**方案:** `_check_all()` 里对 `last_verified_ok=false` 且 `consecutive_failures >= 阈值` 的 key 置 `is_active=false` + 审计告警;恢复正常(verify 通过)的 key 复位。

**修改文件:** `backend/core/key_health.py`
**验证:** 造一个失效 key(错 key 值)→ 周期检查后 is_active=false + 审计有记录

## 27.04 单测

**方案:** `tests/test_key_failover.py`(新建),mock `get_pg_session`(沿用 test_cost_summary 的桩模式,不真连 PG):
- get_key_chain:排序、排除冷却中 key、limit
- mark_key_failed:计数递增、达阈值摘除
- 重试逻辑:第一个 key 429 → 换第二个成功 → 断言用了 key2 + key1 进冷却;5xx 不切 key
- clear_key_failure:成功后归零

**修改文件:** `tests/test_key_failover.py`(新建)
**验证:** `uv run pytest`(87 + 新增);`make check`

---

## 验收标准(Task 27 全部)

- [ ] 27.01 迁移加列 + 候选链 + 冷却/摘除方法,get_key 兼容不破
- [ ] 27.02 429/401 切 key 重试(≤3 次),5xx 不切走断路器;切换有审计
- [ ] 27.03 健康检查自动摘除失效 key
- [ ] 27.04 单测覆盖(桩模式,不连 PG)
- [ ] `make verify` / `make check` / pytest 全绿

## Cursor 会踩的坑

1. **27.01:** `get_key()` 的调用方很多(model_router / key_health / admin 列表),改成薄封装后行为必须完全一致(tenant → provider → latest active),别顺手改语义;冷却用 `last_failed_at > now() - cooldown`,别用连续失败计数做冷却(计数只用于摘除)
2. **27.02:** 错误分类必须看 HTTP 状态码(429/401),不要用异常字符串匹配(供应商错误格式不统一);5xx 千万别切 key —— 那是断路器的事,切了也是白切还脏冷却数据;重试只对"这次调用"重试,别在循环里无限重试
3. **27.03:** 摘除要防抖动 —— 单次 verify 失败别摘,连续失败才摘;恢复路径(verify 通过复位 is_active)必须有,否则误伤后 key 永久躺尸
4. **27.04:** 桩 session 的 execute 返回结构要和 repo 的 row 访问方式匹配(fetchone/fetchall);测试里造两个 key 行验证切换顺序
