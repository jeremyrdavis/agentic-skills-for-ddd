# Canonical Example — Repository, Mirror, and Use-Case-Optimal Query (Java)

Load this file when implementing a repository or adding a non-trivial query in
Java. For placement and review decisions, the Core Rules and Quick Reference
tables in SKILL.md are sufficient on their own. Comments cite rule numbers.

## 1. The interface — `<bc>/domain`, ubiquitous language only

```java
package com.example.orders.domain;

import java.util.Optional;

public interface OrderRepository {
    Optional<Order> findById(OrderId id);                    // aggregate out (Rule 2)
    void persist(Order order);                                // whole aggregate, upsert (Rule 6)
    void remove(OrderId id);
    OrderPage findRecent(int page, int size);                 // use-case-optimal VO (Rules 9, 10)
}
```

## 2. The purpose-built query VOs — `<bc>/domain`, per `ddd-value-objects`

Domain-specific, immutable, equality by value — explicitly **not** DTOs (Rule 9).

```java
package com.example.orders.domain;

public record OrderSummary(OrderId id, CustomerId customerId,
                           Money total, Order.Status status) {}

public record OrderPage(java.util.List<OrderSummary> items,
                        long totalCount, int page, int size) {
    public OrderPage { items = java.util.List.copyOf(items); }   // defensive immutability
}
```

## 3. The implementation — `<bc>/infrastructure`, same name (Rule 3)

```java
package com.example.orders.infrastructure;

import com.example.orders.domain.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.persistence.EntityManager;
import java.util.Optional;

@ApplicationScoped
// Same simple name as the domain interface — the package is the distinguisher.
// Java can't import a type with its own name, so the implements clause is
// fully qualified. No technology name in the class name (Rule 3).
public class OrderRepository implements com.example.orders.domain.OrderRepository {

    private final EntityManager em;

    OrderRepository(EntityManager em) { this.em = em; }

    @Override
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(em.find(OrderRecord.class, id.value()))
                .map(OrderRecord::toDomain);                 // rehydrate inside (Rule 5)
    }

    @Override
    public void persist(Order order) {
        em.merge(OrderRecord.fromDomain(order));             // whole aggregate (Rule 6)
        // @Version on OrderRecord does the compare-and-set (Rule 7):
        // a stale write raises OptimisticLockException.
    }

    @Override
    public OrderPage findRecent(int page, int size) {        // Vernon query (Rules 9, 10)
        var rows = em.createQuery(
                "select r from OrderRecord r order by r.placedAt desc", OrderRecord.class)
                .setFirstResult(page * size).setMaxResults(size)
                .getResultList();
        long total = em.createQuery("select count(r) from OrderRecord r", Long.class)
                .getSingleResult();
        var items = rows.stream().map(OrderRecord::toSummary).toList();
        return new OrderPage(items, total, page, size);
    }
}
```

The mirror `OrderRecord` (`@Entity`, `@Version`, column mappings, `toDomain` /
`fromDomain` / `toSummary`) lives in this package, per `ddd-aggregates`' mirror
table — nothing outside the implementation touches it (Rule 4).

**Conflict surfacing (Rule 7):** an exception mapper (or a small try/catch in
the implementation) converts `OptimisticLockException` into the domain-neutral
`ConcurrencyConflictException`; the api layer maps that to **409 Conflict**.
The repository never retries — the use case decides what a conflict means.

## 4. An atomic operation — from the Positive Thoughts spec (Rule 8)

`thumbsUp` has no invariant, so load-mutate-persist would be a lost-update
hazard. The interface gets an intention-revealing method; the implementation is
one atomic UPDATE:

```java
// thoughts/domain — on ThoughtRepository:
void incrementThumbsUp(ThoughtId id);

// thoughts/infrastructure — in the implementation:
@Override
public void incrementThumbsUp(ThoughtId id) {
    em.createQuery("update ThoughtRecord t set t.thumbsUp = t.thumbsUp + 1 where t.id = :id")
      .setParameter("id", id.value())
      .executeUpdate();                                      // atomic under concurrency
}
```

## What this demonstrates

- **Interface in domain vocabulary (Rule 2):** typed IDs in, aggregates and
  domain VOs out; zero framework imports in `<bc>/domain`.
- **Same-name implementation (Rule 3):** package is the distinguisher;
  fully-qualified `implements` is Java's spelling of it.
- **Translation is the whole job (Rule 5):** `toDomain` calls
  `Order.rehydrate(...)`; VO constructors re-validate on the way in.
- **Locking (Rule 7):** the mirror's `@Version` does the compare-and-set; the
  boundary surfaces a domain-neutral conflict → 409.
- **Atomic op (Rule 8):** one UPDATE behind a named method.
- **Vernon query (Rules 9, 10):** `OrderPage`/`OrderSummary` are immutable
  domain records — not DTOs; the application service maps them onward if the
  api needs a different shape.

## Java idioms

- `@Version` on the mirror gives compare-and-set for free; keep the aggregate's
  own `version` a plain `long` (per `ddd-aggregates` Rule 10).
- JPQL constructor expressions (`select new ...OrderSummary(...)`) can build
  the query VO directly when the mapping is trivial — same rule, fewer lines.
- Migrations: Flyway (or Liquibase), versioned scripts — mechanics in the
  per-stack persistence skill.
- Repository tests split: contract tests against an in-memory fake for callers;
  implementation tests against a real database (test containers) for the
  mapping, the lock, and the atomic UPDATE.
