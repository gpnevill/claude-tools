---
name: review-pr
description: Adversarially review a Bitbucket pull request given its PR id or URL. Reads the PR's rendered diff from origin refs, holds it to the plugin's CODING_STANDARDS.md, the user-scoped CLAUDE.md, and the scrutiny of a hostile human reviewer, and reports findings in four severity sections from merge-blocking to nice-to-have. Use when the user says "review PR 1234", "give that PR a hard review", or invokes /review-pr <id>.
argument-hint: '<pr-id-or-url>'
---

# Review PR

Given a Bitbucket pull request, interrogate its diff as its most hostile reviewer and report what it finds, graded by severity. The skill reads; it changes nothing.

Hard rules that apply throughout:

- Read-only everywhere. Never write to Bitbucket; never check out, reset, or otherwise modify a local branch or the working tree. Every git computation uses `origin/*` refs only.
- All Bitbucket access is via the `mcp__bitbucket__bb_get` MCP tool (load it with ToolSearch if deferred). If the tool is unavailable, fail.

## Step 1 — Resolve the PR

1. **PR id**: a bare number or a Bitbucket PR URL (`https://bitbucket.org/<ws>/<repo>/pull-requests/<id>[/…]`). None, or none parseable → fail: `review-pr requires a PR id or a Bitbucket PR URL.`
2. **Workspace/repo**: parse `git remote get-url origin` (strip `git@bitbucket.org:` or `https://bitbucket.org/` and a trailing `.git`) → `<ws>/<repo>`. Origin not a bitbucket.org remote → fail. A URL naming a different `<ws>/<repo>` than origin → fail.
3. `bb_get` `/repositories/<ws>/<repo>/pullrequests/<id>`, extracting `title`, `description`, `source.branch.name`, `source.commit.hash`, `destination.branch.name`, `destination.commit.hash`. Not found → fail: `PR <id> not found in <ws>/<repo>.`

**Target** = the source branch; **base** = the destination branch.

## Step 2 — Fetch and verify the refs

`git fetch origin <base> <target>`. Verify `git rev-parse origin/<target>` starts with the PR's source hash and `git rev-parse origin/<base>` starts with its destination hash (Bitbucket returns short hashes). On mismatch, fetch once more; still mismatched → fail, reporting the PR-recorded hash versus the fetched ref. The review must cover exactly the state the PR renders.

## Step 3 — Establish the change set

The diff under review is `git diff origin/<base>...origin/<target>`, which is what Bitbucket renders. Read every changed file in full via `git show origin/<target>:<path>`, and every call site of every changed or removed symbol (`git grep -n <symbol> origin/<target>`), including in files the diff does not touch. A review confined to hunks cannot judge a hunk.

## Step 4 — Load the standard

Read `${CLAUDE_PLUGIN_ROOT}/CODING_STANDARDS.md` in full, now, then `${CLAUDE_CONFIG_DIR}/CLAUDE.md` in full where it exists. Their current content is the standard; where the two differ, CLAUDE.md wins. Never work from memory of either.

## Step 5 — Review

The diff carries the burden of proof. The PR title, description, commit messages, the presence of tests, and a green pipeline are claims to verify against the code, not evidence for it. Do not credit the author's judgment; conformity with surrounding code that is itself wrong is not a defense.

Hold the change set to, in order:

1. **The standard** from Step 4, rule by rule, including its own review posture. Where the standard assigns a check to review (hand-copied external literals, for one), perform it against the external source.
2. **Correctness.** Trace every changed path through its boundary values, error paths, nullability, ordering, and concurrency. Verify every changed signature at every call site. A test counts only if it can fail and pins behavior the specification requires.
3. **What a human reviewer flags.** Changes unrelated to the PR's stated intent; dead, unused, or debug code; dishonest or vague names; magic literals; comments the code could have carried; inconsistency with the surrounding file and the repository's root documentation; behavior changes the description does not disclose.

Every finding cites the quoted code or the named rule it rests on. Severity is set by impact, not by confidence; where confidence is less than certain, say so in the finding. An empty section stays empty.

## Step 6 — Report

Open with the PR id, title, `<target> → <base>`, and the hashes reviewed. Then exactly four sections, in this order, each a numbered list of findings or the single line `None.`:

1. **Must resolve — blocking merge.** Defects, data loss, security exposure, breaking changes, tests that cannot fail, and violations of anything the standard prohibits outright.
2. **Should resolve — material concern.** Standard violations short of outright prohibition, unverified error paths with plausible impact, specified behavior no test pins, boundary leaks.
3. **Would prefer another way — reasonably up to the author, worth a discussion.** Defensible choices a reviewer would not have made. Each names the alternative and its tradeoff; without one there is nothing to discuss.
4. **Nice to have — small quality tweaks.** Naming, idiom, ordering, minor redundancy.

Each finding: `file:line` on the target side, what the code does, and the grounds. Nothing else in the output; the report is the deliverable.

## Failure conditions

End immediately with a clear report and no mutation when: no parseable PR id (Step 1); origin is not a bitbucket.org remote, or the URL names another repository (Step 1); `mcp__bitbucket__bb_get` is unavailable; the PR is not found (Step 1); the fetched refs do not match the PR after a second fetch (Step 2).
