---
name: check-failure-report
description: Walk the user through every failure in a failure report, asking for each whether it is a valid failure that needs resolving or is fine as-is, then narrow the report to only the confirmed failures - deleting the report file entirely if none remain. Use when a failure report needs user confirmation, or when the user invokes /check-failure-report.
argument-hint: '<failure-report-path>'
---

# Check failure report

Reduce a failure report to only the failures the user confirms are real. The report's author had no say in what matters — the user does. The outcome is binary per item: it stays verbatim, or it goes.

## Step 1 — Parse

Read the report at the given path (no path → ask for it and stop until provided). Extract every itemized failure (`## F1`, `## F2`, … — or, if the report uses another itemization, each discernible item). Zero items → state that the report contains nothing checkable and stop, leaving the file untouched.

## Step 2 — Ask about every single failure

For **each item, without exception**, ask the user via `AskUserQuestion` — batched up to 4 questions per call — one question per failure, quoting the item's identifier and its substance (Where / What / Why, condensed but faithful) so the user judges the actual finding, not a paraphrase. Options, exactly two:

- **Valid — needs resolving** — the failure is real and must be fixed.
- **Not a failure — drop it** — the finding is acceptable as-is and leaves the report.

Any nuance the user adds via Other or notes (e.g. "valid, but only the naming part") is honored: keep the item and append the user's qualification to it verbatim, attributed to the user.

Never advocate. The questions present the findings; they do not defend them.

## Step 3 — Narrow the report

- **Some items confirmed** → rewrite the file in place containing only the confirmed items, verbatim and with their original identifiers (no renumbering), plus any user qualifications from Step 2. Nothing else in the file changes.
- **No items confirmed** → delete the file.

## Step 4 — Report

State the outcome: which item identifiers were kept and which were dropped — or that all items were dropped and the report file was removed.
