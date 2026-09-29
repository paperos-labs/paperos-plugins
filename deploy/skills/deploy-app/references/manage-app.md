# Manage an Existing App

Read this for code updates, post-deploy SSO, env, DNS, or lifecycle requests. Do not call
`create_app_deployment` for routine configuration changes. It creates a new container.

## Identify the Target

SSO and env tools take `app_name` and operate for the authenticated user; live
pushes target that user's latest active deployment for the app. Lifecycle, status,
and logs take `run_id`. Reuse IDs from tool results or the user's identified
dashboard deployment. Resolve ambiguous targets before mutation; do not guess
run IDs or use another user's deployment.

The dashboard is at the connected server's `/dashboard/`, currently
`https://deploy.onpaper.co/dashboard/`. It lists deployments and exposes actions
not available as MCP tools. Do not invent tools or harvest browser tokens to
access its authenticated API.

## Redeploy Code

```python
redeploy_app(run_id="1778174516-weather")
```

To change the deployment's branch when requested:

```python
redeploy_app(run_id="1778174516-weather", branch="release/v2")
```

For deployments created with a v5+ agent and a saved deployment recipe. Existing
containers are not upgraded. The container must already be running; redeploy
does not boot it or create a replacement. Use only tools the connected server advertises.

The server creates a **new run ID**, linked to the same container, then runs
`check-agent`, `pull-code`, `build-app`, `restart-app`, and `smoke-test`. Poll the
returned ID every 5-10 seconds and report step changes without asking the user
to check. Read failed-step logs. On success return the exact `live_url` and any
`custom_domain`; a URL in the initial acknowledgment is not proof of completion.

Without `branch`, the server keeps the saved branch; older runs without branch
metadata keep the checkout's tracked branch. An explicit branch requires a v7+
agent advertising `branch-selection`; unsupported agents fail before pulling.
Git fetches the exact remote branch, switches without discarding local changes,
and fast-forwards only. Missing branches and tag-only names fail without fallback.
The actual checkout branch is saved, including after a successful switch followed
by a failed build; a refused switch keeps the previous branch. The dashboard's
Redeploy button retains this saved branch. The authenticated REST equivalent is
`POST /api/deployments/{run_id}/redeploy` with `{"branch":"release/v2"}`; an empty
body or `{}` keeps the saved branch. Install/build uses the original
server-rendered recipe and the current on-container `.env`, including build-time
frontend values. Only the app service restarts, not the container. Env, SSO gate,
DNS and app data are preserved. PaperOS keeps its existing GitHub credential;
partner deployments obtain a fresh read-only GitHub App token for each pull and
remove/revoke it afterward. Never supply credentials in the redeploy request.

This is an in-place update: downtime is possible and there is no automatic
rollback. A failed build or health check can leave the checkout updated; do not
claim the previous version is still serving. Report the actual failure before
retrying. Do not erase local changes, silently swap templates, disable SSO, or
create another container as automatic recovery. Legacy deployments require an
explicitly authorized replacement, not an automatic image or agent upgrade.

## SSO Gate

```python
set_app_sso_gate(app_name="weather", enabled=True)
```

Enabling registers an OAuth client and pushes a Caddy/oauth2-proxy gate to the
latest live container. Success returns `enabled=True`, `client_id`, and `url`.
Use that returned URL. Backend, frontend, and fullstack apps are supported when
the necessary serving config exists. An older frontend/fullstack deployment
without a captured serving block needs a replacement deployment; explain that before
creating a replacement container.

The wrapper authenticates visitors with PaperOS without modifying app code.
It gates static files and APIs as well as pages; it does not configure a per-app
membership allowlist or webhook bypass. A successful toggle reports applied
configuration, not a completed user login.

```python
set_app_sso_gate(app_name="weather", enabled=False)
```

Disabling removes the gate and makes the app public. Only do this when that
access change is intended by the user, never just to pass a test. It returns
`enabled=False`. With no live container it can merely clear stored state; do not
claim to have tested an app that is down.

There is no MCP `get_app_sso` tool. `get_deployment_run_status` shows the original run's
steps, not the current gate after later toggles. Do not repeatedly enable SSO to
poll: each enable registers a new OAuth client. The dashboard reads stored state
and polls a live `gate-status` probe (`gated`, `open`, `down`) through transient
restarts. That probe still does not exercise the full login flow.

Stored SSO state is not automatically applied to a fresh deployment; use
`sso=True` for new protected deployments. OAuth clients are not currently
deregistered on disable/destroy. Do not promise complete client cleanup or
blindly retry registration after an uncertain response.

## Custom DNS

During deployment use `custom_subdomain`; afterward use the dashboard's custom
domain action. There is no standalone DNS MCP tool. The existing authenticated
REST route, for clients already authorized to use it, is
`POST /api/deployments/{run_id}/custom-domain` with `{"subdomain":"weather"}`.
It returns `custom_domain` and `url`. Knowing the route is not authorization to
acquire another client's credentials or alter a different deployment.

Only `<prefix>.onpaper.co` is supported. Prefixes are 2-32 lowercase
alphanumeric/hyphen characters, starting and ending with an alphanumeric.
An existing record is a conflict, not permission to replace it. Do not advertise
arbitrary bring-your-own-domain support or automatically choose another name.
The deploy's `live_url` remains canonical; `custom_domain` is the alias.

SSO registers its callback against the custom domain if present. Adding DNS to
an already gated app does **not** automatically update that callback. After
attaching the alias, reapply `set_app_sso_gate(app_name, enabled=True)` when authorized
to update that gate, and verify login at the returned URL. Do not disable SSO
first. Reapplying registers a new client; it is not a read-only check. Do not
assume login/cookies are interchangeable across both hostnames.

## Persistent Environment Variables

```python
get_stored_app_env(app_name="weather")
```

Returns `env_vars` and `updated_at`; values may be secrets. Read only what the
requested change needs and do not echo values. Users can enter private values
in the dashboard env editor to keep them out of the agent conversation.

`replace_app_env` **replaces** the entire stored map, not a merge. To change one key,
read the current map, preserve unrelated entries, apply the requested change,
and send the full map. Omit a key only when deletion is intended. An empty map
clears stored vars. Preserve app signing/session secrets across deployments.

```python
current = get_stored_app_env(app_name="weather")
updated = dict(current["env_vars"])
updated["LOG_LEVEL"] = "info"
replace_app_env(app_name="weather", env_vars=updated)
```

Do not include `PORT`: the tool strips it and templates own the runtime port.
Stored vars win over same-named `create_app_deployment.env_vars` on subsequent deploys.
Deploy-time vars are not saved to this store; replacing live env can drop such
transient keys. Reconcile needed transient values before replacing the map
rather than assuming `get_stored_app_env` contains the entire live env.

Distinguish storage from application in the result:
- `live_pushed=True`: the agent received the env update and restarted the service.
- `live_pushed=False`: vars were stored, but the live app was not updated.
- `push_error`: report the live-push failure; do not claim the app uses the new
  settings just because persistence succeeded.

Frontend build-time values require `redeploy_app` on a supported deployment. `replace_app_env` only
rewrites env and restarts; it does not rerun frontend builds. Never put secrets
into browser-exposed env variables.

## Container Lifecycle

Use these only for the user's intended action on the identified deployment:
all three act on the entire container, not just its app service. There is no
standalone MCP tool to stop or restart only the application process.

```python
shutdown_container(run_id="1778174516-weather")
```

Requests graceful shutdown, preserving disk. The returned `stopped` state records
the action; it is not a separate live power probe.

```python
reboot_container(run_id="1778174516-weather")
```

Requests a reboot; code, container, and URL remain the same. `restarted` is an
acknowledgment, not proof the app is healthy again. It does not pull code, rebuild,
or implement a start operation for a stopped container. There is no exposed
`start_container` tool; report that limitation instead of improvising.

```python
destroy_container(run_id="1778174516-weather")
```

Permanently deletes the container and on-container data; history is retained.
Ensure the user intended deletion of that exact deployment. Never use destroy
as automatic repair or cleanup after a failure. DNS cleanup is best-effort and
can fail even when destroy succeeds. Stored env is retained; OAuth client cleanup
is not implemented.

Lifecycle calls need a recorded `vmid` and server-side Proxmox credentials.
Surface missing-credential/legacy-run errors to the operator; do not ask the user
to put infrastructure credentials into app env vars.
