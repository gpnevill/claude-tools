---
name: create-ticket
description: Create a Jira ticket for an issue - written from what the session already established about it, or investigated first when it has not been established. Use when the user says "raise a ticket for this issue", "create a Jira ticket for [issue]", or invokes /create-ticket.
argument-hint: '[issue description]'
---

# Create ticket

File an issue as a Jira ticket that states **the problem and what "done" looks like**, and nothing else. How the issue gets resolved is the assignee's concern; a ticket carrying a proposed fix has taken a decision that was not the reporter's to take.

All Jira operations use the Atlassian MCP tools (load via ToolSearch if deferred).

One invocation files one ticket. If the session holds several distinct issues and the argument does not single one out, ask which before anything else.

## Step 1 — Subject and evidence

The subject is the issue described in the argument, or — with no argument, or an argument that only points ("for this issue", "for the issue about X") — the issue the session has been discussing. Neither present → ask which issue and stop.

The argument may reference existing tickets — `link to M2X-1234 with "is blocked by"`, `write it similar to M2X-5678`. Read each with `getJiraIssue` and take from the phrasing what it is for. A reference is not a subject; that still comes from the session, or from asking.

Investigation is required **unless both** of these already hold:

- a specific symptom or requirement is established — what actually happens and where it surfaces, not a suspicion; and
- its cause and extent were established in this session **from evidence** — code read, output seen, a reproduction — rather than assumed.

The user's phrasing does not decide this. `/create-ticket for this issue` after a passing mention goes to Step 2; a session that has just diagnosed and discussed the issue in detail goes straight to Step 3. Where only part of it is established, investigate the unestablished part — whether it reproduces, how far it reaches, etc....

## Step 2 — Investigate (when required)

Read-only. Establish, from evidence:

- **that the issue is real** — reproduce it, or read the path that produces it;
- **what happens and what should instead**;
- **its extent** — where it surfaces and who it affects, which is what sets the ticket's scope.

Fix nothing, edit nothing, commit nothing — including a one-line fix that is obviously right. This skill reports and files.

**A negative result is a result.** The issue may not reproduce, may already be fixed, may be intended behaviour, or may be materially narrower or broader than described. Say so and stop — no ticket — unless what you found is itself worth filing, in which case the ticket is about that and the user is told the subject changed.

Report the findings in chat: what was verified, how, the extent, and anything that came back negative. Then ask via `AskUserQuestion` whether to go ahead with a ticket on those findings, and stop unless the answer is yes.

## Step 3 — Draft the ticket

**Summary**: one plain, specific line — no ticket-key prefix, no `Bug:` prefix; the issue type carries that.

**Body sections, in this order:**

- **Background** (or **Problem**, whichever fits the subject — a requirement or a defect) — required, first. The requirement or the observed behaviour that calls for the work: what happens now, what is expected instead, and who or what it affects. A few sentences or a short bullet list.
- **Detailed requirements** — optional, and genuinely rare. Only for product-level specifics too granular for acceptance criteria: the validation expected on every field of a new API, the exact data dependencies that must trigger a reactive process. Where present, the acceptance criteria reference it rather than restate it. Its absence is the normal case; never manufacture one.
- **Acceptance criteria** — required, last. Observable outcomes a reviewer can check, each a single testable statement about resulting behaviour, never about the means.

The investigation's findings **shape** the ticket — they set its scope and let the problem be stated precisely — but they do not enter it. Everything code-specific that the investigation produced stays in the chat report.

The draft contains none of:

- file paths, or function, class, component, table or variable names;
- code blocks or snippets;
- a proposed fix, an implementation approach, or an effort estimate;
- narration of the investigation, or of this session.

Plain headings and short bullets only, minimal tables if valuable, so the body survives whatever the description field accepts.

A relationship to another ticket is carried by its link, not by prose in the body.

## Step 4 — Project, issue type, and fields

- **Project**: candidates from the tickets referenced in the argument, then ticket keys named elsewhere in this session, then the current branch name, then `getVisibleJiraProjects`.
- **Issue type**: from the project's create metadata (`getJiraProjectIssueTypesMetadata` or equivalent) — a defect in existing behaviour is a Bug, a new requirement is the project's task/story type.

Ask both in a single `AskUserQuestion` call, inferred option first; skip an axis only when its single candidate is certain.

If the create metadata for that project and issue type exposes an acceptance-criteria field, the criteria go **there** and the description ends at the background (and detailed requirements); otherwise they are the description's last section.

**Assignee and priority**: a second `AskUserQuestion` call once the project is settled. Each offers a suggestion first then **Jira default**; the built-in Other takes any other value, resolved via `lookupJiraAccountId` or the project's priority scheme.

Every remaining field is left to Jira's defaults. Reporter, labels, sprint, epic and components are not this skill's to guess.

## Step 5 — Approval gate

Present the draft in full — summary, project, issue type, assignee, priority, the links to be filed, and every section verbatim as it will be filed — then ask via `AskUserQuestion` to create or amend. Amend → apply, re-present, ask again. Create only on an explicit yes.

This gate is separate from Step 2's and stands in every case — including one that needed no investigation, and one whose findings gate already returned yes: a ticket is visible to other people, and the draft is the user's only chance to see it before they are.

## Step 6 — Create and verify

`createJiraIssue`, then `createJiraIssueLink` where specified. Re-read with `getJiraIssue` and check the summary, sections, acceptance criteria, assignee, priority and links all landed where they were meant to. Anything dropped, mangled or refused is fixed with `editJiraIssue` and re-verified, or reported exactly as it stands — never silently accepted.

## Step 7 — Report

Ticket key and URL, project and issue type, assignee, priority, the links created, the sections it holds, and where the acceptance criteria went. State plainly anything left for the user to correct in Jira.
