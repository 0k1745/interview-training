# 12. System Design for the Interview

**Context**: The recruiter's message lists "system design" alongside DDD/EDD. This doc gives a repeatable framework for answering system design questions, plus a worked fintech example (money transfer / fraud detection) that ties together concurrency, DDD, and EDD from the other docs in this repo.

---

## 1. A Repeatable Framework (use this structure out loud in the interview)

1. **Clarify requirements** — functional (what must it do) and non-functional (scale, latency, consistency needs).
2. **Estimate scale** — back-of-envelope numbers (requests/sec, data volume, read/write ratio).
3. **High-level design** — draw boxes: clients, API layer, services, datastore(s), broker, cache.
4. **Deep dive on the hard part** — the interviewer usually wants depth on ONE component, not a shallow pass over everything.
5. **Address bottlenecks & trade-offs** — explicitly name what you're sacrificing (consistency, latency, cost) and why.
6. **Failure modes** — what happens when a dependency is down/slow; retries, timeouts, circuit breakers, idempotency.

---

## 2. Foundational Concepts to Have Crisp

### CAP Theorem
In a network partition, you must choose **Consistency** or **Availability** — you cannot have both. Practically:
- A ledger/balance write path leans **CP** (reject the transfer rather than risk an inconsistent balance).
- A "recent transactions" read view or a notification feed can lean **AP** (show slightly stale data rather than failing).

### Scaling axes
- **Vertical** (bigger machine) — simple, has a ceiling.
- **Horizontal** (more machines) — requires statelessness, load balancing, and a strategy for shared state (sessions, cache, DB).

### Caching
- **Cache-aside** (application checks cache, falls back to DB, populates cache) — most common with Redis.
- Invalidate on write, or use short TTLs for eventually-consistent data (e.g. FX rates refreshed every few seconds).

```java
@Cacheable(value = "fxRates", key = "#currencyPair")
public ExchangeRate getRate(String currencyPair) {
    return fxProviderClient.fetchRate(currencyPair); // only called on cache miss
}

@CacheEvict(value = "fxRates", key = "#currencyPair")
public void invalidateRate(String currencyPair) { }
```

### Load balancing & statelessness
Spring Boot services behind a load balancer (e.g. GCP Load Balancer / Kubernetes Service) must be stateless — no in-memory session state (use a shared store — Redis — for anything that must survive across instances, including the `ThreadLocal`/idempotency caches from doc 08, which only work *per instance* and don't replace a distributed dedupe store).

### Idempotency at the API boundary
Any retryable operation (payment, transfer) needs a client-supplied idempotency key stored server-side, so retries (from a flaky client or a load balancer failover) don't double-process:

```java
@PostMapping("/transfers")
public ResponseEntity<TransferResponse> createTransfer(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody TransferRequest request) {
    return ResponseEntity.ok(transferService.processIdempotent(idempotencyKey, request));
}
```

### Rate limiting & backpressure
Token bucket / leaky bucket at the gateway (e.g. Redis-backed rate limiter, or resilience4j `RateLimiter`) to protect downstream services and enforce fair use per client.

### Circuit breakers & timeouts (resilience4j + Spring)

```java
@CircuitBreaker(name = "fxProvider", fallbackMethod = "fallbackRate")
@TimeLimiter(name = "fxProvider")
@Retry(name = "fxProvider")
public CompletableFuture<ExchangeRate> getRate(String pair) {
    return CompletableFuture.supplyAsync(() -> fxProviderClient.fetchRate(pair));
}

private CompletableFuture<ExchangeRate> fallbackRate(String pair, Throwable ex) {
    return CompletableFuture.completedFuture(ExchangeRate.cached(pair));
}
```

---

## 3. Worked Example: Design a Real-Time Money Transfer System

**Functional requirements**: users transfer money between accounts (possibly cross-currency), see near-instant confirmation, get fraud-checked, and see updated balances.

**Non-functional requirements**: strong consistency on balance (no double-spend), high availability for read paths, sub-second p99 latency for the synchronous part, horizontal scalability, full auditability.

### High-level architecture

```
Client
  │
  ▼
API Gateway (rate limiting, auth, idempotency-key dedupe)
  │
  ▼
Transfer Service (Spring Boot)
  │  - Validates request (domain layer, DDD aggregate 'Transfer')
  │  - Pessimistic/optimistic lock on account rows (Postgres, doc 09)
  │  - Writes Transfer + Outbox event in ONE transaction
  ▼
Postgres (primary) ── streaming replication ──▶ Read Replicas (balance queries, statements)
  │
  ▼ (outbox poller / Debezium CDC)
Kafka topic: transfers.completed
  │
  ├──▶ Fraud Detection Service (async, doc 08 CompletableFuture fan-out for scoring)
  ├──▶ Notification Service
  └──▶ Read-model projector (CQRS-style denormalized "transaction history" view in a
       read-optimized store, e.g. another Postgres schema or Elasticsearch)
```

### Why each piece is there
- **API Gateway idempotency dedupe**: protects against client retries creating duplicate transfers (ties to doc 08's `ConcurrentHashMap.computeIfAbsent` in-process dedupe, and doc 11's consumer-side dedupe for the async fan-out).
- **Pessimistic lock on account rows** (doc 09): the balance mutation is the one place strict consistency (CP) is non-negotiable — chosen over optimistic locking because concurrent transfers on the same popular account (e.g. a merchant) would otherwise retry excessively.
- **Outbox + Kafka** (doc 11): decouples the synchronous transfer confirmation from slower downstream work (fraud scoring, notifications) without risking a dual-write inconsistency.
- **Read replicas / CQRS read model**: the "transaction history" screen is read-heavy and can tolerate a few hundred ms of staleness — classic case for trading consistency for availability/latency (AP-leaning), while the ledger itself stays CP.
- **Circuit breaker on the FX provider call**: an external dependency failing must not take down the whole transfer path — fail fast, fall back to a cached rate, or degrade gracefully (reject only cross-currency transfers, not same-currency ones).

### Back-of-envelope sizing (to demonstrate the estimation step)

- 10M active users, 2 transfers/day average → ~230 transfers/sec average, design for ~10x peak (~2,300/sec) at payday/events.
- Each transfer record ~1 KB → ~20 GB/month of new ledger data → partition the `transactions` table monthly (doc 09) and archive old partitions.

---

## Interview Q&A

**Q1: How do you prevent double-spending in a highly concurrent transfer system?**
Combine: DB-level row locking (pessimistic lock or serializable isolation) on the account balance, an idempotency key at the API boundary to dedupe retries, and treating the ledger as append-only so balance is always derivable/auditable — never a floating mutable number trusted blindly.

**Q2: What would you do if the Fraud Detection Service goes down?**
Depends on the business risk tolerance: either (a) queue the fraud check async (via the outbox/Kafka path) and let the transfer proceed provisionally with a hold/reversal capability, or (b) block completion until fraud check succeeds, with a circuit breaker + fallback (e.g. route to manual review) if the service is degraded — this is a trade-off to discuss explicitly, not a single right answer.

**Q3: How would you scale the read path for transaction history without touching the write path?**
CQRS-style read model: a separate denormalized store (or read replica) updated asynchronously from the same domain/integration events already used for fraud/notifications — no additional load on the primary write path.

**Q4: What single component would you deep-dive on if the interviewer only gives you 10 minutes?**
Ask them — but the safest default for a fintech interview is the consistency/locking strategy at the database layer (doc 09) since it's the one place where getting it wrong causes real financial loss, and it demonstrates the "concurrency at DB level" experience they explicitly asked about.

---

## Sources

- [Martin Kleppmann, *Designing Data-Intensive Applications*](https://dataintensive.net/) (the definitive reference for these trade-offs)
- [AWS: CAP Theorem explained](https://aws.amazon.com/builders-library/cap-theorem-and-distributed-systems/)
- [Resilience4j documentation](https://resilience4j.readme.io/docs)
- [Google SRE Book: Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Stripe: Idempotent Requests](https://stripe.com/blog/idempotency) (a real-world payments API doing exactly this)
- [System Design Primer (GitHub)](https://github.com/donnemartin/system-design-primer)
