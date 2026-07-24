# Beeline Workspace — Claude Plugin

Connects Claude (Cowork, Claude Code, claude.ai) to a Beeline workspace via the
existing OAuth-authenticated `workspace-mcp` server (`clients/workspace-mcp/`).
No JWT-pasting, no per-user setup beyond a normal login — install once, pick
your workspace, done.

Customer-facing setup guide: [Connect an AI assistant to your Beeline workspace](https://www.beeline.life/external-api-docs/workspace-mcp). It includes the OAuth walkthrough, Claude Code command, safe-use guidance, and the [Knowledge Hub guide](https://app.notion.com/p/beelinelearn/Connecting-AI-Assistants-to-Your-Beeline-Workspace-MCP-39f70254f14c8125a19afdc94987e2da).

## What's in the box

- **`.mcp.json`** — points Claude at the hosted MCP server. Claude's MCP
  client auto-negotiates the OAuth 2.1 flow (RFC 9728/8414 discovery, DCR,
  PKCE) the first time you connect: a browser popup opens, you log in with
  your Beeline admin/manager account, pick your workspace, and you're
  authorized. This is the same flow already validated manually per
  `docs/workspace-mcp-plugin/qa-setup.md`.
- **Five skills** (`skills/beeline-build`, `skills/beeline-groups`,
  `skills/beeline-insights`, `skills/beeline-insights-viz`,
  `skills/beeline-role-architect`) — not just a tool list. Each teaches the
  workflow conventions the raw tool schemas don't carry on their own:
  discover-before-edit, preview-then-confirm, forced KPI `evaluation_mode`,
  which tools are admin-only, and to always surface click-out URLs.
  `beeline-role-architect` drives Role Pack invent → patch → confirm.
- **Three commands** — `/beeline-workspace:beeline-status`,
  `/beeline-workspace:beeline-new`, `/beeline-workspace:beeline-roles`.

## Tool surface (69 tools across 8 domains)

| Domain | Count | Covers |
|---|---|---|
| Build | 41 | Discover/create/clone/edit/**publish** beelines, cells, blocks, assessments, surveys, assets, and content sources |
| Groups | 8 | Org hierarchy: create/edit/move/delete groups, move people, sync/async |
| Reporting (Insights v2) | 7 | Group dashboard, group detail, learner list, per-program completion+progression, learner summary, cross-group **compare**, org **capability-gap** readout — all on the fact-table warehouse + gap engine, descendant rollup included |
| Competency | 3 | Frameworks, learner snapshots, role gap analysis |
| Roles | 2 | `list_job_roles` / `get_job_role` — dual-mode KPIs (metric/scale/both), competencies, standards |
| Gap diagnosis | 1 | `diagnose_gaps` — fused five-signal WHY (role requirements, exact failed questions, decay, capability targets, manager KPI ratings) per learner / group subtree / org, with per-signal status honesty |
| Workspace | 2 | List connectable workspaces; link-completed switching (signed browser link → confirm → re-auth with target pre-selected; docs/mcp-workspace-switching/spec.md) |
| Proposals | 5 | Role Pack create/get/patch/confirm/reject (admin-only; poll `get_proposal` while generating) |

> Counts are asserted in `core/modules/mcp_oauth/tests/test_workspace_mcp_surface.py`
> (`EXPECTED_TOOL_COUNT`) — if you add/remove a tool, that test fails until both
> the count there and this table are updated together.

## Install

**From a Cowork session:** zip this directory and upload it via Plugins →
Upload. Run from inside `plugin/` (not its parent) — `-x "*.DS_Store"` keeps
macOS junk out without ever excluding a leading-dot file the plugin actually
needs (`.mcp.json`, `.claude-plugin/plugin.json`):

```bash
cd clients/workspace-mcp/plugin
zip -r beeline-workspace.plugin . -x "*.DS_Store"
```

Or, if this repo is added as a private marketplace, install
`beeline-workspace@<marketplace-name>`.

**Locally, for iteration:** `claude --plugin-dir clients/workspace-mcp/plugin`
loads it live; `claude plugin validate clients/workspace-mcp/plugin` checks
structure before you package it.

**From Claude Code (CLI), without the plugin wrapper:**
```bash
claude mcp add --transport http beeline-workspace https://mcp.beeline.life/mcp
```
Claude Code opens a browser for the OAuth flow automatically.

**From Claude.ai:** Settings → Integrations → Add custom integration → paste
`https://mcp.beeline.life/mcp`.

## First steps after connecting

1. Smoke test: run `/beeline-workspace:beeline-status` (or just ask *"what's our
   completion rate?"*). If it returns real numbers, the connection and OAuth
   scoping are working.
2. Try one prompt per domain:
   - Reporting — *"Which of our regions is falling behind on completion, and by
     how much?"* (region metrics roll up every store under them automatically).
   - Reporting (per-program) — *"How is the Food Safety beeline doing across our
     stores — completion and progression?"*
   - Reporting (per-learner) — *"Give me a rundown on [learner name]."*
   - Build (discover) — *"List our beelines"*, then *"Show me the structure of
     &lt;name&gt;"* and open the returned link.
   - Groups (preview-then-confirm) — *"We opened a new store in Sheffield under
     North region"* — the agent previews before it writes; you confirm.
   - Build (create) — *"Start a beeline on till-point upselling"* (or
     `/beeline-workspace:beeline-new till-point upselling`).
   - Workspace — *"Which Beeline workspaces can I connect to?"* (multi-org users).

## Current distribution status

- **Claude production connector:** `https://mcp.beeline.life/mcp`. The Claude
  plugin format remains Claude-specific and keeps the full 62-tool workspace
  surface.
- **ChatGPT app:** `https://mcp.beeline.life/chatgpt/mcp`. This is a separate
  registration profile behind the same production domain: anonymous Build
  preview/claim tools plus OAuth-scoped, read-only Insights. It never advertises
  Claude's group mutations or low-level workspace Build editor tools.
- **Admin/manager accounts only.** Learner and regular-user accounts see an
  empty workspace picker with no explanation at the OAuth step — a known UX
  gap, not a bug in this plugin.
- OAuth authorisation and tool calls emit metadata-only adoption telemetry
  (assistant, profile, workspace role, tool name, duration and failure type).
  Prompts, tool arguments, source contents and tool results are excluded.

## Versioning

Bump `version` in `.claude-plugin/plugin.json` to publish an update through
whatever distribution channel is used (marketplace repo version bump, or
re-upload the zip). See the parent `docs/mcp-connector-roadmap/spec.md` and
`docs/workspace-mcp-plugin/investigation.md` for the fuller roadmap this
plugin sits inside of.
