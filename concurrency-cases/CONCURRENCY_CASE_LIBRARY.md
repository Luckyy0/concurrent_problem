# Java/Spring Boot Concurrency Case Library

## Catalog status

| Field | Value |
| --- | --- |
| Mode completed | Phase 1 — Design |
| Root | `concurrency-cases/` |
| Depth | Advanced |
| Phase 2 prose language | Vietnamese; canonical technical terms remain English |
| Default stack | Java 21, Spring Boot, Spring Data JPA, Hibernate, PostgreSQL |
| Optional infrastructure | Redis and Kafka only in cases that require them |
| Planned cases | 74 |
| Governance files created in Phase 1 | `CONCURRENCY_CASE_LIBRARY.md`, `RULES.md` |
| Case content created in Phase 1 | None |
| Default Phase 2 batch size | One case |

This file is the single source of truth for IDs, scope, paths, priority, learning
stage, and prerequisites. `RULES.md` is the implementation contract.

## Taxonomy and case counts

| Order | Taxonomy | ID prefix | Planned cases | Purpose |
| ---: | --- | --- | ---: | --- |
| 1 | `jvm/` | `JVM` | 10 | In-process races, memory visibility, collections, locks, executors |
| 2 | `spring/` | `SPR` | 7 | Proxy transactions, async boundaries, propagation, rollback, retry |
| 3 | `postgresql/` | `DB` | 10 | MVCC anomalies, constraints, row locks, deadlocks, serializable work |
| 4 | `locking/` | `LOCK` | 5 | Optimistic, pessimistic, atomic SQL, strategy selection |
| 5 | `banking/` | `BANK` | 10 | Money, payment, transfer, ledger, settlement, reconciliation |
| 6 | `ecommerce/` | `ECOM` | 9 | Inventory, checkout, coupons, points, carts, refunds, order workflow |
| 7 | `booking/` | `BOOK` | 4 | Capacity, seats, holds, expiry, confirmation |
| 8 | `messaging/` | `MSG` | 7 | Kafka delivery, ordering, retries, inbox, outbox |
| 9 | `redis/` | `REDIS` | 6 | Atomic commands, locks, fencing, cache races, stampede |
| 10 | `distributed/` | `DIST` | 6 | Multi-node coordination, ambiguity, sagas, dual writes, stale reads |
|  | **Total** |  | **74** |  |

Taxonomies reflect the layer where the primary correctness failure is explained.
A domain case links to foundational cases rather than repeating their complete
theory.

## Priority model

- `P0` — Critical production correctness: money, inventory, payment, confirmed
  booking, durable side effects.
- `P1` — Common production concurrency problem.
- `P2` — Advanced or high-scale scenario.
- `P3` — Specialized scenario.

Priority is production risk, not learning order.

## Learning progression

| Stage | Focus | Primary cases |
| ---: | --- | --- |
| 1 | JVM race fundamentals | `JVM-*` |
| 2 | Spring transactions and PostgreSQL MVCC | `SPR-*`, `DB-001`–`DB-006` |
| 3 | Locking and conflict control | `DB-007`–`DB-010`, `LOCK-*` |
| 4 | Banking, e-commerce, and booking invariants | `BANK-*`, `ECOM-*`, `BOOK-*` |
| 5 | Messaging and idempotency | `MSG-*` |
| 6 | Redis and distributed concurrency | `REDIS-*`, `DIST-*` |
| 7 | High-contention and failure-mode synthesis | P2/P3 cases from all taxonomies |

Prerequisites in each entry refine this order. `None` means the case may be read
without another case, though shared concepts may still be linked.

## Planned shared references

These files are not created in Phase 1. In Phase 2, create a reference only when
the first dependent case needs it, and keep case-specific reasoning inside the
case.

| Ref | Planned path | Scope |
| --- | --- | --- |
| `C-JMM` | `concepts/java-memory-model-and-atomicity.md` | Visibility, atomicity, happens-before, compound actions |
| `C-SPRING-TX` | `concepts/spring-transaction-boundaries.md` | Proxies, propagation, rollback, async boundaries |
| `C-MVCC` | `concepts/postgresql-mvcc.md` | Snapshots, tuple versions, statement/transaction visibility |
| `C-ISO` | `concepts/isolation-levels.md` | PostgreSQL isolation and anomalies |
| `C-DB-LOCKS` | `concepts/postgresql-locks.md` | Row/table locks, wait queues, timeouts, lock lifetime |
| `C-OPT` | `concepts/optimistic-locking.md` | `@Version`, affected rows, conflicts, bounded retry |
| `C-PESS` | `concepts/pessimistic-locking.md` | `FOR UPDATE`, blocking, ordering, transaction length |
| `C-ATOMIC-SQL` | `concepts/atomic-database-operations.md` | Conditional mutation, predicate recheck, affected rows |
| `C-DEADLOCK` | `concepts/deadlocks-and-retries.md` | Wait-for cycles, detection, deterministic order, retry |
| `C-IDEMP` | `concepts/idempotency-and-uniqueness.md` | Atomic claim, replay, request fingerprint, response storage |
| `C-LEDGER` | `concepts/ledger-balances-and-holds.md` | Ledger, posted/available balance, authorization, settlement |
| `C-KAFKA` | `concepts/kafka-delivery-and-ordering.md` | Partitions, groups, redelivery, transaction boundaries |
| `C-INBOX-OUTBOX` | `concepts/inbox-outbox-patterns.md` | Durable message deduplication and atomic publication |
| `C-LEASE` | `concepts/leases-ownership-and-fencing.md` | TTL, owner token, pauses, stale owner rejection |
| `C-CACHE` | `concepts/cache-consistency.md` | Cache-aside, invalidation, authority, stampede |
| `C-TEST` | `concepts/concurrency-testing.md` | Latches, barriers, executors, Testcontainers, invariant assertions |

## Standard planned file bundle

Every case entry explicitly names these five files under its folder:

1. `README.md`
2. `broken-code.md`
3. `analysis.md`
4. `solutions.md`
5. `experiments.md`

Their required content is defined in `RULES.md`. Paths below are relative to
`concurrency-cases/`.

---

# Case catalog

## Java / JVM

### JVM-001 — Mutable State in a Spring Singleton

- **Domain:** Java/JVM, Spring
- **Problem:** Concurrent requests increment and read a mutable service field,
  losing updates and leaking request state between users.
- **Primary concepts:** shared mutable state, singleton lifecycle,
  read-modify-write, Java Memory Model
- **Files:** `jvm/spring-singleton-mutable-state/README.md`,
  `jvm/spring-singleton-mutable-state/broken-code.md`,
  `jvm/spring-singleton-mutable-state/analysis.md`,
  `jvm/spring-singleton-mutable-state/solutions.md`,
  `jvm/spring-singleton-mutable-state/experiments.md`
- **Priority / stage:** P1 / Stage 1
- **Prerequisites:** None; refs `C-JMM`, `C-TEST`
- **Scope boundary:** In-JVM state only; database entity races belong to
  `DB-001`.

### JVM-002 — Check-Then-Act Resource Registration

- **Domain:** Java/JVM
- **Problem:** Two threads observe an absent key and both create a resource,
  violating one-resource-per-key despite thread-safe individual operations.
- **Primary concepts:** check-then-act, compound action, atomic map APIs,
  safe publication
- **Files:** `jvm/check-then-act-registration/README.md`,
  `jvm/check-then-act-registration/broken-code.md`,
  `jvm/check-then-act-registration/analysis.md`,
  `jvm/check-then-act-registration/solutions.md`,
  `jvm/check-then-act-registration/experiments.md`
- **Priority / stage:** P1 / Stage 1
- **Prerequisites:** `JVM-001`; refs `C-JMM`, `C-TEST`
- **Scope boundary:** A local registry; durable cross-node uniqueness belongs
  to `DB-006` and `DIST-001`.

### JVM-003 — Concurrent HashMap Mutation and Unsafe Publication

- **Domain:** Java/JVM
- **Problem:** Request threads mutate a shared `HashMap` while other threads
  iterate or read it, producing lost entries, visibility bugs, or structural
  corruption.
- **Primary concepts:** non-thread-safe collections, unsafe publication,
  `ConcurrentHashMap`, iteration semantics
- **Files:** `jvm/hashmap-concurrent-mutation/README.md`,
  `jvm/hashmap-concurrent-mutation/broken-code.md`,
  `jvm/hashmap-concurrent-mutation/analysis.md`,
  `jvm/hashmap-concurrent-mutation/solutions.md`,
  `jvm/hashmap-concurrent-mutation/experiments.md`
- **Priority / stage:** P1 / Stage 1
- **Prerequisites:** `JVM-001`; refs `C-JMM`, `C-TEST`
- **Scope boundary:** Map structure and publication, not atomicity of multi-key
  business transactions.

### JVM-004 — Shared ArrayList Mutation During Parallel Work

- **Domain:** Java/JVM
- **Problem:** Parallel tasks append to and traverse one `ArrayList`, causing
  missing elements, stale reads, or fail-fast iteration.
- **Primary concepts:** non-thread-safe list, structural modification,
  confinement, concurrent collectors
- **Files:** `jvm/arraylist-parallel-mutation/README.md`,
  `jvm/arraylist-parallel-mutation/broken-code.md`,
  `jvm/arraylist-parallel-mutation/analysis.md`,
  `jvm/arraylist-parallel-mutation/solutions.md`,
  `jvm/arraylist-parallel-mutation/experiments.md`
- **Priority / stage:** P1 / Stage 1
- **Prerequisites:** `JVM-001`; refs `C-JMM`, `C-TEST`
- **Scope boundary:** In-memory result aggregation; future composition is
  treated in `JVM-009`.

### JVM-005 — Volatile and AtomicInteger Compound-Operation Misuse

- **Domain:** Java/JVM
- **Problem:** A volatile or atomic counter participates in a multi-variable
  limit check, so visible individual reads/writes still violate the capacity
  invariant.
- **Primary concepts:** visibility versus atomicity, compare-and-set,
  compound invariant, AtomicInteger misuse
- **Files:** `jvm/volatile-atomic-compound-invariant/README.md`,
  `jvm/volatile-atomic-compound-invariant/broken-code.md`,
  `jvm/volatile-atomic-compound-invariant/analysis.md`,
  `jvm/volatile-atomic-compound-invariant/solutions.md`,
  `jvm/volatile-atomic-compound-invariant/experiments.md`
- **Priority / stage:** P1 / Stage 1
- **Prerequisites:** `JVM-001`; refs `C-JMM`, `C-TEST`
- **Scope boundary:** Single-JVM compound state, not durable inventory.

### JVM-006 — Mis-scoped synchronized and ReentrantLock

- **Domain:** Java/JVM, Distributed
- **Problem:** A new lock per call, the wrong monitor, or a node-local lock
  appears to serialize work but leaves the real shared state unprotected.
- **Primary concepts:** monitor identity, lock scope, critical section,
  multi-instance limitation
- **Files:** `jvm/mis-scoped-local-lock/README.md`,
  `jvm/mis-scoped-local-lock/broken-code.md`,
  `jvm/mis-scoped-local-lock/analysis.md`,
  `jvm/mis-scoped-local-lock/solutions.md`,
  `jvm/mis-scoped-local-lock/experiments.md`
- **Priority / stage:** P1 / Stage 1
- **Prerequisites:** `JVM-001`; refs `C-JMM`, `C-TEST`
- **Scope boundary:** Explains why local locking fails; authoritative multi-node
  alternatives are developed in `DIST-001`.

### JVM-007 — Opposite Lock Ordering Deadlock

- **Domain:** Java/JVM
- **Problem:** Two threads acquire account locks in opposite order and wait
  forever.
- **Primary concepts:** intrinsic/ReentrantLock deadlock, wait-for cycle,
  deterministic ordering, timed acquisition
- **Files:** `jvm/opposite-lock-order-deadlock/README.md`,
  `jvm/opposite-lock-order-deadlock/broken-code.md`,
  `jvm/opposite-lock-order-deadlock/analysis.md`,
  `jvm/opposite-lock-order-deadlock/solutions.md`,
  `jvm/opposite-lock-order-deadlock/experiments.md`
- **Priority / stage:** P1 / Stage 1
- **Prerequisites:** `JVM-006`; refs `C-DEADLOCK`, `C-TEST`
- **Scope boundary:** JVM deadlock; PostgreSQL deadlock is `DB-008`.

### JVM-008 — Executor Saturation, Starvation, and Nested Blocking

- **Domain:** Java/JVM
- **Problem:** Tasks submitted to a bounded executor block waiting for child
  tasks on the same exhausted executor, starving progress and exhausting
  request latency budgets.
- **Primary concepts:** executor sizing, starvation, queue policy, nested
  blocking, backpressure
- **Files:** `jvm/executor-starvation/README.md`,
  `jvm/executor-starvation/broken-code.md`,
  `jvm/executor-starvation/analysis.md`,
  `jvm/executor-starvation/solutions.md`,
  `jvm/executor-starvation/experiments.md`
- **Priority / stage:** P1 / Stage 1
- **Prerequisites:** None; refs `C-TEST`
- **Scope boundary:** Thread-pool progress; JDBC pool contention is `SPR-007`.

### JVM-009 — CompletableFuture Shared Aggregation Race

- **Domain:** Java/JVM
- **Problem:** Multiple completion stages mutate a shared accumulator while
  errors and cancellation leave a partially visible result.
- **Primary concepts:** CompletableFuture, shared state, confinement,
  composition, failure propagation
- **Files:** `jvm/completable-future-shared-state/README.md`,
  `jvm/completable-future-shared-state/broken-code.md`,
  `jvm/completable-future-shared-state/analysis.md`,
  `jvm/completable-future-shared-state/solutions.md`,
  `jvm/completable-future-shared-state/experiments.md`
- **Priority / stage:** P1 / Stage 1
- **Prerequisites:** `JVM-004`, `JVM-008`; refs `C-JMM`, `C-TEST`
- **Scope boundary:** In-memory aggregation; transaction context loss is
  `SPR-002`.

### JVM-010 — Retry Livelock Under Symmetric Contention

- **Domain:** Java/JVM
- **Problem:** Two polite workers repeatedly detect conflict, back off in the
  same pattern, and retry without either making progress.
- **Primary concepts:** livelock, bounded retry, randomized backoff, progress
  guarantees
- **Files:** `jvm/symmetric-retry-livelock/README.md`,
  `jvm/symmetric-retry-livelock/broken-code.md`,
  `jvm/symmetric-retry-livelock/analysis.md`,
  `jvm/symmetric-retry-livelock/solutions.md`,
  `jvm/symmetric-retry-livelock/experiments.md`
- **Priority / stage:** P2 / Stage 7
- **Prerequisites:** `JVM-005`; refs `C-TEST`
- **Scope boundary:** In-process progress failure; database retry storms are
  covered by `LOCK-002` and `DB-009`.

## Spring transactions

### SPR-001 — Transactional Self-Invocation Bypasses the Proxy

- **Domain:** Spring
- **Problem:** A service calls its own `@Transactional` method, so no intended
  transaction starts and partial concurrent updates become externally visible.
- **Primary concepts:** proxy interception, self-invocation, transaction
  boundary, flush
- **Files:** `spring/transactional-self-invocation/README.md`,
  `spring/transactional-self-invocation/broken-code.md`,
  `spring/transactional-self-invocation/analysis.md`,
  `spring/transactional-self-invocation/solutions.md`,
  `spring/transactional-self-invocation/experiments.md`
- **Priority / stage:** P1 / Stage 2
- **Prerequisites:** None; refs `C-SPRING-TX`, `C-TEST`
- **Scope boundary:** Proxy boundary only; isolation anomalies are cataloged
  under `DB-*`.

### SPR-002 — Async Work Escapes the Calling Transaction

- **Domain:** Spring
- **Problem:** `@Async` or a future runs on another thread without the caller's
  transaction and observes uncommitted/missing state or commits side effects
  after caller rollback.
- **Primary concepts:** thread-bound transaction, `@Async`, after-commit work,
  context propagation
- **Files:** `spring/async-transaction-boundary/README.md`,
  `spring/async-transaction-boundary/broken-code.md`,
  `spring/async-transaction-boundary/analysis.md`,
  `spring/async-transaction-boundary/solutions.md`,
  `spring/async-transaction-boundary/experiments.md`
- **Priority / stage:** P0 / Stage 2
- **Prerequisites:** `SPR-001`, `JVM-009`; refs `C-SPRING-TX`, `C-TEST`
- **Scope boundary:** Local async invocation; durable event publication is
  `MSG-007`.

### SPR-003 — Propagation Creates a Partial-Commit Workflow

- **Domain:** Spring
- **Problem:** `REQUIRES_NEW` audit/payment work commits independently while
  the outer business transaction rolls back, or `REQUIRED` unexpectedly marks
  the whole transaction rollback-only.
- **Primary concepts:** propagation, physical versus logical transaction,
  rollback-only, partial commit
- **Files:** `spring/transaction-propagation-partial-commit/README.md`,
  `spring/transaction-propagation-partial-commit/broken-code.md`,
  `spring/transaction-propagation-partial-commit/analysis.md`,
  `spring/transaction-propagation-partial-commit/solutions.md`,
  `spring/transaction-propagation-partial-commit/experiments.md`
- **Priority / stage:** P0 / Stage 2
- **Prerequisites:** `SPR-001`; refs `C-SPRING-TX`, `C-TEST`
- **Scope boundary:** One database/application transaction model; distributed
  compensation is `DIST-004`.

### SPR-004 — Isolation Annotation Does Not Match the Real Boundary

- **Domain:** Spring, PostgreSQL
- **Problem:** An isolation annotation is placed on an unproxied or nested
  method, or an existing transaction keeps a weaker isolation level than the
  developer assumes.
- **Primary concepts:** transaction creation, isolation inheritance,
  datasource defaults, validation
- **Files:** `spring/isolation-boundary-mismatch/README.md`,
  `spring/isolation-boundary-mismatch/broken-code.md`,
  `spring/isolation-boundary-mismatch/analysis.md`,
  `spring/isolation-boundary-mismatch/solutions.md`,
  `spring/isolation-boundary-mismatch/experiments.md`
- **Priority / stage:** P1 / Stage 2
- **Prerequisites:** `SPR-001`, `DB-001`; refs `C-SPRING-TX`, `C-ISO`, `C-TEST`
- **Scope boundary:** Spring's effective configuration; individual anomalies
  remain in `DB-001`–`DB-005`.

### SPR-005 — Checked Exception Commits Unexpectedly

- **Domain:** Spring
- **Problem:** A business failure throws a checked exception, but default
  rollback rules commit state that callers believe was reverted.
- **Primary concepts:** rollback rules, checked exception, transaction status,
  failure contract
- **Files:** `spring/checked-exception-rollback/README.md`,
  `spring/checked-exception-rollback/broken-code.md`,
  `spring/checked-exception-rollback/analysis.md`,
  `spring/checked-exception-rollback/solutions.md`,
  `spring/checked-exception-rollback/experiments.md`
- **Priority / stage:** P0 / Stage 2
- **Prerequisites:** `SPR-001`; refs `C-SPRING-TX`, `C-TEST`
- **Scope boundary:** Rollback classification, not remote timeout ambiguity.

### SPR-006 — Retry Runs Inside a Doomed Transaction

- **Domain:** Spring
- **Problem:** A retry interceptor or loop reuses a rollback-only persistence
  context, so optimistic/serialization retries cannot reload a clean snapshot.
- **Primary concepts:** retry ordering, new transaction per attempt, persistence
  context, bounded backoff
- **Files:** `spring/retry-transaction-boundary/README.md`,
  `spring/retry-transaction-boundary/broken-code.md`,
  `spring/retry-transaction-boundary/analysis.md`,
  `spring/retry-transaction-boundary/solutions.md`,
  `spring/retry-transaction-boundary/experiments.md`
- **Priority / stage:** P0 / Stage 3
- **Prerequisites:** `SPR-001`, `LOCK-001`, `DB-009`; refs `C-SPRING-TX`,
  `C-OPT`, `C-DEADLOCK`, `C-TEST`
- **Scope boundary:** Retry mechanics; domain-specific retry safety remains in
  each business case.

### SPR-007 — Connection Pool Exhaustion from Long Transactions

- **Domain:** Spring, PostgreSQL
- **Problem:** Transactions hold connections and locks while waiting on remote
  I/O or saturated executors, causing pool starvation and cascading timeouts.
- **Primary concepts:** connection pool, transaction duration, lock duration,
  backpressure, timeout budgeting
- **Files:** `spring/connection-pool-long-transaction/README.md`,
  `spring/connection-pool-long-transaction/broken-code.md`,
  `spring/connection-pool-long-transaction/analysis.md`,
  `spring/connection-pool-long-transaction/solutions.md`,
  `spring/connection-pool-long-transaction/experiments.md`
- **Priority / stage:** P1 / Stage 7
- **Prerequisites:** `JVM-008`, `SPR-001`, `DB-007`; refs `C-SPRING-TX`,
  `C-DB-LOCKS`, `C-TEST`
- **Scope boundary:** Resource starvation, not database deadlock detection.

## PostgreSQL and database concurrency
### DB-001 ? Lost Update Under MVCC

- **Domain:** PostgreSQL, JPA/Hibernate
- **Problem:** Two transactions load the same entity, calculate new state, and overwrite each other at `READ COMMITTED`.
- **Primary concepts:** lost update, MVCC, read-modify-write, Hibernate dirty checking
- **Files:** `postgresql/lost-update-mvcc/README.md`, `postgresql/lost-update-mvcc/broken-code.md`, `postgresql/lost-update-mvcc/analysis.md`, `postgresql/lost-update-mvcc/solutions.md`, `postgresql/lost-update-mvcc/experiments.md`
- **Priority / stage:** P0 / Stage 2
- **Prerequisites:** `JVM-001`; refs `C-MVCC`, `C-ISO`, `C-TEST`
- **Scope boundary:** Generic anomaly; money and inventory consequences are in `BANK-002` and `ECOM-001`.

### DB-002 ? Dirty Read Expectations versus PostgreSQL Reality

- **Domain:** PostgreSQL
- **Problem:** A design assumes `READ_UNCOMMITTED` exposes uncommitted rows, but PostgreSQL provides `READ COMMITTED` behavior.
- **Primary concepts:** dirty read, PostgreSQL isolation mapping, aborted versions, portability
- **Files:** `postgresql/dirty-read-expectation/README.md`, `postgresql/dirty-read-expectation/broken-code.md`, `postgresql/dirty-read-expectation/analysis.md`, `postgresql/dirty-read-expectation/solutions.md`, `postgresql/dirty-read-expectation/experiments.md`
- **Priority / stage:** P2 / Stage 2
- **Prerequisites:** `DB-001`; refs `C-MVCC`, `C-ISO`, `C-TEST`
- **Scope boundary:** Shows PostgreSQL's behavior and why designs must not rely on dirty reads.

### DB-003 ? Non-repeatable Read in a Business Decision

- **Domain:** PostgreSQL
- **Problem:** A transaction reads a row twice at `READ COMMITTED`; a concurrent commit changes the second result and invalidates an earlier decision.
- **Primary concepts:** non-repeatable read, statement snapshot, validation, repeatable read
- **Files:** `postgresql/non-repeatable-read/README.md`, `postgresql/non-repeatable-read/broken-code.md`, `postgresql/non-repeatable-read/analysis.md`, `postgresql/non-repeatable-read/solutions.md`, `postgresql/non-repeatable-read/experiments.md`
- **Priority / stage:** P1 / Stage 2
- **Prerequisites:** `DB-001`; refs `C-MVCC`, `C-ISO`, `C-TEST`
- **Scope boundary:** Re-reading one logical row; changing result sets are `DB-004`.

### DB-004 ? Phantom Rows Break a Capacity Check

- **Domain:** PostgreSQL
- **Problem:** Two transactions query a qualifying set, both see spare capacity, and insert rows that make the final count exceed the limit.
- **Primary concepts:** phantom read, predicate invariant, MVCC, serializable conflict
- **Files:** `postgresql/phantom-capacity-check/README.md`, `postgresql/phantom-capacity-check/broken-code.md`, `postgresql/phantom-capacity-check/analysis.md`, `postgresql/phantom-capacity-check/solutions.md`, `postgresql/phantom-capacity-check/experiments.md`
- **Priority / stage:** P0 / Stage 2
- **Prerequisites:** `DB-003`; refs `C-MVCC`, `C-ISO`, `C-TEST`
- **Scope boundary:** Generic predicate capacity; hotel capacity applies it in `BOOK-001`.

### DB-005 ? Write Skew Across Multiple Rows

- **Domain:** PostgreSQL
- **Problem:** Two transactions update different rows after reading the same multi-row invariant, leaving no on-call operator although neither write directly conflicts.
- **Primary concepts:** write skew, snapshot isolation, predicate dependency, serializable
- **Files:** `postgresql/write-skew/README.md`, `postgresql/write-skew/broken-code.md`, `postgresql/write-skew/analysis.md`, `postgresql/write-skew/solutions.md`, `postgresql/write-skew/experiments.md`
- **Priority / stage:** P0 / Stage 2
- **Prerequisites:** `DB-004`; refs `C-MVCC`, `C-ISO`, `C-TEST`
- **Scope boundary:** Multi-row invariant without a same-row write conflict.

### DB-006 ? Unique Constraint as Atomic Concurrency Control

- **Domain:** PostgreSQL, JPA/Hibernate
- **Problem:** `existsByBusinessKey()` followed by `save()` lets concurrent requests insert duplicate logical records.
- **Primary concepts:** check-then-insert, unique constraint, upsert, exception mapping, atomic claim
- **Files:** `postgresql/unique-constraint-concurrency/README.md`, `postgresql/unique-constraint-concurrency/broken-code.md`, `postgresql/unique-constraint-concurrency/analysis.md`, `postgresql/unique-constraint-concurrency/solutions.md`, `postgresql/unique-constraint-concurrency/experiments.md`
- **Priority / stage:** P0 / Stage 2
- **Prerequisites:** `JVM-002`, `DB-001`; refs `C-IDEMP`, `C-TEST`
- **Scope boundary:** Durable uniqueness; full response replay is addressed in `BANK-005`.

### DB-007 ? Row Locks, Table Locks, and Lock Lifetime

- **Domain:** PostgreSQL
- **Problem:** A team assumes an ordinary `SELECT` locks a row or that a row lock blocks all reads, leading to unsafe or needlessly serialized designs.
- **Primary concepts:** row/table lock modes, MVCC readers, acquisition, commit/rollback release
- **Files:** `postgresql/row-table-lock-lifecycle/README.md`, `postgresql/row-table-lock-lifecycle/broken-code.md`, `postgresql/row-table-lock-lifecycle/analysis.md`, `postgresql/row-table-lock-lifecycle/solutions.md`, `postgresql/row-table-lock-lifecycle/experiments.md`
- **Priority / stage:** P1 / Stage 3
- **Prerequisites:** `DB-001`; refs `C-MVCC`, `C-DB-LOCKS`, `C-TEST`
- **Scope boundary:** Lock mechanics; selection patterns are `LOCK-003` and `DB-010`.

### DB-008 ? PostgreSQL Deadlock in Opposite Row Order

- **Domain:** PostgreSQL, Spring
- **Problem:** Concurrent transfers lock Account A and B in opposite order, so PostgreSQL detects a wait-for cycle and aborts one transaction.
- **Primary concepts:** database deadlock, lock ordering, victim abort, transaction retry
- **Files:** `postgresql/opposite-row-order-deadlock/README.md`, `postgresql/opposite-row-order-deadlock/broken-code.md`, `postgresql/opposite-row-order-deadlock/analysis.md`, `postgresql/opposite-row-order-deadlock/solutions.md`, `postgresql/opposite-row-order-deadlock/experiments.md`
- **Priority / stage:** P0 / Stage 3
- **Prerequisites:** `DB-007`, `JVM-007`; refs `C-DB-LOCKS`, `C-DEADLOCK`, `C-TEST`
- **Scope boundary:** Database lock cycle; transfer correctness is `BANK-003`.

### DB-009 ? Serializable Transaction Abort and Safe Retry

- **Domain:** PostgreSQL, Spring
- **Problem:** A `SERIALIZABLE` transaction is treated as guaranteed success, so serialization failures leak or are retried inside the old transaction.
- **Primary concepts:** SSI, serialization failure, whole-transaction retry, backoff, idempotent attempt
- **Files:** `postgresql/serializable-retry/README.md`, `postgresql/serializable-retry/broken-code.md`, `postgresql/serializable-retry/analysis.md`, `postgresql/serializable-retry/solutions.md`, `postgresql/serializable-retry/experiments.md`
- **Priority / stage:** P0 / Stage 3
- **Prerequisites:** `DB-005`, `DB-008`; refs `C-ISO`, `C-DEADLOCK`, `C-TEST`
- **Scope boundary:** Transaction-level retry; Spring interceptor ordering is `SPR-006`.

### DB-010 ? Concurrent Workers with FOR UPDATE SKIP LOCKED

- **Domain:** PostgreSQL, Spring
- **Problem:** Multiple workers poll the same jobs and either duplicate work or serialize behind the first locked row.
- **Primary concepts:** work claiming, `FOR UPDATE SKIP LOCKED`, fairness, starvation, crash recovery
- **Files:** `postgresql/skip-locked-work-queue/README.md`, `postgresql/skip-locked-work-queue/broken-code.md`, `postgresql/skip-locked-work-queue/analysis.md`, `postgresql/skip-locked-work-queue/solutions.md`, `postgresql/skip-locked-work-queue/experiments.md`
- **Priority / stage:** P1 / Stage 3
- **Prerequisites:** `DB-007`; refs `C-DB-LOCKS`, `C-TEST`
- **Scope boundary:** Database work claiming, not Kafka partition assignment.
## Locking and atomic mutation

### LOCK-001 ? Optimistic Locking with @Version

- **Domain:** JPA/Hibernate, PostgreSQL
- **Problem:** Two editors update the same entity; without a version predicate, the later flush silently overwrites the earlier commit.
- **Primary concepts:** `@Version`, affected-row check, `OptimisticLockException`
- **Files:** `locking/optimistic-version-conflict/README.md`, `locking/optimistic-version-conflict/broken-code.md`, `locking/optimistic-version-conflict/analysis.md`, `locking/optimistic-version-conflict/solutions.md`, `locking/optimistic-version-conflict/experiments.md`
- **Priority / stage:** P0 / Stage 3
- **Prerequisites:** `DB-001`; refs `C-OPT`, `C-MVCC`, `C-TEST`
- **Scope boundary:** Conflict detection; robust retry is `LOCK-002`.

### LOCK-002 ? Bounded Optimistic Retry Under Contention

- **Domain:** Spring, JPA/Hibernate
- **Problem:** Unbounded immediate retries amplify load, reuse stale persistence state, and can starve requests.
- **Primary concepts:** retry limit, new transaction, reload, backoff, jitter
- **Files:** `locking/optimistic-retry-contention/README.md`, `locking/optimistic-retry-contention/broken-code.md`, `locking/optimistic-retry-contention/analysis.md`, `locking/optimistic-retry-contention/solutions.md`, `locking/optimistic-retry-contention/experiments.md`
- **Priority / stage:** P1 / Stage 3
- **Prerequisites:** `LOCK-001`, `SPR-006`; refs `C-OPT`, `C-SPRING-TX`, `C-TEST`
- **Scope boundary:** Low/moderate contention; strategy choice is `LOCK-005`.

### LOCK-003 ? Pessimistic Write Lock with FOR UPDATE

- **Domain:** JPA/Hibernate, PostgreSQL
- **Problem:** A read-dependent mutation must prevent a competitor from making the same decision before the first transaction commits.
- **Primary concepts:** `PESSIMISTIC_WRITE`, `FOR UPDATE`, blocking, timeout, lock lifetime
- **Files:** `locking/pessimistic-write-for-update/README.md`, `locking/pessimistic-write-for-update/broken-code.md`, `locking/pessimistic-write-for-update/analysis.md`, `locking/pessimistic-write-for-update/solutions.md`, `locking/pessimistic-write-for-update/experiments.md`
- **Priority / stage:** P0 / Stage 3
- **Prerequisites:** `DB-007`; refs `C-PESS`, `C-DB-LOCKS`, `C-TEST`
- **Scope boundary:** Known rows; predicate-wide invariants may require constraints or serializable transactions.

### LOCK-004 ? Conditional Atomic UPDATE

- **Domain:** PostgreSQL, Spring Data JPA
- **Problem:** Application read-check-write permits overspend or negative stock although one guarded SQL mutation can protect the invariant.
- **Primary concepts:** conditional `UPDATE`, predicate recheck, affected rows, atomic mutation
- **Files:** `locking/conditional-atomic-update/README.md`, `locking/conditional-atomic-update/broken-code.md`, `locking/conditional-atomic-update/analysis.md`, `locking/conditional-atomic-update/solutions.md`, `locking/conditional-atomic-update/experiments.md`
- **Priority / stage:** P0 / Stage 3
- **Prerequisites:** `DB-001`, `DB-007`; refs `C-ATOMIC-SQL`, `C-DB-LOCKS`, `C-TEST`
- **Scope boundary:** Invariants expressible in one mutation predicate.

### LOCK-005 ? Strategy Selection Under High Contention

- **Domain:** Architecture, PostgreSQL
- **Problem:** A hot key uses a default lock that preserves correctness but collapses throughput or creates retry storms.
- **Primary concepts:** atomic SQL, optimistic/pessimistic lock, serial queue, contention
- **Files:** `locking/high-contention-strategy-selection/README.md`, `locking/high-contention-strategy-selection/broken-code.md`, `locking/high-contention-strategy-selection/analysis.md`, `locking/high-contention-strategy-selection/solutions.md`, `locking/high-contention-strategy-selection/experiments.md`
- **Priority / stage:** P2 / Stage 7
- **Prerequisites:** `LOCK-001`, `LOCK-002`, `LOCK-003`, `LOCK-004`, `DB-009`; refs `C-OPT`, `C-PESS`, `C-ATOMIC-SQL`, `C-TEST`
- **Scope boundary:** Qualitative selection framework; domain decisions stay in domain cases.

## Banking and fintech

### BANK-001 ? Concurrent Withdrawal and Double Spending

- **Domain:** Banking
- **Problem:** Two withdrawals both approve against the same available balance, allowing total debits beyond funds.
- **Primary concepts:** double spending, check-then-debit, atomic SQL, pessimistic lock
- **Files:** `banking/concurrent-withdrawal-double-spend/README.md`, `banking/concurrent-withdrawal-double-spend/broken-code.md`, `banking/concurrent-withdrawal-double-spend/analysis.md`, `banking/concurrent-withdrawal-double-spend/solutions.md`, `banking/concurrent-withdrawal-double-spend/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `DB-001`, `LOCK-003`, `LOCK-004`; refs `C-LEDGER`, `C-TEST`
- **Scope boundary:** Concurrent funds safety, not duplicate request prevention.

### BANK-002 ? Lost Account Balance Update

- **Domain:** Banking
- **Problem:** Concurrent independent credits/debits overwrite one another, leaving a plausible but incorrect balance.
- **Primary concepts:** lost update, balance projection, optimistic lock, atomic delta
- **Files:** `banking/lost-balance-update/README.md`, `banking/lost-balance-update/broken-code.md`, `banking/lost-balance-update/analysis.md`, `banking/lost-balance-update/solutions.md`, `banking/lost-balance-update/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `DB-001`, `LOCK-001`, `LOCK-004`; refs `C-LEDGER`, `C-TEST`
- **Scope boundary:** Balance corruption; overspending is `BANK-001`.

### BANK-003 ? Concurrent Transfer and Deterministic Lock Order

- **Domain:** Banking
- **Problem:** Transfers must atomically debit and credit two accounts while avoiding opposite-order deadlocks.
- **Primary concepts:** multi-row transaction, lock ordering, deadlock retry, conservation of money
- **Files:** `banking/concurrent-transfer-lock-order/README.md`, `banking/concurrent-transfer-lock-order/broken-code.md`, `banking/concurrent-transfer-lock-order/analysis.md`, `banking/concurrent-transfer-lock-order/solutions.md`, `banking/concurrent-transfer-lock-order/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `BANK-001`, `DB-008`, `LOCK-003`; refs `C-DEADLOCK`, `C-LEDGER`, `C-TEST`
- **Scope boundary:** Concurrent state mutation; duplicate transfer commands are `BANK-004`.

### BANK-004 ? Duplicate Transfer Request

- **Domain:** Banking, Fintech
- **Problem:** Client/network retries submit one logical transfer twice, producing two valid but unintended postings.
- **Primary concepts:** idempotency key, unique constraint, stored outcome, request fingerprint
- **Files:** `banking/duplicate-transfer-request/README.md`, `banking/duplicate-transfer-request/broken-code.md`, `banking/duplicate-transfer-request/analysis.md`, `banking/duplicate-transfer-request/solutions.md`, `banking/duplicate-transfer-request/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `DB-006`, `BANK-003`; refs `C-IDEMP`, `C-LEDGER`, `C-TEST`
- **Scope boundary:** Duplicate command prevention; balance concurrency remains `BANK-001`?`BANK-003`.

### BANK-005 ? Idempotent Payment Creation

- **Domain:** Payments
- **Problem:** Concurrent requests with the same `Idempotency-Key` create duplicate charges or return inconsistent responses.
- **Primary concepts:** atomic key claim, unique index, in-progress state, response replay
- **Files:** `banking/idempotent-payment-creation/README.md`, `banking/idempotent-payment-creation/broken-code.md`, `banking/idempotent-payment-creation/analysis.md`, `banking/idempotent-payment-creation/solutions.md`, `banking/idempotent-payment-creation/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `DB-006`, `BANK-004`; refs `C-IDEMP`, `C-TEST`
- **Scope boundary:** API idempotency lifecycle, not processor callback duplication.

### BANK-006 ? Duplicate and Out-of-Order Payment Callback

- **Domain:** Payments
- **Problem:** A provider retries callbacks and delivers older states after newer ones, causing duplicate fulfillment or status regression.
- **Primary concepts:** callback deduplication, monotonic state machine, event identity, ordering
- **Files:** `banking/payment-callback-duplication/README.md`, `banking/payment-callback-duplication/broken-code.md`, `banking/payment-callback-duplication/analysis.md`, `banking/payment-callback-duplication/solutions.md`, `banking/payment-callback-duplication/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `BANK-005`, `DB-006`; refs `C-IDEMP`, `C-TEST`
- **Scope boundary:** Incoming provider events; Kafka mechanics are `MSG-*`.

### BANK-007 ? Concurrent Ledger Posting and Balance Projection

- **Domain:** Banking, Ledger
- **Problem:** Append-only postings are correct individually but concurrent projection updates lose deltas or disagree with ledger totals.
- **Primary concepts:** append-only ledger, double entry, projection, atomic delta, auditability
- **Files:** `banking/ledger-posting-projection/README.md`, `banking/ledger-posting-projection/broken-code.md`, `banking/ledger-posting-projection/analysis.md`, `banking/ledger-posting-projection/solutions.md`, `banking/ledger-posting-projection/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `BANK-002`, `DB-006`, `LOCK-004`; refs `C-LEDGER`, `C-IDEMP`, `C-TEST`
- **Scope boundary:** Durable posting and projection; holds are `BANK-008`.

### BANK-008 ? Available versus Posted Balance During Authorization

- **Domain:** Banking, Cards
- **Problem:** Concurrent authorizations reserve the same funds because available balance omits or races with active holds.
- **Primary concepts:** authorization hold, available/posted balance, reservation, expiry
- **Files:** `banking/authorization-available-balance/README.md`, `banking/authorization-available-balance/broken-code.md`, `banking/authorization-available-balance/analysis.md`, `banking/authorization-available-balance/solutions.md`, `banking/authorization-available-balance/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `BANK-001`, `BANK-007`; refs `C-LEDGER`, `C-TEST`
- **Scope boundary:** Funds reservation; capture/reversal ordering is `BANK-009`.

### BANK-009 ? Settlement, Reversal, and Expiry Race

- **Domain:** Payments, Settlement
- **Problem:** Capture, reversal, and hold expiry execute concurrently, releasing and settling the same authorization inconsistently.
- **Primary concepts:** guarded state transition, monotonic workflow, ledger compensation, idempotency
- **Files:** `banking/settlement-reversal-race/README.md`, `banking/settlement-reversal-race/broken-code.md`, `banking/settlement-reversal-race/analysis.md`, `banking/settlement-reversal-race/solutions.md`, `banking/settlement-reversal-race/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `BANK-006`, `BANK-008`; refs `C-LEDGER`, `C-IDEMP`, `C-TEST`
- **Scope boundary:** One authorization lifecycle; cross-system reconciliation is `BANK-010`.

### BANK-010 ? Concurrent Settlement and Reconciliation Workers

- **Domain:** Fintech, Reconciliation
- **Problem:** Parallel workers claim overlapping settlement lines, double-adjust discrepancies, or reconcile against a moving cutoff.
- **Primary concepts:** work claiming, stable cutoff, idempotent adjustment, audit trail
- **Files:** `banking/settlement-reconciliation-workers/README.md`, `banking/settlement-reconciliation-workers/broken-code.md`, `banking/settlement-reconciliation-workers/analysis.md`, `banking/settlement-reconciliation-workers/solutions.md`, `banking/settlement-reconciliation-workers/experiments.md`
- **Priority / stage:** P1 / Stage 7
- **Prerequisites:** `BANK-007`, `BANK-009`, `DB-010`; refs `C-LEDGER`, `C-IDEMP`, `C-TEST`
- **Scope boundary:** Batch ownership and adjustments, not real-time authorization.
## E-commerce

### ECOM-001 ? Overselling Inventory

- **Domain:** E-commerce, Inventory
- **Problem:** Multiple buyers purchase the final units concurrently and all pass an application-side stock check.
- **Primary concepts:** overselling, lost update, atomic decrement, optimistic/pessimistic lock
- **Files:** `ecommerce/overselling-inventory/README.md`, `ecommerce/overselling-inventory/broken-code.md`, `ecommerce/overselling-inventory/analysis.md`, `ecommerce/overselling-inventory/solutions.md`, `ecommerce/overselling-inventory/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `DB-001`, `LOCK-001`, `LOCK-003`, `LOCK-004`; refs `C-ATOMIC-SQL`, `C-TEST`
- **Scope boundary:** Authoritative stock decrement, not order-request duplication.

### ECOM-002 ? Flash-Sale Hot-Stock Contention

- **Domain:** E-commerce, Flash Sale
- **Problem:** Correct inventory protection on a single hot SKU creates lock queues, optimistic retry storms, or database overload.
- **Primary concepts:** hot key, admission control, atomic SQL, serialization, backpressure
- **Files:** `ecommerce/flash-sale-hot-stock/README.md`, `ecommerce/flash-sale-hot-stock/broken-code.md`, `ecommerce/flash-sale-hot-stock/analysis.md`, `ecommerce/flash-sale-hot-stock/solutions.md`, `ecommerce/flash-sale-hot-stock/experiments.md`
- **Priority / stage:** P1 / Stage 7
- **Prerequisites:** `ECOM-001`, `LOCK-005`; refs `C-ATOMIC-SQL`, `C-TEST`
- **Scope boundary:** High-contention architecture; no invented benchmark numbers.

### ECOM-003 ? Duplicate Checkout and Concurrent Order Creation

- **Domain:** E-commerce, Orders
- **Problem:** Double-clicks and retries create multiple orders and payment attempts for one checkout intent.
- **Primary concepts:** idempotency key, unique business key, atomic order creation, response replay
- **Files:** `ecommerce/duplicate-checkout-order/README.md`, `ecommerce/duplicate-checkout-order/broken-code.md`, `ecommerce/duplicate-checkout-order/analysis.md`, `ecommerce/duplicate-checkout-order/solutions.md`, `ecommerce/duplicate-checkout-order/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `DB-006`, `BANK-005`; refs `C-IDEMP`, `C-TEST`
- **Scope boundary:** Duplicate command/order; stock safety remains `ECOM-001`.

### ECOM-004 ? Coupon and Limited-Voucher Double Redemption

- **Domain:** E-commerce, Promotion
- **Problem:** Concurrent redemptions exceed per-user or global usage limits after separate availability checks.
- **Primary concepts:** conditional update, unique redemption, multi-invariant transaction, contention
- **Files:** `ecommerce/coupon-double-redemption/README.md`, `ecommerce/coupon-double-redemption/broken-code.md`, `ecommerce/coupon-double-redemption/analysis.md`, `ecommerce/coupon-double-redemption/solutions.md`, `ecommerce/coupon-double-redemption/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `DB-006`, `LOCK-004`; refs `C-ATOMIC-SQL`, `C-IDEMP`, `C-TEST`
- **Scope boundary:** Coupon/voucher limits; loyalty balance is `ECOM-005`.

### ECOM-005 ? Concurrent Loyalty-Point Spending

- **Domain:** Loyalty, E-commerce
- **Problem:** Parallel purchases spend the same points or overwrite earn/redeem updates.
- **Primary concepts:** non-negative balance, ledger, atomic debit, idempotent redemption
- **Files:** `ecommerce/loyalty-point-concurrency/README.md`, `ecommerce/loyalty-point-concurrency/broken-code.md`, `ecommerce/loyalty-point-concurrency/analysis.md`, `ecommerce/loyalty-point-concurrency/solutions.md`, `ecommerce/loyalty-point-concurrency/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `BANK-001`, `BANK-007`; refs `C-LEDGER`, `C-IDEMP`, `C-TEST`
- **Scope boundary:** Points as auditable entitlement, not money settlement.

### ECOM-006 ? Shopping-Cart Lost Update

- **Domain:** E-commerce, Cart
- **Problem:** Browser tabs concurrently add/remove items and replace the whole cart from stale snapshots.
- **Primary concepts:** aggregate version, merge semantics, optimistic conflict, client intent
- **Files:** `ecommerce/shopping-cart-lost-update/README.md`, `ecommerce/shopping-cart-lost-update/broken-code.md`, `ecommerce/shopping-cart-lost-update/analysis.md`, `ecommerce/shopping-cart-lost-update/solutions.md`, `ecommerce/shopping-cart-lost-update/experiments.md`
- **Priority / stage:** P1 / Stage 4
- **Prerequisites:** `DB-001`, `LOCK-001`; refs `C-OPT`, `C-TEST`
- **Scope boundary:** Mutable cart aggregate; checkout finalization is `ECOM-003`.

### ECOM-007 ? Inventory Reservation Expiry versus Confirmation

- **Domain:** E-commerce, Inventory
- **Problem:** Expiry releases stock while checkout confirms the same reservation, producing oversell or a paid order without stock.
- **Primary concepts:** hold-confirm workflow, guarded transition, lease time, idempotency
- **Files:** `ecommerce/reservation-expiry-confirmation/README.md`, `ecommerce/reservation-expiry-confirmation/broken-code.md`, `ecommerce/reservation-expiry-confirmation/analysis.md`, `ecommerce/reservation-expiry-confirmation/solutions.md`, `ecommerce/reservation-expiry-confirmation/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `ECOM-001`, `ECOM-003`; refs `C-IDEMP`, `C-TEST`
- **Scope boundary:** Inventory hold lifecycle; seat holds are `BOOK-004`.

### ECOM-008 ? Concurrent Refund Requests

- **Domain:** E-commerce, Payments
- **Problem:** Customer service, automation, and callbacks refund the same charge concurrently or exceed the refundable amount.
- **Primary concepts:** refund idempotency, conditional amount, ledger entry, state transition
- **Files:** `ecommerce/concurrent-refund/README.md`, `ecommerce/concurrent-refund/broken-code.md`, `ecommerce/concurrent-refund/analysis.md`, `ecommerce/concurrent-refund/solutions.md`, `ecommerce/concurrent-refund/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `BANK-005`, `BANK-007`, `LOCK-004`; refs `C-IDEMP`, `C-LEDGER`, `C-TEST`
- **Scope boundary:** Refund correctness; provider callback ordering is `BANK-006`.

### ECOM-009 ? Order State-Transition Race

- **Domain:** E-commerce, Orders
- **Problem:** Payment, cancellation, fulfillment, and timeout actors concurrently move an order through incompatible states.
- **Primary concepts:** compare-and-set transition, monotonic state machine, stale command, event ordering
- **Files:** `ecommerce/order-state-transition-race/README.md`, `ecommerce/order-state-transition-race/broken-code.md`, `ecommerce/order-state-transition-race/analysis.md`, `ecommerce/order-state-transition-race/solutions.md`, `ecommerce/order-state-transition-race/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `LOCK-001`, `LOCK-004`, `ECOM-003`; refs `C-OPT`, `C-IDEMP`, `C-TEST`
- **Scope boundary:** Order lifecycle; cross-service saga is `DIST-004`.

## Booking

### BOOK-001 ? Hotel Room Capacity Race

- **Domain:** Booking, Hotel
- **Problem:** Concurrent bookings both see available room capacity for overlapping dates and exceed the property/type limit.
- **Primary concepts:** range overlap, phantom/predicate invariant, serializable, capacity bucket
- **Files:** `booking/hotel-room-capacity/README.md`, `booking/hotel-room-capacity/broken-code.md`, `booking/hotel-room-capacity/analysis.md`, `booking/hotel-room-capacity/solutions.md`, `booking/hotel-room-capacity/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `DB-004`, `DB-009`; refs `C-MVCC`, `C-ISO`, `C-TEST`
- **Scope boundary:** Capacity across date ranges, not one uniquely identified seat.

### BOOK-002 ? Flight Seat Double Booking

- **Domain:** Booking, Airline
- **Problem:** Two passengers confirm the same flight seat after concurrent availability checks.
- **Primary concepts:** unique confirmed seat, reservation state, atomic claim, constraint
- **Files:** `booking/flight-seat-double-booking/README.md`, `booking/flight-seat-double-booking/broken-code.md`, `booking/flight-seat-double-booking/analysis.md`, `booking/flight-seat-double-booking/solutions.md`, `booking/flight-seat-double-booking/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `DB-006`, `LOCK-003`; refs `C-IDEMP`, `C-TEST`
- **Scope boundary:** One seat on one flight; temporary holds are `BOOK-004`.

### BOOK-003 ? Cinema Seat Batch-Selection Race

- **Domain:** Booking, Cinema
- **Problem:** Concurrent users claim overlapping multi-seat selections, leaving a group partially booked or double-confirmed.
- **Primary concepts:** multi-row atomicity, deterministic lock order, unique seat constraint, all-or-nothing
- **Files:** `booking/cinema-seat-batch-race/README.md`, `booking/cinema-seat-batch-race/broken-code.md`, `booking/cinema-seat-batch-race/analysis.md`, `booking/cinema-seat-batch-race/solutions.md`, `booking/cinema-seat-batch-race/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `BOOK-002`, `DB-008`; refs `C-DB-LOCKS`, `C-DEADLOCK`, `C-TEST`
- **Scope boundary:** Atomic multi-seat groups, unlike the single-seat invariant in `BOOK-002`.

### BOOK-004 ? Hold, Confirm, and Timeout Race

- **Domain:** Booking
- **Problem:** Confirmation and timeout workers race after a hold's deadline, so a seat is both released and confirmed.
- **Primary concepts:** hold token, guarded transition, authoritative clock, expiry worker, idempotency
- **Files:** `booking/hold-confirm-timeout/README.md`, `booking/hold-confirm-timeout/broken-code.md`, `booking/hold-confirm-timeout/analysis.md`, `booking/hold-confirm-timeout/solutions.md`, `booking/hold-confirm-timeout/experiments.md`
- **Priority / stage:** P0 / Stage 4
- **Prerequisites:** `BOOK-002`, `DB-010`; refs `C-IDEMP`, `C-TEST`
- **Scope boundary:** Reservation lifecycle shared by hotel/flight/cinema; distributed leases are `REDIS-004`.
## Messaging

### MSG-001 ? Kafka At-Least-Once Delivery and Database Side Effects

- **Domain:** Messaging, Kafka
- **Problem:** A consumer commits a database side effect and crashes before offset commit, so redelivery applies the effect twice.
- **Primary concepts:** at-least-once, offset timing, external side effect, exactly-once boundary
- **Files:** `messaging/kafka-at-least-once-side-effect/README.md`, `messaging/kafka-at-least-once-side-effect/broken-code.md`, `messaging/kafka-at-least-once-side-effect/analysis.md`, `messaging/kafka-at-least-once-side-effect/solutions.md`, `messaging/kafka-at-least-once-side-effect/experiments.md`
- **Priority / stage:** P0 / Stage 5
- **Prerequisites:** `DB-006`; refs `C-KAFKA`, `C-IDEMP`, `C-TEST`
- **Scope boundary:** Delivery/commit gap; durable dedup implementation is `MSG-006`.

### MSG-002 ? Concurrent Consumers Update One Aggregate

- **Domain:** Messaging, Kafka
- **Problem:** Events for one business aggregate are processed in parallel and overwrite shared database state despite consumer-group coordination.
- **Primary concepts:** partition key, group parallelism, lost update, aggregate serialization
- **Files:** `messaging/concurrent-consumer-aggregate-update/README.md`, `messaging/concurrent-consumer-aggregate-update/broken-code.md`, `messaging/concurrent-consumer-aggregate-update/analysis.md`, `messaging/concurrent-consumer-aggregate-update/solutions.md`, `messaging/concurrent-consumer-aggregate-update/experiments.md`
- **Priority / stage:** P0 / Stage 5
- **Prerequisites:** `MSG-001`, `DB-001`; refs `C-KAFKA`, `C-OPT`, `C-TEST`
- **Scope boundary:** Parallel mutation; semantic out-of-order delivery is `MSG-003`.

### MSG-003 ? Out-of-Order Events Regress State

- **Domain:** Messaging, Event-Driven Systems
- **Problem:** An older shipment/payment/order event arrives after a newer one and regresses a materialized state.
- **Primary concepts:** per-key ordering, sequence/version, monotonic transition, stale-event rejection
- **Files:** `messaging/out-of-order-event-state/README.md`, `messaging/out-of-order-event-state/broken-code.md`, `messaging/out-of-order-event-state/analysis.md`, `messaging/out-of-order-event-state/solutions.md`, `messaging/out-of-order-event-state/experiments.md`
- **Priority / stage:** P0 / Stage 5
- **Prerequisites:** `MSG-002`, `ECOM-009`; refs `C-KAFKA`, `C-TEST`
- **Scope boundary:** Event ordering; duplicate identity is `MSG-006`.

### MSG-004 ? Retry Topic Duplicates a Successful Attempt

- **Domain:** Messaging, Kafka
- **Problem:** Processing succeeds but acknowledgment or retry publication fails ambiguously, so the original and retry both execute.
- **Primary concepts:** retry duplication, timeout ambiguity, attempt identity, retry topic
- **Files:** `messaging/retry-topic-duplication/README.md`, `messaging/retry-topic-duplication/broken-code.md`, `messaging/retry-topic-duplication/analysis.md`, `messaging/retry-topic-duplication/solutions.md`, `messaging/retry-topic-duplication/experiments.md`
- **Priority / stage:** P0 / Stage 5
- **Prerequisites:** `MSG-001`; refs `C-KAFKA`, `C-IDEMP`, `C-TEST`
- **Scope boundary:** Retry path; permanently failing records are `MSG-005`.

### MSG-005 ? Poison Message Retry Starvation

- **Domain:** Messaging, Kafka
- **Problem:** One permanently failing record retries indefinitely, blocks a partition, exhausts workers, or creates a retry storm.
- **Primary concepts:** poison message, bounded retry, dead-letter handling, partition progress
- **Files:** `messaging/poison-message-starvation/README.md`, `messaging/poison-message-starvation/broken-code.md`, `messaging/poison-message-starvation/analysis.md`, `messaging/poison-message-starvation/solutions.md`, `messaging/poison-message-starvation/experiments.md`
- **Priority / stage:** P1 / Stage 5
- **Prerequisites:** `MSG-004`; refs `C-KAFKA`, `C-TEST`
- **Scope boundary:** Availability/progress while preserving diagnostic and replay safety.

### MSG-006 ? Idempotent Consumer with Inbox

- **Domain:** Messaging, PostgreSQL
- **Problem:** A check-then-process consumer records message identity too late or outside the business transaction, permitting duplicate side effects.
- **Primary concepts:** inbox table, unique message ID, atomic dedup and mutation, replay
- **Files:** `messaging/idempotent-consumer-inbox/README.md`, `messaging/idempotent-consumer-inbox/broken-code.md`, `messaging/idempotent-consumer-inbox/analysis.md`, `messaging/idempotent-consumer-inbox/solutions.md`, `messaging/idempotent-consumer-inbox/experiments.md`
- **Priority / stage:** P0 / Stage 5
- **Prerequisites:** `MSG-001`, `DB-006`; refs `C-INBOX-OUTBOX`, `C-IDEMP`, `C-TEST`
- **Scope boundary:** Incoming-message atomicity; outgoing publication is `MSG-007`.

### MSG-007 ? Transactional Outbox Publication Race

- **Domain:** Messaging, PostgreSQL, Kafka
- **Problem:** A service commits business state without its event, publishes before rollback, or lets concurrent relays publish an outbox row repeatedly.
- **Primary concepts:** dual write, outbox, relay claiming, duplicate publication, consumer idempotency
- **Files:** `messaging/transactional-outbox-publication/README.md`, `messaging/transactional-outbox-publication/broken-code.md`, `messaging/transactional-outbox-publication/analysis.md`, `messaging/transactional-outbox-publication/solutions.md`, `messaging/transactional-outbox-publication/experiments.md`
- **Priority / stage:** P0 / Stage 5
- **Prerequisites:** `SPR-002`, `DB-010`, `MSG-006`; refs `C-INBOX-OUTBOX`, `C-KAFKA`, `C-TEST`
- **Scope boundary:** Atomic database state plus publish intent; cross-resource transactions are `DIST-006`.

## Redis and cache

### REDIS-001 ? GET-Then-SET Counter Race

- **Domain:** Redis
- **Problem:** Multiple clients increment or enforce a limit with `GET` plus `SET`, losing increments or exceeding capacity.
- **Primary concepts:** command atomicity, compound race, `INCR`, Lua script, authoritative limit
- **Files:** `redis/get-set-counter-race/README.md`, `redis/get-set-counter-race/broken-code.md`, `redis/get-set-counter-race/analysis.md`, `redis/get-set-counter-race/solutions.md`, `redis/get-set-counter-race/experiments.md`
- **Priority / stage:** P1 / Stage 6
- **Prerequisites:** `JVM-005`, `LOCK-004`; refs `C-TEST`
- **Scope boundary:** Atomic Redis mutation; durability/authority assumptions must be explicit.

### REDIS-002 ? SET NX Claim and Allocation Lifecycle

- **Domain:** Redis
- **Problem:** A `SET NX` claim prevents simultaneous creation but has no safe completion/retry model, leaving stuck or duplicate allocations.
- **Primary concepts:** atomic claim, TTL, result record, failure recovery, one-time allocation
- **Files:** `redis/set-nx-allocation-claim/README.md`, `redis/set-nx-allocation-claim/broken-code.md`, `redis/set-nx-allocation-claim/analysis.md`, `redis/set-nx-allocation-claim/solutions.md`, `redis/set-nx-allocation-claim/experiments.md`
- **Priority / stage:** P1 / Stage 6
- **Prerequisites:** `REDIS-001`, `DB-006`; refs `C-IDEMP`, `C-LEASE`, `C-TEST`
- **Scope boundary:** Claim lifecycle; mutual-exclusion lock ownership is `REDIS-003`.

### REDIS-003 ? Distributed Lock TTL and Safe Unlock

- **Domain:** Redis, Distributed Systems
- **Problem:** A client deletes a lock it no longer owns after TTL expiry, or assumes acquisition alone guarantees exclusive work.
- **Primary concepts:** `SET NX PX`, owner token, Lua compare-delete, TTL, Redlock assumptions
- **Files:** `redis/distributed-lock-safe-unlock/README.md`, `redis/distributed-lock-safe-unlock/broken-code.md`, `redis/distributed-lock-safe-unlock/analysis.md`, `redis/distributed-lock-safe-unlock/solutions.md`, `redis/distributed-lock-safe-unlock/experiments.md`
- **Priority / stage:** P0 / Stage 6
- **Prerequisites:** `REDIS-002`; refs `C-LEASE`, `C-TEST`
- **Scope boundary:** Ownership-safe unlock; stale-owner writes are `REDIS-004`.

### REDIS-004 ? Lease Expiry Requires a Fencing Token

- **Domain:** Redis, Distributed Systems
- **Problem:** Client A pauses, its lease expires, Client B acquires the lock, then A resumes and writes stale state concurrently with B.
- **Primary concepts:** process pause, lease expiry, fencing token, stale-owner rejection
- **Files:** `redis/lease-expiry-fencing-token/README.md`, `redis/lease-expiry-fencing-token/broken-code.md`, `redis/lease-expiry-fencing-token/analysis.md`, `redis/lease-expiry-fencing-token/solutions.md`, `redis/lease-expiry-fencing-token/experiments.md`
- **Priority / stage:** P0 / Stage 6
- **Prerequisites:** `REDIS-003`; refs `C-LEASE`, `C-TEST`
- **Scope boundary:** Protected resource must enforce fencing; lock ownership alone is insufficient.

### REDIS-005 ? Cache-Aside Invalidation Race

- **Domain:** Redis, Cache
- **Problem:** A reader loads old database state and writes it to cache after a concurrent updater commits and invalidates, resurrecting stale data.
- **Primary concepts:** cache-aside timing, stale fill, invalidation, authority, versioned cache value
- **Files:** `redis/cache-aside-invalidation-race/README.md`, `redis/cache-aside-invalidation-race/broken-code.md`, `redis/cache-aside-invalidation-race/analysis.md`, `redis/cache-aside-invalidation-race/solutions.md`, `redis/cache-aside-invalidation-race/experiments.md`
- **Priority / stage:** P1 / Stage 6
- **Prerequisites:** `DB-003`; refs `C-CACHE`, `C-TEST`
- **Scope boundary:** Stale consistency window; concurrent miss amplification is `REDIS-006`.

### REDIS-006 ? Concurrent Cache Miss and Stampede

- **Domain:** Redis, Cache
- **Problem:** Many requests miss an expired hot key and concurrently overload the database or downstream service.
- **Primary concepts:** cache stampede, request coalescing, soft TTL, jitter, stale-while-revalidate
- **Files:** `redis/cache-stampede/README.md`, `redis/cache-stampede/broken-code.md`, `redis/cache-stampede/analysis.md`, `redis/cache-stampede/solutions.md`, `redis/cache-stampede/experiments.md`
- **Priority / stage:** P1 / Stage 7
- **Prerequisites:** `REDIS-005`, `JVM-002`; refs `C-CACHE`, `C-TEST`
- **Scope boundary:** Load/progress problem; correctness still belongs to the authoritative store.

## Distributed systems

### DIST-001 ? Local Lock Fails Across Application Instances

- **Domain:** Distributed Systems, Spring
- **Problem:** `synchronized` or `ReentrantLock` serializes App A but App B concurrently mutates the same database record.
- **Primary concepts:** process boundary, load balancing, shared authority, database coordination
- **Files:** `distributed/local-lock-multiple-instances/README.md`, `distributed/local-lock-multiple-instances/broken-code.md`, `distributed/local-lock-multiple-instances/analysis.md`, `distributed/local-lock-multiple-instances/solutions.md`, `distributed/local-lock-multiple-instances/experiments.md`
- **Priority / stage:** P0 / Stage 6
- **Prerequisites:** `JVM-006`, `LOCK-004`; refs `C-ATOMIC-SQL`, `C-TEST`
- **Scope boundary:** Demonstrates node boundary; Redis lease failure is `REDIS-003`?`REDIS-004`.

### DIST-002 ? Network Retry and Timeout Ambiguity

- **Domain:** Distributed Systems
- **Problem:** A caller times out after the server may have committed, retries, and duplicates an irreversible operation.
- **Primary concepts:** unknown outcome, retry duplication, idempotency key, status lookup, reconciliation
- **Files:** `distributed/network-timeout-ambiguity/README.md`, `distributed/network-timeout-ambiguity/broken-code.md`, `distributed/network-timeout-ambiguity/analysis.md`, `distributed/network-timeout-ambiguity/solutions.md`, `distributed/network-timeout-ambiguity/experiments.md`
- **Priority / stage:** P0 / Stage 6
- **Prerequisites:** `BANK-005`; refs `C-IDEMP`, `C-TEST`
- **Scope boundary:** Request/response ambiguity, not broker redelivery.

### DIST-003 ? Eventual-Consistency Stale Read Drives a Write

- **Domain:** Distributed Systems, Replicated Database
- **Problem:** A decision reads stale replica/cache state and writes to the leader, violating an invariant the stale reader thought held.
- **Primary concepts:** stale read, read-your-writes, leader routing, version token, invariant authority
- **Files:** `distributed/stale-read-write-decision/README.md`, `distributed/stale-read-write-decision/broken-code.md`, `distributed/stale-read-write-decision/analysis.md`, `distributed/stale-read-write-decision/solutions.md`, `distributed/stale-read-write-decision/experiments.md`
- **Priority / stage:** P0 / Stage 6
- **Prerequisites:** `DB-003`, `REDIS-005`; refs `C-CACHE`, `C-TEST`
- **Scope boundary:** Replication/cache staleness; single-primary MVCC is `DB-*`.

### DIST-004 ? Concurrent Saga Step and Compensation

- **Domain:** Distributed Systems, Saga
- **Problem:** A delayed success races with timeout compensation, so a reservation is both confirmed and released or charged and refunded.
- **Primary concepts:** saga race, compensation, semantic lock, monotonic state, idempotent step
- **Files:** `distributed/saga-compensation-race/README.md`, `distributed/saga-compensation-race/broken-code.md`, `distributed/saga-compensation-race/analysis.md`, `distributed/saga-compensation-race/solutions.md`, `distributed/saga-compensation-race/experiments.md`
- **Priority / stage:** P0 / Stage 6
- **Prerequisites:** `ECOM-009`, `DIST-002`, `MSG-003`; refs `C-IDEMP`, `C-TEST`
- **Scope boundary:** Workflow correctness without pretending compensation is rollback.

### DIST-005 ? Cross-Service Event Ordering

- **Domain:** Distributed Systems, Messaging
- **Problem:** Events from different producers or partitions represent one workflow but arrive in an order that violates consumer assumptions.
- **Primary concepts:** partial order, causal metadata, aggregate ownership, versioned command
- **Files:** `distributed/cross-service-event-ordering/README.md`, `distributed/cross-service-event-ordering/broken-code.md`, `distributed/cross-service-event-ordering/analysis.md`, `distributed/cross-service-event-ordering/solutions.md`, `distributed/cross-service-event-ordering/experiments.md`
- **Priority / stage:** P1 / Stage 6
- **Prerequisites:** `MSG-003`, `DIST-004`; refs `C-KAFKA`, `C-TEST`
- **Scope boundary:** Cross-producer causality; within-partition ordering is covered by `MSG-002`?`MSG-003`.

### DIST-006 ? Dual Write without a Distributed Transaction

- **Domain:** Distributed Systems
- **Problem:** A service updates its database and a second resource separately, so failure leaves only one side committed.
- **Primary concepts:** distributed transaction, dual write, outbox, idempotent receiver, recovery
- **Files:** `distributed/dual-write-partial-commit/README.md`, `distributed/dual-write-partial-commit/broken-code.md`, `distributed/dual-write-partial-commit/analysis.md`, `distributed/dual-write-partial-commit/solutions.md`, `distributed/dual-write-partial-commit/experiments.md`
- **Priority / stage:** P0 / Stage 6
- **Prerequisites:** `SPR-003`, `MSG-007`, `DIST-002`; refs `C-INBOX-OUTBOX`, `C-IDEMP`, `C-TEST`
- **Scope boundary:** XA/trade-off and recovery choice; outbox mechanics remain `MSG-007`.
---

# Coverage and Phase 1 completion

## Required-topic coverage map

| Required area | Catalog coverage |
| --- | --- |
| JVM shared state, collections, compound races, visibility, atomics | `JVM-001`?`JVM-005` |
| `synchronized`, local locks, deadlock, livelock, starvation, executors, futures | `JVM-006`?`JVM-010` |
| Spring proxy, `@Transactional`, `@Async`, propagation, isolation, rollback, retry, connection pool | `SPR-001`?`SPR-007` |
| MVCC, dirty/non-repeatable/phantom reads, write skew, uniqueness | `DB-001`?`DB-006` |
| Row/table locks, deadlock, `SERIALIZABLE`, `SKIP LOCKED` | `DB-007`?`DB-010` |
| Optimistic, pessimistic, atomic SQL, high-contention choice | `LOCK-001`?`LOCK-005` |
| Withdrawal, transfer, duplicate request/payment, callbacks, ledger, holds, reversal, settlement, reconciliation | `BANK-001`?`BANK-010` |
| Oversell, flash sale, order/checkout, coupons, points, cart, reservation, refund, state transitions | `ECOM-001`?`ECOM-009` |
| Hotel, flight, cinema, hold-confirm-timeout | `BOOK-001`?`BOOK-004` |
| Kafka delivery, consumer concurrency, ordering, retry, poison record, inbox, outbox | `MSG-001`?`MSG-007` |
| Redis counter/Lua, `SET NX`, distributed lock, expiry/fencing, invalidation, stampede | `REDIS-001`?`REDIS-006` |
| Multiple nodes, local-lock failure, retry ambiguity, stale reads, saga, event causality, dual write | `DIST-001`?`DIST-006` |

## Deliberate merge and separation decisions

- Spring singleton state is taught once in `JVM-001`; `SPR-*` focuses on transaction machinery.
- Check-then-insert and unique constraints are foundational in `DB-006`; payment, order, and inbox cases apply them without repeating the full concept.
- JVM deadlock (`JVM-007`) and PostgreSQL deadlock (`DB-008`) remain separate because their detection, victim handling, and observability differ.
- Optimistic conflict (`LOCK-001`) and retry policy (`LOCK-002`) remain separate so retry boundaries are not hidden inside the locking introduction.
- Duplicate checkout and concurrent order creation are one invariant in `ECOM-003`; coupon and limited-voucher redemption are one limit-enforcement case in `ECOM-004`.
- Duplicate payment request (`BANK-005`) and duplicate/out-of-order provider callback (`BANK-006`) remain separate because their trust boundary and replay contract differ.
- Seat-specific flight/cinema cases remain separate from predicate/range hotel capacity and share the generic hold lifecycle in `BOOK-004`.
- Kafka redelivery (`MSG-001`), inbox deduplication (`MSG-006`), and outbox publication (`MSG-007`) remain separate transaction boundaries.
- Redis lock ownership (`REDIS-003`) and stale-owner fencing (`REDIS-004`) remain separate to make clear why safe unlock is not sufficient.
- Domain state ordering (`ECOM-009`), Kafka ordering (`MSG-003`), and cross-service causal ordering (`DIST-005`) are linked but not conflated.

## Phase 1 completion checklist

- [x] Taxonomy designed and ordered.
- [x] All 74 cases have stable IDs.
- [x] Every case has one unique folder path and five planned files.
- [x] Learning stages and explicit prerequisites are recorded.
- [x] Priorities use P0?P3 production-risk semantics.
- [x] Duplicate topics were merged or deliberately separated by layer/failure model.
- [x] Scope boundaries are explicit for every case.
- [x] Shared references are planned and will be generated on demand.
- [x] `RULES.md` defines the Phase 2 quality gate.
- [x] No case content, placeholder folder, shared concept, or sample application was created.

Phase 1 is complete. The first learning-order build is `JVM-001`; a Phase 2 run must still provide `MODE=BUILD` and an explicit `CASE_ID`.
