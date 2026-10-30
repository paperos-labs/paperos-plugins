---
name: paperos-api-integration
description: Register a partner application with PaperOS or integrate PaperOS data into the user's own app, with or without the PaperOS SSO gate and regardless of hosting. Use for app registration, credential recovery, reports, batch uploads, records, documents, or database sync. Not for reading or uploading PaperOS data directly in this chat.
---

# Integrate the PaperOS Developer API into the user's app

This skill handles partner registration through the deploy MCP tools and changes
the **user's app code** to call PaperOS from its backend. Registration management
uses the developer's existing MCP login; the partner app authenticates its setup
and token requests with its client ID and secret. These are separate from
customers logging into the app.
Read the deploy server's guide for the current endpoint contracts and limitations.

## Step 1: Read the guide

Call `get_paperos_api_integration_guide(topic="overview")` first. It returns
`content` (markdown), `topics`, and `docs_url` (https://dev.paperos.com/). Then
read the topics the request needs:

| Need | Topic |
|---|---|
| App without the PaperOS gate, registration, credential recovery, setup requests | `get_paperos_api_integration_guide(topic="partner-integration")` |
| SSO gate token handling and Developer API limitations | `get_paperos_api_integration_guide(topic="auth")` |
| Report data | `get_paperos_api_integration_guide(topic="reports")` |
| Statement / CSV batch uploads | `get_paperos_api_integration_guide(topic="batch-uploads")` |
| Add or update records, documents | `get_paperos_api_integration_guide(topic="records")` |
| App has its own database | `get_paperos_api_integration_guide(topic="sync")` |
| Before reporting done | `get_paperos_api_integration_guide(topic="checklist")` |

Use only tools advertised by the connected server and topics listed by the guide.
Plugin 0.9.0 describes registration tools; an older server may not expose them yet.
If a required tool or topic is absent, report the server/plugin version mismatch.
Use documented endpoints on their documented service: PaperOS `/api/v1/*` data
and registration routes differ from deploy's setup-request REST API. Do not
substitute PaperOS internal `/api/*` or `/api/public/*` routes.

## Partner registration (no app SSO gate required)

For an app using its own authentication, whether hosted by paper-deploy or
elsewhere, read `partner-integration` and use this registration workflow:

1. Call `list_paperos_workspaces()` to identify the **owning PaperOS workspace**.
   Reuse the user's selected workspace when already known. This is registration
   ownership, not a list of customer workspaces or a GitHub organization.
2. Call `list_paperos_integrations(owner_org_id)` to find an existing registration.
   It returns metadata only. For a selected existing app, use
   `get_paperos_integration(owner_org_id, client_id)` to recover its credentials.
3. For a requested new registration, call
   `register_paperos_integration(owner_org_id, client_name, redirect_uris, contacts=None, client_uri=None)`.
   Registration requires full access to the owning workspace. Supply actual
   HTTPS redirect URLs; the API requires them even for this integration use.
   Resolve missing URLs with the developer rather than inventing a callback.
   Optional `client_uri` must also be HTTPS. Contacts are contact information,
   not proof of identity.
4. Keep the returned client ID, client secret, and `client_secret_expires_at`
   in the app's server-side secret configuration as authorized. Do not echo the
   secret in chat or commit it. A zero expiry means no configured secret expiry.
   After a timeout or unclear create result, list and get the registration to
   reconcile the outcome; do not automatically repeat registration.

New registrations start with `requestable_scopes: ["profile"]`. Registration
alone does not enable impersonation, link customers, or grant workspace data access.
Customer access still requires the app-to-user link and that user's workspace
membership; registration ownership does not select the customer's target workspace.

The setup request/status step is REST-only in this release:
`POST` and `GET /api/integrations/setup-request` on the deploy service use the
app's HTTP Basic client ID/secret, with no extra developer login. POST accepts
an empty body or `{}`. It creates/reuses an admin ticket, not the actual admin
setup. There is no setup-request MCP tool. Read `partner-integration` for the
deployment status and the outstanding upstream client-secret validation issue;
do not claim this step is safely exposed or the entire data integration works
from successful registration or ticket completion alone.

## SSO gate token flow

The SSO gate forwards the user's **OAuth access token**, not an ID token or an
already account-scoped PaperOS token. Pass it directly as Bearer to the selected
workspace's API URL. PaperOS checks membership and obtains the account-scoped
token internally; a reports request returns reports, not tokens. Do not add an
OAuth-to-ID-token conversion or a separate account-token call as a prerequisite.

Use public `org_...` IDs from List Orgs. Here org/workspace/account means a
PaperOS data context, not a GitHub organization. If explicit account-token
handling is requested, read `auth` for the public exchange response and current
scoping limitations. Never reuse a token scoped to A for a records URL naming B.

## Step 2: Inspect the codebase

Before writing code, establish:

- Framework and where server code lives (Express/Fastify/Next.js route
  handlers, FastAPI, Go net/http, a fullstack `apps/api` + `apps/web` split...).
  Which paths reach the backend (for fullstack deploys, `BACKEND_ROUTES`).
- Whether the app is deployed behind the paper-deploy SSO gate. The token
  comes from that gate only for that flow. For an app without it, follow
  `partner-integration`; do not enable SSO just to complete registration.
- Database and ORM/migration tool (Prisma, Drizzle, Knex, SQLAlchemy/Alembic,
  Django, sqlc, raw SQL), and how migrations run on deploy. A PaperOS Postgres
  from `create_postgres_database` is a normal Postgres.
- Existing HTTP client, config/env handling, logging, and error-reporting
  setup (for token redaction).

## Step 3: Implement (backend only)

- Keep PaperOS calls server-side with a base URL from an env var such as
  `PAPEROS_API_BASE_URL`, defaulting to `https://staging.paperos.dev`.
  For the gate flow, read the OAuth token **per request** from
  `X-Auth-Request-Access-Token`, send it as `Authorization: Bearer`, and map
  errors by the API's `code` field. Failed gate renewal can require sign-in.
  For the partner flow, follow its guide's token issuance and endpoint
  compatibility limits; registration tools do not issue customer workspace tokens.
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
- **Never ask for passwords or tokens in chat.** The gate supplies tokens for
  the SSO flow; registration/recovery tools supply partner credentials for
  server-side secret configuration. Keep both out of frontend config and Git.
- **App API keys are not PaperOS user tokens.** They let jobs through the app's
  gate without a browser session but supply no `X-Auth-Request-Access-Token`.
  Do not send them to the Developer API or treat them as a workspace grant.
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

## Step 4: Verify the requested flow

- For the SSO flow, the gate only forwards the token if enabled after token forwarding
  shipped. For a browser-authenticated request that lacks the header, re-apply it once
  with `set_app_sso_gate(app_name, enabled=True)` after telling the user that
  legacy gates without saved credentials receive a new client and cookie
  secret on that first update and may require a fresh sign-in. Matching saved
  configurations reuse their credentials. Do not toggle it off and on.
- For an app deployed here, set new env vars with `get_stored_app_env` then
  `replace_app_env` (full map), and ship code with `redeploy_app` when requested.
  Externally hosted apps use their existing configuration and release process.
- For registration-only work, verify the saved metadata with list/get and report
  registration separately from admin setup and customer data access. Stop there
  if that was the requested scope.
- Work through `get_paperos_api_integration_guide(topic="checklist")` and report
  the relevant results for app integration work. Exercise the app's own routes
  using its chosen authentication flow; do not claim success from code review alone.
