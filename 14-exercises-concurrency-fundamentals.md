# 14. Exercises & Leveled Q&A — Concurrency Fundamentals

Covers [01-process-vs-thread.md](./01-process-vs-thread.md), [02-cpu-scheduling-memory.md](./02-cpu-scheduling-memory.md), [03-java-memory-model.md](./03-java-memory-model.md), [04-synchronization-mechanisms.md](./04-synchronization-mechanisms.md), [05-atomic-and-lock-free.md](./05-atomic-and-lock-free.md), [06-locks-and-conditions.md](./06-locks-and-conditions.md). For a first pass at Q&A on these topics, also see [07-interview-questions.md](./07-interview-questions.md).

How to use this file: attempt each exercise yourself first (pen and paper or an IDE), then expand the solution. For the Q&A, cover the answer and try to answer out loud before checking.

---

## Hands-On Exercises

### Exercise 1 (Basic) — Spot the race condition

Given this class, identify the race condition and fix it with the minimal change.

```java
public class HitCounter {
    private int hits = 0;

    public void recordHit() {
        hits++;
    }

    public int getHits() {
        return hits;
    }
}
```

<details>
<summary>Solution</summary>

`hits++` is read-modify-write — three operations (read, add 1, write) that are not atomic. Two threads can both read the same value before either writes back, losing an increment.

Minimal fix — use `AtomicInteger` (no lock needed, CAS-based):

```java
public class HitCounter {
    private final AtomicInteger hits = new AtomicInteger(0);

    public void recordHit() {
        hits.incrementAndGet();
    }

    public int getHits() {
        return hits.get();
    }
}
```

`synchronized` on both methods would also fix it, but `AtomicInteger` is the better answer for a single counter — no blocking, lower contention, and it directly demonstrates you understand *why* CAS-based atomics exist (see [05-atomic-and-lock-free.md](./05-atomic-and-lock-free.md)).
</details>

---

### Exercise 2 (Mid) — `volatile` vs `synchronized`

The following is a common "stop flag" pattern. Explain what's wrong (if anything) and fix it.

```java
public class Worker implements Runnable {
    private boolean running = true;

    public void stop() {
        running = false;
    }

    @Override
    public void run() {
        while (running) {
            doWork();
        }
    }
}
```

<details>
<summary>Solution</summary>

Without `volatile`, there is no happens-before relationship between the write to `running` in `stop()` (called from another thread, e.g. the main thread) and the read in `run()`'s loop. The JIT compiler is legally allowed to cache `running` in a register and never re-read it from main memory, causing an infinite loop that never observes the `stop()` call — this can and does happen in real (especially long-running) JVMs.

Fix:

```java
public class Worker implements Runnable {
    private volatile boolean running = true;

    public void stop() {
        running = false;
    }

    @Override
    public void run() {
        while (running) {
            doWork();
        }
    }
}
```

`volatile` here is sufficient because it's a single boolean flag — a pure visibility/ordering problem, not a compound (read-modify-write) operation, so no atomicity guarantee is needed beyond what `volatile` already provides. This is exactly the "why doesn't `volatile` make `counter++` safe, but it *does* fix this" distinction from [03-java-memory-model.md](./03-java-memory-model.md) and [04-synchronization-mechanisms.md](./04-synchronization-mechanisms.md).
</details>

---

### Exercise 3 (Senior) — Double-checked locking and lazy initialization

This lazy singleton is a well-known broken pattern. Explain precisely why it's broken pre-Java 5, why it works from Java 5+ with one specific keyword, and rewrite it in the idiomatic modern way.

```java
public class ConfigLoader {
    private static ConfigLoader instance;

    public static ConfigLoader getInstance() {
        if (instance == null) {
            synchronized (ConfigLoader.class) {
                if (instance == null) {
                    instance = new ConfigLoader();
                }
            }
        }
        return instance;
    }
}
```

<details>
<summary>Solution</summary>

**Why it's broken without `volatile` (pre-JMM revision, or if the field isn't `volatile`)**: `instance = new ConfigLoader()` is not a single atomic step. The JVM/JIT may (1) allocate memory, (2) publish the reference to `instance`, (3) run the constructor — in that reordered sequence. Another thread entering `getInstance()` between steps (2) and (3) sees a non-null `instance` reference that points to a **partially constructed object**, and returns it — a subtly corrupt singleton that only shows up under concurrent load, notoriously hard to reproduce/debug.

**Why `volatile` fixes it (Java 5+ JMM)**: since JSR-133 (Java 5), a `volatile` write has a happens-before relationship with subsequent `volatile` reads, which — combined with the revised JMM's stronger guarantees around `volatile` and final field semantics — prevents the reordering described above from being observable. Marking the field `volatile` makes double-checked locking actually correct:

```java
private static volatile ConfigLoader instance;
```

**Idiomatic modern rewrite** — avoid double-checked locking entirely using the initialization-on-demand holder idiom (relies on JVM class-loading guarantees, which are inherently thread-safe and lazy):

```java
public class ConfigLoader {
    private ConfigLoader() { }

    private static class Holder {
        private static final ConfigLoader INSTANCE = new ConfigLoader();
    }

    public static ConfigLoader getInstance() {
        return Holder.INSTANCE;
    }
}
```

The `Holder` class is only loaded (and `INSTANCE` initialized) on first access to `getInstance()`, and the JVM spec guarantees class initialization is thread-safe and happens at most once — no explicit synchronization needed at all.

In a Spring context, none of this is usually necessary — a `@Bean`/`@Component` singleton is already lazily-or-eagerly initialized once by the container under a thread-safe guarantee, which is worth stating explicitly if asked ("in practice I'd just let Spring manage the singleton").
</details>

---

## Leveled Interview Q&A

### Basic Level

**Q1: What is the difference between a process and a thread?**
A process is an isolated unit of execution with its own memory space; threads are execution paths within a process that share the same heap but have their own stack. See [01-process-vs-thread.md](./01-process-vs-thread.md).

**Q2: What does "thread-safe" mean?**
A class/method behaves correctly when accessed by multiple threads concurrently, without requiring extra synchronization from the caller.

**Q3: Name two ways to create a thread in Java.**
Extending `Thread` and overriding `run()`, or implementing `Runnable` (or `Callable`) and passing it to a `Thread` or an `ExecutorService`. In practice, prefer `ExecutorService`/thread pools over manually creating `Thread` instances.

**Q4: What is a race condition?**
A situation where the correctness of a program depends on the relative timing/interleaving of multiple threads accessing shared mutable state, typically leading to incorrect results that appear only intermittently.

**Q5: What does `synchronized` do, at a high level?**
It ensures mutual exclusion (only one thread executes the guarded block/method at a time for a given monitor) and establishes a happens-before relationship, so changes made inside the block are visible to the next thread that acquires the same lock.

---

### Mid Level

**Q1: What is the Java Memory Model and why does it exist?**
It's the specification that defines what visibility, ordering, and atomicity guarantees the JVM must provide across threads (the contract between language and JVM), independent of the actual CPU/cache architecture. It exists because different CPU architectures have different memory consistency models, and Java needs a single, portable set of guarantees. See [03-java-memory-model.md](./03-java-memory-model.md).

**Q2: Why doesn't `volatile` make `counter++` thread-safe?**
`volatile` guarantees visibility and ordering for reads/writes of that variable, but `counter++` is three separate operations (read, increment, write) — `volatile` doesn't make that sequence atomic. Two threads can still interleave between the read and the write.

**Q3: What's the difference between `synchronized` and `ReentrantLock`?**
`synchronized` is a built-in, simpler, JVM-managed intrinsic lock (auto-released on exception/block-exit). `ReentrantLock` offers more control: tryLock with timeout, interruptible lock acquisition, fairness policies, and multiple `Condition` objects per lock (vs. one implicit condition with `synchronized`+`wait/notify`). Use `ReentrantLock` when you need that extra control; otherwise prefer `synchronized` for simplicity.

**Q4: What is a happens-before relationship?**
A partial ordering guarantee: if action A happens-before action B, then A's effects (writes) are guaranteed visible to B, and their relative order is preserved. Established by things like: a `volatile` write happens-before a subsequent `volatile` read of the same field; unlocking a monitor happens-before a later thread locking the same monitor; thread start happens-before anything in the started thread's `run()`.

**Q5: Why can CPU/compiler instruction reordering break concurrent code, but not sequential code?**
Reordering is only invisible if you can't observe the intermediate order — true within a single thread (as-if-serial semantics guarantee the same final result). But another thread *can* observe intermediate states of shared memory, so if instructions affecting shared state are reordered, another thread can see an unexpected combination of values (e.g. a flag set before the data it's supposed to guard is fully written).

---

### Senior Level

**Q1: Explain what specifically changed in the JMM with JSR-133 (Java 5) and why the old model was insufficient.**
Before JSR-133, the JMM was ambiguously specified and allowed several unintuitive/undesirable optimizations (e.g. `volatile` reads/writes could still be reordered relative to non-volatile ones in ways that broke common patterns like double-checked locking). JSR-133 formalized the happens-before model, strengthened `volatile` semantics (full happens-before ordering, not just per-variable atomicity), and fixed `final` field semantics so that safely-published immutable objects are visible without synchronization once the constructor completes properly.

**Q2: Why are `long` and `double` writes not guaranteed atomic in Java, even without `volatile`?**
On 32-bit JVMs (and per the JLS, even conceptually today), a 64-bit `long`/`double` may be treated as two separate 32-bit writes, so a concurrent read could observe a "torn" value — half from the old value, half from the new. Marking the field `volatile` restores atomicity for that field's reads/writes (JLS guarantees `volatile long`/`double` operations are atomic).

**Q3: Why is CAS (compare-and-swap) preferred over locking for simple atomic operations at scale, and what's the trade-off?**
CAS is a hardware-supported, lock-free primitive (no OS-level thread suspension/context switch on contention) — under low-to-moderate contention it's much faster than acquiring a lock. The trade-off: CAS-based retry loops degrade under *high* contention (many threads spinning and retrying), and CAS alone doesn't compose across multiple variables/invariants the way a lock naturally does (a lock lets you make several related updates atomic as a unit; a single CAS only protects one memory location, hence why multi-field atomic updates need care — e.g. `AtomicReference` to an immutable holder object, or `VarHandle`s, or fall back to locking).

**Q4: A production thread dump shows several threads `BLOCKED` on the same monitor, and the app appears to be making progress but very slowly. Is this a deadlock? How would you distinguish deadlock from contention/starvation?**
Not necessarily. `BLOCKED` alone indicates lock contention, not deadlock — deadlock requires a *cycle* of threads each waiting on a lock the other holds, with none able to proceed (thread dumps explicitly report "Found one Java-level deadlock" when this is detected). Heavy contention/starvation shows threads eventually acquiring the lock, just slowly, whereas true deadlock shows threads permanently stuck (no state change across repeated dumps taken seconds apart). Distinguish by taking multiple thread dumps a few seconds apart and diffing thread states/stack traces.

**Q5: Why doesn't marking every field `volatile` (or every method `synchronized`) make a class thread-safe?**
Because thread safety is about protecting *invariants across multiple related fields*, not individual variables. If a class has two related fields (e.g. `balance` and `lastTransactionId`) that must be updated together consistently, making each field individually `volatile`/synchronized-on-its-own-accessor still allows another thread to observe an inconsistent combination (one updated, one not) between the two operations. You need a single lock (or an immutable composite object swapped atomically) covering the *whole* invariant, not per-field synchronization.
