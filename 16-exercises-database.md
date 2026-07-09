# 16. Exercises & Leveled Q&A — Database Concurrency & Architecture (PostgreSQL)

Covers [09-database-concurrency-and-architecture.md](./09-database-concurrency-and-architecture.md).

---

## Hands-On Exercises

### Exercise 1 (Basic) — Choose the right isolation level

For each scenario, state the *minimum* PostgreSQL isolation level you'd choose and why:

1. Generating a daily report by running several `SELECT` queries that must all see a consistent snapshot of the data, even if writes happen concurrently.
2. A simple single-row balance update: `UPDATE accounts SET balance = balance - 100 WHERE id = 'acc-1'`.
3. A "check balance, then debit if sufficient" operation implemented as two separate statements in the application (not a single atomic `UPDATE ... WHERE balance >= amount`).

<details>
<summary>Solution</summary>

1. **`REPEATABLE READ`** — you need every `SELECT` in the transaction to see the same snapshot taken at transaction start, even as other transactions commit changes concurrently. `READ COMMITTED` (the default) would let each statement see a fresh snapshot, potentially producing an inconsistent report across queries.
2. **`READ COMMITTED`** (the default) is sufficient — a single atomic `UPDATE` statement is inherently safe under MVCC; Postgres handles the row-level write correctly regardless of isolation level for a single-statement compound update.
3. **`SERIALIZABLE`** (or rewrite as a single atomic statement / use `SELECT ... FOR UPDATE`) — the "check-then-act" pattern across two statements is vulnerable to a race where two concurrent transactions both read a sufficient balance before either debits, resulting in an overdraft. `SERIALIZABLE` detects the dangerous read/write dependency and aborts one transaction with a serialization error the app must retry. The better fix is avoiding the check-then-act pattern entirely: `UPDATE accounts SET balance = balance - 100 WHERE id = 'acc-1' AND balance >= 100` and checking the affected row count.
</details>

---

### Exercise 2 (Mid) — Optimistic locking with Spring Data JPA

Implement optimistic locking on this entity so that concurrent updates to the same `Account` are detected and rejected instead of silently overwriting each other (a "lost update"). Also show the calling code that retries on conflict.

```java
@Entity
public class Account {
    @Id
    private String id;
    private BigDecimal balance;
    // getters/setters omitted
}
```

<details>
<summary>Solution</summary>

```java
@Entity
public class Account {

    @Id
    private String id;

    private BigDecimal balance;

    @Version
    private long version; // Hibernate checks this on UPDATE and increments it automatically
}
```

```sql
-- Hibernate generates roughly:
UPDATE account SET balance = ?, version = version + 1 WHERE id = ? AND version = ?;
-- 0 rows affected => Hibernate throws OptimisticLockException
```

```java
@Service
public class AccountService {

    private final AccountRepository accountRepository;

    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 50, multiplier = 2))
    @Transactional
    public void debit(String accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId).orElseThrow();
        if (account.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException(accountId);
        }
        account.setBalance(account.getBalance().subtract(amount));
        accountRepository.save(account); // conflict detected here at flush/commit time
    }
}
```

Key point to articulate: optimistic locking takes **no locks up front** — it detects the conflict only at write time via the version check, so it's cheap under low contention but requires the caller to handle retries (unlike pessimistic locking, which blocks instead of failing).
</details>

---

### Exercise 3 (Senior) — Diagnose and fix a deadlock

Two services independently implement money transfers between two accounts, locking rows with `SELECT ... FOR UPDATE` in the order the accounts appear in the request:

```java
public void transfer(String fromAccountId, String toAccountId, BigDecimal amount) {
    Account from = accountRepository.findByIdForUpdate(fromAccountId);
    Account to = accountRepository.findByIdForUpdate(toAccountId);
    from.debit(amount);
    to.credit(amount);
}
```

Under load, Postgres logs start showing `deadlock detected` errors. Explain the root cause and fix the code.

<details>
<summary>Solution</summary>

**Root cause**: if Transaction A calls `transfer("acc-1", "acc-2", ...)` while Transaction B concurrently calls `transfer("acc-2", "acc-1", ...)`, A locks `acc-1` then wants `acc-2`, while B locks `acc-2` then wants `acc-1` — a classic lock-ordering cycle. Postgres detects the cycle and aborts one transaction with a `deadlock_detected` error; the other proceeds.

**Fix — always acquire locks in a globally consistent order**, independent of the caller-supplied argument order:

```java
public void transfer(String fromAccountId, String toAccountId, BigDecimal amount) {
    String first = fromAccountId.compareTo(toAccountId) < 0 ? fromAccountId : toAccountId;
    String second = fromAccountId.compareTo(toAccountId) < 0 ? toAccountId : fromAccountId;

    // Lock in a consistent order regardless of which account is "from" or "to"
    Account lockedFirst = accountRepository.findByIdForUpdate(first);
    Account lockedSecond = accountRepository.findByIdForUpdate(second);

    Account from = lockedFirst.getId().equals(fromAccountId) ? lockedFirst : lockedSecond;
    Account to = lockedFirst.getId().equals(fromAccountId) ? lockedSecond : lockedFirst;

    from.debit(amount);
    to.credit(amount);
}
```

Additional production hardening worth mentioning: wrap the whole operation with `@Retryable` on `CannotAcquireLockException`/`DeadlockLoserDataAccessException` as a defense-in-depth measure, since even with consistent lock ordering, deadlocks can still occur with more complex multi-table locking scenarios.
</details>

---

## Leveled Interview Q&A

### Basic Level

**Q1: What does ACID stand for?**
Atomicity (all-or-nothing), Consistency (valid state to valid state), Isolation (concurrent transactions don't interfere), Durability (committed data survives a crash).

**Q2: What connection pool does Spring Boot use by default?**
HikariCP.

**Q3: What is a database index used for?**
Speeding up lookups (equality/range queries) by avoiding a full table scan, at the cost of extra storage and slower writes (the index must be maintained on every insert/update/delete).

**Q4: What's the difference between `@Transactional(readOnly = true)` and a regular `@Transactional`?**
`readOnly = true` is a hint that lets the persistence provider/driver optimize (e.g. skip dirty checking, potentially route to a read replica) since no writes are expected; it doesn't strictly enforce read-only behavior at the SQL level in all setups, but should never be used on a method that writes.

**Q5: What's the default PostgreSQL transaction isolation level?**
`READ COMMITTED`.

---

### Mid Level

**Q1: Explain MVCC in one or two sentences.**
Instead of locking rows for reads, PostgreSQL keeps multiple versions of each row tagged with transaction IDs, so each transaction sees a consistent snapshot without blocking concurrent readers or writers — readers never block writers and vice versa.

**Q2: Why does Postgres need `VACUUM`, and what happens if autovacuum falls behind?**
Because MVCC never updates rows in place — updates/deletes create "dead tuples" (old row versions) that must be reclaimed. If autovacuum falls behind under heavy write load, tables bloat (wasted disk, slower scans) and in extreme cases the database risks transaction ID wraparound issues.

**Q3: When would you choose pessimistic locking over optimistic locking?**
When contention on the same rows is expected to be high and retries would be wasteful/costly (e.g. a popular merchant account receiving many concurrent debits) — pessimistic locking serializes access up front rather than letting transactions race and retry.

**Q4: How do you decide HikariCP's `maximum-pool-size`?**
Not by guessing a large number "to be safe" — size based on the number of concurrent DB operations you actually expect, validated by load testing, considering the database's own connection limits and the fact that more connections increase context switching and contention on Postgres beyond a certain point. A common starting formula is roughly `(core_count * 2) + effective_spindle_count`, but the real answer is always "measure it."

**Q5: What's the difference between a composite index on `(account_id, created_at)` and two separate single-column indexes?**
A composite index supports efficient queries that filter/sort on the leftmost prefix (`account_id` alone, or `account_id` + `created_at` together) in a single index scan. Two separate single-column indexes would each be used independently and Postgres would need to combine (bitmap AND) them, generally less efficient than one well-chosen composite index for a known query pattern.

---

### Senior Level

**Q1: Explain Serializable Snapshot Isolation (SSI) in PostgreSQL — how does it differ from traditional two-phase locking `SERIALIZABLE` implementations?**
Traditional `SERIALIZABLE` (as in some other databases) uses pervasive locking (or predicate locks) to physically prevent conflicting concurrent access. PostgreSQL's SSI instead lets transactions proceed optimistically under snapshot isolation, but tracks read/write dependencies between concurrent transactions; if it detects a "dangerous structure" (a cycle of rw-dependencies that could produce a non-serializable outcome), it aborts one of the transactions with a serialization failure (SQLSTATE `40001`) that the application must retry. This gives full serializability guarantees with less blocking than lock-based approaches, at the cost of needing application-level retry logic.

**Q2: How would you design a ledger table to guarantee auditability under high concurrent write load?**
Make it append-only (`INSERT` only, never `UPDATE`/`DELETE`) so it can never lose history and there's no update-related lock contention on old rows; derive the current balance either as a `SUM()` over the ledger (strong auditability, higher read cost) or maintain a separate `accounts.balance` column updated transactionally alongside each ledger insert (faster reads, requires the transaction to keep both in sync) — the trade-off should be stated explicitly, and mirrors the immutability-removes-races principle from the application layer (see [08-thread-safety-and-concurrency-patterns.md](./08-thread-safety-and-concurrency-patterns.md)).

**Q3: You need to scale read throughput for a "transaction history" endpoint without affecting the write path. What are your options, and what do you give up with each?**
(a) Read replicas via streaming replication — simple, but reads can see replication lag (stale data), and the app must explicitly route read-only queries there (e.g. `AbstractRoutingDataSource`). (b) A denormalized read model updated asynchronously from domain/integration events (CQRS-style) — decouples read scaling entirely from the write schema, but adds eventual consistency and infrastructure complexity. (c) Caching (Redis) hot query results — fastest, but requires careful invalidation and is only good for cacheable access patterns (not arbitrary ad-hoc queries). All three trade some consistency/freshness for read scalability — the right choice depends on how stale a "transaction history" view can tolerably be.

**Q4: How would you detect and prevent table bloat/transaction ID wraparound risk in a high-write-throughput Postgres table before it becomes an incident?**
Monitor `pg_stat_user_tables` for dead tuple counts and `age(relfrozenxid)` for wraparound risk; alert well before the `autovacuum_freeze_max_age` threshold. Tune `autovacuum` more aggressively for hot tables (lower `autovacuum_vacuum_scale_factor`/`autovacuum_vacuum_cost_delay`) rather than relying on defaults tuned for average workloads, and consider partitioning very high-churn tables (e.g. by date) so autovacuum work and index maintenance stay proportional to recent partitions rather than the whole table.

**Q5: In a distributed deployment with an outbox pattern (see [11-event-driven-design.md](./11-event-driven-design.md)) publishing events from a Postgres table, how do you ensure the poller doesn't publish the same event twice or miss one under concurrent poller instances?**
Use `SELECT ... FOR UPDATE SKIP LOCKED` when multiple poller instances compete for unpublished outbox rows — `SKIP LOCKED` lets each poller instance grab a different set of rows without blocking on rows already claimed by another instance, avoiding both double-processing and poller-vs-poller deadlock/blocking. Mark rows published (or delete them) within the same transaction as the actual publish acknowledgment where possible, and make the downstream consumer idempotent regardless (at-least-once delivery is still the realistic guarantee even with `SKIP LOCKED`).
