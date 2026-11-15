# ADR-0008: Notification/Realtime: fanout exchange across instances and a single snapshot builder reading from a Redis replica

- **Status:** Proposed
- **Date:** 2026-11-15
- **Related:** US-01, US-02, US-09, AC-07, NFR-P03; ADR-0001, ADR-0007

## Context

Notification/Realtime runs several instances, each holding SSE connections for some users. Booking events
(order status, seat sold) must reach the instance holding the right user's connection. The seat map must be
cheap to serve to tens of thousands of clients without hitting the master.

## Decision

**Events:** Booking publishes to a fanout (or topic) exchange. Each Notification instance binds its own
exclusive, auto-delete queue and filters events against its local connection map. A shared work queue would
deliver each event to only one instance, which may not hold the user's connection.

**Snapshot:** ONE builder instance scans Redis per section with `MGET` (possible thanks to the per-section hash
tag, ADR-0003), from a Redis replica, about once per second, and writes the snapshot. All instances read the
snapshot and push deltas over SSE.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Single Notification instance (simplest) | No routing or builder election | Connection ceiling; single point of failure |
| Per-user routing with a user→instance table | Each event goes to one instance only | Extra state to keep correct; stale entries on reconnect |
| Every instance scans Redis itself | No builder to elect | Scan load multiplies with instance count |
| Publish HELD events from the hot path | Most real-time | Adds publish cost to the hottest path and floods the broker at peak |
| Fanout + single builder (chosen) | Simple routing; scan load constant | Builder needs failover; each instance receives all events |

## Consequences

- **Positive:** no routing table; Redis scan load does not grow with instance count.
- **Negative / trade-offs:** every instance receives every event; the builder is a single point until failover exists. Events missed while an instance is down are acceptable: clients resync from the order-status endpoint or the snapshot on reconnect.
- **How we verify it:** integration test with two instances, an event reaches the instance holding the connection; reconnect test resyncs from the snapshot (US-02); seat-map p95 under 200 ms (NFR-P03). Not measured yet: milestone M5 measures it.
