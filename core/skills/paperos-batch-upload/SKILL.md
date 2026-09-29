---
name: paperos-batch-upload
description: Submit a CSV to PaperOS for capital statements, distribution notices, portfolio investments, capital contributions, bank transactions, or capital calls. Use for direct file submission or the embedded batch-upload interface.
---

# PaperOS Batch Uploads

Use the `paperos-core` staging connector. Discover the
current tool schema; clients may prefix tool names. Uploads write data and
submit work for processing, so establish the target workspace,
batch type, and intended file before submission. If the user only asked to
review a CSV, do not upload it.

## Direct Submission

Use `upload_batch` when the user has supplied a CSV and wants it submitted.

1. Identify the workspace, using `list_workspaces` if needed. Use an exact
   returned name; matching is case-insensitive.
2. Read the actual file or attachment with the client's available file tools.
   Do not substitute its path, a summary, base64, or invented rows for its
   raw CSV text. Preserve the supplied headers and values unless the user
   has authorized a transformation.
3. Select the exact `batch_type` below. If the request is ambiguous, clarify
   it before making the write.
4. Call `upload_batch` with `workspace`, `batch_type`, `file_contents` (raw
   CSV text), and optionally `file_name` (default `batch_upload.csv`).
5. Report what was submitted and return the server's `batch_url`. A successful
   call means the CSV was uploaded and submitted for processing, not that
   every row or downstream operation has completed. Do not claim notices
   were delivered or transactions finalized without separate evidence.

| Batch | `batch_type` |
|---|---|
| Capital Statements | `capital_statements` |
| Distribution Notices | `distribution_notices` |
| Portfolio Company Investments | `portfolio_investments` |
| Capital Contributions | `capital_contributions` |
| Bank Transactions | `bank_transactions` |
| Send Out Capital Calls | `send_capital_calls` |

## Interactive Submission

When the user wants to select and upload the file themselves in a client
that supports MCP Apps, call `batch_upload()` once. The interface handles
workspace, batch type, and file selection. Opening it is not proof that the
user completed the upload. In a terminal client without embedded interfaces,
use the direct workflow with the user's file and authorization instead.

`render_uploader` and `process_batch` are app-only helpers, not agent commands.
Do not request their storage tokens or recreate the interface's upload flow.

## Errors

Report submission errors without claiming success. Do not automatically
resubmit after a timeout or uncertain response: a write may already have
happened, and repeating it can duplicate work. Establish the result in the
PaperOS batch page before retrying. There is no registered batch-status
polling tool in this plugin; do not invent one.

Authentication is handled by the connector. For an expired or missing
session, reconnect through its sign-in flow; never ask for passwords or
tokens in chat. Missing workspace access or batch setup needs the relevant
workspace administrator, not a different user's credentials.
