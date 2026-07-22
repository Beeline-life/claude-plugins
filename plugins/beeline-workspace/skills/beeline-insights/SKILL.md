---
name: beeline-insights
description: "Use when the user asks about workforce performance, completion or progression rates, learner progress, per-course/program numbers, a single learner's summary, or competency gaps in their Beeline workspace — dashboards, who's falling behind, group breakdowns, how a specific beeline is doing, or role/competency gap analysis. Trigger phrases - how many groups do we have, what's our completion rate, who hasn't finished, list learners in..., how is [beeline/course] doing, give me a summary of [person], competency gaps for..., how are we doing on..., show me a dashboard, who's behind in a group."
---

# Beeline Insights

Read-only reporting and competency tools. **Reporting reads the Insights v2
fact-table warehouse** (the `rep_*` materialized views) — the same source the
in-app Insights tab uses, not the old engine. Two things it gives you: every
metric comes in both **completion** (`completion_pct`, % of assigned learners
fully complete) and **progression** (`avg_progress`, avg in-progress %), 0-100;
and **group metrics roll up descendant-group members automatically** — ask about
a region and you get the whole region, no per-store tallying.

Everything is scoped to the authorized workspace and the caller's role — a
manager only ever sees their managed groups, an admin sees the whole org. Don't
ask for an org id or a group filter unless you need to disambiguate by name.

## Reporting

- `get_group_dashboard(classification_system_type?)` — the workhorse. Org
  `summary` + `classification_counts` + a per-group list, each group carrying
  both `completion_pct` and `avg_progress`, plus learner counts, `total_overdue`,
  and `pending_pa_reviews`. This is where group `group_id`s come from, and how
  you answer "which group is falling behind" — scan for the lowest
  `completion_pct`. Optional filter `classification_system_type`: 'LOC', 'ROL',
  or 'GEN'.
- `get_group_detail(group_id, learner_sample_size?)` — one group's rolled-up
  `summary` (completion, progression, active_7d/30d, overdue, avg assessment) PLUS
  its whole subtree's members, and a sample of learner rows. `group_id` required
  (from the dashboard). Descendant members are included automatically.
- `list_learners(group_id?, include_descendants?, search?, overdue_only?, sort?, page?, page_size?)`
  — paginated learners with completion + progression. To find who's behind, pass
  `overdue_only=true`. A `group_id` includes descendant-group learners by default
  (`include_descendants`, default true). `sort` is one of 'name', 'completion',
  'progress', 'assessment', 'overdue', 'recent_active' (invalid values are rejected).
- `get_program_report(beeline_id, group_id?, include_descendants?)` — completion &
  progression for ONE beeline (program): an `overall` block for the whole scoped
  (descendant-inclusive) cohort plus a per-group `by_group` breakdown. Use for
  "how is [course] doing across our stores". Reports at the beeline level —
  per-cell reporting isn't exposed yet.
- `get_learner_summary(user_id, full?)` — a rich single-learner summary from the
  profile service: job role + KPIs, performance-review rating/cycle, learning
  completion, streak, nectar/rank, certificates, reviews due, and a coaching
  insight. Use for "give me a rundown on [person]". For their 2D competency
  picture use `get_learner_competency_snapshot` (below) — this summary doesn't
  duplicate it. `full=true` returns the complete profile payload.
- `compare_groups(group_ids)` — side-by-side comparison of 2–10 groups with
  cross-group ranks + best/worst performer per metric. The exec "which of our
  regions/stores is winning, which is dragging" view. Each group's numbers
  include its descendant members. Great to visualize (see beeline-insights-viz).
- `get_capability_gaps(group_id?, top_n?, include_insights?, full?)` — the CEO
  "where are our capability gaps?" readout: assessment performance aggregated up
  to COMPETENCIES ("the org is weak on Regulatory Compliance", not "question 123
  fails"), with severity, learners affected, a knowledge-vs-performance split, an
  AI thematic `insights` narrative, and strategic `recommendations`. Org-wide or
  scoped to a group's subtree. Needs the org's capability/competency data to be
  live; `include_insights=false` gives a faster numbers-only read.

**Completion rate is always "completed / assigned" against the SAME population** — if a number looks inconsistent with what the user expects, say so rather than silently reconciling it yourself; this codebase treats denominator mismatches as a real bug class, not a rounding quirk.

## Competency

- `get_competency_frameworks` — every competency framework defined in the org (framework/competency names and ids). Useful for orienting, but note it does NOT give you the `role_id` the gap tool needs — that's a job-role id, a different thing.
- `get_learner_competency_snapshot(user_id)` — one learner's scores across the 2D competency model (Knowledge × Performance). The parameter is `user_id` (a learner's user id, e.g. from `list_learners`). Use for "how is [person] doing" questions.
- `get_role_competency_gaps(role_id? , user_id?)` — gap analysis (required vs actual levels). Give exactly one: `role_id` (a job-role UUID) to see every learner in that role, OR `user_id` to see every role requirement for one person. This connector has no "list job roles" tool, so the practical path to a `role_id` is to call this with `user_id=<a known learner>` first — the result carries that learner's role ids, which you can then analyze role-wide. This is the tool for "who needs training on X" — don't derive it from per-learner snapshots one at a time.

## Answering well

Lead with the number, then the one or two facts that explain it — a manager asking "who's falling behind and why" wants a short diagnosis, not a raw dashboard dump. If a question needs data these tools don't have (e.g. a specific KPI/quest, an assignment's status, a performance review), say clearly that it's out of scope for this connector rather than fabricating a plausible-looking number.
