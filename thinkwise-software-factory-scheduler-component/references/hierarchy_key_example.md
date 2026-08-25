# Technique: prefixed synthetic keys for a hierarchy spanning multiple tables — full example

Verified, real technique from `PROJECT_MANAGER`'s subject — a **department → team → employee**
resource hierarchy built from three physically different tables, unioned into one view, with the
resource key and parent key built like this:

```sql
-- department: top of the tree (no parent)
select  concat('d_', d.department_id) as resource_scheduler_id
       ,concat('d_', d.department_id) as resource_id
       ,d.department_name             as resource_name
       ,null                          as parent_resource
from    department d

union all

-- team: child of department
select  concat('t_', t.team_id)       as resource_scheduler_id
       ,concat('t_', t.team_id)       as resource_id
       ,t.team_name                   as resource_name
       ,concat('d_', d.department_id) as parent_resource   -- points at the row above
from    department d
        join team t on t.department_id = d.department_id

union all

-- employee: child of team
select  concat('e_', te.employee_id)  as resource_scheduler_id
       ,concat('e_', te.employee_id)  as resource_id
       ,e.full_name                   as resource_name
       ,concat('t_', t.team_id)       as parent_resource   -- points at the row above
from    department d
        join team t  on t.department_id = d.department_id
        join team_employee te on te.department_id = t.department_id and te.team_id = t.team_id
        join employee e on e.employee_id = te.employee_id

union all

-- project_task: the activities, resource_id reuses the assigned employee's own prefixed key
select  concat('pt_', pt.project_task_id) as resource_scheduler_id
       ,concat('e_', te.employee_id)      as resource_id
       ,e.full_name                       as resource_name
       ,concat('t_', t.team_id)           as parent_resource
       ,pt.title, pt.description, pt.start_date, pt.end_date
from    department d
        join team t on t.department_id = d.department_id
        join team_employee te on te.department_id = t.department_id and te.team_id = t.team_id
        join employee e on e.employee_id = te.employee_id
        join project_task pt on pt.assigned_to_employee_id = e.employee_id
```

Every branch prefixes its native ID (`d_`, `t_`, `e_`, `pt_`) before it reaches
`resource_scheduler_id` — this is what lets one non-nullable `VARCHAR` primary key column serve four
different source tables without collision (`d_7` and `e_7` are obviously not the same row).
`parent_resource` reuses the exact same prefixed format as the level above, which is all a hierarchy
grouping needs (see "Resource grouping" in SKILL.md). Reach for this whenever a resource hierarchy spans
genuinely different tables rather than one self-referencing one.

**Bonus — this can eliminate the need for an instead-of trigger on resource-dragging.** Because
`resource_id` is the *same prefixed string* on the resource's own row and on every activity assigned to
it, dragging an activity onto a different resource just copies that string across verbatim — no FK
lookup translation required. Building the grouping key as a plain, format-matched string rather than a
raw numeric FK is a legitimate way to avoid the instead-of trigger described in SKILL.md entirely.
