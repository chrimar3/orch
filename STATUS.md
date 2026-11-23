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
| 3. Identities | Pass. Builder: bot account `MaestroOmirou`, `write` on the repo; token is a classic PAT with scope `public_repo`, expiring 2027-02-21, in `.secrets/builder.token`. Reviewer: GitHub App `orch-reviewer`, installed on selected repositories (this one), permissions `metadata: read`, `pull_requests: write`, `statuses: write`, no event subscriptions. Read back through the API, not taken on trust. |
| 4. `scripts/post_verdict` | Partly verified. The JWT signature and the installation lookup work against the real App (`GET /app`, `GET /repos/…/installation`). The status POST and the PR comment have not been exercised yet (see item 9). |
| 5. Branch protection | Partly done. The private-repo plan does not enforce protection (API 403), so the repo is public by Christos's decision. On `main` now: PR required, CODEOWNER review required, 0 approving reviews otherwise, `enforce_admins` on, force-push and deletion off. **Missing: the required status check `orch/reviewer` pinned to the Reviewer App.** The API call that adds it was refused by the Claude Code permission classifier in the session of 2026-11-23; Christos has to allow it or add it in the repo settings. |
| 6. Config dirs | Pass. All three exist and are logged in. `CLAUDE_CONFIG_DIR` is honoured: the two dirs hold different accounts in different organisations (`claude_max` and `claude_team`). `CODEX_HOME` is honoured: the run wrote its logs under `~/.codex-personal` and needed no login. |
| 7. Headless smoke tests | Pass, from `env -i` with only `HOME`, `PATH`, `USER`, `LOGNAME`, `SHELL`. All three returned `ok` with no prompt. Details below. |
| 8. ntfy topic | Topic generated into untracked `.ntfy-topic` (not printed anywhere). Phone not yet subscribed. |
| 9. Merge-blocking tests | Not done (depends on 3 and 5). |
| 10. Test push and @mention | Not done. |

Tooling found: `gh` 2.86.0, `uv` 0.9.25, `claude` 2.1.330, `codex-cli` 0.167.0. Python 3.12.12 installed through `uv python install 3.12` (it was missing).

Headless findings (items 6–7), observed 2026-11-23:
- **`claude -p` needs `USER` in the environment.** With `env -i HOME PATH` only, both dirs answered `Not logged in · Please run /login` as JSON with `is_error: true`; adding `USER` and `LOGNAME` fixed it. The dispatcher and the launchd plist must pass `USER`. The exit code of the failing call was not captured.
- A failed login comes back as well-formed JSON, so the dispatcher must read `is_error`, not only parse the output.
- `claude -p --output-format json`, top level: `type`, `subtype`, `is_error`, `result`, `stop_reason`, `terminal_reason`, `num_turns`, `duration_ms`, `duration_api_ms`, `session_id`, `uuid`, `total_cost_usd`, `usage`, `modelUsage`, `permission_denials`, `api_error_status`, `safety_stops`, `subagent_stats`, `fast_mode_state`.
  - `usage`: `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, `output_tokens_details.thinking_tokens`, `cache_creation.ephemeral_5m_input_tokens`, `cache_creation.ephemeral_1h_input_tokens`, `server_tool_use`, `service_tier`, `speed`.
  - `modelUsage.<model id>`: `inputTokens`, `outputTokens`, `cacheReadInputTokens`, `cacheCreationInputTokens`, `thinkingTokens`, `costUSD`, `contextWindow`, `maxOutputTokens`, `canonicalModel`, `provider`, `costBasis`.
  - `total_cost_usd` is a list-price figure (`costBasis: list`), not a charge against the subscription.
- `codex exec --json` emits JSON lines: `thread.started`, `turn.started`, `item.completed`, `turn.completed`. `turn.completed.usage`: `input_tokens`, `cached_input_tokens`, `cache_write_input_tokens`, `output_tokens`, `reasoning_output_tokens`. `-o <file>` wrote the final message. Run with `--skip-git-repo-check -s read-only` and stdin closed.
- **Neither CLI reports usage against a limit.** Task 6 (quota) therefore counts runs and tokens per lane per window.
- Default models seen: personal dir answered with `claude-fable-5-2`, company dir with `claude-opus-5-5`; `--model sonnet` resolved to `claude-sonnet-5-5`. The dispatcher must always pass `--model`.
- Write permission: `claude -p --permission-mode acceptEdits` created a file in a scratch git repo with `permission_denials: []`; an `echo` through the shell tool also ran. Running `pytest` through the shell tool is not yet exercised; `--allowedTools` is the candidate if it is refused.

CLI flags seen in `--help`; exercised so far: `claude -p`, `--output-format json`, `--model`, `--effort`, `--permission-mode acceptEdits`, `--no-session-persistence`; `codex exec --json`, `-o`, `-s read-only`, `--skip-git-repo-check`:
- `claude`: `-p/--print`, `--output-format`, `--permission-mode`, `--allowedTools`, `--disallowedTools`, `--model`, `--effort`, `--add-dir`, `--settings`, `--setting-sources`, `--json-schema`, `--no-session-persistence`, `--bare`.
- `codex exec`: `--json`, `-o/--output-last-message`, `--output-schema`, `-m/--model`, `-s/--sandbox`, `-C/--cd`, `--skip-git-repo-check`, `--ephemeral`, `--ignore-user-config`.

Incident log:
- One direct commit to `main` (the root commit: docs, `.gitignore`, `judge/`, `skills/`, `.github/CODEOWNERS`, `scripts/post_verdict`), approved by Christos in the session, because `main` had no commits and PRs need a base. No further commits to `main` by Claude Code.
- The repo is public, which departs from ROADMAP step 0 and ORCH-BUILD C.2 ("private repo"). Decision by Christos on 2026-11-23. The first private repo was renamed and replaced by a fresh one with a rewritten root commit, so the earlier history is not in the public repo. Commits use the GitHub no-reply address.

## Conflicts and open questions
- `judge/rubric.md` and `judge/prompt.md` already existed before the session; the overnight prompt says to write them in B1. The existing files will be used unchanged unless Christos says otherwise.
- CODEOWNERS lists `orch/failure.py`, `orch/schedule.py` and `orch/lastgood.py` before they exist (names assumed for Wave 1 tasks 3–5), and adds `CLAUDE.md`, `.gitignore`, `scripts/` and `clients.sha256` as credential/identity/rules configuration.

- **Conflict.** CLAUDE.md says Claude Code never posts a Reviewer status. ORCH-BUILD C.9 needs statuses from the Reviewer App on throwaway PRs, and Day 0 is assigned to Claude Code. Until Christos decides, Claude Code prepares the PRs and Christos runs `scripts/post_verdict` himself.
- Protection has `dismiss_stale_reviews: false`, so a CODEOWNER review given on an older head survives a new push. ROADMAP asks for the review "on the current head". Proposed: turn `dismiss_stale_reviews` on together with the required check. Not changed.
- "Require branches to be up to date" (`strict`) is proposed off: with merge commits only, an update merge would move the head and void the verdict. Not specified in ROADMAP or ORCH-BUILD.
- The Builder has `write`, so once the required check passes it could merge its own ordinary PR; a personal repo cannot restrict who merges. This falls under D4 (accepted in shadow, separate merge credential before live).
- Untracked `AGENTS.md` is a copy of CLAUDE.md in which the CLI rule reads "`Codex -p` and `codex exec`"; it should read `claude -p`. Left untouched.

## First command of the next session
Day 0 item 5: add the required status check `orch/reviewer`, pinned to the Reviewer App, to `main` (needs Christos to allow the call or do it in the settings). Then item 9 on throwaway PRs opened with the Builder token, item 10, and the phone subscription of item 8.
