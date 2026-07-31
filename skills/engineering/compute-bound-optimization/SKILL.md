---
name: compute-bound-optimization
description: >
  Codified method for making compute-bound code fast without losing correctness,
  distilled from a ten-year single-author corpus of GPU and SIMD engines. Use
  when the user asks to optimize, speed up, or rewrite a hot path, kernel, inner
  loop, or data-layout-sensitive routine; when they say "this is too slow",
  "make this faster", "port this to GPU/SIMD", "reduce memory traffic", "fuse
  these passes", or "why isn't this faster"; when a fast rewrite disagrees
  numerically with a slow one; when a performance test is flaky or its tolerance
  was tuned until green; or when planning a from-scratch implementation of a
  numerically-sensitive routine that must beat a reference. Applies in any
  compute-bound domain — simulation, ML kernels, query engines, codecs, DSP,
  solvers, finance. Do not use for I/O-bound latency work, service-level tuning,
  or general code review.
---

# Compute-bound optimization

A state machine, not a pipeline. Visit order can vary; **edge preconditions never do**. Every
rule below is evidence-backed, distilled from a nine-repo, ten-year study of exceptionally fast
GPU/SIMD engines. If the consuming repository carries the companion corpus
(`research/performance-lineage/PROCESS.md` and `RESEARCH.md`), consult it for the citations
behind each rule; the skill stands alone without it.
Vocabulary is substrate-neutral: control plane (host/setup side), data plane (hot path), storage
tiers, transfer granule (cache line or equivalent), permutation primitive (shuffle/permute op).

## Entry gate — pick the mode

**Is the cost structure fixed before run time, or determined by the input data?**

- Static shapes, regular access, exact arithmetic, problem class solved before → **Mode A**:
  derive the schedule off-line; land implementation + test + timing harness in one change.
- Data-dependent sizes, irregular access, floating-point accumulation, first encounter →
  **Mode B**: land the slow obvious version first and say so in the change description.

**Default to Mode B.** Mode A is earned by prior mastery, not chosen. A finished first commit is
evidence design happened elsewhere, not that design was skipped.

## States and exit predicates

**FRAME** — name the resource that bounds the computation, with a number. Fix the acceptance
metric from the external requirement (throughput target, latency budget, cost ceiling) — the
method has no internal stopping criterion, so manufacture one now. Hunt for math restructurings
(change of variables, factorization, symmetry, reuse of partials): the orders of magnitude live
here; everything later is constant factors.
*Exit:* bounding resource and "done" are both written down as numbers.

**GROUND** — implement the definition as-is: obviously correct, slow on purpose, kept forever
(it is the specification). Build a **ladder** of references — level 0 mathematically obvious →
level N structurally close to the intended fast form — all from the same plan/config object.
Adjacent rungs compare with **exact equality**. Prefer a golden **generator** (seeded random
configs + closed-form cases with analytic answers) to a golden file; freeze random draws into
the source next to the command that produced them. Build one comparator: exact for integers,
auto-fail on non-finite, **returns** max deviation so it doubles as a headroom gauge.
*Exit:* ladder self-agrees exactly, seeded, reproducible.
*Permitted shortcut:* a straight-to-LOWER feasibility probe, only if labelled as such
("will probably be too slow").

**TRANSFORM** — machine-independent restructuring. Only three families ever produced wins:
1. **Identity that deletes work/traffic** — symmetry halves, partials reused, assumptions
   dropped, sort only the bits that matter.
2. **Fusion riding a memory touch already paid for** — fold work into an existing load/store;
   goal is never materializing the intermediate, not instruction count.
3. **Hoist to the control plane** — anything constant across an invocation becomes a precomputed
   table; encode hazard decisions as flag bits so the expensive path runs only where the hazard
   exists; make transposes an index permutation (producer and consumer walk the same storage in
   different orders) instead of a pass.
Price the restructuring itself as a printed number. Try the dumb version first — clever
bit-tricks lost to plain loops twice in the corpus, epitaphs in the source.
*Exit (per rung):* proven **exactly equal** to the previous rung in exact arithmetic; plan
object prints its full resource budget before anything runs.

**LOWER** — map onto the machine. Write the physical↔logical index mapping as bit-strings per
storage tier; data movement is then a bit permutation with a known cost. Decompose offsets
against the transfer granule (`offset = granule·coarse + residual` — coarse is free addressing).
Size blocks to a stated capacity budget. Resolve tier contention by layout offset (padded /
co-prime stride) **verified by a debug-mode assertion**, not a profiler. Prefer the tier-local
permutation primitive to a round trip through the next tier out. Reduce contended accumulation
hierarchically: private → shared once per block → global once per output, and skip the global
touch when a cheap group test proves the block is empty. Declare the concurrency contract on
every hot routine. Instantiate a `Debug` flag both ways — asserts on for correctness, off for
timing — and let the asserts be the spec. Test sub-units standalone before the composite. Keep
the readable form of every cryptic expression beside it, equality-asserted over the whole domain.
**Precision:** change it last, never together with structure, in whichever direction the binding
resource dictates (down when transfer-bound, up when range-bound), bracketed by measurements on
both sides. Derive the tolerance from an error model (`k · roundoff · sqrt(depth)`; no sqrt for
shallow sums; integers exact) — then validate it both ways: measure the false-positive rate by
simulation and the detection power against a deliberately planted bug. Never tune a tolerance
until green. Randomize memory **layout** (strides, alignment, contiguity, schedule topology),
not just values; put a random-filled guard region inside the comparison so out-of-bounds writes
fail as ordinary mismatches.
*Exit:* golden tests pass under the derived tolerance **and the timing harness ships in the same
change** — unbenchmarked code stays slow forever (17-day natural experiment).

**MEASURE** — report the acceptance metric and self-relative numbers (N variants side by side,
reference included), never % of peak. Interrogate gaps with the written cost model; a wall clock
confirms hypotheses, a profiler is the exception not the loop. To price a stage, ablate it (a
switch that removes it and makes the answer wrong, guarded by a test-time assert). Hygiene:
drain before iter 1, discard warmup, trailing-window mean, print expected cost first,
machine-readable last line, never benchmark on data that flatters (no all-zero inputs when a
zero-skip exists). Never state a speedup you did not measure.

**HOLD** — stop on one of three licenses: (1) acceptance metric met with margin; (2) measured ≈
modeled irreducible cost (revisit if the structure later shifts — a declined identity became
free when the tiling changed); (3) a priced deferral written into the source (FIXME with
measured cost + reopen condition). Then record: negative results in the source at the point of
temptation; unexplained behavior confessed in the public interface; abstractions harvested only
after a second consumer exists, de-hoisted when one remains.

**STUCK** (fast has never agreed; history bisection useless) — **bisect by construction**: copy
the trusted implementation as a control, morph it toward the fast one **one axis per change**
until it breaks (the corpus's exemplar found a launch-config bug invisible in the kernel body),
then delete the instrument. Isolate sub-units. With no oracle: re-evaluate the objective at a
returned argmax instead of comparing indices; probe linear/positive structure with one-hot
inputs asserting exact zeros on the complement; place probes at boundaries ±1. Commit broken
states with the state named ("compiles but untested" → "fails unit test" → "passes"); commit
dead ends while uncertain, delete them by name when sure.

## Outer loop (across generations and projects)

- **Rewrite trigger:** the interface fights you — features stop going in cleanly.
- **Rewrite protocol:** build alongside → "passes test showing agreement with old code" →
  migrate consumers one by one → delete the old by name. Predecessor stays callable until the
  replacement is proven.
- **Mechanical and semantic changes never share a commit** (rename ≠ edit; API shape first,
  payload second).
- **Generate code only above a combinatorial threshold** (large volatile variant surface);
  check generated output in as source when small and stable. Generators emit asserts checking
  their own output against the readable twin.
- **Capability before application:** metabolize an opaque load-bearing primitive as its own
  project (wrappers, micro-oracles, timing) before betting an engine on it.
- **Port the derivation, not the code** when the memory hierarchy changes; re-derive the same
  math for the new granule.

## Invariants (all states)

Golden tests never turn off. References ship and stay. Exact equality between algorithmic rungs;
tolerance only at the precision hop; tolerances derived, never tuned. State of the artifact
named in every change description; history never rewritten. Knowledge lives in the artifact, not
in side channels. Fail loudly — no silent wrap, no vacuous pass (zero-events-parsed is an error);
robust statistics when a signal can contaminate its own baseline.

## Do not

- Do not open with a profiler; derive from a cost model written into the source, confirm by
  wall clock.
- Do not build an autotuner first; use a closed-form heuristic and keep it.
- Do not change precision and structure in the same step.
- Do not let a fast path exist without its benchmark, or outlive its reference.
- Do not report % of peak without a peak model; without an external requirement, set your own
  stopping criterion in FRAME or you will never halt.
- Do not read a finished first commit as "design is unnecessary".

## Portability check

The source method assumes a single owner with full context, a hard external requirement,
off-repo derivation skill, and self-owned dependencies. On a team, the "friction" it rejects
(CI, review) *is* the coordination — the corpus's only multi-author repo is the only one that
had CI. Derive your own tolerance model; `sqrt(depth)` assumes independent roundoff.
