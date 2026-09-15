# harvest — derive a project's principles from its review history

The goal is a short list (5–10) of mistakes that **keep recurring**, each written as a check that produces evidence. Mined history beats a generic list: every codebase fails in its own way.

Keep raw review data in a scratch directory, not in the repository. Review comments can contain internal details, so publish only the generalized principles.

## 1. Collect

Fetch every PR/MR with its reviewer comments, as JSON.

- **GitHub:** `gh api --paginate 'repos/{owner}/{repo}/pulls?state=all&per_page=100'`. Then for each PR: `pulls/{n}/reviews`, `pulls/{n}/comments`, `issues/{n}/comments`.
- **GitLab:** `glab api --paginate 'projects/{id}/merge_requests?state=all&per_page=100'`. Then for each MR: `merge_requests/{iid}/notes?per_page=100` (drop `system: true`).

Separate reviewer comments from the author's replies. Keep the replies only for context (rebuttals, "fixed in …").

## 2. Condense

Extract each finding's title or first line, and the review verdict, into one file small enough to read completely. Headings in structured reviews, or the first bold line of inline comments, usually carry the finding.

## 3. Tally, as a pointer only

Count candidate category keywords per PR in chronological order (for example: silent/swallowed/no-op, untested/vacuous/green anyway, regression/reopened/same shape, stale/contradicts/overstated, unbounded/starve, symlink/permission, schema/contract/unregistered).

The matrix shows **recurrence over time**, not evidence. Keyword hits include praise and "verified OK" sections, so read the actual findings behind every category before trusting a count.

## 4. Read the recent PRs in full

Read the full reviewer comments of the most recent 3–4 PRs. Recency decides whether a pattern is still active.

## 5. Categorize

- A category needs findings in **at least 3 PRs**.
- Merge near-duplicates.
- Keep "fixed only the flagged instance" separate from "the fix broke something that worked": the checks differ (sibling search vs full re-verification).
- Classify the author's mistake, not the reviewer's topic. "Missing test for a new code path" and "wrong wording in a doc" are different categories even when they appear on the same file.

## 6. Active or fixed

- **Fixed:** not seen in recent PRs **that touched the category's surface**, ideally after a known rule or tool change (a lint rule, a hook, a written convention).
- **Unknown:** not seen, but recent PRs did not touch the surface. Keep it, and mark it that way.
- **Active:** still seen recently. Keep it.
- Do not write a principle for something tooling already enforces mechanically.

## 7. Write each principle

Use the format in `principles.md`: ID, phases, pattern, 3–6 generalized examples citing PR numbers in the project copy, the check, and the evidence line. Assign phases by when the mistake is still cheap to stop:

| Mistake is created | Phase |
|---|---|
| In the design or code shape (assumptions, error handling, contracts) | plan |
| At the moment of fixing a finding | fix |
| By later edits or once code is stable (claims, coverage of final lines) | finish |

## 8. Validate before saving

For each principle, take one past finding and ask: would running this check at its phase have caught it? If not, sharpen the check until it would. A check that would not have caught its own examples is advice, not a check.

## 9. Save and report

Save the list to `.claude/principles.md`, or the path the user names. Report:

| Category | PRs (count, numbers) | Last seen | Status | Principle ID |
|---|---|---|---|---|

Also list the categories dropped as fixed, with the evidence for that call.
