# ADR-0005: Async order creation through an at-least-once queue with an idempotent consumer

- **Status:** Accepted
- **Date:** 2026-11-15
- **Related:** AC-03, US-09, US-10, US-11, NFR-C03, NFR-R03, NFR-R06; ADR-0001, ADR-0006

## Context

After winning in Redis, the order must reach PostgreSQL. A synchronous write ties API latency to the DB and
exposes the master to the spike. A standard queue is at-least-once with no ordering guarantee.

## Decision

We will have Reservation publish a command to the RabbitMQ `commands` queue and return **202** with a
`requestId`; if publishing fails it releases the hold and returns 503. Order will be **idempotent** and
order-independent:
- Unique violation `23505` on `request_id`: duplicate, ack and skip.
- Unique violation on `(event_id, seat_id)`: real conflict, order `FAILED`, hold released if still owned, user told before payment. The two are told apart by **constraint name**.
- Transient errors (timeout, deadlock) are not acked; retry with backoff and jitter, DLQ after N attempts, with a replay tool.
- A message older than the hold TTL has its owner token re-checked first.

We will cover the gap between writing the order and publishing the event with an outbox (ADR-0001).

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Synchronous DB write in the request (simplest) | No queue, immediate result | Latency tied to the DB; master takes the spike. Rejected |
| FIFO / exactly-once queue | Fewer duplicates | Lower throughput; idempotency still needed |
| At-least-once + idempotent consumer (chosen) | Throughput, resilience, controllable write rate | Status tracking via SSE/polling; more consumer logic |

## Consequences

- **Positive:** duplicates and reordering are harmless; failures are retried, not lost.
- **Negative / trade-offs:** results are eventually consistent; DLQ operations are needed.
- **How we verify it:** integration test delivering a message N times equals once (NFR-C03); chaos tests kill a consumer and duplicate messages (US-21). Not measured yet: the p95 under 5 s target (NFR-P04) is measured in milestone M3/M6.
