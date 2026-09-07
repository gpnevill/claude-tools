---
name: orchestrate-pr-lifecycles
description: Read every open or draft pull request I authored or am a reviewer of and run /progress-pr over the whole set. Use when the user says "progress all my PRs", "run my pull request lifecycles", or invokes /orchestrate-pr-lifecycles.
---

# Orchestrate PR lifecycles

Find every open or draft pull request of this repository that is mine and hand the set to `/progress-pr`.

All Bitbucket API operations use the `mcp__bitbucket__bb_*` MCP tools (load them via ToolSearch if deferred).

## Step 1 — Establish the repository and the identity

```bash
git remote get-url origin
```

Not a bitbucket.org remote → **fail**: `orchestrate-pr-lifecycles must run in a checkout whose origin is a Bitbucket repository.` Strip `git@bitbucket.org:` or `https://bitbucket.org/` and a trailing `.git` to give `<ws>/<repo>`.

`bb_get` `/user` with `jq: "{uuid: uuid}"` for my account uuid.

## Step 2 — Find the pull requests

Two `bb_get` calls on `/repositories/<ws>/<repo>/pullrequests`:

```
queryParams: {"q": "(state=\"OPEN\" OR state=\"DRAFT\") AND author.uuid=\"<uuid>\"", "pagelen": "50"}
queryParams: {"q": "(state=\"OPEN\" OR state=\"DRAFT\") AND reviewers.uuid=\"<uuid>\"", "pagelen": "50"}
```

Follow `next` where either response is paged. Merge the two deduplicated by id, keeping the order the first returned them in. Either call failing is a stop — report it and **end**.

## Step 3 — List them back

The ids with their title, source branch, and whether each is mine as author, as reviewer, or both. None → say so and **end**.

## Step 4 — Hand them over

Invoke `/progress-pr` with every id found, joined by `; `. Each pull request is offered a skip there, so the whole set is passed on unfiltered.

## Step 5 — Report

`/progress-pr`'s report is the report. Add the repository swept, the count, and the two queries used.
