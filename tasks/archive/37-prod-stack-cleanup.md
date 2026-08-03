# Task 37: 生产栈清理与一致性修复(deploy/monitoring)

> **状态: 已完成(2026-08-03)。** 来源: Task 16 产物路径不一致与空壳服务审查。
> **范围:** 修 deploy/monitoring 一致性 + 清空壳;**不重写** nginx SSE / prometheus / grafana 看板。
> **交付:** `c7d3c14` — align nginx mounts and drop unused ES stack。

## 子任务

| # | 内容 | 状态 |
|---|------|------|
| 37.01 | compose nginx 挂 `deploy/`（方案 A） | ✓ |
| 37.02 | 注释 ES/kibana + volume（留恢复路径） | ✓ |
| 37.03 | Grafana 只留 Prometheus | ✓ |
| 37.04 | `docs/DEPLOYMENT.md` 路径说明 | ✓ |

## AC

- [x] nginx volumes → `./deploy/nginx.conf` + `./deploy/ssl`
- [x] `docker compose config` 无 elasticsearch/kibana 服务
- [x] datasource.yml 无 elasticsearch 段
- [x] DEPLOYMENT.md 有 nginx/证书 + `make up` / `make up-all` 说明
- [x] 已 commit

## 手动验收(可选)

`make up-all` 后确认 nginx/prometheus/grafana healthy（需自备 `deploy/ssl` 证书）。
