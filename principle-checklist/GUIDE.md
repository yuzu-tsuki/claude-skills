# `principle-checklist` — guide

Human documentation. Not loaded during a run; a run executes `SKILL.md`, plus `harvest.md` in `harvest` mode.
한국어: `GUIDE.ko.md` — identical content; a change to one must be applied to the other.

## What it is

Engineering principles applied as checks that produce evidence, each at the phase where the mistake
it guards against is still cheap to stop: while planning, while fixing each review finding, and
before commit or PR. A single end-of-change checklist gets answered from memory, and whatever it
does catch forces rework, re-review and another test run — splitting the checks by phase is what
moves the catch earlier.

## Invoking

`/principle-checklist <mode> [principles-path]` — `plan`, `fix`, `finish`, or `harvest`. With no
mode, the run infers one from the state and says which it chose.

The principle list comes from the project's `.claude/principles.md` (or the path given) when it
exists, otherwise from the bundled defaults in `principles.md`.

## Installing

Copy this folder into `~/.claude/skills/` for every session, or into a project's `.claude/skills/`
for that project only. Nothing else is needed.

**Status: not yet run on a real change.** One simple change does not validate it. Run `plan`, `fix`
and `finish` on a real change, then `harvest` on a second repo, before sharing it widely.

## Modes

| Mode | When | What it produces |
|---|---|---|
| `plan` | Before writing code | One line per applicable principle in the plan or blueprint — how the change prevents it, or `n/a` with a reason. Probes against real external components run now (P6) |
| `fix` | For every review finding, before calling it closed | The finding restated as a defect class, a repo-wide sibling search (P2), and a full re-run of every earlier verification (P1). One evidence line per finding |
| `finish` | Before commit, push, or PR — after the final code change | A mutation pass over changed lines (P4), every claimed number and strong word re-derived from logs and git (P3), contract artifacts confirmed in the same commit (P8) |
| `harvest` | When a project wants its own list | `.claude/principles.md`, mined from the project's PR/MR review history (`harvest.md`) |

## The default principles

Each entry in `principles.md` carries the pattern, examples, the check, and the evidence line to record.

| ID | Principle | Phases |
|---|---|---|
| P1 | A fix breaks what used to work | fix, finish |
| P2 | Fix the class, not the instance | fix |
| P3 | Claims must match the final artifact | finish — after the last commit, rebase or merge |
| P4 | Deleting new code must fail a test | finish (plan names the intended tests) |
| P5 | Failure paths must be loud and bounded | plan, first review |
| P6 | Verify external behavior on the real system | plan |
| P7 | Filesystem trust boundary | plan, first review — only when the change touches paths |
| P8 | A behavior change updates the contract | plan, finish |

## Rules

- **Evidence, not ticks.** A principle is never `applied` without an evidence line. "The tests pass"
  is not evidence for P1 or P4.
- **`finish` runs after the final code change.** Any later edit invalidates it.
- **First review passes get the principles; follow-up passes don't.** A first pass gets the
  applicable principles as explicit items, split by surface — code: P1, P2, P4–P7; docs and
  contracts: P3, P8. A pass that only checks whether findings were closed gets none: handing it the
  list widens the scope every round and the loop stops converging.
- **Keep the list honest.** Drop a principle only when it has not recurred in recent PRs that
  touched its surface — absence where the surface was untouched means unknown, not fixed.

## Report

```
Principles (<source>) — <mode>
P1 fix-induced regression    applied  | previous mutants 11/11 killed; replay 0 lost of 8,036 inputs
P4 deleting new code         deferred | 2 survivors (optional attr types) tracked in <where>, owner <who>
P7 filesystem trust          n/a      | change handles no paths
```

| State | Means |
|---|---|
| `applied` | Done, with the evidence line |
| `n/a` | Does not apply, with the reason |
| `deferred` | Not done here — where it is recorded and who owns it |
| `failed` | Blocks the commit or PR |

## Cost

An estimate, not a measurement: roughly **10–20% more time** on a non-trivial change than the same
work without the checklist. The per-check figures below were observed in one session.

| Check | Typical cost | What drives it |
|---|---|---|
| P4 mutate each changed line | Minutes — 4 mutants in ~2 min, ~80 in a few | One incremental rebuild per compiled mutant; scripts need no build. **The main variable cost** |
| P1 replay earlier inputs | Under a minute for ~11,000 inputs | Cheap when old versions are scripts; expensive when they need building or deploying |
| P6 probe the real system | Seconds, or minutes to stand up a scratch database once | Near zero with the real component on hand; hours if the environment must be built |
| P2 search the defect class | Minutes | Small |
| P3 re-derive claims | Minutes per PR | Small |
| `plan` answers | A few lines in the plan | Small |
| `harvest` | Tens of minutes of reading, plus a large token bill | Once per project, not per change |

- **Small changes pay a larger share.** The per-principle report and the `finish` mutations are a
  fixed cost that dominates a tiny change.
- **Where it can blow up.** Slow builds make P4 expensive: group mutants by file and start with the
  lines behind the PR's headline claims. With no real system for P6, record the assumption as
  verified against mocks only rather than building infrastructure mid-change.
- **The payback is unmeasured.** Removing one external review round — a reviewer turnaround, a fix
  and a full test run — outweighs the checks, but whether the checklist actually removes rounds has
  not been shown yet.

## Adapting to another repo

The defaults encode one project's pattern; other codebases fail differently. Run `harvest` after a
few PRs, or whenever review keeps finding the same kind of problem, and let its output replace the
defaults for that project.

The bundled files stay project-neutral: no project names, people, decision IDs or internal paths.
Project specifics — PR numbers in examples included — belong only in a project's own
`.claude/principles.md`.
