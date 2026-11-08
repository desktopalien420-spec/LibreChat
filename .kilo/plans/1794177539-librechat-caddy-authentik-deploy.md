# Plan: Deploy LibreChat on chat.plutoniumcloud.com via docker-compose + Caddy + Authentik

## Goal

Run LibreChat as a web app at `chat.plutoniumcloud.com` on a Linux server using Docker Compose (single host), Caddy as reverse proxy/TLS terminator, and Authentik for authentication. Admin group users supply their own API keys; regular users pick from an admin-chosen model list in the config panel; search enabled.

## Context (verified from repo)

- `deploy-compose.yml`: `api` (image `librechat-dev-api`, port 3080), `admin-panel`, `client` (nginx, 80/443), `mongodb` (mongo:8.0.20), `meilisearch`, `vectordb` (pgvector), `rag_api`. Services talk over compose network by service name.
- `Dockerfile.multi` builds `data-provider`, `data-schemas`, `api`, `client`, `chat`, then runs `node server/index.js` on 3080. The published image already contains the built client dist, so no `npm run build` is needed at deploy time.
- Config: `librechat.yaml` (copy `librechat.example.yaml`) + `.env` (copy `.env.example`).
- `getAppConfig({ role })` (packages/api/src/app/service.ts) returns role-scoped endpoint models — admin picks models via the admin panel/config overrides, regular users receive that set. No custom source change needed for "admin chooses, users pick from list."
- Provider keys default to `user_provided` (OpenAI, Anthropic, Google) — users bring their own keys via the UI. Admin can pin keys in `.env`/`librechat.yaml` if desired.
- `CREDS_KEY`/`CREDS_IV` and `JWT_SECRET`/`JWT_REFRESH_SECRET` default to temp generated values — set persistent random values before production.
- `ADMIN_PANEL_SESSION_SECRET` required for bundled admin panel (`openssl rand -hex 32`).
- `SEARCH=false` by default — enable `SEARCH=true` + `MEILI_MASTER_KEY`.
- Auth: `ALLOW_EMAIL_LOGIN=true`, `ALLOW_REGISTRATION=true` by default. Authentik handles auth (OpenID Connect) per user requirement.

## Decisions

| Decision | Value |
|---|---|
| Domain | `chat.plutoniumcloud.com` |
| Reverse proxy / TLS | Caddy (terminates TLS, routes to compose) |
| Host runtime | Docker Compose single host, `deploy-compose.yml` |
| API keys | Users bring their own (`user_provided` default); admin may pin keys optionally |
| Model gating | Role-scoped config via admin panel (ADMIN sets model list; USER sees it) |
| Auth | Authentik (OpenID Connect), not LibreChat email/password |
| Search | Enabled (`SEARCH=true` + `MEILI_MASTER_KEY`) |
| Persistence | Named volumes `librechat-data`, `pgdata2`; bind mounts `data-node`, `uploads`, `logs`, `skill`, `images`, `meili_data` |

## Data flow

Browser (TLS) -> Caddy (`chat.plutoniumcloud.com`) -> `client` nginx (SPA + API proxy to `api:3080`) -> Express API -> MongoDB (history), Meilisearch (search), pgvector/rag_api (file search/RAG), provider LLMs (user-supplied keys). Admin panel on `admin-panel:3000` -> `api` for config/user/role management. Authentik OIDC handles login/session; LibreChat email login disabled.

## Ordered tasks

1. **Provision host**: Docker + Docker Compose installed on the Linux server; `chat.plutoniumcloud.com` DNS points to the server; Caddy installed and capable of terminating TLS for that domain.
2. **Copy config**: `cp librechat.example.yaml librechat.yaml` and `cp .env.example .env` in `/workspace/app` (or the deploy directory on the server).
3. **Generate secrets**: `ADMIN_PANEL_SESSION_SECRET=$(openssl rand -hex 32)`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `CREDS_KEY`, `CREDS_IV`, `MEILI_MASTER_KEY`.
4. **Edit `.env`**: set `HOST=0.0.0.0`, `NODE_ENV=production`, `MONGO_URI=mongodb://mongodb:27017/LibreChat`, `DOMAIN_CLIENT=https://chat.plutoniumcloud.com`, `DOMAIN_SERVER=https://chat.plutoniumcloud.com`, `SEARCH=true`, `MEILI_MASTER_KEY=<generated>`, `ALLOW_EMAIL_LOGIN=false`, `ALLOW_REGISTRATION=false`, `ALLOW_SOCIAL_LOGIN=true`, Authentik OIDC vars (`OPENID_CLIENT_ID`, `OPENID_CLIENT_SECRET`, `OPENID_ISSUER`, etc.), provider keys optional (`OPENAI_API_KEY=user_provided`, etc.), `ADMIN_PANEL_SESSION_SECRET`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `CREDS_KEY`, `CREDS_IV`, `TRUST_PROXY=1` (Caddy is a reverse proxy).
5. **Edit `librechat.yaml`**: enable search, set `version`, `interface` flags, `endpoints:` with models the admin wants available; keep provider keys as `${ENV_VAR}` references so they can be rotated without YAML edits.
6. **Caddyfile**: add `chat.plutoniumcloud.com` block with `reverse_proxy` to the `client` nginx container (localhost:80) or directly to `api:3080` for `/api` and to `client` for static SPA; enable TLS (automatic or via your existing certs); add admin panel route if desired.
7. **Authentik OIDC app**: create an application/provider in Authentik for LibreChat, redirect URI `https://chat.plutoniumcloud.com/api/auth/openid/callback`, copy client ID/secret/issuer into `.env`.
8. **Bring up stack**: `docker compose -f deploy-compose.yml up -d`.
9. **Validate**: `curl https://chat.plutoniumcloud.com/api/config` returns config; register/login via Authentik; admin sets model list in admin panel; regular user sees only that list; send a chat turn and receive a streamed reply; admin panel loads at its URL.
10. **Backup**: schedule backups of `data-node` (Mongo) and `pgdata2` (pgvector/RAG) volumes.

## Failure modes & rollback

- `api` won't start: bad `librechat.yaml` or missing `CREDS_KEY`/`JWT_SECRET` — check `docker compose logs api`.
- Chat turn 500s: provider key missing/invalid or `ENDPOINTS` env doesn't include the provider — check provider env vars and `endpoints:` section.
- Auth fails: Authentik redirect URI/client secret mismatch — verify against Authentik app config.
- Rollback: `docker compose down`, restore previous `.env`/`librechat.yaml`, data persists in named volumes.

## Validation

- All containers healthy after `docker compose -f deploy-compose.yml up -d`
- `curl https://chat.plutoniumcloud.com/api/config` returns endpoints/interface flags (200)
- Authentik login succeeds and issues a LibreChat session
- Admin can set available models in config panel; regular user's model selector shows only that list
- A chat turn streams a reply from a configured provider
- Admin panel reachable and can read users/config

## Open questions (confirm before implementation)

- Where does the deploy directory live on the server (e.g. `/opt/librechat` vs `/workspace/app` copied over)? Affects bind-mount paths and Caddy upstream targets.
- Does Caddy manage TLS certificates automatically (HTTP-01/DNS-01) or are certs provided manually?
- Should the admin panel be exposed at a separate subdomain (e.g. `admin.chat.plutoniumcloud.com`) or only reachable internally?
- Any specific provider models the admin wants pre-listed in `librechat.yaml` `endpoints:` for regular users?

## Out of scope

- Custom source-code changes to LibreChat
- Kubernetes / helm
- Multi-node rollout
- TLS certificate issuance detail (assumes Caddy or your edge manages it)
