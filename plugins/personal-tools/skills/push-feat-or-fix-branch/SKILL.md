---
name: push-feat-or-fix-branch
description: Push a feat/ or fix/ working branch, open a reviewer-less PR, and - when the branch has frontend changes and the user wants them - run the manual preview hosting deploys for test-au/test-nz/test-us and write the resulting hosting URLs into the chosen ticket(s)' test-link fields. Fails on any other branch type. Use when the user says "push this branch", "open the PR", "create feature branches", or invokes /push-feat-or-fix-branch.
argument-hint: '[branch]'
---

# Push feat or fix branch

Push the main working branch, create its PR, and optionally produce feature-branch hosting previews and wire their URLs into Jira. Bitbucket API operations use the `mcp__bitbucket__bb_*` MCP tools; Jira operations use the Atlassian MCP tools (load either via ToolSearch if deferred). The user's checkout is never modified — only pushed.

## Step 0 — Preconditions

1. **Branch**: the skill argument if given (must exist locally), otherwise the current branch.
2. **Hard gate**: the branch must match `^(feat|fix)/`. Anything else — including `bugfix/`, `chore/`, `hotfix/` — fails immediately: `push-feat-or-fix-branch only operates on feat/ or fix/ branches; got <branch>.`
3. **Workspace/repo**: parse `git remote get-url origin` (strip `git@bitbucket.org:` or `https://bitbucket.org/`, strip `.git`) → `<ws>/<repo>`.
4. `git fetch origin master release <branch>` (tolerate the branch not existing on origin yet).

## Step 1 — Push

- Branch not on origin → `git push -u origin <branch>`.
- Branch on origin and local is strictly ahead → plain `git push origin <branch>`.
- Histories diverged → `git push --force-with-lease origin <branch>`. Never plain `--force`.

Record the pushed head SHA. The push automatically starts the `{bugfix,feat,fix,merge}/*` pipeline for the branch.

## Step 2 — PR (no reviewers)

**Destination**: determine which integration branch the work is built on. Compute `mbM = git merge-base origin/master <branch>` and `mbR = git merge-base origin/release <branch>`; the destination is the candidate whose merge-base is a descendant of the other's (`git merge-base --is-ancestor`). Equal or incomparable → ask the user via `AskUserQuestion` (options: master, release).

Reuse an existing open PR if present: `bb_get` `/repositories/<ws>/<repo>/pullrequests` with `queryParams: {"q": "source.branch.name=\"<branch>\" AND state=\"OPEN\""}`. Otherwise create with `bb_post` `/repositories/<ws>/<repo>/pullrequests`:

```json
{
  "title": "<humanized branch name>",
  "source": {"branch": {"name": "<branch>"}},
  "destination": {"branch": {"name": "<destination>"}},
  "reviewers": [],
  "close_source_branch": true
}
```

Title humanization: capitalize the first letter, hyphens to spaces, ticket tokens (`[A-Z]+-\d+`) kept intact: `feat/M2X-23049-interpolate-properly` → `Feat/M2X-23049 interpolate properly`.

**Reviewer scrub (mandatory)**: immediately `bb_get` the PR (`jq: "{id: id, reviewers: reviewers}"`). Bitbucket's CODEOWNERS integration may auto-populate reviewers even when the create call passed an empty list. If reviewers is non-empty, clear with `bb_put` `/repositories/<ws>/<repo>/pullrequests/<id>` and body `{"title": "<same title>", "reviewers": []}`, then re-check. No one gets pinged — not even default CODEOWNERS reviewers.

## Step 3 — Frontend change detection

List the branch's changes: `git diff --name-only origin/<destination>...<branch>`. Judge from the actual files whether any change reaches the frontend (application code, client-reachable package code, hosting assets — as opposed to purely server-side services, infrastructure, or docs). **When uncertain, ask the user** via `AskUserQuestion` rather than guessing.

**No frontend changes** → report the PR URL and **finish**.

## Step 4 — Feature branch questions

Frontend changes exist. Ask via `AskUserQuestion`:

1. **Produce feature branches (hosting previews)?** — yes / no. **No** → report the PR URL and finish.
2. **For which ticket(s)?** — options: the ticket key from the branch slug if one matches `[A-Z]+-\d+`, and **No ticket**. One or more other tickets are supplied via the built-in Other (comma-separated keys).

## Step 5 — Wait for the branch pipeline's build

Locate the pipeline run for the pushed commit: `bb_get` `/repositories/<ws>/<repo>/pipelines/` with `queryParams: {"target.branch": "<branch>", "sort": "-created_on"}`, matching `target.commit.hash` against the pushed SHA. Report its URL (`https://bitbucket.org/<ws>/<repo>/pipelines/results/<build_number>`).

Poll `bb_get` `/repositories/<ws>/<repo>/pipelines/<uuid>/steps/` (`jq: "values[*].{name: name, uuid: uuid, result: state.result.name, state: state.name}"`). Between polls, run `sleep 180` via Bash with `run_in_background: true` and wait for its completion notification; never busy-poll or foreground-sleep. Continue when the build step reports `SUCCESSFUL`; if it reports `FAILED`/`ERROR`/`STOPPED`, report the failing step and pipeline URL and **end** — nothing is deployed from a red build.

## Step 6 — Trigger the three manual preview deploys

The pipeline contains manual steps named `Manual preview test-au deploy`, `Manual preview test-nz deploy`, `Manual preview test-us deploy`.

- First attempt the API: `bb_post` `/repositories/<ws>/<repo>/pipelines/<uuid>/steps/<step_uuid>` (empty body) for each. Bitbucket Cloud has historically not exposed manual-step triggering; treat 4xx/405 as unsupported.
- If unsupported: use claude-in-chrome (load the core tool set via one ToolSearch call; `tabs_context_mcp` first, then a new tab) on the pipeline results URL, and press **Run** on each of the three manual steps, confirming any in-page dialog Bitbucket renders.

Verify via the steps endpoint that all three left the not-run state.

If **No ticket** was chosen in Step 4 → report PR URL + pipeline URL and **finish** (deploys run; nowhere to record them).

## Step 7 — Collect the hosting URLs

Poll the steps endpoint (same background-sleep cadence) until all three deploy steps complete. Any of them `FAILED`/`ERROR`/`STOPPED` → report which env(s) failed with the pipeline URL and **end without writing any ticket links**.

For each of the three steps, fetch the log: `bb_get` `/repositories/<ws>/<repo>/pipelines/<uuid>/steps/<step_uuid>/log`. The step runs `./bin/hosting-deploy-preview.sh`, which ends with `firebase hosting:channel:deploy`; extract the channel URL it prints (an `https://…web.app` URL). One URL per environment: au, nz, us.

## Step 8 — Write the ticket test links

For each chosen ticket, set the custom fields — names verbatim:

- **AU TEST link** ← the test-au URL
- **NZ TEST link** ← the test-nz URL
- **US TEST link** ← the test-us URL

Discover the field ids from the issue's edit metadata (`getJiraIssueTypeMetaWithFields` or equivalent), then `editJiraIssue`, **replacing** any existing values. Re-read the issue to verify all three landed. If the API cannot set them, fall back to claude-in-chrome on the Jira issue page and set the three fields there.

## Step 9 — Report

PR URL, pipeline URL, the three hosting URLs, and per-ticket confirmation of the updated fields. Any partial outcome is stated exactly as it stands.
