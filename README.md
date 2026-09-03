# Claude skill library

Skills for Claude Code. Each folder is one self-contained skill: `SKILL.md` is what runs,
`GUIDE.md` is the human documentation, `agents/` holds any agent presets it needs.

| Skill | What it does |
|---|---|
| [`zero-to-commit`](zero-to-commit/) | Carries one slice of work through the full implementation lifecycle — empty branch to verified commit |

## zero-to-commit

The main session designs, codes, or reviews nothing. It routes the slice between scoped agent
presets — architect, implementer, auditor, gatekeeper — runs each round's single build, and
adjudicates the findings, looping until the review converges and then committing. Every handoff
is a file, so the final gate verifies evidence on disk rather than a summary.

Details, flags, and install steps: [`zero-to-commit/GUIDE.md`](zero-to-commit/GUIDE.md).
