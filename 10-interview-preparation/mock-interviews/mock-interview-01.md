# Mock Interview 01: Design a Ride Sharing System

---

## 1. Interview Setup

### Context

A ride-sharing system like Uber or Lyft allows riders to request on-demand transportation through a mobile application. A rider specifies a pickup and dropoff location, the system matches them to a nearby available driver, the driver travels to pick them up, the trip is completed, and payment is processed automatically. The system must handle the full lifecycle of a ride — from initial request through real-time driver matching, through the trip itself, through fare calculation and payment — while keeping the rider and driver states in sync and emitting events that downstream consumers (notifications, analytics, fraud detection) can react to.

This problem sits at the intersection of state machine design, object-oriented domain modeling, behavioral patterns (Strategy, Observer), and real-world extensibility concerns like surge pricing and scheduled rides. It is deliberately chosen because it has the right level of domain complexity to reveal whether a candidate thinks at the level of a system designer or merely a class builder.

### Time Allocation (45 minutes)

| Phase | Duration | Goal |
|---|---|---|
| Requirements clarification | 5–7 minutes | Surface constraints, identify actors, scope features in/out |
| Entity identification and relationships | 5–7 minutes | Name core entities, distinguish RideRequest from Ride, introduce enums |
| Class design and interface definition | 8–10 minutes | Interfaces, strategy pattern, observer pattern, state machine |
| Live coding of core components | 15–18 minutes | Core domain model, matchers, fare calculator, event publisher, service |
| Extension discussion and edge cases | 5–7 minutes | Scheduled rides, rating system, payment failure handling |

### What This Problem Specifically Tests

This problem is designed to probe six things that most other LLD problems do not surface as clearly:

1. **State machine reasoning**: The ride lifecycle has seven distinct states and a constrained set of valid transitions. A candidate who models only the happy path will miss critical states like `PAYMENT_FAILED` and will fail to encode valid-transition logic, leaving the system open to illegal state jumps. This tests whether the candidate treats state as a first-class design concern rather than an afterthought.

2. **Separation of booking lifecycle from trip lifecycle**: `RideRequest` represents a consumer's intent before a driver is assigned. `Ride` represents the active engagement after assignment. Conflating these into one class produces a bloated entity with nullable fields that are undefined in certain states — a classic anemic-domain-model smell. This problem specifically tests whether the candidate sees this distinction early.

3. **Real-time matching extensibility**: The matching algorithm is naturally a Strategy. The naive approach hardcodes "find nearest driver" in the service. A senior candidate introduces a `RideMatcher` interface immediately, enabling NearestDriverMatcher today and SurgeAwareMatcher or ML-based matchers tomorrow without touching the service.

4. **Fare calculation extensibility**: Similarly, fare logic is a Strategy. Hardcoding fare math in `completeRide` is the trap. A senior candidate defines a `FareStrategy` interface and composes `SurgeFareStrategy(delegate)` as a decorator without modifying the base strategy.

5. **Observer-based event handling**: Notification, analytics, and fraud detection all need to react to ride events. Calling `notificationService.notify(...)` directly inside `completeRide` couples the service to every consumer. A senior candidate introduces `RideEvent`, `RideEventListener`, and `RideEventPublisher` — keeping `RideService` decoupled from all downstream concerns.

6. **Concurrency awareness at the data layer**: Two riders could concurrently match to the same driver. This problem lurks beneath the surface. A senior candidate will note it and either propose optimistic locking at the persistence layer or a driver-reservation step. This distinguishes candidates with distributed systems awareness from those who only think in single-threaded terms.

### Difficulty Rating

**Level: Hard** (for a 45-minute format)

The core domain model alone — seven-state machine, two lifecycle objects, three behavioral patterns — requires careful upfront design. Unlike simpler LLD problems (parking lot, library management) where the entities are obvious, this problem requires non-obvious entity decomposition (`RideRequest` vs `Ride`, `SchedulingPolicy`) and pattern recognition under time pressure. The extensibility questions (scheduled rides, payment failure) are designed to probe whether the candidate's initial design was actually extensible or just appeared to be.

---

## 2. Interviewer Script

This section is written for use in a peer-review mock session. Person B reads these questions aloud to Person A. Use the exact wording provided — the specificity of each question is intentional.

---

### Stage 1: Requirements Clarification

Ask these questions, in order. Give the candidate 60–90 seconds to respond to each before prompting the next.

1. "Please describe the system in your own words — what are the core user-facing capabilities?"
2. "Who are the actors in this system?"
3. "Does a driver actively accept a ride request, or is the driver auto-assigned by the system?"
4. "Can a rider cancel after a driver has been assigned? Can a driver cancel after accepting?"
5. "Do we need surge pricing — that is, multiplying the fare during periods of high demand?"
6. "Is ride-sharing — meaning multiple riders in one car going to different destinations — in scope?"
7. "What should happen if no driver is available within the search radius?"
8. "Do we need to handle payment end-to-end, or just calculate the fare?"

---

### Stage 2: Core Entities

Ask these after the candidate has finished requirements. Allow them to draw or list entities before asking.

1. "What are the core entities you've identified so far?"
2. "What is the difference between a RideRequest and a Ride in your model? Walk me through when each is created."
3. "Where does the lifecycle of a trip begin, and where does it end? Which object owns that lifecycle?"
4. "Is Location a class in your design? Why or why not — could you just use two doubles?"

---

### Stage 3: Class Design

Ask these after the candidate has named entities and sketched relationships.

1. "Walk me through the interfaces you've defined. What contracts have you extracted so far?"
2. "How does a RideRequest become a Ride? What exactly triggers that transition?"
3. "How does the Ride know which driver to assign? Where does that logic live?"
4. "If I want to add a premium matching algorithm based on driver ratings tomorrow, what specifically changes in your design? Show me the delta."

---

### Stage 4: Code a Specific Component

Ask these in order. Give the candidate time to write full code before moving to the next prompt.

1. "Please implement the core domain model: `Ride`, `Driver`, `Rider`, `TripState` enum, `Location`, and `RideRequest`. Take your time — I want to see the full implementations."
2. "Now implement the `RideMatcher` interface and two concrete implementations: `NearestDriverMatcher` and `SurgeAwareMatcher`."
3. "Implement the `FareStrategy` interface with `StandardFareStrategy` and `SurgeFareStrategy`."
4. "Add an observer-based event publisher: implement `RideEvent`, the specific event subtypes, `RideEventListener`, and `RideEventPublisher`."
5. "Finally, implement `RideService` — the orchestrator that ties matching, state transitions, event publishing, and payment together."

---

### Stage 5: Extend the Design

Ask these after the core code is done.

1. "Add support for scheduled rides — a rider books a ride 2 hours in advance. Where does the scheduling information live, and what changes in your existing classes?"
2. "Add support for a rating system — a rider rates the driver and a driver rates the rider after trip completion. Walk me through the design, then write the key classes."
3. "How would you handle the case where payment fails after the trip has already completed and the state has moved to COMPLETED? What state do you transition to, and what events do you publish?"

---

## 3. Expected Senior Response and Flags Per Stage

---

### Stage 1: Requirements — Expected Senior Response

A strong candidate will not wait for the interviewer to ask the clarifying questions. Within the first 30 seconds, they will open proactively: "Before I start designing, let me ask a few questions to scope this correctly." They will ask about driver assignment (manual vs auto), cancellation rules, and surge pricing on their own — these three directly affect entity design, state machine shape, and the FareStrategy interface.

Specifically, a senior candidate will:

- Ask whether drivers **actively accept** or are auto-assigned, noting: "This changes whether I need a `PENDING_ACCEPTANCE` state in my ride state machine. If drivers must explicitly accept, I need to model that window — including what happens if they decline or time out."
- Ask about cancellation from both sides, noting: "This affects which states are 'cancellable' in my transition logic. If a driver can cancel after accepting, I need a DRIVER_CANCELLED transition that returns the ride to a re-matching pool."
- Ask about surge pricing and note immediately: "That suggests the fare calculation should be a Strategy, not hardcoded — I'll keep that in mind when I design FareStrategy."
- Ask about payment processing: "Do I own the payment flow — charging the card — or do I stop at fare calculation?" This scopes out a significant integration concern.
- Explicitly call out scheduled rides as a scope question: "Is booking a ride for a future time in scope? That could affect how I model RideRequest."
- Identify at least three actors: **Rider**, **Driver**, and the **System** (or a background matching/scheduling service). A very strong candidate will mention **Admin** for dispute resolution.

After gathering answers, the candidate explicitly declares in/out of scope:

**In scope**: Immediate ride requests, driver matching (nearest driver), fare calculation, trip lifecycle, cancellation, payment processing, event notification hooks.

**Out of scope**: Ride pooling, driver navigation, live GPS streaming, driver onboarding, admin panel.

This declaration prevents scope creep and signals to the interviewer that the candidate designs deliberately.

### Stage 1 Red Flags

- Starts naming classes (`Ride`, `Driver`, `User`) within 60 seconds without asking a single clarifying question — demonstrates interview pattern-matching rather than system thinking.
- Asks only surface-level, deployment-oriented questions ("Is this a web app or mobile app?", "What database are we using?") that have no bearing on class design or entity modeling.
- Treats `RideRequest` and `Ride` as the same entity from the beginning — says "When a user creates a Ride..." without distinguishing pre-assignment from post-assignment state.
- Does not ask about cancellation flows, missing a critical state machine requirement — the valid cancellation states and the DRIVER_CANCELLED/RIDER_CANCELLED distinction will be invisible to them throughout the design.

### Stage 1 Green Flags

- Asks specifically whether drivers accept rides or are auto-assigned, and explicitly connects the answer to state machine design: "If drivers accept manually, I need a PENDING_ACCEPTANCE state."
- Proactively identifies "scheduled ride vs immediate ride" as a scope clarification, before being asked about it in Stage 5 — this shows forward-thinking design.
- Explicitly states, after gathering requirements, which features are in and which are out of scope — treating the requirements phase as a deliverable, not just a conversation.
- Identifies at least three actors (Rider, Driver, System/Admin) and notes that the System itself is an actor in auto-matching flows.

---

### Stage 2: Core Entities — Expected Senior Response

A senior candidate will immediately distinguish `RideRequest` from `Ride` without being prompted. They will say something like:

"I want to model two separate objects here. A `RideRequest` represents a rider's intent — it exists before any driver is involved. It holds the pickup location, dropoff location, time of request, and eventually a scheduling policy. A `Ride` is created once a driver is matched. It holds a reference to the original request, the assigned driver, the current state in the trip lifecycle, and the fare once calculated. If I merge them into one class, I end up with a `Ride` that has a null `assignedDriver` before matching — which means I can't rely on the type system to enforce that a Ride always has a driver."

The candidate will then sketch entities:

- `RideRequest` — requestId, riderId, pickupLocation, dropoffLocation, requestedAt, vehicleTypePreference, schedulingPolicy
- `Ride` — rideId, request, assignedDriver, state (TripState), startTime, endTime, fareAmount
- `Driver` — driverId, name, currentLocation, vehicle, status (DriverStatus), activeRideId
- `Rider` — riderId, name, email, defaultPaymentMethod
- `Vehicle` — vehicleId, licensePlate, model, vehicleType
- `Location` — latitude, longitude (explained as a value object)

On `Location`, the senior candidate explains: "Location is a value object — it has no identity of its own, only value. Two Locations with the same lat/lon are equal. I'll make it immutable. I won't just use two doubles because I want to attach behavior — specifically `distanceTo(Location other)` using the Haversine formula — to the type itself rather than scattering that calculation across the codebase."

On the trip lifecycle: "The lifecycle begins when a `Ride` object is created — that's when a driver is assigned. It ends when the state transitions to COMPLETED, CANCELLED, or PAYMENT_FAILED. The `Ride` object owns the lifecycle. The `RideRequest` is essentially immutable after creation."

### Stage 2 Red Flags

- Introduces a `User` superclass with `Driver` and `Rider` extending it — a common but problematic choice that leads to the Liskov substitution problem (a Rider cannot go offline, a Driver cannot be a passenger in their own car) and unnecessarily couples two distinct domain concepts.
- Models `Location` as a mutable class with setters — breaks value-object semantics and introduces thread safety issues.
- Cannot articulate a clear answer to "where does the lifecycle begin and end" — says "the Ride class manages everything" without distinguishing creation from termination.
- Forgets to model `DriverStatus` as a separate enum, instead planning to store a String "AVAILABLE"/"ON_TRIP" on the Driver — loses type safety.

### Stage 2 Green Flags

- Immediately separates `RideRequest` from `Ride` and explains the distinction in terms of lifecycle, not just fields.
- Explains `Location` as a value object and mentions `distanceTo` as the behavior that justifies making it a class.
- Mentions `DriverStatus` as a separate enum with at least three states: AVAILABLE, ON_TRIP, OFFLINE.
- Notes that `Ride.request` is a reference to `RideRequest` rather than copying its fields into `Ride` — preserving the original intent and enabling immutable audit trail.

---

### Stage 3: Class Design — Expected Senior Response

A strong candidate will define the following interfaces and explain the rationale for each before writing any concrete class.

**`RideMatcher` interface**: "I'll extract the matching logic into a `RideMatcher` interface with one method: `Optional<Driver> match(RideRequest request, List<Driver> availableDrivers)`. Today's implementation is nearest driver. Tomorrow it could be ML-based, rating-weighted, or surge-aware. By isolating this behind an interface, `RideService` never needs to change when the algorithm changes."

**`FareStrategy` interface**: "Fare calculation is a classic Strategy pattern opportunity. I'll define `FareStrategy` with `Money calculate(Ride ride)`. `StandardFareStrategy` computes base fare + per-minute + per-km. `SurgeFareStrategy` is a decorator that wraps any `FareStrategy` and multiplies the result — this means I can compose `new SurgeFareStrategy(2.0, new StandardFareStrategy())` without modifying either class."

**`RideEventListener` and `RideEventPublisher`**: "Rather than calling notification or analytics services directly in `RideService`, I'll use an observer pattern. `RideEvent` is an abstract base class with event metadata. `RideEventListener` is an interface with `onEvent(RideEvent)`. `RideEventPublisher` holds a thread-safe list of listeners and fan-outs each event to all of them. `RideService` only knows about the publisher — it never directly depends on any consumer."

**`SchedulingPolicy` interface**: "I'll model the scheduling behavior as a small policy interface: `isImmediate()` and `getScheduledTime()`. `ImmediateSchedulingPolicy` returns true/empty. `ScheduledSchedulingPolicy` holds a future Instant. `RideRequest` holds a `SchedulingPolicy` — which means scheduled rides are already modeled in the domain without any change to `Ride` or `RideService`."

On how a `RideRequest` becomes a `Ride`: "In `RideService.requestRide()`, I call `matcher.match(request, availableDrivers)`. If a driver is found, I construct `new Ride(request, driver)` — the `Ride` constructor sets the initial state to `DRIVER_ASSIGNED` and marks the driver's status as `ON_TRIP`. I then publish a `DriverAssignedEvent`."

On what changes to add a new matching algorithm: "Only one thing changes: the concrete class injected into `RideService` at construction time. The service itself, the Ride, the Driver — nothing changes. This is the open/closed principle in practice."

### Stage 3 Red Flags

- Defines no interfaces — puts all matching logic directly in `RideService` as private methods, making it untestable and impossible to extend without modification.
- Suggests inheritance for fare variants ("`SurgeFareStrategy extends StandardFareStrategy`") rather than composition — tightly couples strategies and violates single responsibility.
- Calls `notificationService.sendRideCompletedNotification(...)` directly in `completeRide` without any observer abstraction — creates a fan-out of direct dependencies in the service.
- Cannot explain how a new matching algorithm would be added without modification — says "I'd add an if/else in the matcher" or "I'd add a flag to RideService".

### Stage 3 Green Flags

- Introduces `RideMatcher` and `FareStrategy` interfaces before being asked about extensibility — shows the candidate is thinking in terms of contracts, not implementations.
- Designs `SurgeFareStrategy` as a decorator over `FareStrategy` rather than a separate subclass — demonstrates understanding of the decorator pattern.
- Introduces `RideEventPublisher` and explains the observer pattern rationale in terms of decoupling, not just notification.
- Notes that `SchedulingPolicy` on `RideRequest` handles the scheduled-ride case without touching `Ride` — forward-thinking design.

---

### Stage 4: Code — Expected Senior Response

A senior candidate coding the domain model will make the following decisions visible through their code:

**TripState enum**: Will include all seven states and — critically — will implement `canTransitionTo(TripState next)` as a method on the enum using a switch statement that enumerates valid next-states per current state. Will not rely on the calling code to enforce transition validity. Will implement `isTerminal()` to mark COMPLETED, CANCELLED, and PAYMENT_FAILED.

**Location**: Will implement `distanceTo` using the actual Haversine formula — not a straight-line Euclidean approximation on lat/lon degrees (which is incorrect for geographic coordinates). Will override `equals` and `hashCode` for value-object semantics.

**Money**: Will use `BigDecimal` (not `double`) for the amount field — the senior candidate knows that floating-point arithmetic is inappropriate for currency. Will implement `add` and `multiply` correctly. Will make the class immutable.

**RideRequest with SchedulingPolicy**: Will introduce the `SchedulingPolicy` interface and two implementations during entity coding — not prompted by the Stage 5 extension questions. This demonstrates that the candidate designs for extensibility proactively.

**RideMatcher**: Will correctly filter by `vehicleTypePreference` in `NearestDriverMatcher` before sorting by distance. Will not sort all drivers — will use a stream with `min()` comparator for O(n) instead of O(n log n).

**RideService**: Will keep the service thin — it orchestrates, it does not contain business logic. Fare calculation is in FareStrategy. Transition validation is in TripState. Matching is in RideMatcher. The service only wires them together, handles the side effects (updating driver status, publishing events), and handles the payment failure path.

**Payment failure path**: Will correctly transition to `PAYMENT_FAILED` state (not CANCELLED), publish an appropriate event, and note that a retry mechanism or manual reconciliation process would handle recovery in production.

### Stage 4 Red Flags

- Uses `double` or `float` for money amounts — a hard disqualifier for any financial calculation.
- Implements `canTransitionTo` in the calling code (service or ride) rather than on the enum itself — misses the encapsulation opportunity and scatters transition logic.
- Implements `distanceTo` using `Math.sqrt((lat2-lat1)^2 + (lon2-lon1)^2)` — incorrect for geographic coordinates, reveals lack of awareness that lat/lon degrees are not uniform distances.
- Throws a generic `RuntimeException` or `Exception` from `RideService` — senior code defines domain-specific exceptions like `NoDriverAvailableException`.

### Stage 4 Green Flags

- Uses `BigDecimal` for `Money.amount` and explains why.
- Implements Haversine formula correctly in `Location.distanceTo`.
- Encodes transition validity in `TripState.canTransitionTo` as a switch on `this`, not in the service.
- Uses `CopyOnWriteArrayList` in `RideEventPublisher` and explains the reason: listeners can be registered/deregistered from multiple threads.
- Catches exceptions per listener in `RideEventPublisher.publish` so a failing listener does not break the publish loop.

---

### Stage 5: Extension — Expected Senior Response

**Scheduled rides**: A senior candidate who designed `SchedulingPolicy` on `RideRequest` in Stage 4 will say: "This is already handled. `RideRequest` has a `SchedulingPolicy` field. The `ScheduledSchedulingPolicy` implementation holds a future Instant. A scheduler service polls for requests whose scheduled time has arrived and calls `requestRide`. Neither `Ride` nor `RideService` changes at all." This is a direct demonstration of the value of the earlier design decision.

**Rating system**: The candidate will introduce a `Rating` value object (score 1–5, optional comment, ratedAt), a `RatingService` that accepts post-trip ratings, and note that ratings are logically separate from the ride lifecycle — a `Ride` does not need a `rating` field. Ratings are stored indexed by `rideId`. The candidate may suggest that `RideCompletedEvent` triggers the rating prompt in the notification service — keeping `RideService` decoupled.

**Payment failure**: "When payment fails, I transition the Ride from COMPLETED — or from RIDE_IN_PROGRESS, depending on when I attempt payment — to PAYMENT_FAILED. I publish a `PaymentFailedEvent`. I do not call it CANCELLED because the distinction matters for reconciliation: a PAYMENT_FAILED ride was completed but not settled, while a CANCELLED ride was never started. The retry system or customer support flow picks up PAYMENT_FAILED rides for resolution." This answer demonstrates that the candidate thinks about downstream data consumers, not just the state machine.

### Stage 5 Red Flags

- Cannot accommodate scheduled rides without modifying `Ride` or `RideService` — indicates the SchedulingPolicy abstraction was missing from the design.
- Adds a `rating` field directly to `Ride` — couples the ride lifecycle to the rating lifecycle, and fails to handle the case where a rider doesn't rate immediately.
- Calls payment failure state CANCELLED — confuses operational states that matter to analytics and reconciliation systems.

### Stage 5 Green Flags

- Demonstrates that scheduled rides require zero changes to `Ride` or `RideService` because `SchedulingPolicy` already handles it.
- Proposes a separate `RatingService` and `Rating` value object rather than embedding ratings in `Ride`.
- Distinguishes `PAYMENT_FAILED` from `CANCELLED` and articulates why the distinction matters for downstream systems.
- Mentions that in production, `RideEventPublisher` would publish to a message broker (Kafka, SQS) rather than calling in-process listeners — acknowledging the boundary between LLD and production architecture.

---

## 4. Model Solution

### 4.1 Complete Class Diagram (Text Form)

```
=== ENUMS ===

TripState (enum)
  Values: REQUESTED, DRIVER_ASSIGNED, EN_ROUTE_TO_PICKUP,
          RIDE_IN_PROGRESS, COMPLETED, CANCELLED, PAYMENT_FAILED
  Methods:
    + isTerminal() : boolean
    + canTransitionTo(TripState next) : boolean

DriverStatus (enum)
  Values: AVAILABLE, ON_TRIP, OFFLINE

VehicleType (enum)
  Values: STANDARD, PREMIUM, XL

PaymentMethod (enum)
  Values: CREDIT_CARD, DEBIT_CARD, WALLET, UPI


=== VALUE OBJECTS ===

Location (final class)
  Fields:
    - latitude  : double
    - longitude : double
  Methods:
    + distanceTo(Location other) : double    [Haversine formula, returns km]
    + getLatitude() : double
    + getLongitude() : double
    + equals(Object) : boolean
    + hashCode() : int
    + toString() : String

Money (final class)
  Fields:
    - amount   : BigDecimal
    - currency : String
  Methods:
    + add(Money other) : Money
    + multiply(double factor) : Money
    + getAmount() : BigDecimal
    + getCurrency() : String
    + equals(Object) : boolean
    + hashCode() : int
    + toString() : String
    + static of(double amount, String currency) : Money


=== SCHEDULING POLICY ===

SchedulingPolicy (interface)
  Methods:
    + isImmediate() : boolean
    + getScheduledTime() : Optional<Instant>

ImmediateSchedulingPolicy (class)
  Implements: SchedulingPolicy

ScheduledSchedulingPolicy (class)
  Implements: SchedulingPolicy
  Fields:
    - scheduledTime : Instant


=== DOMAIN ENTITIES ===

Vehicle (class)
  Fields:
    - vehicleId    : String
    - licensePlate : String
    - model        : String
    - vehicleType  : VehicleType

Rider (class)
  Fields:
    - riderId              : String
    - name                 : String
    - email                : String
    - defaultPaymentMethod : PaymentMethod

Driver (class)
  Fields:
    - driverId        : String
    - name            : String
    - email           : String
    - currentLocation : Location   [mutable]
    - vehicle         : Vehicle
    - status          : DriverStatus  [mutable]
    - activeRideId    : String     [nullable, mutable]
  Methods:
    + isAvailable() : boolean
    + setCurrentLocation(Location) : void
    + setStatus(DriverStatus) : void
    + setActiveRideId(String) : void
    + getters for all fields

RideRequest (class)
  Fields:
    - requestId             : String   [UUID, generated on construction]
    - riderId               : String
    - pickupLocation        : Location
    - dropoffLocation       : Location
    - requestedAt           : Instant
    - vehicleTypePreference : VehicleType  [nullable]
    - schedulingPolicy      : SchedulingPolicy
  Methods:
    + isScheduled() : boolean
    + getters for all fields
    + static Builder class

Ride (class)
  Fields:
    - rideId         : String   [UUID, generated on construction]
    - request        : RideRequest
    - assignedDriver : Driver
    - state          : TripState  [mutable]
    - startTime      : Instant    [nullable, mutable]
    - endTime        : Instant    [nullable, mutable]
    - fareAmount     : Money      [nullable, mutable]
  Constructor: Ride(RideRequest request, Driver assignedDriver)
    -> sets state to DRIVER_ASSIGNED
  Methods:
    + transitionTo(TripState newState) : void
        [validates via TripState.canTransitionTo, throws IllegalStateException on invalid]
        [sets startTime on EN_ROUTE_TO_PICKUP, sets endTime on COMPLETED/CANCELLED/PAYMENT_FAILED]
    + getDurationMinutes() : long
    + isCompleted() : boolean
    + getters for all fields


=== MATCHING ===

RideMatcher (interface)
  Methods:
    + match(RideRequest request, List<Driver> availableDrivers) : Optional<Driver>

NearestDriverMatcher (class)
  Implements: RideMatcher
  Methods:
    + match(RideRequest request, List<Driver> availableDrivers) : Optional<Driver>

SurgeAwareMatcher (class)
  Implements: RideMatcher
  Fields:
    - surgeMultiplier     : double
    - fallbackMatcher     : RideMatcher
  Methods:
    + match(RideRequest request, List<Driver> availableDrivers) : Optional<Driver>


=== FARE CALCULATION ===

FareStrategy (interface)
  Methods:
    + calculate(Ride ride) : Money

StandardFareStrategy (class)
  Implements: FareStrategy
  Constants: BASE_FARE=2.00, PER_MINUTE_RATE=0.30, PER_KM_RATE=1.20
  Methods:
    + calculate(Ride ride) : Money

SurgeFareStrategy (class)
  Implements: FareStrategy
  Fields:
    - surgeMultiplier : double
    - delegate        : FareStrategy
  Methods:
    + calculate(Ride ride) : Money


=== EVENTS ===

RideEvent (abstract class)
  Fields:
    - eventId    : String   [UUID]
    - occurredAt : Instant
    - rideId     : String
  Methods:
    + abstract getEventType() : String
    + getters for all fields

RideRequestedEvent (extends RideEvent)
  Fields:
    - riderId : String
  Methods:
    + getEventType() : String  -> "RIDE_REQUESTED"

DriverAssignedEvent (extends RideEvent)
  Fields:
    - driverId : String
  Methods:
    + getEventType() : String  -> "DRIVER_ASSIGNED"

RideCompletedEvent (extends RideEvent)
  Fields:
    - fareAmount : Money
  Methods:
    + getEventType() : String  -> "RIDE_COMPLETED"

RideCancelledEvent (extends RideEvent)
  Fields:
    - cancelledBy : String   ["RIDER", "DRIVER", "SYSTEM"]
    - reason      : String
  Methods:
    + getEventType() : String  -> "RIDE_CANCELLED"

RideEventListener (interface)
  Methods:
    + onEvent(RideEvent event) : void

RideEventPublisher (class)
  Fields:
    - listeners : CopyOnWriteArrayList<RideEventListener>
  Methods:
    + register(RideEventListener listener) : void
    + deregister(RideEventListener listener) : void
    + publish(RideEvent event) : void


=== PAYMENT ===

PaymentProcessor (interface)
  Methods:
    + processPayment(String riderId, Money amount, PaymentMethod method) : PaymentResult

PaymentResult (class)
  Fields:
    - success         : boolean
    - transactionId   : String  [nullable]
    - failureReason   : String  [nullable]
  Methods:
    + isSuccess() : boolean
    + getTransactionId() : Optional<String>
    + getFailureReason() : Optional<String>
    + static success(String transactionId) : PaymentResult
    + static failure(String reason) : PaymentResult


=== SERVICE ===

RideService (class)
  Fields:
    - matcher          : RideMatcher
    - fareStrategy     : FareStrategy
    - eventPublisher   : RideEventPublisher
    - paymentProcessor : PaymentProcessor
  Constructor: RideService(RideMatcher, FareStrategy, RideEventPublisher, PaymentProcessor)
  Methods:
    + requestRide(RideRequest request, List<Driver> availableDrivers) : Ride
        [throws NoDriverAvailableException]
    + startRide(Ride ride) : void
    + completeRide(Ride ride) : void
    + cancelRide(Ride ride, String cancelledBy, String reason) : void


=== EXCEPTIONS ===

NoDriverAvailableException (class, extends RuntimeException)
  Fields:
    - requestId : String


=== RELATIONSHIPS ===

Ride          has-a   RideRequest           (composition)
Ride          has-a   Driver                (association)
Ride          has-a   TripState             (enum field)
Ride          has-a   Money (fareAmount)    (association, nullable)

RideRequest   has-a   Location (pickup)     (composition)
RideRequest   has-a   Location (dropoff)    (composition)
RideRequest   has-a   SchedulingPolicy      (composition)

Driver        has-a   Location              (association, mutable)
Driver        has-a   Vehicle               (composition)
Driver        has-a   DriverStatus          (enum field)

Rider         has-a   PaymentMethod         (enum field)

Vehicle       has-a   VehicleType           (enum field)

NearestDriverMatcher  implements  RideMatcher
SurgeAwareMatcher     implements  RideMatcher
SurgeAwareMatcher     has-a       RideMatcher (delegate/fallback)

StandardFareStrategy  implements  FareStrategy
SurgeFareStrategy     implements  FareStrategy
SurgeFareStrategy     has-a       FareStrategy (delegate, decorator pattern)

RideRequestedEvent    extends  RideEvent
DriverAssignedEvent   extends  RideEvent
RideCompletedEvent    extends  RideEvent
RideCancelledEvent    extends  RideEvent

ImmediateSchedulingPolicy  implements  SchedulingPolicy
ScheduledSchedulingPolicy  implements  SchedulingPolicy

RideService   uses    RideMatcher
RideService   uses    FareStrategy
RideService   uses    RideEventPublisher
RideService   uses    PaymentProcessor
```

---

### 4.2 Full Java Implementation

All classes are written in dependency order: value objects first, then domain entities, then behavioral components, then the service.

---

#### TripState.java

```java
package rideshar.domain;

public enum TripState {

    REQUESTED,
    DRIVER_ASSIGNED,
    EN_ROUTE_TO_PICKUP,
    RIDE_IN_PROGRESS,
    COMPLETED,
    CANCELLED,
    PAYMENT_FAILED;

    /**
     * Returns true if no further transitions are possible from this state.
     */
    public boolean isTerminal() {
        return this == COMPLETED || this == CANCELLED || this == PAYMENT_FAILED;
    }

    /**
     * Returns true if transitioning from the current state to {@code next} is a
     * valid lifecycle step.  All invalid transitions are implicitly rejected.
     *
     * Valid path (happy):  REQUESTED -> DRIVER_ASSIGNED -> EN_ROUTE_TO_PICKUP
     *                      -> RIDE_IN_PROGRESS -> COMPLETED
     *
     * Cancellation:        REQUESTED -> CANCELLED
     *                      DRIVER_ASSIGNED -> CANCELLED
     *                      EN_ROUTE_TO_PICKUP -> CANCELLED
     *
     * Payment failure:     RIDE_IN_PROGRESS -> PAYMENT_FAILED
     *                      (payment attempted at ride completion)
     */
    public boolean canTransitionTo(TripState next) {
        switch (this) {
            case REQUESTED:
                return next == DRIVER_ASSIGNED || next == CANCELLED;
            case DRIVER_ASSIGNED:
                return next == EN_ROUTE_TO_PICKUP || next == CANCELLED;
            case EN_ROUTE_TO_PICKUP:
                return next == RIDE_IN_PROGRESS || next == CANCELLED;
            case RIDE_IN_PROGRESS:
                return next == COMPLETED || next == PAYMENT_FAILED;
            case COMPLETED:
            case CANCELLED:
            case PAYMENT_FAILED:
                // Terminal states — no further transitions allowed.
                return false;
            default:
                return false;
        }
    }
}
```

---

#### DriverStatus.java

```java
package rideshar.domain;

public enum DriverStatus {
    /** Driver is logged in and ready to accept rides. */
    AVAILABLE,

    /** Driver is currently on an active trip. */
    ON_TRIP,

    /** Driver has gone offline and is not accepting rides. */
    OFFLINE
}
```

---

#### VehicleType.java

```java
package rideshar.domain;

public enum VehicleType {
    /** Economy class vehicle — standard sedan. */
    STANDARD,

    /** Business class vehicle — higher-end make/model. */
    PREMIUM,

    /** Large capacity vehicle — SUV or minivan. */
    XL
}
```

---

#### PaymentMethod.java

```java
package rideshar.domain;

public enum PaymentMethod {
    CREDIT_CARD,
    DEBIT_CARD,
    WALLET,
    UPI
}
```

---

#### Location.java

```java
package rideshar.domain;

import java.util.Objects;

/**
 * Immutable value object representing a geographic coordinate.
 * Two Location instances are equal if and only if they have the same
 * latitude and longitude.
 */
public final class Location {

    private final double latitude;
    private final double longitude;

    public Location(double latitude, double longitude) {
        if (latitude < -90 || latitude > 90) {
            throw new IllegalArgumentException(
                "Latitude must be between -90 and 90, got: " + latitude);
        }
        if (longitude < -180 || longitude > 180) {
            throw new IllegalArgumentException(
                "Longitude must be between -180 and 180, got: " + longitude);
        }
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
     * Calculates the great-circle distance between this location and another
     * using the Haversine formula. Returns the distance in kilometres.
     *
     * The Haversine formula:
     *   a = sin²(Δlat/2) + cos(lat1) * cos(lat2) * sin²(Δlon/2)
     *   c = 2 * atan2(√a, √(1−a))
     *   d = R * c    where R = 6371 km (Earth's mean radius)
     */
    public double distanceTo(Location other) {
        final double R = 6371.0; // Earth's mean radius in kilometres

        double lat1Rad = Math.toRadians(this.latitude);
        double lat2Rad = Math.toRadians(other.latitude);
        double deltaLat = Math.toRadians(other.latitude - this.latitude);
        double deltaLon = Math.toRadians(other.longitude - this.longitude);

        double sinHalfDeltaLat = Math.sin(deltaLat / 2);
        double sinHalfDeltaLon = Math.sin(deltaLon / 2);

        double a = sinHalfDeltaLat * sinHalfDeltaLat
                 + Math.cos(lat1Rad) * Math.cos(lat2Rad)
                   * sinHalfDeltaLon * sinHalfDeltaLon;

        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

        return R * c;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Location)) return false;
        Location other = (Location) obj;
        return Double.compare(latitude, other.latitude) == 0
            && Double.compare(longitude, other.longitude) == 0;
    }

    @Override
    public int hashCode() {
        return Objects.hash(latitude, longitude);
    }

    @Override
    public String toString() {
        return String.format("Location{lat=%.6f, lon=%.6f}", latitude, longitude);
    }
}
```

---

#### Money.java

```java
package rideshar.domain;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Objects;

/**
 * Immutable value object representing a monetary amount with a currency code.
 * Uses BigDecimal internally to avoid floating-point rounding errors.
 * All arithmetic operations return new Money instances.
 */
public final class Money {

    private final BigDecimal amount;
    private final String currency;

    private Money(BigDecimal amount, String currency) {
        Objects.requireNonNull(amount, "amount must not be null");
        Objects.requireNonNull(currency, "currency must not be null");
        if (currency.isBlank()) {
            throw new IllegalArgumentException("currency must not be blank");
        }
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency.toUpperCase();
    }

    /**
     * Factory method. Converts a double amount to BigDecimal immediately,
     * rounding to 2 decimal places using HALF_UP.
     */
    public static Money of(double amount, String currency) {
        return new Money(BigDecimal.valueOf(amount), currency);
    }

    /**
     * Factory method for use when the amount is already a BigDecimal.
     */
    public static Money of(BigDecimal amount, String currency) {
        return new Money(amount, currency);
    }

    /**
     * Adds another Money value to this one and returns a new Money.
     * Both must have the same currency.
     *
     * @throws IllegalArgumentException if currencies differ
     */
    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    /**
     * Multiplies this amount by a scalar factor and returns a new Money.
     * The factor is converted to BigDecimal using its string representation
     * to avoid binary floating-point imprecision.
     */
    public Money multiply(double factor) {
        BigDecimal result = this.amount.multiply(
            new BigDecimal(String.valueOf(factor)));
        return new Money(result, this.currency);
    }

    public BigDecimal getAmount() {
        return amount;
    }

    public String getCurrency() {
        return currency;
    }

    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Cannot add Money with different currencies: "
                + this.currency + " vs " + other.currency);
        }
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Money)) return false;
        Money other = (Money) obj;
        return this.amount.compareTo(other.amount) == 0
            && this.currency.equals(other.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount.stripTrailingZeros(), currency);
    }

    @Override
    public String toString() {
        return currency + " " + amount.toPlainString();
    }
}
```

---

#### SchedulingPolicy.java

```java
package rideshar.domain;

import java.time.Instant;
import java.util.Optional;

/**
 * Describes when a ride should be dispatched for matching.
 * Immediate rides are matched as soon as a request is created.
 * Scheduled rides are held until a specified future time.
 */
public interface SchedulingPolicy {

    /** Returns true if this ride should be dispatched immediately. */
    boolean isImmediate();

    /**
     * Returns the scheduled dispatch time for non-immediate rides.
     * Returns empty for immediate rides.
     */
    Optional<Instant> getScheduledTime();
}
```

---

#### ImmediateSchedulingPolicy.java

```java
package rideshar.domain;

import java.time.Instant;
import java.util.Optional;

/**
 * Policy for rides that should be matched and dispatched immediately
 * upon request creation.
 */
public final class ImmediateSchedulingPolicy implements SchedulingPolicy {

    public static final ImmediateSchedulingPolicy INSTANCE =
        new ImmediateSchedulingPolicy();

    private ImmediateSchedulingPolicy() {}

    @Override
    public boolean isImmediate() {
        return true;
    }

    @Override
    public Optional<Instant> getScheduledTime() {
        return Optional.empty();
    }

    @Override
    public String toString() {
        return "ImmediateSchedulingPolicy";
    }
}
```

---

#### ScheduledSchedulingPolicy.java

```java
package rideshar.domain;

import java.time.Instant;
import java.util.Objects;
import java.util.Optional;

/**
 * Policy for rides that should be dispatched at a specific future time.
 * The scheduled time must be in the future relative to when the policy
 * is created, though enforcement of this constraint is left to the
 * application layer.
 */
public final class ScheduledSchedulingPolicy implements SchedulingPolicy {

    private final Instant scheduledTime;

    public ScheduledSchedulingPolicy(Instant scheduledTime) {
        this.scheduledTime = Objects.requireNonNull(scheduledTime,
            "scheduledTime must not be null");
    }

    @Override
    public boolean isImmediate() {
        return false;
    }

    @Override
    public Optional<Instant> getScheduledTime() {
        return Optional.of(scheduledTime);
    }

    @Override
    public String toString() {
        return "ScheduledSchedulingPolicy{scheduledTime=" + scheduledTime + "}";
    }
}
```

---

#### Vehicle.java

```java
package rideshar.domain;

import java.util.Objects;

/**
 * Represents a vehicle registered by a driver on the platform.
 */
public class Vehicle {

    private final String vehicleId;
    private final String licensePlate;
    private final String model;
    private final VehicleType vehicleType;

    public Vehicle(String vehicleId,
                   String licensePlate,
                   String model,
                   VehicleType vehicleType) {
        this.vehicleId    = Objects.requireNonNull(vehicleId,    "vehicleId");
        this.licensePlate = Objects.requireNonNull(licensePlate, "licensePlate");
        this.model        = Objects.requireNonNull(model,        "model");
        this.vehicleType  = Objects.requireNonNull(vehicleType,  "vehicleType");
    }

    public String getVehicleId()    { return vehicleId; }
    public String getLicensePlate() { return licensePlate; }
    public String getModel()        { return model; }
    public VehicleType getVehicleType() { return vehicleType; }

    @Override
    public String toString() {
        return "Vehicle{id=" + vehicleId
             + ", plate=" + licensePlate
             + ", model=" + model
             + ", type=" + vehicleType + "}";
    }
}
```

---

#### Rider.java

```java
package rideshar.domain;

import java.util.Objects;

/**
 * A registered user who can request rides on the platform.
 */
public class Rider {

    private final String riderId;
    private final String name;
    private final String email;
    private final PaymentMethod defaultPaymentMethod;

    public Rider(String riderId,
                 String name,
                 String email,
                 PaymentMethod defaultPaymentMethod) {
        this.riderId               = Objects.requireNonNull(riderId,  "riderId");
        this.name                  = Objects.requireNonNull(name,     "name");
        this.email                 = Objects.requireNonNull(email,    "email");
        this.defaultPaymentMethod  = Objects.requireNonNull(
            defaultPaymentMethod, "defaultPaymentMethod");
    }

    public String getRiderId()                  { return riderId; }
    public String getName()                     { return name; }
    public String getEmail()                    { return email; }
    public PaymentMethod getDefaultPaymentMethod() { return defaultPaymentMethod; }

    @Override
    public String toString() {
        return "Rider{id=" + riderId + ", name=" + name + ", email=" + email + "}";
    }
}
```

---

#### Driver.java

```java
package rideshar.domain;

import java.util.Objects;

/**
 * A driver registered on the platform who can accept ride assignments.
 *
 * Mutable fields: currentLocation, status, activeRideId — these change
 * continuously as the driver moves and takes trips.
 */
public class Driver {

    private final String driverId;
    private final String name;
    private final String email;
    private final Vehicle vehicle;

    private Location currentLocation;   // mutable — driver moves
    private DriverStatus status;        // mutable — changes with assignments
    private String activeRideId;        // nullable — null when not on a trip

    public Driver(String driverId,
                  String name,
                  String email,
                  Vehicle vehicle,
                  Location initialLocation) {
        this.driverId         = Objects.requireNonNull(driverId,         "driverId");
        this.name             = Objects.requireNonNull(name,             "name");
        this.email            = Objects.requireNonNull(email,            "email");
        this.vehicle          = Objects.requireNonNull(vehicle,          "vehicle");
        this.currentLocation  = Objects.requireNonNull(initialLocation,  "initialLocation");
        this.status           = DriverStatus.AVAILABLE;
        this.activeRideId     = null;
    }

    public String getDriverId()             { return driverId; }
    public String getName()                 { return name; }
    public String getEmail()                { return email; }
    public Vehicle getVehicle()             { return vehicle; }
    public Location getCurrentLocation()   { return currentLocation; }
    public DriverStatus getStatus()         { return status; }
    public String getActiveRideId()         { return activeRideId; }

    public void setCurrentLocation(Location location) {
        this.currentLocation = Objects.requireNonNull(location, "location");
    }

    public void setStatus(DriverStatus status) {
        this.status = Objects.requireNonNull(status, "status");
    }

    public void setActiveRideId(String rideId) {
        // rideId may be null (clearing the active ride after completion)
        this.activeRideId = rideId;
    }

    /**
     * Convenience method: returns true only when status is AVAILABLE.
     * Callers should filter on this before passing drivers to the matcher.
     */
    public boolean isAvailable() {
        return this.status == DriverStatus.AVAILABLE;
    }

    @Override
    public String toString() {
        return "Driver{id=" + driverId
             + ", name=" + name
             + ", status=" + status
             + ", location=" + currentLocation + "}";
    }
}
```

---

#### RideRequest.java

```java
package rideshar.domain;

import java.time.Instant;
import java.util.Objects;
import java.util.UUID;

/**
 * Represents a rider's intent to travel from a pickup location to a dropoff
 * location.  A RideRequest exists before any driver is assigned.
 *
 * Once created, a RideRequest is effectively immutable — it records what the
 * rider asked for.  The Ride object records what actually happened.
 *
 * Use the nested Builder for construction.
 */
public class RideRequest {

    private final String requestId;
    private final String riderId;
    private final Location pickupLocation;
    private final Location dropoffLocation;
    private final Instant requestedAt;
    private final VehicleType vehicleTypePreference;  // nullable
    private final SchedulingPolicy schedulingPolicy;

    private RideRequest(Builder builder) {
        this.requestId             = UUID.randomUUID().toString();
        this.riderId               = builder.riderId;
        this.pickupLocation        = builder.pickupLocation;
        this.dropoffLocation       = builder.dropoffLocation;
        this.requestedAt           = builder.requestedAt != null
                                     ? builder.requestedAt : Instant.now();
        this.vehicleTypePreference = builder.vehicleTypePreference;
        this.schedulingPolicy      = builder.schedulingPolicy != null
                                     ? builder.schedulingPolicy
                                     : ImmediateSchedulingPolicy.INSTANCE;
    }

    public String getRequestId()               { return requestId; }
    public String getRiderId()                 { return riderId; }
    public Location getPickupLocation()        { return pickupLocation; }
    public Location getDropoffLocation()       { return dropoffLocation; }
    public Instant getRequestedAt()            { return requestedAt; }
    public VehicleType getVehicleTypePreference() { return vehicleTypePreference; }
    public SchedulingPolicy getSchedulingPolicy() { return schedulingPolicy; }

    /**
     * Convenience: returns true if this request has a future scheduled time.
     */
    public boolean isScheduled() {
        return !schedulingPolicy.isImmediate();
    }

    @Override
    public String toString() {
        return "RideRequest{id=" + requestId
             + ", riderId=" + riderId
             + ", pickup=" + pickupLocation
             + ", dropoff=" + dropoffLocation
             + ", policy=" + schedulingPolicy + "}";
    }

    // -------------------------------------------------------------------------
    // Builder
    // -------------------------------------------------------------------------

    public static Builder builder(String riderId,
                                  Location pickupLocation,
                                  Location dropoffLocation) {
        return new Builder(riderId, pickupLocation, dropoffLocation);
    }

    public static final class Builder {

        private final String riderId;
        private final Location pickupLocation;
        private final Location dropoffLocation;

        private Instant requestedAt;
        private VehicleType vehicleTypePreference;
        private SchedulingPolicy schedulingPolicy;

        private Builder(String riderId,
                        Location pickupLocation,
                        Location dropoffLocation) {
            this.riderId          = Objects.requireNonNull(riderId,          "riderId");
            this.pickupLocation   = Objects.requireNonNull(pickupLocation,   "pickupLocation");
            this.dropoffLocation  = Objects.requireNonNull(dropoffLocation,  "dropoffLocation");
        }

        public Builder requestedAt(Instant requestedAt) {
            this.requestedAt = requestedAt;
            return this;
        }

        public Builder vehicleTypePreference(VehicleType type) {
            this.vehicleTypePreference = type;
            return this;
        }

        public Builder schedulingPolicy(SchedulingPolicy policy) {
            this.schedulingPolicy = policy;
            return this;
        }

        public RideRequest build() {
            return new RideRequest(this);
        }
    }
}
```

---

#### Ride.java

```java
package rideshar.domain;

import java.time.Duration;
import java.time.Instant;
import java.util.Objects;
import java.util.UUID;

/**
 * Represents an active or completed ride engagement between a rider and a
 * driver.  A Ride is created only after a driver has been matched to a
 * RideRequest — it never exists in an unmatched state.
 *
 * The Ride owns the trip lifecycle.  State transitions are validated against
 * TripState.canTransitionTo to ensure the lifecycle cannot be corrupted by
 * callers.
 */
public class Ride {

    private final String rideId;
    private final RideRequest request;
    private final Driver assignedDriver;

    private TripState state;
    private Instant startTime;    // set when EN_ROUTE_TO_PICKUP begins
    private Instant endTime;      // set when state reaches a terminal state
    private Money fareAmount;     // set after fare is calculated at completion

    /**
     * Creates a new Ride.  The initial state is DRIVER_ASSIGNED because a Ride
     * can only be constructed after the matcher has found a driver.
     */
    public Ride(RideRequest request, Driver assignedDriver) {
        this.rideId          = UUID.randomUUID().toString();
        this.request         = Objects.requireNonNull(request,        "request");
        this.assignedDriver  = Objects.requireNonNull(assignedDriver, "assignedDriver");
        this.state           = TripState.DRIVER_ASSIGNED;
        this.startTime       = null;
        this.endTime         = null;
        this.fareAmount      = null;
    }

    public String getRideId()             { return rideId; }
    public RideRequest getRequest()       { return request; }
    public Driver getAssignedDriver()     { return assignedDriver; }
    public TripState getState()           { return state; }
    public Instant getStartTime()         { return startTime; }
    public Instant getEndTime()           { return endTime; }
    public Money getFareAmount()          { return fareAmount; }

    public void setFareAmount(Money fareAmount) {
        this.fareAmount = fareAmount;
    }

    /**
     * Transitions the Ride to the given state after validating that the
     * transition is legal.  Sets timestamps on relevant transitions.
     *
     * @throws IllegalStateException if the transition is not valid
     */
    public void transitionTo(TripState newState) {
        if (!this.state.canTransitionTo(newState)) {
            throw new IllegalStateException(
                "Invalid state transition: " + this.state + " -> " + newState
                + " for ride " + rideId);
        }

        this.state = newState;

        // Record start time when the driver begins moving to the pickup point.
        if (newState == TripState.EN_ROUTE_TO_PICKUP) {
            this.startTime = Instant.now();
        }

        // Record end time when the ride reaches any terminal state.
        if (newState.isTerminal()) {
            this.endTime = Instant.now();
        }
    }

    /**
     * Calculates the elapsed duration of the trip in whole minutes.
     * Returns 0 if the ride has not started or not yet ended.
     */
    public long getDurationMinutes() {
        if (startTime == null || endTime == null) {
            return 0;
        }
        return Duration.between(startTime, endTime).toMinutes();
    }

    /**
     * Convenience: returns true if the ride reached the COMPLETED terminal state.
     */
    public boolean isCompleted() {
        return this.state == TripState.COMPLETED;
    }

    @Override
    public String toString() {
        return "Ride{id=" + rideId
             + ", state=" + state
             + ", driver=" + assignedDriver.getDriverId()
             + ", riderId=" + request.getRiderId()
             + ", fare=" + fareAmount + "}";
    }
}
```

---

#### RideMatcher.java

```java
package rideshar.service;

import rideshar.domain.Driver;
import rideshar.domain.RideRequest;

import java.util.List;
import java.util.Optional;

/**
 * Strategy interface for matching a RideRequest to an available Driver.
 *
 * Implementations are free to apply any algorithm — nearest driver, surge
 * priority, rating-weighted, ML-based — without touching the service layer.
 */
public interface RideMatcher {

    /**
     * Attempts to find the best available driver for the given request.
     *
     * @param request          the ride request containing pickup location
     *                         and vehicle type preference
     * @param availableDrivers the current list of drivers with AVAILABLE status
     * @return the selected Driver, or empty if no suitable driver was found
     */
    Optional<Driver> match(RideRequest request, List<Driver> availableDrivers);
}
```

---

#### NearestDriverMatcher.java

```java
package rideshar.service;

import rideshar.domain.Driver;
import rideshar.domain.Location;
import rideshar.domain.RideRequest;
import rideshar.domain.VehicleType;

import java.util.Comparator;
import java.util.List;
import java.util.Optional;
import java.util.stream.Stream;

/**
 * Matches a ride request to the geographically nearest available driver.
 *
 * If the request specifies a vehicle type preference, only drivers with that
 * vehicle type are considered.  If no drivers of the preferred type are
 * available, the match returns empty (callers should handle this and potentially
 * fall back to any vehicle type if desired).
 */
public class NearestDriverMatcher implements RideMatcher {

    @Override
    public Optional<Driver> match(RideRequest request,
                                  List<Driver> availableDrivers) {
        if (availableDrivers == null || availableDrivers.isEmpty()) {
            return Optional.empty();
        }

        Location pickup = request.getPickupLocation();
        VehicleType preference = request.getVehicleTypePreference();

        Stream<Driver> candidates = availableDrivers.stream()
            .filter(Driver::isAvailable);

        // Apply vehicle type filter only when a preference was specified.
        if (preference != null) {
            candidates = candidates.filter(
                d -> d.getVehicle().getVehicleType() == preference);
        }

        // Find the driver whose current location is closest to the pickup.
        // Using min() with a Comparator is O(n) — preferred over sorting O(n log n).
        return candidates.min(
            Comparator.comparingDouble(
                d -> d.getCurrentLocation().distanceTo(pickup)));
    }
}
```

---

#### SurgeAwareMatcher.java

```java
package rideshar.service;

import rideshar.domain.Driver;
import rideshar.domain.Location;
import rideshar.domain.RideRequest;
import rideshar.domain.VehicleType;

import java.util.Comparator;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

/**
 * A surge-aware matching strategy that prefers PREMIUM vehicles when the
 * surge multiplier exceeds a threshold.  The assumption is that during surge
 * conditions, premium drivers generate higher revenue per trip, making it
 * worthwhile to route them first even if they are slightly farther away.
 *
 * Falls back to the NearestDriverMatcher logic if no PREMIUM drivers are
 * available or if the surge multiplier does not justify premium-first routing.
 */
public class SurgeAwareMatcher implements RideMatcher {

    /**
     * Surge multiplier at or above which PREMIUM drivers are preferred.
     * For example, 1.5 means 50% surge.
     */
    private static final double PREMIUM_PRIORITY_THRESHOLD = 1.5;

    private final double surgeMultiplier;
    private final RideMatcher fallbackMatcher;

    public SurgeAwareMatcher(double surgeMultiplier) {
        this(surgeMultiplier, new NearestDriverMatcher());
    }

    public SurgeAwareMatcher(double surgeMultiplier, RideMatcher fallbackMatcher) {
        if (surgeMultiplier < 1.0) {
            throw new IllegalArgumentException(
                "surgeMultiplier must be >= 1.0, got: " + surgeMultiplier);
        }
        this.surgeMultiplier  = surgeMultiplier;
        this.fallbackMatcher  = fallbackMatcher;
    }

    @Override
    public Optional<Driver> match(RideRequest request,
                                  List<Driver> availableDrivers) {
        if (availableDrivers == null || availableDrivers.isEmpty()) {
            return Optional.empty();
        }

        // During high surge, try to route to PREMIUM drivers first.
        if (surgeMultiplier >= PREMIUM_PRIORITY_THRESHOLD) {
            List<Driver> premiumDrivers = availableDrivers.stream()
                .filter(Driver::isAvailable)
                .filter(d -> d.getVehicle().getVehicleType() == VehicleType.PREMIUM)
                .collect(Collectors.toList());

            if (!premiumDrivers.isEmpty()) {
                // Among premium drivers, still prefer the nearest one.
                Location pickup = request.getPickupLocation();
                return premiumDrivers.stream().min(
                    Comparator.comparingDouble(
                        d -> d.getCurrentLocation().distanceTo(pickup)));
            }
        }

        // No premium drivers available or surge is below threshold —
        // fall back to the standard matching algorithm.
        return fallbackMatcher.match(request, availableDrivers);
    }

    public double getSurgeMultiplier() {
        return surgeMultiplier;
    }
}
```

---

#### FareStrategy.java

```java
package rideshar.service;

import rideshar.domain.Money;
import rideshar.domain.Ride;

/**
 * Strategy interface for fare calculation.
 *
 * Implementations may use any combination of distance, duration, vehicle type,
 * time of day, or surge multipliers.  The RideService receives a FareStrategy
 * at construction time and delegates all fare math to it.
 */
public interface FareStrategy {

    /**
     * Calculates the fare for a completed ride.
     *
     * @param ride the completed ride — must have a non-null endTime
     * @return the calculated fare as a Money value object
     */
    Money calculate(Ride ride);
}
```

---

#### StandardFareStrategy.java

```java
package rideshar.service;

import rideshar.domain.Location;
import rideshar.domain.Money;
import rideshar.domain.Ride;

/**
 * Standard fare calculation based on a flat base fare plus per-minute and
 * per-kilometre charges.
 *
 * Formula:
 *   fare = BASE_FARE
 *        + (trip_duration_minutes * PER_MINUTE_RATE)
 *        + (straight_line_distance_km  * PER_KM_RATE)
 *
 * In production, the straight-line distance would be replaced by the actual
 * route distance from a mapping service.  For LLD purposes, we use the
 * Haversine distance between pickup and dropoff.
 */
public class StandardFareStrategy implements FareStrategy {

    static final double BASE_FARE       = 2.00;
    static final double PER_MINUTE_RATE = 0.30;
    static final double PER_KM_RATE     = 1.20;

    private static final String CURRENCY = "USD";

    @Override
    public Money calculate(Ride ride) {
        long durationMinutes = ride.getDurationMinutes();

        Location pickup  = ride.getRequest().getPickupLocation();
        Location dropoff = ride.getRequest().getDropoffLocation();
        double distanceKm = pickup.distanceTo(dropoff);

        double fare = BASE_FARE
                    + (durationMinutes * PER_MINUTE_RATE)
                    + (distanceKm      * PER_KM_RATE);

        // Enforce a minimum fare — short trips still cost at least BASE_FARE.
        fare = Math.max(fare, BASE_FARE);

        return Money.of(fare, CURRENCY);
    }
}
```

---

#### SurgeFareStrategy.java

```java
package rideshar.service;

import rideshar.domain.Money;
import rideshar.domain.Ride;

/**
 * Decorator that multiplies a base fare strategy's result by a surge
 * multiplier.
 *
 * This uses the decorator pattern: SurgeFareStrategy composes any FareStrategy
 * (the delegate) and applies the multiplier to the result.  Neither this class
 * nor the delegate needs to be modified to change the surge amount — the
 * multiplier is injected at construction time.
 *
 * Usage:
 *   FareStrategy surged = new SurgeFareStrategy(1.8, new StandardFareStrategy());
 */
public class SurgeFareStrategy implements FareStrategy {

    private final double surgeMultiplier;
    private final FareStrategy delegate;

    public SurgeFareStrategy(double surgeMultiplier, FareStrategy delegate) {
        if (surgeMultiplier < 1.0) {
            throw new IllegalArgumentException(
                "surgeMultiplier must be >= 1.0, got: " + surgeMultiplier);
        }
        if (delegate == null) {
            throw new IllegalArgumentException("delegate FareStrategy must not be null");
        }
        this.surgeMultiplier = surgeMultiplier;
        this.delegate        = delegate;
    }

    @Override
    public Money calculate(Ride ride) {
        Money baseFare = delegate.calculate(ride);
        return baseFare.multiply(surgeMultiplier);
    }

    public double getSurgeMultiplier() {
        return surgeMultiplier;
    }
}
```

---

#### RideEvent.java

```java
package rideshar.event;

import java.time.Instant;
import java.util.UUID;

/**
 * Base class for all ride lifecycle events.  Each concrete event type
 * corresponds to a meaningful state change or action in the ride lifecycle.
 *
 * Events are immutable after creation and carry enough context for downstream
 * consumers (notification, analytics, fraud) to act independently.
 */
public abstract class RideEvent {

    private final String eventId;
    private final Instant occurredAt;
    private final String rideId;

    protected RideEvent(String rideId) {
        this.eventId    = UUID.randomUUID().toString();
        this.occurredAt = Instant.now();
        this.rideId     = rideId;
    }

    public String getEventId()    { return eventId; }
    public Instant getOccurredAt() { return occurredAt; }
    public String getRideId()     { return rideId; }

    /**
     * Returns a string tag identifying the event type.
     * Used for routing in event buses and logging.
     */
    public abstract String getEventType();

    @Override
    public String toString() {
        return getEventType() + "{eventId=" + eventId
             + ", rideId=" + rideId
             + ", at=" + occurredAt + "}";
    }
}
```

---

#### RideRequestedEvent.java

```java
package rideshar.event;

/**
 * Published when a new RideRequest enters the system and is ready for matching.
 */
public class RideRequestedEvent extends RideEvent {

    private final String riderId;

    public RideRequestedEvent(String rideId, String riderId) {
        super(rideId);
        this.riderId = riderId;
    }

    public String getRiderId() { return riderId; }

    @Override
    public String getEventType() { return "RIDE_REQUESTED"; }
}
```

---

#### DriverAssignedEvent.java

```java
package rideshar.event;

/**
 * Published when a driver has been successfully matched and assigned to a ride.
 */
public class DriverAssignedEvent extends RideEvent {

    private final String driverId;

    public DriverAssignedEvent(String rideId, String driverId) {
        super(rideId);
        this.driverId = driverId;
    }

    public String getDriverId() { return driverId; }

    @Override
    public String getEventType() { return "DRIVER_ASSIGNED"; }
}
```

---

#### RideCompletedEvent.java

```java
package rideshar.event;

import rideshar.domain.Money;

/**
 * Published when a ride has been completed and fare has been charged
 * successfully.
 */
public class RideCompletedEvent extends RideEvent {

    private final Money fareAmount;

    public RideCompletedEvent(String rideId, Money fareAmount) {
        super(rideId);
        this.fareAmount = fareAmount;
    }

    public Money getFareAmount() { return fareAmount; }

    @Override
    public String getEventType() { return "RIDE_COMPLETED"; }
}
```

---

#### RideCancelledEvent.java

```java
package rideshar.event;

/**
 * Published when a ride is cancelled for any reason, including payment failure.
 *
 * cancelledBy may be "RIDER", "DRIVER", or "SYSTEM" (for automated cancellation
 * due to no driver found or payment failure).
 */
public class RideCancelledEvent extends RideEvent {

    private final String cancelledBy;
    private final String reason;

    public RideCancelledEvent(String rideId,
                               String cancelledBy,
                               String reason) {
        super(rideId);
        this.cancelledBy = cancelledBy;
        this.reason      = reason;
    }

    public String getCancelledBy() { return cancelledBy; }
    public String getReason()      { return reason; }

    @Override
    public String getEventType() { return "RIDE_CANCELLED"; }
}
```

---

#### RideEventListener.java

```java
package rideshar.event;

/**
 * Observer interface for ride lifecycle events.
 *
 * Implementations receive all events published by RideEventPublisher.
 * Each implementation is responsible for filtering by event type if needed.
 */
public interface RideEventListener {

    /**
     * Called by the publisher for every event.  Implementations must not
     * throw unchecked exceptions — the publisher catches and logs them
     * to protect other listeners in the chain.
     *
     * @param event the ride event — never null
     */
    void onEvent(RideEvent event);
}
```

---

#### RideEventPublisher.java

```java
package rideshar.event;

import java.util.concurrent.CopyOnWriteArrayList;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * Fan-out publisher for ride lifecycle events.
 *
 * Thread safety: CopyOnWriteArrayList is used so that listeners can be
 * registered or deregistered concurrently without blocking or corrupting
 * the iteration during publish.  This is the correct choice here because
 * publish is called much more frequently than register/deregister, making
 * the copy-on-write trade-off (expensive writes, cheap reads) appropriate.
 *
 * Error isolation: if a listener throws during onEvent, the exception is
 * caught and logged, but publishing continues to all remaining listeners.
 * One misbehaving listener cannot break the delivery chain.
 */
public class RideEventPublisher {

    private static final Logger LOGGER =
        Logger.getLogger(RideEventPublisher.class.getName());

    private final CopyOnWriteArrayList<RideEventListener> listeners =
        new CopyOnWriteArrayList<>();

    /**
     * Adds a listener to receive future events.  Adding a listener that is
     * already registered results in it receiving duplicate events; callers
     * are responsible for avoiding duplicate registration.
     */
    public void register(RideEventListener listener) {
        if (listener == null) {
            throw new IllegalArgumentException("listener must not be null");
        }
        listeners.add(listener);
    }

    /**
     * Removes a listener.  If the listener is not currently registered,
     * this is a no-op.
     */
    public void deregister(RideEventListener listener) {
        listeners.remove(listener);
    }

    /**
     * Publishes the event to all registered listeners.  Guaranteed to
     * attempt delivery to every listener regardless of whether any earlier
     * listener threw an exception.
     *
     * @param event the event to publish — must not be null
     */
    public void publish(RideEvent event) {
        if (event == null) {
            throw new IllegalArgumentException("event must not be null");
        }
        for (RideEventListener listener : listeners) {
            try {
                listener.onEvent(event);
            } catch (Exception e) {
                LOGGER.log(Level.WARNING,
                    "Listener " + listener.getClass().getName()
                    + " threw while handling event " + event.getEventType()
                    + " for ride " + event.getRideId(),
                    e);
            }
        }
    }

    public int getListenerCount() {
        return listeners.size();
    }
}
```

---

#### PaymentProcessor.java

```java
package rideshar.service;

import rideshar.domain.Money;
import rideshar.domain.PaymentMethod;

/**
 * Abstraction over external payment processing.
 *
 * The concrete implementation would delegate to a payment gateway (Stripe,
 * Braintree, etc.).  Defined as an interface to keep RideService testable
 * without real payment infrastructure.
 */
public interface PaymentProcessor {

    /**
     * Attempts to charge the given amount using the specified payment method.
     *
     * @param riderId the identifier of the rider being charged
     * @param amount  the fare amount to charge
     * @param method  the payment instrument to use
     * @return a PaymentResult indicating success or failure
     */
    PaymentResult processPayment(String riderId, Money amount, PaymentMethod method);
}
```

---

#### PaymentResult.java

```java
package rideshar.service;

import java.util.Optional;

/**
 * Immutable result of a payment processing attempt.
 *
 * Use the static factory methods rather than the constructor directly:
 *   PaymentResult.success("txn-abc-123")
 *   PaymentResult.failure("Insufficient funds")
 */
public final class PaymentResult {

    private final boolean success;
    private final String transactionId;   // non-null on success
    private final String failureReason;   // non-null on failure

    private PaymentResult(boolean success,
                          String transactionId,
                          String failureReason) {
        this.success       = success;
        this.transactionId = transactionId;
        this.failureReason = failureReason;
    }

    /**
     * Creates a successful payment result with the gateway transaction ID.
     */
    public static PaymentResult success(String transactionId) {
        if (transactionId == null || transactionId.isBlank()) {
            throw new IllegalArgumentException(
                "transactionId must be non-blank for a successful payment");
        }
        return new PaymentResult(true, transactionId, null);
    }

    /**
     * Creates a failed payment result with a human-readable reason.
     */
    public static PaymentResult failure(String reason) {
        if (reason == null || reason.isBlank()) {
            throw new IllegalArgumentException(
                "reason must be non-blank for a failed payment");
        }
        return new PaymentResult(false, null, reason);
    }

    public boolean isSuccess()                        { return success; }
    public Optional<String> getTransactionId()        { return Optional.ofNullable(transactionId); }
    public Optional<String> getFailureReason()        { return Optional.ofNullable(failureReason); }

    @Override
    public String toString() {
        if (success) {
            return "PaymentResult{SUCCESS, txn=" + transactionId + "}";
        }
        return "PaymentResult{FAILURE, reason=" + failureReason + "}";
    }
}
```

---

#### NoDriverAvailableException.java

```java
package rideshar.service;

/**
 * Thrown by RideService when the matcher cannot find a suitable available driver
 * for a given ride request.  Callers should surface this to the rider with an
 * appropriate user-facing message and may retry after a backoff period.
 */
public class NoDriverAvailableException extends RuntimeException {

    private final String requestId;

    public NoDriverAvailableException(String requestId) {
        super("No available driver found for ride request: " + requestId);
        this.requestId = requestId;
    }

    public NoDriverAvailableException(String requestId, String message) {
        super(message);
        this.requestId = requestId;
    }

    public String getRequestId() {
        return requestId;
    }
}
```

---

#### RideService.java

```java
package rideshar.service;

import rideshar.domain.Driver;
import rideshar.domain.DriverStatus;
import rideshar.domain.Money;
import rideshar.domain.Ride;
import rideshar.domain.RideRequest;
import rideshar.domain.TripState;
import rideshar.event.DriverAssignedEvent;
import rideshar.event.RideCancelledEvent;
import rideshar.event.RideCompletedEvent;
import rideshar.event.RideEventPublisher;
import rideshar.event.RideRequestedEvent;

import java.util.List;
import java.util.Objects;
import java.util.Optional;
import java.util.logging.Logger;

/**
 * The primary application service for the ride lifecycle.
 *
 * RideService is deliberately thin: it orchestrates matching, state transitions,
 * fare calculation, payment, and event publishing, but it does not contain
 * any algorithm logic directly.  All algorithmic concerns are delegated to
 * injected strategies.
 *
 * This class is the single place where side effects (driver status mutation,
 * payment calls, event publication) are coordinated — making it the correct
 * seam for transactional logic.
 *
 * Thread safety: this class itself is stateless.  The underlying data structures
 * (driver lists, ride storage) must be managed externally with appropriate
 * concurrency controls.  In production, driver assignment would use optimistic
 * locking or a reservation step to prevent double-booking.
 */
public class RideService {

    private static final Logger LOGGER =
        Logger.getLogger(RideService.class.getName());

    private final RideMatcher matcher;
    private final FareStrategy fareStrategy;
    private final RideEventPublisher eventPublisher;
    private final PaymentProcessor paymentProcessor;

    public RideService(RideMatcher matcher,
                       FareStrategy fareStrategy,
                       RideEventPublisher eventPublisher,
                       PaymentProcessor paymentProcessor) {
        this.matcher          = Objects.requireNonNull(matcher,          "matcher");
        this.fareStrategy     = Objects.requireNonNull(fareStrategy,     "fareStrategy");
        this.eventPublisher   = Objects.requireNonNull(eventPublisher,   "eventPublisher");
        this.paymentProcessor = Objects.requireNonNull(paymentProcessor, "paymentProcessor");
    }

    /**
     * Attempts to match a driver to the ride request and creates a new Ride.
     *
     * Steps:
     * 1. Filter availableDrivers to those with AVAILABLE status.
     * 2. Call the matcher to find the best driver.
     * 3. Create a Ride (initial state: DRIVER_ASSIGNED).
     * 4. Mark the driver as ON_TRIP and record the rideId.
     * 5. Publish DriverAssignedEvent.
     * 6. Return the Ride.
     *
     * @throws NoDriverAvailableException if no suitable driver was found
     */
    public Ride requestRide(RideRequest request, List<Driver> availableDrivers) {
        Objects.requireNonNull(request,          "request");
        Objects.requireNonNull(availableDrivers, "availableDrivers");

        // Publish the initial event so analytics knows a request was received.
        eventPublisher.publish(new RideRequestedEvent(
            request.getRequestId(), request.getRiderId()));

        // Only offer truly available drivers to the matcher.
        List<Driver> eligibleDrivers = availableDrivers.stream()
            .filter(Driver::isAvailable)
            .collect(java.util.stream.Collectors.toList());

        Optional<Driver> matched = matcher.match(request, eligibleDrivers);

        if (matched.isEmpty()) {
            throw new NoDriverAvailableException(request.getRequestId());
        }

        Driver driver = matched.get();
        Ride ride = new Ride(request, driver);

        // Update driver state atomically in production (optimistic lock / DB tx).
        driver.setStatus(DriverStatus.ON_TRIP);
        driver.setActiveRideId(ride.getRideId());

        eventPublisher.publish(
            new DriverAssignedEvent(ride.getRideId(), driver.getDriverId()));

        LOGGER.info("Ride " + ride.getRideId()
            + " created and assigned to driver " + driver.getDriverId());

        return ride;
    }

    /**
     * Marks the driver as having begun the trip — they are now heading to
     * the pickup point and subsequently picking up the rider.
     *
     * Transitions: DRIVER_ASSIGNED -> EN_ROUTE_TO_PICKUP -> RIDE_IN_PROGRESS
     *
     * For simplicity, this single method advances through both intermediate
     * states.  In a real system, each state change would be a separate API
     * call triggered by the driver app.
     */
    public void startRide(Ride ride) {
        Objects.requireNonNull(ride, "ride");
        ride.transitionTo(TripState.EN_ROUTE_TO_PICKUP);
        ride.transitionTo(TripState.RIDE_IN_PROGRESS);
        LOGGER.info("Ride " + ride.getRideId() + " is now in progress.");
    }

    /**
     * Completes the ride: calculates fare, processes payment, and publishes events.
     *
     * Steps:
     * 1. Transition state to COMPLETED.
     * 2. Calculate fare using the injected FareStrategy.
     * 3. Attempt payment via PaymentProcessor.
     * 4. If payment succeeds: set fareAmount, publish RideCompletedEvent, free driver.
     * 5. If payment fails: transition to PAYMENT_FAILED, publish RideCancelledEvent
     *    with reason "PAYMENT_FAILED", free driver.
     *
     * Note: the driver is freed regardless of payment outcome — the driver's
     * responsibility ends when they complete the trip.  Payment recovery is
     * handled asynchronously by a separate reconciliation process.
     */
    public void completeRide(Ride ride) {
        Objects.requireNonNull(ride, "ride");

        ride.transitionTo(TripState.COMPLETED);

        Money fare = fareStrategy.calculate(ride);
        ride.setFareAmount(fare);

        Driver driver = ride.getAssignedDriver();
        String riderId = ride.getRequest().getRiderId();

        // Determine which payment method to use.  In production, RideRequest
        // or a separate PaymentProfile would specify the method for this trip.
        // We fall back to the rider's default here.
        // (This would be retrieved from a RiderRepository in production.)
        PaymentResult paymentResult = paymentProcessor.processPayment(
            riderId, fare, rideshar.domain.PaymentMethod.CREDIT_CARD);

        if (paymentResult.isSuccess()) {
            LOGGER.info("Payment succeeded for ride " + ride.getRideId()
                + ". Transaction: "
                + paymentResult.getTransactionId().orElse("unknown"));

            eventPublisher.publish(
                new RideCompletedEvent(ride.getRideId(), fare));
        } else {
            LOGGER.warning("Payment FAILED for ride " + ride.getRideId()
                + ". Reason: "
                + paymentResult.getFailureReason().orElse("unknown"));

            // Transition to PAYMENT_FAILED — not CANCELLED — because the
            // distinction matters for reconciliation: PAYMENT_FAILED rides
            // were physically completed but not settled.
            ride.transitionTo(TripState.PAYMENT_FAILED);

            eventPublisher.publish(new RideCancelledEvent(
                ride.getRideId(),
                "SYSTEM",
                "Payment failed: " + paymentResult.getFailureReason().orElse("unknown")));
        }

        // Free the driver regardless of payment outcome.
        freeDriver(driver);
    }

    /**
     * Cancels a ride.  Valid when the ride has not yet started (REQUESTED,
     * DRIVER_ASSIGNED) or is en route to pickup.  Once RIDE_IN_PROGRESS,
     * only the completeRide path is valid.
     *
     * @param cancelledBy "RIDER" or "DRIVER"
     * @param reason      human-readable reason for the cancellation
     * @throws IllegalStateException if the ride is not in a cancellable state
     */
    public void cancelRide(Ride ride, String cancelledBy, String reason) {
        Objects.requireNonNull(ride,        "ride");
        Objects.requireNonNull(cancelledBy, "cancelledBy");
        Objects.requireNonNull(reason,      "reason");

        // transitionTo validates legality; will throw if ride is in RIDE_IN_PROGRESS
        // or any terminal state.
        ride.transitionTo(TripState.CANCELLED);

        // Free the driver to take new requests.
        freeDriver(ride.getAssignedDriver());

        eventPublisher.publish(new RideCancelledEvent(
            ride.getRideId(), cancelledBy, reason));

        LOGGER.info("Ride " + ride.getRideId()
            + " cancelled by " + cancelledBy + ". Reason: " + reason);
    }

    // -------------------------------------------------------------------------
    // Private helpers
    // -------------------------------------------------------------------------

    private void freeDriver(Driver driver) {
        driver.setStatus(DriverStatus.AVAILABLE);
        driver.setActiveRideId(null);
    }
}
```

---

#### Example Usage (Main-Style Demonstration)

```java
package rideshar;

import rideshar.domain.*;
import rideshar.event.*;
import rideshar.service.*;

import java.util.Arrays;
import java.util.List;

/**
 * End-to-end demonstration of the ride sharing domain model.
 * Wire up all components and simulate a complete ride lifecycle.
 */
public class RideSharingDemo {

    public static void main(String[] args) {

        // ----------------------------------------------------------------
        // 1. Create domain entities
        // ----------------------------------------------------------------

        Vehicle standardCar = new Vehicle("v-001", "KA-01-AB-1234", "Honda City",
                                           VehicleType.STANDARD);
        Vehicle premiumCar  = new Vehicle("v-002", "KA-01-CD-5678", "BMW 5 Series",
                                           VehicleType.PREMIUM);

        // Driver 1 — close to downtown Bangalore
        Driver driver1 = new Driver(
            "d-001", "Ramesh Kumar", "ramesh@example.com",
            standardCar,
            new Location(12.9716, 77.5946)   // near MG Road
        );

        // Driver 2 — slightly farther, in a premium car
        Driver driver2 = new Driver(
            "d-002", "Suresh Nair", "suresh@example.com",
            premiumCar,
            new Location(12.9780, 77.6080)   // near Indiranagar
        );

        Rider rider = new Rider(
            "r-001", "Priya Sharma", "priya@example.com",
            PaymentMethod.CREDIT_CARD
        );

        List<Driver> availableDrivers = Arrays.asList(driver1, driver2);

        // ----------------------------------------------------------------
        // 2. Build a RideRequest
        // ----------------------------------------------------------------

        Location pickup  = new Location(12.9720, 77.5950);  // Central Bangalore
        Location dropoff = new Location(12.9352, 77.6245);  // Koramangala

        RideRequest request = RideRequest
            .builder(rider.getRiderId(), pickup, dropoff)
            .vehicleTypePreference(VehicleType.STANDARD)
            .build();

        System.out.println("Ride request created: " + request.getRequestId());
        System.out.println("Distance pickup->dropoff: "
            + String.format("%.2f km", pickup.distanceTo(dropoff)));

        // ----------------------------------------------------------------
        // 3. Wire up services
        // ----------------------------------------------------------------

        RideMatcher    matcher        = new NearestDriverMatcher();
        FareStrategy   fareStrategy   = new StandardFareStrategy();
        RideEventPublisher publisher  = new RideEventPublisher();

        // Register a console listener to print all events
        publisher.register(event ->
            System.out.println("[EVENT] " + event.getEventType()
                + " | rideId=" + event.getRideId()
                + " | at=" + event.getOccurredAt()));

        // Simulate a payment processor — always succeeds in this demo
        PaymentProcessor paymentProcessor =
            (riderId, amount, method) -> {
                System.out.println("[PAYMENT] Charging " + amount
                    + " to rider " + riderId
                    + " via " + method);
                return PaymentResult.success("txn-" + System.currentTimeMillis());
            };

        RideService rideService = new RideService(
            matcher, fareStrategy, publisher, paymentProcessor);

        // ----------------------------------------------------------------
        // 4. Full ride lifecycle
        // ----------------------------------------------------------------

        System.out.println("\n--- Requesting ride ---");
        Ride ride = rideService.requestRide(request, availableDrivers);

        System.out.println("Ride created: " + ride.getRideId());
        System.out.println("Assigned driver: " + ride.getAssignedDriver().getName()
            + " (vehicle: " + ride.getAssignedDriver().getVehicle().getVehicleType() + ")");
        System.out.println("Driver status: " + ride.getAssignedDriver().getStatus());
        System.out.println("Ride state: " + ride.getState());

        System.out.println("\n--- Starting ride ---");
        rideService.startRide(ride);
        System.out.println("Ride state: " + ride.getState());

        // Simulate some trip duration by artificially setting the start time.
        // In production, start/end times are set by transitionTo internally.

        System.out.println("\n--- Completing ride ---");
        rideService.completeRide(ride);
        System.out.println("Ride state: " + ride.getState());
        System.out.println("Fare: " + ride.getFareAmount());
        System.out.println("Driver status after completion: "
            + ride.getAssignedDriver().getStatus());

        // ----------------------------------------------------------------
        // 5. Demonstrate cancellation (separate ride)
        // ----------------------------------------------------------------

        System.out.println("\n--- Demonstrating cancellation ---");

        // Reset driver1 to AVAILABLE for the new request
        driver1.setStatus(DriverStatus.AVAILABLE);
        driver1.setActiveRideId(null);

        RideRequest request2 = RideRequest
            .builder(rider.getRiderId(),
                     new Location(12.9720, 77.5950),
                     new Location(13.0000, 77.5800))
            .build();

        Ride ride2 = rideService.requestRide(request2, availableDrivers);
        System.out.println("Ride 2 state: " + ride2.getState());

        rideService.cancelRide(ride2, "RIDER", "Change of plans");
        System.out.println("Ride 2 state after cancellation: " + ride2.getState());
        System.out.println("Driver status: " + ride2.getAssignedDriver().getStatus());

        // ----------------------------------------------------------------
        // 6. Demonstrate surge pricing
        // ----------------------------------------------------------------

        System.out.println("\n--- Demonstrating surge pricing ---");

        // Reset drivers
        driver1.setStatus(DriverStatus.AVAILABLE);
        driver1.setActiveRideId(null);
        driver2.setStatus(DriverStatus.AVAILABLE);
        driver2.setActiveRideId(null);

        double surgeMultiplier = 2.0;
        FareStrategy surgeFare = new SurgeFareStrategy(surgeMultiplier,
                                                        new StandardFareStrategy());
        RideMatcher surgeMatcher = new SurgeAwareMatcher(surgeMultiplier);

        RideService surgeService = new RideService(
            surgeMatcher, surgeFare, publisher, paymentProcessor);

        RideRequest surgeRequest = RideRequest
            .builder(rider.getRiderId(), pickup, dropoff)
            .build();

        Ride surgeRide = surgeService.requestRide(surgeRequest, availableDrivers);
        System.out.println("Surge ride assigned to: "
            + surgeRide.getAssignedDriver().getName()
            + " (type: " + surgeRide.getAssignedDriver().getVehicle().getVehicleType() + ")");
        System.out.println("  -> SurgeAwareMatcher should prefer PREMIUM driver during 2x surge");

        surgeService.startRide(surgeRide);
        surgeService.completeRide(surgeRide);
        System.out.println("Surge fare: " + surgeRide.getFareAmount()
            + " (2x multiplier applied)");
    }
}
```

---

## 5. Scoring Guide

This guide is for self-evaluation after completing the mock interview. Each dimension is scored 1–5. Use the descriptions below to calibrate where you landed on each dimension for this specific problem.

---

### Dimension 1: Requirements Clarification

**What to look for in a ride-sharing problem specifically**:
This problem has unusually high requirements sensitivity — the answer to "do drivers accept or are they auto-assigned?" directly adds or removes the `PENDING_ACCEPTANCE` state. The answer to "can a driver cancel?" determines whether `DRIVER_CANCELLED` is a distinct state. Missing these questions means the state machine will be wrong before a single line of code is written.

**Score 3 on this problem**:
Asked whether drivers auto-accept or manually accept. Asked about surge pricing. Did not proactively ask about scheduled rides or payment processing scope. Stated in/out of scope after being prompted.

**Score 5 on this problem**:
Proactively asked all of: driver acceptance mode (and connected it to PENDING_ACCEPTANCE state), cancellation from both sides, surge pricing, scheduled rides, and payment scope. Explicitly declared in/out of scope before proceeding. Identified three actors (Rider, Driver, System) and noted the System is its own actor in auto-matching. No interviewer prompt required for any of these.

**Most common place candidates lose points**:
Not asking about cancellation from both sides — this is a requirement that directly shapes which states are terminal vs reversible, and most candidates simply forget it.

---

### Dimension 2: Abstraction Quality

**What to look for in a ride-sharing problem specifically**:
The key abstraction test is whether the candidate separates `RideRequest` (intent, before matching) from `Ride` (engagement, after matching). A second test is whether `Location` is modeled as a value object with behavior (`distanceTo`) rather than a pair of doubles. A third test is whether `SchedulingPolicy` is extracted to handle the scheduled-ride variant.

**Score 3 on this problem**:
Separated `RideRequest` from `Ride` with some explanation. Modeled `Location` as a class. Did not extract `SchedulingPolicy` — would have added a boolean `isScheduled` flag instead.

**Score 5 on this problem**:
Identified `RideRequest` vs `Ride` separation before being asked, explained it in terms of lifecycle, not just fields. Modeled `Location` as an immutable value object and named `distanceTo` as the justification. Introduced `SchedulingPolicy` as a policy interface on `RideRequest`, showing the scheduled-ride extension costs zero changes to `Ride` or `RideService`.

**Most common place candidates lose points**:
Conflating `RideRequest` and `Ride` into a single `Ride` entity with a nullable `assignedDriver` — the most common structural mistake in this problem.

---

### Dimension 3: Class Design and Relationships

**What to look for in a ride-sharing problem specifically**:
Three relationship tests matter here: (1) `SurgeFareStrategy` decorates `FareStrategy` — is it composition or inheritance? (2) `SurgeAwareMatcher` delegates to `NearestDriverMatcher` — is it composition or reimplementation? (3) `Ride.transitionTo` enforces valid transitions using the enum — is transition logic in the entity or scattered in the service?

**Score 3 on this problem**:
Defined `FareStrategy` and `RideMatcher` as interfaces. Implemented `SurgeFareStrategy` correctly. Did not implement `canTransitionTo` on the enum — left transition validation in `RideService`.

**Score 5 on this problem**:
Defined all interfaces before concrete classes. Implemented `SurgeFareStrategy` as a decorator composing any `FareStrategy`. Implemented `canTransitionTo` on `TripState` enum, making `Ride.transitionTo` clean. Implemented `SurgeAwareMatcher` using a `fallbackMatcher` field rather than reimplementing nearest-driver logic.

**Most common place candidates lose points**:
Putting transition validation logic in `RideService` rather than encoding it on the `TripState` enum — leaving the entity unprotected.

---

### Dimension 4: Pattern Selection Appropriateness

**What to look for in a ride-sharing problem specifically**:
Three patterns are expected: Strategy (fare and matching), Observer (events), and State Machine (trip lifecycle). A candidate who reaches for inheritance instead of Strategy, or direct method calls instead of Observer, shows pattern recognition at a mid-level rather than senior level.

**Score 3 on this problem**:
Used Strategy for fare calculation. Used direct method calls for notification ("I'll call notificationService.sendSms inside completeRide"). Used a state field on Ride.

**Score 5 on this problem**:
Used Strategy for both fare and matching, explicitly noting the open/closed benefit. Used Observer (RideEventPublisher) for all side effects, explaining that this decouples RideService from every consumer. Designed `SurgeFareStrategy` as a decorator (a specific Strategy variant). Encoded the State Machine on the enum itself rather than in the service.

**Most common place candidates lose points**:
Calling notification or analytics services directly in `RideService.completeRide` — tight coupling that makes the service impossible to test and extend without modification.

---

### Dimension 5: Code Quality

**What to look for in a ride-sharing problem specifically**:
Three concrete quality markers: (1) `Money` uses `BigDecimal`, not `double`. (2) `Location.distanceTo` uses Haversine, not Euclidean on degrees. (3) `RideEventPublisher` uses `CopyOnWriteArrayList` and catches exceptions per listener.

**Score 3 on this problem**:
Used `BigDecimal` for `Money`. Did not implement Haversine — used a placeholder or Euclidean approximation. Did not address thread safety in `RideEventPublisher`.

**Score 5 on this problem**:
`BigDecimal` for money with correct `HALF_UP` rounding. Actual Haversine formula with Earth radius constant. `CopyOnWriteArrayList` for thread safety with a clear explanation of the read-heavy trade-off. Per-listener exception catching in publish. Domain-specific exception (`NoDriverAvailableException`) with a `requestId` field. Builder pattern on `RideRequest`.

**Most common place candidates lose points**:
Using `double` for currency — a hard signal that the candidate has not worked with financial systems and lacks awareness of floating-point rounding errors.

---

### Dimension 6: Extensibility Demonstration

**What to look for in a ride-sharing problem specifically**:
The extension questions in Stage 5 are specifically designed to test whether the candidate's upfront design was genuinely extensible. If `SchedulingPolicy` was introduced, scheduled rides cost zero changes. If `FareStrategy` is an interface, adding a new pricing model is a new class. If `RideEventPublisher` is used, adding a new consumer is a new `RideEventListener` implementation.

**Score 3 on this problem**:
Could add a new FareStrategy without modifying existing code. Could not add scheduled rides without touching `Ride` or `RideService` — needed to add a `scheduledTime` field.

**Score 5 on this problem**:
Demonstrated that scheduled rides require zero changes to `Ride` or `RideService` because `SchedulingPolicy` handles it. Demonstrated that new fare strategies are new classes (SurgeFareStrategy as decorator). Demonstrated that new event consumers are new `RideEventListener` implementations. Proactively noted that in production `RideEventPublisher` would delegate to Kafka/SQS, acknowledging the LLD/production boundary.

**Most common place candidates lose points**:
Not introducing `SchedulingPolicy` early, then struggling to retrofit it in Stage 5 without modifying `Ride`.

---

### Dimension 7: Edge Case Handling

**What to look for in a ride-sharing problem specifically**:
Five edge cases are expected: (1) No driver available — throws `NoDriverAvailableException`. (2) Payment failure — transitions to `PAYMENT_FAILED`, not `CANCELLED`. (3) Double-booking — two riders match to the same driver concurrently (optimistic lock at DB layer). (4) Driver cancels mid-route — valid cancellation from `EN_ROUTE_TO_PICKUP` state. (5) State machine violated by caller — `Ride.transitionTo` throws `IllegalStateException`.

**Score 3 on this problem**:
Handled no-driver-available with an exception. Handled payment failure with a state transition. Did not address concurrency/double-booking.

**Score 5 on this problem**:
All five edge cases addressed. Double-booking discussed with an explicit acknowledgment that the in-memory model needs optimistic locking or a driver-reservation step at the persistence layer. Noted that `PAYMENT_FAILED` and `CANCELLED` are distinct states for reconciliation reasons. Demonstrated that the state machine is enforced by `Ride.transitionTo`, not just documented.

**Most common place candidates lose points**:
Conflating `PAYMENT_FAILED` and `CANCELLED` — using only CANCELLED for both, which makes downstream reconciliation impossible.

---

### Dimension 8: Communication Clarity

**What to look for in a ride-sharing problem specifically**:
This problem has three "explain before code" moments where the strongest candidates narrate their design thinking: (1) before naming entities, explain the `RideRequest` vs `Ride` distinction; (2) before writing `FareStrategy`, explain why it's a Strategy rather than a method; (3) before writing `RideEventPublisher`, explain why events decouple the service. Candidates who just write code without narration are harder to evaluate and often get marked down.

**Score 3 on this problem**:
Named entities clearly. Explained interfaces when asked. Did not proactively frame the "why" behind design choices — code appeared without a design narrative.

**Score 5 on this problem**:
Before naming entities: "I want to distinguish RideRequest from Ride — let me explain why." Before introducing `FareStrategy`: "I'm pulling this out as a Strategy because surge pricing is a variant, and I want to add it without touching existing code." Before `RideEventPublisher`: "Rather than calling notification directly, I'll use an observer pattern — here's why that matters." Spoke continuously while writing code. Drew state diagram before any implementation.

**Most common place candidates lose points**:
Jumping straight to code without stating the design rationale — the interviewer cannot tell whether the correct class structure was intentional or accidental.

---

## 6. Debrief Notes

### What Makes This Problem Hard

**The lifecycle separation is non-obvious under time pressure.** The instinct of most candidates — trained on simpler LLD problems — is to create one `Ride` class and add a nullable `assignedDriver` field. The insight that a `RideRequest` represents intent and a `Ride` represents engagement is a design principle, not a pattern, and it requires domain modeling maturity to see quickly.

**The state machine has seven states and non-trivial valid transitions.** Many LLD problems have three or four states with linear transitions. This problem has branching paths — cancellation is legal from three states, payment failure is a distinct terminal state from CANCELLED, and RIDE_IN_PROGRESS cannot be directly cancelled. A candidate who sketches the state diagram before writing code will catch all of this. A candidate who skips the diagram will leave invalid transitions open.

**Matching is extensible in a non-obvious direction.** The first instinct is "find the nearest driver" — a straightforward spatial query. The extensibility insight is that "how you find the driver" should be abstracted behind `RideMatcher`, which opens the door to ML-based matching, pool matching, priority matching, and surge-aware routing. This requires the candidate to see the variability before it's asked about.

**Observer-based event handling is not obvious to candidates without distributed systems exposure.** Candidates who have only worked on monolithic systems tend to call notification and analytics services directly from the service method. The Observer pattern here is not just about testability — it's about the architectural principle that `RideService` should not know who cares about ride events.

**The concurrency issue lurks beneath the surface.** Two riders could both match to driver1 simultaneously. In-memory, the code appears safe because list operations happen sequentially. In production, concurrent threads would create a race condition. A senior candidate notes this and proposes optimistic locking or a driver-reservation step — showing awareness that LLD is not production architecture.

---

### Common Traps

**Trap 1: Creating a single Ride class that holds both booking and trip lifecycle.**
When this happens, `Ride` has a nullable `assignedDriver`, a nullable `requestedAt`, and boolean flags like `isAssigned`. As states are added, the class grows with mutually exclusive fields. The fix — separating `RideRequest` from `Ride` — must be proposed at entity identification time or retrofitting it is painful.

**Trap 2: Hardcoding fare calculation inside Ride or RideService.**
The pattern is: "I'll add a `calculateFare()` method to `RideService`" or worse, to `Ride`. This immediately makes surge pricing impossible without modifying the class. The correct answer is to extract `FareStrategy` as the first thing, before any concrete fare logic is written.

**Trap 3: Calling notification or analytics services directly inside completeRide.**
This couples `RideService` to every downstream consumer. It makes the service impossible to test without mocking 4–5 collaborators. It means adding a new consumer requires modifying `RideService`. The correct answer — `RideEventPublisher` with registered listeners — is not complex, but it requires the candidate to see the problem first.

**Trap 4: Not modeling DriverStatus as a separate enum.**
Some candidates store driver status as a boolean `isAvailable` or a String. A boolean breaks when OFFLINE is needed (a driver who is neither AVAILABLE nor ON_TRIP). A String loses type safety. `DriverStatus` as an enum with three values is the correct model, and it takes 30 seconds to introduce.

**Trap 5: Modeling Location as a mutable class with setters.**
A `Location` with `setLatitude` and `setLongitude` can be modified after being stored in a `RideRequest`. This breaks the immutability contract for value objects and introduces thread safety issues. `Location` must be final and immutable.

---

### How to Stand Out

**Introduce RideRequest as a separate entity from Ride during entity identification — without being prompted.** This alone signals that the candidate thinks at the domain layer rather than the database layer.

**Propose the state machine diagram before writing any code.** Draw the seven states and the valid transitions on a whiteboard. This prevents illegal transition bugs and shows the interviewer that the candidate treats state as a design concern, not an implementation detail.

**Introduce FareStrategy as an interface before being asked about surge pricing.** Say: "I'll model fare calculation as a Strategy because I can already see that surge pricing would be a variant. Let me define the interface first." This demonstrates proactive extensibility thinking.

**Note that RideEventPublisher should use CopyOnWriteArrayList and explain why.** Most candidates who introduce the observer pattern do not think about thread safety for listener registration. Naming `CopyOnWriteArrayList` and explaining the read-heavy trade-off is a concrete signal of production-systems experience.

**When asked about scheduled rides, show that SchedulingPolicy already handles it.** The payoff for introducing `SchedulingPolicy` on `RideRequest` is this moment: "Scheduled rides require zero changes to `Ride` or `RideService`. I introduced `SchedulingPolicy` as a policy interface on `RideRequest` precisely to handle this. The scheduler service just polls for requests whose scheduled time has arrived and calls `requestRide`."

**Mention that in production, RideEventPublisher would publish to a message broker.** Say: "In an LLD exercise we use in-process listeners. In production, I'd replace the `publish` method with a Kafka producer call and run each consumer as a separate microservice. This is the boundary between LLD and production architecture — the Observer pattern here is the in-process stand-in for a real event bus." This single statement demonstrates distributed systems awareness and the ability to connect LLD to real architecture.
