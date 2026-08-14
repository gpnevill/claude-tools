---
name: ship-to-test
description: Ship the current working branch to the test branch via a transient dev/* PR — cherry-pick the working commits onto origin/test in a temp worktree, push, open a reviewer-less PR, run the build-artifacts custom pipeline, and merge fast-forward on green. Use when the user says "ship to test", "ship this to test", "get this branch into test", or invokes /ship-to-test.
---

# Ship to test

Automate the full working-branch → `test` shipping cycle. The outcome is binary: either the PR is merged into `test` with a green `build-artifacts` pipeline, or the workflow reports the failure and **ends**. All Bitbucket API operations use the `mcp__bitbucket__bb_*` MCP tools (load them via ToolSearch if deferred). All git construction happens in a temporary worktree — the user's checkout is never touched.

Fixed workflow constants: destination branch `test`, dev-branch prefix `dev/`, custom pipeline `build-artifacts`.

## Step 0 — Preconditions and derivation

1. **Working branch**: the skill argument if one was given (must exist locally), otherwise the current branch.
   - Must match `^(feat|fix|bugfix|chore|hotfix)/`. If it is `test`, `master`, or a `dev/*` branch, abort and tell the user.
2. **Dev branch name**: swap the prefix for `dev`, keep the rest verbatim.
   `feat/M2X-22718-tank-readiness-macro-report` → `dev/M2X-22718-tank-readiness-macro-report`
3. **Workspace/repo**: parse from `git remote get-url origin` (strip `git@bitbucket.org:` or `https://bitbucket.org/`, strip `.git`) → `<ws>/<repo>`.
4. `git fetch origin test master release`
5. **Base branch**: the working branch may sit on top of either `master` or `release`. Detect which — getting this wrong ships base-branch history into `test`:
   - Compare merge-bases: `mbm=$(git merge-base origin/master <working-branch>)` and `mbr=$(git merge-base origin/release <working-branch>)`. If they are equal, the base is `master`. If `mbm` is an ancestor of `mbr` (`git merge-base --is-ancestor "$mbm" "$mbr"`), the base is `release`; otherwise it is `master`.
   - State the detected base in the output.
6. **Commits to ship**: `git rev-list --reverse origin/<base>..<working-branch>`.
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

Run every subsequent git command with `git -C <dir>`.

### Cherry-pick

`git -C <dir> cherry-pick --empty=drop <sha1> <sha2> ...` (in the rev-list order). If the local git predates `--empty=drop`, omit the flag and resolve each "the previous cherry-pick is now empty" stop with `git cherry-pick --skip`.

Patches already in `test` verbatim (previous iterations of this work) drop silently — that is correct. A **conflict** almost always means `test` holds an *amended or older iteration* of the same work. Resolution doctrine:

- Where `test` contains an older iteration of this branch's work → the working branch's version wins outright.
- Where `test` has genuinely unrelated overlapping changes → integrate both intents; understand the test-side change (`git log origin/test -- <file>`) before writing the resolution.
- The resulting state of `dev/<slug>` must reflect the **latest** state of the working changes. When in doubt, the working branch is the source of truth.

### Verification gate (mandatory before pushing)

```
git -C <dir> diff <working-branch> HEAD -- $(git -C <dir> diff --name-only origin/<base>...<working-branch>)
```

For the files the working branch touched, the dev branch should now match the working branch. The diff must be empty, or every hunk must be attributable to a legitimate test-side change you can name. Anything unexplained means a botched resolution — fix it before proceeding.

### Push

`git -C <dir> push --force-with-lease -u origin dev/<slug>`

## Step 2 — Create the PR (no reviewers)

First check for an existing open PR to reuse:

`bb_get` `/repositories/<ws>/<repo>/pullrequests` with `queryParams: {"q": "source.branch.name=\"dev/<slug>\" AND destination.branch.name=\"test\" AND state=\"OPEN\""}`

If none, create one with `bb_post` `/repositories/<ws>/<repo>/pullrequests`:

```json
{
  "title": "<humanized branch name>",
  "description": "Transient PR to confirm a green build via the build-artifacts pipeline before shipping to test.",
  "source": {"branch": {"name": "dev/<slug>"}},
  "destination": {"branch": {"name": "test"}},
  "reviewers": [],
  "close_source_branch": true
}
```

Title humanization (Bitbucket-default style): capitalize the first letter, replace hyphens with spaces, but keep ticket tokens (`[A-Z]+-\d+`) intact: `dev/M2X-23049-interpolate-properly` → `Dev/M2X-23049 interpolate properly`.

**Reviewer scrub (mandatory):** immediately `bb_get` the PR (`jq: "{id: id, reviewers: reviewers}"`). Bitbucket's CODEOWNERS integration may auto-populate reviewers even when the create call passed an empty list. If reviewers is non-empty, clear it with `bb_put` `/repositories/<ws>/<repo>/pullrequests/<id>` and body `{"title": "<same title>", "reviewers": []}`, then re-check. No one gets pinged.

## Step 3 — Run and watch build-artifacts

Trigger with `bb_post` `/repositories/<ws>/<repo>/pipelines/`:

```json
{
  "target": {
    "type": "pipeline_ref_target",
    "ref_type": "branch",
    "ref_name": "dev/<slug>",
    "selector": {"type": "custom", "pattern": "build-artifacts"}
  }
}
```

Capture `uuid` and `build_number` from the response and report the pipeline URL to the user:
`https://bitbucket.org/<ws>/<repo>/pipelines/results/<build_number>`

**Watch loop** — poll `bb_get` `/repositories/<ws>/<repo>/pipelines/<uuid>` with `jq: "{state: state.name, result: state.result.name, stage: state.stage.name}"`. Between polls, run `sleep 180` via Bash with `run_in_background: true` and wait for its completion notification; do not busy-poll and do not use foreground sleep. Terminal conditions:

- `state == "COMPLETED"` → read `result`: `SUCCESSFUL` proceeds to Step 4; `FAILED` / `ERROR` / `STOPPED` go to the failure endgame.
- A paused/halted stage that requires manual action → treat as failure.

## Step 4 — Endgame

### Green → merge

`bb_post` `/repositories/<ws>/<repo>/pullrequests/<id>/merge` with:

```json
{"merge_strategy": "fast_forward", "close_source_branch": true}
```

`dev/<slug>` was built on `test`'s tip, so `fast_forward` is exactly a rebase-ff merge. If the merge fails because `test` advanced during the run: `git -C <dir> fetch origin test`, `git -C <dir> rebase origin/test`, `git -C <dir> push --force-with-lease origin dev/<slug>`, retry the merge **once**. If it still fails, report and stop. If that rebase conflicts, `git rebase --abort`, then report and stop — the conflicting `test` changes invalidate the green build, so do not resolve and push.

On success report: PR id/URL, pipeline build number, and the list of commits shipped. Then clean up: `git worktree remove <dir>` and delete the local branch (`git branch -D dev/<slug>`). The remote branch is closed by the merge.

### Not green → report and END

State that the build failed, with the pipeline URL, the failing step (`bb_get` `/pipelines/<uuid>/steps/` with `jq: "values[*].{name: name, result: state.result.name}"`), and the PR URL. Leave the remote branch and open PR in place for the next iteration; remove the temp worktree but keep the local `dev/<slug>` branch. **The workflow is finished** — do not wait for instructions, do not attempt fixes, do not re-run the pipeline.

## Invariants

- Never plain `--force`; always `--force-with-lease`.
- Never touch the user's checkout; all branch construction happens in the temp worktree.
- Never push before the verification gate passes.
- Always scrub reviewers after PR creation — CODEOWNERS can re-add them.
- Zero commits to ship, wrong branch prefix, or dev branch checked out elsewhere → abort before any remote side effect.
- On build failure: report and end. The skill has exactly two exits — merged, or failure reported.
