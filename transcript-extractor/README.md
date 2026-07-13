# PaperOS Transcript Extractor — Claude plugin

Run the PaperOS Entity Setup workflow from your own onboarding material. Drop
in a call transcript, free-text notes, Q&A answers, or documents, and the
extractor pulls out questionnaire answers as **draft values you review** in an
embedded UI — nothing commits without you.

One install gets you both halves:

- **The PaperOS Transcript Extractor connector** — the interactive workflow
  (`transcript_extractor` opens the embedded UI) plus the extraction tools it
  drives behind the scenes.
- **The extraction skill** (`paperos-entity-setup-extraction`) — the contract
  Claude follows when extracting: evidence-quoted candidates, confidence
  scores, strict per-type formatting, and never guessing to fill a blank.

Requires a paid Claude plan (Pro/Max/Team/Enterprise — plugins are paid-only).

## Install

1. In claude.ai (or the Claude Desktop **Chat** tab): **Customize → Plugins →
   Add marketplace → Add from a repository** → this repo's URL, then install
   **paperos-transcript-extractor**.
2. First use: click **Connect** on the extractor connector and complete the
   one-time PaperOS sign-in.
3. Start a chat: *"Open the transcript extractor"* — pick your workspace and
   work through it in the embedded UI.

## Layout

```
.claude-plugin/plugin.json   manifest
.mcp.json                    the extractor connector (remote MCP, OAuth)
skills/paperos-entity-setup-extraction/   the extraction contract
```
