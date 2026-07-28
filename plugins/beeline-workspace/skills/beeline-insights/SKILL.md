---
name: beeline-insights
description: "Use when the user asks about workforce performance, completion or progression rates, learner progress, per-course/program numbers, a single learner's summary, practical-assessment pass rates, learner feedback and sentiment, headcount movement or dormancy, at-risk sites, segmentation by tag (area manager / store format / region), or competency gaps in their Beeline workspace. Trigger phrases - how many groups do we have, what's our completion rate, who hasn't finished, list learners in..., how is [beeline/course] doing, give me a summary of [person], competency gaps for..., show me a dashboard, who's behind, which stores are most at risk, break it down by [FSM/region/store type], how are our practical assessments going, what are people saying in feedback, how many new starters this month, who's gone quiet."
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

- `get_group_dashboard(classification_system_type?, group_by_tag_category?)` —
  the workhorse. Org `summary` + `classification_counts` + a per-group list,
  each group carrying both `completion_pct` and `avg_progress`, plus learner
  counts, `total_overdue`, and `pending_pa_reviews`. This is where group
  `group_id`s come from. Optional filter `classification_system_type`: 'LOC',
  'ROL', or 'GEN'. Pass `group_by_tag_category` (a slug from `list_group_tags`)
  to ALSO get a `by_tag` rollup — that's how you answer "break completion down
  by area manager / store format / region" when the cut is a tag rather than a
  group classification. Each bucket is a real warehouse aggregate over that
  tag's distinct learners, not a sum of group rows, so the numbers are safe to
  quote. Groups with no tag in the category appear in an "Untagged" bucket with
  `completion_pct: null` — say so rather than implying full coverage.
- `get_group_detail(group_id, learner_sample_size?)` — one group's rolled-up
  `summary` (completion, progression, active_7d/30d, overdue, avg assessment) PLUS
  its whole subtree's members, and a sample of learner rows. `group_id` required
  (from the dashboard). Descendant members are included automatically.
- `list_learners(...)` — the segmentation workhorse, and the tool for
  people-and-place questions ("how many baristas per store", "who's dormant in
  the Kiosk sites", "everyone scoring under 50"). Paginated learners with
  completion + progression, plus `kpis` computed over the SAME filtered
  population, so the headline and the rows never disagree. Filters:
  `group_id` (+ `include_descendants`, default true), `search`, `overdue_only`,
  `sort` ('name', 'completion', 'progress', 'assessment', 'overdue',
  'recent_active'), `tags` (tag NAMES — see `list_group_tags`), `roles`,
  `activity_window` ('7d'/'30d'/'90d' active, or 'inactive' = never active),
  `progress_min`/`progress_max`, `assessment_min`/`assessment_max`, `pa_status`
  ('pending'/'failed'/'complete'), `exclude_managers`, and `membership_status`
  ('active' default — only use 'all'/'revoked' for historical or audit
  questions). Every invalid value is rejected rather than ignored, so a filter
  either applies or errors — it never silently returns the whole org.
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
- `get_at_risk(mode?, top_n?)` — "who needs attention today", worst first, each
  row with a one-line `why`. `mode='groups'` (default) ranks stores/regions by
  completion ascending; `mode='learners'` ranks individuals. Empty groups are
  excluded (they read as 0% while representing nobody) and reported in
  `excluded_empty_groups`. Prefer this over eyeballing the dashboard.
- `list_group_tags()` — the segmentation vocabulary: every tag category and its
  values. **Call this first whenever the user asks for a breakdown "by
  something" that isn't a group classification** (area manager, FSM, store
  format, region, brand). It tells you whether that cut exists here and what
  it's called, instead of guessing a name that will be rejected. Feed `key` to
  `get_group_dashboard(group_by_tag_category=...)` and `name` to
  `list_learners(tags=[...])`.
- `get_practical_summary(learner_id?)` — practical (observed, hands-on)
  assessments. No `learner_id` → org-wide pass rate, failures,
  awaiting-review backlog, average score — **admin only**, because the
  warehouse aggregate has no group filter to clamp a manager to. With
  `learner_id` → that person's full attempt history including redo chains
  (admin, or their manager). Practical scores are **0-1**; theory assessment
  scores are 0-100 — never average the two together.
- `get_feedback(group_id?, rating?, sentiment?, category?, beeline_id?, page?, page_size?)`
  — what learners actually said about the content: star ratings AND free-text
  comments, enriched with AI sentiment and category analysis. The `analysis`
  block is computed over the whole filtered set, not just the page, so quote it
  directly. Use for "what are people saying about our training".
- `get_content_report(group_id?, search?, content_type?)` — per-course
  completion across the library: assigned, completed, completion %, average
  progress, and **`completed_last_30d`** per beeline. This is the reporting view
  of the catalogue — the Build tool `list_beelines` is metadata only (no
  numbers), and `get_program_report` drills ONE course down by group. Use
  `completed_last_30d` to separate courses actively landing from dead stock: a
  big cumulative `completed` with zero recent completions is not a success.
- `get_completion_trends(bucket?, days?, group_id?, beeline_id?)` — completions
  over time, the momentum question ("are we speeding up or slowing down?").
  `bucket` is 'day', 'week' (default) or 'month'; `days` is the window (default
  90). Only periods with at least one completion appear — treat a missing period
  as zero when charting, don't read it as missing data. Distinct from activity,
  which counts logins and views rather than finished courses.
- `get_activity_report(group_id?)` — headcount movement and dormancy:
  `new_users` (joined in the last 7/30/90 days) and `inactive` (last active over
  30/60/90 days ago, **including people who have never been active**). Both are
  cumulative thresholds, not disjoint buckets — someone inactive 90 days is also
  in the 30d and 60d counts. This answers "how many new starters this month" and
  "who's gone quiet", which no completion percentage can.
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

## The recurring management report

The most common real use of these tools is a repeating three-part report. Run it
as three calls and answer in that order:

1. **Where are we overall?** → `get_group_dashboard` (org `summary`).
2. **Who's at risk?** → `get_at_risk` (named sites, worst first, with reasons).
3. **How is each area/segment doing?** → `get_group_dashboard(group_by_tag_category=...)`
   after resolving the category with `list_group_tags` — e.g. per area manager.

## Answering well

Lead with the number, then the one or two facts that explain it — a manager asking "who's falling behind and why" wants a short diagnosis, not a raw dashboard dump. If a question needs data these tools don't have (e.g. a specific KPI/quest, an assignment's status, a performance review), say clearly that it's out of scope for this connector rather than fabricating a plausible-looking number.

Two scope edges worth naming out loud rather than approximating:
- **This connector covers training data only.** Sales, stock, wastage, customer
  complaints and other operational data are not here, however reasonable the
  question. Say so plainly instead of reaching for a loosely-related number.
- **Overdue reporting is partial.** Per-learner due dates for relative
  assignments aren't fully in the warehouse yet, so treat overdue counts as a
  floor, not an exact figure.
