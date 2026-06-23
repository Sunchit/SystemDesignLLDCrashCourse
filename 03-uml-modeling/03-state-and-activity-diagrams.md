# State Diagrams and Activity Diagrams

> State diagrams model **what an object goes through over its lifetime**. Activity diagrams model **how a process flows from start to finish**.

---

## Table of Contents

**State Diagrams**
1. [Core State Diagram Notation](#1-core-state-diagram-notation)
2. [Example: Order Lifecycle](#2-example-order-lifecycle)
3. [Example: ATM State Machine](#3-example-atm-state-machine)
4. [Example: Traffic Light](#4-example-traffic-light)
5. [When to Use State Diagrams](#5-when-to-use-state-diagrams)
6. [Connection to the State Pattern](#6-connection-to-the-state-pattern)

**Activity Diagrams**
7. [Core Activity Diagram Notation](#7-core-activity-diagram-notation)
8. [Example: Order Fulfillment Process](#8-example-order-fulfillment-process)
9. [Example: User Registration](#9-example-user-registration)
10. [Swimlanes](#10-swimlanes)

---

# State Diagrams

## 1. Core State Diagram Notation

A state diagram (also called a state machine diagram) shows the discrete states an object can be in, the events that trigger transitions between states, the conditions (guards) under which transitions occur, and the actions taken during transitions.

### States

A state is a condition in which an object waits for an event. Represented as a rounded rectangle:

```
+-------------------+
|    StateName      |
+-------------------+
| entry / action    |   <- entry action: runs when state is entered
| do / activity     |   <- do activity: runs while in this state
| exit / action     |   <- exit action: runs when state is exited
+-------------------+
```

For simple diagrams, just the state name is sufficient.

### Initial and Final States

```
(*)    <- filled circle: initial state (pseudostate, not a real state)
(*)    <- filled circle with ring: final state
```

### Transitions

An arrow from one state to another, labeled with:

```
event [guard] / action
```

- **event**: what triggers the transition (e.g., `submit`, `paymentReceived`, `timeout`)
- **guard**: optional condition in square brackets that must be true for the transition to fire
- **action**: optional activity that executes when the transition fires

```
[StateA] ---event [guard] / action---> [StateB]
```

### Full Notation Reference

```
            event / action
[State A] ─────────────────> [State B]

            event [condition] / action
[State A] ─────────────────────────────> [State B]

            / entry action only
[State A] ──────────────────────────────> [State B]
```

---

## 2. Example: Order Lifecycle

This is the most universally relevant state machine for backend engineers. Understanding order states deeply is a prerequisite for any e-commerce LLD question.

```
        (*)
         |
         | create [items not empty, payment provided]
         v
  +------------------+
  |     PENDING      |    <- Order created, awaiting payment confirmation
  +------------------+
         |
         |------ paymentFailed ---------> +---------+
         |                                | PAYMENT |
         | paymentConfirmed               | FAILED  |
         v                                +---------+
  +------------------+                       |
  |    CONFIRMED     |   <- Payment received  | retry
  +------------------+   , awaiting stock     |
         |                                    v (back to PENDING)
         |------ outOfStock -----------> +---------+
         |                               |CANCELLED|<---- cancel [any state before SHIPPED]
         | stockReserved                 +---------+
         v                                    ^
  +------------------+                        |
  |   PROCESSING     |   <- Warehouse is      | cancel [before shipped]
  +------------------+   packing order        |
         |                                    |
         | shipped                            |
         v
  +------------------+
  |    SHIPPED       |   <- In transit with courier
  +------------------+
         |
         |------ deliveryFailed --------> +-----------+
         |                                | DELIVERY  |
         | delivered                      | FAILED    |
         v                                +-----------+
  +------------------+                       |
  |   DELIVERED      |   <- Customer         | redeliver
  +------------------+   received package    v (back to SHIPPED)
         |
         | returnRequested [within 30 days]
         v
  +------------------+
  | RETURN_REQUESTED |
  +------------------+
         |
         | returnApproved
         v
  +------------------+
  |  RETURN_IN_TRANSIT|
  +------------------+
         |
         | returnReceived
         v
  +------------------+
  |    REFUNDED      |
  +------------------+
         |
         v
        (*)
```

**Transition table** (often drawn alongside diagram in interviews):

| Current State | Event | Guard | Next State | Action |
|---------------|-------|-------|-----------|--------|
| PENDING | paymentConfirmed | - | CONFIRMED | sendConfirmationEmail() |
| PENDING | paymentFailed | - | PAYMENT_FAILED | notifyCustomer() |
| CONFIRMED | stockReserved | - | PROCESSING | notifyWarehouse() |
| CONFIRMED | outOfStock | - | CANCELLED | refundPayment() |
| CONFIRMED | cancel | - | CANCELLED | refundPayment() |
| PROCESSING | shipped | - | SHIPPED | sendShipmentEmail(trackingNo) |
| SHIPPED | delivered | - | DELIVERED | - |
| SHIPPED | deliveryFailed | - | DELIVERY_FAILED | notifyCustomer() |
| DELIVERED | returnRequested | within 30 days | RETURN_REQUESTED | createReturnLabel() |
| RETURN_REQUESTED | returnApproved | - | RETURN_IN_TRANSIT | - |
| RETURN_IN_TRANSIT | returnReceived | - | REFUNDED | processRefund() |

**Java implementation using enum-based state machine:**

```java
public enum OrderStatus {
    PENDING,
    CONFIRMED,
    PAYMENT_FAILED,
    PROCESSING,
    SHIPPED,
    DELIVERED,
    DELIVERY_FAILED,
    RETURN_REQUESTED,
    RETURN_IN_TRANSIT,
    REFUNDED,
    CANCELLED;

    // Allowed transitions: which states can this state move to?
    private static final Map<OrderStatus, Set<OrderStatus>> ALLOWED_TRANSITIONS;

    static {
        ALLOWED_TRANSITIONS = new EnumMap<>(OrderStatus.class);
        ALLOWED_TRANSITIONS.put(PENDING, Set.of(CONFIRMED, PAYMENT_FAILED, CANCELLED));
        ALLOWED_TRANSITIONS.put(CONFIRMED, Set.of(PROCESSING, CANCELLED));
        ALLOWED_TRANSITIONS.put(PAYMENT_FAILED, Set.of(PENDING, CANCELLED));
        ALLOWED_TRANSITIONS.put(PROCESSING, Set.of(SHIPPED, CANCELLED));
        ALLOWED_TRANSITIONS.put(SHIPPED, Set.of(DELIVERED, DELIVERY_FAILED));
        ALLOWED_TRANSITIONS.put(DELIVERED, Set.of(RETURN_REQUESTED));
        ALLOWED_TRANSITIONS.put(DELIVERY_FAILED, Set.of(SHIPPED, CANCELLED));
        ALLOWED_TRANSITIONS.put(RETURN_REQUESTED, Set.of(RETURN_IN_TRANSIT, DELIVERED));
        ALLOWED_TRANSITIONS.put(RETURN_IN_TRANSIT, Set.of(REFUNDED));
        ALLOWED_TRANSITIONS.put(REFUNDED, Set.of()); // terminal
        ALLOWED_TRANSITIONS.put(CANCELLED, Set.of()); // terminal
    }

    public boolean canTransitionTo(OrderStatus newStatus) {
        return ALLOWED_TRANSITIONS.getOrDefault(this, Set.of()).contains(newStatus);
    }
}

public class Order {
    private OrderStatus status;

    public void transitionTo(OrderStatus newStatus) {
        if (!status.canTransitionTo(newStatus)) {
            throw new IllegalStateTransitionException(
                String.format("Cannot transition from %s to %s", status, newStatus));
        }
        this.status = newStatus;
    }

    public void confirmPayment() {
        transitionTo(OrderStatus.CONFIRMED);
    }

    public void ship(String trackingNumber) {
        transitionTo(OrderStatus.SHIPPED);
        this.trackingNumber = trackingNumber;
        notifyCustomer("Your order has been shipped. Tracking: " + trackingNumber);
    }

    public void cancel() {
        if (!status.canTransitionTo(OrderStatus.CANCELLED)) {
            throw new IllegalStateTransitionException("Order cannot be cancelled in state: " + status);
        }
        transitionTo(OrderStatus.CANCELLED);
        if (status == OrderStatus.CONFIRMED || status == OrderStatus.PROCESSING) {
            processRefund(); // only refund if payment was collected
        }
    }
}
```

---

## 3. Example: ATM State Machine

The ATM is a classic state machine — each physical interaction corresponds to a clear state.

```
          (*)
           |
           | cardInserted
           v
  +-------------------+
  |    CARD_INSERTED  |
  +-------------------+
           |
           | pinEntered
           v
  +-------------------+           +------------------+
  |  PIN_VERIFICATION |           |   CARD_BLOCKED    |
  +-------------------+           +------------------+
           |                               ^
           |-- [3 failed attempts] --------|
           |                               
           | [pin correct]
           v
  +-------------------+
  |   AUTHENTICATED   |
  +-------------------+
           |
     ______|______
    |              |
    | selectBalance | selectWithdrawal
    v               v
+----------+   +------------------+
| BALANCE  |   | AMOUNT_SELECTION |
| INQUIRY  |   +------------------+
+----------+          |
    |                 | amountEntered
    | displayBalance  v
    |            +------------------+
    |            | PROCESSING       |
    |            | WITHDRAWAL       |
    |            +------------------+
    |                 |
    |         ________|________
    |        |                 |
    |    [sufficient]     [insufficient]
    |        |                 |
    |        v                 v
    |  +----------+     +-------------+
    |  | DISPENSING|     | INSUFFICIENT|
    |  | CASH     |     | FUNDS       |
    |  +----------+     +-------------+
    |        |                 |
    |        v                 |
    |  +----------+            |
    |  | RECEIPT  |<-----------+
    |  | PRINTING |  (go to receipt printing
    |  +----------+   even on failure)
    |        |
    +------->|
             v
       +-----------+
       | SESSION   |
       | COMPLETE  |
       +-----------+
             |
             | ejectCard
             v
            (*)
```

**Java implementation using State pattern:**

```java
// State interface
public interface ATMState {
    void insertCard(ATMContext atm, String cardNumber);
    void enterPin(ATMContext atm, String pin);
    void selectWithdrawal(ATMContext atm, double amount);
    void ejectCard(ATMContext atm);
}

// Context
public class ATMContext {
    private ATMState currentState;
    private String cardNumber;
    private String accountId;
    private int failedPinAttempts;

    public ATMContext() {
        this.currentState = new IdleState();
    }

    public void setState(ATMState state) {
        this.currentState = state;
    }

    public void insertCard(String cardNumber) {
        currentState.insertCard(this, cardNumber);
    }

    public void enterPin(String pin) {
        currentState.enterPin(this, pin);
    }

    public void selectWithdrawal(double amount) {
        currentState.selectWithdrawal(this, amount);
    }

    public void ejectCard() {
        currentState.ejectCard(this);
    }
}

// State implementations
public class IdleState implements ATMState {
    @Override
    public void insertCard(ATMContext atm, String cardNumber) {
        atm.setCardNumber(cardNumber);
        atm.setState(new CardInsertedState());
        System.out.println("Card inserted. Please enter your PIN.");
    }

    @Override
    public void enterPin(ATMContext atm, String pin) {
        System.out.println("Please insert card first.");
    }

    @Override
    public void selectWithdrawal(ATMContext atm, double amount) {
        System.out.println("Please insert card first.");
    }

    @Override
    public void ejectCard(ATMContext atm) {
        System.out.println("No card inserted.");
    }
}

public class CardInsertedState implements ATMState {
    @Override
    public void insertCard(ATMContext atm, String cardNumber) {
        System.out.println("Card already inserted.");
    }

    @Override
    public void enterPin(ATMContext atm, String pin) {
        if (atm.getBankService().validatePin(atm.getCardNumber(), pin)) {
            atm.setFailedPinAttempts(0);
            atm.setState(new AuthenticatedState());
            System.out.println("PIN correct. What would you like to do?");
        } else {
            int attempts = atm.incrementFailedPinAttempts();
            if (attempts >= 3) {
                atm.getBankService().blockCard(atm.getCardNumber());
                atm.setState(new CardBlockedState());
                System.out.println("Card blocked after 3 failed attempts.");
            } else {
                System.out.println("Incorrect PIN. " + (3 - attempts) + " attempts remaining.");
            }
        }
    }

    @Override
    public void selectWithdrawal(ATMContext atm, double amount) {
        System.out.println("Please enter PIN first.");
    }

    @Override
    public void ejectCard(ATMContext atm) {
        atm.setCardNumber(null);
        atm.setState(new IdleState());
        System.out.println("Card ejected.");
    }
}

public class AuthenticatedState implements ATMState {
    @Override
    public void insertCard(ATMContext atm, String cardNumber) {
        System.out.println("Session in progress. Please eject current card first.");
    }

    @Override
    public void enterPin(ATMContext atm, String pin) {
        System.out.println("Already authenticated.");
    }

    @Override
    public void selectWithdrawal(ATMContext atm, double amount) {
        try {
            atm.getBankService().withdraw(atm.getAccountId(), amount);
            atm.getCashDispenser().dispense(amount);
            atm.getReceiptPrinter().print(atm.getAccountId(), amount);
            System.out.printf("Dispensing $%.2f. Please take your cash and receipt.%n", amount);
            atm.setState(new IdleState());
        } catch (InsufficientFundsException e) {
            System.out.println("Insufficient funds. Available: $" + e.getAvailableBalance());
            // Stay in authenticated state — user can try different amount
        }
    }

    @Override
    public void ejectCard(ATMContext atm) {
        atm.setCardNumber(null);
        atm.setAccountId(null);
        atm.setState(new IdleState());
        System.out.println("Thank you. Card ejected.");
    }
}
```

---

## 4. Example: Traffic Light

A simple but complete state machine to illustrate the fundamentals clearly.

```
    (*)
     |
     | start
     v
+----------+
|   RED    |<---------------------------+
+----------+                           |
  entry / stopTraffic()                |
  do / wait(60s)                       |
     |                                 |
     | timer [60s elapsed]             |
     v                                 |
+----------+                           |
|  GREEN   |                           |
+----------+                           |
  entry / allowTraffic()               |
  do / wait(45s)                       |
     |                                 |
     | timer [45s elapsed]             |
     v                                 |
+----------+                           |
|  YELLOW  |                           |
+----------+                           |
  entry / warnTraffic()                |
  do / wait(5s)                        |
     |                                 |
     | timer [5s elapsed] /            |
     | recordCycleCompletion() --------|
```

**Emergency interrupt — internal transition:**

```
+----------+
|   RED    |
+----------+
  emergency [active] / activateEmergencyMode()   <- self-transition (stays in RED)
     |
     | emergency [inactive]
     v (transition back to normal)
```

**Java implementation:**

```java
public enum TrafficLightState {
    RED(60),
    GREEN(45),
    YELLOW(5);

    private final int durationSeconds;

    TrafficLightState(int durationSeconds) {
        this.durationSeconds = durationSeconds;
    }

    public int getDurationSeconds() { return durationSeconds; }

    public TrafficLightState next() {
        return switch (this) {
            case RED    -> GREEN;
            case GREEN  -> YELLOW;
            case YELLOW -> RED;
        };
    }
}

public class TrafficLight {
    private TrafficLightState currentState;
    private final TrafficController controller;
    private int cycleCount;

    public TrafficLight(TrafficController controller) {
        this.controller = controller;
        this.currentState = TrafficLightState.RED;
        onEnter(currentState);
    }

    public void tick() {
        // Called by a timer/scheduler every second
        // Simplified: treat as "timer elapsed" event
        transition(currentState.next());
    }

    private void transition(TrafficLightState nextState) {
        onExit(currentState);
        this.currentState = nextState;
        if (nextState == TrafficLightState.RED) {
            cycleCount++;
            controller.recordCycleCompletion(cycleCount);
        }
        onEnter(nextState);
    }

    private void onEnter(TrafficLightState state) {
        switch (state) {
            case RED    -> controller.stopTraffic();
            case GREEN  -> controller.allowTraffic();
            case YELLOW -> controller.warnTraffic();
        }
    }

    private void onExit(TrafficLightState state) {
        // Any cleanup on exit — not needed for basic traffic light
    }

    public TrafficLightState getState() { return currentState; }
}
```

---

## 5. When to Use State Diagrams

Use state diagrams when an object:

1. **Has a defined lifecycle** with distinct phases — Order, Ticket, Reservation, Session
2. **Responds differently to the same event depending on current state** — an ATM ignores "enterPin" when idle but processes it when a card is inserted
3. **Must restrict which state transitions are valid** — cannot ship a cancelled order
4. **Has actions on entry/exit** — send email on entering SHIPPED state, release inventory on entering CANCELLED state

**Do NOT use state diagrams when:**
- The object has no meaningful state (stateless services)
- The "states" are just boolean flags with no behavioral difference
- The behavior does not change between states (just use sequence diagrams)

---

## 6. Connection to the State Pattern

The State design pattern is the direct implementation of a state diagram in code. When a state diagram has many states with different behavior for the same events, the State pattern prevents large switch/if-else blocks.

```
State Diagram Element     ->    State Pattern Element
─────────────────────────────────────────────────────
Object with states        ->    Context class
Each state                ->    Concrete State class
State interface           ->    State interface
Transition                ->    setState() call inside a state's method
Entry action              ->    Constructor or onEnter() method
Event                     ->    Method on the State interface
```

When to use State pattern vs enum-based switch:

- **Enum/switch:** Few states (2-4), transitions are simple, no per-state behavior variation
- **State pattern:** Many states (5+), each state has complex behavior, transitions have side effects

---

# Activity Diagrams

## 7. Core Activity Diagram Notation

Activity diagrams show the flow of actions in a process. They are UML's version of flowcharts, but with formal notation for parallelism, object flows, and swimlanes.

### Basic Symbols

```
(*)           Initial node (filled circle) — start here
[Action]      Activity/Action — what happens
<>            Decision node — branch based on condition
|---| |---|   Fork — splits into parallel flows
|---| |---|   Join — waits for all parallel flows to complete
( * )         Final node (filled circle with ring) — end here

[guard]       Condition on a transition (in square brackets)
```

### Simple Activity Flow

```
        (*)
         |
         v
  [Receive Order]
         |
         v
  [Validate Items]
         |
         v
        <diamond>
       /         \
  [valid]      [invalid]
     |               |
     v               v
[Process]     [Notify Customer]
     |               |
     v               v
[Send Confirmation]  (*)
     |
     v
    (*)
```

### Fork and Join (Parallelism)

```
                |
         [================]    <- Fork (horizontal bar): all downstream paths start
        /        |          \
       v         v           v
 [Action A]  [Action B]  [Action C]   <- parallel activities
       \         |          /
        v        v         v
         [================]    <- Join: wait for ALL paths to complete
                |
                v
         [Continue...]
```

---

## 8. Example: Order Fulfillment Process

This process involves multiple sequential and parallel steps.

```
                (*)
                 |
                 v
        [Receive Customer Order]
                 |
                 v
        [Validate Payment Info]
                 |
                 v
               <?>
              /    \
      [valid]        [invalid]
         |                |
         v                v
[Check Inventory]  [Notify Payment Failed]
         |                |
         |               (*)
         v
        <?>
       /    \
 [in stock]  [out of stock]
    |              |
    v              v
[Reserve Items] [Notify Out of Stock]
    |              |
    v             (*)
[Charge Payment]
    |
    v
  <?>
 /    \
[ok]  [declined]
 |         |
 v         v
[=====Fork=====]  [Release Reservation]
 |         |      [Notify Failure]
 v         v           |
[Pack]  [Generate      (*)
[Items]  Invoice]
 |         |
[=====Join=====]
    |
    v
[Ship Order]
    |
    v
[Send Tracking Email]
    |
    v
[Wait for Delivery]
    |
    v
   <?>
  /    \
[delivered] [not delivered after 7 days]
    |              |
    v              v
[Mark Delivered]  [Initiate Investigation]
    |              |
    v              v
[Close Order]    [Reship or Refund]
    |              |
   (*)            (*)
```

**Java implementation mapping:**

```java
@Service
public class OrderFulfillmentService {

    public FulfillmentResult fulfill(Order order) {
        // Validate payment
        if (!paymentService.validate(order.getPaymentInfo())) {
            notificationService.notifyPaymentFailed(order.getCustomer());
            return FulfillmentResult.paymentFailed();
        }

        // Check and reserve inventory
        InventoryReservation reservation = inventoryService.reserve(order.getItems());
        if (!reservation.isSuccessful()) {
            List<String> outOfStock = reservation.getOutOfStockItems();
            notificationService.notifyOutOfStock(order.getCustomer(), outOfStock);
            return FulfillmentResult.outOfStock(outOfStock);
        }

        // Charge payment
        PaymentResult payment = paymentService.charge(order);
        if (!payment.isSuccessful()) {
            inventoryService.releaseReservation(reservation);
            notificationService.notifyPaymentDeclined(order.getCustomer());
            return FulfillmentResult.paymentDeclined();
        }

        // Parallel: pack items AND generate invoice
        CompletableFuture<PackingResult> packingFuture =
            CompletableFuture.supplyAsync(() -> warehouseService.pack(order));
        CompletableFuture<Invoice> invoiceFuture =
            CompletableFuture.supplyAsync(() -> invoiceService.generate(order));

        CompletableFuture.allOf(packingFuture, invoiceFuture).join(); // join — wait for both

        PackingResult packing = packingFuture.join();
        Invoice invoice = invoiceFuture.join();

        // Ship
        TrackingInfo tracking = shippingService.ship(packing.getPackageId(), order.getAddress());
        order.setStatus(OrderStatus.SHIPPED);
        order.setTrackingNumber(tracking.getTrackingNumber());

        // Send confirmation with tracking
        notificationService.sendShipmentNotification(
            order.getCustomer(), tracking, invoice);

        return FulfillmentResult.success(tracking);
    }
}
```

---

## 9. Example: User Registration

This process validates user input, checks for duplicates, creates the account, and sends a welcome email.

```
              (*)
               |
               v
   [Receive Registration Form]
               |
               v
      [Validate Input Format]
         (name, email, pwd)
               |
               v
             <?>
            /    \
        [valid]  [invalid]
           |          |
           v          v
 [Check Email Unique]  [Return Validation Errors]
           |          |
           |         (*)
           v
          <?>
         /    \
    [unique]  [duplicate]
       |            |
       v            v
 [Hash Password] [Return "Email already registered"]
       |            |
       v           (*)
 [Create User Record]
       |
       v
 [Generate Verification Token]
       |
       v
 [=======Fork=======]
 |                   |
 v                   v
[Save User]    [Send Welcome Email
               with Verification Link]
 |                   |
 [=======Join=======]
       |
       v
 [Return Success + UserId]
       |
       v
      (*)
```

**Java implementation:**

```java
@Service
public class UserRegistrationService {

    @Transactional
    public RegistrationResult register(RegistrationRequest request) {
        // Step 1: Validate input format
        ValidationResult validation = validator.validate(request);
        if (!validation.isValid()) {
            return RegistrationResult.validationFailed(validation.getErrors());
        }

        // Step 2: Check email uniqueness
        if (userRepository.existsByEmail(request.getEmail())) {
            return RegistrationResult.emailAlreadyRegistered();
        }

        // Step 3: Hash password
        String hashedPassword = passwordEncoder.encode(request.getPassword());

        // Step 4: Create user
        User user = User.builder()
            .name(request.getName())
            .email(request.getEmail().toLowerCase())
            .passwordHash(hashedPassword)
            .status(UserStatus.PENDING_VERIFICATION)
            .build();

        // Step 5: Generate verification token
        String verificationToken = tokenService.generateVerificationToken();

        // Step 6: Parallel save and email
        // (In production, email is typically async via messaging queue)
        User savedUser = userRepository.save(user);
        tokenRepository.save(new VerificationToken(savedUser.getId(), verificationToken));

        // Send email asynchronously — don't block registration on email success
        emailService.sendWelcomeEmailAsync(
            savedUser.getEmail(),
            savedUser.getName(),
            verificationToken);

        return RegistrationResult.success(savedUser.getId());
    }
}
```

---

## 10. Swimlanes

Swimlanes extend activity diagrams by partitioning activities into lanes, where each lane belongs to a specific actor, system, or department. They make it clear **who does what**.

### Swimlane Notation

```
+------------------+------------------+------------------+
|    Customer      |   OrderSystem    |     Warehouse    |
+------------------+------------------+------------------+
|       (*)        |                  |                  |
|        |         |                  |                  |
|  [Place Order]   |                  |                  |
|        |-------->|                  |                  |
|                  |  [Validate Order]|                  |
|                  |        |         |                  |
|                  |  [Process Payment]                  |
|                  |        |         |                  |
|                  |        |-------->|                  |
|                  |                  | [Pick and Pack]  |
|                  |                  |        |         |
|                  |                  |  [Ship Package]  |
|                  |<-----------------|        |         |
|  [Receive Package]                  |       (*)        |
|       (*)        |                  |                  |
+------------------+------------------+------------------+
```

### Example: E-Commerce Order Fulfillment with Swimlanes

```
+------------+----------------+----------------+------------------+
|  Customer  |  OrderService  |  PaymentSvc    |  WarehouseSvc   |
+------------+----------------+----------------+------------------+
|    (*)     |                |                |                  |
|     |      |                |                |                  |
| [Submit    |                |                |                  |
|  Order] -->|                |                |                  |
|            | [Validate      |                |                  |
|            |  Order Items]  |                |                  |
|            |       |        |                |                  |
|            |       +------->|                |                  |
|            |               | [Authorize     |                  |
|            |               |  Payment]      |                  |
|            |               |     |          |                  |
|            |               |    <?>         |                  |
|            |               |   /   \        |                  |
|            |           [ok]|  [fail]        |                  |
|            |               |    |           |                  |
|            |           [fail path]          |                  |
|            |<-- PaymentFailed               |                  |
| [See Error]|                |               |                  |
|    (*)     |               | [Capture      |                  |
|            |               |  Payment] --->|                  |
|            |<-----------Payment OK          |                  |
|            |        |       |               |                  |
|            |        +------------------------------>           |
|            |                |               | [Fulfill Order]  |
|            |                |               |       |          |
|            |                |               | [Generate Label] |
|            |<-------------------------------|       |          |
|            | [Update Order SHIPPED]         | [Ship Package]   |
|            |        |       |               |       (*)        |
| [Tracking  |<-------|       |               |                  |
|  Email]    |                |               |                  |
|    (*)     |                |               |                  |
+------------+----------------+----------------+------------------+
```

### When to Use Swimlanes

Use swimlanes when:
- A process spans multiple actors or systems
- You want to make handoffs between systems explicit
- Responsibility boundaries need to be clear
- You are documenting a business process for non-technical stakeholders

---

## Summary: Choosing the Right Diagram

| Question | Diagram |
|----------|---------|
| What classes exist and how do they relate? | Class Diagram |
| How do objects interact to fulfill a use case? | Sequence Diagram |
| What states can an object be in and how does it transition? | State Diagram |
| How does a process flow from start to end? | Activity Diagram |
| Who does what in a process? | Activity Diagram with Swimlanes |

---

## Combined Interview Example

**Question:** "Design an ATM system."

A complete answer uses multiple diagrams:

1. **Class Diagram** — ATM, BankServer, Account, Card, Transaction, CardReader, CashDispenser, ReceiptPrinter
2. **State Diagram** — ATM states: Idle → CardInserted → PinVerification → Authenticated → TransactionSelection → Processing → Complete
3. **Sequence Diagram** — Withdrawal flow: Customer → ATM → BankServer → Account → CashDispenser
4. **Activity Diagram** — Withdrawal process with fork for cash dispensing and receipt printing in parallel

Each diagram answers a different question about the same system. Together, they give a complete picture.
