# ADR-0003: Atomic Lua script for seat hold, hash tag per section

- **Status:** Accepted
- **Date:** 2026-11-15
- **Related:** AC-02, AC-07, US-05, NFR-C02, NFR-S02, NFR-M03, NFR-M05; ADR-0001, ADR-0002

## Context

A hold covers 1 to 4 seats and must be all-or-nothing and idempotent per `userId`. `GET` then `SET` is a race.
In Redis Cluster a multi-key script needs all keys in one slot, otherwise it fails with `CROSSSLOT`. Putting a
whole event in one slot creates a hot shard.

## Decision

We will hold and release seats with **atomic Lua scripts**: check every seat, then set all or none. A repeated
request by the same owner counts as success and does not extend the TTL. Release compares the owner token. We
will key seats as `seat:{eventId:section}:seatId`, so a hold (one section, max 4 seats) stays in one slot while
sections spread across shards. Scripts live in their own files with their own tests (NFR-M05). Order's
`PENDING_PAYMENT` / `SOLD` transitions (ADR-0001) are also Lua scripts with an owner-token check.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| `SET NX` per seat in a loop (simplest) | Trivial code | Not all-or-nothing; manual rollback can leave partial holds. Rejected |
| `WATCH`/`MULTI` | No scripting | Retries under hot-seat contention |
| Hash tag per event | Simple keys | One hot shard per event |
| Lua + hash tag per section (chosen) | Atomic, single slot, load spread | One-section rule; scripts need dedicated tests |

## Consequences

- **Positive:** exactly one winner per seat; no partial holds.
- **Negative / trade-offs:** "all seats in one section" becomes a business rule; a script blocks its shard while it runs, so it must stay short.
- **How we verify it:** 1,000 concurrent requests for 1 seat give exactly 1 win (US-19); concurrency tests run on a real Redis Cluster to catch `CROSSSLOT` (NFR-M03). Not measured yet: shard load balance (NFR-S02) is measured in milestone M6.
