# Task 41 — 记忆 / 提示词 / 缓存 一体设计（设计稿）

> **状态:** 设计已拍板（2026-08-06 讨论定案）。实现排 7A 验收后；Slice 1/2（LangFuse Prompt + langMem）优先。
> **依赖:** Task 40 链 A（不挡，可并行）；RLS 落地（Slice 4 scope 标签来源）；假名化契约先行（Slice 3）。
> **非目标:** 本任务不写实现代码（先落协议）；不引入 mem0 / Zep / Letta / Cognee / PMB；不做程序记忆的审批流 UI（后置）。
> **挂接 J\*:** 与 Task 32（预算）、Task 34（记忆）、Task 35（缓存）语义对齐；抽取成本挂预算引擎；缓存契约与 docs/CACHE.md 同源。

## 决策记录（2026-08-06 拍板）

- **记忆选型:** 直接用 langMem（background manager + PostgresStore 语义检索，LLM 传 harness 实例，LangFuse 原生同 trace）。mem0 黑盒自调 LLM 违反 EVID-08，不引入，只借鉴 v3 算法（ADD-only / entity linking / 多信号检索 / 时间感知）。PMB / Zep / Letta / Cognee 不引入（类别不符 / 闭源 / runtime 冲突 / 过重）。
- **记忆抽取:** 零 LLM 先行（RuleExtractor，收敛现有 `rule_based_session_summary`），留 SmallModelExtractor 接口（Qwen2.5-7B / GLM-4-9B + 约束解码）。开关 `MEMORY_EXTRACTOR=rule|small_model`，小模型只补 rule 低置信度的 pending 批。
- **Prompt 管理:** 直接用 LangFuse 内置（版本 + label 环境切换 + 生产低延迟 fetch + promptfoo 集成 `langfuse://`）。不引入 PromptLayer / Helicone / Agenta（多一个平台 = 多一个合规面，trace 对不上）。promptfoo 只作评测 / red-team 工具。
- **程序记忆（坑1）:** 策略库 = 数据不代码（结构化 schema + 词法白名单 + 作用域隔离），LLM 只能 propose，审批 + 版本化 + label 切换生效（与 LangFuse prompt 同构）。禁止 LLM 自我改写 system prompt。
- **抽取成本（坑2）:** 分级写入（规则路径零 LLM）+ 启发式预筛（LLM 抽取压到 ~10-20% 轮次）+ 攒批合并 + 预算挂钩（Task 32，租户级记忆写入预算）+ 去重防膨胀（相似度预查，ADD-only）。PII 顺序：input_guard → 假名化 → 抽取 → output_guard → 落库二次扫描（防幻觉补全敏感值）。
- **后台整合耗时（坑3）:** 写路径彻底出请求链路（Redis 队列 + worker，复用 redis_tools），读路径只读（pgvector ms 级）；积压时读原文降级；per-tenant+user 锁 + 超时重试 + 增量整合（不重算全量）；LangFuse 独立 trace（memory-consolidation）+ score 建自己基线（第三方 60s p95 未证实，先 harness replay 实测）。
- **假名化（Slice 3）:** 假名 = HMAC(stable_id)，**不是名字**（防重名串人）。字段 / 场景 / 访问三维分级。身份解析表（aliases → stable_id）+ 假名表（stable_id → token，单独还原密钥，salt 版本化）。还原按权限留痕；**还原结果禁进缓存**（红线）。
- **同权限缓存（Slice 4 / Finding D）:** perm_scope = OrgScope + 权限集指纹（**不绑 RLS**）；footprint 覆盖 ⊇；四档粒度；scope 级失效；缓存假名版，还原禁进缓存。

## 现状事实（代码）

| 面 | 位置 | 现状 |
|----|------|------|
| Prompt | `backend/pipeline/nodes/llm_generate.py:24-29` | system prompt 来自 `state["ab_variant_config"]["system_prompt"]`，无版本化 |
| 记忆读 | `backend/pipeline/nodes/load_memory.py` → `backend/core/memory_service.py` | hot/warm/cold 三层；`rule_based_session_summary` 在 memory_service.py:68 |
| 记忆写 | `backend/pipeline/nodes/write_memory.py` | 同步写 |
| 缓存 | `backend/pipeline/nodes/cache_check.py:41,64` | exact 按用户 `exact:{tid}:{user_id}:{qhash}`；template 跨用户 `template:{tid}:{fingerprint}`；RAG/chat 域（docs/CACHE.md）已跨用户 `rag:a:{epoch}:{tid}:{qhash}` |
| 可观测 | `backend/observability/`（langfuse v2 + `@observe`） | 节点级装饰器，无 prompt 版本元数据 |
| 依赖 | `pyproject.toml` | `langfuse>=2.0.0,<3.0.0`；`langchain-core>=0.2.43,<0.3.0`；`langgraph>=0.2.0,<0.3.0`；**无 langmem** |

⚠️ **langmem 兼容性 — 实测结论（2026-08-06，不兼容）:** `uv add langmem` 解析失败：langmem>=0.0.6 依赖 `langchain-openai>=0.3.1`，与项目 `<0.2.0` 锁冲突，全部版本不可用。**按拍板：不引入 langmem，不立项升级** → 走 PostgresStore 语义检索 + 自研 MemoryExtractor（见 Slice 2）。langchain 大版本升级只由未来需求驱动。

---

## Slice 1（优先）— LangFuse Prompt 管理

**目标:** system prompt 从 AB 配置/硬编码迁到 LangFuse 版本化；每条 trace 的 generation span 带 prompt 名 + 版本。

**Files:**
- `backend/pipeline/nodes/llm_generate.py`（messages 组装处 :24-29）
- `backend/observability/langfuse_client.py`（get_prompt 封装或新增 `backend/core/prompt_service.py`）
- 新增 `backend/core/prompt_service.py`：`get_prompt(name, label)` + 进程内缓存 + 未配置时返回内置默认（静默降级，与 redis_tools 同哲学）

**Steps:**
- [x] 评估 langfuse v2 是否够用（2.60.10，`get_prompt` 可用；v3 OTel-native 大改，**不在本任务升级**）
- [x] `prompt_service.get_prompt`：按**环境** label 取 prompt（Slice 1 不做 tenant→label；后置）；LangFuse 不可用 / 内容未过安全闸门 → 内置默认 `DEFAULT_CHAT_SYSTEM`，链路不 500
- [x] `llm_generate` 改从 prompt_service 取 system prompt；优先级：显式 `ab_variant_config`（过 sanitize）> LangFuse label > 内置默认；`asyncio.to_thread` 包同步 SDK
- [x] generation span 的 `enrich_span` 带 `prompt_name` / `prompt_version` / `prompt_label` / `prompt_source`
- [x] 只缓存 LangFuse **成功**命中；失败不 sticky
- [ ] LangFuse UI 人工确认能看到 prompt 版本（Joe 跑链路后看 trace）

**验证命令:**
```bash
uv run ruff check backend/
LLM_MOCK=true uv run python -m pytest tests/test_prompt_service.py -q
uv run python -c "from backend.core.prompt_service import get_prompt; print(get_prompt('chat.system', label='production').source)"
```

**Commit 提示:** `feat: LangFuse prompt management (Task 41 Slice 1)`

**AC:**
- 允许内置默认 `DEFAULT_CHAT_SYSTEM`；`ab_variant_config.system_prompt` 仅作显式覆盖（且过 sanitize）
- trace 的 generation span 可看到 prompt 名 + 版本 + source（langfuse|builtin|ab）
- LangFuse 未配置时全链路不 500（降级内置默认安全 prompt）

---

## Slice 2（优先）— langMem 记忆优化

**目标:** 记忆检索升级为 PostgresStore 语义检索 + 后台整合；保留 hot/warm/cold 语义与 `UnifiedMemoryService.read` 接口；所有 LLM 调用走 harness。

**Files:**
- `pyproject.toml`（加 langmem，⚠️ 先验证兼容，见上）
- `backend/core/memory_service.py`（read/write 语义保留；warm 检索接 PostgresStore 语义索引）
- `backend/pipeline/nodes/load_memory.py`（检索路径，接口不变）
- `backend/pipeline/nodes/write_memory.py`（写路径异步化：队列 + 状态标记）
- 新增 `backend/core/memory/extractor.py`：`MemoryExtractor` protocol + `RuleExtractor`（默认，收敛 rule_based_session_summary）+ `SmallModelExtractor`（预留）

**Steps:**
- [x] `uv add langmem` 实测 → **不兼容**（langchain-openai>=0.3.1 vs <0.2.0），弃用（见 ⚠️）
- [x] `MemoryExtractor` 抽象 + RuleExtractor 默认实现；`MEMORY_EXTRACTOR=rule` 生效（新增 `backend/core/memory/extractor.py`；small_model 接口预留，工厂按 `MEMORY_EXTRACTOR_MODEL` 门控回退 rule）
- [x] write_memory 节点接入抽取（零 LLM 规则 → warm 落库；失败静默，永不 500）
- [x] **双轨保留（拍板 4A）:** `RuleExtractor` = 显式记忆写 warm；`rule_based_session_summary` = cold session 摘要，不强制收敛同一实现
- [ ] PostgresStore 语义检索接入 warm 检索（复用 pgvector；检索结果仍按现有 hot/warm/cold 结构返回，节点接口零改动）
- [ ] 写路径异步化（Redis 队列 + worker，复用 redis_tools 静默降级）；积压时读原文降级
- [ ] 后台整合独立 trace `memory-consolidation` + score（耗时/条数/成功率）
- [ ] 抽取成本护栏就位：预筛触发率 + 预算挂钩（Task 32 租户级记忆写入预算）

**验证命令:**
```bash
uv run ruff check backend/
LLM_MOCK=true uv run pytest tests/test_memory_service.py tests/ -q -k "memory"
```

**Commit 提示:** `feat: langMem-backed memory retrieval (Task 41 Slice 2)`

**AC:**
- 同一用户跨会话可经语义检索命中 warm 记忆（测试用例：换措辞提问仍命中）
- 写路径不阻塞请求（fire-and-forget + pending/done 状态标记）
- LangFuse 出现 `memory-consolidation` trace 且带 score
- 检索接口（`UnifiedMemoryService.read` 签名）对 pipeline 零破坏

---

## Slice 3 — 假名化地基（依赖 Slice 2 契约，实现后置）

**决策摘要:** 见上文决策记录。**协议现在钉死，实现排 Slice 2 之后**（零 LLM 阶段就建身份解析表，RuleExtractor 实体识别直接落 stable_id）。

**Files（实现时）:**
- 新增 `backend/core/identity/`（resolve.py / pseudonym.py / store.py）
- 迁移：`identity_aliases`（aliases → stable_id）、`identity_pseudonyms`（stable_id → token，加密 + salt_version）
- 假名表加密：单独还原密钥（职责分离，不混用 LLM_KEY_MASTER_KEY）

**AC（届时）:**
- 同名不同人 → 不同假名 token；同一人多别名 → 同一 token
- 还原操作带角色权限 + 理由字段 + 审计留痕
- 还原结果标记 no-cache（红线）

---

## Slice 4 — 同权限缓存（依赖 Slice 3，实现后置）

**决策摘要（Finding D · 拍板 A）:**  
`perm_scope` = 授权上下文指纹（有效权限集含 deny + **OrgScope/数据范围** + qhash）；共享条件 footprint 覆盖 ⊇；scope 粒度 = 数据四档（等保对齐）；**标签来源本期 = OrgScope + 权限求值器输出，不绑 Postgres RLS**；RLS 落地后可替换为同一 `perm_scope` 接口的标签提供者。scope 级失效；缓存永远存假名版。

**Files（实现时）:**
- `backend/pipeline/nodes/cache_check.py`（key：`exact:{tid}:{perm_scope}:{qhash}`；读时校验）
- `docs/CACHE.md`（契约：还原禁缓存、footprint 语义、四档分类表、OrgScope 指纹算法）
- 权限变更入口（角色变更/离职 → scope 级 epoch bump）

**AC（届时）:**
- 同权限同范围用户共享命中；权限集不同 / 数据范围不覆盖 → miss
- 降权后旧缓存不可命中（读时校验 + scope 失效双保险）
- 还原内容不出现在任何缓存条目
- **不**以「尚未上 RLS」阻塞 Slice 4

---

## 依赖顺序

```
Slice 1 (prompt) ── 独立，可并行先行
Slice 2 (langMem) ── 独立于 Slice 1；假名化契约只需先行定义（协议已钉死）
   └─ Slice 3 (假名化) ── 依赖 Slice 2 写路径落位
        └─ Slice 4 (同权限缓存) ── 依赖 Slice 3「存储层永远假名」+ OrgScope 指纹（RLS 后置可选）
程序记忆策略库（坑1 审批流）── 最后，审批 UI 后置
```

**排期（2026-08-06 拍板）:** Slice 1/2 **先搞**，与链 A 并行（Joe：尽快用 LangFuse 调试 prompt + 可视化）。**Slice 1 已实现**（见下）。Slice 3/4 在 RLS 落地后。

### Slice 1 实现记录（2026-08-06）

- 新增 `backend/core/prompt_service.py`：`get_prompt(name, label)` **永不 None**；成功命中 TTL 缓存（默认 30s）；失败/未配置/sanitize 失败 → `DEFAULT_CHAT_SYSTEM`（`source=builtin`）
- `sanitize_prompt_content`：类型 / 空 / NUL / 超长闸门；AB 覆盖与远程内容共用
- `llm_generate`：`ab`（过 sanitize）> LangFuse `chat.system`（`asyncio.to_thread`）> 内置默认；span metadata 带 `prompt_{name,version,label,source}`
- **拍板:** 2A 仅环境 label（tenant 后置）；3A `to_thread`；5A 失败不缓存
- 测试：`tests/test_prompt_service.py`（builtin 降级 / 成功缓存 / 失败不 sticky / sanitize）
- 备注：生产改 prompt = LangFuse UI 把 `production` label 指到新 version，进程在 TTL 内自动拉到（无需发版）；A/B = 多 label + 应用侧分流（见下）

### 生产如何改 Prompt / LangFuse A/B（答疑备忘）

- **生产变更:** 新建 immutable version → 验证（staging label / Experiments）→ 把 **`production` label 挪到该 version**。应用始终 `get_prompt(..., label=production)`，**不必 redeploy**；本仓库进程内缓存最多延迟 `LANGFUSE_PROMPT_CACHE_TTL`（默认 30s）。回滚 = 把 `production` 指回旧 version。
- **A/B:** LangFuse **不替你分流**；做法是给两版打 `prod-a` / `prod-b`（或等价 label），应用按用户哈希/百分比选 label，trace 带 `prompt_version`，在 UI Metrics 对比后再把赢家标成 `production`。本仓库口子：`resolve_prompt_label` + env `LANGFUSE_PROMPT_AB=1` / `LANGFUSE_PROMPT_AB_VARIANTS=prod-a:50,prod-b:50`（默认关；同用户粘性哈希）。与现有 `ab_variant_config.system_prompt` 可并存：网关级 AB 仍可用显式覆盖；LangFuse label AB 适合 prompt-only 实验。

### Review 记录（2026-08-06 · 编码代理互审，Joe 拍板后修复）

- **Finding 1（已修，B 方案）**：身份正则 `我是/我叫` 被口语击穿（`我是说…`/`我是不是…`/`我是做开发的` 实测均误写 warm）。修复：`_is_clean_name`（2-8 字纯中文 + 口语/否定/疑问前缀后缀 denylist：不是/是不是/说/觉得/想/做/干/你/他 + 吗/呢/吧/的/了/这样…）+ `_is_clean_pref`（偏好宾语 ≥2 字 + 同后缀表）；真阳性 `我是张三`/`我叫王小明`/`我喜欢喝咖啡` 保留。新增 4 条单测覆盖全部假阳性。
- **Finding 1B / 2A / 3A（已修，2026-08-06 Joe 拍板）**：
  - **1B** 角色籍贯 denylist：`_ROLE_SUFFIX`（人/员/师/家/…）+ `_ROLE_TITLES`（程序员/经理/管理员…）；`我是中国人/程序员` 拒，`我是张三` 留。
  - **2A** 显式「记住」`_is_safe_fact`：拒密码/密钥/API key + 忽略以上/ignore previous/你现在是。
  - **3A** 语气词：扩 `_FILLER_SUFFIX` + `_strip_particles`（啊/呀/哦/嘛…）；`我叫李四啊` → `identity:李四`。
- **闸门补强（2026-08-06 Joe 拍板 1A/2A/3A/4A）**：
  - **1A** `_UNSAFE_FACT` 补 `password|passwd|pwd`
  - **2A** `_ROLE_SUFFIX` 去掉 `理/生`（放行 `查理`/`李生`），职衔靠 titles
  - **3A** 后缀加 `籍/裔`；titles 加护士/顾问/导演/助理…
  - **4A** `_is_safe_fact` 扩到偏好/身份路径
- **Finding 2（已修，采纳扁平）**：prompt 元数据从 `metadata.prompt{…}` 扁平为顶层 `prompt_name/prompt_version/prompt_label/prompt_source`。理由：LangFuse UI metadata 过滤只认顶层键；langfuse-radar DESIGN §6 默认 `prompt_meta_key=prompt_name` 直接命中（本地 LangFuse 实测 round-trip 通过）。
