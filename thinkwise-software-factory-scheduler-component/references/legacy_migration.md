# Migrating from the legacy Resource Scheduler extender

| Old concept (Resource Scheduler / extender) | New concept (Scheduler component) |
|---|---|
| Three subjects: Resource, Task/Activity, Worktime | One subject/view, each resource producing multiple rows — the `UNION ALL` shape above |
| Object model extender (hand-written Windows GUI code) | Native Universal UI component, no extender code |
| Worktime table driving available/unavailable rendering | No first-class "worktime" — ordinary activities styled distinctly, or an hour-based time-cell condition |
| Zoom slider | Multiple named `scheduler_view` rows, each with its own timescale/hidden-days/business-hours |
| Separate Expand/Collapse control | Folded into the Action Bar (2025.2+) |
| Built-in date-picker dropdown | Recreate with a DUMMY task + `activate_scheduler` process action (790) |
| Auto-select a resource on screen load | Not available — only activities can be auto-selected on load |

**Migration checklist:**
1. Consolidate the three old subjects into one view; decide the activity grain first, then union in
   resource-only rows (null start/end date) exactly like the department/team/employee pattern above.
2. Pick a single-column, non-nullable primary key — don't carry over a composite key, and don't let a
   nullable column from a Worktime/absence union branch leak into it.
3. Re-model resource grouping (single column or hierarchy) from the old Resource table's structure.
4. Decide Worktime's fate: fold it into the same view as styled, non-draggable activities, or leave a
   written-but-disabled `UNION ALL` branch for it (the reference model's own low-risk choice).
5. Rebuild zoom levels as named `scheduler_view` rows instead of a single zoom control.
6. Recreate date-jump navigation with a DUMMY task + `activate_scheduler` process action.
7. Re-wire double-click and Add-activity tasks — these don't carry over from extender event handlers.
8. Re-test drag-and-drop specifically: confirm Update permission on the new subject, and add an
   instead-of trigger for resource-dragging unless using the prefixed-synthetic-key trick.
