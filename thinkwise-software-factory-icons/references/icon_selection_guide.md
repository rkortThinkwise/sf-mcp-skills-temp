# Choosing the right icon

Design guidance derived from Thinkwise Platform documentation and recurring model usage — not a
database rule. See `../SKILL.md` for the entities/fields this guidance gets attached to.

## The core rule: communicate meaning, not decoration

An icon should help a user recognize one of three things:

1. **Object** — what is this? Customer, order, warehouse, report.
2. **Action** — what will happen? Add, approve, import, print.
3. **State** — what is true? Completed, warning, locked, offline.

Don't choose an icon because it looks attractive or fits the available space. If users can't predict
its meaning, use text or icon+text instead.

## Icons do not replace labels

Prefer **icon + text** for: business-specific tasks, destructive/irreversible actions, infrequent
actions, several similar actions shown together, icons whose meaning changes by context, and new
functionality users haven't learned yet.

Icon-only is fine for highly conventional actions (Search, Refresh, Close, overflow) — and even then a
tooltip/accessible name is required, since touch users can't hover. `action_bar.default_display_type`
is the modeled fallback chain (icon+text → text-only / icon-only → overflow → hidden as space runs
out) — design every action so it stays understandable in every configured fallback mode.

## One semantic concept, one icon

Within an application: one plus icon for Add, one trash icon for Delete, one check-circle family for
"completed," one warning family, the same customer icon everywhere Customer is the represented object.
Never let one icon mean different things in different places (a gear meaning Settings here, Execute
there, Processing status elsewhere) — silhouette and semantic family should stay recognizable even when
the presentation size changes.

## Action icon patterns

| Action | Recommended concept | Avoid / confusion risk |
|---|---|---|
| Add/create | Plus, plus-circle, document-plus for document creation | Star, check |
| Edit/modify | Pencil, pen-to-square | Wrench (reads as maintenance/settings) |
| Save/confirm input | Check or save/document symbol, paired with a label where ambiguous | Check also used for bare status |
| Copy/duplicate | Overlapping documents/squares | Clipboard (reads as "copy to clipboard") |
| Delete/remove | Trash can; minus for removing from a collection | Cross (reads as cancel/close) |
| Cancel/close | X/cross | Trash can |
| Approve/accept | Check-circle, stamp/check | Plain Save icon |
| Reject/decline | X-circle, ban symbol | Trash can, unless data is actually deleted |
| Refresh/reload | Circular arrows | Sync icon, if the effect has wider side effects |
| Synchronize | Bidirectional/circular sync arrows, optionally with cloud/database context | Refresh, if remote data is actually changed |
| Search/find | Magnifying glass | Filter/funnel |
| Filter | Funnel | Magnifying glass |
| Sort | Ordered arrows/lines | Filter funnel |
| Import | File/box with an arrow into the system | Export arrow; a bare direction arrow with no containing object |
| Export | File/box with an arrow out; download if the result downloads | Share, when nothing is shared externally |
| Upload | Arrow to cloud/system | Import, if the business import also validates/persists |
| Download | Arrow from cloud/file to device | Export, if it actually starts a business export job |
| Print | Printer | PDF icon, unless output is specifically PDF |
| Generate report | File-chart/report | Printer, if nothing is printed |
| Open/navigate | External/open arrow, folder-open, arrow-right | Edit pencil |
| Lock/unlock | Closed/open padlock | Key, unless it specifically concerns credentials/access |
| Archive | Archive box | Delete/trash |
| Restore | Undo/restore arrow | Refresh |
| Execute/run | Play triangle | Gear by itself |
| Stop | Square/stop-circle | Delete/trash |
| Retry | Rotate arrow with retry context | Refresh, when it would repeat side effects |
| Configure | Gear/sliders | Wrench (reads as repair) |
| Repair/maintenance | Wrench/tools | Gear (reads as ordinary settings) |

**Directional actions need labels.** Import/export, upload/download, inbound/outbound, receive/send are
easily confused because arrow meaning depends on viewpoint — pair the icon with a translated verb, and
put a file/cloud/system-boundary shape in the silhouette where possible.

## Task icons: outcome, not "this is a task"

A task icon should describe the outcome/verb, not merely flag "this is a task": Release order →
unlocked/check or forward-to-production; Create shipment → truck/box-plus; Approve invoice →
invoice/check; Recalculate price → calculator/refresh; Send email → envelope/send; Generate labels →
tag/barcode/printer; Synchronize products → product/sync. Don't give every task a gear, play button,
lightning bolt, or generic tool icon — none of those distinguish actions in a task bar.

**Destructive tasks** need the consequence represented accurately, not just "feels negative": Delete
record → trash; Cancel order → X-circle/cancelled document; Reject application → reject/stamp-X; Remove
item from selection → minus/unlink. These carry different business effects and confirmation text even
though all feel negative.

**Task variants**: change the icon only when the variant represents a visibly different outcome or
context. If a variant just prefills different parameters for the same action, keep the base task's
icon and let translation communicate the distinction — see `thinkwise_software_factory_variants`.

## Subject/table icons

A subject icon represents the business noun shown in the screen/document/menu: Customer →
person/company; Supplier → building/handshake; Employee → person/badge; Product/article/material →
box/cube/tag; Order → document/cart/clipboard fitting the domain; Production order →
factory/gears/document; Warehouse/inventory → warehouse/boxes; Shipment/delivery → truck/package;
Planning/appointment → calendar/timeline; Invoice/finance → invoice/currency; Asset/machine →
machine/tool; Location → map pin; Message/email → envelope/chat; Audit/history → clock/history/list;
Configuration → gear/sliders. Use the same subject icon for its document, menu item, and direct
navigation unless another context truly changes the represented object.

**Tables vs. views**: a view used as a business work queue should get the *work* concept, not a generic
database/view icon — `orders_to_release` → order/check or queue icon; `failed_integrations` →
plug/warning; `production_dashboard` → factory/chart. Reserve database/table/view symbols for
technical/administrative subjects where the implementation object is itself what users manage.

## Menu icons

- **Menu icon** (`menu.icon_id`): represents an application/module when users switch between several
  menus — broad and stable (Warehouse, Planning, Finance, Administration), not the icon of one
  frequently-used screen.
- **Menu-group icon** (`list_bar_grp.icon_id`/`tile_grp.icon_id`/`module_grp.icon_id`): the shared
  business category — Sales → cart/handshake; Purchasing → basket/order-in; Warehouse → warehouse/boxes;
  Finance → currency/ledger; Reports → chart/file; Master data → database/list/catalog; Settings → gear;
  Administration → tools/user-shield. Don't assign an icon to *every* group when labels already scan
  clearly — in dense tree/list menus consistent icons aid recognition, but inconsistent/decorative ones
  add noise.
- **Menu-item icon**: there is no separate field — a `list_bar_item`/`tile` always shows the referenced
  `tab`/`report`/`task`'s own icon (confirmed live: neither entity carries an `icon_id`). Don't try to
  invent a menu-only vocabulary for the same destination.
- **Tiles** rely more heavily on icon recognition than list bars — use a limited number of visually
  distinct icons, large simple silhouettes, icon+short label, consistent stroke weight, and no fine
  internal detail that disappears at tile/mobile sizes.

## Report icons

Choose between **topic**, **output format**, and **action**: Sales analysis → chart/trend; Invoice
document → invoice/file-currency; Picking list → checklist/warehouse; Label report →
tag/barcode/printer; PDF export → PDF/file icon, when PDF itself is the important promise; Spreadsheet
export → spreadsheet icon; Print report → printer, only when immediate printing is the actual outcome.
If several reports in one group share the same output format, a format icon won't distinguish them —
prefer topic icons plus translated labels.

## Prefilter icons

A prefilter icon represents the **resulting subset**, not the filter operation itself — the toolbar
already communicates "this is a filter"; the icon needs to say *which* subset. Good patterns: My work →
person/check or user; Open → open circle/inbox; Completed → check-circle; Overdue → clock/exclamation;
Errors → circle-X/warning; Active → play/check; Inactive → pause/ban; Archived → archive box; Favorites
→ star; Today → calendar-day. Don't give every prefilter a funnel icon, and use icons sparingly when
many prefilters are visible at once — similar colored circles with minor differences scan worse than
clear short labels.

## Status / domain-element icons

Applies to `elemnt.icon_id` when the owning domain's control renders elements as icons (an image/icon
combo or radio-style control) rather than as plain text.

| State | Recommended concept |
|---|---|
| New/draft | Document, pencil, outlined circle |
| Pending/waiting | Clock/hourglass |
| In progress | Play/progress/spinner concept |
| Completed/success | Check-circle |
| Warning/attention | Triangle-exclamation |
| Failed/error | Circle-X |
| Cancelled | Ban/circle-slash |
| Paused/on hold | Pause-circle |
| Locked | Padlock |
| Archived | Archive box |
| Offline/disconnected | Cloud/network slash |
| Synchronized | Sync/check |

Rules: pair the icon with translated text or a tooltip — never depend on shape/color alone. Give every
status a unique silhouette, not just a different hue. Keep the same workflow icon consistent across
grid, form, filters, badges, and reports for the same status. Don't reuse a green check for both "valid"
and "selected" in the same control. Provide a neutral icon for unknown/not-applicable rather than
showing an error glyph. Avoid culturally or socially loaded person/gender imagery unless the business
meaning requires it and it's been reviewed.

## Form navigation icons (`form_next_grp_icon_id` / `next_tab_icon_id`)

These render small (historically ~16×16) and should be very simple — a "jump to related group/tab"
affordance, not a full illustration. Good uses by target subject: Address → map pin/home; Contact →
person/phone/envelope; Planning → calendar; Financial → currency/calculator; Attachments → paperclip;
Audit/history → clock/history; Security → lock/shield; Integration → plug/sync; Advanced settings →
sliders/gear. Don't add an icon to every group — labels and whitespace often provide better hierarchy
on their own; an icon earns its place for collapsed sections, repeated patterns across forms, or groups
needing quick recognition. A collapsible section should keep a meaningful label even with an icon — a
cog alone doesn't say whether the section holds pricing config, integration settings, or permissions.

## Message-option and process-choice icons (`msg_option.icon_id`)

Reinforce distinct outcomes: Continue/accept → check/arrow-forward; Retry → retry arrow; Return to edit
→ pencil/back; Cancel → X; Stop process → stop-square; Open result → external/open-document. Keep
affirmative/negative response semantics consistent with process routing — the icon reinforces the
translated option label, it never substitutes for it. See `thinkwise_software_factory_messages`.

## Integration/automation icon concepts

Useful distinctions for web connections, message brokers, and similar integration objects: API/general
connector → plug/braces/network nodes; Database → database cylinder; Web/cloud → globe/cloud;
MQTT/message broker → broadcast/waves/message queue; File exchange → file with directional arrow;
JSON/XML transformation → braces/code/file-code; Authentication/token → key/shield/lock; Queue →
stacked list/inbox; Synchronization → circular/bidirectional arrows; Failed integration →
connector/cloud plus warning. Don't use a vendor logo unless the vendor identity actually matters to
users, usage is permitted, and the flow specifically targets that vendor — generic technology icons
stay stable when providers change. See `thinkwise_software_factory_web_connections`.

## Format, color, size, accessibility

**Prefer SVG.** It scales without quality loss across task bars, menus, tiles, mobile screens, and
high-density displays, and Universal UI can auto-adjust its color for light/dark themes when the SVG
doesn't embed fixed color — use `task_enrichment_decolorize_svg` (see `../SKILL.md`) for a suitable
monochrome asset that still carries baked-in color. Reserve raster (PNG/JPEG/BMP) for photographic or
detailed brand illustrations, or when multicolor meaning is essential and no SVG is available at
verified quality. Avoid GIF animation for ordinary navigation/action icons — motion distracts, can
affect accessibility, and looks inconsistent with Universal UI.

**Color**: default to monochrome/theme-aware icons for navigation, object, and action icons. Use
semantic color only for an established state (red/error, amber/warning, green/success,
blue/information-or-neutral-primary) — never as the *only* signal, never baked into every SVG as a
theme-dependent choice, and always checked against main/menu/accent/hover/disabled/selected backgrounds
in both light and dark themes. Primary-action emphasis belongs to the action bar's own "Primary action"
setting, not to uploading a uniquely bright icon.

**Size**: common rendered sizes are ~16×16 for form groups/sections, context-menu actions, and compact
task/report/prefilter use; ~20×20 for document/table/detail icons; ~24×24 for list-bar groups; and
16/24/32/48 for configurable action/task bars. SVG avoids needing separate files per size, but the
drawing still has to read at the smallest one — square view box, adequate padding, consistent stroke
width, no tiny text or hairline detail, a recognizable silhouette at 16px, similar optical weight to
neighboring icons.

**Accessibility**: always provide a translated accessible/action name (the icon is never a substitute
for the object/task/report/prefilter's own translation and tooltip/help text). Never encode state by
color alone. Don't rely on hover-only explanation — touch users can't hover. Use icon+text for
unfamiliar/consequential actions. Keep icons visible in high-contrast, dark mode, selected, hover,
focus, and disabled states. Keep touch targets large even when the glyph is small. Never use two
visually similar icons for opposite outcomes. Test at browser zoom and on high/low-density displays.
Respect reduced-motion needs.

## Icons vs. other presentation mechanisms

Use an **icon** for stable object/action/status recognition, a **badge** for a dynamic count/small
state indicator, **conditional formatting** for data-driven emphasis across cells/rows (see
`thinkwise_software_factory_conditional_layouts`), a **message** for an outcome that needs explaining
(see `thinkwise_software_factory_messages`), **label/help text** for precise or unfamiliar meaning, and
an **image** for actual content (a product photo, a signature). Don't use an icon as a miniature
dashboard, encode a numeric count into a static image, or replace a validation message with a red
symbol and no explanation.

## Repository organization, naming, and reuse

**Groups** (`icon_grp`): organize by design system or semantic family — Actions, Status, Navigation,
Business objects, Finance, Logistics, Planning, Integration, Files and reports, Administration,
Brand/application. Avoid grouping by source vendor ("Icons8 downloads") — people search by meaning, not
procurement source.

**Naming**: use stable semantic names (`action-add.svg`, `status-warning.svg`, `object-customer.svg`,
`object-production-order.svg`, `group-finance.svg`, `integration-api.svg`). Avoid vendor/size/version
cruft (`icons8-product-loading (5).svg`, `blue.png`, `new-icon-final-2.svg`, `image48.png`, a bare size
as the name when SVG is used). If one icon is shared across contexts, give it a neutral semantic name,
not the name tied to its first assignment.

**Before adding an icon**: search the icon repository (and relevant `icon_grp`/`icon_tag` values) for an
existing match, check usage of likely candidates, verify theme and small-size behavior, and add only if
nothing in the repository communicates the concept consistently already.

**Before updating/replacing a widely-used icon**: inspect every place it's assigned (the table in
`../SKILL.md`), confirm every assigned object genuinely shares the intended meaning, use
`task_update_icon_usage` to consolidate duplicates rather than hand-editing every field, and
regression-test menus, documents, action bars, domain controls, and both themes afterward. Don't edit a
widely-reused icon to fit one exceptional screen — add a distinct icon instead if the meaning truly
differs there.

## Common anti-patterns

- Mixed visual families (outline, filled, skeuomorphic, clip-art) in the same application.
- Duplicate uploads under numbered filenames (`(1)`, `(2)`) for the same concept.
- A generic gear on every task — nothing distinguishes them in a task bar.
- A funnel on every prefilter — subset meaning stays invisible.
- Trash used for cancel/reject/unlink — different consequences look identical.
- Ambiguous import/export arrows with no direction context.
- Status communicated by color only — inaccessible and theme-fragile.
- Raster icons reused at multiple sizes — blur, or force duplicate assets.
- Embedded black SVG fill — disappears or looks wrong in dark mode.
- Decorative group icons on already-dense forms/menus.
- Icon-only presentation on a business-specific or destructive task.
- The same icon for opposite actions (approve/reject, lock/unlock) differing only by tooltip.
- A vendor logo standing in for a generic integration concept.
- A technical database/table/view icon on what's actually a business work queue.
- Editing a shared icon locally to fix one screen, changing unrelated screens through reuse.
- Filenames used as taxonomy (source/vendor/size) instead of semantic naming.

## Recommended selection workflow

1. Classify the object as noun, action, or state.
2. Write the exact translated label first.
3. Choose the simplest conventional concept matching that label.
4. Search and reuse the repository before uploading anything new.
5. Prefer a monochrome, decolorized SVG from an already-approved family.
6. Check whether the icon conflicts with another meaning used nearby.
7. Choose icon+text for unfamiliar, destructive, or business-specific actions.
8. Check the smallest rendered size and the responsive fallback (`action_bar.default_display_type`).
9. Check light/dark, hover, selected, focused, and disabled states.
10. Verify the tooltip/accessible name and touch recognition.
11. Inspect usage before changing or consolidating an existing icon.
12. Record the mapping (concept → `icon_id`) in the project's own convention so it's reused consistently.

## Baseline vocabulary worth establishing per application

Add, edit, save/confirm, copy, delete, cancel · Approve, reject, archive, restore · Search, filter,
sort, refresh, synchronize · Import, export, upload, download, print · Settings, administration,
security, history · Information, warning, error, success, pending, in-progress · Customer/person,
company/supplier, product/material, order/document · Calendar/planning, warehouse/inventory,
shipment/truck · Finance/currency, report/chart, attachment/file · API/integration, database, cloud,
queue.

Document the exact repository `icon_id` behind each of these once, and reuse it everywhere the concept
recurs.

## Final review checklist

- Does the icon represent an object, action, or state clearly?
- Does its label remain the primary, precise meaning (not just the icon)?
- Is the concept conventional for the target users?
- Is the same semantic icon reused elsewhere for the same meaning?
- Is it visually distinct from nearby and opposite actions?
- Is a destructive consequence represented accurately, not glossed over?
- Would icon+text be safer here than icon-only?
- Is SVG suitable, and free of unwanted embedded color?
- Does it work in both light and dark mode?
- Is it recognizable at 16–20px and at browser zoom?
- Is meaning preserved without color or hover?
- Are accessible name, tooltip, focus, and touch behavior correct?
- Is the visual family/stroke weight consistent with neighboring icons?
- Does a prefilter icon show the subset, not a generic funnel?
- Does a task icon show the outcome, not a generic gear?
- Does a report icon show topic/format appropriately?
- Do variants keep the base icon unless their semantics genuinely differ?
- Has repository usage been checked before replacing a shared icon?
- Are duplicate assets and poor filenames being consolidated (`task_update_icon_usage`)?
- Has the icon been checked in every context where it's assigned?
