# Claude skill library

Skills for Claude Code. Each folder is one self-contained skill: `SKILL.md` is what runs,
`GUIDE.md` is the human documentation, `agents/` holds any agent presets it needs, and `commands/`
holds any short slash-command aliases.

| Skill | What it does |
|---|---|
| [`zero-to-commit`](zero-to-commit/) | Carries one slice of work through the full implementation lifecycle — empty branch to verified commit |
| [`principle-checklist`](principle-checklist/) | Applies engineering principles as evidence-producing checks while planning, fixing review findings, and finishing a change |

## zero-to-commit

The user and the main session draft the blueprint; after that the main session codes and reviews
nothing. It routes the slice between scoped agent presets — architect (reviews the blueprint),
implementer, auditor, gatekeeper — runs each round's single build, and adjudicates the findings,
looping until the review converges and then committing. Every handoff is a file, so the final gate
verifies evidence on disk rather than a summary.

Invoke it as `/zero-to-commit`, or `/ztc` once `commands/ztc.md` is installed.

Details, flags, and install steps: [`zero-to-commit/GUIDE.md`](zero-to-commit/GUIDE.md).

## principle-checklist

Eight default principles — fix regressions, sibling instances, claims, coverage of new code, failure
paths, external behavior, filesystem trust, and contracts — each run as a check at the phase where
it is still cheap: `plan` before coding, `fix` for each review finding, `finish` before commit or PR.
Every check ends in an evidence line, never a tick. `harvest` mines a repo's own PR/MR review
history into a project-specific list.

Invoke it as `/principle-checklist <mode>`. Estimated to add roughly 10–20% to the time a
non-trivial change takes, mostly in mutation checks. It has not yet been run on a real change.

Details, modes, and install steps: [`principle-checklist/GUIDE.md`](principle-checklist/GUIDE.md).
