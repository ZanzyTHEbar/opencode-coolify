# OpenCode SOTA — Full Implementation (Step-by-Step)

Exact decomposition to land the full SOTA setup on your existing Coolify OpenCode app. Execute in order.

**Quick status:** Repo https://github.com/ZanzyTHEbar/opencode-coolify should contain `docker-compose.yml`, `config/opencode.json` (single consolidated config), and `AGENTS.md.example`. Set env in Coolify, deploy, then first-run `/connect` and AGENTS.md.

---

## Phase 0 — Prerequisites

- Coolify OpenCode app exists (UUID `mk80kg8oo8kgkgw4kgcg840g`).
- Repo used by Coolify: https://github.com/ZanzyTHEbar/opencode-coolify (or your fork).
- You have: Coolify UI access, `coolify` CLI, and (optional) SSH to cool-res.

---

## Phase 1 — Repo and Compose

### Step 1.1 — Compose: no Traefik labels (same pattern as opencloud-compose)

**File:** `docker-compose.yml` (in this folder and in opencode-coolify repo).

Coolify **auto-generates** Traefik labels from domain assignments in the UI. The compose must **not** contain any Traefik labels (they conflict with Coolify's). Same approach as `opencloud-compose`: see `opencloud-compose/docs/coolify.md`.

**Compose content:**

- Bind-mount `./config:/root/.config/opencode:rw`, volumes `opencode_storage`, `opencode_mem`, and `/mnt/shared/common:/workspaces:rw`.
- Use **`expose:`** for `4096` and `4747` (no `ports:`); Coolify's proxy routes by domain and port.
- **No `labels:`** for Traefik or forward auth — Coolify adds those when you assign the domain and enable your Authentik Proxy (same as for OpenCloud and other protected apps).

See the updated `docker-compose.yml` in this repo for the full file.

### Step 1.2 — Push compose and full config to the repo Coolify uses

The compose bind-mounts **`./config`** → `/root/.config/opencode`. The repo Coolify clones must contain your **entire** `config/` directory (opencode.json, tui.json, skills/, rules/, command/, agent/, etc.). When Coolify runs compose from the clone, `./config` is that repo's `config/` folder.

If Coolify pulls from **opencode-coolify**, push the full config and compose there:

```bash
cd /path/to/opencode-coolify
# Copy from dragonserver/opencode:
# - docker-compose.yml (and optionally docker-compose.yaml)
# - authentik-path-proxy.conf  (for Authentik flow/static path proxy)
# - config/   (entire directory: opencode.json, tui.json, skills/, rules/, etc.)
# - AGENTS.md.example
git add docker-compose.yml authentik-path-proxy.conf config/ AGENTS.md.example
git commit -m "feat: bind-mount config, SOTA compose, Authentik path proxy"
git push origin main
```

---

## Phase 2 — Config Files (reference and one-time copy)

### Step 2.1 — Single config (opencode.json)

**File in repo:** `config/opencode.json` — single consolidated config (providers, model, plugin, instructions, compaction, agents, permissions, and opencode-mem settings under the `opencode-mem` key). Use it as the source for `OPENCODE_CONFIG_CONTENT` (minify for env) if you set config via env in Coolify.

### Step 2.2 — Config is bind-mounted (no one-time copy)

**`./config` is bind-mounted to `/root/.config/opencode`.** Everything in `opencode/config/` (opencode.json, opencode-mem.jsonc, tui.json, skills/, rules/, command/, agent/) is visible in the container. The main app only accepts schema-defined keys in `opencode.json`; the opencode-mem plugin reads its settings from **`opencode-mem.jsonc`** (separate file), so both files are required.

---

## Phase 3 — Coolify Environment

### Step 3.1 — Required env vars (secrets)

Set in Coolify → **Application** → **Environment** (or via CLI). Replace placeholders with real values.

| Key | Value | Notes |
|-----|--------|------|
| `OPENCODE_SERVER_USERNAME` | `opencode` | Optional when using Authentik only. |
| `OPENCODE_SERVER_PASSWORD` | *(leave unset or empty)* | **When using Authentik:** leave unset so the app does not prompt for basic auth; Authentik protects at the proxy. Only set a password if you need fallback auth. |
| `ANTHROPIC_API_KEY` | `sk-ant-...` | For Zen/Anthropic or direct. |
| `OPENAI_API_KEY` | `sk-...` | Optional; for OpenAI models or opencode-mem fallback. |

**CLI example:**

```bash
APP_UUID=mk80kg8oo8kgkgw4kgcg840g
coolify app env create $APP_UUID --key OPENCODE_SERVER_PASSWORD --value "YOUR_SECRET"
coolify app env create $APP_UUID --key ANTHROPIC_API_KEY --value "YOUR_ANTHROPIC_KEY"
# Update existing placeholder keys in Coolify UI if needed
```

### Step 3.2 — Optional: config via env (OPENCODE_CONFIG_CONTENT)

If you want to override config via environment instead of (or in addition to) the bind-mounted `config/opencode.json`, set **Key:** `OPENCODE_CONFIG_CONTENT` to a single-line minified copy of `config/opencode.json`. Prefer the bind-mounted file so one source of truth lives in the repo.

---

## Phase 4 — Deploy

### Step 4.1 — Redeploy

```bash
coolify deploy uuid mk80kg8oo8kgkgw4kgcg840g
```

Or: Coolify UI → OpenCode application → **Deploy**.

### Step 4.2 — Config already in container (bind mount)

With `./config` mounted as `/root/.config/opencode`, your repo's config (opencode.json, tui.json, skills, rules, agents) is already in the container. No one-time file creation needed.

### Step 4.3 — Restart (only if you change config after deploy)

So OpenCode loads the new plugin config:

```bash
# From cool-res or Coolify UI
coolify app restart mk80kg8oo8kgkgw4kgcg840g
# or redeploy
coolify deploy uuid mk80kg8oo8kgkgw4kgcg840g
```

---

## Phase 5 — First-Run in Web UI

### Step 5.1 — Domain and Authentik

1. **Coolify** → OpenCode application → assign domain (e.g. `https://opencode.zacariahheim.com:4096`). The compose includes Traefik forward-auth and `coolify.traefik.middlewares=opencode-auth@docker` so the router uses Authentik.
2. **Authentik path proxy:** The compose includes an `authentik-path-proxy` service so `/if/`, `/static/`, `/Lw/`, and `/outpost.goauthentik.io/` on the OpenCode host are proxied to Authentik (avoids flow/asset 404s). If your Coolify-assigned domain is not `opencode.zacariahheim.com`, set env `OPENCODE_DOMAIN` to that host (e.g. `OPENCODE_DOMAIN=opencode.example.com`) so the path router matches.
3. **Authentik:** Ensure OpenCode Proxy Provider exists and **OpenCode** is in your outpost's **Selected applications**. Run `opencode/create_authentik_proxy.py` in the Authentik container if needed.
4. **No basic auth:** Leave `OPENCODE_SERVER_PASSWORD` unset in Coolify so the app does not prompt for HTTP basic auth; Authentik protects at the proxy.
5. Open `https://opencode.zacariahheim.com` and log in via Authentik.

### Step 5.2 — Connect providers

In the OpenCode web UI chat, run:

```
/connect
```

Add credentials for:

- **OpenCode Zen** — [opencode.ai/auth](https://opencode.ai/auth), paste API key.
- **OpenCode Go** — Same auth; subscribe to Go and paste key.
- **GitHub Copilot** — Device flow at [github.com/login/device](https://github.com/login/device) if you use Copilot.

### Step 5.3 — Instructions (AGENTS.md)

In each **workspace** (e.g. repo under `/workspaces`), create or paste project rules so the `instructions` array can load them:

- **AGENTS.md** — Coding standards, architecture, conventions (see `AGENTS.md.example`).
- **docs/guidelines.md** — Optional.
- **.cursor/rules/*.md** — Optional; merged by the `instructions` glob.

You can run `/init` in OpenCode to generate a starter AGENTS.md, or copy from `AGENTS.md.example`.

### Step 5.4 — Plugins (opencode-mem, DCP)

**opencode-mem:** Start a short chat; then open the memory web UI. If port 4747 is exposed and the plugin loaded, use:
  - **Same host:** `https://opencode.zacariahheim.com:4747` (if you expose 4747 on the same host), or
  - **Traefik subdomain:** Add a router for e.g. `opencode-mem.zacariahheim.com` → port 4747 (Step 5.5).
- Or from cool-res: `curl -s http://localhost:4747` (if 4747 is bound on the host).

**DCP (Dynamic Context Pruning):** The plugin `@tarquinen/opencode-dcp@latest` is in `config/opencode.json`. It reduces token usage by deduplicating tool calls, purging stale errors, and exposing compress/discard tools to the model. Verify: in a chat run `/dcp` for command help; `/dcp context` for token breakdown. Config: `config/dcp.jsonc` (optional overrides; defaults are created on first run). See [OpenCodeDocs DCP](https://opencodedocs.com/Opencode-DCP/opencode-dynamic-context-pruning/start/getting-started/).

---

## Phase 6 — Optional: Traefik for Memory UI

To expose the opencode-mem web UI at a subdomain (e.g. `opencode-mem.zacariahheim.com`) with the same Authentik protection:

1. **Coolify:** Application → **Advanced** or **Labels** → add Traefik labels for a second router and service pointing at port 4747.
2. **Authentik:** Create a second Proxy Provider / Application for `opencode-mem.zacariahheim.com` (or reuse one and add the host).
3. **Labels (conceptual):** Service must expose 4747; router rule `Host(\`opencode-mem.zacariahheim.com\`)`, middleware = Authentik, entrypoints = websecure.

Exact label names depend on your Coolify/Traefik version; the compose in this repo does not add a second router by default so Coolify does not conflict. Add them in the UI or in a compose override if needed.

---

## Phase 7 — Verification Checklist

| # | Check | How |
|---|--------|-----|
| 1 | App responds on 4096 | Open app URL in browser. |
| 2 | Authentik (or password) works | Log in. |
| 3 | `/connect` shows Zen, Go, GitHub | Run `/connect` in UI. |
| 4 | `/models` lists Zen/Go models | Run `/models`. |
| 5 | Default model is Zen/Go | Start a message; model should be opencode/... or opencode-go/... |
| 6 | Instructions load | Add AGENTS.md in a workspace; ask agent "What are my project rules?" |
| 7 | opencode-mem loaded | Check logs for opencode-mem or open memory UI on 4747. |
| 8 | Orchestrator + reviewer available | Tab or @ to switch agents; reviewer should be read-only. |
| 9 | Option B: flow on auth domain | Incognito → open OpenCode URL → DevTools Network → first 302 **Location** starts with `https://auth.zacariahheim.com/`. |

---

## File Summary

| File | Purpose |
|------|--------|
| `docker-compose.yml` | Compose with config bind mount + storage + mem volumes, ports 4096 + 4747; includes authentik-path-proxy for flow/static paths. |
| `authentik-path-proxy.conf` | Nginx config for proxying /if/, /static/, /Lw/, /outpost.goauthentik.io/ to auth.zacariahheim.com. |
| `config/opencode.json` | Main config (providers, model, plugin, agents, etc.). |
| `config/opencode-mem.jsonc` | opencode-mem plugin config (main schema does not allow this key in opencode.json). |
| `config/dcp.jsonc` | DCP (Dynamic Context Pruning) plugin config; plugin creates defaults on first run. |
| `AGENTS.md.example` | Template for project AGENTS.md. |
| `IMPLEMENTATION.md` | This runbook. |

---

## Forward auth: Option A (path-proxy) vs Option B (flow on auth domain)

**Option B (flow on auth domain)** is preferred when it works: the login flow and all assets are served from `https://auth.zacariahheim.com`, so no path-proxy is required on the app host.

- **Requirement:** The embedded outpost's **authentik_host** must be the auth domain (e.g. `https://auth.zacariahheim.com`). The outpost then builds the authorization redirect to that host (see Authentik `endpoint.go`: for embedded outposts, browser URLs use `authentik_host`). Optionally set **authentik_host_browser** to the same URL (embedded outpost ignores it per [goauthentik/authentik#5922](https://github.com/goauthentik/authentik/issues/5922), but it keeps config consistent).
- **Verify Option B:** In an incognito window, open `https://opencode.zacariahheim.com`. In DevTools → Network, find the **first 302** response. The **Location** header should start with `https://auth.zacariahheim.com/` (e.g. `/application/o/authorize/` or `/if/flow/...`). If Location points to `opencode.zacariahheim.com`, the flow is still on the app host and Option A (path-proxy) is required.
- **Option A (path-proxy):** If Option B does not apply (e.g. redirect still goes to app host), the compose includes `authentik-path-proxy` so `/if/`, `/static/`, `/Lw/`, and `/outpost.goauthentik.io/` on the OpenCode host are proxied to Authentik. Ensure Traefik router priority (10000) and `OPENCODE_DOMAIN` match your setup.

**One-time Option B setup (already applied):** Outpost config was set so `authentik_host` and `authentik_host_browser` are `https://auth.zacariahheim.com`. The embedded outpost uses `authentik_host` for the auth redirect; after login the callback goes to `https://opencode.zacariahheim.com/outpost.goauthentik.io/callback`.

---

## Troubleshooting

**Authentik: stylesheet MIME type 'text/html', 404 for /static/, /Lw/session, /if/flow**

Forward-auth redirects the browser to the flow on the **app host** (e.g. `opencode.zacariahheim.com/if/flow/...`). The app doesn't serve those paths, so assets 404. The compose includes an **authentik-path-proxy** service: an nginx container that proxies `/if/`, `/static/`, `/Lw/`, and `/outpost.goauthentik.io/` to `https://auth.zacariahheim.com`. Traefik should route those path prefixes to this proxy (router priority 10000, Host with or without `:4096`). If it still fails: (1) Ensure both containers are running and on the `coolify` network. (2) In Traefik dashboard or `docker inspect`, confirm a router named `opencode-authentik-paths` exists and has higher priority than the main app router. (3) Some Coolify setups ignore custom router labels from compose; in that case add the path-proxy router via Coolify UI → Application → Advanced / Custom labels, or run Traefik in front of this stack yourself with the path rules. (4) Set `OPENCODE_DOMAIN` in Coolify env if your domain is not `opencode.zacariahheim.com`.

---

**File tree shows `/` instead of `/workspaces` (no projects/repos)**

The compose sets `working_dir: /workspaces` and mounts the host path at `/workspaces`. Ensure on the Coolify host (e.g. cool-res) the path `/mnt/shared/common` exists and contains your git repos (or create a symlink as in opencloud-compose). If the mount is empty, the UI will show an empty `/workspaces`. Redeploy after fixing the host path.

**Terminal fails: POST /pty 500 (Internal Server Error)**

The OpenCode container may lack PTY support or a required binary. Check container logs: `docker logs <opencode-container-id>`. If the image runs as non-root, PTY creation can fail; this is an upstream OpenCode/container issue. As a workaround, use `opencode attach <url>` from a host with the CLI.

---

**Deploy fails: "Bind for 0.0.0:4096 failed: port is already allocated"**

Something on the Coolify host is already using port 4096 (often a previous OpenCode container that didn't stop). On the host (e.g. SSH to cool-res):

```bash
# See what is using 4096
docker ps -a --format '{{.Names}}\t{{.Ports}}' | grep 4096
# or
ss -tlnp | grep 4096

# Stop the container holding the port (use name or ID from above)
docker stop <container_name_or_id>
docker rm <container_name_or_id>
```

Then redeploy from Coolify.

---

## Rollback

- Remove `OPENCODE_CONFIG_CONTENT` or set it to a minimal `{}` to revert to defaults.
- Remove opencode-mem volume and plugin from compose and redeploy to drop memory.
- Keep `opencode_config` and `opencode_storage` backups if you need to restore sessions/auth.
