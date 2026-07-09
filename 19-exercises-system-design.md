# 19. Exercises & Leveled Q&A — System Design

Covers [12-system-design-for-interview.md](./12-system-design-for-interview.md).

---

## Hands-On Exercises

### Exercise 1 (Basic) — Apply the CAP theorem to two endpoints

For each endpoint below, state whether you'd design it as CP (consistency-favoring) or AP (availability-favoring), and why:

1. `POST /transfers` — initiates a money transfer, debiting one account and crediting another.
2. `GET /transactions/recent` — shows a user's last 20 transactions on their home screen.

<details>
<summary>Solution</summary>

1. **CP** — a money transfer must never produce an incorrect balance (e.g. double-spend, lost debit). If there's any doubt about consistency (e.g. a network partition between the app and the primary database), it's safer to reject/retry the request than to risk an inconsistent financial state.
2. **AP** — showing a slightly stale list of recent transactions (e.g. missing the last few seconds of activity, or served from a read replica/cache with some lag) is an acceptable trade-off for availability and low latency; refusing to show the screen at all because of a minor replication lag would be a poor user experience for something that isn't safety-critical.
</details>

---

### Exercise 2 (Mid) — Add resilience to a downstream FX rate call

The following code calls an external FX rate provider synchronously with no protection. Add a timeout, a circuit breaker, and a sensible fallback.

```java
@Service
public class FxService {
    private final FxProviderClient fxProviderClient;

    public ExchangeRate getRate(String currencyPair) {
        return fxProviderClient.fetchRate(currencyPair); // no timeout, no fallback
    }
}
```

<details>
<summary>Solution</summary>

```java
@Service
public class FxService {

    private final FxProviderClient fxProviderClient;
    private final ExchangeRateCache exchangeRateCache; // last-known-good rates

    @CircuitBreaker(name = "fxProvider", fallbackMethod = "fallbackRate")
    @TimeLimiter(name = "fxProvider")
    @Retry(name = "fxProvider")
    public CompletableFuture<ExchangeRate> getRate(String currencyPair) {
        return CompletableFuture.supplyAsync(() -> fxProviderClient.fetchRate(currencyPair));
    }

    private CompletableFuture<ExchangeRate> fallbackRate(String currencyPair, Throwable ex) {
        log.warn("FX provider unavailable for {}, falling back to cached rate", currencyPair, ex);
        return CompletableFuture.completedFuture(exchangeRateCache.getLastKnownRate(currencyPair));
    }
}
```

```yaml
resilience4j:
  timelimiter:
    instances:
      fxProvider:
        timeout-duration: 800ms
  circuitbreaker:
    instances:
      fxProvider:
        sliding-window-size: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
  retry:
    instances:
      fxProvider:
        max-attempts: 2
        wait-duration: 100ms
```

Key points to articulate: the timeout prevents one slow dependency from exhausting the caller's thread pool; the circuit breaker stops hammering a provider that's already failing (fail fast instead of queueing more doomed requests); the retry handles transient blips only (small `max-attempts`, short backoff — not a substitute for the circuit breaker); the fallback degrades gracefully (a slightly stale cached rate) instead of failing the whole transfer outright for a non-critical dependency.
</details>

---

### Exercise 3 (Senior) — Design the read path for "transaction history" at scale

The write path (money transfers) is already designed as CP with row-level locking on Postgres (see doc 09/16). Design the **read path** for a "transaction history" screen that must serve millions of users with sub-200ms p99 latency, without adding load to the primary write database.

<details>
<summary>Solution</summary>

**Design**:

1. **Don't query the primary write database directly for this read path.** The primary is reserved for the low-latency, consistency-critical write path (transfers).
2. **Build a denormalized read model** (CQRS-style) specifically shaped for the "transaction history" screen — e.g. a separate table/index (could be another Postgres schema, or a search-optimized store like Elasticsearch/OpenSearch if the screen supports rich filtering/search) with exactly the fields the UI needs, pre-joined and pre-formatted.
3. **Populate the read model asynchronously** by consuming the same domain/integration events already published for other purposes (fraud scoring, notifications — see doc 11/18) via a dedicated projector service. This means zero additional load on the write path — the read model catches up independently.
4. **Cache the hottest data** (e.g. the most recent page of transactions per user) in Redis with a short TTL or event-driven invalidation, in front of the read model, to shave off further latency for the common case (users mostly look at their most recent transactions).
5. **Accept eventual consistency** for this screen explicitly: a transaction that just completed may take up to a few hundred milliseconds to appear (event propagation + projection time) — state this trade-off proactively, and consider an optimistic UI update on the client immediately after a successful transfer response, backed by the eventually-consistent server-side history for subsequent loads/refreshes.

**Why this satisfies the latency and load requirements**: reads never touch the write-optimized primary (removing contention with the transfer write path entirely), the read model's schema is purpose-built for this one query pattern (no runtime joins), and the cache absorbs the highest-frequency access pattern (recent activity) with minimal database load at all.

**Follow-up the interviewer may ask**: "what if the projector falls behind during a traffic spike, and history looks stale for longer than usual?" Answer: monitor consumer lag (e.g. Kafka consumer group lag) as an explicit SLO/alert, scale the projector horizontally (partitioned by user ID to preserve per-user ordering, per doc 18 Senior Q3), and communicate staleness to the client if lag exceeds a threshold (e.g. a "data as of Xs ago" indicator) rather than silently serving increasingly stale data.
</details>

---

## Leveled Interview Q&A

### Basic Level

**Q1: What is the CAP theorem?**
In the presence of a network partition, a distributed system must choose between Consistency (all nodes see the same data) and Availability (every request gets a response) — it cannot guarantee both simultaneously.

**Q2: What's the difference between vertical and horizontal scaling?**
Vertical scaling means making a single machine more powerful (more CPU/RAM); horizontal scaling means adding more machines and distributing load across them. Horizontal scaling requires the application to be stateless (or externalize state) to work correctly behind a load balancer.

**Q3: Why is idempotency important for a payment API?**
Clients (or intermediate network components) may retry a request after a timeout without knowing whether the original request succeeded. Without idempotency, a retried "create transfer" request could create a duplicate transfer; an idempotency key lets the server recognize and safely no-op a retried request.

**Q4: What is a cache-aside pattern?**
The application checks the cache first; on a miss, it reads from the database, then populates the cache for subsequent reads. Writes typically invalidate (or update) the corresponding cache entry.

**Q5: What is a circuit breaker used for?**
Detecting that a downstream dependency is failing and "opening" the circuit to stop sending it requests for a period, failing fast instead — protecting both the caller (no wasted threads/time waiting on a doomed call) and the struggling dependency (no pile-up of retries making its recovery harder).

---

### Mid Level

**Q1: Why should a service behind a load balancer be stateless?**
Because any given request can be routed to any instance; if session/state lived only in one instance's memory, subsequent requests routed to a different instance wouldn't see it, breaking correctness. Externalize required state to a shared store (Redis, database) so any instance can serve any request.

**Q2: What's the difference between a timeout, a retry, and a circuit breaker, and why do you need all three together?**
A timeout bounds how long you wait for a single call. A retry re-attempts a failed/timed-out call, usually for transient errors, with a small number of attempts and backoff. A circuit breaker tracks the failure rate across many calls and stops attempting calls altogether (fail fast) once a dependency is clearly unhealthy — retries alone would keep hammering a truly-down dependency, making recovery harder; a circuit breaker prevents that pile-up while the timeout keeps individual calls from hanging indefinitely.

**Q3: When would you choose eventual consistency over strong consistency for a given feature?**
When the feature can tolerate briefly stale data in exchange for better availability/latency/scalability, and when the cost of occasional staleness is low compared to the cost of reduced availability — e.g. a "recent activity" feed vs. an account balance used to authorize a debit.

**Q4: How do read replicas help scale reads, and what do you need to be careful about?**
They offload read traffic from the primary, allowing horizontal read scaling without touching the write path. The caveat: replication lag means a read immediately following a write on a replica might not yet reflect that write (read-after-write consistency issue) — routing must be deliberate (e.g. reads that must be fresh go to the primary; tolerant reads go to replicas).

**Q5: What is backpressure, and why does an unbounded queue in front of a service hide rather than solve an overload problem?**
Backpressure is a mechanism for a system under load to signal upstream producers to slow down (reject, delay, or shed load) rather than accept unbounded work. An unbounded queue lets requests pile up invisibly — the system appears "fine" (still accepting requests) while latency silently grows unbounded, until it eventually runs out of memory or the queued work becomes so stale it's no longer useful — a bounded queue with an explicit rejection policy surfaces the overload immediately instead.

---

### Senior Level

**Q1: Walk through, end-to-end, how you'd prevent a double-spend in a real-time transfer system serving millions of concurrent users, touching on both the application and database layers.**
Layered defenses: (1) an idempotency key at the API boundary deduped both in-process (short-lived cache) and durably (unique DB constraint) so client retries never create a second transfer; (2) row-level locking or serializable isolation on the account balance mutation at the database layer (consistent lock ordering to avoid deadlocks) so concurrent transfers on the same account can't both read-then-write an inconsistent balance; (3) an append-only ledger as the audit-grade source of truth, with the mutable `balance` column treated as a derived/cached value that can always be reconciled against the ledger; (4) monitoring/alerting on any detected serialization failures or reconciliation mismatches as an early warning signal, not just a "retry and move on" situation.

**Q2: How would you evolve a monolithic transfer service into event-driven microservices without a big-bang rewrite?**
Apply the Strangler Fig pattern: keep the monolith as the system of record initially, but start publishing domain events for the actions you want to decouple (e.g. `TransferCompleted`) via the transactional outbox pattern, and stand up new services (fraud scoring, notifications) as pure consumers of those events first — no behavior is removed from the monolith yet, just observed externally. Once a capability (e.g. fraud scoring) is fully replicated and validated in the new service, cut the monolith over to call/depend on the new service (or retire the old in-process logic), one bounded context at a time, rather than attempting a full rewrite.

**Q3: What are the trade-offs of choosing Kafka vs. a simpler solution (e.g. Postgres LISTEN/NOTIFY, or a scheduled polling table) for propagating transfer-completed events?**
Kafka provides durability (events aren't lost even if consumers are down when published), replay capability (new consumers can be added and read historical events), partitioned ordering guarantees, and horizontal consumer scaling — at the cost of operational complexity (a cluster to run/monitor, schema evolution discipline, at-least-once semantics requiring idempotent consumers). `LISTEN/NOTIFY` is much simpler but not durable (a notification is lost if no listener is connected at the moment it fires) and doesn't scale well beyond a single/small number of consumers. A polling table (essentially the outbox pattern without a broker) is simple and durable but adds latency and doesn't give you fan-out to many independent consumer types as cleanly as a pub/sub topic. The right choice depends on required durability, number/diversity of consumers, and acceptable operational overhead — for a fintech-scale system with multiple independent downstream consumers (fraud, notifications, read models), Kafka's trade-offs are usually justified.

**Q4: How would you design the system so it degrades gracefully rather than failing completely when the Fraud Detection Service (an async downstream consumer) is completely unavailable for an extended period?**
Since fraud scoring is decoupled via the outbox/Kafka path (not synchronous with the transfer itself), a fraud service outage doesn't block transfers from completing at all — events simply queue up in Kafka (bounded by topic retention) until the service recovers and catches up (consumer lag). The business decision to make explicit with stakeholders: is it acceptable for transfers to complete without a *timely* fraud check during an extended outage (accepting some risk), or should there be a compensating control (e.g. a temporary lower transfer limit, or a manual review queue) activated when fraud-service lag/outage crosses a threshold? This is a genuine business trade-off to surface, not purely a technical one — a strong senior answer explicitly separates the technical resilience (queueing, no blocking) from the business risk decision (what to do about reduced fraud coverage during the outage).
