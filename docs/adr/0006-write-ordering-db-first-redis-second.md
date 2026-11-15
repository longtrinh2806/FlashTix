# ADR-0006: Write ordering: DB first, Redis second

- **Status:** Accepted
- **Date:** 2026-11-15
- **Related:** AC-04, US-12, US-16, NFR-D03; ADR-0001, ADR-0002, ADR-0005

## Context

State lives in PostgreSQL and Redis with no distributed transaction between them. When both change (for
example payment confirmed: order `PAID`, seat `SOLD`), a crash between the two writes leaves them inconsistent.

## Decision

We will make Booking write **PostgreSQL first, then update Redis**. The crash window is: DB committed, Redis
not yet updated. In that state the DB (truth) is ahead of Redis, so the unique constraint still blocks any
conflicting booking (ADR-0002), and the **reconciliation job** compares `SOLD` / `PENDING_PAYMENT` in the DB
with Redis and repairs the difference. A rebuild-Redis-from-DB operation exists for larger loss.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Redis first, DB second | Faster perceived response | A crash can leave Redis claiming a sale the DB never recorded. Rejected |
| Two-phase commit | Atomic | Not available across Redis and Postgres; slow |
| DB first, Redis second (chosen, simplest safe order) | Failure leaves the safe side ahead | Needs reconciliation and rebuild tooling |

## Consequences

- **Positive:** inconsistencies can only err on the safe side.
- **Negative / trade-offs:** a short window where Redis lags the DB; reconciliation and rebuild must be built and tested.
- **How we verify it:** chaos tests restart PostgreSQL and Redis under load and the invariant holds (US-21); reconciliation discrepancy metric returns to 0 (US-16). Not measured yet: milestone M6.
