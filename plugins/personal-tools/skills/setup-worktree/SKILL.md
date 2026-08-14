---
name: setup-worktree
description: Set up a worktree for a ticket or described piece of work - ask for the base and new branch, fetch the base, and create the worktree as a Herdr workspace via `herdr worktree create`. Use when the user says "set up a worktree for M2X-1234", "make me a branch for this work", or invokes /setup-worktree.
argument-hint: '<ticket-key and/or work description>'
---

# Setup worktree

Create the branch and checkout a piece of work will be done in, as a Herdr workspace.

## Step 1 — Inputs

A ticket key (`[A-Z]+-\d+`) and/or a description of required work. Neither → ask for one and stop until provided.

With a key, read the ticket via the Atlassian MCP tools (`getJiraIssue`; load via ToolSearch if deferred) for its issue type and summary — the two fields a branch name is proposed from. A key that cannot be read is not fatal: say so and carry on with the key as given.

## Step 2 — Ask for the base and the branch

Both in one `AskUserQuestion` call:

- **Base branch**: master / release / other (other → the user names it).
- **Branch**: type `feat` or `fix`, and the slug — suggest `<type>/<KEY>-<short-slug>`, or `<type>/<short-slug>` when there is no ticket. From a ticket, the type follows its issue type and the slug its summary. The key stays verbatim and uppercase in the branch name.

## Step 3 — Fetch the base

```bash
git fetch origin <base>
```

Non-zero → stop and report. A branch cut from a stale base starts out behind.

## Step 4 — Preflight

```bash
git rev-parse --verify <branch>
git ls-remote --heads origin <branch>
git worktree list --porcelain
```

If the branch already exists locally or on the remote, or a worktree already holds it, stop and report what exists and where. What happens to work that already has a checkout is the user's call, not this skill's.

## Step 5 — Create the worktree

```bash
herdr worktree create --cwd "$PWD" --branch <branch> --base origin/<base> --label <KEY, or the slug when there is no ticket> --no-focus
```

`--path` is omitted so Herdr places the checkout; `--no-focus` leaves this session focused where it is.

Success prints one JSON line: the checkout is `.result.worktree.path`, the workspace `.result.workspace.label` (`.result.worktree.label` is the repository's name, not the label passed in). A non-zero exit, or a response without that path, is a stop — report the CLI's stderr verbatim. Never fall back to `git worktree add`: Herdr manages the worktrees, and one created behind its back has no workspace.

## Step 6 — Report

The worktree path, the branch, the base it was cut from, and the Herdr workspace label. The new worktree is open as a Herdr workspace; this session stays where it is.
