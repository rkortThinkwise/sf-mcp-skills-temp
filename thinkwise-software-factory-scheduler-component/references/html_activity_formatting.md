# HTML and multiline activity formatting

Set the domain behind the title/tooltip column to control type **HTML** or **MULTILINE**. The column
is usually a calculated/expression column building an HTML string with `concat`. **Verified**:
`PROJECT_MANAGER`'s `title` column uses domain `title_html` (control `HTML`), while `description`
(the tooltip) uses domain `scheduler_tooltip` (control `MULTILINE`, plain text) — HTML isn't
automatically the right choice for every rich-text column.

## The dispatcher pattern: one domain, ten layouts

The reference model drives `title_html` from an `OUTER APPLY … CASE` keyed on a domain column
`activity_display_type` — effectively a small, reusable design system for activities, selectable per
row. Verified real domain, 10 elements:

| `db_value` | `elemnt_id` | Renders |
|---|---|---|
| 0 | `outlook_second_line` | Bold title, italic second line underneath |
| 1 | `status_round` | Filled circle swatch + title |
| 2 | `status_rounded_rectangle` | Rounded-square swatch + title |
| 3 | `status_slim_bar` | Thin vertical bar swatch + title |
| 4 | `status_outlined` | Outlined circle swatch (border colour only) + title |
| 5 | `pill` | Title + small rounded badge |
| 6 | `progress_bar` | Percentage slider (below) |
| 7 | `activity_card` | Multi-line card: eyebrow label, title, avatar initials + name |
| 8 | `activity_card_compact` | Same card, no avatar row |
| 9 | `none` | Plain title, no HTML wrapper — needs an ordinary conditional layout (above) instead |

## Status colour, computed once and reused

```sql
outer apply (
  select case
    when s.task_status = 0 then '#00bcd4'
    when s.task_status = 1 then '#2196f3'
    when s.task_status = 2 then '#009688'
    when s.task_status = 3 then '#ffeb3b'
    when s.task_status = 4 then '#8bc34a'
    when s.task_status = 5 then '#f44336'
    else null
  end as activity_status_color
) ac
```

## The percentage slider (progress bar), verbatim from production

```sql
when s.activity_display_type = 6 then concat(
  s.title,
  '<br /><div style="width: 100%; height: 0.5rem; background: #e0e0e0;
   border-radius:5px; margin-top: 3px;">
     <div style="width: ', s.activity_percentage, '%; height: 100%;
     background: ', s.activity_status_color, '; border-radius:5px;
     margin-top: 3px;"></div>
   </div>'
)
```

`activity_percentage` is a plain `int` column (0–100), with the `%` sign appended in the template
rather than baked into the stored value; the fill colour reuses `activity_status_color` from the same
dispatcher so the bar's colour and the activity's status colour never drift apart.

## Other verified branches

```sql
-- 1: filled-circle status indicator
when s.activity_display_type = 1 then concat('<div style="display: inline-flex; height: 16px;">
  <div style="width: 14px; height: 14px; background: ', s.activity_status_color, ';
  border-radius: 50%; margin-top:4px; margin-right: 5px;"></div>
  <span style="margin-top:3px;">', s.title, '</span></div>')

-- 5: pill badge
when s.activity_display_type = 5 then concat('<div style="display: block;">', s.title,
  '<span class="tag" style="display: inline-block; margin-left: 10px; padding: 0px 7px;
  background-color: red; color: #fff; border-radius: 20px;"><b>!</b></span></div>')

-- 9: no HTML at all
when s.activity_display_type = 9 then s.title
else s.title
end as title_html
```

**A demo shortcut worth flagging**: display types 7/8 (`activity_card`/`_compact`) have a hardcoded
`"AT RISK"` label and, in the full card, a hardcoded avatar (`"SM"` / `"Selena Martin"`) baked into the
SQL string rather than pulled from a column. Fine for a demo; in production, join to the assigned
employee and interpolate their real name/initials the same way `activity_status_color` is interpolated
— don't copy the hardcoded text along with the pattern.

## Tooltip with emoji/unicode

```sql
'<h1>Checklist</h1>&#x2705; Check oil levels<br>&#10060; Fix engine warning light<br>&#10060; Check tire pressure'
```

## Height management

HTML content has **no maximum height by default** — an activity grows as tall as its content, and a
`<br />`-heavy title can blow out the row height for an entire resource. Cap it by wrapping the fragment
in an extra `div` with `max-height`:

```sql
select '<div class="activity-wrapper" style="max-height:200px; overflow:hidden;">
<style>.activity-wrapper h2 { margin: 0; }</style>' + t1.event_html + '</div>'
```

Tooltip HTML support is comparatively new (Universal UI 2026.1.15+); before that, tooltips were plain
text with `CHAR(13)` as the only line-break trick.
