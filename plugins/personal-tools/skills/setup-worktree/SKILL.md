---
name: setup-worktree
description: Set up a worktree for a ticket or described piece of work - adopt the branch that already exists for it, or ask for a base and cut a new one, and create the worktree as a Herdr workspace via `herdr worktree create`. Use when the user says "set up a worktree for M2X-1234", "make me a branch for this work", or invokes /setup-worktree.
argument-hint: '<ticket-key and/or work description>'
---

# Setup worktree

Create the checkout a piece of work will be done in, as a Herdr workspace.

## Step 1 — Inputs

A ticket key (`[A-Z]+-\d+`) and/or a description of required work. Neither → ask for one and stop until provided.

With a key, read the ticket via the Atlassian MCP tools (`getJiraIssue`; load via ToolSearch if deferred) for its issue type and summary — the two fields a branch name is proposed from. A key that cannot be read is not fatal: say so and carry on with the key as given.

## Step 2 — Look for the branch

Search both sides on the ticket key, or on the argument itself when it has none:

```bash
git branch --list --format='%(objectname) %(refname:short)' | grep -i -- '<term>'
git ls-remote --heads origin | sed 's#refs/heads/##' | grep -i -- '<term>'
```

- **More than one branch name across the two** → stop and report them all.
- **One name on both sides, at different commits** → stop and report both commits.
- **One name** → that is the branch, and it is adopted. Skip step 3.
- **None** → step 3.

## Step 3 — Ask for the base and the new branch

Both in one `AskUserQuestion` call:

- **Base branch**: master / release / other (other → the user names it).
- **Branch**: type `feat` or `fix`, and the slug — suggest `<type>/<KEY>-<short-slug>`, or `<type>/<short-slug>` when there is no ticket. From a ticket, the type follows its issue type and the slug its summary. The key stays verbatim and uppercase in the branch name.

## Step 4 — Preflight

```bash
git worktree list --porcelain
```

A worktree already holds the branch → stop and report what holds it and where. What happens to work that already has a checkout is the user's call, not this skill's.

Then fetch what the next step needs — `git fetch origin <branch>` for a branch adopted from the remote alone, `git fetch origin <base>` for one being cut. Non-zero → stop and report; a branch cut from a stale base starts out behind.

## Step 5 — Create the worktree

```bash
herdr worktree create --cwd "$PWD" --branch <branch> --base origin/<base> --label <branch without its type prefix> --no-focus
```

`--base` is passed only when the branch is being cut; adopting one omits it. The label is the branch stripped of its leading `<type>/` segment, so `fix/M2X-23659-location-instruction-staleness` labels its workspace `M2X-23659-location-instruction-staleness`. `--path` is omitted so Herdr places the checkout; `--no-focus` leaves this session focused where it is.

Success prints one JSON line: the checkout is `.result.worktree.path`, the workspace `.result.workspace.label` (`.result.worktree.label` is the repository's name, not the label passed in). A non-zero exit, or a response without that path, is a stop — report the CLI's stderr verbatim. Never fall back to `git worktree add`: Herdr manages the worktrees, and one created behind its back has no workspace.

## Step 6 — Report

The worktree path, the branch, whether it was adopted or cut — naming the base in that last case — and the Herdr workspace label. The new worktree is open as a Herdr workspace; this session stays where it is.
