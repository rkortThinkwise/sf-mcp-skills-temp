# Common use cases (design guidance)

These patterns don't need further API verification — they're about *what* to condition on and *how* to
style it, using the entities above.

**Status visualization** (the dominant pattern — 1,467 of the definitions scanned). Suggested visual
hierarchy — treat this table as a proposal to confirm with the user (see "Before creating anything"
above), not a default to apply unasked, especially when a state's severity is genuinely ambiguous (is
this status actually exceptional enough for red, or just a normal waiting state?):

| State | Suggested treatment |
|---|---|
| Successful/completed | Muted green |
| In progress | Blue or neutral accent |
| Waiting/attention required | Amber |
| Blocked/failed/rejected | Red |
| Cancelled/inactive/obsolete | Grey, optionally strikethrough |
| Informational/new | Subtle blue |
| Unknown/exceptional | Neutral warning style, not automatically red |

Avoid a strongly saturated colour per status value — reserve strong emphasis for exceptional/actionable
states.

**Missing or incomplete data** — target the specific missing field, not the whole row, with a light
warning background; add real validation if the value is genuinely required.

**Planning and shop-floor attention** — on a busy screen, colour must answer one operational question
(what needs attention now / is blocked / is ready / is complete). Avoid a decorative multi-colour
palette; operators should read state at a glance under imperfect lighting.

**Deadlines and expiration** — when the comparison needs date arithmetic (`due_date < today`), model an
expression field (`is_overdue`) and condition on that simple result rather than duplicating the logic
across several condition rows.

**Financial exceptions** — target the specific exceptional amount (negative margin, over-threshold
invoice) rather than the whole row; a red row often overstates the problem and hides the actual value.

**Generated/system-maintained values** — a subtle neutral background for calculated/system-derived
values communicates "informational," not "erroneous" — don't reuse a warning colour just because a
value was auto-generated.
