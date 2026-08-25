# Menu information architecture — what to put where, and why

This file is about *design*, not API mechanics: whether something belongs in the menu at all, how to
organize and name groups, how to order things, and how to review the result. For menu type choice,
new-menu-vs-existing, new-group-vs-existing, and security-by-grant, see `SKILL.md` — that framework
already covers those decisions well. For the actual entity/task calls once a decision is made, see
`SKILL.md`'s "Creating things" reference.

## What belongs in the menu

Add an object only when it's a legitimate **starting point** for at least one target user — the menu
is a curated set of places to begin, not an inventory of every model object.

**Include:**
- **Primary subjects** users initiate or resume recurring work from — orders, customers, production
  planning, warehouse movements, service calls, timesheets, management dashboards. Point the item at a
  useful list/dashboard/queue/search view, not an implementation table that forces users to understand
  foreign keys.
- **Work queues and exception lists** — "Orders to approve," "Picking work," "Failed integrations" —
  when genuinely actionable. Prefer one subject variant or prefiltered subject over many near-identical
  entries per status (see `thinkwise_software_factory_prefilters` / `thinkwise_software_factory_variants`).
- **Global tasks** startable without selecting a record — start a planning run, import orders,
  synchronize master data, open a work shift. If the task acts on a *selected* record, it belongs on
  that subject's action bar/context instead, not the menu (see `thinkwise_software_factory_tasks`).
- **Global or frequently-requested reports** whose parameters supply enough context on their own.
  Reports tied to a selected customer/order/invoice/shipment are usually better offered from that
  subject.
- **Configuration and master data** — only for the users responsible for maintaining it, in a clearly
  separated Settings/Configuration/Master data/Administration group, normally ordered after operational
  groups.

**Do not include:**
- **Every table in the data model.** Link tables, child tables, history tables, staging tables, and
  technical configuration don't become useful navigation just because they have a generated screen. If
  an object is naturally reached through a master-detail relationship, keep it a detail — direct menu
  access removes the context needed to interpret or safely edit it.
- **Contextual tasks and reports** — "Cancel order," "Print this invoice," "Release selected jobs"
  require a current record; they don't belong in global navigation.
- **One-off implementation/migration utilities** — deployment fixes, developer diagnostics, seed-data
  tasks, repair operations, temporary migration screens. Put supported admin tools in a restricted
  advanced/admin menu; remove temporary ones once their purpose ends.
- **Duplicate paths without a demonstrated user benefit.** An object appearing in several groups/menus
  just makes users unsure which location is canonical — only duplicate when distinct audiences
  genuinely use different vocabulary or workflows.
- **Empty organizational headings** that mirror a department chart without the items forming a coherent
  user task — organizations change more often than business capabilities do.
- **External destinations masquerading as application subjects** — use an intentional integration,
  process action, or clearly labeled link instead of a vague item that unexpectedly leaves the app.
- **Security-sensitive objects relying on menu absence as their protection** — hiding an item is not
  sufficient; deny it through roles (see `SKILL.md`'s "Security: grant, don't fork").

## Choosing the organizing principle

Use exactly one primary organizing principle at each level — a user should be able to infer the rule
from the group names alone. Don't mix incompatible categories (e.g. **Sales**, **Reports**, **Tables**,
**John's screens**) at the same level.

| Principle | Examples | Best fit |
|---|---|---|
| Business capability/domain | Sales, Purchasing, Warehouse, Finance, Service | Strongest general-purpose pattern for broad back-office apps — vocabulary stays stable across roles |
| User workflow | Plan, Execute, Review, Close period | Focused operational apps with a clear lifecycle; don't mix workflow stages with domains at the same level |
| Object family | Customers, Products, Employees, Assets | Master-data-heavy apps, or one level below a business-domain group |
| Operational horizon | Today, Planning, History, Exceptions | Shop-floor, fieldwork, logistics, control-room — urgency matters more than department ownership |
| Administration level | Operations, Master data, Settings, Administration | Deliberate separation between everyday work and occasional maintenance |

### Common group patterns (condensed)

- **Domain-first back-office**: Sales / Purchasing / Warehouse / Finance / Master data / Administration
  — each domain holding its own subjects (e.g. Sales → Customers, Quotations, Orders).
- **Work-first operational**: My work / Exceptions / Planning / Execution / History — use variants or
  prefilters so "My work"/"Exceptions" open actionable lists, not generic tables.
- **Shop-floor/warehouse launchpad**: Start work / Scan / Current order / Material / Problems / Shift —
  usually a tile menu with a handful of large targets; keep admin data out.
- **Management and reporting**: Overview / Performance / Exceptions / Reports / Configuration — group
  reports by decision/business topic once the list grows, not by file format.
- **Core versus administration**: Daily work / Master data / Settings / Administration — keeps
  occasional maintenance from crowding operational navigation.
- **Main versus advanced menu**: a `1_main_menu`/`2_advanced_menu`-style split works when advanced users
  need direct access to diagnostic/configuration subjects that would confuse the general audience — the
  advanced menu still needs proper authorization and must not become a dumping ground.
- **Import/export/API group**: only when integration monitoring and manual exchange are real,
  standalone user responsibilities. If importing is just one step in Sales/Purchasing, keep it with
  that business process instead of forming a technical integration silo.

## Naming menu groups

- **Prefer recognizable business nouns** — Sales, Warehouse, Production planning, Customer service,
  Master data, Administration. Avoid General, Other, Miscellaneous, Tables, Module 1, Processing — a
  generic name hides the organizing principle; if "Other" grows past one or two exceptions, the
  taxonomy needs revision.
- **Keep labels short** — one to three words; put explanations in help text, not the label.
- **Use the users' vocabulary** — match terminology from screens/processes/documentation/training;
  never expose technical IDs like `tab_sales_hdr` or internal team names.
- **Use parallel grammar within a level** — either a noun set (Sales, Purchasing, Warehouse) or an
  activity set (Plan, Execute, Review), not a mixture (Customers, Create invoice, Warehouse management,
  Done).
- **Avoid redundant suffixes** — "Sales" beats "Sales menu group"; inside Administration, "Users" beats
  "User administration management."
- **Translate labels, stabilize IDs** — lower-case, consistent IDs (`production_planning`), translated
  into every application language (see `thinkwise_software_factory_translation_objects`). Don't encode a
  temporary organizational owner or display order into the ID — sequence belongs in `order_no`, not the
  name (see "Ordering" below).

## Ordering groups and items

Recommended top-level order, by frequency and workflow rather than alphabet:

1. Home, personal work, or current work.
2. Primary operational domains, in workflow order.
3. Monitoring, exceptions, and history.
4. Reports and analysis.
5. Master data.
6. Settings and administration.

Within a group: put the most common starting points first, keep related items together, sequence
lifecycle items naturally, keep destructive/exceptional tasks away from routine navigation, and use
alphabetical order only when users already know the item name and no stronger sequence applies. Set
`order_no` directly to express this — never encode order in the ID (e.g. `1_main_menu` reads fine once,
then goes stale the moment priorities change).

## Group size and menu search

- A group with roughly **2-8 distinct entry points** is usually easy to scan.
- **10 or more items** should trigger a review — subgroups, stronger ordering, search, contextual
  actions, or removing items that aren't real starting points.
- A **one-item group** needs a clear reason, especially in a list/tree menu.
- These are review signals, not hard limits — a familiar alphabetical catalogue can legitimately be
  longer, while a high-pressure operator menu should be much shorter.
- Turn on menu search (`menu.show_filter`) once a menu has enough destinations that recall and
  hierarchy alone are inefficient. Search complements a good structure — it does not excuse a poor one,
  and shouldn't be relied on in scanner/touch/high-speed operational flows where users should recognize
  one of a few targets immediately.

## Contextual navigation instead of menu growth

Many items found crowding an oversized menu belong closer to their context instead:

- **Details** for child data (reached via master-detail, not a top-level menu entry).
- **Task/report buttons** on the subject, for record-specific actions and output
  (`thinkwise_software_factory_tasks`).
- **Process flows** for guided multi-step work (`thinkwise_software_factory_process_flows`).
- **Prefilters** for alternate states of the same subject (`thinkwise_software_factory_prefilters`).
- **Variants** for role- or purpose-specific presentation (`thinkwise_software_factory_variants`).
- **Dashboards, badges, and work queues** for exceptions
  (`thinkwise_software_factory_create_control_procedures`'s Badge guidance,
  `thinkwise_software_factory_cubes`).
- **Deep links** from notifications for a specific record.

The menu's job is to get users into the right workspace; the workspace then exposes the next relevant
actions. When a candidate menu item is really "act on this specific record" or "see more about this
record," that's a signal it belongs on the subject, not the menu.

## Icons and visual hierarchy

- Use icons with established meaning and reuse them consistently; never rely on color/icon alone —
  always pair with a clear label.
- Avoid a unique decorative icon per item; it raises learning effort for no navigational benefit.
- In a tile menu, reserve large/wide tiles for genuinely frequent or important starting points — tile
  size should express real priority, not decoration.
- Test labels and tile wrapping in every application language, and touch targets on rugged/gloved
  devices where relevant.

## Open documents

Keep "show open documents" enabled when users work across several subjects and benefit from resuming
context. Consider disabling it for kiosks/shared terminals, highly guided operational flows, and
privacy-sensitive shared-device scenarios, or where open documents would displace the few essential
actions on a small menu.

## Anti-patterns

- **Alphabetical schema menu** — `A_tables`, `B_tables`, etc. exposes implementation structure instead
  of business meaning.
- **Role-menu explosion** — a full copy of the menu for every small permission difference (use grants
  instead — see `SKILL.md`'s "Security: grant, don't fork").
- **Device clones** — desktop/mobile menus that stay identical but now require double maintenance.
- **Technical group names** — API, DB, Tables used where users think in business processes.
- **Administration mixed into operations** — settings appearing between daily-work items.
- **Catch-all groups** — General/Other steadily accumulating unrelated screens.
- **Direct child-table access** — users opening context-dependent records without their master.
- **Contextual tasks in global navigation** — execution starting without a reliable record selection.
- **Report dump** — dozens of reports grouped only because they're reports, with no topical structure.
- **Deep tree** — users forced to remember several levels of arbitrary categories.
- **Duplicate canonical paths** — the same destination reachable from everywhere, with no clear "real"
  location.
- **Authorization by hiding** — sensitive objects absent from the menu but still reachable elsewhere.
- **Order encoded in names** — prefixes like `1_`/`2_` go stale the moment priorities change; use
  `order_no`.
- **Empty groups after authorization** — a role sees a heading with no usable destination underneath.
- **Untranslated IDs** — raw technical menu/group IDs leaking into the UI (see `SKILL.md`'s
  translation section).

## Recommended design workflow

1. Identify user personas, devices, recurring goals, and frequency of work.
2. List legitimate starting points — not every table, task, and report.
3. Choose one organizing principle for the top level.
4. Cluster entry points using language users recognize.
5. Choose list bar / tile / tree based on hierarchy and interaction context (see `SKILL.md`'s "Which
   type to use").
6. Decide whether one menu can adapt through authorization, or navigation truly differs (see
   `SKILL.md`'s "New menu, or the existing one?").
7. Order groups and items by frequency and workflow.
8. Move contextual items into subjects, details, or process flows.
9. Add translations, help text, and consistent icons.
10. Configure menu/platform availability and role rights.
11. Test reachable objects and direct routes.
12. Usability-test with real users — ask them where they expect to find common tasks.
13. Measure search use, navigation errors, and seldom-used items after deployment.
14. Remove obsolete paths and consolidate categories as the application evolves.

## Review questions for every group

- Can a target user predict what belongs here from the label alone?
- Does every item share the same organizing principle?
- Is the group named in business language, not technical/internal terms?
- Is it short and scannable?
- Are its most frequent items first?
- Should any item actually be a detail, task button, prefilter, or process step instead?
- Would authorization leave the group empty or incoherent for some roles?
- Is this group clearly distinct from its neighbors?
- Does "Other"/"General" conceal a missing category?
- Are labels and icons clear in every language and screen size?

## Review questions for a proposed separate menu

- Is the navigation model materially different, or only the permission set? (If only permissions,
  use a grant — see `SKILL.md`'s "Security: grant, don't fork.")
- Does it serve a genuinely distinct device, persona, kiosk, guest, or advanced context?
- How much content overlaps the existing menu?
- Could one shared menu plus role rights remain coherent instead?
- Who owns keeping it in sync when a shared subject changes?
- Is the default menu correct for each platform/context?
- Are all menus covered by translation, authorization, and regression testing?
- Have reachable objects been checked independently of menu visibility (see `SKILL.md`'s
  `menu_tab`/`menu_report`/`menu_task` cross-reference)?
