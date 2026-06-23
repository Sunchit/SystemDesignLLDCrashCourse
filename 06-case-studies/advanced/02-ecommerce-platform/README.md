# E-Commerce Platform — Low-Level Design Case Study

> **Difficulty**: Advanced
> **Primary Patterns**: Strategy, Observer, Composite, Factory, Template Method
> **Secondary Patterns**: Builder, Decorator, Chain of Responsibility, State
> **Language**: Java 17

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Assumptions and Constraints](#4-assumptions-and-constraints)
5. [Domain Model Description](#5-domain-model-description)
6. [Class Diagram](#6-class-diagram)
7. [Core Classes — Complete Java Implementation](#7-core-classes--complete-java-implementation)
8. [Design Patterns Used](#8-design-patterns-used)
9. [Key Design Decisions and Trade-offs](#9-key-design-decisions-and-trade-offs)
10. [Extension Points](#10-extension-points)
11. [Interview Discussion Points](#11-interview-discussion-points)
12. [Common Mistakes in This Design](#12-common-mistakes-in-this-design)

---

## 1. Problem Statement

Design the core backend of an e-commerce platform — similar to Amazon or Flipkart — that supports the complete customer shopping journey: browsing a product catalog organized into categories, managing a shopping cart, placing orders, processing payments, and receiving notifications. The platform must handle product variants (size, color, etc.), real-time inventory management to prevent overselling, flexible discount and coupon logic, and multiple payment methods.

The design must be extensible enough that new payment gateways, discount strategies, and notification channels can be plugged in without touching existing code. Inventory must be reserved at order placement time and only decremented on successful payment, so that a payment failure releases the hold and the item becomes available again.

This is a rich domain with complex business rules: an order transitions through well-defined states; coupons have eligibility conditions; category hierarchies are arbitrarily deep; and a single product can have many variants each with their own price and stock level.

---

## 2. Functional Requirements

1. A **User** can register, log in, and maintain a profile with one or more saved addresses.
2. An **Admin** can create and manage the product catalog, including categories and subcategories to arbitrary depth.
3. Each **Product** belongs to exactly one **Category** and can have one or more **ProductVariants** (e.g., "Red / XL", "Blue / M"), each with its own price and SKU.
4. The system shall maintain an **Inventory** count per variant; inventory may be updated by admins and is decremented automatically upon successful order placement.
5. A **User** can add ProductVariants to a **ShoppingCart**, update quantities, and remove items.
6. The cart shall display a running subtotal, and applying a coupon shall update the displayed discounted total in real time.
7. A **User** can apply a single **Coupon** to the cart; the system shall validate the coupon's expiry, minimum order value, applicable categories, and usage limits before accepting it.
8. A **User** can place an **Order** from their cart, selecting a saved delivery address and a payment method.
9. The system shall **reserve** inventory for all items in the order at placement time to prevent overselling; if any item is out of stock, the entire order shall be rejected with a clear per-item error.
10. The system shall process **Payment** through pluggable payment gateways (Credit Card, Debit Card, UPI, Wallet, COD).
11. On successful payment, the order status shall advance to `CONFIRMED` and inventory reservations shall be converted to permanent decrements.
12. On payment failure, the order shall move to `PAYMENT_FAILED`, inventory reservations shall be released, and the user shall be notified.
13. An **Order** shall progress through the following states: `PENDING` → `CONFIRMED` → `PROCESSING` → `SHIPPED` → `DELIVERED`; cancellation is only allowed before `SHIPPED`.
14. Users and Admins shall receive **Notifications** (Email, SMS, Push) at key lifecycle events: order placed, payment confirmed, order shipped, order delivered, and payment failed.
15. An **Admin** can view all orders and update order status (e.g., mark as SHIPPED with a tracking number).
16. A **User** can view their complete order history with item details and status.

---

## 3. Non-Functional Requirements

| Quality Attribute | Requirement |
|---|---|
| **Correctness** | All monetary amounts stored as `long` in the smallest currency unit (paise/cents) to eliminate floating-point errors |
| **Consistency** | Inventory reservation and order creation happen atomically; partial reservations are never committed |
| **Extensibility** | Payment gateways, discount strategies, and notification channels implement defined interfaces; new ones require zero changes to existing code |
| **Maintainability** | Order state transitions are enforced by an explicit state machine; no scattered `if (status == X)` guards across services |
| **Testability** | All external dependencies (payment gateway, notification sender) hidden behind interfaces; fully mockable |
| **Auditability** | Every order state transition is timestamped and stored in a transition log |
| **Data Integrity** | A coupon's `usageCount` is updated atomically; concurrent redemption cannot exceed `maxUsage` |
| **Scalability** | Domain model makes no assumptions about storage; repository interfaces allow swap between in-memory and persistent stores |

---

## 4. Assumptions and Constraints

- **Currency**: All amounts are in Indian Paise (1 INR = 100 Paise). The `Money` class wraps a `long` amount and a `Currency` enum.
- **Single coupon per order**: Only one coupon may be applied per cart at a time.
- **Cart ownership**: A shopping cart belongs to exactly one user and persists until the user checks out or explicitly clears it.
- **Inventory granularity**: Stock is tracked at the `ProductVariant` level, not at the `Product` level.
- **Reservation TTL**: Inventory reservations expire after 30 minutes if payment is not completed. TTL enforcement is out of scope for this design (would be handled by a scheduler in production).
- **Payment atomicity**: We model the payment processing as synchronous for simplicity. A production system would use sagas or two-phase commit with a payment service.
- **Address immutability on order**: Once an order is placed, a snapshot of the delivery address is stored on the order; changing the user's saved address later does not affect existing orders.
- **Thread safety**: Not in scope for this LLD. A production system would use optimistic locking on `InventoryItem.reservedQuantity` and `Coupon.usageCount`.
- **Authentication/Authorization**: UserRole is modeled but the authentication mechanism (JWT, sessions) is out of scope.
- **Returns and refunds**: Out of scope for this design.
- **Ratings and reviews**: Out of scope for this design.
- **Search and recommendations**: Out of scope for this design.

---

## 5. Domain Model Description

The domain is organized around six bounded contexts:

**User and Identity**
The `User` entity holds profile information, a `UserRole` (CUSTOMER or ADMIN), and a list of saved `Address` objects. An `Address` is a value object — it has no identity of its own and is meaningful only in the context of a user or an order. The `Money` value object wraps a `long` amount and a `Currency` to ensure all arithmetic is safe and explicit.

**Catalog**
The `Category` entity uses the **Composite pattern**: a category can contain both leaf categories and other categories, enabling an arbitrarily deep tree (e.g., Electronics > Phones > Smartphones). A `Product` belongs to one `Category`, holds shared metadata (name, description, brand), and aggregates one or more `ProductVariant` objects. Each `ProductVariant` represents a unique, sellable combination of attributes (e.g., colour + size) and carries its own `SKU`, `Money` price, and a reference to its `InventoryItem`.

**Inventory**
`InventoryItem` is tightly coupled 1-to-1 with a `ProductVariant`. It tracks `availableQuantity` and `reservedQuantity` separately. Reserving stock increments `reservedQuantity` and decrements `availableQuantity`; committing a sale decrements both. This two-phase model is the key mechanism that prevents overselling.

**Cart**
`ShoppingCart` is a per-user aggregate that holds a list of `CartItem` objects. Each `CartItem` references a `ProductVariant` and a quantity. The cart applies a `Coupon` and exposes a computed `total` that respects the discount. Adding an item to the cart does NOT yet reserve inventory — reservation happens only at order placement.

**Order**
`Order` is the central transactional aggregate. It captures a snapshot of `CartItem` objects at checkout time, the delivery `Address` snapshot, the chosen `PaymentMethod`, and the `OrderStatus` state. `OrderStatusTransition` logs each status change with a timestamp and optional note (e.g., a tracking number). The `Payment` entity is associated 1-to-1 with an order and captures the gateway reference, `PaymentStatus`, and amount charged.

**Discounts and Coupons**
`Coupon` holds the discount definition (`DiscountType`: PERCENTAGE or FLAT) plus eligibility rules: expiry date, minimum order value, applicable category IDs, and max usage count. The discount calculation is delegated to a **Strategy** object keyed by `DiscountType`.

**Notifications**
`NotificationEvent` carries the event type (`NotificationType`), the recipient user, and a payload map. `NotificationSender` is an interface implemented per channel (Email, SMS, Push). The `NotificationService` uses the **Observer** pattern — order state changes publish events that the notification observers consume.

---

## 6. Class Diagram

```
+---------------------------+          +---------------------------+
|          User             |          |         Address           |
+---------------------------+          +---------------------------+
| - id: String              |1      *  | - street: String          |
| - name: String            |--------->| - city: String            |
| - email: String           |          | - state: String           |
| - passwordHash: String    |          | - pinCode: String         |
| - role: UserRole          |          | - country: String         |
| - addresses: List<Address>|          +---------------------------+
+---------------------------+

+---------------------------+       +----------------------------+
|        Category           |       |          Product           |
+---------------------------+       +----------------------------+
| - id: String              |1    * | - id: String               |
| - name: String            |<------| - name: String             |
| - parent: Category        |       | - description: String      |
| - children: List<Category>|       | - brand: String            |
|   <<Composite Pattern>>   |       | - category: Category       |
+---------------------------+       | - variants: List<PV>       |
                                    | - active: boolean          |
                                    +----------------------------+
                                              | 1
                                              | has many
                                              v *
                                    +----------------------------+
                                    |      ProductVariant        |
                                    +----------------------------+
                                    | - id: String               |
                                    | - sku: String              |
                                    | - attributes: Map<String,  |
                                    |               String>      |
                                    | - price: Money             |
                                    | - product: Product         |
                                    +----------------------------+
                                              | 1
                                              | tracked by
                                              v 1
                                    +----------------------------+
                                    |       InventoryItem        |
                                    +----------------------------+
                                    | - variantId: String        |
                                    | - availableQuantity: int   |
                                    | - reservedQuantity: int    |
                                    +----------------------------+

+---------------------------+       +----------------------------+
|       ShoppingCart        |       |          CartItem          |
+---------------------------+       +----------------------------+
| - id: String              |1    * | - variant: ProductVariant  |
| - user: User              |------>| - quantity: int            |
| - items: List<CartItem>   |       | - unitPrice: Money         |
| - appliedCoupon: Coupon   |       +----------------------------+
+---------------------------+

+---------------------------+       +----------------------------+
|          Order            |       |     OrderStatusTransition  |
+---------------------------+       +----------------------------+
| - id: String              |1    * | - fromStatus: OrderStatus  |
| - user: User              |------>| - toStatus: OrderStatus    |
| - items: List<CartItem>   |       | - timestamp: LocalDateTime |
| - deliveryAddress: Address|       | - note: String             |
| - status: OrderStatus     |       +----------------------------+
| - payment: Payment        |
| - coupon: Coupon          |       +----------------------------+
| - transitions: List<OST>  |       |          Payment           |
+---------------------------+       +----------------------------+
         | 1                        | - id: String               |
         |                          | - order: Order             |
         v 1                        | - amount: Money            |
+---------------------------+       | - method: PaymentMethod    |
|          Payment          |       | - status: PaymentStatus    |
+---------------------------+       | - gatewayReference: String |
                                    | - paidAt: LocalDateTime    |
                                    +----------------------------+

+---------------------------+
|          Coupon           |
+---------------------------+
| - code: String            |
| - discountType:           |
|     DiscountType          |
| - discountValue: Money    |
| - minOrderValue: Money    |
| - expiryDate: LocalDate   |
| - maxUsage: int           |
| - usageCount: int         |
| - applicableCategories:   |
|     Set<String>           |
+---------------------------+

Enums:
  OrderStatus:   PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, PAYMENT_FAILED
  PaymentStatus: PENDING, SUCCESS, FAILED, REFUNDED
  PaymentMethod: CREDIT_CARD, DEBIT_CARD, UPI, WALLET, CASH_ON_DELIVERY
  UserRole:      CUSTOMER, ADMIN
  NotifType:     ORDER_PLACED, PAYMENT_SUCCESS, PAYMENT_FAILED, ORDER_SHIPPED, ORDER_DELIVERED, ORDER_CANCELLED
  DiscountType:  PERCENTAGE, FLAT_AMOUNT
```

---

## 7. Core Classes — Complete Java Implementation

### 7.1 Enums

```java
// OrderStatus.java
package com.ecommerce.model.enums;

/**
 * Represents all possible states an Order can be in.
 * Valid transitions:
 *   PENDING       -> CONFIRMED, PAYMENT_FAILED, CANCELLED
 *   CONFIRMED     -> PROCESSING, CANCELLED
 *   PROCESSING    -> SHIPPED
 *   SHIPPED       -> DELIVERED
 *   DELIVERED     -> (terminal)
 *   CANCELLED     -> (terminal)
 *   PAYMENT_FAILED -> (terminal — user must place a new order)
 */
public enum OrderStatus {
    PENDING,
    CONFIRMED,
    PROCESSING,
    SHIPPED,
    DELIVERED,
    CANCELLED,
    PAYMENT_FAILED;

    public boolean isTerminal() {
        return this == DELIVERED || this == CANCELLED || this == PAYMENT_FAILED;
    }

    public boolean canTransitionTo(OrderStatus next) {
        switch (this) {
            case PENDING:
                return next == CONFIRMED || next == PAYMENT_FAILED || next == CANCELLED;
            case CONFIRMED:
                return next == PROCESSING || next == CANCELLED;
            case PROCESSING:
                return next == SHIPPED;
            case SHIPPED:
                return next == DELIVERED;
            default:
                return false; // DELIVERED, CANCELLED, PAYMENT_FAILED are terminal
        }
    }
}
```

```java
// PaymentStatus.java
package com.ecommerce.model.enums;

/**
 * Lifecycle of a Payment attempt.
 */
public enum PaymentStatus {
    /** Payment has been initiated but not yet confirmed by the gateway. */
    PENDING,
    /** Gateway confirmed the charge succeeded. */
    SUCCESS,
    /** Gateway rejected or timed out. */
    FAILED,
    /** Payment was successfully reversed (post-cancellation). */
    REFUNDED
}
```

```java
// PaymentMethod.java
package com.ecommerce.model.enums;

/**
 * Payment instruments supported by the platform.
 */
public enum PaymentMethod {
    CREDIT_CARD,
    DEBIT_CARD,
    UPI,
    WALLET,
    CASH_ON_DELIVERY;

    /**
     * Cash on Delivery does not require upfront payment processing.
     */
    public boolean requiresOnlineProcessing() {
        return this != CASH_ON_DELIVERY;
    }
}
```

```java
// UserRole.java
package com.ecommerce.model.enums;

/**
 * Roles that govern what operations a User may perform.
 */
public enum UserRole {
    CUSTOMER,
    ADMIN
}
```

```java
// NotificationType.java
package com.ecommerce.model.enums;

/**
 * Events that trigger outbound notifications to users.
 */
public enum NotificationType {
    ORDER_PLACED,
    PAYMENT_SUCCESS,
    PAYMENT_FAILED,
    ORDER_SHIPPED,
    ORDER_DELIVERED,
    ORDER_CANCELLED
}
```

```java
// DiscountType.java
package com.ecommerce.model.enums;

/**
 * Determines how a Coupon's discountValue is interpreted.
 */
public enum DiscountType {
    /** discountValue is a percentage, e.g. 10 means 10% off. */
    PERCENTAGE,
    /** discountValue is an absolute amount to subtract, stored in the smallest currency unit. */
    FLAT_AMOUNT
}
```

---

### 7.2 Custom Exceptions

```java
// InsufficientInventoryException.java
package com.ecommerce.exception;

/**
 * Thrown when placing an order and one or more ProductVariants do not have
 * enough available inventory to satisfy the requested quantity.
 */
public class InsufficientInventoryException extends RuntimeException {

    private final String variantId;
    private final int requested;
    private final int available;

    public InsufficientInventoryException(String variantId, int requested, int available) {
        super(String.format(
            "Insufficient inventory for variant '%s': requested %d, available %d.",
            variantId, requested, available
        ));
        this.variantId = variantId;
        this.requested = requested;
        this.available = available;
    }

    public String getVariantId() {
        return variantId;
    }

    public int getRequested() {
        return requested;
    }

    public int getAvailable() {
        return available;
    }
}
```

```java
// InvalidCouponException.java
package com.ecommerce.exception;

/**
 * Thrown when a coupon code is invalid, expired, already exhausted,
 * or the cart does not meet the coupon's eligibility conditions.
 */
public class InvalidCouponException extends RuntimeException {

    private final String couponCode;

    public InvalidCouponException(String couponCode, String reason) {
        super(String.format("Coupon '%s' is invalid: %s", couponCode, reason));
        this.couponCode = couponCode;
    }

    public String getCouponCode() {
        return couponCode;
    }
}
```

```java
// OrderNotFoundException.java
package com.ecommerce.exception;

/**
 * Thrown when an order lookup by ID returns no result.
 */
public class OrderNotFoundException extends RuntimeException {

    private final String orderId;

    public OrderNotFoundException(String orderId) {
        super(String.format("Order with id '%s' was not found.", orderId));
        this.orderId = orderId;
    }

    public String getOrderId() {
        return orderId;
    }
}
```

```java
// PaymentFailedException.java
package com.ecommerce.exception;

/**
 * Thrown when the payment gateway rejects a payment attempt.
 * Carries the gateway's own error code for logging and display.
 */
public class PaymentFailedException extends RuntimeException {

    private final String orderId;
    private final String gatewayErrorCode;

    public PaymentFailedException(String orderId, String gatewayErrorCode, String message) {
        super(String.format(
            "Payment failed for order '%s' (gateway code: %s): %s",
            orderId, gatewayErrorCode, message
        ));
        this.orderId = orderId;
        this.gatewayErrorCode = gatewayErrorCode;
    }

    public String getOrderId() {
        return orderId;
    }

    public String getGatewayErrorCode() {
        return gatewayErrorCode;
    }
}
```

```java
// ProductNotFoundException.java
package com.ecommerce.exception;

/**
 * Thrown when a product or product variant lookup by ID returns no result.
 */
public class ProductNotFoundException extends RuntimeException {

    private final String productId;

    public ProductNotFoundException(String productId) {
        super(String.format("Product or variant with id '%s' was not found.", productId));
        this.productId = productId;
    }

    public String getProductId() {
        return productId;
    }
}
```

```java
// InvalidOrderStateException.java
package com.ecommerce.exception;

import com.ecommerce.model.enums.OrderStatus;

/**
 * Thrown when an operation is attempted that is illegal in the order's current state,
 * or when a requested status transition is not permitted by the state machine.
 */
public class InvalidOrderStateException extends RuntimeException {

    private final String orderId;
    private final OrderStatus currentStatus;
    private final OrderStatus requestedStatus;

    /**
     * Use this constructor for illegal transitions.
     */
    public InvalidOrderStateException(String orderId, OrderStatus current, OrderStatus requested) {
        super(String.format(
            "Cannot transition order '%s' from %s to %s.",
            orderId, current, requested
        ));
        this.orderId = orderId;
        this.currentStatus = current;
        this.requestedStatus = requested;
    }

    /**
     * Use this constructor for operations that are illegal in the current state
     * (e.g., cancelling a DELIVERED order).
     */
    public InvalidOrderStateException(String orderId, OrderStatus current, String operation) {
        super(String.format(
            "Operation '%s' is not allowed for order '%s' in state %s.",
            operation, orderId, current
        ));
        this.orderId = orderId;
        this.currentStatus = current;
        this.requestedStatus = null;
    }

    public String getOrderId() {
        return orderId;
    }

    public OrderStatus getCurrentStatus() {
        return currentStatus;
    }

    public OrderStatus getRequestedStatus() {
        return requestedStatus;
    }
}
```

---

### 7.3 Core Model / Entity Classes

```java
// Address.java
package com.ecommerce.model;

import java.util.Objects;

/**
 * Value object representing a physical delivery address.
 * Address objects are immutable by design — to change an address, create a new one.
 * When stored on an Order, it is a snapshot that must not be affected by later
 * changes to the user's saved addresses.
 */
public class Address {

    private final String street;
    private final String city;
    private final String state;
    private final String pinCode;
    private final String country;

    public Address(String street, String city, String state, String pinCode, String country) {
        if (street == null || street.isBlank())   throw new IllegalArgumentException("Street must not be blank.");
        if (city == null || city.isBlank())        throw new IllegalArgumentException("City must not be blank.");
        if (state == null || state.isBlank())      throw new IllegalArgumentException("State must not be blank.");
        if (pinCode == null || pinCode.isBlank())  throw new IllegalArgumentException("PinCode must not be blank.");
        if (country == null || country.isBlank())  throw new IllegalArgumentException("Country must not be blank.");
        this.street  = street;
        this.city    = city;
        this.state   = state;
        this.pinCode = pinCode;
        this.country = country;
    }

    public String getStreet()  { return street;  }
    public String getCity()    { return city;    }
    public String getState()   { return state;   }
    public String getPinCode() { return pinCode; }
    public String getCountry() { return country; }

    /** Creates a deep copy (safe to embed as an immutable snapshot on an Order). */
    public Address snapshot() {
        return new Address(street, city, state, pinCode, country);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Address)) return false;
        Address a = (Address) o;
        return Objects.equals(street, a.street)
            && Objects.equals(city, a.city)
            && Objects.equals(state, a.state)
            && Objects.equals(pinCode, a.pinCode)
            && Objects.equals(country, a.country);
    }

    @Override
    public int hashCode() {
        return Objects.hash(street, city, state, pinCode, country);
    }

    @Override
    public String toString() {
        return street + ", " + city + ", " + state + " - " + pinCode + ", " + country;
    }
}
```

```java
// Money.java
package com.ecommerce.model;

import java.util.Objects;

/**
 * Value object representing a monetary amount in the smallest currency unit
 * (e.g., paise for INR, cents for USD).
 *
 * All arithmetic is performed in long to avoid floating-point imprecision.
 * Money objects are immutable; arithmetic operations return new instances.
 */
public class Money {

    public static final String INR = "INR";
    public static final String USD = "USD";

    private final long amount;     // in smallest unit (paise / cents)
    private final String currency;

    public static final Money ZERO_INR = new Money(0L, INR);
    public static final Money ZERO_USD = new Money(0L, USD);

    public Money(long amount, String currency) {
        if (amount < 0)               throw new IllegalArgumentException("Monetary amount cannot be negative.");
        if (currency == null || currency.isBlank()) throw new IllegalArgumentException("Currency must not be blank.");
        this.amount   = amount;
        this.currency = currency;
    }

    public long getAmount()    { return amount;   }
    public String getCurrency(){ return currency; }

    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount + other.amount, this.currency);
    }

    public Money subtract(Money other) {
        assertSameCurrency(other);
        if (other.amount > this.amount) {
            throw new IllegalArgumentException(
                "Cannot subtract " + other + " from " + this + ": result would be negative.");
        }
        return new Money(this.amount - other.amount, this.currency);
    }

    public Money multiply(int factor) {
        if (factor < 0) throw new IllegalArgumentException("Multiplication factor cannot be negative.");
        return new Money(this.amount * factor, this.currency);
    }

    /**
     * Returns this amount reduced by the given percentage.
     * @param percentage integer 0-100
     */
    public Money applyPercentageDiscount(int percentage) {
        if (percentage < 0 || percentage > 100)
            throw new IllegalArgumentException("Percentage must be between 0 and 100.");
        long discount = (this.amount * percentage) / 100;
        return new Money(this.amount - discount, this.currency);
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return this.amount > other.amount;
    }

    public boolean isGreaterThanOrEqual(Money other) {
        assertSameCurrency(other);
        return this.amount >= other.amount;
    }

    public boolean isZero() {
        return this.amount == 0;
    }

    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Currency mismatch: " + this.currency + " vs " + other.currency);
        }
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money m = (Money) o;
        return amount == m.amount && Objects.equals(currency, m.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }

    @Override
    public String toString() {
        // Display as decimal: 10050 paise -> "100.50 INR"
        return String.format("%.2f %s", amount / 100.0, currency);
    }
}
```

```java
// User.java
package com.ecommerce.model;

import com.ecommerce.model.enums.UserRole;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * Represents a registered platform user — either a Customer or an Admin.
 */
public class User {

    private final String id;
    private String name;
    private String email;
    private String passwordHash;   // BCrypt hash; never the raw password
    private final UserRole role;
    private final List<Address> addresses;
    private final LocalDateTime createdAt;
    private boolean active;

    public User(String id, String name, String email, String passwordHash, UserRole role) {
        Objects.requireNonNull(id,           "User id must not be null.");
        Objects.requireNonNull(name,         "Name must not be null.");
        Objects.requireNonNull(email,        "Email must not be null.");
        Objects.requireNonNull(passwordHash, "Password hash must not be null.");
        Objects.requireNonNull(role,         "UserRole must not be null.");
        this.id           = id;
        this.name         = name;
        this.email        = email;
        this.passwordHash = passwordHash;
        this.role         = role;
        this.addresses    = new ArrayList<>();
        this.createdAt    = LocalDateTime.now();
        this.active       = true;
    }

    public void addAddress(Address address) {
        Objects.requireNonNull(address, "Address must not be null.");
        addresses.add(address);
    }

    public boolean removeAddress(Address address) {
        return addresses.remove(address);
    }

    public boolean isAdmin() {
        return role == UserRole.ADMIN;
    }

    public boolean isActive() {
        return active;
    }

    public void deactivate() {
        this.active = false;
    }

    // --- Getters ---
    public String getId()           { return id;           }
    public String getName()         { return name;         }
    public String getEmail()        { return email;        }
    public String getPasswordHash() { return passwordHash; }
    public UserRole getRole()       { return role;         }
    public LocalDateTime getCreatedAt() { return createdAt; }

    public List<Address> getAddresses() {
        return Collections.unmodifiableList(addresses);
    }

    // --- Setters ---
    public void setName(String name) {
        if (name == null || name.isBlank()) throw new IllegalArgumentException("Name must not be blank.");
        this.name = name;
    }

    public void setEmail(String email) {
        if (email == null || !email.contains("@")) throw new IllegalArgumentException("Invalid email.");
        this.email = email;
    }

    public void setPasswordHash(String passwordHash) {
        Objects.requireNonNull(passwordHash, "Password hash must not be null.");
        this.passwordHash = passwordHash;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        return Objects.equals(id, ((User) o).id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

    @Override
    public String toString() {
        return "User{id='" + id + "', name='" + name + "', email='" + email + "', role=" + role + "}";
    }
}
```

```java
// Category.java
package com.ecommerce.model;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * Represents a product category node in an arbitrarily deep hierarchy.
 *
 * Implements the COMPOSITE PATTERN:
 *   - A leaf category has no children (getChildren() returns empty list).
 *   - A composite category has one or more children.
 *
 * Both leaf and composite are treated uniformly as Category.
 * All navigation (parent, children) is held by the Category itself, enabling
 * traversal without knowing whether you're at a leaf or branch.
 *
 * Example tree:
 *   Electronics
 *     ├── Phones
 *     │     ├── Smartphones
 *     │     └── Feature Phones
 *     └── Laptops
 */
public class Category {

    private final String id;
    private String name;
    private String description;
    private Category parent;                   // null for root categories
    private final List<Category> children;
    private boolean active;

    public Category(String id, String name, String description) {
        Objects.requireNonNull(id,   "Category id must not be null.");
        Objects.requireNonNull(name, "Category name must not be null.");
        this.id          = id;
        this.name        = name;
        this.description = description;
        this.children    = new ArrayList<>();
        this.active      = true;
    }

    // --- Composite operations ---

    public void addChild(Category child) {
        Objects.requireNonNull(child, "Child category must not be null.");
        if (child == this) throw new IllegalArgumentException("A category cannot be its own child.");
        child.parent = this;
        children.add(child);
    }

    public boolean removeChild(Category child) {
        boolean removed = children.remove(child);
        if (removed) child.parent = null;
        return removed;
    }

    public boolean isLeaf() {
        return children.isEmpty();
    }

    public boolean isRoot() {
        return parent == null;
    }

    /**
     * Returns the full path from root to this category as a slash-separated string.
     * Example: "Electronics / Phones / Smartphones"
     */
    public String getFullPath() {
        if (parent == null) return name;
        return parent.getFullPath() + " / " + name;
    }

    /**
     * Returns all descendant category IDs (including this category's own ID).
     * Useful for coupon eligibility checks — a coupon valid for "Electronics"
     * should also apply to any product in "Smartphones".
     */
    public List<String> getAllDescendantIds() {
        List<String> ids = new ArrayList<>();
        collectDescendantIds(this, ids);
        return ids;
    }

    private void collectDescendantIds(Category node, List<String> ids) {
        ids.add(node.id);
        for (Category child : node.children) {
            collectDescendantIds(child, ids);
        }
    }

    // --- Getters ---
    public String getId()          { return id;          }
    public String getName()        { return name;        }
    public String getDescription() { return description; }
    public Category getParent()    { return parent;      }
    public boolean isActive()      { return active;      }

    public List<Category> getChildren() {
        return Collections.unmodifiableList(children);
    }

    // --- Setters ---
    public void setName(String name) {
        if (name == null || name.isBlank()) throw new IllegalArgumentException("Category name must not be blank.");
        this.name = name;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public void deactivate() {
        this.active = false;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Category)) return false;
        return Objects.equals(id, ((Category) o).id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

    @Override
    public String toString() {
        return "Category{id='" + id + "', name='" + name + "', path='" + getFullPath() + "'}";
    }
}
```

```java
// ProductVariant.java
package com.ecommerce.model;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Objects;

/**
 * Represents one specific, sellable configuration of a Product.
 *
 * A ProductVariant identifies itself with a unique SKU and a map of
 * attribute name → value pairs, e.g.:
 *   { "color": "Red", "size": "XL" }
 *
 * Each variant has its own price and is linked 1-to-1 with an InventoryItem.
 */
public class ProductVariant {

    private final String id;
    private final String sku;
    private final Map<String, String> attributes;  // e.g. color->Red, size->XL
    private Money price;
    private final Product product;                 // back-reference to owning Product
    private boolean active;

    public ProductVariant(String id, String sku, Map<String, String> attributes,
                          Money price, Product product) {
        Objects.requireNonNull(id,         "Variant id must not be null.");
        Objects.requireNonNull(sku,        "SKU must not be null.");
        Objects.requireNonNull(attributes, "Attributes map must not be null.");
        Objects.requireNonNull(price,      "Price must not be null.");
        Objects.requireNonNull(product,    "Parent product must not be null.");
        this.id         = id;
        this.sku        = sku;
        this.attributes = new LinkedHashMap<>(attributes);
        this.price      = price;
        this.product    = product;
        this.active     = true;
    }

    /**
     * Returns a human-readable variant descriptor built from attributes.
     * Example: "Color: Red, Size: XL"
     */
    public String getDisplayLabel() {
        StringBuilder sb = new StringBuilder();
        attributes.forEach((k, v) -> {
            if (sb.length() > 0) sb.append(", ");
            sb.append(k).append(": ").append(v);
        });
        return sb.toString();
    }

    // --- Getters ---
    public String getId()      { return id;     }
    public String getSku()     { return sku;    }
    public Money getPrice()    { return price;  }
    public Product getProduct(){ return product;}
    public boolean isActive()  { return active; }

    public Map<String, String> getAttributes() {
        return Collections.unmodifiableMap(attributes);
    }

    public String getAttribute(String key) {
        return attributes.get(key);
    }

    // --- Setters ---
    public void setPrice(Money price) {
        Objects.requireNonNull(price, "Price must not be null.");
        this.price = price;
    }

    public void deactivate() {
        this.active = false;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ProductVariant)) return false;
        return Objects.equals(id, ((ProductVariant) o).id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

    @Override
    public String toString() {
        return "ProductVariant{id='" + id + "', sku='" + sku + "', label='" + getDisplayLabel()
             + "', price=" + price + "}";
    }
}
```

```java
// Product.java
package com.ecommerce.model;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.Optional;

/**
 * Represents a product in the catalog.
 *
 * A Product is the aggregate root for ProductVariants. Products hold shared
 * metadata (name, description, brand) while each ProductVariant holds the
 * variant-specific price and SKU.
 *
 * A product must always have at least one active variant to be purchasable.
 */
public class Product {

    private final String id;
    private String name;
    private String description;
    private String brand;
    private Category category;
    private final List<ProductVariant> variants;
    private final LocalDateTime createdAt;
    private boolean active;

    public Product(String id, String name, String description, String brand, Category category) {
        Objects.requireNonNull(id,       "Product id must not be null.");
        Objects.requireNonNull(name,     "Product name must not be null.");
        Objects.requireNonNull(category, "Category must not be null.");
        this.id          = id;
        this.name        = name;
        this.description = description;
        this.brand       = brand;
        this.category    = category;
        this.variants    = new ArrayList<>();
        this.createdAt   = LocalDateTime.now();
        this.active      = true;
    }

    public void addVariant(ProductVariant variant) {
        Objects.requireNonNull(variant, "Variant must not be null.");
        if (!variant.getProduct().equals(this)) {
            throw new IllegalArgumentException("Variant does not belong to this product.");
        }
        variants.add(variant);
    }

    /**
     * Finds a variant by its ID.
     */
    public Optional<ProductVariant> findVariantById(String variantId) {
        return variants.stream()
                       .filter(v -> v.getId().equals(variantId))
                       .findFirst();
    }

    /**
     * Returns all active variants.
     */
    public List<ProductVariant> getActiveVariants() {
        return variants.stream()
                       .filter(ProductVariant::isActive)
                       .collect(java.util.stream.Collectors.toList());
    }

    /**
     * A product is purchasable only when both the product itself and at least
     * one of its variants are active.
     */
    public boolean isPurchasable() {
        return active && !getActiveVariants().isEmpty();
    }

    // --- Getters ---
    public String getId()            { return id;          }
    public String getName()          { return name;        }
    public String getDescription()   { return description; }
    public String getBrand()         { return brand;       }
    public Category getCategory()    { return category;    }
    public LocalDateTime getCreatedAt(){ return createdAt; }
    public boolean isActive()        { return active;      }

    public List<ProductVariant> getVariants() {
        return Collections.unmodifiableList(variants);
    }

    // --- Setters ---
    public void setName(String name) {
        if (name == null || name.isBlank()) throw new IllegalArgumentException("Product name must not be blank.");
        this.name = name;
    }

    public void setDescription(String description) { this.description = description; }
    public void setBrand(String brand)             { this.brand = brand;             }

    public void setCategory(Category category) {
        Objects.requireNonNull(category, "Category must not be null.");
        this.category = category;
    }

    public void deactivate() {
        this.active = false;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Product)) return false;
        return Objects.equals(id, ((Product) o).id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

    @Override
    public String toString() {
        return "Product{id='" + id + "', name='" + name + "', brand='" + brand
             + "', category=" + category.getName() + ", variants=" + variants.size() + "}";
    }
}
```

```java
// InventoryItem.java
package com.ecommerce.model;

import com.ecommerce.exception.InsufficientInventoryException;

import java.util.Objects;

/**
 * Tracks stock levels for a single ProductVariant using a two-phase reserve/commit model.
 *
 * Two-phase inventory flow:
 *   1. reserve(qty)  — called at Order placement. Moves stock from available to reserved.
 *   2. commit(qty)   — called after payment succeeds. Decrements reserved (permanent sale).
 *   3. release(qty)  — called on payment failure or order cancellation. Returns reserved to available.
 *
 * The invariant: availableQuantity + reservedQuantity <= totalStock at all times.
 *
 * This design prevents overselling: two concurrent orders cannot both reserve the last unit
 * because the second reserve() will see availableQuantity == 0.
 */
public class InventoryItem {

    private final String variantId;
    private int availableQuantity;
    private int reservedQuantity;

    public InventoryItem(String variantId, int initialQuantity) {
        Objects.requireNonNull(variantId, "variantId must not be null.");
        if (initialQuantity < 0) throw new IllegalArgumentException("Initial quantity cannot be negative.");
        this.variantId         = variantId;
        this.availableQuantity = initialQuantity;
        this.reservedQuantity  = 0;
    }

    /**
     * Reserves {@code quantity} units for a pending order.
     * Moves stock from available to reserved bucket.
     *
     * @throws InsufficientInventoryException if available stock is below requested quantity.
     */
    public synchronized void reserve(int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException("Reserve quantity must be positive.");
        if (quantity > availableQuantity) {
            throw new InsufficientInventoryException(variantId, quantity, availableQuantity);
        }
        availableQuantity -= quantity;
        reservedQuantity  += quantity;
    }

    /**
     * Commits a previously reserved quantity as a completed sale.
     * Called after successful payment. Decrements reserved bucket permanently.
     *
     * @throws IllegalStateException if reserved stock is less than requested.
     */
    public synchronized void commit(int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException("Commit quantity must be positive.");
        if (quantity > reservedQuantity) {
            throw new IllegalStateException(
                "Cannot commit " + quantity + " units for variant '" + variantId
                + "': only " + reservedQuantity + " units are reserved.");
        }
        reservedQuantity -= quantity;
    }

    /**
     * Releases a previously reserved quantity back to available.
     * Called when payment fails or an order is cancelled before payment.
     *
     * @throws IllegalStateException if reserved stock is less than requested.
     */
    public synchronized void release(int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException("Release quantity must be positive.");
        if (quantity > reservedQuantity) {
            throw new IllegalStateException(
                "Cannot release " + quantity + " units for variant '" + variantId
                + "': only " + reservedQuantity + " units are reserved.");
        }
        reservedQuantity  -= quantity;
        availableQuantity += quantity;
    }

    /**
     * Adds newly stocked units directly to available quantity (admin restocking).
     */
    public synchronized void addStock(int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException("Stock addition quantity must be positive.");
        availableQuantity += quantity;
    }

    public int getTotalStock() {
        return availableQuantity + reservedQuantity;
    }

    public boolean isInStock() {
        return availableQuantity > 0;
    }

    // --- Getters ---
    public String getVariantId()        { return variantId;         }
    public int getAvailableQuantity()   { return availableQuantity; }
    public int getReservedQuantity()    { return reservedQuantity;  }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof InventoryItem)) return false;
        return Objects.equals(variantId, ((InventoryItem) o).variantId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(variantId);
    }

    @Override
    public String toString() {
        return "InventoryItem{variantId='" + variantId
             + "', available=" + availableQuantity
             + ", reserved=" + reservedQuantity + "}";
    }
}
```

```java
// CartItem.java
package com.ecommerce.model;

import java.util.Objects;

/**
 * Represents a single line item in a ShoppingCart or a placed Order.
 *
 * CartItem captures the ProductVariant and the requested quantity. The
 * unitPrice is snapshotted at the time the item is added to the cart so
 * that price changes to the variant after cart addition do not silently
 * alter the cart total.
 *
 * NOTE: CartItem is a shared structure: the same CartItem type is used both
 * in ShoppingCart (mutable, quantity can change) and in Order (immutable
 * snapshot after checkout). The immutability contract on Order is enforced
 * by OrderService, which stores an unmodifiable list.
 */
public class CartItem {

    private final ProductVariant variant;
    private int quantity;
    private final Money unitPrice;  // price snapshot at time of cart addition

    public CartItem(ProductVariant variant, int quantity) {
        Objects.requireNonNull(variant, "ProductVariant must not be null.");
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive.");
        this.variant   = variant;
        this.quantity  = quantity;
        this.unitPrice = variant.getPrice(); // snapshot
    }

    /**
     * Updates the quantity of this line item.
     * Only valid for cart usage; Order items should not be mutated.
     */
    public void setQuantity(int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive.");
        this.quantity = quantity;
    }

    /**
     * Returns the line total: unitPrice * quantity.
     */
    public Money getLineTotal() {
        return unitPrice.multiply(quantity);
    }

    // --- Getters ---
    public ProductVariant getVariant()  { return variant;   }
    public int getQuantity()            { return quantity;  }
    public Money getUnitPrice()         { return unitPrice; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CartItem)) return false;
        CartItem ci = (CartItem) o;
        return Objects.equals(variant, ci.variant);
    }

    @Override
    public int hashCode() {
        return Objects.hash(variant);
    }

    @Override
    public String toString() {
        return "CartItem{variant=" + variant.getSku()
             + ", qty=" + quantity
             + ", unitPrice=" + unitPrice
             + ", lineTotal=" + getLineTotal() + "}";
    }
}
```

```java
// ShoppingCart.java
package com.ecommerce.model;

import com.ecommerce.exception.ProductNotFoundException;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.Optional;

/**
 * Represents a User's active shopping cart.
 *
 * Business rules enforced here:
 *   - Adding the same variant twice increases the quantity of the existing item.
 *   - Removing an item that does not exist is a no-op (idempotent).
 *   - Applying a coupon replaces any previously applied coupon.
 *   - getTotal() always reflects the discounted price when a coupon is applied.
 *
 * The cart does NOT reserve inventory — that happens only at order placement.
 */
public class ShoppingCart {

    private final String id;
    private final User user;
    private final List<CartItem> items;
    private Coupon appliedCoupon;

    public ShoppingCart(String id, User user) {
        Objects.requireNonNull(id,   "Cart id must not be null.");
        Objects.requireNonNull(user, "User must not be null.");
        this.id    = id;
        this.user  = user;
        this.items = new ArrayList<>();
    }

    /**
     * Adds a variant to the cart with the specified quantity.
     * If the variant is already in the cart, the quantity is incremented.
     */
    public void addItem(ProductVariant variant, int quantity) {
        Objects.requireNonNull(variant, "Variant must not be null.");
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive.");

        Optional<CartItem> existing = findItemByVariantId(variant.getId());
        if (existing.isPresent()) {
            existing.get().setQuantity(existing.get().getQuantity() + quantity);
        } else {
            items.add(new CartItem(variant, quantity));
        }
    }

    /**
     * Updates the quantity of an existing cart item.
     *
     * @throws ProductNotFoundException if no item with the given variantId exists.
     */
    public void updateItemQuantity(String variantId, int newQuantity) {
        if (newQuantity <= 0) throw new IllegalArgumentException("Quantity must be positive.");
        CartItem item = findItemByVariantId(variantId)
                .orElseThrow(() -> new ProductNotFoundException(variantId));
        item.setQuantity(newQuantity);
    }

    /**
     * Removes the cart item for the given variantId.
     * If no such item exists, this is a no-op.
     */
    public void removeItem(String variantId) {
        items.removeIf(i -> i.getVariant().getId().equals(variantId));
    }

    /**
     * Applies a coupon to the cart, replacing any previously applied coupon.
     * Validation (expiry, minimum order value, etc.) must be done by CartService
     * before calling this method.
     */
    public void applyCoupon(Coupon coupon) {
        Objects.requireNonNull(coupon, "Coupon must not be null.");
        this.appliedCoupon = coupon;
    }

    /**
     * Removes any applied coupon from the cart.
     */
    public void removeCoupon() {
        this.appliedCoupon = null;
    }

    /**
     * Returns the subtotal before any discount: sum of all line totals.
     */
    public Money getSubtotal() {
        if (items.isEmpty()) return Money.ZERO_INR;
        return items.stream()
                    .map(CartItem::getLineTotal)
                    .reduce(Money.ZERO_INR, Money::add);
    }

    /**
     * Returns the total after applying the coupon discount, or the subtotal if no coupon.
     */
    public Money getTotal() {
        Money subtotal = getSubtotal();
        if (appliedCoupon == null) return subtotal;
        return appliedCoupon.applyDiscount(subtotal);
    }

    /**
     * Returns the discount amount: subtotal - total.
     */
    public Money getDiscountAmount() {
        return getSubtotal().subtract(getTotal());
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }

    public void clear() {
        items.clear();
        appliedCoupon = null;
    }

    private Optional<CartItem> findItemByVariantId(String variantId) {
        return items.stream()
                    .filter(i -> i.getVariant().getId().equals(variantId))
                    .findFirst();
    }

    // --- Getters ---
    public String getId()                   { return id;            }
    public User getUser()                   { return user;          }
    public Coupon getAppliedCoupon()        { return appliedCoupon; }
    public boolean hasCoupon()              { return appliedCoupon != null; }

    public List<CartItem> getItems() {
        return Collections.unmodifiableList(items);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ShoppingCart)) return false;
        return Objects.equals(id, ((ShoppingCart) o).id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

    @Override
    public String toString() {
        return "ShoppingCart{id='" + id + "', user=" + user.getEmail()
             + ", items=" + items.size()
             + ", subtotal=" + getSubtotal()
             + ", total=" + getTotal()
             + (appliedCoupon != null ? ", coupon='" + appliedCoupon.getCode() + "'" : "")
             + "}";
    }
}
```

```java
// Coupon.java
package com.ecommerce.model;

import com.ecommerce.exception.InvalidCouponException;
import com.ecommerce.model.enums.DiscountType;

import java.time.LocalDate;
import java.util.Collections;
import java.util.HashSet;
import java.util.Objects;
import java.util.Set;

/**
 * Represents a promotional coupon that can be applied to a cart.
 *
 * Eligibility rules:
 *   1. Coupon must not be expired (expiryDate >= today).
 *   2. Cart subtotal must meet the minimum order value.
 *   3. If applicableCategories is non-empty, at least one cart item must belong
 *      to a category in that set (or a descendant of it).
 *   4. usageCount must be below maxUsage (0 = unlimited).
 *
 * Discount calculation:
 *   PERCENTAGE  — reduce subtotal by discountValue percent.
 *   FLAT_AMOUNT — subtract discountValue from subtotal (floor at zero).
 */
public class Coupon {

    private final String id;
    private final String code;
    private final DiscountType discountType;
    private final Money discountValue;          // % value or flat amount
    private final Money minOrderValue;          // minimum cart subtotal to apply
    private final LocalDate expiryDate;
    private final int maxUsage;                 // 0 = unlimited
    private int usageCount;
    private final Set<String> applicableCategories; // empty = all categories
    private boolean active;

    public Coupon(String id, String code, DiscountType discountType, Money discountValue,
                  Money minOrderValue, LocalDate expiryDate, int maxUsage,
                  Set<String> applicableCategories) {
        Objects.requireNonNull(id,            "Coupon id must not be null.");
        Objects.requireNonNull(code,          "Coupon code must not be null.");
        Objects.requireNonNull(discountType,  "DiscountType must not be null.");
        Objects.requireNonNull(discountValue, "Discount value must not be null.");
        Objects.requireNonNull(minOrderValue, "Min order value must not be null.");
        Objects.requireNonNull(expiryDate,    "Expiry date must not be null.");
        if (maxUsage < 0) throw new IllegalArgumentException("maxUsage cannot be negative.");

        if (discountType == DiscountType.PERCENTAGE) {
            if (discountValue.getAmount() <= 0 || discountValue.getAmount() > 100)
                throw new IllegalArgumentException("Percentage discount must be between 1 and 100.");
        }

        this.id                    = id;
        this.code                  = code.toUpperCase();
        this.discountType          = discountType;
        this.discountValue         = discountValue;
        this.minOrderValue         = minOrderValue;
        this.expiryDate            = expiryDate;
        this.maxUsage              = maxUsage;
        this.usageCount            = 0;
        this.applicableCategories  = applicableCategories != null
                                     ? new HashSet<>(applicableCategories)
                                     : new HashSet<>();
        this.active                = true;
    }

    /**
     * Validates this coupon against the given cart subtotal and the set of
     * category IDs present in the cart (all ancestors included).
     *
     * @param subtotal        the cart subtotal before discount
     * @param cartCategoryIds all category IDs touched by the cart (including ancestors)
     * @param today           the current date (injected for testability)
     * @throws InvalidCouponException if any eligibility rule fails
     */
    public void validate(Money subtotal, Set<String> cartCategoryIds, LocalDate today) {
        if (!active) {
            throw new InvalidCouponException(code, "coupon is inactive.");
        }
        if (today.isAfter(expiryDate)) {
            throw new InvalidCouponException(code,
                "coupon expired on " + expiryDate + ".");
        }
        if (maxUsage > 0 && usageCount >= maxUsage) {
            throw new InvalidCouponException(code,
                "coupon usage limit of " + maxUsage + " has been reached.");
        }
        if (subtotal.isGreaterThan(Money.ZERO_INR) && !subtotal.isGreaterThanOrEqual(minOrderValue)) {
            throw new InvalidCouponException(code,
                "minimum order value of " + minOrderValue + " not met (cart total: " + subtotal + ").");
        }
        if (!applicableCategories.isEmpty()) {
            boolean categoryMatch = cartCategoryIds.stream()
                                                   .anyMatch(applicableCategories::contains);
            if (!categoryMatch) {
                throw new InvalidCouponException(code,
                    "coupon is not applicable to any item in the current cart.");
            }
        }
    }

    /**
     * Applies this coupon's discount to the given subtotal and returns the discounted total.
     * Does NOT validate eligibility — call validate() first.
     */
    public Money applyDiscount(Money subtotal) {
        switch (discountType) {
            case PERCENTAGE:
                return subtotal.applyPercentageDiscount((int) discountValue.getAmount());
            case FLAT_AMOUNT:
                if (subtotal.isGreaterThan(discountValue)) {
                    return subtotal.subtract(discountValue);
                }
                return new Money(0L, subtotal.getCurrency()); // discount can't exceed total
            default:
                throw new IllegalStateException("Unknown DiscountType: " + discountType);
        }
    }

    /**
     * Increments the usage count atomically (in production, use DB-level locking).
     */
    public synchronized void incrementUsage() {
        if (maxUsage > 0 && usageCount >= maxUsage) {
            throw new InvalidCouponException(code, "coupon usage limit exceeded.");
        }
        usageCount++;
    }

    public void deactivate() {
        this.active = false;
    }

    // --- Getters ---
    public String getId()                          { return id;                   }
    public String getCode()                        { return code;                 }
    public DiscountType getDiscountType()          { return discountType;         }
    public Money getDiscountValue()                { return discountValue;        }
    public Money getMinOrderValue()                { return minOrderValue;        }
    public LocalDate getExpiryDate()               { return expiryDate;           }
    public int getMaxUsage()                       { return maxUsage;             }
    public int getUsageCount()                     { return usageCount;           }
    public boolean isActive()                      { return active;               }

    public Set<String> getApplicableCategories() {
        return Collections.unmodifiableSet(applicableCategories);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Coupon)) return false;
        return Objects.equals(id, ((Coupon) o).id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

    @Override
    public String toString() {
        return "Coupon{code='" + code + "', type=" + discountType
             + ", value=" + discountValue + ", expiry=" + expiryDate
             + ", usage=" + usageCount + "/" + (maxUsage == 0 ? "unlimited" : maxUsage) + "}";
    }
}
```

---

### 7.4 Order and Pricing Classes

```java
// OrderItem.java
package com.ecommerce.model;

import java.util.Objects;

/**
 * Represents a single line item in a placed Order.
 *
 * OrderItem is an immutable snapshot created at order placement time.
 * It captures all information needed to reconstruct the order line
 * independently of the current state of the product catalog:
 *   - identifiers for the product and variant (for returns / history lookups)
 *   - human-readable names snapshotted at checkout (product and variant names
 *     may be edited later in the catalog without affecting historical orders)
 *   - quantity and unit price as they were at checkout
 *   - the seller's ID so marketplace revenue splits can be computed
 *   - a delivery address snapshot (allows per-item drop-shipping in future)
 *
 * totalPrice is always unitPrice * quantity and is stored explicitly to avoid
 * recomputing across currency boundaries and to make order history reads fast.
 */
public final class OrderItem {

    private final String productId;
    private final String variantId;
    private final String sellerId;
    private final String productName;
    private final String variantDescription;   // e.g. "Color: Red, Size: XL"
    private final int quantity;
    private final Money unitPrice;
    private final Money totalPrice;            // unitPrice * quantity
    private final Address deliveryAddressSnapshot;

    /**
     * Constructs an immutable OrderItem. All fields are required.
     *
     * @param productId              catalog product ID
     * @param variantId              catalog variant ID
     * @param sellerId               seller / merchant ID (marketplace support)
     * @param productName            human-readable product name at time of order
     * @param variantDescription     human-readable variant label at time of order
     * @param quantity               number of units ordered (must be >= 1)
     * @param unitPrice              per-unit price at time of order
     * @param deliveryAddressSnapshot deep copy of the delivery address for this item
     */
    public OrderItem(String productId,
                     String variantId,
                     String sellerId,
                     String productName,
                     String variantDescription,
                     int quantity,
                     Money unitPrice,
                     Address deliveryAddressSnapshot) {
        Objects.requireNonNull(productId,              "productId must not be null.");
        Objects.requireNonNull(variantId,              "variantId must not be null.");
        Objects.requireNonNull(sellerId,               "sellerId must not be null.");
        Objects.requireNonNull(productName,            "productName must not be null.");
        Objects.requireNonNull(variantDescription,     "variantDescription must not be null.");
        Objects.requireNonNull(unitPrice,              "unitPrice must not be null.");
        Objects.requireNonNull(deliveryAddressSnapshot,"deliveryAddressSnapshot must not be null.");
        if (quantity < 1) throw new IllegalArgumentException("Quantity must be at least 1.");

        this.productId              = productId;
        this.variantId              = variantId;
        this.sellerId               = sellerId;
        this.productName            = productName;
        this.variantDescription     = variantDescription;
        this.quantity               = quantity;
        this.unitPrice              = unitPrice;
        this.totalPrice             = unitPrice.multiply(quantity);
        this.deliveryAddressSnapshot = deliveryAddressSnapshot.snapshot();
    }

    // --- Getters (no setters — fully immutable) ---

    public String getProductId()              { return productId;              }
    public String getVariantId()              { return variantId;              }
    public String getSellerId()               { return sellerId;               }
    public String getProductName()            { return productName;            }
    public String getVariantDescription()     { return variantDescription;     }
    public int getQuantity()                  { return quantity;               }
    public Money getUnitPrice()               { return unitPrice;              }
    public Money getTotalPrice()              { return totalPrice;             }
    public Address getDeliveryAddressSnapshot(){ return deliveryAddressSnapshot; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof OrderItem)) return false;
        OrderItem other = (OrderItem) o;
        return quantity == other.quantity
            && Objects.equals(variantId,  other.variantId)
            && Objects.equals(unitPrice,  other.unitPrice);
    }

    @Override
    public int hashCode() {
        return Objects.hash(variantId, quantity, unitPrice);
    }

    @Override
    public String toString() {
        return "OrderItem{product='" + productName + "', variant='" + variantDescription
             + "', qty=" + quantity + ", unit=" + unitPrice + ", total=" + totalPrice + "}";
    }
}
```

```java
// Order.java
package com.ecommerce.model;

import com.ecommerce.exception.InvalidOrderStateException;
import com.ecommerce.model.enums.OrderStatus;
import com.ecommerce.model.enums.PaymentMethod;
import com.ecommerce.model.enums.PaymentStatus;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * Central transactional aggregate representing a placed order.
 *
 * Design decisions:
 *   - Built exclusively via the inner {@link Builder} to prevent partially-constructed instances.
 *   - All monetary fields (subtotal, discountAmount, taxAmount, totalAmount) are computed by
 *     {@link #calculateTotals()} and stored explicitly so order history reads are fast.
 *   - Status transitions are enforced by the state machine in {@link OrderStatus#canTransitionTo}.
 *     Callers must use {@link #transitionTo(OrderStatus)} rather than mutating status directly.
 *   - statusHistory is a human-readable audit log; each entry is a timestamped string.
 *   - Address fields hold deep snapshots (via Address#snapshot()) so later profile changes
 *     do not silently alter historical orders.
 */
public class Order {

    private final String orderId;
    private final String customerId;
    private final List<OrderItem> items;           // immutable after construction
    private final Address shippingAddress;
    private final Address billingAddress;
    private OrderStatus status;
    private PaymentStatus paymentStatus;
    private final PaymentMethod paymentMethod;
    private Money subtotal;
    private Money discountAmount;
    private Money taxAmount;
    private Money totalAmount;
    private final String couponCode;               // nullable — null when no coupon applied
    private String trackingNumber;                 // set when order is shipped
    private final LocalDateTime placedAt;
    private LocalDateTime updatedAt;
    private final List<String> statusHistory;      // timestamped audit log

    // Private constructor — use Builder
    private Order(Builder builder) {
        this.orderId         = builder.orderId;
        this.customerId      = builder.customerId;
        this.items           = Collections.unmodifiableList(new ArrayList<>(builder.items));
        this.shippingAddress = builder.shippingAddress.snapshot();
        this.billingAddress  = builder.billingAddress.snapshot();
        this.status          = builder.status;
        this.paymentStatus   = builder.paymentStatus;
        this.paymentMethod   = builder.paymentMethod;
        this.subtotal        = builder.subtotal;
        this.discountAmount  = builder.discountAmount;
        this.taxAmount       = builder.taxAmount;
        this.totalAmount     = builder.totalAmount;
        this.couponCode      = builder.couponCode;
        this.trackingNumber  = null;
        this.placedAt        = LocalDateTime.now();
        this.updatedAt       = this.placedAt;
        this.statusHistory   = new ArrayList<>();
        addStatusHistory("Order placed with status " + status + ".");
    }

    // -------------------------------------------------------------------------
    // Status-machine methods
    // -------------------------------------------------------------------------

    /**
     * Returns true if the order can legally move to {@code next} from its current status.
     */
    public boolean canTransition(OrderStatus next) {
        return status.canTransitionTo(next);
    }

    /**
     * Transitions the order to {@code next}, recording the change in statusHistory.
     *
     * @throws InvalidOrderStateException if the transition is not permitted by the state machine.
     */
    public void transitionTo(OrderStatus next) {
        if (!status.canTransitionTo(next)) {
            throw new InvalidOrderStateException(orderId, status, next);
        }
        OrderStatus previous = this.status;
        this.status    = next;
        this.updatedAt = LocalDateTime.now();
        addStatusHistory("Status changed from " + previous + " to " + next
                       + " at " + updatedAt + ".");
    }

    /**
     * Appends a free-text entry to the status history log, prefixed with the current timestamp.
     */
    public void addStatusHistory(String note) {
        Objects.requireNonNull(note, "History note must not be null.");
        statusHistory.add("[" + LocalDateTime.now() + "] " + note);
    }

    // -------------------------------------------------------------------------
    // Totals calculation
    // -------------------------------------------------------------------------

    /**
     * (Re-)calculates subtotal, discountAmount, taxAmount, and totalAmount from the order items.
     *
     * subtotal      = sum of all OrderItem#getTotalPrice()
     * totalAmount   = subtotal - discountAmount + taxAmount
     *
     * discountAmount and taxAmount must be set (via builder or directly) before calling this;
     * this method only recomputes subtotal and the final totalAmount.
     */
    public void calculateTotals() {
        Money rawSubtotal = items.stream()
                                 .map(OrderItem::getTotalPrice)
                                 .reduce(Money.ZERO_INR, Money::add);
        this.subtotal    = rawSubtotal;
        // totalAmount = subtotal - discount + tax
        Money afterDiscount = discountAmount.isZero()
                ? subtotal
                : subtotal.subtract(discountAmount);
        this.totalAmount = afterDiscount.add(taxAmount);
        this.updatedAt   = LocalDateTime.now();
    }

    // -------------------------------------------------------------------------
    // Getters
    // -------------------------------------------------------------------------

    public String getOrderId()              { return orderId;        }
    public String getCustomerId()           { return customerId;     }
    public List<OrderItem> getItems()       { return items;          }   // already unmodifiable
    public Address getShippingAddress()     { return shippingAddress;}
    public Address getBillingAddress()      { return billingAddress; }
    public OrderStatus getStatus()          { return status;         }
    public PaymentStatus getPaymentStatus() { return paymentStatus;  }
    public PaymentMethod getPaymentMethod() { return paymentMethod;  }
    public Money getSubtotal()              { return subtotal;       }
    public Money getDiscountAmount()        { return discountAmount; }
    public Money getTaxAmount()             { return taxAmount;      }
    public Money getTotalAmount()           { return totalAmount;    }
    public String getCouponCode()           { return couponCode;     }
    public String getTrackingNumber()       { return trackingNumber; }
    public LocalDateTime getPlacedAt()      { return placedAt;       }
    public LocalDateTime getUpdatedAt()     { return updatedAt;      }

    public List<String> getStatusHistory() {
        return Collections.unmodifiableList(statusHistory);
    }

    // -------------------------------------------------------------------------
    // Setters for mutable fields
    // -------------------------------------------------------------------------

    /** Sets the tracking number when the order transitions to SHIPPED. */
    public void setTrackingNumber(String trackingNumber) {
        Objects.requireNonNull(trackingNumber, "Tracking number must not be null.");
        this.trackingNumber = trackingNumber;
        this.updatedAt = LocalDateTime.now();
    }

    /** Updates the payment status (called by PaymentService after gateway response). */
    public void setPaymentStatus(PaymentStatus paymentStatus) {
        Objects.requireNonNull(paymentStatus, "PaymentStatus must not be null.");
        this.paymentStatus = paymentStatus;
        this.updatedAt     = LocalDateTime.now();
    }

    // -------------------------------------------------------------------------
    // equals / hashCode / toString
    // -------------------------------------------------------------------------

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        return Objects.equals(orderId, ((Order) o).orderId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(orderId);
    }

    @Override
    public String toString() {
        return "Order{id='" + orderId + "', customer='" + customerId
             + "', status=" + status + ", payment=" + paymentStatus
             + ", total=" + totalAmount + ", placed=" + placedAt + "}";
    }

    // =========================================================================
    // Static Builder
    // =========================================================================

    /**
     * Builder for {@link Order}.
     *
     * Mandatory fields: orderId, customerId, at least one item, shippingAddress,
     * billingAddress, paymentMethod.
     *
     * Optional fields: couponCode, discountAmount, taxAmount (default to zero INR).
     */
    public static class Builder {

        // Required
        private String orderId;
        private String customerId;
        private List<OrderItem> items = new ArrayList<>();
        private Address shippingAddress;
        private Address billingAddress;
        private PaymentMethod paymentMethod;

        // Optional / defaulted
        private OrderStatus status         = OrderStatus.PENDING;
        private PaymentStatus paymentStatus = PaymentStatus.PENDING;
        private Money subtotal             = Money.ZERO_INR;
        private Money discountAmount       = Money.ZERO_INR;
        private Money taxAmount            = Money.ZERO_INR;
        private Money totalAmount          = Money.ZERO_INR;
        private String couponCode          = null;

        public Builder orderId(String orderId) {
            this.orderId = orderId;
            return this;
        }

        public Builder customerId(String customerId) {
            this.customerId = customerId;
            return this;
        }

        public Builder items(List<OrderItem> items) {
            Objects.requireNonNull(items, "Items list must not be null.");
            this.items = new ArrayList<>(items);
            return this;
        }

        public Builder addItem(OrderItem item) {
            Objects.requireNonNull(item, "OrderItem must not be null.");
            this.items.add(item);
            return this;
        }

        public Builder shippingAddress(Address shippingAddress) {
            this.shippingAddress = shippingAddress;
            return this;
        }

        public Builder billingAddress(Address billingAddress) {
            this.billingAddress = billingAddress;
            return this;
        }

        public Builder paymentMethod(PaymentMethod paymentMethod) {
            this.paymentMethod = paymentMethod;
            return this;
        }

        public Builder status(OrderStatus status) {
            this.status = status;
            return this;
        }

        public Builder paymentStatus(PaymentStatus paymentStatus) {
            this.paymentStatus = paymentStatus;
            return this;
        }

        public Builder discountAmount(Money discountAmount) {
            this.discountAmount = discountAmount;
            return this;
        }

        public Builder taxAmount(Money taxAmount) {
            this.taxAmount = taxAmount;
            return this;
        }

        public Builder couponCode(String couponCode) {
            this.couponCode = couponCode;
            return this;
        }

        /**
         * Validates required fields and constructs the {@link Order}.
         * totalAmount is computed from items - discount + tax during construction.
         *
         * @throws IllegalStateException if any required field is missing or items is empty.
         */
        public Order build() {
            if (orderId == null || orderId.isBlank())
                throw new IllegalStateException("orderId is required.");
            if (customerId == null || customerId.isBlank())
                throw new IllegalStateException("customerId is required.");
            if (items == null || items.isEmpty())
                throw new IllegalStateException("An order must contain at least one item.");
            if (shippingAddress == null)
                throw new IllegalStateException("shippingAddress is required.");
            if (billingAddress == null)
                throw new IllegalStateException("billingAddress is required.");
            if (paymentMethod == null)
                throw new IllegalStateException("paymentMethod is required.");

            // Compute subtotal and totalAmount from items
            this.subtotal = items.stream()
                                 .map(OrderItem::getTotalPrice)
                                 .reduce(Money.ZERO_INR, Money::add);
            Money afterDiscount = discountAmount.isZero()
                    ? subtotal
                    : subtotal.subtract(discountAmount);
            this.totalAmount = afterDiscount.add(taxAmount);

            return new Order(this);
        }
    }
}
```

```java
// PricingContext.java
package com.ecommerce.pricing;

import com.ecommerce.model.Money;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

/**
 * Value object that travels through the pricing decorator chain,
 * accumulating applied discounts and ultimately holding the final price.
 *
 * The context is built once with the base price, passed through every
 * PricingStrategy in the chain, and read at the end to get the final price
 * and a full breakdown of what was applied.
 *
 * PricingContext is intentionally mutable within a single pricing pipeline
 * run: each decorator calls addDiscount() and updates finalPrice.
 * It should NOT be reused across separate pricing requests.
 */
public class PricingContext {

    private final Money basePrice;
    private final List<DiscountApplication> appliedDiscounts;
    private Money finalPrice;

    public PricingContext(Money basePrice) {
        Objects.requireNonNull(basePrice, "Base price must not be null.");
        this.basePrice        = basePrice;
        this.finalPrice       = basePrice;
        this.appliedDiscounts = new ArrayList<>();
    }

    /**
     * Records an applied discount in the context and reduces finalPrice accordingly.
     *
     * @param application the discount that was just applied by a decorator
     */
    public void addDiscount(DiscountApplication application) {
        Objects.requireNonNull(application, "DiscountApplication must not be null.");
        appliedDiscounts.add(application);
        // Keep finalPrice in sync: subtract the discount amount from the current final price
        if (application.getAmount().isGreaterThan(Money.ZERO_INR)) {
            if (finalPrice.isGreaterThanOrEqual(application.getAmount())) {
                finalPrice = finalPrice.subtract(application.getAmount());
            } else {
                finalPrice = new Money(0L, finalPrice.getCurrency());
            }
        }
    }

    // --- Getters ---

    public Money getBasePrice()  { return basePrice;  }
    public Money getFinalPrice() { return finalPrice; }

    /** Updates finalPrice directly (used by tax decorators that add to price). */
    public void setFinalPrice(Money finalPrice) {
        Objects.requireNonNull(finalPrice, "Final price must not be null.");
        this.finalPrice = finalPrice;
    }

    public List<DiscountApplication> getAppliedDiscounts() {
        return Collections.unmodifiableList(appliedDiscounts);
    }

    /**
     * Returns the total discount amount applied so far
     * (sum of all DiscountApplication amounts).
     */
    public Money getTotalDiscountAmount() {
        return appliedDiscounts.stream()
                               .map(DiscountApplication::getAmount)
                               .reduce(new Money(0L, basePrice.getCurrency()), Money::add);
    }

    @Override
    public String toString() {
        return "PricingContext{base=" + basePrice + ", final=" + finalPrice
             + ", discounts=" + appliedDiscounts.size() + "}";
    }

    // =========================================================================
    // Inner class: DiscountApplication
    // =========================================================================

    /**
     * Immutable record of a single discount that was applied to the price.
     * Held inside PricingContext#appliedDiscounts for audit / receipt display.
     */
    public static final class DiscountApplication {

        /** Type of discount (matches the decorator that produced it). */
        public enum DiscountApplicationType {
            SELLER_DISCOUNT,
            COUPON,
            BULK,
            TAX           // technically not a discount but tracked here for receipt line items
        }

        private final String name;                    // e.g. "10% Seller Discount"
        private final Money amount;                   // absolute amount subtracted (or added for tax)
        private final DiscountApplicationType type;

        public DiscountApplication(String name, Money amount, DiscountApplicationType type) {
            Objects.requireNonNull(name,   "Discount name must not be null.");
            Objects.requireNonNull(amount, "Discount amount must not be null.");
            Objects.requireNonNull(type,   "Discount type must not be null.");
            this.name   = name;
            this.amount = amount;
            this.type   = type;
        }

        public String getName()                    { return name;   }
        public Money getAmount()                   { return amount; }
        public DiscountApplicationType getType()   { return type;   }

        @Override
        public String toString() {
            return "DiscountApplication{name='" + name + "', amount=" + amount + ", type=" + type + "}";
        }
    }
}
```

```java
// PricingStrategy.java
package com.ecommerce.pricing;

import com.ecommerce.model.Money;

/**
 * Strategy interface for price calculation.
 *
 * Implementations may be plain strategies (e.g. a flat-rate pricer) or
 * decorators that wrap another PricingStrategy and apply incremental
 * modifications.
 *
 * The contract:
 *   - calculate() must return the final price after any modifications.
 *   - calculate() must record every discount it applies into the context
 *     via PricingContext#addDiscount(), so callers can reconstruct a receipt.
 *   - calculate() must never return a negative Money value.
 */
public interface PricingStrategy {

    /**
     * Calculates the final price given a base price and a mutable pricing context.
     *
     * @param basePrice the starting price for this strategy's calculation
     * @param context   shared context accumulating discount history; mutated in-place
     * @return the price after this strategy's modifications (>= zero)
     */
    Money calculate(Money basePrice, PricingContext context);
}
```

```java
// BasePricingDecorator.java
package com.ecommerce.pricing;

import com.ecommerce.model.Money;

import java.util.Objects;

/**
 * Abstract base for all pricing decorators.
 *
 * Implements the DECORATOR PATTERN:
 *   - Wraps a delegate PricingStrategy (the "wrapped" component).
 *   - Subclasses call delegate.calculate() first to get the price so far,
 *     then apply their own modification on top.
 *
 * This ordering ensures discounts stack correctly:
 *   Tax(CouponDiscount(SellerDiscount(basePrice)))
 */
public abstract class BasePricingDecorator implements PricingStrategy {

    protected final PricingStrategy delegate;

    protected BasePricingDecorator(PricingStrategy delegate) {
        Objects.requireNonNull(delegate, "Delegate PricingStrategy must not be null.");
        this.delegate = delegate;
    }

    /**
     * Subclasses must call {@code delegate.calculate(basePrice, context)} to obtain
     * the price computed by the inner chain, then apply their own adjustment.
     */
    @Override
    public abstract Money calculate(Money basePrice, PricingContext context);
}
```

```java
// SellerDiscountDecorator.java
package com.ecommerce.pricing;

import com.ecommerce.model.Money;
import com.ecommerce.pricing.PricingContext.DiscountApplication;
import com.ecommerce.pricing.PricingContext.DiscountApplication.DiscountApplicationType;

import java.util.Objects;

/**
 * Applies a percentage discount when the seller has an active promotional rate.
 *
 * If sellerDiscountPercent == 0 (seller has no active discount), this decorator
 * is a transparent pass-through — it calls the delegate and returns its result.
 *
 * Example: a seller running a "15% off everything" sale.
 */
public class SellerDiscountDecorator extends BasePricingDecorator {

    private final String sellerId;
    private final int sellerDiscountPercent;  // 0 means no active discount

    /**
     * @param delegate              inner pricing strategy to decorate
     * @param sellerId              identifier of the seller (for audit label)
     * @param sellerDiscountPercent percentage discount (0-100); 0 = no discount
     */
    public SellerDiscountDecorator(PricingStrategy delegate,
                                   String sellerId,
                                   int sellerDiscountPercent) {
        super(delegate);
        Objects.requireNonNull(sellerId, "sellerId must not be null.");
        if (sellerDiscountPercent < 0 || sellerDiscountPercent > 100)
            throw new IllegalArgumentException("sellerDiscountPercent must be 0-100.");
        this.sellerId               = sellerId;
        this.sellerDiscountPercent  = sellerDiscountPercent;
    }

    @Override
    public Money calculate(Money basePrice, PricingContext context) {
        // Let the inner chain compute its price first
        Money priceAfterInner = delegate.calculate(basePrice, context);

        if (sellerDiscountPercent == 0) {
            // No active seller discount — transparent pass-through
            return priceAfterInner;
        }

        // Compute discount amount on the post-inner price
        long discountAmt = (priceAfterInner.getAmount() * sellerDiscountPercent) / 100L;
        Money discountMoney = new Money(discountAmt, priceAfterInner.getCurrency());
        Money priceAfterDiscount = priceAfterInner.subtract(discountMoney);

        context.addDiscount(new DiscountApplication(
            "Seller '" + sellerId + "' " + sellerDiscountPercent + "% discount",
            discountMoney,
            DiscountApplicationType.SELLER_DISCOUNT
        ));

        context.setFinalPrice(priceAfterDiscount);
        return priceAfterDiscount;
    }
}
```

```java
// CouponDiscountDecorator.java
package com.ecommerce.pricing;

import com.ecommerce.model.Coupon;
import com.ecommerce.model.Money;
import com.ecommerce.model.enums.DiscountType;
import com.ecommerce.pricing.PricingContext.DiscountApplication;
import com.ecommerce.pricing.PricingContext.DiscountApplication.DiscountApplicationType;

import java.util.Objects;

/**
 * Applies a coupon's discount to the price returned by the inner strategy.
 *
 * Coupon discount is applied to the price AFTER any seller discounts have
 * already reduced it (because the coupon is typically a customer-facing
 * promotional code applied on top of whatever price the seller offers).
 *
 * If coupon is null (no coupon applied), this decorator is a pass-through.
 */
public class CouponDiscountDecorator extends BasePricingDecorator {

    private final Coupon coupon;  // nullable — null means no coupon

    /**
     * @param delegate inner pricing strategy to decorate
     * @param coupon   the coupon to apply, or null if none is active
     */
    public CouponDiscountDecorator(PricingStrategy delegate, Coupon coupon) {
        super(delegate);
        this.coupon = coupon;
    }

    @Override
    public Money calculate(Money basePrice, PricingContext context) {
        Money priceAfterInner = delegate.calculate(basePrice, context);

        if (coupon == null || !coupon.isActive()) {
            // No coupon or coupon is deactivated — transparent pass-through
            return priceAfterInner;
        }

        Money discountMoney;
        if (coupon.getDiscountType() == DiscountType.PERCENTAGE) {
            int pct = (int) coupon.getDiscountValue().getAmount();
            long discountAmt = (priceAfterInner.getAmount() * pct) / 100L;
            discountMoney = new Money(discountAmt, priceAfterInner.getCurrency());
        } else {
            // FLAT_AMOUNT — cap at priceAfterInner so we don't go negative
            discountMoney = coupon.getDiscountValue().isGreaterThanOrEqual(priceAfterInner)
                    ? priceAfterInner
                    : coupon.getDiscountValue();
        }

        Money priceAfterCoupon = priceAfterInner.subtract(discountMoney);

        context.addDiscount(new DiscountApplication(
            "Coupon '" + coupon.getCode() + "'",
            discountMoney,
            DiscountApplicationType.COUPON
        ));

        context.setFinalPrice(priceAfterCoupon);
        return priceAfterCoupon;
    }
}
```

```java
// TaxDecorator.java
package com.ecommerce.pricing;

import com.ecommerce.model.Money;
import com.ecommerce.pricing.PricingContext.DiscountApplication;
import com.ecommerce.pricing.PricingContext.DiscountApplication.DiscountApplicationType;

import java.util.Objects;

/**
 * Applies a tax rate on top of the price returned by the inner strategy.
 *
 * Unlike the discount decorators, TaxDecorator ADDS to the price rather than
 * subtracting from it. It still records a DiscountApplication of type TAX
 * so the receipt can display the tax line item separately.
 *
 * Tax is always applied LAST in the chain (outermost decorator), so the rate
 * is computed on the fully-discounted price.
 *
 * Example: India GST 18% on top of discounted price.
 */
public class TaxDecorator extends BasePricingDecorator {

    private final String taxLabel;       // e.g. "GST 18%", "VAT 20%"
    private final int taxRatePercent;    // integer percentage, e.g. 18 for 18%

    /**
     * @param delegate       inner pricing strategy to decorate
     * @param taxLabel       human-readable tax label for the receipt
     * @param taxRatePercent tax rate as an integer percentage (0-100)
     */
    public TaxDecorator(PricingStrategy delegate, String taxLabel, int taxRatePercent) {
        super(delegate);
        Objects.requireNonNull(taxLabel, "Tax label must not be null.");
        if (taxRatePercent < 0 || taxRatePercent > 100)
            throw new IllegalArgumentException("taxRatePercent must be 0-100.");
        this.taxLabel       = taxLabel;
        this.taxRatePercent = taxRatePercent;
    }

    @Override
    public Money calculate(Money basePrice, PricingContext context) {
        Money priceAfterInner = delegate.calculate(basePrice, context);

        if (taxRatePercent == 0) {
            return priceAfterInner;
        }

        long taxAmount = (priceAfterInner.getAmount() * taxRatePercent) / 100L;
        Money taxMoney = new Money(taxAmount, priceAfterInner.getCurrency());
        Money priceWithTax = priceAfterInner.add(taxMoney);

        // Record tax as a DiscountApplication of type TAX (positive = added to price)
        context.addDiscount(new DiscountApplication(
            taxLabel,
            taxMoney,
            DiscountApplicationType.TAX
        ));

        context.setFinalPrice(priceWithTax);
        return priceWithTax;
    }
}
```

```java
// BulkDiscountDecorator.java
package com.ecommerce.pricing;

import com.ecommerce.model.Money;
import com.ecommerce.pricing.PricingContext.DiscountApplication;
import com.ecommerce.pricing.PricingContext.DiscountApplication.DiscountApplicationType;

import java.util.Map;
import java.util.Objects;
import java.util.TreeMap;

/**
 * Applies a tiered bulk discount based on the quantity being purchased.
 *
 * The discount tiers are defined as a sorted map of:
 *   minimumQuantity -> discountPercent
 *
 * Example tiers:
 *   {  5 -> 5,   // buy 5+  get  5% off
 *     10 -> 10,  // buy 10+ get 10% off
 *     20 -> 15 } // buy 20+ get 15% off
 *
 * The highest applicable tier (largest minimumQuantity <= ordered qty) is selected.
 * If quantity falls below all tiers, no discount is applied.
 */
public class BulkDiscountDecorator extends BasePricingDecorator {

    /**
     * Sorted ascending by key (minimumQuantity) so we can find the highest
     * applicable tier by iterating and tracking the last match.
     */
    private final TreeMap<Integer, Integer> tiers;  // minQty -> discountPercent
    private final int orderedQuantity;

    /**
     * @param delegate        inner pricing strategy to decorate
     * @param tiers           map of minimumQuantity -> discountPercent (must not be null/empty)
     * @param orderedQuantity the number of units being purchased
     */
    public BulkDiscountDecorator(PricingStrategy delegate,
                                 Map<Integer, Integer> tiers,
                                 int orderedQuantity) {
        super(delegate);
        Objects.requireNonNull(tiers, "Bulk discount tiers must not be null.");
        if (tiers.isEmpty()) throw new IllegalArgumentException("Bulk discount tiers must not be empty.");
        if (orderedQuantity < 1) throw new IllegalArgumentException("orderedQuantity must be >= 1.");

        this.tiers           = new TreeMap<>(tiers);
        this.orderedQuantity = orderedQuantity;
    }

    @Override
    public Money calculate(Money basePrice, PricingContext context) {
        Money priceAfterInner = delegate.calculate(basePrice, context);

        int applicableDiscountPercent = resolveApplicableTier();
        if (applicableDiscountPercent == 0) {
            // No tier matched — transparent pass-through
            return priceAfterInner;
        }

        long discountAmt = (priceAfterInner.getAmount() * applicableDiscountPercent) / 100L;
        Money discountMoney = new Money(discountAmt, priceAfterInner.getCurrency());
        Money priceAfterBulk = priceAfterInner.subtract(discountMoney);

        context.addDiscount(new DiscountApplication(
            "Bulk discount " + applicableDiscountPercent
                + "% (qty " + orderedQuantity + ")",
            discountMoney,
            DiscountApplicationType.BULK
        ));

        context.setFinalPrice(priceAfterBulk);
        return priceAfterBulk;
    }

    /**
     * Finds the highest applicable tier for the ordered quantity.
     *
     * @return the discount percentage for the best matching tier, or 0 if none match.
     */
    private int resolveApplicableTier() {
        int bestPercent = 0;
        for (Map.Entry<Integer, Integer> entry : tiers.entrySet()) {
            if (orderedQuantity >= entry.getKey()) {
                bestPercent = entry.getValue();
            } else {
                // tiers are sorted ascending; once we exceed qty, no higher tier applies
                break;
            }
        }
        return bestPercent;
    }
}
```

```java
// DiscountStrategy.java
package com.ecommerce.pricing;

import com.ecommerce.model.Money;

import java.util.Map;

/**
 * Strategy interface for computing a discount amount to be subtracted from a price.
 *
 * Unlike PricingStrategy (which operates at the full pipeline level with a context),
 * DiscountStrategy focuses on a single, isolated discount calculation.
 * It is used by services that need to evaluate multiple discount policies
 * independently before selecting the best one to apply.
 *
 * Implementations must be stateless — all inputs required for the calculation
 * are passed through the context map.
 */
public interface DiscountStrategy {

    /**
     * Computes the discount amount to subtract from {@code price}.
     *
     * @param price   the price to discount (must not be null, must be >= 0)
     * @param context arbitrary key/value parameters needed by the strategy
     *                (e.g. "quantity" -> 5, "userId" -> "u123")
     * @return the absolute discount amount (>= 0, never > price)
     */
    Money apply(Money price, Map<String, Object> context);
}
```

```java
// PercentageDiscountStrategy.java
package com.ecommerce.pricing;

import com.ecommerce.model.Money;

import java.util.Map;
import java.util.Objects;

/**
 * Applies a fixed percentage off the given price.
 *
 * The percentage is set at construction time and does not depend on the context.
 * The context map is accepted for API uniformity but is not read.
 *
 * Example: 20% off a price of 1000 INR → discount = 200 INR.
 */
public class PercentageDiscountStrategy implements DiscountStrategy {

    private final int discountPercent;  // 1-100

    /**
     * @param discountPercent integer percentage to discount (1-100 inclusive)
     */
    public PercentageDiscountStrategy(int discountPercent) {
        if (discountPercent < 1 || discountPercent > 100)
            throw new IllegalArgumentException("discountPercent must be between 1 and 100.");
        this.discountPercent = discountPercent;
    }

    @Override
    public Money apply(Money price, Map<String, Object> context) {
        Objects.requireNonNull(price, "Price must not be null.");
        long discountAmt = (price.getAmount() * discountPercent) / 100L;
        return new Money(discountAmt, price.getCurrency());
    }

    public int getDiscountPercent() { return discountPercent; }

    @Override
    public String toString() {
        return "PercentageDiscountStrategy{" + discountPercent + "% off}";
    }
}
```

```java
// FixedAmountDiscountStrategy.java
package com.ecommerce.pricing;

import com.ecommerce.model.Money;

import java.util.Map;
import java.util.Objects;

/**
 * Subtracts a fixed monetary amount from the price.
 *
 * If the fixed amount exceeds the price, the discount is capped at the price
 * value so the result is never negative (discount returned == price in that case).
 *
 * Example: flat Rs 100 off a price of Rs 800 → discount = Rs 100 (price becomes Rs 700).
 *          flat Rs 100 off a price of Rs  60 → discount = Rs  60 (price becomes Rs   0).
 */
public class FixedAmountDiscountStrategy implements DiscountStrategy {

    private final Money fixedDiscount;

    /**
     * @param fixedDiscount the fixed amount to subtract (must be a non-negative Money)
     */
    public FixedAmountDiscountStrategy(Money fixedDiscount) {
        Objects.requireNonNull(fixedDiscount, "Fixed discount amount must not be null.");
        this.fixedDiscount = fixedDiscount;
    }

    @Override
    public Money apply(Money price, Map<String, Object> context) {
        Objects.requireNonNull(price, "Price must not be null.");
        // Cap discount at the price to avoid negative result
        if (fixedDiscount.isGreaterThanOrEqual(price)) {
            return price;  // full price discounted away
        }
        return fixedDiscount;
    }

    public Money getFixedDiscount() { return fixedDiscount; }

    @Override
    public String toString() {
        return "FixedAmountDiscountStrategy{discount=" + fixedDiscount + "}";
    }
}
```

```java
// BuyXGetYDiscountStrategy.java
package com.ecommerce.pricing;

import com.ecommerce.model.Money;

import java.util.Map;
import java.util.Objects;

/**
 * Implements a "Buy X, Get Y Free" discount strategy.
 *
 * Given that the customer buys {@code buyQuantity} units and gets {@code freeQuantity}
 * units free, this strategy computes the effective per-unit discount when applied
 * to a batch of {@code totalQuantity} units.
 *
 * Algorithm:
 *   1. Determine how many complete "buy-X-get-Y" cycles fit in totalQuantity.
 *      cycleSize    = buyQuantity + freeQuantity
 *      cycles       = totalQuantity / cycleSize
 *      freeUnits    = cycles * freeQuantity
 *      (remaining units outside complete cycles are charged at full price)
 *   2. Total discount = freeUnits * unitPrice
 *
 * The context map must contain:
 *   "quantity"  (Integer) — total number of units being purchased
 *   "unitPrice" (Money)   — price per single unit
 *
 * Example: Buy 2 Get 1 free, quantity=7, unitPrice=500 INR
 *   cycleSize = 3, cycles = 2, freeUnits = 2
 *   discount  = 2 * 500 = 1000 INR
 *   (5 units charged, 2 free)
 */
public class BuyXGetYDiscountStrategy implements DiscountStrategy {

    private final int buyQuantity;   // X: units customer must buy
    private final int freeQuantity;  // Y: units customer gets free per cycle

    /**
     * @param buyQuantity  number of units that must be purchased (>= 1)
     * @param freeQuantity number of units that are free per buy cycle (>= 1)
     */
    public BuyXGetYDiscountStrategy(int buyQuantity, int freeQuantity) {
        if (buyQuantity  < 1) throw new IllegalArgumentException("buyQuantity must be >= 1.");
        if (freeQuantity < 1) throw new IllegalArgumentException("freeQuantity must be >= 1.");
        this.buyQuantity  = buyQuantity;
        this.freeQuantity = freeQuantity;
    }

    /**
     * Computes the effective discount for the entire purchase batch.
     *
     * @param price   ignored (the per-unit price is read from context key "unitPrice")
     * @param context must contain "quantity" (Integer) and "unitPrice" (Money)
     * @return total discount amount for the batch
     * @throws IllegalArgumentException if required context keys are missing or of wrong type
     */
    @Override
    public Money apply(Money price, Map<String, Object> context) {
        Objects.requireNonNull(context, "Context must not be null.");

        Object qtyObj       = context.get("quantity");
        Object unitPriceObj = context.get("unitPrice");

        if (!(qtyObj instanceof Integer)) {
            throw new IllegalArgumentException(
                "BuyXGetYDiscountStrategy requires context key 'quantity' of type Integer.");
        }
        if (!(unitPriceObj instanceof Money)) {
            throw new IllegalArgumentException(
                "BuyXGetYDiscountStrategy requires context key 'unitPrice' of type Money.");
        }

        int totalQuantity = (Integer) qtyObj;
        Money unitPrice   = (Money) unitPriceObj;

        if (totalQuantity < 1) {
            return new Money(0L, unitPrice.getCurrency());
        }

        int cycleSize  = buyQuantity + freeQuantity;
        int cycles     = totalQuantity / cycleSize;
        int freeUnits  = cycles * freeQuantity;

        if (freeUnits == 0) {
            return new Money(0L, unitPrice.getCurrency());
        }

        // Total discount = number of free units * price per unit
        return unitPrice.multiply(freeUnits);
    }

    public int getBuyQuantity()  { return buyQuantity;  }
    public int getFreeQuantity() { return freeQuantity; }

    @Override
    public String toString() {
        return "BuyXGetYDiscountStrategy{buy=" + buyQuantity + ", getFree=" + freeQuantity + "}";
    }
}
```

```java
// CouponService.java
package com.ecommerce.service;

import com.ecommerce.exception.InvalidCouponException;
import com.ecommerce.model.Coupon;
import com.ecommerce.model.Money;
import com.ecommerce.model.enums.DiscountType;

import java.time.LocalDate;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;

/**
 * Service responsible for managing and validating coupon codes.
 *
 * In a production system the coupon store would be backed by a persistent
 * repository; here an in-memory Map is used so the class remains testable
 * without a database dependency.
 *
 * Responsibilities:
 *   - Register new coupons (admin operation).
 *   - Validate a coupon against an order total and user context.
 *   - Return the monetary discount amount to apply when validation passes.
 *   - Increment the coupon's usage count atomically (within the constraints
 *     of the in-memory model; a real system would use DB row-level locking).
 *
 * Thread safety note: registerCoupon and validateAndApply are NOT thread-safe
 * in this implementation. A production service would use transactions or
 * optimistic locking on usageCount.
 */
public class CouponService {

    /** Primary store keyed by upper-cased coupon code for O(1) lookup. */
    private final Map<String, Coupon> couponStore;

    /**
     * Constructs a CouponService with an empty coupon store.
     */
    public CouponService() {
        this.couponStore = new HashMap<>();
    }

    /**
     * Registers a coupon in the store, making it available for validation.
     * If a coupon with the same code already exists it is replaced.
     *
     * @param coupon the coupon to register (must not be null)
     */
    public void registerCoupon(Coupon coupon) {
        Objects.requireNonNull(coupon, "Coupon must not be null.");
        couponStore.put(coupon.getCode().toUpperCase(), coupon);
    }

    /**
     * Validates the coupon identified by {@code couponCode} against the given
     * order total and user context, and returns the discount amount to apply.
     *
     * Validation steps (in order):
     *   1. Coupon exists in the store.
     *   2. Coupon is active (not manually deactivated).
     *   3. Coupon has not expired (expiryDate >= today).
     *   4. Coupon has not exceeded its usage limit (0 = unlimited).
     *   5. Order total meets the minimum order value requirement.
     *   6. If user-specific restriction is present: the userId must match
     *      the "restrictedUserId" key in the extraContext map.
     *
     * On success, the coupon's usage count is incremented atomically and the
     * discount amount is returned.
     *
     * @param couponCode   the coupon code entered by the user (case-insensitive)
     * @param orderTotal   the order subtotal before discount (must not be null)
     * @param userId       the ID of the user applying the coupon (must not be null)
     * @return the Money amount to subtract from the order total (>= 0)
     * @throws InvalidCouponException if any validation step fails
     */
    public Money validateAndApply(String couponCode, Money orderTotal, String userId) {
        Objects.requireNonNull(couponCode,  "Coupon code must not be null.");
        Objects.requireNonNull(orderTotal,  "Order total must not be null.");
        Objects.requireNonNull(userId,      "User ID must not be null.");

        String normalizedCode = couponCode.toUpperCase();

        // Step 1: existence check
        Coupon coupon = Optional.ofNullable(couponStore.get(normalizedCode))
                .orElseThrow(() -> new InvalidCouponException(normalizedCode,
                    "coupon code does not exist."));

        // Step 2: active check
        if (!coupon.isActive()) {
            throw new InvalidCouponException(normalizedCode, "coupon is inactive.");
        }

        // Step 3: expiry check
        LocalDate today = LocalDate.now();
        if (today.isAfter(coupon.getExpiryDate())) {
            throw new InvalidCouponException(normalizedCode,
                "coupon expired on " + coupon.getExpiryDate() + ".");
        }

        // Step 4: usage limit check
        if (coupon.getMaxUsage() > 0 && coupon.getUsageCount() >= coupon.getMaxUsage()) {
            throw new InvalidCouponException(normalizedCode,
                "coupon has reached its maximum usage limit of " + coupon.getMaxUsage() + ".");
        }

        // Step 5: minimum order value check
        if (!orderTotal.isGreaterThanOrEqual(coupon.getMinOrderValue())) {
            throw new InvalidCouponException(normalizedCode,
                "minimum order value of " + coupon.getMinOrderValue()
                + " not met (order total: " + orderTotal + ").");
        }

        // Step 6: user-specific restriction
        // Coupons may carry a restrictedUserId stored in their applicable categories set
        // using the sentinel prefix "user:", e.g. "user:u123".
        // If such a restriction exists, the requesting userId must match.
        boolean hasUserRestriction = coupon.getApplicableCategories().stream()
                .anyMatch(s -> s.startsWith("user:"));
        if (hasUserRestriction) {
            String requiredUserId = coupon.getApplicableCategories().stream()
                    .filter(s -> s.startsWith("user:"))
                    .map(s -> s.substring("user:".length()))
                    .findFirst()
                    .orElse("");
            if (!requiredUserId.equals(userId)) {
                throw new InvalidCouponException(normalizedCode,
                    "coupon is restricted to a specific user and cannot be applied by user '"
                    + userId + "'.");
            }
        }

        // All checks passed — increment usage and compute discount
        coupon.incrementUsage();

        return computeDiscountAmount(coupon, orderTotal);
    }

    /**
     * Deactivates a coupon by code, preventing further use.
     * No-op if the coupon does not exist.
     *
     * @param couponCode the coupon code to deactivate
     */
    public void deactivateCoupon(String couponCode) {
        Objects.requireNonNull(couponCode, "Coupon code must not be null.");
        Coupon coupon = couponStore.get(couponCode.toUpperCase());
        if (coupon != null) {
            coupon.deactivate();
        }
    }

    /**
     * Returns the Coupon for the given code, or empty if not registered.
     *
     * @param couponCode the code to look up (case-insensitive)
     */
    public Optional<Coupon> findCoupon(String couponCode) {
        Objects.requireNonNull(couponCode, "Coupon code must not be null.");
        return Optional.ofNullable(couponStore.get(couponCode.toUpperCase()));
    }

    /**
     * Returns the current number of registered coupons in the store.
     */
    public int getCouponCount() {
        return couponStore.size();
    }

    // -------------------------------------------------------------------------
    // Private helpers
    // -------------------------------------------------------------------------

    /**
     * Computes the absolute discount amount to subtract from the order total
     * based on the coupon's discount type and value.
     *
     * @param coupon     the validated coupon
     * @param orderTotal the order subtotal before discount
     * @return the discount amount (always >= 0, capped at orderTotal)
     */
    private Money computeDiscountAmount(Coupon coupon, Money orderTotal) {
        if (coupon.getDiscountType() == DiscountType.PERCENTAGE) {
            int pct = (int) coupon.getDiscountValue().getAmount();
            long discountAmt = (orderTotal.getAmount() * pct) / 100L;
            return new Money(discountAmt, orderTotal.getCurrency());
        } else {
            // FLAT_AMOUNT — cap at orderTotal
            Money flat = coupon.getDiscountValue();
            return flat.isGreaterThanOrEqual(orderTotal) ? orderTotal : flat;
        }
    }
}
```

---

## Payment System (Factory Pattern)

### PaymentRequest

```java
import java.time.Instant;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

/**
 * Immutable value object encapsulating everything needed to initiate a payment.
 */
public final class PaymentRequest {

    private final String orderId;
    private final Money amount;
    private final PaymentMethod paymentMethod;
    private final String userId;
    private final Map<String, String> metadata;
    private final Instant createdAt;

    public PaymentRequest(String orderId,
                          Money amount,
                          PaymentMethod paymentMethod,
                          String userId,
                          Map<String, String> metadata) {
        this.orderId = Objects.requireNonNull(orderId, "orderId must not be null");
        this.amount = Objects.requireNonNull(amount, "amount must not be null");
        this.paymentMethod = Objects.requireNonNull(paymentMethod, "paymentMethod must not be null");
        this.userId = Objects.requireNonNull(userId, "userId must not be null");
        this.metadata = metadata != null
                ? Collections.unmodifiableMap(new HashMap<>(metadata))
                : Collections.emptyMap();
        this.createdAt = Instant.now();
    }

    public String getOrderId() {
        return orderId;
    }

    public Money getAmount() {
        return amount;
    }

    public PaymentMethod getPaymentMethod() {
        return paymentMethod;
    }

    public String getUserId() {
        return userId;
    }

    public Map<String, String> getMetadata() {
        return metadata;
    }

    public Instant getCreatedAt() {
        return createdAt;
    }

    @Override
    public String toString() {
        return "PaymentRequest{orderId='" + orderId + "', amount=" + amount
                + ", paymentMethod=" + paymentMethod + ", userId='" + userId + "'}";
    }
}
```

---

### PaymentResult

```java
import java.time.Instant;
import java.util.Objects;

/**
 * Immutable value object representing the outcome of a payment attempt.
 */
public final class PaymentResult {

    private final boolean success;
    private final String transactionId;
    private final String failureReason;
    private final Instant processedAt;

    private PaymentResult(boolean success, String transactionId, String failureReason) {
        this.success = success;
        this.transactionId = transactionId;
        this.failureReason = failureReason;
        this.processedAt = Instant.now();
    }

    // -------------------------------------------------------------------------
    // Static factory methods
    // -------------------------------------------------------------------------

    /**
     * Creates a successful PaymentResult.
     *
     * @param transactionId the unique transaction identifier issued by the processor
     */
    public static PaymentResult success(String transactionId) {
        Objects.requireNonNull(transactionId, "transactionId must not be null for a successful result");
        return new PaymentResult(true, transactionId, null);
    }

    /**
     * Creates a failed PaymentResult.
     *
     * @param failureReason human-readable description of why the payment failed
     */
    public static PaymentResult failure(String failureReason) {
        Objects.requireNonNull(failureReason, "failureReason must not be null for a failed result");
        return new PaymentResult(false, null, failureReason);
    }

    // -------------------------------------------------------------------------
    // Accessors
    // -------------------------------------------------------------------------

    public boolean isSuccess() {
        return success;
    }

    public String getTransactionId() {
        return transactionId;
    }

    public String getFailureReason() {
        return failureReason;
    }

    public Instant getProcessedAt() {
        return processedAt;
    }

    @Override
    public String toString() {
        if (success) {
            return "PaymentResult{success=true, transactionId='" + transactionId
                    + "', processedAt=" + processedAt + "}";
        }
        return "PaymentResult{success=false, failureReason='" + failureReason
                + "', processedAt=" + processedAt + "}";
    }
}
```

---

### PaymentProcessor Interface

```java
/**
 * Strategy interface for processing payments.
 * Each implementation handles one payment method (credit card, UPI, wallet, COD, etc.).
 */
public interface PaymentProcessor {

    /**
     * Processes the given payment request and returns a result indicating
     * success or failure.
     *
     * @param request the payment request (never null)
     * @return a non-null PaymentResult
     */
    PaymentResult process(PaymentRequest request);
}
```

---

### CreditCardProcessor

```java
import java.util.UUID;
import java.util.logging.Logger;

/**
 * Simulates credit card payment processing.
 * Validates that the amount is positive, then issues a transaction ID.
 */
public class CreditCardProcessor implements PaymentProcessor {

    private static final Logger logger = Logger.getLogger(CreditCardProcessor.class.getName());

    @Override
    public PaymentResult process(PaymentRequest request) {
        logger.info("CreditCardProcessor: initiating payment for order=" + request.getOrderId()
                + ", amount=" + request.getAmount());

        if (request.getAmount().getAmount() <= 0) {
            String reason = "Invalid amount for credit card payment: " + request.getAmount();
            logger.warning("CreditCardProcessor: " + reason);
            return PaymentResult.failure(reason);
        }

        // Simulate network call / card-network validation
        String transactionId = "CC-" + UUID.randomUUID().toString().replace("-", "").toUpperCase();
        logger.info("CreditCardProcessor: transaction approved, transactionId=" + transactionId);
        return PaymentResult.success(transactionId);
    }
}
```

---

### UpiProcessor

```java
import java.util.UUID;
import java.util.logging.Logger;

/**
 * Simulates UPI (Unified Payments Interface) payment processing.
 */
public class UpiProcessor implements PaymentProcessor {

    private static final Logger logger = Logger.getLogger(UpiProcessor.class.getName());

    @Override
    public PaymentResult process(PaymentRequest request) {
        logger.info("UpiProcessor: initiating UPI payment for order=" + request.getOrderId()
                + ", amount=" + request.getAmount() + ", user=" + request.getUserId());

        if (request.getAmount().getAmount() <= 0) {
            String reason = "Invalid amount for UPI payment: " + request.getAmount();
            logger.warning("UpiProcessor: " + reason);
            return PaymentResult.failure(reason);
        }

        // Retrieve optional VPA from metadata
        String vpa = request.getMetadata().getOrDefault("upiVpa", "unknown@upi");
        logger.info("UpiProcessor: sending collect request to VPA=" + vpa);

        String transactionId = "UPI-" + UUID.randomUUID().toString().replace("-", "").toUpperCase();
        logger.info("UpiProcessor: payment successful, transactionId=" + transactionId);
        return PaymentResult.success(transactionId);
    }
}
```

---

### WalletProcessor

```java
import java.util.UUID;
import java.util.logging.Logger;

/**
 * Simulates wallet-based payment processing.
 * Checks whether the user has sufficient wallet balance before approving.
 */
public class WalletProcessor implements PaymentProcessor {

    private static final Logger logger = Logger.getLogger(WalletProcessor.class.getName());

    /**
     * Simulated wallet balance lookup. In production this would call a wallet-balance service.
     *
     * @param userId the user whose wallet is queried
     * @return the available balance in the smallest currency unit (e.g. paise for INR)
     */
    private long fetchWalletBalance(String userId) {
        // Stubbed: every user has a fixed simulated balance of 50000 paise (INR 500)
        return 50_000L;
    }

    @Override
    public PaymentResult process(PaymentRequest request) {
        logger.info("WalletProcessor: initiating wallet payment for order=" + request.getOrderId()
                + ", amount=" + request.getAmount() + ", user=" + request.getUserId());

        if (request.getAmount().getAmount() <= 0) {
            String reason = "Invalid amount for wallet payment: " + request.getAmount();
            logger.warning("WalletProcessor: " + reason);
            return PaymentResult.failure(reason);
        }

        long balance = fetchWalletBalance(request.getUserId());
        long required = request.getAmount().getAmount();

        if (balance < required) {
            String reason = "Insufficient wallet balance. Required=" + required
                    + ", available=" + balance;
            logger.warning("WalletProcessor: " + reason + " for user=" + request.getUserId());
            return PaymentResult.failure(reason);
        }

        // Debit is simulated; in production this would call the wallet service
        String transactionId = "WLT-" + UUID.randomUUID().toString().replace("-", "").toUpperCase();
        logger.info("WalletProcessor: wallet debited successfully, transactionId=" + transactionId);
        return PaymentResult.success(transactionId);
    }
}
```

---

### CodProcessor

```java
import java.util.UUID;
import java.util.logging.Logger;

/**
 * Cash-on-Delivery payment processor.
 * Payment is collected at delivery time, so placement always succeeds.
 */
public class CodProcessor implements PaymentProcessor {

    private static final Logger logger = Logger.getLogger(CodProcessor.class.getName());

    @Override
    public PaymentResult process(PaymentRequest request) {
        logger.info("CodProcessor: registering COD order=" + request.getOrderId()
                + ", amount=" + request.getAmount() + ". Payment will be collected on delivery.");

        // COD orders always succeed at placement; payment is deferred to delivery
        String transactionId = "COD-" + UUID.randomUUID().toString().replace("-", "").toUpperCase();
        logger.info("CodProcessor: COD registered, transactionId=" + transactionId);
        return PaymentResult.success(transactionId);
    }
}
```

---

### PaymentProcessorFactory

```java
import java.util.Objects;

/**
 * Static factory that maps a {@link PaymentMethod} to the appropriate
 * {@link PaymentProcessor} implementation.
 *
 * Adding a new payment method requires:
 *   1. Adding the enum constant to {@link PaymentMethod}.
 *   2. Adding a case here.
 *   3. Implementing the new processor class.
 */
public final class PaymentProcessorFactory {

    // Utility class — prevent instantiation
    private PaymentProcessorFactory() {
        throw new UnsupportedOperationException("PaymentProcessorFactory is a utility class");
    }

    /**
     * Returns a {@link PaymentProcessor} suitable for the given payment method.
     *
     * @param paymentMethod the method chosen by the buyer (never null)
     * @return a non-null PaymentProcessor instance
     * @throws IllegalArgumentException if the payment method is not supported
     */
    public static PaymentProcessor createProcessor(PaymentMethod paymentMethod) {
        Objects.requireNonNull(paymentMethod, "paymentMethod must not be null");
        switch (paymentMethod) {
            case CREDIT_CARD:
            case DEBIT_CARD:
                return new CreditCardProcessor();
            case UPI:
                return new UpiProcessor();
            case WALLET:
                return new WalletProcessor();
            case CASH_ON_DELIVERY:
                return new CodProcessor();
            default:
                throw new IllegalArgumentException(
                        "Unsupported payment method: " + paymentMethod);
        }
    }
}
```

---

## Notification System (Observer Pattern)

### NotificationEvent

```java
import java.time.Instant;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

/**
 * Immutable value object representing a notification event published to observers.
 */
public final class NotificationEvent {

    private final NotificationType eventType;
    private final String orderId;
    private final String userId;
    private final String message;
    private final Map<String, String> metadata;
    private final Instant timestamp;

    public NotificationEvent(NotificationType eventType,
                             String orderId,
                             String userId,
                             String message,
                             Map<String, String> metadata) {
        this.eventType = Objects.requireNonNull(eventType, "eventType must not be null");
        this.orderId = Objects.requireNonNull(orderId, "orderId must not be null");
        this.userId = Objects.requireNonNull(userId, "userId must not be null");
        this.message = Objects.requireNonNull(message, "message must not be null");
        this.metadata = metadata != null
                ? Collections.unmodifiableMap(new HashMap<>(metadata))
                : Collections.emptyMap();
        this.timestamp = Instant.now();
    }

    public NotificationType getEventType() {
        return eventType;
    }

    public String getOrderId() {
        return orderId;
    }

    public String getUserId() {
        return userId;
    }

    public String getMessage() {
        return message;
    }

    public Map<String, String> getMetadata() {
        return metadata;
    }

    public Instant getTimestamp() {
        return timestamp;
    }

    @Override
    public String toString() {
        return "NotificationEvent{eventType=" + eventType + ", orderId='" + orderId
                + "', userId='" + userId + "', message='" + message
                + "', timestamp=" + timestamp + "}";
    }
}
```

---

### NotificationObserver Interface

```java
/**
 * Observer interface for the notification sub-system.
 * Implementors receive notification events and dispatch them via their respective channel.
 */
public interface NotificationObserver {

    /**
     * Called by {@link NotificationService} whenever a notification event is published.
     *
     * @param event the event to handle (never null)
     */
    void onEvent(NotificationEvent event);
}
```

---

### EmailNotificationObserver

```java
import java.util.logging.Logger;

/**
 * Sends (simulated) email notifications to the user.
 */
public class EmailNotificationObserver implements NotificationObserver {

    private static final Logger logger = Logger.getLogger(EmailNotificationObserver.class.getName());

    @Override
    public void onEvent(NotificationEvent event) {
        logger.info("[EMAIL] To: user=" + event.getUserId()
                + " | Subject: Order " + event.getOrderId() + " - " + event.getEventType()
                + " | Body: " + event.getMessage()
                + " | Sent at: " + event.getTimestamp());
    }
}
```

---

### SmsNotificationObserver

```java
import java.util.logging.Logger;

/**
 * Sends (simulated) SMS notifications to the user's registered mobile number.
 */
public class SmsNotificationObserver implements NotificationObserver {

    private static final Logger logger = Logger.getLogger(SmsNotificationObserver.class.getName());

    @Override
    public void onEvent(NotificationEvent event) {
        logger.info("[SMS] To: user=" + event.getUserId()
                + " | Event: " + event.getEventType()
                + " | Message: " + event.getMessage()
                + " | Sent at: " + event.getTimestamp());
    }
}
```

---

### PushNotificationObserver

```java
import java.util.logging.Logger;

/**
 * Sends (simulated) push notifications to the user's mobile/web app.
 */
public class PushNotificationObserver implements NotificationObserver {

    private static final Logger logger = Logger.getLogger(PushNotificationObserver.class.getName());

    @Override
    public void onEvent(NotificationEvent event) {
        logger.info("[PUSH] To: user=" + event.getUserId()
                + " | Title: " + event.getEventType()
                + " | Body: " + event.getMessage()
                + " | Metadata: " + event.getMetadata()
                + " | Sent at: " + event.getTimestamp());
    }
}
```

---

### NotificationService

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.logging.Logger;

/**
 * Central notification hub that maintains a list of {@link NotificationObserver}s
 * and dispatches {@link NotificationEvent}s to all of them.
 *
 * <p>Usage:
 * <pre>
 *   NotificationService svc = new NotificationService();
 *   svc.subscribe(new EmailNotificationObserver());
 *   svc.subscribe(new SmsNotificationObserver());
 *   svc.notifyOrderStatusChange(order, previousStatus);
 * </pre>
 */
public class NotificationService {

    private static final Logger logger = Logger.getLogger(NotificationService.class.getName());

    private final List<NotificationObserver> observers = new ArrayList<>();

    // -------------------------------------------------------------------------
    // Observer management
    // -------------------------------------------------------------------------

    /**
     * Registers an observer to receive future notification events.
     *
     * @param observer the observer to add (must not be null)
     */
    public void subscribe(NotificationObserver observer) {
        Objects.requireNonNull(observer, "observer must not be null");
        if (!observers.contains(observer)) {
            observers.add(observer);
            logger.fine("NotificationService: subscribed " + observer.getClass().getSimpleName());
        }
    }

    /**
     * Removes a previously registered observer.
     *
     * @param observer the observer to remove (must not be null)
     */
    public void unsubscribe(NotificationObserver observer) {
        Objects.requireNonNull(observer, "observer must not be null");
        boolean removed = observers.remove(observer);
        if (removed) {
            logger.fine("NotificationService: unsubscribed " + observer.getClass().getSimpleName());
        }
    }

    /**
     * Dispatches the event to every registered observer.
     * Exceptions thrown by individual observers are caught and logged so that
     * one misbehaving observer does not prevent others from receiving the event.
     *
     * @param event the event to broadcast (must not be null)
     */
    public void notifyAll(NotificationEvent event) {
        Objects.requireNonNull(event, "event must not be null");
        logger.info("NotificationService: broadcasting event=" + event.getEventType()
                + " for orderId=" + event.getOrderId()
                + " to " + observers.size() + " observer(s)");

        for (NotificationObserver observer : observers) {
            try {
                observer.onEvent(event);
            } catch (Exception ex) {
                logger.warning("NotificationService: observer "
                        + observer.getClass().getSimpleName()
                        + " threw an exception for event=" + event.getEventType()
                        + ": " + ex.getMessage());
            }
        }
    }

    /**
     * Convenience method that builds and broadcasts a {@link NotificationEvent}
     * whenever an order transitions to a new {@link OrderStatus}.
     *
     * @param order          the order that changed status (must not be null)
     * @param previousStatus the status the order held before the transition
     */
    public void notifyOrderStatusChange(Order order, OrderStatus previousStatus) {
        Objects.requireNonNull(order, "order must not be null");
        Objects.requireNonNull(previousStatus, "previousStatus must not be null");

        String message = buildStatusChangeMessage(order, previousStatus);

        Map<String, String> metadata = new HashMap<>();
        metadata.put("previousStatus", previousStatus.name());
        metadata.put("currentStatus", order.getStatus().name());
        metadata.put("totalAmount", order.getTotalAmount().toString());

        NotificationEvent event = new NotificationEvent(
                NotificationType.ORDER_STATUS_UPDATE,
                order.getOrderId(),
                order.getUserId(),
                message,
                metadata
        );

        notifyAll(event);
    }

    /**
     * Returns an unmodifiable view of the currently registered observers.
     */
    public List<NotificationObserver> getObservers() {
        return Collections.unmodifiableList(observers);
    }

    // -------------------------------------------------------------------------
    // Private helpers
    // -------------------------------------------------------------------------

    private String buildStatusChangeMessage(Order order, OrderStatus previousStatus) {
        return "Your order #" + order.getOrderId()
                + " has been updated from " + previousStatus
                + " to " + order.getStatus() + "."
                + " Total amount: " + order.getTotalAmount() + ".";
    }
}
```

---

## Inventory Service

### InventoryService

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.logging.Logger;

/**
 * Manages inventory for all sellers and product variants.
 *
 * <p>Each inventory slot is keyed by a composite string "{sellerId}::{variantId}".
 * Inventory operations are not thread-safe in this in-memory reference implementation.
 * In production, these operations would be backed by a database with row-level locking
 * or optimistic concurrency control.
 *
 * <p>Terminology:
 * <ul>
 *   <li><b>available</b>  – units that can be reserved right now.</li>
 *   <li><b>reserved</b>   – units set aside for a pending order (not yet shipped).</li>
 *   <li><b>committed</b>  – a reservation that has been finalised (order confirmed / shipped).</li>
 * </ul>
 */
public class InventoryService {

    private static final Logger logger = Logger.getLogger(InventoryService.class.getName());

    /** Composite key separator. */
    private static final String KEY_SEP = "::";

    /** Master inventory store: compositeKey -> InventoryItem. */
    private final Map<String, InventoryItem> inventoryStore = new HashMap<>();

    // -------------------------------------------------------------------------
    // Key helpers
    // -------------------------------------------------------------------------

    private String buildKey(String sellerId, String variantId) {
        return sellerId + KEY_SEP + variantId;
    }

    // -------------------------------------------------------------------------
    // Public API
    // -------------------------------------------------------------------------

    /**
     * Adds (or creates) inventory for a seller-variant pair.
     *
     * @param sellerId  the seller owning the stock
     * @param variantId the product variant identifier
     * @param quantity  the number of units to add (must be >= 1)
     * @throws IllegalArgumentException if quantity < 1
     */
    public void addInventory(String sellerId, String variantId, int quantity) {
        Objects.requireNonNull(sellerId, "sellerId must not be null");
        Objects.requireNonNull(variantId, "variantId must not be null");
        if (quantity < 1) {
            throw new IllegalArgumentException("quantity must be >= 1, got: " + quantity);
        }

        String key = buildKey(sellerId, variantId);
        inventoryStore.compute(key, (k, existing) -> {
            if (existing == null) {
                logger.info("InventoryService: creating new slot key=" + k
                        + " with quantity=" + quantity);
                return new InventoryItem(sellerId, variantId, quantity);
            }
            existing.addStock(quantity);
            logger.info("InventoryService: added " + quantity + " units to key=" + k
                    + ", new available=" + existing.getAvailableQuantity());
            return existing;
        });
    }

    /**
     * Returns the {@link InventoryItem} for the given seller-variant pair, or empty if absent.
     *
     * @param sellerId  the seller identifier
     * @param variantId the product variant identifier
     */
    public Optional<InventoryItem> getInventoryItem(String sellerId, String variantId) {
        Objects.requireNonNull(sellerId, "sellerId must not be null");
        Objects.requireNonNull(variantId, "variantId must not be null");
        return Optional.ofNullable(inventoryStore.get(buildKey(sellerId, variantId)));
    }

    /**
     * Reserves {@code quantity} units for a seller-variant pair.
     * Reserved units are deducted from the available count immediately.
     *
     * @param sellerId  the seller identifier
     * @param variantId the product variant identifier
     * @param quantity  number of units to reserve (must be >= 1)
     * @throws InsufficientInventoryException if available stock is less than {@code quantity}
     */
    public void reserveInventory(String sellerId, String variantId, int quantity)
            throws InsufficientInventoryException {
        Objects.requireNonNull(sellerId, "sellerId must not be null");
        Objects.requireNonNull(variantId, "variantId must not be null");
        if (quantity < 1) {
            throw new IllegalArgumentException("quantity must be >= 1, got: " + quantity);
        }

        String key = buildKey(sellerId, variantId);
        InventoryItem item = inventoryStore.get(key);

        if (item == null) {
            throw new InsufficientInventoryException(
                    "No inventory record found for sellerId=" + sellerId
                            + ", variantId=" + variantId);
        }

        if (item.getAvailableQuantity() < quantity) {
            throw new InsufficientInventoryException(
                    "Insufficient inventory for variantId=" + variantId
                            + " from sellerId=" + sellerId
                            + ". Requested=" + quantity
                            + ", available=" + item.getAvailableQuantity());
        }

        item.reserve(quantity);
        logger.info("InventoryService: reserved " + quantity + " unit(s) for key=" + key
                + ", remaining available=" + item.getAvailableQuantity());
    }

    /**
     * Commits a previously made reservation, reducing the reserved count.
     * Call this when the order is shipped or confirmed beyond the point of cancellation.
     *
     * @param sellerId  the seller identifier
     * @param variantId the product variant identifier
     * @param quantity  number of units to commit
     * @throws IllegalStateException if there are not enough reserved units to commit
     */
    public void commitReservation(String sellerId, String variantId, int quantity) {
        Objects.requireNonNull(sellerId, "sellerId must not be null");
        Objects.requireNonNull(variantId, "variantId must not be null");
        if (quantity < 1) {
            throw new IllegalArgumentException("quantity must be >= 1, got: " + quantity);
        }

        String key = buildKey(sellerId, variantId);
        InventoryItem item = inventoryStore.get(key);

        if (item == null) {
            throw new IllegalStateException(
                    "Cannot commit: no inventory record for key=" + key);
        }

        item.commit(quantity);
        logger.info("InventoryService: committed " + quantity + " unit(s) for key=" + key
                + ", reserved remaining=" + item.getReservedQuantity());
    }

    /**
     * Releases a previously made reservation back to available stock.
     * Call this when an order is cancelled before shipment.
     *
     * @param sellerId  the seller identifier
     * @param variantId the product variant identifier
     * @param quantity  number of units to release
     */
    public void releaseReservation(String sellerId, String variantId, int quantity) {
        Objects.requireNonNull(sellerId, "sellerId must not be null");
        Objects.requireNonNull(variantId, "variantId must not be null");
        if (quantity < 1) {
            throw new IllegalArgumentException("quantity must be >= 1, got: " + quantity);
        }

        String key = buildKey(sellerId, variantId);
        InventoryItem item = inventoryStore.get(key);

        if (item == null) {
            logger.warning("InventoryService: releaseReservation called for unknown key=" + key
                    + ", ignoring.");
            return;
        }

        item.release(quantity);
        logger.info("InventoryService: released " + quantity + " unit(s) back to available for key="
                + key + ", available now=" + item.getAvailableQuantity());
    }

    /**
     * Returns the number of units currently available (not reserved) for the given
     * seller-variant pair.
     *
     * @param sellerId  the seller identifier
     * @param variantId the product variant identifier
     * @return available quantity, or 0 if no record exists
     */
    public int getAvailableQuantity(String sellerId, String variantId) {
        Objects.requireNonNull(sellerId, "sellerId must not be null");
        Objects.requireNonNull(variantId, "variantId must not be null");
        InventoryItem item = inventoryStore.get(buildKey(sellerId, variantId));
        return item == null ? 0 : item.getAvailableQuantity();
    }

    /**
     * Attempts to reserve inventory for every line item in the order atomically.
     * If any reservation fails, all previously made reservations in this call are
     * rolled back before the exception is re-thrown.
     *
     * @param order the order whose items need inventory reservation (must not be null)
     * @throws InsufficientInventoryException if any item cannot be reserved
     */
    public void reserveForOrder(Order order) throws InsufficientInventoryException {
        Objects.requireNonNull(order, "order must not be null");

        List<OrderItem> reserved = new ArrayList<>();

        for (OrderItem item : order.getItems()) {
            String sellerId = item.getSellerId();
            String variantId = item.getVariantId();
            int qty = item.getQuantity();

            try {
                reserveInventory(sellerId, variantId, qty);
                reserved.add(item);
            } catch (InsufficientInventoryException ex) {
                // Roll back all reservations made so far in this call
                logger.warning("InventoryService: reservation failed for variantId=" + variantId
                        + " from sellerId=" + sellerId + ". Rolling back " + reserved.size()
                        + " earlier reservation(s). Reason: " + ex.getMessage());

                for (OrderItem rolledBack : reserved) {
                    try {
                        releaseReservation(rolledBack.getSellerId(),
                                rolledBack.getVariantId(),
                                rolledBack.getQuantity());
                    } catch (Exception rollbackEx) {
                        logger.severe("InventoryService: rollback failed for variantId="
                                + rolledBack.getVariantId() + ": " + rollbackEx.getMessage());
                    }
                }

                throw new InsufficientInventoryException(
                        "Order " + order.getOrderId() + " could not be reserved: " + ex.getMessage(),
                        ex);
            }
        }

        logger.info("InventoryService: successfully reserved inventory for all "
                + reserved.size() + " item(s) in order=" + order.getOrderId());
    }
}
```


---

## 7. Service Implementations (Part 2)

### ProductCatalogService

```java
import java.util.*;
import java.util.stream.Collectors;

public class ProductCatalogService {

    private final Map<String, Product> products = new HashMap<>();
    private final Map<String, Category> categories = new HashMap<>();

    // ---------------------------------------------------------------
    // Category management
    // ---------------------------------------------------------------

    public void addCategory(Category category) {
        categories.put(category.getCategoryId(), category);
    }

    public Optional<Category> getCategory(String categoryId) {
        return Optional.ofNullable(categories.get(categoryId));
    }

    // ---------------------------------------------------------------
    // Product management
    // ---------------------------------------------------------------

    public void addProduct(Product product) {
        products.put(product.getProductId(), product);
    }

    public Product getProduct(String id) throws ProductNotFoundException {
        Product p = products.get(id);
        if (p == null) {
            throw new ProductNotFoundException("Product not found: " + id);
        }
        return p;
    }

    // ---------------------------------------------------------------
    // Search
    // ---------------------------------------------------------------

    /**
     * Filter products by free-text query, category, and price range, then
     * sort by the requested field.
     *
     * @param query      substring to look for in name / description (nullable)
     * @param categoryId filter to this category and all descendants (nullable)
     * @param minPrice   inclusive lower bound in paise/cents (nullable)
     * @param maxPrice   inclusive upper bound in paise/cents (nullable)
     * @param sortBy     "price_asc" | "price_desc" | "rating" (nullable → name asc)
     */
    public List<Product> searchProducts(String query,
                                        String categoryId,
                                        Double minPrice,
                                        Double maxPrice,
                                        String sortBy) {

        // Pre-compute the set of matching category ids once
        Set<String> allowedCategories = (categoryId != null)
                ? getAllDescendantIds(categoryId)
                : null;

        List<Product> result = products.values().stream()
                .filter(p -> matchesQuery(p, query))
                .filter(p -> allowedCategories == null
                        || allowedCategories.contains(p.getCategoryId()))
                .filter(p -> minPrice == null
                        || p.getBasePrice() >= minPrice)
                .filter(p -> maxPrice == null
                        || p.getBasePrice() <= maxPrice)
                .collect(Collectors.toList());

        // Sort
        if ("price_asc".equalsIgnoreCase(sortBy)) {
            result.sort(Comparator.comparingDouble(Product::getBasePrice));
        } else if ("price_desc".equalsIgnoreCase(sortBy)) {
            result.sort(Comparator.comparingDouble(Product::getBasePrice).reversed());
        } else if ("rating".equalsIgnoreCase(sortBy)) {
            result.sort(Comparator.comparingDouble(Product::getAverageRating).reversed());
        } else {
            result.sort(Comparator.comparing(Product::getName));
        }

        return result;
    }

    /** Returns the top-N featured products sorted by average rating. */
    public List<Product> getFeaturedProducts(int limit) {
        return products.values().stream()
                .sorted(Comparator.comparingDouble(Product::getAverageRating).reversed())
                .limit(limit)
                .collect(Collectors.toList());
    }

    /** Returns all products in a specific category (direct membership only). */
    public List<Product> getProductsByCategory(String categoryId) {
        return products.values().stream()
                .filter(p -> categoryId.equals(p.getCategoryId()))
                .collect(Collectors.toList());
    }

    // ---------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------

    /**
     * Returns the given categoryId plus all its descendant category ids
     * (breadth-first traversal of the category tree).
     */
    private Set<String> getAllDescendantIds(String rootId) {
        Set<String> result = new HashSet<>();
        Queue<String> queue = new LinkedList<>();
        queue.add(rootId);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            result.add(current);
            // Find children
            for (Category cat : categories.values()) {
                if (current.equals(cat.getParentCategoryId())
                        && !result.contains(cat.getCategoryId())) {
                    queue.add(cat.getCategoryId());
                }
            }
        }
        return result;
    }

    private boolean matchesQuery(Product p, String query) {
        if (query == null || query.isEmpty()) return true;
        String q = query.toLowerCase();
        return (p.getName() != null && p.getName().toLowerCase().contains(q))
                || (p.getDescription() != null && p.getDescription().toLowerCase().contains(q));
    }
}
```

---

### ReviewService

```java
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

public class ReviewService {

    // ------------------------------------------------------------------
    // Inner value class
    // ------------------------------------------------------------------

    public static class Review {
        private final String reviewId;
        private final String productId;
        private final String userId;
        private final int rating;          // 1–5
        private final String title;
        private final String body;
        private int helpfulVotes;
        private final LocalDateTime createdAt;

        public Review(String reviewId, String productId, String userId,
                      int rating, String title, String body) {
            this.reviewId   = reviewId;
            this.productId  = productId;
            this.userId     = userId;
            this.rating     = rating;
            this.title      = title;
            this.body       = body;
            this.helpfulVotes = 0;
            this.createdAt  = LocalDateTime.now();
        }

        // Getters
        public String  getReviewId()    { return reviewId; }
        public String  getProductId()   { return productId; }
        public String  getUserId()      { return userId; }
        public int     getRating()      { return rating; }
        public String  getTitle()       { return title; }
        public String  getBody()        { return body; }
        public int     getHelpfulVotes(){ return helpfulVotes; }
        public LocalDateTime getCreatedAt() { return createdAt; }

        void incrementHelpful() { helpfulVotes++; }

        @Override
        public String toString() {
            return String.format("Review{id=%s, rating=%d, title='%s', helpful=%d}",
                    reviewId, rating, title, helpfulVotes);
        }
    }

    // ------------------------------------------------------------------
    // State
    // ------------------------------------------------------------------

    /** productId -> list of reviews for that product */
    private final Map<String, List<Review>> reviewsByProduct = new HashMap<>();

    /** reviewId -> Review (for O(1) lookup in markHelpful) */
    private final Map<String, Review> reviewIndex = new HashMap<>();

    // ------------------------------------------------------------------
    // Operations
    // ------------------------------------------------------------------

    /**
     * Adds a new review for a product.
     *
     * @throws IllegalArgumentException if rating is outside 1–5
     */
    public Review addReview(String productId, String userId,
                            int rating, String title, String body) {
        if (rating < 1 || rating > 5) {
            throw new IllegalArgumentException(
                    "Rating must be between 1 and 5; got " + rating);
        }

        String reviewId = "REV-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        Review review = new Review(reviewId, productId, userId, rating, title, body);

        reviewsByProduct.computeIfAbsent(productId, k -> new ArrayList<>()).add(review);
        reviewIndex.put(reviewId, review);

        System.out.println("ReviewService: review " + reviewId
                + " added for product=" + productId
                + " by user=" + userId
                + " rating=" + rating);
        return review;
    }

    /** Returns all reviews for a product (immutable view). */
    public List<Review> getProductReviews(String productId) {
        return Collections.unmodifiableList(
                reviewsByProduct.getOrDefault(productId, Collections.emptyList()));
    }

    /** Computes the average rating for a product; returns 0.0 if no reviews exist. */
    public double getAverageRating(String productId) {
        List<Review> reviews = reviewsByProduct.getOrDefault(productId, Collections.emptyList());
        if (reviews.isEmpty()) return 0.0;
        return reviews.stream()
                .mapToInt(Review::getRating)
                .average()
                .orElse(0.0);
    }

    /**
     * Increments the helpful-vote counter for the specified review.
     *
     * @param reviewId  the review to upvote
     * @param productId only used for logging; the review is looked up by reviewId
     */
    public void markHelpful(String reviewId, String productId) {
        Review review = reviewIndex.get(reviewId);
        if (review == null) {
            System.err.println("ReviewService: review not found: " + reviewId);
            return;
        }
        review.incrementHelpful();
        System.out.println("ReviewService: review " + reviewId
                + " helpful votes=" + review.getHelpfulVotes());
    }

    /**
     * Returns the top-N reviews for a product, sorted by helpful votes descending.
     */
    public List<Review> getTopReviews(String productId, int limit) {
        return reviewsByProduct.getOrDefault(productId, Collections.emptyList())
                .stream()
                .sorted(Comparator.comparingInt(Review::getHelpfulVotes).reversed())
                .limit(limit)
                .collect(Collectors.toList());
    }
}
```

---

### RecommendationEngine (interface + SimpleRecommendationEngine)

```java
import java.util.List;

/**
 * Contract for product recommendation engines.
 * Implementations can range from a simple shuffle (stub) to a full
 * collaborative-filtering ML model — the OrderService and
 * EcommercePlatform facade depend only on this interface.
 */
public interface RecommendationEngine {

    /**
     * Returns a personalised list of product ids for the given user.
     *
     * @param userId the customer whose history / preferences drive the result
     * @param limit  maximum number of product ids to return
     * @return ordered list of product ids (most relevant first)
     */
    List<String> getPersonalizedRecommendations(String userId, int limit);

    /**
     * Returns product ids that are similar to the given product
     * (e.g. same category, similar price, same brand).
     *
     * @param productId the seed product
     * @param limit     maximum number of product ids to return
     * @return ordered list of similar product ids
     */
    List<String> getSimilarProducts(String productId, int limit);

    /**
     * Returns the currently trending product ids in a category.
     *
     * @param categoryId the category to scope the trend to (may be null for global)
     * @param limit      maximum number of product ids to return
     * @return ordered list of trending product ids
     */
    List<String> getTrendingProducts(String categoryId, int limit);
}
```

```java
import java.util.*;
import java.util.stream.Collectors;

/**
 * A stub implementation of {@link RecommendationEngine} that keeps the demo
 * compilable and runnable without an external ML service.
 *
 * Strategy: return a randomly shuffled subset of the products held in the
 * provided {@link ProductCatalogService}.  Real implementations would replace
 * this with collaborative filtering, item embeddings, or a dedicated
 * recommendation micro-service.
 */
public class SimpleRecommendationEngine implements RecommendationEngine {

    private final ProductCatalogService catalogService;
    private final Random random = new Random();

    public SimpleRecommendationEngine(ProductCatalogService catalogService) {
        this.catalogService = catalogService;
    }

    @Override
    public List<String> getPersonalizedRecommendations(String userId, int limit) {
        System.out.println("RecommendationEngine: generating personalised recs for user=" + userId);
        return shuffledProductIds(limit);
    }

    @Override
    public List<String> getSimilarProducts(String productId, int limit) {
        System.out.println("RecommendationEngine: finding products similar to productId=" + productId);
        // Exclude the seed product itself
        return shuffledProductIds(limit + 1).stream()
                .filter(id -> !id.equals(productId))
                .limit(limit)
                .collect(Collectors.toList());
    }

    @Override
    public List<String> getTrendingProducts(String categoryId, int limit) {
        System.out.println("RecommendationEngine: trending products for categoryId="
                + (categoryId != null ? categoryId : "GLOBAL"));
        if (categoryId != null) {
            List<String> inCategory = catalogService
                    .getProductsByCategory(categoryId)
                    .stream()
                    .map(Product::getProductId)
                    .collect(Collectors.toList());
            Collections.shuffle(inCategory, random);
            return inCategory.stream().limit(limit).collect(Collectors.toList());
        }
        return shuffledProductIds(limit);
    }

    // ---------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------

    private List<String> shuffledProductIds(int limit) {
        List<String> ids = new ArrayList<>(
                catalogService.getFeaturedProducts(Integer.MAX_VALUE)
                        .stream()
                        .map(Product::getProductId)
                        .collect(Collectors.toList()));
        Collections.shuffle(ids, random);
        return ids.stream().limit(limit).collect(Collectors.toList());
    }
}
```

---

### OrderService

```java
import java.util.*;
import java.util.logging.Logger;
import java.util.stream.Collectors;

public class OrderService {

    private static final Logger logger = Logger.getLogger(OrderService.class.getName());

    // ------------------------------------------------------------------
    // Dependencies
    // ------------------------------------------------------------------

    private final Map<String, Order> orders = new HashMap<>();
    private final InventoryService inventoryService;
    private final CouponService couponService;
    private final NotificationService notificationService;
    private final PricingStrategy pricingStrategy;

    public OrderService(InventoryService inventoryService,
                        CouponService couponService,
                        NotificationService notificationService,
                        PricingStrategy pricingStrategy) {
        this.inventoryService    = inventoryService;
        this.couponService       = couponService;
        this.notificationService = notificationService;
        this.pricingStrategy     = pricingStrategy;
    }

    // ------------------------------------------------------------------
    // Place order
    // ------------------------------------------------------------------

    /**
     * Full order-placement flow:
     * <ol>
     *   <li>Validate cart is non-empty.</li>
     *   <li>Price each item through the strategy chain.</li>
     *   <li>Apply coupon (if any) to the total.</li>
     *   <li>Build the Order via its Builder.</li>
     *   <li>Reserve inventory (atomic across all items; rolls back on failure).</li>
     *   <li>Process payment.</li>
     *   <li>Transition order to CONFIRMED and persist.</li>
     *   <li>Send notification.</li>
     * </ol>
     *
     * @throws IllegalArgumentException       if the cart is empty
     * @throws InsufficientInventoryException if stock cannot be reserved
     * @throws PaymentFailedException         if the payment processor declines
     */
    public Order placeOrder(String customerId,
                            ShoppingCart cart,
                            Address shippingAddress,
                            Address billingAddress,
                            PaymentMethod paymentMethod)
            throws InsufficientInventoryException, PaymentFailedException {

        // 1. Validate cart
        if (cart.getItems().isEmpty()) {
            throw new IllegalArgumentException("Cannot place an order with an empty cart.");
        }

        // 2. Price each item
        List<OrderItem> orderItems = new ArrayList<>();
        long subtotal = 0;

        for (CartItem cartItem : cart.getItems()) {
            long unitPrice = pricingStrategy.calculatePrice(
                    cartItem.getBaseUnitPrice(),
                    cartItem.getQuantity(),
                    cartItem.getSellerId());

            long lineTotal = unitPrice * cartItem.getQuantity();
            subtotal += lineTotal;

            orderItems.add(new OrderItem(
                    cartItem.getProductId(),
                    cartItem.getVariantId(),
                    cartItem.getSellerId(),
                    cartItem.getProductName(),
                    cartItem.getQuantity(),
                    unitPrice,
                    lineTotal));
        }

        // 3. Apply coupon
        long discount = 0;
        String couponCode = cart.getAppliedCouponCode();
        if (couponCode != null && !couponCode.isEmpty()) {
            try {
                discount = couponService.applyCoupon(couponCode, subtotal, customerId);
                logger.info("OrderService: coupon " + couponCode
                        + " applied, discount=" + discount + " paise");
            } catch (Exception e) {
                logger.warning("OrderService: coupon application failed: " + e.getMessage());
            }
        }

        long totalAmount = subtotal - discount;
        if (totalAmount < 0) totalAmount = 0;

        // 4. Build order
        String orderId = "ORD-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();

        Order order = new Order.Builder(orderId, customerId)
                .items(orderItems)
                .shippingAddress(shippingAddress)
                .billingAddress(billingAddress)
                .paymentMethod(paymentMethod)
                .subtotal(subtotal)
                .discount(discount)
                .totalAmount(totalAmount)
                .status(OrderStatus.PENDING)
                .build();

        // 5. Reserve inventory
        inventoryService.reserveInventoryForOrder(order);

        // 6. Process payment
        PaymentProcessor processor =
                PaymentProcessorFactory.getProcessor(paymentMethod.getType());
        PaymentResult result = processor.processPayment(
                paymentMethod, totalAmount, order.getOrderId());

        if (!result.isSuccess()) {
            // Roll back inventory reservations
            for (OrderItem item : orderItems) {
                try {
                    inventoryService.releaseReservation(
                            item.getSellerId(), item.getVariantId(), item.getQuantity());
                } catch (Exception ex) {
                    logger.severe("OrderService: rollback inventory failed for "
                            + item.getVariantId() + ": " + ex.getMessage());
                }
            }
            throw new PaymentFailedException(
                    "Payment declined for order " + orderId + ": " + result.getMessage());
        }

        // 7. Confirm and persist
        order.setStatus(OrderStatus.CONFIRMED);
        order.setPaymentTransactionId(result.getTransactionId());
        orders.put(orderId, order);

        logger.info("OrderService: order " + orderId
                + " placed successfully for customer=" + customerId
                + " total=" + totalAmount);

        // 8. Notify
        notificationService.notifyOrderPlaced(order);

        return order;
    }

    // ------------------------------------------------------------------
    // Query
    // ------------------------------------------------------------------

    public Order getOrder(String orderId) throws OrderNotFoundException {
        Order order = orders.get(orderId);
        if (order == null) {
            throw new OrderNotFoundException("Order not found: " + orderId);
        }
        return order;
    }

    public List<Order> getOrdersByCustomer(String customerId) {
        return orders.values().stream()
                .filter(o -> customerId.equals(o.getCustomerId()))
                .sorted(Comparator.comparing(Order::getCreatedAt).reversed())
                .collect(Collectors.toList());
    }

    // ------------------------------------------------------------------
    // Cancel
    // ------------------------------------------------------------------

    /**
     * Cancels an order if it is in a cancellable state (PENDING or CONFIRMED).
     * Releases inventory reservations and logs a refund.
     *
     * @throws OrderNotFoundException      if the order does not exist
     * @throws InvalidOrderStateException  if the order cannot be cancelled in its current state
     */
    public void cancelOrder(String orderId, String userId)
            throws OrderNotFoundException, InvalidOrderStateException {

        Order order = getOrder(orderId);

        if (order.getStatus() != OrderStatus.PENDING
                && order.getStatus() != OrderStatus.CONFIRMED) {
            throw new InvalidOrderStateException(
                    "Order " + orderId + " cannot be cancelled in state " + order.getStatus());
        }

        // Release inventory
        for (OrderItem item : order.getItems()) {
            try {
                inventoryService.releaseReservation(
                        item.getSellerId(), item.getVariantId(), item.getQuantity());
            } catch (Exception ex) {
                logger.severe("OrderService: failed to release inventory on cancel for "
                        + item.getVariantId() + ": " + ex.getMessage());
            }
        }

        // Log refund (in production this would call a refund API)
        if (order.getPaymentTransactionId() != null) {
            logger.info("OrderService: initiating refund for txn="
                    + order.getPaymentTransactionId()
                    + " amount=" + order.getTotalAmount());
        }

        order.setStatus(OrderStatus.CANCELLED);
        logger.info("OrderService: order " + orderId + " cancelled by user=" + userId);

        notificationService.notifyOrderCancelled(order);
    }

    // ------------------------------------------------------------------
    // Status transitions
    // ------------------------------------------------------------------

    /**
     * Moves an order through its state machine.
     * Also sets tracking number when transitioning to SHIPPED.
     *
     * @param trackingNumber may be null for non-SHIPPED transitions
     */
    public void updateOrderStatus(String orderId,
                                  OrderStatus newStatus,
                                  String trackingNumber) {
        Order order = orders.get(orderId);
        if (order == null) {
            logger.warning("OrderService: updateOrderStatus called for unknown orderId=" + orderId);
            return;
        }

        OrderStatus previous = order.getStatus();
        order.setStatus(newStatus);

        if (newStatus == OrderStatus.SHIPPED && trackingNumber != null) {
            order.setTrackingNumber(trackingNumber);
            logger.info("OrderService: order " + orderId
                    + " shipped with tracking=" + trackingNumber);
        }

        logger.info("OrderService: order " + orderId
                + " status " + previous + " -> " + newStatus);

        notificationService.notifyOrderStatusChanged(order, previous, newStatus);
    }

    // ------------------------------------------------------------------
    // Returns
    // ------------------------------------------------------------------

    /**
     * Processes a return for a delivered order.
     * Returns inventory to available stock and transitions the order to RETURNED.
     */
    public void processReturn(String orderId, String userId)
            throws OrderNotFoundException, InvalidOrderStateException {

        Order order = getOrder(orderId);

        if (order.getStatus() != OrderStatus.DELIVERED) {
            throw new InvalidOrderStateException(
                    "Order " + orderId + " cannot be returned in state " + order.getStatus());
        }

        for (OrderItem item : order.getItems()) {
            try {
                // Return units to available stock (not just release reservation)
                inventoryService.returnToStock(
                        item.getSellerId(), item.getVariantId(), item.getQuantity());
            } catch (Exception ex) {
                logger.severe("OrderService: failed to return stock for "
                        + item.getVariantId() + ": " + ex.getMessage());
            }
        }

        order.setStatus(OrderStatus.RETURNED);
        logger.info("OrderService: order " + orderId + " returned by user=" + userId);

        notificationService.notifyOrderStatusChanged(order, OrderStatus.DELIVERED, OrderStatus.RETURNED);
    }
}
```

---

### EcommercePlatform (Facade / Orchestrator)

```java
import java.util.*;

/**
 * Facade that wires together all subsystems and exposes a single, cohesive
 * API for the e-commerce platform.
 *
 * Responsibilities:
 * <ul>
 *   <li>Initialise every service and wire their dependencies.</li>
 *   <li>Register default notification observers (Email, SMS, Push).</li>
 *   <li>Set up the default pricing-decorator chain.</li>
 *   <li>Delegate every public call to the appropriate service.</li>
 * </ul>
 *
 * This class is the only entry point callers (controllers, demo, tests) need.
 */
public class EcommercePlatform {

    // ------------------------------------------------------------------
    // Services (all final; wired in constructor)
    // ------------------------------------------------------------------

    private final ProductCatalogService catalogService;
    private final InventoryService      inventoryService;
    private final CouponService         couponService;
    private final NotificationService   notificationService;
    private final OrderService          orderService;
    private final ReviewService         reviewService;
    private final RecommendationEngine  recommendationEngine;

    // ------------------------------------------------------------------
    // Simple in-memory stores for users and carts
    // ------------------------------------------------------------------

    private final Map<String, User>         users = new HashMap<>();
    private final Map<String, ShoppingCart> carts = new HashMap<>();

    // ------------------------------------------------------------------
    // Constructor
    // ------------------------------------------------------------------

    public EcommercePlatform() {

        // Core services with no upstream dependencies
        this.catalogService      = new ProductCatalogService();
        this.inventoryService    = new InventoryService();
        this.couponService       = new CouponService();
        this.notificationService = new NotificationService();
        this.reviewService       = new ReviewService();
        this.recommendationEngine = new SimpleRecommendationEngine(catalogService);

        // Register default notification observers
        notificationService.registerObserver(new EmailNotificationObserver());
        notificationService.registerObserver(new SmsNotificationObserver());
        notificationService.registerObserver(new PushNotificationObserver());

        // Build the pricing decorator chain:
        //   request -> BulkDiscount -> SellerDiscount -> Tax -> BasePricingStrategy
        PricingStrategy baseStrategy   = new BasePricingStrategy();
        PricingStrategy withTax        = new TaxDecorator(baseStrategy);
        PricingStrategy withSeller     = new SellerDiscountDecorator(withTax);
        PricingStrategy withBulk       = new BulkDiscountDecorator(withSeller);

        // OrderService depends on the fully assembled strategy chain
        this.orderService = new OrderService(
                inventoryService,
                couponService,
                notificationService,
                withBulk);
    }

    // ------------------------------------------------------------------
    // User management
    // ------------------------------------------------------------------

    public User registerUser(String userId, String name, String email,
                             String phone, UserRole role) {
        User user = new User(userId, name, email, phone, role);
        users.put(userId, user);
        System.out.println("EcommercePlatform: registered user " + userId
                + " (" + role + ") name=" + name);
        return user;
    }

    public User getUser(String userId) {
        return users.get(userId);
    }

    // ------------------------------------------------------------------
    // Catalog
    // ------------------------------------------------------------------

    public void addCategory(Category category) {
        catalogService.addCategory(category);
    }

    public void addProduct(Product product) {
        catalogService.addProduct(product);
    }

    public Product getProduct(String productId) throws ProductNotFoundException {
        return catalogService.getProduct(productId);
    }

    public List<Product> searchProducts(String query,
                                        String categoryId,
                                        Double minPrice,
                                        Double maxPrice,
                                        String sortBy) {
        return catalogService.searchProducts(query, categoryId, minPrice, maxPrice, sortBy);
    }

    // ------------------------------------------------------------------
    // Inventory
    // ------------------------------------------------------------------

    public void addInventory(String sellerId, String variantId, int quantity) {
        inventoryService.addStock(sellerId, variantId, quantity);
    }

    // ------------------------------------------------------------------
    // Cart
    // ------------------------------------------------------------------

    /** Returns the cart for the customer; creates an empty one if absent. */
    public ShoppingCart getCart(String customerId) {
        return carts.computeIfAbsent(customerId, ShoppingCart::new);
    }

    public void addToCart(String customerId, CartItem item) {
        getCart(customerId).addItem(item);
    }

    public void removeFromCart(String customerId, String variantId) {
        getCart(customerId).removeItem(variantId);
    }

    // ------------------------------------------------------------------
    // Coupon
    // ------------------------------------------------------------------

    public void addCoupon(Coupon coupon) {
        couponService.addCoupon(coupon);
    }

    public void applyCouponToCart(String customerId, String couponCode) {
        getCart(customerId).setAppliedCouponCode(couponCode);
        System.out.println("EcommercePlatform: coupon " + couponCode
                + " applied to cart of customer=" + customerId);
    }

    // ------------------------------------------------------------------
    // Orders
    // ------------------------------------------------------------------

    public Order placeOrder(String customerId,
                            Address shippingAddress,
                            Address billingAddress,
                            PaymentMethod paymentMethod)
            throws InsufficientInventoryException, PaymentFailedException {

        ShoppingCart cart = getCart(customerId);
        Order order = orderService.placeOrder(
                customerId, cart, shippingAddress, billingAddress, paymentMethod);
        // Clear cart after successful order
        carts.remove(customerId);
        return order;
    }

    public Order getOrder(String orderId) throws OrderNotFoundException {
        return orderService.getOrder(orderId);
    }

    public void cancelOrder(String orderId, String userId)
            throws OrderNotFoundException, InvalidOrderStateException {
        orderService.cancelOrder(orderId, userId);
    }

    public void updateOrderStatus(String orderId,
                                  OrderStatus newStatus,
                                  String trackingNumber) {
        orderService.updateOrderStatus(orderId, newStatus, trackingNumber);
    }

    public List<Order> getOrdersByCustomer(String customerId) {
        return orderService.getOrdersByCustomer(customerId);
    }

    // ------------------------------------------------------------------
    // Reviews
    // ------------------------------------------------------------------

    public ReviewService.Review addReview(String productId, String userId,
                                          int rating, String title, String body) {
        return reviewService.addReview(productId, userId, rating, title, body);
    }

    public List<ReviewService.Review> getProductReviews(String productId) {
        return reviewService.getProductReviews(productId);
    }

    // ------------------------------------------------------------------
    // Recommendations
    // ------------------------------------------------------------------

    public List<String> getRecommendations(String userId, int limit) {
        return recommendationEngine.getPersonalizedRecommendations(userId, limit);
    }

    public List<String> getSimilarProducts(String productId, int limit) {
        return recommendationEngine.getSimilarProducts(productId, limit);
    }

    public List<String> getTrendingProducts(String categoryId, int limit) {
        return recommendationEngine.getTrendingProducts(categoryId, limit);
    }
}
```

---

### EcommercePlatformDemo

```java
import java.util.List;

/**
 * End-to-end demonstration of the e-commerce platform.
 *
 * Scenario:
 *   - Two sellers and one customer are registered.
 *   - Categories are created (Electronics > Laptops, Electronics > Phones).
 *   - Two products are listed with variants.
 *   - Inventory is stocked by each seller.
 *   - The customer searches, adds items to the cart, applies a coupon,
 *     and places an order.
 *   - The order is advanced through every status (CONFIRMED -> PROCESSING
 *     -> SHIPPED -> DELIVERED).
 *   - A review is left and recommendations are fetched.
 *   - A second order is placed and then immediately cancelled to demonstrate
 *     the cancellation flow.
 */
public class EcommercePlatformDemo {

    public static void main(String[] args) {

        System.out.println("=== E-Commerce Platform Demo ===\n");

        EcommercePlatform platform = new EcommercePlatform();

        // ------------------------------------------------------------------
        // 1. Register users
        // ------------------------------------------------------------------
        System.out.println("--- Step 1: Register Users ---");

        platform.registerUser("SELLER-001", "TechGadgets Inc.", "seller1@techgadgets.com",
                "+91-9000000001", UserRole.SELLER);
        platform.registerUser("SELLER-002", "MobileZone", "seller2@mobilezone.com",
                "+91-9000000002", UserRole.SELLER);
        platform.registerUser("CUST-001", "Alice Sharma", "alice@example.com",
                "+91-9000000003", UserRole.CUSTOMER);

        System.out.println();

        // ------------------------------------------------------------------
        // 2. Create category hierarchy
        // ------------------------------------------------------------------
        System.out.println("--- Step 2: Create Categories ---");

        Category electronics = new Category("CAT-ELEC", "Electronics", null);
        Category laptops     = new Category("CAT-LAPTOP", "Laptops", "CAT-ELEC");
        Category phones      = new Category("CAT-PHONE", "Phones", "CAT-ELEC");

        platform.addCategory(electronics);
        platform.addCategory(laptops);
        platform.addCategory(phones);

        System.out.println("Created categories: Electronics > Laptops, Electronics > Phones");
        System.out.println();

        // ------------------------------------------------------------------
        // 3. Create products
        // ------------------------------------------------------------------
        System.out.println("--- Step 3: Create Products ---");

        // Laptop with two RAM variants
        Product laptop = new Product.Builder("PROD-001", "UltraBook Pro 14", "CAT-LAPTOP")
                .description("A sleek 14-inch ultrabook with 12th-gen Intel Core i7")
                .basePrice(89999_00L)   // 89,999.00 INR stored in paise
                .brand("TechBrand")
                .addVariant(new ProductVariant("VAR-001-8GB",  "8GB RAM / 512GB SSD",  85000_00L))
                .addVariant(new ProductVariant("VAR-001-16GB", "16GB RAM / 512GB SSD", 95000_00L))
                .build();

        // Phone with two storage variants
        Product phone = new Product.Builder("PROD-002", "SmartPhone X12", "CAT-PHONE")
                .description("Flagship Android phone with 200MP camera and 5G")
                .basePrice(49999_00L)
                .brand("PhoneBrand")
                .addVariant(new ProductVariant("VAR-002-128", "128GB", 49999_00L))
                .addVariant(new ProductVariant("VAR-002-256", "256GB", 54999_00L))
                .build();

        platform.addProduct(laptop);
        platform.addProduct(phone);

        System.out.println("Added products: " + laptop.getName() + ", " + phone.getName());
        System.out.println();

        // ------------------------------------------------------------------
        // 4. Add inventory
        // ------------------------------------------------------------------
        System.out.println("--- Step 4: Stock Inventory ---");

        platform.addInventory("SELLER-001", "VAR-001-8GB",  50);
        platform.addInventory("SELLER-001", "VAR-001-16GB", 30);
        platform.addInventory("SELLER-002", "VAR-002-128",  100);
        platform.addInventory("SELLER-002", "VAR-002-256",  60);

        System.out.println("Inventory stocked by both sellers.");
        System.out.println();

        // ------------------------------------------------------------------
        // 5. Add a coupon
        // ------------------------------------------------------------------
        System.out.println("--- Step 5: Create Coupon ---");

        Coupon welcomeCoupon = new Coupon("WELCOME10", DiscountType.PERCENTAGE, 10, 5000_00L);
        platform.addCoupon(welcomeCoupon);
        System.out.println("Coupon WELCOME10 created: 10% off (max 5000 INR).");
        System.out.println();

        // ------------------------------------------------------------------
        // 6. Customer browses and searches
        // ------------------------------------------------------------------
        System.out.println("--- Step 6: Customer Browses & Searches ---");

        List<Product> searchResults = platform.searchProducts(
                "ultrabook", null, null, null, "price_asc");
        System.out.println("Search 'ultrabook' -> " + searchResults.size() + " result(s):");
        searchResults.forEach(p ->
                System.out.println("  - " + p.getProductId() + " : " + p.getName()));

        List<Product> electronicsResults = platform.searchProducts(
                null, "CAT-ELEC", null, null, "rating");
        System.out.println("Browse Electronics category -> "
                + electronicsResults.size() + " product(s) found.");
        System.out.println();

        // ------------------------------------------------------------------
        // 7. Add items to cart and apply coupon
        // ------------------------------------------------------------------
        System.out.println("--- Step 7: Add Items to Cart ---");

        CartItem laptopItem = new CartItem(
                "PROD-001", "VAR-001-16GB", "SELLER-001",
                "UltraBook Pro 14 (16GB)", 1, 95000_00L);

        CartItem phoneItem = new CartItem(
                "PROD-002", "VAR-002-128", "SELLER-002",
                "SmartPhone X12 (128GB)", 1, 49999_00L);

        platform.addToCart("CUST-001", laptopItem);
        platform.addToCart("CUST-001", phoneItem);
        platform.applyCouponToCart("CUST-001", "WELCOME10");

        System.out.println("Cart for CUST-001 ready with 2 items and coupon WELCOME10.");
        System.out.println();

        // ------------------------------------------------------------------
        // 8. Place order
        // ------------------------------------------------------------------
        System.out.println("--- Step 8: Place Order ---");

        Address shippingAddress = new Address(
                "12, MG Road", "Bengaluru", "Karnataka", "560001", "India");
        Address billingAddress = shippingAddress;   // same for demo
        PaymentMethod creditCard = new PaymentMethod(
                PaymentType.CREDIT_CARD, "4111111111111111", "Alice Sharma");

        Order order = null;
        try {
            order = platform.placeOrder(
                    "CUST-001", shippingAddress, billingAddress, creditCard);

            System.out.println("\nOrder placed successfully!");
            System.out.println("  Order ID  : " + order.getOrderId());
            System.out.println("  Status    : " + order.getStatus());
            System.out.println("  Items     : " + order.getItems().size());
            System.out.println("  Subtotal  : " + order.getSubtotal() / 100.0 + " INR");
            System.out.println("  Discount  : " + order.getDiscount() / 100.0 + " INR");
            System.out.println("  Total     : " + order.getTotalAmount() / 100.0 + " INR");
            System.out.println("  Txn ID    : " + order.getPaymentTransactionId());

        } catch (InsufficientInventoryException | PaymentFailedException e) {
            System.err.println("Order placement failed: " + e.getMessage());
            return;
        }

        System.out.println();

        // ------------------------------------------------------------------
        // 9. Advance order through lifecycle
        // ------------------------------------------------------------------
        System.out.println("--- Step 9: Order Lifecycle ---");

        String orderId = order.getOrderId();

        platform.updateOrderStatus(orderId, OrderStatus.PROCESSING, null);
        System.out.println("  -> PROCESSING: warehouse preparing shipment...");

        platform.updateOrderStatus(orderId, OrderStatus.SHIPPED, "TRK-9988776655");
        System.out.println("  -> SHIPPED: tracking number TRK-9988776655 assigned.");

        platform.updateOrderStatus(orderId, OrderStatus.DELIVERED, null);
        System.out.println("  -> DELIVERED: customer received the package.");

        try {
            Order updated = platform.getOrder(orderId);
            System.out.println("Final order status: " + updated.getStatus()
                    + " | Tracking: " + updated.getTrackingNumber());
        } catch (OrderNotFoundException e) {
            System.err.println(e.getMessage());
        }

        System.out.println();

        // ------------------------------------------------------------------
        // 10. Leave a review
        // ------------------------------------------------------------------
        System.out.println("--- Step 10: Add Review ---");

        ReviewService.Review review = platform.addReview(
                "PROD-001", "CUST-001", 5,
                "Excellent laptop!",
                "Battery life is amazing and performance is top-notch. Highly recommend.");

        System.out.println("Review added: " + review);
        System.out.println();

        // ------------------------------------------------------------------
        // 11. Get recommendations
        // ------------------------------------------------------------------
        System.out.println("--- Step 11: Recommendations ---");

        List<String> recs = platform.getRecommendations("CUST-001", 3);
        System.out.println("Personalised recommendations for CUST-001: " + recs);

        List<String> similar = platform.getSimilarProducts("PROD-001", 2);
        System.out.println("Products similar to PROD-001: " + similar);

        List<String> trending = platform.getTrendingProducts("CAT-ELEC", 3);
        System.out.println("Trending in Electronics: " + trending);

        System.out.println();

        // ------------------------------------------------------------------
        // 12. Demonstrate order cancellation
        // ------------------------------------------------------------------
        System.out.println("--- Step 12: Order Cancellation Demo ---");

        // Give Alice a fresh cart for a second order
        platform.addToCart("CUST-001",
                new CartItem("PROD-002", "VAR-002-256", "SELLER-002",
                        "SmartPhone X12 (256GB)", 1, 54999_00L));

        Order secondOrder = null;
        try {
            secondOrder = platform.placeOrder(
                    "CUST-001", shippingAddress, billingAddress, creditCard);
            System.out.println("Second order placed: " + secondOrder.getOrderId()
                    + " | Status: " + secondOrder.getStatus());

            platform.cancelOrder(secondOrder.getOrderId(), "CUST-001");
            System.out.println("Second order cancelled: " + secondOrder.getOrderId()
                    + " | Status: " + secondOrder.getStatus());

        } catch (Exception e) {
            System.err.println("Error in cancellation demo: " + e.getMessage());
        }

        System.out.println();
        System.out.println("=== Demo Complete ===");
    }
}
```

---

## 8. Design Patterns Used

| Pattern | Where Used | Why |
|---|---|---|
| **Builder** | `Order`, `Product` construction | Orders and products have many optional/required fields; Builder prevents telescoping constructors and enforces invariants before the object is handed out. |
| **Strategy** | `DiscountStrategy`, `PricingStrategy` | Pricing rules (bulk, seller, tax) vary per context. Strategy makes each rule a first-class object that can be swapped or tested in isolation. |
| **Observer** | `NotificationService` (Email, SMS, Push observers) | Order-lifecycle events are consumed by multiple channels that should not be coupled to the order-placement code. Observer decouples emitters from recipients. |
| **Decorator** | `PricingDecorator` chain (`BulkDiscountDecorator` → `SellerDiscountDecorator` → `TaxDecorator`) | Pricing rules compose at runtime — you can add, remove, or reorder wrappers without touching existing code, respecting Open/Closed Principle. |
| **Factory** | `PaymentProcessorFactory` | Payment-method implementations (Credit Card, UPI, Wallet, Net Banking) share the same interface but are selected at runtime by payment type. Factory hides instantiation and keeps calling code open for extension. |
| **Composite** | `Category` tree (parent/child relationships) | A category can be either a leaf or a node with children. `getAllDescendantIds` traverses the tree polymorphically, letting callers treat a subtree the same way they treat a single leaf. |

**Builder** is essential for `Order` because an order aggregates dozens of fields that arrive from different subsystems (cart, coupon, payment, address). Allowing partial construction or setter-based assembly would leave the object in an inconsistent state during payment failures.

**Decorator** outperforms a monolithic pricing function because business rules change independently: tax rates change by jurisdiction, seller discounts are negotiated per contract, bulk thresholds are tuned by category managers. Each concern lives in its own class.

**Observer** keeps `OrderService` unaware of notification channels. Adding a WhatsApp channel later requires only a new observer class registered at startup — zero changes to existing code.

---

## 9. Key Design Decisions and Trade-offs

### Coupon / Discount Engine Design

Coupons (`CouponService`) are validated and consumed before order creation, while structural discounts (bulk, seller) are applied during pricing. This two-layer model separates one-time promotional codes (which must be atomically redeemed to prevent double-spend) from persistent rules that are always active. The trade-off is that coupons must be pre-validated and reserved; in a distributed system this requires a database lock or optimistic concurrency on the coupon's `usageCount`.

### Order State Machine

`OrderStatus` forms a directed acyclic graph (PENDING → CONFIRMED → PROCESSING → SHIPPED → DELIVERED; PENDING/CONFIRMED → CANCELLED; DELIVERED → RETURNED). Transitions are enforced at the service layer rather than the model layer to keep the model a simple value object. The trade-off is that state-machine logic is scattered across `OrderService` methods rather than centralised in an explicit `StateMachine<OrderStatus>` class. For a production system, an event-sourced model (each transition appended as an immutable event) would provide a full audit log.

### Multi-Seller Inventory

Inventory is keyed by `(sellerId, variantId)` rather than just `variantId` because the same SKU may be sold by multiple sellers at different prices from different warehouses. Reservation and release must be atomic per seller-variant combination. In a single-JVM demo this is managed with `synchronized` blocks; in production each would be a row-level lock or a Redis atomic decrement with a Lua script.

### Money as `long` (paise / cents)

All monetary values are stored as `long` integers in the smallest currency unit (paise in India, cents internationally). This avoids floating-point rounding errors that compound over thousands of transactions. The trade-off is that all display code must divide by 100, and the unit must be documented clearly across service boundaries.

### Pricing Decorator Chain

The chain (`BulkDiscount → SellerDiscount → Tax → Base`) applies discounts before tax so the tax base reflects the discounted price. Reversing the order would over-charge tax. Changing this policy requires only reordering the decorators in `EcommercePlatform`'s constructor — no business logic changes.

---

## 10. Extension Points

- **Pluggable recommendation engine**: Replace `SimpleRecommendationEngine` with a ML-backed implementation (e.g. calling a Python FastAPI service) without changing `EcommercePlatform` — the `RecommendationEngine` interface is the only contract.
- **Additional payment methods**: Implement `PaymentProcessor` for any new method (BNPL, crypto) and register it in `PaymentProcessorFactory`; no other class changes.
- **Dynamic coupon types**: Add new `DiscountType` variants (BOGO, free-shipping, loyalty-points) by adding a new branch to `CouponService.applyCoupon` or by introducing a `CouponStrategy` interface.
- **Multi-currency support**: Introduce a `CurrencyConverter` wrapper around `PricingStrategy`; all internal arithmetic stays in base units.
- **Search indexing**: Swap `ProductCatalogService.searchProducts` to delegate to an Elasticsearch/OpenSearch client while keeping the method signature unchanged.
- **Seller onboarding workflow**: Add a `SellerVerificationService` that gates `addInventory` behind KYC checks without touching the `InventoryService` itself.
- **Subscription / recurring orders**: Introduce a `SubscriptionService` that schedules calls to `OrderService.placeOrder` via a job scheduler (e.g. Quartz) — no changes to the core order flow.
- **A/B pricing experiments**: Add an `ExperimentPricingDecorator` at the top of the chain that routes to one of two inner strategies based on a feature flag for the customer; the rest of the chain is unaffected.
- **Audit log**: Register an `AuditObserver` in `NotificationService` that persists every order event to an append-only audit table without touching order-placement code.

---

## 11. Interview Discussion Points

**Q: How would you scale the inventory service to a distributed system?**
Replace in-memory maps with a relational database (PostgreSQL) with row-level locks on `(seller_id, variant_id)`. For higher throughput use Redis with atomic Lua scripts: `DECRBY quantity delta` returns the new value; if it goes negative, roll back with `INCRBY`. Use a distributed lock (Redlock) when the reservation spans multiple variants to prevent partial reservations under concurrent load.

**Q: How do you handle payment failures after inventory has been reserved?**
The saga pattern: each step (reserve inventory, process payment) is a local transaction with a compensating transaction (release inventory, void payment). `OrderService.placeOrder` already implements this compensating rollback synchronously. In a distributed system, publish an `InventoryReserved` domain event; if a subsequent `PaymentFailed` event arrives, a saga orchestrator or choreography-based consumer triggers the compensating `ReleaseInventory` command.

**Q: How do you prevent double-spend on coupon codes?**
Add a `usage_count` column to the coupons table with an optimistic-locking version field. The redemption query is: `UPDATE coupons SET usage_count = usage_count + 1, version = version + 1 WHERE code = ? AND version = ? AND usage_count < max_usage`. If the update affects 0 rows, the coupon is exhausted or was already used concurrently. Wrap this in an idempotency key tied to the `(orderId, couponCode)` pair so a retry does not double-deduct.

**Q: How would you implement a distributed order state machine?**
Persist orders in a database with a `status` and `version` column. Every transition is an `UPDATE orders SET status=?, version=version+1 WHERE order_id=? AND status=? AND version=?`. Emit an `OrderStatusChanged` domain event to a Kafka topic after each successful transition. Downstream consumers (notifications, analytics, warehouse) react to events; the order service itself is not coupled to them.

**Q: How would you scale product search?**
Synchronise the product catalogue to an Elasticsearch index via a Debezium CDC connector on the products table. The `searchProducts` method delegates to an Elasticsearch query (multi-match for full-text, term filters for category/price). Elasticsearch's inverted index handles text search orders-of-magnitude faster than a full table scan.

**Q: How do you keep inventory counts consistent in real time across multiple seller warehouses?**
Each warehouse runs a local inventory service that publishes `StockUpdated` events. A central inventory aggregator consumes these events and updates the canonical count. The platform's reservation call still goes to the canonical store with a two-phase commit or saga to keep local and central counts in sync. For eventually consistent reads, the product detail page shows the aggregated count; the actual reservation validates against the canonical store.

**Q: Why are recommendations eventually consistent, and is that acceptable?**
Recommendation models are trained on batch data (hourly or daily) and served from a pre-computed cache. A user's click or purchase is buffered in a Kafka stream and included in the next training run. The staleness window (minutes to hours) is acceptable because personalisation quality matters more than real-time accuracy. Critical paths (inventory, payment) must be strongly consistent; recommendations can trade consistency for low-latency reads.

**Q: How do you handle partial order fulfilment (one seller has stock, another does not)?**
Introduce the concept of shipments: an order can have multiple `Shipment` records, each tied to a subset of `OrderItem`s and a single seller. If one seller is out of stock, their shipment is marked `BACKORDERED` while others proceed. The customer is notified of the partial dispatch. This requires splitting the reservation loop and transitioning each shipment's status independently while the parent order aggregates the overall state.

---

## 12. Production Considerations

### Database Schema Hints

```
orders          (order_id PK, customer_id FK, status, total_amount, discount,
                 payment_txn_id, tracking_number, created_at, updated_at, version)

order_items     (item_id PK, order_id FK, product_id, variant_id, seller_id,
                 quantity, unit_price, line_total)

inventory       (seller_id, variant_id, available_qty, reserved_qty,
                 PRIMARY KEY (seller_id, variant_id), version)

coupons         (code PK, discount_type, discount_value, min_order_value,
                 max_usage, usage_count, valid_from, valid_until, version)

reviews         (review_id PK, product_id FK, user_id FK, rating, title, body,
                 helpful_votes, created_at)
```

Index `orders(customer_id, created_at DESC)` for customer order history queries. Partition `order_items` by `order_id` range or hash to keep the table manageable at scale.

### Caching Strategy

- **Product catalogue**: Cache product detail pages in Redis with a TTL of 5–15 minutes. Invalidate on `ProductUpdated` events.
- **Inventory counts**: Cache available quantity per `(seller_id, variant_id)` with a short TTL (30 seconds) for product-detail page display. Never cache the reservation check — always read from the source of truth.
- **Recommendations**: Pre-compute and store personalised recommendation lists in Redis keyed by `userId`. Refresh asynchronously after purchase events.
- **Coupon validity**: Cache active coupon metadata for 60 seconds; always perform the atomic decrement against the database.

### Message Queue Integration

Publish domain events to Kafka topics:
- `order.placed`, `order.status.changed`, `order.cancelled` → consumed by notifications, analytics, warehouse management.
- `inventory.reserved`, `inventory.released`, `inventory.low-stock` → consumed by procurement alerts.
- `review.added` → consumed by the recommendation engine's training pipeline.
- `payment.processed`, `payment.failed` → consumed by the finance reconciliation service.

### Monitoring & Alerting

- **P99 latency** on `placeOrder` (target < 2 s end-to-end including payment gateway).
- **Inventory reservation failure rate** — alert if > 1% of reservation attempts fail due to insufficient stock (may indicate catalogue errors or a flash sale).
- **Payment failure rate** — alert if > 2% to catch gateway outages early.
- **Dead-letter queue depth** on Kafka consumers — indicates processing backlogs.
- **Coupon abuse** — alert on sudden spikes in coupon usage per IP or user to detect promotional code leakage.

### Security Considerations

- **PCI-DSS**: Never log or persist raw card numbers. Store only a tokenised reference returned by the payment gateway. The `PaymentMethod` object in this demo is illustrative; in production replace the card number field with a vault token.
- **Authorisation**: Ensure `cancelOrder` and `processReturn` verify that the requesting `userId` is either the order owner or an admin. Add a permission check before every mutating operation.
- **Coupon code brute-force**: Rate-limit coupon validation attempts per user / IP. Use sufficiently long random codes (12+ alphanumeric characters) to make enumeration impractical.
- **Inventory manipulation**: Validate that the `sellerId` in a cart item actually owns inventory for that variant before accepting the order; otherwise a malicious client could route fulfilment to an arbitrary seller.
- **Input validation**: Sanitise all free-text search queries before passing them to any data store to prevent injection attacks (SQL, NoSQL, Elasticsearch query injection).
