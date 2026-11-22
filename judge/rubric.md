# Judge rubric — orch MVP (ROADMAP.md Part 1, steps 1–3b)

Score each aspect from 0 to 10. An aspect passes at 8. Score only what you can point to: a file and line, a test name, or PR/status state. If you cannot find evidence, the score is low; do not give credit for intent.

## Scoring anchors (same for every aspect)

- **0–2** Missing or does not run.
- **3–5** Present, but a required clause is unmet or a mapped acceptance test is red.
- **6–7** Every clause is met, with a real weakness: a gap a second failure would expose, or behaviour shown only on the happy path.
- **8** Every clause is met and shown by a test or by observed PR state. Remaining issues are cosmetic.
- **9–10** As 8, and edge cases named in the roadmap are tested too. Nothing you would change before merging.

## Aspects

1. **Roadmap conformance.** Every clause of the step 1, 2, 3a and 3b gates is met as written. Nothing on the Deferred or Rejected lists was built.
2. **Tests and gates.** `gates.py` enforces pytest, schema, evidence and no-test-deleted. The ledger interface passes the same tests on the sqlite and GitHub backends. Tests would fail if the behaviour broke (check for tests that assert nothing).
3. **Reviewer correctness.** The 3a PR has a passing verdict from the Reviewer App on its current head. Each of the three fixture PRs has a failing verdict from the App on its current head, citing a specific criterion, and is unmergeable. The Reviewer reads criteria frozen at dispatch, and each verdict names task ID, criteria hash and head SHA.
4. **Guardrails.** CODEOWNERS covers every protected category in the roadmap, including moves and deletions. No secret, token, ntfy topic or plain client name appears in the repo, PRs or issues. Intake checks title, body and criteria against client-name hashes before the first ledger write.
5. **Failure behaviour.** The per-run timeout kills the worker and its children. Branch name equals task ID; running the same task twice leaves one branch and one PR. Every error path ends in a recorded outcome.
6. **Simplicity.** Small modules, no speculative abstraction, no dead code, no component beyond the six in the MVP. A reader can follow `orch add` to an open PR in one sitting.
7. **Evidence.** Each PR carries its envelope: task ID, criteria hash, head SHA, gate results, verdict. STATUS.md matches what the tests and PRs actually show.
