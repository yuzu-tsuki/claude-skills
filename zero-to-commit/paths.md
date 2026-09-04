# zero-to-commit — path details

Loaded on demand, not with `SKILL.md`: read this file in full before step 0 whenever the
invocation carries `review-fix:` or `no-commit:`, or combines two or more flags. Everything here
extends `SKILL.md`; the mechanics at its top apply unchanged.

## Flag combinations

`simple:` and `opus-architect:` combine, but the combination is moot: the fast path launches no architect at all, so the design preset only matters if a fallback to step 1 happens. `review-fix:` combines with `opus-architect:` (the disposition role runs on that preset) and excludes `simple:` — the review-fix path already is the reduced shape. `no-commit:` combines with `simple:` and `opus-architect:`, but not with `review-fix:` — working an MR's feedback is commit-shaped by definition, so treat that combination as a contradiction and ask the user which they meant.

## Review-fix path (`review-fix:` — MR feedback rounds)

Working an MR reviewer's feedback is a recurring shape this loop otherwise leaves to improvisation. It replaces steps 1–2; steps 3–6 apply with the deltas below.

1. **Fetch the feedback and the branch state first.** `git fetch` + `glab mr view <n> --comments`; check out the MR's source branch (expect the build trees to be stale after the switch — the first gate run will be a full rebuild; say so in the checkpoint rather than treating it as an anomaly). Save the reviewer's comment verbatim to the scratchpad as `mr<n>-feedback-r<round>.md`.
2. **Disposition before coding.** For each reviewer item, decide fix-now / defer-with-registration / rebut — this is the architect's role; on slices where you hold the design context yourself, do it inline but still write `dispositions-r<round>.md`. A reviewer item that needs the user's ruling pauses the loop, same as a blueprint conflict.
3. **Implement the fix-now items**, then run step 3 (auditor — scoped to the fixes plus anything the reviewer's reasoning implicates) and step 4 as usual. The round-closure rule applies with full force here: every reviewer launched must report before gates start.
   - **Use the reviewer wait** per the mechanics in `SKILL.md` — the auditor has measured ~16 min at xhigh on a full round's diff; draft only findings-invariant work while it runs (mechanical tracker fields, the reply-file skeleton, commit bodies for already-verified fixes) — not the mutation list, which depends on findings that haven't landed yet.
4. **Commit shape**: docs (SSOT/ICD/ask) as a leading commit, code fix, tracker — the established three-way split. Append commits; never rewrite history the reviewer has seen.
5. **Write the reply file** (scratchpad, delta-only per CLAUDE.md §2's reply rules) after the gatekeeper verdict, so the numbers it cites are the verified ones.

## No-commit substitutions (`no-commit:`)

Steps 1–4 run unchanged; verification is never reduced — only the git writes are.

- **Step 0**: the branch ceremony is optional — staying on the current checkout is fine, since nothing will be committed to it.
- **Step 5**: skip the delegation to a finishing skill entirely — a finishing skill commits on its own authority — and run the ritual with these substitutions: item 1 (full gates) runs unchanged, ceremony scales with risk but verification never shrinks; item 2's tracker update is drafted to the scratchpad as `tracker-draft.md` instead of applied to the repo docs; item 3 becomes drafting the commit message(s) to `commit-drafts.md` with nothing committed or staged; item 4's checkpoint report says "verified, uncommitted" and points at the drafts so the user can commit later with one paste.
- **Step 6**: declare the run no-commit to the gatekeeper; it marks the commit-shape checks N/A instead of UNVERIFIED and verdicts on gates, review completeness, and format alone.
