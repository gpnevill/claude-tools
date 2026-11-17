---
name: write-solution-design
description: Produce a solution design artifact from the current working session - the solution, its architectural changes, key decisions, refactoring, and change distribution - for a reviewer to read before the code in at most six minutes. Fails in a session without a change set and the reasoning behind it. Use when the user says "write up the solution design", "produce the design artifact for this work", or invokes /write-solution-design.
---

# Write solution design

Publish an artifact that presents the design of this session's work to a reviewer ahead of the diff: the solution, the reasons for its design, and the justification for each change. It has exactly two sources: the change set, and the design record held by the session. Nothing in it is inferred from the diff alone.

## Step 1 — Check the session

Both are required:

- **Change set**: commits on the working branch that are not on its base, or uncommitted changes in the working tree.
- **Design record**: the reasons for the solution's design — decisions made and alternatives considered — in the conversation, or in plan and iteration files the session wrote or read.

Either missing → publish nothing and stop with: `write-solution-design requires a working session with a change set and the reasoning behind it; this session has no <change set | design record>.`

## Step 2 — Gather the material

1. **Base ref**: taken from the session — plan frontmatter `base`, the base confirmed by a delivery run, or the base stated by the user. Unknown → ask the user.
2. **Change set**: the changes on the branch since its merge base with `origin/<base>`, plus uncommitted changes and untracked files, with added and removed line counts per file.
3. **Design record**: the conversation, plus every plan and iteration file in the session's work dir, where one exists, re-read in full.

## Step 3 — Analyse

- **Architectural changes**: every contract or interface added or changed, and every boundary created, moved, or crossed differently — ports, adapters, events, and API, schema and data-model changes. For each: what it is, and why the solution requires it.
- **Key decisions**: decisions for which a different choice would have changed the solution's design. A decision must be materially significant and justify its inclusion in the table. For each: the question, the options considered, the choice, and the reason.
- **Refactoring**: behaviour-preserving changes to code that predates the work. For each: its location, what it enables, and the cost to the solution of omitting it.
- **Distribution**: each changed file is classified as tests, generated, refactoring, or one of the areas named by the architectural changes. Changed lines (added plus removed) are summed per class and given as rounded figures.

## Step 4 — Ask for missing justification

Ask the user to justify every architectural change, refactoring, or unclassified file that the design record does not justify. Answers are used as given. A change the user declines to justify is stated in the artifact as having no established justification.

## Step 5 — Write the artifact

Use the built-in artifact skills where they exist. Sections, in order:

1. **Title and read time** — the title begins with the ticket key, where the work has one. The read-time estimate is on the line below the title.
2. **Solution** — the problem; the solution in outline and how it resolves the problem; then each architectural change and why it is required. Diagrams are included where they aid understanding. A contract's signature is shown where the contract itself is changed.
3. **Key architectural decisions** — a table: decision, options, chosen, why.
4. **Refactoring** — where, purpose, justification, per item. None → one sentence stating so.
5. **Change distribution** — a table of changed lines per class: tests, generated, refactoring, each area.

Requirements:

- **Order**: the Solution section follows the reasoning from problem to solution, not the session's chronology. It excludes attempts, iterations, reviews, and process. Alternatives are presented in the decisions table.
- **Prose**: technical, professional, easy to read, and in the codebase's vocabulary. Objective and report-like. It is documentation, not literature. Structured for scanning, with no dense blocks of text. No trailers.
- **Read time**: at most six minutes — approximately 1000 words and two diagrams. Calculated from the final text at 200 words per minute across all visible text, including tables, plus half a minute per diagram, rounded up to the nearest half minute.

## Step 6 — Publish and report

Write the artifact to the session's work dir, or to the session scratchpad where there is no work dir, and publish it with the Artifact tool. Report the artifact URL, the file path, the read time, and every change stated as having no established justification.
