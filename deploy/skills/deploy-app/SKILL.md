---
name: deploy-app
description: Deploy an application to PaperOS infrastructure. Triggers when the user asks to deploy a repo, app, codebase, or service — e.g. "deploy this", "ship this app to PaperOS", "/deploy <repo>". Provisions an LXC container, deploys the code, and runs a smoke test.
---

# Deploy App to PaperOS

This skill walks the deploy-mcp server's workflow correctly. The MCP server is not freeform — it deploys apps via **setup-script templates**. The server owns the templates and runs them; you pick a template and supply the **values** for its agent-fill block. You never construct or send a script.

## Critical rules (read first)

1. **You never write or send a setup script.** You pick a template name and supply a `template_values` dict. The server renders and runs it.
2. **ALWAYS call `list_templates` first** to discover available stacks. Don't assume names.
3. **ALWAYS call `get_template(name)`** before deploying — it returns the template content (read it to understand each value's purpose) and a `keys` list of what `template_values` can contain.
4. **Only the keys in the `keys` list (and nothing else)** belong in `template_values`. Unknown keys are rejected server-side.
5. **APP_NAME is automatic** — you pass it as the top-level `app_name` param to `deploy_app`. Don't also put it in `template_values`.
6. **Pick the template carefully.** Once chosen, the script that runs in the container is fixed — you cannot change install commands, systemd unit content, ports, or paths. If a template doesn't fit, ask the user before deploying.

## Workflow

### Step 1 — Discover templates

Call:
```
list_templates()
```

Returns something like `{"templates": ["go", "node", "python-fastapi"]}`. Always use this list — new templates may have been added without prompt updates.

### Step 2 — Detect the stack from the repo

Look at the repo's root for signals:

| File present | Use template |
|---|---|
| `package.json` | `node` |
| `requirements.txt` or `pyproject.toml` | `python-fastapi` |
| `go.mod` | `go` |
| `index.html` at root (and **none** of the above) | `static` |

If no template matches, **stop and ask the user** before improvising. Never deploy a stack you don't have a template for.

**Detecting `fullstack`:** if the repo contains BOTH a backend signal (one of the first three) AND a frontend signal (Vite/build tool, or a separate `index.html`-rooted folder, or a clear `apps/web` + `apps/api` split typical of monorepos), use the `fullstack` template. Common shapes:
- `apps/api/` + `apps/web/` (monorepo)
- `backend/` + `frontend/`
- A single Node repo with both an Express server and a Vite-built UI

When in doubt, ask the user before assuming fullstack — a misclassification can be confusing.

### Step 3 — Read the template to learn its values

Call:
```
get_template(name="<chosen_name>")
```

You'll get back `{"name": "<name>", "content": "<full shell script>", "keys": ["KEY1", "KEY2", ...]}`.

- `keys` lists the exact value-names you may put in `template_values` (APP_NAME is excluded because you pass it as the top-level `app_name` param).
- Read `content` to understand what each key means — per-key purpose lives in the comments next to each line in the agent-fill block.
- Everything outside that block is fixed by the server and not yours to influence.

### Step 4 — Determine each value

Use the repo to fill the keys. Derive `app_name` (the top-level deploy_app param) from the repo name — lowercase, hyphens, min 2 chars. The per-template keys below go in `template_values`.

The container has the runtimes pre-baked (Node, Go, system Python 3), so
the templates have no version field — you only fill in app-specific values.

**`node` template:**
- `ENTRY_FILE` — read `package.json#scripts.start` and extract the file (`"node server.js"` → `"server.js"`). Fallback: look for `server.js`, `index.js`, `app.js`, `main.js`.

**`python-fastapi` template:**
- `APP_MODULE` — find the FastAPI app entry. Usually `main:app` or `app.main:app`. Look for `app = FastAPI()` in the codebase.
- `WORKERS` — default `"2"` unless the user requested otherwise.

**`go` template:**
- `BINARY_NAME` — `"app"` is the safe default. Match what `go build` would produce.

**`static` template:**
- No values to fill — pass `template_values={}`.

**`vite` template:**
- No values to fill — pass `template_values={}`.

> *Identifying `vite` when uncertain:* you usually already know what build tool the project uses (you built the app with the user). If you need to verify from the repo, look for `vite.config.{js,ts}` at root or `vite` listed in `package.json`'s `dependencies` / `devDependencies`. Use this template for **any** Vite-based SPA — React, Vue, Svelte, vanilla — the source framework inside is irrelevant; from the pipeline's view it's all "npm run build → `dist/`".

**`spa` template** (catch-all for build-to-static cases that aren't Vite):
- `BUILD_CMD` — defaults to `npm run build`. Override only if the project's build script lives under a different name.
- `BUILD_DIR` — the directory the build produces, relative to repo root (no default — varies by tool).

> *When to pick `spa` vs `vite` vs `static`:*
> - `static` — vanilla HTML/CSS/JS, no build step.
> - `vite` — project uses Vite (see vite tips above).
> - `spa` — any other build-to-static case: CRA, Gatsby, Astro (static), Parcel, custom Webpack, etc.
>
> *Cheat sheet for `BUILD_DIR` per common tool:*
>
> | Tool | `BUILD_CMD` | `BUILD_DIR` |
> |---|---|---|
> | CRA (Create React App) | `npm run build` | `build` |
> | Gatsby | `npm run build` | `public` |
> | Astro (static) | `npm run build` | `dist` |
> | Parcel | `npm run build` | `dist` |
> | Custom Webpack | varies | usually `dist` or `build` |
>
> If unsure, check `package.json#scripts.build` for `BUILD_CMD`, and look at the build tool's config or output for `BUILD_DIR`.

**`fullstack` template** (one backend + one frontend in the same container):
- `BACKEND_TEMPLATE` — one of `node` / `python-fastapi` / `go`. Pick by the same signals as the standalone backend templates.
- `BACKEND_DIR` — subdir holding the backend code (e.g. `apps/api`, `backend`). Use `"."` if the backend lives at the repo root.
- `BACKEND_ENTRY` — stack-specific: node entry file, python `module:variable`, or go binary name. Same logic as the standalone backend templates.
- `BACKEND_WORKERS` — python-fastapi only; default `"2"`.
- `FRONTEND_TEMPLATE` — one of `static` / `vite` / `spa`. Pick by the same signals as the standalone frontend templates.
- `FRONTEND_DIR` — subdir holding the frontend code (e.g. `apps/web`, `frontend`). Use `"."` if the frontend lives at the repo root.
- `FRONTEND_BUILD_OUTPUT` — for `static`: source dir to serve (`"."` if same as `FRONTEND_DIR`). For `vite`: typically `"dist"`. For `spa`: the build output dir (varies — `dist`, `build`, `public`, `out`).
- `FRONTEND_BUILD_CMD` — vite/spa only; default `"npm run build"`. Override only if the project uses a different build script.
- `BACKEND_ROUTES` — space-separated Caddy path matchers that should hit the backend (defaults to `/api/*`). Inspect the backend code and list every prefix it owns — e.g. `"/api/* /auth/* /webhooks/*"`. Everything else falls through to the frontend bundle. Getting this wrong is a common silent failure mode: a backend route under a path that's not listed will return the frontend's `index.html` instead of the API response.

Pass these as `template_values={...}`. You never produce a setup script — the server renders the template with your values. Unknown keys are rejected; missing keys keep the template default.

### Step 5 — Collect environment variables (best-effort)

Gather the env vars the app needs and pass them as the `env_vars` dict.
This is a **best-effort, optional check** — not a hard gate. If the app
needs nothing, deploy without it. `PORT=3080` is added automatically
either way, so never include `PORT`.

**5a — Look for a local `.env` file first.**
If the deploy is from a local repo checkout, check the repo root for a
`.env` file. A gitignored `.env` is still readable on disk — `.gitignore`
only controls git tracking, not file reads, so try to read it regardless.
If a local `.env` exists and is readable:
- Parse it into `env_vars` — one `KEY=VALUE` per line; skip blank lines,
  `#` comments, and `PORT`; strip surrounding quotes from values.
- This is the **primary source**. Prefer it over guessing.

**5b — Cross-check against the code.**
Grep the repo for env var references:
- Node: `process.env.X`  |  Python: `os.getenv("X")` / `os.environ["X"]`  |  Go: `os.Getenv("X")`  |  Vite: `import.meta.env.VITE_X`
For any referenced var **not** already covered by the `.env`:
- Auto-generate secrets — `SECRET_KEY`, `JWT_SECRET`, `SESSION_SECRET` → random 32-char hex.
- Leave anything else (API keys, DB URLs) for 5c.

**5c — Ask the user only when needed.**
Ask the user to supply env vars when **both** hold:
- there is **no readable local `.env`** — no local checkout, no `.env`
  present, it can't be read, OR the user deployed by passing a bare git
  URL directly; **and**
- the app **does reference env vars** in code (from 5b) that you can't
  auto-generate.

When you ask, make it explicitly optional:
> "This app reads these env vars: `X`, `Y`. If it needs them to run, paste
> your `.env` contents or give me the values. Otherwise I'll deploy
> without them."

The user can decline — then deploy anyway. If the app references **no**
env vars at all, skip this step entirely. (This is the typical case for
the `static` template — vanilla HTML/CSS/JS doesn't read env vars at
runtime; the browser does whatever the page tells it to.)

Pass whatever you collected as the `env_vars` dict to `deploy_app`.

### Step 6 — Call `deploy_app`

```
deploy_app(
    app_name="<derived-name>",
    repo_url="git@github.com:<org>/<repo>.git",       # SSH format
    template_name="<the name you passed to get_template>",
    template_values={"ENTRY_FILE": "server.js"},      # whatever keys this template accepts
    env_vars={"DATABASE_URL": "...", "SECRET_KEY": "..."},
    smoke_paths=["/"],                                # see below
)
```

The repo URL **must** be SSH format. Containers have GitHub SSH keys; HTTPS won't work for private repos.

`template_name` is required — the exact name you chose in Step 3 (e.g. `"node"`, `"python-fastapi"`, `"go"`, `"fullstack"`). The dashboard uses it to show the right runtime, and the server uses it to pick the template file to render.

**`smoke_paths`** — paths the smoke test will probe on both the private and public URLs. Defaults to `["/"]`, which is right for single-half deploys (backend-only or frontend-only). For `fullstack`, pass one path per half so a dead backend can't hide behind a healthy frontend file_server response:
- `["/", "/api/"]` for the default `/api/*` routing
- `["/", "/api/", "/auth/login"]` if you added `/auth/*` to `BACKEND_ROUTES`

Pick a path that exists on each side. A 200-499 response counts as "responsive"; 0 / 5xx fails.

**`custom_subdomain`** — optional, off by default. If the user explicitly asks for a friendly URL ("deploy at weather.onpaper.co"), pass just the prefix (`"weather"`) — the `.onpaper.co` zone is hardcoded. The pipeline creates a CNAME via Cloudflare after the smoke test passes, and the app becomes reachable at `https://<subdomain>.onpaper.co` alongside the canonical `tls-*.vms.paperos.net` URL.

- Only pass it if the user actually asked for one. Don't auto-pick a name.
- Must be `[a-z0-9][a-z0-9-]{0,30}[a-z0-9]` (DNS-safe).
- If the subdomain is taken, the deploy still succeeds (canonical URL works) but the `setup-dns` step shows failed in the dashboard with a clear error.
- The user can also add a custom domain later from the dashboard's "Add a service" panel — so if you're unsure, deploy without and let them choose post-hoc.

You get back `{"run_id": "...", "status": "running"}`.

### Step 7 — Poll until done

Immediately call `get_deploy_status(run_id)` and keep polling every 5–10 seconds until status is `"done"` or `"failed"`.

**You own the polling loop.** Don't ask the user to wait or check. Report progress as steps complete:

- "Container created (10.11.4.x)..."
- "Deploying code..."
- "Waiting for TLS cert (can take ~5 min for new containers)..."
- "Live at https://tls-X-X-X-X.vms.paperos.net"

If status becomes `"failed"`, call `get_deploy_logs(run_id, step="<failed-step-name>")` and report the actual error from the logs, not a guess.

## Anti-patterns (do NOT do these)

- ❌ Trying to send a `setup_script` param — it doesn't exist on this tool anymore. Pass `template_name` + `template_values` instead.
- ❌ Putting `APP_NAME` inside `template_values`. It's auto-injected from the top-level `app_name` param; the server overwrites any APP_NAME you put in the dict.
- ❌ Inventing keys not returned by `get_template(...).keys`. The server rejects unknown keys with a hard error.
- ❌ Deploying without calling `get_template` first — the per-key comments and the `keys` list tell you what each template needs.
- ❌ Asking the user "should I check the deploy now?". You poll. Always.

## After a successful deploy

Report the live URL back to the user. It's at `https://tls-<ip-with-dashes>.vms.paperos.net`. The TLS cert is real and managed.
