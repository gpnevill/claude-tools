---
name: write-solution-design
description: Produce a solution design artifact from the current working session - the solution and its architectural changes, why it is designed that way, the decisions that shaped it, the refactoring it carries, and where the line changes fall - published as a page a reviewer reads in under five minutes before the code. Fails in a session without the working to draw it from. Use when the user says "write up the solution design", "produce the design artifact for this work", or invokes /write-solution-design.
---

# Write solution design

Publish a page that takes a reviewer through the design of this session's work before they open the diff: what the solution is, why it is designed the way it is, and the justification for every change in it. The outcome is binary: either the artifact is published and its link reported, or the skill fails and publishes nothing.

The artifact draws on exactly two sources: the change set, and the design record this session holds. Nothing in it may be inferred from the diff alone — reasoning the session never established is fabricated reasoning. This skill runs in the main conversation; the session is its material and cannot be handed to a subagent.

## Step 1 — Sufficiency gate

Both must hold:

- **A change set.** The solution exists as code: commits on the working branch beyond its base, and/or uncommitted changes in the working tree.
- **A design record.** This session holds why the solution took its shape — decisions made, alternatives weighed, constraints discovered — in the conversation, or in plan and iteration files the session wrote or read.

Either missing → fail: `write-solution-design requires a working session with a change set and the reasoning behind it; this session has no <change set | design record>.` A fresh session on a branch with commits fails on the second; a planning session that produced no code fails on the first.

## Step 2 — Gather the material

1. **Base ref**: from the session — plan frontmatter `base`, a delivery run's confirmed base, or what the user stated. Unknown → ask via `AskUserQuestion`, the branch's likeliest base first.
2. **Change set**: `git fetch origin <base>`, then `git diff origin/<base>...HEAD` plus the working tree — `git diff HEAD` and untracked files. `git diff --numstat` over the same set for Step 3.
3. **Design record**: re-read every plan and iteration file in the session's work dir, if one exists, **now, in full**, even if read earlier; the conversation supplies the rest. Decisions, alternatives and constraints are taken as recorded, not as remembered.

## Step 3 — Analyse

- **Architectural changes.** Every contract or interface added or changed; every boundary created, moved, or crossed differently — ports, adapters, events, API and schema changes, data-model changes. For each: what it is, and why the solution cannot be fulfilled without it, as the record states.
- **Key decisions.** Keep a decision only if choosing otherwise would have shaped the solution differently; drop naming, file layout, formatting and ordering. For each: the question, the options considered, the choice, the reason.
- **Refactoring.** Behaviour-preserving changes to code that existed before the work. For each: where, what it enables for the solution, and why it is justified — what the solution would cost without it.
- **Distribution.** Classify each file in the numstat as tests, generated, refactoring, or one of the areas the architectural analysis names; sum the changed lines (added plus removed) per class; round to a rough figure.

## Step 4 — Ask for what the session does not hold

List every architectural change, refactoring, or unattributed file whose justification the record lacks. None → skip this step. Otherwise ask for all of it in one batched `AskUserQuestion` call, naming each change concretely and offering the likely reasons as options. Answers go in as the user gives them; whatever the user declines is stated in the artifact as having no established justification — never guessed.

## Step 5 — Write the artifact

Load the `artifact-design` skill before writing, and `artifact-diagramming` before drawing. Content, in this order:

1. **Title and read time** — the estimate on the first line below the title.
2. **Solution** — the design journey: the shape of the solution; the constraints that bound it; then each architectural change and why it is required. Diagrams where illustrative and valuable. A contract's signature appears where the contract itself is changed.
3. **Key architectural decisions** — a table: decision, options, chosen, why.
4. **Refactoring** — where, purpose, justification, per item. None → one sentence saying so.
5. **Change distribution** — a table of changed lines per class from Step 3: tests, generated, refactoring, each area.

Rules:

- The journey is the reasoning path from problem to solution, never the chronology of the session: no narration of attempts, iterations, reviews, or process. Alternatives live in the decisions table.
- Technical, professional prose in the codebase's own vocabulary. No trailers.
- **Read time**: 200 words per minute over every word the reader sees, tables included, plus half a minute per diagram, rounded up to the half minute. The budget is five minutes — roughly 800 words alongside two diagrams. Over → cut until under. The estimate is recomputed from the final text, never guessed.

Acceptance test: a reviewer who reads only this page can state what the solution is, why each major change exists, and which part of the diff each section accounts for.

## Step 6 — Publish and report

Write the HTML to `<work dir>/design.html` where the session has a work dir, else the session scratchpad, and publish it with the Artifact tool. Report the artifact URL, the file path, the read time, and every change the artifact states as having no established justification.

## Failure convention

Wherever this skill says fail: emit one clear error explaining what went wrong and stop. Nothing is published on any failure path.
