# Java Concurrency Interview Preparation

**Target: Senior Java Engineer at Revolut**

This repository contains comprehensive, deep-dive documentation on Java concurrency fundamentals. The goal is to prepare you to answer "Why?" five times in a row and demonstrate true senior-level understanding.

## Week 1: Foundations

### Day 1: Processes, Threads, and the Java Memory Model

- **Section 1**: Process vs Thread
- **Section 2**: CPU Scheduling
- **Section 3**: Shared Memory
- **Section 4**: CPU Cache
- **Section 5**: The Java Memory Model
- **Section 6**: Volatile
- **Section 7**: Synchronized
- **Section 8**: Intrinsic Locks vs Explicit Locks
- **Section 9**: CAS (Compare-And-Swap)
- **Section 10**: Why Lock-Free Isn't Always Better

## Key Interview Questions

1. Why does `counter++` compile into multiple operations?
2. What guarantees does the Java Memory Model provide?
3. Explain the difference between visibility, atomicity, and ordering.
4. Why doesn't `volatile` make `counter++` safe?
5. What exactly happens when entering and leaving a `synchronized` block?
6. When would you choose `ReentrantLock` over `synchronized`?
7. Why are atomic classes implemented with CAS instead of `synchronized`?
8. What is the happens-before relationship, and why is it fundamental?
9. Why can compiler and CPU instruction reordering break concurrent code?
10. Why can a thread continue seeing an outdated value even after another thread has modified it?

## What's Coming Next

- `ConcurrentHashMap`, `ExecutorService`, `CompletableFuture`
- `ForkJoinPool`, Virtual Threads (Java 21)
- Deadlocks, starvation, livelocks
- Thread pool tuning

---

Each section has its own detailed guide with code examples, visual explanations, and interview tips.
