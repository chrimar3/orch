# ROADMAP — orch MVP and the whole system

Final, frozen for Day 0 · 2026-11-23 · Source of truth: Master Spec v1.7 · Companion to ORCH-BUILD.md · Supersedes v1–v3 and the 2026-11-22 final

Part 1 is frozen. Wave 2 and Wave 3 gates are reviewed again at the MVP gate, with real ledger data, not before.

Two roadmaps. The first is the orch MVP, measured in days and gates. The second is the whole system (brain + orch + agents), measured in waves and triggers. The MVP is the first item of Wave 1; nothing in Wave 1 starts before it except what runs in parallel with it by design.

This roadmap sets order and gates. ORCH-BUILD.md sets implementation. Where they differ on the Reviewer identity, protected paths or the local lock, this file wins and ORCH-BUILD.md is updated on Day 0.

## Decisions taken

| # | Decision |
| --- | --- |
| D1 | Wave 1 runs on the Mac only. Windows joins in Wave 2. |
| D2 | The Reviewer's verdict is a required status check posted by a dedicated GitHub App. |
| D3 | Notifications go to two channels. Record: the orch assigns and @mentions Christos on the GitHub issue. Push: ntfy, which still works when GitHub is unreachable. The ntfy topic name is a long random string kept untracked with the credentials. Messages carry only task ID, event and link. |
| D4 | Credential isolation between Builder and Reviewer is an accepted risk in shadow, because a human merges everything. It is a hard requirement before live. |
| D5 | 2026-12-05 is a hard deadline for the shadow MVP. 2027-01-24 and 2027-03-31 are planning targets; a missed gate carries forward. |

## How this version came about

Four review rounds: Claude on v1, GPT Astra on v2, Claude on v3, GPT Astra on the first final. What survived:

| Finding | Resolution here |
| --- | --- |
| "No human action" clashed with shadow mode | The merge click is the single human action in shadow. Live merge is a separate gate. |
| The orch could rewrite its own guardrails | Protected paths need Christos in every mode. |
| A PR author cannot approve their own PR | Builder opens PRs under its own identity; Reviewer App posts a required status on the current head. |
| The MVP proved only the happy path | Three injected bad branches that pass the gates must be rejected by the Reviewer. |
| No failure policy before the scheduler | Failure policy and safe execution come before the scheduler. |
| Reviewer had no criteria | `orch add` requires at least one checkable criterion. |
| v3 made almost every orch task "protected", leaving no eligible tasks | A seeded backlog of non-protected tasks is a Wave 1 item. |
| v3 built two-machine lease logic for one machine | Local lock in Wave 1; lease, takeover and owner checks move to "Windows joins". |
| v3's router trial could never be conclusive at Wave 1 volume | Router v1 waits behind a trigger. |
| v3's Day 0 did not fit its estimate | Merge-authority credentials and Builder sandbox move to "before live". Day 0 is one day. |
| Any model swap forced full requalification | Light requalification for minor version changes. |
| The client-name list was itself a leak | The list stays untracked; only hashes are committed. |
| The Builder could edit a task's criteria to match its work | Criteria are frozen into the run at dispatch; the Reviewer judges against that snapshot. |
| Required checks were switched off during bootstrap | Christos posts his manual verdict through the Reviewer App with the Day 0 script, so the required check never comes off. |
| A paused orch could not review its own revert | Recovery PRs run checks and take a manual verdict while ordinary work is paused. |
| Backlog eligibility was assumed in advance | Eligibility is decided by the actual diff. |

---

## Part 1 — orch MVP

**MVP definition.** Christos submits with `orch add` and starts one `orch run`. That run produces a PR with passing gates and the other vendor's verdict on the current head, with no human action in between. On an ordinary PR the merge click is the only human action after dispatch, and it is the approval. Three injected bad branches that pass the gates are rejected by the Reviewer. The MVP needs no scheduler; the Wave 1 exit does, because there the scheduler performs the dispatch.

**In the MVP (6 components):** Intake (criteria + client-name check), Ledger (GitHub backend), static Router, Dispatcher, Gates, Reviewer.
**Deferred:** Classifier, Planner, Quota telemetry, Scheduler, lease, Omnigraph backend, auto-merge, Builder sandbox.
**Rejected:** UI/cockpit, approval inbox beyond the PR, Hive integration, anything outside the orch.

**Protected paths (need Christos's approval in every mode).** `gates.py`, `review.py`, `route.py`, `routing.yaml`, `tests/`, `.github/`, failure-policy and lock code, `last-good` promotion, credential and identity configuration, this file. Also the dispatcher, test configuration, and any code or config that selects gates, publishes a verdict or authorizes a merge. The exact list is recorded in CODEOWNERS when the repo is seeded and covers moves and deletions; at seed time one representative change per category is shown to require Christos. On a protected PR Christos submits a CODEOWNER review on the current head before merging. On an ordinary shadow PR he only merges. Runs that touch protected paths are logged separately and do not count toward shadow qualification or the Wave 1 exit.

**Identities.** Builder: a bot identity that opens PRs. Reviewer: the GitHub App. Christos: merges, and approves protected changes. Because the Builder is not Christos, he can approve its PRs.

| Step | What | Who | Gate | Estimate |
| --- | --- | --- | --- | --- |
| 0 | Day 0: `gh` auth, repo + labels, Builder bot identity, Reviewer GitHub App, branch protection + CODEOWNERS, one account per config dir, headless smoke tests for `claude -p` and `codex exec`, ntfy topic created and subscribed on the phone. A small script posts a status as the Reviewer App (used for the tests here and for manual verdicts in steps 1–2). Confirm the GitHub plan enforces branch protection on a private repo; if not, enable equivalent enforcement and rerun every blocking test, or mark Day 0 blocked. Record exactly what usage fields each CLI's JSON returns. Findings in STATUS.md; ORCH-BUILD.md updated. | Claude Code, interactive, Mac | `gh`, `claude -p`, `codex exec` return without a login prompt. Merge is blocked in each of three cases: no Reviewer status, a status on an older head, a protected change without Christos's approval. The merge rule accepts the named status from the Reviewer App on the current head and rejects the same status name from any other identity. Usage fields written down. A test ntfy push arrives on the phone and a test @mention arrives from GitHub. | 1 day |
| 1 | Seed: `ledger/` (interface + sqlite + github), `cli.py` (add with ≥1 criterion + client-name check, status, run), `dispatch.py` (worktree + CLI + envelope + PR, per-run timeout, branch name = task ID), `gates.py` (pytest + schema + evidence + no-test-deleted). Lane hardcoded. Client-name hashes in place before the first issue. | Claude Code, interactive | `orch add` → `orch run` → PR open with envelope, tests green. Christos reviews the head, posts his verdict through the Reviewer App with the Day 0 script (logged as manual), adds his CODEOWNER review and merges. Required checks stay on. | 1 day |
| 2 | Task #1 through the orch: static `route.py` + `routing.yaml` + `review.py` (other vendor judges the current head against the task's criteria; App posts status + comment) | orch (Builder Claude, Reviewer Christos this once, same manual procedure as step 1) | PR merged | 2 h of Christos's time |
| 3a | Task #2 through the orch: a small non-protected change (e.g. `orch status` as a table) | orch (Builder Claude, Reviewer Codex) | PR green with Codex verdict on the current head, no human action before the merge click | 2 h (first live run of `review.py`) |
| 3b | Negative fixtures: three pre-made branches, each passing all gates and violating one stated criterion, injected past the Builder. Expected answers are recorded before dispatch and kept out of the Reviewer's input. Criteria are frozen into the run at dispatch; every verdict names task ID, criteria hash and PR head, and the Reviewer reads the frozen copy, not the live issue. | Reviewer Codex | **MVP gate:** 3a merged; all three fixtures receive explicit failing verdicts from the App on their current heads, each citing the violated criterion; merge stays blocked on all three. Editing the issue's criteria after dispatch does not change the verdict of that run. | 2 h |

**Hard deadline: 2026-12-05.** If a gate is still open on that date, the miss and its blocker are recorded in STATUS.md. The gate stays in force.

### After the MVP, still Wave 1 (each is an `orch add`; tasks 3–5 touch protected paths)

| # | Task | Why now | Gate |
| --- | --- | --- | --- |
| 3 | **Failure policy.** Task ID, attempt count and outcome persist across restarts. Max 2 attempts. Timeout kills the worker and its children. `needs-human` label + notification. Kill switch checked before dispatch and before publication; if the switch cannot be read, the orch stops. | Bounds unattended work | A forced failure followed by a restart stops at attempt 2, labels `needs-human`, and both the @mention and the ntfy push arrive. A switch set mid-run blocks publication and new dispatch until reset. With GitHub unreachable, nothing is dispatched and an ntfy push still arrives. |
| 4 | **Safe execution.** Scheduler and recovery run from the `last-good` tag. After each merge, smoke-test that exact commit, then move the tag. On failure, pause dispatch and open a revert PR through normal protections. | A bad merge must not break the thing that fixes it | A broken merge leaves `last-good` on the old commit. A clean revert restores service once smoke passes. A conflicting or protected revert keeps dispatch paused and notifies Christos. While paused, the recovery PR still runs required checks and takes a manual verdict and approval. `last-good` only ever moves forward to a descendant on `main`, so an older smoke result cannot overwrite a newer one. |
| 5 | **Scheduler v0.** launchd runs `orch run next` every 15 min under a local file lock. Before retrying a task, reconcile with any existing branch or PR for that task ID. | Turns "closes without me" into "starts without me" (goal G1) | A task runs with nobody at the keyboard. Two overlapping invocations run one task. A crash after PR creation but before the ledger write leaves one PR and one recorded outcome. |
| 6 | **Quota.** If Day 0 shows usage against limit, parse it; otherwise count runs and tokens per lane per window and treat a rate-limit error as headroom = 0. | First time a lane hits a limit, routing must know | `orch status` shows headroom per lane, marked measured, estimated (with basis) or unknown |
| 7 | **Eligible backlog.** Seed ≥30 small non-protected tasks with criteria: brain upkeep, skill fixes, docs, `orch status` improvements. | Shadow qualification and the Wave 1 exit need eligible tasks | 30 open issues, each with ≥1 criterion, none expected to touch protected paths. Eligibility is decided by the actual diff, not the issue. |

---

## Part 2 — the whole system

Three waves, as in the Master Spec. Each item has a trigger or a gate.

### Wave 1 — "one task a day closes without me" · now → target 2027-01-24

| Item | Depends on | Gate |
| --- | --- | --- |
| orch MVP (Part 1, steps 0–3b) | — | MVP gate |
| Failure policy + safe execution + scheduler (tasks 3–5) | MVP | First unattended run; forced-failure, broken-merge and crash-reconcile tests pass |
| Quota + eligible backlog (tasks 6–7) | Scheduler | Gates of tasks 6 and 7 |
| `goals.md` → `status: active` | Christos | Three goals, each with a metric computable from the ledger and a baseline value |
| `ship-skill` iteration 2 in Claude Code (stale results, RELEASE.txt, platform tool notes, missing file refs) | skill-creator | 4/4 evals green with subagents |
| First client skill through skill-forge/skill-creator → ship-skill | ship-skill it. 2 | Shipped with conformance 3/3 on ≥2 targets (goal G2, 1 of 3) |
| Brain repo created: schema, `goals.md` moves there, client-name hashes + pre-commit, `wiki/lessons/` | — | Phase 2 gate of the Second Brain blueprint |
| Project brain: `brain/` in the first client repo, `/harvest`, `projects/<client>.md` card | brain schema | One harvest ran and the card updated without Christos |
| Ops agent: weekly report every Friday from the ledger | orch, ledger data | First report (goal G3, 1 of 10) |

**Wave 1 exit.** In one seven-day window (Europe/Athens), at least five distinct days each have one eligible task merged from a single `add`, with at most the merge click after submission. Fixtures, drills, protected-path runs and reverted work do not count. Each goal has one ledger-backed data point: a shipped skill for G2, a delivered weekly report for G3. Every eligible submission, dispatch, human intervention and final outcome is recorded in the ledger, and the seven-day summary in `goals.md` is generated from those events.

**If the January target slips,** defer in this order: Ops agent (generate the Friday report by hand from the ledger, which still counts for G3), project brain, quota. Never deferred: failure policy, safe execution, protected paths, eligible backlog. Record the missed exit and carry the work forward.

### Wave 2 — "the system decides what it can decide" · 2027-01-25 → target 2027-03-31

| Item | Trigger | Gate |
| --- | --- | --- |
| Builder + Reviewer out of shadow | Tasks 3–5 passed. Then, in order: (1) the credential boundary is in place: no process that runs candidate code, including the test gate, can read Reviewer or merge credentials or change the verdict publisher's configuration; merge authority has its own credential; (2) 3a and 3b are repeated under that setup; (3) ten consecutive eligible shadow runs for the lane pair (one run counts for both roles) are merged without edits, counting every eligible dispatch in sequence so a failure resets the streak | Christos sets `mode: live` for that lane pair: auto-merge on Reviewer pass, non-protected paths only. The kill switch is checked at the merge decision, and a pause cancels queued auto-merges (shown by a queued PR staying unmerged). A pending review holds its PR. A failed post-merge smoke, a review that never arrives, or a verdict that is invalid at merge time suspends live. |
| Approval inbox + confidence tiers (first priority per Spec v1.5) | Live, ≥50 merged tasks | Human actions per completed task, across all tasks including protected ones, fall by ≥50% against the four weeks before rollout, with counts reported; zero reverts caused by an auto-merged task in 4 weeks. Zero live exposure means inconclusive, not passed. |
| Router v1: choose among eligible lanes by headroom; static route when evidence is thin | ≥3 rate-limit events in one week | Rate-limit events per 100 dispatches over the following 4 weeks, reported with counts against the 4 weeks before. Routing is a reversible config change, so a before/after count is enough evidence. If events do not fall, revert to static. |
| Input isolation: task bodies from transcripts or other submitters are data, run without merge rights, always need a human merge | Before the first non-Christos task source goes live | A synthetic transcript carrying an injected instruction changes nothing outside the task's files and creates no external write |
| Classifier (Haiku): `type`, `difficulty`, `goal_id` | Someone other than Christos submits tasks, or >50 tasks | ≥9 of 10 held-out fixtures correct |
| Planner (Opus): brief with ≥3 checkable criteria | Same trigger | Every new task has criteria without Christos; a deliberately incomplete brief is rejected by the Reviewer |
| Omnigraph "beads" spike (3 h, sandbox) → `ledger/omnigraph.py` only if it passes | Decision 003; a free half day | Interface tests pass on the third backend, or the spike is closed |
| Brain maintainers (Distiller, Linker, Reconciler) as filler tasks | Scheduler + headroom | Inbox ≤20 items for 4 consecutive weeks |
| Meeting agent: transcript → decisions in the brain, tasks in Wrike | `meeting` note type, Wrike MCP, input isolation | One real meeting processed end to end; Wrike writes only within explicit task scope |
| Model radar: weekly check of vendor changelogs; minor best-practice changes auto-apply, new models swap into a role after eval, major changes as PR | Release radar skill | First auto-applied change and first PR. Requalification: a minor version passes the role's positive and negative fixtures plus 3 shadow runs; a new model family repeats the full 10. |
| Omnigraph as brain index (Linker writes nodes/edges from markdown) | A server running, or ≥3 agents writing concurrently | 3 pre-written questions answered correctly that a grep baseline gets wrong |
| BA agent pilot on the first client engagement (kickoff → harvest) | `company-ba` plugin, project brain, Intake client-name check | Harvest into `work/` with zero client-name hits in commits and issues |
| Windows machine joins: Day 0 there, `machine:mac` label, lease as a git ref with renewal and expiry | Scheduler stable for 4 weeks | Two contenders yield one owner. An owner paused past expiry cannot publish after takeover. Both machines run tasks for a week; zero tasks run twice. |

### Wave 3 — "the system proposes its own next step" · 2027-04 →

| Item | Trigger | Gate |
| --- | --- | --- |
| Architect agent: monthly proposals to the approval inbox with leverage scores | Ledger ≥3 months, lessons ≥20 | First approved proposal |
| Omnigraph graph-canonical brain (Spec v2) | The company asks for a multi-client company brain with contractual data isolation | Decision record, not before |
| Hive decision: dispatcher backend `hive` on Windows | Hive exposes a headless start + completion hook, and the orch has run blind for ≥2 months | Cost/benefit against the `cli` backend, with data |
| Company product pilot: client as-is mapping → MCP connection → skills packaged as a plugin, one client only | Three client skills shipped through the factory | Pilot scoped with one client and one success metric |
| Fourth lane: company ChatGPT | Quota data shows the three lanes saturate | One accepted shadow task and one rejected fixture on the new lane, reviewed by the other vendor |

---

## Rules that hold across all waves

- Nothing enters a wave without a gate it can fail. A gate with no exposure is inconclusive, not passed.
- Every addition to the Master Spec scores leverage ≥ 2.0 and gets a decision record.
- Every new agent starts in shadow. Shadow means a human merges; live means auto-merge on non-protected paths.
- Protected paths need Christos in every mode.
- The orch builds its own components; the seed is the only hand-written code. Exception: repairing a broken `last-good` is done by hand and logged as a lesson.
- Client names never enter the ledger or the repos. The plain list stays untracked on the machine; only hashes are committed. Intake checks title, body and criteria before the first ledger write; the same check runs before comments, PR text and commits. A missing list pauses publication. Committed text uses client aliases.
- If a vendor blocks headless subscription use, that lane is disabled in `routing.yaml`, the orch runs single-vendor with Christos as Reviewer, live merge is suspended, and a decision record is opened. Those runs do not count toward qualification. No silent switch to API keys.
- When a trigger has not fired, the item waits. Waiting is a valid state.
