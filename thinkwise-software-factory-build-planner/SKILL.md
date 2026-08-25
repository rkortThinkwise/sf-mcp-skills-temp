---
name: thinkwise-software-factory-build-planner
description: Interview the user about a goal or problem in a Thinkwise Software Factory application, then produce a reasoned build plan across data model, screens, tasks, process flows, control procedures, subroutines, and messages. Use when the user brings a raw goal or feature idea without an already-formed technical plan — not for stress-testing an existing plan, or a small, self-evident ask. Hands off each artifact to the skill that implements it; this skill only plans.
---

# Software Factory Build Planner

Interview the user about what they're actually trying to achieve, then turn that into a concrete,
reasoned build plan expressed in Thinkwise Software Factory building blocks — data model, screens,
tasks, process flows, control procedures, scheduler/views/prefilters, menu placement. This skill
plans; it does not build. Every artifact in the final plan points at the sibling skill that
implements it.

## How to run the interview

1. **Anchor the goal first, before any technical branching.** Ask one question: restate the
   problem as *outcome + who benefits + trigger event + success condition*, give your own
   recommended restatement, and get it confirmed. Nothing below should start until this is settled
   — a wrong anchor makes every downstream decision wrong too.
2. **Explore the live model before asking anything else.** Use the connected MCP tools
   (`search_capabilities` → `search_domain_capabilities` → `get_entity_definition`/
   `get_task_definition`, or `get_available_domains`/`get_domain_definition` if routing is
   ambiguous) to find existing tables, screens, tasks, and processes relevant to the goal. Never
   ask the user something the model can already answer — e.g. don't ask "do you have a customer
   table" when `get_domain_definition` can tell you.
3. **Ask one question at a time** — never bundle multiple decisions together.
4. **Walk the decision tree top-down, resolving dependencies before dependents.** Use the tree
   below as the default shape, but skip branches the anchor step already ruled out.
5. **Provide your own recommended answer** for every question, and the principle behind it — think
   it through and state what you'd do and why, then let the user confirm, correct, or refine. The
   *why* matters as much as the *what*: this is what separates a build plan from a checklist.
6. **Keep going** until every significant decision is resolved and there are no unresolved
   branches left.
7. **Every "default to X" line in the decision tree below is a recommendation, not a silent pick.**
   Two spots phrase it that way — subroutine reuse placement (step 4) and message severity/location
   (step 6) — but wherever this skill says "default to X," treat X as the recommended answer only:
   surface it and get it confirmed the same way as every other branch above, never apply it without
   asking. This is also what keeps this skill aligned with
   `thinkwise_software_factory_mcp_base`'s "Shared conventions" (confirm-before-mutate, ask-don't-
   default) — this skill's own recommend-and-confirm pattern already satisfies both, so there's
   nothing further to add here.

## Default decision tree

Resolve in this order — later branches often depend on earlier ones:

1. **Data** — does this need new tables/columns/domains, or does existing structure already cover
   it? Check the live model first (step 2 above) before asking. Defer exact naming/typing/reference
   conventions to `thinkwise_datamodeling_guidelines` — don't relitigate them here, just decide
   *what* is needed.
2. **Process shape** — is this a single CRUD interaction, a multi-step orchestration with decision
   points (process flow), a scheduled/recurring job (scheduler / system flow), or a one-shot
   batch operation? This determines most of what follows.

   **Before planning a process flow, confirm orchestration is actually warranted.** Several statements
   happening in sequence is not by itself a reason to reach for a flow — a single task, a subroutine,
   or a Default/Layout/Context control procedure often already covers it (see
   `thinkwise_software_factory_process_flows`'s `references/process_flow_design_guide.md`, "When to
   use a process flow" / "When not to"). Reserve a process flow for genuine multi-action coordination:
   a guided user sequence, an approval flow with real alternatives, an integration pipeline, a
   background queue/scheduled job, or a reusable subflow.
3. **Interaction surface** — which screen type and components fit the data and process shape:
   list/card/detail/tree, scheduler view, map? **A new `screen_type` cannot be created through the
   MCP tools** — every plan must select from screen types that already exist in the model, not
   invent one.

   **Name the subject's job before choosing its components.** Find-and-open, work queue, compare
   records, maintain a record, review history/exceptions, analyze totals, or select-in-a-lookup each
   want a different column set/sort/filter, not just a different component — see
   `thinkwise_datamodeling_guidelines`'s `references/subject_presentation_design.md`. If the same table
   needs to serve two different jobs for different users, that's the signal to plan a variant
   (`thinkwise_software_factory_variants`) rather than one screen trying to cover both.
   - **Ask the user what they want the screen to look like** — grid, tree view, card list, form,
     scheduler, map, cube/pivot, or some combination — rather than assuming one from the data/process
     shape alone.
   - Query the live model for existing screen types (`screen_type`, filtered by `model_id`/
     `branch_id` — see the unscoped-query gotcha in `thinkwise_datamodeling_guidelines`'s quirks
     section) and offer the ones that reasonably match as options, grounded in what actually exists.
   - **If no existing screen type contains the requested component** (e.g. no Scheduler-only screen
     type yet exists), say so explicitly in the plan and mark creating that screen type as a manual
     prerequisite the user must do themselves in the Software Factory UI before the rest of the plan
     can be executed — never silently substitute a different component.
   Defer how the chosen Grid/Form/Card list/Tree view itself should be configured (title/image
   source, column order, grouping, drag-and-drop) to `thinkwise_software_factory_subject_components`
   — the plan itself must still name the intended Form groups/sections as a decision line item (see
   "When you're done" below); only the field-level mechanics are deferred.
4. **Logic placement** — for each piece of business logic, decide *where it lives* and argue why:
   - **declarative modeling first** — before reaching for any control procedure, check whether the
     rule is really just a domain, mandatory property, reference, unique/check/foreign-key
     constraint, permission, filter, or workflow setting. If so, plan that instead — it's enforced on
     every write path for free and needs no logic to review or test. Only fall through to a control
     procedure once this is genuinely ruled out.
   - control procedure — pure data derivation/defaulting, no user interaction, used from exactly one
     place (see `thinkwise_software_factory_create_control_procedures`). **Name the specific concept,
     not just "control procedure"** — Default (fill/derive a value), Layout (visibility/editable/
     mandatory state), Context (task/report/detail availability), Badge (a count), Change detection
     (an auto-refresh decision), Trigger or a constraint (integrity on every write path), or Handler
     (custom GUI/API CRUD). Picking among these is a real design decision, not an implementation
     detail to leave for later — see the quick-selection table and per-concept guidance in
     `thinkwise_software_factory_create_control_procedures`'s
     `references/logic_concept_design_guide.md`.
   - **subroutine** — reusable logic with an explicit parameter/return contract, called from *more
     than one* place (two+ control procedures, a view plus a task, a process flow plus a report, or
     an external API consumer) — see "When to reach for a subroutine" below and
     `thinkwise_software_factory_subroutines`
   - task — a user-triggered action with parameters/form (see `thinkwise_software_factory_tasks`)
   - process flow — orchestration across multiple objects/steps, or anything needing a decision
     point or a scheduled trigger (see `thinkwise_software_factory_process_flows`, and step 2's
     orchestration gate above before defaulting to this option)
   Getting this placement right — and being able to say why — is the main value of this skill;
   don't let it collapse into "put it wherever."

   **The same rule can legitimately need two layers, not an either/or pick.** Example: a Layout
   disables editing on a finalized invoice for usability, while a Trigger/Handler/constraint enforces
   the same rule for integrity on every write path (API, import, direct SQL). Don't stop at the first
   layer that satisfies the UI if the requirement also has an integrity half — record both, with the
   principle for each.

   **Flag hot-path placements as an open risk.** Default, Layout, Context, Badge, Change detection,
   and Trigger all fire per-row, per-selection, or per-write — if the plan puts a non-trivial query
   behind one of these, note it under "Open risks" as worth an indexing/efficiency pass, rather than
   leaving it implicit until it's slow in practice.

   **When to reach for a subroutine, specifically:** the deciding question is *reuse*, not
   complexity. A calculation, validation, or lookup used from only one Default/Layout/Handler stays
   a plain control procedure on that one code type — wrapping it in a subroutine adds an extra call
   hop for no benefit. Reach for a subroutine the moment any of these hold:
   - The same logic needs to run from **two or more** call sites — e.g. a discount calculation used
     by both an order-total Default and a reporting view; a VAT-number check used by both a Handler
     and a Task.
   - The logic needs to be **queryable from SQL directly** (a `select`/`join`/calculated column), not
     just triggered by an event — a scalar or table function is the only placement that composes
     inside a query at all.
   - The operation needs to be **callable from outside the database** — once something is published
     via API, it has to be a subroutine (`api`/`basic_api`); a Default/Layout/Handler has no API
     surface of its own.
   - The logic is **independently unit-testable business logic** worth versioning on its own contract
     (parameters in, return out) rather than tangled into one code type's variables.
   Recording the decision: note *which* call sites justify the subroutine, not just "this could be
   reused someday" — if the plan can't name a second caller yet, plain control-procedure placement
   is the right call, but propose this as your recommendation and confirm it with the user, same as
   any other branch, and revisit if a second caller actually shows up.
5. **Entry points** — menu placement (`thinkwise_software_factory_menu`), and whether
   prefilters/variants are needed for different roles or contexts
   (`thinkwise_software_factory_prefilters`).

   **Gate every proposed menu item on "is this a genuine independent starting point?" first.**
   Something a user selects a record and then acts on — a contextual task, a record-specific report, a
   child/detail table — belongs on that subject (a detail, task/report button, process step, prefilter,
   or variant), not the menu, no matter how convenient a menu shortcut would be. Only plan a menu entry
   for a subject/task/report a user genuinely starts or resumes work from directly. See
   `thinkwise_software_factory_menu`'s `references/menu_design_guide.md` for the full in/out criteria
   and organizing-principle choice.

   **Different roles needing different visibility is a grant, not a new menu or group.** If the only
   thing distinguishing two audiences is *who's allowed to see it*, plan a role grant on the existing
   menu/group/item, not a forked menu or a role-named group — a forked structure just doubles the
   translation/testing/maintenance cost for no navigation benefit.

   **Recording the decision:** note which organizing principle (business capability, workflow, object
   family, operational horizon, or admin level) the target menu/group already uses, and place the new
   item under that same principle rather than whichever group seems locally convenient — a menu mixing
   principles at one level stops being predictable to scan.
6. **Cross-cutting concerns** — translations (`thinkwise_software_factory_translation_objects`),
   role/security visibility, and (if relevant) a maps component
   (`thinkwise_software_factory_maps_component`).

   **Layout/Context are visibility only, never the security control.** If a requirement is really
   about *who's allowed* to see or do something, the plan needs actual role rights/row-security —
   recommend Layout/Context for that requirement only as a usability nicety on top of real
   authorization, never as the enforcement itself.

   **Messages** (`thinkwise_software_factory_messages`) are cross-cutting in the same way: every
   failure mode, confirmation, and choice surfaced during step 4's logic placement needs its own
   deliberate message decision, not an afterthought bolted on once the logic already works. For each
   one, decide and record:
   - **Severity and location** — does this stop the operation (Error/Popup), flag a risk the user must
     decide on (Warning/Popup, usually with a choice), or just confirm what already happened
     (Information/Panel)? Getting this wrong is the single most common message mistake — the least
     disruptive option that's still honest about the outcome is usually right, but propose this as
     your recommendation and confirm it with the user, same as any other branch.
   - **Plain error vs. a choice message** — if there's a real "do it anyway" path, that's a Warning
     with `msg_option`s wired into a process flow's Show message action, not a hard-aborting Error;
     if there's no valid way to proceed, it's an Error, full stop, not a Warning dressed up with an
     "anyway" button.
   - **Confirmation vs. a named button** — a destructive/expensive/broad/externally-visible task
     action gets `ask_confirmation` (and, for anything acting on multiple selected rows,
     an explicit call on `popup_for_each_row` — batch confirmation is almost always right, per-row is
     the message-storm anti-pattern). Don't reach for confirmation just because a button "sounds
     risky" — a clearly-named button is often enough on its own.
   - **Reuse vs. a new message** — reuse an existing `msg_id` only when severity, location, parameter
     contract, *and* remedy are all genuinely identical to an existing one; a different remedy for the
     same underlying technical failure earns its own message.
   - **Does this need a database-capture message too?** — if the logic placement in step 4 relies on
     a database constraint as the real integrity guarantee (a unique index, an FK, a check), plan a
     contextual business validation *and*, where races or alternate write paths are possible, a
     capture message layered on top of it with a priority below the generic base-model catch-all.
   - **Will this path ever run unattended** (API, scheduled process flow, offline sync)? If so, the
     plan needs a stable non-interactive outcome alongside the message — never a design that only
     works when a human is present to click a popup.

## Recording decisions

For every resolved branch, capture:

- **Decision** — what was chosen
- **Principle** — why this over the alternatives that were on the table
- **Owning skill** — which `thinkwise_software_factory_*` skill implements it

Don't wait until the end to write these down — capture each as it's resolved so nothing gets lost
if the conversation runs long.

## Tone

Same as a thorough design review with a thoughtful colleague: direct, and willing to push back on
a default that doesn't fit, but constructive — the goal is a plan the user believes in, not a
gauntlet.

## When you're done

Produce the build plan with this content, regardless of output format:

1. **Goal statement** — the confirmed anchor from step 1
2. **Data model deltas** — new/changed tables, columns, domains (if any)
3. **Artifact list** — every screen/task/process flow/control procedure/scheduler/view/prefilter/
   menu entry needed, each with its decision, principle, and owning skill from "Recording
   decisions" above — including Form groups/sections (and any collapsible section) for a table
   gaining or changing a Form, not just the screen/task-level artifacts
4. **Sequencing** — build order, respecting the dependencies surfaced during the interview, and
   listing any manual screen-type-creation prerequisite from step 3 first, since nothing downstream
   of it can be built until the user has done it
5. **Open risks** — anything still uncertain or deferred
6. **Final verification gate** — always the last line item, regardless of how small the plan is: run
   the translation completeness query from `thinkwise_datamodeling_guidelines`'s "Translating new
   objects" section across the whole branch before considering the plan executed. Make this an
   explicit item in the plan itself, not an assumed follow-up left to whichever skill happens to
   create the last object — a real build translated the new tables it created and still left every
   one of their columns at bracket-placeholder text, because nothing forced a final check.

This plan is the handoff point: implementation proceeds by invoking each named
`thinkwise_software_factory_*` skill in sequence, not by this skill making model changes directly.

### Choosing the output format

A plan this small stays as plain text in the chat, formatted with headings/lists as usual — no
artifact:

- Everything fits in a single category from "Artifact list" (e.g. one new control procedure, one
  task tweak), or
- The full artifact list has roughly 5 items or fewer with no more than two categories touched.

Otherwise — spans three or more of the categories below, or has more than ~5 artifact-list items —
render the plan as a Thinkwise-branded HTML **Artifact** instead of a chat wall of text, so the user
can scan and drill into just the sections they need:

1. **Load the `artifact-design` skill first** to calibrate the design for this plan, then open
   `references/plan_artifact_template.html` in this skill's folder as the starting structure —
   copy it, don't rebuild it from scratch.
2. **Header, always expanded, never collapsible**: the Thinkwise logo (inline the SVG from
   `references/assets/thinkwise_logo_inline.svg` — never link it externally, Artifacts must be
   self-contained), the plan title, the confirmed **goal statement** from step 1 verbatim, and a
   short **summary** paragraph (2-4 sentences: what's being built and the overall shape of the
   solution) written fresh for this plan.
3. **One collapsible accordion group per touched category**, in this order, each titled and using
   native `<details>/<summary>` (accessible and theme-safe with no extra JS) — omit any group with
   nothing to report, never ship an empty accordion section: Data Model, Screens & Interaction,
   Form Groups & Sections, Tasks, Process Flows, Control Procedures & Logic, Subroutines, Messages,
   Menu & Entry Points, Cross-Cutting Concerns, Unit Tests, Sequencing, Open Risks. Each item inside
   a group shows its
   Decision, Principle, and Owning skill, exactly as captured under "Recording decisions" — the
   accordion is a presentation layer over that same content, not a reason to capture decisions any
   differently during the interview.
4. **Final verification gate renders as an always-visible callout below the accordions, never
   collapsed inside one** — it must never be one tap away from being missed.
5. Keep the Thinkwise brand colors from the template (the blues from the logo) for accents, headers,
   and the expand/collapse controls, and follow the template's light/dark token structure so the
   artifact matches the viewer's theme.
6. Give the artifact a stable file path so re-publishing later revisions of the same plan updates
   the same artifact rather than minting a new URL each time, and pick a favicon that fits (e.g. a
   clipboard or blueprint emoji).

When in doubt between the two formats, default to plain chat text — only reach for the artifact
once the plan is genuinely too broad to scan as a wall of text.
