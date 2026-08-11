---
description: Quick health check of the authorized Beeline workspace — groups, completion, and any obviously at-risk group.
---

Call `get_group_dashboard` — one call returns both the org `summary` AND the per-group breakdown (`groups[]`), plus a `metric_definitions` block that states exactly what each metric means. Read it before you word the summary. Summarize for the user in this shape, using real numbers from the tool result — never fabricate a number:

1. One line: total groups, total learners, and `summary.users_fully_complete_pct` — describe it as "% of users fully complete (100% of all their assigned content)", not bare "completion".
2. The single lowest-`users_fully_complete_pct` group in `groups[]`, named, with its rate AND its `avg_user_progress_pct` — this is the "at risk" callout. If progression is healthy, say the group is mid-flight rather than failing; if the row's `assignment_scope` is `assigned_to_this_group_only`, add that the 0% is against content assigned to that grouping and its members may be complete on their own totals.
3. If the caller is a manager (their view is already scoped to managed groups), say so explicitly rather than implying this is the whole org.

Keep the whole answer under 6 lines. Offer to drill into that at-risk group with `get_group_detail(group_id=...)` or `list_learners(group_id=..., overdue_only=true)` if the user wants learner-level detail or names — do NOT call `get_group_detail` for the breakdown itself (it needs a specific group_id and only returns one group's learners).
