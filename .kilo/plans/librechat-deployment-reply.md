# LibreChat Deployment Plan — chat.plutoniumcloud.com

**Status:** Implementation-ready (all decisions resolved; one non-blocking open question)  
**Target directory:** `/opt/librechat`  
**Stack:** Docker Compose (`deploy-compose.yml`) + Caddy (auto-ACME TLS) + Authentik (OIDC)

---

## Decisions (all resolved)

| Decision | Value |
|---|---|
| Main domain | `chat.plutoniumcloud.com` |
| Admin domain | `admin.chat.plutoniumcloud.com` |
| Deploy directory | `/opt/librechat` on the Linux server |
| Reverse proxy / TLS | Caddy, auto ACME (HTTP-01), ports 80/443 |
| Runtime | Docker Compose single host (`deploy-compose.yml`) |
| API keys | Users bring their own (`user_provided` default) |
| Model gating | Role-scoped via admin panel (ADMIN sets list; USER sees subset) |
| Initial models | OpenAI (gpt-4o-mini, gpt-4o); Anthropic (claude-3-5-sonnet, claude-3-5-haiku, claude-3-haiku); Google (gemini-2.0-flash, gemini-2.5-flash, gemini-1.5-pro) |
| Auth | Authentik OpenID Connect; LibreChat email/password login & registration disabled |
| Search | Enabled (`SEARCH=true` + `MEILI_MASTER_KEY`) |
| Persistence | Named volumes `librechat-data`, `pgdata2`; bind mounts `data-node`, `uploads`, `logs`, `skill`, `images`, `meili_data` |

---

## Plan contents (summary)

1. **Prerequisites** — Docker + Compose, Caddy, DNS A records for both domains pointing to the server, ports 80/443 open.
2. **Clone & prepare** — `git clone` LibreChat into `/opt/librechat`, copy `.env.example` → `.env` and `librechat.example.yaml` → `librechat.yaml`.
3. **Secrets** — One-liner to generate `ADMIN_PANEL_SESSION_SECRET`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `CREDS_KEY`, `CREDS_IV`, `MEILI_MASTER_KEY`, `OPENID_SESSION_SECRET`.
4. **`.env` template** — Production values with `REPLACE_ME` placeholders for Authentik client ID/secret/issuer and generated secrets. Disables local email login/registration; sets provider keys to `user_provided`; `TRUST_PROXY=1`; `ADMIN_PANEL_URL=https://admin.chat.plutoniumcloud.com`.
5. **`librechat.yaml`** — Bootstrap config with the curated model list per provider. Admin panel can override role-scoped model lists later without restarting.
6. **Caddyfile** — Server blocks for `chat.plutoniumcloud.com` and `admin.chat.plutoniumcloud.com` with security headers and streaming/flush support; proxies to the nginx `client` container on port 80.
7. **nginx.conf adjustment** — Change `server_name` to `admin.chat.plutoniumcloud.com` (admin) and `chat.plutoniumcloud.com` (main); ensure `/api` is proxied to `api:3080`.
8. **Authentik OIDC app** — Create Application + OAuth2/OIDC Provider; redirect URIs `https://chat.plutoniumcloud.com/oauth/openid/callback` and `https://chat.plutoniumcloud.com/api/admin/oauth/openid/callback`; copy Client ID, Secret, and OpenID Configuration URL into `.env`.
9. **Bring-up** — `docker compose -f deploy-compose.yml pull` then `up -d`; Caddy obtains certificates automatically.
10. **Validation** — `curl` the `/api/config`, main UI, and admin panel endpoints; manual login/chat/admin-panel checks.
11. **Backup** — Cron example tarring `librechat_librechat-data` and `librechat_pgdata2` named volumes; also back up `.env` and `librechat.yaml`.
12. **Failure modes & rollback** — Table of symptoms/causes/fixes; `docker compose down` + restore config files.

---

## ⚠️ One correction needed before deploy

The plan's Caddyfile routes everything (including `/oauth/openid/callback`) through the nginx `client` container. In LibreChat, the Auth0-style OIDC callback is handled by the Express **API** service at `/api/auth/openid/callback`, not the SPA container.

**Fix:** Add an `/api` location block to `/opt/librechat/client/nginx.conf` proxying to `api:3080`, e.g.:

```nginx
location /api/ {
    proxy_pass http://api:3080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 300s;
}
```

Then `Caddy` can send `/api` to the client container (which forwards to the API) or directly to `api:3080`. Without this, Authentik login callbacks will 404.

---

## Open question (non-blocking, can be patched post-deploy)

Before production, confirm:
1. The exact Authentik issuer URL (the `/.well-known/openid-configuration` path) → goes in `OPENID_ISSUER`
2. Which Authentik group (if any) should map to LibreChat `ADMIN` vs `USER` role → configure via `OPENID_REQUIRED_ROLE` or promote users in the admin panel after first login

These do not block the initial bring-up.

---

## Files to place on the server

| File | Server location |
|---|---|
| `.env` | `/opt/librechat/.env` |
| `librechat.yaml` | `/opt/librechat/librechat.yaml` |
| Caddyfile | `/etc/caddy/Caddyfile` |
| `client/nginx.conf` | edit `server_name` + add `/api` proxy block |

After the first successful deploy, model lists and most UI flags can be managed from the Admin Panel without further file edits.

---

## Next steps

1. Copy the plan's `.env`, `librechat.yaml`, and Caddyfile sections onto the server.
2. Apply the nginx.conf `/api` proxy fix and `server_name` changes.
3. Run `docker compose -f deploy-compose.yml up -d` in `/opt/librechat`.
4. Validate with the checklist, then share the Authentik issuer URL + group mapping if you want the `.env` OIDC block refined.

---

## Appendix: OpenRouter rerank with nvidia/nemotron-3.5-rerank-8b:free

Use this to reorder retrieved documents by relevance to a query (e.g. for RAG/file-search result re-ranking in LibreChat).

### 1. Get your API key

Create an API key from your OpenRouter dashboard and set it as an environment variable:

```bash
export OPENROUTER_API_KEY=sk-or-v1-...
```

### 2. Make your first request

Use `nvidia/nemotron-3.5-rerank-8b:free` with the OpenRouter rerank API:

```python
import requests
import json

response = requests.post(
    url="https://openrouter.ai/api/v1/rerank",
    headers={
        "Authorization": "Bearer $OPENROUTER_API_KEY",
        "Content-Type": "application/json",
        "HTTP-Referer": "<YOUR_SITE_URL>",     # Optional. Site URL for rankings on openrouter.ai.
        "X-OpenRouter-Title": "<YOUR_SITE_NAME>", # Optional. Site title for rankings on openrouter.ai.
    },
    data=json.dumps({
        "model": "nvidia/nemotron-3.5-rerank-8b:free",
        "query": "a photograph of a cat",
        # Documents can mix images and text. Images are remote URLs or base64 data URIs.
        "documents": [
            {"image": "https://upload.wikimedia.org/wikipedia/commons/3/3a/Cat03.jpg"},
            {"text": "A fluffy cat sitting on a windowsill in the sun."},
            {"text": "A street map of downtown Berlin."}
        ],
        "top_n": 3
    })
)

results = response.json()
for result in results["results"]:
    document = result["document"]
    source = document.get("image") or document.get("text")
    print(f"Index: {result['index']}, Score: {result['relevance_score']}, Source: {source}")
```

### Endpoint reference

| Item | Value |
|---|---|
| Method | `POST` |
| URL | `https://openrouter.ai/api/v1/rerank` |
| Authorization | `Bearer $OPENROUTER_API_KEY` |
| Content-Type | `application/json` |
| HTTP-Referer | Optional — your site URL, for OpenRouter rankings |
| X-OpenRouter-Title | Optional — your site name, for OpenRouter rankings |
| Model | `nvidia/nemotron-3.5-rerank-8b:free` |

OpenRouter-specific headers (`HTTP-Referer`, `X-OpenRouter-Title`) are optional; setting them allows your app to appear on the OpenRouter leaderboards. For information about third-party SDKs and frameworks with OpenRouter, see the OpenRouter frameworks documentation.

