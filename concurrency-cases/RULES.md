# Concurrency Case Library — Authoring Rules

## 1. Authority and scope

`CONCURRENCY_CASE_LIBRARY.md` is the single source of truth for case IDs, paths,
priority, prerequisites, and scope. This file defines how a cataloged case is
implemented.

- Phase 1 creates only governance files. It does not create case folders,
  shared-concept files, code samples, or experiments.
- Phase 2 builds exactly one `CASE_ID` per run unless the input explicitly
  authorizes a small, closely related batch.
- Do not rename an ID, folder, file, or taxonomy after Phase 1 without recording
  the reason in the catalog.
- Do not create uncataloged cases. Amend the catalog first when scope changes.
- Java 21, Spring Boot, Spring Data JPA, Hibernate, and PostgreSQL are the
  default stack. Redis and Kafka are used only when their failure model is part
  of the case.

## 2. Phase 2 read boundary

Before building a case, read only:

1. its entry in `CONCURRENCY_CASE_LIBRARY.md`;
2. this file;
3. its existing target folder, if any;
4. the shared concept files named by its prerequisites.

Do not scan the entire repository unless validation cannot otherwise be
completed.

## 3. Language and terminology

- All explanatory content in Phase 2 Markdown files must be written in Vietnamese.
- Keep canonical technical terms in English, for example: `race condition`,
  `lost update`, `write skew`, `optimistic locking`, `pessimistic locking`,
  `idempotency`, `deadlock`, `livelock`, `starvation`, `fencing token`, and
  `transactional outbox`.
- Add a Vietnamese translation or short explanation in parentheses when it
  materially improves comprehension, especially on the first occurrence. The
  English term remains the canonical term used throughout the case.
- Do not translate Java identifiers, annotations, exception names, API names,
  SQL keywords, PostgreSQL concepts, Redis/Kafka commands, configuration keys,
  filenames, folder names, or `CASE_ID` values.
- Headings, prose, timelines, table commentary, test explanations, production
  considerations, and trade-off analysis must be Vietnamese. Code and SQL
  remain idiomatic and may use English identifiers.
- Use one stable English term consistently; do not alternate between multiple
  Vietnamese translations that could imply different concepts.

## 4. Standard case bundle

Every case uses the five files declared in its catalog entry:

### `README.md`

- Business scenario and actors
- Explicit invariant
- Relevant data and transaction boundaries
- Contention point
- Short navigation map to the other case files
- Production consequences
- When to use the recommended approach

### `broken-code.md`

- Minimal, realistic Java/Spring/JPA implementation
- Supporting entity, repository, SQL, or configuration only when needed
- Preconditions that make the bug reproducible
- No deliberately foolish code and no pseudocode when real code is practical

### `analysis.md`

- Initial state
- Mandatory two-or-more-actor interleaving timeline
- Expected versus actual result
- Exact root cause at the JVM, Spring, Hibernate, PostgreSQL, Redis, Kafka, or
  distributed-system layer
- MVCC snapshot and lock behavior when relevant
- Commit, rollback, timeout, retry, and crash behavior
- Why a JVM-local primitive does or does not protect multiple application nodes

### `solutions.md`

- At least one correct solution with complete-enough Java and SQL
- Why the invariant is protected
- Where conflict is detected
- Whether the loser blocks, fails, retries, or observes a no-op
- Comparison of credible alternatives
- Qualitative trade-offs: correctness, throughput, latency, contention, retry
  rate, deadlock risk, database load, operational complexity, and horizontal
  scalability
- No invented benchmarks or production numbers

### `experiments.md`

- Deterministic or probabilistically repeatable concurrency test strategy
- `CountDownLatch`, barriers, futures, or executor coordination rather than
  `Thread.sleep` as the primary synchronization mechanism
- PostgreSQL Testcontainers for MVCC, isolation, row locks, deadlocks,
  `@Version`, `FOR UPDATE`, `SKIP LOCKED`, and `SERIALIZABLE`
- Assertions on business invariants, not only exception types
- Observability and production-verification notes

Files may remain compact, but required sections must not be moved into new
micro-files. A sixth file requires a catalog amendment and a clear independent
purpose.

## 5. Required case narrative

Every case must tell this sequence:

> concrete production problem → broken implementation → race timeline →
> consequence → correct solution → why it works → trade-off

The case must name:

- shared business state;
- concurrent actors;
- transaction boundaries;
- invariant;
- contention point;
- failure/recovery behavior;
- multi-instance implications.

“Multiple requests run at the same time” is never a sufficient root cause.
Identify the non-atomic sequence, such as `read → decide → write` or
`check → insert`.

## 6. Correctness rules

### JVM versus database coordination

`synchronized`, `ReentrantLock`, atomics, and concurrent collections coordinate
threads only inside the JVM that owns them. They do not coordinate App A with
App B behind a load balancer. A multi-instance invariant must ultimately be
enforced at the authoritative shared boundary, commonly a conditional database
write, database constraint, row lock, serializable transaction, durable
idempotency record, or carefully designed lease plus fencing.

### Transactions

- Show the real proxy and transaction boundary.
- State propagation and isolation where they affect behavior.
- Explain Hibernate flush timing when it affects conflict detection.
- A retry must start a new transaction and reload state.
- State which exceptions roll back and which do not.
- Keep blocking transactions short and exclude remote I/O where practical.

### PostgreSQL

For database-centered cases, explain:

- the effective PostgreSQL isolation behavior;
- the snapshot each statement/transaction observes;
- the lock or predicate conflict involved;
- lock acquisition and release;
- behavior of the competing transaction;
- commit and rollback effects.

Do not claim PostgreSQL permits dirty reads: its `READ UNCOMMITTED` behaves as
`READ COMMITTED`. Do not use H2 as evidence for PostgreSQL concurrency
semantics.

### Optimistic locking

When using `@Version`, include the versioned `UPDATE ... WHERE version = ?`,
the affected-row count, Hibernate's conflict signal, and a bounded retry with
backoff/jitter where retry is appropriate. Discuss why high contention can
cause retry amplification.

### Pessimistic locking

When using `PESSIMISTIC_WRITE` or `FOR UPDATE`, show:

- the first transaction acquiring the row lock;
- the second blocking, timing out, or failing;
- lock lifetime through commit/rollback;
- deterministic lock order for multi-row operations;
- deadlock and lock-timeout handling.

### Atomic database operations

Treat conditional SQL as a first-class solution:

```sql
UPDATE inventory
SET stock = stock - :quantity
WHERE product_id = :id
  AND stock >= :quantity;
```

Explain the row lock, re-evaluation of the predicate, affected-row count, and
how that outcome maps to a domain result.

### Idempotency and uniqueness

Never present `exists` followed by `insert` as safe. Use a database uniqueness
constraint or another atomic claim. Distinguish:

- duplicate-command prevention; and
- concurrent mutation safety.

They solve different invariants. Specify whether stored responses, request
fingerprints, failed attempts, and in-progress attempts are replayed.

### Financial state

Do not assume a mutable balance is the only model. Distinguish ledger entries,
posted balance, available balance, authorization/hold, capture/settlement,
reversal, and reconciliation. Financial history must be append-only and
auditable when the case uses a ledger.

### Kafka and messaging

Distinguish partition ordering, consumer-group parallelism, redelivery,
database side effects, and Kafka transaction scope. Never claim an external
database side effect is exactly once merely because Kafka transactions are in
use.

### Redis and leases

Redis is not a magic correctness boundary. For locks and leases, analyze
acquisition, TTL, process pause, partition, expiry, owner token, safe unlock,
and fencing. Discuss Redlock only in the context of its assumptions and the
protected resource's ability to reject stale owners.

### Cache

State which store is authoritative. Cover cache-aside timing, stale reads,
invalidation failure, concurrent misses, and recovery. Never silently promote a
cache to the system of record.

## 7. Solution selection

Choose solutions from the invariant and failure model, not from a preferred
primitive. Compare relevant options using:

- contention and hot-key concentration;
- criticality and loss tolerance;
- transaction duration;
- database topology;
- number of application instances;
- retry tolerance;
- throughput and latency requirements;
- operational failure modes.

Prefer the smallest authoritative mechanism that protects the invariant.
Database constraints and atomic writes generally outrank a distributed lock
for a single-database invariant. A queue is justified only when serialization,
backpressure, ordering, or asynchronous workflow is actually required.

## 8. Shared concepts

Shared concept files are created on demand at the paths in the catalog. A case
links to them and explains only its case-specific application. Do not paste the
full general treatment of MVCC, isolation, `@Version`, `FOR UPDATE`,
idempotency, deadlocks, Kafka delivery, leases, or cache consistency into every
case.

A shared concept file must be independently useful and referenced by at least
two cases. Otherwise keep the explanation in the case.

## 9. Java style

- Java 21+
- Constructor injection; no field injection
- Small domain-focused samples
- Descriptive names and explicit transaction boundaries
- No fake enterprise layers or unnecessary abstractions
- Compilable or near-compilable code with imports/configuration described when
  omitted
- No giant sample application when a service, repository, entity, and test are
  sufficient

## 10. Phase 2 quality gate

Before marking a case complete, verify:

- [ ] Catalog ID, folder, and filenames are unchanged.
- [ ] Business invariant and contention point are explicit.
- [ ] Broken implementation is realistic.
- [ ] Race timeline shows a concrete interleaving.
- [ ] Expected and actual outcomes are compared.
- [ ] Root cause identifies the correct layer and non-atomic operation.
- [ ] Technical and business consequences are included.
- [ ] Fixed Java code exists.
- [ ] SQL exists where database behavior matters.
- [ ] Transaction, snapshot, lock, commit, and rollback behavior are clear.
- [ ] Multi-instance behavior is considered.
- [ ] Retry, timeout, crash, and duplicate behavior are considered where
      relevant.
- [ ] The solution protects the invariant.
- [ ] Alternatives and qualitative trade-offs are compared.
- [ ] Test strategy asserts the invariant.
- [ ] PostgreSQL Testcontainers is used when PostgreSQL semantics matter.
- [ ] Shared concepts are linked instead of duplicated.
- [ ] No unnecessary architecture or files were added.

## 11. Completion report

A Phase 2 run reports only:

```text
Created:
...

Updated:
...

Case:
...

Next suggested case:
...
```

Then stop. Do not begin another case automatically.
