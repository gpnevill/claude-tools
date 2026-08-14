---
name: deliver
description: Orchestrate the full delivery lifecycle for a ticket or described piece of work - verify the working environment, exhaustive interrogation-based planning, implementation, review, UI testing, iteration loops, ship to test, PR with description, and moving tickets to review. Use when the user says "deliver M2X-1234", "run the full pipeline on this", or invokes /deliver.
argument-hint: '<ticket-key and/or work description>'
---

# Deliver

Run the delivery lifecycle end to end. This skill is a state machine: it sequences skills and agents, owns the file-path conventions they hand off through, and enforces the gates between phases. It holds no domain knowledge and makes no implementation decisions.

## Non-negotiable mechanics

- **Handoffs are paths, never content.** Every plan, report, and request is a file; agents receive absolute paths and read the canonical bytes. Never paraphrase, summarize, or inline a plan or report into an agent prompt.
- **Fresh agents, always.** Every launch of an implementer, reviewer, iterator, or tester is a new agent. Never continue a previous one, never reuse its context.
- **Artifact numbering**: `NN` is the next unused zero-padded integer per prefix in the work dir (`gap-01.md`, `review-01.md`, `iteration-01.md`, `test-01.md`, …). Numbers are never reused, even for discarded rounds.
- **Undo** means, in the worktree: `git reset --hard <start ref>` then `git clean -fd`, against the start ref Phase 0 recorded. It erases everything this run produced.
- **Escalation valve**: if substantially the same failure survives three rounds of any loop, stop looping and put the situation to the user instead of grinding silently.

## Conventions

- **KEY**: the uppercase ticket key; with no ticket, a short kebab slug derived from the work description.
- **Work dir**: `~/.claude/work/<KEY>/`.
- **Plan set**: `plan.md` plus every approved `iteration-NN.md`, in creation order. Precedence: iteration plans supersede the base plan; later iteration plans supersede earlier ones. Every reviewer and iterator receives the full set with this precedence stated.

## Phase 0 — Setup

1. Inputs: a ticket key (`[A-Z]+-\d+`) and/or a description of required work. Neither → ask and stop until provided.
2. Read the environment — branch (`git rev-parse --abbrev-ref HEAD`), worktree (`--show-toplevel`), start ref (`git rev-parse HEAD`) — and match it against the inputs: a ticket key must appear verbatim in the branch name; description-only work is a judgement on the branch slug, ambiguous counting as a mismatch. All subsequent phases operate in that worktree.
3. On a mismatch: report that the working environment does not appear to be configured for this task, and ask via `AskUserQuestion` — **Exit**, for the user to configure it, or **Continue** on the current branch and worktree. Exit → stop, nothing written. Matched or not, flag a branch prefix outside `feat/`/`fix/`.
4. Derive the base — `git fetch origin master release`, then `mbm=$(git merge-base origin/master <branch>)` against `mbr=$(git merge-base origin/release <branch>)`: equal → `master`; `mbm` an ancestor of `mbr` (`--is-ancestor`) → `release`; descended from neither → undetermined — and confirm it via `AskUserQuestion`, the detected value first, the other of master/release second, anything else via the built-in Other (undetermined → ask openly). The confirmed answer is the `<base>` every later phase uses.
5. Create the work dir. Write the user's initial request **verbatim** to `request.md` — before anything else happens, and never edited afterwards.

## Phase 1 — Plan

Invoke `/plan-work` with the ticket key and/or description and `--work-dir <work dir>`. Proceed only when `plan.md` exists with `status: approved`. Record `base`, `branch`, and `worktree` in its frontmatter.

## Phase 2 — Implement

Launch a fresh `work-implementer` agent with exactly: the worktree path, the plan-set paths with precedence, and a `gap-NN.md` output path.

**On `GAP`**:

1. Undo.
2. Invoke `/plan-work` with `--work-dir <work dir> --gap <gap path>`; wait for the re-approved plan.
3. Draft concrete improvement suggestions for `~/.claude/skills/plan-work/SKILL.md` — what interrogation would have surfaced these decisions up front — and present them to the user. Apply only what the user approves; never edit the skill unprompted.
4. Launch a fresh implementer with the improved plan. Repeat this phase until an implementer reports `DONE`.

## Phase 3 — Review loop

Each round `N`:

1. Launch a fresh `work-reviewer` with exactly: the worktree path, `origin/<base>`, the plan-set paths with precedence, and a `review-NN.md` output path.
2. **All four checks PASS** → Phase 4.
3. **Any `plan-conformance` failure** (regardless of other categories):
   - Undo.
   - Tighten the plan yourself — no interrogator: edit `plan.md` (and affected iteration plans) so the deviated decisions are stated strictly enough that the deviation cannot recur. Tightening adds precision to decided things; it never adds new decisions — a missing decision is a gap, and belongs to Phase 2's gap route.
   - Draft improvement suggestions for `~/.claude/skills/plan-work/SKILL.md` and/or `~/.claude/agents/work-implementer.md` — non-conformance should be impossible if those two are well-defined — and present them to the user; apply only on approval.
   - Launch a fresh implementer (Phase 2 discipline), then restart this loop.
4. **Other failures** (`quality` / `correctness` / `build`):
   - Invoke `/check-failure-report` on `review-NN.md`.
   - Report deleted → treat this round as PASS → Phase 4.
   - Report remains → invoke `/plan-iteration` with the report, work dir, plan set, and worktree → launch a fresh `work-iterator` with exactly: the worktree path, the checked report path, the new iteration-plan path, the prior plan-set paths, the precedence rule, and a `gap-NN.md` output path. An iterator `GAP` is handled like Phase 2's, with `/plan-iteration` re-run in place of `/plan-work`. Then a new round of this loop.

## Phase 4 — Test loop

Ask via `AskUserQuestion` whether to run the test loop. Options: **Yes** and **No**. **No** → Phase 5.

Each round:

1. Launch a fresh `work-tester` with **only**: the ticket key (or, for no-ticket work, the `request.md` path), the worktree path, and a `test-NN.md` output path. Nothing else — no plans, no reports, no history. The tester's isolation is the point.
2. `PASS` or `NOT APPLICABLE` → Phase 5.
3. `FAIL` → invoke `/check-failure-report` on `test-NN.md`.
   - Report deleted → relaunch a fresh tester (the failures were declared non-failures, but the pass must still be earned).
   - Report remains → `/plan-iteration` on it → fresh `work-iterator` → then the **full Phase 3 review loop until it passes** → then a fresh tester. Fresh checker, planner, iterator, reviewer, and tester every time around.

## Phase 5 — Commit

Stage and commit all work in the worktree with conventional-commit message(s); behavioral changes carry the ticket key (`feat: [M2X-1234] …`).

## Phase 6 — User confirmation

Ask via `AskUserQuestion` whether to continue to shipping. Options: **Continue**; feedback or changes via the built-in Other. Anything other than continue is acted on as the user directs, then this phase repeats.

## Phase 7 — Ship to test

Invoke `/ship-to-test` with the branch name as its argument. Its outcome is its own; a failure there is reported and ends the run as that skill defines.

## Phase 8 — PR

1. Invoke `/push-feat-or-fix-branch` (current branch). Capture the PR id from its report.
2. Invoke `/write-pr-description <PR id>`.

## Phase 9 — Tickets to review

Ask via `AskUserQuestion`: move any tickets to review? Options: the original ticket key (when one exists) and **None**; other ticket(s) via the built-in Other. For each chosen ticket, invoke `/move-ticket-to-review <ticket>`.

## Final report

Branch, worktree, PR URL, shipped-to-test outcome, hosting preview URLs if produced, tickets moved to review, and the work-dir path holding the full plan and report trail.
