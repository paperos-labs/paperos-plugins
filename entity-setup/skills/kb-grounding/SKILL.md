---
name: kb-grounding
description:
  The PaperOS Knowledge Bases are a primary source for any fact you need to assist the
  user — how PaperOS works, entity/formation rules, fees, deadlines, document templates,
  AND fund LPA/PPM legal terms (management fees, carry, waterfalls, governance, LP
  rights) with market benchmarks. Consult them via kb_search whenever you'd otherwise
  answer from memory or guess. Carries both KBs' article templates, taxonomies, and how
  to search.
---

**Two PaperOS Knowledge Bases** sit behind the one `kb_search` tool, selected by its
`kb` argument. Treat them as ground truth: prefer them over your own training, which is
stale on anything jurisdiction-, market-, or time-specific.

1. **`kb="help"`** (default) — the PaperOS help center (`learn.paperos.com`): how the
   platform works, entity formation, workflows, templates, filings.
2. **`kb="fund"`** — Fund Builder education: plain-English articles on the legal terms
   in a fund's LPA (Limited Partnership Agreement) and PPM (Private Placement
   Memorandum), each with negotiation points and an industry-standard benchmark.
3. **`kb="all"`** — search both at once (hits are tagged with their `kb`).

You already know how to work over a body of articles — search for what you need, read
the hit, ground your answer, cite the source. You do **not** browse or load a whole KB;
you pull only what the task needs. This skill just hands you the map.

## What an article looks like

Both KBs: `title` · `subtitle` · `body` (markdown) · `category` · `subcategory` ·
`url` · `last_modified`. Fund articles have no subtitle/subcategory, and their body
always has four sections: **Legal Term** · **Legal Significance** · **Potential
Variations / Negotiation Points** · **Industry Standard Benchmark**.

## The taxonomies (use a `category` to scope a search)

The set may grow; `kb_search` returns each hit's real `category`, so trust those over
this list.

- **help**: FAQ · Templates · Workflows · User Guide · Founder Guide · Partner
  Services · Fund Guide · Services
- **fund**: Fund Economics · Fund Structure & Term · Governance · Investment
  Parameters · Distributions · LP Rights & Protections

## Which KB when

- Platform questions, entity formation, filings, "how do I … in PaperOS" → **help**.
- Fund terms and economics — management fees, carried interest, waterfalls, hurdle
  rates, GP/LP rights, LPAC, key person, side letters, capital calls, PPM contents —
  → **fund**. This is THE source when assisting a user building an investment fund.
- Unsure, or the question straddles both (e.g. forming a fund entity) → **all**.

## When to ground (the policy)

- **Authoritative / volatile facts — ground in the KB, don't trust memory:** filing
  fees, annual-report and tax deadlines, registered-agent rules, anything state- or
  jurisdiction-specific, market-standard fund terms (fee %, carry %, hurdle rates),
  anything that changes over time. Search the KB; cite the article.
- **Settled conventions you're sure of** (e.g. 1 vote per share for common stock): you
  may draft directly and only `kb_search` to confirm if unsure.

When you ground a fact in an article, cite its `url` and reflect real confidence; when
the KB returns nothing useful, fall back to your judgment and lower that confidence.
(Fund article urls are Box file links — cite them as-is.) Fund benchmarks are
educational market conventions, not legal advice — say so when it matters.

## How to search — `kb_search`

One tool, three modes (pass at least one of `query` / `keywords`; both = hybrid):

- `kb_search(keywords="Delaware annual report franchise tax")` — exact terms (BM25).
- `kb_search(query="when is a Delaware C-corp annual report due?")` — natural-language.
- `kb_search(query="what carry do VC funds charge?", kb="fund")` — fund KB.
- `kb_search(query="...", keywords="...", category="Fund Economics", kb="fund")` —
  blended, scoped to a section.
- `kb_search(query="...", kb="all")` — both KBs, merged.

Returns up to 5 hits (`title`, `url`, `category`, `body`, `kb`, …). Read the `body` of
the most relevant hit; open the `url` for the full article (the body is capped).

## Gotchas (PaperOS-specific — only we know these)

<!-- TODO(team): fill in the tribal knowledge the model can't infer, e.g.:
  - which category a given fact actually lives under (e.g. "<X> is filed under Templates/Financing, not FAQ/Entity")
  - article titles that are misleading (e.g. "the 'Fees' article is about platform fees, not state filing fees")
  - synonyms the KB uses vs. what a user would say
  - sections that are intentionally out of scope -->
