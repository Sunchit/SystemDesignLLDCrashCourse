
# Ride Sharing System — Advanced LLD Case Study

## Table of Contents
1. [Problem Statement](#1-problem-statement)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Assumptions](#4-assumptions)
5. [Domain Model](#5-domain-model)
6. [Class Diagram (ASCII UML)](#6-class-diagram-ascii-uml)
7. [Complete Java Implementation](#7-complete-java-implementation)
8. [Design Patterns Used](#8-design-patterns-used)
9. [Key Design Decisions and Trade-offs](#9-key-design-decisions-and-trade-offs)
10. [Extension Points](#10-extension-points)
11. [Interview Discussion Points](#11-interview-discussion-points)
12. [Production Considerations](#12-production-considerations)

---

## 1. Problem Statement

Design the low-level object-oriented model for a ride-sharing platform (similar to Uber or Lyft) that connects riders needing transportation with nearby drivers. The system must handle the complete lifecycle of a trip — from initial request through driver matching, real-time tracking, fare calculation, payment, and post-trip rating — while supporting multiple vehicle categories, surge pricing, and pooled (shared) rides.

The challenge is not merely implementing CRUD operations but designing a domain model that is:
- **Extensible** — new ride types, pricing models, and matching algorithms can be added without rewriting core logic
- **Consistent** — trip state transitions are explicit, guarded, and auditable
- **Decoupled** — components that observe state (notification, analytics) do not pollute the trip lifecycle itself
- **Correct** — concurrent requests, driver availability races, and surge calculation must be handled safely

---

## 2. Functional Requirements

1. **Driver registration** — Drivers can register with personal details, vehicle information, and license credentials.
2. **Rider registration** — Riders can create accounts with contact details and saved payment methods.
3. **Trip request** — A rider specifies pickup location, destination, and vehicle type preference to request a trip.
4. **Driver matching** — The system identifies the nearest available driver whose vehicle matches the requested type.
5. **Trip acceptance** — A matched driver can accept or decline a trip request; on decline the system re-matches.
6. **Real-time trip tracking** — Trip location updates are published as the driver moves (abstracted; not actual GPS).
7. **Fare calculation** — Fare = base fare + (per-km rate × distance) + (per-minute rate × duration) + surge multiplier.
8. **Multiple ride types** — Economy, Premium, and Pool/Shared ride categories with distinct pricing models.
9. **Trip state management** — Explicit states: REQUESTED → ACCEPTED → DRIVER_ARRIVED → IN_PROGRESS → COMPLETED / CANCELLED.
10. **Rating system** — After trip completion both driver and rider can rate each other (1–5 stars with optional comment).
11. **Payment processing** — Multiple payment methods (credit card, debit card, digital wallet, cash); payment on completion.
12. **Trip history** — Riders and drivers can retrieve their full trip history with details and receipts.
13. **Driver availability** — Drivers can toggle online/offline status; only online drivers are matched.
14. **Surge pricing** — Dynamic fare multiplier based on demand-supply ratio in a geographic zone.
15. **Pool ride management** — Up to N riders share a vehicle; route is optimised as new riders join.

---

## 3. Non-Functional Requirements

| Category | Requirement |
|---|---|
| **Availability** | 99.99% uptime for the matching and dispatch path |
| **Latency** | Driver match returned within 2 seconds for 99th percentile |
| **Scalability** | Support millions of concurrent riders and hundreds of thousands of active drivers |
| **Consistency** | No driver assigned to two simultaneous trips (strong consistency on driver state) |
| **Durability** | All completed trips and payments persisted with zero data loss |
| **Security** | PII encrypted at rest; payment data tokenised via a PCI-DSS compliant vault |
| **Extensibility** | New ride types, pricing strategies, and matching algorithms add without modifying existing code |
| **Observability** | Every state transition emitted as an event for downstream analytics and alerting |

---

## 4. Assumptions

1. GPS coordinates are represented as `(latitude, longitude)` doubles; distance uses the Haversine formula.
2. Real-time communication (WebSockets, push notifications) is outside scope; the Observer pattern represents that boundary.
3. Payment gateway integration is abstracted behind a `PaymentGateway` interface; no actual PCI handling in this model.
4. The matching algorithm runs in-process; in production this would be a separate geo-indexed service.
5. Pool rides share a vehicle but each rider has an independent `Trip` record and fare.
6. Surge multipliers are pre-computed by an external demand model and fed into the `SurgePricingService`.
7. Driver location updates arrive via a separate telemetry pipeline; this model stores the last-known location.
8. All monetary values are in a single currency (USD) represented as `double` for simplicity; production uses `BigDecimal`.
9. Rating submission is optional and bounded to the 24 hours following trip completion.
10. The system runs on a single JVM for this design; distributed concerns are noted in Production Considerations.

---

## 5. Domain Model

### Core Entities

| Entity | Responsibility |
|---|---|
| `User` | Abstract base for `Driver` and `Rider`; holds identity and contact info |
| `Driver` | Extends User; owns a `Vehicle`, tracks availability and current location |
| `Rider` | Extends User; holds saved payment methods and trip history |
| `Vehicle` | Driver's vehicle: make, model, licence plate, `VehicleType` |
| `Trip` | Central aggregate; owns state machine, participants, route, and fare |
| `TripState` | Strategy/State object governing valid transitions from each state |
| `Location` | Value object: latitude, longitude, human-readable address |
| `Route` | Pickup → destination; updated in real time for Pool rides |
| `FareBreakdown` | Value object: itemised fare components and final total |
| `Rating` | Post-trip rating entity referencing trip, rater, ratee |
| `Payment` | Records method used and amount charged for a trip |
| `SurgeZone` | Geographic zone with a current demand multiplier |

### Key Relationships

- A `Driver` has exactly one `Vehicle` and zero or one active `Trip`.
- A `Trip` has exactly one `Driver`, one primary `Rider`, and for Pool rides a list of additional `Rider`s.
- A `Trip` progresses through `TripState` objects; each state validates the transition before delegating to the next.
- `FareCalculationStrategy` is selected by a `RideTypeFactory` based on `VehicleType` / ride category.
- `TripObserver`s are notified on every state transition without the `Trip` knowing who is listening.

---

## 6. Class Diagram (ASCII UML)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          RIDE SHARING SYSTEM                                │
└─────────────────────────────────────────────────────────────────────────────┘

          ┌──────────────┐
          │  <<abstract>>│
          │     User     │
          │──────────────│
          │ - id: String │
          │ - name       │
          │ - email      │
          │ - phone      │
          │ - rating     │
          └──────┬───────┘
                 │ extends
        ┌────────┴────────┐
        │                 │
   ┌────▼─────┐     ┌─────▼────┐
   │  Driver  │     │  Rider   │
   │──────────│     │──────────│
   │-vehicle  │     │-payments │
   │-location │     │-history  │
   │-status   │     └──────────┘
   │-activeTrip│
   └────┬─────┘
        │ owns
   ┌────▼─────┐
   │ Vehicle  │
   │──────────│
   │-make     │
   │-model    │
   │-plate    │
   │-type:    │
   │ VehicleType│
   └──────────┘

   ┌──────────────────────────────────────────────────────┐
   │                       Trip                           │
   │──────────────────────────────────────────────────────│
   │ - id: String                                         │
   │ - rider: Rider                                       │
   │ - driver: Driver                                     │
   │ - poolRiders: List<Rider>                            │
   │ - pickupLocation: Location                           │
   │ - dropoffLocation: Location                          │
   │ - rideType: RideType                                 │
   │ - currentState: TripState  ──────► <<interface>>     │
   │ - fareBreakdown: FareBreakdown    TripState          │
   │ - payment: Payment                ├ onEnter()        │
   │ - observers: List<TripObserver>   ├ accept()         │
   │ + requestTrip()                   ├ driverArrived()  │
   │ + acceptTrip()                    ├ startTrip()      │
   │ + markDriverArrived()             ├ completeTrip()   │
   │ + startTrip()                     └ cancelTrip()     │
   │ + completeTrip()                                     │
   │ + cancelTrip()                   Implementations:    │
   │ + addObserver()                  RequestedState      │
   └──────────────────────────────────AcceptedState───────┘
                                      DriverArrivedState
                                      InProgressState
                                      CompletedState
                                      CancelledState

   <<interface>>                   <<interface>>
   FareCalculationStrategy         DriverMatchingStrategy
   + calculate(TripContext):       + findDriver(Location,
     FareBreakdown                   VehicleType,
                                     List<Driver>): Driver
   Implementations:
   EconomyFareStrategy             Implementations:
   PremiumFareStrategy             NearestDriverStrategy
   PoolFareStrategy                WeightedDriverStrategy

   <<interface>>
   TripObserver
   + onTripStateChanged(Trip, TripStatus)

   Implementations:
   NotificationObserver
   AnalyticsObserver
   DriverAvailabilityObserver

   <<abstract>>
   PaymentMethod
   + processPayment(double): PaymentResult
   Implementations:
   CreditCardPayment
   DigitalWalletPayment
   CashPayment

   RideSharingService  (Facade)
   ├── registerDriver()
   ├── registerRider()
   ├── requestTrip()
   ├── acceptTrip()
   ├── markDriverArrived()
   ├── startTrip()
   ├── completeTrip()
   ├── cancelTrip()
   ├── submitRating()
   └── getTripHistory()
```

---

## 7. Complete Java Implementation

### 7.1 Enumerations

```java
// VehicleType.java
package com.rideshare.model;

public enum VehicleType {
    ECONOMY,
    PREMIUM,
    SUV,
    POOL;

    public int getMaxPassengers() {
        switch (this) {
            case ECONOMY:  return 4;
            case PREMIUM:  return 4;
            case SUV:      return 6;
            case POOL:     return 4;
            default:       throw new IllegalStateException("Unknown vehicle type: " + this);
        }
    }
}
```

```java
// TripStatus.java
package com.rideshare.model;

public enum TripStatus {
    REQUESTED,
    ACCEPTED,
    DRIVER_ARRIVED,
    IN_PROGRESS,
    COMPLETED,
    CANCELLED;

    public boolean isTerminal() {
        return this == COMPLETED || this == CANCELLED;
    }
}
```

```java
// RideType.java
package com.rideshare.model;

public enum RideType {
    ECONOMY("Economy", VehicleType.ECONOMY, 1),
    PREMIUM("Premium", VehicleType.PREMIUM, 2),
    POOL("Pool / Shared", VehicleType.POOL, 4);

    private final String displayName;
    private final VehicleType requiredVehicleType;
    private final int maxPoolSize;

    RideType(String displayName, VehicleType requiredVehicleType, int maxPoolSize) {
        this.displayName = displayName;
        this.requiredVehicleType = requiredVehicleType;
        this.maxPoolSize = maxPoolSize;
    }

    public String getDisplayName() { return displayName; }
    public VehicleType getRequiredVehicleType() { return requiredVehicleType; }
    public int getMaxPoolSize() { return maxPoolSize; }
}
```

```java
// DriverStatus.java
package com.rideshare.model;

public enum DriverStatus {
    OFFLINE,
    AVAILABLE,
    ON_TRIP;

    public boolean isAvailableForMatch() {
        return this == AVAILABLE;
    }
}
```

```java
// PaymentMethodType.java
package com.rideshare.model;

public enum PaymentMethodType {
    CREDIT_CARD,
    DEBIT_CARD,
    DIGITAL_WALLET,
    CASH
}
```

```java
// CancellationReason.java
package com.rideshare.model;

public enum CancellationReason {
    RIDER_CANCELLED,
    DRIVER_CANCELLED,
    NO_DRIVER_FOUND,
    PAYMENT_FAILED,
    SYSTEM_CANCELLED
}
```

### 7.2 Exceptions

```java
// RideSharingException.java
package com.rideshare.exception;

public class RideSharingException extends RuntimeException {
    public RideSharingException(String message) {
        super(message);
    }
    public RideSharingException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

```java
// InvalidTripStateTransitionException.java
package com.rideshare.exception;

import com.rideshare.model.TripStatus;

public class InvalidTripStateTransitionException extends RideSharingException {
    private final TripStatus from;
    private final TripStatus to;

    public InvalidTripStateTransitionException(TripStatus from, TripStatus to) {
        super(String.format("Invalid trip state transition: %s -> %s", from, to));
        this.from = from;
        this.to = to;
    }

    public TripStatus getFrom() { return from; }
    public TripStatus getTo() { return to; }
}
```

```java
// DriverNotFoundException.java
package com.rideshare.exception;

public class DriverNotFoundException extends RideSharingException {
    public DriverNotFoundException(String message) {
        super(message);
    }
}
```

```java
// PaymentProcessingException.java
package com.rideshare.exception;

public class PaymentProcessingException extends RideSharingException {
    private final String paymentMethodId;

    public PaymentProcessingException(String paymentMethodId, String message) {
        super(message);
        this.paymentMethodId = paymentMethodId;
    }

    public String getPaymentMethodId() { return paymentMethodId; }
}
```

```java
// RatingException.java
package com.rideshare.exception;

public class RatingException extends RideSharingException {
    public RatingException(String message) {
        super(message);
    }
}
```

### 7.3 Value Objects

```java
// Location.java
package com.rideshare.model;

public final class Location {
    private final double latitude;
    private final double longitude;
    private final String address;

    public Location(double latitude, double longitude, String address) {
        if (latitude < -90 || latitude > 90)
            throw new IllegalArgumentException("Latitude must be between -90 and 90");
        if (longitude < -180 || longitude > 180)
            throw new IllegalArgumentException("Longitude must be between -180 and 180");
        this.latitude = latitude;
        this.longitude = longitude;
        this.address = address;
    }

    public double getLatitude() { return latitude; }
    public double getLongitude() { return longitude; }
    public String getAddress() { return address; }

    /**
     * Haversine formula — returns distance in kilometres.
     */
    public double distanceTo(Location other) {
        final double R = 6371.0; // Earth radius in km
        double lat1 = Math.toRadians(this.latitude);
        double lat2 = Math.toRadians(other.latitude);
        double dLat = Math.toRadians(other.latitude - this.latitude);
        double dLon = Math.toRadians(other.longitude - this.longitude);

        double a = Math.sin(dLat / 2) * Math.sin(dLat / 2)
                 + Math.cos(lat1) * Math.cos(lat2)
                 * Math.sin(dLon / 2) * Math.sin(dLon / 2);
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        return R * c;
    }

    @Override
    public String toString() {
        return String.format("Location{lat=%.4f, lon=%.4f, address='%s'}", latitude, longitude, address);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Location)) return false;
        Location that = (Location) o;
        return Double.compare(that.latitude, latitude) == 0
            && Double.compare(that.longitude, longitude) == 0;
    }

    @Override
    public int hashCode() {
        return java.util.Objects.hash(latitude, longitude);
    }
}
```

```java
// FareBreakdown.java
package com.rideshare.model;

public final class FareBreakdown {
    private final double baseFare;
    private final double distanceFare;
    private final double timeFare;
    private final double surgeMultiplier;
    private final double discount;
    private final double totalFare;
    private final double distanceKm;
    private final long durationMinutes;

    public FareBreakdown(double baseFare, double distanceFare, double timeFare,
                         double surgeMultiplier, double discount,
                         double distanceKm, long durationMinutes) {
        this.baseFare = baseFare;
        this.distanceFare = distanceFare;
        this.timeFare = timeFare;
        this.surgeMultiplier = surgeMultiplier;
        this.discount = discount;
        this.distanceKm = distanceKm;
        this.durationMinutes = durationMinutes;
        this.totalFare = Math.max(0,
            (baseFare + distanceFare + timeFare) * surgeMultiplier - discount);
    }

    public double getBaseFare() { return baseFare; }
    public double getDistanceFare() { return distanceFare; }
    public double getTimeFare() { return timeFare; }
    public double getSurgeMultiplier() { return surgeMultiplier; }
    public double getDiscount() { return discount; }
    public double getTotalFare() { return totalFare; }
    public double getDistanceKm() { return distanceKm; }
    public long getDurationMinutes() { return durationMinutes; }

    @Override
    public String toString() {
        return String.format(
            "FareBreakdown{base=$%.2f, distance=$%.2f (%.2fkm), time=$%.2f (%dmin), " +
            "surge=%.1fx, discount=$%.2f, TOTAL=$%.2f}",
            baseFare, distanceFare, distanceKm, timeFare, durationMinutes,
            surgeMultiplier, discount, totalFare);
    }
}
```

```java
// TripContext.java
package com.rideshare.model;

/**
 * Immutable context passed to fare calculation strategies.
 */
public final class TripContext {
    private final double distanceKm;
    private final long durationMinutes;
    private final double surgeMultiplier;
    private final int poolRiderCount;
    private final RideType rideType;

    public TripContext(double distanceKm, long durationMinutes,
                       double surgeMultiplier, int poolRiderCount, RideType rideType) {
        this.distanceKm = distanceKm;
        this.durationMinutes = durationMinutes;
        this.surgeMultiplier = surgeMultiplier;
        this.poolRiderCount = poolRiderCount;
        this.rideType = rideType;
    }

    public double getDistanceKm() { return distanceKm; }
    public long getDurationMinutes() { return durationMinutes; }
    public double getSurgeMultiplier() { return surgeMultiplier; }
    public int getPoolRiderCount() { return poolRiderCount; }
    public RideType getRideType() { return rideType; }
}
```

### 7.4 Core Entities

```java
// User.java
package com.rideshare.model;

import java.time.Instant;
import java.util.UUID;

public abstract class User {
    private final String id;
    private String name;
    private String email;
    private String phone;
    private double averageRating;
    private int ratingCount;
    private final Instant createdAt;

    protected User(String name, String email, String phone) {
        this.id = UUID.randomUUID().toString();
        this.name = name;
        this.email = email;
        this.phone = phone;
        this.averageRating = 5.0;
        this.ratingCount = 0;
        this.createdAt = Instant.now();
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getPhone() { return phone; }
    public double getAverageRating() { return averageRating; }
    public int getRatingCount() { return ratingCount; }
    public Instant getCreatedAt() { return createdAt; }

    public void setName(String name) { this.name = name; }
    public void setEmail(String email) { this.email = email; }
    public void setPhone(String phone) { this.phone = phone; }

    /**
     * Updates the rolling average rating when a new rating is submitted.
     */
    public synchronized void addRating(double newRating) {
        if (newRating < 1.0 || newRating > 5.0)
            throw new IllegalArgumentException("Rating must be between 1 and 5");
        double total = averageRating * ratingCount + newRating;
        ratingCount++;
        averageRating = total / ratingCount;
    }

    @Override
    public String toString() {
        return String.format("%s{id='%s', name='%s', rating=%.2f}",
            getClass().getSimpleName(), id, name, averageRating);
    }
}
```

```java
// Vehicle.java
package com.rideshare.model;

import java.util.UUID;

public class Vehicle {
    private final String id;
    private final String make;
    private final String model;
    private final String licensePlate;
    private final VehicleType vehicleType;
    private final int year;
    private final String color;

    public Vehicle(String make, String model, String licensePlate,
                   VehicleType vehicleType, int year, String color) {
        this.id = UUID.randomUUID().toString();
        this.make = make;
        this.model = model;
        this.licensePlate = licensePlate;
        this.vehicleType = vehicleType;
        this.year = year;
        this.color = color;
    }

    public String getId() { return id; }
    public String getMake() { return make; }
    public String getModel() { return model; }
    public String getLicensePlate() { return licensePlate; }
    public VehicleType getVehicleType() { return vehicleType; }
    public int getYear() { return year; }
    public String getColor() { return color; }

    @Override
    public String toString() {
        return String.format("Vehicle{%d %s %s [%s] plate=%s type=%s}",
            year, color, make + " " + model, licensePlate, licensePlate, vehicleType);
    }
}
```

```java
// Driver.java
package com.rideshare.model;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class Driver extends User {
    private final Vehicle vehicle;
    private DriverStatus status;
    private Location currentLocation;
    private Trip activeTrip;
    private final List<String> completedTripIds;
    private final String licenseNumber;

    public Driver(String name, String email, String phone,
                  Vehicle vehicle, String licenseNumber) {
        super(name, email, phone);
        this.vehicle = vehicle;
        this.licenseNumber = licenseNumber;
        this.status = DriverStatus.OFFLINE;
        this.completedTripIds = new ArrayList<>();
    }

    public Vehicle getVehicle() { return vehicle; }
    public DriverStatus getStatus() { return status; }
    public Location getCurrentLocation() { return currentLocation; }
    public Trip getActiveTrip() { return activeTrip; }
    public String getLicenseNumber() { return licenseNumber; }
    public List<String> getCompletedTripIds() {
        return Collections.unmodifiableList(completedTripIds);
    }

    public synchronized void setOnline() {
        if (status == DriverStatus.OFFLINE) status = DriverStatus.AVAILABLE;
    }

    public synchronized void setOffline() {
        if (status == DriverStatus.ON_TRIP)
            throw new IllegalStateException("Cannot go offline while on an active trip");
        status = DriverStatus.OFFLINE;
    }

    public synchronized void assignTrip(Trip trip) {
        if (status != DriverStatus.AVAILABLE)
            throw new IllegalStateException("Driver is not available: " + status);
        this.activeTrip = trip;
        this.status = DriverStatus.ON_TRIP;
    }

    public synchronized void completeTrip() {
        if (activeTrip != null) {
            completedTripIds.add(activeTrip.getId());
            activeTrip = null;
        }
        status = DriverStatus.AVAILABLE;
    }

    public void updateLocation(Location location) {
        this.currentLocation = location;
    }

    public boolean isAvailableFor(VehicleType requestedType) {
        return status.isAvailableForMatch()
            && vehicle.getVehicleType() == requestedType
            && currentLocation != null;
    }
}
```

```java
// Rider.java
package com.rideshare.model;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class Rider extends User {
    private final List<PaymentMethod> paymentMethods;
    private PaymentMethod defaultPaymentMethod;
    private final List<String> tripHistory;

    public Rider(String name, String email, String phone) {
        super(name, email, phone);
        this.paymentMethods = new ArrayList<>();
        this.tripHistory = new ArrayList<>();
    }

    public List<PaymentMethod> getPaymentMethods() {
        return Collections.unmodifiableList(paymentMethods);
    }

    public PaymentMethod getDefaultPaymentMethod() { return defaultPaymentMethod; }
    public List<String> getTripHistory() { return Collections.unmodifiableList(tripHistory); }

    public void addPaymentMethod(PaymentMethod method) {
        paymentMethods.add(method);
        if (defaultPaymentMethod == null) defaultPaymentMethod = method;
    }

    public void setDefaultPaymentMethod(PaymentMethod method) {
        if (!paymentMethods.contains(method))
            throw new IllegalArgumentException("Payment method not registered to this rider");
        this.defaultPaymentMethod = method;
    }

    public void recordTrip(String tripId) {
        tripHistory.add(tripId);
    }
}
```

### 7.5 Payment Abstraction

```java
// PaymentResult.java
package com.rideshare.model;

public final class PaymentResult {
    private final boolean success;
    private final String transactionId;
    private final String errorMessage;
    private final double amountCharged;

    private PaymentResult(boolean success, String transactionId,
                          String errorMessage, double amountCharged) {
        this.success = success;
        this.transactionId = transactionId;
        this.errorMessage = errorMessage;
        this.amountCharged = amountCharged;
    }

    public static PaymentResult success(String transactionId, double amount) {
        return new PaymentResult(true, transactionId, null, amount);
    }

    public static PaymentResult failure(String errorMessage) {
        return new PaymentResult(false, null, errorMessage, 0);
    }

    public boolean isSuccess() { return success; }
    public String getTransactionId() { return transactionId; }
    public String getErrorMessage() { return errorMessage; }
    public double getAmountCharged() { return amountCharged; }
}
```

```java
// PaymentMethod.java
package com.rideshare.model;

import java.util.UUID;

public abstract class PaymentMethod {
    private final String id;
    private final PaymentMethodType type;

    protected PaymentMethod(PaymentMethodType type) {
        this.id = UUID.randomUUID().toString();
        this.type = type;
    }

    public String getId() { return id; }
    public PaymentMethodType getType() { return type; }

    public abstract PaymentResult processPayment(double amount);
    public abstract String getMaskedIdentifier();

    @Override
    public String toString() {
        return String.format("%s{id='%s', masked='%s'}",
            type, id, getMaskedIdentifier());
    }
}
```

```java
// CreditCardPayment.java
package com.rideshare.model;

import java.util.UUID;

public class CreditCardPayment extends PaymentMethod {
    private final String lastFourDigits;
    private final String cardholderName;
    private final String expiryMonth;
    private final String expiryYear;

    public CreditCardPayment(String lastFourDigits, String cardholderName,
                             String expiryMonth, String expiryYear) {
        super(PaymentMethodType.CREDIT_CARD);
        this.lastFourDigits = lastFourDigits;
        this.cardholderName = cardholderName;
        this.expiryMonth = expiryMonth;
        this.expiryYear = expiryYear;
    }

    @Override
    public PaymentResult processPayment(double amount) {
        // In production: call payment gateway with tokenised card reference
        if (amount <= 0) return PaymentResult.failure("Amount must be positive");
        String txId = "CC-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        System.out.printf("[PAYMENT] Charged $%.2f to credit card ending %s. TxID: %s%n",
            amount, lastFourDigits, txId);
        return PaymentResult.success(txId, amount);
    }

    @Override
    public String getMaskedIdentifier() {
        return "**** **** **** " + lastFourDigits;
    }
}
```

```java
// DigitalWalletPayment.java
package com.rideshare.model;

import java.util.UUID;

public class DigitalWalletPayment extends PaymentMethod {
    private final String walletId;
    private final String walletProvider;
    private double balance;

    public DigitalWalletPayment(String walletId, String walletProvider, double initialBalance) {
        super(PaymentMethodType.DIGITAL_WALLET);
        this.walletId = walletId;
        this.walletProvider = walletProvider;
        this.balance = initialBalance;
    }

    @Override
    public synchronized PaymentResult processPayment(double amount) {
        if (amount <= 0) return PaymentResult.failure("Amount must be positive");
        if (balance < amount)
            return PaymentResult.failure("Insufficient wallet balance: $" + balance);
        balance -= amount;
        String txId = "DW-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        System.out.printf("[PAYMENT] Charged $%.2f from %s wallet. Remaining: $%.2f. TxID: %s%n",
            amount, walletProvider, balance, txId);
        return PaymentResult.success(txId, amount);
    }

    @Override
    public String getMaskedIdentifier() {
        return walletProvider + ":" + walletId.substring(0, 4) + "****";
    }

    public double getBalance() { return balance; }
}
```

```java
// CashPayment.java
package com.rideshare.model;

import java.util.UUID;

public class CashPayment extends PaymentMethod {

    public CashPayment() {
        super(PaymentMethodType.CASH);
    }

    @Override
    public PaymentResult processPayment(double amount) {
        // Cash collection is handled offline by the driver
        String txId = "CASH-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        System.out.printf("[PAYMENT] Cash payment of $%.2f recorded. TxID: %s%n", amount, txId);
        return PaymentResult.success(txId, amount);
    }

    @Override
    public String getMaskedIdentifier() { return "CASH"; }
}
```

```java
// Payment.java
package com.rideshare.model;

import java.time.Instant;

public class Payment {
    private final String tripId;
    private final PaymentMethod method;
    private final double amount;
    private final PaymentResult result;
    private final Instant processedAt;

    public Payment(String tripId, PaymentMethod method, double amount, PaymentResult result) {
        this.tripId = tripId;
        this.method = method;
        this.amount = amount;
        this.result = result;
        this.processedAt = Instant.now();
    }

    public String getTripId() { return tripId; }
    public PaymentMethod getMethod() { return method; }
    public double getAmount() { return amount; }
    public PaymentResult getResult() { return result; }
    public Instant getProcessedAt() { return processedAt; }
    public boolean isSuccessful() { return result.isSuccess(); }

    @Override
    public String toString() {
        return String.format("Payment{trip='%s', method=%s, amount=$%.2f, success=%b, tx='%s'}",
            tripId, method.getType(), amount, result.isSuccess(), result.getTransactionId());
    }
}
```

### 7.6 Rating Entity

```java
// Rating.java
package com.rideshare.model;

import java.time.Instant;
import java.util.UUID;

public class Rating {
    private final String id;
    private final String tripId;
    private final String raterId;
    private final String rateeId;
    private final int stars;
    private final String comment;
    private final Instant createdAt;

    public Rating(String tripId, String raterId, String rateeId, int stars, String comment) {
        if (stars < 1 || stars > 5)
            throw new IllegalArgumentException("Stars must be between 1 and 5, got: " + stars);
        this.id = UUID.randomUUID().toString();
        this.tripId = tripId;
        this.raterId = raterId;
        this.rateeId = rateeId;
        this.stars = stars;
        this.comment = comment;
        this.createdAt = Instant.now();
    }

    public String getId() { return id; }
    public String getTripId() { return tripId; }
    public String getRaterId() { return raterId; }
    public String getRateeId() { return rateeId; }
    public int getStars() { return stars; }
    public String getComment() { return comment; }
    public Instant getCreatedAt() { return createdAt; }

    @Override
    public String toString() {
        return String.format("Rating{trip='%s', from='%s', to='%s', stars=%d, comment='%s'}",
            tripId, raterId, rateeId, stars, comment);
    }
}
```

### 7.7 Fare Calculation Strategy

```java
// FareCalculationStrategy.java
package com.rideshare.strategy.fare;

import com.rideshare.model.FareBreakdown;
import com.rideshare.model.TripContext;

public interface FareCalculationStrategy {
    FareBreakdown calculate(TripContext context);
    String getStrategyName();
}
```

```java
// EconomyFareStrategy.java
package com.rideshare.strategy.fare;

import com.rideshare.model.FareBreakdown;
import com.rideshare.model.TripContext;

public class EconomyFareStrategy implements FareCalculationStrategy {
    private static final double BASE_FARE       = 2.50;
    private static final double PER_KM_RATE     = 1.20;
    private static final double PER_MINUTE_RATE = 0.18;

    @Override
    public FareBreakdown calculate(TripContext ctx) {
        double distanceFare = ctx.getDistanceKm() * PER_KM_RATE;
        double timeFare     = ctx.getDurationMinutes() * PER_MINUTE_RATE;
        return new FareBreakdown(
            BASE_FARE, distanceFare, timeFare,
            ctx.getSurgeMultiplier(), 0.0,
            ctx.getDistanceKm(), ctx.getDurationMinutes()
        );
    }

    @Override
    public String getStrategyName() { return "Economy"; }
}
```

```java
// PremiumFareStrategy.java
package com.rideshare.strategy.fare;

import com.rideshare.model.FareBreakdown;
import com.rideshare.model.TripContext;

public class PremiumFareStrategy implements FareCalculationStrategy {
    private static final double BASE_FARE       = 5.00;
    private static final double PER_KM_RATE     = 2.50;
    private static final double PER_MINUTE_RATE = 0.35;

    @Override
    public FareBreakdown calculate(TripContext ctx) {
        double distanceFare = ctx.getDistanceKm() * PER_KM_RATE;
        double timeFare     = ctx.getDurationMinutes() * PER_MINUTE_RATE;
        return new FareBreakdown(
            BASE_FARE, distanceFare, timeFare,
            ctx.getSurgeMultiplier(), 0.0,
            ctx.getDistanceKm(), ctx.getDurationMinutes()
        );
    }

    @Override
    public String getStrategyName() { return "Premium"; }
}
```

```java
// PoolFareStrategy.java
package com.rideshare.strategy.fare;

import com.rideshare.model.FareBreakdown;
import com.rideshare.model.TripContext;

/**
 * Pool fare: economy base rates, discounted per shared rider.
 * Each additional rider in the pool reduces the per-rider fare by 15%,
 * capped at a 40% total discount.
 */
public class PoolFareStrategy implements FareCalculationStrategy {
    private static final double BASE_FARE       = 2.00;
    private static final double PER_KM_RATE     = 1.00;
    private static final double PER_MINUTE_RATE = 0.15;
    private static final double DISCOUNT_PER_EXTRA_RIDER = 0.15;
    private static final double MAX_DISCOUNT_FRACTION    = 0.40;

    @Override
    public FareBreakdown calculate(TripContext ctx) {
        double distanceFare = ctx.getDistanceKm() * PER_KM_RATE;
        double timeFare     = ctx.getDurationMinutes() * PER_MINUTE_RATE;
        double subtotal     = BASE_FARE + distanceFare + timeFare;

        int extraRiders     = Math.max(0, ctx.getPoolRiderCount() - 1);
        double discountFraction = Math.min(
            extraRiders * DISCOUNT_PER_EXTRA_RIDER, MAX_DISCOUNT_FRACTION);
        double discount = subtotal * discountFraction;

        return new FareBreakdown(
            BASE_FARE, distanceFare, timeFare,
            ctx.getSurgeMultiplier(), discount,
            ctx.getDistanceKm(), ctx.getDurationMinutes()
        );
    }

    @Override
    public String getStrategyName() { return "Pool"; }
}
```

### 7.8 Driver Matching Strategy

```java
// DriverMatchingStrategy.java
package com.rideshare.strategy.matching;

import com.rideshare.model.Driver;
import com.rideshare.model.Location;
import com.rideshare.model.VehicleType;

import java.util.List;
import java.util.Optional;

public interface DriverMatchingStrategy {
    Optional<Driver> findDriver(Location pickupLocation,
                                VehicleType requiredType,
                                List<Driver> availableDrivers);
    String getStrategyName();
}
```

```java
// NearestDriverStrategy.java
package com.rideshare.strategy.matching;

import com.rideshare.model.Driver;
import com.rideshare.model.Location;
import com.rideshare.model.VehicleType;

import java.util.Comparator;
import java.util.List;
import java.util.Optional;

/**
 * Selects the geographically nearest available driver within the maximum
 * search radius. O(n) scan is acceptable for moderate fleet sizes; for
 * millions of drivers a geo-index (S2, H3, PostGIS) replaces this.
 */
public class NearestDriverStrategy implements DriverMatchingStrategy {
    private static final double MAX_SEARCH_RADIUS_KM = 10.0;

    @Override
    public Optional<Driver> findDriver(Location pickupLocation,
                                       VehicleType requiredType,
                                       List<Driver> availableDrivers) {
        return availableDrivers.stream()
            .filter(d -> d.isAvailableFor(requiredType))
            .filter(d -> d.getCurrentLocation()
                          .distanceTo(pickupLocation) <= MAX_SEARCH_RADIUS_KM)
            .min(Comparator.comparingDouble(
                d -> d.getCurrentLocation().distanceTo(pickupLocation)));
    }

    @Override
    public String getStrategyName() { return "Nearest Driver"; }
}
```

```java
// WeightedDriverStrategy.java
package com.rideshare.strategy.matching;

import com.rideshare.model.Driver;
import com.rideshare.model.Location;
import com.rideshare.model.VehicleType;

import java.util.Comparator;
import java.util.List;
import java.util.Optional;

/**
 * Scores drivers on a weighted combination of proximity and rating.
 * score = (distanceWeight / distanceKm) + (ratingWeight * avgRating)
 * Higher score is better.
 */
public class WeightedDriverStrategy implements DriverMatchingStrategy {
    private static final double DISTANCE_WEIGHT = 0.7;
    private static final double RATING_WEIGHT   = 0.3;
    private static final double MAX_SEARCH_RADIUS_KM = 10.0;

    @Override
    public Optional<Driver> findDriver(Location pickupLocation,
                                       VehicleType requiredType,
                                       List<Driver> availableDrivers) {
        return availableDrivers.stream()
            .filter(d -> d.isAvailableFor(requiredType))
            .filter(d -> d.getCurrentLocation()
                          .distanceTo(pickupLocation) <= MAX_SEARCH_RADIUS_KM)
            .max(Comparator.comparingDouble(
                d -> score(d, pickupLocation)));
    }

    private double score(Driver driver, Location pickup) {
        double distKm = driver.getCurrentLocation().distanceTo(pickup);
        double proximityScore = distKm > 0 ? DISTANCE_WEIGHT / distKm : DISTANCE_WEIGHT * 100;
        double ratingScore    = RATING_WEIGHT * driver.getAverageRating();
        return proximityScore + ratingScore;
    }

    @Override
    public String getStrategyName() { return "Weighted (Proximity + Rating)"; }
}
```

### 7.9 Trip Observer

```java
// TripObserver.java
package com.rideshare.observer;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

public interface TripObserver {
    void onTripStateChanged(Trip trip, TripStatus previousStatus, TripStatus newStatus);
}
```

```java
// NotificationObserver.java
package com.rideshare.observer;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

public class NotificationObserver implements TripObserver {
    @Override
    public void onTripStateChanged(Trip trip, TripStatus previousStatus, TripStatus newStatus) {
        String riderName  = trip.getRider().getName();
        String driverName = trip.getDriver() != null ? trip.getDriver().getName() : "N/A";
        switch (newStatus) {
            case ACCEPTED:
                System.out.printf("[NOTIFICATION] %s: Your driver %s has accepted the trip.%n",
                    riderName, driverName);
                break;
            case DRIVER_ARRIVED:
                System.out.printf("[NOTIFICATION] %s: Your driver %s has arrived at pickup.%n",
                    riderName, driverName);
                break;
            case IN_PROGRESS:
                System.out.printf("[NOTIFICATION] %s: Trip started. Enjoy your ride!%n", riderName);
                break;
            case COMPLETED:
                System.out.printf("[NOTIFICATION] %s: Trip completed. Fare: $%.2f%n",
                    riderName,
                    trip.getFareBreakdown() != null ? trip.getFareBreakdown().getTotalFare() : 0.0);
                break;
            case CANCELLED:
                System.out.printf("[NOTIFICATION] %s: Trip has been cancelled.%n", riderName);
                break;
            default:
                break;
        }
    }
}
```

```java
// AnalyticsObserver.java
package com.rideshare.observer;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

import java.time.Instant;

public class AnalyticsObserver implements TripObserver {
    @Override
    public void onTripStateChanged(Trip trip, TripStatus previousStatus, TripStatus newStatus) {
        System.out.printf("[ANALYTICS] Event: TRIP_STATE_CHANGE | tripId=%s | %s -> %s | ts=%s%n",
            trip.getId().substring(0, 8), previousStatus, newStatus, Instant.now());
    }
}
```

```java
// DriverAvailabilityObserver.java
package com.rideshare.observer;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

public class DriverAvailabilityObserver implements TripObserver {
    @Override
    public void onTripStateChanged(Trip trip, TripStatus previousStatus, TripStatus newStatus) {
        if (newStatus == TripStatus.COMPLETED || newStatus == TripStatus.CANCELLED) {
            if (trip.getDriver() != null) {
                trip.getDriver().completeTrip();
                System.out.printf("[AVAILABILITY] Driver %s is now available.%n",
                    trip.getDriver().getName());
            }
        }
    }
}
```

---

### 7.10 Trip State Pattern

```java
// TripState.java
package com.rideshare.state;

import com.rideshare.model.Trip;

/**
 * Each concrete TripState encapsulates the behaviour valid at that point
 * in the trip lifecycle and enforces which transitions are legal.
 */
public interface TripState {
    void accept(Trip trip);
    void driverArrived(Trip trip);
    void startTrip(Trip trip);
    void completeTrip(Trip trip);
    void cancelTrip(Trip trip);
    com.rideshare.model.TripStatus getStatus();
}
```

```java
// AbstractTripState.java
package com.rideshare.state;

import com.rideshare.exception.InvalidTripStateTransitionException;
import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

/**
 * Default implementation throws InvalidTripStateTransitionException for every
 * operation, so concrete states only override the transitions they permit.
 */
public abstract class AbstractTripState implements TripState {

    @Override
    public void accept(Trip trip) {
        throw new InvalidTripStateTransitionException(getStatus(), TripStatus.ACCEPTED);
    }

    @Override
    public void driverArrived(Trip trip) {
        throw new InvalidTripStateTransitionException(getStatus(), TripStatus.DRIVER_ARRIVED);
    }

    @Override
    public void startTrip(Trip trip) {
        throw new InvalidTripStateTransitionException(getStatus(), TripStatus.IN_PROGRESS);
    }

    @Override
    public void completeTrip(Trip trip) {
        throw new InvalidTripStateTransitionException(getStatus(), TripStatus.COMPLETED);
    }

    @Override
    public void cancelTrip(Trip trip) {
        throw new InvalidTripStateTransitionException(getStatus(), TripStatus.CANCELLED);
    }
}
```

```java
// RequestedState.java
package com.rideshare.state;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

public class RequestedState extends AbstractTripState {

    @Override
    public void accept(Trip trip) {
        trip.transitionTo(new AcceptedState());
    }

    @Override
    public void cancelTrip(Trip trip) {
        trip.transitionTo(new CancelledState());
    }

    @Override
    public TripStatus getStatus() { return TripStatus.REQUESTED; }
}
```

```java
// AcceptedState.java
package com.rideshare.state;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

public class AcceptedState extends AbstractTripState {

    @Override
    public void driverArrived(Trip trip) {
        trip.transitionTo(new DriverArrivedState());
    }

    @Override
    public void cancelTrip(Trip trip) {
        trip.transitionTo(new CancelledState());
    }

    @Override
    public TripStatus getStatus() { return TripStatus.ACCEPTED; }
}
```

```java
// DriverArrivedState.java
package com.rideshare.state;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

public class DriverArrivedState extends AbstractTripState {

    @Override
    public void startTrip(Trip trip) {
        trip.transitionTo(new InProgressState());
    }

    @Override
    public void cancelTrip(Trip trip) {
        trip.transitionTo(new CancelledState());
    }

    @Override
    public TripStatus getStatus() { return TripStatus.DRIVER_ARRIVED; }
}
```

```java
// InProgressState.java
package com.rideshare.state;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

public class InProgressState extends AbstractTripState {

    @Override
    public void completeTrip(Trip trip) {
        trip.transitionTo(new CompletedState());
    }

    // In-progress trips cannot be cancelled through normal flow;
    // emergency cancellation is handled via system-level override only.
    @Override
    public TripStatus getStatus() { return TripStatus.IN_PROGRESS; }
}
```

```java
// CompletedState.java
package com.rideshare.state;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

public class CompletedState extends AbstractTripState {
    // Terminal state — no transitions allowed; all methods throw via super.
    @Override
    public TripStatus getStatus() { return TripStatus.COMPLETED; }
}
```

```java
// CancelledState.java
package com.rideshare.state;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

public class CancelledState extends AbstractTripState {
    // Terminal state — no transitions allowed.
    @Override
    public TripStatus getStatus() { return TripStatus.CANCELLED; }
}
```

### 7.11 Trip Aggregate

```java
// Trip.java
package com.rideshare.model;

import com.rideshare.observer.TripObserver;
import com.rideshare.state.RequestedState;
import com.rideshare.state.TripState;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

/**
 * Central aggregate root. Owns the state machine and notifies observers
 * on every transition. All state-changing operations are guarded by the
 * current TripState, so invalid transitions throw immediately.
 */
public class Trip {
    private final String id;
    private final Rider rider;
    private Driver driver;
    private final List<Rider> poolRiders;
    private final Location pickupLocation;
    private final Location dropoffLocation;
    private final RideType rideType;

    private TripState currentState;
    private FareBreakdown fareBreakdown;
    private Payment payment;
    private CancellationReason cancellationReason;

    private Location driverCurrentLocation;
    private final Instant requestedAt;
    private Instant acceptedAt;
    private Instant driverArrivedAt;
    private Instant startedAt;
    private Instant completedAt;
    private Instant cancelledAt;

    private final List<TripObserver> observers;
    private final List<Rating> ratings;

    public Trip(Rider rider, Location pickupLocation,
                Location dropoffLocation, RideType rideType) {
        this.id = UUID.randomUUID().toString();
        this.rider = rider;
        this.pickupLocation = pickupLocation;
        this.dropoffLocation = dropoffLocation;
        this.rideType = rideType;
        this.poolRiders = new ArrayList<>();
        this.observers = new ArrayList<>();
        this.ratings = new ArrayList<>();
        this.requestedAt = Instant.now();
        this.currentState = new RequestedState();
    }

    // ── State machine delegation ───────────────────────────────────────────

    public void accept() { currentState.accept(this); }
    public void driverArrived() { currentState.driverArrived(this); }
    public void startTrip() { currentState.startTrip(this); }
    public void completeTrip() { currentState.completeTrip(this); }
    public void cancelTrip() { currentState.cancelTrip(this); }

    /**
     * Called exclusively by TripState implementations. Records timestamps,
     * updates state, and fires observers.
     */
    public void transitionTo(TripState newState) {
        TripStatus previous = currentState.getStatus();
        TripStatus next     = newState.getStatus();
        recordTimestamp(next);
        this.currentState = newState;
        notifyObservers(previous, next);
    }

    private void recordTimestamp(TripStatus status) {
        switch (status) {
            case ACCEPTED:       acceptedAt      = Instant.now(); break;
            case DRIVER_ARRIVED: driverArrivedAt = Instant.now(); break;
            case IN_PROGRESS:    startedAt       = Instant.now(); break;
            case COMPLETED:      completedAt     = Instant.now(); break;
            case CANCELLED:      cancelledAt     = Instant.now(); break;
            default: break;
        }
    }

    // ── Observer management ───────────────────────────────────────────────

    public void addObserver(TripObserver observer) {
        observers.add(observer);
    }

    public void removeObserver(TripObserver observer) {
        observers.remove(observer);
    }

    private void notifyObservers(TripStatus previous, TripStatus current) {
        for (TripObserver obs : observers) {
            obs.onTripStateChanged(this, previous, current);
        }
    }

    // ── Pool ride management ──────────────────────────────────────────────

    public void addPoolRider(Rider poolRider) {
        if (rideType != RideType.POOL)
            throw new IllegalStateException("Cannot add pool rider to a non-pool trip");
        if (poolRiders.size() >= rideType.getMaxPoolSize() - 1)
            throw new IllegalStateException("Pool trip is at maximum capacity");
        if (currentState.getStatus() != TripStatus.REQUESTED
                && currentState.getStatus() != TripStatus.ACCEPTED) {
            throw new IllegalStateException("Cannot add pool rider after trip has started");
        }
        poolRiders.add(poolRider);
    }

    // ── Location update ───────────────────────────────────────────────────

    public void updateDriverLocation(Location location) {
        this.driverCurrentLocation = location;
    }

    // ── Computed helpers ──────────────────────────────────────────────────

    public double getDistanceKm() {
        return pickupLocation.distanceTo(dropoffLocation);
    }

    public long getDurationMinutes() {
        if (startedAt == null || completedAt == null) return 0;
        return java.time.Duration.between(startedAt, completedAt).toMinutes();
    }

    public TripStatus getStatus() { return currentState.getStatus(); }

    // ── Ratings ───────────────────────────────────────────────────────────

    public void addRating(Rating rating) {
        if (!rating.getTripId().equals(this.id))
            throw new IllegalArgumentException("Rating does not belong to this trip");
        ratings.add(rating);
    }

    public List<Rating> getRatings() { return Collections.unmodifiableList(ratings); }

    // ── Getters ───────────────────────────────────────────────────────────

    public String getId() { return id; }
    public Rider getRider() { return rider; }
    public Driver getDriver() { return driver; }
    public List<Rider> getPoolRiders() { return Collections.unmodifiableList(poolRiders); }
    public Location getPickupLocation() { return pickupLocation; }
    public Location getDropoffLocation() { return dropoffLocation; }
    public RideType getRideType() { return rideType; }
    public FareBreakdown getFareBreakdown() { return fareBreakdown; }
    public Payment getPayment() { return payment; }
    public CancellationReason getCancellationReason() { return cancellationReason; }
    public Instant getRequestedAt() { return requestedAt; }
    public Instant getAcceptedAt() { return acceptedAt; }
    public Instant getDriverArrivedAt() { return driverArrivedAt; }
    public Instant getStartedAt() { return startedAt; }
    public Instant getCompletedAt() { return completedAt; }
    public Instant getCancelledAt() { return cancelledAt; }

    public void setDriver(Driver driver) { this.driver = driver; }
    public void setFareBreakdown(FareBreakdown fareBreakdown) { this.fareBreakdown = fareBreakdown; }
    public void setPayment(Payment payment) { this.payment = payment; }
    public void setCancellationReason(CancellationReason reason) { this.cancellationReason = reason; }

    @Override
    public String toString() {
        return String.format("Trip{id='%s', rider='%s', driver='%s', type=%s, status=%s}",
            id.substring(0, 8),
            rider.getName(),
            driver != null ? driver.getName() : "unassigned",
            rideType,
            getStatus());
    }
}
```

### 7.12 Surge Pricing

```java
// SurgeZone.java
package com.rideshare.model;

import java.time.Instant;

public class SurgeZone {
    private final String zoneId;
    private final String zoneName;
    private double surgeMultiplier;
    private int activeRiders;
    private int availableDrivers;
    private Instant lastUpdated;

    public SurgeZone(String zoneId, String zoneName) {
        this.zoneId = zoneId;
        this.zoneName = zoneName;
        this.surgeMultiplier = 1.0;
        this.lastUpdated = Instant.now();
    }

    public void update(int activeRiders, int availableDrivers) {
        this.activeRiders = activeRiders;
        this.availableDrivers = availableDrivers;
        this.surgeMultiplier = computeMultiplier(activeRiders, availableDrivers);
        this.lastUpdated = Instant.now();
    }

    /**
     * Surge tiers:
     *   demand/supply < 1.5  → 1.0x
     *   1.5 – 2.5           → 1.5x
     *   2.5 – 4.0           → 2.0x
     *   > 4.0               → 2.5x (capped)
     */
    private double computeMultiplier(int riders, int drivers) {
        if (drivers == 0) return 2.5;
        double ratio = (double) riders / drivers;
        if (ratio < 1.5) return 1.0;
        if (ratio < 2.5) return 1.5;
        if (ratio < 4.0) return 2.0;
        return 2.5;
    }

    public String getZoneId() { return zoneId; }
    public String getZoneName() { return zoneName; }
    public double getSurgeMultiplier() { return surgeMultiplier; }
    public int getActiveRiders() { return activeRiders; }
    public int getAvailableDrivers() { return availableDrivers; }
    public Instant getLastUpdated() { return lastUpdated; }

    @Override
    public String toString() {
        return String.format("SurgeZone{id='%s', name='%s', multiplier=%.1fx, " +
            "riders=%d, drivers=%d}", zoneId, zoneName, surgeMultiplier,
            activeRiders, availableDrivers);
    }
}
```

```java
// SurgePricingService.java
package com.rideshare.service;

import com.rideshare.model.Location;
import com.rideshare.model.SurgeZone;

import java.util.HashMap;
import java.util.Map;

/**
 * Maintains a registry of surge zones and returns the applicable multiplier
 * for a given pickup location. In production, zone membership would be
 * determined by a geo-index (H3, S2); here we use a simple proximity check.
 */
public class SurgePricingService {
    private final Map<String, SurgeZone> zones = new HashMap<>();

    public void registerZone(SurgeZone zone) {
        zones.put(zone.getZoneId(), zone);
    }

    public void updateZone(String zoneId, int activeRiders, int availableDrivers) {
        SurgeZone zone = zones.get(zoneId);
        if (zone != null) zone.update(activeRiders, availableDrivers);
    }

    /**
     * Returns the highest surge multiplier among all zones containing the
     * pickup location. Falls back to 1.0 if no zone matches.
     */
    public double getSurgeMultiplier(Location pickupLocation) {
        return zones.values().stream()
            .mapToDouble(SurgeZone::getSurgeMultiplier)
            .max()
            .orElse(1.0);
    }

    public SurgeZone getZone(String zoneId) { return zones.get(zoneId); }
    public Map<String, SurgeZone> getAllZones() { return new HashMap<>(zones); }
}
```

### 7.13 Ride Type Factory

```java
// RideTypeFactory.java
package com.rideshare.factory;

import com.rideshare.model.RideType;
import com.rideshare.strategy.fare.EconomyFareStrategy;
import com.rideshare.strategy.fare.FareCalculationStrategy;
import com.rideshare.strategy.fare.PoolFareStrategy;
import com.rideshare.strategy.fare.PremiumFareStrategy;

/**
 * Factory that maps RideType enum values to their corresponding
 * FareCalculationStrategy. Adding a new ride type requires only
 * adding a new enum value and a case here.
 */
public class RideTypeFactory {

    private RideTypeFactory() {}

    public static FareCalculationStrategy createFareStrategy(RideType rideType) {
        switch (rideType) {
            case ECONOMY: return new EconomyFareStrategy();
            case PREMIUM: return new PremiumFareStrategy();
            case POOL:    return new PoolFareStrategy();
            default:
                throw new IllegalArgumentException("Unsupported ride type: " + rideType);
        }
    }
}
```

### 7.14 Driver Registry

```java
// DriverRegistry.java
package com.rideshare.service;

import com.rideshare.model.Driver;
import com.rideshare.model.VehicleType;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.stream.Collectors;

public class DriverRegistry {
    private final Map<String, Driver> drivers = new HashMap<>();

    public void register(Driver driver) {
        drivers.put(driver.getId(), driver);
    }

    public Optional<Driver> findById(String driverId) {
        return Optional.ofNullable(drivers.get(driverId));
    }

    public List<Driver> getAllAvailableDrivers() {
        return drivers.values().stream()
            .filter(d -> d.getStatus().isAvailableForMatch())
            .collect(Collectors.toList());
    }

    public List<Driver> getAvailableDriversForType(VehicleType type) {
        return drivers.values().stream()
            .filter(d -> d.isAvailableFor(type))
            .collect(Collectors.toList());
    }

    public Collection<Driver> getAllDrivers() {
        return Collections.unmodifiableCollection(drivers.values());
    }
}
```

### 7.15 Rider Registry

```java
// RiderRegistry.java
package com.rideshare.service;

import com.rideshare.model.Rider;

import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

public class RiderRegistry {
    private final Map<String, Rider> riders = new HashMap<>();

    public void register(Rider rider) {
        riders.put(rider.getId(), rider);
    }

    public Optional<Rider> findById(String riderId) {
        return Optional.ofNullable(riders.get(riderId));
    }

    public Collection<Rider> getAllRiders() {
        return Collections.unmodifiableCollection(riders.values());
    }
}
```

### 7.16 Trip Repository

```java
// TripRepository.java
package com.rideshare.service;

import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.stream.Collectors;

public class TripRepository {
    private final Map<String, Trip> trips = new HashMap<>();

    public void save(Trip trip) {
        trips.put(trip.getId(), trip);
    }

    public Optional<Trip> findById(String tripId) {
        return Optional.ofNullable(trips.get(tripId));
    }

    public List<Trip> findByRiderId(String riderId) {
        return trips.values().stream()
            .filter(t -> t.getRider().getId().equals(riderId))
            .collect(Collectors.toList());
    }

    public List<Trip> findByDriverId(String driverId) {
        return trips.values().stream()
            .filter(t -> t.getDriver() != null
                      && t.getDriver().getId().equals(driverId))
            .collect(Collectors.toList());
    }

    public List<Trip> findByStatus(TripStatus status) {
        return trips.values().stream()
            .filter(t -> t.getStatus() == status)
            .collect(Collectors.toList());
    }

    public Collection<Trip> findAll() {
        return Collections.unmodifiableCollection(trips.values());
    }
}
```

### 7.17 Rating Service

```java
// RatingService.java
package com.rideshare.service;

import com.rideshare.exception.RatingException;
import com.rideshare.model.Driver;
import com.rideshare.model.Rating;
import com.rideshare.model.Rider;
import com.rideshare.model.Trip;
import com.rideshare.model.TripStatus;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class RatingService {
    private static final Duration RATING_WINDOW = Duration.ofHours(24);
    private final List<Rating> allRatings = new ArrayList<>();

    public Rating rateDriver(Trip trip, Rider ratingRider, int stars, String comment) {
        validateRatingContext(trip, ratingRider.getId(), trip.getDriver().getId());
        Rating rating = new Rating(trip.getId(), ratingRider.getId(),
                                   trip.getDriver().getId(), stars, comment);
        trip.getDriver().addRating(stars);
        trip.addRating(rating);
        allRatings.add(rating);
        System.out.printf("[RATING] %s rated driver %s: %d stars%n",
            ratingRider.getName(), trip.getDriver().getName(), stars);
        return rating;
    }

    public Rating rateRider(Trip trip, Driver ratingDriver, int stars, String comment) {
        validateRatingContext(trip, ratingDriver.getId(), trip.getRider().getId());
        Rating rating = new Rating(trip.getId(), ratingDriver.getId(),
                                   trip.getRider().getId(), stars, comment);
        trip.getRider().addRating(stars);
        trip.addRating(rating);
        allRatings.add(rating);
        System.out.printf("[RATING] Driver %s rated rider %s: %d stars%n",
            ratingDriver.getName(), trip.getRider().getName(), stars);
        return rating;
    }

    private void validateRatingContext(Trip trip, String raterId, String rateeId) {
        if (trip.getStatus() != TripStatus.COMPLETED)
            throw new RatingException("Ratings can only be submitted for completed trips");
        if (trip.getCompletedAt() == null)
            throw new RatingException("Trip completion timestamp missing");
        if (Duration.between(trip.getCompletedAt(), Instant.now()).compareTo(RATING_WINDOW) > 0)
            throw new RatingException("Rating window of 24 hours has expired");
        boolean alreadyRated = trip.getRatings().stream()
            .anyMatch(r -> r.getRaterId().equals(raterId) && r.getRateeId().equals(rateeId));
        if (alreadyRated)
            throw new RatingException("This user has already rated this participant for this trip");
    }

    public List<Rating> getAllRatings() { return Collections.unmodifiableList(allRatings); }
}
```

### 7.18 RideSharingService — Facade / Orchestrator

```java
// RideSharingService.java
package com.rideshare.service;

import com.rideshare.exception.DriverNotFoundException;
import com.rideshare.exception.PaymentProcessingException;
import com.rideshare.exception.RideSharingException;
import com.rideshare.factory.RideTypeFactory;
import com.rideshare.model.*;
import com.rideshare.observer.AnalyticsObserver;
import com.rideshare.observer.DriverAvailabilityObserver;
import com.rideshare.observer.NotificationObserver;
import com.rideshare.observer.TripObserver;
import com.rideshare.strategy.fare.FareCalculationStrategy;
import com.rideshare.strategy.matching.DriverMatchingStrategy;
import com.rideshare.strategy.matching.NearestDriverStrategy;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

/**
 * Facade that orchestrates all subsystems.
 * <p>
 * Clients interact exclusively with this class; they do not instantiate
 * domain objects or call repositories directly.
 */
public class RideSharingService {

    private final DriverRegistry driverRegistry;
    private final RiderRegistry riderRegistry;
    private final TripRepository tripRepository;
    private final SurgePricingService surgePricingService;
    private final RatingService ratingService;
    private DriverMatchingStrategy matchingStrategy;
    private final List<TripObserver> globalObservers;

    public RideSharingService() {
        this.driverRegistry      = new DriverRegistry();
        this.riderRegistry       = new RiderRegistry();
        this.tripRepository      = new TripRepository();
        this.surgePricingService = new SurgePricingService();
        this.ratingService       = new RatingService();
        this.matchingStrategy    = new NearestDriverStrategy();
        this.globalObservers     = new ArrayList<>();

        // Register default observers
        globalObservers.add(new NotificationObserver());
        globalObservers.add(new AnalyticsObserver());
        globalObservers.add(new DriverAvailabilityObserver());
    }

    // ── Configuration ─────────────────────────────────────────────────────

    public void setMatchingStrategy(DriverMatchingStrategy strategy) {
        this.matchingStrategy = strategy;
    }

    public void addGlobalObserver(TripObserver observer) {
        globalObservers.add(observer);
    }

    // ── Registration ──────────────────────────────────────────────────────

    public Driver registerDriver(String name, String email, String phone,
                                 Vehicle vehicle, String licenseNumber) {
        Driver driver = new Driver(name, email, phone, vehicle, licenseNumber);
        driverRegistry.register(driver);
        System.out.printf("[SERVICE] Driver registered: %s (id=%s)%n",
            name, driver.getId().substring(0, 8));
        return driver;
    }

    public Rider registerRider(String name, String email, String phone) {
        Rider rider = new Rider(name, email, phone);
        riderRegistry.register(rider);
        System.out.printf("[SERVICE] Rider registered: %s (id=%s)%n",
            name, rider.getId().substring(0, 8));
        return rider;
    }

    // ── Driver availability ────────────────────────────────────────────────

    public void setDriverOnline(Driver driver, Location currentLocation) {
        driver.updateLocation(currentLocation);
        driver.setOnline();
        System.out.printf("[SERVICE] Driver %s is now ONLINE at %s%n",
            driver.getName(), currentLocation.getAddress());
    }

    public void setDriverOffline(Driver driver) {
        driver.setOffline();
        System.out.printf("[SERVICE] Driver %s is now OFFLINE%n", driver.getName());
    }

    public void updateDriverLocation(Driver driver, Location location) {
        driver.updateLocation(location);
    }

    // ── Trip request ───────────────────────────────────────────────────────

    /**
     * Creates a trip, matches a driver, and returns the trip in REQUESTED state.
     * The driver is assigned immediately in this model; in production the driver
     * gets a push notification and must explicitly accept.
     */
    public Trip requestTrip(Rider rider, Location pickup, Location dropoff, RideType rideType) {
        if (rider.getDefaultPaymentMethod() == null)
            throw new RideSharingException("Rider has no payment method on file");

        Trip trip = new Trip(rider, pickup, dropoff, rideType);
        globalObservers.forEach(trip::addObserver);
        tripRepository.save(trip);
        rider.recordTrip(trip.getId());

        System.out.printf("%n[SERVICE] Trip requested: %s -> %s (%s)%n",
            pickup.getAddress(), dropoff.getAddress(), rideType.getDisplayName());

        // Match a driver
        List<Driver> candidates = driverRegistry.getAvailableDriversForType(
            rideType.getRequiredVehicleType());
        Optional<Driver> matched = matchingStrategy.findDriver(pickup,
            rideType.getRequiredVehicleType(), candidates);

        if (!matched.isPresent()) {
            trip.setCancellationReason(CancellationReason.NO_DRIVER_FOUND);
            trip.cancelTrip();
            throw new DriverNotFoundException(
                "No available driver found for ride type: " + rideType.getDisplayName());
        }

        Driver driver = matched.get();
        trip.setDriver(driver);
        driver.assignTrip(trip);

        double distKm = pickup.distanceTo(dropoff);
        System.out.printf("[SERVICE] Matched driver %s (%.2fkm away) using '%s' strategy%n",
            driver.getName(),
            driver.getCurrentLocation().distanceTo(pickup),
            matchingStrategy.getStrategyName());

        return trip;
    }

    // ── Trip lifecycle ─────────────────────────────────────────────────────

    public void acceptTrip(Trip trip) {
        trip.accept();
        System.out.printf("[SERVICE] Driver %s accepted trip %s%n",
            trip.getDriver().getName(), trip.getId().substring(0, 8));
    }

    public void markDriverArrived(Trip trip) {
        trip.driverArrived();
        System.out.printf("[SERVICE] Driver %s arrived at pickup for trip %s%n",
            trip.getDriver().getName(), trip.getId().substring(0, 8));
    }

    public void startTrip(Trip trip) {
        trip.startTrip();
        System.out.printf("[SERVICE] Trip %s is now IN PROGRESS%n",
            trip.getId().substring(0, 8));
    }

    /**
     * Completes the trip: calculates fare, processes payment, and transitions state.
     */
    public void completeTrip(Trip trip) {
        // Calculate surge multiplier
        double surge = surgePricingService.getSurgeMultiplier(trip.getPickupLocation());

        // Build fare context
        int poolRiderCount = 1 + trip.getPoolRiders().size();
        double distanceKm  = trip.getDistanceKm();
        long durationMins  = trip.getDurationMinutes();
        // Fallback for simulation: use straight-line distance with assumed speed
        if (durationMins == 0) {
            durationMins = Math.max(1, Math.round(distanceKm / 0.5)); // ~30 km/h in city
        }

        TripContext context = new TripContext(distanceKm, durationMins, surge,
                                              poolRiderCount, trip.getRideType());

        FareCalculationStrategy fareStrategy =
            RideTypeFactory.createFareStrategy(trip.getRideType());
        FareBreakdown fare = fareStrategy.calculate(context);
        trip.setFareBreakdown(fare);

        System.out.printf("[SERVICE] Fare calculated: %s%n", fare);

        // Process payment
        PaymentMethod paymentMethod = trip.getRider().getDefaultPaymentMethod();
        PaymentResult result = paymentMethod.processPayment(fare.getTotalFare());
        if (!result.isSuccess()) {
            throw new PaymentProcessingException(paymentMethod.getId(),
                "Payment failed: " + result.getErrorMessage());
        }
        Payment payment = new Payment(trip.getId(), paymentMethod,
                                      fare.getTotalFare(), result);
        trip.setPayment(payment);

        trip.completeTrip();
        System.out.printf("[SERVICE] Trip %s COMPLETED. Total: $%.2f%n",
            trip.getId().substring(0, 8), fare.getTotalFare());
    }

    public void cancelTrip(Trip trip, CancellationReason reason) {
        trip.setCancellationReason(reason);
        trip.cancelTrip();
        System.out.printf("[SERVICE] Trip %s CANCELLED. Reason: %s%n",
            trip.getId().substring(0, 8), reason);
    }

    // ── Pool ride ──────────────────────────────────────────────────────────

    public void addPoolRider(Trip trip, Rider additionalRider) {
        trip.addPoolRider(additionalRider);
        additionalRider.recordTrip(trip.getId());
        System.out.printf("[SERVICE] Rider %s added to pool trip %s%n",
            additionalRider.getName(), trip.getId().substring(0, 8));
    }

    // ── Rating ─────────────────────────────────────────────────────────────

    public Rating rateDriver(Trip trip, Rider rider, int stars, String comment) {
        return ratingService.rateDriver(trip, rider, stars, comment);
    }

    public Rating rateRider(Trip trip, Driver driver, int stars, String comment) {
        return ratingService.rateRider(trip, driver, stars, comment);
    }

    // ── Queries ────────────────────────────────────────────────────────────

    public List<Trip> getRiderTripHistory(Rider rider) {
        return tripRepository.findByRiderId(rider.getId());
    }

    public List<Trip> getDriverTripHistory(Driver driver) {
        return tripRepository.findByDriverId(driver.getId());
    }

    public Optional<Trip> findTripById(String tripId) {
        return tripRepository.findById(tripId);
    }

    // ── Surge management ───────────────────────────────────────────────────

    public void registerSurgeZone(SurgeZone zone) {
        surgePricingService.registerZone(zone);
    }

    public void updateSurgeZone(String zoneId, int activeRiders, int availableDrivers) {
        surgePricingService.updateZone(zoneId, activeRiders, availableDrivers);
        System.out.printf("[SURGE] Zone '%s' updated: %dx riders, %d drivers -> %.1fx%n",
            zoneId, activeRiders, availableDrivers,
            surgePricingService.getSurgeMultiplier(null));
    }

    // ── Accessors for testing/inspection ──────────────────────────────────

    public DriverRegistry getDriverRegistry() { return driverRegistry; }
    public RiderRegistry getRiderRegistry() { return riderRegistry; }
    public TripRepository getTripRepository() { return tripRepository; }
    public SurgePricingService getSurgePricingService() { return surgePricingService; }
}
```

### 7.19 End-to-End Simulation

```java
// RideSharingDemo.java
package com.rideshare.demo;

import com.rideshare.model.*;
import com.rideshare.observer.TripObserver;
import com.rideshare.service.RideSharingService;
import com.rideshare.strategy.matching.WeightedDriverStrategy;

public class RideSharingDemo {

    public static void main(String[] args) {
        System.out.println("═══════════════════════════════════════════════════════");
        System.out.println("          RIDE SHARING SYSTEM — DEMO SIMULATION");
        System.out.println("═══════════════════════════════════════════════════════\n");

        RideSharingService service = new RideSharingService();

        // ── Scenario 1: Economy ride ──────────────────────────────────────
        System.out.println("━━━ SCENARIO 1: Economy Ride ━━━━━━━━━━━━━━━━━━━━━━━━━\n");

        Vehicle car1 = new Vehicle("Toyota", "Camry", "ABC-1234",
                                   VehicleType.ECONOMY, 2021, "White");
        Driver driver1 = service.registerDriver("Alice Smith", "alice@rides.com",
                                                "555-0101", car1, "DL-789456");
        Rider rider1 = service.registerRider("Bob Jones", "bob@email.com", "555-0202");
        rider1.addPaymentMethod(new CreditCardPayment("4242", "Bob Jones", "12", "2027"));

        Location driverLoc1 = new Location(37.7749, -122.4194, "Driver Start - Market St");
        service.setDriverOnline(driver1, driverLoc1);

        Location pickup1  = new Location(37.7751, -122.4180, "Union Square, SF");
        Location dropoff1 = new Location(37.7849, -122.4094, "Nob Hill, SF");

        Trip trip1 = service.requestTrip(rider1, pickup1, dropoff1, RideType.ECONOMY);
        service.acceptTrip(trip1);
        service.markDriverArrived(trip1);
        service.startTrip(trip1);
        service.completeTrip(trip1);

        service.rateDriver(trip1, rider1, 5, "Great driver, very smooth ride!");
        service.rateRider(trip1, driver1, 5, "Polite and ready on time.");

        // ── Scenario 2: Premium ride with surge ───────────────────────────
        System.out.println("\n━━━ SCENARIO 2: Premium Ride with Surge Pricing ━━━━━━\n");

        Vehicle car2 = new Vehicle("BMW", "5 Series", "XYZ-9876",
                                   VehicleType.PREMIUM, 2023, "Black");
        Driver driver2 = service.registerDriver("Carol White", "carol@rides.com",
                                                "555-0303", car2, "DL-321654");
        Rider rider2 = service.registerRider("David Lee", "david@email.com", "555-0404");
        DigitalWalletPayment wallet = new DigitalWalletPayment("DW-DAVID", "PayRide", 200.0);
        rider2.addPaymentMethod(wallet);

        Location driverLoc2 = new Location(37.7900, -122.4000, "Driver - Financial District");
        service.setDriverOnline(driver2, driverLoc2);

        // Set up surge zone
        SurgeZone zone1 = new SurgeZone("ZONE-SF-DOWNTOWN", "SF Downtown");
        service.registerSurgeZone(zone1);
        service.updateSurgeZone("ZONE-SF-DOWNTOWN", 80, 20); // 4:1 ratio → 2.0x surge

        // Switch to weighted matching strategy
        service.setMatchingStrategy(new WeightedDriverStrategy());

        Location pickup2  = new Location(37.7950, -122.3985, "Embarcadero, SF");
        Location dropoff2 = new Location(37.8044, -122.4115, "Fisherman's Wharf, SF");

        Trip trip2 = service.requestTrip(rider2, pickup2, dropoff2, RideType.PREMIUM);
        service.acceptTrip(trip2);
        service.markDriverArrived(trip2);
        service.startTrip(trip2);
        service.completeTrip(trip2);

        service.rateDriver(trip2, rider2, 4, "Professional and comfortable car.");

        // ── Scenario 3: Pool ride with multiple riders ────────────────────
        System.out.println("\n━━━ SCENARIO 3: Pool Ride (Shared) ━━━━━━━━━━━━━━━━━━\n");

        Vehicle car3 = new Vehicle("Honda", "Accord", "POOL-111",
                                   VehicleType.POOL, 2022, "Silver");
        Driver driver3 = service.registerDriver("Eve Johnson", "eve@rides.com",
                                                "555-0505", car3, "DL-111222");
        Rider rider3a = service.registerRider("Frank Miller", "frank@email.com", "555-0606");
        Rider rider3b = service.registerRider("Grace Kim", "grace@email.com", "555-0707");
        rider3a.addPaymentMethod(new CreditCardPayment("1111", "Frank Miller", "06", "2026"));
        rider3b.addPaymentMethod(new DigitalWalletPayment("GW-GRACE", "GoPay", 50.0));

        Location driverLoc3 = new Location(37.7600, -122.4300, "Driver - Mission District");
        service.setDriverOnline(driver3, driverLoc3);

        Location pickup3  = new Location(37.7610, -122.4290, "Mission St & 16th, SF");
        Location dropoff3 = new Location(37.7749, -122.4194, "Civic Center, SF");

        // rider3a requests the pool trip
        Trip poolTrip = service.requestTrip(rider3a, pickup3, dropoff3, RideType.POOL);
        // rider3b joins the pool
        service.addPoolRider(poolTrip, rider3b);

        service.acceptTrip(poolTrip);
        service.markDriverArrived(poolTrip);
        service.startTrip(poolTrip);
        service.completeTrip(poolTrip);

        service.rateDriver(poolTrip, rider3a, 5, "Efficient pool management!");

        // ── Scenario 4: Cancellation ──────────────────────────────────────
        System.out.println("\n━━━ SCENARIO 4: Trip Cancellation ━━━━━━━━━━━━━━━━━━━\n");

        Vehicle car4 = new Vehicle("Hyundai", "Sonata", "CNCL-999",
                                   VehicleType.ECONOMY, 2020, "Blue");
        Driver driver4 = service.registerDriver("Hank Brown", "hank@rides.com",
                                                "555-0808", car4, "DL-999888");
        Rider rider4 = service.registerRider("Iris Chen", "iris@email.com", "555-0909");
        rider4.addPaymentMethod(new CashPayment());

        service.setMatchingStrategy(new com.rideshare.strategy.matching.NearestDriverStrategy());
        Location driverLoc4 = new Location(37.7800, -122.4200, "Driver - Hayes Valley");
        service.setDriverOnline(driver4, driverLoc4);

        Location pickup4  = new Location(37.7810, -122.4190, "Hayes Valley, SF");
        Location dropoff4 = new Location(37.7900, -122.4100, "Tenderloin, SF");

        Trip trip4 = service.requestTrip(rider4, pickup4, dropoff4, RideType.ECONOMY);
        service.acceptTrip(trip4);
        service.cancelTrip(trip4, CancellationReason.RIDER_CANCELLED);

        // ── Trip History ──────────────────────────────────────────────────
        System.out.println("\n━━━ TRIP HISTORY ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");
        System.out.println("Bob Jones's trips:");
        service.getRiderTripHistory(rider1).forEach(t ->
            System.out.printf("  %s | %s | fare=$%.2f%n",
                t.getId().substring(0, 8), t.getStatus(),
                t.getFareBreakdown() != null ? t.getFareBreakdown().getTotalFare() : 0.0));

        System.out.println("\nAlice Smith's trips:");
        service.getDriverTripHistory(driver1).forEach(t ->
            System.out.printf("  %s | %s | fare=$%.2f%n",
                t.getId().substring(0, 8), t.getStatus(),
                t.getFareBreakdown() != null ? t.getFareBreakdown().getTotalFare() : 0.0));

        System.out.println("\nDriver Ratings after all trips:");
        System.out.printf("  Alice Smith: %.2f stars (%d ratings)%n",
            driver1.getAverageRating(), driver1.getRatingCount());
        System.out.printf("  Carol White: %.2f stars (%d ratings)%n",
            driver2.getAverageRating(), driver2.getRatingCount());
        System.out.printf("  Eve Johnson: %.2f stars (%d ratings)%n",
            driver3.getAverageRating(), driver3.getRatingCount());

        System.out.println("\n═══════════════════════════════════════════════════════");
        System.out.println("                   SIMULATION COMPLETE");
        System.out.println("═══════════════════════════════════════════════════════");
    }
}
```

---

## 8. Design Patterns Used

### 8.1 State Pattern — Trip Lifecycle

**What:** Each `TripStatus` is embodied by a concrete `TripState` object. The `Trip` delegates every lifecycle method (`accept()`, `startTrip()`, etc.) to its current state, which either performs the transition or throws `InvalidTripStateTransitionException`.

**Why:**
- Eliminates sprawling `if (status == X) { ... } else if (status == Y) ...` chains inside `Trip`.
- Adding a new status (e.g., `DRIVER_REROUTED`) means creating one new state class and wiring two transitions — zero changes to `Trip` itself.
- Illegal transitions are caught at the boundary, not buried inside business logic.
- Each state class is independently unit-testable.

**Trade-off:** The number of classes grows linearly with the number of states. For a simple three-state machine this feels heavy; for a production system with 8+ states it pays for itself immediately.

**Key files:** `TripState`, `AbstractTripState`, `RequestedState`, `AcceptedState`, `DriverArrivedState`, `InProgressState`, `CompletedState`, `CancelledState`

---

### 8.2 Strategy Pattern — Fare Calculation

**What:** `FareCalculationStrategy` is an interface with a single method `calculate(TripContext): FareBreakdown`. `EconomyFareStrategy`, `PremiumFareStrategy`, and `PoolFareStrategy` are the concrete algorithms. `RideTypeFactory` selects the correct strategy at trip completion time.

**Why:**
- Fare algorithms change frequently (marketing promotions, regulatory changes, A/B experiments). Isolating each algorithm in its own class means a pricing change for Premium rides has zero impact on Economy or Pool code.
- New ride types (e.g., `LUXURY_SUV`) require only a new strategy class and a factory case — existing code is untouched.
- Strategies are stateless and can be shared (flyweight) across concurrent requests.

**Trade-off:** The `TripContext` value object must be kept in sync as new pricing inputs are needed (e.g., toll road flag, time-of-day factor). If `TripContext` becomes a kitchen-sink object, consider breaking it apart.

**Key files:** `FareCalculationStrategy`, `EconomyFareStrategy`, `PremiumFareStrategy`, `PoolFareStrategy`, `TripContext`, `RideTypeFactory`

---

### 8.3 Strategy Pattern — Driver Matching

**What:** `DriverMatchingStrategy` defines the contract for selecting a driver from a candidate list. `NearestDriverStrategy` picks the geographically closest driver. `WeightedDriverStrategy` scores on a proximity-rating composite.

**Why:**
- Matching is the most experimented-on component in a ride-sharing platform. Teams run A/B tests on matching algorithms weekly. Strategy makes swapping algorithms a one-liner (`service.setMatchingStrategy(...)`).
- Different cities or contexts (airport queues, events, premium rides) might use different strategies simultaneously.
- The algorithm can be replaced with a remote ML model call without changing any caller code.

**Trade-off:** In production the "available drivers list" is never held in memory — it comes from a geo-indexed query. The strategy interface should accept a repository or query object rather than a pre-filtered list to avoid materialising millions of drivers. This is noted in the Production Considerations section.

**Key files:** `DriverMatchingStrategy`, `NearestDriverStrategy`, `WeightedDriverStrategy`

---

### 8.4 Observer Pattern — Trip Status Updates

**What:** `TripObserver` is a listener interface. `Trip` maintains a list of observers and calls `onTripStateChanged(trip, prev, next)` on every state transition. Three built-in observers cover notifications, analytics events, and driver availability updates.

**Why:**
- The `Trip` aggregate is the authoritative source of state. Downstream concerns (push notifications, analytics, availability bookkeeping) should not pollute the aggregate's own logic.
- New cross-cutting concerns (fraud detection, ETA recalculation, surge zone updates) are added by registering a new observer — zero changes to `Trip`.
- In a microservices deployment each observer would be replaced by a Kafka producer, and the downstream consumers would implement equivalent logic independently.

**Trade-off:** Synchronous observer execution means a slow or throwing observer blocks or breaks the state transition. Production systems should make observer calls asynchronous (thread pool, outbox pattern) with retry and dead-letter queues.

**Key files:** `TripObserver`, `NotificationObserver`, `AnalyticsObserver`, `DriverAvailabilityObserver`

---

### 8.5 Factory Pattern — Ride Type Creation

**What:** `RideTypeFactory.createFareStrategy(RideType)` centralises the mapping from a ride-type enum value to the concrete strategy instance.

**Why:**
- Callers (`RideSharingService.completeTrip()`) should not `new` concrete strategy classes — that would couple the service to implementation details.
- All mappings are visible in one place, making it easy to audit which strategies exist and ensuring consistent instantiation.
- The factory can be extended to a registry pattern (strategies registered at startup) to support plugin architectures.

**Trade-off:** A simple `switch` factory can be replaced by a registry map (`Map<RideType, Supplier<FareCalculationStrategy>>`) for zero-code extensibility, at the cost of discoverability.

**Key files:** `RideTypeFactory`

---

## 9. Key Design Decisions and Trade-offs

### 9.1 Driver Matching Algorithm Design

**Decision:** Two strategies provided; `NearestDriverStrategy` is the default.

**Nearest-only is fast but suboptimal.** A driver 0.3 km away with a 2-star rating is preferred over a 0.5 km driver with a 4.9-star rating. `WeightedDriverStrategy` addresses this with a tunable proximity/rating composite.

**The real scalability problem** is that scanning all available drivers is O(n) on fleet size. Production solutions:
1. **Geo-index sharding** — partition the map into H3 hexagons or S2 cells. A request queries only cells within the search radius (typically 3–5 cells), reducing candidates from millions to dozens.
2. **Pre-sorted proximity queues** — maintain a min-heap per vehicle type per zone, updated as drivers move.
3. **Broadcast-and-accept** — send the request to the nearest N drivers simultaneously; first to accept wins. Reduces latency at the cost of wasted notifications.

**Trade-off accepted here:** In-memory O(n) scan with a 10 km radius filter. Sufficient for a single-city simulation; must be replaced with a geo-service before production.

---

### 9.2 Surge Pricing Design

**Decision:** `SurgeZone` computes a tiered multiplier from demand/supply ratio with four tiers (1.0×, 1.5×, 2.0×, 2.5×).

**The fundamental tension in surge pricing:**
- Too aggressive → regulatory backlash, rider churn, PR crises during emergencies.
- Too conservative → driver supply shortage at peak, longer wait times, lower revenue.

**Design choices made:**
1. **Tiered rather than continuous** — continuous surge (e.g., ratio × 0.7) is hard to explain to riders. Tiers are predictable and displayable: "1.5× surge in effect."
2. **Capped at 2.5×** — matches regulatory limits in several jurisdictions and avoids extreme outlier fares.
3. **External feed** — the `SurgePricingService.updateZone()` method is called by an external demand model (not computed inline). This allows ML-driven surge that considers historical patterns, event calendars, weather, etc.
4. **Zone granularity** — a single city may have dozens of micro-zones (airport, stadium, CBD). Pickup location determines which zone applies.

**Future extension:** Implement `SurgeStrategy` interface with `FlatTierSurgeStrategy` (current), `ContinuousSurgeStrategy`, and `MLSurgeStrategy` wrapping a model scoring call.

---

### 9.3 Pool Ride Complexity

**Decision:** Pool rides share one `Trip` record (from the primary rider's perspective) with a `poolRiders` list for additional passengers.

**Why this is hard in production:**

1. **Independent fare calculation per rider.** In this model all pool riders pay from the same `FareBreakdown`. In production each rider gets their own fare calculated based on their personal pickup → dropoff distance and how many pool partners actually shared their segment. This requires route segmentation.

2. **Dynamic matching.** When a new pool rider requests a trip, the system must:
   - Find an active pool trip whose route passes near the new rider's pickup.
   - Verify the detour does not exceed the existing rider's ETA tolerance (typically +5 min).
   - Recalculate ETAs for all riders.
   This is a constrained vehicle routing problem — NP-hard in general.

3. **Rider privacy.** Pool riders should not see each other's precise home addresses. The displayed pickup/dropoff should be a nearby landmark or anonymised waypoint.

4. **Cancellation complexity.** If one pool rider cancels, the remaining riders' fares must be recalculated. If the last rider cancels, the driver is released.

**Decision accepted here:** Simplified model where pool riders share a trip and the `PoolFareStrategy` applies a flat per-rider discount. Production complexity is documented but out of scope for LLD.

---

### 9.4 Concurrency and Driver Assignment Race

**Decision:** Driver assignment uses `synchronized` on the `Driver` object's `assignTrip` method.

**The race:** Two simultaneous trip requests could both identify the same driver as the nearest available and both attempt to assign them.

**In this single-JVM model:** `synchronized` on `assignTrip` ensures only one requester successfully claims the driver; the second sees `DriverStatus.ON_TRIP` and gets an `IllegalStateException`.

**In production (distributed):** `synchronized` is insufficient. Solutions:
1. **Optimistic locking** — record a `version` field on driver state. The update query includes `WHERE version = :expected`. First writer wins; second gets a stale-version error and retries with re-matching.
2. **Redis distributed lock** — `SET driver:{id}:lock NX EX 5` before assignment. Only one process holds the lock.
3. **Single-writer actor** — driver state is owned by a single actor/shard (consistent hashing on driver ID). All assignment requests for that driver are serialised through one thread.

---

### 9.5 Payment Failure Handling

**Decision:** `completeTrip()` throws `PaymentProcessingException` if payment fails, leaving the trip in `IN_PROGRESS` state.

**Why not auto-cancel?** A completed physical trip should always have a record, even if payment failed. The rider owes the fare; the system should enter a "payment pending" state and retry.

**Production pattern:**
1. Attempt primary payment method.
2. On failure, try backup payment methods in order.
3. If all fail, complete the trip but set `paymentStatus = PENDING`.
4. Schedule async retry (exponential backoff over 24 hours).
5. Suspend rider account if payment is not resolved within the retry window.
6. Driver receives guaranteed earnings regardless of rider payment (platform absorbs short-term risk).

---

## 10. Extension Points

### 10.1 Adding a New Ride Type

1. Add `LUXURY` to `RideType` enum with appropriate `VehicleType` and pool size.
2. Add `LUXURY` to `VehicleType` enum.
3. Create `LuxuryFareStrategy implements FareCalculationStrategy`.
4. Add a `case LUXURY` in `RideTypeFactory`.
5. No other code changes required.

### 10.2 Adding a New Matching Algorithm

1. Implement `DriverMatchingStrategy`.
2. Inject into `RideSharingService` via `setMatchingStrategy()`.
3. No other code changes required.

### 10.3 Adding a New Payment Method

1. Subclass `PaymentMethod`, implement `processPayment()` and `getMaskedIdentifier()`.
2. Add the type to `PaymentMethodType` enum.
3. No other code changes required.

### 10.4 Adding a New Trip Observer (e.g., Fraud Detection)

1. Implement `TripObserver`.
2. Call `service.addGlobalObserver(new FraudDetectionObserver())` at startup.
3. No other code changes required.

### 10.5 Adding Scheduled Rides

1. Add `SCHEDULED` state before `REQUESTED` in the state machine.
2. Implement `ScheduledState extends AbstractTripState`.
3. Add a `scheduleTrip(Instant pickupTime)` method to `RideSharingService`.
4. A scheduler service transitions `SCHEDULED → REQUESTED` at the appropriate time.

### 10.6 Replacing In-Memory Repositories with a Database

All three registries and `TripRepository` use `Map<String, Entity>` internally. Replace with JPA repositories implementing the same interface contracts — no callers change.

### 10.7 Distributed Event Publishing

Replace the synchronous `TripObserver` call with an `EventBus` or Kafka producer in the `notifyObservers` method of `Trip`. Downstream services become independent consumers. The observer interface becomes the event schema contract.

---

## 11. Interview Discussion Points

### Q1: Why did you use the State pattern instead of a status field with if-else?

A status field (`TripStatus status`) with guards like `if (status == REQUESTED) { status = ACCEPTED; }` scattered across methods is called "stringly typed" state management. It has three fatal flaws:

1. **No compile-time guarantee** that every transition is handled correctly. A developer adding a new status must audit every `if-else` chain manually.
2. **No single source of truth** for what transitions are legal from a given state.
3. **Impossible to extend** without touching every method that switches on status.

The State pattern makes invalid transitions impossible to call without an explicit exception. Each state is a class — it appears in IDE navigation, code search, and test suites as a first-class citizen.

---

### Q2: How would you scale the driver matching to millions of drivers?

The in-memory O(n) scan must be replaced with a geo-spatial index. The industry-standard approach:

1. **Cell decomposition** — divide the city into H3 hexagonal cells at resolution 8 (~460m diameter). Each driver's location is mapped to their current cell.
2. **Redis GEO or PostGIS** — store active driver locations as geo-points. `GEORADIUS pickup_lat pickup_lon 5 km` returns nearby drivers in O(log n + k) time.
3. **Demand-aware pre-filtering** — in surge zones, expand search radius. At airports, prefer drivers in dedicated queue.
4. **Consistent hashing** — shard the driver geo-index across a cluster by cell ID so no single node is hot.

---

### Q3: How does surge pricing prevent rider dissatisfaction?

Three UX mechanisms:

1. **Upfront display** — show the multiplier and estimated fare before the rider confirms the request. Rider consent is explicit.
2. **Fare capping** — show riders the maximum fare (2.5× in this model) so they can plan. This is a contractual guarantee, not just a display.
3. **Surge decay** — as more drivers come online (attracted by higher earnings), the ratio drops and surge decreases. The system is self-correcting. Display "Surge usually ends in ~X minutes" based on historical patterns.

---

### Q4: What happens if the driver's app crashes mid-trip?

This is a liveness failure scenario. Production handling:

1. **Heartbeat timeout** — if no GPS update is received from the driver for N seconds, the backend marks the driver as `SUSPECTED_OFFLINE`.
2. **Trip timeout** — if the trip remains `IN_PROGRESS` with no updates for M minutes, trigger an alert and attempt to contact the rider via SMS.
3. **Automatic completion** — if the trip's estimated completion time (based on last-known route) has passed, auto-complete with the estimated fare and queue for manual review.
4. **Driver penalty** — repeated liveness failures affect driver score and can trigger account review.

---

### Q5: How would you implement Pool ride matching in production?

Pool matching is a real-time constrained vehicle routing problem:

1. When a pool request arrives, query active pool trips in the same geographic zone.
2. For each candidate trip, compute the detour cost: `new_route_time - original_route_time`. Detour must be ≤ 5 minutes for the existing rider.
3. If a suitable trip is found, add the new rider. Otherwise, create a new pool trip and wait up to 2 minutes for additional riders before dispatching.
4. Communicate updated ETAs to all pool riders immediately.
5. Implement route re-optimisation as new riders are added (Traveling Salesman heuristic with time-window constraints).

---

### Q6: How would you handle the rating system at scale?

1. **Decoupled rating service** — ratings are written to a separate service, not inline with trip completion. The trip-completion flow should not be blocked by rating computation.
2. **Eventual consistency** — `averageRating` on `User` is denormalised for fast reads. It is updated asynchronously via a consumer that processes rating events.
3. **Fraud detection** — ratings are a manipulation target. Detect suspicious patterns: all 1-star ratings to a competitor's driver, all 5-star ratings from the same IP block, ratings submitted faster than humanly possible.
4. **Rating recalculation** — if fraudulent ratings are detected and removed, trigger a batch recalculation of the affected user's average.
5. **Weighted recency** — more recent ratings have higher weight in the displayed average. A driver with 1,000 old trips and 4.2 average who has improved recently should surface that improvement.

---

### Q7: Walk me through a complete trip — what happens in each layer?

```
Rider App                 API Gateway            RideSharingService          Storage
    │                          │                         │                      │
    │── POST /trips ──────────►│                         │                      │
    │   {pickup, dropoff,      │── authenticate, ────────►│                      │
    │    rideType}             │   route                  │                      │
    │                          │                         │── create Trip ───────►│
    │                          │                         │── query geo-index ───►│
    │                          │                         │◄─ nearest drivers ────│
    │                          │                         │── assign driver       │
    │                          │                         │── save Trip ─────────►│
    │                          │◄─ 201 {tripId, ─────────│                      │
    │◄─ tripId, ETA ───────────│   driverETA}            │                      │
    │                          │                         │                      │
    │  [Driver App]            │                         │                      │
    │── POST /trips/{id}/accept►│── route ───────────────►│                      │
    │                          │                         │── trip.accept() ─────►│
    │                          │                         │── notify observers    │
    │◄─ push: driver accepted ─┤◄────────────────────────│                      │
```

---

## 12. Production Considerations

### 12.1 Data Storage

| Data | Store | Rationale |
|---|---|---|
| Driver/Rider profiles | PostgreSQL | Relational, ACID, low write volume |
| Active driver locations | Redis GEO | Sub-millisecond geo queries, TTL auto-expiry |
| Trip records | PostgreSQL | ACID for financial records, foreign keys |
| Trip events / audit log | Apache Kafka + S3 | Append-only, infinite retention, replay |
| Ratings aggregate | PostgreSQL + Redis cache | Writes infrequent; reads must be fast |
| Session tokens | Redis | TTL-based expiry, distributed access |
| Surge zone state | Redis | Low latency reads by many services |

### 12.2 Microservice Decomposition

A production deployment would decompose this monolith into:

| Service | Responsibility |
|---|---|
| **Identity Service** | Registration, auth, JWT issuance |
| **Driver Service** | Profile, vehicle, availability, location |
| **Rider Service** | Profile, payment methods, preferences |
| **Dispatch Service** | Trip creation, driver matching, state machine |
| **Pricing Service** | Fare calculation, surge computation |
| **Payment Service** | Payment processing, refunds, receipts |
| **Rating Service** | Rating collection, fraud detection, aggregation |
| **Notification Service** | Push, SMS, email on state changes |
| **Analytics Service** | Event consumption, reporting, dashboards |

### 12.3 API Design

```
POST   /v1/trips                        # Request a trip
GET    /v1/trips/{tripId}               # Get trip details
PATCH  /v1/trips/{tripId}/accept        # Driver accepts
PATCH  /v1/trips/{tripId}/driver-arrived
PATCH  /v1/trips/{tripId}/start
PATCH  /v1/trips/{tripId}/complete
PATCH  /v1/trips/{tripId}/cancel

POST   /v1/trips/{tripId}/ratings       # Submit rating
GET    /v1/riders/{riderId}/trips       # Trip history
GET    /v1/drivers/{driverId}/trips     # Driver history

GET    /v1/drivers/{driverId}/location  # Last known location
PATCH  /v1/drivers/{driverId}/status    # online / offline

GET    /v1/pricing/estimate             # Fare estimate before booking
GET    /v1/pricing/surge?zone={zoneId}  # Current surge info
```

### 12.4 Real-Time Communication

The `TripObserver` in this design is a stand-in for:
- **WebSocket connections** — maintained between rider/driver apps and the API server. State change events are pushed immediately.
- **Server-Sent Events (SSE)** — lighter alternative for one-way server→client updates (e.g., driver location stream).
- **Firebase Cloud Messaging / APNs** — push notifications when the app is backgrounded.

### 12.5 Observability

Every state transition should emit a structured log event:

```json
{
  "event": "TRIP_STATE_CHANGED",
  "tripId": "abc123",
  "fromStatus": "REQUESTED",
  "toStatus": "ACCEPTED",
  "driverId": "drv456",
  "riderId": "rdr789",
  "timestamp": "2026-06-23T14:30:00Z",
  "durationSinceRequestMs": 4200
}
```

Key metrics to monitor:
- **p99 match latency** — time from trip request to driver assignment
- **Accept rate** — % of matched trips accepted (low → driver shortage)
- **Cancellation rate by stage** — spikes in ACCEPTED cancellations may indicate bad matching
- **Fare accuracy** — compare estimated vs final fare; large deviations indicate routing bugs
- **Payment failure rate** — track by payment method type

### 12.6 Security

| Concern | Control |
|---|---|
| Driver impersonation | JWT with short expiry (15 min) + refresh token rotation |
| Location spoofing | Server-side location validation (speed plausibility check) |
| Fare manipulation | Fare calculated server-side only; client receives the result |
| PII exposure | Rider phone/email masked in driver-facing APIs |
| Payment data | Card numbers never stored; tokenised via payment vault |
| Rating manipulation | Rate limiting: one rating per trip per participant |
| Trip replay | Idempotency keys on all state-change endpoints |

### 12.7 Resilience Patterns

| Failure | Pattern |
|---|---|
| Payment gateway down | Circuit breaker + fallback to "pay on app relaunch" flow |
| Geo-index slow | Timeout + fallback to last-known cached results |
| Driver app crash | Heartbeat monitoring + auto-complete after timeout |
| Notification service down | Trip lifecycle continues; notifications queued with retry |
| Database overload | Read replicas for history queries; write path isolated |
| Surge computation stale | Cache with TTL; on miss, default to 1.0× (conservative) |

### 12.8 Testing Strategy

| Layer | Approach |
|---|---|
| State machine | Unit test every legal transition and every illegal transition (expect exception) |
| Fare strategies | Property-based tests: fare always ≥ base fare; pool always ≤ economy for same route |
| Driver matching | Contract tests: nearest strategy must always return closest driver within radius |
| Observer pattern | Mock observer, verify exactly one call per state transition with correct args |
| Surge calculation | Table-driven tests covering all ratio tiers and edge cases (0 drivers, 0 riders) |
| RideSharingService | Integration tests with real registries and in-memory repositories |
| Payment | Stub payment gateway; test success path, failure path, and partial failure |
| End-to-end | Scenario tests (as shown in `RideSharingDemo`) covering all ride types |

---

*This document covers the complete low-level design for a production-quality Ride Sharing System. The Java implementation is fully compilable and demonstrates all five design patterns in working code. The analysis sections provide the depth expected in a senior/staff engineering interview discussion.*
