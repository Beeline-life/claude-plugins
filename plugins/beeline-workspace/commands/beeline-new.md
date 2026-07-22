---
description: Start a brand-new beeline from a topic/description, one section at a time, confirming with the user before each step.
argument-hint: [topic or one-line description of the beeline]
---

The user wants to start a new beeline about: $ARGUMENTS

1. If $ARGUMENTS is empty or too vague to name a beeline from, ask one clarifying question (topic, audience, or roughly how many sections) before creating anything — don't guess a full course structure from nothing.
2. Call `create_beeline(name=..., description=...)` using a real title derived from $ARGUMENTS, not a placeholder like "New Beeline". Report the returned `url` immediately so the user can watch it take shape. If this call (or any later one) returns an error, STOP the flow, show the error verbatim, and suggest reconnecting the Beeline Workspace connector with an admin or manager account — do not silently retry.
3. Note that `create_beeline` already leaves ONE empty default section in the new beeline. Read `get_beeline_structure` to get that section's `cell_id` — you'll reuse it for the first outline section rather than adding a duplicate.
4. Propose a short section outline (3-5 sections) based on $ARGUMENTS and ask the user to confirm or adjust it before adding any of them — do not add every section unprompted.
5. Once confirmed: rename the existing empty default section to the first outline section with `rename_cell`, then add any further sections one at a time with `add_section`, giving each a short status update, not a silent batch.
6. Stop after structure is in place and ask whether the user wants content drafted now (via `generate_block` per cell) or wants to add cells/content themselves first.
