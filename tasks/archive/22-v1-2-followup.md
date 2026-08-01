# Task 22: v1.2 收尾 — 补测试 + 模型策略补全 + A/B 转化落库

> **状态:✅ 完成(Cursor)**
> **基线:main @ 4df9c08;验收:make verify + make check + pytest 全绿(audit_consistency 太重,批次收尾由 Hermes 跑)**
> **每个 Subtask 完成后 git commit,Signed-off-by: Joe**
> **背景:Task 21 已完成的 Code Review 四项拍板结论(Hermes 2026-08-01),全部落地在本 Task。**

## 22.01 模型选择补 cheapest-in-tier + 修死模型名

> **现状:** `select_model_for_intent`(backend/core/model_registry.py:122)intent→tier 后返回「第一个 enabled 模型」— 同 tier 多模型时是注册顺序决定,不是策略。另外 `deepseek-chat` 已 2026-07-24 停用,registry 里还有 4 处残留。

**方案:**
- `select_model_for_intent` 同 tier 内改为取 `cost_per_1k` 最低者;仍无则 fallback 任意 enabled chat;最后 env fallback
- 死模型名替换(共 4 处):`_default_models()` 的 MODEL_CHEAP/GOOD/BEST 默认值(29/30/31 行)→ `deepseek-v4-flash`;140 行 fallback 同样替换
- `config.env.example` 17-20 行 `DEFAULT_MODEL` / `MODEL_CHEAP` / `MODEL_GOOD` / `MODEL_BEST` = `deepseek-chat` → `deepseek-v4-flash`(重任务可手动配 GOOD/BEST = `deepseek-v4-pro`)

**修改文件:** `backend/core/model_registry.py`、`config.env.example`
**验证:** `uv run python -c "from backend.core.model_registry import select_model_for_intent; print(select_model_for_intent('knowledge_query'))"`;MODEL_REGISTRY_JSON 注册两个同 tier 模型后断言返回 cost 低者;`make check`

## 22.02 补单测(registry / A-B / cost_summary)

> **现状:** tests/ 仅 4 个文件(auth/circuit_breaker/guardrails/harness),Task 21 三个新模块零测试,验收标准「分流比例 ≈ 配置」「聚合数字正确」无证据。

**方案:**
- `tests/test_model_registry.py`(新建):intent→tier 映射断言(greeting→cheap、knowledge_query→good、未知→best)、cheapest-in-tier(注入两个同 tier 模型断言取 cost 低者)、disabled 模型跳过、fallback 非 None
- `tests/test_ab.py`(新建):从 `assign_variant` 提取纯函数 `_pick_variant(score, groups, weights)`(可测性小重构,保持 zip(strict=False) 行为一致);断言确定性(同 score 同结果)、分布(1000 个 hash score,50/50 weights → 比例误差 < 0.1)、边界(score=0 取第一组、score≈1 取最后一组);`_stable_bucket` 确定性断言
- `tests/test_cost_summary.py`(新建):monkeypatch `get_pg_session` 返回桩 → 断言 SQL 参数拼装(无参 / 仅 tenant / from_ts+to_ts 窗口 / granularity=hour 走 hour 截断);**不要真连 PG**。真实数字正确性走 MANUAL_TEST 人工路线(造 audit_logs → curl /api/admin/cost-summary)

**修改文件:** `backend/core/ab/service.py`(仅提取 _pick_variant,不改行为)、`tests/test_model_registry.py`(新建)、`tests/test_ab.py`(新建)、`tests/test_cost_summary.py`(新建)
**验证:** `uv run pytest`(全绿,计数从 21 上升);`make check`

## 22.03 Spec 备注对齐(纯文档)

> **现状:** tasks/21 的 21.05 写 `?from=&to=`,实现是 `from_ts/to_ts`(Python 保留字,正确);21.01 的「预算/租户」策略未实现;21.04 未提 conversion。

**方案:**
- 21.05:端点参数改记为 `?tenant_id=&from_ts=&to_ts=&granularity=day|hour`,注明「from/to 是 Python 保留字,故用 *_ts,时间戳语义明确」
- 21.01:注明 cheapest-in-tier 已实现(22.01);tenant budget 移 v2.0,配额语义(硬限/软限、超额拒绝还是降级、超额审计留痕)待产品决策,不在本版本赶
- 21.04:注明 conversion 由 22.04 自动落库(不再依赖手动 /events)

**修改文件:** `tasks/21-v1-2-enterprise.md`
**验证:** 无(diff review 即可)

## 22.04 A/B conversion 自动落库(双路径覆盖)

> **现状:** experiment_hook 只记 exposure;conversion 仅手动 /events,业务方必然漏报。管线短路径(model_router→END)和长路径(→llm_generate→guardrails_output→write_memory→END)都缺 conversion。

**方案:**
- `backend/pipeline/nodes/conversion_hook.py`(新建):若 `state["ab_experiment_id"]` 存在且产出最终响应(`state.get("response")` 非空)→ `record_event(event_type="conversion", event_data={trace_id, session_id})`;整体 try/except 防御(参考 experiment_hook.py 风格,DB 故障不拖垮管线)
- `backend/pipeline/graph.py`:短路径 model_router 条件边 `"end"` → `"conversion_hook"`(route_short_or_long 返回值同步改);长路径 `guardrails_output → write_memory → conversion_hook → END`;新增边 `conversion_hook → END`
- 一致性:缓存命中(cache_check→END)与输入拦截(guardrails_input→END)不经过 experiment_hook(无 exposure),conversion_hook 同样不达 — 无 exposure 即无 conversion,天然一致

**修改文件:** `backend/pipeline/nodes/conversion_hook.py`(新建)、`backend/pipeline/graph.py`、`backend/pipeline/nodes/model_router.py`(route_short_or_long 返回值)
**验证:** 起服务(curl /chat 短路径 + 长路径各一次),`SELECT * FROM ab_test_events WHERE event_type='conversion'` 各出现对应记录;`make check`

---

## 验收标准(Task 22 全部)

- [x] 22.01 同 tier 取 cost 最低;deepseek-chat 全仓 0 残留(`grep -rn "deepseek-chat" backend/ config.env.example` 无命中)
- [x] 22.02 三个测试文件全绿,覆盖率:registry 路由 / A-B 分流纯函数 / cost_summary 参数拼装
- [x] 22.03 tasks/21 备注与实现一致
- [x] 22.04 双路径 conversion 落库,exposure/conversion 数量关系合理
- [x] `make verify` / `make check` / pytest 全绿(audit_consistency 批次收尾由 Hermes 跑)

## Cursor 会踩的坑

1. **22.01:** deepseek-chat 有 4 处(29/30/31 行 + 140 行 fallback),别只改 config.env.example;cheapest-in-tier 的查找顺序必须是「同 tier cost 最低 → 任意 enabled → env fallback」,别把 fallback 提前
2. **22.02:** _pick_variant 提取是纯重构,zip(strict=False) 语义不能变,行为不能变 — 提取后先跑旧行为等价再写新断言;分布测试用统计容差(误差 < 0.1),不要断言精确值;cost_summary 测试必须桩掉 get_pg_session,连真 PG 会挂在没有 docker 的环境
3. **22.04:** graph.py 条件边 destination 字符串必须和 route_short_or_long 返回值逐字符一致(大小写、拼写);conversion 只在 conversion_hook 一处记,别在 write_memory / model_router 里重复记;conversion_hook 的 record_event 失败必须静默(参考 experiment_hook 的 except: pass)
