---
name: open-worktrees
description: Open every git worktree of the current repository as a tab in Sublime Merge by running `smerge` once per worktree path. Use when the user says "open the worktrees in Sublime Merge", "smerge all my worktrees", or invokes /open-worktrees.
---

# Open worktrees in Sublime Merge

Open one Sublime Merge tab per existing worktree of the repository containing the current working directory.

## Step 1 — Verify the repository

```bash
git rev-parse --git-dir
```

If the working directory is not inside a git repository, tell the user and stop.

## Step 2 — Open a tab per worktree

Enumerate the worktrees and hand each existing path to `smerge`:

```bash
git worktree list --porcelain | sed -n 's/^worktree //p' | while read -r wt; do
  [ -d "$wt" ] && smerge "$wt"
done
```

Paths whose directory is gone (stale worktree records) are skipped rather than opened. Sublime Merge adds each repository as a tab in the existing window, so the loop yields one tab per worktree, in the order `git worktree list` reports them — the main working tree first.

## Step 3 — Report

List the worktree paths that were opened, and name any that were skipped as missing.
