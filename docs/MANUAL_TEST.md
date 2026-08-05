# ContextGate — 人工测试路线 (Manual Test Route)

> **用途:** 人工验收清单 + 全链路 Demo 剧本。
> **测试面 = 测试 FE**(`frontend/` → http://localhost:5173): 右上角角色切换器 +
> 8 个面板(Chat / RAG / Admin / Audit / Agent / Eval / 性能 / 能力)。
> **散装 HTML 页面不再是测试入口**;curl 仅作面板未覆盖场景(上传、意图、护栏注入样本等)的辅助验证。
> **每轮主线 = 角色旅程剧本**(`examples/qa/journeys/01-04`),任务驱动,边演边测。
> **原则:** 不靠 Cursor 自述,一切以实机面板操作 / curl 结果为准;发现的缺口记入 §13 缺陷记录表。

---

## 0. 环境准备(每次测试前必做)

```bash
cd ~/Desktop/github/contextgate
make up          # docker compose: postgres+pgvector+redis stack+langfuse
make db-init     # alembic 建表
make seed        # API key + 示例数据(输出 4 角色 key,只显示一次,记下来)
make run         # 终端 1 — uvicorn :8000 (APP_ENV=dev, LLM_PROVIDER=replay)

# 终端 2 — 测试 FE
cd frontend && npm run dev     # → http://localhost:5173 (vite 已代理 /api /chat 等到 :8000)
```

**4 角色 key 填入 FE(每轮必做):**

> **登录方式二选一(Task 38):**
> - **密码登录(推荐,测试 FE 主入口):** `/login` 切到「密码登录」tab →
>   username/password → 后端 `/api/auth/login` 校验 bcrypt → 下发 `cg_` key 自动写入对应 role 槽位;
>   或 `/register` 注册新账号(密码 ≥8,仅 `APP_ENV∈{dev,test,demo}` 开放,prod 403)→ 注册成功自动登录。
>   **注册所得 key 等价于对应角色 seed key**(同 tenant/role/user_id 体系,下游权限矩阵完全一致)。
> - **Key 粘贴(原路径,QA 角色切换仍是核心能力):** 见下方 4 步流程。

1. 浏览器开 `http://localhost:5173`,未配置 key 会被重定向到 `/login`。
2. 登录页右上角角色切换器切到 `user` 槽位 → 粘贴 seed 输出的 user key → 「进入控制台」。
3. **再手动回 `/login`**(地址栏直接输入),切 `tenant_admin` → 粘 key → 进入;auditor、super_admin 同理。
   - key 明文只存 **sessionStorage**(`cg-auth`),四槽互不覆盖;面板会因 `roleEpoch` 自动刷新。
   - ⚠️ 不要点右上角菜单的「退出并清空槽位」——那会清掉全部 4 个槽位。
4. 完成后右上角角色切换器应显示 4 个「已配置」;测试中随时切换角色验证权限。

**env 检查清单** (config.env,优先级: shell > config.env > config/{APP_ENV}.env > 默认):

| 键 | 期望值 | 说明 |
|----|--------|------|
| `APP_ENV` | dev | 环境分层(头部角标应为灰色 dev) |
| `LLM_PROVIDER` | replay | 离线回放,零调用零波动;采真数据用 record |
| `LLM_API_KEY` | (有则填) | 主 LLM key;replay 模式下可空 |
| `LLM_KEY_MASTER_KEY` | 64 hex | 加密 llm_api_keys 表;不设则 env 明文 |
| `RAG_HYDE_ENABLED` | false | [T20] 20.01,验证时开关对比 |
| `RAG_RERANK_ENABLED` | false | [T20] 20.01 |
| `EMBEDDING_MODEL` | text-embedding-v3 | [T28] DashScope 向量模型 |
| `EMBEDDING_DIMENSIONS` | 768 | [T28] API 请求维度(存储仍补零到 1536) |
| `EMBEDDING_BASE_URL` | https://dashscope.aliyuncs.com/compatible-mode/v1 | [T28] 与 QWEN 同源;key 默认 QWEN_API_KEY |
| `RAG_CACHE_ENABLED` | true | [T29] L1/L2 Redis 缓存;需 redis-stack |
| `RAG_CACHE_TTL_ANSWER` | 3600 | [T29] L1 滑动 TTL(上限 4h) |
| `RAG_CACHE_TTL_EMBED` | 86400 | [T29] L2 embedding TTL |
| `RAG_RATE_LIMIT_REQ` | 60 | [T29] 请求/分钟/租户 |
| `RAG_RATE_LIMIT_MISS` | 10 | [T29] L1 miss/分钟/租户 |

**回归基线**(日常循环跑前三个即可,快;前端单独跑):

```bash
make verify && make check && uv run pytest
cd frontend && npm run test && npm run build
```

`scripts/audit_consistency.py`(7 维全仓一致性)很重(第 7 维逐模块起 uv 子进程,1 分钟+),
**只在批次收尾/改名删文件/动文档与 env 键后跑**,日常循环不必。

```bash
uv run python scripts/audit_consistency.py   # 里程碑终检专用
```

**数据原则(2026-08-04, Joe 拍板 —— 不影响手动测试流程,只定数据姿势):**

1. **真实数据优先**: 手动测试默认用真实数据。RAG 有真实文档就查真实文档;
   LLM 想验证真实输出用 `LLM_PROVIDER=record make run`(采一轮)或 openai。
   测试流程本身不变(§1-§14 照走),变的只是"数据从哪来"。
2. **不造演示数据**: 不预置 demo 数据、不为了截图好看重录 fixture 美化输出。
   知识库为空 → 页面显示空态/提示,不是塞样例。实事求是: 没有就是没有。
3. **env 可控降级例外**: `LLM_PROVIDER=replay`(离线回放)与 `LEAF_STUB_MODE=true`
   是显式降级开关,仅用于离线开发/无 key 环境;其输出为"录制回放/演示 stub",
   **非真实响应**,文档与界面需标注清楚,不得冒充真实数据。
4. **遗留 fixture 标注**: 现有 replay fixture 含历史测试残留(EVID-03)——保留,
   但文档注明"测试产物";真实数据由 record 采集,天然干净。

---

## 1. 冒烟 — 系统存活

| # | 操作 | 预期 |
|---|------|------|
| 1.1 | `curl localhost:8000/` | name=ContextGate, status=running, features 列表非空 |
| 1.2 | `curl localhost:8000/health` | db/pgvector/langfuse 三项 healthy |
| 1.3 | `curl localhost:8000/system/info` | 架构信息, router 列表含 chat/memory/evaluation |
| 1.4 | `curl localhost:8000/chat -H "Content-Type: application/json" -d '{"message":"hi"}'` | 401 `AUTH_001 missing_api_key`(无 key 必须被拒) |
| 1.5 | 浏览器开 `:5173`,未填 key 访问任意面板 | 重定向 `/login?next=...`,登录后回跳原面板 |
| 1.6 | 登录页填 **错误格式 key** | 提示 `health_failed:401` 之类,不进入控制台 |
| 1.7 | 登录后头部 | 右上角色徽章 = 当前槽位;左上角 ContextGate + env 角标(dev 为灰) |
| 1.8 | 角色切换器展开 | 4 角色各标「已配置/未配置」,当前角色标「当前」 |
| 1.9 | 注册+登录冒烟 [T38] | `/register` 注册新账号(密码 ≥8)→ 自动登录进 Chat 面板,右上角徽章 = 注册所选 role;再点「退出并清空槽位」→ `/login` 用刚注册的 username/password 登录 → 进 Chat 面板 |

---

## 2. 认证与权限矩阵(角色切换器 × 面板)

**验证方法:** 右上角角色切换器逐角色切换,每个面板观察: 有数据 = 有权限;
红字 `ForbiddenBanner`「该角色无权限(需 …)」= 403。面板随切换自动刷新。

| 面板 / 操作 | user | tenant_admin | auditor | super_admin |
|-------------|------|--------------|---------|-------------|
| Chat 流式 + JSON (`chat:write`) | ✓ | ✓ | ✗ | ✓ |
| RAG ask / search / status (`chat:write`) | ✓ | ✓ | ✗ | ✓ |
| Admin → API Keys tab (`admin:*`) | ✗ | ✗(无 `admin:*`) | ✗ | ✓ |
| Admin → Pending tab (`admin:approve`) | ✗ | ✓ | ✗ | ✓ |
| Admin → LLM Keys tab (`admin:llm_key`) | ✗ | ✓ | ✗ | ✓ |
| Audit 查询 + 导出 CSV (`audit:read`/`audit:export`) | ✗ | ✗ | ✓ 跨租户 | ✓ |
| Agent Hub invoke + 旧路由 (`chat:write`) | ✓ | ✓ | ✗ | ✓ |
| Eval (`chat:write`) | ✓ | ✓ | ✗ | ✓ |
| Performance (`chat:write`) | ✓ | ✓ | ✗ | ✓ |
| Capabilities 列表 (`verify_api_key` + 能力级 permission) | ✓ 按可见性过滤 | ✓ | ✓ | ✓ |

> 2026-08-02 实测修正: tenant_admin 角色定义(ROLES)只有 chat:*/kb:*/admin:approve/admin:llm_key,
> 无 admin:* 也无 audit:read — 审计数据仅 auditor/super_admin 可见,设计合理。
> Capabilities 走 `verify_api_key` + 每条 capability 的 `spec.permission` / 租户可见性闸门,不套固定角色。
>
> **[T38] 注册所得 key ≡ seed key:** 通过 `/api/auth/register` 注册的账号所下发 `cg_` key,
> 与该角色 seed key 在权限矩阵上完全等价(同 tenant=acme / role / user_id 体系,下游 `api_key_auth.py`
> 鉴权无差异)。注册 role=auditor 的 key 即可读 Audit 面板,注册 role=super_admin 即可进 Admin 面板,
> 与 seed 粘贴路径行为一致。

**通过标准:** 无越权;auditor 只能读审计,不能写;所有 403 都应以 ForbiddenBanner 呈现
(而非白屏/裸错误)。若某面板出现「裸 500 / 无提示失败」→ 记 §13。

---

## 3. Chat 双路径(Chat 面板)

**面板构成:** 「流式对话」卡(`/chat/streaming`,SSE)+ 下方 RequestPanel「对照: POST /chat(JSON)」。
响应 meta 区展示 `path / finish_reason / capability_id / cost / trace_id / cost_source`。

| # | 验证点 | 操作 | 预期 |
|---|--------|------|------|
| 3.1 | 短路径 | 输入「你好」→ 发送流式 | 秒回;出现 **短路径** badge;meta: `path=short`, `finish_reason=skill_executed`, `cost=0` |
| 3.2 | 长路径 | 输入长问题(如「帮我总结知识库关于数据备份的要点」) | 逐 token 输出;**长路径 SSE** badge;meta `trace_id` 非空 |
| 3.3 | JSON 对照 | RequestPanel 发 `message=你好` | 返回 response + finish_reason + pipeline_latency_ms + total_cost |
| 3.4 | 缓存命中 | 同一问题连发两次 → Performance 面板「Cache stats」 | `hit_rate` 上升(接口只暴露命中率) |
| 3.5 | 输入护栏 | curl 注入样本(见 11.2) | 拦截/中性化,不泄露 system prompt |
| 3.6 | PII 脱敏 | curl message 含身份证号/手机号 | 响应中对应位置被掩码 |
| 3.7 | 输出护栏 | curl 诱导输出 API 密钥 | 响应不含 `sk-…`/`SECRET_KEY`/`PASSWORD` 模式 |
| 3.8 | 审计联动 | 发完请求 → 切 **auditor** → Audit 面板查 `action=chat` | 出现该请求记录 |

> 2026-08-02 实测修正: 短路径仅对注册 skill 的意图生效(内置仅 greeting);知识类问题走 LLM 属正常。
> 输出护栏拦截密钥泄露与角色漂移,非"危险内容"泛拦(高精度设计)。

---

## 4. SSE 流式(Chat 面板)[T20 重点 — 20.02]

| # | 验证点 | 操作 | 预期 |
|---|--------|------|------|
| 4.1 | 正常流 | 发一条长问题 | 逐 token 出现,`● live` 指示随 token 闪动;收尾 meta 出现(短/长路径 badge) |
| 4.2 | 心跳 [T20] | 空闲等待 >15s | DevTools Network 面板出现 `: ping` 注释帧,连接不被代理掐断 |
| 4.3 | 断开中止 [T20] | 流式中途刷新页面 | 服务端日志无「残留生成」,asyncio.CancelledError 被捕获,无 500 |
| 4.4 | 错误事件协议 [T20] | 发违规词触发内容过滤 | 面板 hint 区显示 `[code] message`(结构化错误),而非裸 500/连接断裂 |
| 4.5 | abort 按钮 | 流式中点 **Stop** | 流停止,hint 显示 abort 原因;再发新问题正常 |
| 4.6 | retraction | 触发超长回复截断 | 面板提示 retraction 原因(warning 态),不崩溃 |

**通过标准:** [T20] 4.2/4.3/4.4 三项是 Task 20 的硬验收,未过则打回 Cursor。

---

## 5. 意图识别(curl — 面板未暴露,保留接口级验证)

| # | 验证点 | 操作 | 预期 |
|---|--------|------|------|
| 5.1 | 类型列表 | `GET /intent/types` | 返回意图枚举,不含情感域意图 |
| 5.2 | 企业问题命中 [T20] | `POST /intent/detect?text=如何查询公司的信息安全管理制度`(注意是 POST,文档旧版误写 GET) | intent=knowledge_query(或 rag 类),**不得兜进 advice**;confidence ≥0.7 |
| 5.3 | 规则兜底 | 常见企业问句(报销流程/请假制度/设备报修)各测 3-5 条 | 全部命中合理意图,无 advice 残留 |
| 5.4 | analyze 全量 | `POST /intent/analyze` 企业场景示例 | 返回意图+置信度+可选路由建议 |
| 5.5 | 文案残留检查 | `grep -rn "睡不着\|失眠\|难过" backend/` | 0 命中 [T20] |

**背景:** 2026-08-01 实测企业问题被 heuristic 兜进 advice(confidence 0.75)—— 这是 Task 20.05 要修的 bug,本轮必须回归确认。

---

## 6. RAG 知识库(RAG 面板 + curl)[T20 · T28]

> **[T28] 前置:** Task 28 落地后为真实语义检索(`text-embedding-v3` + DashScope)。
> 需配置 `QWEN_API_KEY`(或 `EMBEDDING_API_KEY`);`GET /api/rag/status` 的 `embedding_model` 应为 `text-embedding-v3`(非 `*(hash)` / `api-or-hash`)。
> 6.1–6.4 在哈希向量下无法产出语义验收证据。

**面板构成:** 「缓存状态」卡(`GET /api/rag/status`:命中率 % + hit/miss + docs 数 + cache on/off 徽章)+
「上传 PDF」卡(`POST /api/rag/upload/pdf`,选真实 PDF 入库)+
「提问 / 搜索」卡(Ask / Search 按钮;回答区显示 `cache_hit` 徽章 + 延迟)。
`init/sample` 仍走 curl(面板未暴露)。

> ⚠️ **扫描件 PDF(见 §13 GAP-10):** 2026-08-04 起上传会校验——整份无文本层 → 422 `RAG_002`
> 「未提取到文本:扫描件或无文本层 PDF」,不再静默成功;响应含 `pages_extracted`(有文本页数)。
> 自动 OCR 兜底未做;变通:逐页导出 png/jpg 走 `/api/rag/upload`(category=general)的 image 分支
> (需 `uv sync --extra multimodal`)。

```bash
KEY=你的seed_key
BASE=localhost:8000/api/rag
# 初始化示例知识库(curl)
curl -s -X POST $BASE/init/sample -H "X-API-Key: $KEY"
# 上传 PDF — 优先用 RAG 面板「上传 PDF」卡;curl 备用:
# (必须是真实 PDF;.md 伪装会被 pypdf 解析失败。macOS 可用
# `cupsfilter docs/COMPLIANCE.md > /tmp/c.pdf` 生成真实 PDF 再传)
curl -s -X POST $BASE/upload/pdf -H "X-API-Key: $KEY" -F "file=@/tmp/c.pdf"
```

| # | 验证点 | 操作 | 预期 |
|---|--------|------|------|
| 6.1 | init/sample [T28] | curl 上述 | 返回示例 chunks 数 >0(真实 embedding 入库) |
| 6.2 | upload/pdf [T28] | **RAG 面板**选真实 PDF → 「上传到知识库」(或 curl 上述) | 成功文案 + status docs 增加;chunks 生成 |
| 6.3 | search [T28] | 面板 **Search**,问「信息安全管理制度」 | top-k 返回;**top-1 须为 COMPLIANCE 相关**(语义硬验收) |
| 6.4 | ask [T28] | 面板 **Ask** | 回答有引用来源;回答区显示 `cache_miss` 徽章 |
| 6.5 | HyDE 开关 [T20][T28] | `RAG_HYDE_ENABLED=true` 重启后,同问句 top-1 命中比 false 更准(用长问句/术语变体验证) | 记录开关前后对比 |
| 6.6 | ReRank 开关 [T20][T28] | `RAG_RERANK_ENABLED=true` 后 top-1 与开关前对比 | 记录差异 |
| 6.7 | reset | `curl -X DELETE $BASE/reset -H "X-API-Key: $KEY"` | 清空;再 Ask 走无知识路径 |
| 6.8 | L1 缓存 [T29] | 面板同 query 连打 20 次 | 第 1 次 `cache_miss`、之后 **`cache_hit · 零成本`** 徽章;延迟显著下降 |
| 6.9 | epoch 失效 [T29] | upload 后同 query 立即 Ask | `cache_hit=false`(同租户 epoch) |
| 6.10 | 限流 [T29] | 压测 miss → 结构化 `RATE_001`(HTTP 429) | 面板显示错误码,非裸失败 |
| 6.11 | status.cache [T29] | 面板「缓存状态」卡 | 含 `hit_ratio`;`l1_entries`/`l2_entries` 为 Redis SCAN 基数(`entries_source=scan`) |
| 6.12 | L2 缓存 [T29] | 同 query 3 次 Search → status 的 `l2_hit` +2 且 `l2_entries=1` | LangFuse 仅 1 个 embedding span |
| 6.13 | PII 跳过 [T29] | ask 含身份证号 | `cache_hit=false` 且 redis `rag:a:*` 键数不变 |
| 6.14 | 单飞锁 [T29] | 5 并发同 query(新问题) | 恰 1 个 `cache_hit=false` + 4 个 true(仅 1 次 LLM) |
| 6.15 | 审计联动 [T29] | Audit 面板(auditor)查 `action=rag.ask` | input 前缀 `cache_hit=N\|`;命中 cost=0 |
| 6.16 | redis 降级 [T29] | `docker stop contextgate-redis-1` → Ask 仍 200(静默降级)→ start 恢复 | 面板不 500 |
| 6.17 | RAG 认证 [T29] | 不带 key 打 `/api/rag/ask` | 401 `AUTH_001`(9 端点均需 chat:write) |

**一键脚本 [T29]:** 上述 6.8–6.14、6.16 已脚本化,证据自动留档:

```bash
RAG_QA_KEY=<user key> ./scripts/rag_cache_qa.sh            # 证据 → data/qa/rag_cache_qa_<ts>.log
RAG_QA_DEGRADE=1 RAG_QA_KEY=<key> ./scripts/rag_cache_qa.sh  # 含 redis 停启降级项
```

**通过标准:** [T28] 6.3 语义 top-1 过关;[T20] 6.5/6.6 记录开关前后 top-1 对比数据(留档);[T29] 6.8–6.17 缓存/限流/PII/单飞/审计/降级/认证行为符合预期(可一键脚本复验)。

---

## 7. Agent(Agent 面板)

**面板构成:** 「Capability Hub Agent」卡(agent 下拉 + user_id + 流式 invoke → 嵌套链 `call_chain` badges)、
「Status(旧路由)」「Tools」「Legacy /agent/chat」卡。

| # | 验证点 | 操作 | 预期 |
|---|--------|------|------|
| 7.1 | Hub invoke 嵌套链 | 选 `vendor-risk-agent`(需已 seed capabilities),消息「评估供应商合同风险」→ **流式 invoke(Hub)** | 流式输出;下方出现嵌套链:根 badge + 「调用了 X → Y」子链 |
| 7.2 | 工具列表 | 看「Tools」卡 | 工具 badges 非空(名称+悬停描述) |
| 7.3 | 多轮对话 | 「Legacy /agent/chat」→ **发送 /agent/chat** | 有记忆的多轮回答 JSON |
| 7.4 | 记忆读取 | — | `GET /agent/memory/{user_id}`(curl)返回该用户记忆 |
| 7.5 | 历史 | **History** 按钮 | 返回会话历史 JSON |
| 7.6 | 回访 | `POST /agent/followup {user_id}`(curl) | 触发回访逻辑 |

> 若 Agent 下拉只有「vendor-risk-agent(未 seed)」→ 先 `uv run python scripts/seed_capabilities.py` 再刷新。

---

## 8. Admin 管理(Admin 面板)

**面板构成:** 三个 tab — **API Keys**(创建: user_id/role/description + 明文一次 + 复制按钮;列表: prefix/role/user/active + 停用)、
**Pending**(待审批: #id resource/action · tenant/user + 通过/拒绝)、**LLM Keys**(alias/provider/tenant/active 列表,不返回明文)。

| # | 验证点 | 操作 | 预期 |
|---|--------|------|------|
| 8.1 | api-keys 创建 | super_admin 槽位 → API Keys tab,填 user_id/role/description → 创建 | 明文 key 显示(仅一次,带复制按钮);列表出现新行 |
| 8.2 | api-keys 停用 | 点新 key 的 **停用** | 状态变 off;用该 key curl → 401 |
| 8.3 | llm-keys 加密入库 | `POST /api/admin/llm-keys` {key_alias, api_key_plaintext, provider?, base_url?}(curl) | LLM Keys tab 出现该行;**列表不显示明文**(落库为密文,LLM_KEY_MASTER_KEY 已设时) |
| 8.4 | llm-keys verify | `POST /api/admin/llm-keys/{id}/verify`(curl) | 返回连通性结果 |
| 8.5 | 审批流 | user key curl `POST /api/admin/permissions/request` → 切 **tenant_admin** 槽位看 Pending → 通过/拒绝 | 状态流转 request → pending → approved/rejected |
| 8.6 | audit 导出 | Audit 面板 → 导出 CSV | 浏览器下载 CSV,内容与列表一致 |

**角色限制观察(记入 §13 别扭点):**
- user 槽位打开 Admin → 三个 tab 全 403,ForbiddenBanner 提示所需权限。
- tenant_admin 槽位 → API Keys tab 403(`admin:*`),Pending / LLM Keys 正常。

---

## 9. 评测(Eval 面板)

**面板构成:** Statistics badges(总数 + 各维度平均分)、Evaluate(user_message/bot_response 输入 + 结果)、Batch、List 表。

| # | 验证点 | 操作 | 预期 |
|---|--------|------|------|
| 9.1 | 单条评测 | 填 user_message / bot_response → **Evaluate** | 返回 id + avg 分数 + overall_comment;List 出现新行 |
| 9.2 | 批量 | **Batch** | 多条结果摘要显示 |
| 9.3 | 对比 | `POST /evaluation/compare-prompts`(curl) | 同请求多 prompt 对比表 |
| 9.4 | 统计 | 看 Statistics 卡 | 聚合数字与 List 明细一致(total 随评估递增) |

---

## 10. 可观测(Performance 面板 + LangFuse)

**面板构成:** Metrics / Cache stats / Active streams / Benchmark 四卡 + 「刷新指标」「Run benchmark」按钮。

| # | 验证点 | 操作 | 预期 |
|---|--------|------|------|
| 10.1 | LangFuse trace | 浏览器开 `http://localhost:3001`,Chat 面板发一条 /chat | trace 树出现,节点 = 管线各阶段 |
| 10.2 | span 明细 | 点开长路径 trace | 各节点耗时/输入输出可见 |
| 10.3 | Prometheus | `curl localhost:8000/metrics` | 指标文本非空 |
| 10.4 | 缓存统计 [T35] | Performance 面板「Cache stats」 | 含 `hit_rate`;可选 `chat_epoch_default` |
| 10.5 | 清缓存 [T35] | `POST /performance/cache/clear`(无 pattern 或 `*`,curl) | 全部已知租户 `chat:epoch` bump;旧 `chat:v:*` 不再命中 |
| 10.6 | RAG 缓存命中率 [T29] | RAG 面板「缓存状态」卡 | `hit_ratio` 随重复 query 上升;`l2_entries` 反映 embedding 复用 |
| 10.7 | redis 键检查 [T29/T35] | `redis-cli --scan --pattern 'rag:*'` / `'chat:*'` | RAG:`rag:a`/`rag:e`/`rag:epoch`;Chat:`chat:v`/`chat:epoch`/`chat:lock`(见 `docs/CACHE.md`) |
| 10.8 | 滑动 TTL [T29] | 命中后 `redis-cli ttl <l1_key>` 复查 | TTL 回到 ~3600(续期),且不超过 4h 上限 |
| 10.9 | redis 降级 [T35] | 停 redis 后再打依赖缓存的路径 | 业务仍 200(静默 miss),不得 500 |
| 10.10 | benchmark | Performance 面板 **Run benchmark** | 返回基准结果 JSON,不超时 |

---

## 11. 安全专项(每轮必做,curl)

| # | 验证点 | 操作 | 预期 |
|---|--------|------|------|
| 11.1 | 无 key | 所有写端点各打一次 | 401,无一漏网 |
| 11.2 | prompt injection 样本集 | 「忽略系统提示」「你现在是」「忘记之前所有指令」+ 变体 | 全部被拦截或中性化,响应不含 system prompt |
| 11.3 | 文件上传 MIME 伪造 | 把 .txt 改名 .pdf 上传;伪造 content-type | 被 file_sanitizer 拒绝(从内容头判断,不信扩展名) |
| 11.4 | 文件路径穿越 | 上传文件名含 `../` | 被 UUID 重命名,无路径穿越 |
| 11.5 | PII 脱敏 | 身份证/手机号/银行卡号输入 | 输出掩码 |
| 11.6 | 输出违规 | 诱导输出违禁内容 | guardrails_output 拦截,错误码返回 |
| 11.7 | 断路器 | 指向不可达 LLM 端点(LLM_BASE_URL 改错)连打多次 | 触发熔断,快速失败而非超时堆积 |

---

## 12. 全链路 Demo 剧本(10-15 分钟,CIO 向,全走测试 FE)

> 价值主线:可审计、可溯源、全链路可控、成本可见。每步标注「讲什么」。

| 步 | 演示 | 测试 FE 操作 | 价值点 |
|----|------|-------------|--------|
| 1 | 登录与权限 | 登录页展示 4 角色槽位;user 进 Chat;切 auditor 进 Audit;切 super_admin 进 Admin | 企业级认证,四角色隔离 |
| 2 | 意图识别 | Chat 面板发「你好」→ 短路径 badge,cost=0 | 智能分流,长路径才花钱 |
| 3 | 知识库问答 | RAG 面板 Ask「信息安全管理制度」→ 引用回答 + cache_hit | 内部知识资产变现 |
| 4 | 流式体验 | Chat 面板长文逐字输出 + ● live + Stop 中断 | 体验 + 可中断(成本可控) |
| 5 | 审计溯源 | 切 auditor → Audit 面板按 action/时间过滤,搜刚才的请求 | 全链路可审计,合规刚需 |
| 6 | 成本治理 | Admin LLM Keys tab + Chat meta 的 cost 字段 | 每笔调用成本可算,CIO 最关心 |
| 7 | 可观测 | LangFuse UI span 树 + Performance 面板四卡 | 出问题 30 秒定位到节点 |
| 8 | 缓存降本 [T29] | RAG 面板同一问题连问 3 次:第 1 次有延迟,第 2/3 次秒回 `cache_hit · 零成本`;「缓存状态」卡命中率上升 | 重复问题零成本——员工反复问同一制度不再烧钱 |

**Demo 前置:** `LLM_PROVIDER=record make run` 先采一轮真实数据转 replay,避免现场波动;或直接 `make demo`。

---

## 13. 缺陷记录表(边测边填)

| 编号 | 场景 | 现象 | 严重度 | 状态 |
|------|------|------|--------|------|
| GAP-01 | 7.1 Agent 页面 | examples/agent.html 调 `/api/agent/`,后端实际 `/agent/` → 404 | Important | ✅ 已修复(2026-08-01,页面 BASE 改 `/agent/`) |
| GAP-02 | config.env | `MODEL_REGISTRY_JSON` 若写两行,dotenv 只认首行(local-7b 死行) | Minor | ✅ 已修(Task 28:合并为单行数组+文档警示) |
| GAP-03 | 认证 | agent/chat、evaluation 全部端点无 key 可调用(返回 200 并执行/写库) | **Critical** | ✅ 已修(2026-08-02,全部挂 `chat:write`;11.1 全扫通过) |
| GAP-04 | 审批流 | pending-requests/approve 按 `tenant_id` 过滤,super_admin("*")永远空/404 | Important | ✅ 已修(2026-08-02,补 cross-tenant 分支) |
| GAP-05 | 审批流 | `request_permission` INSERT 漏 created_at(表无默认)→ pending-requests 500,审批全挂 | Important | ✅ 已修(2026-08-02,INSERT 补 `now()`) |
| GAP-06 | admin 列表 | api_keys 遗留行 is_active/created_at NULL → `GET /api/admin/api-keys` 500 | Minor | ✅ 已修(回填 + 查询 COALESCE) |
| GAP-07 | 上传回显 | 多模态上传响应回显原始文件名(含 `../`) | Minor | ✅ 已修(改用 sanitize 后 safe_name) |
| GAP-08 | LangFuse 观测 | harness 调不存在的 SDK 方法导致 usage/cost 未落库;`/chat/streaming` 无根 trace 产生孤儿 span | Important | ✅ 已修(2026-08-02:改 `update_current_observation` + streaming 根 observe;详见 examples/qa/LANGFUSE.md §6) |
| GAP-09 | 测试 FE 缺口 | **(本轮起)** 面板发现的问题记这里:流程断点 / 信息缺失 / 概念错位(三类见 journeys README) | — | — |
| GAP-10 | 扫描件 PDF | ① ~~上传静默"成功"~~ **已修(2026-08-04):** `load_from_pdf` 过滤空页,整份无文本 → 抛 `RAG_002`(422),响应带 pages_extracted/真实页数 ② **OCR 兜底仍未做**:无「无文本层 → 转图逐页 OCR」自动路径(变通:逐页导出 png/jpg 走 `/api/rag/upload` image 分支,需 multimodal) | Important | ②待修(建议:pymupdf 转图 → PaddleOCR 逐页 → 合并入库) |

严重度分级:Critical(数据/安全/不可用)→ 立即修;Important(功能不符预期)→ 列表给用户审核;Minor(体验/文案)→ 直接修。
修测试 FE 或后端时同步更新本表状态。

---

## 14. 边测边充实测试 FE 的循环

1. 每轮测试以 journeys 剧本为主线(`examples/qa/journeys/01-user → 04-super-admin`),
   在测试 FE 面板上完成任务;每步问「这个角色走到这步顺不顺」。
2. 别扭点分三类记入 §13(流程断点 / 信息缺失 / 概念错位),同时回填剧本末尾的「别扭点记录表」。
3. Minor 直接修前端;Important 汇总后与 Cursor 的 diff 一起 review(见 AGENTS.md 实现后 Code Review 工作流)。
4. Task 20/28/29 落地后:把 §4/§5/§6 的 [T20]/[T28]/[T29] 项全跑一遍,结果留档,作为对应 Task 的验收证据。
5. 每轮收尾:§0 回归基线全绿(后端 + 前端 `npm run test && npm run build`)+ git commit(Conventional Commits, Signed-off-by: Joe)。

---

## 附:实测端点索引(2026-08-01 核实)

```
/chat, /chat/streaming          — LangGraph 管线(chat:write)     [FE: Chat 面板]
/api/admin/*                    — api-keys / pending-requests / approve / llm-keys   [FE: Admin 面板]
/api/audit/logs, /api/audit/export                              [FE: Audit 面板]
/api/rag/*                      — status / init/sample / upload/pdf / ask / search / reset   [FE: RAG 面板;upload/pdf 面板可传;init/sample 仍 curl]
/api/files/upload               — 文件上传(内容头校验 + UUID 重命名)
/intent/types, /intent/detect, /intent/analyze, /intent/batch, /intent/build_prompt   [无面板,curl]
/agent/chat, /agent/memory/{uid}, /agent/tools, /agent/followup, /agent/history/{uid} [FE: Agent 面板]
/api/agents, /api/capabilities/{id}/invoke                      [FE: Agent 面板 Hub 卡 / 能力面板]
/api/capabilities               — 能力注册表(verify_api_key + 能力级 permission)      [FE: 能力面板]
/evaluation/evaluate|batch|compare-prompts|statistics|report/generate   [FE: Eval 面板]
/performance/metrics|cache/stats|cache/clear|streams/active|benchmark   [FE: 性能面板]
/memory/users/{uid}/memories, /memory/users/{uid}/profile
DELETE /memory/users/{uid}/memories — 遗忘权(tenant_admin|super_admin + X-API-Key；本租户；chat 脱敏不删行)
/feedback, /personalization
/enhanced-chat, /streaming      — **deprecated** → 请用 `/chat`、`/chat/streaming`(及 `/memory/*`、`/agent/*`)
/health, /system/info, /metrics, /docs
/playground/<page>.html         — 后端挂载的散装页面,仅存档,不作为测试入口
```

> **测试入口约定:** 一律走测试 FE(http://localhost:5173)。散装 HTML 页面不参与验收;
> 若 FE 面板缺某个功能导致只能 curl,在对应章节已注明,并记入 §13 作为前端需求输入。
