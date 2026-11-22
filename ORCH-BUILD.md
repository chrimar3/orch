# ORCH-BUILD — brief for Claude Code

Date: 2026-11-23 (revised from 2026-11-09 to match the frozen ROADMAP.md) · Source of truth: Master Spec v1.7 (https://claude.ai/code/artifact/6ee3934d-72d4-4fbe-9232-945158b3efef) plus decisions `spec/decisions/003`, `004`, and decisions D1–D5 in ROADMAP.md (to be recorded as v1.8).

ROADMAP.md sets order and gates. This file sets implementation. If they disagree, ROADMAP.md wins and you flag it in STATUS.md. If something is covered nowhere, stop and ask — do not improvise architecture.

## What we are building

`orch`: an orchestrator that takes a task with acceptance criteria, routes it to a lane (account + model + effort) through the subscription CLIs, runs it in a git worktree, hands it to the other vendor for review, and closes it as a PR. The orch itself is the first project it builds: after the seed (section D), every component is a task submitted to the orch.

MVP success (ROADMAP Part 1): Christos submits with `orch add` and starts one `orch run`. That run produces a PR with passing gates and the other vendor's verdict on the current head, with no human action in between. His merge click is the only human action and is the approval. Three injected bad branches that pass the gates are rejected by the Reviewer.

## A. Decisions (closed)

1. **Subscription CLIs only.** `claude -p` and `codex exec`. No API keys anywhere. If you find an API key in the environment, do not use it.
2. **Wave 1 runs on the Mac only.** The Windows laptop joins in Wave 2. One account = one config directory (`CLAUDE_CONFIG_DIR`, `CODEX_HOME`); the lane says which. Keep code portable (`pathlib`, no bash-only assumptions in code the orch runs).
3. **Ledger = interface.** Backends: `github` (default, Wave 1), `sqlite` (tests, offline), `omnigraph` (later, behind the same interface — do not implement now).
4. **GitHub as ledger:** issue = task, labels = state/type/lane, comments = events, PR = run result and approval inbox.
5. **Three identities.** Builder: a bot account that opens PRs. Reviewer: a dedicated GitHub App that posts the verdict as a required status check on the current PR head; the same status name from any other identity is rejected. Christos: merges, and gives a CODEOWNER review on protected changes. Credentials live outside the repo. Isolation between Builder and Reviewer credentials is an accepted risk in shadow and a requirement before live.
6. **Every run in its own worktree, never on main.** Branch `agent/builder-<task_id>`: the name is derived from the task ID alone, so a rerun finds the existing branch and PR. Merge commits only, no squash or rebase, so a verdict stays attached to its head.
7. **Review always by the other vendor.** Claude builds → Codex reviews, and vice versa.
8. **Criteria.** `orch add` refuses a task without at least one checkable criterion. At dispatch the criteria are frozen into the run and hashed (`criteria_hash`). Builder and Reviewer both work from the frozen copy; editing the issue afterwards does not affect that run.
9. **Client names.** The plain list is an untracked local file; only hashes are committed. Intake checks title, body and criteria before the first ledger write, and the same check runs before comments, PR text and commits. A missing list pauses publication.
10. **Protected paths.** Recorded in `.github/CODEOWNERS` at seed time, covering moves and deletions: `orch/gates.py`, `orch/review.py`, `orch/route.py`, `orch/dispatch.py`, `routing.yaml`, `envelope.schema.yaml`, `tests/`, test configuration, `acceptance/`, `judge/`, `.github/`, failure-policy and lock code, `last-good` promotion, credential and identity configuration, `ROADMAP.md`.
11. **Notifications.** GitHub assign + @mention as the record; ntfy as the push. Messages carry task ID, event and link only. The topic name is untracked.
12. **Stack:** Python 3.12, `uv`, `typer` for the CLI, `pytest`, `pyyaml`. Nothing else without asking.

## B. Architecture

```
orch/
├── orch/
│   ├── cli.py            # typer: add, status, run                (seed)
│   ├── intake.py         # criteria check, client-name check      (seed)
│   ├── ledger/
│   │   ├── base.py       # Ledger interface                       (seed)
│   │   ├── github.py     # gh CLI backend                         (seed)
│   │   └── sqlite.py     # tests/offline backend                  (seed)
│   ├── dispatch.py       # worktree + CLI call + envelope + PR    (seed)
│   ├── gates.py          # tests/schema/evidence/no-test-deleted  (seed)
│   ├── route.py          # routing.yaml → lane, static            (task #1)
│   ├── review.py         # other vendor, verdict via the App      (task #1)
│   ├── schedule.py       # launchd + local lock                   (Wave 1, task #5)
│   ├── quota.py          # counters from CLI usage JSON           (Wave 1, task #6)
│   ├── classify.py       # Haiku: type, difficulty, goal_id       (Wave 2)
│   └── plan.py           # Opus: brief with criteria              (Wave 2)
├── routing.yaml
├── envelope.schema.yaml
├── goals.md              # lives here for now (GOALS_PATH); moves to the brain later
├── skills/
│   ├── builder/SKILL.md  # Builder role skill (orch-roles)
│   └── reviewer/SKILL.md # Reviewer role skill
├── scripts/
│   └── post_verdict      # posts a status as the Reviewer App (Day 0; manual verdicts in steps 1–2)
├── acceptance/           # frozen acceptance tests and fixtures (overnight run)
├── judge/                # frozen rubric and judge prompt
├── tests/
├── .github/CODEOWNERS
├── STATUS.md             # Claude Code updates it at every gate
├── CLAUDE.md
├── ROADMAP.md
└── ORCH-BUILD.md
```

### Ledger interface (`ledger/base.py`)

```python
class Ledger(Protocol):
    def create_task(self, title: str, body: str, criteria: list[str], labels: list[str]) -> TaskId: ...
    def get_task(self, task_id: TaskId) -> Task: ...
    def list_tasks(self, state: str | None = None, labels: list[str] = ()) -> list[Task]: ...
    def claim(self, task_id: TaskId, worker: str, lease_minutes: int) -> bool: ...   # False if already claimed with an active lease
    def release(self, task_id: TaskId, worker: str) -> None: ...
    def set_state(self, task_id: TaskId, state: str) -> None: ...                   # queued|claimed|building|review|approved|merged|failed|needs-human
    def append_event(self, task_id: TaskId, kind: str, payload: dict) -> None: ...
    def write_envelope(self, task_id: TaskId, envelope: dict) -> None: ...
    def read_envelope(self, task_id: TaskId) -> dict | None: ...
```

Quota and lock methods are not part of the MVP interface. Quota arrives with task #6. Wave 1 locking is a local file lock in the scheduler (task #5); the shared lease arrives with the Windows machine in Wave 2.

The `sqlite` backend implements the same interface, for tests. Interface tests run against both backends (parametrize); in tests the GitHub backend uses a throwaway repo or a mock of `gh`.

### GitHub mapping

| Concept | GitHub |
| --- | --- |
| task | issue; body = request + criteria |
| state | label `state:<queued\|claimed\|building\|review\|approved\|merged\|failed\|needs-human>` |
| type / difficulty / lane | labels `type:<…>`, `difficulty:<normal\|hard>`, `lane:<…>` |
| claim | assignee = worker (`mac`), comment `lease: <worker> until <ISO>` |
| event | comment `event: <kind>` + JSON block |
| frozen criteria | comment `event: dispatched` with the criteria text and `criteria_hash` |
| envelope | PR description (plus a comment on the issue with the link) |
| run result | PR from branch `agent/builder-<id>`, opened by the Builder bot, label `needs-approval` |
| verdict | required status check on the PR head from the Reviewer App, plus a PR comment with the verdict YAML |
| approval | PR merge by Christos; on protected paths, his CODEOWNER review first |

### Envelope (`envelope.schema.yaml`)

```yaml
task_id: ""
role: builder | reviewer
lane: ""                     # e.g. claude:personal:sonnet-5.5:high
criteria_hash: ""            # hash of the criteria frozen at dispatch
started: ""                  # ISO
ended: ""
summary: ""                  # 3 lines
changed_files: []
tests: { ran: 0, passed: 0, failed: 0 }
criteria: [{ text: "", met: true|false, evidence: "" }]
confidence: 0.0              # 0–1, declared by the agent
usage: { tokens_in: 0, tokens_out: 0 }
blockers: []
next: ""                     # what is needed to proceed
```

The dispatcher adds the head SHA and gate results to the PR description after the push. The verdict carries `task_id`, `criteria_hash` and `head_sha`.

### routing.yaml (shape)

```yaml
lanes:
  claude:personal:sonnet-5.5:high:
    cli: claude
    config_dir: ~/.claude-personal
    model: sonnet
    effort: high
  claude:company:opus-5.5:high:
    cli: claude
    config_dir: ~/.claude-company
    model: opus
    effort: high
  chatgpt:personal:gpt-6.1-sol:high:
    cli: codex
    config_dir: ~/.codex-personal
    model: gpt-6.1-sol
rules:
  - match: { type: implement, difficulty: normal }
    lanes: [claude:personal:sonnet-5.5:high, chatgpt:personal:gpt-6.1-sol:high]
  - match: { type: review }
    rule: other_vendor_than_builder
  - match: { type: triage }
    lanes: [claude:personal:haiku-5.5:low]
escalation: [effort_up, haiku, sonnet, opus, fable_or_astra]
choose_among_lanes: first            # static in Wave 1; max_quota_headroom arrives with Router v1 (Wave 2)
fable_budget_pct: 15
```

Verify exact model names and CLI flags with `claude --help` and `codex --help` at implementation time — do not trust this file for flags.

## C. Day 0 — prerequisites (Mac, with Christos at the keyboard)

This is ROADMAP step 0. Write no product code before everything passes. Christos does by hand anything that creates an account or App, logs in, or needs a password or code. Secrets go into untracked files or the keychain, never into the chat or the repo.

1. `gh auth status` → logged in to the account that owns the repo.
2. Private repo `orch` on GitHub. Labels created: all `state:*`, `type:*`, `difficulty:*`, `lane:*`, `needs-approval`.
3. Identities: the Builder bot account with write access to the repo; the Reviewer GitHub App installed on the repo with permission to write commit statuses and PR comments only.
4. `scripts/post_verdict`: posts a named status on a given head SHA as the Reviewer App.
5. Branch protection on `main`: PR required, the Reviewer App's status required from that App specifically, CODEOWNER review required on protected paths. Confirm the GitHub plan enforces this on a private repo. If it does not, enable equivalent enforcement and rerun every test in item 9, or mark Day 0 blocked.
6. Config dirs: `~/.claude-personal`, `~/.claude-company`, `~/.codex-personal`. For each: log in to the right account. Verify the CLI honours the variable (`CLAUDE_CONFIG_DIR`, `CODEX_HOME`) — if not, stop and report the correct mechanism.
7. Headless smoke tests, from a clean shell, with no interactive prompt:
   - `CLAUDE_CONFIG_DIR=~/.claude-personal claude -p "reply with the single word ok" --output-format json` → JSON. Record exactly which usage fields it returns.
   - Same for `~/.claude-company`.
   - `CODEX_HOME=~/.codex-personal codex exec "reply with the single word ok"` → answer with no login prompt. Find the flag for structured/JSON output and record its usage fields.
   - Note which permission flag `claude -p` needs to write files without prompting inside a worktree, and keep it as narrow as possible (no blanket skip if a narrower option exists).
8. ntfy: topic with a long random name, stored untracked; Christos subscribes on his phone.
9. Merge-blocking tests, on throwaway PRs opened by the Builder bot. Each must be shown blocked:
   - no Reviewer status;
   - a Reviewer status on an older head;
   - the same status name posted by an identity other than the App;
   - a protected-path change with the App status but without Christos's CODEOWNER review.
   And one must be shown mergeable: an ordinary change with the App status on the current head.
10. A test ntfy push arrives on the phone and a test @mention arrives from GitHub.
11. Write the findings to `STATUS.md` under "Day 0 findings".

**Day 0 gate:** items 1–11 pass and are recorded.

## D. Day 1 — the seed (you write this by hand)

ROADMAP step 1. Only this. Lane hardcoded; no classifier, router, reviewer or scheduler.

1. `ledger/base.py`, `ledger/sqlite.py`, `ledger/github.py` + tests parametrized over both backends.
2. `intake.py` and `cli.py`: `orch add "<title>" --criterion "<text>" [--criterion …] [--body file]` refuses a task with no criterion and refuses any title, body or criterion that matches the client-name hashes; otherwise creates the issue, state:queued. `orch status` lists issues by state. `orch run <id>` does claim → dispatch → gates → PR.
3. `dispatch.py`: freeze the criteria and record `criteria_hash` on the issue; worktree `../orch-wt/<id>` from the base branch, branch `agent/builder-<id>`; call the lane's CLI with prompt = `skills/builder/SKILL.md` + the task body + the frozen criteria + the allowed files + an instruction to write the envelope to `.orch/envelope.yaml`. Per-run timeout that kills the worker and its children. Commit, push, open the PR as the Builder bot with the envelope as description, label `needs-approval`, state:review. Idempotent: if the branch or a PR for that task ID exists, reconcile with it and do not create a second one.
4. `gates.py`: `pytest` passes in the worktree; the envelope exists and validates against the schema; every criterion has evidence; no test was deleted, skipped or weakened by the diff. Failure → state:failed, comment with the reason, worktree kept for inspection.
5. `.github/CODEOWNERS` with the protected paths of A.10. Client-name hashes committed; the plain list stays untracked.
6. `STATUS.md`: update "Seed" with the PR number.

**Day 1 gate (ROADMAP step 1):** the seed is a PR on a branch, tests green. With the seed checked out, `orch add` → `orch run` opens a PR with an envelope. Christos reviews the head, posts his verdict with `scripts/post_verdict` (logged as manual), adds his CODEOWNER review and merges. Required checks stay on.

## E. After the seed — tasks through the orch

Each of these is submitted with `orch add`, not written by hand. Numbering follows ROADMAP.md. The task body is the row text verbatim plus the criteria: "tests for every public function; envelope with evidence per criterion; no changes outside the listed files."

| # | Task | Files | Acceptance |
| --- | --- | --- | --- |
| 1 | Static router + Reviewer: `routing.yaml` → lane by rule (first candidate, no quota); `review.py` runs the other vendor with prompt = `skills/reviewer/SKILL.md` + frozen criteria + diff at the head SHA, and posts the verdict through the Reviewer App as a status on that head plus a PR comment | `route.py`, `review.py`, `routing.yaml` | ROADMAP step 2. Reviewer this once is Christos, by the manual procedure of section D. PR merged. |
| 2 | Any small non-protected change (e.g. `orch status` as a table) | `cli.py` | ROADMAP step 3a. Builder Claude, Reviewer Codex. PR green with Codex's verdict on the current head, no human action before the merge click. |
| 2b | Negative fixtures: three pre-made branches, each passing all gates and violating one stated criterion, injected past the Builder. Expected answers kept out of the Reviewer's input. | fixture branches only | ROADMAP step 3b. **MVP gate:** 3a merged; all three get failing verdicts from the App on their current heads, each citing the violated criterion; merge stays blocked. |
| 3 | Failure policy: persisted attempt count, max 2 attempts, timeout, `needs-human`, kill switch, notifications | per ROADMAP task 3 | ROADMAP task 3 gate |
| 4 | Safe execution: run from `last-good`, post-merge smoke test, revert on failure | per ROADMAP task 4 | ROADMAP task 4 gate |
| 5 | Scheduler v0: launchd runs `orch run next` every 15 min under a local file lock; reconcile before retry | `schedule.py`, launchd plist | ROADMAP task 5 gate |
| 6 | Quota: parse usage from the CLIs' JSON if Day 0 found usage against limit; otherwise count per lane per window | `quota.py` | ROADMAP task 6 gate |
| 7 | Eligible backlog: ≥30 small non-protected tasks with criteria | issues only | ROADMAP task 7 gate |

Wave 2, each behind its trigger in ROADMAP.md: Router v1 (`max_quota_headroom`, escalation ladder), Classifier (`classify.py`, Haiku, reads `GOALS_PATH`), Planner (`plan.py`, Opus, ≥3 criteria), the Omnigraph "beads" spike, the Windows machine and the shared lease.

## F. Working rules for you (Claude Code)

- Never commit to `main`. Even the seed lands through a PR. Never merge, approve, or post a Reviewer status.
- One commit per step of section D, message prefixed `seed:`.
- Stop at every **gate** and wait. Do not continue "to save time". Exception: a session started with `PROMPT-orch-mvp-overnight.md` stops only where that prompt says, and builds steps 1–3b as a stack of PRs for Christos to merge.
- Install nothing beyond A.12 without asking.
- Do not write the router, reviewer, scheduler, classifier or planner in the seed, even if easy. They are orch tasks — the orch building them is the test.
- Every assumption about CLI flags goes into `STATUS.md` with the `--help` output that confirms it.
- At each gate, update `STATUS.md`: what happened, what remains, first command of the next session.
