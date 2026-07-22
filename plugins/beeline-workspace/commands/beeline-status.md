---
description: Quick health check of the authorized Beeline workspace — groups, completion, and any obviously at-risk group.
---

Call `get_group_dashboard` — one call returns both the org summary AND the per-group breakdown (`by_classification[].groups[]`). Summarize for the user in this shape, using real numbers from the tool result — never fabricate a number:

1. One line: total groups, total learners, overall completion rate (from the dashboard `summary`).
2. The single lowest-`completion_percentage` group across `by_classification[].groups[]`, named, with its rate — this is the "at risk" callout.
3. If the caller is a manager (their view is already scoped to managed groups), say so explicitly rather than implying this is the whole org.

Keep the whole answer under 6 lines. Offer to drill into that at-risk group with `get_group_detail(group_id=...)` or `list_learners(group_id=..., overdue_only=true)` if the user wants learner-level detail or names — do NOT call `get_group_detail` for the breakdown itself (it needs a specific group_id and only returns one group's learners).
