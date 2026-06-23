# Facade Pattern

> "Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use."
> — Gang of Four, *Design Patterns: Elements of Reusable Object-Oriented Software*

---

## Table of Contents

1. [Intent](#1-intent)
2. [When to Use / When NOT to Use](#2-when-to-use--when-not-to-use)
3. [UML Class Diagram](#3-uml-class-diagram)
4. [Complete Java Implementations](#4-complete-java-implementations)
   - 4.1 [Home Theater System](#41-home-theater-system)
   - 4.2 [E-Commerce Checkout](#42-e-commerce-checkout)
   - 4.3 [Compiler Facade](#43-compiler-facade)
5. [Real-World Analogy](#5-real-world-analogy)
6. [Real-World Java Ecosystem Examples](#6-real-world-java-ecosystem-examples)
7. [Facade vs Mediator](#7-facade-vs-mediator)
8. [Facade vs Adapter](#8-facade-vs-adapter)
9. [API Gateway as Facade in Microservices](#9-api-gateway-as-facade-in-microservices)
10. [Trade-offs](#10-trade-offs)
11. [Common Interview Discussion Points](#11-common-interview-discussion-points)
12. [Interview Questions with Model Answers](#12-interview-questions-with-model-answers)

---

## 1. Intent

The Facade pattern provides a **simplified, unified interface** to a complex subsystem. It does not hide the subsystem — the subsystem remains directly accessible for advanced clients who need fine-grained control. The Facade simply makes the common path easy.

### The Core Insight

Complex systems are built from many collaborating components. Most clients do not need to understand all those components — they need to accomplish a task. The Facade captures the most common workflow and presents it as a single, coherent interface.

**What the Facade does:**

- Reduces the number of objects a client must interact with
- Decouples the client from the internal structure of the subsystem
- Makes the subsystem easier to use without restricting what the subsystem can do
- Provides a stable public API over an unstable internal structure

**What the Facade does NOT do:**

- It does not prevent access to the subsystem classes (unlike Encapsulation, which actively restricts access)
- It does not mediate communication between subsystem components
- It does not add new behavior or business logic to the subsystem
- It does not replace the subsystem — it wraps it

### Simplifying Without Hiding

This distinction is critical. A common mistake is to build a Facade that completely hides the subsystem, preventing clients from doing anything the Facade does not expose. That turns the Facade into a bottleneck and an anti-pattern. The correct approach is:

```
Client A  ──→  Facade  ──→  SubsystemA
                        ──→  SubsystemB
                        ──→  SubsystemC

Client B  ─────────────→  SubsystemA  (bypasses Facade for advanced use)
```

Client B is a power user. It should be able to bypass the Facade and call subsystem components directly. The Facade serves Client A — the common case.

---

## 2. When to Use / When NOT to Use

### When to Use

**You want to provide a simple interface to a complex body of code.**
When a subsystem has grown complex through evolution or by design (e.g., a graphics engine, a compiler, an ORM), clients should not need to understand every detail. A Facade gives them a simple on-ramp.

**You are designing a layered architecture.**
In a layered system, each layer should communicate with the layer below only through a well-defined surface. A Facade on each layer's boundary keeps layers loosely coupled. The classic example: the service layer in a Spring application is a Facade over the repository and domain layers.

**You want to decouple client code from a poorly-designed or legacy subsystem.**
When you must integrate with a messy legacy library, wrapping it in a Facade lets you isolate the ugliness behind a clean interface. Clients only see your clean API. When you replace the legacy library, only the Facade changes.

**You have many clients performing the same sequence of steps against a subsystem.**
If every client does `init() → configure() → execute() → teardown()` in the same order, extract that workflow into a Facade. Centralizing the workflow prevents drift and bugs.

**You are building an SDK or library for external consumption.**
Library authors routinely expose a high-level Facade (e.g., `Hibernate.buildSessionFactory()`) while also giving advanced users direct access to the lower-level API. This is the canonical use of the pattern.

### When NOT to Use

**When the subsystem is simple to begin with.**
Do not wrap a single-class service in a Facade just to introduce a pattern. Every abstraction has a cost (indirection, documentation, maintenance). If there is nothing complex to hide, the Facade is noise.

**When all clients need full, fine-grained control of the subsystem.**
If every client is a "power user" who needs to configure every knob, a Facade just gets in the way. You end up with a pass-through object that adds no value.

**When the Facade would become a God Object.**
If the Facade must aggregate twenty subsystems and expose fifty methods, it has stopped being a Facade and has become a monolith entry point. Break it into multiple, more focused facades.

**When you need cross-cutting coordination between subsystem components.**
If the coordination logic between subsystems is complex, conditional, or event-driven, consider the Mediator pattern instead. A Facade should contain workflow, not branching orchestration logic.

**When you are tempted to block access to the subsystem.**
Never make subsystem classes package-private just to force clients through the Facade. Doing so turns the Facade into a gatekeeping anti-pattern that prevents legitimate advanced use cases.

---

## 3. UML Class Diagram

### General Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                          Client                                 │
│                                                                 │
│   facade.doSomething()                                          │
└──────────────────────────────┬──────────────────────────────────┘
                               │ uses
                               ▼
              ┌────────────────────────────────┐
              │           <<Facade>>           │
              │         SystemFacade           │
              │                                │
              │  - subsystemA: SubsystemA      │
              │  - subsystemB: SubsystemB      │
              │  - subsystemC: SubsystemC      │
              │                                │
              │  + doSomething(): void         │
              │  + doSomethingElse(): Result   │
              └────┬──────────┬──────────┬─────┘
                   │          │          │
           creates │  creates │  creates │
                   ▼          ▼          ▼
        ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
        │ SubsystemA   │ │ SubsystemB   │ │ SubsystemC   │
        │              │ │              │ │              │
        │ + op1()      │ │ + op2()      │ │ + op3()      │
        │ + op2()      │ │ + op3()      │ │ + op4()      │
        └──────────────┘ └──────────────┘ └──────────────┘
               ▲                                  ▲
               │                                  │
               └──────── Advanced Client ─────────┘
                    (bypasses Facade directly)
```

### Home Theater System (Concrete Example)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         HomeTheaterClient                           │
│   facade.watchMovie("Inception")                                    │
│   facade.endMovie()                                                 │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
         ┌────────────────────────────────────────┐
         │         HomeTheaterFacade              │
         │                                        │
         │ - tv        : Television               │
         │ - sound     : SoundSystem              │
         │ - dvd       : DVDPlayer                │
         │ - lights    : AmbientLights            │
         │ - projector : Projector                │
         │                                        │
         │ + watchMovie(title: String): void      │
         │ + endMovie(): void                     │
         │ + listenToMusic(source: String): void  │
         └──┬────────┬────────┬────────┬────────┬─┘
            │        │        │        │        │
            ▼        ▼        ▼        ▼        ▼
     ┌────────┐ ┌─────────┐ ┌───────┐ ┌───────┐ ┌──────────┐
     │  TV    │ │ Sound   │ │  DVD  │ │Lights │ │Projector │
     │        │ │ System  │ │Player │ │       │ │          │
     │+on()   │ │+on()    │ │+on()  │ │+dim() │ │+on()     │
     │+off()  │ │+off()   │ │+off() │ │+on()  │ │+off()    │
     │+setIn()│ │+setVol()│ │+play()│ │       │ │+setMode()│
     └────────┘ └─────────┘ └───────┘ └───────┘ └──────────┘
```

---

## 4. Complete Java Implementations

### 4.1 Home Theater System

This example demonstrates the classic Facade use case: a home theater system with five independent subsystems. A client who wants to watch a movie must coordinate all five — unless a Facade is provided.

#### Subsystem Classes

```java
package facade.theater;

/**
 * Subsystem component: controls the television display.
 * Can be used standalone or via the HomeTheaterFacade.
 */
public class Television {

    private final String model;
    private boolean on;
    private String inputSource;

    public Television(String model) {
        this.model = model;
        this.on = false;
    }

    public void on() {
        on = true;
        System.out.println("[TV:" + model + "] Powered on.");
    }

    public void off() {
        on = false;
        System.out.println("[TV:" + model + "] Powered off.");
    }

    public void setInputSource(String source) {
        this.inputSource = source;
        System.out.println("[TV:" + model + "] Input source set to: " + source);
    }

    public boolean isOn() {
        return on;
    }

    public String getInputSource() {
        return inputSource;
    }
}
```

```java
package facade.theater;

/**
 * Subsystem component: controls the surround sound system.
 */
public class SoundSystem {

    private final String brand;
    private boolean on;
    private int volume;
    private String surroundMode;

    public SoundSystem(String brand) {
        this.brand = brand;
        this.volume = 0;
    }

    public void on() {
        on = true;
        System.out.println("[Sound:" + brand + "] Powered on.");
    }

    public void off() {
        on = false;
        System.out.println("[Sound:" + brand + "] Powered off.");
    }

    public void setVolume(int level) {
        if (level < 0 || level > 100) {
            throw new IllegalArgumentException("Volume must be between 0 and 100, got: " + level);
        }
        this.volume = level;
        System.out.println("[Sound:" + brand + "] Volume set to: " + level);
    }

    public void setSurroundMode(String mode) {
        this.surroundMode = mode;
        System.out.println("[Sound:" + brand + "] Surround mode: " + mode);
    }

    public int getVolume() {
        return volume;
    }
}
```

```java
package facade.theater;

/**
 * Subsystem component: controls the Blu-ray / DVD player.
 */
public class DVDPlayer {

    private final String model;
    private boolean on;
    private String currentDisc;

    public DVDPlayer(String model) {
        this.model = model;
    }

    public void on() {
        on = true;
        System.out.println("[DVD:" + model + "] Powered on.");
    }

    public void off() {
        on = false;
        System.out.println("[DVD:" + model + "] Powered off.");
    }

    public void play(String title) {
        if (!on) {
            throw new IllegalStateException("DVD player is off. Call on() first.");
        }
        this.currentDisc = title;
        System.out.println("[DVD:" + model + "] Playing: " + title);
    }

    public void stop() {
        System.out.println("[DVD:" + model + "] Stopped playback of: " + currentDisc);
        this.currentDisc = null;
    }

    public void eject() {
        System.out.println("[DVD:" + model + "] Ejecting disc.");
        this.currentDisc = null;
    }

    public String getCurrentDisc() {
        return currentDisc;
    }
}
```

```java
package facade.theater;

/**
 * Subsystem component: controls ambient lighting.
 */
public class AmbientLights {

    private int brightnessPercent;

    public AmbientLights() {
        this.brightnessPercent = 100;
    }

    public void dim(int percent) {
        if (percent < 0 || percent > 100) {
            throw new IllegalArgumentException("Brightness must be 0-100, got: " + percent);
        }
        this.brightnessPercent = percent;
        System.out.println("[Lights] Dimmed to: " + percent + "%");
    }

    public void on() {
        this.brightnessPercent = 100;
        System.out.println("[Lights] Full brightness.");
    }

    public void off() {
        this.brightnessPercent = 0;
        System.out.println("[Lights] Lights off.");
    }

    public int getBrightness() {
        return brightnessPercent;
    }
}
```

```java
package facade.theater;

/**
 * Subsystem component: controls the projector.
 */
public class Projector {

    private final String brand;
    private boolean on;
    private String displayMode; // "widescreen", "4:3", "zoom"

    public Projector(String brand) {
        this.brand = brand;
    }

    public void on() {
        on = true;
        System.out.println("[Projector:" + brand + "] Warming up...");
    }

    public void off() {
        on = false;
        System.out.println("[Projector:" + brand + "] Cooling down...");
    }

    public void setDisplayMode(String mode) {
        this.displayMode = mode;
        System.out.println("[Projector:" + brand + "] Display mode: " + mode);
    }

    public boolean isOn() {
        return on;
    }
}
```

#### The Facade

```java
package facade.theater;

/**
 * Facade for the home theater subsystem.
 *
 * <p>Provides simplified operations for common home theater workflows:
 * watching a movie, ending a movie, and listening to music. Advanced
 * users can still obtain references to individual subsystem components
 * via the package-level getters if they need fine-grained control.
 *
 * <p>This class intentionally does NOT make the subsystem components
 * private to the package — they remain directly instantiable and usable
 * without the Facade by any client that needs lower-level access.
 */
public class HomeTheaterFacade {

    private final Television tv;
    private final SoundSystem soundSystem;
    private final DVDPlayer dvdPlayer;
    private final AmbientLights lights;
    private final Projector projector;

    /**
     * Constructs the Facade with pre-existing subsystem components.
     * Use this constructor when the caller manages subsystem lifecycles
     * (e.g., in a Spring context with dependency injection).
     */
    public HomeTheaterFacade(Television tv,
                              SoundSystem soundSystem,
                              DVDPlayer dvdPlayer,
                              AmbientLights lights,
                              Projector projector) {
        this.tv = tv;
        this.soundSystem = soundSystem;
        this.dvdPlayer = dvdPlayer;
        this.lights = lights;
        this.projector = projector;
    }

    /**
     * Convenience factory method that creates default subsystem components.
     * Suitable for standalone use without a DI framework.
     */
    public static HomeTheaterFacade createDefault() {
        return new HomeTheaterFacade(
            new Television("Samsung QLED 75\""),
            new SoundSystem("Sonos Arc"),
            new DVDPlayer("Sony UBP-X800"),
            new AmbientLights(),
            new Projector("Epson 4K Pro")
        );
    }

    /**
     * Prepares the entire home theater to watch a movie.
     *
     * <p>Coordinates: lights → projector → sound → DVD → TV in the
     * correct sequence for an optimal viewing experience.
     *
     * @param title the movie title to play
     * @throws IllegalStateException if any subsystem fails to initialize
     */
    public void watchMovie(String title) {
        System.out.println("\n--- Get ready to watch: " + title + " ---");
        lights.dim(10);
        projector.on();
        projector.setDisplayMode("widescreen");
        soundSystem.on();
        soundSystem.setSurroundMode("Dolby Atmos");
        soundSystem.setVolume(40);
        dvdPlayer.on();
        dvdPlayer.play(title);
        tv.on();
        tv.setInputSource("HDMI-1");
        System.out.println("--- Enjoy your movie! ---\n");
    }

    /**
     * Shuts down the home theater after a movie.
     * Reverses the startup sequence to properly cool down devices.
     */
    public void endMovie() {
        System.out.println("\n--- Shutting down home theater ---");
        dvdPlayer.stop();
        dvdPlayer.off();
        soundSystem.off();
        projector.off();
        tv.off();
        lights.on();
        System.out.println("--- Goodnight! ---\n");
    }

    /**
     * Configures the system for music listening without video output.
     *
     * @param source the audio source (e.g., "Spotify", "Bluetooth", "AUX")
     */
    public void listenToMusic(String source) {
        System.out.println("\n--- Setting up for music: " + source + " ---");
        lights.dim(50);
        soundSystem.on();
        soundSystem.setSurroundMode("Stereo");
        soundSystem.setVolume(30);
        // TV and projector are intentionally left off for music-only mode
        System.out.println("--- Enjoy your music! ---\n");
    }

    // --- Accessors for advanced clients who need subsystem-level control ---

    /** Returns the TV component for direct control. */
    public Television getTelevision() { return tv; }

    /** Returns the sound system for direct control. */
    public SoundSystem getSoundSystem() { return soundSystem; }

    /** Returns the DVD player for direct control. */
    public DVDPlayer getDvdPlayer() { return dvdPlayer; }
}
```

#### Client Code

```java
package facade.theater;

/**
 * Demonstrates the Facade in use.
 * The client only needs to know about HomeTheaterFacade.
 */
public class HomeTheaterClient {

    public static void main(String[] args) {
        HomeTheaterFacade homeTheater = HomeTheaterFacade.createDefault();

        // Simple workflow through the Facade
        homeTheater.watchMovie("Inception");
        homeTheater.endMovie();

        // Advanced client uses subsystem directly for fine-grained control
        homeTheater.getSoundSystem().setVolume(55); // Override volume
        homeTheater.listenToMusic("Spotify");
    }
}
```

**Sample Output:**

```
--- Get ready to watch: Inception ---
[Lights] Dimmed to: 10%
[Projector:Epson 4K Pro] Warming up...
[Projector:Epson 4K Pro] Display mode: widescreen
[Sound:Sonos Arc] Powered on.
[Sound:Sonos Arc] Surround mode: Dolby Atmos
[Sound:Sonos Arc] Volume set to: 40
[DVD:Sony UBP-X800] Powered on.
[DVD:Sony UBP-X800] Playing: Inception
[TV:Samsung QLED 75"] Powered on.
[TV:Samsung QLED 75"] Input source set to: HDMI-1
--- Enjoy your movie! ---
```

---

### 4.2 E-Commerce Checkout Facade

A production-grade e-commerce checkout involves at least five independent services. Without a Facade, every checkout entry point (web, mobile, API) must duplicate the coordination logic.

#### Domain Objects

```java
package facade.ecommerce;

import java.math.BigDecimal;
import java.util.List;

/** Represents a customer's purchase order. */
public class Order {

    private final String orderId;
    private final String customerId;
    private final List<OrderItem> items;
    private final String shippingAddress;
    private OrderStatus status;
    private BigDecimal totalAmount;

    public Order(String orderId, String customerId,
                 List<OrderItem> items, String shippingAddress) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.items = List.copyOf(items);
        this.shippingAddress = shippingAddress;
        this.status = OrderStatus.PENDING;
    }

    public String getOrderId() { return orderId; }
    public String getCustomerId() { return customerId; }
    public List<OrderItem> getItems() { return items; }
    public String getShippingAddress() { return shippingAddress; }
    public OrderStatus getStatus() { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }

    public void setStatus(OrderStatus status) { this.status = status; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }

    public enum OrderStatus {
        PENDING, CONFIRMED, PAYMENT_FAILED, SHIPPED, DELIVERED, CANCELLED
    }
}
```

```java
package facade.ecommerce;

import java.math.BigDecimal;

/** A single line item within an order. */
public record OrderItem(String productId, String productName,
                        int quantity, BigDecimal unitPrice) {

    public BigDecimal lineTotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```

```java
package facade.ecommerce;

import java.math.BigDecimal;

/** Encapsulates the result of a checkout operation. */
public class CheckoutResult {

    private final boolean success;
    private final String orderId;
    private final String trackingNumber;
    private final String failureReason;
    private final BigDecimal loyaltyPointsEarned;

    private CheckoutResult(Builder builder) {
        this.success = builder.success;
        this.orderId = builder.orderId;
        this.trackingNumber = builder.trackingNumber;
        this.failureReason = builder.failureReason;
        this.loyaltyPointsEarned = builder.loyaltyPointsEarned;
    }

    public boolean isSuccess() { return success; }
    public String getOrderId() { return orderId; }
    public String getTrackingNumber() { return trackingNumber; }
    public String getFailureReason() { return failureReason; }
    public BigDecimal getLoyaltyPointsEarned() { return loyaltyPointsEarned; }

    @Override
    public String toString() {
        if (success) {
            return "CheckoutResult{success=true, orderId='" + orderId
                + "', tracking='" + trackingNumber
                + "', loyaltyPoints=" + loyaltyPointsEarned + "}";
        }
        return "CheckoutResult{success=false, reason='" + failureReason + "'}";
    }

    public static Builder successBuilder(String orderId) {
        return new Builder(true, orderId);
    }

    public static Builder failureBuilder(String orderId, String reason) {
        return new Builder(false, orderId).withFailureReason(reason);
    }

    public static class Builder {
        private final boolean success;
        private final String orderId;
        private String trackingNumber;
        private String failureReason;
        private BigDecimal loyaltyPointsEarned = BigDecimal.ZERO;

        Builder(boolean success, String orderId) {
            this.success = success;
            this.orderId = orderId;
        }

        public Builder withTrackingNumber(String trackingNumber) {
            this.trackingNumber = trackingNumber;
            return this;
        }

        public Builder withFailureReason(String reason) {
            this.failureReason = reason;
            return this;
        }

        public Builder withLoyaltyPoints(BigDecimal points) {
            this.loyaltyPointsEarned = points;
            return this;
        }

        public CheckoutResult build() {
            return new CheckoutResult(this);
        }
    }
}
```

#### Subsystem Service Interfaces and Implementations

```java
package facade.ecommerce;

import java.util.List;

/**
 * Subsystem: manages product inventory.
 */
public interface InventoryService {

    /**
     * Checks whether all items in the order are available in sufficient quantity.
     *
     * @param items the list of order items to check
     * @return true if all items are available
     */
    boolean checkAvailability(List<OrderItem> items);

    /**
     * Reserves inventory for the given items, preventing overselling.
     *
     * @param orderId the order ID to associate with the reservation
     * @param items   the items to reserve
     */
    void reserveInventory(String orderId, List<OrderItem> items);

    /**
     * Releases a previously held inventory reservation.
     * Called on payment failure or order cancellation.
     *
     * @param orderId the order whose reservation should be released
     */
    void releaseReservation(String orderId);
}
```

```java
package facade.ecommerce;

import java.math.BigDecimal;

/**
 * Subsystem: handles payment processing.
 */
public interface PaymentService {

    /**
     * Charges the customer for the given amount.
     *
     * @param customerId    the customer being charged
     * @param amount        the amount to charge
     * @param paymentMethod the payment method token
     * @return a payment transaction ID on success
     * @throws PaymentException if the charge fails
     */
    String charge(String customerId, BigDecimal amount, String paymentMethod)
        throws PaymentException;

    /**
     * Refunds a previously charged transaction.
     *
     * @param transactionId the transaction to refund
     * @param amount        the amount to refund
     */
    void refund(String transactionId, BigDecimal amount) throws PaymentException;
}
```

```java
package facade.ecommerce;

/** Thrown when a payment operation fails. */
public class PaymentException extends Exception {

    private final String errorCode;

    public PaymentException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}
```

```java
package facade.ecommerce;

/**
 * Subsystem: creates shipments and returns tracking numbers.
 */
public interface ShippingService {

    /**
     * Creates a shipping label and schedules pickup for the order.
     *
     * @param order the order to ship
     * @return a tracking number
     */
    String createShipment(Order order);

    /**
     * Estimates the delivery date for the given order.
     *
     * @param shippingAddress the destination address
     * @return estimated delivery in ISO-8601 date format
     */
    String estimateDelivery(String shippingAddress);
}
```

```java
package facade.ecommerce;

/**
 * Subsystem: sends customer notifications via email/SMS/push.
 */
public interface NotificationService {

    /**
     * Sends an order confirmation to the customer.
     *
     * @param customerId    the recipient
     * @param orderId       the confirmed order
     * @param trackingNumber the assigned tracking number
     */
    void sendOrderConfirmation(String customerId, String orderId, String trackingNumber);

    /**
     * Notifies the customer that their payment failed.
     *
     * @param customerId the recipient
     * @param orderId    the failed order
     * @param reason     a human-readable failure reason
     */
    void sendPaymentFailedNotification(String customerId, String orderId, String reason);
}
```

```java
package facade.ecommerce;

import java.math.BigDecimal;

/**
 * Subsystem: manages customer loyalty points.
 */
public interface LoyaltyService {

    /**
     * Awards points to a customer based on their purchase.
     *
     * @param customerId  the customer to award
     * @param orderAmount the order total used to calculate points
     * @return the number of points awarded
     */
    BigDecimal awardPoints(String customerId, BigDecimal orderAmount);

    /**
     * Returns the current point balance for a customer.
     *
     * @param customerId the customer whose balance to fetch
     * @return current point balance
     */
    BigDecimal getBalance(String customerId);
}
```

#### Stub Implementations (for demonstration)

```java
package facade.ecommerce;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

/** Simple in-memory stub implementations for demonstration. */
public class ServiceStubs {

    public static InventoryService inventoryService() {
        return new InventoryService() {
            @Override
            public boolean checkAvailability(List<OrderItem> items) {
                System.out.println("[Inventory] Checking availability for "
                    + items.size() + " items...");
                return true; // Always available in this stub
            }

            @Override
            public void reserveInventory(String orderId, List<OrderItem> items) {
                System.out.println("[Inventory] Reserved inventory for order: " + orderId);
            }

            @Override
            public void releaseReservation(String orderId) {
                System.out.println("[Inventory] Released reservation for order: " + orderId);
            }
        };
    }

    public static PaymentService paymentService() {
        return new PaymentService() {
            @Override
            public String charge(String customerId, BigDecimal amount, String paymentMethod)
                throws PaymentException {
                System.out.println("[Payment] Charging $" + amount
                    + " to customer: " + customerId);
                return "TXN-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
            }

            @Override
            public void refund(String transactionId, BigDecimal amount) throws PaymentException {
                System.out.println("[Payment] Refunding $" + amount
                    + " for transaction: " + transactionId);
            }
        };
    }

    public static ShippingService shippingService() {
        return new ShippingService() {
            @Override
            public String createShipment(Order order) {
                System.out.println("[Shipping] Creating shipment for order: "
                    + order.getOrderId());
                return "TRACK-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
            }

            @Override
            public String estimateDelivery(String shippingAddress) {
                return "2026-07-01"; // Fixed stub date
            }
        };
    }

    public static NotificationService notificationService() {
        return new NotificationService() {
            @Override
            public void sendOrderConfirmation(String customerId, String orderId,
                                              String trackingNumber) {
                System.out.println("[Notification] Order confirmation sent to: "
                    + customerId + " | Tracking: " + trackingNumber);
            }

            @Override
            public void sendPaymentFailedNotification(String customerId, String orderId,
                                                      String reason) {
                System.out.println("[Notification] Payment failure email sent to: "
                    + customerId + " | Reason: " + reason);
            }
        };
    }

    public static LoyaltyService loyaltyService() {
        return new LoyaltyService() {
            @Override
            public BigDecimal awardPoints(String customerId, BigDecimal orderAmount) {
                BigDecimal points = orderAmount.multiply(BigDecimal.valueOf(0.01))
                    .setScale(0, java.math.RoundingMode.FLOOR);
                System.out.println("[Loyalty] Awarded " + points
                    + " points to: " + customerId);
                return points;
            }

            @Override
            public BigDecimal getBalance(String customerId) {
                return BigDecimal.valueOf(500); // Stub balance
            }
        };
    }
}
```

#### The Checkout Facade

```java
package facade.ecommerce;

import java.math.BigDecimal;
import java.util.Objects;
import java.util.logging.Logger;

/**
 * Facade for the e-commerce checkout process.
 *
 * <p>Orchestrates five independent subsystems — Inventory, Payment, Shipping,
 * Notification, and Loyalty — into a single transactional checkout workflow.
 * Clients (REST controllers, CLI tools, batch jobs) call {@link #checkout}
 * without needing to know the internal coordination sequence.
 *
 * <p>The Facade also handles compensation: if payment fails after inventory
 * was reserved, the reservation is automatically released. This prevents
 * the kind of partial-state bugs that inevitably arise when clients must
 * manage the coordination themselves.
 *
 * <p>Design note: All five subsystem interfaces are injected rather than
 * constructed internally. This makes the Facade testable (mocks can be
 * injected) and compatible with Spring dependency injection.
 */
public class CheckoutFacade {

    private static final Logger log = Logger.getLogger(CheckoutFacade.class.getName());

    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    private final NotificationService notificationService;
    private final LoyaltyService loyaltyService;

    public CheckoutFacade(InventoryService inventoryService,
                          PaymentService paymentService,
                          ShippingService shippingService,
                          NotificationService notificationService,
                          LoyaltyService loyaltyService) {
        this.inventoryService = Objects.requireNonNull(inventoryService);
        this.paymentService = Objects.requireNonNull(paymentService);
        this.shippingService = Objects.requireNonNull(shippingService);
        this.notificationService = Objects.requireNonNull(notificationService);
        this.loyaltyService = Objects.requireNonNull(loyaltyService);
    }

    /**
     * Executes the complete checkout workflow for a given order.
     *
     * <p>The workflow:
     * <ol>
     *   <li>Validate input
     *   <li>Check and reserve inventory
     *   <li>Calculate order total
     *   <li>Charge the customer (with compensation on failure)
     *   <li>Create shipment
     *   <li>Award loyalty points
     *   <li>Send confirmation notification
     * </ol>
     *
     * @param order         the order to process
     * @param paymentMethod the payment method token (e.g., Stripe token)
     * @return a {@link CheckoutResult} indicating success or failure
     */
    public CheckoutResult checkout(Order order, String paymentMethod) {
        Objects.requireNonNull(order, "order must not be null");
        Objects.requireNonNull(paymentMethod, "paymentMethod must not be null");

        log.info("Starting checkout for order: " + order.getOrderId());

        // Step 1: Check inventory availability
        if (!inventoryService.checkAvailability(order.getItems())) {
            String reason = "One or more items are out of stock";
            notificationService.sendPaymentFailedNotification(
                order.getCustomerId(), order.getOrderId(), reason);
            return CheckoutResult.failureBuilder(order.getOrderId(), reason).build();
        }

        // Step 2: Reserve inventory (prevent overselling by concurrent checkouts)
        inventoryService.reserveInventory(order.getOrderId(), order.getItems());

        // Step 3: Calculate total
        BigDecimal total = order.getItems().stream()
            .map(OrderItem::lineTotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        order.setTotalAmount(total);

        // Step 4: Process payment — release reservation on failure
        String transactionId;
        try {
            transactionId = paymentService.charge(
                order.getCustomerId(), total, paymentMethod);
        } catch (PaymentException e) {
            log.warning("Payment failed for order " + order.getOrderId()
                + ": " + e.getMessage());
            // Compensating action: release the inventory reservation
            inventoryService.releaseReservation(order.getOrderId());
            order.setStatus(Order.OrderStatus.PAYMENT_FAILED);

            notificationService.sendPaymentFailedNotification(
                order.getCustomerId(), order.getOrderId(), e.getMessage());

            return CheckoutResult.failureBuilder(order.getOrderId(), e.getMessage()).build();
        }

        // Step 5: Create shipment
        String trackingNumber = shippingService.createShipment(order);
        order.setStatus(Order.OrderStatus.SHIPPED);

        // Step 6: Award loyalty points (non-critical — failure here is logged, not fatal)
        BigDecimal pointsEarned = BigDecimal.ZERO;
        try {
            pointsEarned = loyaltyService.awardPoints(order.getCustomerId(), total);
        } catch (Exception e) {
            // Loyalty is a best-effort service. A failure here must NOT fail the checkout.
            log.warning("Failed to award loyalty points for order "
                + order.getOrderId() + ": " + e.getMessage());
        }

        // Step 7: Send confirmation
        notificationService.sendOrderConfirmation(
            order.getCustomerId(), order.getOrderId(), trackingNumber);

        log.info("Checkout complete for order: " + order.getOrderId());

        return CheckoutResult.successBuilder(order.getOrderId())
            .withTrackingNumber(trackingNumber)
            .withLoyaltyPoints(pointsEarned)
            .build();
    }
}
```

#### Client Code

```java
package facade.ecommerce;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

public class CheckoutClient {

    public static void main(String[] args) {
        // Wire up the Facade with subsystem implementations
        CheckoutFacade checkoutFacade = new CheckoutFacade(
            ServiceStubs.inventoryService(),
            ServiceStubs.paymentService(),
            ServiceStubs.shippingService(),
            ServiceStubs.notificationService(),
            ServiceStubs.loyaltyService()
        );

        // Create a sample order
        List<OrderItem> items = List.of(
            new OrderItem("P-001", "Wireless Headphones", 1, new BigDecimal("149.99")),
            new OrderItem("P-002", "USB-C Cable", 2, new BigDecimal("19.99"))
        );

        Order order = new Order(
            UUID.randomUUID().toString(),
            "CUST-42",
            items,
            "123 Main St, San Francisco, CA 94105"
        );

        // The entire 5-service workflow is a single method call
        CheckoutResult result = checkoutFacade.checkout(order, "tok_stripe_test_4242");
        System.out.println(result);
    }
}
```

---

### 4.3 Compiler Facade

A compiler is one of the most naturally complex subsystems in software engineering. Each compilation phase is a distinct component. The Facade lets you compile source code with a single method call.

```java
package facade.compiler;

import java.util.ArrayList;
import java.util.List;

/**
 * Subsystem 1: Lexical analysis — breaks source text into tokens.
 */
public class Lexer {

    /** Represents a single lexical token. */
    public record Token(String type, String value, int line) {}

    /**
     * Tokenizes the given source code.
     *
     * @param source the raw source code string
     * @return an ordered list of tokens
     * @throws LexerException if an unrecognized character is encountered
     */
    public List<Token> tokenize(String source) throws LexerException {
        System.out.println("[Lexer] Tokenizing " + source.length() + " characters...");

        if (source == null || source.isBlank()) {
            throw new LexerException("Empty source provided to lexer", 0);
        }

        // Simplified simulation: split on whitespace and classify tokens
        List<Token> tokens = new ArrayList<>();
        String[] words = source.split("\\s+");
        int line = 1;

        for (String word : words) {
            if (word.equals("int") || word.equals("void") || word.equals("return")) {
                tokens.add(new Token("KEYWORD", word, line));
            } else if (word.matches("[a-zA-Z_][a-zA-Z0-9_]*")) {
                tokens.add(new Token("IDENTIFIER", word, line));
            } else if (word.matches("\\d+")) {
                tokens.add(new Token("INTEGER_LITERAL", word, line));
            } else if (word.equals("{") || word.equals("}") || word.equals(";")) {
                tokens.add(new Token("PUNCTUATION", word, line));
                if (word.equals(";")) line++;
            } else {
                tokens.add(new Token("OPERATOR", word, line));
            }
        }

        System.out.println("[Lexer] Produced " + tokens.size() + " tokens.");
        return tokens;
    }
}
```

```java
package facade.compiler;

/** Thrown when the lexer encounters invalid input. */
public class LexerException extends Exception {

    private final int line;

    public LexerException(String message, int line) {
        super(message + " (line " + line + ")");
        this.line = line;
    }

    public int getLine() { return line; }
}
```

```java
package facade.compiler;

import java.util.List;

/**
 * Subsystem 2: Syntax analysis — builds an AST from the token stream.
 */
public class Parser {

    /**
     * Minimal AST node representation.
     * A real parser would produce a rich typed hierarchy.
     */
    public record ASTNode(String type, String value, List<ASTNode> children) {

        public ASTNode(String type, String value) {
            this(type, value, List.of());
        }

        @Override
        public String toString() {
            return "ASTNode{type='" + type + "', value='" + value + "'}";
        }
    }

    /**
     * Parses a flat token list into an abstract syntax tree.
     *
     * @param tokens the tokens produced by the Lexer
     * @return the root node of the AST
     * @throws ParseException if the token stream does not conform to the grammar
     */
    public ASTNode parse(List<Lexer.Token> tokens) throws ParseException {
        System.out.println("[Parser] Parsing " + tokens.size() + " tokens...");

        if (tokens.isEmpty()) {
            throw new ParseException("No tokens to parse", 0);
        }

        // Simplified: produce a stub AST root from the token list
        List<ASTNode> children = tokens.stream()
            .map(t -> new ASTNode(t.type(), t.value()))
            .toList();

        ASTNode root = new ASTNode("PROGRAM", "root", children);
        System.out.println("[Parser] AST built with " + children.size() + " nodes.");
        return root;
    }
}
```

```java
package facade.compiler;

/** Thrown when parsing fails. */
public class ParseException extends Exception {

    private final int position;

    public ParseException(String message, int position) {
        super(message + " at token position " + position);
        this.position = position;
    }

    public int getPosition() { return position; }
}
```

```java
package facade.compiler;

import java.util.List;

/**
 * Subsystem 3: Semantic analysis — type checking, scope resolution,
 * symbol table construction.
 */
public class SemanticAnalyzer {

    /**
     * Performs semantic validation on the AST.
     *
     * @param ast the root node of the AST to analyze
     * @throws SemanticException if a type error, undefined reference,
     *                           or other semantic violation is found
     */
    public void analyze(Parser.ASTNode ast) throws SemanticException {
        System.out.println("[SemanticAnalyzer] Analyzing AST: " + ast.type() + "...");

        // Walk the AST and perform validation
        validateNode(ast);

        System.out.println("[SemanticAnalyzer] Semantic analysis passed.");
    }

    private void validateNode(Parser.ASTNode node) throws SemanticException {
        // Real implementation would do type inference, scope resolution, etc.
        for (Parser.ASTNode child : node.children()) {
            if ("IDENTIFIER".equals(child.type()) && child.value().startsWith("__")) {
                throw new SemanticException("Identifiers prefixed with '__' are reserved: "
                    + child.value());
            }
            validateNode(child);
        }
    }
}
```

```java
package facade.compiler;

/** Thrown when semantic analysis detects a violation. */
public class SemanticException extends Exception {

    public SemanticException(String message) {
        super(message);
    }
}
```

```java
package facade.compiler;

/**
 * Subsystem 4: Code generation — translates the validated AST into
 * target bytecode or machine code.
 */
public class CodeGenerator {

    /**
     * Generates target code from the semantically validated AST.
     *
     * @param ast    the validated AST
     * @param target the compilation target ("JVM_BYTECODE", "LLVM_IR", "x86_64")
     * @return the generated code as a byte array
     * @throws CodeGenerationException if code generation fails
     */
    public byte[] generate(Parser.ASTNode ast, String target)
        throws CodeGenerationException {

        System.out.println("[CodeGenerator] Generating " + target
            + " for AST root: " + ast.type() + "...");

        if (!isTargetSupported(target)) {
            throw new CodeGenerationException("Unsupported target: " + target);
        }

        // In reality this would walk the AST and emit instructions
        byte[] code = ("// " + target + " output for " + ast.value()).getBytes();
        System.out.println("[CodeGenerator] Generated " + code.length + " bytes.");
        return code;
    }

    private boolean isTargetSupported(String target) {
        return target.equals("JVM_BYTECODE")
            || target.equals("LLVM_IR")
            || target.equals("x86_64");
    }
}
```

```java
package facade.compiler;

/** Thrown when code generation fails. */
public class CodeGenerationException extends Exception {

    public CodeGenerationException(String message) {
        super(message);
    }
}
```

#### The Compiler Facade

```java
package facade.compiler;

import java.util.List;

/**
 * Facade for the compilation pipeline.
 *
 * <p>Sequences four compiler phases — Lexer → Parser → SemanticAnalyzer
 * → CodeGenerator — behind a single {@link #compile} method. Clients that
 * need only tokenization or only AST construction can still call the
 * subsystem classes directly; this Facade serves the common case of
 * full compilation.
 *
 * <p>Error handling strategy: each phase may throw a different checked
 * exception. The Facade catches them all and wraps them in a single
 * {@link CompilationException}, giving the caller a uniform failure type
 * without losing the original cause.
 */
public class CompilerFacade {

    private final Lexer lexer;
    private final Parser parser;
    private final SemanticAnalyzer semanticAnalyzer;
    private final CodeGenerator codeGenerator;
    private final String defaultTarget;

    public CompilerFacade() {
        this(new Lexer(), new Parser(), new SemanticAnalyzer(),
             new CodeGenerator(), "JVM_BYTECODE");
    }

    public CompilerFacade(Lexer lexer, Parser parser,
                          SemanticAnalyzer semanticAnalyzer,
                          CodeGenerator codeGenerator,
                          String defaultTarget) {
        this.lexer = lexer;
        this.parser = parser;
        this.semanticAnalyzer = semanticAnalyzer;
        this.codeGenerator = codeGenerator;
        this.defaultTarget = defaultTarget;
    }

    /**
     * Compiles source code using the default target.
     *
     * @param source the raw source code
     * @return compiled bytecode
     * @throws CompilationException if any compilation phase fails
     */
    public byte[] compile(String source) throws CompilationException {
        return compile(source, defaultTarget);
    }

    /**
     * Compiles source code to the specified target.
     *
     * @param source the raw source code
     * @param target the compilation target (e.g., "JVM_BYTECODE", "LLVM_IR")
     * @return compiled bytecode/IR
     * @throws CompilationException if any compilation phase fails
     */
    public byte[] compile(String source, String target) throws CompilationException {
        System.out.println("\n=== Starting compilation (target: " + target + ") ===");

        try {
            List<Lexer.Token> tokens = lexer.tokenize(source);
            Parser.ASTNode ast = parser.parse(tokens);
            semanticAnalyzer.analyze(ast);
            byte[] output = codeGenerator.generate(ast, target);

            System.out.println("=== Compilation successful (" + output.length + " bytes) ===\n");
            return output;

        } catch (LexerException e) {
            throw new CompilationException("Lexical error: " + e.getMessage(), e,
                CompilationException.Phase.LEXING);
        } catch (ParseException e) {
            throw new CompilationException("Parse error: " + e.getMessage(), e,
                CompilationException.Phase.PARSING);
        } catch (SemanticException e) {
            throw new CompilationException("Semantic error: " + e.getMessage(), e,
                CompilationException.Phase.SEMANTIC_ANALYSIS);
        } catch (CodeGenerationException e) {
            throw new CompilationException("Code generation error: " + e.getMessage(), e,
                CompilationException.Phase.CODE_GENERATION);
        }
    }
}
```

```java
package facade.compiler;

/**
 * Uniform exception for any compilation failure, regardless of which
 * internal phase produced it.
 */
public class CompilationException extends Exception {

    public enum Phase {
        LEXING, PARSING, SEMANTIC_ANALYSIS, CODE_GENERATION
    }

    private final Phase phase;

    public CompilationException(String message, Throwable cause, Phase phase) {
        super(message, cause);
        this.phase = phase;
    }

    public Phase getPhase() { return phase; }
}
```

#### Client Code

```java
package facade.compiler;

public class CompilerClient {

    public static void main(String[] args) {
        CompilerFacade compiler = new CompilerFacade();

        String source = "int main { return 42 ; }";

        try {
            byte[] bytecode = compiler.compile(source);
            System.out.println("Bytecode size: " + bytecode.length + " bytes");
        } catch (CompilationException e) {
            System.err.println("Compilation failed at phase: " + e.getPhase());
            System.err.println("Reason: " + e.getMessage());
        }
    }
}
```

---

## 5. Real-World Analogy

### The Hotel Concierge

When you check into a hotel, you interact with one person: the concierge. Behind that single interface lies an entire ecosystem of services — housekeeping, room service, valet parking, restaurant reservations, taxi dispatch, and maintenance.

You do not call housekeeping directly to request fresh towels. You do not negotiate with valet directly to get your car. You tell the concierge what you need and they coordinate the appropriate subsystems.

That is the Facade. The concierge is not the only way to interact with the hotel (you can call the restaurant yourself), but for the common case — getting things done without deep hotel knowledge — the concierge is the simplest path.

**The mapping:**

| Hotel World       | Software World                |
|-------------------|-------------------------------|
| Guest             | Client code                   |
| Concierge         | Facade class                  |
| Housekeeping      | Subsystem A                   |
| Restaurant        | Subsystem B                   |
| Valet             | Subsystem C                   |
| Emergency call    | Client bypassing the Facade   |

The key insight: the concierge does not prevent you from calling the restaurant yourself. They just make the common case easier.

---

## 6. Real-World Java Ecosystem Examples

### 6.1 SLF4J — The Logging Facade

SLF4J (Simple Logging Facade for Java) is the most downloaded Java library. It is, literally, a facade.

```java
// Client code only depends on the SLF4J API
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderService {

    // One line. No knowledge of Log4j2, Logback, JUL, or Log4j1.
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public void processOrder(String orderId) {
        log.info("Processing order: {}", orderId);
        // ...
        log.debug("Order {} processed successfully", orderId);
    }
}
```

Behind SLF4J's `Logger` interface lies a binding (typically Logback) that delegates to the actual logging implementation. SLF4J abstracts away:

- The configuration format (XML vs properties)
- The appender types (console, file, rolling)
- The async vs sync logging decision
- MDC (Mapped Diagnostic Context) mechanics

Applications can switch from Logback to Log4j2 by swapping a JAR dependency. The client code never changes. This is the Facade pattern used at the ecosystem level.

### 6.2 Spring's JdbcTemplate

Before `JdbcTemplate`, a JDBC query looked like this:

```java
// Without Facade — raw JDBC (10+ steps for every query)
Connection conn = null;
PreparedStatement stmt = null;
ResultSet rs = null;
try {
    conn = dataSource.getConnection();
    stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    stmt.setLong(1, userId);
    rs = stmt.executeQuery();
    if (rs.next()) {
        user = mapRow(rs);
    }
} catch (SQLException e) {
    throw new DataAccessException("Query failed", e);
} finally {
    if (rs != null) try { rs.close(); } catch (SQLException ignored) {}
    if (stmt != null) try { stmt.close(); } catch (SQLException ignored) {}
    if (conn != null) try { conn.close(); } catch (SQLException ignored) {}
}
```

With `JdbcTemplate`:

```java
// With Facade — JdbcTemplate handles connection, statement, result set, and cleanup
@Repository
public class UserRepository {

    private final JdbcTemplate jdbcTemplate;

    public UserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public Optional<User> findById(long userId) {
        return jdbcTemplate.query(
            "SELECT * FROM users WHERE id = ?",
            (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name")),
            userId
        ).stream().findFirst();
    }
}
```

`JdbcTemplate` is a Facade over raw JDBC. It manages the lifecycle of `Connection`, `PreparedStatement`, and `ResultSet`, translates `SQLException` into Spring's `DataAccessException` hierarchy, and handles transactions. The JDBC subsystem is not hidden — you can still obtain a raw `Connection` from the `DataSource` — but for 95% of use cases, `JdbcTemplate` is the right interface.

### 6.3 Hibernate Session

`Session` in Hibernate is a Facade over JDBC. It hides:

- Connection pool management (`SessionFactory` → `Connection`)
- SQL generation (from HQL or Criteria API)
- ResultSet mapping to Java objects
- First-level cache (identity map)
- Dirty checking and automatic `UPDATE` generation
- Transaction demarcation

```java
// Hibernate Session as Facade
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = session.get(User.class, userId);   // Facade hides: SQL, ResultSet mapping, caching
user.setEmail("new@example.com");              // Dirty tracking — no explicit update needed
tx.commit();                                   // Facade generates UPDATE only if entity changed
session.close();
```

### 6.4 JavaMail

The JavaMail API (`javax.mail`) is a Facade over the SMTP, POP3, and IMAP protocols. Clients deal with `Session`, `Message`, `Transport` — not TCP connections, SMTP command sequences, or base64 encoding of attachments.

```java
import javax.mail.*;
import javax.mail.internet.*;

public class EmailFacade {

    private final Session mailSession;

    public EmailFacade(String host, String username, String password) {
        Properties props = new Properties();
        props.put("mail.smtp.host", host);
        props.put("mail.smtp.port", "587");
        props.put("mail.smtp.auth", "true");
        props.put("mail.smtp.starttls.enable", "true");

        this.mailSession = Session.getInstance(props, new Authenticator() {
            @Override
            protected PasswordAuthentication getPasswordAuthentication() {
                return new PasswordAuthentication(username, password);
            }
        });
    }

    /**
     * Sends a plain-text email, hiding all SMTP protocol complexity.
     */
    public void sendEmail(String to, String subject, String body)
        throws MessagingException {

        Message message = new MimeMessage(mailSession);
        message.setFrom(new InternetAddress("noreply@example.com"));
        message.setRecipients(Message.RecipientType.TO, InternetAddress.parse(to));
        message.setSubject(subject);
        message.setText(body);
        Transport.send(message);
    }
}
```

---

## 7. Facade vs Mediator

These two patterns are frequently confused because both involve a central object that coordinates other objects. The difference is fundamental: it is about **direction of knowledge** and **complexity location**.

### Side-by-Side Comparison

| Dimension              | Facade                                          | Mediator                                                 |
|------------------------|-------------------------------------------------|----------------------------------------------------------|
| **Intent**             | Simplify access to a subsystem from outside     | Decouple objects that know about each other              |
| **Who initiates?**     | External client initiates via Facade            | Peers trigger the Mediator via events                    |
| **Knowledge**          | Facade knows subsystems; subsystems do NOT know the Facade | Mediator knows all peers; peers know the Mediator (not each other) |
| **Communication**      | One-directional (client → Facade → subsystems)  | Multi-directional (peer ↔ Mediator ↔ peer)               |
| **Subsystem coupling** | Subsystems are fully independent                | Colleagues are coupled to the Mediator interface         |
| **Complexity location**| Client complexity moves into the Facade         | Peer-to-peer communication complexity moves to Mediator  |
| **Use case**           | "Make this subsystem easy to use"               | "Stop these 10 objects from calling each other directly" |

### Structural Illustration

```
FACADE (one-directional flow from client):

  Client ──→ Facade ──→ SubsystemA
                    ──→ SubsystemB
                    ──→ SubsystemC

  SubsystemA does NOT know the Facade exists.

MEDIATOR (bidirectional: peers ↔ mediator):

  ColleagueA ──→ Mediator ──→ ColleagueB
  ColleagueB ──→ Mediator ──→ ColleagueA
  ColleagueC ──→ Mediator ──→ ColleagueA, ColleagueB

  Each Colleague holds a reference to the Mediator.
  The Mediator holds a reference to each Colleague.
```

### Java Example: When Each Applies

**Facade:** You have a `PaymentService`, `ShippingService`, and `NotificationService`. A checkout page must call all three. A `CheckoutFacade` wraps the sequence. The individual services have zero knowledge of the Facade.

**Mediator:** An air traffic control system. Multiple Aircraft objects need to coordinate landing slots. Each `Aircraft` sends messages to the `AirTrafficControl` mediator rather than communicating with other aircraft directly. The mediator decides which aircraft can land. Each `Aircraft` holds a reference to the mediator.

The critical question: **do the "colleagues" need to trigger actions in each other?** If yes, use Mediator. If an external client just needs a simple way to drive the subsystem, use Facade.

---

## 8. Facade vs Adapter

These patterns are structurally similar (one object wrapping another) but serve entirely different purposes.

### Side-by-Side Comparison

| Dimension              | Facade                                      | Adapter                                              |
|------------------------|---------------------------------------------|------------------------------------------------------|
| **Intent**             | Simplify a complex subsystem                | Make an incompatible interface compatible            |
| **Interface change?**  | Creates a new, simpler interface            | Translates one existing interface to another         |
| **Scope**              | Wraps many classes (a whole subsystem)      | Usually wraps a single class                         |
| **Client need**        | Ease of use — fewer methods to call         | Interface compatibility — same contract, different impl |
| **Existing interface?**| Clients did not use the subsystem directly  | Clients already have code targeting a specific interface |
| **Design driver**      | Make the common case easy                   | Plug in an incompatible component                   |

### Structural Illustration

```
ADAPTER (interface translation):

  Client expects: Target.request()
  Adaptee provides: LegacySystem.specificRequest()

  Client ──→ Adapter.request() ──→ Adaptee.specificRequest()
                                    (translated call)

FACADE (simplification):

  Client wants to watch a movie.
  Complex reality: TV + Sound + DVD + Lights + Projector

  Client ──→ Facade.watchMovie() ──→ tv.on()
                                 ──→ sound.setMode("Atmos")
                                 ──→ dvd.play(title)
                                 ──→ lights.dim(10)
                                 ──→ projector.on()
```

### A Facade Can Contain an Adapter (and vice versa)

In practice, a Facade that wraps a legacy system often uses Adapters internally — it adapts the old interfaces to fit the new model while presenting a simple surface to clients. Recognizing the layering is a sign of design maturity.

```java
// Adapter translates the legacy payment system into a modern interface
public class LegacyPaymentAdapter implements PaymentService {

    private final OldPaymentGateway legacy;

    public LegacyPaymentAdapter(OldPaymentGateway legacy) {
        this.legacy = legacy;
    }

    @Override
    public String charge(String customerId, BigDecimal amount, String method)
        throws PaymentException {
        // Translate: modern interface → old gateway's proprietary API
        boolean result = legacy.processPaymentV1(customerId, amount.doubleValue(), method);
        if (!result) {
            throw new PaymentException("Legacy gateway rejected payment", "LEGACY_DECLINED");
        }
        return "LEGACY-" + System.currentTimeMillis();
    }

    @Override
    public void refund(String transactionId, BigDecimal amount) throws PaymentException {
        legacy.reverseTransaction(transactionId, amount.doubleValue());
    }
}

// CheckoutFacade uses the Adapter transparently — it sees PaymentService
CheckoutFacade facade = new CheckoutFacade(
    inventoryService,
    new LegacyPaymentAdapter(new OldPaymentGateway()), // Adapter inside Facade
    shippingService,
    notificationService,
    loyaltyService
);
```

---

## 9. API Gateway as Facade in Microservices

In microservices architecture, the API Gateway is the Facade pattern applied at the infrastructure level.

### The Problem Without a Gateway

Without an API Gateway, a mobile client building a product page must make five separate calls:

```
Mobile App ──→ ProductService         GET /products/123
           ──→ ReviewService          GET /reviews?productId=123
           ──→ InventoryService       GET /inventory/123
           ──→ PricingService         GET /pricing/123
           ──→ RecommendationService  GET /recommendations?productId=123
```

Problems:
- Five round trips over a mobile network (high latency)
- Client must know the address of five services
- Five separate authentication checks
- If one service is slow, the client waits or gives up
- Adding SSL termination, rate limiting, and logging for five services

### The API Gateway Solves This as a Facade

```
Mobile App ──→ API Gateway ──→ ProductService
                           ──→ ReviewService
                           ──→ InventoryService
                           ──→ PricingService
                           ──→ RecommendationService
```

The Gateway:
1. Accepts one request from the mobile app
2. Fans out to the required backend services (possibly in parallel)
3. Aggregates and transforms the responses
4. Returns a single response to the client

### Gateway Capabilities Beyond a Basic Facade

| Concern               | How the Gateway Handles It                               |
|-----------------------|----------------------------------------------------------|
| Authentication        | Validates JWT once at the gateway; services trust it     |
| Rate limiting         | Per-client throttling at the edge, not per service       |
| SSL termination       | TLS handled once; internal traffic can be plaintext      |
| Circuit breaking      | If `ReviewService` is down, return cached data           |
| Request transformation| Mobile gets a compact response; desktop gets full data   |
| Observability         | Single point for distributed tracing, request logging    |
| BFF pattern           | Separate gateway facades per client type (mobile vs web) |

### BFF: Backend for Frontend as Specialized Facades

```
Mobile App ──→ Mobile BFF (Facade) ──→ ProductService
Web App    ──→ Web BFF (Facade)    ──→ ProductService
                                   ──→ AnalyticsService
                                   ──→ A/B TestingService
```

Each BFF is a specialized Facade tuned to the needs of its client. The mobile BFF returns compact, bandwidth-efficient responses. The web BFF returns rich responses with analytics data. The backend services remain unchanged.

---

## 10. Trade-offs

### Pros

**Reduced coupling between clients and subsystems.**
When a subsystem's internal structure changes, only the Facade changes. Clients are insulated. This makes refactoring internal components much safer.

**Simpler client code.**
A single `checkout(order, paymentMethod)` call is far less error-prone than manually coordinating five services in the correct order with proper error handling and compensation logic.

**Layered architecture enforcement.**
In a layered system, a Facade on each layer boundary prevents layer-skipping. Client code in the web layer cannot accidentally bypass the service layer and call the repository directly.

**Easier testing at the integration level.**
You can test the Facade's coordination logic without testing each subsystem. Mock the subsystems, verify the Facade calls them in the right order with the right arguments.

**Library evolution without breaking clients.**
Library maintainers can completely rewrite internal implementation while preserving the Facade's public API. Client code compiled against the old version continues to work.

### Cons

**Risk of becoming a God Object.**
If not carefully bounded, the Facade accumulates responsibility. Over time, every new feature gets added to the Facade rather than to the appropriate subsystem. The Facade becomes a large, complex class that defeats its own purpose.

**Thin Facade adds indirection without value.**
If the Facade does nothing but delegate calls one-for-one to a single subsystem class, it is pure overhead — an extra layer that clients must navigate without gaining any simplification.

**Can mask important subsystem evolution.**
If all clients go through the Facade, it becomes harder to evolve the subsystem's API in ways the Facade does not expose. Power users are locked out of new capabilities until the Facade exposes them.

**False sense of encapsulation.**
Developers sometimes assume that because a Facade exists, the subsystem is "hidden" and can be changed freely. But if any client bypasses the Facade, those clients will break when the subsystem changes. You must track both usage patterns.

**Coordination logic is hard to test in isolation.**
The Facade combines multiple service calls, often with conditional logic, compensation paths, and error handling. This coordination is typically unit-testable only by mocking all subsystems — which can lead to brittle tests tied to the Facade's specific calling sequence.

**Not a substitute for proper subsystem design.**
A Facade on a poorly designed subsystem still contains a poorly designed subsystem. The Facade makes it easier to use, but does not fix its fundamental flaws. If the subsystem is bad enough, the Facade just defers the inevitable redesign.

---

## 11. Common Interview Discussion Points

### Point 1: Facade Does Not Mean No Direct Access

Interviewers often present Facade as "hiding" a subsystem. The correct answer is that Facade *simplifies access* — it does not prevent direct access. Making subsystem classes package-private just to force clients through a Facade is an anti-pattern. It creates inflexibility and is not what the GoF described.

### Point 2: Subsystem Classes Should Not Know the Facade

The Facade knows the subsystem. The subsystem does not know the Facade. This is a one-way dependency. If a subsystem class imports or references the Facade, the design is wrong — you have created a circular dependency.

### Point 3: Facade Is a Structural Pattern, Not Behavioral

Facade is about how you structure the relationship between a client and a set of objects. It is not about changing behavior. The subsystems do the same work they always did — the Facade just makes them easier to reach.

### Point 4: Dependency Injection Matters

In production systems, the Facade should not instantiate its own subsystems (that would violate the Dependency Inversion Principle). Subsystem instances should be injected — either via constructor injection or a DI framework. This makes the Facade testable because test doubles can be injected in place of real subsystems.

### Point 5: The Compensation Pattern in Facades

When a Facade orchestrates multiple services and one fails midway, the Facade is responsible for compensating actions. In the `CheckoutFacade`, if payment fails, the Facade releases the inventory reservation. This coordination logic — which would otherwise be duplicated in every client — is a core value of the Facade.

### Point 6: Facade at Multiple Levels of Abstraction

A system can have Facades at multiple levels. A `HomeTheaterFacade` is itself a subsystem component that could be wrapped by a `SmartHomeFacade` alongside a `SecuritySystemFacade`, a `ClimateControlFacade`, and a `KitchenApplianceFacade`. Nesting Facades is legitimate when each level provides genuine simplification.

---

## 12. Interview Questions with Model Answers

---

### Q1: What is the Facade pattern and how does it differ from simply wrapping an object in another object?

**Model Answer:**

The Facade pattern provides a simplified interface to a complex *subsystem* — typically a group of collaborating classes. The key word is "subsystem." A Facade always wraps multiple objects working together, not a single object. If you are wrapping a single class, you are probably looking at an Adapter (to translate an interface) or a Proxy (to add behavior like caching or access control), not a Facade.

The Facade's value comes from hiding the *coordination complexity* of multiple objects. When watching a movie requires turning on a TV, a projector, a sound system, dimming lights, and starting a DVD player — all in a specific sequence — a Facade captures that sequence in one place. The alternative is every client duplicating the sequence, which leads to drift and bugs.

A naive "wrapper" that just delegates one-for-one to a single object adds only indirection — no simplification. A true Facade adds simplification by eliminating the N-call workflow from the client's perspective.

---

### Q2: How does the Facade pattern support the Dependency Inversion Principle?

**Model Answer:**

This is a subtle but important relationship. On its own, a concrete Facade class that directly instantiates concrete subsystem classes does *not* support DIP — it creates a chain of concrete dependencies.

The DIP-compliant version introduces two levels of abstraction:

1. Define the Facade behind an interface so clients depend on an abstraction, not the concrete Facade.
2. Inject subsystem interfaces into the Facade's constructor rather than having the Facade instantiate concrete subsystem classes.

```java
// DIP-compliant: clients depend on this interface
public interface CheckoutService {
    CheckoutResult checkout(Order order, String paymentMethod);
}

// Facade implements the interface and receives injected subsystem interfaces
public class CheckoutFacade implements CheckoutService {
    public CheckoutFacade(InventoryService inventory,
                          PaymentService payment,
                          ShippingService shipping,
                          NotificationService notification,
                          LoyaltyService loyalty) { ... }
}
```

With this structure, clients depend only on `CheckoutService`. The Facade depends only on subsystem interfaces. No code depends on concrete classes. This enables full testability (inject mocks) and full replaceability (swap any subsystem implementation without changing the Facade or any client).

In Spring, this is the idiomatic way to build Facades — the DI container wires everything together, and no class instantiates its own dependencies.

---

### Q3: When would you choose Facade over Mediator? Give a concrete example of each.

**Model Answer:**

The structural overlap between these patterns is significant, but the intent and communication direction are fundamentally different.

Choose **Facade** when an *external client* needs a simple way to drive a subsystem. The communication is one-directional: client → Facade → subsystems. The subsystems have no knowledge of the Facade and no reason to communicate back through it. Example: a `ReportGeneratorFacade` that calls `DataExtractor`, `DataTransformer`, `TemplateRenderer`, and `PdfWriter` in sequence. Each component does its job independently. A report-generation client calls `generateReport(params)` and gets back a PDF.

Choose **Mediator** when a set of *peer objects* need to communicate with each other but you want to avoid direct coupling between them. The communication is bidirectional: peer → Mediator → peer. Each peer holds a reference to the Mediator. Example: a chat room application. `ChatUser` objects should not hold references to every other `ChatUser`. Instead, each `ChatUser` holds a reference to `ChatRoom` (the Mediator). When Alice sends a message, she calls `chatRoom.send("Hello", alice)`. The ChatRoom delivers the message to all registered users except Alice. If Alice is blocked by Bob, the Mediator handles that logic without Alice knowing Bob exists.

The litmus test: do the "secondary" objects (subsystems or colleagues) ever need to initiate communication? If not — if they are pure servants — use Facade. If they are peers that react to each other — use Mediator.

---

### Q4: What are the risks of overusing the Facade pattern?

**Model Answer:**

The Facade is one of the easier patterns to overuse precisely because it feels helpful. Three risks deserve attention:

**The God Object risk.** Over time, a Facade tends to accumulate. Every new feature gets a method on the Facade rather than on the appropriate subsystem. Within a year, the Facade has forty methods and crosses multiple bounded contexts. It becomes harder to understand than the subsystem it was meant to simplify. The remedy is to create multiple, focused Facades rather than one omnibus Facade.

**The thin Facade anti-pattern.** A Facade that delegates every call one-for-one to a single class adds zero value — only indirection. If `OrderFacade.createOrder(params)` just calls `orderService.createOrder(params)` with no transformation, no coordination, and no additional logic, delete the Facade. Call `orderService` directly.

**The false encapsulation trap.** When developers see a Facade, they sometimes assume the underlying subsystem is unreachable and can be changed freely. But if any part of the codebase bypasses the Facade, those callers will break when the subsystem changes. A Facade is not a visibility modifier. It provides convenience, not encapsulation. If you want genuine encapsulation, use module boundaries (Java modules with `module-info.java`) or service boundaries (microservices).

---

### Q5: You have a legacy payment gateway with an ugly API. You also need to orchestrate it with a shipping service and a notification service. Should you use Adapter, Facade, or both? Explain.

**Model Answer:**

Both, in a layered composition: an Adapter wraps the legacy gateway, and a Facade orchestrates the adapted gateway with the other services.

The Adapter is needed because the legacy gateway has an incompatible interface. It might accept amounts as `double` when your domain uses `BigDecimal`. It might throw opaque error codes where your domain expects typed exceptions. The Adapter's job is pure interface translation — making the legacy gateway conform to your `PaymentService` interface. The Adapter knows the legacy API intimately but exposes a clean modern interface.

The Facade is needed because checkout requires coordinating the (adapted) payment gateway, the shipping service, and the notification service in the right sequence with proper error handling and compensation. The Facade's job is workflow coordination. It sees only the `PaymentService` interface — it has no knowledge of the legacy gateway or the Adapter.

```
CheckoutClient
    └─→ CheckoutFacade          (Facade: orchestrates workflow)
          ├─→ LegacyPaymentAdapter → OldGateway  (Adapter: translates interface)
          ├─→ ShippingService
          └─→ NotificationService
```

This is a clean separation of concerns. The Adapter solves interface incompatibility. The Facade solves coordination complexity. Neither class is responsible for both. This layering is common in real codebases when integrating external or legacy systems.

---

### Q6: How does SLF4J embody the Facade pattern? Why was this design decision significant for the Java ecosystem?

**Model Answer:**

SLF4J is a textbook implementation of the Facade pattern at the ecosystem level. The `org.slf4j.Logger` interface is the Facade interface. `LoggerFactory.getLogger(Class)` is the Facade factory. The subsystems — Logback, Log4j2, java.util.logging, and others — are the subsystem implementations that SLF4J bridges.

Before SLF4J, every logging framework had its own proprietary API. Libraries that depended on Log4j would force their users to also use Log4j. If an application used five libraries that each depended on a different logging framework, the application ended up with five logging frameworks on its classpath — a maintenance nightmare.

SLF4J solved this by providing a neutral API that all libraries can use. The logging implementation is chosen at deployment time by including the appropriate "binding" JAR. Logback binds to SLF4J natively. Log4j2 provides an SLF4J bridge. Java's built-in `java.util.logging` can also be bridged.

The ecosystem impact was profound: it became safe for library authors to use SLF4J without imposing a logging implementation on users. Application developers gained the freedom to choose their logging stack without being constrained by transitive dependencies. This is impossible with an Adapter (which would be one-to-one) — only the Facade's one-to-many abstraction could solve the problem.

The lesson for design: a well-placed Facade at an ecosystem boundary can reshape how an entire community builds software.

---

### Q7: How do you test a Facade in isolation? What should and should not be tested at the Facade level?

**Model Answer:**

A Facade should be unit-tested by injecting mock or stub implementations of its subsystem interfaces. The goal of a Facade unit test is to verify **coordination logic** — that the Facade calls subsystems in the right sequence, with the right arguments, and handles failures correctly.

```java
@ExtendWith(MockitoExtension.class)
class CheckoutFacadeTest {

    @Mock InventoryService inventoryService;
    @Mock PaymentService paymentService;
    @Mock ShippingService shippingService;
    @Mock NotificationService notificationService;
    @Mock LoyaltyService loyaltyService;

    private CheckoutFacade facade;

    @BeforeEach
    void setUp() {
        facade = new CheckoutFacade(inventoryService, paymentService,
            shippingService, notificationService, loyaltyService);
    }

    @Test
    void checkout_whenPaymentFails_shouldReleaseInventoryReservation() throws Exception {
        // Arrange
        Order order = buildTestOrder();
        when(inventoryService.checkAvailability(any())).thenReturn(true);
        when(paymentService.charge(any(), any(), any()))
            .thenThrow(new PaymentException("Card declined", "DECLINED"));

        // Act
        CheckoutResult result = facade.checkout(order, "tok_test");

        // Assert
        assertFalse(result.isSuccess());
        verify(inventoryService).reserveInventory(eq(order.getOrderId()), any());
        verify(inventoryService).releaseReservation(order.getOrderId()); // Compensation
        verify(notificationService).sendPaymentFailedNotification(any(), any(), any());
        verifyNoInteractions(shippingService, loyaltyService); // Not reached
    }
}
```

What to test at the Facade level:
- Correct call ordering when the happy path executes
- Compensation actions when specific phases fail
- That non-critical services (e.g., loyalty) failing does not fail the overall operation
- That the Facade returns the correct result type in each scenario

What NOT to test at the Facade level:
- The internal behavior of `PaymentService` (test that in `PaymentServiceTest`)
- Whether `InventoryService` correctly checks database quantities (test in `InventoryServiceTest`)
- Complex business rules that belong in individual subsystem tests

Facade tests are about workflow. Subsystem tests are about behavior.

---

### Q8: A junior developer proposes making all subsystem classes package-private so clients are forced through the Facade. What is your response?

**Model Answer:**

I would not do this, and I would explain why before offering an alternative.

Making subsystem classes package-private to force Facade usage conflates two different design goals: **simplicity** and **encapsulation**. The Facade is a simplicity tool — it makes the common path easy. Encapsulation is an information-hiding tool — it restricts what is externally visible. These are not the same thing, and trying to achieve encapsulation via Facade invariably creates problems.

**The practical problem:** Not all clients are the same. Some clients will legitimately need fine-grained control over a subsystem — more than the Facade exposes. By making subsystem classes package-private, you eliminate that option. You have built a bottleneck. Any future requirement that exceeds what the Facade provides requires modifying the Facade rather than extending the subsystem directly. This causes the Facade to grow without bound.

**The GoF position:** The Gang of Four explicitly states that the Facade pattern "does not prevent applications from using subsystem classes if they need to." The Facade provides a simplified interface; it does not monopolize access.

**Better alternatives for genuine encapsulation:**

1. Use **Java modules** (`module-info.java`). Export only the module's public API. Subsystem classes can be public within the module but not exported to other modules. This is encapsulation enforced by the compiler, not by package placement.

2. Use **package structure and naming conventions** to signal intent. Placing subsystem classes in an `internal` sub-package communicates "these are implementation details, use at your own risk" — without blocking legitimate advanced use.

3. **Document** the Facade as the recommended entry point while acknowledging that direct subsystem access is available for advanced use cases.

The junior developer's instinct — protecting the Facade from being bypassed — is understandable. But the right solution is better documentation and module boundaries, not package-private access modifiers that break legitimate use cases.

---

*This document is part of the LLD & OOP Design Patterns series. Next: [04-proxy.md](./04-proxy.md)*
