---
name: plan-iteration
description: Interrogate the user exhaustively about how to resolve a confirmed failure report, then write a self-contained, approved iteration plan that supersedes prior plans where it says so. Scope ranges from extracting one literal to overhauling the entire solution. Use when a failure report needs a resolution plan, or when the user invokes /plan-iteration.
argument-hint: '<failure-report-path> [--work-dir <dir>] [--plans <plan-path>...] [--worktree <path>] [--out <iteration-plan-path>]'
---

# Plan iteration

Produce a plan for resolving the failures in a confirmed failure report — a plan so complete that **the user could not possibly want the failures resolved any other way**. Every resolution decision is extracted from the user by interrogation; none is assumed.

You are **not tied to the prior plans**. The prior plans record what was decided before these failures were known; the failures may prove those decisions wrong. The only obligation is that the iteration plan emerges from full interrogation and collaboration with the user — and nothing prevents that collaboration from overhauling the entire solution.

This skill runs in the main conversation; the interrogation cannot be delegated.

## Step 0 — Inputs

- **Failure-report path** (required): a report whose failures the user has already confirmed as real.
- **Work dir**: from `--work-dir`, else the directory containing the failure report.
- **Prior plan path(s)**: from `--plans`, else `plan.md` and every `iteration-*.md` in the work dir, in that order.
- **Worktree path**: from `--worktree`, else the `worktree` recorded in the base plan's frontmatter.
- **Output path**: from `--out`, else the next free `<work-dir>/iteration-NN.md` (`NN` zero-padded, starting `01`).

Missing failure report → ask for it and stop until provided.

## Step 1 — Understand the failures in context

Read in full: the failure report, every prior plan (noting the decisions currently in force), and the actual state of the failing code in the worktree. For each failure item, understand precisely what fails and which prior decisions, if any, produced it.

## Step 2 — Interrogation

For **every failure item**, enumerate the full space of resolutions — from the minimal local fix to a total overhaul of the solution — and put the choice to the user.

Calibration of the range:

- A magic-literal failure warrants asking how and where the value is extracted: the constant's name, which file and scope owns it, whether it is derived rather than declared.
- A naming failure warrants candidate names and where each ripples.
- A correctness or design failure may warrant reopening the approach itself — alternative designs are presented as first-class options, including designs that discard the current implementation entirely.

Protocol:

- Ask via `AskUserQuestion`, batched up to 4 per call, in rounds. Options are concrete — real names, real file paths, real designs — with trade-offs, the recommended option first where one exists, free text via the built-in Other.
- One question per decision; **a decision that was not asked is not a decision** — no silent defaults, no "obvious" resolutions adopted unconfirmed.
- Answers breed follow-ups: descend into every consequence — a resolution that touches a prior decision raises the question of that decision explicitly.
- Where two failures interact, ask how the combined resolution should look rather than resolving each blindly.

**Exhaustion criterion — the only exit**: interrogation ends when you can no longer formulate a question whose answer would change any part of the iteration plan. Before ending, try to break it: for each resolution, ask yourself what a differing implementer could still choose differently; any answer becomes another round.

## Step 3 — Write the iteration plan

Write the output file:

```
---
status: draft
key: <work key>
resolves: <failure-report filename>
---
```

Body sections, all mandatory:

- **Failures addressed** — each `F<n>` from the report mapped to the decisions that resolve it.
- **Decision log** — numbered `D1`, `D2`, …: every question asked, the user's answer, the resulting decision.
- **Supersedes** — an explicit list of the prior-plan decisions and steps this plan overrides, each named (plan file + decision/step). Prior decisions not listed here remain in force. An empty section means the plan is purely additive.
- **Implementation steps** — ordered, concrete, naming files and changes, including any removal of previously written code.
- **Verification criteria** — what proves each failure resolved.

**Acceptance test**: a fresh engineer holding only the prior plans, this file, and the failure report — and zero conversation history — must be able to apply it and could not produce a different result than the one the user wants.

## Step 4 — Approval gate

Present the iteration plan in full, then ask via `AskUserQuestion` whether it is approved or needs changes. Any change → apply, re-present, ask again. Only on explicit approval set `status: approved`. Never end with the plan in `draft`; the plan path is the completion statement.
