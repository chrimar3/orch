---
type: goals
period: 2026-11-09 → 2027-02-07   # rolling 90 days
reviewed: 2026-11-09
status: draft                     # becomes active once Christos approves it
---
# Mission
A system that does my work with the least possible effort from me, proven at my employer before it becomes my own product.

# Goals

## G1 · One task a day closes without me
metric: tasks with state:merged where no event has actor=christos between task.created and review.approved — from the ledger
target: ≥ 5 per week by 2027-02-07; first one by 2026-11-23
serves: orch
weight: 0.5

## G2 · Client skills through the production line
metric: skills shipped via ship-skill with conformance 3/3 on every target
target: 3 skills, ≥ 2 targets each
serves: company
weight: 0.3

## G3 · The system measures itself
metric: Ops weekly reports in the ledger, consecutive Fridays
target: 10 consecutive by 2027-02-07; first by 2026-11-27
serves: agents
weight: 0.2

# Open problems
- The brain repo does not exist yet; goals.md lives in the orch until it does.
- The Omnigraph/beads spike has not been run.

# Out of scope (this quarter)
- Architect agent, Release radar, BA agent in production.
- Omnigraph as brain or index.
- Anything for the investment system or real estate through the orch.
