# 18. Exercises & Leveled Q&A — Event-Driven Design (EDD)

Covers [11-event-driven-design.md](./11-event-driven-design.md).

---

## Hands-On Exercises

### Exercise 1 (Basic) — Fix the `@EventListener` timing bug

The following code publishes a domain event and reacts to it with a notification, but users occasionally receive a "transfer completed" notification for transfers that later fail to commit (e.g. a downstream validation error rolls back the transaction after the event fires). Fix it.

```java
@Service
public class TransferService {

    @Transactional
    public void completeTransfer(TransferId id) {
        Transfer transfer = transferPort.findById(id);
        transfer.complete();
        transferPort.save(transfer);
        eventPublisher.publishEvent(new TransferCompletedEvent(transfer.getId()));
        // ... more code that might still throw and roll back the transaction
        auditService.recordCompletion(transfer.getId()); // this can throw
    }
}

@Component
public class NotificationListener {

    @EventListener
    public void onTransferCompleted(TransferCompletedEvent event) {
        notificationService.notifyUser(event.transferId());
    }
}
```

<details>
<summary>Solution</summary>

`@EventListener` fires synchronously as soon as `publishEvent` is called — *before* the surrounding `@Transactional` method finishes and commits. If `auditService.recordCompletion(...)` (called after the event is published) throws and rolls back the transaction, the notification has already been sent for a transfer that was never actually persisted as completed.

Fix: use `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)` so the listener only runs once the transaction has successfully committed:

```java
@Component
public class NotificationListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onTransferCompleted(TransferCompletedEvent event) {
        notificationService.notifyUser(event.transferId());
    }
}
```

(As a secondary improvement, move `auditService.recordCompletion` before publishing the event, or make it part of the same aggregate save, so failures there are caught before anything is published at all.)
</details>

---

### Exercise 2 (Mid) — Implement the Transactional Outbox pattern

You currently publish integration events directly to Kafka right after saving to Postgres, in two separate calls. Explain the risk and rewrite it using the transactional outbox pattern.

```java
@Transactional
public void completeTransfer(TransferId id) {
    Transfer transfer = transferPort.findById(id);
    transfer.complete();
    transferPort.save(transfer); // Postgres write #1
    kafkaTemplate.send("transfers.completed", new TransferCompletedIntegrationEvent(transfer.getId())); // separate system, no shared transaction
}
```

<details>
<summary>Solution</summary>

**Risk**: the Postgres write and the Kafka send are two independent systems with no shared transaction (the "dual write" problem). If the process crashes after the DB commit but before/during the Kafka send, the event is lost even though the transfer completed — downstream consumers (fraud scoring, notifications) never find out. Conversely, if Kafka send happens but the DB transaction later rolls back, you've published an event for something that never actually happened.

**Fix — write an outbox row in the same DB transaction, and let a separate poller publish it:**

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

```java
@Transactional
public void completeTransfer(TransferId id) {
    Transfer transfer = transferPort.findById(id);
    transfer.complete();
    transferPort.save(transfer); // same transaction as the outbox insert below

    OutboxEvent event = OutboxEvent.forDomainEvent(
            "Transfer", "TransferCompleted", new TransferCompletedIntegrationEvent(transfer.getId()));
    outboxRepository.save(event);
    // No Kafka call here at all - a separate poller/CDC process handles publishing
}
```

```java
@Component
public class OutboxPoller {

    @Scheduled(fixedDelay = 500)
    @Transactional
    public void pollAndPublish() {
        List<OutboxEvent> pending = outboxRepository.findUnpublishedForUpdateSkipLocked(50);
        for (OutboxEvent event : pending) {
            kafkaTemplate.send(event.topic(), event.aggregateId(), event.payload());
            event.markPublished();
        }
        outboxRepository.saveAll(pending);
    }
}
```

Now the business change and the "intent to publish" are atomic (same DB transaction) — the event can never be lost relative to the business change, and duplicate publishing (if the poller crashes mid-batch) is handled by making the Kafka consumer idempotent (see Exercise 3 below and doc 08's idempotency-key pattern).
</details>

---

### Exercise 3 (Senior) — Design an idempotent Kafka consumer

A `FraudScoringConsumer` listens to `transfers.completed` and must never double-score the same transfer, even though the topic guarantees only **at-least-once** delivery (consumer crash after processing but before committing the offset causes redelivery). Design the consumer to be safely idempotent.

<details>
<summary>Solution</summary>

```sql
CREATE TABLE processed_event (
    event_id UUID PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```java
@Component
public class FraudScoringConsumer {

    private final ProcessedEventRepository processedEventRepository;
    private final FraudDetectionService fraudDetectionService;

    @KafkaListener(topics = "transfers.completed", groupId = "fraud-scoring")
    @Transactional
    public void handle(TransferCompletedIntegrationEvent event) {
        // Insert-or-skip: relies on a unique constraint on event_id, same DB transaction
        // as the business side effect below, so both succeed or both roll back together.
        boolean firstTimeSeen = processedEventRepository.tryMarkProcessed(event.eventId());
        if (!firstTimeSeen) {
            return; // duplicate delivery, already handled - safe no-op
        }
        fraudDetectionService.scoreCompletedTransfer(event);
    }
}
```

```java
@Repository
public class ProcessedEventRepository {

    private final JdbcTemplate jdbcTemplate;

    public boolean tryMarkProcessed(UUID eventId) {
        // ON CONFLICT DO NOTHING + checking affected row count is atomic and race-safe
        // under concurrent redelivery/multiple consumer instances of the same group.
        int rows = jdbcTemplate.update(
                "INSERT INTO processed_event (event_id) VALUES (?) ON CONFLICT (event_id) DO NOTHING",
                eventId);
        return rows == 1;
    }
}
```

Why this is correct: the dedupe check-and-mark is a single atomic SQL statement (`ON CONFLICT DO NOTHING`), avoiding a TOCTOU race between two consumer instances (or two redeliveries) both checking "have I seen this event?" before either marks it processed. The dedupe insert and the actual business side effect (`scoreCompletedTransfer`) run in the *same* database transaction, so if scoring fails and rolls back, the dedupe marker rolls back too — allowing a legitimate retry, not permanently blocking reprocessing of an event that never actually got the side effect applied.

**Follow-up**: what if `scoreCompletedTransfer` itself calls an external, non-transactional system (e.g. sends an email)? Then true exactly-once is impossible end-to-end — you'd need that external call to also be idempotent (e.g. keyed by `event.eventId()`) or accept "at-least-once with an idempotent downstream" as the realistic guarantee, same as the DB-level dedupe.
</details>

---

## Leveled Interview Q&A

### Basic Level

**Q1: What is the difference between a Domain Event and an Integration Event?**
A Domain Event stays inside a bounded context/process (e.g. published via Spring's `ApplicationEventPublisher`, often within the same transaction). An Integration Event crosses service/bounded-context boundaries, typically via a message broker, and must be treated as a versioned public contract other teams depend on.

**Q2: Why use events instead of direct synchronous calls between services?**
To decouple producers from consumers — the producer doesn't need to know who consumes the event or how many consumers exist, and consumers can be added/removed without changing the producer. It also lets slow/non-critical downstream work (notifications, analytics) happen asynchronously without blocking the main request.

**Q3: What does "eventual consistency" mean?**
Different parts of the system may be temporarily out of sync after a change, but are guaranteed to converge to a consistent state eventually (e.g. once all relevant events have been processed), as opposed to strong consistency where every read immediately reflects the latest write.

**Q4: What is Kafka, in one sentence?**
A distributed, durable, publish-subscribe message broker/log used to reliably move events between producers and consumers at scale.

**Q5: What does "at-least-once delivery" mean, and what does it imply for consumers?**
The broker guarantees a message will be delivered one or more times (never silently dropped), but may deliver duplicates (e.g. after a consumer crash before acknowledging). It implies consumers must be idempotent — processing the same message twice must be safe.

---

### Mid Level

**Q1: What is the "dual write" problem and why is it dangerous?**
Writing to two separate systems (e.g. a database and a message broker) without a shared transaction — if the process crashes between the two writes, the systems can end up inconsistent (e.g. the DB change committed but the event was never published, or vice versa). It's dangerous because it silently produces missing or contradictory events, hard to detect until a downstream discrepancy is noticed.

**Q2: How does the Transactional Outbox pattern solve the dual-write problem?**
By writing the "intent to publish an event" as a row in an outbox table, in the *same* database transaction as the business change. A separate poller (or CDC tool like Debezium) later reads unpublished outbox rows and publishes them to the broker — since the outbox insert is atomic with the business change, the event can never be lost or fabricated relative to what was actually committed.

**Q3: Why must you use `@TransactionalEventListener(phase = AFTER_COMMIT)` instead of the default `@EventListener` for side effects like sending notifications?**
The default `@EventListener` fires synchronously the moment the event is published, which can be before the surrounding transaction actually commits (or even if it later rolls back) — risking side effects (like a notification) for changes that never persisted. `AFTER_COMMIT` guarantees the listener only runs once the transaction has successfully committed.

**Q4: What's the difference between Event Sourcing and simply publishing domain events after a CRUD-style update?**
In Event Sourcing, the sequence of events **is** the source of truth — current state is derived by replaying events, and there is no separately-stored "current state" row that's the primary record. Publishing events after a CRUD update is much simpler: the database row is still the source of truth, and events are just a notification mechanism for other consumers — no replay/rebuild capability, but far less operational complexity.

**Q5: What's the risk of a Kafka consumer processing a message but crashing before committing its offset?**
The message will be redelivered when the consumer restarts (that's the "at-least-once" guarantee) — meaning the same message could be processed twice unless the consumer's business logic (or an explicit dedupe table) is idempotent.

---

### Senior Level

**Q1: How would you evolve an integration event's schema (e.g. adding a new required field) without breaking existing consumers that haven't been updated yet?**
Treat integration events like a public API contract: add new fields as optional/nullable first (backward-compatible additive change), never remove or repurpose existing fields without a coordinated migration, and consider consumer-driven contract testing or a schema registry (e.g. Avro/Protobuf with Confluent Schema Registry) to catch breaking changes at build/deploy time rather than in production. If a genuinely breaking change is required, publish to a new versioned topic and run both versions in parallel until all consumers have migrated.

**Q2: When would you justify Event Sourcing/CQRS for a new service, and when would you actively avoid it?**
Justify it when you need a true audit trail/temporal queries (e.g. "what was the state at time T"), when write and read scaling/access patterns diverge significantly, or when the domain is naturally a sequence of events (ledgers, order books, workflow state machines). Avoid it for simple CRUD-shaped domains where the added operational complexity (event store infrastructure, replay/projection logic, eventual consistency between write and read models) isn't justified by an actual requirement — it's a well-known anti-pattern to reach for Event Sourcing "because it's interesting" rather than because the domain needs it.

**Q3: How do you handle event ordering guarantees when scaling out consumers of a Kafka topic?**
Kafka guarantees ordering only within a single partition. If ordering matters for a given entity (e.g. all events for one `accountId` must be processed in order), you must key the topic by that entity's ID so all its events land on the same partition, and ensure you don't parallelize processing *within* that partition beyond a single consumer thread per partition. Cross-partition (i.e. cross-entity) ordering is not guaranteed and usually shouldn't be relied upon.

**Q4: How would you design for exactly-once *effective* processing when the downstream side effect is not itself transactional with your database (e.g. calling an external payment provider)?**
True exactly-once across two independent systems is not achievable in the general case (this is the essence of the dual-write problem applied to an outbound call instead of an event). The practical answer: make the external call itself idempotent by sending a stable idempotency key the provider deduplicates on (many payment APIs, e.g. Stripe, support this natively), combined with recording locally whether the call was attempted/acknowledged, so retries after a crash are safe no-ops from the provider's perspective rather than double charges.

**Q5: What's the operational cost of the Transactional Outbox + poller pattern, and how would you reduce publish latency without reintroducing the dual-write risk?**
The main costs are extra storage/write load on the outbox table and added publish latency (bounded by the poller's interval). To reduce latency while keeping atomicity, use Change Data Capture (e.g. Debezium reading the Postgres WAL) instead of a polling loop — CDC picks up new outbox rows near-instantly by tailing the write-ahead log rather than polling on an interval, while preserving the same atomicity guarantee (the row only exists because the original transaction committed).

---

## Sources

See [11-event-driven-design.md](./11-event-driven-design.md) for the full reading list (Fowler, microservices.io, Debezium, Confluent).
