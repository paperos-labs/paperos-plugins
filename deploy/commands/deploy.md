---
description: Deploy a repository to PaperOS infrastructure
argument-hint: <repo-url-or-path>
---

Deploy the repository indicated by the user to PaperOS.

Use the `paperos-deploy-mcp:deploy-app` skill from this plugin. For standalone
database requests, follow its database reference instead of the app workflow below.
For a new app deployment:

1. Call `list_app_templates` to discover available stacks
2. Inspect the repo and select a matching template from that list, including frontend or fullstack templates when appropriate
3. Call `get_app_template(name=<chosen_name>)` and read its `content` and accepted `keys`
4. Build `template_values` using ONLY those keys; do not include `APP_NAME`, which comes from the top-level `app_name`
5. Resolve env vars without exposing secrets or overriding stored app settings blindly; templates own `PORT`. Call `create_app_deployment` with `app_name`, `repo_url`, `template_name`, and `template_values`. Include `env_vars` when needed and `smoke_paths` for both halves of a fullstack app
6. For requested DNS, pass only the `custom_subdomain` prefix on `onpaper.co`. For requested PaperOS protection, pass `sso=True`; the wrapper supports backend, frontend, and fullstack. DNS runs before SSO. Explain that the soft, post-deploy SSO gate is not fail-closed if the user requires never-public deployment
7. Poll `get_deployment_run_status(run_id)` every 5-10 seconds through every requested step. Top-level `done` is not terminal while any step is `pending` or `running`, including `setup-dns` and `enable-sso`. A top-level `failed`, `stopped`, or `destroyed` run ends polling even if later steps remain pending. Report step changes without asking the user to check or confirm
8. Return the exact `live_url` and any successful `custom_domain`. Inspect every failed step with `get_deployment_run_logs`, including optional failures on a `done` run. Report DNS failure separately; SSO failure means protection is not confirmed and the app may be public, not a successful protected deployment

Target repo: $ARGUMENTS

If `$ARGUMENTS` is a local path, inspect that repo, then deploy using its git remote
URL. Local uncommitted/unpushed changes are not deployed. Use plain GitHub SSH or
HTTPS URLs without credentials. PaperOS repos retain existing access; other owners
need the paper-deploy GitHub App installed with the repository selected. The server
supplies a fresh token for HTTPS clone and pull. For a URL, inspect only
what is needed to choose the template and env. Do not embed tokens in the URL.

For an in-place code update, use `redeploy_app(run_id)` on a supported new
deployment and poll the returned new run ID through health checks. Preserve the
existing container, env, DNS, and SSO. Old containers are not upgraded; do not
silently replace them. Redeploy may have downtime and has no automatic rollback.

For existing-app configuration or lifecycle requests, use the skill's management
reference and the matching tools/dashboard action, not a duplicate deployment.
`shutdown_container`, `reboot_container`, and `destroy_container` operate on the
whole container, not only the app process. Do not use them for an app-only restart.

For standalone PostgreSQL requests, use the skill's database reference and
create_postgres_database, list_postgres_databases, or delete_postgres_database.
No app/template is required. Only the database name is supplied; credentials are
generated. Connection strings are secrets. Database deletion requires explicit
intent and is independent of app/container deletion.

Critical rule (see the `deploy-app` skill for the full list):
- **NEVER** write, modify, or send a setup script. The server renders the selected template from `template_values`; you supply only the accepted values.
