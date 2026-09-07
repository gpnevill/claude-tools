---
name: progress-ticket
description: Progress a semicolon-separated list of tickets through the state tree in ticket-workflow.yaml - per ticket, derive its states from live evidence and its log, run the actions that apply there, offload the long-lived ones to their own Herdr sessions, and log everything to TICKETS.md. Use when the user says "progress M2X-1234; M2X-1235", "move these tickets along", or invokes /progress-ticket.
argument-hint: '<ticket-key>; <ticket-key>; …'
---

# Progress ticket

Advance each of a list of tickets by one turn of the state tree in `ticket-workflow.yaml`. This skill derives states, runs actions and logs; what any state means is the workflow file's to say.

All Jira operations use the Atlassian MCP tools (load via ToolSearch if deferred).

## Non-negotiable mechanics

- **This session never moves.** Never `cd` into a worktree, never adopt one as this session's working directory, never focus a workspace, tab or agent it opens.
- **Tickets are independent.** Whatever stops one is reported against that one alone; the rest still run.
- **An action that cannot be carried out is a failure to report**, never something to approximate with a different action.

## Step 1 — Preconditions

```bash
test "${HERDR_ENV:-}" = 1
git rev-parse --show-toplevel
```

The first non-zero → this session cannot open workspaces or start agents; stop and say so. The second → not inside a git repository; stop. Worktrees are created for this repository.

## Step 2 — Parse and load

Split the argument on `;`, trim, drop the empties. Each remaining segment is one **ticket key**. No keys → ask for them and stop until given. List them back.

Read `ticket-workflow.yaml` and `TICKETS.md`, both in user scope in `CLAUDE_CONFIG_DIR`, in full and now — their current content is the specification, never memory of it. Create `TICKETS.md` if missing, as a bare `# Tickets` heading.

Every ticket without an entry gets one:

```markdown
## M2X-1234 — Ticket summary

- **Link**: <ticket URL>

### Log

- 2026-09-06T09:12:03+00:00 — Entry created.
```

Timestamps come from `date -Iseconds`.

## Step 3 — Gather what is shared, and choose the tickets

My own account identity, from the Atlassian server's current-user tool — the tree's `when` tests are written in the first person and cannot be evaluated without it — and `herdr worktree list --cwd "$PWD"`, re-read in 4.6 where an offloaded action needs a current one. Both hold for every ticket and are read once.

Then one `AskUserQuestion` question per ticket, batched up to four questions per call, offering **Progress** and **Skip**. A skipped ticket is logged as skipped and takes no further part in the run.

## Step 4 — Per ticket, in order

### 4.1 Gather the ticket's evidence

`getJiraIssue` with comments and linked issues, and the ticket's own log.

### 4.2 Derive the states

Walk the tree from the top, evaluating **every** state's `when` at each level and descending into each one that holds. Siblings need not exclude each other — a ticket matching two of them is at both, and every branch entered is active. A branch stops where no child holds, and the root is a valid resting place.

Log each active state as a `/`-joined path of ids — `assigned-to-me / ready-to-work`, or `(root)` where no top-level state held — **every time it is derived**. A `when` may test what earlier runs derived.

Straight to 4.8 only where every active state carries `closes_workflow: true`; one live state alongside means the workflow is not over, and the closing one contributes nothing. Otherwise, follow the `instruction` of every active state that has one.

### 4.3 Collect the applicable actions

The file's top-level `actions` first, then every active state's, in the order the tree declares them — a state active on two branches contributes its actions once. Drop any whose `when` does not hold. None left → log the derivation and go to 4.7.

### 4.4 Run them, in order

Four passes, each of them top-level down:

1. Every action with `auto_start: true` and `auto_commit: true`.
2. Every action with `auto_start: true` and `auto_commit: false`.
3. Ask about the rest — one `AskUserQuestion` question per action, batched up to four questions per call, each offering **Yes**, **No** and **Skip**, with anything else via the built-in Other. **No** and **Skip** are logged as a decision taken against the action, where a later `when` can see it.
4. Every action answered yes.

An action's `synchronous` field decides how it is carried out, in every pass alike — `true` in this session (4.5), `false` in a session of its own (4.6). A second progression from 4.7 runs these passes exactly as written, save that every action counts as `auto_start: false` — nothing repeats itself unasked.

### 4.5 Carrying out a synchronous action

Carry out `instruction`. With `auto_commit: false`, show what is about to land — the comment text, the field change, the message — and ask before it lands, applying whatever the user changes. With `auto_commit: true` it lands unasked. Then re-read what was written and confirm it landed; a mismatch is a failure to report, not to silently accept. Log the action and its outcome.

### 4.6 Carrying out an offloaded action

Locate the ticket's worktree by matching the key case-insensitively against each open worktree's `branch` and `path`. No worktree → invoke `/setup-worktree <KEY>`, which is the only permitted way to create one; take the worktree path from its report.

Open a tab for the action and take its pane from `.result.root_pane.pane_id`:

```bash
herdr tab create --workspace <ws-id> --cwd <worktree path> --label <action id> --no-focus
```

One tab per action, all in the one workspace. The workspace id of a worktree this skill did not just create comes from `herdr worktree list --cwd "$PWD"`.

Name the agent for the key and the action — `M2X-1234` + `investigate` → `m2x-1234-investigate` — replacing other characters with `-`, trimming to 32 characters, and suffixing `-2`, `-3`, … against `herdr agent list` until unique:

```bash
herdr agent start <name> --kind claude --pane <pane id> --timeout 120000 -- --permission-mode auto
herdr pane read <pane id> --source detection --lines 40
```

Offloaded sessions always run in auto permission mode, whatever the action's `auto_start` and `auto_commit` say. Expect Claude's empty input prompt in that read — a trust question or any other confirmation stops this action and is reported, because the next command's Enter would answer that instead.

```bash
herdr agent prompt <name> '<prompt>'
```

The action's `prompt` is used **verbatim** when it has one; otherwise write a brief one from `instruction`, adding no detail the workflow did not state. Append to it, in both cases:

- with `auto_commit: false`, that nothing is to be posted, sent or changed until the user has seen it and approved it;
- that when the work is done, one line is to be appended to the `## <KEY>` entry's `### Log` in `$CLAUDE_CONFIG_DIR/TICKETS.md` — timestamped with `date -Iseconds`, saying whether `<action id>` completed or failed and, on a failure, why — and that nothing else in that file is to change.

Quote the whole prompt as one single-quoted argument, escaping any embedded single quote. Never pass `--wait`. Log the action started, with its workspace, tab and agent.

### 4.7 Progress again, or done

Every applicable action having been run, ask **Progress again** or **Done with this ticket**. Progress again → back to 4.2, deriving afresh, since what has just run may have moved the ticket. A derivation that leaves nothing to run says so and moves on.

### 4.8 Close out

Every active state carrying `closes_workflow: true` ends the ticket's workflow and offers nothing else. Show what would be lost first:

```bash
git -C <worktree path> status --short
git -C <worktree path> log --branches --not --remotes --oneline
```

Ask via `AskUserQuestion` to close out or leave it open, with that summary in the question. On yes, `herdr worktree remove --workspace <ws-id>`, asking again before `--force` on a refusal over uncommitted changes. Then move the entry, log and all, to `TICKETS_ARCHIVE.md` alongside `TICKETS.md`, appending a closing line, and delete it from `TICKETS.md`.

## Step 5 — Report

Open with the repository this run was scoped to. Then per ticket: the derived states, what was run or offloaded, the workspace, tab and agent of anything offloaded, and the log lines written — or that it was skipped, or the step that stopped it and why. Close by naming the sessions now running.
