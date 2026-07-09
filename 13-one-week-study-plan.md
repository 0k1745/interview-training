# 13. One-Week Study Plan (1–2 hours/day) — Revolut Java Engineer Interview

**Goal**: be interview-ready on concurrency/multithreading (app + DB level), PostgreSQL/database architecture, DDD, EDD, and system design, in 7 days at 1–2h/day (~10–12h total).

Each day: **~60% reading/understanding, ~40% active recall** (explain out loud / write code from memory / answer the Q&A section without looking).

| Day | Focus | Materials | Active recall exercise |
|---|---|---|---|
| **Day 1** | App-level concurrency refresher: process/thread, JMM, synchronization | [01](./01-process-vs-thread.md), [03](./03-java-memory-model.md), [04](./04-synchronization-mechanisms.md) | Explain happens-before out loud, without notes, in under 2 minutes |
| **Day 2** | Locks, atomics, thread safety patterns with Spring | [05](./05-atomic-and-lock-free.md), [06](./06-locks-and-conditions.md), [08](./08-thread-safety-and-concurrency-patterns.md) | Write (from memory) a `@Async` fan-out with `CompletableFuture.allOf` |
| **Day 3** | Database concurrency: MVCC, isolation levels, locking | [09](./09-database-concurrency-and-architecture.md) sections 1–4 | Answer Q1–Q3 of doc 09 without looking; sketch a deadlock scenario and its fix |
| **Day 4** | Database architecture: pooling, indexing, partitioning, replication | [09](./09-database-concurrency-and-architecture.md) sections 5–7 | Explain HikariCP pool sizing trade-offs; design a partitioned `transactions` table on paper |
| **Day 5** | Domain-Driven Design | [10](./10-domain-driven-design.md) | Model a `Transfer` aggregate from scratch (root, invariants, domain events) without looking at the example |
| **Day 6** | Event-Driven Design | [11](./11-event-driven-design.md) | Explain the transactional outbox pattern and why dual writes are dangerous, to a rubber duck / colleague |
| **Day 7** | System design + full mock interview | [12](./12-system-design-for-interview.md) + [07](./07-interview-questions.md) | Do a 45-min mock: design the money transfer system out loud, then answer 5 random Q&A questions picked from all docs |

### Suggested daily time split (for the 1.5h slots)
- 20 min: read/re-read the day's doc(s).
- 20–30 min: rewrite key code examples from memory (don't copy-paste — type them out).
- 20–30 min: answer that doc's "Interview Q&A" section out loud or in writing, then check against the doc.
- 10 min: note 2–3 follow-up questions you'd ask the interviewer to show engagement (e.g. "what isolation level do you run in production?", "do you use event sourcing anywhere, or plain CRUD + outbox?").

### Day 7 mock interview checklist
- [ ] Can explain why `counter++` isn't atomic, and what `volatile` does/doesn't fix.
- [ ] Can explain MVCC and why Postgres readers don't block writers.
- [ ] Can compare optimistic vs pessimistic locking with a concrete "when would you pick each" answer.
- [ ] Can model an aggregate with correct invariants and domain events.
- [ ] Can explain the transactional outbox pattern and why naive dual writes fail.
- [ ] Can whiteboard the money transfer system end-to-end in under 10 minutes, naming CAP trade-offs explicitly.
- [ ] Has 2–3 questions ready for the interviewer about Revolut's actual stack (Kafka? outbox/Debezium? isolation level in prod? sharding strategy?).

---

## Full Reading List (all sources referenced across the docs)

**Concurrency (application level)**
- [Java Concurrency in Practice — Brian Goetz](https://jcip.net/)
- [Baeldung: Java ExecutorService](https://www.baeldung.com/java-executor-service-tutorial)
- [Baeldung: Spring @Async](https://www.baeldung.com/spring-async)
- [Spring Docs: Task Execution and Scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)

**Database / PostgreSQL**
- [PostgreSQL Docs: MVCC](https://www.postgresql.org/docs/current/mvcc.html)
- [PostgreSQL Docs: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL Docs: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [HikariCP: About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [Baeldung: Optimistic vs Pessimistic Locking in JPA](https://www.baeldung.com/jpa-optimistic-locking)

**DDD**
- [Martin Fowler: Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Martin Fowler: Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)
- Eric Evans, *Domain-Driven Design* (the "blue book")
- Vaughn Vernon, [Effective Aggregate Design](https://www.dddcommunity.org/library/vernon_2011/)

**EDD**
- [Martin Fowler: What do you mean by Event-Driven?](https://martinfowler.com/articles/201701-event-driven.html)
- [microservices.io: Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [Martin Fowler: Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) / [CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Debezium outbox event router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)

**System Design**
- Martin Kleppmann, [*Designing Data-Intensive Applications*](https://dataintensive.net/)
- [AWS: CAP Theorem explained](https://aws.amazon.com/builders-library/cap-theorem-and-distributed-systems/)
- [Resilience4j docs](https://resilience4j.readme.io/docs)
- [Stripe: Idempotent Requests](https://stripe.com/blog/idempotency)
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
