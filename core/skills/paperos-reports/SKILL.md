---
name: paperos-reports
description: List PaperOS workspaces and reports, retrieve report data for analysis or CSV export, and open the interactive reports viewer. Use for requests about data stored in PaperOS staging.
---

# PaperOS Reports

Use the `paperos-core` staging connector. Discover the actual tool names and
schemas; clients may prefix the names below. Do not substitute a similarly
named tool from another connector.

## Direct Data

1. Use `list_workspaces` when the workspace is not already identified in
   staging. It returns `name`, `account_id`, and `role`. Reuse exact returned
   names; name matching is case-insensitive.
2. Use `list_reports(workspace)` to discover exact report names and record
   counts. Do not invent a report name or treat an inaccessible workspace as
   an empty one.
3. For one report, call `get_report_data(workspace, report)`. For several,
   call `get_reports(requests)` with an array of `{workspace, report}` pairs,
   including pairs across workspaces. Do not loop over `get_report_data` for
   multiple reports.
4. Work from the returned `headers` and `rows`. Preserve the association
   between each result, its workspace, and its report name. Batch retrieval
   can return both `results` and `errors`; report partial results as partial.

`get_report_data` accepts zero-based `offset` and `max_rows`. Omitted or zero
`max_rows` returns all rows. For a large report, request a bounded page and
use `record_count`, `returned`, and `offset` to decide whether more is needed.
Do not present one page as a complete report. `get_reports` does not have
those pagination inputs; use the single-report tool when paging one report.

Sensitive report columns are masked by default and named in `masked_columns`.
Set `reveal_sensitive: true` only when the user explicitly requests the full
values. For `get_reports`, that flag applies to every report in the call;
do not include unrelated reports in a request to reveal sensitive data.

## Interactive Reports

Use `view_reports()` when the user asks to browse reports interactively or
use the embedded charts interface, and the client supports MCP Apps. Call
it once; the interface handles workspace and report selection. Opening it
does not mean Claude has read the report data. Use direct tools for analysis,
comparison, filtering, or generating a CSV.

The helpers `fetch_reports_for_account` and `fetch_report_data` are app-only,
not agent commands. Do not call historical chart-generation tools that are
absent from the connected server's tool list.

## Connection and Results

Authentication belongs to the connector's sign-in flow. If the server reports
an expired or missing session, ask the user to reconnect that connector, not
to paste passwords or tokens into chat. Access is checked by the server.

Base conclusions on the retrieved data, distinguish missing data from zero,
and include any retrieval errors that limit the answer.
