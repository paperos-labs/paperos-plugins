# paperos-core-mcp

PaperOS core workspace tools, served by the PaperOS application server at
`/mcp`. This plugin packages a remote connector and four workflow skills; it
does not run a second server or copy the PaperOS backend into Claude.

## Install

```sh
claude plugin marketplace add paperos-labs/paperos-plugins
claude plugin install paperos-core-mcp@paperos
```

Select the environment below, start Claude Code, then use `/mcp` to connect
PaperOS Core through the browser sign-in flow. Sign in to the account for that
environment. No API key, database password, or server `.env` file is needed.

After connecting, ask "List my PaperOS workspaces" for a read-only first check.
The tools can access only the workspaces available to the signed-in user.

## Environments

The default is **production**. Claude Code expands `PAPEROS_CORE_MCP_URL` in
the plugin's `.mcp.json`; set it before starting the session. This is a client
setting, not a change to the server's environment.

| Environment | MCP URL |
|---|---|
| Production | `https://app.paperos.com/mcp` |
| Staging | `https://staging.paperos.dev/mcp` |
| Demo | `https://demo.paperos.net/mcp` |

```sh
# Production (also the default when PAPEROS_CORE_MCP_URL is unset)
PAPEROS_CORE_MCP_URL=https://app.paperos.com/mcp claude

# Staging
PAPEROS_CORE_MCP_URL=https://staging.paperos.dev/mcp claude

# Demo
PAPEROS_CORE_MCP_URL=https://demo.paperos.net/mcp claude
```

For a provisioned development instance, use its actual HTTPS `/mcp` URL.
The plugin connects to one environment at a time; it never retries a request
against a different environment. Changing the variable requires a new Claude
Code session. Check the resolved connection and authenticate as needed using
`/mcp` before accessing data or submitting an upload. Do not carry workspace
IDs, document IDs, or credentials from one environment to another.

This URL substitution is a documented
[Claude Code feature](https://code.claude.com/docs/en/mcp#environment-variable-expansion-in-mcpjson).
It is not a Claude web/Chat environment selector. In those clients, configure
a remote connector with the literal URL for the intended environment; do not
paste the `${...}` expression into a connector URL. If a plugin-provided
connection points to a different environment, disable that connection for
the conversation. Keep the plugin's skills enabled. Embedded reports and
upload interfaces require a client with MCP Apps support; use the direct
tools in a terminal session without that support.

## Tools

The connector discovers tool schemas from the chosen server. This inventory
matches `server` revision `f5e25432c` (2026-09-29); deployments of different
environments may expose different versions. The connected server's tool
list is authoritative.

### Available to Claude

| Tool | Purpose |
|---|---|
| `list_workspaces` | List the user's accessible workspaces and roles. |
| `list_reports` | List reports in a named workspace. |
| `get_report_data` | Read one report, with optional row pagination. |
| `get_reports` | Read several reports across workspaces in one call. |
| `list_documents` | List document metadata in a workspace. |
| `get_document` | Read document details and request temporary download links, up to 100 documents per call. |
| `list_signature_status` | Find signing status and pending signers across accessible workspaces. |
| `find_investor` | Look up an investor by email across accessible workspaces. |
| `upload_batch` | Upload CSV text and submit a batch for processing. This changes workspace data. |
| `view_reports` | Open the embedded reports interface. |
| `batch_upload` | Open the embedded CSV upload interface. |

### Embedded-interface helpers

These four tools are registered on the same server but marked app-only and
excluded from the model's `tools/list`. The embedded interfaces call them;
the plugin does not turn them into additional agent commands.

| Tool | Purpose |
|---|---|
| `fetch_reports_for_account` | Load reports for the workspace selected in the interface. |
| `fetch_report_data` | Load a selected report into the interface. |
| `render_uploader` | Prepare the selected workspace and batch type for upload. |
| `process_batch` | Submit the file uploaded through the interface. |

The server also serves `ui://reports/view.html` and
`ui://capital-statement-uploader/upload.html`.
Their assets and authentication remain server-side.

## Skills

- [paperos-reports](skills/paperos-reports/SKILL.md): workspace discovery,
  report retrieval, pagination, masking, and interactive viewing.
- [paperos-documents](skills/paperos-documents/SKILL.md): document details,
  temporary download links, and cross-workspace signature status.
- [paperos-investors](skills/paperos-investors/SKILL.md): email lookup and
  careful interpretation of investment matches and totals.
- [paperos-batch-upload](skills/paperos-batch-upload/SKILL.md): direct and
  interactive CSV submission for the six supported batch types.

## Source and Scope

The tool inventory comes from `server/cmd/paperapi/main.go`, the handlers in
`server/internal/mcptools/`, and the visibility filter in
`server/internal/mcplib/server/server.go`.

The source repo's only `mcp-skills` entry is `paperos-fetch`, but its
`fetch_page` tool is disabled and not registered. It is intentionally not
installed as an active skill here. `get_upload_url`, `generate_chart`, and
`fetch_chart_config` also appear in older instructions or documentation but
are not registered tools in this revision. The plugin does not advertise
them. The two OAuth/MCP skills under `server/skills/` are backend development
and infrastructure guides, not end-user workflows, so they stay in `server`.

The four bundled skills describe the current registered tools. There are no
extra deploy, entity-formation, arbitrary web-fetch, or admin-access tools
in this plugin.

For maintainers, validate changes with:

```sh
claude plugin validate ./core
claude plugin validate ./.claude-plugin/marketplace.json
```
