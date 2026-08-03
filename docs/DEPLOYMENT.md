# Deployment

## Local

```bash
cp config.env.example config.env
docker compose -f docker-compose.local.yml up -d
uv sync
uv run alembic upgrade head
uv run python scripts/seed_api_keys.py
uv run uvicorn backend.app:app --reload
```

- `make up` — 本地依赖（postgres + redis；见 `docker-compose.local.yml`）
- `make up-all` — 完整生产向栈（`docker-compose.yml`：API、nginx、prometheus、grafana；**不含** Elasticsearch）

## 生产栈路径

- Nginx 配置与 TLS：`deploy/nginx.conf` + `deploy/ssl/`（自备 `cert.pem` / `key.pem`）
- Grafana 仅配置 Prometheus；日志聚合（ES）留待 V2.0

## Production checklist

- [ ] Set strong `DB_PASSWORD`, `NEXTAUTH_SECRET`, `SALT`
- [ ] Set `LLM_KEY_MASTER_KEY` (32-byte hex): `python -c 'import secrets; print(secrets.token_hex(32))'`
- [ ] Seed / create tenant LLM keys via Admin API (do not ship plaintext in images)
- [ ] TLS certs under `deploy/ssl/`
- [ ] `make up-all` 或 `docker compose up -d --build`
- [ ] Verify `/health` and `/playground/playground.html`
- [ ] Restrict CORS (`CORS_ALLOW_ALL=false`)
- [ ] Confirm audit retention and backup

## Health

```bash
curl -s http://localhost:8000/health
```
