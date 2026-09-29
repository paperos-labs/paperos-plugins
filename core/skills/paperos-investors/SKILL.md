---
name: paperos-investors
description: Look up an investor by email across accessible PaperOS workspaces and explain their investments, match evidence, and recorded totals. Use for investor lookup and investment-history questions.
---

# PaperOS Investor Lookup

Use `find_investor` on the `paperos-core` staging connection. Discover the
current tool schema; its client-visible name may have a prefix. Do not switch
accounts to get more results without the user's direction.

1. Obtain the investor's email from the user or the supplied task context.
   Do not guess it from their name.
2. Call `find_investor(email, include_name_matches?)`. Email matching is
   case-insensitive. It searches all workspaces accessible to the signed-in
   user, so there is no need to pull every workspace's investor report.
3. Read `investors`, each investment's `workspace`, `matched_by`, and the
   returned `notes` before summarizing.

`include_name_matches` defaults to true. This can find investments sharing
a name with an email-matched record but using another email or no email.
`matched_by: "name"` is weaker evidence than `"email"`; identify it as such.
Set `include_name_matches: false` when the request is explicitly limited to
that email's records, and explain that other-address records were excluded.

If the result contains several investors, present them separately. The
server deliberately did not merge them; do not combine their records or
totals yourself. If `amounts_missing` is nonzero, `total_invested` is partial,
not a complete lifetime investment total. An empty result means no match in
the workspaces this user can access, not that the person has no investments.

For a name-only search, use `list_workspaces`, `list_reports`, and the report
retrieval tools to find the appropriate investor report. Discover the report
name rather than assuming it exists everywhere. For signing status, use
`list_signature_status`; an investment record alone does not prove signing.

The tool is read-only. It does not merge investors, change investments, or
contact them. If authentication fails, reconnect through the connector;
never request passwords or tokens in chat.
