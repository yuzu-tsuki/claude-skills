# Default principles

Generalized from about 40 merge requests of review history on one systems codebase (C++, SQL, schema generators, security-sensitive services). The examples are real findings with project details removed. Replace or extend this list with `harvest`.

---

## P1 — A fix breaks what used to work

**Phases:** fix, finish

**Pattern:** the fix for a review finding breaks behavior that previously worked. Narrowing a detection or validation rule, or replacing an implementation, silently drops cases the old version handled. This was the most frequent category.

**Examples**
- A static check was rewritten from a regex to a tokenizer to cut false positives; queries assembled from several string fragments, which the regex caught, were no longer detected.
- A fix for a status-reporting bug reintroduced the same symptom in the opposite direction.
- A constraint added to close a security gap ("accept exactly one CA certificate") broke the only production deployment layout.
- The second round of re-verification ran 3 of the 5 earlier mutants; one of the dropped ones was exactly what the new change broke.

**Check:** re-run the whole previous verification set; replay old-vs-new on prior inputs and variants; keep the replaced rule as a union backstop; run one legitimate configuration against any new constraint.

**Evidence:** `previous mutants N/N killed; replay: 0 inputs caught before and missed now; legit config <name> passes`.

---

## P2 — Fix the class, not the instance

**Phases:** fix

**Pattern:** only the flagged spot is fixed. Siblings with the same defect remain, or the defect is copied into new code.

**Examples**
- A missing assertion was fixed in one test helper and reappeared when the helper was copied into a new test file.
- A retracted sentence was corrected in one table cell and survived in four other places in the same commit.
- An allocation that could throw was wrapped in one `noexcept` function; the identical call in its sibling was left.
- An output-path alias guard compared file paths but missed an intermediate directory symlink.
- A config probe listed 3 of the 7 knobs it was meant to cover.

**Check:** write the defect class in one sentence; search the repository and diff; fix every site or record the count; state the count in the review reply.

**Evidence:** `class: <sentence>; search: <command>; N sites found, N fixed / M tracked at <where>`.

---

## P3 — Claims must match the final artifact

**Phases:** finish, after the last commit, rebase or merge

**Pattern:** statements in commit messages, PR descriptions, review replies, changelogs, trackers or design docs are stronger than what was verified, or go stale after later commits.

**Examples**
- A PR description's test table showed pass counts that matched no log.
- A verification record listed "1 surviving mutant" that was in fact killed by later tests.
- A runbook called a non-atomic `install` command an "atomic rename".
- "All sleeps removed from the test file" while one remained.
- A tracker note said a gap was on the false-positive side; measurement showed it was a miss.
- A commit message said "move, behavior unchanged" for a commit that only added the copy.
- Commit and file counts in a PR description went stale after follow-up commits.

**Check:** after the final change, re-derive every number from logs and git; for each strong word (all, complete, atomic, removed, unchanged, guaranteed) cite the proving command or test, or narrow the sentence; re-read statements written in earlier review rounds too.

**Evidence:** `N claims checked; sources: <logs/commits>; narrowed: <list>`.

---

## P4 — Deleting new code must fail a test

**Phases:** finish (plan: name the intended tests)

**Pattern:** verification targets the test assertions that were written, not the production lines that were added. New assignments, wiring, config propagation and bounds can be deleted with the suite still green.

**Examples**
- All five assignments that propagated a new field through the pipeline could be deleted with the suite green.
- The process startup wiring had no test at all and shipped an infinite recursion.
- Reverting the argument passed from a computed wait time to the actual wait call kept every test green.
- New rule fixtures passed only because a different rule or a fail-closed fallback caught them.
- A concurrency test had no start barrier, so both threads could run one after the other.
- Retry tests depended on the wall clock with a three-second margin.

**Check:** enumerate changed production lines; mutate each and record the failing test by name; add a test for each survivor or record the gap; for each headline claim, name the test that fails when it breaks; scan new tests for wall clock, sleeps and thread ordering.

**Evidence:** `M mutants, K killed (<test names>), S survivors recorded at <where>`.

---

## P5 — Failure paths must be loud and bounded

**Phases:** plan, first review

**Pattern:** error, drop, suppression, fallback and retry branches are silent or have no resource bound.

**Examples**
- A limit-restore call failed and its return value was ignored.
- A recovery scan failure was still marked "done".
- A scheduled job stopped for good without a log line.
- A catch block meant to count lost events did `fetch_add(0)`.
- Observation counters were only ever read by tests.
- A security-critical event was swallowed by a global rate limiter.
- One disk-full error disabled log compaction for the rest of the process lifetime.
- Unbounded reads of verification files, unbounded session buffers, and unbounded copies of query results.
- A persistent misconfiguration emitted a critical event on every reconnect attempt.

**Check:** for each new branch, name the operator-visible surface that reports it ("a test reads it" does not count), the numeric bound, and the behavior at the bound; check system-call and query results fully, not only return codes.

**Evidence:** `branch → surface → bound`, one line each.

---

## P6 — Verify external behavior on the real system

**Phases:** plan

**Pattern:** the design assumes how a library, database, OS or protocol behaves — from memory, documentation or mocks.

**Examples**
- A daemon protocol framing that only the fake test server accepted; the real daemon rejected it.
- A TLS library validity check that silently became a no-op in newer library versions.
- A pin format mismatch (hex vs base64) that would have rejected every legitimate server.
- A database function returning the effective role rather than the login role, letting a role switch pass the identity check.
- A 64-bit identifier the database returns as a signed type.
- A compile-time assertion that is always true under the language version in use.

**Check:** one probe per assumption against the real component, at both the minimum supported and current version when they differ; record the result; state explicitly anything verified only against mocks.

**Evidence:** `assumption → command → observed result (version)`.

---

## P7 — Filesystem trust boundary

**Phases:** plan, first review — only when the change reads, writes or documents filesystem paths

**Pattern:** path handling skips type, owner, mode, symlink, alias, atomic-replace or errno checks.

**Examples**
- A database file's mode depended on the process umask.
- A "not a directory" check used a stat that follows symlinks.
- An audit sink accepted a non-regular file.
- `EEXIST` was folded into "not linked", reopening a race window for an attacker.
- An output path could alias a ledger or key file and destroy it on success.
- A runbook replaced files non-atomically and then told operators to trigger an immediate reload.

**Check:** regular-file check with no symlink following; owner and mode set at creation and verified for existing files; ancestor directory symlinks and writability; inode alias comparison between outputs and inputs; same-directory temp file plus `rename(2)` for replacement (runbooks too); distinguish errno classes.

**Evidence:** `path input → checks applied`.

---

## P8 — A behavior change updates the contract

**Phases:** plan, finish

**Pattern:** emitted events, fields, meanings, error codes or operator instructions change without the interface docs, schemas, generated artifacts and digest pins changing with them. Or a merge from the base branch brings in a contract the branch already violates.

**Examples**
- An event's meaning was widened while the interface document still described the narrow meaning.
- A new audit event was never registered in the interface document.
- Diagnostic attributes violated a closed event schema that was not yet enforced at runtime, so every gate stayed green.
- A documented "success" guarantee was stronger than the actual emission order.
- A one-character edit to a generator input invalidated digests in 24 generated files and several downstream pins.
- A checker that no test ran kept a stale hash pin for weeks.

**Check:** list the changed emitted surfaces; update the contract in the same commit; run the schema validator on real payloads; after merging or rebasing, diff newly arrived contract files against the branch; grep for digests of every edited input to find derived artifacts; regenerate in a clean checkout.

**Evidence:** `surface → contract file → validator result; regenerated: <list>`.
