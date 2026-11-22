# STATUS — orch

## Current state
- Day 0 (ROADMAP step 0): in progress
- Seed (step 1): not started
- Tasks through the orch (steps 2, 3a, 3b): —

## Day 0 findings
Observed in the session of 2026-11-23:

| Item | Result |
| --- | --- |
| 1. `gh auth` | Pass. Logged in as `chrimar3`, scopes `gist, read:org, repo, workflow`, no login prompt. |
| 2. Repo + labels | Pass. Repo `chrimar3/orch` created, **public** by Christos's decision (see item 5). Labels: all `state:*`, `type:implement/review/triage`, `difficulty:normal/hard`, `lane:claude`, `lane:codex`, `needs-approval`, `needs-human`. Merge commits only (squash and rebase disabled in repo settings). |
| 3. Identities | Not done. Builder bot account and Reviewer App need Christos. |
| 4. `scripts/post_verdict` | Written (stdlib + `openssl` binary for the RS256 signature). Not yet run against the App: no App exists yet. Unverified. |
| 5. Branch protection | The plan does not enforce branch protection on a private repo (API 403: "Upgrade to GitHub Pro or make this repository public"). Work stopped; Christos decided to make the repo public. Protection itself: pending the Reviewer App. |
| 6. Config dirs | Not done. `~/.claude-personal`, `~/.claude-company`, `~/.codex-personal` do not exist yet; logins need Christos. |
| 7. Headless smoke tests | Not done (depends on 6). |
| 8. ntfy topic | Topic generated into untracked `.ntfy-topic` (not printed anywhere). Phone not yet subscribed. |
| 9. Merge-blocking tests | Not done (depends on 3 and 5). |
| 10. Test push and @mention | Not done. |

Tooling found: `gh` 2.86.0, `uv` 0.9.25, `claude` 2.1.330, `codex-cli` 0.167.0. Python 3.12.12 installed through `uv python install 3.12` (it was missing).

CLI flags seen in `--help` (not yet exercised):
- `claude`: `-p/--print`, `--output-format`, `--permission-mode`, `--allowedTools`, `--disallowedTools`, `--model`, `--effort`, `--add-dir`, `--settings`, `--setting-sources`, `--json-schema`, `--no-session-persistence`, `--bare`.
- `codex exec`: `--json`, `-o/--output-last-message`, `--output-schema`, `-m/--model`, `-s/--sandbox`, `-C/--cd`, `--skip-git-repo-check`, `--ephemeral`, `--ignore-user-config`.

Incident log:
- One direct commit to `main` (the root commit: docs, `.gitignore`, `judge/`, `skills/`, `.github/CODEOWNERS`, `scripts/post_verdict`), approved by Christos in the session, because `main` had no commits and PRs need a base. No further commits to `main` by Claude Code.
- The repo is public, which departs from ROADMAP step 0 and ORCH-BUILD C.2 ("private repo"). Decision by Christos on 2026-11-23. The first private repo was renamed and replaced by a fresh one with a rewritten root commit, so the earlier history is not in the public repo. Commits use the GitHub no-reply address.

## Conflicts and open questions
- `judge/rubric.md` and `judge/prompt.md` already existed before the session; the overnight prompt says to write them in B1. The existing files will be used unchanged unless Christos says otherwise.
- CODEOWNERS lists `orch/failure.py`, `orch/schedule.py` and `orch/lastgood.py` before they exist (names assumed for Wave 1 tasks 3–5), and adds `CLAUDE.md`, `.gitignore`, `scripts/` and `clients.sha256` as credential/identity/rules configuration.

## First command of the next session
Continue Day 0 from item 3 (identities), then item 5.
