# PaperOS Entity Setup — Claude plugin

One install gets you both halves of PaperOS Entity Setup in Claude:

- **The PaperOS connector** — formation workflow tools (`get_profile`, `start_formation`,
  `submit_formation`, steering, predictions, dashboards) plus `kb_search` over the PaperOS
  knowledge bases (help center + fund LPA/PPM education).
- **The skills** — `entity-setup` (the workflow router), `onboarding` (the new-user
  formation conversation), and `kb-grounding` (how to ground PaperOS facts in the KBs).

Requires a paid Claude plan (Pro/Max/Team/Enterprise — plugins are paid-only).

## Install

1. In claude.ai (or the Claude Desktop **Chat** tab): **Customize → Plugins**.
2. Either **upload this plugin** (the ZIP of this directory) or — once published —
   **Add marketplace → Add from a repository** with this repo's URL, then **Install**.
3. First use: click **Connect** on the PaperOS connector and complete the sign-in
   (one-time OAuth via PaperOS).
4. Start a chat: *"I want to form a Delaware LLC"* — the skills route the rest.

## Migrating from the separate connector + uploaded skills

If you previously added the PaperOS custom connector and uploaded the three skills
(`paperos-entity-setup`, `paperos-entity-setup-onboarding`, `paperos-kb-grounding`)
individually:

1. Install this plugin (above).
2. **Delete the three uploaded skills** in Customize → Skills — otherwise both copies stay
   listed and can drift apart.
3. Optionally remove the manually-added PaperOS connector (the plugin bundles it).

## Updates

- **ZIP installs:** re-upload a new ZIP with the same plugin name — it overwrites in place.
- **Marketplace installs:** updates flow from the repo automatically.

## Layout

```
.claude-plugin/plugin.json   manifest
.mcp.json                    the PaperOS connector (remote MCP, OAuth)
skills/entity-setup/         workflow router (start here)
skills/onboarding/           new-user formation conversation
skills/kb-grounding/         KB grounding policy (help + fund KBs)
```
