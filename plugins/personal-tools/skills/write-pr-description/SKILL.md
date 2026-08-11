---
name: write-pr-description
description: Write and publish the description of an existing Bitbucket pull request given its PR id or URL. Reads the PR's real base branch, verifies the fetched refs match the PR, runs the repository's pr-description command with that base, asks the user for the information a diff cannot provide, and updates the PR description via the Bitbucket API. Use when the user says "write the PR description for PR 1234", "update the description on that PR", or invokes /write-pr-description <id>.
argument-hint: '<pr-id-or-url>'
---

# Write PR description

Given a Bitbucket pull request, generate its description with the repository's own `pr-description` command and publish it to the PR. The outcome is binary: either the PR's description on Bitbucket is updated with a fully completed body, or the workflow reports a failure and ends **without modifying the PR**.

All Bitbucket API operations use the `mcp__bitbucket__bb_*` MCP tools (load them via ToolSearch if deferred). No local branches are created, checked out, or modified at any point — every git computation uses `origin/*` remote-tracking refs only, and the user's working tree is never touched.

## Step 1 — Resolve the PR and repository

1. **PR id**: from the skill argument — either a bare number (`1234`) or a Bitbucket PR URL (`https://bitbucket.org/<ws>/<repo>/pull-requests/1234[/…]`; take the number after `pull-requests/`). No argument, or no parseable number → fail: `write-pr-description requires a PR id or a Bitbucket PR URL.`
2. **Workspace/repo**: parse `git remote get-url origin` (strip `git@bitbucket.org:` or `https://bitbucket.org/`, strip trailing `.git`) → `<ws>/<repo>`. If origin is not a bitbucket.org remote, fail. If the argument was a URL naming a different `<ws>/<repo>` than origin, fail: the PR belongs to a different repository than this checkout.

## Step 2 — Read the PR

GET `/repositories/<ws>/<repo>/pullrequests/<id>`, extracting exactly: `title`, `state`, `source.branch.name`, `source.commit.hash`, `destination.branch.name`, `destination.commit.hash`, `reviewers` (uuid list), `description`.

- Not found / error → fail: `PR <id> not found in <ws>/<repo>.`
- `state` is not `OPEN` → fail, reporting the state. Merged or declined PRs do not get their descriptions rewritten.

Terminology for the rest of this skill: **target** = the PR's source branch; **base** = the PR's destination branch. The PR's destination is authoritative for the base — never derive the base from branch-name conventions.

## Step 3 — Fetch and verify the refs match the PR

1. `git fetch origin <base> <target>`
2. Verify `git rev-parse origin/<target>` starts with the PR's source commit hash, and `git rev-parse origin/<base>` starts with the PR's destination commit hash (Bitbucket returns short hashes — prefix-match).
3. On mismatch, fetch once more (transient lag); if still mismatched → fail, reporting the PR-recorded hash versus the fetched ref for whichever side disagrees. The description must be generated from exactly the state the PR renders.

## Step 4 — Sanity-check the base against target history

Determine which integration branch (`test`, `master`, or `release`) the target is actually built on top of, using merge-bases over `origin/` refs (skip any candidate missing on origin):

1. If `git merge-base origin/test origin/<target>` is **not** an ancestor of `origin/master` (check with `git merge-base --is-ancestor`) → built on `test`.
2. Else if `git merge-base origin/master origin/<target>` is **not** an ancestor of `origin/release` → built on `master`.
3. Else → built on `release`.

If the derived branch differs from the PR's base, warn the user via AskUserQuestion — the PR's rendered diff will contain commits belonging to the derived branch rather than to this change — and ask whether to proceed or stop. Stop → end without modifying the PR. Proceed → continue; the base used is always the PR's destination, and the derived branch is a cross-check only.

## Step 5 — Run the repository's pr-description command

1. Read `.claude/commands/pr-description.md` at the repository root **now, in full** — even if it was read earlier in the session. Its current content is the specification; never work from memory of what it says. Missing → fail: `This repository has no .claude/commands/pr-description.md.`
2. Read, also in full, **every template or reference file that its current text points at**. Which files those are is discovered from this fresh read, never assumed.
3. Execute the command's steps with exactly two overrides:
   - **Target** = `origin/<target>` — the pushed state the PR renders, not any local branch.
   - **Base** = `origin/<base>` from Step 2. Whatever base-resolution rule the command's own text contains is superseded; do not apply it.
4. Any failure the command's text defines → report its error and end without modifying the PR.

## Step 6 — Ask the user for what the diff cannot tell you

From the files read in Step 5 — as they exist right now — list every piece of content the final description requires that is **not obtainable from the diff, the commits, or the repository state**. What qualifies is determined entirely by that fresh read; do not rely on a remembered or assumed list of sections or questions. Whether and how the changes were verified is the classic example of such content, but only the current files define the actual set.

Ask the user for all of it in one batched AskUserQuestion call, and do so **after** the command's diff analysis so every question is concrete: name the feature and areas the diff touches, and offer likely answers as options (free text is always available via "Other").

Incorporate the answers where the command and its templates call for them. The published description must contain no "fill in"-style placeholders and nothing fabricated: whatever the user declines to answer is represented honestly, not guessed.

## Step 7 — Publish to the PR

1. Compose the final PR body: the description content the command produced (excluding any diagnostics or ancillary output its format defines), with the user's answers merged in.
2. If the body contains checklist-style items, tick exactly those that the user's answers or the composed content substantiate; leave every other item unchecked.
3. PUT `/repositories/<ws>/<repo>/pullrequests/<id>` with body `{"title": <unchanged from Step 2>, "description": <final body>, "reviewers": [<uuids unchanged from Step 2, each as {"uuid": …}>]}`. Title and reviewers are echoed back because the Bitbucket PR PUT clears reviewers when the field is omitted.
4. Confirm from the PUT response that the description was applied; on error, report the failure.

## Step 8 — Report

- The PR link (`https://bitbucket.org/<ws>/<repo>/pull-requests/<id>`) and a statement that the description was updated.
- The published body.
- If the previous description from Step 2 was non-empty and materially different from an untouched template, include it verbatim under "Previous description" so nothing is lost.

## Failure convention

Wherever this skill — or the command it runs — says fail: emit one clear error explaining what went wrong and stop. The PR is never modified on any failure path.
