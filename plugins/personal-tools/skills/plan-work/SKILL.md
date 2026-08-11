---
name: plan-work
description: Interrogate the user exhaustively about a Jira ticket or a described piece of work until every implementation decision is made, then write a self-contained, approved plan file. Also resolves gap reports against an existing plan. Use when the user says "plan MP-1234", "interrogate me about this ticket", "plan this work", or invokes /plan-work.
argument-hint: '<ticket-key and/or work description> [--work-dir <dir>] [--gap <gap-report-path>]'
---

# Plan work

Produce a plan so complete that **the user could not possibly want the work implemented any other way**. The plan is extracted from the user by interrogation — never assumed, never defaulted, never inferred silently. The written plan file is the sole output that matters: it must be implementable by a fresh engineer with zero knowledge of this conversation.

This skill runs in the main conversation; the interrogation cannot be delegated.

## Step 0 — Inputs and work directory

- **Subject**: a ticket key (`[A-Z]+-\d+`), a prose description of required work, or both. Neither present → ask for one and stop until provided.
- **Work dir**: from `--work-dir` if given; otherwise `~/.claude/work/<KEY>/` where KEY is the uppercase ticket key, or — with no ticket — a short kebab slug derived from the description. Create it if absent.
- **Gap mode**: `--gap <path>` switches to Gap mode (see the final section).

## Step 1 — Ingest the subject

- Ticket given: read it in full via the Atlassian MCP tools (`getJiraIssue` with description, acceptance criteria, comments, linked issues; load via ToolSearch if deferred). Snapshot it verbatim to `<work-dir>/ticket.md` — fields as-is, no paraphrase. Note attachments by name.
- Description given: ensure `<work-dir>/request.md` holds it verbatim; write it if it does not already exist. Never edit an existing `request.md`.

## Step 2 — Research before asking

Ground every question in reality. Explore the codebase — fan out read-only Explore agents where the surface is broad — to find:

- the code the work will touch, and its current shape;
- existing patterns, components, and utilities a solution could build on;
- prior art: how comparable features were done here;
- constraints that eliminate options (so they are presented as findings, not questions).

Research informs the questions; it never substitutes for asking. A constraint discovered in code may close a question — a preference never may.

## Step 3 — Interrogation

This is the heart of the skill. Enumerate **every axis on which the implementation could differ**, and put every open axis to the user. Axes to work through (not exhaustive — add any the subject raises):

- Scope: exactly what is in, and — just as explicitly — what is out.
- Data: model shape, source of truth, persistence, migration/backfill, compatibility with existing data.
- Placement: which app/package/module owns each piece; new code vs. extension of existing code.
- Behavior: every user-visible interaction, including empty, loading, error, permission-denied, and concurrent-edit states.
- UI: layout, components used, copy, validation timing and messages.
- Naming: domain vocabulary for new entities, fields, routes, and files.
- Edge cases: enumerate them one by one and ask what each should do.
- Failure handling: what may fail, what the user sees, what is retried or surfaced.
- Rollout: feature flags, environment differences, ordering constraints.
- Testing: what gets automated tests, at which level.

Protocol:

- Ask via `AskUserQuestion`, batched up to 4 questions per call, in rounds. Each question offers concrete options grounded in the research — real file paths, real components, real trade-offs — with a recommended option first where one exists. Free-form answers arrive via the built-in Other.
- One question per decision. Compound questions hide decisions.
- **A decision that was not asked is not a decision.** Never adopt a default silently; if an answer seems obvious, ask it as a confirmation with the obvious option first.
- Answers breed follow-ups: descend into every consequence of every answer until that thread is exhausted.
- Where research eliminated options, state the finding and confirm rather than ask open-endedly.

**Exhaustion criterion — the only exit**: interrogation ends when you can no longer formulate a question whose answer would change any part of the plan. Before ending, actively try to break the plan: for each section, ask yourself what a differing implementer could still choose differently; any answer becomes another round of questions.

## Step 4 — Write the plan

Write `<work-dir>/plan.md`:

```
---
status: draft
key: <KEY>
ticket: <ticket key or none>
base: <base branch, if known>
branch: <working branch, if known>
worktree: <worktree path, if known>
---
```

Body sections, all mandatory:

- **Context** — the problem, why the work exists, intended outcome.
- **Scope** — what will be built.
- **Non-goals** — what deliberately will not, each with the reason decided.
- **Decision log** — numbered `D1`, `D2`, …: every question asked, the user's answer, and the resulting decision. This section is what makes the interrogation lossless; nothing decided may live only in conversation.
- **Implementation steps** — ordered, concrete, naming the files to create or modify and what each change is.
- **Verification criteria** — the observable behaviors that prove the work is done.

**Acceptance test for the plan itself**: a fresh engineer with zero conversation history must be able to implement from this file alone, and could not produce a different implementation than the one the user wants. If any step or decision requires the conversation to interpret, the plan is not done.

## Step 5 — Approval gate

Present the plan content to the user in full, then ask via `AskUserQuestion` whether it is approved or needs changes. Any change → apply, re-present, ask again. Only on explicit approval set `status: approved` in the frontmatter. Never end the skill with the plan in `draft`; the plan path is the completion statement.

## Gap mode (`--gap <path>`)

An implementer found the plan incomplete. The full work state: read the existing `plan.md` and the gap report in full.

- Each gap item is a new interrogation thread, run with the full Step 3 protocol.
- A gap may invalidate earlier decisions: reopen anything it undermines — there is no obligation to preserve the original plan beyond the decisions that still stand.
- Rewrite `plan.md` as a complete, coherent whole (not a patch): extend the decision log with new numbered decisions, revise superseded ones in place, and update implementation steps and verification criteria accordingly.
- Reset `status: draft`, then run the Step 5 approval gate again.
