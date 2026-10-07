---
name: resolve-pr-comments
description: Triage and apply the review comments of a Bitbucket pull request given its PR id or URL. Checks out the PR's source branch (failing if local and remote are out of sync), itemizes the comments that call for changes, asks the user per comment whether the change is justified (yes / no / agent decides — auto or with confirmation), neutrally investigates the "agent decides" ones, applies all valid changes, commits and pushes them, then replies on the threads and resolves the actioned ones. Use when the user says "resolve the comments on PR 1234", "handle the review feedback on that PR", or invokes /resolve-pr-comments <id or URL>.
argument-hint: <pr-id or bitbucket PR URL>
---

# Resolve PR comments

Given a Bitbucket pull request, work through its review comments with the user: identify the comments that call for changes, let the user rule on each one (or delegate the ruling to you), apply every change deemed valid, commit and push, and answer every thread on the PR.

Hard rules that apply throughout:

- Git writes are limited to new commits of this skill's own edits on the PR branch and a plain push of that branch. Never amend, rebase, reset, force-push, or otherwise rewrite history.
- Bitbucket writes are limited to replies on the threads this skill handled and resolving the ones it actioned. Never update the PR itself. All Bitbucket access goes through the `mcp__bitbucket__bb_get` and `mcp__bitbucket__bb_post` MCP tools. Load them with ToolSearch (`select:mcp__bitbucket__bb_get,mcp__bitbucket__bb_post`) if they are deferred. If either is unavailable in this session, fail (see Failure conditions) — do not fall back to curl or any other access method.
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

Run `git status --porcelain`. If the output is non-empty, fail and tell the user to commit or stash their work first — this skill commits what it edits, and a dirty starting tree would mix unrelated changes into that commit.

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

Classify each remaining comment: does it call for a change to the code (a request, a defect report, a "should/must/please change …")? Praise, pure questions, and discussion that requests nothing are excluded. Replies within a thread belong to the thread's request — treat each thread as one item, keyed by its root comment's id and using the thread's full text as its content.

Present the resulting items to the user as a numbered list. For each item show: the location (`file:line` for inline comments, "PR-level" otherwise), the author, and the comment text — verbatim where short, faithfully condensed where long.

If there are no change-requesting comments, report that and end the skill successfully.

## Step 6 — Ask the user about every item

For every single item, ask the user whether the comment is justified and the change necessary. Use the AskUserQuestion tool, batching up to 4 items per call until all items have been asked. One question per item: header like "Comment 3", author of the comment, question text restating the comment and its location. The options, exactly and in this order, with none marked as recommended:

1. **Agent decides (confirmation)** — you investigate and rule, then put the ruling back to the user before acting on it
2. **Agent decides (auto)** — you investigate, rule on its validity yourself, and act on your ruling.
3. **Yes** — the comment is justified; make the change.
4. **No** — the comment is not justified; skip it.

Collect every answer before investigating or editing anything.

## Step 7 — Investigate every delegated item

For each item the user delegated, rule on its validity yourself. Begin from a uniform prior: the comment is exactly as likely to be invalid as valid. Assume nothing about it — do not defer to the reviewer's authority, and do not defer to the PR author's existing code. Read the relevant code, its surrounding conventions, and whatever else is needed to verify or refute the comment's actual claim on the evidence alone.

Record a verdict for each: **valid**, **invalid**, or **partially valid** — naming the part that holds — with a one-or-two-sentence justification grounded in what you found.

## Step 8 — Confirm the "agent decides (confirmation)" rulings

For every item delegated with confirmation, put the verdict and its justification to the user via AskUserQuestion. One question per item.

1. **Confirm** — the verdict stands.
2. **Override** — the verdict is reversed: a valid item is skipped, an invalid one is actioned. A partially valid item is expected to have an approach provided via 'Other' -- if not, ask again for how it should be resolved.

Anything finer — a different part, a different change — comes via Other and is honored as the user's ruling. Edit nothing until every confirmation has been collected.

## Step 9 — Apply all valid changes

The work set is: every item the user answered "Yes", every auto item ruled valid or partially valid, and every confirmation item whose final ruling — confirmed or overridden — is valid or partially valid.

Implement each change, following the conventions of the surrounding code and `${CLAUDE_PLUGIN_ROOT}/CODING_STANDARDS.md`. Make exactly the change the ruling calls for — nothing more.

## Step 10 — Commit and push

Stage exactly the files this skill edited and commit them as new commits with no commit message trailers, carrying the branch's ticket key where it has one (`fix: [M2X-1234] …`).

Then `git push origin <branch>`. Two ways this fails, handled differently:

- The push is rejected because the remote moved: report both heads and end the skill, with the commits left local. Do not fetch, rebase, or force — as in Step 3, reconciling is the user's decision.
- The push cannot be performed from this session (no credentials, no agent): give the user the exact line to run in this session — `! git push origin <branch>` — and continue to Step 11 only once its output shows the push succeeded.

## Step 11 — Respond on the PR

Per item, keyed by its thread's root comment id:

- **Actioned in full** → reply "Done" or a minimally short explanation of how the comment was resolved if not trivial from the comment itself, then resolve the thread: `bb_post` `/repositories/{workspace}/{repo}/pullrequests/{id}/comments/{root id}/resolve`.
- **Not actioned** → reply explaining why, and leave the thread open.
- **Actioned in part** → reply stating what was actioned, what was not, and why, and leave the thread open.

The explanation is the ruling's justification, or the user's reason. Where the user left an item unactioned without stating why — a bare **No**, or an **Override** with nothing added — ask for the reason via AskUserQuestion before replying: options **Out of scope for this PR** and **Intended as is**, anything else via Other.

A reply is `bb_post` `/repositories/{workspace}/{repo}/pullrequests/{id}/comments` with body `{"content": {"raw": "<text>"}, "parent": {"id": <root id>}}`. After every write, re-read the thread and confirm it landed — a reply that is not under its thread, or a thread that did not resolve, is a failure to report, not to silently accept.

Replies must be **minimally concise**. Where replying to a person — anything not obviously an automation or app account — be polite and epistemically humble: frame things collaboratively and open-endedly, and ask for their thoughts where that is justified. Present decisions as opinions with justification, not assertions. Communicate collaboration and response-invitation through phrasing and structure of the main text, instead of ending replies with standalone questions such as "happy to do your way" or "what do you think". Minimal conciseness should not drop any explanation or justification.

## Step 12 — Final report

Report, per item: the decision (user-yes, user-no, agent-valid, agent-invalid, agent-partial — and for confirmation items whether it was confirmed or overridden), your justification for any agent ruling, the files edited, and what was posted on its thread. Then the commits made, and whether the branch was pushed or the push was handed to the user. The skill is done.

## Failure conditions

End the skill immediately with a clear report and no repository mutation when:

- the origin remote is not a bitbucket.org repository and no full PR URL was given (Step 0);
- the `mcp__bitbucket__bb_get` or `mcp__bitbucket__bb_post` tool is unavailable in this session;
- the PR id cannot be found (Step 1);
- the working tree is dirty at the start (Step 2);
- the PR's source branch does not exist on the remote (Step 3);
- the local and remote branch states are out of sync (Step 3).
