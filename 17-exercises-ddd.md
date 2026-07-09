# 17. Exercises & Leveled Q&A — Domain-Driven Design (DDD)

Covers [10-domain-driven-design.md](./10-domain-driven-design.md).

---

## Hands-On Exercises

### Exercise 1 (Basic) — Entity or Value Object?

Classify each of the following as an **Entity** or a **Value Object**, and justify briefly:

1. `Address` (street, city, postal code) attached to a customer profile.
2. `Account` (has an account number, opens on a date, balance changes over time).
3. `Money` (amount + currency) used inside a `Transfer`.
4. `Customer` (has a customer ID, name can change, email can change).

<details>
<summary>Solution</summary>

1. **Value Object** — two addresses with the same street/city/postal code are interchangeable; it has no identity of its own, only defined by its attributes. If the customer moves, you replace the whole `Address`, you don't "mutate" the old one.
2. **Entity** — identified by its account number, and its balance/state changes over time while remaining "the same account". Two accounts with identical balances are still different entities.
3. **Value Object** — `Money(100, EUR)` is interchangeable with any other `Money(100, EUR)`; it's immutable and compared by value.
4. **Entity** — identified by customer ID; name/email can change but it's still the same customer (identity persists through state changes).
</details>

---

### Exercise 2 (Mid) — Enforce an invariant inside an aggregate

The following code allows a `Transfer` to be completed twice, and allows negative amounts to slip through because validation lives in the service layer instead of the domain model. Refactor so the aggregate itself enforces its invariants and cannot be put into an invalid state, no matter which service calls it.

```java
// Anemic domain model - state mutated freely from outside
public class Transfer {
    public String status;
    public BigDecimal amount;
}

@Service
public class TransferService {
    public void complete(Transfer transfer) {
        transfer.status = "COMPLETED"; // no check against current status
    }

    public Transfer create(BigDecimal amount) {
        Transfer transfer = new Transfer();
        transfer.amount = amount; // no validation, could be negative
        transfer.status = "PENDING";
        return transfer;
    }
}
```

<details>
<summary>Solution</summary>

```java
public class Transfer {

    private final TransferId id;
    private final Money amount;
    private TransferStatus status;

    private Transfer(TransferId id, Money amount, TransferStatus status) {
        this.id = id;
        this.amount = amount;
        this.status = status;
    }

    // Factory method enforces invariants at creation time
    public static Transfer initiate(Money amount) {
        if (amount.amount().signum() <= 0) {
            throw new IllegalArgumentException("Transfer amount must be positive");
        }
        return new Transfer(TransferId.newId(), amount, TransferStatus.PENDING);
    }

    // Behavior lives on the aggregate, not in an external service mutating public fields
    public void complete() {
        if (status != TransferStatus.PENDING) {
            throw new IllegalStateException("Only a PENDING transfer can be completed, was: " + status);
        }
        this.status = TransferStatus.COMPLETED;
    }

    public TransferId getId() { return id; }
    public Money getAmount() { return amount; }
    public TransferStatus getStatus() { return status; }
}

@Service
public class TransferService {

    private final TransferPort transferPort;

    @Transactional
    public TransferId create(Money amount) {
        Transfer transfer = Transfer.initiate(amount); // invariant enforced inside the aggregate
        transferPort.save(transfer);
        return transfer.getId();
    }

    @Transactional
    public void complete(TransferId id) {
        Transfer transfer = transferPort.findById(id);
        transfer.complete(); // throws if already completed - impossible to double-complete
        transferPort.save(transfer);
    }
}
```

Key point: fields are now `private`/`final` where possible, mutation only happens through methods that check invariants, and there is no code path (anywhere in the codebase) that can construct or mutate a `Transfer` into an invalid state — this is what "rich domain model" means as opposed to the anemic domain model anti-pattern.
</details>

---

### Exercise 3 (Senior) — Split a monolithic model into bounded contexts

A single `User` entity currently has: `email`, `passwordHash`, `firstName`, `lastName`, `cardNumber`, `cardExpiryDate`, `walletBalance`, `kycStatus`, `notificationPreferences`. Propose a bounded-context split (name the contexts, list which fields/behavior belong to each), and explain how they'd communicate.

<details>
<summary>Solution</summary>

Proposed bounded contexts:

- **Identity context**: `email`, `passwordHash`, authentication/login behavior. Owns the concept of "user" as a login/security principal.
- **Customer Profile context**: `firstName`, `lastName`, `notificationPreferences`. Owns the concept of "customer" as a person with preferences/contact details.
- **Card Issuing context**: `cardNumber`, `cardExpiryDate`, and card lifecycle behavior (issue, freeze, replace). Owns the concept of "cardholder".
- **Wallet/Ledger context**: `walletBalance` (derived from the ledger), transfer/debit/credit behavior. Owns the concept of "account".
- **Compliance/KYC context**: `kycStatus` and verification workflow. Owns the concept of "verifiable subject".

Each context has **its own model of "the user"**, scoped to what it actually needs — Card Issuing doesn't need to know about `notificationPreferences`, and Identity doesn't need to know about `walletBalance`. This is exactly the point of Bounded Contexts: the same real-world person is modeled differently and independently in each context, rather than forcing one giant shared `User` entity that every team must coordinate changes to.

**Communication between contexts**:
- A shared, minimal identifier (`userId`/`customerId`) is the only thing referenced across contexts — never share the full entity or database table.
- Domain/integration events for cross-context reactions, e.g. Identity publishes `UserRegisteredEvent` → Customer Profile creates its own profile record, Wallet creates an account, Compliance starts a KYC check — each consumer builds its own local representation from the event (see [11-event-driven-design.md](./11-event-driven-design.md)).
- If one context needs *read* access to another's data synchronously (e.g. Card Issuing needs to check KYC status before issuing a card), it calls Compliance's API — never queries Compliance's database directly. If Compliance's model has quirks/legacy shape, Card Issuing wraps that call behind an Anti-Corruption Layer/adapter so its own domain model stays clean.
</details>

---

## Leveled Interview Q&A

### Basic Level

**Q1: What is an Entity in DDD?**
An object with a persistent identity that survives across state changes; compared by identity (ID), not by attribute values.

**Q2: What is a Value Object?**
An immutable object defined entirely by its attributes, with no identity; two value objects with the same attributes are considered equal/interchangeable.

**Q3: What is a Repository in DDD?**
An abstraction that provides collection-like access to aggregates, hiding the persistence details (SQL, ORM, etc.) behind an interface owned by the domain layer.

**Q4: What is a Bounded Context?**
An explicit boundary within which a domain model has one consistent, unambiguous meaning; the same term can mean different things in different bounded contexts.

**Q5: What is a Domain Event?**
Something meaningful that happened in the domain (e.g. `TransferCompletedEvent`) that other parts of the system (same context or other contexts) may want to react to.

---

### Mid Level

**Q1: What is an Aggregate and why does it need a single Aggregate Root?**
A cluster of entities/value objects treated as one consistency unit; only the Aggregate Root is accessible from outside, and all invariants spanning the aggregate's members are enforced through it — this guarantees no external code can put the aggregate into an inconsistent state by mutating an internal part directly.

**Q2: What's the "anemic domain model" anti-pattern, and why is it discouraged?**
A design where domain objects are just data bags (public getters/setters) with all behavior/validation living in external service classes. It's discouraged because business rules end up scattered across services (easy to bypass or duplicate) instead of being co-located with, and enforced by, the data they protect.

**Q3: What's the difference between a Domain Service and a method on an Entity/Aggregate?**
A method on an Entity/Aggregate models behavior that naturally belongs to that one object's own state and invariants. A Domain Service models an operation that doesn't fit naturally on a single entity — typically because it coordinates across multiple aggregates or needs external, stateless computation (e.g. `FxConversionService` converting between two `Money` value objects of different currencies).

**Q4: How does an Anti-Corruption Layer relate to the `infrastructure/mapper` package in a hexagonal architecture?**
The ACL is the concept: a translation layer preventing an external/legacy model's shape or quirks from leaking into your clean domain model. The `infrastructure/[tech]/mapper` package is the concrete implementation of that concept — it converts external DTOs/entities into `domain.model` objects (and back), so the domain layer never depends on infrastructure-specific representations.

**Q5: Why should aggregates reference other aggregates only by ID, not by direct object reference?**
To keep aggregate boundaries (and therefore transaction/consistency boundaries) small and independent — if `Transfer` held a direct reference to the full `Account` object graph, saving a `Transfer` could accidentally cascade into loading/locking/modifying the whole `Account` aggregate, defeating the purpose of having separate aggregates with independent consistency and scaling characteristics.

---

### Senior Level

**Q1: How do you decide where to draw aggregate boundaries in a new domain model?**
Start from the true business invariants that must be enforced *atomically* (in the same transaction) — group only what must change together consistently into one aggregate. Everything else should be a separate aggregate referenced by ID, coordinated via eventual consistency (domain events) rather than being pulled into the same transactional boundary. The guiding heuristic (Vaughn Vernon): design small aggregates; large aggregates cause lock contention and don't scale.

**Q2: How would you handle an invariant that spans two aggregates (e.g. "total balance across a user's accounts cannot exceed a fraud threshold")?**
Don't force both accounts into one aggregate just to enforce this. Instead, accept eventual consistency: each account aggregate enforces its own local invariants, and a domain/integration event triggered on each balance change feeds a separate process (e.g. a Fraud Monitoring service) that evaluates the cross-account rule asynchronously and takes compensating action (freeze, alert) if violated. This is the standard DDD answer to invariants that don't map cleanly to a single aggregate — most real "global" invariants in distributed systems are enforced eventually, not atomically.

**Q3: What's the relationship between Bounded Contexts and microservice boundaries — are they always 1:1?**
Not necessarily always 1:1, but they should align: a microservice should never span multiple bounded contexts's data ownership (that recreates coupling at the infrastructure level even if the code is "split"), and a single bounded context is sometimes implemented as more than one microservice for operational reasons (e.g. splitting a command-side and query-side service under CQRS, see [11-event-driven-design.md](./11-event-driven-design.md)) while still conceptually being one context with one canonical model.

**Q4: How do you evolve a Shared Kernel (e.g. a common `Money` type) across teams without recreating tight coupling?**
Treat the shared kernel as a small, deliberately-versioned, rarely-changing library (matches the repo's own `common/` module convention) — changes require cross-team review/agreement since both sides depend on it. Keep the shared kernel intentionally minimal (pure value objects, no business logic specific to one context) precisely to minimize the surface area that needs coordinated change; anything that starts accumulating context-specific behavior is a signal it should be pulled out of the shared kernel and duplicated/adapted per context instead.

**Q5: A legacy system exposes account data in a shape that doesn't match your clean domain model (e.g. mixed concerns, denormalized fields, inconsistent naming). How do you integrate without letting that leak into your domain?**
Build an explicit Anti-Corruption Layer: an adapter + mapper pair in the infrastructure layer that talks to the legacy system's actual shape and translates it into your own `domain.model` objects before anything reaches the domain/service layer. The domain layer should never know the legacy system exists — if the legacy system is replaced later, only the adapter/mapper changes, the domain and application logic remain untouched. This is a direct, practical justification for the hexagonal architecture's port/adapter separation already used across this codebase.

---

## Sources

See [10-domain-driven-design.md](./10-domain-driven-design.md) for the full reading list (Evans, Vernon, Fowler).
