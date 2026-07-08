---
name: onboarding
description:
  Run the conversation that onboards a user into forming a legal entity with
  PaperOS Entity Setup — set up their workspace, fill the Entity Assessment from
  live questions, and save their profile. Use when a user wants to form an entity
  (LLC, C-Corp, LP, fund, holding company, etc.) and hasn't started yet — i.e.
  the first thing you do is call get_profile.
---

PaperOS is a legal entity-management platform. "Entity Setup" is the workflow
that forms a legal entity. This product is the standalone front door to it: a
user shows up wanting to form something, you set up their workspace and walk them
through the first questionnaire conversationally — no forms, no clicking.

## The flow — three tool calls

1. **`get_profile`** — call this FIRST. It returns `state`:
   - **`"returning"`** — they already have at least one formation. Give them a
     choice (offer it as options they can pick):
     - **Continue an existing one** → call `entity_setup` to open the dashboard
       (it lists every entity), and stop.
     - **Start a new one** → run the conversation below. There's nothing to reuse
       up front: no profile is collected before tx0, and the archetype's steering
       is asked *after* tx0 via `get_steering_questions` (in the prediction flow
       below), which already skips anything answered for this workspace. Just proceed.
   - **`"new"`** — no profile yet. Run the conversation below in full.
2. **`start_formation(company_name, company_purpose)`** — but first ask the user
   the two things needed to create the workspace: a **name** for the entity and
   its **purpose** ("For-profit Company", "Investment Vehicle", or "Other"). Open
   with something light, e.g. *"Let's set up your workspace first — what would you
   like to name the entity, and is it a for-profit company, an investment vehicle,
   or something else?"* Then call `start_formation`. It creates the workspace and
   returns `collect.context` (the LIVE Entity Assessment questions) plus a
   `run_id`. (No profile to gather — context only; steering comes after tx0.)
3. **`submit_formation(run_id, context)`** — once you've gathered every
   required answer, submit. See the contract at the bottom.

## Asking questions — ALWAYS use the option picker

**Any question that carries an `options` list MUST be asked via the
`AskUserQuestion` tool (clickable choices) — NEVER as a plain-text list the user
has to type back.** This applies to every options-bearing question in the flow:
the Entity Assessment selects (entity type, sub-type, purpose) and
every steering question that has options. Present the question + its options as a
single/multi-select picker, record the user's pick exactly, and move on.

**The only exception is a very large enumeration** — e.g. **Domestic State**
(~50 options), where a chip picker is unusable. Ask those as a normal typed
prompt instead. Use judgment: a bounded handful of options → always the picker;
a long list → typed.

Do not paste options as a numbered/bulleted text list and wait for the user to
type one — that's the failure mode this rule exists to prevent.

## What you collect (from start_formation's `collect`)

**`collect.context`** — the live Entity Assessment questions. Each is a real
PaperOS question object. Two things to honor on every one:

- **Selects:** if a question carries an `options` list, ask it via the
  **`AskUserQuestion` tool** (clickable choices) — see the rule below — and the
  answer MUST be one of those option values, exactly. Never invent a value
  outside the list.
- **Conditionals:** a question with a `required_question_id` is a follow-up — only
  answer it when the parent question (the one whose `id` equals
  `required_question_id`) has the answer equal to `required_question_value`.
  Otherwise skip it. (This is how the LLC/LP/etc. sub-type questions work — ask
  the LLC sub-type only when the entity type you settled on is the one its
  `required_question_value` names.)

The entity **name** question's answer is the same name you used for
`company_name`.

**No profile is collected before tx0.** `start_formation` returns only
`collect.context` (above). All operator/intent context — experience, goal,
founders, etc. — is gathered AFTER tx0 by `get_steering_questions` (the
per-archetype steering, isolated per route), once the archetype is known.

You can still *read* the user's experience from how they talk (to set your
depth) and *explore what they're building* to help them pick the right
`entity_type` in the assessment — just don't formally collect those here.

## Running the conversation

- **Read their experience from how they talk** (experienced → quick Q&A;
  first-timer → explain and steer) and let that set your depth — it's captured
  formally as steering *after* tx0, not asked here.
- **One topic at a time.** Never dump the whole question list. Ask, listen,
  follow up. A founder rarely thinks in "C-Corp vs LP" — get at it through what
  they're building (`goal`) and steer them to the right `entity_type`.
- **Pick select answers from the real options.** When a question has `options`,
  map what the user tells you to the exact option string.
- **Confirm before submitting.** Read back the entity name, type, and state in
  plain language and get a yes.

## Submitting — call `submit_formation`

Pass:

- `run_id` — from `start_formation`.
- `context` — a list of answers to the tx0 questions, each
  `{"question_id": <int>, "resource_variable_name": "<str>", "value": "<answer>"}`.
  Echo back the question's own `id` and `resource_variable_name`. Include a
  conditional sub-type ONLY when it applied. Select answers must be exact option
  values. You don't need to include the entity **name** question — it's filled
  automatically from the workspace name you set in `start_formation`.

It commits the answers and finalizes the Entity Assessment, then routes to the
next step — PaperOS work first, so a success means everything landed. (No upfront
profile: experience, goal, and the rest are gathered *after* tx0 by
`get_steering_questions`.)

**WAIT for the result and act on it:**

- `{"ok": true, "next_step": ...}` → tx0 is set up. Continue on `next_step`:
  - **`"steering"`** (forming fresh — pre-fill the formation):
    1. Call `get_steering_questions(run_id)`. Ask the user EVERY returned question
       conversationally (adapt tone to their `experience`), then call
       `submit_steering_answers(run_id, answers)` with `{question_key: answer, ...}`
       for ALL of them — it requires the complete set and rejects a partial one
       (re-ask the missing ones and resubmit the full set). If it returns no
       questions, skip straight to step 2.
    2. Call `get_prediction_context(run_id)` — returns the archetype's `predict`
       list, `given` (tx0 + steering), `multi_instance`, and the shared contract.
       Follow the contract: produce `value` + `confidence` (0–1) + `reasoning` +
       `sources` for EVERY field; for a `multi_instance` resource predict as many
       instances as `given` warrants, each tagged with an `instance` index;
       best-effort web-search the `effort: deep` fields.
       NEVER leave a predict field blank — always draft a concrete best-effort
       value (a sensible default or your best inference from `given`) and let
       `confidence` carry any doubt. A low confidence is fine; an empty `value`
       is invalid. A field nobody could possibly draft belongs in `user_fills`,
       not `predict`.
    3. Call `submit_predictions(run_id, predictions)` in the contract's schema.
       Everything is a DRAFT — nothing auto-commits.
    4. Tell the user their formation is pre-filled with drafts; point them to open
       `entity_setup` to review + save each field.
  - **`"dashboard"`** → tell the user their formation is set up and point
    them to open `entity_setup` to continue.
- `{"ok": false, "failed_step": "validate", "retryable": true}` → the `reason`
  lists missing required answers. Gather those from the user and call
  `submit_formation` again. This is the only safe-to-retry failure.
- `{"ok": false, "retryable": false}` (e.g. `failed_step: "commit"` or
  `"paperos"`) → something failed mid-commit. Do NOT just resubmit — relay the
  `reason` to the user and stop, since part of the setup may already have landed.
