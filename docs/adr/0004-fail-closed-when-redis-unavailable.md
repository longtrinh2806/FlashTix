# ADR-0004: Fail closed when Redis is unavailable

- **Status:** Accepted
- **Date:** 2026-11-15
- **Related:** G2, AC-06, NFR-R01, NFR-R04, NFR-R07, US-21; ADR-0001, ADR-0002

## Context

If Redis is down, Reservation could reject requests or fall back to deciding in PostgreSQL. A fallback would
send the whole spike to the master, which the architecture exists to prevent.

## Decision

We will make Reservation **fail closed**: on Redis unavailability or timeout it returns **503** (with
`Retry-After` and jitter). It never falls back to the DB. Every outbound call has a timeout, retry and circuit
breaker so requests never hang.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Fall back to a DB conditional `UPDATE` | Sales continue | Master overloaded; can take everything down. Rejected |
| Fail closed (chosen, simplest) | Master protected, predictable | Sales pause while Redis is down |
| Per-instance in-memory state | Fast | Wrong across instances |

## Consequences

- **Positive:** the master survives a Redis outage.
- **Negative / trade-offs:** lower availability during an outage; recovery may need a rebuild from the DB (US-16).
- **How we verify it:** chaos test "stop Redis" (US-21): 503 returned, master load stays flat, invariant holds. Not measured yet: the 60 s recovery target (NFR-R05) is measured in milestone M6.
