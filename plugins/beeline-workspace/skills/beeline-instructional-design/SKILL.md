---
name: beeline-instructional-design
description: "Use when the user wants to make training more interactive, pick the right learning format, add flashcards or flip boxes, build a Good vs Bad Manager (or Do/Don't) comparison, add a spoken voiceover/AI narration, or asks which block or assessment type to use. Trigger phrases - make this interactive, which format should I use, add flashcards, good vs bad manager, do and don't, add a voiceover, narrate this lesson, turn this into a knowledge check, how should learners practice this."
---

# Beeline Instructional Design

Pedagogy for Build tools — **when** to use each interactive format. For discover-before-edit, anchors, and Slate schema rules, use `beeline-build`. Always resolve `beeline_id` / `cell_id` first (`search_workspace_content` / `get_beeline_structure` / `get_cell_blocks`), then insert. After any insert, surface the cell’s click-out `url`.

## Pick the right format

| Learning pattern | Prefer | Do **not** use |
|---|---|---|
| Term / definition, Q&A, active recall, “test yourself” | `add_flip_boxes` | `add_comparison_block` |
| Good/Bad, Do/Don't, paired behaviors side-by-side | `add_comparison_block` | flip boxes or tabs for the same contrast |
| Spoken lesson, accessibility audio, “read this aloud” | `add_ai_narration` | a callout labelled “narration” with no TTS |
| Graded proof of knowledge (MCQ, scored check) | assessment tools (`generate` / edit assessment) | flip boxes as a fake quiz |

## Tool contracts (short)

- **`add_flip_boxes(cell_id, instruction)`** — ≥2 reveal cards. Prompt for vocabulary, misconception checks, or short Q→A pairs.
- **`add_comparison_block(cell_id, instruction, left_label?, right_label?)`** — paired rows (`behaviorPairs`). Defaults labels to Bad Manager / Good Manager. **v1 is text-only** (no avatars/image gen).
- **`add_ai_narration(cell_id, script?, instruction?, voice?, provider?)`** — drafts a script if needed, runs **metered** TTS billed to this workspace, inserts an `ai_narration` block. Returns `asset_id` + `final_audio_url` (never raw audio bytes). Warn the user that VO spends AI credits before a long script.

## Decision shortcuts

- “Make this interactive” without a format → choose from the table; if ambiguous, ask one clarifying question (recall vs contrast vs listen vs graded).
- Contrast of two approaches or manager behaviors → **comparison**, not flips.
- Many small facts to memorize → **flips**, not one giant comparison.
- Learner must *prove* they know it for reporting → **assessment**, not flips.
- User asks only for a highlight/callout → use `insert_cell_block` / callout tools; that is **not** voiceover.

## Quality bar

- One instructional job per block; don’t stack flip + comparison + VO for the same sentence.
- Keep comparison pairs concrete and observable (“says X / does Y”), not vague adjectives.
- Flip fronts should be answerable without reading the back first.
- VO scripts: flowing prose, no headings/bullets; stay faithful to the cell.
