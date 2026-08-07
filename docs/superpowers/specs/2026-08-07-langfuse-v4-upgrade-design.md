# Langfuse OSS v4 + Python SDK v4 升级设计

> **状态:** 实现中（2026-08-07 Joe 批准方案 1；本地 wipe）  
> **拍板:** 方案 1 官方完整栈；目标 **Server OSS v4 + Python SDK v4（≥4.7，取当前最新）**；本地 **wipe 重建**；prod 同步拓扑。  
> **非目标:** 迁移旧 v2 trace；Slice 2 记忆剩余；Langfuse Cloud；第一期不做 CH 集群。

## 1. 背景与问题

| 现状 | 问题 |
|------|------|
| `ghcr.io/langfuse/langfuse:2` 单容器 | Server **OSS v2 = EOL** |
| `langfuse>=2,<3` + `langfuse.decorators` | SDK v2；在 OSS v4 上 **Unsupported** |
| 与业务共用 postgres 上的 `langfuse` 库 | v3+ 架构已变；不宜继续「塞一个 web」 |

官方版本矩阵（自学要点）：**Server 与 SDK 独立 major**；最新自托管 GA 是 **OSS v4**；配套 Python **SDK v4 GA**（v3 Deprecated）。  
「升到 3+」**不等于最新**——要最新应对齐 **v4 + v4**。

## 2. 目标

1. 本地/prod 跑官方推荐的 **Langfuse v4 多服务拓扑**（web + worker + ClickHouse + MinIO + 专用 PG + 专用 Redis）。
2. 应用代码切到 **Python SDK v4**，保留现有 `@observe` / sampling / prompt_service / enrich_span 语义（兼容层包装）。
3. 文档写清：**起步 compose ≠ 生产伸缩终态**；附 Phase 2 扩展计划与官方坑。
4. 本地可 wipe；seed 仍用 `LANGFUSE_INIT_*`（key 尽量保持现有 `pk-lf-local-contextgate` / `sk-lf-local-contextgate` 减少改 env）。

## 3. 架构（Phase 1 — 起步）

```
NexusAI API ──SDK v4──► langfuse-web:4 (:3001→3000)
                              │
                         langfuse-worker:4
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         ClickHouse      Langfuse Redis    MinIO/S3
         (observations)   (BullMQ)         (events/media)
              │
         Langfuse Postgres (metadata / auth)
```

**隔离原则（重要）:**

- Langfuse **不要**复用业务 `redis-stack`（密码、maxmemory-policy、模块不同；官方 Redis 要 `noeviction` + requirepass）。
- Langfuse Postgres **建议独立服务或至少独立卷/库**；避免与业务 pgvector 抢连接与升级节奏。
- UI 端口保持开发习惯：`localhost:3001`。

**Compose 落点:**

- 改 `docker-compose.local.yml` / `docker-compose.prod.yml`：删除 `langfuse:2`；并入官方 v4 服务集合（可裁剪官方示例中的重复 postgres/redis，换成我们命名的 `langfuse-postgres` / `langfuse-redis`）。
- Image 钉 `docker.io/langfuse/langfuse:4` 与 `langfuse-worker:4`（或当时最新 4.x digest）；ClickHouse / MinIO 跟官方 compose 对齐。

## 4. 代码改动面

| 文件 | 变更 |
|------|------|
| `pyproject.toml` / `uv.lock` | `langfuse>=4.7.0,<5` |
| `backend/observability/decorators.py` | `from langfuse import observe, get_client`；去掉 `langfuse.decorators` |
| `backend/observability/langfuse_client.py` | v4 client / flush / discard；`LANGFUSE_BASE_URL` 与 `LANGFUSE_HOST` 对齐 |
| `backend/core/prompt_service.py` | 核对 `get_prompt`；保持静默降级 + 只缓存成功 |
| `backend/pipeline/router.py` / `graph.py` / harness | parent trace / `update_current_trace` / observation API 迁到 v4 |
| `config.env.example`、`examples/qa/LANGFUSE.md`、learning 观测文 | 拓扑、env、排障 |

**兼容策略:** 对外仍暴露 `observe` / `langfuse_context` / `enrich_span` / `get_langfuse`，内部适配 v4，减少全仓改注解成本。

## 5. Phase 2 — 扩展计划（学习用，本期不实现）

官方 docker-compose 是 **单机起步拓扑**。用量上来后 **要扩展**，且按组件独立扩：

| 压力信号 | 先扩什么 | 怎么做 |
|----------|----------|--------|
| ingestion 延迟、队列堆积 | **langfuse-worker** | 水平加副本（最常见第一步） |
| UI/API 慢 | **langfuse-web** | 加副本 + LB |
| 分析查询慢、写入打满 | **ClickHouse** | 先垂直升配 → 再集群/分片 |
| 队列不稳 | **Redis** | 托管 Redis / HA；保持与业务 Redis 隔离 |
| 事件/媒体膨胀 | **对象存储** | MinIO → 云 S3 |
| 元数据连接打满 | **Postgres** | 连接池 / 升配（通常最晚） |

**产品侧减负（比盲目扩容更便宜）:**

- 继续用现有长短路径 **采样**
- SDK v4 OTel 会吸附大量基础设施 span → **按 instrumentation scope 过滤**（官方专门提醒，否则账单/负载虚高）
- retention、字段截断、`LANGFUSE_OBSERVATION_FIELD_*`

**部署演进示意:**

```
Phase 1: 单机 compose（本设计）
    → Phase 2a: worker/web 多副本
    → Phase 2b: 托管 CH + S3 + Redis
    → Phase 2c: K8s/Helm（官方有 chart）按需
```

## 6. 官方文档坑（必读）

1. **版本名陷阱:** 「升到 3」不是最新；自托管 GA 是 **v4**；Python 也应 **SDK v4**（官方建议 v2 用户直接升到 v4）。
2. **Server≠SDK major:** 矩阵独立；OSS v4 **不支持** Python SDK v2。
3. **compose ≠ 可无限扛量:** 多容器是功能完整，不是自动弹性；worker/CH 要单独规划。
4. **SDK v4 = OTel-native:** 会收录其它库的 span；不滤会拖垮自托管与费用（Cloud）。
5. **实时 ingestion:** Python 建议 **≥4.7.0**；过旧的 v4 patch 在 v4 数据模型上可能延迟约 10 分钟。
6. **环境变量:** v4 倾向 `LANGFUSE_BASE_URL`；旧 `LANGFUSE_HOST` / `LANGFUSE_BASEURL` 兼容窗口有限，文档双写对齐。
7. **Redis 策略:** 官方要求队列 Redis `noeviction`；与业务 cache Redis（常 LRU）**不可混用**。
8. **数据模型:** v4 observations-first；旧 Public API 读路径大量 Deprecated → 排障脚本要改 Observations/Metrics v2。
9. **本地升级:** v2→v4 无平滑「只改 tag」；**wipe 重建** 是开发环境正解（本期拍板）。
10. **init seed:** `LANGFUSE_INIT_*` 仅首次；wipe 后靠 env 重建项目/key，勿假设旧 DB 里的 key 还在。

权威入口:

- [Versions & Compatibility](https://langfuse.com/self-hosting/upgrade/versioning)
- [Upgrade v3→v4 (self-host)](https://langfuse.com/self-hosting/upgrade/upgrade-guides/upgrade-v3-to-v4)
- [Python SDK upgrade path](https://langfuse.com/docs/observability/sdk/upgrade-path)
- 官方 compose: `https://github.com/langfuse/langfuse/blob/main/docker-compose.yml`

## 7. 迁移步骤（实现时）

1. 文档/spec 定稿（本文）→ 实现计划  
2. wipe 本地 langfuse 相关卷；改 compose；起栈；UI + seed key 冒烟  
3. `uv add` SDK v4；改 observability 兼容层；跑单测  
4. 真链路：chat 长路径出 trace；`prompt_service` 降级/命中  
5. 更新 QA/learning 文档；prod compose 同步  
6. commit（不含无关脏文件）

## 8. 验收

- [ ] `langfuse-web` / `worker` / clickhouse / minio / langfuse-redis / langfuse-postgres healthy  
- [ ] UI `http://localhost:3001` 可登录（init user）  
- [ ] 应用 `LANGFUSE_*` 能打出 chat 长路径 trace（含节点 span）  
- [ ] `get_prompt` 在未建 prompt 时 builtin 降级不 500  
- [ ] ruff + 相关 pytest 通过  
- [ ] 本文 Phase 2 / 官方坑 已进仓库，可供面试/运维讲解  

## 9. 决策记录

| 项 | 选择 |
|----|------|
| 拓扑 | 方案 1：官方完整栈并入 compose |
| 版本 | Server v4 + SDK v4 最新 |
| 本地数据 | wipe |
| 扩展 | Phase 1 单机；Phase 2 文档化，本期不实现 |
| Redis/PG | 与业务隔离 |
