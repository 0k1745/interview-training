# 9. Database Concurrency & Architecture (PostgreSQL)

**Context**: This is the "database level" half of the interview's stated focus ("concurrency & multithreading on both database and application level" + "experience with PostgreSQL and database architecture"). Pair this with [08-thread-safety-and-concurrency-patterns.md](./08-thread-safety-and-concurrency-patterns.md).

---

## 1. Transactions & the ACID Guarantees

| Guarantee | Meaning | PostgreSQL mechanism |
|---|---|---|
| **Atomicity** | All statements in a transaction commit or none do | WAL (Write-Ahead Log) + rollback segments |
| **Consistency** | DB moves from one valid state to another | Constraints, triggers, foreign keys |
| **Isolation** | Concurrent transactions don't see each other's uncommitted changes | MVCC (see below) |
| **Durability** | Committed data survives a crash | WAL fsync'd to disk before commit ack |

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'acc-1';
UPDATE accounts SET balance = balance + 100 WHERE id = 'acc-2';
COMMIT;
```

If the process crashes between the two `UPDATE`s, PostgreSQL rolls back the entire transaction on recovery — the money transfer either fully happens or not at all.

---

## 2. MVCC: How PostgreSQL Handles Concurrency Without Blocking Readers

PostgreSQL uses **Multi-Version Concurrency Control**: instead of locking rows for reads, every row version is tagged with the transaction IDs (`xmin`/`xmax`) that created/deleted it. Each transaction sees a *snapshot* of the database as of its start (depending on isolation level).

- Readers never block writers.
- Writers never block readers.
- Writers **do** block writers on the same row (until commit/rollback).

This is why PostgreSQL rarely needs `SELECT ... LOCK IN SHARE MODE` for read consistency, unlike older locking-based engines.

Consequence senior engineers must know: **dead tuples accumulate** from updates/deletes (old row versions aren't removed immediately) — this is what `VACUUM` (autovacuum) cleans up. Neglecting autovacuum tuning under high write throughput is a classic production incident (table bloat, transaction ID wraparound).

---

## 3. Isolation Levels

PostgreSQL implements 4 SQL standard isolation levels (default: `READ COMMITTED`):

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Serialization Anomaly |
|---|---|---|---|---|
| Read Uncommitted* | ✅ (not really, PG treats it as Read Committed) | ✅ | ✅ | ✅ |
| **Read Committed** (default) | ❌ | ✅ | ✅ | ✅ |
| **Repeatable Read** | ❌ | ❌ | ❌ (PG's snapshot isolation blocks phantoms too) | ✅ |
| **Serializable** | ❌ | ❌ | ❌ | ❌ |

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 'acc-1'; -- snapshot taken here
-- another transaction commits a change to acc-1 concurrently
SELECT balance FROM accounts WHERE id = 'acc-1'; -- same result as above, no dirty/non-repeatable read
COMMIT;
```

`SERIALIZABLE` in PostgreSQL is implemented via **Serializable Snapshot Isolation (SSI)** — it detects dangerous read/write dependency cycles between concurrent transactions and aborts one with a serialization error (`40001`), which the application must retry.

**Spring/JPA mapping:**

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferFunds(String fromId, String toId, BigDecimal amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();

    from.debit(amount);
    to.credit(amount);
    // JPA flushes on commit; a serialization failure throws
    // org.springframework.dao.CannotAcquireLockException -> retry with @Retryable
}
```

```java
@Retryable(
    retryFor = CannotAcquireLockException.class,
    maxAttempts = 3,
    backoff = @Backoff(delay = 50, multiplier = 2))
public void transferFundsWithRetry(String fromId, String toId, BigDecimal amount) {
    transferFunds(fromId, toId, amount);
}
```

---

## 4. Optimistic vs Pessimistic Locking

### Pessimistic locking — `SELECT ... FOR UPDATE`

Locks the row immediately; other transactions wanting the same lock **block** until release.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select a from Account a where a.id = :id")
Account findByIdForUpdate(@Param("id") String id);
```

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 'acc-1' FOR UPDATE; -- blocks other FOR UPDATE/writers on this row
UPDATE accounts SET balance = balance - 100 WHERE id = 'acc-1';
COMMIT;
```

Use when contention is **high** and retries would be wasteful (e.g. ledger row updated by many concurrent transfers).

### Optimistic locking — version column

No locks taken; conflict detected at commit time via a version check.

```java
@Entity
public class Account {

    @Id
    private String id;

    private BigDecimal balance;

    @Version
    private long version; // Hibernate auto-increments and checks this on UPDATE
}
```

```sql
UPDATE accounts SET balance = ?, version = version + 1 WHERE id = ? AND version = ?;
-- 0 rows updated => OptimisticLockException, caller retries
```

Use when contention is **low** and you want to avoid holding locks (better throughput, no blocking).

**Interview framing**: "Pessimistic trades throughput for certainty; optimistic trades a retry cost for throughput. For a wallet ledger with many concurrent small transfers on the *same* account, pessimistic (or a queue-based serialization) is usually safer; for independent per-user resources, optimistic locking scales better."

---

## 5. Deadlocks at the Database Level

Classic cause: two transactions lock rows in opposite order.

```
Tx1: locks acc-1, then wants acc-2
Tx2: locks acc-2, then wants acc-1  --> deadlock
```

PostgreSQL detects the cycle and aborts one transaction automatically (`deadlock_detected` error), but the *prevention* is on the application: **always acquire locks/update rows in a consistent order** (e.g., sort account IDs before locking).

```java
public void transfer(String accountA, String accountB, BigDecimal amount) {
    String first = accountA.compareTo(accountB) < 0 ? accountA : accountB;
    String second = accountA.compareTo(accountB) < 0 ? accountB : accountA;
    Account lockedFirst = accountRepository.findByIdForUpdate(first);
    Account lockedSecond = accountRepository.findByIdForUpdate(second);
    // proceed with debit/credit using consistent lock order
}
```

---

## 6. Connection Pooling: HikariCP

Spring Boot defaults to HikariCP. Key production tuning knobs:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20          # rule of thumb: (core_count * 2) + effective_spindle_count, tune with load tests
      minimum-idle: 5
      connection-timeout: 3000       # ms to wait for a connection before failing fast
      idle-timeout: 600000
      max-lifetime: 1800000          # recycle connections before PG/proxy kills them
      leak-detection-threshold: 5000 # logs a warning if a connection isn't returned in time
```

**Interview trap**: a bigger pool is not always faster — beyond the number of CPU cores/DB backend processes available, more connections just increase context switching and lock contention on Postgres. Pool size should be validated with load testing (JMH/k6 — see the application-tuning experience mentioned in your own background), not guessed.

---

## 7. Database Architecture & Schema Design

### Indexing

```sql
-- B-tree (default) for equality/range queries
CREATE INDEX idx_transactions_account_id ON transactions (account_id);

-- Partial index: smaller, faster, for a common filtered query
CREATE INDEX idx_transactions_pending ON transactions (account_id) WHERE status = 'PENDING';

-- Composite index: order matters, leftmost-prefix rule
CREATE INDEX idx_transactions_account_created ON transactions (account_id, created_at DESC);
```

### Normalization vs denormalization

- **Normalize** (3NF) for transactional/OLTP write paths (accounts, ledgers) — avoids update anomalies, keeps data consistent.
- **Denormalize** (or use materialized views / read replicas / CQRS read models) for reporting/analytics or high-read endpoints — trades storage/consistency for read latency.

### Partitioning (scale consideration for a fintech-scale table)

```sql
CREATE TABLE transactions (
    id UUID,
    account_id UUID,
    amount NUMERIC,
    created_at TIMESTAMPTZ
) PARTITION BY RANGE (created_at);

CREATE TABLE transactions_2026_01 PARTITION OF transactions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

Keeps indexes small, speeds up time-range queries, and makes retention/archival trivial (drop old partitions instead of `DELETE`).

### Replication & read scaling

- **Streaming replication** (primary → read replicas) for horizontal read scaling.
- Application must be aware of **replication lag** — a read-after-write on a replica can return stale data. Spring can route via `@Transactional(readOnly = true)` + a routing `DataSource` (`AbstractRoutingDataSource`) to send reads to replicas and writes to the primary.

```java
public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? "replica" : "primary";
    }
}
```

---

## Interview Q&A

**Q1: What's the difference between `READ COMMITTED` and `REPEATABLE READ` in Postgres?**
`READ COMMITTED` re-evaluates its snapshot on every statement within the transaction (so it can see other transactions' commits between statements — non-repeatable reads possible). `REPEATABLE READ` takes one snapshot at transaction start and uses it for every statement — the transaction always sees the same data even if others commit changes concurrently.

**Q2: Why does Postgres need `VACUUM`?**
Because MVCC never overwrites rows in place — `UPDATE`/`DELETE` create dead tuples. `VACUUM` reclaims that space and updates statistics for the query planner; `autovacuum` does this automatically but can fall behind under heavy write load, causing bloat.

**Q3: How do you prevent a lost update between two concurrent transactions modifying the same row?**
Either pessimistic lock (`SELECT FOR UPDATE`) to serialize access, or optimistic locking (`@Version`) to detect the conflict at commit and retry — the JMM/CAS parallel from application-level concurrency (compare-and-swap) maps directly to optimistic locking at the DB level.

**Q4: How would you design the accounts/ledger schema for a banking app?**
Append-only ledger table (immutable transaction log) + a materialized/derived `balance` on the account (or computed via `SUM` over the ledger for strong auditability). Favor immutability for the ledger (never `UPDATE`/`DELETE`, only `INSERT`) — mirrors the "immutability removes race conditions" principle from the application layer.

---

## Sources

- [PostgreSQL Docs: Concurrency Control (MVCC)](https://www.postgresql.org/docs/current/mvcc.html)
- [PostgreSQL Docs: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL Docs: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [Use The Index, Luke — SQL indexing guide](https://use-the-index-luke.com/)
- [HikariCP: About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [Baeldung: Optimistic vs Pessimistic Locking in JPA](https://www.baeldung.com/jpa-optimistic-locking)
- [Spring Docs: Dynamic DataSource Routing](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/lookup/AbstractRoutingDataSource.html)
