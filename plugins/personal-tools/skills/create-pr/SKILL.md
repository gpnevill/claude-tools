---
name: create-pr
description: Create a Bitbucket pull request from a source branch to a target branch with exactly the reviewers given - none when none are given, never Bitbucket's auto-added CODEOWNERS reviewers - and the source branch set to close on merge. Fails unless the source branch exists on origin. Reports the PR id and URL. Use when the user says "create a PR from feat/x to master", "open a PR for this branch with Jane as reviewer", or invokes /create-pr.
argument-hint: '<source-branch> <target-branch> [reviewer ...]'
---

# Create PR

Open a pull request for an already-pushed branch with exactly the reviewers given. Bitbucket API operations use the `mcp__bitbucket__bb_*` MCP tools (load via ToolSearch if deferred). Nothing local is modified.

## Step 1 — Inputs

1. **Source** and **target**: the first two arguments. Either missing → fail: `create-pr requires a source branch and a target branch.`
2. **Workspace/repo**: parse `git remote get-url origin` (strip `git@bitbucket.org:` or `https://bitbucket.org/`, strip `.git`) → `<ws>/<repo>`.
3. `git ls-remote --exit-code --heads origin <source>`, and likewise `<target>`; either absent → fail: `create-pr: <branch> does not exist on origin.`
4. **Reviewers**: every further argument — a Bitbucket display name, nickname, or account uuid. Resolve each to a uuid against the workspace members (`bb_get` `/workspaces/<ws>/members`, following `next` pages); unresolved or ambiguous → fail naming it. None given → no reviewers.
5. An open or draft PR from `<source>` to `<target>` already exists (`bb_get` `/repositories/<ws>/<repo>/pullrequests` with `queryParams: {"q": "source.branch.name=\"<source>\" AND destination.branch.name=\"<target>\" AND (state=\"OPEN\" OR state=\"DRAFT\")"}`) → fail with its URL.

## Step 2 — Create

`bb_post` `/repositories/<ws>/<repo>/pullrequests`:

```json
{
  "title": "<humanized source branch>",
  "source": {"branch": {"name": "<source>"}},
  "destination": {"branch": {"name": "<target>"}},
  "reviewers": [{"uuid": "<uuid>"}, …],
  "close_source_branch": true
}
```

Title humanization: capitalize the first letter, hyphens to spaces, ticket tokens (`[A-Z]+-\d+`) kept intact: `feat/M2X-23049-interpolate-properly` → `Feat/M2X-23049 interpolate properly`.

**Reviewer scrub (mandatory)**: immediately `bb_get` the PR (`jq: "{id: id, reviewers: reviewers}"`). Bitbucket's CODEOWNERS integration may add reviewers beyond those passed. If the reviewer set differs from the input, reset it with `bb_put` `/repositories/<ws>/<repo>/pullrequests/<id>` and body `{"title": "<same title>", "reviewers": [<input uuids>]}`, then re-check. Only the given reviewers get pinged.

## Step 3 — Report

PR id and URL: `https://bitbucket.org/<ws>/<repo>/pull-requests/<id>`.
