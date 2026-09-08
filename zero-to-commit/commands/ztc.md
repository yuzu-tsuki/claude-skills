---
description: Alias for the zero-to-commit skill — carry one slice from branch to verified commit.
argument-hint: "[simple:] [review-fix:] [opus-architect:] [no-commit:] <slice description>"
---

Invoke the `zero-to-commit` skill with the Skill tool, passing `$ARGUMENTS` through verbatim as its
arguments (flags included, unparsed — `SKILL.md` reads them itself). Do nothing else first: no
exploration, no branch inspection, no restating the slice. The skill owns the whole run from its
step 0.
