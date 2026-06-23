# Strategy Design Pattern

## Table of Contents
1. [Intent and Problem It Solves](#1-intent-and-problem-it-solves)
2. [When to Use / When NOT to Use](#2-when-to-use--when-not-to-use)
3. [UML Class Diagram](#3-uml-class-diagram)
4. [Complete Example 1: Order Discount System](#4-complete-example-1-order-discount-system)
5. [Complete Example 2: Sorting and Payment Processing](#5-complete-example-2-sorting-and-payment-processing)
6. [Runtime Algorithm Switching](#6-runtime-algorithm-switching)
7. [Real-World Framework Examples](#7-real-world-framework-examples)
8. [Trade-offs](#8-trade-offs)
9. [Common Interview Discussion Points](#9-common-interview-discussion-points)
10. [Interview Questions and Model Answers](#10-interview-questions-and-model-answers)
11. [Comparison with Related Patterns](#11-comparison-with-related-patterns)

---

## 1. Intent and Problem It Solves

**GoF Definition:** Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

### The Core Problem: The if/else/switch Smell

Consider building an e-commerce order processing system. You start with a simple discount requirement: apply a 10% discount to all orders. Six months later, the business has grown:

```java
// The Anti-Pattern: Monolithic Discount Logic
public class Order {
    private String customerType;    // "REGULAR", "PREMIUM", "SEASONAL"
    private boolean isLoyalMember;
    private double orderAmount;

    public double calculateFinalPrice() {
        // Every new requirement adds another branch here
        if (customerType.equals("REGULAR")) {
            return orderAmount;
        } else if (customerType.equals("PREMIUM")) {
            return orderAmount * 0.85; // 15% discount
        } else if (customerType.equals("SEASONAL")) {
            if (isBlackFriday()) {
                return orderAmount * 0.60; // 40% off
            } else if (isChristmas()) {
                return orderAmount * 0.75; // 25% off
            } else {
                return orderAmount * 0.90; // 10% off
            }
        } else if (isLoyalMember) {
            if (customerType.equals("PREMIUM")) {
                // Compound discount — but wait, we already handled PREMIUM above
                // Now what? Duplicate the logic? Add another branch?
                return orderAmount * 0.80;
            }
            return orderAmount * 0.95; // 5% loyalty discount
        } else {
            return orderAmount; // fallback — is this right?
        }
    }

    // This method grows every sprint. It violates:
    // - Open/Closed Principle (can't extend without modifying)
    // - Single Responsibility Principle (Order class knows about ALL discount algorithms)
    // - Testability (you can't test discount logic in isolation)
}
```

**Problems with this approach:**

1. **Open/Closed Principle violation** — every new discount type requires modifying `Order`, risking regression in existing logic.
2. **Single Responsibility violation** — `Order` is responsible for domain state (items, quantities, addresses) AND the algorithmic details of every discount type that ever existed.
3. **Untestable in isolation** — you cannot unit-test "percentage discount logic" without constructing a full `Order` object and threading through all the conditional branches.
4. **Combinatorial explosion** — when conditions compound (loyalty + seasonal + premium), branch count grows exponentially.
5. **Duplication** — the same discount percentages appear in multiple places; a pricing change requires hunting through all branches.
6. **Merge conflicts** — three developers simultaneously adding new discount types will conflict on the same method.

### How Strategy Solves It

Strategy extracts each algorithm into its own class behind a common interface. The `Order` class holds a reference to the interface, not any concrete implementation. The result:

- Adding a new discount type means adding a new class, zero changes to existing code.
- Each algorithm is independently testable.
- The algorithm in use can be swapped at runtime.
- Code reads like business language: `order.setDiscountStrategy(new LoyaltyDiscount())`.

---

## 2. When to Use / When NOT to Use

### Use Strategy When

- **You have a family of algorithms** that differ only in behavior, not in the data they operate on. Sorting algorithms, compression algorithms, discount strategies, payment methods, encryption schemes.
- **You want to eliminate conditionals** that select between algorithm variants. Any `if/else` or `switch` that dispatches to different algorithm implementations is a candidate.
- **You need runtime switching** of behavior without touching the client class. A user changing their preferred payment method mid-session, or an admin switching pricing models for an A/B test.
- **Algorithm variations must be independently tested**. Encapsulating each variant in its own class makes unit testing trivial.
- **Clients should not know about data structures used by algorithms**. The `Order` class does not need to know that `LoyaltyDiscount` tracks purchase history; the strategy handles that internally.
- **The class has a field that defines how methods behave** — this is the classic "smelly field" code smell that Strategy cures. If you have a `String type` or `int mode` field that you switch on in multiple methods, extract to Strategy.

### Do NOT Use Strategy When

- **You only have one or two algorithms** that never change. The pattern introduces indirection and a proliferation of small classes. For simple cases, a direct implementation or even a lambda is cleaner.
- **Algorithms don't share a common interface naturally.** If forcing a common interface means the interface becomes bloated with parameters or special-purpose methods, the abstraction is wrong.
- **The client and algorithm are tightly coupled** — the strategy needs so much data from the context that you would effectively pass the entire context object into the strategy, making the decoupling illusory.
- **Performance is the absolute top priority** and the virtual dispatch overhead is measurable. This is rare in business applications but relevant in game physics engines or financial tick processing.
- **The behavior never needs to vary at runtime.** If you know at compile time which algorithm you'll use, inheritance (Template Method) or simple composition without the pattern might be cleaner.
- **You are over-engineering a simple function.** A one-line comparator does not need to be a Strategy. `Comparator.comparingInt(Person::getAge)` is already idiomatic Java — extracting it further adds noise.

---

## 3. UML Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         «interface»                             │
│                       DiscountStrategy                          │
├─────────────────────────────────────────────────────────────────┤
│  + apply(OrderContext ctx) : DiscountResult                     │
│  + getDescription() : String                                    │
└─────────────────────────────────────────────────────────────────┘
            ▲               ▲              ▲              ▲
            │               │              │              │
            │               │              │              │
 ┌──────────┴──┐  ┌─────────┴───┐  ┌──────┴──────┐  ┌───┴──────────────┐
 │ NoDiscount  │  │ Percentage  │  │FlatDiscount │  │SeasonalDiscount  │
 │  Strategy   │  │  Discount   │  │  Strategy   │  │    Strategy      │
 ├─────────────┤  ├─────────────┤  ├─────────────┤  ├──────────────────┤
 │             │  │-percentage  │  │-flatAmount  │  │-discountCalendar │
 │             │  │ :double     │  │ :double     │  │ :DiscountCalendar│
 ├─────────────┤  ├─────────────┤  ├─────────────┤  ├──────────────────┤
 │+apply()     │  │+apply()     │  │+apply()     │  │+apply()          │
 │+getDesc()   │  │+getDesc()   │  │+getDesc()   │  │+getDesc()        │
 └─────────────┘  └─────────────┘  └─────────────┘  └──────────────────┘

 ┌────────────────────────────────────────────────────────────────────┐
 │                        LoyaltyDiscount                             │
 ├────────────────────────────────────────────────────────────────────┤
 │ - tierThresholds : Map<LoyaltyTier, Double>                        │
 │ - loyaltyRepository : LoyaltyRepository                            │
 ├────────────────────────────────────────────────────────────────────┤
 │ + apply(OrderContext ctx) : DiscountResult                         │
 │ + getDescription() : String                                        │
 └────────────────────────────────────────────────────────────────────┘

 ┌─────────────────────────────────────────┐
 │                 Order                   │       ┌──────────────────┐
 │          (Context class)                │       │   OrderContext   │
 ├─────────────────────────────────────────┤  uses │  (Value Object)  │
 │ - orderId         : UUID                │──────>├──────────────────┤
 │ - items           : List<OrderItem>     │       │ - orderAmount    │
 │ - customer        : Customer            │       │ - customer       │
 │ - discountStrategy: DiscountStrategy   │       │ - orderDate      │
 ├─────────────────────────────────────────┤       │ - itemCount      │
 │ + setDiscountStrategy(DiscountStrategy) │       └──────────────────┘
 │ + calculateFinalPrice() : PriceSummary  │
 │ + applyStrategyChain(...) : PriceSummary│
 └─────────────────────────────────────────┘
           │ holds reference to
           ▼
    «interface» DiscountStrategy
```

**Key Relationships:**
- `Order` (Context) holds a reference to the `DiscountStrategy` interface — not any concrete class.
- `Order` delegates the discount calculation to the strategy, passing an `OrderContext` value object.
- Concrete strategies implement the interface independently.
- `Order` and all concrete strategies can evolve independently.

---

## 4. Complete Example 1: Order Discount System

### Package Structure
```
com.ecommerce.discount/
├── strategy/
│   ├── DiscountStrategy.java
│   ├── NoDiscountStrategy.java
│   ├── PercentageDiscountStrategy.java
│   ├── FlatDiscountStrategy.java
│   ├── SeasonalDiscountStrategy.java
│   └── LoyaltyDiscountStrategy.java
├── model/
│   ├── Order.java
│   ├── OrderItem.java
│   ├── OrderContext.java
│   ├── DiscountResult.java
│   ├── PriceSummary.java
│   ├── Customer.java
│   └── LoyaltyTier.java
├── calendar/
│   └── DiscountCalendar.java
└── factory/
    └── DiscountStrategyFactory.java
```

### Strategy Interface

```java
package com.ecommerce.discount.strategy;

import com.ecommerce.discount.model.DiscountResult;
import com.ecommerce.discount.model.OrderContext;

/**
 * Strategy interface for all discount algorithms.
 *
 * <p>Each concrete strategy encapsulates one discount algorithm. The interface
 * is intentionally minimal: strategies receive a rich OrderContext value object
 * instead of individual parameters, so new contextual data can be added without
 * changing any strategy signature.
 *
 * <p>Implementations MUST be thread-safe. Strategies are typically stateless
 * singletons or lightweight objects that hold only configuration data.
 */
public interface DiscountStrategy {

    /**
     * Applies the discount to the given order context.
     *
     * @param context an immutable snapshot of the order state relevant to discounting
     * @return a DiscountResult containing the discounted amount, discount amount, and description
     */
    DiscountResult apply(OrderContext context);

    /**
     * Returns a human-readable description of this discount strategy,
     * suitable for display on receipts and invoices.
     */
    String getDescription();
}
```

### Value Objects and Model Classes

```java
package com.ecommerce.discount.model;

import java.time.LocalDate;
import java.util.Objects;

/**
 * Immutable value object passed to discount strategies.
 * Captures all order data that any strategy might need.
 * Adding new fields here does not break existing strategy signatures.
 */
public final class OrderContext {

    private final double originalAmount;
    private final int itemCount;
    private final Customer customer;
    private final LocalDate orderDate;
    private final String couponCode;

    private OrderContext(Builder builder) {
        this.originalAmount = builder.originalAmount;
        this.itemCount = builder.itemCount;
        this.customer = Objects.requireNonNull(builder.customer, "customer must not be null");
        this.orderDate = Objects.requireNonNull(builder.orderDate, "orderDate must not be null");
        this.couponCode = builder.couponCode; // nullable
    }

    public double getOriginalAmount() { return originalAmount; }
    public int getItemCount()         { return itemCount; }
    public Customer getCustomer()     { return customer; }
    public LocalDate getOrderDate()   { return orderDate; }
    public String getCouponCode()     { return couponCode; }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private double originalAmount;
        private int itemCount;
        private Customer customer;
        private LocalDate orderDate = LocalDate.now();
        private String couponCode;

        public Builder originalAmount(double originalAmount) {
            if (originalAmount < 0) throw new IllegalArgumentException("Amount cannot be negative");
            this.originalAmount = originalAmount;
            return this;
        }
        public Builder itemCount(int itemCount)       { this.itemCount = itemCount; return this; }
        public Builder customer(Customer customer)     { this.customer = customer; return this; }
        public Builder orderDate(LocalDate orderDate) { this.orderDate = orderDate; return this; }
        public Builder couponCode(String couponCode)  { this.couponCode = couponCode; return this; }

        public OrderContext build() { return new OrderContext(this); }
    }
}
```

```java
package com.ecommerce.discount.model;

/**
 * Immutable result produced by a DiscountStrategy.
 */
public final class DiscountResult {

    private final double originalAmount;
    private final double discountAmount;
    private final double finalAmount;
    private final String appliedDescription;

    public DiscountResult(double originalAmount, double discountAmount, String appliedDescription) {
        if (discountAmount < 0) {
            throw new IllegalArgumentException("Discount amount cannot be negative");
        }
        if (discountAmount > originalAmount) {
            throw new IllegalArgumentException("Discount cannot exceed original amount");
        }
        this.originalAmount = originalAmount;
        this.discountAmount = discountAmount;
        this.finalAmount = originalAmount - discountAmount;
        this.appliedDescription = appliedDescription;
    }

    public double getOriginalAmount()      { return originalAmount; }
    public double getDiscountAmount()      { return discountAmount; }
    public double getFinalAmount()         { return finalAmount; }
    public String getAppliedDescription()  { return appliedDescription; }

    @Override
    public String toString() {
        return String.format(
            "DiscountResult{original=%.2f, discount=%.2f, final=%.2f, desc='%s'}",
            originalAmount, discountAmount, finalAmount, appliedDescription
        );
    }
}
```

```java
package com.ecommerce.discount.model;

/**
 * Loyalty tier classification for customers.
 */
public enum LoyaltyTier {
    NONE(0),
    BRONZE(100),
    SILVER(500),
    GOLD(2000),
    PLATINUM(10000);

    private final int minimumLifetimeSpend;

    LoyaltyTier(int minimumLifetimeSpend) {
        this.minimumLifetimeSpend = minimumLifetimeSpend;
    }

    public int getMinimumLifetimeSpend() { return minimumLifetimeSpend; }

    public static LoyaltyTier fromLifetimeSpend(double lifetimeSpend) {
        LoyaltyTier result = NONE;
        for (LoyaltyTier tier : values()) {
            if (lifetimeSpend >= tier.minimumLifetimeSpend) {
                result = tier;
            }
        }
        return result;
    }
}
```

```java
package com.ecommerce.discount.model;

import java.util.Objects;
import java.util.UUID;

/**
 * Domain entity representing a customer.
 */
public class Customer {

    private final UUID customerId;
    private final String name;
    private final String email;
    private final LoyaltyTier loyaltyTier;
    private final double lifetimeSpend;
    private final boolean isPremiumMember;

    public Customer(UUID customerId, String name, String email,
                    double lifetimeSpend, boolean isPremiumMember) {
        this.customerId = Objects.requireNonNull(customerId);
        this.name = Objects.requireNonNull(name);
        this.email = Objects.requireNonNull(email);
        this.lifetimeSpend = lifetimeSpend;
        this.isPremiumMember = isPremiumMember;
        this.loyaltyTier = LoyaltyTier.fromLifetimeSpend(lifetimeSpend);
    }

    public UUID getCustomerId()        { return customerId; }
    public String getName()            { return name; }
    public String getEmail()           { return email; }
    public LoyaltyTier getLoyaltyTier(){ return loyaltyTier; }
    public double getLifetimeSpend()   { return lifetimeSpend; }
    public boolean isPremiumMember()   { return isPremiumMember; }

    @Override
    public String toString() {
        return String.format("Customer{name='%s', tier=%s, premium=%b}", name, loyaltyTier, isPremiumMember);
    }
}
```

```java
package com.ecommerce.discount.model;

/**
 * Final summary presented to the customer after all discounts are applied.
 */
public final class PriceSummary {

    private final double subtotal;
    private final double totalDiscount;
    private final double taxAmount;
    private final double grandTotal;
    private final String discountDescription;

    public PriceSummary(double subtotal, double totalDiscount,
                        double taxRate, String discountDescription) {
        this.subtotal = subtotal;
        this.totalDiscount = totalDiscount;
        double discountedSubtotal = subtotal - totalDiscount;
        this.taxAmount = discountedSubtotal * taxRate;
        this.grandTotal = discountedSubtotal + taxAmount;
        this.discountDescription = discountDescription;
    }

    public double getSubtotal()            { return subtotal; }
    public double getTotalDiscount()       { return totalDiscount; }
    public double getTaxAmount()           { return taxAmount; }
    public double getGrandTotal()          { return grandTotal; }
    public String getDiscountDescription() { return discountDescription; }

    @Override
    public String toString() {
        return String.format(
            "\n=== Price Summary ===\n" +
            "  Subtotal:         $%.2f\n" +
            "  Discount (%s): -$%.2f\n" +
            "  Tax:              $%.2f\n" +
            "  ─────────────────────\n" +
            "  Grand Total:      $%.2f\n",
            subtotal, discountDescription, totalDiscount, taxAmount, grandTotal
        );
    }
}
```

### Concrete Strategy Implementations

```java
package com.ecommerce.discount.strategy;

import com.ecommerce.discount.model.DiscountResult;
import com.ecommerce.discount.model.OrderContext;

/**
 * Applies no discount. Used as the null-object / default strategy.
 * Eliminates null checks throughout the codebase — Order always has a strategy.
 */
public class NoDiscountStrategy implements DiscountStrategy {

    // Stateless — safe to share as a singleton
    public static final NoDiscountStrategy INSTANCE = new NoDiscountStrategy();

    private NoDiscountStrategy() {}

    @Override
    public DiscountResult apply(OrderContext context) {
        return new DiscountResult(context.getOriginalAmount(), 0.0, getDescription());
    }

    @Override
    public String getDescription() {
        return "No Discount";
    }
}
```

```java
package com.ecommerce.discount.strategy;

import com.ecommerce.discount.model.DiscountResult;
import com.ecommerce.discount.model.OrderContext;

/**
 * Applies a fixed percentage discount to the entire order.
 *
 * <p>Thread-safe: percentage is set at construction time and never mutated.
 */
public class PercentageDiscountStrategy implements DiscountStrategy {

    private final double percentage;  // e.g. 15.0 means 15%
    private final String label;

    /**
     * @param percentage discount percentage, e.g. 15.0 for 15%
     * @param label      human-readable label for this discount, e.g. "Premium Member"
     */
    public PercentageDiscountStrategy(double percentage, String label) {
        if (percentage < 0 || percentage > 100) {
            throw new IllegalArgumentException("Percentage must be between 0 and 100, got: " + percentage);
        }
        this.percentage = percentage;
        this.label = label;
    }

    @Override
    public DiscountResult apply(OrderContext context) {
        double discountAmount = context.getOriginalAmount() * (percentage / 100.0);
        // Round to 2 decimal places to avoid floating-point drift on receipts
        discountAmount = Math.round(discountAmount * 100.0) / 100.0;
        return new DiscountResult(
            context.getOriginalAmount(),
            discountAmount,
            getDescription()
        );
    }

    @Override
    public String getDescription() {
        return String.format("%s (%.0f%% off)", label, percentage);
    }
}
```

```java
package com.ecommerce.discount.strategy;

import com.ecommerce.discount.model.DiscountResult;
import com.ecommerce.discount.model.OrderContext;

/**
 * Applies a fixed flat-amount discount, capped at the order total.
 * Common use case: "$20 off your next order" coupon codes.
 *
 * <p>A minimum order threshold prevents abuse on tiny orders.
 */
public class FlatDiscountStrategy implements DiscountStrategy {

    private final double flatAmount;
    private final double minimumOrderAmount;
    private final String label;

    /**
     * @param flatAmount          the fixed discount in dollars, e.g. 20.0
     * @param minimumOrderAmount  minimum order value to qualify, e.g. 50.0
     * @param label               display label, e.g. "Welcome Coupon"
     */
    public FlatDiscountStrategy(double flatAmount, double minimumOrderAmount, String label) {
        if (flatAmount < 0) throw new IllegalArgumentException("Flat amount cannot be negative");
        if (minimumOrderAmount < 0) throw new IllegalArgumentException("Minimum amount cannot be negative");
        this.flatAmount = flatAmount;
        this.minimumOrderAmount = minimumOrderAmount;
        this.label = label;
    }

    @Override
    public DiscountResult apply(OrderContext context) {
        double orderAmount = context.getOriginalAmount();

        if (orderAmount < minimumOrderAmount) {
            // Threshold not met — no discount, but return a meaningful description
            return new DiscountResult(
                orderAmount,
                0.0,
                getDescription() + String.format(" (not applied: min order $%.2f required)", minimumOrderAmount)
            );
        }

        // Never discount more than the order is worth
        double actualDiscount = Math.min(flatAmount, orderAmount);
        return new DiscountResult(orderAmount, actualDiscount, getDescription());
    }

    @Override
    public String getDescription() {
        return String.format("%s ($%.2f flat off)", label, flatAmount);
    }
}
```

```java
package com.ecommerce.discount.strategy;

import com.ecommerce.discount.model.DiscountResult;
import com.ecommerce.discount.model.OrderContext;

import java.time.LocalDate;
import java.time.MonthDay;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Applies time-based seasonal discounts.
 *
 * <p>Reads a configurable discount calendar so new seasons can be added
 * without touching this class. The strategy is thread-safe — the calendar
 * is immutable after construction.
 *
 * <p>Discounts are evaluated in insertion order; the first matching season wins.
 */
public class SeasonalDiscountStrategy implements DiscountStrategy {

    /**
     * Represents a named discount window with a percentage.
     */
    public static final class SeasonWindow {
        private final MonthDay start;
        private final MonthDay end;
        private final double discountPercentage;
        private final String seasonName;

        public SeasonWindow(MonthDay start, MonthDay end,
                            double discountPercentage, String seasonName) {
            this.start = start;
            this.end = end;
            this.discountPercentage = discountPercentage;
            this.seasonName = seasonName;
        }

        public boolean includes(LocalDate date) {
            MonthDay md = MonthDay.from(date);
            // Handle year-crossing windows (e.g. Dec 26 – Jan 5)
            if (start.isAfter(end)) {
                return !md.isBefore(start) || !md.isAfter(end);
            }
            return !md.isBefore(start) && !md.isAfter(end);
        }

        public double getDiscountPercentage() { return discountPercentage; }
        public String getSeasonName()         { return seasonName; }
    }

    private final Map<String, SeasonWindow> seasons;

    public SeasonalDiscountStrategy() {
        // Defaults: common retail seasons in insertion (priority) order
        this.seasons = new LinkedHashMap<>();
        addSeason(new SeasonWindow(
            MonthDay.of(11, 24), MonthDay.of(11, 27), 40.0, "Black Friday"
        ));
        addSeason(new SeasonWindow(
            MonthDay.of(12, 1), MonthDay.of(12, 31), 25.0, "Holiday Season"
        ));
        addSeason(new SeasonWindow(
            MonthDay.of(1, 1), MonthDay.of(1, 7), 20.0, "New Year Sale"
        ));
        addSeason(new SeasonWindow(
            MonthDay.of(7, 4), MonthDay.of(7, 4), 15.0, "Independence Day"
        ));
    }

    /** Constructor for custom season configurations (e.g., injected from database). */
    public SeasonalDiscountStrategy(Map<String, SeasonWindow> seasons) {
        this.seasons = new LinkedHashMap<>(seasons);
    }

    public void addSeason(SeasonWindow window) {
        seasons.put(window.getSeasonName(), window);
    }

    @Override
    public DiscountResult apply(OrderContext context) {
        LocalDate orderDate = context.getOrderDate();

        for (SeasonWindow window : seasons.values()) {
            if (window.includes(orderDate)) {
                double discountAmount = context.getOriginalAmount()
                    * (window.getDiscountPercentage() / 100.0);
                discountAmount = Math.round(discountAmount * 100.0) / 100.0;

                return new DiscountResult(
                    context.getOriginalAmount(),
                    discountAmount,
                    String.format("%s (%.0f%% seasonal off)", window.getSeasonName(),
                                  window.getDiscountPercentage())
                );
            }
        }

        // Outside all seasonal windows — no discount
        return new DiscountResult(
            context.getOriginalAmount(),
            0.0,
            "Seasonal (no active season)"
        );
    }

    @Override
    public String getDescription() {
        return "Seasonal Discount";
    }
}
```

```java
package com.ecommerce.discount.strategy;

import com.ecommerce.discount.model.Customer;
import com.ecommerce.discount.model.DiscountResult;
import com.ecommerce.discount.model.LoyaltyTier;
import com.ecommerce.discount.model.OrderContext;

import java.util.EnumMap;
import java.util.Map;

/**
 * Applies tiered loyalty discounts based on a customer's lifetime spend.
 *
 * <p>Tier thresholds and discount percentages are configurable at construction time,
 * enabling the marketing team to adjust tiers without a code deployment (load from DB
 * or config service and pass in via constructor).
 *
 * <p>Also grants a bonus discount for premium members on top of the tier discount,
 * demonstrating how strategies can compose simple rules internally.
 */
public class LoyaltyDiscountStrategy implements DiscountStrategy {

    private static final double PREMIUM_BONUS_PERCENTAGE = 5.0;

    // Default tier discount map — overridable via constructor
    private static final Map<LoyaltyTier, Double> DEFAULT_TIER_DISCOUNTS;
    static {
        DEFAULT_TIER_DISCOUNTS = new EnumMap<>(LoyaltyTier.class);
        DEFAULT_TIER_DISCOUNTS.put(LoyaltyTier.NONE,     0.0);
        DEFAULT_TIER_DISCOUNTS.put(LoyaltyTier.BRONZE,   3.0);
        DEFAULT_TIER_DISCOUNTS.put(LoyaltyTier.SILVER,   7.0);
        DEFAULT_TIER_DISCOUNTS.put(LoyaltyTier.GOLD,    12.0);
        DEFAULT_TIER_DISCOUNTS.put(LoyaltyTier.PLATINUM, 20.0);
    }

    private final Map<LoyaltyTier, Double> tierDiscounts;

    public LoyaltyDiscountStrategy() {
        this.tierDiscounts = new EnumMap<>(DEFAULT_TIER_DISCOUNTS);
    }

    /** Constructor for externally configured tiers (e.g., from a feature flag service). */
    public LoyaltyDiscountStrategy(Map<LoyaltyTier, Double> tierDiscounts) {
        this.tierDiscounts = new EnumMap<>(tierDiscounts);
    }

    @Override
    public DiscountResult apply(OrderContext context) {
        Customer customer = context.getCustomer();
        LoyaltyTier tier = customer.getLoyaltyTier();

        double tierPercentage = tierDiscounts.getOrDefault(tier, 0.0);
        double premiumBonus = customer.isPremiumMember() ? PREMIUM_BONUS_PERCENTAGE : 0.0;
        double totalPercentage = tierPercentage + premiumBonus;

        if (totalPercentage == 0.0) {
            return new DiscountResult(
                context.getOriginalAmount(),
                0.0,
                "Loyalty (no tier discount — spend more to unlock)"
            );
        }

        double discountAmount = context.getOriginalAmount() * (totalPercentage / 100.0);
        discountAmount = Math.round(discountAmount * 100.0) / 100.0;

        String description = buildDescription(tier, tierPercentage, customer.isPremiumMember(), premiumBonus);
        return new DiscountResult(context.getOriginalAmount(), discountAmount, description);
    }

    private String buildDescription(LoyaltyTier tier, double tierPct,
                                    boolean isPremium, double premiumBonus) {
        StringBuilder sb = new StringBuilder();
        sb.append(tier.name()).append(" Loyalty (").append(String.format("%.0f%%", tierPct)).append(" off)");
        if (isPremium && premiumBonus > 0) {
            sb.append(" + Premium Bonus (").append(String.format("%.0f%%", premiumBonus)).append(" off)");
        }
        return sb.toString();
    }

    @Override
    public String getDescription() {
        return "Loyalty Discount";
    }
}
```

### Context Class: Order

```java
package com.ecommerce.discount.model;

import com.ecommerce.discount.strategy.DiscountStrategy;
import com.ecommerce.discount.strategy.NoDiscountStrategy;

import java.time.LocalDate;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.UUID;

/**
 * The Strategy Pattern Context class.
 *
 * <p>Order holds a reference to a DiscountStrategy interface, delegating
 * all discount calculation to it. The Order class knows nothing about
 * how discounts are calculated — it only knows that it CAN apply one.
 *
 * <p>Key design decisions:
 * <ul>
 *   <li>Default strategy is NoDiscountStrategy (Null Object) — eliminates null checks</li>
 *   <li>Strategy can be swapped at runtime via setDiscountStrategy()</li>
 *   <li>Order constructs an OrderContext value object to pass to the strategy,
 *       keeping the strategy interface stable even as Order evolves</li>
 * </ul>
 */
public class Order {

    private static final double DEFAULT_TAX_RATE = 0.08; // 8% tax

    private final UUID orderId;
    private final Customer customer;
    private final List<OrderItem> items;
    private final LocalDate orderDate;
    private DiscountStrategy discountStrategy;

    public Order(Customer customer) {
        this.orderId = UUID.randomUUID();
        this.customer = Objects.requireNonNull(customer, "customer must not be null");
        this.items = new ArrayList<>();
        this.orderDate = LocalDate.now();
        // Default: Null Object pattern — no null checks needed downstream
        this.discountStrategy = NoDiscountStrategy.INSTANCE;
    }

    // ─── Item Management ────────────────────────────────────────────────────

    public void addItem(OrderItem item) {
        items.add(Objects.requireNonNull(item));
    }

    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }

    // ─── Strategy Setter (runtime switching point) ───────────────────────────

    /**
     * Replaces the current discount strategy. Can be called at any time
     * before calculateFinalPrice() is invoked — enabling runtime switching.
     */
    public void setDiscountStrategy(DiscountStrategy discountStrategy) {
        this.discountStrategy = Objects.requireNonNull(discountStrategy,
            "discountStrategy must not be null — use NoDiscountStrategy.INSTANCE instead");
    }

    public DiscountStrategy getDiscountStrategy() {
        return discountStrategy;
    }

    // ─── Price Calculation (delegates to strategy) ───────────────────────────

    /**
     * Calculates the final price by delegating to the current strategy.
     *
     * @return a PriceSummary with subtotal, discount, tax, and grand total
     */
    public PriceSummary calculateFinalPrice() {
        double subtotal = calculateSubtotal();
        OrderContext context = buildOrderContext(subtotal);
        DiscountResult result = discountStrategy.apply(context);

        return new PriceSummary(
            subtotal,
            result.getDiscountAmount(),
            DEFAULT_TAX_RATE,
            result.getAppliedDescription()
        );
    }

    // ─── Internal Helpers ────────────────────────────────────────────────────

    private double calculateSubtotal() {
        return items.stream()
            .mapToDouble(item -> item.getUnitPrice() * item.getQuantity())
            .sum();
    }

    private OrderContext buildOrderContext(double subtotal) {
        return OrderContext.builder()
            .originalAmount(subtotal)
            .itemCount(items.size())
            .customer(customer)
            .orderDate(orderDate)
            .build();
    }

    // ─── Accessors ───────────────────────────────────────────────────────────

    public UUID getOrderId()       { return orderId; }
    public Customer getCustomer()  { return customer; }
    public LocalDate getOrderDate(){ return orderDate; }

    @Override
    public String toString() {
        return String.format("Order{id=%s, customer='%s', items=%d, strategy=%s}",
            orderId, customer.getName(), items.size(), discountStrategy.getDescription());
    }
}
```

```java
package com.ecommerce.discount.model;

import java.util.Objects;

/**
 * Represents a single line item in an order.
 */
public final class OrderItem {

    private final String productId;
    private final String productName;
    private final double unitPrice;
    private final int quantity;

    public OrderItem(String productId, String productName, double unitPrice, int quantity) {
        if (unitPrice < 0) throw new IllegalArgumentException("Unit price cannot be negative");
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive");
        this.productId = Objects.requireNonNull(productId);
        this.productName = Objects.requireNonNull(productName);
        this.unitPrice = unitPrice;
        this.quantity = quantity;
    }

    public String getProductId()   { return productId; }
    public String getProductName() { return productName; }
    public double getUnitPrice()   { return unitPrice; }
    public int getQuantity()       { return quantity; }
    public double getLineTotal()   { return unitPrice * quantity; }

    @Override
    public String toString() {
        return String.format("OrderItem{%s x%d @ $%.2f}", productName, quantity, unitPrice);
    }
}
```

### Strategy Factory

```java
package com.ecommerce.discount.factory;

import com.ecommerce.discount.model.Customer;
import com.ecommerce.discount.model.LoyaltyTier;
import com.ecommerce.discount.strategy.*;

/**
 * Factory for selecting the appropriate DiscountStrategy based on business rules.
 *
 * <p>This is one level above the Strategy pattern itself: it encapsulates the
 * logic for WHICH strategy to select, keeping that decision out of both the
 * Order class and the call site.
 *
 * <p>In a Spring application, this would be a @Service with injected strategy beans.
 */
public class DiscountStrategyFactory {

    private final SeasonalDiscountStrategy seasonalStrategy;
    private final LoyaltyDiscountStrategy loyaltyStrategy;

    public DiscountStrategyFactory() {
        this.seasonalStrategy = new SeasonalDiscountStrategy();
        this.loyaltyStrategy = new LoyaltyDiscountStrategy();
    }

    /**
     * Resolves the best strategy for the given customer.
     *
     * <p>Priority: Premium Member > Seasonal > Loyalty > No Discount.
     * In a real system this might query a promotions engine or feature flag service.
     */
    public DiscountStrategy resolve(Customer customer) {
        if (customer.isPremiumMember()) {
            return new PercentageDiscountStrategy(15.0, "Premium Member");
        }

        if (customer.getLoyaltyTier() != LoyaltyTier.NONE) {
            return loyaltyStrategy;
        }

        // Check if there is an active seasonal discount
        // (we apply seasonal to all non-premium, non-loyalty customers)
        return seasonalStrategy;
    }

    /**
     * Returns a flat-discount strategy for coupon-code scenarios.
     */
    public DiscountStrategy forCoupon(String couponCode, double discountAmount, double minOrder) {
        return new FlatDiscountStrategy(discountAmount, minOrder, "Coupon[" + couponCode + "]");
    }

    public DiscountStrategy noDiscount() {
        return NoDiscountStrategy.INSTANCE;
    }
}
```

### Running the Discount System

```java
package com.ecommerce.discount;

import com.ecommerce.discount.factory.DiscountStrategyFactory;
import com.ecommerce.discount.model.*;
import com.ecommerce.discount.strategy.*;

import java.util.UUID;

/**
 * Demonstrates the Strategy pattern in action for the discount system.
 * Run main() to see all strategies applied to various order scenarios.
 */
public class DiscountSystemDemo {

    public static void main(String[] args) {

        DiscountStrategyFactory factory = new DiscountStrategyFactory();

        // ── Scenario 1: Regular customer, no discount ───────────────────────
        System.out.println("=== Scenario 1: Regular Customer ===");
        Customer regularCustomer = new Customer(
            UUID.randomUUID(), "Alice Smith", "alice@example.com",
            50.0, false   // $50 lifetime spend, not premium
        );
        Order order1 = buildSampleOrder(regularCustomer);
        order1.setDiscountStrategy(factory.resolve(regularCustomer)); // resolves to seasonal
        System.out.println(order1.calculateFinalPrice());

        // ── Scenario 2: Premium member ───────────────────────────────────────
        System.out.println("=== Scenario 2: Premium Member ===");
        Customer premiumCustomer = new Customer(
            UUID.randomUUID(), "Bob Jones", "bob@example.com",
            3000.0, true  // Gold tier, premium member
        );
        Order order2 = buildSampleOrder(premiumCustomer);
        order2.setDiscountStrategy(new PercentageDiscountStrategy(15.0, "Premium Member"));
        System.out.println(order2.calculateFinalPrice());

        // ── Scenario 3: Platinum loyalty customer ────────────────────────────
        System.out.println("=== Scenario 3: Platinum Loyalty Customer ===");
        Customer platinumCustomer = new Customer(
            UUID.randomUUID(), "Carol White", "carol@example.com",
            15000.0, false // Platinum tier, not premium
        );
        Order order3 = buildSampleOrder(platinumCustomer);
        order3.setDiscountStrategy(new LoyaltyDiscountStrategy());
        System.out.println(order3.calculateFinalPrice());

        // ── Scenario 4: Flat coupon discount ────────────────────────────────
        System.out.println("=== Scenario 4: Coupon Code ===");
        Order order4 = buildSampleOrder(regularCustomer);
        order4.setDiscountStrategy(factory.forCoupon("WELCOME20", 20.0, 50.0));
        System.out.println(order4.calculateFinalPrice());

        // ── Scenario 5: Runtime strategy switching ───────────────────────────
        System.out.println("=== Scenario 5: Runtime Strategy Switch ===");
        Order order5 = buildSampleOrder(premiumCustomer);

        System.out.println("Before switch (No Discount):");
        System.out.println(order5.calculateFinalPrice());

        // Customer applies a coupon code at checkout — switch strategy on the fly
        order5.setDiscountStrategy(factory.forCoupon("SAVE30", 30.0, 0.0));
        System.out.println("After applying coupon SAVE30:");
        System.out.println(order5.calculateFinalPrice());
    }

    private static Order buildSampleOrder(Customer customer) {
        Order order = new Order(customer);
        order.addItem(new OrderItem("SKU-001", "Wireless Headphones", 89.99, 1));
        order.addItem(new OrderItem("SKU-002", "USB-C Cable",          12.99, 2));
        order.addItem(new OrderItem("SKU-003", "Phone Case",           24.99, 1));
        // Subtotal: 89.99 + 25.98 + 24.99 = $140.96
        return order;
    }
}
```

---

## 5. Complete Example 2: Sorting and Payment Processing

### Part A: Sorting with Comparator (Comparator IS the Strategy Pattern)

One of the most elegant proofs that Strategy is everywhere in Java: `java.util.Comparator` is a textbook Strategy interface. The client (`Collections.sort`, `List.sort`, `Arrays.sort`) does not know how elements are compared; it delegates to whatever `Comparator` you pass in.

```java
package com.examples.sorting;

import java.util.*;
import java.util.stream.Collectors;

/**
 * Demonstrates that Comparator<T> is the Strategy interface for sorting.
 *
 * Context:   Collections.sort() / List.sort()
 * Strategy:  Comparator<T>
 * Concrete:  ByAgeComparator, ByNameComparator, ByDepartmentThenSalary, etc.
 */
public class SortingStrategyDemo {

    // ─── Domain Object ────────────────────────────────────────────────────

    public static final class Employee {
        private final String name;
        private final String department;
        private final int age;
        private final double salary;
        private final int yearsOfExperience;

        public Employee(String name, String department, int age,
                        double salary, int yearsOfExperience) {
            this.name = name;
            this.department = department;
            this.age = age;
            this.salary = salary;
            this.yearsOfExperience = yearsOfExperience;
        }

        public String getName()            { return name; }
        public String getDepartment()      { return department; }
        public int getAge()                { return age; }
        public double getSalary()          { return salary; }
        public int getYearsOfExperience()  { return yearsOfExperience; }

        @Override
        public String toString() {
            return String.format("%-20s | %-15s | Age:%-3d | $%,.0f | %d yrs",
                name, department, age, salary, yearsOfExperience);
        }
    }

    // ─── Concrete Strategy Implementations (Comparators) ─────────────────

    /** Strategy 1: Sort by name alphabetically */
    public static class ByNameComparator implements Comparator<Employee> {
        @Override
        public int compare(Employee e1, Employee e2) {
            return e1.getName().compareToIgnoreCase(e2.getName());
        }
    }

    /** Strategy 2: Sort by age ascending */
    public static class ByAgeComparator implements Comparator<Employee> {
        @Override
        public int compare(Employee e1, Employee e2) {
            return Integer.compare(e1.getAge(), e2.getAge());
        }
    }

    /** Strategy 3: Sort by salary descending */
    public static class BySalaryDescendingComparator implements Comparator<Employee> {
        @Override
        public int compare(Employee e1, Employee e2) {
            // Note: reversed order for descending
            return Double.compare(e2.getSalary(), e1.getSalary());
        }
    }

    /** Strategy 4: Multi-field sort — department, then salary descending within department */
    public static class ByDepartmentThenSalaryComparator implements Comparator<Employee> {
        @Override
        public int compare(Employee e1, Employee e2) {
            int deptCompare = e1.getDepartment().compareToIgnoreCase(e2.getDepartment());
            if (deptCompare != 0) return deptCompare;
            // Within same department, higher salary first
            return Double.compare(e2.getSalary(), e1.getSalary());
        }
    }

    /**
     * Strategy 5: Rank employees by composite score.
     * Demonstrates that a strategy can have its own internal algorithm complexity.
     */
    public static class ByCompositeScoreComparator implements Comparator<Employee> {
        // Weights configurable at construction time
        private final double salaryWeight;
        private final double experienceWeight;

        public ByCompositeScoreComparator(double salaryWeight, double experienceWeight) {
            this.salaryWeight = salaryWeight;
            this.experienceWeight = experienceWeight;
        }

        @Override
        public int compare(Employee e1, Employee e2) {
            double score1 = computeScore(e1);
            double score2 = computeScore(e2);
            return Double.compare(score2, score1); // descending: higher score first
        }

        private double computeScore(Employee e) {
            // Normalize salary to a 0-100 scale assuming max salary ~$200K
            double normalizedSalary = Math.min(e.getSalary() / 2000.0, 100.0);
            double normalizedExperience = Math.min(e.getYearsOfExperience() * 5.0, 100.0);
            return (salaryWeight * normalizedSalary) + (experienceWeight * normalizedExperience);
        }
    }

    // ─── Context Class ────────────────────────────────────────────────────

    /**
     * EmployeeRoster wraps the sort operation as a proper Strategy context,
     * making the pattern explicit and allowing runtime sort order changes.
     */
    public static class EmployeeRoster {
        private final List<Employee> employees;
        private Comparator<Employee> sortStrategy;

        public EmployeeRoster(List<Employee> employees) {
            this.employees = new ArrayList<>(employees);
            this.sortStrategy = new ByNameComparator(); // default
        }

        /** Swap the sort algorithm at runtime */
        public void setSortStrategy(Comparator<Employee> sortStrategy) {
            this.sortStrategy = Objects.requireNonNull(sortStrategy);
        }

        /** Returns a new sorted list — does not mutate internal state */
        public List<Employee> getSortedEmployees() {
            return employees.stream()
                .sorted(sortStrategy)
                .collect(Collectors.toList());
        }

        public void printSorted(String header) {
            System.out.println("\n=== " + header + " ===");
            getSortedEmployees().forEach(System.out::println);
        }
    }

    // ─── Demo ──────────────────────────────────────────────────────────────

    public static void main(String[] args) {
        List<Employee> staff = Arrays.asList(
            new Employee("Diana Prince",   "Engineering", 32, 120000, 8),
            new Employee("Bruce Wayne",    "Finance",     45, 185000, 20),
            new Employee("Clark Kent",     "Engineering", 28, 95000,  4),
            new Employee("Barry Allen",    "Engineering", 26, 88000,  2),
            new Employee("Arthur Curry",   "Finance",     38, 110000, 12),
            new Employee("Hal Jordan",     "Marketing",   41, 102000, 15),
            new Employee("Victor Stone",   "Engineering", 30, 130000, 6)
        );

        EmployeeRoster roster = new EmployeeRoster(staff);

        // Switch sorting strategy at runtime
        roster.setSortStrategy(new ByNameComparator());
        roster.printSorted("Sorted by Name");

        roster.setSortStrategy(new BySalaryDescendingComparator());
        roster.printSorted("Sorted by Salary (Descending)");

        roster.setSortStrategy(new ByDepartmentThenSalaryComparator());
        roster.printSorted("Sorted by Department, then Salary");

        // Lambda as inline strategy — still the Strategy pattern, just more concise
        roster.setSortStrategy(
            Comparator.comparingInt(Employee::getYearsOfExperience).reversed()
        );
        roster.printSorted("Sorted by Experience (Most Experienced First)");

        // Composite score strategy with custom weights
        roster.setSortStrategy(new ByCompositeScoreComparator(0.7, 0.3));
        roster.printSorted("Sorted by Composite Score (70% salary, 30% experience)");
    }
}
```

### Part B: Payment Processor with Multiple Strategies

```java
package com.examples.payment;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.Instant;
import java.util.UUID;

// ─── Strategy Interface ────────────────────────────────────────────────────

/**
 * Strategy interface for all payment methods.
 *
 * <p>The interface intentionally avoids payment-method-specific fields.
 * Each concrete strategy carries its own credentials and configuration,
 * received at construction time (e.g., from a Spring @Value or Vault secret).
 */
public interface PaymentStrategy {

    /**
     * Processes the payment.
     *
     * @param request  the payment request (amount, currency, customer info)
     * @return a PaymentResult with transaction ID, status, and fee breakdown
     * @throws PaymentException if the payment gateway rejects the request
     */
    PaymentResult processPayment(PaymentRequest request) throws PaymentException;

    /** Whether this payment method supports refunds. */
    boolean supportsRefund();

    /** Human-readable name for logging and receipts. */
    String getPaymentMethodName();
}

// ─── Value Objects ──────────────────────────────────────────────────────────

class PaymentRequest {
    private final String customerId;
    private final BigDecimal amount;
    private final String currency;
    private final String description;

    public PaymentRequest(String customerId, BigDecimal amount,
                          String currency, String description) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Payment amount must be positive");
        }
        this.customerId = customerId;
        this.amount = amount;
        this.currency = currency;
        this.description = description;
    }

    public String getCustomerId()  { return customerId; }
    public BigDecimal getAmount()  { return amount; }
    public String getCurrency()    { return currency; }
    public String getDescription() { return description; }
}

class PaymentResult {
    public enum Status { SUCCESS, FAILED, PENDING }

    private final String transactionId;
    private final Status status;
    private final BigDecimal chargedAmount;
    private final BigDecimal processingFee;
    private final String gatewayReference;
    private final Instant processedAt;

    public PaymentResult(String transactionId, Status status, BigDecimal chargedAmount,
                         BigDecimal processingFee, String gatewayReference) {
        this.transactionId = transactionId;
        this.status = status;
        this.chargedAmount = chargedAmount;
        this.processingFee = processingFee;
        this.gatewayReference = gatewayReference;
        this.processedAt = Instant.now();
    }

    public String getTransactionId()     { return transactionId; }
    public Status getStatus()            { return status; }
    public BigDecimal getChargedAmount() { return chargedAmount; }
    public BigDecimal getProcessingFee() { return processingFee; }
    public String getGatewayReference()  { return gatewayReference; }
    public Instant getProcessedAt()      { return processedAt; }

    @Override
    public String toString() {
        return String.format(
            "PaymentResult{txId=%s, status=%s, charged=%s, fee=%s, ref=%s}",
            transactionId, status, chargedAmount, processingFee, gatewayReference
        );
    }
}

class PaymentException extends Exception {
    private final String errorCode;

    public PaymentException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}

// ─── Concrete Strategy 1: Credit Card ──────────────────────────────────────

class CreditCardPaymentStrategy implements PaymentStrategy {

    // In production: injected from Vault or environment variables
    private final String stripeApiKey;
    private final String cardToken;   // tokenized card reference, never raw PAN
    private static final BigDecimal PROCESSING_FEE_RATE = new BigDecimal("0.029"); // 2.9%
    private static final BigDecimal FIXED_FEE = new BigDecimal("0.30");            // $0.30

    public CreditCardPaymentStrategy(String stripeApiKey, String cardToken) {
        this.stripeApiKey = stripeApiKey;
        this.cardToken = cardToken;
    }

    @Override
    public PaymentResult processPayment(PaymentRequest request) throws PaymentException {
        // In production: call Stripe API with stripeApiKey and cardToken
        validateCardToken();

        BigDecimal fee = request.getAmount()
            .multiply(PROCESSING_FEE_RATE)
            .add(FIXED_FEE)
            .setScale(2, RoundingMode.HALF_UP);

        BigDecimal totalCharge = request.getAmount().add(fee);
        String gatewayRef = "stripe_" + UUID.randomUUID().toString().replace("-", "").substring(0, 16);

        System.out.printf("[CreditCard] Charged %s %s (fee: %s) via Stripe. Ref: %s%n",
            totalCharge, request.getCurrency(), fee, gatewayRef);

        return new PaymentResult(
            UUID.randomUUID().toString(),
            PaymentResult.Status.SUCCESS,
            totalCharge,
            fee,
            gatewayRef
        );
    }

    private void validateCardToken() throws PaymentException {
        if (cardToken == null || cardToken.isBlank()) {
            throw new PaymentException("Invalid card token", "INVALID_CARD_TOKEN");
        }
    }

    @Override
    public boolean supportsRefund() { return true; }

    @Override
    public String getPaymentMethodName() { return "Credit Card (Stripe)"; }
}

// ─── Concrete Strategy 2: PayPal ───────────────────────────────────────────

class PayPalPaymentStrategy implements PaymentStrategy {

    private final String paypalAccessToken;
    private final String payerEmail;
    private static final BigDecimal FEE_RATE = new BigDecimal("0.034"); // 3.4%

    public PayPalPaymentStrategy(String paypalAccessToken, String payerEmail) {
        this.paypalAccessToken = paypalAccessToken;
        this.payerEmail = payerEmail;
    }

    @Override
    public PaymentResult processPayment(PaymentRequest request) throws PaymentException {
        if (!payerEmail.contains("@")) {
            throw new PaymentException("Invalid PayPal account email", "INVALID_PAYPAL_ACCOUNT");
        }

        BigDecimal fee = request.getAmount()
            .multiply(FEE_RATE)
            .setScale(2, RoundingMode.HALF_UP);

        BigDecimal totalCharge = request.getAmount().add(fee);
        String gatewayRef = "PAYPAL-" + UUID.randomUUID().toString().toUpperCase().replace("-", "").substring(0, 17);

        System.out.printf("[PayPal] Payment of %s %s from %s. Fee: %s. OrderID: %s%n",
            totalCharge, request.getCurrency(), payerEmail, fee, gatewayRef);

        return new PaymentResult(
            UUID.randomUUID().toString(),
            PaymentResult.Status.SUCCESS,
            totalCharge,
            fee,
            gatewayRef
        );
    }

    @Override
    public boolean supportsRefund() { return true; }

    @Override
    public String getPaymentMethodName() { return "PayPal"; }
}

// ─── Concrete Strategy 3: Cryptocurrency ───────────────────────────────────

class CryptoPaymentStrategy implements PaymentStrategy {

    public enum CryptoCurrency { BTC, ETH, USDT }

    private final CryptoCurrency cryptoCurrency;
    private final String walletAddress;
    private final double exchangeRate; // crypto per USD

    public CryptoPaymentStrategy(CryptoCurrency cryptoCurrency,
                                  String walletAddress,
                                  double exchangeRate) {
        this.cryptoCurrency = cryptoCurrency;
        this.walletAddress = walletAddress;
        this.exchangeRate = exchangeRate;
    }

    @Override
    public PaymentResult processPayment(PaymentRequest request) throws PaymentException {
        if (walletAddress == null || walletAddress.length() < 26) {
            throw new PaymentException("Invalid wallet address", "INVALID_WALLET");
        }

        // Network fee is flat (gas fee simulation)
        BigDecimal networkFee = new BigDecimal("0.50"); // $0.50 equivalent
        BigDecimal totalUsd = request.getAmount().add(networkFee);
        double cryptoAmount = totalUsd.doubleValue() * exchangeRate;

        String txHash = "0x" + UUID.randomUUID().toString().replace("-", "");

        System.out.printf("[Crypto] Sent %.8f %s to %s. Network fee: $%s. TxHash: %s%n",
            cryptoAmount, cryptoCurrency, walletAddress, networkFee, txHash.substring(0, 20) + "...");

        return new PaymentResult(
            UUID.randomUUID().toString(),
            PaymentResult.Status.PENDING,  // blockchain confirmation takes time
            totalUsd,
            networkFee,
            txHash
        );
    }

    @Override
    public boolean supportsRefund() {
        return false; // blockchain transactions are irreversible
    }

    @Override
    public String getPaymentMethodName() {
        return "Cryptocurrency (" + cryptoCurrency + ")";
    }
}

// ─── Concrete Strategy 4: Bank Transfer (ACH) ──────────────────────────────

class BankTransferPaymentStrategy implements PaymentStrategy {

    private final String routingNumber;
    private final String accountNumberLast4;  // Store only last 4 digits for security
    private final String accountHolderName;
    private static final BigDecimal ACH_FEE = new BigDecimal("0.25"); // flat $0.25

    public BankTransferPaymentStrategy(String routingNumber, String accountNumberLast4,
                                        String accountHolderName) {
        this.routingNumber = routingNumber;
        this.accountNumberLast4 = accountNumberLast4;
        this.accountHolderName = accountHolderName;
    }

    @Override
    public PaymentResult processPayment(PaymentRequest request) throws PaymentException {
        if (routingNumber == null || routingNumber.length() != 9) {
            throw new PaymentException("Invalid ACH routing number", "INVALID_ROUTING");
        }

        String achRef = "ACH" + System.currentTimeMillis();

        System.out.printf("[ACH] Initiating transfer of %s %s from account ending %s. Ref: %s%n",
            request.getAmount(), request.getCurrency(), accountNumberLast4, achRef);

        return new PaymentResult(
            UUID.randomUUID().toString(),
            PaymentResult.Status.PENDING,  // ACH takes 1-3 business days
            request.getAmount().add(ACH_FEE),
            ACH_FEE,
            achRef
        );
    }

    @Override
    public boolean supportsRefund() { return true; }

    @Override
    public String getPaymentMethodName() {
        return "Bank Transfer (ACH) — account ending " + accountNumberLast4;
    }
}

// ─── Context Class: PaymentProcessor ───────────────────────────────────────

class PaymentProcessor {

    private PaymentStrategy paymentStrategy;

    public PaymentProcessor(PaymentStrategy initialStrategy) {
        this.paymentStrategy = initialStrategy;
    }

    /** Runtime strategy switch — e.g., user changes payment method at checkout */
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = java.util.Objects.requireNonNull(strategy);
        System.out.println("[PaymentProcessor] Switched to: " + strategy.getPaymentMethodName());
    }

    public PaymentResult process(PaymentRequest request) throws PaymentException {
        System.out.printf("%n--- Processing payment via %s ---%n",
            paymentStrategy.getPaymentMethodName());

        PaymentResult result = paymentStrategy.processPayment(request);

        System.out.printf("Result: %s%n", result);

        if (result.getStatus() == PaymentResult.Status.PENDING
                && !paymentStrategy.supportsRefund()) {
            System.out.println("[WARNING] This payment method does not support refunds.");
        }

        return result;
    }

    public boolean currentStrategySupportsRefund() {
        return paymentStrategy.supportsRefund();
    }
}

// ─── Demo ───────────────────────────────────────────────────────────────────

class PaymentDemo {

    public static void main(String[] args) throws PaymentException {

        PaymentRequest request = new PaymentRequest(
            "CUST-001",
            new BigDecimal("249.99"),
            "USD",
            "Order #ORD-20240615-9842"
        );

        // Start with credit card
        PaymentProcessor processor = new PaymentProcessor(
            new CreditCardPaymentStrategy("sk_live_...", "tok_visa_4242")
        );
        processor.process(request);

        // Customer switches to PayPal at checkout — runtime switch
        processor.setPaymentStrategy(
            new PayPalPaymentStrategy("A21AAFV...", "customer@gmail.com")
        );
        processor.process(request);

        // Customer wants to pay with Ethereum
        processor.setPaymentStrategy(
            new CryptoPaymentStrategy(
                CryptoPaymentStrategy.CryptoCurrency.ETH,
                "0x742d35Cc6634C0532925a3b844Bc9e7595fCbAb4",
                0.000373 // ETH per USD
            )
        );
        processor.process(request);

        // ACH for large B2B transactions
        processor.setPaymentStrategy(
            new BankTransferPaymentStrategy("021000021", "6789", "TechCorp Inc.")
        );
        processor.process(request);
    }
}
```

---

## 6. Runtime Algorithm Switching

One of the defining advantages of Strategy is that the algorithm can change at any point in the object's lifecycle. The following example shows a more realistic runtime switching scenario: a recommendation engine that switches algorithm based on current load.

```java
package com.examples.recommendation;

import java.util.*;
import java.util.concurrent.atomic.AtomicReference;

/**
 * Demonstrates runtime strategy switching driven by external conditions.
 *
 * Real-world analogy: a recommendation engine that degrades gracefully
 * under high load by switching from a compute-intensive ML model
 * to a faster rule-based approach.
 */
public class AdaptiveRecommendationEngine {

    // ─── Strategy Interface ─────────────────────────────────────────────

    public interface RecommendationStrategy {
        List<String> recommend(String userId, int maxResults);
        String getAlgorithmName();
        boolean isHeavyweight(); // indicates compute cost
    }

    // ─── Concrete Strategy: Collaborative Filtering (ML) ───────────────

    public static class CollaborativeFilteringStrategy implements RecommendationStrategy {
        private final String modelVersion;

        public CollaborativeFilteringStrategy(String modelVersion) {
            this.modelVersion = modelVersion;
        }

        @Override
        public List<String> recommend(String userId, int maxResults) {
            // In production: calls ML inference service (latency: 150-400ms)
            System.out.printf("[CF-%s] Running collaborative filtering for user %s...%n",
                modelVersion, userId);
            // Simulate model output
            return Arrays.asList("PROD-101", "PROD-234", "PROD-088", "PROD-456", "PROD-312")
                .subList(0, Math.min(maxResults, 5));
        }

        @Override
        public String getAlgorithmName() { return "Collaborative Filtering v" + modelVersion; }

        @Override
        public boolean isHeavyweight() { return true; }
    }

    // ─── Concrete Strategy: Content-Based Filtering ─────────────────────

    public static class ContentBasedStrategy implements RecommendationStrategy {

        @Override
        public List<String> recommend(String userId, int maxResults) {
            // In production: fast attribute matching (latency: 20-50ms)
            System.out.printf("[ContentBased] Running content-based filtering for user %s...%n", userId);
            return Arrays.asList("PROD-500", "PROD-501", "PROD-502").subList(0, Math.min(maxResults, 3));
        }

        @Override
        public String getAlgorithmName() { return "Content-Based Filtering"; }

        @Override
        public boolean isHeavyweight() { return false; }
    }

    // ─── Concrete Strategy: Trending Items Fallback ──────────────────────

    public static class TrendingItemsStrategy implements RecommendationStrategy {

        @Override
        public List<String> recommend(String userId, int maxResults) {
            // In production: reads pre-computed trending list from Redis (latency: <5ms)
            System.out.printf("[Trending] Serving trending items for user %s (fast path)%n", userId);
            return Arrays.asList("PROD-999", "PROD-998", "PROD-997").subList(0, Math.min(maxResults, 3));
        }

        @Override
        public String getAlgorithmName() { return "Trending Items"; }

        @Override
        public boolean isHeavyweight() { return false; }
    }

    // ─── Context: AdaptiveRecommendationEngine ────────────────────────────

    // AtomicReference makes strategy swaps thread-safe without locking
    private final AtomicReference<RecommendationStrategy> currentStrategy;
    private final RecommendationStrategy premiumStrategy;
    private final RecommendationStrategy standardStrategy;
    private final RecommendationStrategy fallbackStrategy;

    // Simulated load metrics — in production: from a metrics registry
    private volatile double currentCpuLoad = 0.0;

    public AdaptiveRecommendationEngine() {
        this.premiumStrategy = new CollaborativeFilteringStrategy("3.1");
        this.standardStrategy = new ContentBasedStrategy();
        this.fallbackStrategy = new TrendingItemsStrategy();
        this.currentStrategy = new AtomicReference<>(premiumStrategy);
    }

    /**
     * Called by a background health-check thread (e.g., every 10 seconds).
     * Automatically switches strategy based on system conditions.
     */
    public void adaptStrategyToLoad(double cpuLoad) {
        this.currentCpuLoad = cpuLoad;
        RecommendationStrategy chosen;

        if (cpuLoad < 0.60) {
            chosen = premiumStrategy;    // Normal: use best algorithm
        } else if (cpuLoad < 0.85) {
            chosen = standardStrategy;  // Degraded: use lighter algorithm
        } else {
            chosen = fallbackStrategy;  // High load: serve cached trending items
        }

        RecommendationStrategy previous = currentStrategy.getAndSet(chosen);
        if (!previous.getAlgorithmName().equals(chosen.getAlgorithmName())) {
            System.out.printf("[Engine] CPU load %.0f%% — switching from '%s' to '%s'%n",
                cpuLoad * 100, previous.getAlgorithmName(), chosen.getAlgorithmName());
        }
    }

    /**
     * External override — admin can force a specific strategy via feature flag.
     */
    public void forceStrategy(RecommendationStrategy strategy) {
        currentStrategy.set(Objects.requireNonNull(strategy));
        System.out.println("[Engine] Strategy forced to: " + strategy.getAlgorithmName());
    }

    public List<String> getRecommendations(String userId, int maxResults) {
        RecommendationStrategy strategy = currentStrategy.get();
        System.out.printf("[Engine] Serving user %s with: %s%n",
            userId, strategy.getAlgorithmName());
        return strategy.recommend(userId, maxResults);
    }

    // ─── Demo ─────────────────────────────────────────────────────────────

    public static void main(String[] args) {
        AdaptiveRecommendationEngine engine = new AdaptiveRecommendationEngine();

        System.out.println("=== Normal Load (30% CPU) ===");
        engine.adaptStrategyToLoad(0.30);
        System.out.println(engine.getRecommendations("user-42", 3));

        System.out.println("\n=== Elevated Load (72% CPU) ===");
        engine.adaptStrategyToLoad(0.72);
        System.out.println(engine.getRecommendations("user-42", 3));

        System.out.println("\n=== High Load (91% CPU) ===");
        engine.adaptStrategyToLoad(0.91);
        System.out.println(engine.getRecommendations("user-42", 3));

        System.out.println("\n=== Admin Forces Premium Strategy ===");
        engine.forceStrategy(new CollaborativeFilteringStrategy("3.2-beta"));
        System.out.println(engine.getRecommendations("user-42", 3));
    }
}
```

---

## 7. Real-World Framework and Library Examples

### 7.1 `java.util.Comparator` — Strategy in the JDK

As shown above, `Comparator<T>` is the most pervasive Strategy interface in Java. The JDK team understood this pattern so deeply that Java 8 added `Comparator.comparing()`, `thenComparing()`, and other factory methods for composing strategies:

```java
// Composing strategies via the Comparator API
Comparator<Employee> complexSort = Comparator
    .comparing(Employee::getDepartment)            // primary sort
    .thenComparingDouble(Employee::getSalary)      // secondary sort
    .reversed()                                    // invert the whole thing
    .thenComparing(Employee::getName);             // tertiary fallback

employees.sort(complexSort);
```

`Comparator.thenComparing()` is a composite/chain strategy — it delegates to the next strategy only when the first returns zero. This is Strategy + Chain of Responsibility in one method.

### 7.2 Spring Framework's `ResourceLoader`

Spring's `ResourceLoader` interface is a Strategy for how resources (files, classpath resources, URLs) are loaded:

```java
// org.springframework.core.io.ResourceLoader — the Strategy interface
public interface ResourceLoader {
    Resource getResource(String location);
    ClassLoader getClassLoader();
}

// Concrete strategies (Spring provides these; you can implement your own):
// ClassPathXmlApplicationContext    — loads from classpath
// FileSystemXmlApplicationContext   — loads from filesystem
// XmlWebApplicationContext          — loads from web context

// At the call site, code depends on ResourceLoader, not any concrete loader:
@Component
public class ConfigImporter {
    private final ResourceLoader resourceLoader;

    public ConfigImporter(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;  // injected by Spring — could be any impl
    }

    public Properties load(String location) throws IOException {
        Resource resource = resourceLoader.getResource(location);
        Properties props = new Properties();
        try (InputStream is = resource.getInputStream()) {
            props.load(is);
        }
        return props;
    }
}
```

### 7.3 Java's `ExecutorService` and Thread Pools

`ThreadPoolExecutor` accepts a `RejectedExecutionHandler` — a Strategy for what to do when the task queue is full:

```java
// Four built-in strategies, all implementing RejectedExecutionHandler:
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4, 8, 60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),
    new ThreadPoolExecutor.CallerRunsPolicy() // Strategy: run in caller's thread
    // Alternatives:
    // new ThreadPoolExecutor.AbortPolicy()       — throw RejectedExecutionException
    // new ThreadPoolExecutor.DiscardPolicy()     — silently drop task
    // new ThreadPoolExecutor.DiscardOldestPolicy()— drop oldest queued task
);

// Custom strategy for logging before discarding:
RejectedExecutionHandler loggingRejectHandler = (runnable, pool) -> {
    System.err.println("[WARN] Task rejected: " + runnable + ". Pool: " + pool);
    // optionally: send to dead-letter queue
};
executor.setRejectedExecutionHandler(loggingRejectHandler);
```

### 7.4 Spring Security's `PasswordEncoder`

```java
// The Strategy interface
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
}

// Concrete strategies:
// BCryptPasswordEncoder — slow bcrypt (recommended for passwords)
// Pbkdf2PasswordEncoder — PBKDF2
// SCryptPasswordEncoder — memory-hard scrypt
// NoOpPasswordEncoder   — plaintext (only for testing)
// DelegatingPasswordEncoder — wraps multiple strategies with version prefix

@Configuration
public class SecurityConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        // Switch encoding algorithm without changing any service code
        return new BCryptPasswordEncoder(12);
    }
}
```

### 7.5 Jackson's `JsonSerializer` / `JsonDeserializer`

```java
// Strategy interface (simplified)
public abstract class JsonSerializer<T> {
    public abstract void serialize(T value, JsonGenerator gen,
                                   SerializerProvider serializers) throws IOException;
}

// Custom strategy for serializing Money objects
public class MoneySerializer extends JsonSerializer<BigDecimal> {
    @Override
    public void serialize(BigDecimal value, JsonGenerator gen,
                          SerializerProvider serializers) throws IOException {
        // Always write as a string with 2 decimal places to prevent JS float issues
        gen.writeString(value.setScale(2, RoundingMode.HALF_UP).toPlainString());
    }
}

// Register the strategy
@JsonSerialize(using = MoneySerializer.class)
private BigDecimal price;
```

---

## 8. Trade-offs

### Advantages

| Advantage | Detail |
|-----------|--------|
| **Open/Closed Principle** | New algorithms are added as new classes. No existing code changes. The context class never needs modification. |
| **Single Responsibility** | Each strategy class has exactly one reason to change: the algorithm it encapsulates changes. |
| **Eliminates conditionals** | Replaces complex `if/else` or `switch` chains that dispatch on type/mode with polymorphic dispatch. |
| **Independent testability** | Each strategy can be unit-tested in complete isolation. No need to set up a full Order or PaymentProcessor. |
| **Runtime flexibility** | The algorithm in use can change at any point during the object's lifecycle without reconstructing the context. |
| **Composition over inheritance** | The context acquires behavior through a held reference, not through class hierarchy. This avoids the fragile base class problem. |
| **Supports feature flags** | A/B testing different algorithms is trivial: inject strategyA or strategyB based on a flag. |

### Disadvantages

| Disadvantage | Detail |
|--------------|--------|
| **Increased number of classes** | Each algorithm becomes a class. For N algorithms, you add N classes. In large systems this can feel like class proliferation, though each class is small and focused. |
| **Client awareness** | Clients must be aware that multiple strategies exist and choose the appropriate one, unless a factory or DI container hides this decision. |
| **Overhead for simple cases** | For one or two non-varying algorithms, Strategy adds indirection without benefit. A simple private method or even a lambda suffices. |
| **Strategy/Context coupling** | The strategy must have access to the data it needs. If the strategy needs many fields from the context, you either pass the whole context object (coupling) or bloat the strategy's method signature. |
| **Communication overhead** | When strategy and context share no common data structure, strategies may receive data they don't need, or the context must transform its data for each strategy. |
| **Stateless requirement** | Strategies that are used as singletons must be stateless (or thread-safe if stateful). Forgetting this causes subtle concurrency bugs. |

---

## 9. Common Interview Discussion Points

### The Null Object + Strategy Combination
Rather than setting `discountStrategy = null` when no discount applies and writing `if (discountStrategy != null)` at every call site, always default to a `NoDiscountStrategy`. This is the **Null Object pattern** composing with Strategy — it removes null checks and ensures the context always has a valid strategy.

### Strategy vs. Lambda (Java 8+)
When a Strategy interface has exactly one abstract method (a `@FunctionalInterface`), concrete strategy classes can be replaced with lambdas:
```java
// Old way: full class
order.setDiscountStrategy(new PercentageDiscountStrategy(15.0, "Premium"));

// With lambda (if DiscountStrategy were a functional interface):
order.setDiscountStrategy(ctx -> new DiscountResult(ctx.getOriginalAmount(),
    ctx.getOriginalAmount() * 0.15, "Premium 15%"));
```
Lambdas are appropriate when the strategy is short-lived, stateless, and defined at the call site. Named classes are better when the strategy has configuration fields, needs DI, or will be reused across multiple places. For interview: "I use lambdas for simple, inline strategies and named classes for complex, configurable, reusable ones."

### Strategy Factory vs. Enum-Dispatch vs. Map-Dispatch
Three common approaches to selecting a strategy at runtime:
```java
// 1. Factory method (most explicit)
DiscountStrategy strategy = factory.resolve(customer);

// 2. Enum-keyed map (fast O(1) lookup, no if/else)
Map<CustomerType, DiscountStrategy> strategyMap = new EnumMap<>(CustomerType.class);
strategyMap.put(CustomerType.PREMIUM, new PercentageDiscountStrategy(15.0, "Premium"));
DiscountStrategy strategy = strategyMap.getOrDefault(customer.getType(), NoDiscountStrategy.INSTANCE);

// 3. Chain of responsibility (tries each strategy until one accepts)
List<DiscountStrategy> chain = List.of(loyaltyStrategy, seasonalStrategy, noDiscount);
DiscountResult result = chain.stream()
    .map(s -> s.apply(context))
    .filter(r -> r.getDiscountAmount() > 0)
    .findFirst()
    .orElse(new DiscountResult(amount, 0, "No discount"));
```

### Strategy in Spring Boot (Dependency Injection)
In production Spring Boot applications, strategies are typically Spring beans injected via constructor. A common pattern:
```java
@Service
public class DiscountService {
    private final Map<String, DiscountStrategy> strategyRegistry;

    // Spring collects all DiscountStrategy beans into the map,
    // keyed by bean name. Zero configuration needed.
    public DiscountService(Map<String, DiscountStrategy> strategyRegistry) {
        this.strategyRegistry = strategyRegistry;
    }

    public DiscountResult apply(String strategyName, OrderContext ctx) {
        DiscountStrategy strategy = strategyRegistry.getOrDefault(
            strategyName, NoDiscountStrategy.INSTANCE);
        return strategy.apply(ctx);
    }
}
```

### Thread Safety Considerations
Strategies that are stateless singletons are inherently thread-safe. Strategies that hold configuration (like percentage values set at construction) are also thread-safe if those fields are `final`. Strategies that mutate state (e.g., accumulating statistics) must use synchronization or be scoped per-request.

---

## 10. Interview Questions and Model Answers

---

### Q1: What is the Strategy pattern and what problem does it solve? Give a real-world example.

**Model Answer:**

The Strategy pattern defines a family of algorithms, encapsulates each one in its own class behind a common interface, and makes them interchangeable at runtime. The core problem it solves is the proliferation of conditional logic that dispatches between algorithm variants — what Martin Fowler calls the "replace conditional with polymorphism" refactoring.

**The problem without Strategy:** Imagine an e-commerce checkout that started with simple percentage discounts. As the business grew, the discount logic expanded into a massive conditional tree: premium members get 15% off, loyalty platinum members get 20%, seasonal Black Friday gets 40%, first-time customers get a $20 flat off. All of this lives in one method in the `Order` class. Every new discount type requires modifying `Order`, which risks breaking existing discounts, makes the class harder to test, and causes merge conflicts when multiple teams add discounts simultaneously.

**The solution with Strategy:** Extract each discount algorithm into its own class behind a `DiscountStrategy` interface. `Order` holds a reference to the interface and delegates calculation to it. Adding a new discount type means adding a new class — `Order` never changes. Each strategy is independently testable. The strategy can be swapped at runtime when a customer applies a coupon code mid-checkout.

**Real-world examples:** `java.util.Comparator` (sorting strategy), Spring's `PasswordEncoder` (hashing strategy), `ThreadPoolExecutor.RejectedExecutionHandler` (task rejection strategy), Jackson's `JsonSerializer` (serialization strategy).

---

### Q2: How does Strategy differ from the Template Method pattern? When would you choose one over the other?

**Model Answer:**

Both patterns address the same problem: varying algorithm steps while keeping the overall structure stable. The fundamental difference is the mechanism:

- **Template Method** uses **inheritance**. A base class defines the algorithm skeleton in a `final` or concrete method, with abstract or overridable hook methods for the varying steps. Subclasses override the hooks.
- **Strategy** uses **composition**. The context holds a reference to a strategy interface. The entire algorithm (or the varying part of it) lives in the strategy object.

```java
// Template Method: inheritance
public abstract class DataExporter {
    // Template method — fixed skeleton
    public final void export(Dataset data) {
        data = preprocess(data);           // fixed step
        String formatted = format(data);   // abstract — subclass defines
        write(formatted);                  // fixed step
    }
    protected abstract String format(Dataset data); // hook
    // ...
}

public class CsvExporter extends DataExporter {
    @Override
    protected String format(Dataset data) { /* CSV logic */ return "..."; }
}

// Strategy: composition
public interface FormatStrategy {
    String format(Dataset data);
}

public class DataExporter {
    private FormatStrategy formatStrategy; // holds the variable part

    public void export(Dataset data) {
        data = preprocess(data);
        String formatted = formatStrategy.format(data);
        write(formatted);
    }
}
```

**When to choose Template Method:**
- The invariant skeleton and the variable hooks are deeply intertwined (the skeleton calls the hooks multiple times, or passes internal state to them).
- Algorithm variations are known at compile time and each represents a distinct type of exporter.
- You want to leverage protected visibility to prevent clients from calling hook methods directly.
- Fewer algorithm variants that are unlikely to change at runtime.

**When to choose Strategy:**
- You need runtime algorithm switching.
- Algorithms should be independently composable (you might combine a `PercentageDiscount` with a `SeasonalDiscount`).
- You want to avoid deep inheritance hierarchies (diamond problem, fragile base class).
- Algorithms should be independently testable without constructing a concrete subclass.
- The varying behavior should be swappable via dependency injection.

**Design principle:** GoF and modern OOP design prefer composition over inheritance. Strategy is often the more flexible choice because it avoids the tight coupling of inheritance. A common refactoring path is to start with Template Method when you control the hierarchy, then migrate to Strategy when you need runtime switching or DI.

---

### Q3: How do you handle the case where different strategies need different data from the context?

**Model Answer:**

This is a genuinely tricky design challenge. There are four main approaches, each with distinct trade-offs:

**Approach 1: Rich Context Value Object (Recommended)**
Pass an `OrderContext` (or similar value object) that contains all fields any strategy might need. Strategies pick what they need and ignore the rest.

```java
// OrderContext is a comprehensive snapshot
public final class OrderContext {
    private final double amount;
    private final Customer customer;
    private final LocalDate orderDate;
    private final String couponCode;
    private final boolean isFirstOrder;
    // ... any field a strategy might ever need
}
```

*Pros:* Strategy interface signature never changes as new strategies are added. Immutable, thread-safe. Easy to construct in tests.
*Cons:* OrderContext can become a "god object" over time. Strategies receive data they don't use.

**Approach 2: Visitor/Double Dispatch**
The context accepts a strategy and passes itself to the strategy, allowing the strategy to call back for what it needs:

```java
public interface DiscountStrategy {
    DiscountResult apply(Order order); // order exposes query methods
}
```

*Pros:* Strategy gets exactly what it needs via public query methods. No intermediate DTO.
*Cons:* Couples strategy to the full context class — changing Order's API breaks strategies.

**Approach 3: Strategy-Specific Builders**
Each strategy has its own data object, and a factory builds the right one before calling the strategy:

```java
public interface DiscountStrategy<C> {
    DiscountResult apply(C context);
}
// LoyaltyDiscountStrategy implements DiscountStrategy<LoyaltyContext>
// SeasonalDiscountStrategy implements DiscountStrategy<SeasonalContext>
```

*Pros:* Perfect type safety. No unnecessary data in context.
*Cons:* Destroys the interchangeability that makes Strategy valuable. You can't hold them behind a single interface.

**Approach 4: Strategy Receives Only What It Needs via Interface Methods**
Keep `OrderContext` focused but add dedicated query methods:

```java
public interface DiscountStrategy {
    DiscountResult apply(DiscountContext ctx);
}

public interface DiscountContext {
    double getOrderAmount();
    LoyaltyTier getCustomerLoyaltyTier();
    boolean isCustomerPremium();
    LocalDate getOrderDate();
    Optional<String> getCouponCode();
}
```

**My recommendation:** Use Approach 1 (rich value object) for most systems, with discipline to keep the context to genuinely relevant fields. If strategies need drastically different data, consider whether they actually belong to the same family of algorithms — if they don't, they may not be appropriate candidates for the same Strategy interface.

---

### Q4: Can Strategy patterns be chained or composed? How would you implement a discount system that applies multiple discounts?

**Model Answer:**

Yes, strategies can be composed using two additional patterns: **Decorator** and **Composite**. Both are standard approaches to applying multiple discount strategies.

**Option A: Composite Strategy (apply all, return best)**
```java
public class BestOfDiscountStrategy implements DiscountStrategy {
    private final List<DiscountStrategy> candidates;

    public BestOfDiscountStrategy(DiscountStrategy... strategies) {
        this.candidates = Arrays.asList(strategies);
    }

    @Override
    public DiscountResult apply(OrderContext context) {
        return candidates.stream()
            .map(s -> s.apply(context))
            .max(Comparator.comparingDouble(DiscountResult::getDiscountAmount))
            .orElse(new DiscountResult(context.getOriginalAmount(), 0, "No discount"));
    }

    @Override
    public String getDescription() { return "Best of " + candidates.size() + " discounts"; }
}
```

**Option B: Chained/Sequential Strategy (apply all, sum up)**
```java
public class ChainedDiscountStrategy implements DiscountStrategy {
    private final List<DiscountStrategy> chain;
    private final double maxDiscountPercentage; // cap to prevent over-discounting

    public ChainedDiscountStrategy(double maxDiscountPercentage, DiscountStrategy... strategies) {
        this.chain = Arrays.asList(strategies);
        this.maxDiscountPercentage = maxDiscountPercentage;
    }

    @Override
    public DiscountResult apply(OrderContext context) {
        double originalAmount = context.getOriginalAmount();
        double totalDiscount = 0.0;
        List<String> descriptions = new ArrayList<>();

        for (DiscountStrategy strategy : chain) {
            // Each strategy applies to the original amount, not the reduced one
            // (non-stacking discounts)
            DiscountResult result = strategy.apply(context);
            totalDiscount += result.getDiscountAmount();
            if (result.getDiscountAmount() > 0) {
                descriptions.add(result.getAppliedDescription());
            }
        }

        // Cap total discount
        double maxDiscount = originalAmount * (maxDiscountPercentage / 100.0);
        totalDiscount = Math.min(totalDiscount, maxDiscount);

        return new DiscountResult(
            originalAmount,
            Math.round(totalDiscount * 100.0) / 100.0,
            String.join(" + ", descriptions)
        );
    }

    @Override
    public String getDescription() { return "Chained Discount"; }
}

// Usage:
order.setDiscountStrategy(new ChainedDiscountStrategy(
    35.0, // max 35% total discount
    new LoyaltyDiscountStrategy(),
    new SeasonalDiscountStrategy(),
    new FlatDiscountStrategy(10.0, 50.0, "Welcome Bonus")
));
```

**Key design decision:** Decide whether discounts **stack additively** (each applies to original price), **stack multiplicatively** (each applies to already-discounted price), or whether **only the best applies**. This is a business rule, not a technical one — make it explicit in the strategy class name and documentation.

---

### Q5: How would you implement the Strategy pattern in a Spring Boot application using dependency injection?

**Model Answer:**

Spring Boot's DI container makes Strategy even more powerful: you can inject all strategy beans automatically without explicit configuration.

```java
// 1. Mark each strategy as a Spring bean
@Component("percentageDiscount")
public class PercentageDiscountStrategy implements DiscountStrategy { /* ... */ }

@Component("flatDiscount")
public class FlatDiscountStrategy implements DiscountStrategy { /* ... */ }

@Component("loyaltyDiscount")
public class LoyaltyDiscountStrategy implements DiscountStrategy { /* ... */ }

// 2. Spring auto-discovers all DiscountStrategy beans and injects them as a Map.
//    The map key is the bean name.
@Service
public class DiscountService {
    private final Map<String, DiscountStrategy> strategies;

    // Spring injects all beans implementing DiscountStrategy
    public DiscountService(Map<String, DiscountStrategy> strategies) {
        this.strategies = strategies;
    }

    public DiscountResult apply(String strategyBeanName, OrderContext context) {
        DiscountStrategy strategy = strategies.get(strategyBeanName);
        if (strategy == null) {
            throw new IllegalArgumentException(
                "No discount strategy found for: " + strategyBeanName
                + ". Available: " + strategies.keySet());
        }
        return strategy.apply(context);
    }
}

// 3. Use a qualifier-based approach for type-safe injection at specific sites:
@Service
public class CheckoutService {
    private final DiscountStrategy premiumStrategy;
    private final DiscountStrategy defaultStrategy;

    public CheckoutService(
            @Qualifier("percentageDiscount") DiscountStrategy premiumStrategy,
            @Qualifier("noDiscount") DiscountStrategy defaultStrategy) {
        this.premiumStrategy = premiumStrategy;
        this.defaultStrategy = defaultStrategy;
    }
}

// 4. For prototype-scoped strategies (new instance per use):
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class SeasonalDiscountStrategy implements DiscountStrategy {
    // Spring creates a new instance each time it's requested
}
```

**Production considerations:**
- Strategies injected as Spring singletons must be **stateless or thread-safe**.
- Use `@Primary` to designate a default strategy when multiple candidates qualify.
- Combine with `@ConditionalOnProperty` to enable/disable strategies based on feature flags.
- When strategy selection depends on runtime data (customer type, request parameters), use a `StrategyResolver` service rather than injecting strategies directly into business code.

---

### Q6: What is the difference between Strategy and State patterns? They look structurally identical — how do you distinguish them?

**Model Answer:**

Structurally, Strategy and State are nearly identical: both use an interface held by a context, with concrete implementations providing behavior. The difference is **intent, ownership of transition logic, and awareness of siblings**:

| Dimension | Strategy | State |
|-----------|----------|-------|
| **Who changes the algorithm?** | The **client** sets the strategy explicitly | The **context or concrete state** transitions automatically |
| **Awareness of siblings** | Strategies are independent — `PercentageDiscount` has no knowledge that `FlatDiscount` exists | States know about each other — `LockedState` transitions to `UnlockedState` on correct PIN |
| **How many switches happen?** | Infrequent — chosen at setup or on specific user action | Frequent — transitions happen as part of normal processing |
| **Intent** | Select between interchangeable **algorithms** | Model a **lifecycle or workflow** with distinct behavioral phases |

```java
// STRATEGY: Client decides which algorithm to use
order.setDiscountStrategy(new PremiumDiscountStrategy()); // client drives this
price = order.calculateFinalPrice(); // strategy executes, doesn't change itself

// STATE: State drives its own transitions
vendingMachine.insertCoin();  // internally transitions: IdleState -> HasMoneyState
vendingMachine.selectItem();  // internally transitions: HasMoneyState -> DispensingState
vendingMachine.dispense();    // internally transitions: DispensingState -> IdleState
// The client doesn't call machine.setState(new IdleState()) — the state does
```

**Practical test to tell them apart:**
1. Can a strategy object transition itself to a different strategy? If yes, it's probably State.
2. Is the client explicitly choosing which algorithm to use, or is the object reacting to its own internal events? Client choice = Strategy. Self-transition = State.
3. Does the description use "algorithm" or "behavior variation"? Strategy. Does it use "phase", "lifecycle", "workflow", or "mode"? State.

**Same code structure, different stories:** A traffic light system where `RedLight`, `GreenLight`, `YellowLight` all implement `LightBehavior`, and each state transitions itself to the next after a timer fires — that's State. A report formatter where `PdfFormat`, `CsvFormat`, `ExcelFormat` implement `ReportFormat`, and the user selects the format — that's Strategy.

---

### Q7: How do you ensure thread safety when using the Strategy pattern in a concurrent system?

**Model Answer:**

Thread safety in Strategy has three dimensions: the strategy itself, the context's strategy reference, and the context's other mutable state.

**1. Make strategies stateless or immutable**

The safest strategy is one that holds only `final` configuration fields set at construction:

```java
// Thread-safe: all fields are final
public class PercentageDiscountStrategy implements DiscountStrategy {
    private final double percentage; // final — safe
    private final String label;      // final — safe

    // No shared mutable state — this class is inherently thread-safe
}
```

If a strategy must maintain state (e.g., caching results), use thread-safe data structures:

```java
public class CachingLoyaltyDiscountStrategy implements DiscountStrategy {
    // ConcurrentHashMap is thread-safe
    private final ConcurrentMap<String, Double> tierCache = new ConcurrentHashMap<>();

    @Override
    public DiscountResult apply(OrderContext context) {
        String customerId = context.getCustomer().getCustomerId().toString();
        double percentage = tierCache.computeIfAbsent(customerId, this::loadFromDb);
        // ...
    }
}
```

**2. Make strategy swapping on the context thread-safe**

If the strategy reference on the context is mutable and shared across threads, use `volatile` or `AtomicReference`:

```java
public class Order {
    // volatile ensures visibility — one thread's write is immediately visible to others
    private volatile DiscountStrategy discountStrategy = NoDiscountStrategy.INSTANCE;

    // AtomicReference for compare-and-swap operations:
    private final AtomicReference<DiscountStrategy> strategyRef =
        new AtomicReference<>(NoDiscountStrategy.INSTANCE);

    public void setDiscountStrategy(DiscountStrategy strategy) {
        strategyRef.set(Objects.requireNonNull(strategy));
    }

    public PriceSummary calculateFinalPrice() {
        // Get reference once — atomic snapshot prevents TOCTOU race
        DiscountStrategy strategy = strategyRef.get();
        return strategy.apply(buildContext());
    }
}
```

**3. Be careful with prototype strategies**

If a strategy is prototype-scoped (new instance per request), it can safely be stateful because it's never shared. But make sure the factory that creates it is thread-safe.

**4. Spring singletons**

Spring singleton beans are shared across all threads by default. Any `DiscountStrategy` bean in a Spring context must be stateless or use `@Scope("prototype")` if it needs per-request state.

**Interview summary:** "Stateless strategies are always thread-safe. For mutable strategy state, use `ConcurrentHashMap` or `AtomicReference`. For the strategy reference on the context, use `volatile` or `AtomicReference` if the context is shared across threads. In Spring, always make singleton-scoped strategy beans stateless."

---

### Q8: How would you test a Strategy pattern implementation effectively?

**Model Answer:**

Strategy dramatically improves testability because each algorithm is independently testable in isolation. Here's a complete testing approach:

**1. Unit test each concrete strategy in complete isolation**

```java
class PercentageDiscountStrategyTest {

    private PercentageDiscountStrategy strategy;
    private Customer customer;

    @BeforeEach
    void setUp() {
        strategy = new PercentageDiscountStrategy(20.0, "Test Discount");
        customer = new Customer(UUID.randomUUID(), "Test User", "test@example.com",
                                500.0, false);
    }

    @Test
    void apply_shouldCalculate20PercentDiscount() {
        OrderContext context = OrderContext.builder()
            .originalAmount(100.0)
            .customer(customer)
            .build();

        DiscountResult result = strategy.apply(context);

        assertThat(result.getDiscountAmount()).isEqualTo(20.0);
        assertThat(result.getFinalAmount()).isEqualTo(80.0);
    }

    @Test
    void apply_shouldRoundDiscountTo2DecimalPlaces() {
        OrderContext context = OrderContext.builder()
            .originalAmount(99.99)
            .customer(customer)
            .build();

        DiscountResult result = strategy.apply(context);

        // 99.99 * 0.20 = 19.998 → should round to 20.00
        assertThat(result.getDiscountAmount()).isEqualTo(20.00);
    }

    @ParameterizedTest
    @ValueSource(doubles = {0.0, 50.0, 100.0, 199.99, 1000.0})
    void apply_shouldAlwaysProduceNonNegativeFinalAmount(double amount) {
        OrderContext context = OrderContext.builder()
            .originalAmount(amount)
            .customer(customer)
            .build();

        DiscountResult result = strategy.apply(context);

        assertThat(result.getFinalAmount()).isGreaterThanOrEqualTo(0.0);
        assertThat(result.getDiscountAmount()).isLessThanOrEqualTo(amount);
    }

    @Test
    void constructor_shouldRejectPercentageOutOfRange() {
        assertThatThrownBy(() -> new PercentageDiscountStrategy(-5.0, "Invalid"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Percentage must be between");
    }
}
```

**2. Test the context delegates correctly to the strategy (use a test double)**

```java
class OrderTest {

    @Test
    void calculateFinalPrice_shouldDelegateToCurrentStrategy() {
        // Arrange
        Customer customer = new Customer(UUID.randomUUID(), "Alice", "alice@test.com", 0, false);
        Order order = new Order(customer);
        order.addItem(new OrderItem("SKU-001", "Widget", 100.0, 1));

        // Inject a controlled test strategy
        DiscountStrategy mockStrategy = mock(DiscountStrategy.class);
        when(mockStrategy.apply(any(OrderContext.class)))
            .thenReturn(new DiscountResult(100.0, 15.0, "Mock 15%"));
        when(mockStrategy.getDescription()).thenReturn("Mock Strategy");

        order.setDiscountStrategy(mockStrategy);

        // Act
        PriceSummary summary = order.calculateFinalPrice();

        // Assert
        assertThat(summary.getTotalDiscount()).isEqualTo(15.0);
        verify(mockStrategy, times(1)).apply(any(OrderContext.class));
    }

    @Test
    void setDiscountStrategy_shouldRejectNull() {
        Customer customer = new Customer(UUID.randomUUID(), "Alice", "alice@test.com", 0, false);
        Order order = new Order(customer);

        assertThatThrownBy(() -> order.setDiscountStrategy(null))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void calculateFinalPrice_withDefaultStrategy_shouldApplyNoDiscount() {
        Customer customer = new Customer(UUID.randomUUID(), "Alice", "alice@test.com", 0, false);
        Order order = new Order(customer);
        order.addItem(new OrderItem("SKU-001", "Widget", 100.0, 1));

        // No strategy set — should use NoDiscountStrategy (Null Object default)
        PriceSummary summary = order.calculateFinalPrice();

        assertThat(summary.getTotalDiscount()).isEqualTo(0.0);
    }
}
```

**3. Integration test the factory**

```java
class DiscountStrategyFactoryTest {

    private DiscountStrategyFactory factory = new DiscountStrategyFactory();

    @Test
    void resolve_shouldReturnPercentageDiscount_forPremiumMember() {
        Customer premium = new Customer(UUID.randomUUID(), "Bob", "bob@test.com", 500.0, true);
        DiscountStrategy strategy = factory.resolve(premium);
        assertThat(strategy).isInstanceOf(PercentageDiscountStrategy.class);
    }

    @Test
    void resolve_shouldReturnLoyaltyDiscount_forNonPremiumWithTier() {
        Customer loyal = new Customer(UUID.randomUUID(), "Carol", "carol@test.com", 5000.0, false);
        // $5000 lifetime spend → Gold tier
        DiscountStrategy strategy = factory.resolve(loyal);
        assertThat(strategy).isInstanceOf(LoyaltyDiscountStrategy.class);
    }
}
```

**Testing principles for Strategy:**
- Each strategy gets its own test class. Never test strategy logic inside Order tests.
- Use parameterized tests to cover edge cases (zero amount, maximum amount, boundary percentages).
- Mock the strategy when testing the context — you're testing delegation, not calculation.
- Test that illegal construction arguments are rejected (negative percentages, etc.).
- Test the Null Object (`NoDiscountStrategy`) explicitly — it should never throw exceptions.

---

## 11. Comparison with Related Patterns

### Strategy vs. Template Method (Deep Dive)

This is the most important comparison and the one most frequently asked about in interviews.

| Dimension | Strategy (Composition) | Template Method (Inheritance) |
|-----------|------------------------|-------------------------------|
| **Mechanism** | Holds a reference to an interface | Subclass overrides abstract methods |
| **Algorithm scope** | The **entire algorithm** (or key part) is swapped | **Steps within a fixed skeleton** are customized |
| **Runtime switching** | Yes — call `setStrategy()` anytime | No — class is fixed at compile/instantiation time |
| **Relationship** | "Has-a" (context has a strategy) | "Is-a" (subclass is a specialization) |
| **Adding new variant** | Add a new class | Add a new subclass |
| **Shared code** | Must be duplicated or extracted to utility | Lives in the base class — naturally shared |
| **Testing** | Strategy tested independently | Must instantiate concrete subclass |
| **OCP** | Fully open for extension, closed for modification | Open for extension but base class may need modification for new hooks |

```java
// Template Method: Skeleton is fixed, hooks vary
public abstract class ReportGenerator {
    // Skeleton — invariant
    public final Report generate(Dataset data) {
        DataSet filtered = filterData(data);      // hook 1
        List<Row> rows = transformRows(filtered); // hook 2
        return buildReport(rows);                 // hook 3 (optional: has default)
    }

    protected abstract DataSet filterData(DataSet data);
    protected abstract List<Row> transformRows(DataSet data);

    // Default implementation — subclasses may override
    protected Report buildReport(List<Row> rows) {
        return new Report(rows);
    }
}

// Strategy: Entire formatting algorithm is swapped
public class ReportGenerator {
    private final DataFilterStrategy filter;      // strategy 1
    private final RowTransformStrategy transform; // strategy 2
    private final ReportBuilderStrategy builder;  // strategy 3

    // Runtime composition: inject different strategies
    public ReportGenerator(DataFilterStrategy filter,
                           RowTransformStrategy transform,
                           ReportBuilderStrategy builder) {
        this.filter = filter;
        this.transform = transform;
        this.builder = builder;
    }
    // ...
}
```

**Rule of thumb:** Use Template Method when you own the hierarchy and variations are conceptually "types of" the base. Use Strategy when you need runtime flexibility, DI, independent testability, or multiple independently varying dimensions.

### Strategy vs. Command

| Dimension | Strategy | Command |
|-----------|----------|---------|
| **Purpose** | Select one algorithm from a family | Encapsulate a request as an object |
| **Execution timing** | Executed immediately when context uses it | May be stored, queued, undone, or replayed |
| **Undo support** | No | Yes (Command can hold inverse operation) |
| **Parameters** | Algorithm receives data via method args | Command carries its own parameters |
| **Typical use** | Sorting, discounting, formatting | Undo/redo, task queues, macros, transactions |

### Strategy vs. Decorator

| Dimension | Strategy | Decorator |
|-----------|----------|-----------|
| **Purpose** | Replace the algorithm entirely | Add responsibility to an object |
| **Relationship** | Context delegates to one strategy | Decorator wraps and delegates to the same interface |
| **Layering** | One strategy active at a time | Multiple decorators stack infinitely |
| **Transparency** | Context knows it's calling a strategy | Decorated object doesn't know it's wrapped |
| **Use case** | Swap payment method | Add logging, caching, retry to existing behavior |

```java
// Decorator applied to Strategy: wrap a strategy to add logging
public class LoggingDiscountStrategy implements DiscountStrategy {
    private final DiscountStrategy delegate;

    public LoggingDiscountStrategy(DiscountStrategy delegate) {
        this.delegate = delegate;
    }

    @Override
    public DiscountResult apply(OrderContext context) {
        System.out.printf("[Discount] Applying %s to order of $%.2f%n",
            delegate.getDescription(), context.getOriginalAmount());
        DiscountResult result = delegate.apply(context);
        System.out.printf("[Discount] Applied $%.2f discount%n", result.getDiscountAmount());
        return result;
    }

    @Override
    public String getDescription() { return "Logged[" + delegate.getDescription() + "]"; }
}

// Usage: Strategy + Decorator
order.setDiscountStrategy(
    new LoggingDiscountStrategy(
        new PercentageDiscountStrategy(15.0, "Premium")
    )
);
```

### Strategy vs. State

Already covered in Q6 above. Summary: Strategy is about algorithm selection (client controls). State is about lifecycle transitions (state controls itself).

### Pattern Interaction Summary

```
Strategy ──────► solves "which algorithm?" (client-controlled selection)
Template Method ► solves "how is the skeleton structured?" (inheritance-controlled steps)
State ──────────► solves "what phase am I in?" (self-controlled transitions)
Command ────────► solves "what request to record?" (encapsulates request as object)
Decorator ──────► solves "what to add to existing behavior?" (transparent wrapping)

Strategy + Null Object ──► eliminates null checks on strategy reference
Strategy + Factory ──────► encapsulates strategy selection logic
Strategy + Decorator ────► adds cross-cutting concerns (logging, caching) to strategies
Strategy + Composite ────► applies multiple strategies as if they were one
```

---

*End of Strategy Design Pattern*

*Next: [02 - Observer Pattern](02-observer.md)*
