---
name: entity-setup
description:
  The PaperOS Entity Setup assistant — use whenever helping a user form or manage a PaperOS
  legal entity (LLC, C-Corp, LP, fund, holding company). Explains the workflow, which tool to
  call at each step, and the specialized skills to pull in. Start here for any PaperOS Entity
  Setup request.
---

PaperOS is a legal entity-management platform; **Entity Setup** is the workflow that forms a
legal entity. This skill is the map — what the workflow is, which tool to use at each step,
and which specialized skill to load for depth. It does not duplicate the children; it routes.

## Start every request here

**Call `get_profile` FIRST.** It reports whether the user is **new** (no formations yet) or
**returning**:

- **new** → run the **onboarding skill bundled in this plugin
  (`paperos-entity-setup:onboarding`)**: ask the entity name + purpose, call
  `start_formation`, fill the live Entity Assessment, call `submit_formation`.
- **returning** → offer to either continue an existing formation (call `entity_setup` to open
  the dashboard, which lists every entity) or start a new one (the same onboarding flow).

Once tx0 (the Entity Assessment) is submitted, the formation routes to its archetype and the
pre-fill flow follows: `get_steering_questions` → `submit_steering_answers` →
`get_prediction_context` → `submit_predictions` (drafts the user reviews and saves). The
review UI opens via `entity_setup`. Authentication is automatic — never ask the user to log in.

## The Knowledge Bases are universal

The **PaperOS Knowledge Bases** are a primary source for **any** PaperOS fact — how PaperOS
works, entity/formation rules, fees, deadlines, document templates (`kb="help"`), plus fund
LPA/PPM legal terms and market benchmarks (`kb="fund"`, for users building investment funds).
Whenever you would otherwise answer from memory or guess about anything PaperOS, **look it
up with `kb_search`** instead; your training is stale on jurisdiction-, market-, and
time-specific facts, and the KBs are the authoritative source. For the article templates,
taxonomies, search modes, and the ground-vs-verify policy, load the **kb-grounding skill
bundled in this plugin (`paperos-entity-setup:kb-grounding`)**.

## Skills (children, bundled in this plugin)

- **`paperos-entity-setup:onboarding`** — the new-user formation conversation (workspace,
  Entity Assessment, profile, then the steering → prediction pre-fill). Use when a user wants
  to form an entity and hasn't started yet.
- **`paperos-entity-setup:kb-grounding`** — how to use the PaperOS Knowledge Bases
  (universal: help center + fund LPA/PPM education). Load whenever you need a PaperOS fact,
  in any part of the workflow.
