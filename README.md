# Java Concurrency & System Design Interview Preparation

**Target: Senior Java Engineer at Revolut**

This repository contains comprehensive, deep-dive documentation covering: Java concurrency fundamentals, thread safety with Spring, PostgreSQL concurrency & database architecture, Domain-Driven Design, Event-Driven Design, and system design — the exact focus areas communicated for this interview ("concurrency & multithreading on both database and application level", "PostgreSQL and database architecture", "system design, DDD/EDD"). The goal is to prepare you to answer "Why?" five times in a row and demonstrate true senior-level understanding.

## Documentation Index

### Concurrency Fundamentals (Application Level)

- [01. Process vs Thread](./01-process-vs-thread.md)
- [02. CPU Scheduling and Memory](./02-cpu-scheduling-memory.md)
- [03. The Java Memory Model](./03-java-memory-model.md)
- [04. Synchronization Mechanisms (volatile, synchronized)](./04-synchronization-mechanisms.md)
- [05. Atomic Classes and Lock-Free Programming](./05-atomic-and-lock-free.md)
- [06. Explicit Locks and Conditions](./06-locks-and-conditions.md)
- [07. Concurrency Interview Questions & Answers](./07-interview-questions.md)
- [08. Thread Safety & Concurrency Patterns (Java/Spring)](./08-thread-safety-and-concurrency-patterns.md)

### Database (Concurrency & Architecture)

- [09. Database Concurrency & Architecture (PostgreSQL)](./09-database-concurrency-and-architecture.md)

### Domain & Event-Driven Design

- [10. Domain-Driven Design (DDD)](./10-domain-driven-design.md)
- [11. Event-Driven Design (EDD)](./11-event-driven-design.md)

### System Design

- [12. System Design for the Interview](./12-system-design-for-interview.md)

### Study Plan

- [13. One-Week Study Plan (1–2h/day)](./13-one-week-study-plan.md)

### Exercises & Leveled Q&A (Basic / Mid / Senior)

Hands-on exercises with solutions, plus interview questions split into **Basic**, **Mid**, and **Senior** tiers, for each subject area:

- [14. Exercises & Leveled Q&A — Concurrency Fundamentals](./14-exercises-concurrency-fundamentals.md)
- [15. Exercises & Leveled Q&A — Thread Safety & Spring](./15-exercises-thread-safety-spring.md)
- [16. Exercises & Leveled Q&A — Database (PostgreSQL)](./16-exercises-database.md)
- [17. Exercises & Leveled Q&A — Domain-Driven Design](./17-exercises-ddd.md)
- [18. Exercises & Leveled Q&A — Event-Driven Design](./18-exercises-edd.md)
- [19. Exercises & Leveled Q&A — System Design](./19-exercises-system-design.md)

## Key Interview Questions (Concurrency Fundamentals)

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

See each document's own "Interview Q&A" section for questions on thread safety, PostgreSQL, DDD, EDD, and system design — and see files 14-19 for a Basic/Mid/Senior leveled version with hands-on exercises.

## What's Coming Next

- `ForkJoinPool`, Virtual Threads (Java 21) deep dive
- Sharding strategies and multi-region architecture
- Chaos engineering / resilience testing patterns

---

Each section has its own detailed guide with code examples, visual explanations, and interview tips. Start with [13. One-Week Study Plan](./13-one-week-study-plan.md) for a day-by-day schedule.
