# Task 39: 管线早预处理 — 归一化 + cheap gate + 推迟 load_memory

> **状态:** 设计已拍板，待实现
> **拍板(2026-08-05, Joe):** 方案 **C** = 缓存命中率（归一化）+ 被拦/垃圾流量降本（cheap gate + 推迟 memory），落地形态取先前讨论的 **B**（`preprocess` 节点 + 重排边），不把完整 injection/PII 护栏整体前移。
> **动机:** 当前 `make_query_hash` 对原始 `message` 哈希，空白/全角差异导致精确缓存命中率≈0；`load_memory` 在 cache hit / 注入拦截之前就打存储，被拦请求仍付记忆 I/O。
> **依赖:** Task 34✓（`UnifiedMemoryService`）；与 Task 35（缓存语义）互补——35 统一 Redis/key 规范，39 补齐 chat 管线「读/写同一归一化 key」与早退。
> **非目标:** 同义词改写、语义缓存、护栏配置化（属 Task 32 V2.0 冻结）、产品 FE。

---

## 1. 问题与目标

### 1.1 现状主链

```text
auth_check → load_memory → rate_limiter → cache_check
  → [miss] guardrails_input → analyze → build_context → …
```

| 缝隙 | 证据 |
|------|------|
| 精确缓存几乎不命中 | `cache_check.make_query_hash` / `write_memory` 写 exact 均对**原始**字符串 SHA256；无 NFKC/空白折叠 |
| 归一化能力未复用 | RAG 已有 `backend/modules/rag/cache.py:normalize`（NFKC + lower + 折叠空白），chat 管线未用 |
| 贵操作太早 | `load_memory` 在 hit/block 之前；拦截路径仍读 hot/warm/cold |
| 无廉价早退 | 空串/超长/租户 deny-list 挤在较晚的 `guardrails_input`（且超长目前是截断 redacted，不是硬拦） |

### 1.2 成功标准

1. `"你好"` / `"你好 "` / 全角空格变体 → **同一** exact cache key（读写一致）。
2. empty / oversize / blacklist 命中 → **不**调用 `load_memory`，直接 END + 审计可区分错误码。
3. cache hit → **不**调用 `load_memory`。
4. 完整 injection + PII 仍由 `guardrails_input` 负责（位置：cache miss 之后、analyze 之前）；不因本任务改变「缓存内容必须是已过护栏的干净答案」这一不变量。
5. `learning/05-pipeline.md` 管线地图与诚实版说明同步更新。

---

## 2. 设计决策（已定）

| # | 决策点 | 定案 | 依据 |
|---|--------|------|------|
| D1 | 范围 | 归一化 + cheap gate + 推迟 `load_memory`；**不**把 full guardrails 挪到 cache 前 | 完整护栏更贵；PII 改写文本会搅乱 key；cache 前只做廉价、确定性检查 |
| D2 | 归一化语义 | 与 RAG **同一算法**：NFKC + `lower` + 空白折叠 + `strip`；**不做**同义词 | 避免两套 norm；可抽公共函数一处维护 |
| D3 | raw vs normalized | `raw_input` 始终保留原始入站文本；`message` 在 preprocess 后改为**归一化**文本（供意图/缓存/路由）；审计 `input_text` 记 **raw**（或同时落 raw+norm，见 39.05） | 合规要原话；缓存/路由要稳定 key |
| D4 | cache key 源 | 读写 exact 一律：`hash(normalize(text))`；state 在 preprocess 写入 `query_hash`，`cache_check`/`write_memory` **禁止再对裸 message 哈希** | 消除 read/write 漂移；PII 若改写 `message`，写回仍用预处理时的 `query_hash` |
| D5 | cheap gate 内容 | （1）归一化后空 → block；（2）原始长度 > `PIPELINE_MAX_INPUT_CHARS`（默认 10000）→ block；（3）租户/全局 deny-list 子串或正则命中 → block | 与现有 `check_input` 长度逻辑对齐但改为**硬拦**（现状是 truncate redacted） |
| D6 | deny-list 存储 | V1：**环境变量 / 配置文件**（全局列表）+ 可选 `tenant_id` 覆盖 map；不做 DB 配置台（配置化属 32） | 先可运维、禁大坨 |
| D7 | rate_limiter 位置 | `preprocess` **之后**、`cache_check` **之前**；cheap block **计入**限流（防刷 blacklist 探测） | 攻击面仍受桶令牌约束 |
| D8 | load_memory 新位置 | 仅 cache miss 且 guardrails pass 之后、`analyze_parallel` **之前**（或之后二选一见下） | **定案：guardrails pass → load_memory → analyze_parallel**。记忆不参与意图规则时也可 analyze 后再 load；选「先 load」与今日 `build_context` 依赖一致且改动面小 |
| D9 | finish_reason / 错误码 | cheap block：`finish_reason=blocked`，`error_code` 区分 `GATE_001`(empty) / `GATE_002`(oversize) / `GATE_003`(blacklist)；injection 仍 `GUARD_001` | 观测与审计可分桶 |
| D10 | 与 mock 写缓存 | `write_memory` 在 `LLM_MOCK` 下写 exact 时必须用 `state["query_hash"]`（或同一 normalize），并评估是否在非 mock 路径也写 exact（可跟 35 收尾；本任务至少修 mock 路径一致性） | learning 文档曾写「管线不写缓存」已过时；本任务以代码为准并改文档 |

---

## 3. 目标主链

```text
[START]
  │
  ▼
auth_check
  │
  ▼
preprocess          ← 新增：normalize + cheap_gate；写 query_hash / 更新 message
  │
  ├─[block]──► [END]   (GATE_00x，无 load_memory)
  │
 [pass]
  ▼
rate_limiter
  │
  ▼
cache_check         ← 只用 state["query_hash"]；fingerprint 启发式对 normalized message
  │
  ├─[hit]──► [END]     (无 load_memory)
  │
 [miss]
  ▼
guardrails_input    ← injection + PII（可改 message；不改 query_hash）
  │
  ├─[block]──► [END]   (GUARD_001，无 load_memory)
  │
 [pass]
  ▼
load_memory         ← 从入口后挪到此处
  │
  ▼
analyze_parallel
  │
  ▼
build_context → experiment_hook → model_router → …
```

贯穿：`audit_logs` + LangFuse；cheap/guard block 均须可审计。

---

## 4. 模块落点

| 组件 | 路径 | 职责 |
|------|------|------|
| 公共 normalize | 建议抽到 `backend/core/text_normalize.py`（或 `backend/core/utils/normalize.py`），RAG `cache.normalize` **改为 re-export/调用**，禁止复制粘贴第二份 | 单一真相 |
| preprocess 节点 | `backend/pipeline/nodes/preprocess.py`（新） | normalize → cheap checks → 写 `query_hash`、`message`、block 字段 |
| deny-list | `backend/core/guardrails/deny_list.py`（新）或 preprocess 内小模块；配置 `PIPELINE_DENY_LIST`（逗号分隔）+ 可选 JSON 文件路径 | 可测、可 mock |
| graph 重排 | `backend/pipeline/graph.py` | 边：见 §3；条件边 `should_block_to_end` 复用或拆 `should_gate_block` |
| state | `backend/pipeline/state.py` | 增加 `query_hash: str`（必填在 preprocess 后）；可选 `gate_reason: str` |
| cache 读写 | `cache_check.py` / `write_memory.py` | 删除本地「裸 hash」；统一读 `state["query_hash"]` |
| 长度逻辑 | `input_guard.check_input` | 超长改为由 preprocess 硬拦后，guard 内 **删除或降级** truncate 分支，避免双重语义 |

---

## 5. Subtasks

### Subtask 39.01: 公共 `normalize` + `query_hash`

> **现状:** RAG 有 `normalize`/`norm_hash`；chat `make_query_hash(message)` 无 norm。

**方案:**
1. 抽出 `normalize_text` / `make_normalized_query_hash`（hash 前 16 位，与现 exact 一致）。
2. RAG `cache.normalize` / `norm_hash` 改为调用公共实现（行为单测对齐现有 `test_rag_cache`）。
3. 废弃 chat 侧「裸 `sha256(message)`」写法（或令 `make_query_hash` 内部先 normalize，并在文档标明）。

**AC:**
- [ ] `"你好 "` 与 `"你好"` → 同一 16-hex
- [ ] RAG 既有 normalize 单测仍绿
- [ ] 无第二份复制粘贴的 NFKC 逻辑

**验证:**
```bash
uv run pytest tests/test_rag_cache.py -q --tb=short
uv run pytest tests/test_text_normalize.py -q --tb=short   # 新建
```

---

### Subtask 39.02: `preprocess` 节点 + cheap gate

> **现状:** 无独立预处理；空/黑名单无早退。

**方案:**
1. 新节点 `preprocess(state)`：
   - `raw_input = state.get("raw_input") or state["message"]`（若入口已设 raw 则保留）
   - `normalized = normalize_text(raw_input)`
   - empty → block `GATE_001`
   - `len(raw_input) > MAX` → block `GATE_002`（用 raw 长度，防归一化后变短绕过）
   - deny-list on **normalized**（及可选 raw）→ `GATE_003`
   - pass：`state["message"]=normalized`，`state["query_hash"]=...`，`gate` 通过
2. `should_gate_block` 条件边 → END。
3. 指标：复用或新增 `guardrails_blocked` label `guard=gate_empty|gate_oversize|gate_deny`（或独立 counter，选改动小者）。
4. 配置：`PIPELINE_MAX_INPUT_CHARS`（默认 10000）、`PIPELINE_DENY_LIST`（可选）。

**AC:**
- [ ] 空/纯空白 → 200 业务响应或管线 END，`finish_reason=blocked`，`error_code=GATE_001`，**无** memory 读
- [ ] 超长 → `GATE_002`
- [ ] deny-list 命中 → `GATE_003`，审计可见
- [ ] 正常句 → `message` 已归一化且 `query_hash` 非空

**验证:**
```bash
uv run pytest tests/test_pipeline_preprocess.py -q --tb=short
```

---

### Subtask 39.03: graph 重排 — 推迟 `load_memory`

> **现状:** `auth → load_memory → rate_limiter → cache_check → …`

**方案:**
1. `build_pipeline` 改为：
   `auth_check → preprocess →(block?)→ rate_limiter → cache_check →(hit?)→ END`
   `→ guardrails_input →(block?)→ load_memory → analyze_parallel → …`（其后不变）
2. 单测/集成：mock memory，断言 cache hit 与 GATE/GUARD block **零次** `UnifiedMemoryService.read`。
3. 流式路径若共用 `build_pipeline`，一并覆盖；若另有旁路入口，列清单确认无「旧序」残留。

**AC:**
- [ ] `graph.py` 边与 §3 一致
- [ ] hit / GATE_* / GUARD_001 路径不触发 `load_memory`
- [ ] miss + pass 路径仍能读到 hot/warm/cold 并进入 `build_context`

**验证:**
```bash
uv run pytest tests/test_pipeline_graph_order.py -q --tb=short   # 新建或扩现有
uv run ruff check backend/pipeline/
```

---

### Subtask 39.04: cache 读写对齐 `query_hash`

> **现状:** `cache_check` 与 `write_memory` 各自 `sha256(message)`；PII 改写后写 key 可能与读不一致。

**方案:**
1. `cache_check`：exact key = `exact:{tenant}:{user}:{state["query_hash"]}`；禁止重算裸 hash。
2. `write_memory`：写 exact 用同一 `query_hash`；fingerprint/template 逻辑不变。
3. 明确：**PII redaction 不更新 `query_hash`**（key 锚定用户原始意图的归一化形式）。
4. 可选小清理：非 mock 是否写 exact —— 若本任务只修一致性，在 AC 注明「行为与现网一致，仅修 hash」；若顺手打开非 mock 写入，需另开勾选并写 MANUAL_TEST（默认 **不扩大写入面**）。

**AC:**
- [ ] 读写 key 对 `"你好"` / `"你好 "` 一致
- [ ] PII 脱敏后再次写回，exact key 仍等于 preprocess 时的 `query_hash`
- [ ] 现有 cache 相关单测更新到 normalize 语义

**验证:**
```bash
uv run pytest tests/ -k "cache or preprocess or fingerprint" -q --tb=short
```

---

### Subtask 39.05: 审计 / 观测 / 文档

**方案:**
1. block 路径确保 `log_audit`（或现有 router 收尾）带上 `error_code` 与 raw 输入（截断存储若过长）。
2. 更新 `learning/05-pipeline.md` 地图与「诚实版」：归一化已做；load_memory 新位置；去掉过时「管线不写缓存」若仍存在。
3. `docs/CACHE.md` 增一小节：chat exact key = `hash(normalize(raw))`，与 RAG `norm_hash` 算法同源。
4. `docs/MANUAL_TEST.md` 补 2～3 条：空白变体命中、deny-list 早退、hit 无 memory（可用日志/计数断言）。

**AC:**
- [ ] learning + CACHE + MANUAL_TEST 与实现一致
- [ ] 错误码表（GATE_/GUARD_）在 MANUAL_TEST 或 errors 文档可查

---

## 6. 风险与回滚

| 风险 | 缓解 |
|------|------|
| 旧 cache 条目按裸 hash 写入，上线后「暂时更不命中」 | 可接受（旧 TTL 短，如 300s）；或一次性清 `cache_type=exact`；文档注明 |
| lower() 对个别 locale 意外 | 与 RAG 已上线策略一致；中文为主场景无感 |
| deny-list 误杀 | 默认空列表；先 env 灰度；命中打明确 `GATE_003` + reason |
| 测试依赖节点顺序硬编码 | 39.03 用行为断言（memory 调用次数）而非仅读源码字符串 |

回滚：feature flag `PIPELINE_PREPROCESS_ENABLED`（默认 true）——若实现成本低则加；否则靠 revert commit。推荐加 flag，便宜。

---

## 7. 验收（Task 全绿）

```bash
uv run ruff check backend/ scripts/
uv run mypy
uv run pytest tests/test_text_normalize.py tests/test_pipeline_preprocess.py \
  tests/test_pipeline_graph_order.py tests/test_rag_cache.py -q --tb=short
# 行为抽检：
# 1) 连续两次 /chat「你好」与「你好 」——第二次 finish_reason=cache_hit（在写入开启的前提下）
# 2) deny-list 词 —— 无 load_memory，error_code=GATE_003
# 3) 注入句 —— GUARD_001，无 load_memory
```

---

## 8. 实现顺序建议

1. 39.01（无行为风险，纯抽取）
2. 39.04 可与 39.01 同 PR 后半（先让 hash API 就位）
3. 39.02 + 39.03（graph 一次改完，避免中间态）
4. 39.05 文档收尾

**禁止:** 在未落地 `query_hash` 前只改 `load_memory` 位置（会放大「hit 仍不一致」的困惑）。

---

## 9. 与相邻任务关系

| 任务 | 关系 |
|------|------|
| Task 35 | 缓存基础设施；39 补 chat 管线归一化与早退，不重复造 Redis 层 |
| Task 09 / guardrails | full injection/PII 仍在原节点；长度硬拦从 guard「截断」上收到 gate |
| Task 32 | 租户级护栏/黑名单配置台仍冻结；本任务仅 env/文件级 deny-list |
| learning/05 | 文档跟随 39.05 更新，避免再教错误顺序 |
