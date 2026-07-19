---
name: implement-epic
description: >
  Execute an epic of tickets with blocking edges in dependency-order waves on an
  epic branch: parallel /implement per ticket in worktrees (each with its own
  /code-review), serial merges, a report-only deep review suite over the whole
  epic, then one merge to the default branch. Use when a spec has been broken
  into tickets by /to-tickets and the user wants the whole epic driven to done.
argument-hint: "The epic to execute (tracker reference or URL)"
disable-model-invocation: true
---

# Implement Epic

Drive an **epic** — a set of tickets carrying `blocked_by` edges — from its
first ticket to a single review-clean merge into the default branch.

You are the **orchestrator**. You spawn subagents and integrate their work; you
do not implement tickets yourself.

## Inputs and precedence

- The **spec** — wins on every question of *what* to build and its invariants.
- The **epic** and its tickets — the work items and their dependency graph.
- The host repo's **contributing guide** — wins on repo specifics: validation
  gate contents, tracker, commit conventions, writing standards.
- **This skill** — wins on *process* only.

Conflicts: the spec is right about behavior. Fix the process, or raise the spec
on the ticket — never silently deviate.

## Prerequisites

These skills must be installed; verify at preflight and **stop and escalate**
if any are missing:

- `/implement` — per-ticket build; includes its own `/code-review`
- `/code-review`
- `/caveman` — communication style
- `/ponytail` — coding style, used in **ultra** mode
- The deep suite: `/thermo-nuclear-code-quality-review`, `/ponytail-review`,
  `/deslop`, `/improve-codebase-architecture`, `/simplify`

Install any missing skill with `npx skills add`:

```bash
npx skills add https://github.com/mattpocock/skills --skill implement
npx skills add https://github.com/mattpocock/skills --skill code-review
npx skills add https://github.com/mattpocock/skills --skill improve-codebase-architecture
npx skills add https://github.com/juliusbrussee/caveman --skill caveman
npx skills add https://github.com/dietrichgebert/ponytail --skill ponytail
npx skills add https://github.com/dietrichgebert/ponytail --skill ponytail-review
npx skills add https://github.com/cursor/plugins --skill thermo-nuclear-code-quality-review
npx skills add https://github.com/cursor/plugins --skill deslop
npx skills add https://github.com/brianlovin/claude-config --skill simplify
```

## Style

- **Code** — every agent that writes code (implementers, epic-gate fixers)
  works in `/ponytail ultra` mode: the laziest solution that actually works;
  minimal, shortest, no speculative abstraction.
- **Communication** — all agents and subagents speak `/caveman` in inter-agent
  messages and progress notes. **Durable artifacts** — commit messages, PR
  descriptions, ticket comments, docs — follow the host repo's writing
  standards instead.

## Shape

```text
preflight:  verify skills + tracker + green gate → branch epic/<name> off default
per wave:   fan out worktrees (parallel) → /implement each (incl. its /code-review) → merge serially into epic → sync default into epic
epic gate:  deep suite (parallel, report-only) over default...epic → fix serially → re-run flagged only → ≤3 rounds
land:       one epic → default PR → close out
```

## Ticket state

**The tickets are the state machine.** Every lifecycle transition is recorded
on the ticket itself, using the tracker's native status, labels, and comments —
never in orchestrator memory or side files. A fresh orchestrator, or a human,
must be able to reconstruct the epic's exact state from the tracker plus git
alone.

Record, at minimum:

- **In flight** — set when the ticket's worktree/branch is created; record the
  branch name on the ticket.
- **Attempts** — note each failed attempt on the ticket; the 2-attempt HITL
  rule reads this count from the ticket, not from memory.
- **Blocked on human** — the question/failure comment plus the blocked marker.
- **Done** — the merged PR reference and the ticked epic checklist item.

The current wave is always *computed* from the `blocked_by` edges plus these
states — never stored.

## Preflight

1. Verify the prerequisite skills, tracker access, and a green validation gate
   on a clean checkout.
2. Sync the default branch. Create the **epic branch** (`epic/<name>`) off it
   and note the branch name in the epic body.
3. If the epic branch already exists, this is a **resume**: reconstruct the
   wave from ticket state and existing branches, and continue.

## The wave loop

A **wave** is every ticket with no open blocker. Repeat until no tickets
remain:

### 1. Fan out

For each ticket in the wave — cap concurrency and batch a large wave — create a
**worktree** with a fresh branch off the epic branch tip, and spawn an
**implementer subagent** with fresh context, given only the spec, its ticket,
and the host contributing guide. Always a worktree; never the default checkout;
never a worktree shared across tickets. One ticket = one branch = one PR,
**targeting the epic branch**. Mark the ticket in flight and record its branch
name on it.

The implementer runs `/implement` as written, coding in `/ponytail ultra`
mode: build to spec, focused checks while working, the full gate once before
the PR, and its own per-ticket `/code-review` — that review is the ticket's
review; there is no wave-level review. Apply mechanical auto-fixes (formatter, lint `--fix`) inline; they are
never review rounds.

**Docs-only tickets** get a small doc-check subagent instead of `/implement`:
verify the docs are factually correct against the code and follow the host
repo's language standards. That subagent is their review.

### 2. Integrate serially

Merge the wave's green PRs one at a time: rebase each onto the updated epic
tip, let the gate and CI re-run, merge, tick the epic's checklist item, delete
the worktree.

### 3. Sync down

Merge the default branch into the epic branch at every wave boundary. The gate
must be green after the sync.

### 4. Failure rules

- **A stuck ticket doesn't hold its wave**: merge the green members and roll
  the stuck ticket into the next wave.
- **After 2 failed attempts** on a ticket — each attempt already noted on the
  ticket — require the human in the loop: comment the question or failure on
  the ticket, mark it blocked, and continue with unblocked work.
- **If tickets remain but none are unblocked and none are in flight**, the
  graph is wedged — stop and escalate.

### 5. Advance

The merges unblock the next wave. Repeat from fan-out.

## Epic gate — once, after the last wave

Run over `default...epic` — the merge-base is the baseline by construction.

1. Spawn the deep suite **in parallel, each its own subagent, report-only**.
   No reviewer edits code; interactive skills emit their candidates as
   findings.
2. The orchestrator applies fixes **serially** on the epic branch: apply
   findings that are mechanical, local, and behavior-preserving; structural or
   ambitious restructurings become follow-up tickets immediately.
3. Re-run **only the reviewers that flagged something**, over the changed
   files. A reviewer is happy when nothing at blocking severity —
   correctness, security, spec — remains. **Cap at 3 rounds**; leftovers
   become follow-up tickets. Keep the gate green throughout.

## Land

Open the single epic → default-branch PR referencing the epic. Merge with the
strategy the host repo's release automation requires — preserve the per-ticket
history unless the repo says otherwise. Then close out: every checklist item
ticked, follow-ups filed, one summary comment on the epic, epic closed.

## Rules of engagement

- **Spec wins on *what*; this skill wins on *how*; the host contributing guide
  wins on repo specifics.**
- **Always a worktree; one ticket = one branch = one PR**, into the epic
  branch.
- **Per-ticket review only** — `/implement`'s own `/code-review`; the deep
  suite runs once, at the epic gate, report-only.
- **Never bypass hooks or gates; never weaken a test to make it pass.**
- **Behavior-preserving refactors must prove it**: the suite is unchanged and
  green before and after.
- **Out-of-scope findings become follow-up tickets**, never silent scope
  creep.
- **The tickets are the record and the state machine** — context and
  lifecycle state live on the ticket, no parallel artifact trail; a fresh
  orchestrator resumes from tracker + git alone.
