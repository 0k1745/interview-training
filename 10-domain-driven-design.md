# 10. Domain-Driven Design (DDD)

**Context**: Jazmin's message lists "system design, DDD/EDD" as a core focus area. Your own global conventions already follow a Hexagonal Architecture (`boundaries` / `domain` / `infrastructure`) — this doc formalizes the DDD vocabulary behind that structure so you can talk about it fluently in interview terms.

---

## 1. Strategic DDD: Bounded Contexts

A **Bounded Context** is an explicit boundary within which a domain model is consistent and has a single, unambiguous meaning. The same real-world word can mean different things in different contexts.

Example at a fintech company:

- In the **Payments** context, an `Account` means a settlement/ledger account with a balance.
- In the **Identity** context, an `Account` means a user's login credentials/profile.
- In the **Card Issuing** context, an `Account` means the card program membership.

Each context has its own model, its own database (or schema), and its own team ownership. They communicate through well-defined contracts (APIs, events) — never by sharing a database table.

**Context Map** relationships you should be able to name in an interview:
- **Shared Kernel**: two teams share a small common model (e.g. `Money`, `CurrencyCode`).
- **Customer/Supplier**: one team's context is a dependency of another; the supplier accommodates the customer's needs.
- **Anti-Corruption Layer (ACL)**: a translation layer that prevents a legacy or external model from leaking into your clean domain model (this is exactly what an `infrastructure/api` adapter + mapper does in your hexagonal structure).

---

## 2. Tactical DDD Building Blocks

| Concept | Definition | Example |
|---|---|---|
| **Entity** | Has identity that persists across state changes | `Account` (identified by `accountId`, balance changes over time) |
| **Value Object** | Immutable, defined by its attributes, no identity | `Money(amount, currency)`, `Address` |
| **Aggregate** | A cluster of entities/value objects treated as a single consistency unit, with one **Aggregate Root** | `Transfer` aggregate containing `TransferLine` value objects |
| **Repository** | Persistence abstraction for a whole aggregate (mirrors your `domain.port` + adapter pattern) | `TransferRepository` |
| **Domain Service** | Stateless operation that doesn't naturally belong to one entity | `FxConversionService` |
| **Domain Event** | Something meaningful that happened in the domain | `TransferCompletedEvent` |
| **Factory** | Encapsulates complex aggregate creation logic | `TransferFactory.createFromRequest(...)` |

### Entity vs Value Object (Java example)

```java
// Value Object: immutable, equality by value, no identity
public record Money(BigDecimal amount, Currency currency) {

    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
    }

    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }

    private void requireSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new CurrencyMismatchException(currency, other.currency);
        }
    }
}
```

```java
// Entity: has identity (accountId), mutable state, equality by ID
public class Account {

    private final AccountId id;
    private Money balance;

    public Account(AccountId id, Money initialBalance) {
        this.id = Objects.requireNonNull(id);
        this.balance = Objects.requireNonNull(initialBalance);
    }

    public void debit(Money amount) {
        if (balance.amount().compareTo(amount.amount()) < 0) {
            throw new InsufficientFundsException(id, amount);
        }
        this.balance = balance.subtract(amount);
    }

    @Override
    public boolean equals(Object o) {
        return o instanceof Account other && this.id.equals(other.id); // identity-based equality
    }
}
```

### Aggregate: enforcing invariants at the boundary

```java
// domain/model/Transfer.java  -- Aggregate Root
public class Transfer {

    private final TransferId id;
    private final AccountId fromAccount;
    private final AccountId toAccount;
    private final Money amount;
    private TransferStatus status;
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public static Transfer initiate(AccountId from, AccountId to, Money amount) {
        if (from.equals(to)) {
            throw new IllegalArgumentException("Cannot transfer to the same account");
        }
        Transfer transfer = new Transfer(TransferId.newId(), from, to, amount, TransferStatus.PENDING);
        transfer.domainEvents.add(new TransferInitiatedEvent(transfer.id, from, to, amount));
        return transfer;
    }

    public void complete() {
        if (status != TransferStatus.PENDING) {
            throw new IllegalStateException("Only a pending transfer can be completed");
        }
        this.status = TransferStatus.COMPLETED;
        domainEvents.add(new TransferCompletedEvent(id, fromAccount, toAccount, amount));
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = List.copyOf(domainEvents);
        domainEvents.clear();
        return events;
    }
}
```

**Rule of thumb the interviewer wants to hear**: *only the aggregate root is accessible from outside; all invariants (e.g. "cannot debit below zero", "cannot complete a non-pending transfer") are enforced inside the aggregate, never in a service or, worse, in the controller.*

---

## 3. Repository & Port (mirrors your existing hexagonal convention)

```java
// domain/port/TransferPort.java
public interface TransferPort {
    Transfer findById(TransferId id);
    void save(Transfer transfer);
}
```

```java
// infrastructure/postgres/repository/TransferRepositoryAdapter.java
@Repository
public class TransferRepositoryAdapter implements TransferPort {

    private final TransferJpaRepository jpaRepository;
    private final TransferMapper mapper;

    public TransferRepositoryAdapter(TransferJpaRepository jpaRepository, TransferMapper mapper) {
        this.jpaRepository = jpaRepository;
        this.mapper = mapper;
    }

    @Override
    public Transfer findById(TransferId id) {
        return jpaRepository.findById(id.value())
                .map(mapper::toDomain)
                .orElseThrow(() -> new TransferNotFoundException(id));
    }

    @Override
    public void save(Transfer transfer) {
        jpaRepository.save(mapper.toEntity(transfer));
    }
}
```

```java
// domain/service/TransferService.java
@Service
public class TransferService {

    private final TransferPort transferPort;
    private final DomainEventPublisher eventPublisher;

    public TransferService(TransferPort transferPort, DomainEventPublisher eventPublisher) {
        this.transferPort = transferPort;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public TransferId initiateTransfer(AccountId from, AccountId to, Money amount) {
        Transfer transfer = Transfer.initiate(from, to, amount);
        transferPort.save(transfer);
        eventPublisher.publishAll(transfer.pullDomainEvents());
        return transfer.getId();
    }
}
```

This is precisely the pattern you already use: `port` interface in the domain, `Adapter` suffix in infrastructure, constructor injection, mapper for translation. Naming it explicitly as "Repository pattern + Anti-Corruption Layer + Aggregate" in the interview signals you know the DDD vocabulary behind your habitual structure.

---

## Interview Q&A

**Q1: What's the difference between an Entity and a Value Object?**
An Entity has a stable identity that persists through state changes and is compared by ID (`equals` on ID). A Value Object has no identity, is immutable, and is compared structurally (all fields equal ⇒ objects equal). `Money(10, EUR)` created twice are the same value object; two `Account`s with the same balance are still different entities if their IDs differ.

**Q2: Why should aggregates be small?**
Aggregates define a transactional/consistency boundary — everything inside must be saved atomically and locked together. Large aggregates cause lock contention and make it harder to scale; the guidance (Vaughn Vernon, "Effective Aggregate Design") is "design small aggregates, reference other aggregates by ID only, and use domain events for cross-aggregate consistency (eventual consistency)".

**Q3: How do bounded contexts map to microservices?**
Each microservice should own exactly one (or a small number of related) bounded context(s) and its own data store — this avoids the "distributed monolith" anti-pattern where multiple services share one database and become tightly coupled via schema.

**Q4: What's an Anti-Corruption Layer and where have you used one?**
A translation layer that converts an external/legacy model into your clean internal domain model so external changes/quirks don't leak in. In hexagonal architecture, this is exactly the job of the `infrastructure/[tech]/mapper` package converting external DTOs/entities to `domain.model` objects.

---

## Sources

- [Martin Fowler: DomainDrivenDesign](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Martin Fowler: BoundedContext](https://martinfowler.com/bliki/BoundedContext.html)
- [Martin Fowler: AnemicDomainModel](https://martinfowler.com/bliki/AnemicDomainModel.html) (a great "what NOT to do" reference)
- Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software* (the original DDD "blue book")
- Vaughn Vernon, *Implementing Domain-Driven Design* — see also his article [Effective Aggregate Design](https://www.dddcommunity.org/library/vernon_2011/)
- [Baeldung: Domain-Driven Design in Spring](https://www.baeldung.com/spring-data-ddd)
