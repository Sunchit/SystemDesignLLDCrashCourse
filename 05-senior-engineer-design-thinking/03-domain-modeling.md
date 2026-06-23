# Domain Modeling for Senior Engineers

## Introduction

Domain modeling is the practice of creating an explicit, code-level representation of the business problem you are solving. It is distinct from database design, API design, or system architecture — though it influences all three. A domain model captures the vocabulary, rules, relationships, and behaviors of the business domain in a form that is expressed directly in production code.

Senior engineers understand that the majority of complexity in long-lived software systems does not come from infrastructure concerns or framework choices. It comes from an incomplete or inaccurate understanding of the business domain. Domain modeling is the discipline that addresses this root cause.

This document covers domain modeling in depth: the concepts, the patterns, the Java implementations, the tradeoffs, and how to reason about it in both production and interview settings.

---

## 1. What Is a Domain Model?

### Domain Model vs Data Model vs Database Schema

These three artifacts serve different purposes and exist at different levels of abstraction.

A **database schema** is a physical artifact. It defines tables, columns, indexes, foreign keys, and constraints as they exist in a relational or document store. It is shaped by storage efficiency, query patterns, normalization rules, and the capabilities of the database engine. A `VARCHAR(255)` constraint, a junction table for many-to-many relationships, or a nullable column for optional data — these are database concerns, not business concerns.

A **data model** is a structural representation of data entities and their relationships. It is technology-neutral but still primarily concerned with how data is organized and related. Entity-Relationship diagrams are a data modeling artifact. A data model tells you what data exists and how it is connected but says nothing about the rules that govern it or the operations it supports.

A **domain model** is a behavioral and conceptual model. It represents the business concepts, the invariants that must hold, the operations that make sense, and the language that domain experts use. A domain model asks: What is an Order? What rules must an Order always satisfy? What can you do with an Order? Who is responsible for which decisions? The domain model is defined in code — in classes, interfaces, methods, and types — not in a diagram or a schema.

Consider an `Order` as an example:

| Concern | Representation |
|---|---|
| Database schema | `orders` table with `id`, `customer_id`, `total_amount`, `status` columns |
| Data model | Order entity with relationships to Customer and OrderItem |
| Domain model | `Order` class with `addItem()`, `confirm()`, `cancel()` methods; invariants like "a confirmed order must have at least one item"; value objects like `Money` for `totalAmount` |

### Why the Domain Model Is the Most Important Design Artifact

The domain model drives everything else. Once you have a clear domain model:

- The database schema follows naturally (with appropriate mapping concerns handled at the infrastructure layer).
- The API contract reflects domain operations rather than database CRUD operations.
- The service boundaries emerge from aggregate boundaries and bounded contexts.
- Business rules have an obvious home — they live inside the domain model, not scattered across controllers, services, and stored procedures.

Systems built without a coherent domain model accumulate what Martin Fowler calls a Big Ball of Mud. Business logic migrates into the database (stored procedures), into the UI (JavaScript validation), into utility classes with no clear ownership, and into every layer simultaneously. This makes the system brittle, expensive to change, and nearly impossible for new engineers to understand.

### The Relationship Between Domain Model and Business Rules

Business rules are the constraints, policies, and behaviors that define what the business does. They are the most valuable and most volatile part of a system. Technology changes; business rules evolve with the market.

A well-designed domain model makes business rules **explicit, locatable, and enforceable**. When the rule "an order cannot be cancelled after it has been shipped" lives in the `Order` class itself, there is one place to find it, one place to change it, and one place to test it. When that rule lives nowhere in particular — enforced inconsistently across a controller, a service, and a database trigger — changing it becomes archaeology.

---

## 2. DDD Building Blocks Applied to LLD

Domain-Driven Design (DDD), articulated by Eric Evans in 2003, provides a vocabulary and a set of patterns for building domain models. These building blocks apply directly to low-level design.

### Entities

An **entity** is an object defined by its identity, not by its attributes. Two entities with the same attribute values but different identities are distinct objects. An entity has a lifecycle: it is created, it changes over time, and it may be deleted.

Key characteristics:
- Has a unique identifier that persists over the lifetime of the object.
- Is mutable — its state changes as the domain evolves.
- Equality is based on identity, not attribute values.
- Has a lifecycle with defined states and transitions.

```java
import java.util.Objects;
import java.util.UUID;

public class Customer {

    private final CustomerId id;
    private String name;
    private EmailAddress email;
    private CustomerStatus status;

    public Customer(CustomerId id, String name, EmailAddress email) {
        if (id == null) throw new IllegalArgumentException("Customer ID must not be null");
        if (name == null || name.isBlank()) throw new IllegalArgumentException("Name must not be blank");
        if (email == null) throw new IllegalArgumentException("Email must not be null");

        this.id = id;
        this.name = name;
        this.email = email;
        this.status = CustomerStatus.ACTIVE;
    }

    public void updateEmail(EmailAddress newEmail) {
        if (newEmail == null) throw new IllegalArgumentException("Email must not be null");
        if (this.status == CustomerStatus.SUSPENDED) {
            throw new IllegalStateException("Cannot update email of a suspended customer");
        }
        this.email = newEmail;
    }

    public void suspend(String reason) {
        if (reason == null || reason.isBlank()) throw new IllegalArgumentException("Suspension reason required");
        if (this.status == CustomerStatus.SUSPENDED) {
            throw new IllegalStateException("Customer is already suspended");
        }
        this.status = CustomerStatus.SUSPENDED;
    }

    public void reactivate() {
        if (this.status != CustomerStatus.SUSPENDED) {
            throw new IllegalStateException("Only suspended customers can be reactivated");
        }
        this.status = CustomerStatus.ACTIVE;
    }

    public CustomerId getId() { return id; }
    public String getName() { return name; }
    public EmailAddress getEmail() { return email; }
    public CustomerStatus getStatus() { return status; }

    // Identity-based equality — two Customer objects are equal if they have the same ID
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Customer)) return false;
        Customer other = (Customer) obj;
        return Objects.equals(this.id, other.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

    @Override
    public String toString() {
        return "Customer{id=" + id + ", name='" + name + "', status=" + status + "}";
    }
}

public class CustomerId {
    private final UUID value;

    private CustomerId(UUID value) {
        this.value = value;
    }

    public static CustomerId generate() {
        return new CustomerId(UUID.randomUUID());
    }

    public static CustomerId of(String value) {
        return new CustomerId(UUID.fromString(value));
    }

    public String getValue() { return value.toString(); }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof CustomerId)) return false;
        return Objects.equals(this.value, ((CustomerId) obj).value);
    }

    @Override
    public int hashCode() { return Objects.hash(value); }

    @Override
    public String toString() { return value.toString(); }
}

public enum CustomerStatus {
    ACTIVE, SUSPENDED, CLOSED
}
```

### Value Objects

A **value object** is an object defined entirely by its attributes. Two value objects with the same attributes are identical and interchangeable. Value objects have no identity and no lifecycle. They are immutable.

Key characteristics:
- No identity — equality is based on all attribute values.
- Immutable — once created, its state cannot change. Operations return new instances.
- Self-validating — the constructor enforces all constraints. An invalid value object cannot be created.
- Conceptually whole — a value object represents a single concept, even if it has multiple fields.

Value objects eliminate primitive obsession — the antipattern of using raw `String`, `int`, and `double` for domain concepts that have rules and meaning.

```java
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Currency;
import java.util.Objects;
import java.util.regex.Pattern;

// Money: the canonical value object example
public final class Money {

    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        if (amount == null) throw new IllegalArgumentException("Amount must not be null");
        if (currency == null) throw new IllegalArgumentException("Currency must not be null");
        if (amount.compareTo(BigDecimal.ZERO) < 0) throw new IllegalArgumentException("Amount must not be negative");

        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    public static Money of(double amount, String currencyCode) {
        return new Money(BigDecimal.valueOf(amount), Currency.getInstance(currencyCode));
    }

    public static Money zero(String currencyCode) {
        return new Money(BigDecimal.ZERO, Currency.getInstance(currencyCode));
    }

    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        assertSameCurrency(other);
        BigDecimal result = this.amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Subtraction would result in negative money");
        }
        return new Money(result, this.currency);
    }

    public Money multiply(int factor) {
        if (factor < 0) throw new IllegalArgumentException("Multiplier must not be negative");
        return new Money(this.amount.multiply(BigDecimal.valueOf(factor)), this.currency);
    }

    public Money multiply(BigDecimal factor) {
        if (factor.compareTo(BigDecimal.ZERO) < 0) throw new IllegalArgumentException("Factor must not be negative");
        return new Money(this.amount.multiply(factor).setScale(2, RoundingMode.HALF_UP), this.currency);
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return this.amount.compareTo(other.amount) > 0;
    }

    public boolean isZero() {
        return this.amount.compareTo(BigDecimal.ZERO) == 0;
    }

    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Currency mismatch: " + this.currency + " vs " + other.currency
            );
        }
    }

    public BigDecimal getAmount() { return amount; }
    public Currency getCurrency() { return currency; }

    // Value-based equality — two Money instances are equal if amount and currency match
    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof Money)) return false;
        Money other = (Money) obj;
        return this.amount.compareTo(other.amount) == 0
            && Objects.equals(this.currency, other.currency);
    }

    @Override
    public int hashCode() { return Objects.hash(amount.stripTrailingZeros(), currency); }

    @Override
    public String toString() { return currency.getSymbol() + amount.toPlainString(); }
}

// EmailAddress: self-validating value object
public final class EmailAddress {

    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$");

    private final String value;

    public EmailAddress(String value) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("Email address must not be blank");
        }
        String normalized = value.trim().toLowerCase();
        if (!EMAIL_PATTERN.matcher(normalized).matches()) {
            throw new IllegalArgumentException("Invalid email address format: " + value);
        }
        this.value = normalized;
    }

    public String getValue() { return value; }

    public String getDomain() {
        return value.substring(value.indexOf('@') + 1);
    }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof EmailAddress)) return false;
        return Objects.equals(this.value, ((EmailAddress) obj).value);
    }

    @Override
    public int hashCode() { return Objects.hash(value); }

    @Override
    public String toString() { return value; }
}

// PhoneNumber: value object with normalization
public final class PhoneNumber {

    private final String countryCode;
    private final String number;

    public PhoneNumber(String countryCode, String number) {
        if (countryCode == null || !countryCode.matches("\\+\\d{1,3}")) {
            throw new IllegalArgumentException("Invalid country code: " + countryCode);
        }
        if (number == null) throw new IllegalArgumentException("Number must not be null");

        String normalized = number.replaceAll("[\\s\\-\\(\\)]", "");
        if (!normalized.matches("\\d{6,14}")) {
            throw new IllegalArgumentException("Invalid phone number: " + number);
        }
        this.countryCode = countryCode;
        this.number = normalized;
    }

    public String getFullNumber() { return countryCode + number; }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof PhoneNumber)) return false;
        PhoneNumber other = (PhoneNumber) obj;
        return Objects.equals(this.countryCode, other.countryCode)
            && Objects.equals(this.number, other.number);
    }

    @Override
    public int hashCode() { return Objects.hash(countryCode, number); }

    @Override
    public String toString() { return getFullNumber(); }
}
```

### Aggregates

An **aggregate** is a cluster of related entities and value objects that are treated as a single unit for the purpose of data changes. Every aggregate has an **aggregate root** — a single entity that serves as the gateway to the entire cluster.

Key rules:
- External objects may only hold references to the aggregate root, not to internal entities.
- All changes to the aggregate go through the aggregate root.
- The aggregate root is responsible for enforcing all invariants of the aggregate.
- The aggregate is the consistency boundary — all invariants within an aggregate must be satisfied after every operation.
- Aggregates should be kept small. Large aggregates create contention and complexity.

```java
import java.time.Instant;
import java.util.*;

// OrderItem is an entity internal to the Order aggregate
public class OrderItem {

    private final OrderItemId id;
    private final ProductId productId;
    private final String productName;
    private int quantity;
    private final Money unitPrice;

    // Package-private constructor — only the Order aggregate root creates OrderItems
    OrderItem(OrderItemId id, ProductId productId, String productName, int quantity, Money unitPrice) {
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive");
        if (unitPrice == null) throw new IllegalArgumentException("Unit price must not be null");

        this.id = id;
        this.productId = productId;
        this.productName = productName;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    // Package-private — only accessible within the aggregate
    void increaseQuantity(int delta) {
        if (delta <= 0) throw new IllegalArgumentException("Delta must be positive");
        this.quantity += delta;
    }

    public Money getSubtotal() {
        return unitPrice.multiply(quantity);
    }

    public OrderItemId getId() { return id; }
    public ProductId getProductId() { return productId; }
    public String getProductName() { return productName; }
    public int getQuantity() { return quantity; }
    public Money getUnitPrice() { return unitPrice; }
}

// Order is the aggregate root — it owns the consistency boundary
public class Order {

    private static final int MAX_ITEMS = 50;
    private static final Money MINIMUM_ORDER_AMOUNT = Money.of(1.00, "USD");

    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private final Instant createdAt;
    private Instant confirmedAt;
    private Instant cancelledAt;
    private final List<DomainEvent> domainEvents;

    public Order(OrderId id, CustomerId customerId) {
        if (id == null) throw new IllegalArgumentException("Order ID must not be null");
        if (customerId == null) throw new IllegalArgumentException("Customer ID must not be null");

        this.id = id;
        this.customerId = customerId;
        this.items = new ArrayList<>();
        this.status = OrderStatus.DRAFT;
        this.createdAt = Instant.now();
        this.domainEvents = new ArrayList<>();
    }

    // The aggregate root controls all modifications to its internal entities
    public void addItem(ProductId productId, String productName, int quantity, Money unitPrice) {
        assertStatus(OrderStatus.DRAFT, "add items to");

        if (items.size() >= MAX_ITEMS) {
            throw new DomainException("Order cannot have more than " + MAX_ITEMS + " items");
        }

        // Check if item for this product already exists — if so, increase quantity
        Optional<OrderItem> existing = items.stream()
            .filter(item -> item.getProductId().equals(productId))
            .findFirst();

        if (existing.isPresent()) {
            existing.get().increaseQuantity(quantity);
        } else {
            OrderItem newItem = new OrderItem(
                OrderItemId.generate(), productId, productName, quantity, unitPrice
            );
            items.add(newItem);
        }
    }

    public void removeItem(ProductId productId) {
        assertStatus(OrderStatus.DRAFT, "remove items from");
        boolean removed = items.removeIf(item -> item.getProductId().equals(productId));
        if (!removed) {
            throw new DomainException("No item found for product: " + productId);
        }
    }

    // Confirms the order — enforces multiple invariants before state transition
    public void confirm() {
        assertStatus(OrderStatus.DRAFT, "confirm");

        if (items.isEmpty()) {
            throw new DomainException("Cannot confirm an order with no items");
        }

        Money total = calculateTotal();
        if (!total.isGreaterThan(MINIMUM_ORDER_AMOUNT) && !total.equals(MINIMUM_ORDER_AMOUNT)) {
            throw new DomainException(
                "Order total " + total + " is below minimum " + MINIMUM_ORDER_AMOUNT
            );
        }

        this.status = OrderStatus.CONFIRMED;
        this.confirmedAt = Instant.now();

        // Record domain event — what happened in the domain
        domainEvents.add(new OrderPlaced(this.id, this.customerId, total, Instant.now()));
    }

    public void startFulfillment() {
        assertStatus(OrderStatus.CONFIRMED, "start fulfillment of");
        this.status = OrderStatus.FULFILLING;
    }

    public void ship() {
        assertStatus(OrderStatus.FULFILLING, "ship");
        this.status = OrderStatus.SHIPPED;
    }

    public void deliver() {
        assertStatus(OrderStatus.SHIPPED, "deliver");
        this.status = OrderStatus.DELIVERED;
    }

    public void cancel(String reason) {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
            throw new DomainException(
                "Cannot cancel an order in status: " + status
            );
        }
        if (status == OrderStatus.CANCELLED) {
            throw new DomainException("Order is already cancelled");
        }
        this.status = OrderStatus.CANCELLED;
        this.cancelledAt = Instant.now();
        domainEvents.add(new OrderCancelled(this.id, reason, Instant.now()));
    }

    // Invariant: total is always computed from the items, not stored separately
    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.zero("USD"), Money::add);
    }

    // Consumers get an unmodifiable view — they cannot bypass the aggregate root
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }

    // Domain events are collected within the aggregate and dispatched by the application layer
    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(this.domainEvents);
        this.domainEvents.clear();
        return events;
    }

    private void assertStatus(OrderStatus required, String operation) {
        if (this.status != required) {
            throw new DomainException(
                "Cannot " + operation + " an order with status: " + this.status
                + ". Required: " + required
            );
        }
    }

    public OrderId getId() { return id; }
    public CustomerId getCustomerId() { return customerId; }
    public OrderStatus getStatus() { return status; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getConfirmedAt() { return confirmedAt; }
}
```

### Domain Services

A **domain service** encapsulates domain logic that does not naturally belong to any single entity or value object. Domain services are stateless. They operate on domain objects and express significant domain operations.

Common reasons to introduce a domain service:
- The operation involves multiple aggregates.
- The operation requires coordination that no single entity can express without knowing too much.
- The operation expresses a key domain concept (pricing, eligibility, transfer) that is not owned by a single object.

Domain services should not be confused with application services. Application services orchestrate use cases, manage transactions, and call repositories. Domain services express domain logic.

```java
// PricingService: prices an order based on customer tier and promotions
// Operates across Order and Customer — neither owns this logic naturally
public class PricingService {

    private final DiscountPolicy discountPolicy;

    public PricingService(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }

    public PricedOrder calculatePrice(Order order, Customer customer) {
        Money baseTotal = order.calculateTotal();
        Discount discount = discountPolicy.determineDiscount(customer, order);
        Money discountAmount = baseTotal.multiply(discount.getPercentage());
        Money finalTotal = baseTotal.subtract(discountAmount);

        return new PricedOrder(order.getId(), baseTotal, discountAmount, finalTotal, discount.getReason());
    }
}

// TransferService: transfers money between two accounts
// The operation involves two Account aggregates — neither should know about the other
public class TransferService {

    public TransferResult transfer(Account source, Account destination, Money amount) {
        if (source.getId().equals(destination.getId())) {
            throw new DomainException("Cannot transfer money to the same account");
        }
        if (!source.getCurrency().equals(destination.getCurrency())) {
            throw new DomainException("Currency mismatch between accounts");
        }

        source.debit(amount);
        destination.credit(amount);

        return new TransferResult(
            source.getId(),
            destination.getId(),
            amount,
            Instant.now()
        );
    }
}

// InventoryAllocationService: reserves inventory for an order
// Crosses Order and Inventory aggregates — belongs in neither
public class InventoryAllocationService {

    public AllocationResult allocate(Order order, Inventory inventory) {
        List<AllocationFailure> failures = new ArrayList<>();

        for (OrderItem item : order.getItems()) {
            try {
                inventory.reserve(item.getProductId(), item.getQuantity());
            } catch (InsufficientInventoryException e) {
                failures.add(new AllocationFailure(item.getProductId(), e.getMessage()));
            }
        }

        if (!failures.isEmpty()) {
            // Roll back any successful reservations
            inventory.releaseAll(order.getId());
            return AllocationResult.failed(failures);
        }

        return AllocationResult.success(order.getId());
    }
}
```

### Repositories

A **repository** provides a collection-like interface for accessing aggregates. From the domain's perspective, a repository is a collection — you add things to it, find things in it, and remove things from it. The domain layer defines the repository interface; the infrastructure layer provides the implementation.

Critical distinction: a repository is NOT a data access object (DAO). A DAO exposes CRUD operations on database tables. A repository exposes operations meaningful to the domain — it deals in aggregates, not in rows.

```java
// Repository interface — lives in the domain layer
// Note: deals in Order aggregates, not in database rows
public interface OrderRepository {

    void save(Order order);

    Optional<Order> findById(OrderId id);

    List<Order> findByCustomerId(CustomerId customerId);

    List<Order> findByStatus(OrderStatus status);

    // Domain-language query — not a database query
    List<Order> findPendingFulfillmentOlderThan(Instant cutoff);

    void delete(OrderId id);
}

// JPA implementation — lives in the infrastructure layer
// The domain layer never depends on this class
@Repository
public class JpaOrderRepository implements OrderRepository {

    private final OrderJpaRepository jpaRepository;
    private final OrderMapper mapper;

    public JpaOrderRepository(OrderJpaRepository jpaRepository, OrderMapper mapper) {
        this.jpaRepository = jpaRepository;
        this.mapper = mapper;
    }

    @Override
    public void save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        jpaRepository.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.getValue())
            .map(mapper::toDomain);
    }

    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return jpaRepository.findByCustomerId(customerId.getValue())
            .stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
    }

    @Override
    public List<Order> findByStatus(OrderStatus status) {
        return jpaRepository.findByStatus(status.name())
            .stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
    }

    @Override
    public List<Order> findPendingFulfillmentOlderThan(Instant cutoff) {
        return jpaRepository.findByStatusAndConfirmedAtBefore(
            OrderStatus.CONFIRMED.name(), cutoff
        )
        .stream()
        .map(mapper::toDomain)
        .collect(Collectors.toList());
    }

    @Override
    public void delete(OrderId id) {
        jpaRepository.deleteById(id.getValue());
    }
}
```

### Domain Events

A **domain event** is a record of something that happened in the domain. Events are immutable facts — they describe something that has already occurred. They enable decoupling between different parts of the system and make it possible to react to domain changes without direct coupling.

```java
// Base interface for all domain events
public interface DomainEvent {
    String getEventId();
    Instant getOccurredAt();
    String getEventType();
}

// Abstract base with common fields
public abstract class BaseDomainEvent implements DomainEvent {

    private final String eventId;
    private final Instant occurredAt;

    protected BaseDomainEvent() {
        this.eventId = UUID.randomUUID().toString();
        this.occurredAt = Instant.now();
    }

    @Override
    public String getEventId() { return eventId; }

    @Override
    public Instant getOccurredAt() { return occurredAt; }
}

// OrderPlaced — records the fact that an order was placed
public final class OrderPlaced extends BaseDomainEvent {

    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money orderTotal;
    private final Instant placedAt;

    public OrderPlaced(OrderId orderId, CustomerId customerId, Money orderTotal, Instant placedAt) {
        super();
        this.orderId = orderId;
        this.customerId = customerId;
        this.orderTotal = orderTotal;
        this.placedAt = placedAt;
    }

    @Override
    public String getEventType() { return "order.placed"; }

    public OrderId getOrderId() { return orderId; }
    public CustomerId getCustomerId() { return customerId; }
    public Money getOrderTotal() { return orderTotal; }
    public Instant getPlacedAt() { return placedAt; }
}

// PaymentProcessed — records that payment was processed for an order
public final class PaymentProcessed extends BaseDomainEvent {

    private final PaymentId paymentId;
    private final OrderId orderId;
    private final Money amount;
    private final PaymentMethod paymentMethod;

    public PaymentProcessed(PaymentId paymentId, OrderId orderId,
                            Money amount, PaymentMethod paymentMethod) {
        super();
        this.paymentId = paymentId;
        this.orderId = orderId;
        this.amount = amount;
        this.paymentMethod = paymentMethod;
    }

    @Override
    public String getEventType() { return "payment.processed"; }

    public PaymentId getPaymentId() { return paymentId; }
    public OrderId getOrderId() { return orderId; }
    public Money getAmount() { return amount; }
    public PaymentMethod getPaymentMethod() { return paymentMethod; }
}

// OrderCancelled — records that an order was cancelled with a reason
public final class OrderCancelled extends BaseDomainEvent {

    private final OrderId orderId;
    private final String reason;
    private final Instant cancelledAt;

    public OrderCancelled(OrderId orderId, String reason, Instant cancelledAt) {
        super();
        this.orderId = orderId;
        this.reason = reason;
        this.cancelledAt = cancelledAt;
    }

    @Override
    public String getEventType() { return "order.cancelled"; }

    public OrderId getOrderId() { return orderId; }
    public String getReason() { return reason; }
    public Instant getCancelledAt() { return cancelledAt; }
}
```

---

## 3. Anemic Domain Model vs Rich Domain Model

### What Anemic Means

An anemic domain model is one where domain objects (entities) contain only data — getters, setters, and no business logic. All of the business logic is pushed into service classes. The entity is a data container; the service classes do all the work.

Martin Fowler identified this as an anti-pattern in 2003. It remains one of the most common anti-patterns in enterprise Java development.

### Why Anemic Models Are an Anti-Pattern in Complex Domains

In complex domains, anemic models produce several concrete problems:

**Logic fragmentation**: Business rules that belong together are scattered across multiple service classes. To understand what happens when an order is confirmed, you must read `OrderService`, `OrderValidationService`, `PricingService`, `OrderEventPublisher`, and possibly others. The knowledge is distributed and implicit.

**Invariant enforcement failure**: When entities are just data containers, invariants cannot be enforced at the entity level. The rule "a confirmed order must have at least one item" either appears in every service that calls `confirm()`, or it is verified nowhere. Bugs occur when new services are added and the invariant check is forgotten.

**Open for modification everywhere**: Any code with a reference to an `Order` object can call setters directly and put it in an invalid state. The aggregate boundary offers no protection.

**Testing complexity**: Logic in services has external dependencies. Testing a business rule requires constructing the entire service graph. Logic in entities can be tested in isolation.

### When Anemic Models Are Acceptable

Anemic models are acceptable in CRUD-heavy systems where:
- The domain has no complex invariants — operations are simple create, read, update, delete.
- Data is the product — the system is primarily a data store with validation.
- The logic is simple enough that fragmentation is not a problem.
- The team is small and coordination overhead is low.

Admin panels, configuration management services, simple reporting APIs — these are often fine with an anemic model. The cost of richness (discipline, design effort) is not justified.

### Full Java Example: Anemic vs Rich

```java
// ===== ANEMIC ORDER (ANTI-PATTERN) =====

// The entity: a data container with no behavior
public class AnemicOrder {

    private String id;
    private String customerId;
    private List<AnemicOrderItem> items = new ArrayList<>();
    private String status;
    private BigDecimal totalAmount;
    private Instant createdAt;
    private Instant confirmedAt;

    // Getter/setter pairs for everything — no protection, no invariants
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }

    public List<AnemicOrderItem> getItems() { return items; }
    public void setItems(List<AnemicOrderItem> items) { this.items = items; }

    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }

    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }

    public Instant getCreatedAt() { return createdAt; }
    public void setCreatedAt(Instant createdAt) { this.createdAt = createdAt; }

    public Instant getConfirmedAt() { return confirmedAt; }
    public void setConfirmedAt(Instant confirmedAt) { this.confirmedAt = confirmedAt; }
}

// Business logic lives in the service — fragmented, hard to find, hard to test in isolation
@Service
public class AnemicOrderService {

    private final OrderRepository repository;
    private final EventPublisher eventPublisher;

    public void confirmOrder(String orderId) {
        AnemicOrder order = repository.findById(orderId)
            .orElseThrow(() -> new NotFoundException("Order not found: " + orderId));

        // Validation scattered in the service
        if (!"DRAFT".equals(order.getStatus())) {
            throw new IllegalStateException("Cannot confirm order in status: " + order.getStatus());
        }
        if (order.getItems() == null || order.getItems().isEmpty()) {
            throw new IllegalStateException("Cannot confirm order with no items");
        }

        // Calculation in the service
        BigDecimal total = order.getItems().stream()
            .map(item -> item.getUnitPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        if (total.compareTo(new BigDecimal("1.00")) < 0) {
            throw new IllegalStateException("Order total too low: " + total);
        }

        // Direct manipulation of entity state from outside
        order.setStatus("CONFIRMED");
        order.setTotalAmount(total);
        order.setConfirmedAt(Instant.now());
        repository.save(order);

        // Event publishing mixed into service logic
        eventPublisher.publish(new OrderPlacedEvent(orderId, order.getCustomerId(), total));
    }

    public void cancelOrder(String orderId, String reason) {
        AnemicOrder order = repository.findById(orderId)
            .orElseThrow(() -> new NotFoundException("Order not found: " + orderId));

        // Duplicate status validation — this check appears in every method
        List<String> cancellableStatuses = Arrays.asList("DRAFT", "CONFIRMED", "FULFILLING");
        if (!cancellableStatuses.contains(order.getStatus())) {
            throw new IllegalStateException("Cannot cancel order in status: " + order.getStatus());
        }

        order.setStatus("CANCELLED");
        repository.save(order);
    }
}
```

```java
// ===== RICH ORDER (CORRECT APPROACH) =====

// The entity owns its behavior and enforces its invariants
public class Order {

    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items = new ArrayList<>();
    private OrderStatus status;
    private Instant createdAt;
    private Instant confirmedAt;
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public Order(OrderId id, CustomerId customerId) {
        this.id = id;
        this.customerId = customerId;
        this.status = OrderStatus.DRAFT;
        this.createdAt = Instant.now();
    }

    // Behavior methods express domain operations — not setters
    public void addItem(ProductId productId, String productName, int quantity, Money unitPrice) {
        if (status != OrderStatus.DRAFT) {
            throw new DomainException("Cannot add items to an order in status: " + status);
        }
        items.add(new OrderItem(OrderItemId.generate(), productId, productName, quantity, unitPrice));
    }

    public void confirm() {
        if (status != OrderStatus.DRAFT) {
            throw new DomainException("Cannot confirm an order in status: " + status);
        }
        if (items.isEmpty()) {
            throw new DomainException("Cannot confirm an order with no items");
        }
        Money total = calculateTotal();
        if (total.isZero()) {
            throw new DomainException("Cannot confirm an order with zero total");
        }

        this.status = OrderStatus.CONFIRMED;
        this.confirmedAt = Instant.now();
        domainEvents.add(new OrderPlaced(id, customerId, total, confirmedAt));
    }

    public void cancel(String reason) {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
            throw new DomainException("Cannot cancel an order in status: " + status);
        }
        this.status = OrderStatus.CANCELLED;
        domainEvents.add(new OrderCancelled(id, reason, Instant.now()));
    }

    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.zero("USD"), Money::add);
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }

    // No setters for status, confirmedAt, items — they are only modified through behavior methods
    public OrderId getId() { return id; }
    public CustomerId getCustomerId() { return customerId; }
    public OrderStatus getStatus() { return status; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
}

// The application service is thin — it orchestrates, it does not contain domain logic
@Service
public class OrderApplicationService {

    private final OrderRepository repository;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public void confirmOrder(String orderId) {
        Order order = repository.findById(OrderId.of(orderId))
            .orElseThrow(() -> new NotFoundException("Order not found: " + orderId));

        order.confirm(); // All the logic is inside the aggregate

        repository.save(order);
        eventPublisher.publishAll(order.pullDomainEvents());
    }

    @Transactional
    public void cancelOrder(String orderId, String reason) {
        Order order = repository.findById(OrderId.of(orderId))
            .orElseThrow(() -> new NotFoundException("Order not found: " + orderId));

        order.cancel(reason);

        repository.save(order);
        eventPublisher.publishAll(order.pullDomainEvents());
    }
}
```

### The "Where Does This Logic Live?" Decision Framework

When you encounter a piece of logic and are unsure where it belongs, apply this sequence:

1. **Is it a constraint that must always hold for this entity/aggregate to be valid?** If yes, it belongs in the entity or aggregate root. A `confirm()` pre-condition check belongs in `Order`.

2. **Does it require knowledge of only one aggregate's state?** If yes, it belongs in that aggregate or one of its entities.

3. **Does it involve multiple aggregates without belonging naturally to any?** If yes, it belongs in a domain service.

4. **Does it require external systems (email, payment gateways, databases)?** If yes, it belongs in the application layer or infrastructure layer — not in the domain.

5. **Is it a policy that might vary by context or configuration?** If yes, it belongs in a policy object or strategy, injected into the domain service.

---

## 4. Identifying the Right Abstractions

### The Abstraction Identification Process

Identifying the right abstractions is the hardest part of domain modeling. The wrong abstractions produce code that fights the domain — every new feature requires workarounds because the model doesn't match reality.

The process:

1. **Listen to domain experts**. The language they use reveals the abstractions. If they say "we hold a reservation for the customer", `Reservation` is an entity, not just a flag on `Booking`.

2. **Look for implicit concepts**. Hidden domain concepts are often expressed as conditional logic or magic numbers. A pricing rule conditional on customer tier is a `PricingPolicy`. A date range that keeps appearing is a `DateRange` value object.

3. **Follow the operations**. Methods and use cases reveal domain concepts. If you find yourself writing `calculateDiscountForPremiumCustomerWithPromoCode()`, the method name is revealing two separate concepts: `CustomerTier` and `PromotionCode`.

4. **Ask where the invariant lives**. Every invariant implies ownership. If a rule involves multiple entities, you need either an aggregate or a domain service.

5. **Verify against the ubiquitous language**. If you cannot explain an abstraction using terms the domain expert would recognize, the abstraction is probably wrong.

### Nouns That Become Entities vs Value Objects

Use this heuristic: does the concept have continuity and identity beyond its attributes?

- A **customer** exists as the same customer even if their name, address, and email all change. They have identity. Entities are the right abstraction.
- A **price** has no continuity — $9.99 USD is $9.99 USD regardless of context. If you change the amount, you have a different price. Value objects are the right abstraction.
- A **reservation** has identity — it was made, it can be cancelled, it can be transferred. Entity.
- An **address** — this is context-dependent. In a shipping domain, an address might be a value object (the same physical location is always the same). In a demographics domain, it might be an entity with identity (we need to track changes to addresses over time).

### Operations That Reveal Domain Concepts

```java
// BAD: method name hides domain concept
public BigDecimal calculateFinalPrice(BigDecimal basePrice, String customerType, String promoCode, boolean isFirstOrder) {
    // ... complex logic
}

// GOOD: domain concepts extracted, named, and used
public Money calculateFinalPrice(Money basePrice, CustomerTier tier, Optional<PromoCode> promoCode, boolean isFirstOrder) {
    PricingPolicy policy = PricingPolicy.forTier(tier);
    Money discountedPrice = policy.apply(basePrice);

    if (promoCode.isPresent()) {
        PromotionDiscount promotion = promoCode.get().getDiscount();
        discountedPrice = promotion.apply(discountedPrice);
    }

    if (isFirstOrder) {
        discountedPrice = FirstOrderIncentive.apply(discountedPrice);
    }

    return discountedPrice;
}
```

### The Cost of Wrong Abstractions

Wrong abstractions compound over time. Sandro Mancuso's observation is relevant: it is harder to remove a wrong abstraction than to add a right one. A `UserManager` class that accretes responsibilities over time because "users" are too broad — this is a wrong abstraction. It becomes a God class.

Wrong abstractions cost:
- Every feature related to the abstraction costs more to implement than it should.
- Testing requires setting up more state than the scenario needs.
- New engineers take longer to understand the model.
- Bugs are harder to find because the model doesn't match what domain experts describe.

### How to Refactor Toward Better Abstractions

Refactoring a domain model is risky but necessary. The approach:

1. Identify the smell: God class, inappropriate intimacy between classes, duplicate logic, methods that don't fit their class.
2. Extract the concept: give the implicit concept an explicit name.
3. Move behavior: move methods to where their data lives.
4. Tighten access: remove public setters that shouldn't be public.
5. Update the ubiquitous language: rename variables, methods, and classes to match the new understanding.

---

## 5. Ubiquitous Language

### Why Naming in Code Must Match Business Language

The ubiquitous language is the shared vocabulary used by both domain experts and engineers. It is expressed consistently in conversations, documentation, and code. When the language diverges — when engineers use different terms than business people — every conversation requires translation, and that translation introduces error.

If the business says "a Merchant settles their batch at end of day" and the code says `UserService.processEndOfDayTransactionAggregation()`, then:
- The code cannot be read by domain experts.
- Engineers mentally translate between two vocabularies.
- The translation introduces misunderstandings — "settling" and "processing" may have subtly different meanings.
- When requirements change, it is unclear whether the code needs to change.

### Building a Glossary with Domain Experts

The glossary process:
1. In every domain discussion, write down the terms used.
2. For each term, record: what it means, what its synonyms are (and which synonyms to avoid), what its relationship to other terms is.
3. Challenge ambiguous terms: "When you say 'account', do you mean a customer's profile or their financial account?"
4. Update the code when the glossary changes. Renaming a class is not cosmetic — it is a domain insight materialized in code.

### Code Examples: Domain Language vs Technical Language

```java
// POOR: technical language, no domain meaning
public class UserRecord {
    public void processStateTransition(String newState) { ... }
    public boolean checkUserEligibilityForAction() { ... }
    public void updateUserDataAndPersist() { ... }
}

// GOOD: ubiquitous language expressed in code
public class Member {                          // "Member" not "User" — domain term
    public void activate() { ... }            // domain event, not a generic state transition
    public boolean isEligibleForRenewal() { ... }  // domain concept
    public void renewMembership(Period duration) { ... }  // domain operation
}

// POOR: technical language in service
public class OrderProcessingHandler {
    public void executeOrderStatusUpdate(String orderId, String statusCode) { ... }
    public List<Map<String, Object>> getFilteredOrderData(Map<String, String> filters) { ... }
}

// GOOD: domain language in service
public class OrderFulfillmentService {
    public void shipOrder(OrderId orderId) { ... }
    public List<Order> findOrdersPendingShipment() { ... }
    public void notifyCustomerOfDelay(OrderId orderId, String reason) { ... }
}
```

### Why Renaming Matters

A rename is not cosmetic. Consider:
- `User` renamed to `Subscriber` — reveals that the system is subscription-based and that a `User` without a subscription is meaningless.
- `process()` renamed to `disburse()` — reveals that this is a financial disbursement with regulatory implications.
- `flag` renamed to `fraudSuspected` — reveals that this boolean carries domain meaning and should be set through a specific process, not via a setter.

Each renaming eliminates ambiguity and makes the model more accurate. Code becomes executable documentation.

---

## 6. Bounded Contexts (Introduction)

### What a Bounded Context Is

A bounded context is an explicit boundary within which a particular domain model applies. Inside a bounded context, every term has a precise, unambiguous meaning. Outside that boundary, the same term may mean something different.

Large domains cannot be modeled uniformly. "Customer" means different things to marketing, billing, and support. Trying to create one unified `Customer` class that satisfies all three contexts produces a bloated, confused abstraction. Bounded contexts allow each part of the domain to define its own model, its own vocabulary, and its own rules.

### Why the Same Concept Means Different Things in Different Contexts

Consider "Customer" across three bounded contexts in an e-commerce platform:

| Bounded Context | "Customer" Means |
|---|---|
| Order Management | An entity with a billing address, a shipping address, and an order history. Can have multiple payment methods. |
| Loyalty Program | A member with a tier (Bronze/Silver/Gold), points balance, and redemption history. Customer-tier relationship drives discount eligibility. |
| Support | A person with contact details, an open ticket count, and a satisfaction history. May or may not have placed orders. |

These are genuinely different concepts. A support representative does not care about the customer's payment methods. The loyalty program does not care about shipping addresses. If you create a single `Customer` class that carries all of these fields, every context is coupled to concerns it does not own.

### Context Mapping Basics

**Upstream/Downstream**: Context A (upstream) produces a model that Context B (downstream) consumes. The upstream has no dependency on the downstream. If the upstream changes its model, the downstream must adapt — or use an anti-corruption layer to translate.

**Shared Kernel**: Two bounded contexts share a small piece of the domain model (usually basic types like identifiers and money). Changes to the shared kernel require coordination between both teams.

**Anti-Corruption Layer (ACL)**: A translation layer on the boundary of a bounded context that insulates the downstream context from changes to the upstream model. The ACL translates upstream concepts into local concepts without polluting the local model.

```java
// Customer in the Order Management context
public class Customer {          // package: com.example.ordering.domain
    private final CustomerId id;
    private ShippingAddress defaultShippingAddress;
    private BillingAddress billingAddress;
    private List<PaymentMethod> paymentMethods;
}

// Customer in the Loyalty Program context — a completely different class
public class LoyaltyMember {     // package: com.example.loyalty.domain
    private final CustomerId id; // same identifier, but different class
    private LoyaltyTier tier;
    private int pointsBalance;
    private List<PointsTransaction> pointsHistory;
}

// Anti-corruption layer: translates the order context's Customer ID 
// into a LoyaltyMember without leaking the order model into the loyalty context
public class LoyaltyMemberFinder {

    private final LoyaltyMemberRepository memberRepository;

    public Optional<LoyaltyMember> findByOrderCustomerId(CustomerId customerId) {
        return memberRepository.findById(customerId);
        // Translation happens here — if the IDs are different types between contexts,
        // the mapping logic would live in this ACL class
    }
}
```

### Practical Implication: Don't Share Domain Models Across Bounded Contexts

This is one of the most important and most frequently violated rules in microservice design. Sharing a domain model across bounded contexts creates tight coupling between services. When the Order service's `Customer` model changes, the Loyalty service breaks. Microservices that share data models are not truly independent.

Each bounded context should own its model. Integration happens at the edges, through events, APIs, and anti-corruption layers — not through shared libraries containing domain classes.

---

## 7. Modeling Complex Business Rules in Code

### Specification Pattern for Complex Predicates

The Specification pattern encapsulates a business rule as an object with an `isSatisfiedBy()` method. Specifications can be composed using logical operators (and, or, not) to express complex rules from simpler pieces.

```java
// Base specification interface
public interface Specification<T> {
    boolean isSatisfiedBy(T candidate);

    default Specification<T> and(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) && other.isSatisfiedBy(candidate);
    }

    default Specification<T> or(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) || other.isSatisfiedBy(candidate);
    }

    default Specification<T> not() {
        return candidate -> !this.isSatisfiedBy(candidate);
    }
}

// Concrete specifications — each captures one business rule
public class PremiumCustomerSpecification implements Specification<Customer> {
    @Override
    public boolean isSatisfiedBy(Customer customer) {
        return customer.getTier() == CustomerTier.PREMIUM
            || customer.getTier() == CustomerTier.GOLD;
    }
}

public class LongTermCustomerSpecification implements Specification<Customer> {
    private final Period minimumTenure;

    public LongTermCustomerSpecification(Period minimumTenure) {
        this.minimumTenure = minimumTenure;
    }

    @Override
    public boolean isSatisfiedBy(Customer customer) {
        return customer.getMemberSince().isBefore(LocalDate.now().minus(minimumTenure));
    }
}

public class MinimumOrderValueSpecification implements Specification<Order> {
    private final Money minimumValue;

    public MinimumOrderValueSpecification(Money minimumValue) {
        this.minimumValue = minimumValue;
    }

    @Override
    public boolean isSatisfiedBy(Order order) {
        return order.calculateTotal().isGreaterThan(minimumValue)
            || order.calculateTotal().equals(minimumValue);
    }
}

// Usage: complex rule composed from simple specifications
public class DiscountEligibilityService {

    private final Specification<Customer> premiumOrLongTerm;
    private final Specification<Order> minimumOrder;

    public DiscountEligibilityService() {
        Specification<Customer> premium = new PremiumCustomerSpecification();
        Specification<Customer> longTerm = new LongTermCustomerSpecification(Period.ofYears(2));
        this.premiumOrLongTerm = premium.or(longTerm);
        this.minimumOrder = new MinimumOrderValueSpecification(Money.of(50.00, "USD"));
    }

    public boolean isEligibleForDiscount(Customer customer, Order order) {
        return premiumOrLongTerm.isSatisfiedBy(customer)
            && minimumOrder.isSatisfiedBy(order);
    }
}
```

### Policy Objects for Business Policies

A policy object encapsulates a business policy — a rule that determines behavior under specific conditions. Unlike specifications (which answer yes/no), policies also determine what happens.

```java
// Refund policy hierarchy — different policies for different customer tiers
public interface RefundPolicy {
    RefundDecision evaluate(Order order, RefundRequest request);
}

public class StandardRefundPolicy implements RefundPolicy {
    private static final int REFUND_WINDOW_DAYS = 30;

    @Override
    public RefundDecision evaluate(Order order, RefundRequest request) {
        if (order.getDeliveredAt() == null) {
            return RefundDecision.denied("Order has not been delivered");
        }
        long daysSinceDelivery = ChronoUnit.DAYS.between(order.getDeliveredAt(), Instant.now());
        if (daysSinceDelivery > REFUND_WINDOW_DAYS) {
            return RefundDecision.denied("Refund window of " + REFUND_WINDOW_DAYS + " days has passed");
        }
        return RefundDecision.approved(order.calculateTotal());
    }
}

public class PremiumRefundPolicy implements RefundPolicy {
    private static final int REFUND_WINDOW_DAYS = 90;

    @Override
    public RefundDecision evaluate(Order order, RefundRequest request) {
        if (order.getDeliveredAt() == null) {
            return RefundDecision.approved(order.calculateTotal()); // Premium customers get refund even before delivery
        }
        long daysSinceDelivery = ChronoUnit.DAYS.between(order.getDeliveredAt(), Instant.now());
        if (daysSinceDelivery > REFUND_WINDOW_DAYS) {
            return RefundDecision.denied("Refund window of " + REFUND_WINDOW_DAYS + " days has passed");
        }
        return RefundDecision.approved(order.calculateTotal());
    }
}

public class RefundPolicyFactory {
    public RefundPolicy forCustomer(Customer customer) {
        return switch (customer.getTier()) {
            case PREMIUM, GOLD -> new PremiumRefundPolicy();
            default -> new StandardRefundPolicy();
        };
    }
}
```

### State Machines for Lifecycle Management

Complex entities with well-defined lifecycle transitions are best modeled as explicit state machines. State machines make valid transitions visible and prevent invalid transitions at compile time or at runtime.

```java
public enum OrderStatus {
    DRAFT, CONFIRMED, FULFILLING, SHIPPED, DELIVERED, CANCELLED
}

// Order state machine — defines valid transitions
public class OrderStateMachine {

    private static final Map<OrderStatus, Set<OrderStatus>> VALID_TRANSITIONS = new HashMap<>();

    static {
        VALID_TRANSITIONS.put(OrderStatus.DRAFT,
            EnumSet.of(OrderStatus.CONFIRMED, OrderStatus.CANCELLED));

        VALID_TRANSITIONS.put(OrderStatus.CONFIRMED,
            EnumSet.of(OrderStatus.FULFILLING, OrderStatus.CANCELLED));

        VALID_TRANSITIONS.put(OrderStatus.FULFILLING,
            EnumSet.of(OrderStatus.SHIPPED, OrderStatus.CANCELLED));

        VALID_TRANSITIONS.put(OrderStatus.SHIPPED,
            EnumSet.of(OrderStatus.DELIVERED));
            // Note: SHIPPED orders cannot be CANCELLED

        VALID_TRANSITIONS.put(OrderStatus.DELIVERED,
            EnumSet.of());
            // Terminal state — no transitions out

        VALID_TRANSITIONS.put(OrderStatus.CANCELLED,
            EnumSet.of());
            // Terminal state — no transitions out
    }

    public static void assertTransitionAllowed(OrderStatus from, OrderStatus to) {
        Set<OrderStatus> allowed = VALID_TRANSITIONS.getOrDefault(from, EnumSet.noneOf(OrderStatus.class));
        if (!allowed.contains(to)) {
            throw new IllegalStateTransitionException(
                "Transition from " + from + " to " + to + " is not permitted"
            );
        }
    }

    public static boolean canTransition(OrderStatus from, OrderStatus to) {
        return VALID_TRANSITIONS.getOrDefault(from, EnumSet.noneOf(OrderStatus.class)).contains(to);
    }
}

// The Order entity uses the state machine to validate transitions
public class Order {

    private OrderStatus status = OrderStatus.DRAFT;

    private void transitionTo(OrderStatus newStatus) {
        OrderStateMachine.assertTransitionAllowed(this.status, newStatus);
        this.status = newStatus;
    }

    public void confirm() {
        // Pre-condition checks
        if (items.isEmpty()) throw new DomainException("No items in order");
        transitionTo(OrderStatus.CONFIRMED);
    }

    public void startFulfillment() {
        transitionTo(OrderStatus.FULFILLING);
    }

    public void ship() {
        transitionTo(OrderStatus.SHIPPED);
    }

    public void deliver() {
        transitionTo(OrderStatus.DELIVERED);
    }

    public void cancel(String reason) {
        transitionTo(OrderStatus.CANCELLED);
    }
}
```

---

## 8. Domain Events in Detail

### Why Domain Events Are a Senior-Level Pattern

Domain events elevate the quality of a model in three ways:

1. **Explicit causality**: Events make the cause-and-effect relationships in a domain visible. "When an order is placed, send a confirmation email, reserve inventory, and notify the warehouse" — this is expressed as three event handlers for `OrderPlaced`, not as three method calls inside `OrderService`.

2. **Decoupling**: Event publishers and event consumers are decoupled. The `Order` aggregate does not know that sending an email will happen. It only records that it was placed. This eliminates coupling between the order domain and the notification domain.

3. **Audit trail**: Events are records of what happened. A sequence of domain events is a complete history of the aggregate's lifecycle, which is the foundation of Event Sourcing.

### Event Design: Past Tense Naming, Immutability, What to Include

Event naming rules:
- Use past tense: `OrderPlaced`, not `PlaceOrder` or `OrderPlacedEvent`.
- Name from the domain's perspective: `PaymentDeclined`, not `PaymentProcessingFailed`.
- Include the aggregate name as the subject: `OrderShipped`, `CustomerSuspended`, `InventoryDepleted`.

What to include in an event:
- The aggregate's identifier (always).
- The key state that changed (but not a full dump of the aggregate — events are not snapshots).
- The timestamp when it occurred.
- Any contextual information consumers need to react without querying back.
- Do not include references to mutable objects.

```java
// Well-designed domain event — immutable, past-tense, includes necessary context
public final class PaymentDeclined extends BaseDomainEvent {

    private final PaymentId paymentId;
    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money attemptedAmount;
    private final String declineReason;      // consumers need this to decide what to do
    private final int attemptNumber;         // consumers may want to retry logic

    public PaymentDeclined(
        PaymentId paymentId,
        OrderId orderId,
        CustomerId customerId,
        Money attemptedAmount,
        String declineReason,
        int attemptNumber
    ) {
        super();
        this.paymentId = paymentId;
        this.orderId = orderId;
        this.customerId = customerId;
        this.attemptedAmount = attemptedAmount;
        this.declineReason = declineReason;
        this.attemptNumber = attemptNumber;
    }

    @Override
    public String getEventType() { return "payment.declined"; }

    // Immutable — all fields final, no setters
    public PaymentId getPaymentId() { return paymentId; }
    public OrderId getOrderId() { return orderId; }
    public CustomerId getCustomerId() { return customerId; }
    public Money getAttemptedAmount() { return attemptedAmount; }
    public String getDeclineReason() { return declineReason; }
    public int getAttemptNumber() { return attemptNumber; }
}
```

### Event Handling: In-Process vs Out-of-Process

**In-process (synchronous)**: The event is handled within the same transaction, on the same thread. Useful for side effects that must succeed or fail with the aggregate operation.

**Out-of-process (asynchronous)**: The event is published to a message broker (Kafka, RabbitMQ). Consumers in other services or other parts of the same service react eventually. Suitable for cross-context communication, long-running reactions, and notifications.

```java
// Application service pattern for domain event dispatch
@Service
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public void placeOrder(PlaceOrderCommand command) {
        Order order = new Order(OrderId.generate(), command.getCustomerId());

        for (OrderItemCommand itemCmd : command.getItems()) {
            order.addItem(
                itemCmd.getProductId(),
                itemCmd.getProductName(),
                itemCmd.getQuantity(),
                itemCmd.getUnitPrice()
            );
        }

        order.confirm();
        orderRepository.save(order);

        // Domain events are pulled from the aggregate and dispatched by the application layer
        // The domain model has no knowledge of how events are published
        List<DomainEvent> events = order.pullDomainEvents();
        eventPublisher.publishAll(events);
    }
}

// Domain event publisher interface — domain layer defines the contract
public interface DomainEventPublisher {
    void publish(DomainEvent event);
    void publishAll(List<DomainEvent> events);
}

// Infrastructure implementation — routes events to message broker
@Component
public class KafkaDomainEventPublisher implements DomainEventPublisher {

    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final EventSerializer serializer;

    @Override
    public void publishAll(List<DomainEvent> events) {
        events.forEach(this::publish);
    }

    @Override
    public void publish(DomainEvent event) {
        String topic = "domain-events." + event.getEventType();
        kafkaTemplate.send(topic, event.getEventId(), serializer.serialize(event));
    }
}
```

### Full Java Example: Event Publication Within an Aggregate

```java
public class ShoppingCart {

    private final CartId id;
    private final CustomerId customerId;
    private final List<CartItem> items = new ArrayList<>();
    private CartStatus status = CartStatus.ACTIVE;
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public void addItem(ProductId productId, String productName, int quantity, Money unitPrice) {
        if (status != CartStatus.ACTIVE) {
            throw new DomainException("Cannot add items to a " + status.name().toLowerCase() + " cart");
        }

        items.stream()
            .filter(item -> item.getProductId().equals(productId))
            .findFirst()
            .ifPresentOrElse(
                existing -> existing.increaseQuantity(quantity),
                () -> items.add(new CartItem(CartItemId.generate(), productId, productName, quantity, unitPrice))
            );

        domainEvents.add(new CartItemAdded(id, productId, quantity, Instant.now()));
    }

    public void removeItem(ProductId productId) {
        boolean removed = items.removeIf(item -> item.getProductId().equals(productId));
        if (!removed) {
            throw new DomainException("Item not found in cart: " + productId);
        }
        domainEvents.add(new CartItemRemoved(id, productId, Instant.now()));
    }

    public void abandon() {
        if (status == CartStatus.CHECKED_OUT) {
            throw new DomainException("A checked-out cart cannot be abandoned");
        }
        this.status = CartStatus.ABANDONED;
        domainEvents.add(new CartAbandoned(id, customerId, calculateTotal(), Instant.now()));
    }

    public Order checkout() {
        if (items.isEmpty()) throw new DomainException("Cannot checkout an empty cart");
        this.status = CartStatus.CHECKED_OUT;

        Order order = new Order(OrderId.generate(), customerId);
        for (CartItem item : items) {
            order.addItem(item.getProductId(), item.getProductName(), item.getQuantity(), item.getUnitPrice());
        }
        order.confirm();

        domainEvents.add(new CartCheckedOut(id, order.getId(), customerId, Instant.now()));
        return order;
    }

    public Money calculateTotal() {
        return items.stream()
            .map(CartItem::getSubtotal)
            .reduce(Money.zero("USD"), Money::add);
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return Collections.unmodifiableList(events);
    }
}
```

---

## 9. Full Domain Model Example: E-Commerce

This example shows a cohesive domain model for order management in an e-commerce context. Note how each class has a clear responsibility, invariants are enforced within the aggregate, and the domain speaks the language of the business.

```java
// ==========================================
// VALUE OBJECTS
// ==========================================

public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount must be a non-negative value");
        if (currency == null)
            throw new IllegalArgumentException("Currency must not be null");
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    public static Money of(double amount, String currencyCode) {
        return new Money(BigDecimal.valueOf(amount), Currency.getInstance(currencyCode));
    }

    public static Money zero(String currencyCode) {
        return new Money(BigDecimal.ZERO, Currency.getInstance(currencyCode));
    }

    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        assertSameCurrency(other);
        BigDecimal result = this.amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Subtraction would result in negative money");
        return new Money(result, this.currency);
    }

    public Money multiply(int factor) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(factor)), this.currency);
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return this.amount.compareTo(other.amount) > 0;
    }

    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
    }

    public BigDecimal getAmount() { return amount; }
    public Currency getCurrency() { return currency; }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof Money)) return false;
        Money other = (Money) obj;
        return this.amount.compareTo(other.amount) == 0 && this.currency.equals(other.currency);
    }

    @Override
    public int hashCode() { return Objects.hash(amount.stripTrailingZeros(), currency); }

    @Override
    public String toString() { return currency.getSymbol() + amount.toPlainString(); }
}

// ==========================================
// ENTITIES AND AGGREGATES
// ==========================================

// Product — a separate aggregate, referenced by ID from the Order aggregate
public class Product {

    private final ProductId id;
    private String name;
    private String description;
    private Money price;
    private int stockQuantity;
    private ProductStatus status;

    public Product(ProductId id, String name, Money price, int initialStock) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.stockQuantity = initialStock;
        this.status = ProductStatus.AVAILABLE;
    }

    public void restock(int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException("Restock quantity must be positive");
        this.stockQuantity += quantity;
        if (this.status == ProductStatus.OUT_OF_STOCK) {
            this.status = ProductStatus.AVAILABLE;
        }
    }

    public void destock(int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException("Destock quantity must be positive");
        if (this.stockQuantity < quantity) {
            throw new DomainException("Insufficient stock for product: " + id);
        }
        this.stockQuantity -= quantity;
        if (this.stockQuantity == 0) {
            this.status = ProductStatus.OUT_OF_STOCK;
        }
    }

    public boolean isAvailable(int requiredQuantity) {
        return status == ProductStatus.AVAILABLE && stockQuantity >= requiredQuantity;
    }

    public ProductId getId() { return id; }
    public String getName() { return name; }
    public Money getPrice() { return price; }
    public int getStockQuantity() { return stockQuantity; }
    public ProductStatus getStatus() { return status; }
}

// Customer — a separate aggregate
public class Customer {

    private final CustomerId id;
    private final String firstName;
    private final String lastName;
    private final EmailAddress email;
    private CustomerTier tier;
    private Instant memberSince;

    public Customer(CustomerId id, String firstName, String lastName, EmailAddress email) {
        this.id = id;
        this.firstName = firstName;
        this.lastName = lastName;
        this.email = email;
        this.tier = CustomerTier.STANDARD;
        this.memberSince = Instant.now();
    }

    public void upgradeTier(CustomerTier newTier) {
        if (newTier.ordinal() <= this.tier.ordinal()) {
            throw new DomainException("Cannot downgrade customer tier");
        }
        this.tier = newTier;
    }

    public String getFullName() { return firstName + " " + lastName; }
    public CustomerId getId() { return id; }
    public EmailAddress getEmail() { return email; }
    public CustomerTier getTier() { return tier; }
    public Instant getMemberSince() { return memberSince; }
}

// OrderItem — internal entity of the Order aggregate
class OrderItem {

    private final OrderItemId id;
    private final ProductId productId;
    private final String productName;
    private final Money unitPrice;
    private int quantity;

    OrderItem(OrderItemId id, ProductId productId, String productName, int quantity, Money unitPrice) {
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive");
        this.id = id;
        this.productId = productId;
        this.productName = productName;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    void increaseQuantity(int delta) {
        if (delta <= 0) throw new IllegalArgumentException("Delta must be positive");
        this.quantity += delta;
    }

    Money getSubtotal() { return unitPrice.multiply(quantity); }
    OrderItemId getId() { return id; }
    ProductId getProductId() { return productId; }
    String getProductName() { return productName; }
    int getQuantity() { return quantity; }
    Money getUnitPrice() { return unitPrice; }
}

// Order — the aggregate root. Enforces all order invariants.
public class Order {

    private static final int MAX_LINE_ITEMS = 50;
    private static final Money MINIMUM_ORDER_VALUE = Money.of(5.00, "USD");

    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items = new ArrayList<>();
    private OrderStatus status = OrderStatus.DRAFT;
    private ShippingAddress shippingAddress;
    private Instant createdAt;
    private Instant confirmedAt;
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public Order(OrderId id, CustomerId customerId, ShippingAddress shippingAddress) {
        Objects.requireNonNull(id, "Order ID required");
        Objects.requireNonNull(customerId, "Customer ID required");
        Objects.requireNonNull(shippingAddress, "Shipping address required");

        this.id = id;
        this.customerId = customerId;
        this.shippingAddress = shippingAddress;
        this.createdAt = Instant.now();
    }

    public void addItem(ProductId productId, String productName, int quantity, Money unitPrice) {
        requireStatus(OrderStatus.DRAFT);
        if (items.size() >= MAX_LINE_ITEMS) {
            throw new DomainException("Maximum of " + MAX_LINE_ITEMS + " line items per order");
        }

        items.stream()
            .filter(item -> item.getProductId().equals(productId))
            .findFirst()
            .ifPresentOrElse(
                item -> item.increaseQuantity(quantity),
                () -> items.add(new OrderItem(OrderItemId.generate(), productId, productName, quantity, unitPrice))
            );
    }

    public void removeItem(ProductId productId) {
        requireStatus(OrderStatus.DRAFT);
        if (!items.removeIf(item -> item.getProductId().equals(productId))) {
            throw new DomainException("No item for product: " + productId);
        }
    }

    // Invariants enforced here — confirms the order only when all rules are satisfied
    public void confirm() {
        requireStatus(OrderStatus.DRAFT);

        if (items.isEmpty()) {
            throw new DomainException("Order must have at least one item");
        }

        Money total = calculateTotal();
        if (!total.isGreaterThan(MINIMUM_ORDER_VALUE) && !total.equals(MINIMUM_ORDER_VALUE)) {
            throw new DomainException("Order total " + total + " is below minimum " + MINIMUM_ORDER_VALUE);
        }

        this.status = OrderStatus.CONFIRMED;
        this.confirmedAt = Instant.now();

        domainEvents.add(new OrderPlaced(id, customerId, total, shippingAddress, confirmedAt));
    }

    public void ship(TrackingNumber trackingNumber) {
        requireStatus(OrderStatus.CONFIRMED);
        this.status = OrderStatus.SHIPPED;
        domainEvents.add(new OrderShipped(id, customerId, trackingNumber, Instant.now()));
    }

    public void deliver() {
        requireStatus(OrderStatus.SHIPPED);
        this.status = OrderStatus.DELIVERED;
        domainEvents.add(new OrderDelivered(id, customerId, Instant.now()));
    }

    public void cancel(String reason) {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
            throw new DomainException("Cannot cancel an order in status: " + status);
        }
        this.status = OrderStatus.CANCELLED;
        domainEvents.add(new OrderCancelled(id, reason, Instant.now()));
    }

    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.zero("USD"), Money::add);
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }

    private void requireStatus(OrderStatus required) {
        if (this.status != required) {
            throw new DomainException("Expected status " + required + " but was " + status);
        }
    }

    public OrderId getId() { return id; }
    public CustomerId getCustomerId() { return customerId; }
    public OrderStatus getStatus() { return status; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
    public Instant getCreatedAt() { return createdAt; }
}

// ==========================================
// REPOSITORY INTERFACE (domain layer)
// ==========================================

public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    List<Order> findByStatus(OrderStatus status);
    List<Order> findConfirmedOrdersAwaitingFulfillment();
}

// ==========================================
// DOMAIN SERVICE
// ==========================================

public class OrderPricingService {

    private final DiscountPolicy discountPolicy;

    public OrderPricingService(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }

    public Money calculateFinalPrice(Order order, Customer customer) {
        Money baseTotal = order.calculateTotal();
        Discount discount = discountPolicy.determineDiscount(customer, order);
        Money discountAmount = baseTotal.multiply(discount.getRate());
        return baseTotal.subtract(discountAmount);
    }
}

// ==========================================
// DOMAIN EVENTS
// ==========================================

public final class OrderPlaced extends BaseDomainEvent {

    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money orderTotal;
    private final ShippingAddress shippingAddress;
    private final Instant placedAt;

    public OrderPlaced(OrderId orderId, CustomerId customerId, Money orderTotal,
                       ShippingAddress shippingAddress, Instant placedAt) {
        super();
        this.orderId = orderId;
        this.customerId = customerId;
        this.orderTotal = orderTotal;
        this.shippingAddress = shippingAddress;
        this.placedAt = placedAt;
    }

    @Override
    public String getEventType() { return "order.placed"; }

    public OrderId getOrderId() { return orderId; }
    public CustomerId getCustomerId() { return customerId; }
    public Money getOrderTotal() { return orderTotal; }
    public ShippingAddress getShippingAddress() { return shippingAddress; }
    public Instant getPlacedAt() { return placedAt; }
}
```

---

## 10. Full Domain Model Example: Ride-Sharing

```java
// ==========================================
// VALUE OBJECTS
// ==========================================

public final class Location {
    private final double latitude;
    private final double longitude;

    public Location(double latitude, double longitude) {
        if (latitude < -90 || latitude > 90)
            throw new IllegalArgumentException("Latitude must be between -90 and 90");
        if (longitude < -180 || longitude > 180)
            throw new IllegalArgumentException("Longitude must be between -180 and 180");
        this.latitude = latitude;
        this.longitude = longitude;
    }

    public double distanceInKilometersTo(Location other) {
        // Haversine formula
        final int EARTH_RADIUS_KM = 6371;
        double dLat = Math.toRadians(other.latitude - this.latitude);
        double dLon = Math.toRadians(other.longitude - this.longitude);
        double a = Math.sin(dLat / 2) * Math.sin(dLat / 2)
            + Math.cos(Math.toRadians(this.latitude)) * Math.cos(Math.toRadians(other.latitude))
            * Math.sin(dLon / 2) * Math.sin(dLon / 2);
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        return EARTH_RADIUS_KM * c;
    }

    public double getLatitude() { return latitude; }
    public double getLongitude() { return longitude; }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof Location)) return false;
        Location other = (Location) obj;
        return Double.compare(latitude, other.latitude) == 0
            && Double.compare(longitude, other.longitude) == 0;
    }

    @Override
    public int hashCode() { return Objects.hash(latitude, longitude); }

    @Override
    public String toString() { return "(" + latitude + ", " + longitude + ")"; }
}

public final class Fare {
    private final Money baseAmount;
    private final Money surgeAmount;
    private final double surgeMultiplier;

    public Fare(Money baseAmount, double surgeMultiplier) {
        if (surgeMultiplier < 1.0) throw new IllegalArgumentException("Surge multiplier must be >= 1.0");
        this.baseAmount = baseAmount;
        this.surgeMultiplier = surgeMultiplier;
        this.surgeAmount = baseAmount.multiply(BigDecimal.valueOf(surgeMultiplier - 1.0));
    }

    public static Fare standard(Money baseAmount) {
        return new Fare(baseAmount, 1.0);
    }

    public static Fare surge(Money baseAmount, double multiplier) {
        return new Fare(baseAmount, multiplier);
    }

    public Money getTotal() { return baseAmount.add(surgeAmount); }
    public Money getBaseAmount() { return baseAmount; }
    public Money getSurgeAmount() { return surgeAmount; }
    public double getSurgeMultiplier() { return surgeMultiplier; }
    public boolean isSurgeActive() { return surgeMultiplier > 1.0; }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof Fare)) return false;
        Fare other = (Fare) obj;
        return Objects.equals(baseAmount, other.baseAmount)
            && Double.compare(surgeMultiplier, other.surgeMultiplier) == 0;
    }

    @Override
    public int hashCode() { return Objects.hash(baseAmount, surgeMultiplier); }
}

// ==========================================
// RIDE STATUS STATE MACHINE
// ==========================================

public enum RideStatus {
    REQUESTED,      // Rider has requested a ride
    DRIVER_ASSIGNED, // A driver has accepted the request
    DRIVER_EN_ROUTE, // Driver is traveling to the pickup location
    RIDER_PICKED_UP, // Driver has picked up the rider
    IN_PROGRESS,    // Ride is ongoing
    COMPLETED,      // Ride has ended, fare settled
    CANCELLED       // Either party cancelled
}

public class RideStateMachine {

    private static final Map<RideStatus, Set<RideStatus>> VALID_TRANSITIONS = new EnumMap<>(RideStatus.class);

    static {
        VALID_TRANSITIONS.put(RideStatus.REQUESTED,
            EnumSet.of(RideStatus.DRIVER_ASSIGNED, RideStatus.CANCELLED));

        VALID_TRANSITIONS.put(RideStatus.DRIVER_ASSIGNED,
            EnumSet.of(RideStatus.DRIVER_EN_ROUTE, RideStatus.CANCELLED));

        VALID_TRANSITIONS.put(RideStatus.DRIVER_EN_ROUTE,
            EnumSet.of(RideStatus.RIDER_PICKED_UP, RideStatus.CANCELLED));

        VALID_TRANSITIONS.put(RideStatus.RIDER_PICKED_UP,
            EnumSet.of(RideStatus.IN_PROGRESS));

        VALID_TRANSITIONS.put(RideStatus.IN_PROGRESS,
            EnumSet.of(RideStatus.COMPLETED));

        VALID_TRANSITIONS.put(RideStatus.COMPLETED, EnumSet.noneOf(RideStatus.class));
        VALID_TRANSITIONS.put(RideStatus.CANCELLED, EnumSet.noneOf(RideStatus.class));
    }

    public static void assertTransitionAllowed(RideStatus from, RideStatus to) {
        if (!VALID_TRANSITIONS.getOrDefault(from, EnumSet.noneOf(RideStatus.class)).contains(to)) {
            throw new IllegalStateTransitionException(from + " -> " + to);
        }
    }
}

// ==========================================
// AGGREGATES
// ==========================================

// Ride is the central aggregate in the ride-sharing domain
public class Ride {

    private final RideId id;
    private final RiderId riderId;
    private DriverId driverId;           // assigned after matching
    private final Location pickup;
    private final Location dropoff;
    private RideStatus status;
    private Fare fare;
    private Location currentDriverLocation;
    private Instant requestedAt;
    private Instant driverAssignedAt;
    private Instant riderPickedUpAt;
    private Instant completedAt;
    private String cancellationReason;
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public Ride(RideId id, RiderId riderId, Location pickup, Location dropoff) {
        Objects.requireNonNull(id, "Ride ID required");
        Objects.requireNonNull(riderId, "Rider ID required");
        Objects.requireNonNull(pickup, "Pickup location required");
        Objects.requireNonNull(dropoff, "Dropoff location required");

        if (pickup.equals(dropoff)) {
            throw new DomainException("Pickup and dropoff cannot be the same location");
        }

        this.id = id;
        this.riderId = riderId;
        this.pickup = pickup;
        this.dropoff = dropoff;
        this.status = RideStatus.REQUESTED;
        this.requestedAt = Instant.now();

        domainEvents.add(new RideRequested(id, riderId, pickup, dropoff, requestedAt));
    }

    public void assignDriver(DriverId driverId, Fare agreedFare) {
        RideStateMachine.assertTransitionAllowed(status, RideStatus.DRIVER_ASSIGNED);
        Objects.requireNonNull(driverId, "Driver ID required");
        Objects.requireNonNull(agreedFare, "Agreed fare required");

        this.driverId = driverId;
        this.fare = agreedFare;
        this.status = RideStatus.DRIVER_ASSIGNED;
        this.driverAssignedAt = Instant.now();

        domainEvents.add(new DriverAssigned(id, riderId, driverId, agreedFare, driverAssignedAt));
    }

    public void driverEnRoute(Location driverCurrentLocation) {
        RideStateMachine.assertTransitionAllowed(status, RideStatus.DRIVER_EN_ROUTE);
        this.currentDriverLocation = driverCurrentLocation;
        this.status = RideStatus.DRIVER_EN_ROUTE;

        domainEvents.add(new DriverEnRoute(id, driverId, riderId, driverCurrentLocation, Instant.now()));
    }

    public void pickUpRider() {
        RideStateMachine.assertTransitionAllowed(status, RideStatus.RIDER_PICKED_UP);
        this.status = RideStatus.RIDER_PICKED_UP;
        this.riderPickedUpAt = Instant.now();

        domainEvents.add(new RiderPickedUp(id, riderId, driverId, pickup, riderPickedUpAt));
    }

    public void startRide() {
        RideStateMachine.assertTransitionAllowed(status, RideStatus.IN_PROGRESS);
        this.status = RideStatus.IN_PROGRESS;
    }

    public void completeRide() {
        RideStateMachine.assertTransitionAllowed(status, RideStatus.COMPLETED);
        this.status = RideStatus.COMPLETED;
        this.completedAt = Instant.now();

        domainEvents.add(new RideCompleted(id, riderId, driverId, fare, completedAt));
    }

    public void cancel(String reason) {
        RideStateMachine.assertTransitionAllowed(status, RideStatus.CANCELLED);
        Objects.requireNonNull(reason, "Cancellation reason required");

        this.status = RideStatus.CANCELLED;
        this.cancellationReason = reason;

        domainEvents.add(new RideCancelled(id, riderId, driverId, reason, Instant.now()));
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }

    public RideId getId() { return id; }
    public RiderId getRiderId() { return riderId; }
    public DriverId getDriverId() { return driverId; }
    public Location getPickup() { return pickup; }
    public Location getDropoff() { return dropoff; }
    public RideStatus getStatus() { return status; }
    public Fare getFare() { return fare; }
}

// Driver — a separate aggregate
public class Driver {

    private final DriverId id;
    private final String name;
    private DriverStatus status;
    private Location currentLocation;
    private double rating;
    private int totalRides;
    private Vehicle vehicle;

    public Driver(DriverId id, String name, Vehicle vehicle) {
        this.id = id;
        this.name = name;
        this.vehicle = vehicle;
        this.status = DriverStatus.OFFLINE;
        this.rating = 5.0;
        this.totalRides = 0;
    }

    public void goOnline(Location startingLocation) {
        if (status == DriverStatus.ON_RIDE) {
            throw new DomainException("Driver cannot go online while on a ride");
        }
        this.status = DriverStatus.AVAILABLE;
        this.currentLocation = startingLocation;
    }

    public void goOffline() {
        if (status == DriverStatus.ON_RIDE) {
            throw new DomainException("Driver cannot go offline while on a ride");
        }
        this.status = DriverStatus.OFFLINE;
    }

    public void acceptRide() {
        if (status != DriverStatus.AVAILABLE) {
            throw new DomainException("Driver must be AVAILABLE to accept a ride");
        }
        this.status = DriverStatus.ON_RIDE;
    }

    public void completeRide(double riderRating) {
        if (status != DriverStatus.ON_RIDE) {
            throw new DomainException("Driver must be ON_RIDE to complete a ride");
        }
        this.totalRides++;
        this.rating = ((this.rating * (totalRides - 1)) + riderRating) / totalRides;
        this.status = DriverStatus.AVAILABLE;
    }

    public void updateLocation(Location location) {
        this.currentLocation = location;
    }

    public boolean isAvailable() { return status == DriverStatus.AVAILABLE; }

    public DriverId getId() { return id; }
    public String getName() { return name; }
    public DriverStatus getStatus() { return status; }
    public Location getCurrentLocation() { return currentLocation; }
    public double getRating() { return rating; }
    public Vehicle getVehicle() { return vehicle; }
}

// ==========================================
// DOMAIN EVENTS
// ==========================================

public final class RideRequested extends BaseDomainEvent {
    private final RideId rideId;
    private final RiderId riderId;
    private final Location pickup;
    private final Location dropoff;
    private final Instant requestedAt;

    public RideRequested(RideId rideId, RiderId riderId, Location pickup, Location dropoff, Instant requestedAt) {
        super();
        this.rideId = rideId;
        this.riderId = riderId;
        this.pickup = pickup;
        this.dropoff = dropoff;
        this.requestedAt = requestedAt;
    }

    @Override
    public String getEventType() { return "ride.requested"; }

    public RideId getRideId() { return rideId; }
    public RiderId getRiderId() { return riderId; }
    public Location getPickup() { return pickup; }
    public Location getDropoff() { return dropoff; }
    public Instant getRequestedAt() { return requestedAt; }
}

public final class RideCompleted extends BaseDomainEvent {
    private final RideId rideId;
    private final RiderId riderId;
    private final DriverId driverId;
    private final Fare fare;
    private final Instant completedAt;

    public RideCompleted(RideId rideId, RiderId riderId, DriverId driverId, Fare fare, Instant completedAt) {
        super();
        this.rideId = rideId;
        this.riderId = riderId;
        this.driverId = driverId;
        this.fare = fare;
        this.completedAt = completedAt;
    }

    @Override
    public String getEventType() { return "ride.completed"; }

    public RideId getRideId() { return rideId; }
    public RiderId getRiderId() { return riderId; }
    public DriverId getDriverId() { return driverId; }
    public Fare getFare() { return fare; }
    public Instant getCompletedAt() { return completedAt; }
}
```

---

## 11. Common Domain Modeling Mistakes

### Database-Driven Modeling

The most pervasive mistake: letting the database schema drive the domain model. Engineers who start by designing tables — then generate or write entity classes that map one-to-one to those tables — produce a model that serves the database, not the domain.

Signs of database-driven modeling:
- Every entity has a nullable field that is only populated in certain states.
- You find `JOIN` logic leaking into domain methods.
- Domain concepts that should be separate entities share a table (discriminator columns).
- The entity class name matches the table name exactly: `OrdersEntity`, `CustomersEntity`.

Correction: design the domain model first. Map it to a schema afterward, accepting that the mapping may be complex. Tools like JPA are designed to bridge the gap between object models and relational schemas.

### Using Primitives Instead of Value Objects

The second most common mistake: using `String`, `BigDecimal`, `int`, and `double` for domain concepts.

```java
// BAD: what does this method do? What is valid for orderId? What currency is price in?
public void placeOrder(String orderId, String customerId, double price) { ... }

// GOOD: the types carry the domain meaning and enforce validity
public void placeOrder(OrderId orderId, CustomerId customerId, Money price) { ... }
```

Primitives have no validation, no behavior, and no domain meaning. `String email` can hold anything. `EmailAddress email` holds only a valid email. The compiler catches mistakes. Tests are simpler. The code is more readable.

### Bloated Aggregates

An aggregate that grows too large becomes a consistency and concurrency bottleneck. If every operation on any entity within the aggregate locks the aggregate root, high-traffic systems will serialize writes.

Signs of a bloated aggregate:
- The aggregate root has dozens of fields.
- Multiple distinct lifecycle states exist within the same aggregate.
- Business rules from multiple subdomains are enforced by one class.
- Tests for the aggregate require enormous setup.

Correction: identify the minimum set of objects that must change together to satisfy an invariant. That is the aggregate. If a `Customer` and their `Wishlist` do not have invariants that span both, they should be separate aggregates. The `Wishlist` refers to the `Customer` by ID, not by reference.

### Missing Domain Events

When important things happen in the domain — an order is placed, a payment fails, a driver completes a ride — and no event is recorded, the system loses the ability to react to these facts without polling or coupling.

Missing domain events lead to:
- Direct service-to-service calls that create coupling.
- No audit trail of what happened and when.
- Difficulty adding new reactions to domain facts (sending a notification requires modifying the core domain logic).

### God Entities

A `User` class with 80 fields and 40 methods, handling everything from authentication to billing to preferences. A `Product` class that knows about inventory, pricing, promotions, reviews, and shipping.

God entities violate the Single Responsibility Principle at the domain level. They make the bounded context concept meaningless because one class spans multiple concerns.

Correction: split along domain boundaries. A `User` in the authentication context becomes `Credentials`. The same `User` in the billing context becomes `Account`. The same person in the marketing context becomes `Contact`.

### Anemic Models in Complex Domains

Covered in detail in section 3. In complex domains with significant invariants, the anemic model pattern distributes business logic in ways that make it invisible, unenforceable, and fragile.

---

## 12. Domain Modeling in Interview Settings

### How to Approach "Design This System" from a Domain Perspective

The standard interview advice is to start with functional requirements, draw boxes and arrows, and discuss databases. Senior engineers approach design differently: they start with the domain.

When given "design a parking lot system" or "design an online bookstore", the first question is not "what tables do we need?" It is "what are the core concepts of this domain?"

### Step-by-Step: Identify Entities, Value Objects, Aggregates, Services, Events

Walk through this sequence explicitly in interviews. It demonstrates a systematic, senior-level approach.

1. **Listen and restate the domain**. "So we're building a parking management system. The core concern is managing the assignment of vehicles to parking spots and billing for time used."

2. **Identify entities**. Ask: what has identity that persists over time? In a parking lot: `ParkingSpot`, `Vehicle`, `ParkingSession`, `ParkingLot`.

3. **Identify value objects**. Ask: what describes something without needing its own identity? `LicensePlate`, `SpotNumber`, `Fee`, `Duration`.

4. **Identify aggregates**. Ask: which entities form a consistency unit? `ParkingSession` owns the start time, vehicle, spot assignment, and end time — it is the aggregate for a single parking transaction. `ParkingLot` is the aggregate root for spot management.

5. **Identify domain services**. Ask: what operations span multiple aggregates? `FeeCalculationService` spans `ParkingSession` and the fee schedule. `SpotAssignmentService` spans `ParkingLot` and `ParkingSession`.

6. **Identify key invariants**. "A ParkingSpot can only be occupied by one vehicle at a time." "A ParkingSession cannot end before it starts." "An unstarted session cannot be billed."

7. **Identify domain events**. What are the significant facts? `VehicleEntered`, `VehicleExited`, `PaymentCollected`.

### What Interviewers Want to See

- You distinguish between entity identity and value object equality.
- You know where to enforce invariants (in the aggregate, not in the service).
- You use the domain's language, not generic CRUD terminology.
- You identify aggregates based on consistency requirements, not convenience.
- You know what a domain event is and why it matters.
- You avoid putting business logic in the database or in utility classes.

### Example Walkthrough: Parking Lot Domain Model

```java
// ==========================================
// VALUE OBJECTS
// ==========================================

public final class LicensePlate {
    private final String value;
    private final String state;

    public LicensePlate(String value, String state) {
        if (value == null || value.isBlank()) throw new IllegalArgumentException("License plate value required");
        if (state == null || state.isBlank()) throw new IllegalArgumentException("State required");
        this.value = value.trim().toUpperCase();
        this.state = state.trim().toUpperCase();
    }

    public String getFullIdentifier() { return state + "-" + value; }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof LicensePlate)) return false;
        LicensePlate other = (LicensePlate) obj;
        return Objects.equals(value, other.value) && Objects.equals(state, other.state);
    }

    @Override
    public int hashCode() { return Objects.hash(value, state); }
}

public final class SpotNumber {
    private final String level;
    private final int row;
    private final int position;

    public SpotNumber(String level, int row, int position) {
        if (level == null || level.isBlank()) throw new IllegalArgumentException("Level required");
        if (row <= 0 || position <= 0) throw new IllegalArgumentException("Row and position must be positive");
        this.level = level;
        this.row = row;
        this.position = position;
    }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof SpotNumber)) return false;
        SpotNumber other = (SpotNumber) obj;
        return Objects.equals(level, other.level) && row == other.row && position == other.position;
    }

    @Override
    public int hashCode() { return Objects.hash(level, row, position); }

    @Override
    public String toString() { return level + "-" + row + "-" + position; }
}

// ==========================================
// ENTITIES
// ==========================================

public class ParkingSpot {

    private final ParkingSpotId id;
    private final SpotNumber spotNumber;
    private final SpotSize size;
    private SpotStatus status;
    private ParkingSessionId currentSessionId;

    public ParkingSpot(ParkingSpotId id, SpotNumber spotNumber, SpotSize size) {
        this.id = id;
        this.spotNumber = spotNumber;
        this.size = size;
        this.status = SpotStatus.AVAILABLE;
    }

    // Called by ParkingLot aggregate root — not publicly accessible
    void occupy(ParkingSessionId sessionId) {
        if (status != SpotStatus.AVAILABLE) {
            throw new DomainException("Spot " + spotNumber + " is not available");
        }
        this.status = SpotStatus.OCCUPIED;
        this.currentSessionId = sessionId;
    }

    void vacate() {
        if (status != SpotStatus.OCCUPIED) {
            throw new DomainException("Spot " + spotNumber + " is not occupied");
        }
        this.status = SpotStatus.AVAILABLE;
        this.currentSessionId = null;
    }

    public boolean canAccommodate(VehicleSize vehicleSize) {
        return size.canAccommodate(vehicleSize);
    }

    public ParkingSpotId getId() { return id; }
    public SpotNumber getSpotNumber() { return spotNumber; }
    public SpotSize getSize() { return size; }
    public SpotStatus getStatus() { return status; }
    public boolean isAvailable() { return status == SpotStatus.AVAILABLE; }
}

// ==========================================
// AGGREGATE ROOT
// ==========================================

public class ParkingLot {

    private final ParkingLotId id;
    private final String name;
    private final List<ParkingSpot> spots;
    private final int capacity;
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public ParkingLot(ParkingLotId id, String name, List<ParkingSpot> spots) {
        this.id = id;
        this.name = name;
        this.spots = new ArrayList<>(spots);
        this.capacity = spots.size();
    }

    // Invariant: one vehicle per spot
    public ParkingSpot assignSpotForVehicle(Vehicle vehicle, ParkingSessionId sessionId) {
        ParkingSpot availableSpot = spots.stream()
            .filter(spot -> spot.isAvailable() && spot.canAccommodate(vehicle.getSize()))
            .findFirst()
            .orElseThrow(() -> new DomainException("No available spots for vehicle size: " + vehicle.getSize()));

        availableSpot.occupy(sessionId);
        domainEvents.add(new SpotAssigned(id, availableSpot.getId(), vehicle.getLicensePlate(), sessionId, Instant.now()));

        return availableSpot;
    }

    public void releaseSpot(ParkingSpotId spotId) {
        ParkingSpot spot = spots.stream()
            .filter(s -> s.getId().equals(spotId))
            .findFirst()
            .orElseThrow(() -> new DomainException("Spot not found: " + spotId));

        spot.vacate();
        domainEvents.add(new SpotReleased(id, spotId, Instant.now()));
    }

    public int getAvailableSpotCount() {
        return (int) spots.stream().filter(ParkingSpot::isAvailable).count();
    }

    public boolean isFull() {
        return getAvailableSpotCount() == 0;
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }

    public ParkingLotId getId() { return id; }
    public String getName() { return name; }
    public int getCapacity() { return capacity; }
}

// ==========================================
// DOMAIN SERVICE
// ==========================================

public class ParkingFeeCalculationService {

    private final FeeSchedule feeSchedule;

    public ParkingFeeCalculationService(FeeSchedule feeSchedule) {
        this.feeSchedule = feeSchedule;
    }

    public Money calculateFee(ParkingSession session) {
        if (!session.hasEnded()) {
            throw new DomainException("Cannot calculate fee for an active parking session");
        }
        Duration duration = session.getDuration();
        return feeSchedule.calculateFor(duration, session.getSpotSize());
    }
}
```

---

## 13. Senior vs Mid-Level Thinking

| Dimension | Mid-Level Engineer | Senior Engineer |
|---|---|---|
| Starting point | "What tables do we need?" | "What are the domain concepts and their relationships?" |
| Entity design | Entities are data containers with getters and setters | Entities own their behavior and enforce their invariants |
| Value objects | Uses String, int, double for everything | Identifies domain concepts that warrant their own types |
| Invariant enforcement | Checks in service layer methods | Invariants live inside the aggregate and cannot be bypassed |
| Domain events | Not used; side effects are direct method calls | Events record facts; consumers are decoupled |
| Aggregate boundaries | Not explicitly considered; aggregates span the entire domain | Aggregates are kept small, based on invariant scope |
| Business rules location | Rules are scattered: in controllers, services, validators, stored procedures | Rules have a single, explicit home in the domain model |
| Ubiquitous language | Technical naming: UserManager, processRecord, updateState | Domain naming: MembershipService, renewMembership, activate |
| Bounded contexts | One shared model for the entire system | Explicit context boundaries, each with its own model |
| Testing approach | Tests require entire service stack | Aggregate invariants are tested without dependencies |
| Refactoring | Renames are cosmetic changes | Renames reveal domain insights; driven by improved understanding |
| Complexity handling | Adds more if/else and service methods | Introduces specifications, policies, state machines |
| Repository design | DAO with findAll, findById, insert, update, delete | Collection-like interface with domain-meaningful queries |
| New feature cost | Each new feature requires understanding the entire codebase | New features find a natural home in the existing domain model |
| Code communication | Code requires comments to explain intent | Code reads like the domain expert's description |

### The Defining Question

Mid-level engineers ask: "Where do I put this code?"

Senior engineers ask: "What domain concept does this logic belong to, and have I modeled that concept correctly?"

The second question leads to a model that grows in a controlled, intentional direction. Features that belong together stay together. Rules are enforced where they are owned. The language of the code matches the language of the domain. When a domain expert describes a new requirement, an experienced senior engineer can trace it directly to the part of the model that must change — because the model was designed to match the domain, not the database.

---

## Summary

Domain modeling is the art of expressing the business problem accurately in code. The DDD building blocks — entities, value objects, aggregates, domain services, repositories, and domain events — provide a vocabulary for designing models that are cohesive, explicit, and aligned with the language of the business.

Key principles to carry forward:

- Model the domain first. Schema and APIs follow.
- Enforce invariants in the aggregate. They belong nowhere else.
- Use value objects to eliminate primitive obsession and make validation implicit.
- Keep aggregates small. Consistency boundaries, not convenience, define them.
- Name everything in the language of the domain. Naming is design.
- Record domain events. Side effects belong in event handlers, not in aggregate methods.
- Respect bounded context boundaries. Different contexts get different models.
- Prefer rich domain models in complex domains. Anemic models push complexity where it doesn't belong.

The test of a good domain model is whether a domain expert can read the code and recognize it as an accurate description of their business.
