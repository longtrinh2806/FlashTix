# ADR-0001: Service boundaries and data ownership

- **Status:** Accepted
- **Date:** 2026-11-15
- **Related:** G1, G2, AC-01, AC-07, NFR-S01, NFR-S05; ADR-0002, ADR-0005, ADR-0008, ADR-0010

## Context

The system must serve 60k concurrent users and about 14.4k peak hold RPS in a 10 s spike while keeping the
master DB protected (G2). It is a pet project: one repo, several deployables. The hot path (hold seats), the
write path (persist orders) and the push path (SSE connections) have different load profiles and bottlenecks.

## Decision

We will build three core services plus a waiting room (detailed in ADR-0010).

| Service | Responsibility | Scales with | Data it owns | Why separate |
|---|---|---|---|---|
| **Reservation** | Hot path: Lua hold/release in Redis, publish commands, return 202/409 | RPS | HELD keys in Redis and the key schema. No PostgreSQL access | CPU-bound at peak RPS; must never wait on the DB |
| **Order** | Consume commands; own PostgreSQL (`orders`, `order_seats`, outbox); payment webhooks, event/inventory admin and reconciliation job as modules | Number of consumers, bounded by master capacity | PostgreSQL | Only writer to master, so write rate is controlled in one place |
| **Notification** | Hold SSE connections; build and serve the seat-map snapshot | Number of open connections | Nothing (read-only) | Long-lived connections must not compete with the hot path for CPU |

**Redis access is deliberately shared state. Reservation owns the key schema.**

| Service | Redis permission |
|---|---|
| Reservation | Read/write (Lua hold/release) |
| Order | Restricted write: extend to `PENDING_PAYMENT`, mark `SOLD`, both via Lua with owner-token check |
| Notification | Read-only (snapshot scan) |

**Channels:**
- `commands` (Reservation → Order) is a point-to-point queue with competing consumers.
- `events` (Order → Notification) is a pub/sub fanout (see ADR-0008). Order publishes events through an outbox.

**Not separate services:** payment handling, event admin and reconciliation live inside Order as modules.
The payment gateway is an external mock.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Modular monolith, one deployable (simplest) | Easy to run and debug, no network hops | Hot path, DB writes and SSE share CPU and scale together; a DB slowdown or SSE load degrades holds. Rejected: the project must prove the master is protected and each tier scales on its own (US-20) |
| Three services (chosen) | Independent scaling; only Order touches master | More operational cost; Redis shared across services |
| Payment as its own service | Isolates payment failures | A fourth deployable for a mock gateway and a few endpoints; needs its own DB access to `orders` |
| Single Notification instance | No fan-out problem | Connection ceiling; single point of failure |

## Consequences

- **Positive:** the API tier scales by adding Reservation instances; write rate to master is capped in one place.
- **Negative / trade-offs:** more deployables to run locally; Redis is shared, so a key-schema change touches every service; correlation by `requestId` and tracing are mandatory (NFR-O02, NFR-O03).
- **How we verify it:** only Order has PostgreSQL credentials (config review); Notification's Redis user is read-only (ACL test); the stepwise load test (US-20) adds Reservation instances and shows the API tier scaling. Not measured yet: scaling figures come from milestone M6.
