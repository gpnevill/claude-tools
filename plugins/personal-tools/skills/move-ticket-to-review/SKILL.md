---
name: move-ticket-to-review
description: Move a Jira ticket to the Review status and assign the right reviewer - chosen by the user from the ticket's reporter and commenters, or anyone else via Other. Use when the user says "move MP-1234 to review", "put this ticket in review", or invokes /move-ticket-to-review.
argument-hint: '<ticket-key>'
---

# Move ticket to review

Transition a ticket to Review and set the assignee the user actually wants — the transition auto-assigns the reporter, which is often wrong, so the user chooses.

All Jira operations use the Atlassian MCP tools (load via ToolSearch if deferred).

## Step 1 — Read the ticket

Ticket key from the argument (`[A-Z]+-\d+`); none → fail: `move-ticket-to-review requires a ticket key.`

`getJiraIssue` with comments. Collect the candidate reviewers:

- the **reporter**, and
- every **comment author**, deduplicated, excluding obvious automation/app accounts.

## Step 2 — Ask who gets assigned

Ask via `AskUserQuestion` (before transitioning, so a mid-flow abort changes nothing): who should be assigned for review?

- Options: the reporter first, labelled `(reporter)`; then commenters ordered by most recent comment. `AskUserQuestion` shows at most 4 options — if candidates exceed that, keep the reporter plus the most recent commenters and note that anyone omitted can be named via Other.
- The built-in Other covers assigning someone outside the candidate group; resolve such a name with `lookupJiraAccountId`.

## Step 3 — Transition to Review

`getTransitionsForJiraIssue` → select the transition whose target status matches Review (`/review/i`). No match → report the available transitions and stop. More than one match → ask the user which. Then `transitionJiraIssue`.

## Step 4 — Assign

**After** the transition (which auto-assigns the reporter), set the assignee to the chosen person via `editJiraIssue` — including when the choice is the reporter. Re-read the issue and verify both status and assignee; a mismatch is a failure to report, not to silently accept.

## Step 5 — Report

Ticket key, new status, final assignee.
