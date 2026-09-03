# Agent presets for `token-saver`

Usage guide: `../GUIDE.md` · 한국어 `../GUIDE.ko.md`.

Copies of the agent definitions `SKILL.md` invokes, bundled here so the skill can be shared as one
unit. **These copies are inert.** Claude Code loads agents from `.claude/agents/`, not from a skill
folder. `SKILL.md` step 0 self-installs missing presets as a fallback (copy-only, never overwriting
an existing name), but a preset copied mid-session may not be launchable until the next session —
installing up front (below) is the reliable path.

## Install

```
cp .claude/skills/token-saver/agents/*.md  <target-repo>/.claude/agents/
cp -r .claude/skills/token-saver           <target-repo>/.claude/skills/
```

Drop `README.md` from the `agents/` copy — it is documentation, not a preset.

No other skill is required: step 5's commit ritual is built into `SKILL.md`. If the target repo has
its own finishing skill (the source repo's is named `subtask-done`), step 5 delegates to it instead.

## What each preset is for

| Preset | Model / effort | Role in the loop |
|---|---|---|
| `architect` | fable-5 / high | Step 1 blueprint; step 4 disposition |
| `architect-v2` | opus / max | Same brief, for plan tiers without Fable access (`opus-architect:`) |
| `implementer` | sonnet / high | Step 2 code and unit tests; no shell — the orchestrator builds warm and tests once per round |
| `logic-auditor` | opus / xhigh | Step 3 adversarial review; delta passes |
| `docs-conformance` | sonnet / high | Step 3, docs surface only (SSOT / ICD / tracker) |
| `gatekeeper` | sonnet / high | Step 6 process verification before a commit-ready callout |
| `quick-question` | haiku / medium | Side lookups that must not disturb an in-flight agent |

## Repo-specific content to adapt

These were written for one project. A recipient should expect to edit:

- Build and gate commands — `dev-debug`, `ci-gcc11-full`, `asan-full` are CMake presets of this repo.
- Doc paths — the decisions SSOT, the ICD, and `docs/dev-plan/*-implementation-tracker.md`.
- The decision-ID scheme (`D-` / `B-` / `R-`) the auditors are told to check against.
- `CLAUDE.md §` citations — `SKILL.md` and several briefs cite this repo's CLAUDE.md sections
  (§1 adversarial review, §2 branch/MR rules, §4 multi-agent, §6 hard invariants).
- The toolchain constraint in the implementer brief (C++20 on the GCC 11.5 / libstdc++ 11 subset).
- `glab` in the review-fix path — swap for `gh` or the forge CLI the target repo uses.
- Language: several briefs assume Korean prose for commits, trackers, and MR text.

## Two rules that carry the loop

They are in `SKILL.md` and in the auditor's own brief, deliberately duplicated so the rule survives
a badly written brief — keep both copies when adapting.

- **Review scope narrows monotonically.** The first pass fixes the surface for the whole slice. A
  delta pass asks only "was the disposed feedback correctly applied?" — it may not open a
  perspective the first pass did not take, including on lines the fix newly wrote. The implementer
  may iterate freely; the reviewer's scope may not grow.
- **Comments are never findings — in either direction.** Stale ones are batched, unnumbered, and
  never trigger a delta pass; "insufficient" ones are not reportable at all — unclear code is a code
  finding (naming, extraction, types), and details go to compact docs. Inside the loop comments only
  shrink; no auditor report, disposition, or fix may make one longer.
