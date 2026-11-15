# ADR-0002: Redis as fast filter, PostgreSQL unique constraint as fencing layer

- **Status:** Accepted
- **Date:** 2026-11-15
- **Related:** G1, G2, AC-01, NFR-C01, NFR-R02; ADR-0001, ADR-0003

## Context

About 14.4k hold requests per second compete for a limited number of seats, 80% on 20% of the seats. Deciding
each request in PostgreSQL puts all contention on the master. Redis alone cannot guarantee correctness: a
failover can lose recent writes, a hold can expire while a client is paused, and messages can be redelivered.

## Decision

We will use Redis as a **lock for efficiency**: it picks the winner on the hot path and rejects losers with 409
without touching the DB. We will use a unique constraint on `(event_id, seat_id)` in `order_seats` as the
**fencing layer**, so PostgreSQL rejects any wrong operation whatever Redis or the client believes.
`HELD` / `PENDING_PAYMENT` (with TTL) live in Redis; orders and `SOLD` seats live in PostgreSQL.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| PostgreSQL only, conditional `UPDATE` (simplest) | Always correct, one store | Master takes the whole spike and hot-seat lock contention. Rejected as the final design; kept as the baseline in ADR-0011 |
| Redis only | Very fast | Failover or expiry can double-sell; no durable record |
| Redis + DB constraint (chosen) | Fast losers; correctness independent of Redis | Two stores to keep consistent |

## Consequences

- **Positive:** losers get 409 without a DB write; a Redis failover cannot cause a double sale.
- **Negative / trade-offs:** dual-write between Redis and DB (ADR-0006); reconciliation and rebuild tooling are required (US-16).
- **How we verify it:** k6 correctness test, 1 seat and 1,000 requests, exactly 1 win (US-19); invariant SQL returns 0 rows after load and chaos tests (US-20, US-21). Not measured yet: milestone M2 adds the Redis hold, M6 runs the load and chaos tests.
