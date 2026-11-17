# FlashTix — Flash-Sale Concert Ticketing System

> Requirements document · v0.1 · 2026-11-15 · Pet project (the project name is a placeholder)

---

## 1. Overview

**FlashTix** is a concert ticketing system where sales open at one exact moment (for example 1:00 AM) and tens of thousands of people compete for a limited number of tickets. The project focuses on **race conditions and protecting the database under a sudden traffic spike**, not on commercial features.

### Goals

| # | Goal | How it is proven |
|---|---|---|
| G1 | **Never sell the same seat twice**, even during a Redis failover or when a message is delivered more than once | Invariant SQL returns 0 violations after every test |
| G2 | **Protect the master DB**: only winners are allowed to write to the master | Writes to master ≤ number of seats + related orders |
| G3 | **Losers learn the outcome almost instantly** (fail fast) | p95 of the 409 response meets the NFR target |
| G4 | **Measurable and explainable**: load test, chaos test, and an ADR comparing the baseline with the final architecture | k6 report + ADR |

### Out of scope

- Real payments (a mock gateway is used), invoicing, accounting.
- Real email/SMS delivery, native mobile apps, multi-region deployment.
- Dynamic pricing, resale/ticket transfer, gate check-in.
- Complex user management (sign-up, password reset, social login): minimal auth is used.

### Actors

| Actor | Description |
|---|---|
| **Buyer** | A customer buying tickets |
| **Organizer/Admin** | Creates events, loads inventory, monitors operations |
| **Payment Gateway (mock)** | A simulated payment system with configurable latency and failure rate |
| **System jobs** | Reconciliation, cleanup, and cache warm-up jobs |

---

## 2. Scale and assumptions

The numbers below are **design assumptions** and must be updated after real measurements.

| Parameter | Design target | Local profile (scaled to 1/10) |
|---|---|---|
| Concurrent users (CCU) | 60,000 | 6,000 |
| Number of seats | 20,000 (8 sections × 2,500) | 2,000 (8 × 250) |
| Share of users who try to hold seats | 80% | 80% |
| Average attempts per user | 3 | 3 |
| Spike window | 10 seconds | 10 seconds |
| **Peak hold RPS** | **≈ 14,400** | **≈ 1,440** |
| Seat distribution | 80% of requests hit 20% of seats ("hot seats") | unchanged |
| Seat hold time (hold TTL) | 5 minutes | 5 minutes |
| Payment window | 10 minutes (> payment gateway timeout) | 10 minutes |
| Purchase limits | Max 4 seats per hold, all in one section; max 4 tickets per user per event | unchanged |

```
Peak RPS = users × % who try × attempts per user / window (s)
         = 60,000 × 0.8 × 3 / 10 ≈ 14,400 req/s
```

**Local infrastructure:** the API runs as 2 containers, each with 4 vCPU / 8 GB RAM. k6 runs on a separate machine (or in a container with a separate `cpuset`) so it does not compete for CPU with the system under test.

---

## 3. State model

### Seat

```
AVAILABLE ──hold──► HELD(owner, TTL 5') ──order created OK──► PENDING_PAYMENT(owner, TTL 10')
    ▲                    │                                            │
    │                    │ expired / cancelled                        ├─ payment OK ─► SOLD
    └────────────────────┴─────────────── expired / failed ◄──────────┘
```

### Order

```
ACCEPTED ─► PENDING_PAYMENT ─► PAID
   │              │             └─ (payment arrives after expiry) ─► REFUNDED
   └─► FAILED     └─► EXPIRED
```

### Source of truth

| Data | Stored in | Role |
|---|---|---|
| `HELD` / `PENDING_PAYMENT` state (with TTL) | **Redis** | Fast filter, decides the winner on the hot path |
| Orders, sold seats (`SOLD`) | **PostgreSQL master** | **The final source of truth**, protected by a unique constraint |
| Displayed seat map | Redis/CDN snapshot | Display only, **never used to make decisions** |

---

## 4. Architecture constraints

```
Client ─► CDN/WAF ─► Waiting room ─► Reservation Service
                                        │
                                  (1) Redis Lua: hold seats ── lose → 409 immediately
                                        │ win
                                  (2) Publish message ─► Queue (Standard)
                                                             │
                                                      Order Service ─► PostgreSQL master
                                                             │
                                                  (3) Redis: extend / mark
Seat map: Redis/CDN snapshot + SSE        Replica: reporting only
```

| ID | Constraint |
|---|---|
| AC-01 | Redis is a **lock for efficiency** (a fast filter). Correctness is guaranteed by the **unique constraint `(event_id, seat_id)` in PostgreSQL** (the fencing layer). |
| AC-02 | Holding seats is done with an **atomic Lua script** (all-or-nothing, idempotent per `userId`), never `GET` followed by `SET`. |
| AC-03 | The queue is **at-least-once with no ordering guarantee**. The Order Service **must be idempotent** and must not depend on message order. |
| AC-04 | Write order: **DB first, Redis second**. |
| AC-05 | **Never read from a replica to make a decision** (replication lag). Replicas are for reporting only. |
| AC-06 | If Redis is unavailable, **fail closed** (return 503). Do not fall back to the DB, because that would push all the load onto the master. |
| AC-07 | Redis keys use a hash tag per section (`seat:{eventId:section}:seatId`) so multi-seat scripts run in a single slot without creating a hot shard per event. |

---

## 5. User stories

**Conventions:** Priority follows MoSCoW (Must / Should / Could). Estimates are relative story points (1 · 2 · 3 · 5 · 8). Acceptance criteria use Given/When/Then for the important stories and short bullets for the rest.

### Epic A — Event discovery

#### US-01 — View an event and its seat map
- **Priority:** Must · **Estimate:** 3
- **Story:** As a Buyer, I want to see event details and a seat map by section with the status of each seat, so that I can pick the seats I want.
- **Acceptance criteria:**
  - A seat shows one of three states: Available, Held, Sold.
  - The seat map is read from a cached snapshot and **never queries the master**.
  - Data is at most 1 second behind the real state.
  - Latency meets NFR-P03.

#### US-02 — Real-time seat status updates
- **Priority:** Should · **Estimate:** 5
- **Story:** As a Buyer, I want to see a seat being held or bought by someone else as it happens, so that I do not waste effort clicking on seats that are gone.
- **Acceptance criteria:**
  - Seat state changes are pushed over SSE/SignalR.
  - After a disconnect, the client reconnects automatically and resynchronizes the full seat map.
  - A polling fallback with jitter is available when SSE cannot be used.
  - Real-time data is for display only; the hold step remains the place where the decision is made.

### Epic B — Waiting room

#### US-03 — Join the waiting room before the sale opens
- **Priority:** Must · **Estimate:** 5
- **Story:** As a Buyer, I want to join a queue when the event opens and know my position, so that I do not get errors or lose my chance because the system is overloaded.
- **Acceptance criteria:**
  - When sales open, the user sees their queue position and an estimated wait time.
  - The system admits users at a configurable rate (N users per second).
  - Each user has only one position in the queue (no taking several slots with multiple tabs).
  - The waiting room can absorb the entire user population without overloading the Reservation Service.

#### US-04 — Be admitted to seat selection with a token
- **Priority:** Must · **Estimate:** 3
- **Story:** As a Buyer, I want to receive an admission token when it is my turn, so that I can hold seats within the allowed time.
- **Acceptance criteria:**
  - The token is signed and time-limited; the hold API rejects requests without a valid token (401/403).
  - When the token expires, the user must rejoin the queue.
  - A user has only one valid token at a time.

### Epic C — Holding seats

#### US-05 — Hold seats
- **Priority:** Must · **Estimate:** 8
- **Story:** As a Buyer, I want to hold 1 to 4 seats for 5 minutes, so that I have time to pay without someone else taking them.
- **Acceptance criteria:**
  - **Given** the seat is available, **When** I send a hold request, **Then** I receive 202 with a `requestId` and the seat belongs to me for 5 minutes.
  - **Given** a request with several seats, **When** at least one seat already has another owner, **Then** **no seat is held** (all-or-nothing) and the response names the conflicting seat.
  - **Given** I send the same request again (a retry), **When** the seat is still mine, **Then** it counts as success and the TTL is **not extended**.
  - Requests with seats in several sections, more than 4 seats, or exceeding the 4-tickets-per-user-per-event limit are rejected (400/422).
  - **When 1,000 requests compete for one seat concurrently, exactly one request wins.**

#### US-06 — Lose the race for a seat
- **Priority:** Must · **Estimate:** 2
- **Story:** As a Buyer who lost, I want to know immediately that the seat is taken, so that I can pick another seat instead of waiting.
- **Acceptance criteria:**
  - Return **409** with the conflicting seat; no order is created and nothing is written to PostgreSQL.
  - Latency meets NFR-P02.
  - The client suggests choosing another seat.

#### US-07 — Release a seat voluntarily
- **Priority:** Must · **Estimate:** 3
- **Story:** As a Buyer, I want to cancel my hold, so that others can buy the seat.
- **Acceptance criteria:**
  - Only seats held by the caller can be released (owner token comparison).
  - Calling it multiple times gives the same result (idempotent).
  - The seat returns to `AVAILABLE` and the seat map is updated.

#### US-08 — Holds expire automatically
- **Priority:** Must · **Estimate:** 3
- **Story:** As the system, I want seats to be released when the hold time runs out, so that tickets are not stuck forever.
- **Acceptance criteria:**
  - **Given** a hold that is older than 5 minutes, **When** someone else holds that seat, **Then** they win.
  - **Given** the previous user pays after expiry and the seat now belongs to someone else, **When** they confirm, **Then** they are rejected at the owner-token check.
  - Correctness **does not depend** on a cleanup job (expiry is enforced by TTL and token).

### Epic D — Orders

#### US-09 — Create an order asynchronously and track its status
- **Priority:** Must · **Estimate:** 5
- **Story:** As a Buyer who won seats, I want the system to confirm my order and let me track its status, so that I know when I can pay.
- **Acceptance criteria:**
  - After winning in Redis, the Reservation Service publishes a message and returns 202 with a `requestId`.
  - If publishing fails, the **hold is released** and 503 is returned, so no seat is left dangling.
  - The client checks status by `requestId` over SSE or a polling endpoint.
  - Returned statuses: `ACCEPTED`, `PENDING_PAYMENT`, `FAILED`, `EXPIRED`, `PAID`.

#### US-10 — Idempotent processing in the Order Service
- **Priority:** Must · **Estimate:** 8
- **Story:** As the system, I want to process order-creation messages so that duplicate delivery or reordering still produces the correct result.
- **Acceptance criteria:**
  - **Given** a message is delivered twice, **When** the consumer processes it the second time, **Then** a unique violation on `request_id` is treated as a duplicate: ack and skip.
  - A unique violation on `(event_id, seat_id)` is treated as a real conflict (see US-11), distinguished by **constraint name** (error code `23505`).
  - Transient errors (timeout, deadlock) are not acked; retries use backoff, and after N attempts the message goes to the **DLQ**.
  - The number of concurrent consumers can be limited to protect the master.
  - A message that waited in the queue longer than the hold TTL must have its owner token re-checked before processing.

#### US-11 — Handle conflicts between Redis and the DB
- **Priority:** Must · **Estimate:** 5
- **Story:** As a Buyer, if the system discovers that my seat actually has another owner, I want to be told before I am charged.
- **Acceptance criteria:**
  - **Given** Redis says I won but the DB rejects due to the unique constraint, **When** the order is processed, **Then** the order becomes `FAILED` and the hold is released if it still belongs to me.
  - The user is told "seat no longer available" **before the payment step**.
  - No money transaction is created for a `FAILED` order.

### Epic E — Payment

#### US-12 — Pay for an order
- **Priority:** Must · **Estimate:** 5
- **Story:** As a Buyer, I want to pay within the allowed window, so that I can lock in my tickets.
- **Acceptance criteria:**
  - Once the order is created, the seat becomes `PENDING_PAYMENT` with a 10-minute TTL (longer than the gateway timeout).
  - The mock gateway has configurable latency and failure rate for testing.
  - Successful payment: order `PAID`, seat `SOLD`; DB is written first, Redis second.
  - Failed or late payment: order `EXPIRED`, seats are released.

#### US-13 — Idempotent payment webhook
- **Priority:** Must · **Estimate:** 5
- **Story:** As the system, I want to handle duplicate or late webhooks without charging or confirming a ticket twice.
- **Acceptance criteria:**
  - The webhook carries a valid signature and an `idempotencyKey`; calling it again gives the same result.
  - A webhook for an order that is already `PAID` just returns OK and creates no extra records.
  - A webhook with a bad signature is rejected and logged.

#### US-14 — Automatic refund when payment arrives after expiry
- **Priority:** Should · **Estimate:** 5
- **Story:** As a Buyer, if my money was taken but the seat is already gone, I want to be refunded automatically.
- **Acceptance criteria:**
  - **Given** a successful payment but the owner token is no longer valid, **When** confirmation runs, **Then** the order becomes `REFUNDED` and a refund command is issued (compensating saga).
  - The refund command is sent through an outbox so it is not lost on a crash.
  - Refunds are idempotent.

### Epic F — Operations

#### US-15 — Create an event and load inventory
- **Priority:** Must · **Estimate:** 5
- **Story:** As an Organizer, I want to create an event with sections and seats and load the inventory into Redis before sales open.
- **Acceptance criteria:**
  - An event can be created with an on-sale time, a list of sections, and seats.
  - A **warm-up** step loads seat states into Redis and builds the seat map snapshot.
  - A verification step checks that the seat count in Redis matches the DB; a mismatch blocks the sale from opening.

#### US-16 — Reconcile Redis and the DB
- **Priority:** Should · **Estimate:** 5
- **Story:** As the system, I want to reconcile periodically so that discrepancies between Redis and the DB are detected and fixed.
- **Acceptance criteria:**
  - A periodic job compares `SOLD`/`PENDING_PAYMENT` seats in the DB with Redis and reports discrepancies.
  - A **rebuild Redis from DB** operation exists for when Redis loses data or restarts.
  - A metric counts the number of discrepancies found.

#### US-17 — Monitoring dashboard
- **Priority:** Should · **Estimate:** 5
- **Story:** As an Organizer, I want to see system health in real time while sales are open.
- **Acceptance criteria:**
  - Shows: RPS, p95/p99, win/conflict counts, queue depth and consumer lag, Redis ops/s, PostgreSQL lock waits and connections, container CPU throttling.
  - Alerts fire when thresholds are exceeded (see NFR-O04).

#### US-18 — Bot and abuse protection
- **Priority:** Could · **Estimate:** 5
- **Story:** As an Organizer, I want to limit abnormal behavior so that real users get a fair chance.
- **Acceptance criteria:**
  - Rate limiting per user and per IP; exceeding it returns 429 with `Retry-After`.
  - An optional challenge (captcha or proof-of-work) before joining the queue.

### Epic G — Testing

#### US-19 — Correctness test
- **Priority:** Must · **Estimate:** 3
- **Story:** As a developer, I want a test that proves seats are never sold twice.
- **Acceptance criteria:**
  - k6: 1 seat, 1,000 requests from 200 VUs, threshold `hold_wins == 1`.
  - After the test, the invariant query returns **empty**:
    `SELECT event_id, seat_id, count(*) FROM order_seats GROUP BY 1,2 HAVING count(*) > 1;`
  - Runs in CI.

#### US-20 — Flash-sale load test
- **Priority:** Must · **Estimate:** 5
- **Story:** As a developer, I want to simulate a sale opening so that I can find the system's breaking point.
- **Acceptance criteria:**
  - Uses the `ramping-arrival-rate` executor (open model), an 80/20 hot-seat distribution, and losers retry other seats up to 3 times.
  - Thresholds: p95/p99 per the NFRs, `dropped_iterations == 0`, check rate > 99%. A 409 response counts as valid.
  - A stepwise test determines the maximum RPS of **one container** and computes the number of instances needed.
  - A report (numbers and charts) is exported and stored in the repo.

#### US-21 — Chaos test
- **Priority:** Should · **Estimate:** 5
- **Story:** As a developer, I want to verify that the system stays correct when a component fails during load.
- **Acceptance criteria:**
  - Scenarios: stop Redis, kill an order consumer, restart PostgreSQL, simulate duplicate message delivery.
  - After each scenario, the "no double sale" invariant still holds.
  - The system recovers within the NFR-R05 limit without manual intervention (except a Redis rebuild if needed).

#### US-22 — Benchmark the baseline against the final architecture
- **Priority:** Should · **Estimate:** 3
- **Story:** As a developer, I want data comparing the simplest approach with the final architecture, so that I know whether each added component is worth it.
- **Acceptance criteria:**
  - Baseline: PostgreSQL only with a conditional `UPDATE`, no Redis, no queue.
  - Compare using the same k6 scenario: maximum RPS, p95/p99, master CPU and lock waits.
  - Results are recorded as an **ADR** stating why each component was kept or dropped.

---

## 6. Non-functional requirements

The thresholds are **design targets** for 60k CCU; for the local profile, measure and record the actual values and compare them. Any threshold that is missed needs an explanation or an improvement plan in an ADR.

### 6.1 Correctness and consistency

| ID | Requirement | Target | Verification |
|---|---|---|---|
| NFR-C01 | No seat is sold twice | **0 violations** in every test, including during failures | Invariant SQL, US-19/21 |
| NFR-C02 | Each seat has at most one valid hold at any time | 1,000 requests competing for 1 seat → exactly 1 wins | k6 correctness test |
| NFR-C03 | Message processing is idempotent | Delivering N times gives the same result as once | Integration test |
| NFR-C04 | Payment webhook and refund are idempotent | No two records for the same `idempotencyKey` | Integration test |
| NFR-C05 | No money transaction for a `FAILED` order | 0 cases | Check after chaos test |

### 6.2 Performance

| ID | Requirement | Target (design) |
|---|---|---|
| NFR-P01 | Hold API (win) | p95 < 300 ms, p99 < 800 ms at peak RPS |
| NFR-P02 | Hold API (lose, 409) | p95 < 100 ms (fail fast, never touches the DB) |
| NFR-P03 | Seat map read | p95 < 200 ms |
| NFR-P04 | From receiving 202 to the final order status | p95 < 5 s at peak |
| NFR-P05 | Queue lag at steady state during peak | < 10 s, and back to 0 after the spike |
| NFR-P06 | Writes to master | ≤ number of seats + related orders (proves master protection) |
| NFR-P07 | 5xx error rate | < 0.1% (409 and 429 do not count as errors) |

### 6.3 Scalability and resources

| ID | Requirement | Target |
|---|---|---|
| NFR-S01 | Reservation Service and Order Service are **stateless** and scale horizontally | Adding instances increases RPS nearly linearly at the API tier |
| NFR-S02 | Seats are distributed across Redis shards by section | No shard carries more than ~2× the average load |
| NFR-S03 | Determine the real capacity | A figure for "max RPS of 1 container (4 vCPU/8 GB)" at passing p95 and ~70% CPU |
| NFR-S04 | Instances needed = `peak RPS / RPS per container × (1.3 to 1.5)` | A calculation table in the report |
| NFR-S05 | PostgreSQL connection pool is bounded (PgBouncer) | Does not exceed `max_connections` when consumers scale |

### 6.4 Availability and resilience

| ID | Requirement | Target |
|---|---|---|
| NFR-R01 | Redis unavailable → **fail closed** (503), no fallback to the DB | Verified by chaos test |
| NFR-R02 | Redis failover does not cause a double sale | The DB unique constraint blocks it (NFR-C01) |
| NFR-R03 | A failing message does not lose data | Retry with backoff + jitter, DLQ after N attempts, a DLQ replay tool |
| NFR-R04 | Backpressure | Overload returns 429 with `Retry-After` (with jitter); requests are never left hanging |
| NFR-R05 | Recovery after a single component failure | Self-recovers in < 60 s (excluding a manual Redis rebuild) |
| NFR-R06 | Writing an order and publishing a message is not lost to a crash in between | Use an outbox, or compensate (release the hold) when publishing fails |
| NFR-R07 | Every outbound call (gateway, queue) has a timeout, retry, and circuit breaker | Explicit configuration, with tests |

### 6.5 Data durability

| ID | Requirement | Target |
|---|---|---|
| NFR-D01 | PostgreSQL has WAL enabled and periodic backups | Recoverable from backup, with a restore tested at least once |
| NFR-D02 | Redis has AOF `everysec` plus a replica | Accept losing up to ~1 second of `HELD` state (the DB is the safety net) |
| NFR-D03 | Redis can be recovered from the DB | A rebuild script exists and is tested (US-16) |

### 6.6 Security

| ID | Requirement | Target |
|---|---|---|
| NFR-SEC01 | Authenticate every API with JWT; waiting-room tokens are signed and time-limited | Missing/invalid token → 401/403 |
| NFR-SEC02 | Rate limit per user and per IP | Exceeding the limit → 429 |
| NFR-SEC03 | Strict input validation (seat count, section, data types) | Tests with malformed data |
| NFR-SEC04 | Webhooks are signed and replay-protected | Bad signature → rejected |
| NFR-SEC05 | No sensitive data (tokens, payment details) in logs; secrets are kept out of source code | Review logs and repo |
| NFR-SEC06 | The test-auth mechanism (`X-Test-User` header) is enabled only in test environments | Off by default in production configuration |

### 6.7 Observability

| ID | Requirement | Target |
|---|---|---|
| NFR-O01 | Metrics: RPS, p95/p99, win/conflict counts, queue depth, consumer lag, Redis ops/s, PostgreSQL lock waits and connections, CPU throttling | Shown on a dashboard (Prometheus + Grafana) |
| NFR-O02 | Structured logs carrying `requestId` and `userId` across reservation → queue → order processing | A request can be traced through every service |
| NFR-O03 | Distributed tracing (OpenTelemetry) | Redis, queue, and DB spans visible in one trace |
| NFR-O04 | Alerts | At minimum: p95 above threshold, queue lag growing continuously, messages in the DLQ, reconciliation discrepancy detected |

### 6.8 Fairness

| ID | Requirement | Target |
|---|---|---|
| NFR-F01 | The waiting room admits in arrival order at a configurable rate | Admission order is close to queue order (small deviations allowed) |
| NFR-F02 | One user cannot take multiple slots with several tabs or repeated calls | Verified by test |

### 6.9 Maintainability and testability

| ID | Requirement | Target |
|---|---|---|
| NFR-M01 | The whole system starts with one command (`docker compose up`) | A README with instructions |
| NFR-M02 | Unit tests for business logic and integration tests against real services (Testcontainers for PostgreSQL, Redis, RabbitMQ) | Tests run in CI |
| NFR-M03 | Concurrency tests run on a **real Redis Cluster**, not a mock | Catches hash-tag and `CROSSSLOT` bugs |
| NFR-M04 | Architecture decisions are recorded as ADRs | Every major decision has an ADR |
| NFR-M05 | Lua scripts live in their own files with their own tests | No scripts embedded as strings in code |

---

## 7. Proposed technology

| Layer | Choice | Notes |
|---|---|---|
| Backend | .NET (latest LTS), ASP.NET Core minimal API | Reservation Service, Order Service |
| Frontend | React | Seat map, waiting room, status tracking |
| Cache and seat holds | Redis (Cluster, AOF on) + StackExchange.Redis | Atomic Lua scripts |
| Database | PostgreSQL (1 primary + a replica for reporting) + PgBouncer | The unique constraint is the last line of defense |
| Queue | RabbitMQ (local); SQS when moving to the cloud | Standard queue, at-least-once |
| Realtime | SSE or SignalR | Seat map and order status updates |
| Load testing | k6 | `ramping-arrival-rate`, run from a separate machine |
| Observability | OpenTelemetry, Prometheus, Grafana | |

---

## 8. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Redis failover leaves two people believing they hold the same seat | Double sale | DB unique constraint; owner-token check at confirmation (AC-01) |
| Hot key: requests pile onto the best seats | One shard overloaded | Hash tag per section; reject losers early; waiting room |
| Dual write between the DB and Redis/queue | Inconsistent data | DB-before-Redis ordering; outbox; reconciliation job |
| Messages delivered twice or out of order | Duplicate orders | Idempotency by `request_id`; no dependence on order |
| The load generator (k6) saturates before the system does | Misleading test results | Run k6 separately; watch `dropped_iterations` |
| Running every component on one host | CPU contention, distorted numbers | Pin cores (`cpuset`), separate k6, record the setup in reports |
| Extrapolating linearly for Redis and the DB | Wrong estimates | Extrapolate only the API tier; measure Redis and DB separately |
| Pet project scope creep | Never finished | Follow the roadmap, finish all Must stories before Should/Could |

---

## 9. Suggested roadmap

| Milestone | Content | Stories |
|---|---|---|
| **M1 — Baseline** | Seat map, seat holds with PostgreSQL only (conditional `UPDATE` + unique constraint), correctness test | US-01, US-05 (DB version), US-19, US-22 (baseline) |
| **M2 — Redis hold** | Lua scripts for hold/release, expiry, fast 409 | US-05, US-06, US-07, US-08 |
| **M3 — Queue and order processing** | Reservation Service → queue → Order Service, idempotency, DLQ, conflict handling | US-09, US-10, US-11 |
| **M4 — Payment** | Mock gateway, idempotent webhook, refunds | US-12, US-13, US-14 |
| **M5 — Experience** | Waiting room, SSE, purchase limits, inventory loading | US-02, US-03, US-04, US-15 |
| **M6 — Measure and prove** | Observability, load test, chaos test, reconciliation, comparison ADR | US-16, US-17, US-20, US-21, US-22 |
| **M7 — Optional** | Bot protection | US-18 |

---

## 10. Definition of Done and project acceptance

### Definition of Done for each story
- Code is reviewed (self-review against a checklist if working alone) and passes CI.
- Unit/integration tests cover the main acceptance criteria.
- Required metrics and logs are added.
- Documentation or an ADR is updated if an architecture decision was made.

### Project acceptance criteria
1. The "no double sale" invariant holds after **all** correctness, load, and chaos tests (NFR-C01).
2. A load test report shows the maximum RPS of one container and the number of instances needed (NFR-S03, NFR-S04).
3. Writes to master stay within the NFR-P06 limit.
4. An ADR compares the baseline with the final architecture (US-22).
5. The whole system starts with one command and the test scenarios can be rerun from the README (NFR-M01).

---

## 11. Glossary

| Term | Meaning |
|---|---|
| **CCU** | Concurrent users: the number of users online at the same time |
| **VU** | Virtual user: a k6 execution thread (not the same as CCU) |
| **RPS** | Requests per second |
| **p95 / p99** | The latency that 95% / 99% of requests are at or below |
| **TTL** | Time to live: how long a key lives before it is deleted automatically |
| **Hold / Lease** | A time-limited claim on a seat |
| **Idempotent** | Performing it many times has the same effect as performing it once |
| **Fencing** | A final layer that rejects wrong operations regardless of what the client believes (here, the unique constraint) |
| **DLQ** | Dead-letter queue: where messages go after exhausting their retries |
| **Outbox** | Writing the message to be sent into the DB in the same transaction as the data, then sending it later, so no message is lost on a crash |
| **Hot key** | A key that receives a burst of traffic (the best seats) |
| **Open / closed model** | Open: load fixed by requests per second (k6 arrival-rate); closed: a fixed number of VUs, so load drops automatically when the server slows down |
