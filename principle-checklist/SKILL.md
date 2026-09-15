---
name: principle-checklist
description: Apply engineering principles as evidence-producing checks at the moment each is cheap — while planning a change (plan), while fixing each review finding (fix), and before commit/PR (finish) — or build a project's own principle list from its PR/MR review history (harvest). Use when starting a non-trivial change, when addressing review feedback, before declaring a change ready, or when a team wants to turn past review comments into checks.
argument-hint: "[plan|fix|finish|harvest] [principles-path]"
---

# Principle checklist

A checklist helps only when each item runs at the moment the mistake it guards against is still cheap to stop, and produces **evidence** rather than a tick. An end-of-change checklist gets answered from memory, and whatever it does catch forces rework, re-review and another test run.

## Principle source

1. The project's own list, if present: `.claude/principles.md`, or a path given in the arguments.
2. Otherwise `principles.md` next to this file — the 8 default principles.

Each principle carries an ID, the phases it applies to, the pattern, examples, the check, and the evidence to record. Say which source you used.

## Modes

The first word of the arguments picks the mode: `plan`, `fix`, `finish`, `harvest`. With no mode, infer it from the state (a design request → plan; open review findings on a diff → fix; about to commit or open a PR → finish) and say which one you chose.

### plan — before writing code

For every principle whose phases include `plan`, the plan or blueprint answers in one line: how this change prevents it, or `n/a` with a reason. The default principles require actions, not just answers:

- **External behavior (P6):** list each assumption about a library, database, OS or protocol that the design depends on, run one probe per assumption against the real component now, and record the command and result. If a probe contradicts the design, redesign before coding.
- **Failure paths (P5):** list the new error, drop, fallback and retry branches the design introduces; for each, name the operator-visible surface that reports it and the numeric bound.
- **Paths (P7):** if the change takes or documents filesystem paths, state the type/owner/mode/symlink/alias/replace rules.
- **Contracts (P8):** list every doc, schema, generated artifact and digest pin the behavior change touches, including derived artifacts and checkers that no test runs, with their regeneration commands.

### fix — for every review finding, before calling it closed

1. Restate the finding as a defect **class** in one sentence.
2. Search the whole repository and the diff for siblings of that class. Fix all of them, or record the count and where the rest are tracked (P2).
3. After the fix, re-run the **entire** previous verification set — every earlier mutant and every earlier reproduction input, not just the new ones (P1).
4. If the fix replaced or narrowed logic, replay the inputs the old version handled (plus split, reordered and boundary variants) against the new version; zero cases may be lost. Keep the old rule as a union backstop unless there is a measured reason not to (P1).
5. If the fix adds a constraint, run at least one legitimate configuration the constraint could reject (P1).

Record one evidence line per finding.

### finish — before commit, push, or PR

Run only after the final code change; later edits invalidate it.

- **Deleting new code must fail a test (P4):** enumerate the changed production lines — including wiring, configuration propagation, bounds and hook arguments, not only functions. Mutate each (delete, invert, revert) and record which test fails by name. Survivors get a test or a recorded gap. Scan new tests for wall-clock reads, sleeps and thread-start races.
- **Claims (P3):** after the last commit, rebase or merge, re-derive every number in commit messages, the PR description, replies, changelogs and trackers from logs and git. For each strong word (all, complete, atomic, removed, unchanged, guaranteed), point to the command or test that proves it, or narrow the sentence.
- **Contracts (P8):** confirm the planned contract artifacts changed in the same commit, and run the schema validators against real payloads.
- Confirm that `plan` and `fix` evidence exists for every applicable principle.

### harvest — build or refresh a project's principles

Follow `harvest.md` next to this file. The output replaces the default list for that project.

## Using the principles in reviews

- When delegating a first review pass (subagents or people), include the applicable principles as explicit items, split by surface: code gets P1, P2, P4, P5, P6, P7; docs and contracts get P3, P8. A reviewer that never saw a principle will not apply it.
- A follow-up review that checks whether findings were closed does **not** get the checklist. Handing it the full list widens the scope every round and the loop stops converging.

## Report format

```
Principles (<source>) — <mode>
P1 fix-induced regression    applied  | previous mutants 11/11 killed; replay 0 lost of 8,036 inputs
P4 deleting new code         deferred | 2 survivors (optional attr types) tracked in <where>, owner <who>
P7 filesystem trust          n/a      | change handles no paths
```

States: `applied` (with evidence), `n/a` (with reason), `deferred` (where it is recorded and who owns it), `failed` (blocks the commit or PR).

Never mark a principle `applied` without an evidence line. "The tests pass" is not evidence for P1 or P4.

## Keeping the list honest

- The defaults encode one project's failure pattern. Other codebases fail differently — run `harvest` after a few PRs, or whenever review keeps finding the same kind of problem.
- Drop a principle only when it has not recurred in recent PRs **that touched its surface**. Absence where the surface was untouched means unknown, not fixed.
