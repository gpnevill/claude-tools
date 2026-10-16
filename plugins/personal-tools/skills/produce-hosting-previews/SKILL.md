---
name: produce-hosting-previews
description: Produce the test-au/test-nz/test-us hosting preview URLs for a pushed branch - locate its most recent pipeline able to deploy previews, verify that run matches the branch's current remote state, wait for its build, trigger whichever of the three manual preview deploys have not run, wait for them, and report the three URLs. Writes them nowhere. Use when the user says "produce hosting previews for this branch", "get me the preview URLs", or invokes /produce-hosting-previews.
argument-hint: '[branch]'
---

# Produce hosting previews

Turn a pushed branch's pipeline into its three hosting preview URLs. Bitbucket API operations use the `mcp__bitbucket__bb_*` MCP tools (load via ToolSearch if deferred). Nothing local is modified, and the URLs are only reported — never recorded anywhere.

## Step 0 — Preconditions

1. **Branch**: the argument if given, otherwise the current branch.
2. **Workspace/repo**: parse `git remote get-url origin` (strip `git@bitbucket.org:` or `https://bitbucket.org/`, strip `.git`) → `<ws>/<repo>`.
3. `git fetch origin <branch>`; absent on origin → fail: `produce-hosting-previews: <branch> does not exist on origin.` Record `git rev-parse origin/<branch>`.

## Step 1 — Locate the pipeline

`bb_get` `/repositories/<ws>/<repo>/pipelines/` with `queryParams: {"target.branch": "<branch>", "sort": "-created_on"}`. Newest first, `bb_get` each run's `/repositories/<ws>/<repo>/pipelines/<uuid>/steps/` (`jq: "values[*].{name: name, uuid: uuid, result: state.result.name, state: state.name}"`) until one contains the steps `Manual preview test-au deploy`, `Manual preview test-nz deploy`, `Manual preview test-us deploy`. None → fail: `No preview-capable pipeline has run for <branch>.`

Its `target.commit.hash` must prefix-match the recorded SHA; otherwise fail: `The latest preview-capable pipeline for <branch> ran on <hash>; origin/<branch> is at <sha>.`

Report the pipeline URL: `https://bitbucket.org/<ws>/<repo>/pipelines/results/<build_number>`.

## Step 2 — Wait for the build

Poll the steps endpoint. Between polls, run `sleep 180` via Bash with `run_in_background: true` and wait for its completion notification; never busy-poll or foreground-sleep. Build step `SUCCESSFUL` → continue. `FAILED`/`ERROR`/`STOPPED` → fail, naming the step and the pipeline URL — nothing is deployed from a red build.

## Step 3 — Run the preview deploys

For each of the three manual steps still in the not-run state:

- First the API: `bb_post` `/repositories/<ws>/<repo>/pipelines/<uuid>/steps/<step_uuid>` (empty body). Bitbucket Cloud has historically not exposed manual-step triggering; treat 4xx/405 as unsupported.
- If unsupported: claude-in-chrome (load the core tool set via one ToolSearch call; `tabs_context_mcp` first, then a new tab) on the pipeline results URL — press **Run** on each, confirming any in-page dialog.

Steps already running or completed are left alone. Verify via the steps endpoint that all three have left the not-run state.

## Step 4 — Collect the URLs

Poll the steps endpoint (same cadence) until all three deploy steps complete. Any `FAILED`/`ERROR`/`STOPPED` → fail, naming the env(s) and the pipeline URL.

For each, `bb_get` `/repositories/<ws>/<repo>/pipelines/<uuid>/steps/<step_uuid>/log`. The step runs `./bin/hosting-deploy-preview.sh`, which ends with `firebase hosting:channel:deploy`; extract the `https://…web.app` channel URL it prints.

## Step 5 — Report

Pipeline URL and the three hosting URLs — au, nz, us.
