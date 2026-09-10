# `zero-to-commit` — guide

Human documentation. Not loaded during a run; a run executes `SKILL.md`, plus `paths.md` on the `review-fix:` / `no-commit:` paths.
한국어: `GUIDE.ko.md` — identical content; a change to one must be applied to the other.

## What it is

An orchestration loop for one slice of work, built to spend as few main-session tokens as possible
on complex engineering. The main session never designs, codes, or reviews — it routes between
presets, runs each round's single warm build, and adjudicates; the heavy reasoning runs on scoped
presets, each on the cheapest model that can own its surface. Every handoff is a file in the
session scratchpad.

## Invoking

`/zero-to-commit <flags> <slice>`, or `/ztc <flags> <slice>` — the same run either way. `/ztc` is a
one-line command in `commands/`, installed alongside the skill (`agents/README.md`); without it only
the full name resolves.

## Pipeline

| Step | Preset | Output |
|---|---|---|
| 1 | `architect` | `blueprint.md` — files, seams, data shapes, test plan, non-goals, cited decision IDs |
| 2 | `implementer` | Code + unit tests. Builds its own targets and runs its own test binaries in its tree as self-checks (never configure, clean, `all`, or `ctest`); the orchestrator builds warm and tests once per round as the evidence run |
| 3 | `logic-auditor` (+ `docs-conformance`) | `findings-r<n>.md` — ID'd defects with failure scenarios |
| 4 | `architect` | `dispositions-r<n>.md` — every ID becomes fix / defer / invalid |
| 5 | *(commit ritual)* | Tracker first, then gates once, then commits — built in; delegates to a project finishing skill when one exists |
| 6 | `gatekeeper` | Pass/fail per check against disk and git, never against claims |

Steps 2–4 loop. Steps 5–6 run once.

## Flags

| Flag | Effect |
|---|---|
| *(none)* | Full path from step 1 |
| `simple:` | Skip the blueprint; single implementer; orchestrator dispositions findings inline. Auditor and gatekeeper still run |
| `review-fix:` | MR-feedback path. Replaces steps 1–2 with fetch-and-disposition |
| `opus-architect:` | Design on `architect-v2` (opus) instead of `architect` (fable-5) |
| `no-commit:` | Full verification, no git writes — tracker and commit messages become scratchpad drafts; gatekeeper marks commit-shape checks N/A. Excludes `review-fix:` |

Prose instead of a flag is honored — say which flag you read it as.

## Rules

### Review scope narrows monotonically

- The **first** pass fixes the surface and perspective for the whole slice, including what it fails
  to look at.
- A **delta** pass asks one question: was the disposed feedback correctly applied? A finding is
  admissible only where a prior finding is not actually closed — the fix is wrong, incomplete, or
  its test does not pin what it claims.
- It may not open a perspective the first pass did not take, **including on lines the fix newly
  wrote**. Anything else becomes a tracker residual, not a fix in this loop.
- The **implementer** may iterate freely. It is the reviewer's scope that cannot grow.

### Comments are never findings — in either direction

- **Stale or wrong** (drift): auditors batch it, unnumbered and severity-free, under "comment
  sweep". It never triggers a delta pass.
- **Missing or "insufficient"**: not reportable at all. If code needs explaining, the finding is
  the code — naming, extraction, types — never "add a comment".
- Inside the loop comments only shrink or disappear, and no disposition may grow one. The code
  explains itself; a comment is one line for what code cannot say; everything else lives in the
  docs directory — where each file stays compact too.

### One reviewer per surface

- Never put two reviewers on the same surface.
- Disjoint surfaces may run together — `logic-auditor` on code, `docs-conformance` on SSOT/ICD/tracker.
- An auxiliary auditor (the repo-local `agy` preset where it exists, otherwise a fresh
  `logic-auditor` launch) is a tie-breaker for a contested verdict, invoked after the fact.

## Operating notes

- **Launch reviewers and the architect unnamed** — as plain background agents, never named
  teammates. Continue one with `SendMessage` to its id.
- **A round closes only when every reviewer launched for it has delivered.** Do not start the gates
  while any reviewer of the round is still outstanding.
- **Use the reviewer wait, but only for findings-invariant drafts.** Mechanical tracker fields, the
  reply skeleton, and commit messages for already-verified fixes are safe to draft while the auditor
  runs; a mutation list or anything else that depends on the findings is not — mark drafts
  pending-verification and reconcile when findings land.
- **Implementers self-check by building and running; the orchestrator owns the round's evidence run.**
  An implementer builds only its own targets, incrementally, in the tree it was launched in (its
  worktree on a fanned-out round, the live tree when it is alone), runs the test binaries it wrote
  directly, and runs the per-file gates (formatter, complexity, every required compiler) — never
  configure, clean, `all`, or `ctest`. The old blanket no-build rule guarded a shared build dir that
  worktree isolation made race-free; what it cost was unexecuted tests (2026-09-07: four of seven
  defects in one fix round were build- or run-time failures the implementer could not see). The
  orchestrator then builds incrementally in the existing tree (never clean or reconfigure inside the
  loop), runs the repo's per-function static checks on new code at this same point, and sends
  failures back as one batch. One rebuild per fix-batch; three failed round-trips on the same
  failure go to the user. Comment-only changes do not need a build at all.
- **Parallel implementers run in worktree isolation.** PARALLEL-SAFE packages are normally disjoint
  file sets, but the blueprint may instead grant disjoint, append-only ranges within one shared file
  when packages separate along a different axis (declarations / impls / tests). Either way, each
  parallel implementer launches with `isolation: 'worktree'`; the orchestrator merges each worktree's
  changes into the live tree (into its declared range, for range-owned packages) before the round's
  single warm build. This costs a cold configure per worktree — the round's own build point stays one
  warm, incremental build regardless.
- **Cut at coupling boundaries, never at commit boundaries.** Coupling is a different question from
  parallel-safety: packages are coupled when a defect can live in the seam between them, and they
  are often file-disjoint yet coupled. Coupled packages stay in one review scope and one loop —
  commit one before its sibling exists and no reviewer ever holds both halves of the seam.
- **Size each agent between two bounds.** Cap a task at ~60 tool calls (an agent's cost tracks its
  call count), but never split below the ~80–120k per-launch orientation floor. Briefs point at
  artifacts and never restate them.
- **Never message a reviewer mid-pass.** It restarts the report rather than augmenting it.
  Everything goes in the launch brief — including the round's diff, pre-written to the scratchpad —
  and corrections wait for the delta brief.
- **Finish every edit before the evidence run.** Gate logs are evidence for the tree as it stands,
  so a tracker correction made afterward invalidates them all. The test runner, not a list, is the
  authority on what the gates are: run the full suite, never a `-R` subset.
- **Verify agent claims against source** before acting on them — in both directions: a defect
  called clean and a clean line called defective. Every figure you cite comes out of a log, never
  out of memory.

## Cost

Two full-path runs, measured: **~2.9–3.0M subagent tokens, 135–170 min wall, 14 distinct agents**,
~30 files and ~4,000 insertions each. Tokens go to implementers ~52%, reviewers ~31%, architect ~7%,
gatekeeper ~4%.

The shape that matters:

- **An agent's cost tracks its tool-call count and nothing else.** Every call re-sends the whole
  accumulated context, so cost inside one agent grows as `1+2+…+n` while the per-call rate stays
  flat at 2–4k tokens. Capping a task is the only real token lever.
- **Every launch pays an orientation floor of ~80–120k**, whatever the task size — a two-file fix
  still cost 79k, because it must read the disposition, findings, targets, and sibling conventions
  regardless. Splitting below that floor multiplies fixed cost.
- **Subagents are already context-flushed between launches**, so flushing more often does not help
  where the tokens actually are. Cutting a slice into micro-commits costs *more* once each extra
  commit-ready point's gate run and gatekeeper pass are counted — and it ships seam defects.

Wall-clock for a medium slice: implement-plus-build 20–30 min per round, `logic-auditor` 10–20 min
when measured at xhigh (the preset now runs at high), `docs-conformance` and `gatekeeper` ~5 min
each. Overshoot is usually findings volume and user-decision pauses — correctness work, not overhead.

## Adapting to another repo

`agents/README.md` lists what to edit. The short version: gate commands, doc paths, the decision-ID
scheme, the `CLAUDE.md §` citations, the toolchain constraint, `glab`, and the language assumption
in several briefs.
