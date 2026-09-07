---
name: orchestrate-ticket-lifecycles
description: Read every open ticket assigned to me or reported by me and run /progress-ticket over the whole set. Use when the user says "progress all my tickets", "run my ticket lifecycles", or invokes /orchestrate-ticket-lifecycles.
---

# Orchestrate ticket lifecycles

Find every open ticket that is mine and hand the set to `/progress-ticket`.

All Jira operations use the Atlassian MCP tools (load via ToolSearch if deferred).

## Step 1 — Find the tickets

Search by JQL:

```
(assignee = currentUser() OR reporter = currentUser()) AND status != Done AND status != Withdrawn ORDER BY updated DESC
```

A failed search is a stop — report it and **end**.

## Step 2 — List them back

The keys with their summary, status and assignee, and how many. None → say so and **end**.

## Step 3 — Hand them over

Invoke `/progress-ticket` with every key found, joined by `; `, in the order the search returned them. Each ticket is offered a skip there, so the whole set is passed on unfiltered.

## Step 4 — Report

`/progress-ticket`'s report is the report. Add the count swept and the query used.
