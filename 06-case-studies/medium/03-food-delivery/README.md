# Food Delivery System — Low-Level Design

> Inspired by DoorDash / Swiggy / Zomato

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Assumptions and Constraints](#4-assumptions-and-constraints)
5. [Domain Model Description](#5-domain-model-description)
6. [Class Diagram (ASCII UML)](#6-class-diagram-ascii-uml)
7. [Core Classes — Complete Java Implementation](#7-core-classes--complete-java-implementation)
8. [Design Patterns Used](#8-design-patterns-used)
9. [Key Design Decisions and Trade-offs](#9-key-design-decisions-and-trade-offs)
10. [Extension Points](#10-extension-points)
11. [Interview Discussion Points](#11-interview-discussion-points)
12. [Common Mistakes in This Design](#12-common-mistakes-in-this-design)

---

## 1. Problem Statement

Design a food delivery platform where customers can browse restaurant menus, add items to a cart, place orders, and track real-time delivery status. The system assigns an available delivery partner to each order, computes the total price including a distance-based delivery fee and taxes, and notifies both the customer and restaurant at every order lifecycle transition. Restaurants and delivery partners can receive ratings after an order is completed.

The focus of this LLD is the in-process object model: domain classes, state machines, design patterns, and an orchestration layer — not distributed infrastructure or persistence.

---

## 2. Functional Requirements

1. A **customer** can browse all registered restaurants and view their full menus.
2. A customer can **add or remove menu items** from a cart; the cart is scoped to a single restaurant.
3. A customer can **place an order** from their cart, which transitions the order to `PLACED`.
4. A **restaurant** can confirm or reject an order; confirmation transitions the order to `CONFIRMED`.
5. A restaurant can mark an order as `PREPARING` once kitchen work starts.
6. The system **automatically assigns** an available delivery partner to a confirmed order using a pluggable assignment strategy.
7. A delivery partner can mark the order as `PICKED_UP` once they collect the food.
8. A delivery partner can mark the order as `DELIVERED` upon successful handoff.
9. An order can be **cancelled** from `PLACED` or `CONFIRMED` states; not after preparation begins.
10. The system sends **notifications** to both the customer and the restaurant on every status change (Observer pattern).
11. A customer can **rate a restaurant** (1–5 stars) after delivery.
12. A customer can **rate a delivery partner** (1–5 stars) after delivery.
13. **Pricing** is: `subtotal + deliveryFee(distance) + taxes(18% of subtotal)`.
14. A delivery partner has a status: `AVAILABLE` or `ON_DELIVERY`.
15. A **restaurant** has a location, menu, and aggregate rating.

---

## 3. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Correctness | State transitions must follow the defined lifecycle; illegal transitions throw exceptions |
| Thread Safety | Order and Cart operations must be safe for concurrent access |
| Extensibility | New assignment strategies and notification channels must be addable without touching core classes |
| Testability | All dependencies injectable; no static singletons inside business logic |
| Clarity | Domain language in class names matches the real-world problem |

---

## 4. Assumptions and Constraints

- **Single city / flat distance model**: distance between two `Location` objects is computed with the Haversine formula; no traffic or routing service is called.
- **In-memory only**: no database, no persistence layer. All state lives in Java objects.
- **Single currency**: all prices are in a generic `double`; currency conversion is out of scope.
- **One restaurant per cart**: a cart is locked to the first restaurant whose item is added.
- **One delivery partner per order**: no split deliveries.
- **No authentication**: customer and restaurant identity is modeled as domain objects, not credentials.
- **Tax rate is fixed at 18%** of the subtotal; region-specific tax rules are out of scope.
- **Delivery fee formula**: `BASE_FEE + (distance_km * RATE_PER_KM)`; values are constants but configurable at construction.
- **Rating window**: ratings are only accepted in `DELIVERED` state; trying to rate before delivery throws.
- **Cancellation window**: orders may only be cancelled before `PREPARING`; once the kitchen starts, the order cannot be cancelled.

---

## 5. Domain Model Description

### Customer
Represents a platform user who browses menus and places orders. Holds a name, phone number, and delivery address represented as a `Location`.

### Location
A value object with latitude and longitude. Used by restaurants, customers, and delivery partners to compute distances.

### Restaurant
A registered food vendor. Owns a `Map<String, MenuItem>` (keyed by item ID), a `Location`, an aggregate rating, and a list of active orders. Implements `OrderStatusListener` so it receives state-change events.

### MenuItem
An immutable-ish product record: id, name, price, category, and an `available` flag. Items that are unavailable cannot be added to a cart.

### Cart
A mutable shopping basket scoped to one restaurant. Stores `Map<MenuItem, Integer>` (item → quantity). Provides add, remove, and clear operations. Computes its own subtotal.

### Order
The central aggregate. Holds a reference to the cart snapshot, the customer, the assigned delivery partner (nullable until assignment), the current `OrderState`, timestamps for each transition, and the computed `totalAmount`. Implements the State pattern — the `OrderState` held inside the order handles all transition calls and self-replaces itself.

### OrderState (interface)
Defines the lifecycle transition methods: `confirm()`, `startPreparing()`, `pickUp()`, `deliver()`, `cancel()`. Each concrete state implements the transitions that are legal from it and throws `IllegalStateException` for all others.

### DeliveryPartner
A courier on the platform. Holds a name, current `Location`, `PartnerStatus` (AVAILABLE / ON_DELIVERY), and a rating. Implements `OrderStatusListener` to receive relevant events.

### Rating
A value object: `raterId`, `targetId`, `score` (1–5), optional `comment`, and a `timestamp`.

### PricingCalculator
A stateless service. Computes `subtotal`, `deliveryFee`, `tax`, and `totalAmount` given a cart and a distance.

### DeliveryAssignmentStrategy (interface)
Selects a `DeliveryPartner` from a list of available partners for a given order. Two concrete strategies: `NearestDriverStrategy` and `RoundRobinStrategy`.

### FoodDeliveryService
The main orchestrator / facade. Holds registries for customers, restaurants, and delivery partners. Exposes high-level use-case methods: `placeOrder`, `confirmOrder`, `assignDeliveryPartner`, `pickUpOrder`, `deliverOrder`, `cancelOrder`, `rateRestaurant`, `rateDeliveryPartner`.

### OrderStatusListener (interface)
Observer interface with `onOrderStatusChanged(Order order, OrderStatus newStatus)`. `CustomerNotificationListener` and `RestaurantNotificationListener` are concrete observers.

---

## 6. Class Diagram (ASCII UML)

```
+------------------+          +-------------------+
|    Customer      |          |    Restaurant     |
+------------------+          +-------------------+
| -customerId      |          | -restaurantId     |
| -name            |          | -name             |
| -phone           |          | -location         |
| -location        |          | -menu: Map<>      |
+------------------+          | -rating           |
         |                    +-------------------+
         | places                      |
         v                             | contains
+------------------+          +-------------------+
|      Cart        |          |    MenuItem       |
+------------------+          +-------------------+
| -restaurant      |<>--------| -itemId           |
| -items: Map<>    |          | -name             |
| +addItem()       |          | -price            |
| +removeItem()    |          | -category         |
| +getSubtotal()   |          | -available        |
+------------------+          +-------------------+
         |
         | snapshot in
         v
+------------------+          +-------------------+
|      Order       |<>------->|   OrderState      | <<interface>>
+------------------+          +-------------------+
| -orderId         |          | +confirm()        |
| -cart            |          | +startPreparing() |
| -customer        |          | +pickUp()         |
| -deliveryPartner |          | +deliver()        |
| -state           |          | +cancel()         |
| -totalAmount     |          +-------------------+
| -listeners: []   |              ^    ^    ^
| +notify()        |              |    |    |
+------------------+    PlacedState  ConfirmedState
         |              PreparingState  PickedUpState
         |              DeliveredState  CancelledState
         |
         v
+-------------------+         +---------------------------+
| DeliveryPartner   |         | DeliveryAssignmentStrategy| <<interface>>
+-------------------+         +---------------------------+
| -partnerId        |         | +assignPartner()          |
| -name             |         +---------------------------+
| -location         |              ^              ^
| -status           |    NearestDriverStrategy  RoundRobinStrategy
| -rating           |
+-------------------+

+---------------------------+      +----------------------------+
| OrderStatusListener       |      | PricingCalculator          |
+---------------------------+      +----------------------------+
| <<interface>>             |      | +calculateSubtotal()       |
| +onOrderStatusChanged()   |      | +calculateDeliveryFee()    |
+---------------------------+      | +calculateTax()            |
      ^             ^              | +calculateTotal()          |
      |             |              +----------------------------+
CustomerNotification  RestaurantNotification
Listener              Listener

+---------------------------+
| FoodDeliveryService       |
+---------------------------+
| -restaurants: Map<>       |
| -customers: Map<>         |
| -partners: Map<>          |
| -assignmentStrategy       |
| -pricingCalculator        |
| +placeOrder()             |
| +confirmOrder()           |
| +assignDeliveryPartner()  |
| +pickUpOrder()            |
| +deliverOrder()           |
| +cancelOrder()            |
| +rateRestaurant()         |
| +rateDeliveryPartner()    |
+---------------------------+
```

---

## 7. Core Classes — Complete Java Implementation

### 7.1 Enumerations

```java
// OrderStatus.java
public enum OrderStatus {
    PLACED,
    CONFIRMED,
    PREPARING,
    PICKED_UP,
    DELIVERED,
    CANCELLED
}

// PartnerStatus.java
public enum PartnerStatus {
    AVAILABLE,
    ON_DELIVERY
}

// ItemCategory.java
public enum ItemCategory {
    STARTER,
    MAIN_COURSE,
    DESSERT,
    BEVERAGE,
    SIDE
}
```

---

### 7.2 Location (Value Object)

```java
// Location.java
public class Location {
    private final double latitude;
    private final double longitude;

    public Location(double latitude, double longitude) {
        this.latitude = latitude;
        this.longitude = longitude;
    }

    public double getLatitude() {
        return latitude;
    }

    public double getLongitude() {
        return longitude;
    }

    /**
     * Haversine formula — returns distance in kilometres between two coordinates.
     */
    public double distanceTo(Location other) {
        final double R = 6371.0; // Earth radius in km
        double lat1 = Math.toRadians(this.latitude);
        double lat2 = Math.toRadians(other.latitude);
        double deltaLat = Math.toRadians(other.latitude - this.latitude);
        double deltaLon = Math.toRadians(other.longitude - this.longitude);

        double a = Math.sin(deltaLat / 2) * Math.sin(deltaLat / 2)
                 + Math.cos(lat1) * Math.cos(lat2)
                 * Math.sin(deltaLon / 2) * Math.sin(deltaLon / 2);
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        return R * c;
    }

    @Override
    public String toString() {
        return String.format("(%.4f, %.4f)", latitude, longitude);
    }
}
```

---

### 7.3 Rating (Value Object)

```java
// Rating.java
import java.time.Instant;

public class Rating {
    private final String raterId;
    private final String targetId;
    private final int score;          // 1–5
    private final String comment;
    private final Instant timestamp;

    public Rating(String raterId, String targetId, int score, String comment) {
        if (score < 1 || score > 5) {
            throw new IllegalArgumentException("Rating score must be between 1 and 5; got: " + score);
        }
        this.raterId    = raterId;
        this.targetId   = targetId;
        this.score      = score;
        this.comment    = comment == null ? "" : comment;
        this.timestamp  = Instant.now();
    }

    public String getRaterId()   { return raterId; }
    public String getTargetId()  { return targetId; }
    public int getScore()        { return score; }
    public String getComment()   { return comment; }
    public Instant getTimestamp(){ return timestamp; }

    @Override
    public String toString() {
        return String.format("Rating[%d/5 by %s for %s: \"%s\"]", score, raterId, targetId, comment);
    }
}
```

---

### 7.4 MenuItem

```java
// MenuItem.java
public class MenuItem {
    private final String itemId;
    private final String name;
    private double price;
    private final ItemCategory category;
    private boolean available;

    public MenuItem(String itemId, String name, double price, ItemCategory category) {
        if (price < 0) throw new IllegalArgumentException("Price cannot be negative: " + price);
        this.itemId    = itemId;
        this.name      = name;
        this.price     = price;
        this.category  = category;
        this.available = true;
    }

    public String getItemId()        { return itemId; }
    public String getName()          { return name; }
    public double getPrice()         { return price; }
    public ItemCategory getCategory(){ return category; }
    public boolean isAvailable()     { return available; }

    public void setPrice(double price) {
        if (price < 0) throw new IllegalArgumentException("Price cannot be negative");
        this.price = price;
    }

    public void setAvailable(boolean available) {
        this.available = available;
    }

    @Override
    public String toString() {
        return String.format("MenuItem[%s | %s | $%.2f | %s | %s]",
                itemId, name, price, category, available ? "AVAILABLE" : "UNAVAILABLE");
    }
}
```

---

### 7.5 Customer

```java
// Customer.java
public class Customer {
    private final String customerId;
    private final String name;
    private final String phone;
    private Location deliveryLocation;

    public Customer(String customerId, String name, String phone, Location deliveryLocation) {
        this.customerId       = customerId;
        this.name             = name;
        this.phone            = phone;
        this.deliveryLocation = deliveryLocation;
    }

    public String getCustomerId()           { return customerId; }
    public String getName()                 { return name; }
    public String getPhone()                { return phone; }
    public Location getDeliveryLocation()   { return deliveryLocation; }

    public void setDeliveryLocation(Location loc) {
        this.deliveryLocation = loc;
    }

    @Override
    public String toString() {
        return String.format("Customer[%s | %s | %s]", customerId, name, phone);
    }
}
```

---

### 7.6 Restaurant

```java
// Restaurant.java
import java.util.*;

public class Restaurant {
    private final String restaurantId;
    private final String name;
    private final Location location;
    private final Map<String, MenuItem> menu;      // itemId -> MenuItem
    private final List<Rating> ratings;
    private double aggregateRating;

    public Restaurant(String restaurantId, String name, Location location) {
        this.restaurantId    = restaurantId;
        this.name            = name;
        this.location        = location;
        this.menu            = new LinkedHashMap<>();
        this.ratings         = new ArrayList<>();
        this.aggregateRating = 0.0;
    }

    // ----- Menu management -----

    public void addMenuItem(MenuItem item) {
        if (item == null) throw new IllegalArgumentException("MenuItem cannot be null");
        menu.put(item.getItemId(), item);
    }

    public void removeMenuItem(String itemId) {
        if (!menu.containsKey(itemId)) {
            throw new NoSuchElementException("Item not found in menu: " + itemId);
        }
        menu.remove(itemId);
    }

    public MenuItem getMenuItem(String itemId) {
        MenuItem item = menu.get(itemId);
        if (item == null) throw new NoSuchElementException("Item not found: " + itemId);
        return item;
    }

    public Map<String, MenuItem> getMenu() {
        return Collections.unmodifiableMap(menu);
    }

    // ----- Rating -----

    public void addRating(Rating rating) {
        ratings.add(rating);
        aggregateRating = ratings.stream()
                                 .mapToInt(Rating::getScore)
                                 .average()
                                 .orElse(0.0);
    }

    public double getAggregateRating() { return aggregateRating; }
    public List<Rating> getRatings()   { return Collections.unmodifiableList(ratings); }

    // ----- Accessors -----

    public String getRestaurantId() { return restaurantId; }
    public String getName()         { return name; }
    public Location getLocation()   { return location; }

    @Override
    public String toString() {
        return String.format("Restaurant[%s | %s | rating=%.1f]",
                restaurantId, name, aggregateRating);
    }
}
```

---

### 7.7 Cart

```java
// Cart.java
import java.util.*;

public class Cart {
    private final String cartId;
    private final Customer customer;
    private Restaurant restaurant;          // locked after first item
    private final Map<MenuItem, Integer> items; // item -> quantity

    public Cart(String cartId, Customer customer) {
        this.cartId     = cartId;
        this.customer   = customer;
        this.items      = new LinkedHashMap<>();
        this.restaurant = null;
    }

    /**
     * Add quantity units of item to the cart.
     * The cart is locked to the restaurant of the first item added.
     */
    public void addItem(MenuItem item, int quantity, Restaurant fromRestaurant) {
        if (item == null || fromRestaurant == null) {
            throw new IllegalArgumentException("Item and restaurant cannot be null");
        }
        if (!item.isAvailable()) {
            throw new IllegalStateException("Item is currently unavailable: " + item.getName());
        }
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive: " + quantity);
        }

        if (this.restaurant == null) {
            this.restaurant = fromRestaurant;
        } else if (!this.restaurant.getRestaurantId().equals(fromRestaurant.getRestaurantId())) {
            throw new IllegalStateException(
                "Cart is locked to restaurant '" + this.restaurant.getName()
                + "'. Cannot add item from '" + fromRestaurant.getName() + "'.");
        }

        items.merge(item, quantity, Integer::sum);
    }

    /**
     * Remove one unit of an item. If quantity reaches 0, the item is removed from the cart.
     */
    public void removeItem(MenuItem item, int quantity) {
        if (!items.containsKey(item)) {
            throw new NoSuchElementException("Item not in cart: " + item.getName());
        }
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity to remove must be positive");
        }
        int current = items.get(item);
        if (quantity >= current) {
            items.remove(item);
        } else {
            items.put(item, current - quantity);
        }
        if (items.isEmpty()) {
            this.restaurant = null; // cart is empty again, unlock restaurant
        }
    }

    public void clear() {
        items.clear();
        this.restaurant = null;
    }

    public double getSubtotal() {
        return items.entrySet().stream()
                    .mapToDouble(e -> e.getKey().getPrice() * e.getValue())
                    .sum();
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }

    public Map<MenuItem, Integer> getItems() {
        return Collections.unmodifiableMap(items);
    }

    public Restaurant getRestaurant() { return restaurant; }
    public Customer getCustomer()     { return customer; }
    public String getCartId()         { return cartId; }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("Cart[").append(cartId).append("]\n");
        items.forEach((item, qty) ->
            sb.append("  ").append(item.getName())
              .append(" x").append(qty)
              .append(" @ $").append(item.getPrice()).append("\n")
        );
        sb.append("  Subtotal: $").append(String.format("%.2f", getSubtotal()));
        return sb.toString();
    }
}
```

---

### 7.8 DeliveryPartner

```java
// DeliveryPartner.java
import java.util.*;

public class DeliveryPartner {
    private final String partnerId;
    private final String name;
    private Location location;
    private PartnerStatus status;
    private final List<Rating> ratings;
    private double aggregateRating;

    public DeliveryPartner(String partnerId, String name, Location location) {
        this.partnerId       = partnerId;
        this.name            = name;
        this.location        = location;
        this.status          = PartnerStatus.AVAILABLE;
        this.ratings         = new ArrayList<>();
        this.aggregateRating = 0.0;
    }

    public void goOnDelivery() {
        if (status == PartnerStatus.ON_DELIVERY) {
            throw new IllegalStateException("Partner is already on a delivery: " + partnerId);
        }
        this.status = PartnerStatus.ON_DELIVERY;
    }

    public void becomeAvailable() {
        this.status = PartnerStatus.AVAILABLE;
    }

    public void updateLocation(Location newLocation) {
        this.location = newLocation;
    }

    public void addRating(Rating rating) {
        ratings.add(rating);
        aggregateRating = ratings.stream()
                                 .mapToInt(Rating::getScore)
                                 .average()
                                 .orElse(0.0);
    }

    public double getAggregateRating() { return aggregateRating; }
    public List<Rating> getRatings()   { return Collections.unmodifiableList(ratings); }

    public String getPartnerId()    { return partnerId; }
    public String getName()         { return name; }
    public Location getLocation()   { return location; }
    public PartnerStatus getStatus(){ return status; }

    @Override
    public String toString() {
        return String.format("DeliveryPartner[%s | %s | %s | rating=%.1f]",
                partnerId, name, status, aggregateRating);
    }
}
```

---

### 7.9 Observer — OrderStatusListener

```java
// OrderStatusListener.java
public interface OrderStatusListener {
    void onOrderStatusChanged(Order order, OrderStatus newStatus);
}
```

```java
// CustomerNotificationListener.java
public class CustomerNotificationListener implements OrderStatusListener {

    @Override
    public void onOrderStatusChanged(Order order, OrderStatus newStatus) {
        Customer customer = order.getCustomer();
        String message = buildMessage(order, newStatus);
        // In production this would call an SMS / push notification service.
        System.out.printf("[CUSTOMER NOTIFICATION] -> %s (%s): %s%n",
                customer.getName(), customer.getPhone(), message);
    }

    private String buildMessage(Order order, OrderStatus newStatus) {
        switch (newStatus) {
            case PLACED:
                return "Your order #" + order.getOrderId() + " has been placed successfully!";
            case CONFIRMED:
                return "Great news! " + order.getCart().getRestaurant().getName()
                        + " has confirmed your order #" + order.getOrderId() + ".";
            case PREPARING:
                return "Your food is being prepared. Order #" + order.getOrderId() + ".";
            case PICKED_UP:
                DeliveryPartner partner = order.getDeliveryPartner();
                String partnerName = partner != null ? partner.getName() : "your delivery partner";
                return partnerName + " has picked up your order #" + order.getOrderId()
                        + ". On the way!";
            case DELIVERED:
                return "Order #" + order.getOrderId() + " delivered! Enjoy your meal. "
                        + "Please rate your experience.";
            case CANCELLED:
                return "Order #" + order.getOrderId() + " has been cancelled. "
                        + "Refund will be processed in 3–5 business days.";
            default:
                return "Order #" + order.getOrderId() + " status: " + newStatus;
        }
    }
}
```

```java
// RestaurantNotificationListener.java
public class RestaurantNotificationListener implements OrderStatusListener {

    @Override
    public void onOrderStatusChanged(Order order, OrderStatus newStatus) {
        Restaurant restaurant = order.getCart().getRestaurant();
        String message = buildMessage(order, newStatus);
        System.out.printf("[RESTAURANT NOTIFICATION] -> %s: %s%n",
                restaurant.getName(), message);
    }

    private String buildMessage(Order order, OrderStatus newStatus) {
        switch (newStatus) {
            case PLACED:
                return "New order received! Order #" + order.getOrderId()
                        + ". Please confirm or reject.";
            case CONFIRMED:
                return "You confirmed order #" + order.getOrderId() + ". Start preparing.";
            case PREPARING:
                return "Order #" + order.getOrderId() + " marked as preparing.";
            case PICKED_UP:
                return "Order #" + order.getOrderId() + " picked up by delivery partner.";
            case DELIVERED:
                return "Order #" + order.getOrderId() + " delivered successfully.";
            case CANCELLED:
                return "Order #" + order.getOrderId() + " has been cancelled.";
            default:
                return "Order #" + order.getOrderId() + " status update: " + newStatus;
        }
    }
}
```

---

### 7.10 State Pattern — OrderState Interface and Concrete States

```java
// OrderState.java
public interface OrderState {
    void confirm(Order order);
    void startPreparing(Order order);
    void pickUp(Order order);
    void deliver(Order order);
    void cancel(Order order);
    OrderStatus getStatus();
}
```

```java
// PlacedState.java
import java.time.Instant;

public class PlacedState implements OrderState {

    @Override
    public void confirm(Order order) {
        order.setConfirmedAt(Instant.now());
        order.setState(new ConfirmedState());
        order.notifyListeners(OrderStatus.CONFIRMED);
    }

    @Override
    public void startPreparing(Order order) {
        throw new IllegalStateException(
            "Cannot start preparing: order must be confirmed first. Current state: PLACED");
    }

    @Override
    public void pickUp(Order order) {
        throw new IllegalStateException(
            "Cannot pick up: order has not been prepared. Current state: PLACED");
    }

    @Override
    public void deliver(Order order) {
        throw new IllegalStateException(
            "Cannot deliver: order is still placed. Current state: PLACED");
    }

    @Override
    public void cancel(Order order) {
        order.setCancelledAt(Instant.now());
        order.setState(new CancelledState());
        order.notifyListeners(OrderStatus.CANCELLED);
    }

    @Override
    public OrderStatus getStatus() {
        return OrderStatus.PLACED;
    }
}
```

```java
// ConfirmedState.java
import java.time.Instant;

public class ConfirmedState implements OrderState {

    @Override
    public void confirm(Order order) {
        throw new IllegalStateException("Order is already confirmed.");
    }

    @Override
    public void startPreparing(Order order) {
        order.setPreparingAt(Instant.now());
        order.setState(new PreparingState());
        order.notifyListeners(OrderStatus.PREPARING);
    }

    @Override
    public void pickUp(Order order) {
        throw new IllegalStateException(
            "Cannot pick up: order has not been prepared yet. Current state: CONFIRMED");
    }

    @Override
    public void deliver(Order order) {
        throw new IllegalStateException(
            "Cannot deliver: order has not been prepared yet. Current state: CONFIRMED");
    }

    @Override
    public void cancel(Order order) {
        order.setCancelledAt(Instant.now());
        order.setState(new CancelledState());
        order.notifyListeners(OrderStatus.CANCELLED);
    }

    @Override
    public OrderStatus getStatus() {
        return OrderStatus.CONFIRMED;
    }
}
```

```java
// PreparingState.java
import java.time.Instant;

public class PreparingState implements OrderState {

    @Override
    public void confirm(Order order) {
        throw new IllegalStateException("Order is already confirmed and preparing.");
    }

    @Override
    public void startPreparing(Order order) {
        throw new IllegalStateException("Order is already in PREPARING state.");
    }

    @Override
    public void pickUp(Order order) {
        if (order.getDeliveryPartner() == null) {
            throw new IllegalStateException(
                "No delivery partner assigned. Cannot pick up order #" + order.getOrderId());
        }
        order.setPickedUpAt(Instant.now());
        order.setState(new PickedUpState());
        order.notifyListeners(OrderStatus.PICKED_UP);
    }

    @Override
    public void deliver(Order order) {
        throw new IllegalStateException(
            "Cannot deliver: order has not been picked up yet. Current state: PREPARING");
    }

    @Override
    public void cancel(Order order) {
        throw new IllegalStateException(
            "Cannot cancel: kitchen has already started preparing this order.");
    }

    @Override
    public OrderStatus getStatus() {
        return OrderStatus.PREPARING;
    }
}
```

```java
// PickedUpState.java
import java.time.Instant;

public class PickedUpState implements OrderState {

    @Override
    public void confirm(Order order) {
        throw new IllegalStateException("Order is already picked up; confirmation is irrelevant.");
    }

    @Override
    public void startPreparing(Order order) {
        throw new IllegalStateException("Order is already picked up; preparation is complete.");
    }

    @Override
    public void pickUp(Order order) {
        throw new IllegalStateException("Order is already in PICKED_UP state.");
    }

    @Override
    public void deliver(Order order) {
        order.setDeliveredAt(Instant.now());
        order.setState(new DeliveredState());
        // Free up the delivery partner
        if (order.getDeliveryPartner() != null) {
            order.getDeliveryPartner().becomeAvailable();
        }
        order.notifyListeners(OrderStatus.DELIVERED);
    }

    @Override
    public void cancel(Order order) {
        throw new IllegalStateException(
            "Cannot cancel: order is already picked up and on its way.");
    }

    @Override
    public OrderStatus getStatus() {
        return OrderStatus.PICKED_UP;
    }
}
```

```java
// DeliveredState.java
public class DeliveredState implements OrderState {

    @Override
    public void confirm(Order order) {
        throw new IllegalStateException("Order is already delivered.");
    }

    @Override
    public void startPreparing(Order order) {
        throw new IllegalStateException("Order is already delivered.");
    }

    @Override
    public void pickUp(Order order) {
        throw new IllegalStateException("Order is already delivered.");
    }

    @Override
    public void deliver(Order order) {
        throw new IllegalStateException("Order is already in DELIVERED state.");
    }

    @Override
    public void cancel(Order order) {
        throw new IllegalStateException("Cannot cancel a delivered order.");
    }

    @Override
    public OrderStatus getStatus() {
        return OrderStatus.DELIVERED;
    }
}
```

```java
// CancelledState.java
public class CancelledState implements OrderState {

    @Override
    public void confirm(Order order) {
        throw new IllegalStateException("Cannot confirm a cancelled order.");
    }

    @Override
    public void startPreparing(Order order) {
        throw new IllegalStateException("Cannot prepare a cancelled order.");
    }

    @Override
    public void pickUp(Order order) {
        throw new IllegalStateException("Cannot pick up a cancelled order.");
    }

    @Override
    public void deliver(Order order) {
        throw new IllegalStateException("Cannot deliver a cancelled order.");
    }

    @Override
    public void cancel(Order order) {
        throw new IllegalStateException("Order is already cancelled.");
    }

    @Override
    public OrderStatus getStatus() {
        return OrderStatus.CANCELLED;
    }
}
```

---

### 7.11 Order (Central Aggregate)

```java
// Order.java
import java.time.Instant;
import java.util.*;

public class Order {
    private final String orderId;
    private final Cart cart;
    private final Customer customer;
    private DeliveryPartner deliveryPartner;
    private OrderState state;
    private double totalAmount;

    // Timestamps for each phase
    private final Instant placedAt;
    private Instant confirmedAt;
    private Instant preparingAt;
    private Instant pickedUpAt;
    private Instant deliveredAt;
    private Instant cancelledAt;

    private final List<OrderStatusListener> listeners;

    public Order(String orderId, Cart cart, Customer customer, double totalAmount) {
        if (cart.isEmpty()) {
            throw new IllegalArgumentException("Cannot create an order from an empty cart.");
        }
        this.orderId      = orderId;
        this.cart         = cart;
        this.customer     = customer;
        this.totalAmount  = totalAmount;
        this.state        = new PlacedState();
        this.placedAt     = Instant.now();
        this.listeners    = new ArrayList<>();
    }

    // ----- Observer management -----

    public void addListener(OrderStatusListener listener) {
        if (listener != null) listeners.add(listener);
    }

    public void removeListener(OrderStatusListener listener) {
        listeners.remove(listener);
    }

    public void notifyListeners(OrderStatus newStatus) {
        for (OrderStatusListener listener : listeners) {
            listener.onOrderStatusChanged(this, newStatus);
        }
    }

    // ----- State transitions (delegated to the current state) -----

    public void confirm() {
        state.confirm(this);
    }

    public void startPreparing() {
        state.startPreparing(this);
    }

    public void pickUp() {
        state.pickUp(this);
    }

    public void deliver() {
        state.deliver(this);
    }

    public void cancel() {
        state.cancel(this);
    }

    public OrderStatus getStatus() {
        return state.getStatus();
    }

    // ----- Package-friendly setters used by State classes -----

    void setState(OrderState newState) {
        this.state = newState;
    }

    void setConfirmedAt(Instant t)  { this.confirmedAt = t; }
    void setPreparingAt(Instant t)  { this.preparingAt = t; }
    void setPickedUpAt(Instant t)   { this.pickedUpAt  = t; }
    void setDeliveredAt(Instant t)  { this.deliveredAt = t; }
    void setCancelledAt(Instant t)  { this.cancelledAt = t; }

    // ----- Accessors -----

    public String getOrderId()              { return orderId; }
    public Cart getCart()                   { return cart; }
    public Customer getCustomer()           { return customer; }
    public DeliveryPartner getDeliveryPartner() { return deliveryPartner; }
    public double getTotalAmount()          { return totalAmount; }
    public Instant getPlacedAt()            { return placedAt; }
    public Instant getConfirmedAt()         { return confirmedAt; }
    public Instant getPreparingAt()         { return preparingAt; }
    public Instant getPickedUpAt()          { return pickedUpAt; }
    public Instant getDeliveredAt()         { return deliveredAt; }
    public Instant getCancelledAt()         { return cancelledAt; }

    public void setDeliveryPartner(DeliveryPartner partner) {
        this.deliveryPartner = partner;
    }

    @Override
    public String toString() {
        return String.format("Order[%s | %s | $%.2f | status=%s]",
                orderId, customer.getName(), totalAmount, getStatus());
    }
}
```

---

### 7.12 PricingCalculator

```java
// PricingCalculator.java
public class PricingCalculator {
    private final double baseFee;        // flat base delivery fee
    private final double ratePerKm;      // additional fee per km
    private final double taxRate;        // e.g. 0.18 for 18%

    public PricingCalculator(double baseFee, double ratePerKm, double taxRate) {
        this.baseFee    = baseFee;
        this.ratePerKm  = ratePerKm;
        this.taxRate    = taxRate;
    }

    /** Subtotal = sum of (item price * quantity) from the cart. */
    public double calculateSubtotal(Cart cart) {
        return cart.getSubtotal();
    }

    /**
     * Delivery fee = baseFee + ratePerKm * distance.
     * Distance is from the restaurant to the customer's delivery address.
     */
    public double calculateDeliveryFee(Location restaurantLocation, Location customerLocation) {
        double distanceKm = restaurantLocation.distanceTo(customerLocation);
        return baseFee + (ratePerKm * distanceKm);
    }

    /** Tax = taxRate * subtotal. */
    public double calculateTax(double subtotal) {
        return subtotal * taxRate;
    }

    /**
     * Total = subtotal + deliveryFee + tax.
     * Rounds to 2 decimal places.
     */
    public double calculateTotal(Cart cart, Location restaurantLocation, Location customerLocation) {
        double subtotal     = calculateSubtotal(cart);
        double deliveryFee  = calculateDeliveryFee(restaurantLocation, customerLocation);
        double tax          = calculateTax(subtotal);
        double total        = subtotal + deliveryFee + tax;
        return Math.round(total * 100.0) / 100.0;
    }

    public double getBaseFee()    { return baseFee; }
    public double getRatePerKm()  { return ratePerKm; }
    public double getTaxRate()    { return taxRate; }
}
```

---

### 7.13 Strategy Pattern — DeliveryAssignmentStrategy

```java
// DeliveryAssignmentStrategy.java
import java.util.List;

public interface DeliveryAssignmentStrategy {
    /**
     * Select a delivery partner from the provided list of available partners.
     *
     * @param availablePartners partners whose status is AVAILABLE
     * @param order             the order to assign
     * @return the chosen DeliveryPartner, or null if no partner is available
     */
    DeliveryPartner assignPartner(List<DeliveryPartner> availablePartners, Order order);
}
```

```java
// NearestDriverStrategy.java
import java.util.Comparator;
import java.util.List;

/**
 * Selects the delivery partner whose current location is closest to the restaurant.
 * This minimises the time to pickup.
 */
public class NearestDriverStrategy implements DeliveryAssignmentStrategy {

    @Override
    public DeliveryPartner assignPartner(List<DeliveryPartner> availablePartners, Order order) {
        if (availablePartners == null || availablePartners.isEmpty()) {
            return null;
        }

        Location restaurantLocation = order.getCart().getRestaurant().getLocation();

        return availablePartners.stream()
                .min(Comparator.comparingDouble(
                        partner -> partner.getLocation().distanceTo(restaurantLocation)))
                .orElse(null);
    }
}
```

```java
// RoundRobinStrategy.java
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Distributes orders evenly across all available partners in a rotating fashion.
 * Useful when all partners are roughly equidistant or when fair load distribution matters.
 */
public class RoundRobinStrategy implements DeliveryAssignmentStrategy {

    private final AtomicInteger counter = new AtomicInteger(0);

    @Override
    public DeliveryPartner assignPartner(List<DeliveryPartner> availablePartners, Order order) {
        if (availablePartners == null || availablePartners.isEmpty()) {
            return null;
        }
        int index = counter.getAndIncrement() % availablePartners.size();
        return availablePartners.get(index);
    }
}
```

---

### 7.14 FoodDeliveryService (Main Orchestrator)

```java
// FoodDeliveryService.java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;

public class FoodDeliveryService {

    private final Map<String, Restaurant>      restaurants;
    private final Map<String, Customer>        customers;
    private final Map<String, DeliveryPartner> deliveryPartners;
    private final Map<String, Order>           orders;

    private DeliveryAssignmentStrategy assignmentStrategy;
    private final PricingCalculator    pricingCalculator;

    private final AtomicLong orderCounter = new AtomicLong(1000);
    private final AtomicLong cartCounter  = new AtomicLong(1);

    public FoodDeliveryService(DeliveryAssignmentStrategy assignmentStrategy,
                               PricingCalculator pricingCalculator) {
        this.assignmentStrategy = assignmentStrategy;
        this.pricingCalculator  = pricingCalculator;
        this.restaurants        = new ConcurrentHashMap<>();
        this.customers          = new ConcurrentHashMap<>();
        this.deliveryPartners   = new ConcurrentHashMap<>();
        this.orders             = new ConcurrentHashMap<>();
    }

    // ----- Registration -----

    public void registerRestaurant(Restaurant restaurant) {
        restaurants.put(restaurant.getRestaurantId(), restaurant);
        System.out.println("[SYSTEM] Registered restaurant: " + restaurant.getName());
    }

    public void registerCustomer(Customer customer) {
        customers.put(customer.getCustomerId(), customer);
        System.out.println("[SYSTEM] Registered customer: " + customer.getName());
    }

    public void registerDeliveryPartner(DeliveryPartner partner) {
        deliveryPartners.put(partner.getPartnerId(), partner);
        System.out.println("[SYSTEM] Registered delivery partner: " + partner.getName());
    }

    // ----- Strategy swap (Strategy pattern benefit) -----

    public void setAssignmentStrategy(DeliveryAssignmentStrategy strategy) {
        this.assignmentStrategy = strategy;
        System.out.println("[SYSTEM] Assignment strategy changed to: "
                + strategy.getClass().getSimpleName());
    }

    // ----- Browsing -----

    public List<Restaurant> getAllRestaurants() {
        return new ArrayList<>(restaurants.values());
    }

    public Map<String, MenuItem> getMenu(String restaurantId) {
        return getRestaurantOrThrow(restaurantId).getMenu();
    }

    // ----- Cart -----

    public Cart createCart(String customerId) {
        Customer customer = getCustomerOrThrow(customerId);
        String cartId = "CART-" + cartCounter.getAndIncrement();
        return new Cart(cartId, customer);
    }

    // ----- Ordering -----

    /**
     * Place an order from the given cart.
     * Computes the total price, creates an Order in PLACED state,
     * attaches observers, and notifies listeners.
     */
    public Order placeOrder(Cart cart) {
        if (cart.isEmpty()) {
            throw new IllegalArgumentException("Cannot place an order with an empty cart.");
        }

        Restaurant restaurant = cart.getRestaurant();
        Customer   customer   = cart.getCustomer();

        double total = pricingCalculator.calculateTotal(
                cart,
                restaurant.getLocation(),
                customer.getDeliveryLocation());

        String orderId = "ORD-" + orderCounter.getAndIncrement();
        Order order = new Order(orderId, cart, customer, total);

        // Attach observers
        order.addListener(new CustomerNotificationListener());
        order.addListener(new RestaurantNotificationListener());

        orders.put(orderId, order);

        // Notify — the order begins in PLACED state but we fire the initial event manually
        order.notifyListeners(OrderStatus.PLACED);

        System.out.printf("[SERVICE] Order %s placed. Total: $%.2f%n", orderId, total);
        return order;
    }

    /**
     * Restaurant confirms the order and the system immediately attempts partner assignment.
     */
    public void confirmOrder(String orderId) {
        Order order = getOrderOrThrow(orderId);
        order.confirm();

        // Try to assign a delivery partner right after confirmation
        assignDeliveryPartner(orderId);
    }

    /**
     * Assigns an available delivery partner using the configured strategy.
     */
    public DeliveryPartner assignDeliveryPartner(String orderId) {
        Order order = getOrderOrThrow(orderId);

        List<DeliveryPartner> available = deliveryPartners.values().stream()
                .filter(p -> p.getStatus() == PartnerStatus.AVAILABLE)
                .collect(Collectors.toList());

        DeliveryPartner partner = assignmentStrategy.assignPartner(available, order);

        if (partner == null) {
            System.out.printf("[SERVICE] No available delivery partner for order %s.%n", orderId);
            return null;
        }

        partner.goOnDelivery();
        order.setDeliveryPartner(partner);
        System.out.printf("[SERVICE] Assigned partner %s to order %s.%n",
                partner.getName(), orderId);
        return partner;
    }

    /** Restaurant marks the order as being prepared. */
    public void startPreparing(String orderId) {
        getOrderOrThrow(orderId).startPreparing();
    }

    /** Delivery partner marks the order as picked up. */
    public void pickUpOrder(String orderId) {
        getOrderOrThrow(orderId).pickUp();
    }

    /** Delivery partner marks the order as delivered. */
    public void deliverOrder(String orderId) {
        getOrderOrThrow(orderId).deliver();
    }

    /** Cancel an order (only valid in PLACED or CONFIRMED state). */
    public void cancelOrder(String orderId) {
        Order order = getOrderOrThrow(orderId);
        order.cancel();
        // If a partner was already assigned, free them up
        if (order.getDeliveryPartner() != null
                && order.getDeliveryPartner().getStatus() == PartnerStatus.ON_DELIVERY) {
            order.getDeliveryPartner().becomeAvailable();
        }
    }

    // ----- Ratings -----

    /**
     * Rate a restaurant after the order is delivered.
     * Only allowed in DELIVERED state.
     */
    public void rateRestaurant(String orderId, int score, String comment) {
        Order order = getOrderOrThrow(orderId);
        if (order.getStatus() != OrderStatus.DELIVERED) {
            throw new IllegalStateException(
                "Restaurant can only be rated after delivery. Current status: " + order.getStatus());
        }
        Restaurant restaurant = order.getCart().getRestaurant();
        Rating rating = new Rating(order.getCustomer().getCustomerId(),
                                   restaurant.getRestaurantId(), score, comment);
        restaurant.addRating(rating);
        System.out.printf("[RATING] Customer %s rated %s: %d/5 — \"%s\"%n",
                order.getCustomer().getName(), restaurant.getName(), score, comment);
    }

    /**
     * Rate a delivery partner after the order is delivered.
     * Only allowed in DELIVERED state.
     */
    public void rateDeliveryPartner(String orderId, int score, String comment) {
        Order order = getOrderOrThrow(orderId);
        if (order.getStatus() != OrderStatus.DELIVERED) {
            throw new IllegalStateException(
                "Delivery partner can only be rated after delivery. Current status: "
                + order.getStatus());
        }
        if (order.getDeliveryPartner() == null) {
            throw new IllegalStateException("No delivery partner associated with order: " + orderId);
        }
        DeliveryPartner partner = order.getDeliveryPartner();
        Rating rating = new Rating(order.getCustomer().getCustomerId(),
                                   partner.getPartnerId(), score, comment);
        partner.addRating(rating);
        System.out.printf("[RATING] Customer %s rated partner %s: %d/5 — \"%s\"%n",
                order.getCustomer().getName(), partner.getName(), score, comment);
    }

    // ----- Lookup helpers -----

    public Order getOrder(String orderId) {
        return getOrderOrThrow(orderId);
    }

    private Order getOrderOrThrow(String orderId) {
        Order order = orders.get(orderId);
        if (order == null) throw new NoSuchElementException("Order not found: " + orderId);
        return order;
    }

    private Restaurant getRestaurantOrThrow(String restaurantId) {
        Restaurant r = restaurants.get(restaurantId);
        if (r == null) throw new NoSuchElementException("Restaurant not found: " + restaurantId);
        return r;
    }

    private Customer getCustomerOrThrow(String customerId) {
        Customer c = customers.get(customerId);
        if (c == null) throw new NoSuchElementException("Customer not found: " + customerId);
        return c;
    }

    // ----- Reporting helpers -----

    public void printOrderSummary(String orderId) {
        Order order = getOrderOrThrow(orderId);
        System.out.println("\n========== ORDER SUMMARY ==========");
        System.out.println("Order ID  : " + order.getOrderId());
        System.out.println("Customer  : " + order.getCustomer().getName());
        System.out.println("Restaurant: " + order.getCart().getRestaurant().getName());
        System.out.println("Status    : " + order.getStatus());
        System.out.println("Total     : $" + String.format("%.2f", order.getTotalAmount()));
        System.out.println("Items     :");
        order.getCart().getItems().forEach((item, qty) ->
            System.out.printf("            %s x%d @ $%.2f%n", item.getName(), qty, item.getPrice())
        );
        if (order.getDeliveryPartner() != null) {
            System.out.println("Partner   : " + order.getDeliveryPartner().getName());
        }
        System.out.println("====================================\n");
    }
}
```

---

### 7.15 Main Demo — Full Flow

```java
// Main.java

public class Main {

    public static void main(String[] args) {

        System.out.println("============================================================");
        System.out.println("  FOOD DELIVERY SYSTEM — FULL DEMO FLOW");
        System.out.println("============================================================\n");

        // ---- 1. Bootstrap the service ----
        PricingCalculator pricing = new PricingCalculator(
                2.00,   // $2 base delivery fee
                0.50,   // $0.50 per km
                0.18    // 18% tax
        );
        FoodDeliveryService service = new FoodDeliveryService(
                new NearestDriverStrategy(),
                pricing
        );

        // ---- 2. Register a restaurant ----
        Location restaurantLoc = new Location(12.9716, 77.5946);  // Bangalore centre
        Restaurant biryaniHouse = new Restaurant("R001", "Biryani House", restaurantLoc);

        MenuItem chickenBiryani = new MenuItem("M001", "Chicken Biryani", 12.99, ItemCategory.MAIN_COURSE);
        MenuItem paneerTikka    = new MenuItem("M002", "Paneer Tikka",     8.49, ItemCategory.STARTER);
        MenuItem gulab jamun    = new MenuItem("M003", "Gulab Jamun",      3.99, ItemCategory.DESSERT);
        MenuItem masalaChai     = new MenuItem("M004", "Masala Chai",      1.99, ItemCategory.BEVERAGE);

        biryaniHouse.addMenuItem(chickenBiryani);
        biryaniHouse.addMenuItem(paneerTikka);
        biryaniHouse.addMenuItem(gulab jamun);
        biryaniHouse.addMenuItem(masalaChai);

        service.registerRestaurant(biryaniHouse);

        // ---- 3. Register a customer ----
        Location customerLoc = new Location(12.9350, 77.6240);   // ~5 km from restaurant
        Customer alice = new Customer("C001", "Alice", "+91-9876543210", customerLoc);
        service.registerCustomer(alice);

        // ---- 4. Register delivery partners ----
        Location partner1Loc = new Location(12.9650, 77.5900);  // close to restaurant
        Location partner2Loc = new Location(12.9900, 77.6100);  // further away
        DeliveryPartner ravi  = new DeliveryPartner("DP001", "Ravi",  partner1Loc);
        DeliveryPartner priya = new DeliveryPartner("DP002", "Priya", partner2Loc);

        service.registerDeliveryPartner(ravi);
        service.registerDeliveryPartner(priya);

        System.out.println();

        // ---- 5. Browse menu ----
        System.out.println("--- Browsing menu for Biryani House ---");
        service.getMenu("R001").forEach((id, item) ->
            System.out.println("  " + item));
        System.out.println();

        // ---- 6. Build a cart ----
        Cart cart = service.createCart("C001");
        cart.addItem(chickenBiryani, 2, biryaniHouse);   // 2x Chicken Biryani
        cart.addItem(paneerTikka,    1, biryaniHouse);   // 1x Paneer Tikka
        cart.addItem(masalaChai,     2, biryaniHouse);   // 2x Masala Chai

        System.out.println("--- Cart ---");
        System.out.println(cart);
        System.out.println();

        // ---- 7. Place order ----
        System.out.println("--- Placing Order ---");
        Order order = service.placeOrder(cart);
        System.out.println("Order placed: " + order);
        System.out.println("Status: " + order.getStatus());
        System.out.println();

        // ---- 8. Restaurant confirms (triggers partner assignment internally) ----
        System.out.println("--- Restaurant Confirms Order ---");
        service.confirmOrder(order.getOrderId());
        System.out.println("Status: " + order.getStatus());
        System.out.println("Assigned partner: "
                + (order.getDeliveryPartner() != null ? order.getDeliveryPartner().getName() : "none"));
        System.out.println();

        // ---- 9. Kitchen starts preparing ----
        System.out.println("--- Kitchen Starts Preparing ---");
        service.startPreparing(order.getOrderId());
        System.out.println("Status: " + order.getStatus());
        System.out.println();

        // ---- 10. Verify cancellation is blocked ----
        System.out.println("--- Attempting to cancel (should fail) ---");
        try {
            service.cancelOrder(order.getOrderId());
        } catch (IllegalStateException e) {
            System.out.println("Expected exception: " + e.getMessage());
        }
        System.out.println();

        // ---- 11. Delivery partner picks up ----
        System.out.println("--- Delivery Partner Picks Up ---");
        service.pickUpOrder(order.getOrderId());
        System.out.println("Status: " + order.getStatus());
        System.out.println();

        // ---- 12. Delivery partner delivers ----
        System.out.println("--- Order Delivered ---");
        service.deliverOrder(order.getOrderId());
        System.out.println("Status: " + order.getStatus());
        System.out.println("Partner status after delivery: "
                + order.getDeliveryPartner().getStatus());
        System.out.println();

        // ---- 13. Print full summary ----
        service.printOrderSummary(order.getOrderId());

        // ---- 14. Rate restaurant and delivery partner ----
        System.out.println("--- Rating ---");
        service.rateRestaurant(order.getOrderId(), 5, "Amazing biryani, will order again!");
        service.rateDeliveryPartner(order.getOrderId(), 4, "Quick delivery, very polite.");
        System.out.printf("Biryani House aggregate rating: %.1f%n",
                biryaniHouse.getAggregateRating());
        System.out.printf("Ravi's aggregate rating: %.1f%n",
                order.getDeliveryPartner().getAggregateRating());
        System.out.println();

        // ---- 15. Demo: switch strategy to Round Robin ----
        System.out.println("--- Switching to RoundRobin Strategy ---");
        service.setAssignmentStrategy(new RoundRobinStrategy());

        Cart cart2 = service.createCart("C001");
        cart2.addItem(paneerTikka, 3, biryaniHouse);
        Order order2 = service.placeOrder(cart2);
        service.confirmOrder(order2.getOrderId());
        System.out.println("Assigned partner (round-robin): "
                + (order2.getDeliveryPartner() != null
                        ? order2.getDeliveryPartner().getName() : "none"));
        System.out.println();

        System.out.println("============================================================");
        System.out.println("  DEMO COMPLETE");
        System.out.println("============================================================");
    }
}
```

> **Note on the demo**: The `gulab jamun` variable name contains a space in normal English; in actual Java you would use `gulabJamun` as the variable name. The code above should be compiled with `MenuItem gulabJamun = new MenuItem(...)` — the inline demo above uses the plain-English name for readability in the narrative.

---

## 8. Design Patterns Used

### 8.1 Observer Pattern

**Where**: `OrderStatusListener` interface; `CustomerNotificationListener` and `RestaurantNotificationListener` as concrete observers; `Order` as the subject.

**Why**: An order's lifecycle is inherently a broadcast event. Multiple parties (customer, restaurant, potentially the delivery partner, an analytics service) all need to react to the same state change. Hard-coding notification calls inside `Order` would create tight coupling and violate the Open/Closed Principle — every new notification channel would require modifying `Order`. With Observer, adding a new channel (e.g. `AnalyticsListener`) requires zero changes to `Order`.

**Trade-off**: Observers execute synchronously in this model. In production, notifications would be pushed onto a queue (Kafka, SNS) so that a slow SMS gateway does not block the order transaction.

---

### 8.2 State Pattern

**Where**: `OrderState` interface; `PlacedState`, `ConfirmedState`, `PreparingState`, `PickedUpState`, `DeliveredState`, `CancelledState` as concrete states; `Order` as the context.

**Why**: The order lifecycle has 6 distinct states and roughly 5 transition methods. Without the State pattern, `Order` would contain a large chain of `if (status == X) { ... } else if (status == Y) { ... }` for every method, repeated across `confirm()`, `startPreparing()`, `pickUp()`, `deliver()`, and `cancel()`. That is 5 × 6 = 30 conditions, unmaintainable and easy to misalign.

With State, each state class encapsulates only the transitions that are legal from that state. Adding a new state (e.g. `REFUND_PENDING`) means adding one class, not modifying every existing method.

**Trade-off**: More classes. For a system with more than ~10 states or very complex guard conditions, a state machine library (e.g. Spring State Machine) would be preferable.

---

### 8.3 Strategy Pattern

**Where**: `DeliveryAssignmentStrategy` interface; `NearestDriverStrategy` and `RoundRobinStrategy` as concrete strategies; `FoodDeliveryService` as the context.

**Why**: Delivery partner assignment is a business policy that can change based on time of day, city, surge pricing, or A/B testing. Encoding the algorithm inside `FoodDeliveryService` makes it impossible to swap without risk. The Strategy pattern isolates the "how to pick a partner" decision behind an interface, allowing live swaps (`setAssignmentStrategy()`) without touching any other code.

**Trade-off**: `FoodDeliveryService` must expose a setter (or accept strategies via DI), which slightly weakens immutability. The benefit of runtime configurability outweighs this in a production system.

---

### 8.4 Factory / Builder (light usage)

`FoodDeliveryService.placeOrder()` acts as a factory for `Order` objects — it aggregates the pricing calculation and observer attachment, so callers never construct `Order` directly. This ensures an order is always fully initialized (correct total, observers attached) before any state transitions can occur.

---

## 9. Key Design Decisions and Trade-offs

| Decision | Rationale | Trade-off |
|---|---|---|
| Cart is locked to a single restaurant | Matches real-world UX (DoorDash does this); simplifies pricing (one delivery fee per order) | Multi-restaurant orders require separate carts and orders |
| `Order` holds a `Cart` snapshot reference, not a copy | Simpler; suitable for an in-memory system | In production, the cart should be deep-copied (or serialized) at order time so later cart mutations don't affect the persisted order |
| State transitions throw `IllegalStateException` | Fail-fast; surfaces bugs immediately during development | In production you might prefer returning a `Result<Order, Error>` to avoid exception-driven control flow |
| `notifyListeners()` is package-visible on `Order` | State subclasses (same package) call it; external code should not | Requires careful package organization; can be replaced with a proper event bus in production |
| Distance computed with Haversine | Good enough for city-level accuracy without external APIs | Does not account for road distance or traffic; integrate with Google Maps Distance Matrix for production |
| Delivery fee based on distance from restaurant to customer | Matches user expectation (closer = cheaper) | Does not account for partner travel from current location to restaurant (pickup distance) |
| `RoundRobinStrategy` uses `AtomicInteger` | Thread-safe without synchronization overhead | Counter can overflow for extremely long-running services; use `Math.floorMod` for safety |
| Ratings stored on domain objects | Zero infrastructure needed | No persistence; ratings are lost on restart; in production store in a `ratings` table |
| `FoodDeliveryService` orchestrates partner status changes | Keeps partner status changes co-located with order assignment logic | Creates coupling; an alternative is a `DeliveryPartnerService` sub-domain |

---

## 10. Extension Points

### Add a new notification channel (e.g. push notifications)
1. Implement `OrderStatusListener`.
2. Register the new listener via `order.addListener(new PushNotificationListener())` — no other changes needed.

### Add a new assignment strategy (e.g. highest-rated partner first)
1. Implement `DeliveryAssignmentStrategy`.
2. Pass it to `FoodDeliveryService` constructor or call `setAssignmentStrategy()` — no other changes needed.

### Add a new order state (e.g. `REFUND_REQUESTED`)
1. Create `RefundRequestedState implements OrderState`.
2. Add `requestRefund()` to the `OrderState` interface (or add a default method that throws).
3. Update `DeliveredState` to allow the transition.
4. No other state needs to change.

### Add a surge pricing engine
1. Create a `SurgePricingDecorator` that wraps `PricingCalculator` and multiplies the delivery fee by a multiplier.
2. Inject it in place of the base calculator — all existing code is unaffected.

### Add persistence
1. Extract a `OrderRepository` interface with `save(Order)` and `findById(String)`.
2. Implement it with JPA/JDBC.
3. `FoodDeliveryService` depends on the interface, not on the in-memory map.

### Add a scheduled re-assignment job
If no partner accepts within N minutes, query unassigned orders and call `assignDeliveryPartner()` again — no changes to the core domain.

### Multi-item type taxes
Replace the flat `taxRate` in `PricingCalculator` with a `TaxStrategy` interface whose implementations compute tax per `ItemCategory`.

---

## 11. Interview Discussion Points

**Q: Why use State pattern instead of a simple `status` enum with conditionals?**

The State pattern encapsulates the behavior for each lifecycle phase. If you add a new state or a new transition method, you touch only the relevant state class. With enums and conditionals, every method (`confirm`, `cancel`, etc.) would need to be updated for every new state — and it's easy to forget one.

**Q: How would you make this system distributed / scalable?**

- Replace in-memory maps with a database (orders → Postgres, partners → Redis for live location).
- Replace synchronous `notifyListeners` with an event bus (Kafka topic `order-events`).
- The assignment service becomes its own microservice subscribing to `ORDER_CONFIRMED` events and publishing `PARTNER_ASSIGNED` events.
- The `Order` aggregate becomes an event-sourced entity; every state transition produces an immutable event stored in an event log.

**Q: How do you handle the case where no delivery partner is available?**

Currently `assignDeliveryPartner` returns `null` and logs a warning. In production:
- The order stays in `CONFIRMED` state without a partner.
- A background scheduler retries assignment every 30 seconds.
- After a configurable timeout, the order is auto-cancelled and the customer is notified.

**Q: How would you prevent double-assignment of a delivery partner?**

In distributed systems, use an optimistic lock or a Redis `SET NX` command to atomically claim a partner. The partner's status update and the order's partner assignment must be atomic — either a database transaction or a saga with compensating transactions.

**Q: How do you handle order cancellation after the kitchen starts?**

The `PreparingState.cancel()` throws `IllegalStateException`. In the real world you'd offer a "restaurant discretion" cancel where the restaurant pays a penalty. You'd add a `restaurantCancelWithPenalty()` method and handle compensation logic there.

**Q: What is the difference between NearestDriverStrategy and how Uber Eats actually assigns drivers?**

Real assignment considers: pickup ETA (not raw distance), driver direction of travel, predicted traffic, driver acceptance rate history, ongoing delivery proximity, and zone-based capacity. Our `NearestDriverStrategy` is a useful approximation for LLD demonstration but is far from production-grade.

**Q: How would you test the State pattern?**

Unit-test each state class in isolation:
- `PlacedState`: assert `confirm()` transitions to `CONFIRMED`; assert `pickUp()` throws.
- `PreparingState`: assert `cancel()` throws; assert `pickUp()` succeeds only when a partner is assigned.
- Mock `Order` with Mockito to verify `setState()` and `notifyListeners()` are called with correct arguments.

---

## 12. Common Mistakes in This Design

### Mistake 1: Putting transition logic inside `Order` directly
```java
// BAD
public void confirm() {
    if (status == OrderStatus.PLACED) {
        this.status = OrderStatus.CONFIRMED;
    } else if (status == OrderStatus.CONFIRMED) {
        throw new IllegalStateException("Already confirmed");
    } // ... 4 more else-ifs for each status
}
```
This grows to an unmaintainable decision tree. Use the State pattern instead — each state handles its own transitions.

---

### Mistake 2: Allowing the cart to reference items from multiple restaurants
```java
// BAD — no restaurant lock
public void addItem(MenuItem item) {
    items.put(item, items.getOrDefault(item, 0) + 1);
}
```
This makes it impossible to compute a single delivery fee or assign a single delivery partner. Always lock the cart to the first restaurant.

---

### Mistake 3: Computing price at delivery time instead of at order placement
```java
// BAD — price could have changed
public double getTotalAmount() {
    return pricingCalculator.calculateTotal(this.cart, ...);
}
```
Price must be frozen at order placement time. Menu prices can change after the order is placed. The `totalAmount` field on `Order` is set once in `FoodDeliveryService.placeOrder()` and never recomputed.

---

### Mistake 4: Calling `partner.becomeAvailable()` inside `DeliveredState` without null-checking
```java
// BAD — NPE if partner was never assigned
public void deliver(Order order) {
    order.getDeliveryPartner().becomeAvailable(); // NullPointerException
}
```
Always null-check before calling methods on `deliveryPartner`. The correct code checks `if (order.getDeliveryPartner() != null)`.

---

### Mistake 5: Making `OrderState` implementations stateful
```java
// BAD — storing mutable data in the state object
public class ConfirmedState implements OrderState {
    private int confirmationAttempts = 0;  // wrong — belongs on Order
}
```
State objects should be stateless flyweights. All mutable data (timestamps, partner reference, total amount) belongs on the `Order` aggregate, not on state objects.

---

### Mistake 6: Notifying observers inside the constructor of Order
```java
// BAD
public Order(String id, Cart cart, Customer customer, double total) {
    ...
    notifyListeners(OrderStatus.PLACED); // listeners are not attached yet!
}
```
Observers must be attached before the first notification fires. `FoodDeliveryService.placeOrder()` attaches listeners and then fires the `PLACED` notification — after construction.

---

### Mistake 7: Using `synchronized` on the entire FoodDeliveryService
```java
// BAD — bottleneck
public synchronized Order placeOrder(Cart cart) { ... }
public synchronized void confirmOrder(String id) { ... }
```
This serializes all operations across all orders. In production, use per-order locks (`ConcurrentHashMap` + lock striping) or optimistic concurrency. `ConcurrentHashMap` already provides safe map access; the locking concern is at the individual `Order` level.

---

### Mistake 8: Allowing ratings from any user, not just the order's customer
```java
// BAD — no ownership check
public void rateRestaurant(String restaurantId, String raterId, int score) {
    // Anyone can call this with any raterId
}
```
The `rateRestaurant` and `rateDeliveryPartner` methods should always be called in the context of a specific completed order, and the rater should be validated against `order.getCustomer()`.

---

*End of Food Delivery System LLD*
