---
name: paperos-documents
description: Find PaperOS workspace documents, retrieve document details and temporary download links, and check signature status or pending signers across workspaces. Use for document access and signing-status questions, not for sending reminders or signing documents.
---

# PaperOS Documents and Signatures

Use the `paperos-core` tools from the intended environment. Clients may prefix
tool names. Discover their current schemas, and use workspace names and IDs
from that same connection. Use `list_workspaces` when workspace selection is
needed. Never switch environments to work around an access failure.

## Find and Retrieve Documents

- `list_documents(workspace, offset?, max_rows?)` returns metadata, not download
  links. Match the requested documents using the returned filenames and
  `pub_id` or numeric `id`. Use zero-based pagination for large workspaces;
  `count` is the total and `returned` is this page. Omitted or zero `max_rows`
  returns all documents.
- `get_document(workspace, document_id)` returns one document's details.
  Use `document_ids` for several IDs in one workspace, as strings, with at
  most 100 per call. Split larger selections and warn that minting many
  download links can take a moment.
- Check `not_found` in multi-document results. A document can also be returned
  without a download link if storage could not provide one; do not invent
  links or claim every requested file was retrieved.
- Returned `download_url` values last approximately one hour. Fetch them when
  needed, share them only for the user's requested documents, and retrieve
  a fresh link if an old one expires. Metadata or a link alone is not evidence
  that you have read the file contents.

## Signature Status

Use `list_signature_status` for who has signed, who is outstanding, and
follow-up lists. It searches accessible workspaces in one call; do not loop
through every workspace's documents just to answer a signing-status question.

Supported filters:

- `workspaces`: exact workspace names; omit for all accessible workspaces.
- `states`: defaults to `["outstanding"]`. Other choices are `completed`,
  `draft`, `declined`, `cancelled`, `unknown`, or `all`.
- `filename_contains`: case-insensitive filename text, such as an agreement
  type or investor name.
- `offset`, `max_rows`: zero-based paging; default 100, maximum 500 per page.

Use `state`, not raw `status`, to describe completion: a fully signed document
can have a later raw `document.deleted` event. An outstanding-only search
cannot establish that a missing document was signed; widen the state filter
when the user asks whether a specific agreement is complete.

Distinguish `awaiting_company_signature` from `awaiting_other_signatures`.
Use recipients' `is_company_signatory` field, not guesses about template
role labels. A non-company signer is not necessarily the investor.
If `email_sent` is false, PaperOS has no record of emailing that recipient
the signing link; mention this before suggesting a reminder. Do not claim
the link was never shared by any other means.

Carry returned `notes` into the answer when they affect certainty, and page
until the requested scope is covered. `document_id` from these results can
be passed to `get_document` for details or a download link. These tools do not
send reminders, cancel agreements, or sign on anyone's behalf.

If authentication fails, use the connector's reconnection flow. Do not ask
for passwords or tokens, bypass workspace access, or present failed lookups
as proof that a document does not exist.
