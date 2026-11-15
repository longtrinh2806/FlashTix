# ADR-0009: PgBouncer and bounded consumer concurrency to protect the master

- **Status:** Proposed (needs measurement)
- **Date:** 2026-11-15
- **Related:** G2, NFR-S05, NFR-P06, US-10; ADR-0001, ADR-0005

## Context

Each Booking instance has its own Npgsql pool (default maximum 100). With several instances the total can
exceed PostgreSQL `max_connections`, and each Postgres connection is a separate process.

## Decision

We will run **PgBouncer** in transaction pooling mode between Booking and the PostgreSQL primary, point Npgsql
at it with `No Reset On Close=true` and `Max Auto Prepare=0`, and **limit concurrent consumers** and pool sizes
so the total stays under the PgBouncer and Postgres limits. We will avoid session-level features (session
advisory locks, long-lived `SET`) on that path.

## Alternatives considered

| Option | Pros | Cons |
|---|---|---|
| Npgsql pool only (simplest) | Nothing extra to run | Total connections grow with instance count. Rejected if measurement shows we exceed `max_connections` |
| Raise `max_connections` | Trivial | More memory, worse performance, only delays the problem |
| PgBouncer (chosen) | Bounded server connections | Extra component; transaction-mode limits |

## Consequences

- **Positive:** scaling consumers cannot exhaust master connections.
- **Negative / trade-offs:** one more component to run and monitor; some Postgres features unavailable.
- **How we verify it:** scale consumers in the load test (US-20) and watch PostgreSQL connections and lock waits (NFR-O01). Not measured yet: milestone M6 decides whether PgBouncer is needed at our scale; move to Accepted with that data.
