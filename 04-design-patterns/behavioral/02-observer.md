# Observer Pattern

## Table of Contents

1. [Intent and Problem It Solves](#1-intent-and-problem-it-solves)
2. [When to Use / When NOT to Use](#2-when-to-use--when-not-to-use)
3. [UML Class Diagrams](#3-uml-class-diagrams)
4. [Push vs Pull Model](#4-push-vs-pull-model)
5. [Java Observable (Deprecated) and Why](#5-java-observable-deprecated-and-why)
6. [Implementation 1: Stock Price Notification System](#6-implementation-1-stock-price-notification-system)
7. [Implementation 2: Generic EventBus (Publish/Subscribe)](#7-implementation-2-generic-eventbus-publishsubscribe)
8. [Thread Safety in Observers](#8-thread-safety-in-observers)
9. [Real-World Framework Examples](#9-real-world-framework-examples)
10. [Trade-offs: Pros and Cons](#10-trade-offs-pros-and-cons)
11. [Common Interview Discussion Points](#11-common-interview-discussion-points)
12. [Interview Questions and Model Answers](#12-interview-questions-and-model-answers)
13. [Comparison with Related Patterns](#13-comparison-with-related-patterns)
14. [Memory Leak Issues and Fixes](#14-memory-leak-issues-and-fixes)

---

## 1. Intent and Problem It Solves

### Definition

The Observer pattern defines a **one-to-many dependency** between objects so that when one object (the **Subject** or **Publisher**) changes state, all its dependents (the **Observers** or **Subscribers**) are notified and updated automatically.

This is the foundational pattern behind **event-driven programming**, **reactive systems**, **GUI frameworks**, and **message-driven architectures**.

### The Core Problem

Consider a news agency and its subscribers. When a news story breaks, the agency does not call every subscriber individually — subscribers register interest and receive updates. The agency does not need to know anything about who reads the news or how many subscribers exist. Subscribers come and go without the agency changing.

Without the Observer pattern, you are forced into one of two bad options:

**Option A — Polling (Busy Wait)**
```java
// Without Observer: consumers must poll for changes
while (true) {
    if (stockTracker.hasChanged()) {
        double price = stockTracker.getPrice();
        emailSystem.sendAlert(price);
        smsSystem.sendAlert(price);
        tradingBot.evaluate(price);
        // Tight coupling: StockTracker must know about every consumer
    }
    Thread.sleep(100);
}
```

Problems with polling:
- Wastes CPU cycles checking for changes that may not have occurred
- The Subject must be aware of all consumers — high coupling
- Adding a new consumer requires modifying the Subject
- Difficult to enable/disable consumers at runtime
- Response latency equals the polling interval

**Option B — Direct Calls (Tight Coupling)**
```java
public class StockTracker {
    private EmailSystem emailSystem;    // direct dependency
    private SMSSystem smsSystem;        // direct dependency
    private TradingBot tradingBot;      // direct dependency

    public void priceChanged(double newPrice) {
        emailSystem.sendAlert(newPrice);   // Subject knows too much
        smsSystem.sendAlert(newPrice);
        tradingBot.evaluate(newPrice);
    }
}
```

Problems with direct calls:
- Open/Closed Principle violation — adding a new consumer requires modifying StockTracker
- Impossible to add consumers at runtime
- Unit testing StockTracker requires all consumers to be instantiated
- StockTracker becomes a kitchen sink class

### What the Observer Pattern Achieves

```
Subject <---registers--- Observer
Subject ----notifies---> Observer (without knowing the concrete type)
```

- **Loose coupling**: Subject knows only about the `Observer` interface, not concrete implementations
- **Open/Closed compliance**: New observers can be added without touching the Subject
- **Runtime flexibility**: Observers register and deregister dynamically
- **Single Responsibility**: Subject manages state; observers handle their own reactions
- **Foundation for event-driven programming**: every event system, from Swing to Kafka, is an Observer variant

---

## 2. When to Use / When NOT to Use

### Use Observer When

1. **A change in one object requires changing others, and you do not know how many objects need to change.**
   - Example: user logs in, and multiple UI components (profile widget, notification badge, shopping cart) all need to update.

2. **An object should be able to notify other objects without making assumptions about what those objects are.**
   - Example: a Domain Event in DDD where the Order domain object fires `OrderPlaced` without knowing whether Inventory, Billing, or Email services handle it.

3. **You need to implement distributed event handling systems.**
   - Example: a message broker where producers and consumers are decoupled across services.

4. **Temporary or conditional interest in events.**
   - A user toggles email alerts on/off — the observer registers and deregisters dynamically.

5. **You want to implement the MVC (Model-View-Controller) pattern.**
   - The Model is the Subject. Views are Observers. When the Model changes, all Views update automatically.

6. **You need audit logging, metrics collection, or monitoring as a cross-cutting concern.**
   - Register a logging observer without polluting business logic.

### Do NOT Use Observer When

1. **The dependency graph is complex and circular.**
   - Observer A updates Subject B, which notifies Observer B, which updates Subject A — this causes cascading updates, infinite loops, and debugging nightmares.

2. **Notification order matters and must be guaranteed.**
   - Observers are typically notified in registration order (an implementation detail), but this is fragile. If correctness depends on ordering, use a Pipeline or Chain of Responsibility instead.

3. **You need synchronous, transactional consistency across updates.**
   - If Observer A's failure must roll back Observer B's changes, Observer is the wrong pattern. Use a Saga, Unit of Work, or transactional outbox instead.

4. **The number of observers is small, fixed, and known at compile time.**
   - Direct calls are simpler, easier to trace, and carry no registration overhead.

5. **High-frequency events with heavy observers.**
   - Firing 10,000 events/second to 100 heavy-weight observers will overwhelm the system. Consider batching, backpressure (RxJava), or a dedicated message queue.

6. **Observers need to influence whether the subject proceeds.**
   - If an observer needs to veto an action (e.g., a validation step), use Chain of Responsibility or an interceptor pipeline instead.

---

## 3. UML Class Diagrams

### 3.1 Classic Subject/Observer

```
+------------------+           +------------------+
|    <<interface>> |           |   <<interface>>  |
|      Subject     |           |     Observer     |
+------------------+           +------------------+
| +attach(o)       |---------->| +update(data)    |
| +detach(o)       |    1..*   +------------------+
| +notifyObservers()|                   ^
+------------------+                   |
         ^                    +--------+--------+
         |                    |                 |
+------------------+  +--------------+  +--------------+
| ConcreteSubject  |  |ConcreteObsA  |  |ConcreteObsB  |
+------------------+  +--------------+  +--------------+
| -state           |  | -subject     |  | -subject     |
| -observers: List |  | +update(d)   |  | +update(d)   |
| +getState()      |  +--------------+  +--------------+
| +setState()      |
| +attach(o)       |
| +detach(o)       |
| +notify()        |
+------------------+
```

**Key relationships:**
- Subject holds a collection of Observers (`List<Observer>`)
- ConcreteSubject extends/implements Subject
- ConcreteObservers implement the Observer interface
- Subject only depends on the `Observer` interface — never on concrete types

### 3.2 Publisher/Subscriber Variation (with Event Bus)

In the pure Pub/Sub variation, publishers and subscribers are **completely decoupled** — they do not reference each other at all. An intermediary (EventBus / MessageBroker) sits between them.

```
+-------------+     publish(event)    +------------------+
|  Publisher  |---------------------->|    EventBus      |
+-------------+                       +------------------+
                                       | -subscribers:   |
                                       |  Map<Topic,List>|
                                       | +publish(event) |
                                       | +subscribe(t,s) |
                                       | +unsubscribe()  |
                                       +------------------+
                                              |
                              deliver(event)  |
                    +--------------------------+
                    |                          |
          +------------------+      +------------------+
          |  <<interface>>   |      |  <<interface>>   |
          |   Subscriber     |      |   Subscriber     |
          +------------------+      +------------------+
                    ^                          ^
                    |                          |
          +------------------+      +------------------+
          |  EmailSubscriber |      |  SMSSubscriber   |
          +------------------+      +------------------+
```

**Key difference from classic Observer:**
- Publisher does not hold a list of subscribers — the EventBus does
- Publisher publishes to a **topic**, not to subscribers directly
- Subscriber subscribes to a **topic**, not to a publisher directly
- Publishers and subscribers can be in different processes (Kafka, RabbitMQ)

### 3.3 Structural Comparison

| Dimension          | Subject/Observer           | Publisher/Subscriber       |
|--------------------|----------------------------|----------------------------|
| Coupling           | Subject knows Observer interface | Neither knows the other |
| Intermediary       | None (direct notification) | EventBus / Broker          |
| Filtering          | All observers notified     | Topic-based filtering      |
| Deployment         | In-process only            | Can span processes/services|
| Complexity         | Simple                     | More infrastructure        |

---

## 4. Push vs Pull Model

This is one of the most important design decisions when implementing Observer. Both are valid; the choice depends on your context.

### 4.1 Push Model

The Subject **pushes** all relevant data to the observer via the `update` method parameters. Observers receive the data without asking.

```java
// Observer interface — data is pushed as parameter
public interface PriceObserver {
    void onPriceChange(String symbol, double oldPrice, double newPrice, long timestamp);
}

// Subject pushes full data packet
public class StockMarket {
    private final List<PriceObserver> observers = new ArrayList<>();
    private final Map<String, Double> prices = new HashMap<>();

    public void addObserver(PriceObserver observer) {
        observers.add(observer);
    }

    public void removeObserver(PriceObserver observer) {
        observers.remove(observer);
    }

    public void updatePrice(String symbol, double newPrice) {
        double oldPrice = prices.getOrDefault(symbol, 0.0);
        prices.put(symbol, newPrice);
        // Push all data the observer might need
        notifyObservers(symbol, oldPrice, newPrice, System.currentTimeMillis());
    }

    private void notifyObservers(String symbol, double oldPrice, double newPrice, long timestamp) {
        for (PriceObserver observer : observers) {
            observer.onPriceChange(symbol, oldPrice, newPrice, timestamp);
        }
    }
}

// Observer receives complete data — does not need to call back
public class AlertObserver implements PriceObserver {
    @Override
    public void onPriceChange(String symbol, double oldPrice, double newPrice, long timestamp) {
        double changePercent = ((newPrice - oldPrice) / oldPrice) * 100;
        if (Math.abs(changePercent) > 5.0) {
            System.out.printf("[%s] ALERT: %s moved %.2f%% from %.2f to %.2f%n",
                new java.util.Date(timestamp), symbol, changePercent, oldPrice, newPrice);
        }
    }
}
```

**Push model characteristics:**
- Observer is **self-contained** — it has everything it needs in the method call
- Subject decides what data to send — may send data observers do not need (over-notification)
- Adding new data fields to the notification requires changing the `update` interface
- Simpler observer implementations
- Works well when all observers need the same information

### 4.2 Pull Model

The Subject notifies observers that **something changed**, but passes only a minimal reference (or nothing). Observers call back into the Subject to pull the specific data they care about.

```java
// Observer interface — receives only a reference to the subject
public interface StockObserver {
    void onStockChanged(StockSubject subject, String symbol);
}

// Subject interface — exposes query methods
public interface StockSubject {
    double getCurrentPrice(String symbol);
    double getPreviousPrice(String symbol);
    double getDailyHigh(String symbol);
    double getDailyLow(String symbol);
    long getVolume(String symbol);
    double getMarketCap(String symbol);
}

// Concrete subject exposes rich state
public class StockExchange implements StockSubject {
    private final List<StockObserver> observers = new ArrayList<>();

    // Internal state — full data model
    private final Map<String, Double> currentPrices = new HashMap<>();
    private final Map<String, Double> previousPrices = new HashMap<>();
    private final Map<String, Double> dailyHighs = new HashMap<>();
    private final Map<String, Double> dailyLows = new HashMap<>();
    private final Map<String, Long> volumes = new HashMap<>();

    public void addObserver(StockObserver observer) {
        observers.add(observer);
    }

    public void removeObserver(StockObserver observer) {
        observers.remove(observer);
    }

    public void updatePrice(String symbol, double newPrice) {
        previousPrices.put(symbol, currentPrices.getOrDefault(symbol, newPrice));
        currentPrices.put(symbol, newPrice);
        dailyHighs.merge(symbol, newPrice, Math::max);
        dailyLows.merge(symbol, newPrice, Math::min);
        volumes.merge(symbol, 1L, Long::sum);

        // Notify with minimal data — just the reference and symbol
        for (StockObserver observer : observers) {
            observer.onStockChanged(this, symbol);
        }
    }

    @Override
    public double getCurrentPrice(String symbol) {
        return currentPrices.getOrDefault(symbol, 0.0);
    }

    @Override
    public double getPreviousPrice(String symbol) {
        return previousPrices.getOrDefault(symbol, 0.0);
    }

    @Override
    public double getDailyHigh(String symbol) {
        return dailyHighs.getOrDefault(symbol, 0.0);
    }

    @Override
    public double getDailyLow(String symbol) {
        return dailyLows.getOrDefault(symbol, 0.0);
    }

    @Override
    public long getVolume(String symbol) {
        return volumes.getOrDefault(symbol, 0L);
    }

    @Override
    public double getMarketCap(String symbol) {
        return currentPrices.getOrDefault(symbol, 0.0) * 1_000_000; // simplified
    }
}

// Observer A: Only cares about price and previous price
public class PriceAlertObserver implements StockObserver {
    private final double thresholdPercent;

    public PriceAlertObserver(double thresholdPercent) {
        this.thresholdPercent = thresholdPercent;
    }

    @Override
    public void onStockChanged(StockSubject subject, String symbol) {
        // Pull only what this observer needs
        double current = subject.getCurrentPrice(symbol);
        double previous = subject.getPreviousPrice(symbol);

        if (previous == 0) return;
        double changePercent = ((current - previous) / previous) * 100;
        if (Math.abs(changePercent) >= thresholdPercent) {
            System.out.printf("PRICE ALERT: %s changed by %.2f%%%n", symbol, changePercent);
        }
    }
}

// Observer B: Cares about high/low/volume — different data needs
public class TechnicalAnalysisObserver implements StockObserver {
    @Override
    public void onStockChanged(StockSubject subject, String symbol) {
        // Pull only what this observer needs — different fields than Observer A
        double high = subject.getDailyHigh(symbol);
        double low = subject.getDailyLow(symbol);
        double current = subject.getCurrentPrice(symbol);
        long volume = subject.getVolume(symbol);

        double range = high - low;
        double position = (range > 0) ? ((current - low) / range) * 100 : 50;
        System.out.printf("TECHNICAL: %s | H:%.2f L:%.2f Vol:%d Position:%.1f%%%n",
            symbol, high, low, volume, position);
    }
}
```

**Pull model characteristics:**
- Observer pulls **only the data it needs** — no over-notification
- Subject's `update()` signature stays stable even when the Subject's data model grows
- Adding new data to Subject does not break existing observers
- Observers require a reference back to the Subject — creates a bidirectional dependency
- Observers can pull **stale** data if the Subject is mutable and other observers ran first
- More complex observer implementations (must know the Subject's API)

### 4.3 Hybrid Approach (Best of Both)

Most production systems use a hybrid: push a small, typed event object that carries the most commonly needed data, and optionally include a reference to the source for further queries.

```java
// Typed event object (pushed)
public final class StockEvent {
    private final String symbol;
    private final double previousPrice;
    private final double currentPrice;
    private final long timestamp;
    private final StockSubject source; // optional pull-back reference

    public StockEvent(String symbol, double previousPrice, double currentPrice,
                      long timestamp, StockSubject source) {
        this.symbol = symbol;
        this.previousPrice = previousPrice;
        this.currentPrice = currentPrice;
        this.timestamp = timestamp;
        this.source = source;
    }

    public String getSymbol() { return symbol; }
    public double getPreviousPrice() { return previousPrice; }
    public double getCurrentPrice() { return currentPrice; }
    public long getTimestamp() { return timestamp; }
    public double getChangePercent() {
        if (previousPrice == 0) return 0;
        return ((currentPrice - previousPrice) / previousPrice) * 100;
    }
    // Observer can pull more if needed
    public StockSubject getSource() { return source; }
}

public interface StockEventObserver {
    void onStockEvent(StockEvent event);
}
```

### 4.4 When to Choose Each

| Scenario                                         | Recommendation |
|--------------------------------------------------|----------------|
| All observers need the same data                 | Push           |
| Observers have very different data needs         | Pull           |
| Subject's data model grows frequently            | Pull or Hybrid |
| Observers are lightweight and fast               | Push           |
| Performance-critical, minimal data transfer      | Pull           |
| Clean API, easy testability                      | Hybrid         |
| Network transmission (serialization needed)      | Push (typed DTO) |

---

## 5. Java Observable (Deprecated) and Why

Java 1.0 included `java.util.Observable` (class) and `java.util.Observer` (interface). They were **deprecated in Java 9** and should never be used in new code.

### What it Looked Like

```java
// The old Java way — DO NOT USE
import java.util.Observable;   // deprecated
import java.util.Observer;     // deprecated

class OldStyleStockTracker extends Observable {
    private double price;

    public void setPrice(double price) {
        this.price = price;
        setChanged();           // must call this or notifyObservers does nothing
        notifyObservers(price); // push model
    }

    public double getPrice() { return price; }
}

class OldStyleEmailAlert implements Observer {
    @Override
    public void update(Observable o, Object arg) {
        // Must downcast — no type safety
        OldStyleStockTracker tracker = (OldStyleStockTracker) o;
        double price = (Double) arg;
        System.out.println("Email alert: price is " + price);
    }
}

// Usage
OldStyleStockTracker tracker = new OldStyleStockTracker();
tracker.addObserver(new OldStyleEmailAlert());
tracker.setPrice(150.0);
```

### Why It Was Deprecated — Five Critical Reasons

**1. Observable is a class, not an interface — breaks inheritance**

Java has single inheritance. If your Subject already extends another class (e.g., `Thread`, `JPanel`, or a domain base class), you cannot also extend `Observable`. You are forced into composition workarounds.

```java
// Cannot do this in Java:
class StockTracker extends BusinessEntity, Observable { ... } // COMPILE ERROR
```

**2. `setChanged()` is not thread-safe and is easily forgotten**

`setChanged()` sets an internal boolean flag. `notifyObservers()` only fires if this flag is true and then resets it. This design is error-prone: forget `setChanged()` and observers never fire. The pattern forces a two-step call that should conceptually be one.

```java
// Easy to forget setChanged():
public void setPrice(double price) {
    this.price = price;
    // Forgot setChanged() — notifyObservers() is now a no-op
    notifyObservers(price); // silently does nothing
}
```

**3. The `update(Observable o, Object arg)` signature is not type-safe**

Both the Observable reference and the argument are passed as raw types. Every observer must downcast, which produces `ClassCastException` at runtime rather than compile-time errors.

```java
public void update(Observable o, Object arg) {
    // Every observer does this unsafe dance:
    OldStyleStockTracker tracker = (OldStyleStockTracker) o; // ClassCastException risk
    Double price = (Double) arg;                              // ClassCastException risk
}
```

**4. Notification order is undefined and implementation-dependent**

`Observable` uses a `Vector` internally (itself deprecated), and observers are notified in **reverse registration order** — which is an accident of the implementation, not a guaranteed contract. Code relying on order will break.

**5. No support for concurrent modification**

If an observer tries to register or deregister another observer during notification, a `ConcurrentModificationException` is thrown. Modern implementations use a copy-on-write list to handle this safely.

### The Modern Replacement

There is no single replacement — Java's designers intentionally left Observer as a pattern to implement yourself using interfaces, giving you full control over typing, threading, and the notification contract. The implementations in Sections 6 and 7 demonstrate the modern approach.

---

## 6. Implementation 1: Stock Price Notification System

This is a complete, compilable, production-quality implementation demonstrating:
- Typed events with immutable value objects
- Multiple concrete observer types
- Registration/deregistration at runtime
- Filtering within observers
- Clean separation of concerns

```java
package com.course.patterns.observer.stock;

import java.time.Instant;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

// ─────────────────────────────────────────────
// 1. DOMAIN VALUE OBJECT: StockEvent (immutable)
// ─────────────────────────────────────────────
final class StockEvent {
    private final String symbol;
    private final double previousPrice;
    private final double currentPrice;
    private final long volume;
    private final Instant timestamp;

    StockEvent(String symbol, double previousPrice, double currentPrice, long volume) {
        this.symbol = Objects.requireNonNull(symbol, "symbol must not be null");
        if (currentPrice < 0) throw new IllegalArgumentException("Price cannot be negative");
        this.previousPrice = previousPrice;
        this.currentPrice = currentPrice;
        this.volume = volume;
        this.timestamp = Instant.now();
    }

    public String getSymbol() { return symbol; }
    public double getPreviousPrice() { return previousPrice; }
    public double getCurrentPrice() { return currentPrice; }
    public long getVolume() { return volume; }
    public Instant getTimestamp() { return timestamp; }

    public double getChangeAmount() {
        return currentPrice - previousPrice;
    }

    public double getChangePercent() {
        if (previousPrice == 0) return 0;
        return ((currentPrice - previousPrice) / previousPrice) * 100.0;
    }

    public boolean isPriceUp() { return currentPrice > previousPrice; }
    public boolean isPriceDown() { return currentPrice < previousPrice; }

    @Override
    public String toString() {
        return String.format("StockEvent{symbol='%s', prev=%.2f, curr=%.2f, chg=%.2f%%}",
            symbol, previousPrice, currentPrice, getChangePercent());
    }
}

// ─────────────────────────────────────────────
// 2. OBSERVER INTERFACE
// ─────────────────────────────────────────────
interface StockObserver {
    /**
     * Called by the Subject when a stock price changes.
     * Implementations must not throw exceptions — handle errors internally.
     *
     * @param event the immutable event describing the change
     */
    void onStockEvent(StockEvent event);

    /**
     * A human-readable name for logging and debugging.
     */
    String getName();
}

// ─────────────────────────────────────────────
// 3. SUBJECT INTERFACE
// ─────────────────────────────────────────────
interface StockMarket {
    void subscribe(StockObserver observer);
    void unsubscribe(StockObserver observer);
    void updatePrice(String symbol, double newPrice, long volume);
    double getCurrentPrice(String symbol);
    List<String> getTrackedSymbols();
}

// ─────────────────────────────────────────────
// 4. CONCRETE SUBJECT: StockExchange
// ─────────────────────────────────────────────
class StockExchange implements StockMarket {

    private final String exchangeName;

    // CopyOnWriteArrayList: thread-safe, allows modification during iteration
    private final List<StockObserver> observers = new CopyOnWriteArrayList<>();

    // State per symbol
    private final Map<String, Double> currentPrices = new HashMap<>();
    private final Map<String, Double> previousPrices = new HashMap<>();
    private final Map<String, Long> volumes = new HashMap<>();

    StockExchange(String exchangeName) {
        this.exchangeName = Objects.requireNonNull(exchangeName);
    }

    @Override
    public void subscribe(StockObserver observer) {
        Objects.requireNonNull(observer, "observer must not be null");
        if (!observers.contains(observer)) {
            observers.add(observer);
            System.out.printf("[%s] Observer registered: %s%n", exchangeName, observer.getName());
        }
    }

    @Override
    public void unsubscribe(StockObserver observer) {
        boolean removed = observers.remove(observer);
        if (removed) {
            System.out.printf("[%s] Observer unregistered: %s%n", exchangeName, observer.getName());
        }
    }

    @Override
    public void updatePrice(String symbol, double newPrice, long volume) {
        Objects.requireNonNull(symbol, "symbol must not be null");

        double oldPrice = currentPrices.getOrDefault(symbol, 0.0);
        currentPrices.put(symbol, newPrice);
        previousPrices.put(symbol, oldPrice);
        volumes.put(symbol, volume);

        StockEvent event = new StockEvent(symbol, oldPrice, newPrice, volume);
        notifyObservers(event);
    }

    @Override
    public double getCurrentPrice(String symbol) {
        return currentPrices.getOrDefault(symbol, 0.0);
    }

    @Override
    public List<String> getTrackedSymbols() {
        return Collections.unmodifiableList(new ArrayList<>(currentPrices.keySet()));
    }

    private void notifyObservers(StockEvent event) {
        for (StockObserver observer : observers) { // CopyOnWriteArrayList: safe iteration
            try {
                observer.onStockEvent(event);
            } catch (Exception e) {
                // One bad observer must not prevent others from being notified
                System.err.printf("[%s] Observer '%s' threw exception: %s%n",
                    exchangeName, observer.getName(), e.getMessage());
            }
        }
    }

    public String getExchangeName() { return exchangeName; }
}

// ─────────────────────────────────────────────
// 5. CONCRETE OBSERVER A: EmailAlertObserver
//    Sends email when price moves beyond a threshold
// ─────────────────────────────────────────────
class EmailAlertObserver implements StockObserver {
    private final String emailAddress;
    private final double alertThresholdPercent;
    private final Set<String> watchedSymbols; // filter to specific symbols

    EmailAlertObserver(String emailAddress, double alertThresholdPercent, String... symbols) {
        this.emailAddress = Objects.requireNonNull(emailAddress);
        this.alertThresholdPercent = alertThresholdPercent;
        this.watchedSymbols = new HashSet<>(Arrays.asList(symbols));
    }

    @Override
    public void onStockEvent(StockEvent event) {
        // Filter: only handle watched symbols
        if (!watchedSymbols.isEmpty() && !watchedSymbols.contains(event.getSymbol())) {
            return;
        }

        // Filter: only alert on significant moves
        if (Math.abs(event.getChangePercent()) < alertThresholdPercent) {
            return;
        }

        String direction = event.isPriceUp() ? "UP" : "DOWN";
        String subject = String.format("PRICE ALERT: %s %s %.2f%%",
            event.getSymbol(), direction, Math.abs(event.getChangePercent()));

        String body = String.format(
            "Stock: %s%n" +
            "Previous Price: $%.2f%n" +
            "Current Price:  $%.2f%n" +
            "Change:         $%.2f (%.2f%%)%n" +
            "Volume:         %,d%n" +
            "Timestamp:      %s%n",
            event.getSymbol(),
            event.getPreviousPrice(),
            event.getCurrentPrice(),
            event.getChangeAmount(),
            event.getChangePercent(),
            event.getVolume(),
            event.getTimestamp()
        );

        sendEmail(emailAddress, subject, body);
    }

    private void sendEmail(String to, String subject, String body) {
        // In production: inject an EmailService and call it here
        System.out.printf("[EMAIL -> %s]%nSubject: %s%nBody:%n%s%n", to, subject, body);
    }

    @Override
    public String getName() {
        return "EmailAlert(" + emailAddress + ", threshold=" + alertThresholdPercent + "%)";
    }
}

// ─────────────────────────────────────────────
// 6. CONCRETE OBSERVER B: SMSAlertObserver
//    Sends SMS for extreme price movements
// ─────────────────────────────────────────────
class SMSAlertObserver implements StockObserver {
    private final String phoneNumber;
    private final double extremeThresholdPercent; // only SMS on big moves

    SMSAlertObserver(String phoneNumber, double extremeThresholdPercent) {
        this.phoneNumber = Objects.requireNonNull(phoneNumber);
        this.extremeThresholdPercent = extremeThresholdPercent;
    }

    @Override
    public void onStockEvent(StockEvent event) {
        if (Math.abs(event.getChangePercent()) < extremeThresholdPercent) {
            return; // SMS only for extreme moves
        }

        String message = String.format("URGENT: %s %s %.2f%% | Now: $%.2f",
            event.getSymbol(),
            event.isPriceUp() ? "surged" : "crashed",
            Math.abs(event.getChangePercent()),
            event.getCurrentPrice());

        sendSMS(phoneNumber, message);
    }

    private void sendSMS(String to, String message) {
        System.out.printf("[SMS -> %s]: %s%n", to, message);
    }

    @Override
    public String getName() {
        return "SMSAlert(" + phoneNumber + ", extreme=" + extremeThresholdPercent + "%)";
    }
}

// ─────────────────────────────────────────────
// 7. CONCRETE OBSERVER C: TradingBotObserver
//    Executes automated trades based on price signals
// ─────────────────────────────────────────────
class TradingBotObserver implements StockObserver {
    private final String botId;
    private final Map<String, TradingStrategy> strategyMap;
    private final List<TradeOrder> executedOrders = new ArrayList<>();

    enum TradingStrategy { MOMENTUM, MEAN_REVERSION, VOLUME_SPIKE }

    record TradeOrder(String symbol, String action, double price, int quantity, Instant time) {
        @Override
        public String toString() {
            return String.format("Order{%s %s %d shares @ $%.2f at %s}",
                action, symbol, quantity, price, time);
        }
    }

    TradingBotObserver(String botId) {
        this.botId = botId;
        this.strategyMap = new HashMap<>();
    }

    public void addStrategy(String symbol, TradingStrategy strategy) {
        strategyMap.put(symbol, strategy);
    }

    @Override
    public void onStockEvent(StockEvent event) {
        TradingStrategy strategy = strategyMap.getOrDefault(event.getSymbol(), TradingStrategy.MOMENTUM);

        TradeOrder order = switch (strategy) {
            case MOMENTUM -> {
                // Buy on strong upward momentum, sell on strong downward
                if (event.getChangePercent() > 3.0) {
                    yield new TradeOrder(event.getSymbol(), "BUY",
                        event.getCurrentPrice(), calculateQuantity(event), Instant.now());
                } else if (event.getChangePercent() < -3.0) {
                    yield new TradeOrder(event.getSymbol(), "SELL",
                        event.getCurrentPrice(), calculateQuantity(event), Instant.now());
                }
                yield null;
            }
            case MEAN_REVERSION -> {
                // Buy when oversold, sell when overbought
                if (event.getChangePercent() < -5.0) {
                    yield new TradeOrder(event.getSymbol(), "BUY",
                        event.getCurrentPrice(), calculateQuantity(event), Instant.now());
                } else if (event.getChangePercent() > 5.0) {
                    yield new TradeOrder(event.getSymbol(), "SELL",
                        event.getCurrentPrice(), calculateQuantity(event), Instant.now());
                }
                yield null;
            }
            case VOLUME_SPIKE -> {
                // Buy on high-volume upward moves
                if (event.getVolume() > 1_000_000 && event.isPriceUp()) {
                    yield new TradeOrder(event.getSymbol(), "BUY",
                        event.getCurrentPrice(), 100, Instant.now());
                }
                yield null;
            }
        };

        if (order != null) {
            executeOrder(order);
        }
    }

    private int calculateQuantity(StockEvent event) {
        // Simplified: trade more when volatility is higher
        return (int) (Math.abs(event.getChangePercent()) * 10);
    }

    private void executeOrder(TradeOrder order) {
        executedOrders.add(order);
        System.out.printf("[BOT-%s] Executing: %s%n", botId, order);
        // In production: call a brokerage API here
    }

    public List<TradeOrder> getExecutedOrders() {
        return Collections.unmodifiableList(executedOrders);
    }

    @Override
    public String getName() { return "TradingBot-" + botId; }
}

// ─────────────────────────────────────────────
// 8. CONCRETE OBSERVER D: DashboardObserver
//    Maintains in-memory state for a UI dashboard
// ─────────────────────────────────────────────
class DashboardObserver implements StockObserver {
    private final String dashboardId;

    // Dashboard state — latest data per symbol
    private final Map<String, Double> latestPrices = new LinkedHashMap<>();
    private final Map<String, Double> latestChanges = new LinkedHashMap<>();
    private final Map<String, Long> latestVolumes = new LinkedHashMap<>();
    private final Map<String, Double> dailyHighs = new LinkedHashMap<>();
    private final Map<String, Double> dailyLows = new LinkedHashMap<>();

    private static final DateTimeFormatter FORMATTER = DateTimeFormatter
        .ofPattern("HH:mm:ss")
        .withZone(ZoneId.systemDefault());

    DashboardObserver(String dashboardId) {
        this.dashboardId = dashboardId;
    }

    @Override
    public void onStockEvent(StockEvent event) {
        String symbol = event.getSymbol();

        // Update internal state
        latestPrices.put(symbol, event.getCurrentPrice());
        latestChanges.put(symbol, event.getChangePercent());
        latestVolumes.put(symbol, event.getVolume());

        dailyHighs.merge(symbol, event.getCurrentPrice(), Math::max);
        dailyLows.merge(symbol, event.getCurrentPrice(), Math::min);

        renderUpdate(event);
    }

    private void renderUpdate(StockEvent event) {
        String arrow = event.isPriceUp() ? "▲" : (event.isPriceDown() ? "▼" : "─");
        String color = event.isPriceUp() ? "GREEN" : (event.isPriceDown() ? "RED" : "GRAY");
        System.out.printf("[DASHBOARD-%s | %s] %s %s $%.2f (%+.2f%%) Vol:%,d%n",
            dashboardId,
            FORMATTER.format(event.getTimestamp()),
            event.getSymbol(),
            arrow,
            event.getCurrentPrice(),
            event.getChangePercent(),
            event.getVolume());
    }

    public void printFullDashboard() {
        System.out.println("\n=== DASHBOARD-" + dashboardId + " FULL STATE ===");
        System.out.printf("%-8s %10s %10s %10s %10s %12s%n",
            "SYMBOL", "PRICE", "CHANGE%", "HIGH", "LOW", "VOLUME");
        System.out.println("-".repeat(65));
        for (String symbol : latestPrices.keySet()) {
            System.out.printf("%-8s %10.2f %9.2f%% %10.2f %10.2f %12,d%n",
                symbol,
                latestPrices.get(symbol),
                latestChanges.getOrDefault(symbol, 0.0),
                dailyHighs.getOrDefault(symbol, 0.0),
                dailyLows.getOrDefault(symbol, 0.0),
                latestVolumes.getOrDefault(symbol, 0L));
        }
        System.out.println("=".repeat(65) + "\n");
    }

    @Override
    public String getName() { return "Dashboard-" + dashboardId; }
}

// ─────────────────────────────────────────────
// 9. DEMO / MAIN
// ─────────────────────────────────────────────
public class StockMarketDemo {

    public static void main(String[] args) throws InterruptedException {
        StockExchange nyse = new StockExchange("NYSE");

        // Create observers
        EmailAlertObserver emailAlert = new EmailAlertObserver(
            "trader@example.com", 3.0, "AAPL", "GOOGL", "TSLA");

        SMSAlertObserver smsAlert = new SMSAlertObserver("+1-555-0100", 8.0);

        TradingBotObserver tradingBot = new TradingBotObserver("ALPHA");
        tradingBot.addStrategy("AAPL", TradingBotObserver.TradingStrategy.MOMENTUM);
        tradingBot.addStrategy("TSLA", TradingBotObserver.TradingStrategy.MEAN_REVERSION);
        tradingBot.addStrategy("AMZN", TradingBotObserver.TradingStrategy.VOLUME_SPIKE);

        DashboardObserver dashboard = new DashboardObserver("MAIN");

        // Register all observers
        nyse.subscribe(emailAlert);
        nyse.subscribe(smsAlert);
        nyse.subscribe(tradingBot);
        nyse.subscribe(dashboard);

        System.out.println("\n--- Simulating market activity ---\n");

        // Normal moves — email fires, SMS does not
        nyse.updatePrice("AAPL", 185.00, 500_000);
        nyse.updatePrice("AAPL", 191.55, 750_000);  // +3.5% -> email fires

        // Extreme crash — both email and SMS fire
        nyse.updatePrice("TSLA", 200.00, 2_000_000);
        nyse.updatePrice("TSLA", 182.00, 5_000_000); // -9% -> email + SMS

        // Volume spike on AMZN
        nyse.updatePrice("AMZN", 185.00, 300_000);
        nyse.updatePrice("AMZN", 187.00, 1_500_000); // volume spike -> bot buys

        // Simulate GoogleDynamic subscription: a new observer registers mid-stream
        EmailAlertObserver lateSubscriber = new EmailAlertObserver(
            "latecomer@example.com", 1.0, "GOOGL");
        System.out.println("\n--- Late subscriber joins ---\n");
        nyse.subscribe(lateSubscriber);

        nyse.updatePrice("GOOGL", 140.00, 200_000);
        nyse.updatePrice("GOOGL", 142.50, 400_000); // +1.8% -> late subscriber gets it

        // Unsubscribe SMS observer
        System.out.println("\n--- SMS observer unsubscribes ---\n");
        nyse.unsubscribe(smsAlert);

        nyse.updatePrice("TSLA", 160.00, 8_000_000); // SMS no longer receives this

        // Print full dashboard
        dashboard.printFullDashboard();

        // Show trading bot's executed orders
        System.out.println("=== TRADING BOT ORDERS ===");
        tradingBot.getExecutedOrders().forEach(System.out::println);
    }
}
```

---

## 7. Implementation 2: Generic EventBus (Publish/Subscribe)

This is a production-quality, type-safe, generic EventBus. It supports:
- Type-safe event delivery using generics
- Topic-based routing
- Priority-ordered subscribers
- Dead letter handling for unhandled events
- Synchronous and asynchronous delivery modes

```java
package com.course.patterns.observer.eventbus;

import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Consumer;
import java.util.function.Predicate;

// ─────────────────────────────────────────────
// 1. BASE EVENT TYPE
// ─────────────────────────────────────────────
abstract class Event {
    private final String eventId;
    private final long timestamp;
    private final String source;

    protected Event(String source) {
        this.eventId = UUID.randomUUID().toString();
        this.timestamp = System.currentTimeMillis();
        this.source = Objects.requireNonNull(source);
    }

    public String getEventId() { return eventId; }
    public long getTimestamp() { return timestamp; }
    public String getSource() { return source; }

    @Override
    public String toString() {
        return String.format("%s{id='%s', source='%s', ts=%d}",
            getClass().getSimpleName(), eventId, source, timestamp);
    }
}

// ─────────────────────────────────────────────
// 2. CONCRETE EVENT TYPES
// ─────────────────────────────────────────────
final class UserRegisteredEvent extends Event {
    private final String userId;
    private final String email;
    private final String plan;

    UserRegisteredEvent(String source, String userId, String email, String plan) {
        super(source);
        this.userId = userId;
        this.email = email;
        this.plan = plan;
    }

    public String getUserId() { return userId; }
    public String getEmail() { return email; }
    public String getPlan() { return plan; }
}

final class OrderPlacedEvent extends Event {
    private final String orderId;
    private final String userId;
    private final double amount;
    private final List<String> itemIds;

    OrderPlacedEvent(String source, String orderId, String userId,
                     double amount, List<String> itemIds) {
        super(source);
        this.orderId = orderId;
        this.userId = userId;
        this.amount = amount;
        this.itemIds = Collections.unmodifiableList(new ArrayList<>(itemIds));
    }

    public String getOrderId() { return orderId; }
    public String getUserId() { return userId; }
    public double getAmount() { return amount; }
    public List<String> getItemIds() { return itemIds; }
}

final class PaymentProcessedEvent extends Event {
    private final String orderId;
    private final boolean success;
    private final String failureReason;

    PaymentProcessedEvent(String source, String orderId, boolean success, String failureReason) {
        super(source);
        this.orderId = orderId;
        this.success = success;
        this.failureReason = failureReason;
    }

    public String getOrderId() { return orderId; }
    public boolean isSuccess() { return success; }
    public String getFailureReason() { return failureReason; }
}

// ─────────────────────────────────────────────
// 3. SUBSCRIPTION — wraps a handler with metadata
// ─────────────────────────────────────────────
final class Subscription<T extends Event> {
    private static final AtomicLong ID_GENERATOR = new AtomicLong(0);

    private final long subscriptionId;
    private final String subscriberName;
    private final Consumer<T> handler;
    private final Predicate<T> filter;   // optional filter
    private final int priority;           // higher = called first
    private final boolean async;          // true = handle in thread pool

    Subscription(String subscriberName, Consumer<T> handler,
                 Predicate<T> filter, int priority, boolean async) {
        this.subscriptionId = ID_GENERATOR.incrementAndGet();
        this.subscriberName = Objects.requireNonNull(subscriberName);
        this.handler = Objects.requireNonNull(handler);
        this.filter = filter != null ? filter : event -> true;
        this.priority = priority;
        this.async = async;
    }

    public long getSubscriptionId() { return subscriptionId; }
    public String getSubscriberName() { return subscriberName; }
    public Consumer<T> getHandler() { return handler; }
    public Predicate<T> getFilter() { return filter; }
    public int getPriority() { return priority; }
    public boolean isAsync() { return async; }

    void deliver(T event) {
        if (filter.test(event)) {
            handler.accept(event);
        }
    }
}

// ─────────────────────────────────────────────
// 4. SUBSCRIPTION HANDLE (returned to subscriber for later cancellation)
// ─────────────────────────────────────────────
final class SubscriptionHandle {
    private final long subscriptionId;
    private final Runnable canceller;
    private volatile boolean cancelled = false;

    SubscriptionHandle(long subscriptionId, Runnable canceller) {
        this.subscriptionId = subscriptionId;
        this.canceller = canceller;
    }

    public void cancel() {
        if (!cancelled) {
            cancelled = true;
            canceller.run();
        }
    }

    public boolean isCancelled() { return cancelled; }
    public long getSubscriptionId() { return subscriptionId; }
}

// ─────────────────────────────────────────────
// 5. EVENTBUS — the central broker
// ─────────────────────────────────────────────
public class EventBus {

    private final String busName;

    // Map from event type → sorted list of subscriptions (by priority desc)
    // ConcurrentHashMap for thread-safe map access; CopyOnWriteArrayList for safe iteration
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription<?>>> subscriptions =
        new ConcurrentHashMap<>();

    // Async delivery thread pool
    private final ExecutorService asyncExecutor;

    // Dead letter queue — events with no subscribers
    private final Queue<Event> deadLetterQueue = new ConcurrentLinkedQueue<>();

    // Metrics
    private final AtomicLong totalPublished = new AtomicLong(0);
    private final AtomicLong totalDelivered = new AtomicLong(0);
    private final AtomicLong totalDeadLetters = new AtomicLong(0);

    public EventBus(String busName) {
        this(busName, Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors(),
            r -> {
                Thread t = new Thread(r, "EventBus-" + busName + "-async");
                t.setDaemon(true);
                return t;
            }
        ));
    }

    public EventBus(String busName, ExecutorService asyncExecutor) {
        this.busName = Objects.requireNonNull(busName);
        this.asyncExecutor = Objects.requireNonNull(asyncExecutor);
    }

    // ── Subscribe (fluent builder) ───────────────────────────────────────
    public <T extends Event> SubscriptionHandle subscribe(Class<T> eventType,
                                                          String subscriberName,
                                                          Consumer<T> handler) {
        return subscribeWithOptions(eventType, subscriberName, handler, null, 0, false);
    }

    public <T extends Event> SubscriptionHandle subscribeFiltered(Class<T> eventType,
                                                                   String subscriberName,
                                                                   Consumer<T> handler,
                                                                   Predicate<T> filter) {
        return subscribeWithOptions(eventType, subscriberName, handler, filter, 0, false);
    }

    public <T extends Event> SubscriptionHandle subscribeAsync(Class<T> eventType,
                                                                String subscriberName,
                                                                Consumer<T> handler) {
        return subscribeWithOptions(eventType, subscriberName, handler, null, 0, true);
    }

    public <T extends Event> SubscriptionHandle subscribeWithPriority(Class<T> eventType,
                                                                       String subscriberName,
                                                                       Consumer<T> handler,
                                                                       int priority) {
        return subscribeWithOptions(eventType, subscriberName, handler, null, priority, false);
    }

    @SuppressWarnings("unchecked")
    private <T extends Event> SubscriptionHandle subscribeWithOptions(
            Class<T> eventType, String subscriberName, Consumer<T> handler,
            Predicate<T> filter, int priority, boolean async) {

        Subscription<T> subscription = new Subscription<>(
            subscriberName, handler, filter, priority, async);

        subscriptions
            .computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
            .add(subscription);

        // Keep sorted by priority (descending)
        subscriptions.get(eventType).sort(
            Comparator.comparingInt(Subscription::getPriority).reversed());

        System.out.printf("[EventBus-%s] Subscribed: '%s' to %s (priority=%d, async=%b)%n",
            busName, subscriberName, eventType.getSimpleName(), priority, async);

        return new SubscriptionHandle(
            subscription.getSubscriptionId(),
            () -> unsubscribe(eventType, subscription.getSubscriptionId())
        );
    }

    // ── Publish ──────────────────────────────────────────────────────────
    @SuppressWarnings("unchecked")
    public <T extends Event> void publish(T event) {
        Objects.requireNonNull(event, "event must not be null");
        totalPublished.incrementAndGet();

        CopyOnWriteArrayList<Subscription<?>> subs = subscriptions.get(event.getClass());

        if (subs == null || subs.isEmpty()) {
            totalDeadLetters.incrementAndGet();
            deadLetterQueue.offer(event);
            System.out.printf("[EventBus-%s] DEAD LETTER: %s (no subscribers)%n",
                busName, event);
            return;
        }

        for (Subscription<?> rawSub : subs) {
            Subscription<T> sub = (Subscription<T>) rawSub; // safe: keyed by type
            if (sub.isAsync()) {
                asyncExecutor.submit(() -> deliverSafely(sub, event));
            } else {
                deliverSafely(sub, event);
            }
        }
    }

    private <T extends Event> void deliverSafely(Subscription<T> sub, T event) {
        try {
            sub.deliver(event);
            totalDelivered.incrementAndGet();
        } catch (Exception e) {
            System.err.printf("[EventBus-%s] Handler '%s' threw exception for %s: %s%n",
                busName, sub.getSubscriberName(), event.getClass().getSimpleName(),
                e.getMessage());
        }
    }

    // ── Unsubscribe ──────────────────────────────────────────────────────
    private void unsubscribe(Class<?> eventType, long subscriptionId) {
        CopyOnWriteArrayList<Subscription<?>> subs = subscriptions.get(eventType);
        if (subs != null) {
            subs.removeIf(s -> s.getSubscriptionId() == subscriptionId);
            System.out.printf("[EventBus-%s] Unsubscribed: subscription #%d from %s%n",
                busName, subscriptionId, eventType.getSimpleName());
        }
    }

    // ── Metrics and Admin ────────────────────────────────────────────────
    public void printMetrics() {
        System.out.printf("[EventBus-%s] Metrics: published=%d, delivered=%d, deadLetters=%d%n",
            busName, totalPublished.get(), totalDelivered.get(), totalDeadLetters.get());
    }

    public List<Event> drainDeadLetterQueue() {
        List<Event> drained = new ArrayList<>();
        Event event;
        while ((event = deadLetterQueue.poll()) != null) {
            drained.add(event);
        }
        return drained;
    }

    public void shutdown() {
        asyncExecutor.shutdown();
        try {
            if (!asyncExecutor.awaitTermination(5, TimeUnit.SECONDS)) {
                asyncExecutor.shutdownNow();
            }
        } catch (InterruptedException e) {
            asyncExecutor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}

// ─────────────────────────────────────────────
// 6. CONCRETE SUBSCRIBERS (handlers)
// ─────────────────────────────────────────────
class WelcomeEmailService {
    public void handleUserRegistered(UserRegisteredEvent event) {
        System.out.printf("[WelcomeEmailService] Sending welcome email to %s (plan: %s)%n",
            event.getEmail(), event.getPlan());
    }
}

class InventoryService {
    private final Map<String, Integer> reservations = new ConcurrentHashMap<>();

    public void handleOrderPlaced(OrderPlacedEvent event) {
        System.out.printf("[InventoryService] Reserving inventory for order %s: items=%s%n",
            event.getOrderId(), event.getItemIds());
        for (String itemId : event.getItemIds()) {
            reservations.merge(itemId, 1, Integer::sum);
        }
    }
}

class BillingService {
    public void handleOrderPlaced(OrderPlacedEvent event) {
        System.out.printf("[BillingService] Initiating payment of $%.2f for order %s%n",
            event.getAmount(), event.getOrderId());
    }

    public void handlePaymentProcessed(PaymentProcessedEvent event) {
        if (event.isSuccess()) {
            System.out.printf("[BillingService] Payment confirmed for order %s%n",
                event.getOrderId());
        } else {
            System.out.printf("[BillingService] Payment FAILED for order %s: %s%n",
                event.getOrderId(), event.getFailureReason());
        }
    }
}

class AuditLogger {
    private final List<String> auditLog = new CopyOnWriteArrayList<>();

    public void logUserRegistered(UserRegisteredEvent event) {
        String entry = String.format("[AUDIT] UserRegistered: userId=%s, email=%s, ts=%d",
            event.getUserId(), event.getEmail(), event.getTimestamp());
        auditLog.add(entry);
        System.out.println(entry);
    }

    public void logOrderPlaced(OrderPlacedEvent event) {
        String entry = String.format("[AUDIT] OrderPlaced: orderId=%s, amount=%.2f, ts=%d",
            event.getOrderId(), event.getAmount(), event.getTimestamp());
        auditLog.add(entry);
        System.out.println(entry);
    }

    public List<String> getAuditLog() {
        return Collections.unmodifiableList(auditLog);
    }
}

// ─────────────────────────────────────────────
// 7. DEMO / MAIN
// ─────────────────────────────────────────────
class EventBusDemo {

    public static void main(String[] args) throws InterruptedException {
        EventBus bus = new EventBus("ECOMMERCE");

        WelcomeEmailService welcomeService = new WelcomeEmailService();
        InventoryService inventoryService = new InventoryService();
        BillingService billingService = new BillingService();
        AuditLogger auditLogger = new AuditLogger();

        // Register subscribers
        bus.subscribe(UserRegisteredEvent.class, "WelcomeEmail",
            welcomeService::handleUserRegistered);

        bus.subscribe(UserRegisteredEvent.class, "AuditLogger-UserReg",
            auditLogger::logUserRegistered);

        // Inventory gets priority 10 — must reserve before billing charges
        bus.subscribeWithPriority(OrderPlacedEvent.class, "InventoryService",
            inventoryService::handleOrderPlaced, 10);

        bus.subscribeWithPriority(OrderPlacedEvent.class, "BillingService",
            billingService::handleOrderPlaced, 5);

        bus.subscribe(OrderPlacedEvent.class, "AuditLogger-OrderPlaced",
            auditLogger::logOrderPlaced);

        // Filtered subscription: only handle failed payments
        bus.subscribeFiltered(PaymentProcessedEvent.class, "BillingService-Payments",
            billingService::handlePaymentProcessed,
            event -> !event.isSuccess()); // only failures

        // Async subscriber for a heavy analytics job
        bus.subscribeAsync(OrderPlacedEvent.class, "AnalyticsService",
            event -> {
                System.out.printf("[AnalyticsService ASYNC] Processing order analytics for %s%n",
                    event.getOrderId());
                // Simulate slow analytics computation
            });

        System.out.println("\n--- Publishing events ---\n");

        // User registration
        bus.publish(new UserRegisteredEvent("UserService", "user-001",
            "alice@example.com", "PREMIUM"));

        // Order placed — inventory and billing both handle it
        bus.publish(new OrderPlacedEvent("OrderService", "order-001",
            "user-001", 149.99,
            Arrays.asList("item-A", "item-B", "item-C")));

        // Successful payment — filtered subscriber ignores it
        bus.publish(new PaymentProcessedEvent("PaymentService", "order-001", true, null));

        // Failed payment — filtered subscriber fires
        bus.publish(new PaymentProcessedEvent("PaymentService", "order-002", false,
            "INSUFFICIENT_FUNDS"));

        // Event with no subscribers — goes to dead letter queue
        // (not demonstrated to keep code clean, but the DLQ mechanism is wired)

        // Dynamic unsubscription
        SubscriptionHandle temporaryHandle = bus.subscribe(OrderPlacedEvent.class,
            "TemporaryAudit", e -> System.out.println("[TemporaryAudit] Order: " + e.getOrderId()));

        bus.publish(new OrderPlacedEvent("OrderService", "order-002",
            "user-002", 59.99, Arrays.asList("item-D")));

        System.out.println("\n--- Cancelling temporary subscriber ---\n");
        temporaryHandle.cancel();

        bus.publish(new OrderPlacedEvent("OrderService", "order-003",
            "user-003", 29.99, Arrays.asList("item-E")));
        // TemporaryAudit no longer receives this

        // Wait for async tasks to complete
        Thread.sleep(200);

        bus.printMetrics();
        bus.shutdown();
    }
}
```

---

## 8. Thread Safety in Observers

Concurrency is where most Observer implementations break in production. There are five distinct problems and their solutions.

### 8.1 Problem 1: ConcurrentModificationException During Notification

```java
// BROKEN: Using ArrayList for observers
public class UnsafeSubject {
    private final List<Observer> observers = new ArrayList<>(); // NOT thread-safe

    public void notifyObservers(String event) {
        for (Observer obs : observers) { // iterating here
            obs.update(event);           // observer might call unsubscribe() here
            // -> ConcurrentModificationException
        }
    }

    public void removeObserver(Observer obs) {
        observers.remove(obs); // modifying while iteration is in progress = boom
    }
}
```

**Solution A: CopyOnWriteArrayList**

```java
// CORRECT: Snapshot on write, safe to iterate
public class SafeSubjectCOW {
    // Reads (iteration) are lock-free; writes (add/remove) create a new array copy
    private final List<Observer> observers = new CopyOnWriteArrayList<>();

    public void notifyObservers(String event) {
        // Iterates over a snapshot — concurrent add/remove during iteration is safe
        for (Observer obs : observers) {
            obs.update(event);
        }
    }

    public void addObserver(Observer obs) {
        observers.add(obs); // thread-safe: copies the array
    }

    public void removeObserver(Observer obs) {
        observers.remove(obs); // thread-safe: copies the array
    }
}
// Trade-off: CopyOnWriteArrayList is expensive for frequent writes but
// cheap for frequent reads. Ideal when subscriptions are stable.
```

**Solution B: Synchronized Snapshot Before Notification**

```java
// CORRECT: Manual synchronization with snapshot
public class SafeSubjectSnapshot {
    private final List<Observer> observers = new ArrayList<>();

    public synchronized void addObserver(Observer obs) {
        observers.add(obs);
    }

    public synchronized void removeObserver(Observer obs) {
        observers.remove(obs);
    }

    public void notifyObservers(String event) {
        List<Observer> snapshot;
        synchronized (this) {
            snapshot = new ArrayList<>(observers); // take a snapshot under lock
        }
        // Notify from snapshot — original list can be safely modified concurrently
        for (Observer obs : snapshot) {
            obs.update(event); // NOT holding the lock during notification
        }
    }
}
// Trade-off: More memory allocation per notification, but allows observers
// to register/deregister without deadlock.
```

### 8.2 Problem 2: Deadlock When Observer Calls Back to Subject

```java
// DEADLOCK SCENARIO
public class DeadlockSubject {
    private synchronized void notifyObservers(String event) {
        for (Observer obs : observers) {
            obs.update(event); // calls observer while holding lock
        }
    }
}

// Observer calls back to subject during notification
class DeadlockObserver implements Observer {
    private final DeadlockSubject subject;

    @Override
    public void update(String event) {
        // This tries to acquire the lock on subject, which is already held
        // by the notification thread — DEADLOCK
        subject.addObserver(new AnotherObserver());
    }
}
```

**Solution: Never call external code while holding a lock.**

```java
public class DeadlockFreeSubject {
    private final List<Observer> observers = new ArrayList<>();
    private final Object lock = new Object();

    public void notifyObservers(String event) {
        List<Observer> snapshot;
        synchronized (lock) {
            snapshot = new ArrayList<>(observers); // short critical section
        }
        // Notify WITHOUT holding the lock — observers can call back freely
        for (Observer obs : snapshot) {
            obs.update(event);
        }
    }

    public void addObserver(Observer obs) {
        synchronized (lock) { observers.add(obs); }
    }

    public void removeObserver(Observer obs) {
        synchronized (lock) { observers.remove(obs); }
    }
}
```

### 8.3 Problem 3: Stale State in Pull Model

In the pull model, when an observer pulls from the subject, the subject's state may have already changed by the time it reads.

```java
// PROBLEM: Race condition in pull model
class RacingObserver implements Observer {
    @Override
    public void update(StockSubject subject, String symbol) {
        // Subject notifies us that AAPL changed
        // But between the notification and this read, another thread
        // updated AAPL again — we are pulling a newer value, not the
        // value that triggered our notification!
        double price = subject.getCurrentPrice(symbol); // stale!
    }
}
```

**Solution: Push immutable event objects.**

```java
// The StockEvent in Implementation 1 is immutable and captures the state
// at the moment the event was created — no race condition possible.
final class StockEvent {
    private final double previousPrice; // captured at creation time
    private final double currentPrice;  // captured at creation time
    // ... immutable fields, no setters
}
```

### 8.4 Problem 4: Observer Exception Corrupts Notification Chain

```java
// BROKEN: One observer failure stops the chain
public class FragileSubject {
    public void notifyObservers(String event) {
        for (Observer obs : observers) {
            obs.update(event); // if this throws, remaining observers never notified
        }
    }
}
```

**Solution: Isolate each observer's execution.**

```java
// CORRECT: Each observer is isolated
public class ResilientSubject {
    private static final Logger log = Logger.getLogger(ResilientSubject.class.getName());

    public void notifyObservers(String event) {
        for (Observer obs : observers) {
            try {
                obs.update(event);
            } catch (Exception e) {
                // Log the failure but continue notifying remaining observers
                log.severe(String.format(
                    "Observer %s threw exception: %s", obs.getClass().getSimpleName(),
                    e.getMessage()));
            }
        }
    }
}
```

### 8.5 Problem 5: Slow Observer Blocking the Notification Thread

```java
// PROBLEM: Slow observers block the subject's thread
class SlowDatabaseObserver implements Observer {
    @Override
    public void update(String event) {
        // This takes 2 seconds — blocks every other observer and the subject
        database.insertAuditRecord(event);
    }
}
```

**Solution: Asynchronous dispatch for slow observers.**

```java
public class AsyncNotifyingSubject {
    private final List<Observer> syncObservers = new CopyOnWriteArrayList<>();
    private final List<Observer> asyncObservers = new CopyOnWriteArrayList<>();
    private final ExecutorService pool = Executors.newFixedThreadPool(4);

    public void addSyncObserver(Observer obs) { syncObservers.add(obs); }
    public void addAsyncObserver(Observer obs) { asyncObservers.add(obs); }

    public void notifyObservers(String event) {
        // Sync observers get notified in-thread (fast, order-guaranteed)
        for (Observer obs : syncObservers) {
            try { obs.update(event); }
            catch (Exception e) { /* log */ }
        }

        // Async observers run on thread pool (no blocking, no order guarantee)
        for (Observer obs : asyncObservers) {
            pool.submit(() -> {
                try { obs.update(event); }
                catch (Exception e) { /* log */ }
            });
        }
    }
}
```

### 8.6 Complete Thread-Safe Observer Implementation

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;

public final class ThreadSafeEventSource<T> {

    private final CopyOnWriteArrayList<EventHandler<T>> handlers = new CopyOnWriteArrayList<>();
    private final ExecutorService asyncPool;
    private final AtomicBoolean shutdown = new AtomicBoolean(false);

    @FunctionalInterface
    public interface EventHandler<T> {
        void handle(T event) throws Exception;
    }

    public ThreadSafeEventSource(int asyncThreads) {
        this.asyncPool = Executors.newFixedThreadPool(asyncThreads,
            r -> { Thread t = new Thread(r, "event-handler"); t.setDaemon(true); return t; });
    }

    public void subscribe(EventHandler<T> handler) {
        Objects.requireNonNull(handler);
        handlers.add(handler);
    }

    public void unsubscribe(EventHandler<T> handler) {
        handlers.remove(handler);
    }

    public void publish(T event) {
        if (shutdown.get()) throw new IllegalStateException("EventSource is shut down");
        Objects.requireNonNull(event);

        // Snapshot is implicit with CopyOnWriteArrayList
        List<CompletableFuture<Void>> futures = new ArrayList<>();

        for (EventHandler<T> handler : handlers) {
            CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                try {
                    handler.handle(event);
                } catch (Exception e) {
                    System.err.println("Handler failed: " + e.getMessage());
                }
            }, asyncPool);
            futures.add(future);
        }

        // Wait for all handlers to complete (for synchronous publish semantics)
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    }

    public void shutdown() {
        shutdown.set(true);
        asyncPool.shutdown();
    }
}
```

---

## 9. Real-World Framework Examples

### 9.1 Java Swing — ActionListener (Classic Observer)

Swing is the canonical Java example of the Observer pattern. Every UI component is a Subject; listeners are Observers.

```java
import javax.swing.*;
import java.awt.event.*;

public class SwingObserverExample {
    public static void main(String[] args) {
        JButton button = new JButton("Click me");

        // Observer 1: Update status label (anonymous lambda)
        JLabel statusLabel = new JLabel("Ready");
        button.addActionListener(e -> statusLabel.setText("Button clicked at " + System.currentTimeMillis()));

        // Observer 2: Log to console
        button.addActionListener(e -> System.out.println("LOG: Button action at " + e.getWhen()));

        // Observer 3: Show dialog
        ActionListener dialogListener = e -> JOptionPane.showMessageDialog(null, "Hello!");
        button.addActionListener(dialogListener);

        // Dynamic unregistration
        button.removeActionListener(dialogListener);
        // dialogListener will no longer receive events

        // JButton (Subject) notifies all ActionListeners (Observers) on click
        // Internally, JButton maintains a javax.swing.event.EventListenerList
    }
}
```

Swing's `EventListenerList` is the internal Subject implementation. `addActionListener` / `removeActionListener` map to `attach` / `detach`. `fireActionPerformed` maps to `notifyObservers`.

### 9.2 Spring Framework — ApplicationEvent / ApplicationListener

Spring's event system is a full Pub/Sub implementation built into the IoC container.

```java
import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;
import org.springframework.context.ApplicationEventPublisher;

// Custom event (Subject state)
public class UserCreatedEvent extends ApplicationEvent {
    private final String userId;
    private final String email;

    public UserCreatedEvent(Object source, String userId, String email) {
        super(source);
        this.userId = userId;
        this.email = email;
    }

    public String getUserId() { return userId; }
    public String getEmail() { return email; }
}

// Publisher (acts as Subject)
@Component
public class UserService {
    private final ApplicationEventPublisher eventPublisher;

    public UserService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void createUser(String userId, String email) {
        // Business logic...
        // Publish event — UserService has NO knowledge of who handles it
        eventPublisher.publishEvent(new UserCreatedEvent(this, userId, email));
    }
}

// Observer 1: implements ApplicationListener<T> (traditional style)
@Component
public class WelcomeEmailListener implements ApplicationListener<UserCreatedEvent> {
    @Override
    public void onApplicationEvent(UserCreatedEvent event) {
        System.out.println("Sending welcome email to: " + event.getEmail());
    }
}

// Observer 2: @EventListener annotation (modern style, Spring 4.2+)
@Component
public class AuditService {

    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        System.out.println("Audit: User " + event.getUserId() + " created");
    }

    // Filtered event handling — only fire for premium users
    @EventListener(condition = "#event.email.endsWith('@enterprise.com')")
    public void handleEnterpriseUserCreated(UserCreatedEvent event) {
        System.out.println("Enterprise onboarding for: " + event.getEmail());
    }

    // Async event handling
    @EventListener
    @org.springframework.scheduling.annotation.Async
    public void handleUserCreatedAsync(UserCreatedEvent event) {
        System.out.println("Async analytics processing for: " + event.getUserId());
    }
}
```

Spring's `ApplicationEventPublisher` is the Subject. `ApplicationContext` is the EventBus. Observers are Spring beans that implement `ApplicationListener` or use `@EventListener`. The container manages registration — no manual `addObserver` calls needed.

### 9.3 RxJava — Reactive Streams

RxJava extends the Observer pattern with operators, backpressure, and schedulers. It is the foundation of reactive programming in Java.

```java
import io.reactivex.rxjava3.core.Observable;
import io.reactivex.rxjava3.schedulers.Schedulers;
import java.util.concurrent.TimeUnit;

public class RxJavaObserverExample {

    public static void main(String[] args) throws InterruptedException {

        // Observable is the Subject; it emits items to Observers
        Observable<StockEvent> stockStream = Observable.create(emitter -> {
            // Simulate real-time stock feed
            emitter.onNext(new StockEvent("AAPL", 185.0, 188.0, 500_000));
            emitter.onNext(new StockEvent("GOOGL", 140.0, 141.5, 300_000));
            emitter.onNext(new StockEvent("TSLA", 200.0, 182.0, 2_000_000));
            emitter.onComplete();
        });

        // RxJava operators compose the processing pipeline
        stockStream
            // Filter: only significant moves
            .filter(e -> Math.abs(e.getChangePercent()) > 2.0)
            // Transform: enrich with human-readable direction
            .map(e -> String.format("%s %s by %.2f%%",
                e.getSymbol(),
                e.isPriceUp() ? "UP" : "DOWN",
                Math.abs(e.getChangePercent())))
            // Backpressure: buffer bursts
            .onBackpressureBuffer(1000)
            // Observe on IO thread pool (non-blocking)
            .subscribeOn(Schedulers.io())
            // Subscribe (Observer)
            .subscribe(
                message -> System.out.println("Alert: " + message),
                error -> System.err.println("Error: " + error.getMessage()),
                () -> System.out.println("Stream complete")
            );

        // RxJava adds: operators (map, filter, merge, zip), backpressure,
        // schedulers (which thread), error handling, completion, and hot/cold observables.
        // It is Observer + Iterator + functional composition.
    }
}
```

**Key difference from basic Observer**: RxJava Observables are composable chains. You can merge two stock feeds, debounce rapid events, retry on failure, and switch threads — all in a single expression. The basic Observer pattern has none of this.

---

## 10. Trade-offs: Pros and Cons

### Advantages

| Advantage | Explanation |
|-----------|-------------|
| Loose coupling | Subject depends only on the Observer interface, not concrete types |
| Open/Closed Principle | Add new observers without modifying the Subject |
| Runtime flexibility | Subscribe/unsubscribe without recompiling or restarting |
| Single Responsibility | Subject handles state; observers handle reactions |
| Broadcast communication | One change notifies many consumers simultaneously |
| Testability | Mock the Observer interface to test Subject in isolation |
| Foundation for reactive | Every reactive and event-driven framework is built on this |

### Disadvantages

| Disadvantage | Explanation |
|--------------|-------------|
| Unexpected updates | Observers may fire in unexpected situations if the Subject over-notifies |
| Cascade / avalanche | Observer A updates Subject B, notifies Observer B, which updates Subject A |
| Notification order | Registration order is fragile; do not rely on it for correctness |
| Memory leaks | Observers registered but never removed keep objects alive indefinitely |
| Hidden dependencies | Hard to trace which observers fire and in what order from reading code |
| Thread safety complexity | Requires careful synchronization; easy to introduce race conditions |
| Debugging difficulty | Distributed notification makes stack traces confusing |
| Performance overhead | CopyOnWriteArrayList and synchronization add overhead for high-frequency events |

---

## 11. Common Interview Discussion Points

These are the topics interviewers probe in senior-level discussions. Memorizing the pattern itself is table stakes — interviewers care about your ability to reason about trade-offs.

### Thread Safety Discussion

Interviewers expect you to proactively identify: (a) `ConcurrentModificationException` from iterating while adding/removing, (b) race conditions in the pull model, (c) deadlocks from calling external code under a lock, and (d) slow observers blocking the notification thread. The recommended solution for (a) is `CopyOnWriteArrayList` or a synchronized snapshot. For (d), async dispatch with a thread pool or `CompletableFuture`.

### Memory Leak Discussion

The most common Observer bug in production. If an observer registers with a long-lived subject but is later garbage-collected (or should be), the subject's reference to it prevents collection. Interviewers look for: weak references, explicit deregistration in `close()`/`@PreDestroy`, or listener lifecycle management. See Section 14 for complete analysis.

### Push vs Pull Trade-off

Know both models cold. Push is simpler and works when all observers need the same data. Pull is more flexible and scales better as the Subject's data model grows. The hybrid approach (immutable event object with an optional reference back to the Subject) is the production-standard answer.

### Observer vs Pub/Sub

Observer: Subject and Observer know each other (through an interface). Direct, synchronous, in-process. Pub/Sub: Publisher and Subscriber are completely decoupled through a broker. Can be asynchronous, distributed, and cross-process. Modern systems like Kafka, RabbitMQ, and Spring Events implement Pub/Sub.

### Cascading Update Problem

If an observer changes the Subject's state during notification, the Subject fires another round of notifications. This can cascade infinitely. Solution: use a flag (`isNotifying`) that prevents re-entrant notifications, or queue state changes and apply them after the notification cycle completes.

### Event Sourcing Connection

Observer is related to Event Sourcing. Instead of notifying observers of state changes, store the events themselves as the system of record. Observers (projections) replay events to build their views. This gives time-travel debugging, audit logs, and eventual consistency for free.

---

## 12. Interview Questions and Model Answers

### Q1: Explain the Observer pattern and why it is important.

**Model Answer:**

The Observer pattern defines a one-to-many dependency so that when a Subject changes state, all registered Observers are notified automatically. It solves the problem of keeping multiple dependent objects consistent without coupling them together directly.

Its importance goes beyond the pattern itself: it is the conceptual foundation of every event-driven system ever built. Java Swing's ActionListener, Spring's ApplicationEvent, Kafka's consumer groups, JavaScript's DOM events, RxJava's Observable, React's state updates — all are Observer variations. Understanding Observer deeply means understanding why modern systems are built the way they are.

The core value it provides is the Open/Closed Principle in practice: you can add new behavior (new observers) to a system without modifying existing code (the Subject). This is what makes large systems maintainable over time.

---

### Q2: What is the difference between the Observer pattern and the Pub/Sub pattern?

**Model Answer:**

Both decouple event producers from consumers, but they differ in the degree of decoupling and the presence of an intermediary.

In **Observer**, the Subject maintains a direct list of Observer references. The Subject calls each Observer directly. They are in the same process and the Subject knows about the Observer interface. It is synchronous by default.

In **Pub/Sub**, publishers and subscribers have no knowledge of each other. A message broker (EventBus, Kafka, RabbitMQ) sits between them. Publishers write to a topic; subscribers read from topics. This allows:
- Cross-process and cross-machine communication
- Topic-based filtering (subscribers only get events they care about)
- Asynchronous, buffered delivery
- Temporal decoupling (publisher and subscriber do not need to be running simultaneously)

The distinction matters in system design interviews: Observer is used for in-process coordination (e.g., updating UI when model changes), while Pub/Sub is used for inter-service communication in microservices.

Spring's `ApplicationEventPublisher` is an in-process Pub/Sub that starts as Observer but allows async handlers via `@Async`. Kafka is a distributed Pub/Sub system.

---

### Q3: How would you handle a slow observer that blocks the notification thread?

**Model Answer:**

This is a real production concern. A slow database-writing observer or network-calling observer can block every other observer and the calling thread.

There are three approaches, each with trade-offs:

**Approach 1: Async thread pool per slow observer.** The subject dispatches to a thread pool for known-slow observers and notifies fast observers synchronously. This gives the best of both worlds but requires knowing which observers are slow at registration time.

```java
pool.submit(() -> slowObserver.update(event));
```

**Approach 2: Non-blocking event queue.** Instead of calling observers directly, the subject enqueues events to a bounded `BlockingQueue`. A consumer thread drains the queue and notifies observers. This provides natural backpressure — if the queue is full, the producer blocks or drops. This is essentially the actor model.

**Approach 3: RxJava / Project Reactor.** These libraries provide built-in schedulers, backpressure, and operators. The `observeOn(Schedulers.io())` operator transparently moves observer execution to a thread pool. This is the most sophisticated approach and handles backpressure, retries, and error handling declaratively.

The choice depends on the SLA: if observers must complete before the Subject's `setState` returns, use synchronous with an aggressive timeout. If eventual delivery is acceptable, use async with a queue.

---

### Q4: Describe how memory leaks occur with the Observer pattern and how to fix them.

**Model Answer:**

Memory leaks in Observer are one of the most common bugs in Java applications, especially in GUI frameworks.

The problem: a long-lived Subject holds strong references to all registered Observers. If an Observer falls out of scope but forgets to deregister, the Subject's list keeps it alive indefinitely. In Android (which had this problem extensively) and Swing, this means View/Activity objects — which hold references to large object graphs — never get garbage-collected.

**Example of the leak:**

```java
// Subject lives as long as the app
StockExchange exchange = ...; // long-lived singleton

// Observer is created for a short-lived operation
class TemporaryReport implements StockObserver {
    private final byte[] largeDataBuffer = new byte[10_000_000]; // 10MB
    public void onStockEvent(StockEvent e) { /* ... */ }
}

exchange.subscribe(new TemporaryReport()); // registered...
// TemporaryReport goes out of local scope
// But exchange still holds a reference — 10MB never freed
```

**Fixes:**

1. **WeakReference in the Subject**: Store observers as `WeakReference<Observer>`. When the observer is garbage-collected, the weak reference returns null and gets cleaned up automatically. Downside: observers must be held by the caller, which is counter-intuitive.

2. **Explicit deregistration in lifecycle methods**: In Spring, use `@PreDestroy`. In Swing, use `removeNotify()`. In Android, use `onStop()`/`onDestroy()`. This is the most reliable approach.

3. **Return a SubscriptionHandle / Disposable**: The `subscribe()` method returns a handle that the caller must call `.cancel()` on. This makes the lifecycle contract explicit and easy to audit. RxJava's `Disposable` and the `SubscriptionHandle` in Implementation 2 use this pattern.

4. **Event bus with automatic cleanup**: If using a Pub/Sub bus (Spring, Guava), the bus can scan for dead references periodically.

The `SubscriptionHandle` approach from Implementation 2 is the industry standard because it makes the cleanup contract visible in the code.

---

### Q5: How does Observer relate to the MVC pattern?

**Model Answer:**

Observer is the mechanism that makes MVC work. MVC separates an application into three layers, and their interaction is coordinated by Observer:

- **Model** is the Subject. It holds business state and logic.
- **View** is the Observer. Multiple Views can observe the same Model.
- **Controller** modifies the Model in response to user actions.

When the Controller calls `model.setData(...)`, the Model changes state and notifies all registered Views. Each View pulls or receives the new state and re-renders itself. The Model has no knowledge of which Views exist or how many there are.

This is why, in JavaFX or Swing, you can have a `TableView`, a `ChartView`, and a `FormView` all displaying the same data. When the underlying `ObservableList` (the Model) changes, all three update simultaneously — through Observer.

In React (JavaScript), this pattern is implemented through the component state system. In Angular, it is implemented through RxJS Observables. In Spring MVC, it appears as a lighter-weight version through `@ModelAttribute` and session-scoped beans.

The MVC connection is also why Observer is sometimes called the "MVC pattern's backbone" in GoF literature.

---

### Q6: When would you choose Mediator over Observer?

**Model Answer:**

Observer and Mediator both decouple objects, but they solve different problems with opposite communication topologies.

**Observer** is **decentralized**. The Subject broadcasts to many observers. Communication flows from one to many. Observers do not interact with each other — they each independently react to the Subject's state. Adding a new observer does not affect other observers. The coupling flows through a shared interface, not a central hub.

**Mediator** is **centralized**. Multiple components (Colleagues) communicate only through a single Mediator object. No component knows about any other component directly. The Mediator contains the coordination logic. Communication flows through the hub.

**Choose Observer when:**
- One thing changes, and multiple things need to react independently
- Reactions are independent of each other (each observer operates in isolation)
- The number of "interesting things" (subjects) is small and known
- Example: a stock price changing, and email, SMS, dashboard each react independently

**Choose Mediator when:**
- Multiple things need to interact with each other, and the interactions are complex
- The interaction logic itself needs to be centralized, auditable, and changeable
- Objects have bidirectional relationships that would create a web of dependencies
- Example: a flight booking UI where selecting a flight affects seat options, meal options, price display, and available add-ons — all of which affect each other

A chat room is the classic Mediator example: users do not message each other directly; they message the chat room (Mediator), which distributes to others. If you used Observer here, each user would be both Subject and Observer of every other user — O(n²) connections. Mediator reduces it to O(n).

---

### Q7: How would you implement the Observer pattern in a distributed system?

**Model Answer:**

In a distributed system, the in-process Observer pattern is replaced by a messaging infrastructure, but the conceptual model is identical.

**Key challenges that do not exist in-process:**
- **Network partitions**: a subscriber may be unreachable when an event fires
- **Message ordering**: events may arrive out of order (network delays, retries)
- **At-least-once vs exactly-once delivery**: most message brokers guarantee at-least-once, so subscribers must be idempotent
- **Backpressure**: a slow subscriber must not crash under a flood of events
- **Schema evolution**: event formats change over time; old subscribers must still work

**Architecture choices:**

1. **Message broker (Kafka, RabbitMQ, AWS SNS/SQS)**: Publisher writes to a topic; broker stores and delivers to subscribers. Provides durability, replay, and cross-language support. This is the production standard for microservices.

2. **Webhook / HTTP callbacks**: The Subject (service A) calls registered HTTP endpoints when events occur. Simpler but requires the subscriber to be up and expose an endpoint. No durability.

3. **Database polling with CDC (Change Data Capture)**: Debezium reads the database's change log and publishes events. Publishers write to the database naturally; Debezium acts as the bridge. Provides exactly-once delivery guarantees tied to the database transaction.

4. **Event sourcing**: Events are the primary data store. Subscribers replay the event log to build their state. This gives perfect at-least-once delivery and time-travel capabilities.

In a design interview, the answer is usually Kafka for high-throughput durable events, or a simple transactional outbox pattern (write event to DB in the same transaction, then relay with Debezium) for correctness-critical flows.

---

### Q8: How do you test a class that implements the Observer pattern?

**Model Answer:**

Testing Observer-based code requires testing both sides: the Subject (that it notifies correctly) and the Observer (that it reacts correctly).

**Testing the Subject (ConcreteSubject):**

Use mock observers to verify notification behavior without testing real observer implementations.

```java
@Test
void shouldNotifyAllObserversWhenPriceChanges() {
    StockExchange exchange = new StockExchange("NYSE");
    List<StockEvent> received = new ArrayList<>();

    // Register a lambda observer — captures events for assertion
    StockObserver capturer = new StockObserver() {
        @Override public void onStockEvent(StockEvent e) { received.add(e); }
        @Override public String getName() { return "capturer"; }
    };
    exchange.subscribe(capturer);

    exchange.updatePrice("AAPL", 185.0, 500_000);
    exchange.updatePrice("AAPL", 191.0, 700_000);

    assertEquals(2, received.size());
    assertEquals("AAPL", received.get(0).getSymbol());
    assertEquals(185.0, received.get(0).getCurrentPrice());
    assertEquals(191.0, received.get(1).getCurrentPrice());
}

@Test
void shouldNotNotifyUnsubscribedObserver() {
    StockExchange exchange = new StockExchange("NYSE");
    List<StockEvent> received = new ArrayList<>();
    StockObserver obs = new StockObserver() {
        @Override public void onStockEvent(StockEvent e) { received.add(e); }
        @Override public String getName() { return "obs"; }
    };

    exchange.subscribe(obs);
    exchange.updatePrice("AAPL", 185.0, 100_000);
    exchange.unsubscribe(obs);
    exchange.updatePrice("AAPL", 190.0, 200_000); // should not reach obs

    assertEquals(1, received.size()); // only first event
}

@Test
void shouldContinueNotifyingIfOneObserverThrows() {
    StockExchange exchange = new StockExchange("NYSE");
    List<String> notified = new ArrayList<>();

    StockObserver badObserver = new StockObserver() {
        @Override public void onStockEvent(StockEvent e) { throw new RuntimeException("crash!"); }
        @Override public String getName() { return "bad"; }
    };
    StockObserver goodObserver = new StockObserver() {
        @Override public void onStockEvent(StockEvent e) { notified.add("good notified"); }
        @Override public String getName() { return "good"; }
    };

    exchange.subscribe(badObserver);
    exchange.subscribe(goodObserver);

    assertDoesNotThrow(() -> exchange.updatePrice("AAPL", 185.0, 100_000));
    assertEquals(1, notified.size()); // good observer still got notified
}
```

**Testing the Observer (ConcreteObserver):**

Inject a mock or stub Subject, or call `onStockEvent()` directly with constructed events.

```java
@Test
void emailAlertShouldFireWhenThresholdExceeded() {
    // Arrange
    List<String> sentEmails = new ArrayList<>();
    EmailAlertObserver alert = new EmailAlertObserver("test@example.com", 3.0, "AAPL") {
        @Override
        protected void sendEmail(String to, String subject, String body) {
            sentEmails.add(subject); // intercept email sending
        }
    };

    // Act: price moves 5% — exceeds 3% threshold
    StockEvent event = new StockEvent("AAPL", 100.0, 105.0, 1000);
    alert.onStockEvent(event);

    // Assert
    assertEquals(1, sentEmails.size());
    assertTrue(sentEmails.get(0).contains("AAPL"));
}

@Test
void emailAlertShouldNotFireBelowThreshold() {
    List<String> sentEmails = new ArrayList<>();
    EmailAlertObserver alert = new EmailAlertObserver("test@example.com", 5.0, "AAPL") {
        @Override
        protected void sendEmail(String to, String subject, String body) {
            sentEmails.add(subject);
        }
    };

    StockEvent event = new StockEvent("AAPL", 100.0, 102.0, 1000); // only 2% move
    alert.onStockEvent(event);

    assertTrue(sentEmails.isEmpty());
}
```

**Key testing principles for Observer:**
- Test Subject and Observer independently — they should be testable in isolation
- Use lambda observers or anonymous classes as test doubles for Subject tests
- Test the failure case: bad observer should not crash notification chain
- Test registration/deregistration lifecycle
- For thread safety, write concurrent tests with `CountDownLatch` or `Phaser`

---

## 13. Comparison with Related Patterns

### Observer vs Mediator

| Dimension | Observer | Mediator |
|-----------|----------|----------|
| Communication direction | One Subject → Many Observers | Many Colleagues ↔ Mediator ↔ Many Colleagues |
| Coupling | Subject to Observer interface | Colleagues to Mediator interface |
| Knowledge | Subject does not know observers exist | Mediator knows all colleagues |
| Interaction complexity | Independent reactions | Complex bidirectional interactions |
| Adding behavior | Add new observer (easy) | Modify mediator (single point of change) |
| Example | Stock exchange notifying subscribers | Air traffic control, chat room |

**The mental model**: Observer is broadcast (one-to-many, unidirectional). Mediator is a switchboard (many-to-many, bidirectional through the hub).

A practical heuristic: if you find yourself writing observers that need to call back to other observers, or observers that modify shared state that other observers depend on, you have outgrown Observer — switch to Mediator.

### Observer vs Command

Observer is about **notification** (something happened). Command is about **encapsulating a request** (do something) as an object. They are often used together: an event fires an Observer notification, which creates a Command object and queues it for execution. Event sourcing stores Commands as events.

### Observer vs Strategy

Strategy replaces the algorithm a Subject uses. Observer adds behavior to the Subject's output. A Subject with a Strategy pattern uses a single algorithm chosen at construction or configuration time. A Subject with Observer can have many observers that all react to the same event differently.

### Observer vs Iterator

GoF explicitly notes the connection. Both define a traversal of a collection. Observer is "push" iteration: the collection (Subject) pushes each change to consumers. Iterator is "pull" iteration: the consumer pulls each item. RxJava's Observable is famously described as a "dual" of Iterable — one pushes, the other pulls, but both represent a sequence of values.

### Observer vs Decorator

Both add behavior without modifying the existing class. Observer adds behavior at the notification boundary (react to changes from the outside). Decorator adds behavior at the method call boundary (wrap and extend method calls from the inside). Decorators are invisible to callers; Observers are separate registered participants.

---

## 14. Memory Leak Issues and Fixes

Memory leaks in Observer are insidious because they do not crash the application — they slowly consume memory until an `OutOfMemoryError` occurs, often hours or days after the actual bug was introduced.

### 14.1 The Classic Leak Scenario

```java
// Subject: long-lived singleton
class StockExchange {
    // This list keeps observers alive as long as StockExchange lives
    private final List<StockObserver> observers = new CopyOnWriteArrayList<>();
    // ...
}

// Short-lived component (e.g., per-request, per-screen, per-session)
class TradeReport implements StockObserver {
    private final byte[] reportData = new byte[5_000_000]; // 5MB per report
    private final Map<String, Object> cache = new HashMap<>(); // can be large

    TradeReport(StockExchange exchange) {
        exchange.subscribe(this); // registers with long-lived subject
    }

    @Override
    public void onStockEvent(StockEvent event) { /* ... */ }

    // MISSING: no cleanup in a close() / dispose() method
}

// Usage — creates 1000 reports over the day, none are GC'd
for (int i = 0; i < 1000; i++) {
    TradeReport report = new TradeReport(exchange);
    // report goes out of scope but is kept alive by exchange.observers
    // 5MB * 1000 = 5GB heap consumed, OutOfMemoryError
}
```

### 14.2 Fix 1: WeakReference — Automatic Cleanup

Store observers as `WeakReference<Observer>`. When the observer is no longer referenced by the caller, it becomes eligible for GC. The Subject's list then contains dead weak references, which must be cleaned up on next notification.

```java
import java.lang.ref.WeakReference;
import java.util.Iterator;
import java.util.concurrent.CopyOnWriteArrayList;

class WeakReferenceSubject {
    private final CopyOnWriteArrayList<WeakReference<StockObserver>> weakObservers =
        new CopyOnWriteArrayList<>();

    public void subscribe(StockObserver observer) {
        // Wrap in weak reference — subject does not prevent GC
        weakObservers.add(new WeakReference<>(observer));
    }

    public void notifyObservers(StockEvent event) {
        Iterator<WeakReference<StockObserver>> iter = weakObservers.iterator();
        List<WeakReference<StockObserver>> toRemove = new ArrayList<>();

        while (iter.hasNext()) {
            WeakReference<StockObserver> ref = iter.next();
            StockObserver observer = ref.get(); // null if GC'd

            if (observer == null) {
                toRemove.add(ref); // mark dead reference for cleanup
            } else {
                try {
                    observer.onStockEvent(event);
                } catch (Exception e) {
                    System.err.println("Observer error: " + e.getMessage());
                }
            }
        }

        weakObservers.removeAll(toRemove); // clean up dead references
    }
}
```

**Critical caveat**: When using `WeakReference`, the observer object must be kept alive by the caller (stored in a field), not just passed to `subscribe()`. If the caller does this:

```java
// WRONG — observer is eligible for GC immediately after this line
exchange.subscribe(new EmailAlertObserver(...));
// GC may collect the observer before it ever receives an event!
```

The caller must hold a strong reference:

```java
// CORRECT — caller keeps the observer alive
private final EmailAlertObserver emailAlert = new EmailAlertObserver(...);

exchange.subscribe(emailAlert); // subscribe the field, not a local variable
```

This is the primary reason `WeakReference` in Observer is controversial — it trades memory leaks for incorrect silent behavior (observers that silently stop receiving events).

### 14.3 Fix 2: SubscriptionHandle Pattern — Explicit Lifecycle (Recommended)

The cleanest solution. `subscribe()` returns an opaque handle. The caller is responsible for calling `handle.cancel()` when done. This is what RxJava's `Disposable` and Reactor's `Disposable` implement.

```java
// Subject returns a handle
public interface Observable<T> {
    SubscriptionHandle subscribe(Observer<T> observer);
}

// Caller manages the handle
public class TradeReport implements AutoCloseable {
    private final SubscriptionHandle subscriptionHandle;
    private final byte[] reportData = new byte[5_000_000];

    TradeReport(StockMarket market) {
        // subscribe() returns a handle — we must store it
        this.subscriptionHandle = market.subscribe(this::onStockEvent);
    }

    private void onStockEvent(StockEvent event) { /* ... */ }

    @Override
    public void close() {
        // Cancel the subscription — subject removes its reference
        subscriptionHandle.cancel();
        // Now this object is eligible for GC
    }
}

// Usage with try-with-resources
try (TradeReport report = new TradeReport(exchange)) {
    report.generateReport();
} // close() is called automatically — subscription cancelled, memory freed
```

### 14.4 Fix 3: Spring @PreDestroy / Lifecycle Integration

In Spring-managed beans, use the `@PreDestroy` annotation to ensure cleanup when the bean is destroyed.

```java
@Component
@Scope("prototype") // prototype scope = new instance per injection point
public class TradeReportBean implements StockObserver {
    private SubscriptionHandle handle;
    private final StockMarket market;

    public TradeReportBean(StockMarket market) {
        this.market = market;
    }

    @PostConstruct
    public void init() {
        this.handle = market.subscribe(this);
        System.out.println("TradeReportBean: subscribed");
    }

    @PreDestroy
    public void cleanup() {
        if (handle != null) {
            handle.cancel(); // guaranteed to run before bean is destroyed
        }
        System.out.println("TradeReportBean: unsubscribed");
    }

    @Override
    public void onStockEvent(StockEvent event) { /* ... */ }

    @Override
    public String getName() { return "TradeReportBean"; }
}
```

### 14.5 Fix 4: Time-Based Expiry

For scenarios where explicit deregistration is impractical, the Subject can automatically expire observers after a period of inactivity.

```java
class TimedSubscription {
    final StockObserver observer;
    final Instant expiresAt;

    TimedSubscription(StockObserver observer, Duration ttl) {
        this.observer = observer;
        this.expiresAt = Instant.now().plus(ttl);
    }

    boolean isExpired() {
        return Instant.now().isAfter(expiresAt);
    }
}

class ExpiringStockExchange {
    private final CopyOnWriteArrayList<TimedSubscription> subscriptions =
        new CopyOnWriteArrayList<>();

    public void subscribe(StockObserver observer, Duration ttl) {
        subscriptions.add(new TimedSubscription(observer, ttl));
    }

    private void notifyObservers(StockEvent event) {
        // Remove expired subscriptions first
        subscriptions.removeIf(TimedSubscription::isExpired);

        for (TimedSubscription sub : subscriptions) {
            try {
                sub.observer.onStockEvent(event);
            } catch (Exception e) {
                System.err.println("Observer error: " + e.getMessage());
            }
        }
    }
}
```

### 14.6 Memory Leak Detection Strategies

In production, use these approaches to detect Observer leaks:

1. **Heap dump analysis**: Take a heap dump with `jmap -dump:format=b,file=heap.hprof <pid>` and analyze with Eclipse Memory Analyzer (MAT). Look for large `CopyOnWriteArrayList` instances with unexpectedly high retention counts.

2. **JMX / Actuator metrics**: Expose observer count as a JMX MBean or Spring Actuator metric. Alert when the count grows unboundedly.

3. **WeakHashMap for weak-keyed observer maps**: If observers are keyed by topic/symbol, a `WeakHashMap<Observer, Topic>` automatically removes entries when the observer key is GC'd.

4. **Load testing with heap monitoring**: Run a load test that creates and discards many observers. Monitor heap with VisualVM or JConsole. If heap grows monotonically without leveling off, there is a leak.

### 14.7 Summary of Memory Leak Fixes

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| WeakReference | Automatic GC cleanup | Silent failures if caller doesn't hold ref | Internal framework use |
| SubscriptionHandle | Explicit, auditable | Caller must call cancel() | APIs you control |
| @PreDestroy / Lifecycle | Spring-native, automatic | Spring beans only | Spring apps |
| Time-based expiry | No caller cooperation needed | Observers stop working silently on expiry | Temporary subscriptions |
| AutoCloseable + try-with-resources | Compiler-enforced | Verbose, requires try block | Short-lived operations |

The **SubscriptionHandle** pattern (Fix 2) is the recommended default for any new Java code. It makes the lifecycle contract explicit, catches leaks at code review time, and works in any framework.

---

*This document is part of the LLD & OOP Design Patterns course. For related patterns, see:*
- *01-strategy.md — replacing algorithms*
- *03-command.md — encapsulating requests*
- *04-mediator.md — centralized coordination (complement to Observer)*
- *05-chain-of-responsibility.md — sequential processing pipelines*
