---
name: beeline-insights-viz
description: "Use when someone wants to SEE Beeline insights, not just read numbers — a chart, graph, dashboard, or visual comparison of group completion/progression, how a beeline/program is doing across groups, or a learner scorecard. Render it as a self-contained HTML artifact. Trigger phrases - show me a chart, graph this, visualize, plot, make a dashboard, compare our stores visually, chart completion by region, turn this into a scorecard, show it as a picture."
---

# Beeline Insights — Visualize

When the user wants to *see* insights (chart / graph / dashboard / "compare
visually" / scorecard), pull the data with the `beeline-insights` reporting
tools and render ONE self-contained HTML artifact. The tools already return
chart-ready JSON — your job is to shape a clean, accurate visual from it.

## Hard rules (artifacts are sandboxed)

- **Self-contained only.** Inline all CSS and JS. Use inline SVG or a plain
  `<canvas>` you draw with vanilla JS — NO external scripts, CDNs, chart
  libraries, web fonts, or remote images. The artifact sandbox blocks every
  outbound request; anything external silently fails to render.
- **Responsive.** `max-width: 100%`, no horizontal page scroll; if a chart is
  wide (many groups), put it in a scrollable container, don't overflow the page.
- **Chart only real returned numbers.** If a metric is `null`/absent for a row,
  omit that bar/row — never invent or interpolate. If you truncate (e.g. top/
  bottom 15 of 60 groups), say so in a caption.
- **Don't re-fetch** data you already have in the conversation. One tool call
  per view unless the user changes scope.

## What to render for each source

**Group comparison — `get_group_dashboard` (or `get_group_detail` for one group's subtree).**
Each group has `completion_pct` AND `avg_progress`. Draw a horizontal
**paired bar** chart (two bars per group: completion vs progression), **sorted
ascending by `completion_pct`** so the groups falling behind are at the top.
Label each bar with its value; visually highlight the lowest. Title it with the
org and the metric. This directly answers "which of our regions is behind" — and
remember dashboard/detail numbers already include descendant-group members.

**Program (beeline) report — `get_program_report`.**
Top: a row of stat tiles from `overall` (completion %, progression %, learners
assigned, completed). Below: a bar chart of `by_group[].completion_pct` (add a
second series for progression only if the rows carry it). Caption with the
beeline name and whether the scope was org-wide or a specific group's subtree.

**Learner scorecard — `get_learner_summary`.**
A compact card, not a big chart: stat tiles (completion %, avg performance
rating, streak, org rank, certificates), a small bar for the job-role KPI count
or `section_counts`, and the `coaching_insight` as a highlighted callout at the
bottom. Header = learner name, role, org.

**Cohort comparison — `compare_groups`.**
A grouped/ranked bar chart across the compared groups (completion %, with a
second series for another metric like active-30d% if useful), sorted so the
leader and laggard read instantly; annotate the best/worst performer the tool
returns. Small "rank / N" chips per group work well.

**Capability gaps — `get_capability_gaps`.**
Lead with the `insights` narrative as a short highlighted summary (it's the
"what and why"), then a horizontal bar chart of the top competency gaps by
`average_failure_rate` (label learners-affected and severity; flag CRITICAL in
the warning colour), and list the `recommendations` as next-step bullets. This
is the CEO one-screen "where are we weak and what do we do" readout.

## Style

Clean and legible over decorative. A readable neutral palette that works on
light and dark backgrounds; use one accent colour for completion and a second,
clearly distinct one for progression, and reserve a warning colour (amber/red)
only to flag the worst performer or overdue. Every chart gets a title, a legend
when there are two series, axis/value labels, and enough contrast to be
accessible. Keep it to roughly one screen — summarize, don't dump 60 rows.

## Efficiency

Prefer a single artifact that answers the question over several. If the user
iterates ("now sort by progression", "just the North region"), edit the same
artifact rather than starting over. Compute derived numbers (gaps, averages) in
the JS from the returned data rather than asking for more tool calls.
