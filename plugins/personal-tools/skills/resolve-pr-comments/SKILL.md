---
name: resolve-pr-comments
description: Triage and apply the review comments of a Bitbucket pull request given its PR id or URL. Checks out the PR's source branch (failing if local and remote are out of sync), itemizes the comments that call for changes, asks the user per comment whether the change is justified (yes / no / agent decides), neutrally investigates the "agent decides" ones, applies all valid changes, and leaves them unstaged for the user to review. Use when the user says "resolve the comments on PR 1234", "handle the review feedback on that PR", or invokes /resolve-pr-comments <id or URL>.
argument-hint: <pr-id or bitbucket PR URL>
---

# Resolve PR comments

Given a Bitbucket pull request, work through its review comments with the user: identify the comments that call for changes, let the user rule on each one (or delegate the ruling to you), apply every change deemed valid, and stop with the edits unstaged.

Hard rules that apply throughout:

- Never run `git add`, `git commit`, `git push`, or any other command that stages, commits, publishes, or rewrites history. The end state is a dirty working tree on the PR branch and nothing else.
- Never write to Bitbucket (no comment replies, no resolving threads, no PR updates). All Bitbucket access is read-only via the `mcp__bitbucket__bb_get` MCP tool. Load it with ToolSearch (`select:mcp__bitbucket__bb_get`) if it is deferred. If the tool is unavailable in this session, fail (see Failure conditions) — do not fall back to curl or any other access method.
- On any failure condition, stop immediately with a clear report and leave the repository untouched.

## Step 0 — Resolve the PR coordinates

Accept either a bare numeric PR id or a full URL of the form `https://bitbucket.org/<workspace>/<repo>/pull-requests/<id>`.

- From a URL, take workspace, repo, and id directly.
- From a bare id, derive workspace and repo from `git remote get-url origin`, handling both `git@bitbucket.org:<workspace>/<repo>.git` and `https://bitbucket.org/<workspace>/<repo>.git` forms.
- If no URL was given and origin is not a bitbucket.org remote, fail.

## Step 1 — Fetch the PR

`bb_get` with path `/repositories/{workspace}/{repo}/pullrequests/{id}` and a `jq` filter selecting only `title`, `description`, `state`, `source.branch.name`, `source.commit.hash`, and `destination.branch.name`.

The source branch name is the working branch for the rest of the skill. If the PR does not exist, fail.

## Step 2 — Precondition: clean working tree

Run `git status --porcelain`. If the output is non-empty, fail and tell the user to commit or stash their work first — this skill ends with unstaged edits, and a dirty starting tree would mix unrelated changes into them.

## Step 3 — Check out the branch, with sync guard

1. `git fetch origin <branch>`. If the remote branch does not exist, fail.
2. If no local branch of that name exists (`git rev-parse --verify refs/heads/<branch>` fails): create one with `git checkout -b <branch> origin/<branch>` and continue.
3. If a local branch exists: compare `git rev-parse refs/heads/<branch>` with `git rev-parse refs/remotes/origin/<branch>`.
   - Equal: check the branch out (a no-op if it is already current) and continue.
   - Different: **fail**. Report that the local and remote states of `<branch>` are out of sync, show both commit hashes, and end the skill. Do not fast-forward, merge, reset, or otherwise reconcile — that decision belongs to the user.

## Step 4 — Read the PR and all comments

Read the PR title and description from Step 1, then fetch every comment:

- `bb_get` with path `/repositories/{workspace}/{repo}/pullrequests/{id}/comments`, `queryParams: {"pagelen": "100"}`, and a `jq` filter selecting per comment: `id`, author display name, raw content, inline `path` and line if present, `deleted`, parent comment id if present, and any resolution field the payload carries.
- Follow pagination: while the response has a `next` link, request the next `page` until exhausted. Every comment must be read.
- Discard comments marked deleted and threads already marked resolved.

## Step 5 — Itemize the change-requesting comments

Classify each remaining comment: does it call for a change to the code (a request, a defect report, a "should/must/please change …")? Praise, pure questions, and discussion that requests nothing are excluded. Replies within a thread belong to the thread's request — treat each thread as one item, using the thread's full text as its content.

Present the resulting items to the user as a numbered list. For each item show: the location (`file:line` for inline comments, "PR-level" otherwise), the author, and the comment text — verbatim where short, faithfully condensed where long.

If there are no change-requesting comments, report that and end the skill successfully.

## Step 6 — Ask the user about every item

For every single item, ask the user whether the comment is justified and the change necessary. Use the AskUserQuestion tool, batching up to 4 items per call until all items have been asked. One question per item: header like "Comment 3", question text restating the comment and its location. The options, exactly and in this order, with none marked as recommended:

1. **Yes** — the comment is justified; make the change.
2. **No** — the comment is not justified; skip it.
3. **Agent decides** — you investigate and rule on its validity yourself.

Collect every answer before investigating or editing anything.

## Step 7 — Investigate every "agent decides" item

For each item the user delegated, rule on its validity yourself. Begin from a uniform prior: the comment is exactly as likely to be invalid as valid. Assume nothing about it — do not defer to the reviewer's authority, and do not defer to the PR author's existing code. Read the relevant code, its surrounding conventions, and whatever else is needed to verify or refute the comment's actual claim on the evidence alone.

Record a verdict for each: **valid** or **invalid**, with a one-or-two-sentence justification grounded in what you found.

## Step 8 — Apply all valid changes

The work set is: every item the user answered "Yes" plus every delegated item you ruled valid.

Implement each change, following the conventions of the surrounding code. Make exactly the change the comment calls for — nothing more. Leave everything unstaged.

## Step 9 — Final report

Report, per item: the decision (user-yes, user-no, agent-valid, agent-invalid), your justification for any agent decision, and for applied changes the files edited. Close by stating that all edits sit unstaged on `<branch>` for the user to review and handle. The skill is done.

## Failure conditions

End the skill immediately with a clear report and no repository mutation when:

- the origin remote is not a bitbucket.org repository and no full PR URL was given (Step 0);
- the `mcp__bitbucket__bb_get` tool is unavailable in this session;
- the PR id cannot be found (Step 1);
- the working tree is dirty at the start (Step 2);
- the PR's source branch does not exist on the remote (Step 3);
- the local and remote branch states are out of sync (Step 3).
