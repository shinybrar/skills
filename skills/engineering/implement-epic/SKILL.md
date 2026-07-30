---
name: implement-epic
description: >
  Execute an epic of tickets with blocking edges in dependency-order waves on an
  epic branch: parallel /implement per ticket in worktrees (each with its own
  /code-review), serial merges, scheduled human updates, a report-only deep
  review suite at a mid-epic checkpoint and at the epic gate, then one merge to
  the default branch. Use when a spec has been broken into tickets by
  /to-tickets and the user wants the whole epic driven to done.
argument-hint: "The epic to execute (tracker reference or URL)"
disable-model-invocation: true
---

# Implement Epic

Drive an **epic** — tickets carrying `blocked_by` edges — from first ticket to
one review-clean merge into the default branch. You are the **orchestrator**:
spawn subagents and integrate their work; never implement tickets yourself.

## Inputs and precedence

The **spec** wins on *what* to build and its invariants; **this skill** wins on
*process*; the host repo's **contributing guide** wins on repo specifics (gate
contents, tracker, commit conventions, writing standards). On conflict the spec
is right about behavior — fix the process or raise the spec on the ticket,
never silently deviate.

## Prerequisites

Verify at preflight; **stop and escalate** if any are missing:

- `/implement` (includes its own `/code-review`), `/code-review`
- `/caveman`, `/ponytail` (used in **ultra** mode)
- Deep suite: `/thermo-nuclear-code-quality-review`, `/ponytail-review`,
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

- **Code** — every writing agent uses `/ponytail ultra`: the laziest solution
  that works, no speculative abstraction.
- **Inter-agent messages and progress notes** — `/caveman`.
- **Durable artifacts** (commits, PRs, ticket comments, docs) — the host repo's
  writing standards.
- **Human updates** — neither of the above and never `/caveman`; fixed shape
  below. The reader is re-anchoring, not skimming.
- **Effort by role** — judging unfamiliar work is harder than writing to a
  known spec. Relative allocation, restated at preflight: orchestrator and
  implementers at the session default; mid-epic checkpoint and epic-gate
  mechanical reviewers one step above; the epic-gate adversarial and
  spec-conformance passes one step above that — the largest diff and the last
  line of defense, the one place the top tier pays.

## Shape

```text
preflight:  verify skills + tracker + green gate → find serialization points → branch epic/<name> off default
per wave:   fan out worktrees (parallel) → /implement each (incl. its /code-review) → merge serially into epic → sync default into epic → update the human
throughout: escalations reported at once; never >20 min of orchestration silent
mid-epic:   at the first wave boundary past the churn threshold, run the deep suite once
epic gate:  deep suite (parallel, report-only) over default...epic → fix serially → re-run flagged only → ≤3 rounds
land:       one epic → default PR → close out
```

## Ticket state

**The tickets are the state machine.** Every transition is recorded on the
ticket via the tracker's native status, labels, and comments — never in
orchestrator memory or side files. A fresh orchestrator, or a human, resumes
from tracker + git alone. Record at minimum:

- **In flight** — plus the branch name.
- **Attempts** — each failure, with its **error signature** (file + assertion +
  message); the escalation rules read these from the ticket, not from memory.
- **Proven baseline** — the repro loop and commands per target, for the next
  ticket on that target to reuse.
- **Blocked on human** — the decision needed, plus the blocked marker.
- **Done** — the merged PR and the ticked epic checklist item.

The current wave is always *computed* from `blocked_by` + these states, never
stored.

## Preflight

1. Verify skills, tracker access, and a green gate on a clean checkout; state
   the effort allocation by role.
2. **Find the serialization points**: generated artifacts any ticket may
   regenerate wholesale (lock files, vendored trees, generated bindings).
   Tickets touching the same one **cannot share a wave** — a solver-generated
   lock conflict cannot be merged, only re-solved from the manifest. Serialize
   with edges, or give the artifact one owner ticket the rest depend on. A
   fully coupled epic is simply a chain of one-ticket waves; no special mode.
3. Sync default; create `epic/<name>` off it; note the branch in the epic body.
4. Epic branch already exists → **resume**: reconstruct the wave from ticket
   state and existing branches.

## Reporting to the human

**Scheduled, not event-driven** — an unpredictable cadence forces the human to
steer by interruption.

**When:** every wave boundary; every escalation, immediately; never more than
~20 minutes of orchestration silence (long single tickets are where thrashing
hides); before anything irreversible or outward-facing; before landing.

**Fields, same order every time:**

1. **Plan**, one line — restating it makes a dropped instruction visible on the
   very next update.
2. **Since last update** — merged, started.
3. **In flight** — ticket, target, and attempt count for anything past its
   first attempt (surfaces thrashing before the signature rule fires).
4. **Blocked** — the decision needed, not merely "stuck".
5. **Remaining** — tickets and roughly how many waves.
6. **Churn** — authored vs generated lines.
7. **Decisions for you** — called out as decisions, not buried in prose.

These fields are a strict subset of a handoff. Keep them so: a handoff is then
the current update plus what was established and ruled out, never a document
assembled from memory on demand.

## The wave loop

A **wave** is every ticket with no open blocker. Repeat until none remain.

### 1. Fan out

Per ticket (cap concurrency; batch a large wave): fresh **worktree** and branch
off the epic tip; spawn an implementer with fresh context, given only the spec,
its ticket, and the contributing guide. Always a worktree, never shared, never
the default checkout. One ticket = one branch = one PR, **into the epic
branch**. Mark in flight; record the branch.

The implementer runs `/implement` in `/ponytail ultra`: build to spec, focused
checks while working, the full gate once before the PR, and its own
`/code-review` — that is the ticket's review; there is no wave-level review.
Mechanical auto-fixes (formatter, lint `--fix`) are inline, never review
rounds.

**Repro loop before any change.** For each target the ticket touches: establish
the fastest loop that faithfully reproduces the gate, run it **green on the
untouched base**, record the commands on the ticket — or reuse an earlier
ticket's proven baseline. The green baseline is what makes a later red
attributable to the change. CI confirms, never debugs (a CI round trip is
20–100× a local one); a target that truly cannot run locally is named as an
exception, with the reason. Fidelity counts: a loop differing in privilege,
filesystem semantics, or locale lies in both directions — a "write must fail"
test passes spuriously as root.

**Docs-only tickets** get a doc-check subagent instead of `/implement`: verify
facts against the code and the host language standards. That is their review.

### 2. Integrate serially

Merge green PRs one at a time: rebase onto the epic tip, gate + CI re-run,
merge, tick the checklist item, delete the worktree.

### 3. Sync down

Merge default into epic at every wave boundary; gate green after.

### 4. Failure rules

- A stuck ticket doesn't hold its wave: merge the green members, roll it into
  the next wave.
- **After 2 failed attempts** on a ticket → HITL: comment the question or
  failure, mark blocked, continue with unblocked work.
- **After 2 attempts at the same error signature** (same file + assertion +
  message, whatever the hypothesis) → escalate the same way even though the
  ticket has not failed. Comment what was tried and **what each attempt ruled
  out** — that list is what the human needs in order to decide. A third try at
  an unchanged error is a decision made by attrition.
- Tickets remain, none unblocked, none in flight → the graph is wedged: stop
  and escalate.

### 5. Advance

The merges unblock the next wave. Check the churn threshold first and run the
mid-epic checkpoint if crossed and not yet run. Repeat from fan-out.

## Mid-epic checkpoint

Once, at the first wave boundary where **authored churn exceeds 1500 lines**
(insertions + deletions, generated files excluded). Same mechanics as the epic
gate. A line count, not a judgement call, so it fires without anyone deciding
to be thorough: a defect found at the tip is cheap; the same defect ten commits
later is a structural fix that gets deferred. Record declined findings so the
gate does not re-raise them.

## Epic gate — after the last wave

Over `default...epic` — the merge-base is the baseline by construction.

1. Deep suite **in parallel, each its own subagent, report-only**. No reviewer
   edits code.
2. The orchestrator applies fixes **serially**: only mechanical, local,
   behavior-preserving ones; anything structural becomes a follow-up ticket
   immediately. **Re-run the gate after each applied finding** — mechanical in
   appearance is not behavior-preserving (hoisting a repeated expression fails
   when its validity is scope-dependent). A **declined** finding gets its
   reason recorded **at the site**, or the next reviewer re-raises it and the
   next implementer re-breaks it.
3. Re-run only the reviewers that flagged, over the changed files, until
   nothing at blocking severity (correctness, security, spec) remains. **Cap at
   3 rounds**; leftovers become follow-up tickets. Gate green throughout.

## Land

Open the single epic → default PR referencing the epic; merge per the host
repo's release automation, preserving per-ticket history unless the repo says
otherwise. **Report the diff split authored vs generated**, naming the
generated files — a regenerated lock otherwise makes a net-negative epic look
like +20k lines. Close out: checklist ticked, follow-ups filed, one summary
comment, epic closed.

## Rules of engagement

- Spec wins on *what*; this skill on *how*; the contributing guide on repo
  specifics.
- Always a worktree; one ticket = one branch = one PR, into the epic branch.
- Per-ticket review via `/implement`'s own `/code-review`; the deep suite runs
  report-only, at the checkpoint and the gate.
- Never bypass hooks or gates; never weaken a test to make it pass.
- **A new guard must be shown to fail before it is trusted to pass**: induce
  the condition, observe the refusal, restore, record the induction in the
  commit. Applies to any assertion not written test-first — `/tdd` already
  supplies the failing observation for anything built red-green. Where "found
  nothing" and "examined nothing" look identical, also assert a non-zero
  examined count.
- Behavior-preserving refactors prove it: suite unchanged and green before and
  after.
- **Clean working tree at every handoff, pause, and escalation.** Unproven work
  is reverted, not parked; a rejected approach is recorded as rejected, with
  the reason, or someone restores it as an oversight.
- Out-of-scope findings become follow-up tickets, never silent scope creep.
- The tickets are the record and the state machine; a fresh orchestrator
  resumes from tracker + git alone.
- Update the human on a schedule, not on events. If they have to ask what is
  happening, the cadence was wrong; if they have to repeat an instruction, the
  update was not restating the plan.
