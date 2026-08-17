---
name: kickoff
description: Kick off a semicolon-separated list of tickets and/or described pieces of work at once - per item, set up a Herdr worktree and start a Claude Code session in it, in plan mode, running /deliver on that item. Use when the user says "kick off M2X-1234; M2X-1235", "start these in their own worktrees", or invokes /kickoff.
argument-hint: '<ticket-key and/or work description>; <ticket-key and/or work description>; …'
---

# Kickoff

Fan a list of work items out into one Herdr worktree per item, each with its own Claude Code session in plan mode already running `/deliver` on that item. This skill sets the worktrees up and starts those sessions; it never follows them.

## Non-negotiable mechanics

- **This session never moves.** Never `cd` into a created worktree, never adopt one as this session's working directory, never focus a created workspace, tab, or agent. Every item is delivered by the session started for it — delivering one here would put the work in the wrong checkout, which is the whole reason this skill exists.
- **The worktree is `/setup-worktree`'s to create.** Invoke that skill per item; never call `herdr worktree create` or `git worktree add` in its place.
- **Items are independent.** Whatever stops one item is reported against that item alone; the remaining items still run.

## Step 1 — Precondition

```bash
test "${HERDR_ENV:-}" = 1
```

Non-zero → this session is not running inside Herdr and cannot open workspaces or start agents. Stop and say so.

## Step 2 — Parse the items

Split the argument on `;`, trim each segment, drop the empty ones. Each remaining segment is one **item**, carried **verbatim** from here on: `M2X-124 fix issue` stays `M2X-124 fix issue` — trailing prose is kept, and nothing is re-derived from a ticket. No items → ask for them and stop until given.

Verbatim has to survive the shell as well. Every item and path below reaches its command as one literal argument — single-quoted, with any embedded single quote escaped — and never interpolated into a double-quoted string, where a `$`, a backtick or a backslash in a description is eaten before the command runs and the wrong work is delivered under a zero exit status.

List the parsed items back before creating anything, so a mis-split is caught while it still costs nothing.

## Step 3 — Per item, in order

One item at a time. Step 3.1 interrogates the user, and questions covering several worktrees at once are not that skill's contract.

### 3.1 Create the worktree

Invoke `/setup-worktree <item>`, and take the **worktree path** and the **branch** from its report.

That skill stops on its own preflight failures — an existing branch, a failed fetch. Carry its reason to the report and move to the next item.

### 3.2 Resolve the pane

A freshly created worktree workspace holds exactly one tab with one pane, sitting at its shell prompt:

```bash
ws=$(herdr worktree list --cwd "$PWD" | jq -r --arg p '<worktree path>' '.result.worktrees[] | select(.path == $p) | .open_workspace_id')
herdr pane list --workspace "$ws" | jq -r '.result.panes[].pane_id'
```

Anything but exactly one pane id — no workspace open on the path, or a pane count that says someone is already working there — is a stop for this item: report it and move on.

### 3.3 Name the agent

Herdr agent names match `[a-z][a-z0-9_-]{0,31}` and must be unique among live agents. Take the branch from step 3.1, drop its `<type>/` prefix and lowercase the rest (`feat/M2X-24022-bulk-endpoint` → `m2x-24022-bulk-endpoint`), replacing any other character with `-` and prefixing a letter when the result would not otherwise start with one. Check it against `herdr agent list` and suffix `-2`, `-3`, … until it is unique, trimming the tail to fit 32 characters including the suffix.

### 3.4 Start Claude Code

```bash
herdr agent start <name> --kind claude --pane <pane id> --timeout 120000 -- --permission-mode plan
```

The session starts in plan mode, so it reaches the user for approval before it writes anything. Arguments after `--` are Claude Code's own; everything before it is Herdr's. The startup allowance is raised over the 30-second default because a repository whose session hooks and configuration are slow to load would otherwise fail an item whose worktree already exists.

A non-zero exit says nothing about why. Read the pane before concluding anything:

```bash
herdr pane read <pane id> --source detection --lines 40
```

Report that output verbatim against the item and move to the next. Leave the worktree in place — what happens to a checkout no session took up is the user's call.

### 3.5 Check the session is taking input

```bash
herdr pane read <pane id> --source detection --lines 40
```

A zero exit from step 3.4 is not proof the session can be prompted: Claude Code asks a first-time repository whether its folder is trusted, and Herdr reports that agent as started and idle with the question still on screen. Read the pane and expect Claude's empty input prompt. Anything else — a trust question, a numbered choice, any confirmation — stops this item and is reported, because the next step's Enter would answer that question instead and the delivery would be lost under a zero exit status. Trust is recorded against the repository's main checkout, so at most the first worktree of a repository is ever affected.

### 3.6 Submit the delivery

```bash
herdr agent prompt <name> '/deliver <item>'
```

The item text is passed through exactly as parsed, and the leading `/` reaches Claude Code as the slash command it is.

Never pass `--wait`: `/deliver` runs for as long as the work takes and puts questions to the user along the way, so every state this could wait for is either hours away or a question mistaken for a failure. A zero exit means the prompt was submitted, and submitted is all this skill needs. A non-zero exit takes the same route as step 3.4 — read the pane, report it verbatim against the item, next item.

## Step 4 — Report

One row per item: the item text, worktree path, branch, Herdr workspace label, agent name, and that the `/deliver` prompt was submitted — or, for an item that did not get that far, the step that stopped it and why.

Close by saying that each session is running in plan mode and will ask for approval there before it writes anything, that some are likely already waiting on the user, and that this session has not moved.
