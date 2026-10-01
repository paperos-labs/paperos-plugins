---
name: paperos-api-integration
description: Make the user's own app (usually one deployed with paper-deploy behind the PaperOS SSO gate) read or write PaperOS data through the PaperOS Developer API. Use when they want their app to pull report data, batch-upload statements or other CSVs, add or update investor/fund records, list documents, or keep its own database in sync with PaperOS. Not for reading or uploading PaperOS data directly in this chat.
---

# Integrate the PaperOS Developer API into the user's app

This skill changes the **user's app code** so its backend calls the PaperOS
Developer API (`/api/v1`). It does not call PaperOS itself and does not need the
user's PaperOS credentials. The authoritative guidance comes from the deploy
server's read-only guide tool; read it rather than relying on memory.

## Step 1: Read the guide

Call `get_paperos_api_integration_guide(topic="overview")` first. It returns
`content` (markdown), `topics`, and `docs_url` (https://dev.paperos.com/). Then
read the topics the request needs:

| Need | Topic |
|---|---|
| Token handling (always) | `get_paperos_api_integration_guide(topic="auth")` |
| Report data | `get_paperos_api_integration_guide(topic="reports")` |
| Statement / CSV batch uploads | `get_paperos_api_integration_guide(topic="batch-uploads")` |
| Add or update records, documents | `get_paperos_api_integration_guide(topic="records")` |
| App has its own database | `get_paperos_api_integration_guide(topic="sync")` |
| Before reporting done | `get_paperos_api_integration_guide(topic="checklist")` |

Use only endpoints the guide documents. Do not call `/api/public/*` or
non-`v1` `/api/*` routes, and do not invent endpoints or fields; when a detail
is missing, check `docs_url` or ask the user.

## Step 2: Inspect the codebase

Before writing code, establish:

- Framework and where server code lives (Express/Fastify/Next.js route
  handlers, FastAPI, Go net/http, a fullstack `apps/api` + `apps/web` split...).
  Which paths reach the backend (for fullstack deploys, `BACKEND_ROUTES`).
- Whether the app is deployed behind the paper-deploy SSO gate. The token
  comes from that gate; without it there is no user token.
- Database and ORM/migration tool (Prisma, Drizzle, Knex, SQLAlchemy/Alembic,
  Django, sqlc, raw SQL), and how migrations run on deploy. A PaperOS Postgres
  from `create_postgres_database` is a normal Postgres.
- Existing HTTP client, config/env handling, logging, and error-reporting
  setup (for token redaction).

## Step 3: Implement (backend only)

- One small server-side PaperOS client module: base URL from an env var such as
  `PAPEROS_API_BASE_URL` defaulting to `https://staging.paperos.dev`; token read
  **per request** from the `X-Auth-Request-Access-Token` header and sent as
  `Authorization: Bearer`; errors mapped by the API's `code` field.
- App routes the browser calls; they call PaperOS on the server and return only
  what the UI needs.
- If the app has a database: add sync tables through the app's own migration
  tool (records keyed by `(org_id, record_id)` on the public `rec_...` id, a
  sync-run log, a `batch_uploads` ledger with a unique CSV SHA-256), plus pull,
  push, re-pull, and reconcile flows as described in the `sync` topic.
- Follow existing code style and structure; add tests where the project has them.

## Safety rules

- **No tokens in the browser.** Never return, embed, or expose the PaperOS
  token to client-side JavaScript, URLs, or JS-readable cookies. Never log it or
  store it in the database; redact it in request logging and error reporting.
- **Never ask for passwords or tokens in chat**, and never tell the user to paste
  one into code or `.env`. The SSO gate supplies the token; if it is missing,
  explain the gate requirement instead.
- **Batch uploads are not idempotent.** Always dry-run (`dry_run=true`) first,
  get the user's confirmation, POST once, and never auto-retry a batch POST
  (no retry wrappers, queue redelivery, or loops). On a timeout or 5xx, record
  the outcome as unknown and check the report or `batch_url` first.
- **Staging for prototypes.** Use `https://staging.paperos.dev`; switch to
  production only when the user explicitly asks.
- **No SSNs/EINs.** Do not use `reveal_sensitive=true` or store tax ids unless
  the user has an explicit, approved need. Ask first.
- **PaperOS is the source of truth.** Soft-delete records missing from a full
  pull; never hard-delete or silently overwrite PaperOS from the app's copy.
- Running a real (non-dry-run) batch upload or creating/updating records while
  testing writes data to PaperOS staging: ask the user before doing it.

## Step 4: Gate, deploy, verify

- The gate only forwards the token if it was enabled after token forwarding
  shipped. For an existing gated app that lacks the header, re-apply it once
  with `set_app_sso_gate(app_name, enabled=True)` after telling the user that
  it registers a new OAuth client and users sign in again. Do not toggle it
  off and on.
- Set new env vars with `get_stored_app_env` then `replace_app_env` (full map),
  then ship code with `redeploy_app` per the `deploy-app` skill.
- Work through `get_paperos_api_integration_guide(topic="checklist")` and report
  each result. Exercise the app's own routes through the deployed URL in a
  signed-in session; do not claim success from code review alone.
