# Canonical Example — `Order` Aggregate Root (Java)

Load this file when creating or substantially reshaping an aggregate in Java. For
placement and review decisions, the Core Rules and Quick Reference tables in
SKILL.md are sufficient on their own.

The `Order` aggregate lives in `com.example.orders.domain`. **Plain object — zero
`jakarta.persistence` imports, no framework base class.** Exercises every Core
Rule; comments cite rule numbers.

## The aggregate

```java
package com.example.orders.domain;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Currency;
import java.util.List;
import java.util.UUID;

public class Order {

    private final UUID id;                             // identity (Rule 9)
    private long version;                              // optimistic-lock cursor (Rule 10) — plain long
    private final CustomerId customerId;               // reference by ID, not by object (Rules 8, 9)
    private final List<OrderLine> lines;
    private Status status;
    private Money total;                               // value object as a plain field (Rule 11)
    private final List<DomainEvent> pendingEvents = new ArrayList<>();

    public enum Status { PLACED, FULFILLED, CANCELLED }

    /** Private — used only by the factory and rehydrate (Rule 4). */
    private Order(UUID id, CustomerId customerId, List<OrderLine> lines,
                  Status status, Money total, long version) {
        this.id = id;
        this.customerId = customerId;
        this.lines = new ArrayList<>(lines);
        this.status = status;
        this.total = total;
        this.version = version;
    }

    /**
     * Named factory — the only way domain code brings a new Order into existence.
     * Enforces creation invariants (Rule 5) and raises OrderPlacedEvent (Rule 12).
     */
    public static Order place(OrderId id, CustomerId customerId, List<OrderLine> lines) {
        if (lines == null || lines.isEmpty())
            throw new IllegalArgumentException("an order must have at least one line");
        Currency currency = lines.get(0).unitPrice().currency();
        if (!lines.stream().allMatch(l -> l.unitPrice().currency().equals(currency)))
            throw new IllegalArgumentException("all lines must share a currency");

        Money total = lines.stream()
                .map(OrderLine::lineTotal)
                .reduce(new Money(BigDecimal.ZERO, currency), Money::plus);

        Order order = new Order(id.value(), customerId, lines, Status.PLACED, total, 0L);
        order.pendingEvents.add(new OrderPlacedEvent(id, customerId));   // Rule 12
        return order;
    }

    /**
     * Reconstruction path for the persistence layer (Rule 6). Trusts inputs that
     * already passed validation when first persisted; does NOT re-emit
     * OrderPlacedEvent. Only the repository implementation in
     * <bc>/infrastructure calls this (covered by ddd-persistence).
     */
    public static Order rehydrate(UUID id, CustomerId customerId, List<OrderLine> lines,
                                  Status status, Money total, long version) {
        return new Order(id, customerId, lines, status, total, version);
    }

    /** Transition method — enforces the state-machine invariant (Rule 7). */
    public void cancel() {
        if (status == Status.FULFILLED)
            throw new OrderCancellationNotAllowedException(new OrderId(id));
        this.status = Status.CANCELLED;
        this.pendingEvents.add(new OrderCancelledEvent(new OrderId(id)));
    }

    public void markFulfilled() {
        if (status != Status.PLACED)
            throw new IllegalStateException("only PLACED orders can be fulfilled");
        this.status = Status.FULFILLED;
        this.pendingEvents.add(new OrderFulfilledEvent(new OrderId(id)));
    }

    // Read accessors — only what callers need (Rule 3).
    public OrderId id()            { return new OrderId(id); }
    public Status status()         { return status; }
    public Money total()           { return total; }
    public CustomerId customerId() { return customerId; }
    public List<OrderLine> lines() { return List.copyOf(lines); }   // defensive copy
    public long version()          { return version; }

    /** Drained by the application service after persist; published after commit (Rule 12). */
    public List<DomainEvent> pullPendingEvents() {
        List<DomainEvent> drained = List.copyOf(pendingEvents);
        pendingEvents.clear();
        return drained;
    }
}
```

## Companion types

Also plain, also persistence-free (full coverage in `ddd-value-objects` and
`ddd-domain-events`):

```java
public record OrderId(UUID value) {
    public OrderId { java.util.Objects.requireNonNull(value); }
}

public record CustomerId(UUID value) {
    public CustomerId { java.util.Objects.requireNonNull(value); }
}

public record OrderLine(String productSku, int quantity, Money unitPrice) {
    public OrderLine {
        if (quantity <= 0) throw new IllegalArgumentException("quantity must be positive");
    }
    public Money lineTotal() { return unitPrice.times(quantity); }
}

// Events: past tense, *Event suffix (per ddd-foundations). Every event carries
// eventId + occurredAt, assigned once when raised (ddd-domain-events Rule 3);
// the short constructor keeps the raising sites to the domain facts.
public interface DomainEvent {
    UUID eventId();
    Instant occurredAt();          // when it happened — not an ordering key; order on version if a consumer needs order
}

public record OrderPlacedEvent(UUID eventId, Instant occurredAt,
                               OrderId orderId, CustomerId customerId) implements DomainEvent {
    public OrderPlacedEvent(OrderId orderId, CustomerId customerId) {
        this(UUID.randomUUID(), Instant.now(), orderId, customerId);
    }
}

public record OrderCancelledEvent(UUID eventId, Instant occurredAt, OrderId orderId) implements DomainEvent {
    public OrderCancelledEvent(OrderId orderId) { this(UUID.randomUUID(), Instant.now(), orderId); }
}

public record OrderFulfilledEvent(UUID eventId, Instant occurredAt, OrderId orderId) implements DomainEvent {
    public OrderFulfilledEvent(OrderId orderId) { this(UUID.randomUUID(), Instant.now(), orderId); }
}
```

## The persistence mirror (lives in `<bc>/infrastructure`)

The aggregate has no idea this class exists. Sketch only — full coverage in
`ddd-persistence`:

```java
// com.example.orders.infrastructure
@Entity
@Table(name = "orders")
public class OrderRecord {
    @Id public UUID id;
    @Version public Long version;                  // the framework's lock column (Rule 10)
    public UUID customerId;
    @Enumerated(EnumType.STRING) public Order.Status status;
    public BigDecimal totalAmount;                 // Money split into columns — repository's call
    public String totalCurrency;
    @ElementCollection public List<OrderLineRecord> lines;

    static OrderRecord fromDomain(Order order) { /* map fields */ }
    Order toDomain() {
        return Order.rehydrate(id, new CustomerId(customerId), mapLines(lines),
                status, new Money(totalAmount, Currency.getInstance(totalCurrency)), version);
    }
}
```

**Optimistic locking mechanics:** JPA's `@Version` on `OrderRecord` raises
`OptimisticLockException` on a stale write. The repository implementation lets it
propagate (or wraps it in a domain-neutral `ConcurrencyConflictException`); the
api layer's exception mapper turns it into **409 Conflict**.

## What this demonstrates

- **Plain object (Rule 1):** compiles with only the JDK and the BC's own domain types.
- **No setters, no public mutable fields (Rules 2, 3):** only `version`, `status`,
  `total`, and `pendingEvents` are mutable, and only domain methods change them.
- **Factory enforces creation invariants (Rules 4, 5):** empty lines and mixed
  currencies throw before the aggregate exists. No `validate()` callers might forget.
- **Rehydrate path (Rule 6):** the repository reconstructs via `rehydrate(...)`;
  no `OrderPlacedEvent` re-emitted; domain code never calls it.
- **Cross-aggregate reference by ID (Rule 8):** `CustomerId customerId`, never a
  `Customer` field.
- **Typed identity (Rule 9):** `OrderId`/`CustomerId` records at the API surface;
  raw `UUID` internally to keep field declarations uncluttered.
- **Plain `long` version (Rule 10):** the mirror carries `@Version`; the aggregate
  carries `private long version`. Same data, different layer.
- **Value object as plain field (Rule 11):** `Money total`, no `@Embedded`.
- **Events (Rule 12):** `*Event` suffix, past tense, raised inside, drained by the
  application service via `pullPendingEvents()`, published after commit. Each carries its id and time, assigned once at the raising site (`ddd-domain-events` Rule 3).

## Java idioms

- Value objects, IDs, and events are `record`s; the aggregate is a class with a
  `private` constructor — the language enforces Rule 4 at compile time.
- Invariant unit tests are pure JVM: no framework bootstrap, no test containers —
  `Order.place(...)` and `order.cancel()` are directly testable.
- Domain purity: no `jakarta.persistence.*`, `io.quarkus.*`, or
  `org.springframework.*` imports under `<bc>.domain` — enforced by the ArchUnit
  rules from `ddd-foundations`.
