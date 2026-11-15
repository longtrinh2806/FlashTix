# Architecture Decision Records

Each ADR records one significant decision: the context, the choice, and its consequences.
Format: [template.md](template.md). Numbering is sequential and never reused. A decision that is
replaced gets status `Superseded by ADR-XXXX`; the old file is kept.

Statuses: `Proposed` · `Accepted` · `Superseded` · `Deprecated`

| ADR | Title | Status |
|---|---|---|
| [0001](0001-service-boundaries-and-data-ownership.md) | Service boundaries and data ownership | Accepted |
| [0002](0002-redis-fast-filter-postgres-fencing.md) | Redis as fast filter, PostgreSQL unique constraint as fencing layer | Accepted |
| [0003](0003-atomic-lua-hold-with-section-hash-tag.md) | Atomic Lua script for seat hold, hash tag per section | Accepted |
| [0004](0004-fail-closed-when-redis-unavailable.md) | Fail closed when Redis is unavailable | Accepted |
| [0005](0005-async-booking-via-queue-with-idempotent-consumer.md) | Async booking through an at-least-once queue with an idempotent consumer | Accepted |
| [0006](0006-write-ordering-db-first-redis-second.md) | Write ordering: DB first, Redis second | Accepted |
| [0007](0007-no-replica-reads-for-decisions.md) | No replica reads for decisions | Accepted |
| [0008](0008-notification-fanout-and-snapshot-builder.md) | Notification/Realtime: fanout exchange across instances and a single snapshot builder reading from a Redis replica | Proposed |
| [0009](0009-pgbouncer-and-bounded-consumers.md) | PgBouncer and bounded consumer concurrency to protect the master | Proposed (needs measurement) |
| [0010](0010-waiting-room-and-admission-token.md) | Waiting room with signed admission token | Proposed |
| [0011](0011-baseline-vs-final-architecture.md) | Baseline (PostgreSQL only) vs final architecture | Proposed (needs benchmark data) |
