# ADR-0011: Baseline (PostgreSQL only) vs final architecture

- **Status:** Proposed (needs benchmark data)
- **Date:** 2026-11-15
- **Related:** G4, US-22, NFR-S03, NFR-S04; ADR-0002, ADR-0005, ADR-0009, ADR-0010

## Context

Redis, a queue, a waiting room and PgBouncer add complexity. A component should stay only if measurement shows
it is worth it. The simplest correct approach is PostgreSQL alone with a conditional `UPDATE` and the unique constraint.

## Decision

We will build the baseline first (milestone M1), measure it, then add components one by one (M2 to M6) and re-run
the **same k6 scenario** after each step, recording keep or drop for each component and why.

| Metric | Baseline | + Redis hold | + Queue/Booking | Final |
|---|---|---|---|---|
| Max RPS of one container (4 vCPU / 8 GB) | Not measured yet | Not measured yet | Not measured yet | Not measured yet |
| p95/p99, winners; p95, losers (409) | Not measured yet | Not measured yet | Not measured yet | Not measured yet |
| Master CPU, lock waits, writes to master | Not measured yet | Not measured yet | Not measured yet | Not measured yet |
| Invariant violations | Not measured yet | Not measured yet | Not measured yet | Not measured yet |

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Keep only the baseline (simplest) | Least complexity | Unknown whether it meets the targets; decided by data, not assumption |
| Build the final architecture without comparing | Faster | No evidence that each component earns its cost |
| Staged comparison (chosen) | Every component justified by data | Needs repeatable load tests (open model, k6 on a separate machine, watch `dropped_iterations`) |

## Consequences

- **Positive:** the final architecture is justified, or simplified, by data.
- **Negative / trade-offs:** discipline and time for repeatable benchmarking.
- **How we verify it:** the table above is filled from US-22 and US-20 runs, with the invariant checked after each run (US-19). Not measured yet: milestones M1 and M6; then set the status to Accepted.
