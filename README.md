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
| **paperos-deploy-mcp** | `paperos-deploy-mcp@paperos` | Deploy, redeploy, and manage apps, including GitHub App repository access, health checks, environment variables, SSO, and container lifecycle. |
| **paperos-transcript-extractor** | `paperos-transcript-extractor@paperos` | Run Entity Setup from your onboarding material — drop in a call transcript, notes, or documents and review extracted questionnaire answers in an embedded UI. |

Install from the directory UI (Customize → Plugins → Browse), or in the CLI:
```
claude plugin install paperos-entity-setup@paperos
claude plugin install paperos-deploy-mcp@paperos
```

On first use, click **Connect** on the plugin's PaperOS connector and complete
the one-time sign-in.

## Layout

```
.claude-plugin/marketplace.json   catalog of the plugins below
entity-setup/                     the paperos-entity-setup plugin (skills + connector pointer)
deploy/                           the paperos-deploy-mcp plugin (skills + command + connector pointer)
```

Each plugin's source of truth lives in its own private PaperOS repo; the
plugin-only slice is published here for distribution.

For deploy releases, synchronize `deploy/skills/` (including references),
`deploy/commands/`, and `deploy/.mcp.json` from `paperos-labs/deploy-mcp`.
Keep the plugin version and description aligned in both repositories' plugin
manifests and marketplace entries; preserve this repository's distribution URLs.
Updating the server or source repository alone does not update this marketplace copy.
