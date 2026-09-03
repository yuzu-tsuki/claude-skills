---
name: token-saver
description: Orchestrate one slice of work through the project's agent presets — architect designs a blueprint, implementer codes it, logic-auditor reviews adversarially, architect dispositions the findings — looping until converged, then finish with the commit ritual (delegated to the project's own finishing skill when one exists). Use when starting implementation of a picked slice on its feature branch, OR when working reviewer feedback on an open MR back into the branch (prefix "review-fix:"). Prefix the slice with "simple:" for the fast path that skips the blueprint, "opus-architect:" to design on the opus preset from a plan tier without Fable access, or "no-commit:" to run full verification but leave the work uncommitted with drafts in the scratchpad.
argument-hint: "[simple:] [review-fix:] [opus-architect:] [no-commit:] <slice description>"
---

# Token saver

Run the slice through the `.claude/agents/` presets. Copies of those presets are bundled in
`agents/` next to this file so the skill can be shared as one unit — see `agents/README.md` for
installation; the bundled copies are inert until installed into `.claude/agents/`.

**You are the orchestrator**: the agents cannot call each other (none has the Agent tool) — every handoff below goes through you. Do the whole loop in one continuous run; the only pause points are commit boundaries and blockers that need the user (CLAUDE.md §2).

Three mechanics apply to every step:

- **Handoffs are files, not prose relays.** Save each agent's report verbatim to the session scratchpad (`blueprint.md`, `findings-r1.md`, `dispositions-r1.md`, `findings-r2.md`, ... — one file per report per round) and hand downstream agents the absolute path plus a two-line orientation, never a paraphrase. The gatekeeper later reads these files as its evidence. Save each report the moment it arrives — an unsaved report that fails to deliver is unrecoverable, a saved one is.
- **Launch agents you will continue — reviewers and the architect — as plain background agents; continue via SendMessage.** Do not give these launches a `name` — the named-teammate mailbox mode has dropped final reports in practice (2026-09-03: two named auditors went idle without delivering; the unnamed background relaunches both delivered via task notification). An unnamed launch returns an agent id; send round-2+ work (delta reviews, disposition follow-ups) to that id via SendMessage so the agent keeps its review context. If a reviewer goes idle without a report: request it once via SendMessage, wait briefly, then relaunch fresh as another unnamed background agent — never sit in a polling loop.
- **Findings carry IDs.** Auditors number findings (F-1, ..., dimension-prefixed when fanned out); the architect dispositions every ID; deferred IDs are cited in the tracker. The gatekeeper mechanically checks that no ID lacks a disposition.
- **A round is closed only when every reviewer launched for it has delivered.** Do not start the gates (step 5) while any reviewer of the current round is still outstanding — a late report after gates/commit forces a second full fix-gate-commit cycle (measured 2026-09-03: it roughly doubled the round). If you decide to abandon a stuck reviewer, do it explicitly: stop it, note the abandonment in the tracker, and only then proceed.

## Path selection

Read the invocation arguments before anything else. Flags are colon-suffixed words at the front of the argument string, may appear in any order, and everything after the last flag is the slice description.

- **`simple:`** (equivalent form: `small:`; e.g. `/token-saver simple: fix the off-by-one in journal compaction`) — take the fast path: skip step 1 and launch a single implementer with the slice statement, then run steps 3–6 unchanged.
- **`review-fix:`** (e.g. `/token-saver review-fix: MR !35 3차 리뷰 반영`) — the MR-feedback path; see the "Review-fix path" section below. Steps 1–2 are replaced by fetch-and-disposition; the auditor and gatekeeper still run.
- **`opus-architect:`** (equivalent forms: `cheap-architect:`, `architect-v2:`) — run step 1 and every later disposition round on the `architect-v2` preset (opus, effort max) instead of `architect` (fable-5). Use it on the cheaper plan tiers, which do not include Fable model access. Substitute the preset wherever these instructions say `architect`; nothing else about the loop changes. Never launch both design presets on the same slice — one design authority per slice, or the blueprint has two owners.
- **`no-commit:`** — the user wants the work implemented and verified but not committed. Steps 1–4 run unchanged; step 5 runs the full gates but replaces every git write with a scratchpad draft (see the no-commit paragraph there); step 6 still runs, with the commit-shape checks marked N/A. Verification is never reduced by this flag — only the git writes are.
- **No flag** — full path on the `architect` preset, step 1 onward.
- **Prose instead of a flag** ("this is a tiny fix", "skip the design", "use the opus architect", "don't commit anything") — honor it, but say in one line that you read it as that flag, since the trigger was judgment rather than the literal form.

If a launch of `architect` fails because the account cannot reach the Fable model, retry once on `architect-v2` and say you switched — do not treat a model-access error as a design blocker.

`simple:` and `opus-architect:` combine, but the combination is moot: the fast path launches no architect at all, so the design preset only matters if a fallback to step 1 happens. `review-fix:` combines with `opus-architect:` (the disposition role runs on that preset) and excludes `simple:` — the review-fix path already is the reduced shape. `no-commit:` combines with `simple:` and `opus-architect:`, but not with `review-fix:` — working an MR's feedback is commit-shaped by definition, so treat that combination as a contradiction and ask the user which they meant.

The auditor and gatekeeper are **never** skipped on any path — ceremony scales with risk, verification does not. If a fast-path implementer reports a design question or a blueprint gap, the slice was not simple: fall back to step 1 with the design preset in effect and say so.

## 0. Preconditions

- **Presets installed.** Check `.claude/agents/` for the seven preset files this skill launches (`architect`, `architect-v2`, `implementer`, `logic-auditor`, `docs-conformance`, `gatekeeper`, `quick-question`). Copy any missing one there from the bundled `agents/` folder next to this file — copy only, **never overwrite an existing name** (it may be the user's own preset; report the skip instead), and never copy `README.md`. A preset copied mid-session may not be launchable until a new session: if the next launch still fails with an unknown agent type, stop and tell the user to restart the session once — the install holds from then on. When you copied into a repo other than the skill's home, add one line that the briefs carry repo-specific content to adapt (`agents/README.md` lists it).
- The user has picked the slice and the work happens on a `features/*` branch off `develop` (create with `--no-track` per CLAUDE.md §2 if it doesn't exist yet). On a `no-commit:` run the branch ceremony is optional — staying on the current checkout is fine, since nothing will be committed to it.
- State the slice in one or two sentences before launching anything; if the slice itself is ambiguous, ask the user first — agents inherit your framing.

## 1. Architect — blueprint

Launch `architect` with: the slice statement, the branch, and pointers to the docs it must verify against (decisions SSOT `docs/dev-plan/00-open-decisions-resolved.md`, the ICD, the module's implementation tracker).

- Expected output: a blueprint — affected files/namespaces, interfaces and seams, data shapes, error handling, test plan, explicit non-goals, cited D-/B-/R- IDs, and the work-package partition with PARALLEL-SAFE markings. Save it to the scratchpad as `blueprint.md`.
- If the architect reports a conflict, open decision, or ambiguity: **stop and put it to the user**. Do not resolve it yourself and do not let a downstream agent resolve it.

## 2. Implementer — code (fan out when the blueprint allows)

The blueprint partitions the work into packages and marks which are PARALLEL-SAFE (strictly disjoint file sets, no compile-time dependency between packages).

**Implementers never build, test, or run anything — the preset has no shell tool.** You own the single build-and-test point of each round; this split is what keeps the loop out of fix–build–fix churn:

- After the implementer(s) return, build **incrementally in the existing build tree** and run the fast dev suite (whatever the repo's quick build-and-unit-test loop is) **once**. Full commit gates stay deferred to the commit ritual.
- **Builds are warm.** Never delete, clean, or reconfigure the build directory inside the loop. A cold configure happens at most once per slice, on the first build of a fresh checkout — the review-fix path's branch switch already calls that case out.
- Send **all** failures from that one run back to the owning implementer as a single batch with the exact error output, let it fix everything in one pass, then rebuild once. One rebuild per fix-batch, never one per fix. If the same failure survives three fix–build round-trips, stop and put it to the user instead of running a fourth.

Launch shape:

- **One package, or sequential blueprint**: launch a single `implementer` with the `blueprint.md` path plus any user rulings from step 1. It writes code and unit tests, then reports.
- **Two or more parallel-safe packages**: launch one `implementer` per package **in a single message** so they run concurrently. Each prompt carries the `blueprint.md` path, the package's exact file list, and the rule: touch only the owned files. After all return: diff `git status` against the union of the declared file lists — any file outside a package's ownership is treated as corruption and investigated before anything else (CLAUDE.md §4 live-repo multi-agent rule) — then do the single build-and-test run above, routing each failure to the package that owns the failing file.
- Cap the fan-out at what the packages genuinely support — do not split work to manufacture parallelism; a package too small to justify an agent rides along with its nearest neighbor.
- If it reports a blueprint gap, relay the gap back to `architect` for a blueprint delta, then re-launch the implementer with the delta. Never redesign on the implementer's behalf.
- It does not commit and does not touch the tracker; those belong to the ritual in step 5.

## 3. Logic-auditor — adversarial review (fan out over large diffs)

Launch `logic-auditor` (unnamed, per the mechanics above) with the diff scope (files touched), the `blueprint.md` path when one exists (the `simple:` path has none — hand it the slice statement instead), and the implementer's report. This is the CLAUDE.md §1 adversarial review — the auditor argues against the work.

**The first pass fixes the surface for the whole slice.** Whatever perspective it takes — and whatever it fails to take — is the perspective the slice is reviewed under from here on. Later passes narrow; they never widen. This is what makes the loop terminate: round N+1's scope is a strict subset of round N's, so the rounds are bounded by the finding count, not by how much of the code a fresh reader could still find fault with. Put the effort into scoping this pass well, because it is the only one that gets to choose.

- **Large diff** (roughly: several hundred changed lines or multiple modules): launch the reviewers concurrently in a single message — `logic-auditor` scoped to correctness/edge-cases (prefix F-), `logic-auditor` scoped to security (prefix FS-), and `docs-conformance` for the SSOT/ICD/tracker surface (prefix FD-). Merge nothing yourself; the architect dispositions the union in step 4.
- **Any slice that touches a documented contract** — ICD fields, a D-/B-/R- decision, module paths, tracker status — gets `docs-conformance` even when the diff is small and a single `logic-auditor` covers the code. It is cheap (sonnet) and its surface is the one the logic-auditor is instructed not to duplicate.
- Expected output: ID'd findings with severity, concrete failure scenario, `file:line`, and a minimal fix suggestion; confirmed defects separated from unverified concerns. Save each report as `findings-r<round>[-<dimension>].md`.
- **Comments are never findings — in either direction.** Stale or wrong comments (drift) go in one trailing "comment sweep" batch, unnumbered and severity-free; fix the batch in one pass and do not review the result. A missing or "insufficient" comment is not reportable at all: judge the source by whether it explains itself — naming, extraction, types — and if it doesn't, the finding is the code change, never "add a comment". Inside the loop comments only shrink or disappear (measured in practice: admitting comment findings made every round grow the comments until they were paragraphs). Explanation that must exist goes to the project docs as one compact line — and the doc files stay compact too.
- The auditor is read-only. If a future review step needs an agent that runs or modifies anything, give that agent `isolation: 'worktree'` and diff `git status` afterward before staging.

## 4. Architect — disposition, then converge

Send the findings file path(s) to the **same architect** via SendMessage (it still holds the blueprint context) for disposition: every finding ID becomes **fix now** (with a blueprint delta), **defer** (with a reason to record in the tracker/SSOT), or **invalid** (with a rebuttal). Save the result as `dispositions-r<round>.md`.

On the `simple:` path no architect exists: disposition the findings yourself, inline (same as the review-fix path), and still write `dispositions-r<round>.md` — the gatekeeper checks the file, not who wrote it. If a finding needs a genuine design ruling rather than a fix, that is the fall-back signal from "Path selection": the slice was not simple — launch the design preset with the findings and continue on the full path.

- Fix-now deltas go back to the implementer (step 2), then the changed area gets a delta pass from the **same auditor** via SendMessage; it continues its finding numbering.
- **A delta pass asks one question: was the disposed feedback correctly applied?** It does not re-review the code, and it does not open a surface or perspective the first pass did not take — not on the fix's newly written lines either. A finding is admissible in a delta pass only if it shows a prior finding was **not actually closed** (the fix is wrong, incomplete, or its test does not pin what it claims). Anything else the auditor notices — a new angle, an adjacent concern, a fresh dimension — is recorded as a residual in the tracker for a future slice, **not** fixed in this loop.
  - The orchestrator enforces this in the brief. Do not write "attack the new logic specifically" into a delta pass; that is a first pass wearing a delta pass's name, and it restarts the round count (measured 2026-09-03: a delta pass briefed to attack newly written fix code returned four new findings and turned a two-round slice into three).
  - The implementer may iterate as many times as the feedback needs. It is the **reviewer's** scope that is forbidden to grow, not the fix count.
- **No disposition may add or expand a comment.** If a finding's real complaint is unclear code, the fix-now delta names the code change; if it is a missing explanation, the delta is one compact line in the tracker/docs. Reject any disposition draft that grows source prose — comments only shrink inside the loop.
- **A delta pass is required only when logic or tests changed.** Comment-only and doc-only fixes skip review entirely — they cannot change behavior, and reviewing them is what turns one round into three (measured 2026-09-03: 14 of 23 findings on one MR were stale comments, each fix re-triggering a pass). Batch comment corrections into the next round that has real code in it.
- Converge — don't loop indefinitely. When a delta pass reports every disposed finding correctly closed, the loop is done. If the same disagreement survives two rounds, put it to the user instead of a third round.
- **Two reviewers may run in the same round only on disjoint surfaces** (for example `logic-auditor` on code and `docs-conformance` on the SSOT/ICD/tracker). Never put two reviewers on the same surface: they diverge, and the orchestrator becomes a judge of other agents' verdicts instead of the work. An auxiliary auditor — the repo-local `agy` preset where it exists, otherwise a fresh unnamed `logic-auditor` launch — is a **tie-breaker for a contested verdict**, invoked after the fact — not a second opinion run alongside by default.

## Review-fix path (`review-fix:` — MR feedback rounds)

Working an MR reviewer's feedback is a recurring shape this loop otherwise leaves to improvisation. It replaces steps 1–2; steps 3–6 apply with the deltas below.

1. **Fetch the feedback and the branch state first.** `git fetch` + `glab mr view <n> --comments`; check out the MR's source branch (expect the build trees to be stale after the switch — the first gate run will be a full rebuild; say so in the checkpoint rather than treating it as an anomaly). Save the reviewer's comment verbatim to the scratchpad as `mr<n>-feedback-r<round>.md`.
2. **Disposition before coding.** For each reviewer item, decide fix-now / defer-with-registration / rebut — this is the architect's role; on slices where you hold the design context yourself, do it inline but still write `dispositions-r<round>.md`. A reviewer item that needs the user's ruling pauses the loop, same as a blueprint conflict.
3. **Implement the fix-now items**, then run step 3 (auditor — scoped to the fixes plus anything the reviewer's reasoning implicates) and step 4 as usual. The round-closure rule applies with full force here: every reviewer launched must report before gates start.
   - **Use the reviewer wait — don't idle through it.** While the auditor runs (measured ~16 min at xhigh on a full round's diff), draft everything that doesn't depend on its verdict: the tracker section, the reply-file skeleton, the mutation list, the commit message bodies. Mark each draft as pending-verification and reconcile it against the findings when they land — never publish or commit a draft the round hasn't confirmed.
4. **Commit shape**: docs (SSOT/ICD/ask) as a leading commit, code fix, tracker — the established three-way split. Append commits; never rewrite history the reviewer has seen.
5. **Write the reply file** (scratchpad, delta-only per CLAUDE.md §2's reply rules) after the gatekeeper verdict, so the numbers it cites are the verified ones.

## 5. Commit ritual

Everything below runs **once**, after the loop has converged — this is the full-gate run that steps 2–4 deliberately deferred. Before starting it, make sure every report from steps 3–4 is saved to the scratchpad, and that any code changed after the last auditor pass got a delta pass.

**On a `no-commit:` run**, skip the delegation below entirely — a finishing skill commits on its own authority — and run the ritual with these substitutions: item 1 (full gates) runs unchanged, ceremony scales with risk but verification never shrinks; item 2's tracker update is drafted to the scratchpad as `tracker-draft.md` instead of applied to the repo docs; item 3 becomes drafting the commit message(s) to `commit-drafts.md` with nothing committed or staged; item 4's checkpoint report says "verified, uncommitted" and points at the drafts so the user can commit later with one paste.

**If the project defines its own finishing skill, invoke it and let it own this step.** (This loop was built alongside one named `subtask-done`; a well-built finishing skill detects the saved `findings-*.md` + `dispositions-*.md` artifacts and skips spawning a second reviewer of its own — `subtask-done` §1 does.) The ritual carries the findings table into its final report.

**Otherwise, run the ritual yourself:**

1. **Full gates, once.** Run the project's complete commit-gate suite — full build, entire test suite, sanitizers/analyzers, formatter check; whatever the repo's CLAUDE.md or CI defines as required before a commit. The loop only ran the fast dev suite; everything else runs here, exactly once.
2. **Update the project's tracking docs**, if it keeps them (implementation tracker, changelog): what the slice changed, plus every deferred finding ID with its recorded reason.
3. **Commit** per repo convention — code and tracker/doc changes as separate commits where the repo separates them. Do not push or open an MR unless the user asked.
4. **Checkpoint report** to the user: the slice, rounds run, the findings table (every ID with its disposition), and the gate results — delivered only after the step 6 gatekeeper verdict.

## 6. Gatekeeper — verify before the callout

Before declaring the result commit-ready in the checkpoint report — **on every path, the review-fix path included** — launch `gatekeeper` (tier: commit-ready) with the branch name and the scratchpad paths of every `findings-*.md` and `dispositions-*.md` file — it reads those files as evidence and rejects prose summaries. Tell it which reviewers were launched this round; one of its checks is that every launched reviewer's report file exists (the round-closure rule made mechanical — a missing file means the round is not closed, regardless of gate results). On a `no-commit:` run, declare that too: it marks the commit-shape checks N/A instead of UNVERIFIED and verdicts on gates, review completeness, and format alone. It independently checks gate-log evidence, branch/upstream state, tracker separation, and that the adversarial review actually happened — it trusts disk and git, not your claims. Report its per-check verdicts; if it says not ready, fix the blocking checks and re-run it before the callout. Launch it again at tier MR-ready (rebase freshness, post-rebase gates, squash state, open decisions) when the user asks to prepare an MR.

## Quick lookups

`quick-question` (read-only, fast) answers trivial or context-breaking lookups at any point without disturbing the loop — use it instead of derailing an in-flight agent.
