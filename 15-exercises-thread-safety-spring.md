# 15. Exercises & Leveled Q&A — Thread Safety & Concurrency Patterns (Java/Spring)

Covers [08-thread-safety-and-concurrency-patterns.md](./08-thread-safety-and-concurrency-patterns.md).

---

## Hands-On Exercises

### Exercise 1 (Basic) — Fix the leaking `ThreadLocal`

This filter sets a correlation ID for logging but leaks state across requests when deployed behind a thread pool. Find the bug and fix it.

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final ThreadLocal<String> CORRELATION_ID = new ThreadLocal<>();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {
        CORRELATION_ID.set(UUID.randomUUID().toString());
        chain.doFilter(request, response);
    }
}
```

<details>
<summary>Solution</summary>

The `ThreadLocal` value is never removed. Since Tomcat (and most servlet containers) reuse a pool of worker threads across requests, the correlation ID set for request A stays attached to that thread and leaks into request B's logs if B happens to be handled by the same thread — plus it's a slow, permanent memory leak if the value is ever large.

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final ThreadLocal<String> CORRELATION_ID = new ThreadLocal<>();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {
        CORRELATION_ID.set(UUID.randomUUID().toString());
        try {
            chain.doFilter(request, response);
        } finally {
            CORRELATION_ID.remove(); // always clear in a finally block
        }
    }
}
```
</details>

---

### Exercise 2 (Mid) — Fan-out three independent service calls in parallel

Write a method that calls three independent, blocking service methods (`fraudDetectionService.score(tx)`, `fxService.getRate(currency)`, `accountService.getBalance(accountId)`) **in parallel**, combines their results, and handles the case where any of them fails without blowing up the whole request.

<details>
<summary>Solution</summary>

```java
@Service
public class TransferOrchestrator {

    private final FraudDetectionService fraudDetectionService;
    private final FxService fxService;
    private final AccountService accountService;
    private final Executor executor; // injected bean, e.g. a bounded ThreadPoolTaskExecutor

    public TransferOrchestrator(FraudDetectionService fraudDetectionService,
                                 FxService fxService,
                                 AccountService accountService,
                                 @Qualifier("fraudCheckExecutor") Executor executor) {
        this.fraudDetectionService = fraudDetectionService;
        this.fxService = fxService;
        this.accountService = accountService;
        this.executor = executor;
    }

    public TransferResult processTransfer(TransferRequest request) {
        CompletableFuture<FraudScore> fraudCheck = CompletableFuture
                .supplyAsync(() -> fraudDetectionService.score(request.toTransaction()), executor)
                .exceptionally(ex -> FraudScore.unavailable());

        CompletableFuture<ExchangeRate> rate = CompletableFuture
                .supplyAsync(() -> fxService.getRate(request.currency()), executor)
                .exceptionally(ex -> ExchangeRate.cachedFallback(request.currency()));

        CompletableFuture<AccountBalance> balance = CompletableFuture
                .supplyAsync(() -> accountService.getBalance(request.fromAccountId()), executor);
        // no fallback here: balance is on the critical path, we WANT it to fail loudly

        return CompletableFuture.allOf(fraudCheck, rate, balance)
                .thenApply(v -> new TransferResult(fraudCheck.join(), rate.join(), balance.join()))
                .join(); // join only at this top boundary, not inside nested service calls
    }
}
```

Key points to call out: each call runs on a bounded, named executor (never the common `ForkJoinPool` for I/O-bound work); non-critical calls (`fraudCheck`, `rate`) degrade gracefully via `.exceptionally(...)`, while the critical one (`balance`) is allowed to propagate its exception; `join()` is only called once, at the orchestration boundary.
</details>

---

### Exercise 3 (Senior) — Design an idempotent, concurrency-safe request deduplicator

Design (and implement) a component that, given a client-supplied idempotency key, ensures that if the *same* key is submitted concurrently (e.g. a retried HTTP request racing the original), only one underlying operation actually executes, and all callers get the same result.

<details>
<summary>Solution</summary>

```java
@Component
public class IdempotentOperationExecutor {

    private final ConcurrentHashMap<String, CompletableFuture<TransferResult>> inFlight =
            new ConcurrentHashMap<>();

    public CompletableFuture<TransferResult> execute(String idempotencyKey,
                                                       Supplier<TransferResult> operation,
                                                       Executor executor) {
        return inFlight.computeIfAbsent(idempotencyKey, key ->
                CompletableFuture.supplyAsync(operation, executor)
                        .whenComplete((result, ex) -> inFlight.remove(key))
        );
    }
}
```

Why this is correct under concurrency:
- `ConcurrentHashMap.computeIfAbsent` guarantees the mapping function runs **at most once per key**, even under concurrent calls with the same key — this is the atomic "check-then-act" that a plain `HashMap` + manual check could never give you (a classic TOCTOU race).
- All concurrent callers with the same key get the *same* `CompletableFuture` instance, so they all observe the same eventual result (success or failure) without duplicating the underlying side effect.
- The entry is removed once complete (`whenComplete`) so the map doesn't grow unbounded and a *new* request with the same key (e.g. a genuinely new transfer reusing an old key after TTL expiry) can proceed.

**Follow-up the interviewer will likely ask**: "this only works within one JVM instance — what if you have multiple instances behind a load balancer?" Answer: this in-process cache only dedupes concurrent requests landing on the *same* instance. For cross-instance idempotency (the realistic production requirement), you need a shared store — e.g. a unique constraint on `idempotency_key` in Postgres (insert-or-return-existing pattern) or a distributed lock/cache (Redis `SETNX`) — this is the system-design-level generalization of the same problem, see [12-system-design-for-interview.md](./12-system-design-for-interview.md).
</details>

---

## Leveled Interview Q&A

### Basic Level

**Q1: What are the three main strategies for making code thread-safe?**
Confinement (never share the mutable state, e.g. `ThreadLocal`), immutability (no mutable state to race on), and synchronization (guard shared mutable state with locks/atomics).

**Q2: What is `ThreadLocal` used for?**
Giving each thread its own independent copy of a variable, avoiding the need for locking. Common uses: correlation IDs for logging (MDC), per-thread `SimpleDateFormat` instances (pre-`java.time`), Spring's transaction/security context propagation.

**Q3: Why are immutable objects automatically thread-safe?**
Because there's no mutable state for concurrent threads to race on — once constructed, an immutable object's state never changes, so it can be freely shared/read by any number of threads with no synchronization needed.

**Q4: What does `@Async` do in Spring?**
Marks a method to run asynchronously on a separate thread (from a configured `Executor`), returning immediately (often wrapped in a `Future`/`CompletableFuture`) instead of blocking the caller's thread.

**Q5: What's a common mistake when using `@Async` in Spring?**
Calling the `@Async` method from within the same class — Spring's proxy-based AOP doesn't intercept self-invocation, so the call runs synchronously on the caller's thread instead of asynchronously.

---

### Mid Level

**Q1: Why should Spring `@Service` singleton beans avoid mutable instance state?**
Singleton beans are shared across all requests/threads by default. Any mutable instance field becomes shared mutable state across concurrent HTTP requests, requiring synchronization; stateless services (only `final`, injected, ideally immutable collaborators) are thread-safe "for free" without any explicit locking.

**Q2: Why is `ConcurrentHashMap` preferred over `Collections.synchronizedMap(new HashMap<>())`?**
`ConcurrentHashMap` uses fine-grained internal locking (historically segment-based, now more granular per-bucket), so different threads can operate on different parts of the map concurrently with minimal contention. `synchronizedMap` wraps the whole map with a single coarse lock — every operation, even on unrelated keys, contends for the same lock.

**Q3: How would you size a `ThreadPoolTaskExecutor` for a Spring service making downstream HTTP calls?**
Base it on the workload type (I/O-bound calls tolerate a larger pool than CPU-bound work), and validate empirically with load testing (JMH/k6/Datadog) rather than guessing — oversizing risks overwhelming downstream services and connection pools, undersizing causes queueing/latency. Always define a bounded queue and an explicit `RejectedExecutionHandler` (e.g. `CallerRunsPolicy`) rather than an unbounded queue that can hide backpressure problems until an OOM.

**Q4: What's the danger of an unbounded thread pool under a traffic spike?**
Unbounded thread creation can exhaust memory/OS thread limits and cause context-switch thrashing, and can overwhelm downstream dependencies (DB connections, external APIs) faster than they can handle — turning a transient spike into a cascading outage.

**Q5: When would you choose `Semaphore` over a thread pool's built-in bounding?**
When you need to limit concurrency to a *specific resource* (e.g. max 20 concurrent calls to a rate-limited external FX provider) independently of how many threads are actually available/used for other work — the semaphore bounds concurrent access to that resource specifically, not the whole executor.

---

### Senior Level

**Q1: You need to run a callback only after a transaction commits successfully (e.g. sending a notification after a transfer completes) — what's the risk of using a plain `@EventListener`, and how do you fix it?**
A plain `@EventListener` fires synchronously as soon as the event is published, which may be *before* the surrounding `@Transactional` method actually commits (or even if it later rolls back) — leading to a notification sent for a transfer that never actually persisted. Fix: use `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)` so the listener only runs after a successful commit.

**Q2: How would you design a rate limiter for outbound calls to a flaky third-party API, and how does it interact with your thread pool sizing?**
Use a token-bucket/leaky-bucket limiter (e.g. resilience4j `RateLimiter`, or a `Semaphore` for a simple fixed-concurrency cap) sized to the third party's documented limits, combined with a circuit breaker (fail fast + fallback when the provider is degraded) and a timeout on each call. This should be sized independently from (but coordinated with) your executor pool size — a large executor pool with no rate limiter just means you can generate more concurrent failing/queued calls faster, making an incident worse, not better.

**Q3: In a distributed (multi-instance) deployment, why doesn't an in-process `ConcurrentHashMap`-based dedupe cache fully solve request idempotency?**
Because each instance has its own JVM heap — the map only dedupes concurrent requests landing on the *same* instance behind the load balancer. A retried request routed to a different instance would not see the in-flight entry and could execute the operation twice. True idempotency at scale requires a shared, cross-instance store (a unique DB constraint on the idempotency key with an insert-or-fetch-existing pattern, or a distributed cache/lock like Redis).

**Q4: How do you decide between `synchronized`/`ReentrantLock` and a lock-free (CAS-based) approach for a hot code path?**
Profile first — lock-free approaches (atomics, `ConcurrentHashMap.compute*`) generally win under low-to-moderate contention and avoid thread suspension/context-switch overhead, but under very high contention CAS retry loops can burn CPU spinning. Locks provide clearer semantics for protecting multi-step invariants across several fields. In practice: default to higher-level concurrent utilities (`java.util.concurrent`), only reach for a manual `synchronized`/`ReentrantLock` when the invariant genuinely spans multiple related fields that must change together atomically, and only micro-optimize toward raw CAS after profiling shows lock contention is an actual bottleneck.

**Q5: What's the interaction between virtual threads (Java 21, Project Loom) and the executor-sizing advice above — does it change anything?**
Virtual threads make blocking I/O calls cheap (blocking a virtual thread doesn't block an OS thread), which removes much of the motivation for tuning a small, carefully-bounded platform-thread pool purely to avoid OS thread exhaustion for I/O-bound work — `Executors.newVirtualThreadPerTaskExecutor()` can spawn very large numbers of virtual threads cheaply. However, downstream backpressure concerns don't disappear: you still need to bound *concurrency toward a given downstream dependency* (via a `Semaphore` or a rate limiter), because the bottleneck shifts from "too many OS threads" to "too many concurrent requests hitting a downstream service/connection pool that has its own real limits" (e.g. HikariCP's max pool size, see [09-database-concurrency-and-architecture.md](./09-database-concurrency-and-architecture.md)).
