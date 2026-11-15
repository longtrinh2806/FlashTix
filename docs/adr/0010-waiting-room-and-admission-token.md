# ADR-0010: Waiting room with signed admission token

- **Status:** Proposed
- **Date:** 2026-11-15
- **Related:** US-03, US-04, NFR-F01, NFR-F02, NFR-SEC01, NFR-R04; ADR-0001

## Context

At sale open up to 60k users arrive at once. Letting all of them reach Reservation overloads it and users fail
at random. Users should be admitted fairly at a rate the system can handle.

## Decision

We will put a **waiting room** in front of Reservation that admits users at a configurable rate (N per second),
close to arrival order, and issues a **signed, time-limited token**. The hold API rejects requests without a
valid token (401/403); an expired token means rejoining the queue. One user has one queue position and one
valid token regardless of tabs. Overload beyond that returns 429 with `Retry-After` and jitter.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| No waiting room, rate limit only (simplest) | Cheap | Unfair (fast clients win); poor user experience. Rejected as the sole control; still used for abuse (US-18) |
| Autoscaling only | No new component | The spike is too sudden to scale into |
| Waiting room + token (chosen) | Fair, bounded load | Extra component and state; must absorb the full population |

## Consequences

- **Positive:** Reservation sees a controlled rate; fairness is explicit.
- **Negative / trade-offs:** added latency before seat selection; token handling must be secure.
- **How we verify it:** test that one user cannot take several slots (NFR-F02); admission order matches queue order within small deviations (NFR-F01); hold API rejects missing or expired tokens (NFR-SEC01). Not measured yet: milestone M5 builds it; load test (US-20) shows its effect.
