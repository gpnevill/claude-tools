---
name: ship-to-test
description: Ship the current working branch to the test branch locally — cherry-pick the working commits onto origin/test in a temp worktree, run the repository's full build and checks there, fast-forward test onto the result, and push test. Use when the user says "ship to test", "ship this to test", "get this branch into test", or invokes /ship-to-test.
---

# Ship to test

Automate the full working-branch → `test` shipping cycle. The outcome is binary: either `test` is pushed with the work fast-forwarded in after a green local build and check, or the workflow reports the failure and **ends**. All git construction and all building happen in a temporary worktree — the user's checkout is never touched.

Fixed workflow constants: destination branch `test`, dev-branch prefix `dev/`.

## Step 0 — Preconditions and derivation

1. **Working branch**: the skill argument if one was given (must exist locally), otherwise the current branch.
   - Must match `^(feat|fix|bugfix|chore|hotfix)/`. If it is `test`, `master`, or a `dev/*` branch, abort and tell the user.
2. **Dev branch name**: swap the prefix for `dev`, keep the rest verbatim.
   `feat/M2X-22718-tank-readiness-macro-report` → `dev/M2X-22718-tank-readiness-macro-report`
3. `git fetch origin test master release`
4. **Base branch**: the working branch may sit on top of either `master` or `release`. Detect which — getting this wrong ships base-branch history into `test`:
   - Compare merge-bases: `mbm=$(git merge-base origin/master <working-branch>)` and `mbr=$(git merge-base origin/release <working-branch>)`. If they are equal, the base is `master`. If `mbm` is an ancestor of `mbr` (`git merge-base --is-ancestor "$mbm" "$mbr"`), the base is `release`; otherwise it is `master`.
   - State the detected base in the output.
5. **Commits to ship**: `git rev-list --reverse origin/<base>..<working-branch>`.
   - Empty → abort: nothing to ship.
   - Print the list (`git log --oneline`) so the user sees exactly what will ship. Sanity-check it: the list must contain only the branch's own work — a tail of `master`/`release` history here means the base detection went wrong; stop and re-derive rather than shipping it.
   - If `git status --porcelain` shows uncommitted changes on the working branch, note that they will NOT ship — only commits do. Continue.

## Step 1 — Build the dev branch in a temp worktree

Never check out branches in the user's working directory. Use a worktree in a temporary directory outside the repo (the session scratchpad if available, else `mktemp -d`):

- If local `dev/<slug>` does not exist:
  `git worktree add -b dev/<slug> <dir> origin/test`
- If it exists but is checked out in another worktree (`git worktree list`), abort and tell the user.
- If it exists and is free:
  `git worktree add <dir> dev/<slug>` then `git -C <dir> reset --hard origin/test`

Run every subsequent git command with `git -C <dir>`. The dev branch is local construction only — it is never pushed.

### Cherry-pick

`git -C <dir> cherry-pick --empty=drop <sha1> <sha2> ...` (in the rev-list order). If the local git predates `--empty=drop`, omit the flag and resolve each "the previous cherry-pick is now empty" stop with `git cherry-pick --skip`.

Patches already in `test` verbatim (previous iterations of this work) drop silently — that is correct. A **conflict** almost always means `test` holds an *amended or older iteration* of the same work. Resolution doctrine:

- Where `test` contains an older iteration of this branch's work → the working branch's version wins outright.
- Where `test` has genuinely unrelated overlapping changes → integrate both intents; understand the test-side change (`git log origin/test -- <file>`) before writing the resolution.
- The resulting state of `dev/<slug>` must reflect the **latest** state of the working changes. When in doubt, the working branch is the source of truth.

### Verification gate (mandatory before building)

```
git -C <dir> diff <working-branch> HEAD -- $(git -C <dir> diff --name-only origin/<base>...<working-branch>)
```

For the files the working branch touched, the dev branch should now match the working branch. The diff must be empty, or every hunk must be attributable to a legitimate test-side change you can name. Anything unexplained means a botched resolution — fix it before proceeding.

## Step 2 — Full build and check, locally

Everything runs inside `<dir>`, against the dev branch as constructed.

1. **Install dependencies.** A fresh worktree has none. Discover the repository's package manager and install command from its root documentation and manifests — do not assume them — and install exactly as the lockfile dictates (frozen; never let the install rewrite it).
2. **Discover the commands.** From the repository's root documentation and manifests, find the build, lint, typecheck, and test commands. Run the full set for the whole repository, unscoped — no narrowing to affected packages.
3. **Run them.** Each command's output goes to its own log file in the temporary directory. Run each via Bash with `run_in_background: true` and wait for its completion notification; a full build routinely exceeds the foreground timeout, so never run these in the foreground and never busy-poll.
4. **Judge.** Every command must exit zero. Any non-zero exit → the failure endgame. Do not fix, do not re-run, do not narrow the set to make it pass.
5. **Tree gate.** `git -C <dir> status --porcelain` must show no modified tracked files (untracked build output is fine). A modified tracked file means a command rewrote source, so the tree that would ship is not the tree that was checked → the failure endgame.

Report each command and its result to the user as it completes.

## Step 3 — Merge into test

`dev/<slug>` was built on `test`'s tip, so merging it into `test` is exactly a fast-forward: `test` moves to the dev branch's `HEAD`. Do that with a plain push:

`git -C <dir> push origin HEAD:refs/heads/test`

The remote rejects a non-fast-forward, and that rejection is the signal that `test` advanced during the build. If the push is rejected: `git -C <dir> fetch origin test`, `git -C <dir> rebase origin/test`, then retry the push **once**. A clean rebase does not require a rebuild — push it. If it still fails, report and stop. If that rebase conflicts, `git rebase --abort`, then report and stop — the conflicting `test` changes invalidate the green build, so do not resolve and push.

## Step 4 — Endgame

### Green → shipped

On success report: the new `test` tip (`git -C <dir> rev-parse HEAD`), each build/check command and its result, and the list of commits shipped. Then clean up: `git worktree remove --force <dir>` (the worktree holds installed dependencies and build output) and delete the local branch (`git branch -D dev/<slug>`).

### Not green → report and END

State that the build or check failed, with the failing command, its exit code, and the relevant excerpt of its log. Remove the temp worktree (`git worktree remove --force <dir>`) but keep the local `dev/<slug>` branch so the failed state can be re-attached and inspected. **The workflow is finished** — do not wait for instructions, do not attempt fixes, do not re-run anything.

## Invariants

- Never force-push `test` in any form; a rejected non-fast-forward is a signal, not an obstacle.
- Never touch the user's checkout; all branch construction and all building happen in the temp worktree.
- Never build before the verification gate passes; never push before every build and check command has exited zero.
- Never scope, skip, or re-run a build or check command to reach green.
- Zero commits to ship, wrong branch prefix, or dev branch checked out elsewhere → abort before any remote side effect.
- On build or check failure: report and end. The skill has exactly two exits — shipped, or failure reported.
