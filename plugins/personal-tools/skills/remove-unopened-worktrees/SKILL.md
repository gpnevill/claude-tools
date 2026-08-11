---
name: remove-unopened-worktrees
description: Remove every git worktree of the current repository that is not open as a workspace in Herdr, asking per worktree before force-deleting one that git refuses over uncommitted changes. Use when the user says "remove the worktrees that aren't open in herdr", "clean up the worktrees herdr isn't using", or invokes /remove-unopened-worktrees.
---

# Remove worktrees not open in Herdr

Delete the checkouts of the repository containing the current working directory that Herdr does not have open as a workspace.

## Step 1 — Confirm Herdr answers

```bash
herdr worktree list --cwd "$PWD" | jq -e '.result.worktrees | type == "array"'
```

If this exits non-zero, stop and report that Herdr's worktree list is unavailable — the server is down, or the working directory is not inside a git work tree. Never fall through to step 2 on a failed read: without Herdr's answer every worktree looks closed and every one would be deleted.

## Step 2 — Select the candidates

```bash
herdr worktree list --cwd "$PWD" \
  | jq -r --arg self "$(git rev-parse --show-toplevel)" '
      .result.worktrees[]
      | select(.open_workspace_id == null and .is_linked_worktree and .path != $self)
      | .path'
```

`open_workspace_id` is absent on a worktree no workspace has open. `is_linked_worktree` excludes the repository's main checkout, which is never a candidate even when Herdr has no workspace on it. The current worktree is excluded so the removal cannot delete the directory it runs from.

If no candidate remains, say so and stop.

## Step 3 — Remove each candidate

```bash
git worktree remove "$path"
```

Never pass `--force` at this step, and never delete a branch — the commits of a removed worktree stay reachable on its branch, which is what makes removal recoverable.

Classify each failure by its stderr:

- `contains modified or untracked files` — carry to step 4.
- anything else (locked worktree, submodules, …) — skip it and report the message verbatim.

## Step 4 — Ask before discarding uncommitted work

For each worktree git refused, show what would be lost:

```bash
git -C "$path" status --short
```

Ask the user one yes/no question **per worktree** via AskUserQuestion, naming its branch and path and summarising that status — up to 4 questions per call, never one lumped question covering several worktrees.

On yes, discard the changes:

```bash
git worktree remove --force "$path"
```

On no, leave the worktree in place.

## Step 5 — Report

List the worktrees removed, the ones force-removed with their uncommitted changes discarded, the ones kept by the user's answer, and the ones skipped with the reason git gave.
