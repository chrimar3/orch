# orch — repo rules

Read `ROADMAP.md` (order and gates; Part 1 is frozen), `ORCH-BUILD.md` (how to build it) and `STATUS.md` (where we are) first. Do not re-explain the system; start from "First command of the next session" in STATUS.md.

- Precedence: Master Spec v1.7 as amended by the decisions in ROADMAP.md, then ROADMAP.md, then ORCH-BUILD.md. Flag any conflict you find in STATUS.md. If something is covered nowhere, ask.
- Python 3.12, `uv`, `typer`, `pytest`, `pyyaml`. Nothing else without asking.
- Never commit to `main`. Branch + PR always. Merge commits only, no squash or rebase.
- Never merge, approve, or post a Reviewer status. Christos merges; only the Reviewer lane posts verdicts, through the Reviewer App.
- No API keys. Only `claude -p` and `codex exec` through the config dirs in `routing.yaml`.
- Wave 1 runs on the Mac only. Keep code portable (`pathlib`, no shell-only commands in code the orch runs) so Windows can join in Wave 2.
- Tests for every public function. Ledger tests run parametrized over all backends.
- No secret, token, ntfy topic or plain client name in the repo, PRs or issues. Use client aliases.
- Stop at every gate in ORCH-BUILD.md and update `STATUS.md` before stopping. Exception: a session started with `PROMPT-orch-mvp-overnight.md` stops only where that prompt says.
