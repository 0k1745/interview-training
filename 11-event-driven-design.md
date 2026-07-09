# 11. Event-Driven Design (EDD)

**Context**: Second half of "system design, DDD/EDD" from Jazmin's message. Builds directly on the Domain Events introduced in [10-domain-driven-design.md](./10-domain-driven-design.md) — DDD gives you the events, EDD is about how those events flow through the system.

---

## 1. Why Event-Driven at a Fintech Scale

Revolut-style systems (fraud detection, ledgers, notifications, real-time trading) need to:
- Decouple producers from consumers (payments team doesn't need to know who consumes `TransferCompletedEvent`).
- Scale reads/writes independently.
- Support asynchronous, eventually-consistent workflows (a transfer completing triggers fraud scoring, notification, statement update — none of which should block the transfer itself).

Trade-off to always name explicitly in an interview: **you give up strong consistency for availability/scalability/decoupling (this is the practical, systems-level flavor of the CAP theorem)**.

---

## 2. Core Patterns

### a) Domain Events (in-process, same transaction)

Published within a bounded context, typically via Spring's `ApplicationEventPublisher`, still inside the same DB transaction.

```java
@Component
public class DomainEventPublisher {

    private final ApplicationEventPublisher springPublisher;

    public DomainEventPublisher(ApplicationEventPublisher springPublisher) {
        this.springPublisher = springPublisher;
    }

    public void publishAll(List<DomainEvent> events) {
        events.forEach(springPublisher::publishEvent);
    }
}
```

```java
@Component
public class NotificationOnTransferCompleted {

    // Runs only after the surrounding transaction commits successfully
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onTransferCompleted(TransferCompletedEvent event) {
        notificationService.notifyUser(event.toAccount(), event.amount());
    }
}
```

**Interview trap**: using the default `@EventListener` here (instead of `@TransactionalEventListener(AFTER_COMMIT)`) means the listener fires even if the transaction later rolls back — a classic bug where a "transfer completed" notification is sent for a transfer that never actually committed.

### b) Integration Events (cross-service, via a broker)

Once an event needs to cross a service/bounded-context boundary, publish it to a broker (Kafka, GCP Pub/Sub) instead of calling the other service synchronously.

```java
@Component
public class KafkaIntegrationEventPublisher {

    private final KafkaTemplate<String, TransferCompletedIntegrationEvent> kafkaTemplate;

    public void publish(TransferCompletedIntegrationEvent event) {
        kafkaTemplate.send("transfers.completed", event.transferId().toString(), event);
    }
}
```

```java
@Service
public class FraudScoringConsumer {

    @KafkaListener(topics = "transfers.completed", groupId = "fraud-scoring")
    public void handle(TransferCompletedIntegrationEvent event) {
        fraudDetectionService.scoreCompletedTransfer(event);
    }
}
```

### c) The Transactional Outbox Pattern (critical reliability piece)

**Problem**: if you `save()` to Postgres and then `send()` to Kafka as two separate calls, a crash between them causes either a lost event or a duplicate — the classic "dual write" problem.

**Solution**: write the event to an `outbox` table in the *same DB transaction* as the business change; a separate poller/CDC process (e.g. Debezium) reads the outbox and publishes to Kafka, then marks it sent.

```java
@Transactional
public void completeTransfer(TransferId id) {
    Transfer transfer = transferPort.findById(id);
    transfer.complete();
    transferPort.save(transfer);

    // Same transaction, same DB -> atomic with the business change
    outboxRepository.save(OutboxEvent.from(transfer.pullDomainEvents()));
}
```

```sql
CREATE TABLE outbox_event (
    id UUID PRIMARY KEY,
    aggregate_type TEXT NOT NULL,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);
```

This gives you **at-least-once delivery** with atomicity — the event is guaranteed to be created if and only if the business transaction committed.

---

## 3. Event Sourcing & CQRS (advanced, likely to come up as a "have you heard of" question)

- **Event Sourcing**: instead of storing current state, store the sequence of events that led to it; current state is derived by replaying events. A ledger of `TransferInitiated` → `TransferCompleted` events *is* the source of truth, not a `balance` column.
- **CQRS (Command Query Responsibility Segregation)**: separate the write model (commands, enforcing invariants, often event-sourced) from the read model (denormalized projections optimized for queries, updated asynchronously from the events).

```java
// Write side: command handler, appends events
public class AccountCommandHandler {
    public void handle(DebitAccountCommand command) {
        Account account = eventStore.loadAggregate(command.accountId(), Account::new);
        account.debit(command.amount()); // validates invariants, raises AccountDebitedEvent
        eventStore.append(account.uncommittedEvents());
    }
}

// Read side: projection updated by consuming the same events, optimized for fast reads
@KafkaListener(topics = "account-events")
public void on(AccountDebitedEvent event) {
    accountBalanceReadModelRepository.decrementBalance(event.accountId(), event.amount());
}
```

**When to justify this complexity in an interview**: only when audit trail, temporal queries ("what was the balance at 3pm yesterday"), or strong read/write scaling divergence justify it — not a default choice. Overusing event sourcing/CQRS for simple CRUD is a well-known anti-pattern.

---

## 4. Delivery Guarantees & Idempotency

| Guarantee | Meaning | Cost |
|---|---|---|
| At-most-once | Message may be lost, never duplicated | Simplest, riskiest |
| At-least-once | Message never lost, may be duplicated | Requires idempotent consumers (the realistic default) |
| Exactly-once | Never lost, never duplicated | Very costly/complex; Kafka supports it only within its own ecosystem (transactions), not end-to-end |

Because most real systems run **at-least-once**, consumers must be idempotent:

```java
@KafkaListener(topics = "transfers.completed")
@Transactional
public void handle(TransferCompletedIntegrationEvent event) {
    if (processedEventRepository.existsById(event.eventId())) {
        return; // already handled, safe no-op
    }
    processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
    notificationService.notifyUser(event.toAccount(), event.amount());
}
```

This directly reuses the idempotency-key pattern from the concurrency doc ([08-thread-safety-and-concurrency-patterns.md](./08-thread-safety-and-concurrency-patterns.md), section 3) — same problem (dedupe concurrent/duplicate triggers), applied at the messaging layer instead of in-process.

---

## Interview Q&A

**Q1: What's the dual-write problem and how do you solve it?**
Writing to two systems (DB + broker) without a shared transaction can leave them inconsistent on failure. Solved with the Transactional Outbox pattern (write an outbox row in the same DB transaction, publish asynchronously via a poller or CDC tool like Debezium).

**Q2: Why must consumers be idempotent in an event-driven system?**
Because most brokers guarantee at-least-once delivery (a consumer crash after processing but before acknowledging causes redelivery). Consumers should track processed event IDs (dedupe table, unique constraint) so re-processing the same event is a safe no-op.

**Q3: When would you choose Event Sourcing/CQRS over a simple CRUD + events approach?**
When you need a full audit trail / time-travel queries, when write and read scaling patterns diverge significantly, or when the domain is naturally event-centric (ledgers, order books). Not as a default — it adds real operational and cognitive complexity.

**Q4: Domain Events vs Integration Events — what's the difference?**
Domain Events stay inside a bounded context/process, often published in the same transaction (Spring `ApplicationEventPublisher`). Integration Events cross service boundaries via a broker and must be versioned, backward-compatible, and delivered reliably (outbox pattern) since other teams depend on their schema.

---

## Sources

- [Martin Fowler: Event-Driven Architecture / What do you mean by "Event-Driven"?](https://martinfowler.com/articles/201701-event-driven.html)
- [Chris Richardson (microservices.io): Transactional Outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [Martin Fowler: Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Martin Fowler: CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Confluent: Idempotent Consumer pattern](https://www.confluent.io/blog/messaging-patterns-for-event-driven-microservices/)
- [Debezium (Change Data Capture for the outbox pattern)](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)
- [Spring Docs: Application Events](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events)
- [Spring for Apache Kafka reference](https://docs.spring.io/spring-kafka/reference/)
