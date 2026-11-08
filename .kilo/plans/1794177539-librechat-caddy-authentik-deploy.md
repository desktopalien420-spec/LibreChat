# Plan: Deploy LibreChat at chat.plutoniumcloud.com via docker-compose + Caddy + Authentik

## Goal

Run LibreChat as a web app at `chat.plutoniumcloud.com` on a Linux server using Docker Compose (single host), Caddy as reverse proxy/TLS terminator (auto ACME), and Authentik for authentication. Admin group users bring their own API keys; regular users pick from an admin-chosen model list. Search enabled.

## Context (verified from repo)

- `deploy-compose.yml`: `api` (image `librechat-dev-api`, port 3080), `admin-panel`, `client` (nginx, 80/443), `mongodb` (mongo:8.0.20), `meilisearch`, `vectordb` (pgvector), `rag_api`. Services talk over compose network by service name.
- `Dockerfile.multi` builds all packages and runs `node server/index.js` on 3080. Published image contains built client dist; no `npm run build` needed at deploy time.
- Config: `librechat.yaml` (copy `librechat.example.yaml`) + `.env` (copy `.env.example`).
- `getAppConfig({ role })` (packages/api/src/app/service.ts) returns role-scoped endpoint models — ADMIN sets the model list via admin panel/config overrides; USER sees only that list. No source changes needed.
- Provider keys default to `user_provided` — users bring their own keys via the UI. Admin can pin keys optionally.
- `CREDS_KEY`/`CREDS_IV`, `JWT_SECRET`/`JWT_REFRESH_SECRET` default to temp values — set persistent random values before production.
- `ADMIN_PANEL_SESSION_SECRET` required for admin panel (`openssl rand -hex 32`).
- `SEARCH=false` by default — enable `SEARCH=true` + `MEILI_MASTER_KEY`.
- Auth: `ALLOW_EMAIL_LOGIN=true`, `ALLOW_REGISTRATION=true` by default — disable both; use Authentik OIDC.

## Decisions (all resolved)

| Decision | Value |
|---|---|
| Domain | `chat.plutoniumcloud.com` |
| Deploy directory | `/opt/librechat` on the Linux server |
| Reverse proxy / TLS | Caddy, auto ACME (HTTP-01), ports 80/443 |
| Host runtime | Docker Compose single host, `deploy-compose.yml` |
| API keys | Users bring their own (`user_provided` default) |
| Model gating | Role-scoped config via admin panel (ADMIN sets list; USER sees it) |
| Initial models | OpenAI gpt-4o-mini/gpt-4o; Anthropic claude-3-5-sonnet/claude-3-haiku; Google gemini-2.5-flash |
| Auth | Authentik OpenID Connect; LibreChat email/password login & registration disabled |
| Admin panel | `admin.chat.plutoniumcloud.com` behind Caddy + Authentik |
| Search | Enabled (`SEARCH=true` + `MEILI_MASTER_KEY`) |
| Persistence | Named volumes `librechat-data`, `pgdata2`; bind mounts `data-node`, `uploads`, `logs`, `skill`, `images`, `meili_data` |

## Data flow

Browser (TLS) -> Caddy -> `client` nginx (SPA + `/api` proxy to `api:3080`) -> Express API -> MongoDB (history), Meilisearch (search), pgvector/rag_api (file search/RAG), provider LLMs (user-supplied keys). Admin panel at `admin.chat.plutoniumcloud.com` -> `api`. Authentik OIDC handles login/session.

## Ordered tasks (implementation)

1. **Provision host**: Docker + Docker Compose on the server; `chat.plutoniumcloud.com` and `admin.chat.plutoniumcloud.com` DNS point to the server; ports 80/443 reachable.
2. **Copy repo files to `/opt/librechat`**: `docker-compose.yml`, `deploy-compose.yml`, `Dockerfile.multi`, `librechat.example.yaml`, `.env.example`, `client/nginx.conf`.
3. **Generate secrets**: `ADMIN_PANEL_SESSION_SECRET=$(openssl rand -hex 32)`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `CREDS_KEY`, `CREDS_IV`, `MEILI_MASTER_KEY`.
4. **Write `.env`** (from `.env.example`): `HOST=0.0.0.0`, `NODE_ENV=production`, `MONGO_URI=mongodb://mongodb:27017/LibreChat`, `DOMAIN_CLIENT=https://chat.plutoniumcloud.com`, `DOMAIN_SERVER=https://chat.plutoniumcloud.com`, `SEARCH=true`, `MEILI_MASTER_KEY=<generated>`, `ALLOW_EMAIL_LOGIN=false`, `ALLOW_REGISTRATION=false`, `ALLOW_SOCIAL_LOGIN=true`, Authentik OIDC vars (`OPENID_CLIENT_ID`, `OPENID_CLIENT_SECRET`, `OPENID_ISSUER`, `OPENID_CALLBACK_URL=/api/auth/openid/callback`, etc.), provider keys `OPENAI_API_KEY=user_provided`, `ANTHROPIC_API_KEY=user_provided`, `GOOGLE_KEY=user_provided`, `ADMIN_PANEL_SESSION_SECRET`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `CREDS_KEY`, `CREDS_IV`, `TRUST_PROXY=1`, `ADMIN_PANEL_URL=https://admin.chat.plutoniumcloud.com/admin`.
5. **Write `librechat.yaml`** (from `librechat.example.yaml`): set `version`, `interface` flags, `endpoints:` with the initial models (referencing `${ENV_VAR}` for keys), search block, and optional role config.
6. **Caddyfile**: `chat.plutoniumcloud.com` -> reverse_proxy to `client` nginx (localhost:80) for SPA and `api:3080` for `/api`; `admin.chat.plutoniumcloud.com` -> `admin-panel:3000`; enable auto TLS.
7. **Authentik OIDC app**: create provider/application in Authentik, redirect URI `https://chat.plutoniumcloud.com/api/auth/openid/callback`, copy client ID/secret/issuer into `.env`.
8. **Bring up stack**: `docker compose -f deploy-compose.yml up -d` in `/opt/librechat`.
9. **Validate**: `curl https://chat.plutoniumcloud.com/api/config` returns 200; Authentik login succeeds; admin sets model list in admin panel; regular user's model selector shows only that list; chat turn streams a reply; admin panel loads at its URL.
10. **Backup**: schedule backups of `data-node` (Mongo) and `pgdata2` (pgvector/RAG) volumes.

## Failure modes & rollback

- `api` won't start: bad `librechat.yaml` or missing `CREDS_KEY`/`JWT_SECRET` — check `docker compose logs api`.
- Chat turn 500s: provider key missing/invalid or `ENDPOINTS` env doesn't include provider — check env vars and `endpoints:` section.
- Auth fails: Authentik redirect URI/client secret mismatch — verify against Authentik app config.
- Rollback: `docker compose down`, restore previous `.env`/`librechat.yaml`, data persists in named volumes.

## Out of scope

- Custom source-code changes to LibreChat
- Kubernetes / helm
- Multi-node rollout
- ACME DNS-01 detail (HTTP-01 assumed)

## Open question (non-blocking)

- Confirm the Authentik issuer URL and any role/group mapping (which Authentik group maps to LibreChat ADMIN role vs USER role) before step 7.
