# Plan: Deploy LibreChat on a single Linux host via docker-compose

## Goal

Run LibreChat as a web app inside a website hosted on a Linux server, using Docker Compose as the single-host runtime. The app exposes a chat UI backed by an Express API, MongoDB, Meilisearch, pgvector (RAG/vector DB), and a bundled admin panel.

## Scope

In scope:
- Provision `librechat.yaml` and `.env` configuration
- Bring up the stack with `deploy-compose.yml` (production profile incl. nginx client) or `docker-compose.yml`
- Wire provider API keys so chat turns reach LLMs
- Expose the app behind a reverse proxy / domain (your website) on port 80/443
- Verify startup: API health, login/registration, a chat turn, admin panel reachable

Out of scope:
- Kubernetes / helm (declined in favor of single-host compose)
- TLS certificate issuance / ACME (assumes you manage certs at your reverse proxy or domain edge)
- Multi-node / multi-replica rollout
- Custom source-code modifications to LibreChat itself

## Context (verified from repo)

- `deploy-compose.yml` defines: `api` (image `librechat-dev-api`), `admin-panel`, `client` (nginx), `mongodb` (mongo:8.0.20), `meilisearch`, `vectordb` (pgvector), `rag_api`.
- `docker-compose.yml` is the dev/self-contained variant where `api` serves the client build in-process (no separate nginx container).
- The Docker image `api-build` stage (Dockerfile.multi) builds `packages/data-provider`, `data-schemas`, `api`, `client`, `chat`, then runs `node server/index.js` on port 3080.
- Config lives in `librechat.yaml` (copy `librechat.example.yaml`) and `.env` (copy `.env.example`).
- Provider keys default to `user_provided` in `.env.example` (OpenAI, Anthropic, Google) — meaning end users can supply their own keys via the UI, but you can hard-code admin-configured keys if desired.
- `MONGO_URI=mongodb://mongodb:27017/LibreChat`; services communicate over compose network by service name.
- `CREDS_KEY`/`CREDS_IV` and `JWT_SECRET`/`JWT_REFRESH_SECRET` should be set to persistent random values before production use (currently blank defaults to generated temp values).
- `ADMIN_PANEL_SESSION_SECRET` required for the bundled admin panel (`openssl rand -hex 32`).
- `SEARCH=false` by default; enable `SEARCH=true` + `MEILI_MASTER_KEY` if you want Meilisearch-backed chat/file search.

## Assumptions / decisions

1. **Compose file**: Use `deploy-compose.yml` (has dedicated nginx `client` container serving the SPA on 80/443, separate from API on 3080). Fall back to `docker-compose.yml` if you prefer the API to serve the client build directly.
2. **Hosting model**: The stack runs on your Linux server behind your existing website's reverse proxy (nginx/Caddy/Traefik/etc.). Compose publishes `api` on 3080, `client` (nginx) on 80/443, `admin-panel` on 3000. Your reverse proxy routes `chat.yourdomain.com` (or a path under your site) to the `client`/`api` containers.
3. **API keys**: Start with `user_provided` for OpenAI/Anthropic/Google so users bring their own keys; optionally pin admin keys in `.env`/`librechat.yaml` if you want to subsidize usage. Record this decision explicitly.
4. **Auth**: `ALLOW_EMAIL_LOGIN=true`, `ALLOW_REGISTRATION=true` by default. Adjust if you want invite-only or SSO (OpenID/SAML/GitHub/Google) instead.
5. **Persistence**: Named volumes `librechat-data` and `pgdata2`, plus bind mounts `./data-node`, `./uploads`, `./logs`, `./skill`, `./images`, `./meili_data_v1.35.1`. Back up `data-node` (Mongo) and `pgdata2` (pgvector/RAG) regularly.

## Data flow

Browser -> reverse proxy (80/443) -> nginx `client` container (serves SPA + API proxy to `api:3080`) -> Express API -> MongoDB (history), Meilisearch (search), pgvector/rag_api (file search/RAG), provider APIs (LLM turns via keys in `.env`/`librechat.yaml`).

Admin panel on 3000 -> `api` for config/user management.

## Failure modes & rollback

- If `api` crashes on startup: most likely bad `librechat.yaml` or missing `CREDS_KEY`/`JWT_SECRET`. Check `docker compose logs api`.
- If a chat turn 500s: provider key missing/invalid, or `ENDPOINTS` env doesn't include the provider. Check provider env vars and `endpoints:` section of `librechat.yaml`.
- Rollback: `docker compose down` + restore previous `.env`/`librechat.yaml` + named volumes; data persists in volumes.

## Validation steps

1. `docker compose -f deploy-compose.yml up -d` — all containers healthy.
2. `curl http://localhost:3080/api/config` returns config (endpoints, interface flags).
3. Register a user, log in, start a chat, send a message, receive a streamed reply from a configured provider.
4. Open admin panel (`http://localhost:3000` or your reverse-proxied admin URL) and confirm it loads and can read config.
5. (Optional) `docker compose logs -f api mongodb rag_api meilisearch vectordb` — no repeated crash loops.

## Open questions (to confirm before implementing)

- Which domain/path will serve the chat app, and which reverse proxy (nginx/Caddy/Traefik) will terminate TLS and route to compose?
- Will users bring their own API keys (`user_provided`, default) or will you hard-code provider keys?
- Email login + self-registration on, or invite-only / SSO?
- Enable Meilisearch search (`SEARCH=true`) or leave off?

## Next step

Confirm the open questions above, then I can finalize this plan and hand it to an implementation agent (or you) to execute the compose-up and config steps.
