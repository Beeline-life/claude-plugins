---
description: Guided job-role + KPI architect — inspect workspace roles, then draft a Role Pack proposal with explicit evaluation modes.
---

Help the user design or refine Beeline job roles, KPIs, and competencies.

1. Call `list_job_roles` and summarize what already exists.
2. Ask whether they want to invent from a JD/notes or refine existing roles.
3. Follow the `beeline-role-architect` skill: every KPI needs `evaluation_mode` (`metric_only` | `scale_only` | `both`).
4. When ready, `create_role_pack_proposal` with pasted text (and `file_keys` if they have uploaded JDs), then poll `get_proposal`.
5. Patch until excellent; **confirm only on explicit approval**.
