---
name: beeline-build
description: "Use when the user wants to create, edit, restructure, or review a Beeline course (a beeline) — building a new beeline from scratch, adding or rewriting a section, cell, or lesson, editing an assessment or survey, cloning an existing beeline as a template, or keeping a beeline in sync with a source document (SOP, policy, script) that changed. Trigger phrases - build a beeline, create a course for..., add a section on..., rewrite the intro of..., clone this beeline, duplicate this course, what's in this beeline, the SOP changed update the beeline, add a knowledge check."
---

# Beeline Build

Beelines are courses made of ordered sections, each holding ordered content cells (rich text, assessments, surveys, digital resources, linked beelines). All build tools are org-scoped to the workspace you authorized — never pass an org id yourself.

## Discover before you edit

Never guess a beeline_id or cell_id. Always resolve them first:

1. Use `search_workspace_content` when the user knows a cell, phrase, source, file, or asset but not its Beeline/id. It searches Beelines, cell names and body text, ContentSources, and workspace Assets; narrow with `entity_types` when useful. Use `list_beelines` when the request is specifically to browse/filter Beeline titles and statuses. Surface returned click-out URLs.
2. `get_beeline_metadata(beeline_id)` for status, structure type, language, and delivery method — use this alongside structure when you need full context before editing, e.g. to check a beeline isn't already PUBLISHED before making a destructive change.
3. `get_beeline_structure(beeline_id)` to see the section/cell tree, each entry with `cell_id`, `name`, `type`, `slug`, and a click-out `url`. This also returns the `graph_version` — pass it back as `expected_version` on structure edits so a conflicting concurrent edit is rejected instead of silently overwritten.
4. `get_cell_blocks(cell_id)` to read a cell's actual Slate content blocks before editing them — the returned block `id`s are how you address them in edit calls.

## Starting from nothing vs. starting from a template

- **From nothing:** `create_beeline(name, description)` makes an empty beeline with one empty section. Immediately follow with `add_section`/`add_cell` to fill it in — don't stop after create_beeline and call it done.
- **From an existing beeline:** `duplicate_beeline(beeline_id)` clones the whole thing (all sections, cells, questions, survey fields) into a new independent copy in the same content library. Use this when the user says "make one like X for Y" instead of building from zero. Note: it only clones content — it does not reassign the new copy to a different group; that's a separate step outside this tool's scope.

## Editing content — pick the right granularity

- **Whole-cell rewrite:** `set_blocks(cell_id, new_blocks)` replaces everything in a cell in one write. Use this after generating a full new draft of a cell's content, not after a small fix.
- **Surgical edit:** `replace_cell_block` / `insert_cell_block` / `delete_cell_block` / `move_cell_block` / `update_cell_block_text` for changing one block or reordering a few — use these for "fix this one paragraph" or "add a callout after the second block" requests, not for a full rewrite.
- **AI-assisted:** `generate_block(cell_id, instruction)` and `rewrite_block(cell_id, block_id, instruction)` ask the platform's own AI to draft or rewrite a block — prefer these over hand-writing Slate JSON yourself when the user wants new prose, not a mechanical edit. Both accept `source_reference_ids` to ground generation in an attached content source.

Every content-edit call takes an optional `expected_version` (the cell's `content_version` from your last read) — pass it when you want a stale-edit conflict to raise an error instead of silently clobbering someone else's concurrent change.

## Structure edits

`add_section`, `add_cell`, `rename_cell`, `move_cell`, `reorder_sections`, `reorder_cells_in_section`, `remove_cell`, `remove_section` — all take `beeline_id` and an optional `expected_version` (the beeline's `graph_version`). `rename_cell` renames either a section or any content-cell type in place, preserving its id, slug, children, content, linked-object reference, assets, and learner progress. By default it also synchronizes the title of a linked assessment, survey, practical assessment, performance review, digital resource, or SCORM course; pass `rename_linked_object=false` only when that title should intentionally remain independent. It never renames a linked beeline (the cell is only its link label), and feedback forms have no title field. Remember that intentionally shared cells or linked objects will show the new name everywhere. `add_cell`'s `cell_type` is one of SLATE (rich text), DIRES (digital resource), BEELI (linked beeline), ASSES (assessment), SCORM, FDBK (feedback), PERF (performance review), PRACT (practical assessment), SURV (survey) — FRAME (a section) is never a valid `cell_type` here, use `add_section` for that.

For a DIRES cell from an already-uploaded file, `add_digital_resource_cell(beeline_id, section_id, file_key, name?)` does it in one call — it creates the learner-facing resource AND mints the file's content-source fuel (embeddings + competency-ready) at the same time, returning both the new `cell_id` and a `content_source_id` you can feed into `generate_block` to write the surrounding beeline grounded in that file.

## Assessments and surveys

Once a cell of type ASSES exists, manage its questions with `list_questions`/`add_question`/`update_question`/`delete_question`/`reorder_questions`. The valid `question_type` codes are: `MUCQ` (multiple choice), `SHAN` (short answer), `DDMQ` (drag & drop match), `DTWQ` (drag the word), `ORDQ` (ordering), `TEFE` (true/false), `ACKN` (acknowledgement). Multiple choice is `MUCQ` — not "MCQU" or "MCSA", both of which the validator rejects. Same shape for SURV cells via `list_survey_fields`/`add_survey_field`/`update_survey_field`/`delete_survey_field`/`reorder_survey_fields`.

## Keeping a beeline in sync with its source document

If a beeline was built from a URL or uploaded file and that source changed, the flow has a strict order — each step's output feeds the next:

1. `ingest_content_source` is the general source front door: provide exactly one of pasted `text`, a safe public `url`, an existing workspace `file_key`, or an MCP attachment `file_reference`. It returns the `content_source_id`, extraction status, and source Asset ids using the real org-scoped extraction/embedding pipeline. Prefer it over `attach_source`; the latter remains a smaller compatibility shortcut for an already-uploaded `file_key`.
2. `refresh_content_source(content_source_id, url)` or `refresh_content_source_from_upload(content_source_id, file_key)` re-fetches the SAME source's origin and detects drift. If nothing changed it returns `drifted: false` with `new_content_source_id: null` — always safe to call speculatively, never assume drift without checking. If it *has* changed, it mints a **new** `ContentSource` (`new_content_source_id`) that supersedes the old one.
3. `find_content_using_source(old_content_source_id)` shows every beeline/cell grounded in the stale source, filtered to what the caller can actually access — use this before touching anything, especially if the source is shared org-wide.
4. `resync_beeline_from_source(beeline_id, old_content_source_id, new_content_source_id, confirm=False)` — preview-then-confirm, like the group tools: without `confirm` it returns exactly which cells would be rewritten; with `confirm=True` it rewrites all of them in one transaction (all-or-nothing — a failure rolls back everything). `new_content_source_id` MUST be the id `refresh_content_source*` just returned for this exact `old_content_source_id` — it's rejected (`content_source_not_successor`) if it isn't a real successor of it.

Never assume drift, and never skip the preview step of `resync_beeline_from_source` — read the result rather than telling the user "it's updated" without confirming.

## Surface the click-out link

Some tools return a `url` into the real webapp builder — `search_workspace_content` returns URLs for Beeline/cell matches (never private Asset URLs); the other discovery/create tools (`list_beelines`, `get_beeline_metadata`, `create_beeline`, `duplicate_beeline`) return a beeline `url`; and structure tools (`get_beeline_structure`, `add_section`, `add_cell`, `rename_cell`, `move_cell`, `reorder_*`, `remove_*`) return a top-level `beeline_url` plus a per-cell/section `url` inside `structure`. Content, question, survey, ingestion, and source-sync tools do NOT return a private URL. Cache the Beeline URL from a discovery/structure call and surface it after edits, but never invent or guess a URL from a tool that did not return one.
