# 8. Thread Safety & Concurrency Patterns (Application Level)

**Context**: The technical screen explicitly calls out "concurrency & multithreading on both database and application level". This guide covers the **application level** half with Java/Spring examples. Database-level concurrency is covered in [09-database-concurrency-and-architecture.md](./09-database-concurrency-and-architecture.md).

This builds on the JMM/synchronization fundamentals already covered in this repo:
- [01-process-vs-thread.md](./01-process-vs-thread.md)
- [03-java-memory-model.md](./03-java-memory-model.md)
- [04-synchronization-mechanisms.md](./04-synchronization-mechanisms.md)
- [05-atomic-and-lock-free.md](./05-atomic-and-lock-free.md)
- [06-locks-and-conditions.md](./06-locks-and-conditions.md)

---

## What "Thread Safety" Actually Means

A class is thread-safe if it behaves correctly (per its specification) when accessed from multiple threads, regardless of the scheduling or interleaving of execution, and without extra synchronization from the caller.

Three complementary ways to achieve it:

| Strategy | How | Example |
|---|---|---|
| **Confinement** | Never share the mutable state across threads | `ThreadLocal`, stack-local variables |
| **Immutability** | Make the state impossible to change after construction | `record`, `final` fields, defensive copies |
| **Synchronization** | Guard shared mutable state with locks/atomics | `synchronized`, `ReentrantLock`, `AtomicInteger` |

Senior-level framing: **the goal is not "add synchronized everywhere"**, it's identifying which of these three strategies removes the need for locking altogether, and only synchronizing what's left.

---

## 1. Confinement: `ThreadLocal`

Used to give each thread its own copy of a variable — no locking overhead.

```java
public class RequestContextHolder {

    private static final ThreadLocal<String> CORRELATION_ID = new ThreadLocal<>();

    public static void set(String id) {
        CORRELATION_ID.set(id);
    }

    public static String get() {
        return CORRELATION_ID.get();
    }

    public static void clear() {
        CORRELATION_ID.remove(); // ALWAYS clear in a finally block to avoid leaks in thread pools
    }
}
```

**Spring example** — `RequestContextHolder`/MDC pattern via a `Filter`, common for correlation IDs in logs:

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {
        String correlationId = Optional.ofNullable(request.getHeader("X-Correlation-Id"))
                .orElse(UUID.randomUUID().toString());
        MDC.put("correlationId", correlationId); // MDC is backed by ThreadLocal
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear(); // critical: thread pool threads are reused, must clear
        }
    }
}
```

**Pitfall**: In a thread pool (Tomcat, `ExecutorService`), threads are reused. Forgetting to clear a `ThreadLocal` leaks state (and memory) into the next request handled by that thread — a classic interview trap question.

---

## 2. Immutability

An immutable object cannot be a source of a data race because there is no mutable state to race on.

```java
// Effectively immutable domain object - safe to share across threads without synchronization
public record TransferRequest(
        UUID transferId,
        String fromAccountId,
        String toAccountId,
        BigDecimal amount,
        Currency currency,
        Instant requestedAt) {

    public TransferRequest {
        Objects.requireNonNull(fromAccountId);
        Objects.requireNonNull(toAccountId);
        if (amount.signum() <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
    }
}
```

Rules to keep a class truly immutable:
1. All fields `final` and `private`.
2. No setters.
3. Defensive copies for mutable fields passed in/out (e.g. `List.copyOf(list)`).
4. Class is not subclassable in a way that breaks invariants (`final` class, or use `record`/`sealed`).

**Spring tie-in**: DTOs and value objects in a hexagonal architecture (`domain.model`) should default to immutable records — they cross thread boundaries constantly (HTTP thread → async executor → Kafka consumer thread) and immutability removes an entire class of bugs for free.

---

## 3. Synchronization: Executors, `@Async`, and `CompletableFuture`

### Why raw `Thread` / `new Thread(...)` is a red flag in Spring code

Interviewers expect you to reach for `ExecutorService`/Spring abstractions instead of creating raw threads, because you get pooling, backpressure, and lifecycle management for free.

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    @Bean(name = "fraudCheckExecutor")
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("fraud-check-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> log.error("Async error in {}", method, ex);
    }
}
```

```java
@Service
public class FraudDetectionService {

    @Async("fraudCheckExecutor")
    public CompletableFuture<FraudScore> scoreTransactionAsync(Transaction transaction) {
        FraudScore score = computeScore(transaction); // CPU/IO bound work
        return CompletableFuture.completedFuture(score);
    }
}
```

**Combining independent async calls (fan-out/fan-in)** — a very common interview question ("how would you call 3 services in parallel and combine the results?"):

```java
@Service
public class TransferOrchestrator {

    public TransferResult processTransfer(TransferRequest request) {
        CompletableFuture<FraudScore> fraudCheck = fraudDetectionService.scoreTransactionAsync(request.toTransaction());
        CompletableFuture<ExchangeRate> rate = fxService.getRateAsync(request.currency());
        CompletableFuture<AccountBalance> balance = accountService.getBalanceAsync(request.fromAccountId());

        return CompletableFuture.allOf(fraudCheck, rate, balance)
                .thenApply(v -> new TransferResult(fraudCheck.join(), rate.join(), balance.join()))
                .exceptionally(ex -> TransferResult.failed(ex.getMessage()))
                .join(); // join only at the boundary (e.g. controller), never inside a service chain
    }
}
```

**Interview trap**: `@Async` on a method called from within the *same class* is silently ignored (Spring AOP proxies don't intercept self-invocation). Always call `@Async` methods through an injected bean/proxy.

### Explicit locking when you truly need it

Prefer higher-level utilities before reaching for `synchronized`/`ReentrantLock` directly:

```java
public class IdempotencyKeyCache {

    // ConcurrentHashMap.computeIfAbsent is atomic per key -> no external lock needed
    private final ConcurrentHashMap<String, CompletableFuture<TransferResult>> inFlight = new ConcurrentHashMap<>();

    public CompletableFuture<TransferResult> execute(String idempotencyKey, Supplier<TransferResult> action) {
        return inFlight.computeIfAbsent(idempotencyKey,
                key -> CompletableFuture.supplyAsync(action).whenComplete((r, ex) -> inFlight.remove(key)));
    }
}
```

This pattern (dedupe concurrent requests sharing the same idempotency key) is exactly the kind of "why?" question a fintech interviewer will drill into.

---

## 4. Concurrent Collections Cheat Sheet

| Need | Use | Avoid |
|---|---|---|
| Thread-safe map | `ConcurrentHashMap` | `Collections.synchronizedMap(new HashMap<>())` (coarse lock) |
| Read-heavy, rarely-written list | `CopyOnWriteArrayList` | `synchronized List` under high write load |
| Producer/consumer queue | `BlockingQueue` (`ArrayBlockingQueue`, `LinkedBlockingQueue`) | manual `wait`/`notify` |
| Counters | `LongAdder` / `AtomicLong` | `synchronized` increment |
| Bounded parallel work | `Semaphore` | ad-hoc counters + `wait` |

```java
// Rate limiting outbound calls to a downstream FX provider
public class FxProviderRateLimiter {

    private final Semaphore permits = new Semaphore(20); // max 20 concurrent calls

    public ExchangeRate getRate(String pair) throws InterruptedException {
        permits.acquire();
        try {
            return fxClient.fetchRate(pair);
        } finally {
            permits.release();
        }
    }
}
```

---

## Interview Q&A

**Q1: Is `@Service` in Spring thread-safe by default?**
Spring singleton beans are shared across all requests/threads. They are thread-safe *only if they have no mutable instance state* (or that state is itself thread-safe). Stateless services with only injected `final` collaborators are safe by construction — this is why constructor injection + immutable dependencies is the default Spring pattern.

**Q2: Why prefer `ConcurrentHashMap.computeIfAbsent` over `synchronized` + `HashMap`?**
`ConcurrentHashMap` uses lock striping (segments/buckets), so contention is limited to the specific bucket, not the whole map. `computeIfAbsent` also makes the check-then-act sequence atomic, avoiding a classic race condition where two threads both see the key absent and both insert.

**Q3: What happens if an unbounded thread pool is used to fan out microservice calls?**
Under load spikes it can create unbounded threads → OOM / context-switch thrashing / exhausting downstream connection pools. Always size pools deliberately (bounded queue + explicit rejection policy), and prefer reactive/backpressure-aware clients (WebClient) for high-fanout scenarios.

**Q4: Deadlock — how would you detect and prevent one in production?**
Detect: thread dumps (`jstack`, or `Thread.getAllStackTraces()`), APM tools (Datadog thread-state dashboards). Prevent: consistent lock ordering, lock timeouts (`tryLock(timeout)`), avoid holding a lock while calling into unknown/external code.

---

## Sources

- [Java Concurrency in Practice — Brian Goetz (the canonical reference)](https://jcip.net/)
- [Baeldung: Guide to the Java ExecutorService](https://www.baeldung.com/java-executor-service-tutorial)
- [Baeldung: Spring @Async](https://www.baeldung.com/spring-async)
- [Spring Framework Reference — Task Execution and Scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)
- [Java 21 Virtual Threads (JEP 444)](https://openjdk.org/jeps/444)
- [Oracle: java.util.concurrent package summary](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
