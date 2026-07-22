---
name: beeline-groups
description: "Use when the user wants to change their org's group or hierarchy structure — creating a new store, branch, or team, renaming or reclassifying a group, moving a group under a different parent, moving people between groups, merging or closing a location, or deleting a group. Trigger phrases - add a new store, we opened a branch in..., move everyone from X to Y, reclassify these groups, the old store is closing, delete this group, rename this group, reorganize our regions."
---

# Beeline Groups

Groups model an org's structure — locations, regions, teams, roles — as a tree, max 3 levels deep. Every structural tool here is preview-then-confirm: call once without `confirm` to see exactly what would happen (zero writes), then call again with `confirm=True` to actually do it. Never skip the preview step, and always show the preview to the user before confirming unless they've explicitly said to just do it.

## Auth model — know who can do what

- **Structure tools** (`create_group`, `bulk_create_groups`, `edit_group`, `bulk_edit_groups`, `move_group`, `delete_group`) are **admin-only**. A manager calling these gets a clear rejection — don't retry, tell the user they need an admin.
- **`move_group_members`** and **`get_group_move_status`** are admin OR a manager scoped to BOTH the source and destination group. If a manager isn't cleared for one side, the call fails — resolve which groups the user actually manages before assuming a move will work.

## Creating and editing groups

- `create_group(name, classification, parent_group_id?)` — `classification` is a NAME ("Location", "Role", or a custom one already in the workspace), not an id. An unknown classification name fails with the list of what's actually available — read it back to the user rather than guessing a spelling.
- `bulk_create_groups(groups: [...])` for several at once ("add 5 new stores under North region") — invalid rows show up as warnings without failing the whole batch, so check `warnings` even on a successful-looking preview.
- `edit_group` changes name/classification/active/location/dates — it deliberately does NOT move a group to a new parent. For that, use `move_group`.
- `bulk_edit_groups(group_ids, classification?, add_tags?, remove_tags?)` applies the same change to many groups at once. Tags are given by NAME via `add_tags`/`remove_tags` (there is no single `tags` parameter); an unknown or ambiguous tag name is rejected. All `group_ids` must be in the workspace or the whole call is rejected — it never silently applies to a valid subset.

## Moving and deleting groups

- `move_group(group_id, new_parent_group_id)` moves a group **and everything under it** — the preview shows `descendant_count` so "move this branch" doesn't surprise anyone about what else comes with it. Rejected if it would push any descendant past depth 3, or if it's a self-parent/cycle.
- `delete_group(group_id)` is a **permanent hard delete** — sub-groups are promoted to root, never orphaned, but the group itself is gone. If the user actually wants a reversible hide, use `edit_group(active=False)` instead — don't reach for delete by default.

## Moving people between groups

`move_group_members(from_group_id, to_group_id, user_ids? | all_members?)` — give exactly one of an explicit user list or `all_members=True`, never both, never neither. Preview always shows the real affected count and a name sample regardless of size.

Moves of **25 people or fewer** execute synchronously on confirm and return `moved_count` immediately. Moves of **more than 25** enqueue a background job instead and return `{job_id, status: "processing"}` — poll `get_group_move_status(job_id)` until it reports `completed` (with the final `moved_count`) or `failed`. Don't tell the user a large move is done until you've polled and confirmed completion.

## Resolving group ids

This connector has NO list-groups or search-groups tool, and nothing returns the parent/child tree. The one place group ids come from is `get_group_dashboard` (the reporting/insights domain): its `by_classification[].groups[]` entries carry each group's `id` and `name`. Resolve a named group ("North region") to its id there first. Note the dashboard rows do NOT expose `parent_group_id` — for the effect of a reparent on descendants, rely on `move_group`'s preview `descendant_count` rather than trying to read the tree upfront.

## Common flows

- *"New store opened in Sheffield under North region"* → resolve North region's id from `get_group_dashboard`, `create_group(confirm=False)` to preview, show it, then `confirm=True`.
- *"Move everyone from the old Leeds store to the new one, old one's closing"* → resolve both ids from `get_group_dashboard`, `move_group_members(all_members=True, confirm=False)`, show the count, confirm — if over 25, poll `get_group_move_status` before reporting done.
- *"Merge North and Midlands into one region"* — this is NOT directly supported by these tools (they only move/edit named, existing groups) — say so rather than improvising a multi-step workaround that could leave the tree in a half-merged state.
