# PaperOS plugins for Claude

The PaperOS plugin marketplace. Add it once and install any PaperOS plugin
directly in Claude — each is a thin client: skills + a pointer to a PaperOS
service. No PaperOS source code lives here.

> Requires a paid Claude plan (Pro / Max / Team / Enterprise — plugins are paid-only).

## Add the marketplace

**In Claude (web or the Desktop Chat tab):** Customize → Plugins → **+** →
**Add marketplace** → **Add from a repository** → paste this repo's URL.

**In Claude Code (CLI):**
```
claude plugin marketplace add paperos-labs/paperos-plugins
```

## Plugins

| Plugin | Install | What it does |
|---|---|---|
| **paperos-entity-setup** | `paperos-entity-setup@paperos` | Form & manage PaperOS legal entities (LLC, C-Corp, LP, funds) in chat — formation workflow, field predictions, review dashboards, and the PaperOS knowledge bases. |
| **paperos-deploy-mcp** | `paperos-deploy-mcp@paperos` | Deploy and manage apps and independent PostgreSQL databases. Register PaperOS partner integrations, list registrations, and recover credentials, regardless of app hosting or login. Guide your app's PaperOS API integration (reports, batch uploads, records, DB sync). |
| **paperos-transcript-extractor** | `paperos-transcript-extractor@paperos` | Run Entity Setup from your onboarding material — drop in a call transcript, notes, or documents and review extracted questionnaire answers in an embedded UI. |
| **paperos-core-mcp** | `paperos-core-mcp@paperos` | Query PaperOS staging workspaces, reports, documents, signature status, and investor records; submit batch CSV uploads. [Setup](core/README.md). |

Install from the directory UI (Customize → Plugins → Discover), or in the CLI:
```
claude plugin install paperos-entity-setup@paperos
claude plugin install paperos-deploy-mcp@paperos
claude plugin install paperos-transcript-extractor@paperos
claude plugin install paperos-core-mcp@paperos
```

On first use, click **Connect** on the plugin's PaperOS connector and complete
the one-time sign-in.

For app deployment, also open the [paper-deploy dashboard](https://deploy.onpaper.co/dashboard/#github-connection)
with that same PaperOS account and choose **Connect GitHub**. Each interactive
deploy/redeploy checks your approved work-email domain, repository organization,
linked GitHub account's current collaborator access, and the App's repository
selection. Database-only use and integration registration do not require a GitHub connection. See the
[Integration Kit](https://deploy.onpaper.co/integration-kit) for setup and access errors.

To make your app use PaperOS data, use the deploy plugin's
[`paperos-api-integration` skill](deploy/skills/paperos-api-integration/SKILL.md).
For apps behind the SSO gate, it supplies a user-level OAuth token to the backend;
the API checks access to the selected workspace and returns data. For apps using
their own login, including apps hosted elsewhere, read the `partner-integration`
guide topic. Registration tools use the developer's MCP login and owning PaperOS
workspace; the app's setup and token requests use its client ID and secret.
Registration starts with `profile` scope and does not itself enable customer
impersonation or grant workspace data access.

Plugin 0.9.0 documents `list_paperos_workspaces`, `list_paperos_integrations`,
`get_paperos_integration`, and `register_paperos_integration`. Use them only when
advertised by the connected server; updating this plugin does not deploy the
server. Registration requires actual HTTPS redirect URLs. List returns metadata;
register/get return credentials that must stay in server-side secret configuration.
Reconcile an uncertain registration with list/get instead of automatically creating again.

Admin setup request/status remains a separate REST step, not an MCP tool in this
release. The guide documents its deployment status and the outstanding upstream
client-secret validation issue. Registration or a completed setup ticket alone
does not prove the app's data integration works.
This differs from the core plugin, which uses tools to work with data in chat.

The core plugin connects only to `https://staging.paperos.dev/mcp`.
See its [setup guide](core/README.md) for sign-in and usage.

## Layout

```
.claude-plugin/marketplace.json   catalog of the plugins below
entity-setup/                     the paperos-entity-setup plugin (skills + connector pointer)
deploy/                           the paperos-deploy-mcp plugin (skills + command + connector pointer)
transcript-extractor/             the paperos-transcript-extractor plugin (skill + connector pointer)
core/                             the paperos-core-mcp plugin (skills + staging connector pointer)
```

The deploy plugin (`deploy/`) is maintained only here. Its skill, command,
connector config, and manifest no longer exist in `paperos-labs/deploy-mcp`, so
make every deploy plugin change in this repository. When the deploy server
changes tool names, parameters, or behavior, update `deploy/` in the same release.

The core plugin (`core/`) is also maintained here. Its tools and embedded
interfaces run in `paperos-labs/server`; update the plugin's skills and tool
inventory when those server contracts change. No server code or credentials
are bundled in the plugin.

The entity-setup and transcript-extractor plugins' source of truth lives in
their own private PaperOS repos; the plugin-only slice is published here for
distribution.
