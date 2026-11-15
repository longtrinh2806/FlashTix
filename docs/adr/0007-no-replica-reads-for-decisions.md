# ADR-0007: No replica reads for decisions

- **Status:** Accepted
- **Date:** 2026-11-15
- **Related:** AC-05, US-01, NFR-D02; ADR-0001, ADR-0008

## Context

Replicas lag behind the primary. A decision based on replica data (hold, confirm) can see a seat as free after
it was sold.

## Decision

We will never read from a replica to make a decision. Hold and confirm use only Redis primaries and the
PostgreSQL primary. PostgreSQL replicas serve reporting only. **Exception, display only:** the seat map may
be built from a Redis replica (ADR-0008) because stale data is acceptable there; the hold step still decides.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Read everything from the primary (simplest) | No lag | Display scans add load to the primary. Kept for decisions; rejected for the seat map |
| Read decisions from replicas | Offloads the primary | Replication lag causes wrong decisions. Rejected |

## Consequences

- **Positive:** decisions never rely on stale reads.
- **Negative / trade-offs:** the primary carries all decision traffic; the seat map can be about a second stale.
- **How we verify it:** code review and integration test that decision paths use only primary connections; the seat map is at most 1 s behind (US-01). Not measured yet: seat-map staleness is measured in milestone M5.
