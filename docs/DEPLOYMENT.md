# Deployment

## Local

```bash
cp config.env.example config.env
docker compose -f docker-compose.local.yml up -d
uv sync
uv run python -c "from backend.database.pgvector_session import PGVectorSession; PGVectorSession().init_db()"
uv run python scripts/seed_api_keys.py
uv run uvicorn backend.app:app --reload
```

## Production checklist

- [ ] Set strong `DB_PASSWORD`, `NEXTAUTH_SECRET`, `SALT`
- [ ] Set `LLM_KEY_MASTER_KEY` (32-byte hex): `python -c 'import secrets; print(secrets.token_hex(32))'`
- [ ] Seed / create tenant LLM keys via Admin API (do not ship plaintext in images)
- [ ] TLS certs under `deploy/ssl/`
- [ ] `docker compose -f docker-compose.prod.yml up -d --build`
- [ ] Verify `/health` and `/playground/playground.html`
- [ ] Restrict CORS (`CORS_ALLOW_ALL=false`)
- [ ] Confirm audit retention and backup

## Health

```bash
curl -s http://localhost:8000/health
```
