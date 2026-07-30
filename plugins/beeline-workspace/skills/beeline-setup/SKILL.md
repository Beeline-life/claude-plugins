---
name: beeline-setup
description: "Use when the Beeline connection isn't working or the user is connecting for the first time — the workspace picker was empty at login, tools return an authentication or permission error, no Beeline tools appear at all, the user is signed into the wrong workspace, or they ask how to connect / reconnect / switch workspace. Trigger phrases - it says I have no workspaces, the picker was empty, I can't connect, authentication failed, it's asking me to log in again, no Beeline tools, which workspace am I in, I'm in the wrong workspace, how do I connect, how do I switch workspace, setup, reconnect."
---

# Beeline Setup & Troubleshooting

This plugin talks to a Beeline workspace over a hosted MCP server
(`https://mcp.beeline.life/mcp`) using a normal browser login. There are no API
keys, tokens, or config files for the user to fill in — if something is wrong,
it is almost always one of the five cases below.

Work through them in order. Diagnose before instructing: a wrong guess here
sends someone to their IT team for a problem that was actually their account
role.

## 1. Confirm whether the connection works at all

Call `list_workspaces`. It is the cheapest read on the surface and needs only a
valid login.

- **Returns one or more workspaces** → the connection and OAuth are fine. Any
  failure the user is describing is about *scope* (cases 3–4), not connectivity.
- **Returns an empty list** → go to case 2. This is the single most common
  report and it is almost never a bug.
- **Errors with an auth challenge / login link** → the session isn't authorized
  yet or the token expired. Surface the link and ask them to complete the
  browser login, then retry. This is normal on first use and after a long gap.

## 2. Empty workspace picker → the account is not an admin or manager

**This is the known trap.** Beeline's MCP surface is admin/manager-only. A
learner or plain user account completes the browser login successfully and then
sees an empty workspace picker **with no explanation** — it looks broken and
isn't.

Say so plainly, and don't send them round the login loop again:

> Your login worked, but this integration is only available to Beeline admin and
> manager accounts. A learner account will always see an empty workspace list
> here. Ask whoever administers your Beeline workspace to give your account a
> manager or admin role, then reconnect.

Do **not** suggest reinstalling the plugin, clearing caches, or re-authorizing —
none of those change the account's role, and each one costs the user time and
credibility.

## 3. Manager accounts see less than they expect

A manager's reads and writes are scoped to the groups they manage. If numbers
look "too small" or a group they named is missing, the likely cause is that the
group sits outside their managed subtree — not that the data is wrong.

Confirm before speculating: `list_workspaces` shows their role, and
`get_group_dashboard` shows what they can actually see. If they need wider
visibility, that is a role change in Beeline, not a plugin setting.

## 4. Wrong workspace, or a multi-org user

`list_workspaces` shows every workspace the account can connect to and which
one is currently active. To move: `switch_workspace` returns a signed browser
link that re-runs authorization with the target workspace pre-selected. It is a
read — it changes nothing on its own, and the user must complete the link.

Always state which workspace you are operating in before doing anything that
writes. Acting on the wrong org is worse than doing nothing.

## 5. No Beeline tools present at all

If none of the tools exist in the session, the plugin isn't loaded — a
different problem from a failed login.

- Claude Code: `/reload-plugins`, then `/plugin` → **Installed** to confirm
  `beeline-workspace` is there and enabled. Check the **Errors** tab.
- Cowork: confirm an admin has made the plugin available to the user, then start
  a fresh session.

## Smoke test once connected

Run `/beeline-workspace:beeline-status`, or just ask *"what's our completion
rate?"*. Real numbers coming back means the connection, the OAuth scoping, and
the reporting warehouse are all working end to end.

## What not to promise

This connector reads and writes training content, group structure, reporting,
competency and role data. It does **not** cover performance reviews or coaching
— there are no tools for either. If someone asks for those, say they aren't part
of this integration rather than reaching for the nearest-looking tool.
