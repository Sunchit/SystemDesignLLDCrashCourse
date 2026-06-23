# State Design Pattern

> **Category:** Behavioral | **Complexity:** Medium-High | **GoF Pattern #29**

---

## Table of Contents

1. [Intent and Problem Statement](#1-intent-and-problem-statement)
2. [The Anti-Pattern: Switch/If-Else State Machines](#2-the-anti-pattern-switchif-else-state-machines)
3. [When to Use / When NOT to Use](#3-when-to-use--when-not-to-use)
4. [UML Class Diagram](#4-uml-class-diagram)
5. [Implementation 1: ATM System](#5-implementation-1-atm-system)
6. [Implementation 2: Vending Machine](#6-implementation-2-vending-machine)
7. [State Transition Diagrams](#7-state-transition-diagrams)
8. [Finite State Machine (FSM) Design Principles](#8-finite-state-machine-fsm-design-principles)
9. [Real-World Framework Examples](#9-real-world-framework-examples)
10. [Trade-offs](#10-trade-offs)
11. [Common Interview Discussion Points](#11-common-interview-discussion-points)
12. [Interview Questions and Model Answers](#12-interview-questions-and-model-answers)
13. [Comparison with Related Patterns](#13-comparison-with-related-patterns)

---

## 1. Intent and Problem Statement

### GoF Definition

> Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

### The Core Problem

Many real-world objects exhibit **state-dependent behavior** — the same operation produces completely different results depending on the current condition of the object:

- An ATM machine behaves differently before a card is inserted vs. after authentication
- A traffic light cycles through Red, Yellow, and Green, with distinct rules at each phase
- A TCP connection behaves differently when it is in CLOSED, LISTEN, ESTABLISHED, or TIME_WAIT states
- A vending machine accepts money differently when it already has a product selected

Without a deliberate pattern, developers instinctively reach for `if-else` chains or `switch` statements to handle these cases. This leads directly to the most pervasive anti-pattern in stateful system design.

### What the State Pattern Does

The State pattern:

1. **Encapsulates each state** into its own class implementing a common `State` interface
2. **Delegates state-dependent behavior** from the context object to the current state object
3. **Encodes transitions** either inside state classes (decentralized) or inside the context class (centralized)
4. **Makes adding new states trivial** — add a class, update transition references, done

The context object holds a reference to its current state and delegates all state-sensitive operations to it. The context appears to change class because its behavior changes entirely.

---

## 2. The Anti-Pattern: Switch/If-Else State Machines

### BEFORE: The Spaghetti State Machine

This is what most engineers write the first time they encounter state-dependent behavior. It starts simple and becomes catastrophically unmaintainable.

```java
// ATM.java — THE ANTI-PATTERN
// This looks innocent at first. After 6 months it becomes a maintenance nightmare.

public class ATM_Antipattern {

    // State represented as a simple enum or integer constant
    public enum ATMState {
        IDLE, HAS_CARD, AUTHENTICATED, DISPENSING, OUT_OF_SERVICE
    }

    private ATMState currentState;
    private int cashAvailable;
    private String insertedCard;
    private String authenticatedAccount;

    public ATM_Antipattern(int cashAvailable) {
        this.cashAvailable = cashAvailable;
        this.currentState = ATMState.IDLE;
    }

    // ============================================================
    // PROBLEM 1: Every method is a massive switch/if-else block.
    // Each method must handle ALL states, even when an operation
    // makes no sense for that state.
    // ============================================================

    public void insertCard(String cardNumber) {
        switch (currentState) {
            case IDLE:
                System.out.println("Card inserted: " + cardNumber);
                this.insertedCard = cardNumber;
                currentState = ATMState.HAS_CARD;
                break;
            case HAS_CARD:
                System.out.println("Card already inserted. Please remove existing card.");
                break;
            case AUTHENTICATED:
                System.out.println("Already authenticated. Please complete transaction.");
                break;
            case DISPENSING:
                System.out.println("Transaction in progress. Please wait.");
                break;
            case OUT_OF_SERVICE:
                System.out.println("ATM is out of service.");
                break;
            // PROBLEM: Every time we add a new state, we MUST update this method.
            // Miss one case? Runtime bugs. No compiler enforcement.
        }
    }

    public void enterPin(String pin) {
        switch (currentState) {
            case IDLE:
                System.out.println("Please insert card first.");
                break;
            case HAS_CARD:
                // Imagine real pin validation logic here
                if (pin.equals("1234")) {
                    System.out.println("PIN accepted. Authentication successful.");
                    authenticatedAccount = "ACC-" + insertedCard;
                    currentState = ATMState.AUTHENTICATED;
                } else {
                    System.out.println("Invalid PIN. Please try again.");
                    // PROBLEM: What about lockout after 3 attempts?
                    // That logic also has to live here, bloating this method further.
                }
                break;
            case AUTHENTICATED:
                System.out.println("Already authenticated.");
                break;
            case DISPENSING:
                System.out.println("Transaction in progress.");
                break;
            case OUT_OF_SERVICE:
                System.out.println("ATM is out of service.");
                break;
        }
    }

    public void requestCash(int amount) {
        switch (currentState) {
            case IDLE:
                System.out.println("Please insert card first.");
                break;
            case HAS_CARD:
                System.out.println("Please authenticate first.");
                break;
            case AUTHENTICATED:
                if (amount <= 0) {
                    System.out.println("Invalid amount.");
                } else if (amount > cashAvailable) {
                    System.out.println("Insufficient cash in ATM.");
                } else {
                    System.out.println("Dispensing $" + amount);
                    currentState = ATMState.DISPENSING;
                    cashAvailable -= amount;
                    // PROBLEM: Transition logic is buried inside a 50-line switch block.
                    // Where does DISPENSING end and IDLE begin?
                    // Is this synchronous? Async? The model is completely unclear.
                    System.out.println("Cash dispensed. Please take your cash.");
                    currentState = ATMState.IDLE;
                    this.insertedCard = null;
                    this.authenticatedAccount = null;
                }
                break;
            case DISPENSING:
                System.out.println("Already dispensing. Please wait.");
                break;
            case OUT_OF_SERVICE:
                System.out.println("ATM is out of service.");
                break;
        }
    }

    public void ejectCard() {
        switch (currentState) {
            case IDLE:
                System.out.println("No card inserted.");
                break;
            case HAS_CARD:
            case AUTHENTICATED:
                System.out.println("Card ejected.");
                currentState = ATMState.IDLE;
                this.insertedCard = null;
                this.authenticatedAccount = null;
                break;
            case DISPENSING:
                System.out.println("Cannot eject card during dispensing.");
                break;
            case OUT_OF_SERVICE:
                System.out.println("ATM is out of service. Contact support.");
                break;
        }
    }

    // ============================================================
    // PROBLEM 2: Adding a new state means editing EVERY method.
    // If the ATM needs a "MAINTENANCE" state, you update 4+ methods.
    // If you add a new operation "requestBalance()", you write another
    // full switch block from scratch. This is O(states * operations)
    // code that must be maintained in sync — a violation of OCP.
    //
    // PROBLEM 3: Testing is a nightmare. You cannot test one state
    // in isolation. Every test must set up the ATM object and drive it
    // to the desired state through a sequence of operations.
    //
    // PROBLEM 4: State transitions are scattered everywhere.
    // To understand what states X can transition to, you must read
    // ALL methods in the class — there is no single source of truth.
    //
    // PROBLEM 5: Thread safety is impossible to reason about.
    // The currentState field is accessed and mutated in every method.
    // ============================================================
}
```

### Why It Becomes Unmaintainable

| Growth Vector | Impact on Switch/If-Else |
|---|---|
| Add 1 new state | Must edit every existing method |
| Add 1 new operation | Must write a full switch covering every state |
| Change a transition rule | Must grep across all methods to find all mentions |
| Test a specific state | Must drive the object through prerequisite operations |
| Add thread safety | Must synchronize the entire context object |
| Onboard a new engineer | Must read thousands of lines to understand the state machine |

At 5 states and 5 operations, the switch-based approach has 25 code blocks to maintain. At 10 states and 10 operations, it is 100. The State pattern keeps this linear: N state classes, each with M methods. Adding a state adds one class. Adding an operation adds one method per class.

---

## 3. When to Use / When NOT to Use

### Use the State Pattern When

- **Object behavior varies significantly by state** and the number of states is non-trivial (3+)
- **State transitions are complex** with conditional rules about which transitions are legal
- **Code has massive conditional blocks** that check the same state variable across multiple methods
- **New states will be added** over the lifetime of the project (OCP violation risk is high)
- **States need encapsulated data** — each state may hold state-local variables (e.g., retry count in an authentication state)
- **You need auditable state transitions** — each State class is a natural audit point
- **States represent domain concepts** that deserve first-class citizenship in the codebase (e.g., `OrderState`, `PaymentState`)

### Do NOT Use the State Pattern When

- **The object has only 2 states** — a simple boolean or enum is cleaner and less indirection
- **State-dependent behavior is trivial** — a single `if` statement does not warrant a pattern
- **States don't need to be added** and are unlikely to change — the added classes are pure overhead
- **The lifecycle is linear with no branching** — a sequential pipeline (Builder, Chain of Responsibility) is more appropriate
- **You're in a value-object domain** — stateless, immutable objects don't benefit from this pattern
- **Performance is critical and state changes are extremely frequent** — object allocation for each state change adds GC pressure

---

## 4. UML Class Diagram

```
                        +------------------+
                        |   <<interface>>  |
                        |    ATMState      |
                        +------------------+
                        | +insertCard()    |
                        | +enterPin()      |
                        | +requestCash()   |
                        | +ejectCard()     |
                        | +getName()       |
                        +------------------+
                                 △
                                 | implements
          ┌──────────────────────┼──────────────────────────┐
          │                      │                           │
+-----------------+   +--------------------+   +---------------------+
|   IdleState     |   |   HasCardState     |   | AuthenticatedState  |
+-----------------+   +--------------------+   +---------------------+
| -atm: ATMMachine|   | -atm: ATMMachine   |   | -atm: ATMMachine    |
+-----------------+   +--------------------+   +---------------------+
| +insertCard()   |   | +insertCard()      |   | +insertCard()       |
| +enterPin()     |   | +enterPin()        |   | +enterPin()         |
| +requestCash()  |   | +requestCash()     |   | +requestCash()      |
| +ejectCard()    |   | +ejectCard()       |   | +ejectCard()        |
+-----------------+   +--------------------+   +---------------------+
                                  △
                                  | implements
              ┌───────────────────┴───────────────────┐
              │                                       │
   +---------------------+              +---------------------+
   |   DispensingState   |              |  OutOfServiceState  |
   +---------------------+              +---------------------+
   | -atm: ATMMachine    |              | -atm: ATMMachine    |
   +---------------------+              +---------------------+
   | +insertCard()       |              | +insertCard()       |
   | +enterPin()         |              | +enterPin()         |
   | +requestCash()      |              | +requestCash()      |
   | +ejectCard()        |              | +ejectCard()        |
   +---------------------+              +---------------------+

                     Uses (delegates to current state)
+------------------+ ─────────────────────────────────────► +------------------+
|   ATMMachine     |                                        |    ATMState      |
|   (Context)      |                                        |   (Interface)    |
+------------------+                                        +------------------+
| -currentState    |
| -cashAvailable   |
| -cardNumber      |
+------------------+
| +insertCard()    |  ── delegates to ──► currentState.insertCard()
| +enterPin()      |  ── delegates to ──► currentState.enterPin()
| +requestCash()   |  ── delegates to ──► currentState.requestCash()
| +ejectCard()     |  ── delegates to ──► currentState.ejectCard()
| +setState()      |
| +getState()      |
+------------------+
```

---

## 5. Implementation 1: ATM System

### Package Structure

```
atm/
├── ATMState.java              — State interface
├── ATMMachine.java            — Context class
├── state/
│   ├── IdleState.java
│   ├── HasCardState.java
│   ├── AuthenticatedState.java
│   ├── DispensingState.java
│   └── OutOfServiceState.java
├── exception/
│   └── InvalidOperationException.java
└── ATMDemo.java               — Entry point
```

### ATMState.java — The State Interface

```java
package atm;

/**
 * State interface defining all operations an ATM can perform.
 *
 * Every concrete state must implement all operations. States that
 * do not support an operation throw InvalidOperationException,
 * providing a clear contract for callers.
 *
 * Design note: we use checked exceptions for invalid operations
 * because callers in production MUST handle the case where a user
 * performs an out-of-sequence action (e.g., entering PIN before
 * inserting a card).
 */
public interface ATMState {

    /**
     * User inserts a bank card.
     * @param cardNumber  the card identifier read by the reader
     */
    void insertCard(String cardNumber);

    /**
     * User enters their PIN for authentication.
     * @param pin  the entered PIN (in production this would be hashed)
     */
    void enterPin(String pin);

    /**
     * User requests a cash withdrawal.
     * @param amount  the requested cash amount in dollars
     */
    void requestCash(int amount);

    /**
     * User or system ejects the card.
     */
    void ejectCard();

    /**
     * Returns a human-readable name for this state.
     * Used for logging, monitoring, and audit trails.
     */
    String getName();
}
```

### InvalidOperationException.java

```java
package atm.exception;

/**
 * Thrown when a caller attempts an operation that is not valid
 * for the ATM's current state.
 *
 * This is a RuntimeException because state validation is a
 * programming/user-flow concern. UI layers should catch this
 * and display an appropriate message to the user.
 */
public class InvalidOperationException extends RuntimeException {

    private final String currentState;
    private final String attemptedOperation;

    public InvalidOperationException(String currentState, String attemptedOperation) {
        super(String.format(
            "Operation '%s' is not valid when ATM is in state '%s'",
            attemptedOperation, currentState
        ));
        this.currentState = currentState;
        this.attemptedOperation = attemptedOperation;
    }

    public String getCurrentState() {
        return currentState;
    }

    public String getAttemptedOperation() {
        return attemptedOperation;
    }
}
```

### ATMMachine.java — The Context

```java
package atm;

import atm.exception.InvalidOperationException;
import atm.state.*;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * ATMMachine is the Context in the State pattern.
 *
 * It holds a reference to the current state and delegates all
 * state-sensitive operations to it. The ATMMachine also:
 *  - Provides state factory methods so states can navigate to one another
 *  - Tracks an audit log of state transitions
 *  - Enforces invariants (e.g., cash amount cannot go negative)
 *  - Exposes package-private setters so State classes can mutate context data
 *
 * Thread safety note: In production, state transitions would be guarded
 * by a ReentrantLock or synchronized block to prevent race conditions
 * (e.g., two threads both authenticating simultaneously). This is omitted
 * here for clarity but noted as a required production concern.
 */
public class ATMMachine {

    // ---------------------------------------------------------------
    // State instances — created once and reused (Flyweight + State)
    // This avoids allocation on every transition.
    // ---------------------------------------------------------------
    private final ATMState idleState;
    private final ATMState hasCardState;
    private final ATMState authenticatedState;
    private final ATMState dispensingState;
    private final ATMState outOfServiceState;

    // ---------------------------------------------------------------
    // Context data — mutable fields managed by the context
    // State classes read and update these via package-private accessors
    // ---------------------------------------------------------------
    private ATMState currentState;
    private int cashAvailable;
    private String insertedCardNumber;
    private String authenticatedAccountId;
    private int pinAttemptCount;

    // Audit trail — append-only for compliance requirements
    private final List<String> auditLog;

    // ---------------------------------------------------------------
    // Constructor
    // ---------------------------------------------------------------

    public ATMMachine(int initialCash) {
        if (initialCash < 0) {
            throw new IllegalArgumentException("Initial cash cannot be negative");
        }

        // Create all state instances, passing 'this' so they can call
        // back into the context to trigger transitions
        this.idleState         = new IdleState(this);
        this.hasCardState      = new HasCardState(this);
        this.authenticatedState = new AuthenticatedState(this);
        this.dispensingState   = new DispensingState(this);
        this.outOfServiceState = new OutOfServiceState(this);

        this.cashAvailable  = initialCash;
        this.auditLog       = new ArrayList<>();
        this.pinAttemptCount = 0;

        // ATM starts in IDLE state
        this.currentState = idleState;
        log("ATM initialized with $" + initialCash + " cash. State: IDLE");
    }

    // ---------------------------------------------------------------
    // Public API — delegates to current state
    // ---------------------------------------------------------------

    public void insertCard(String cardNumber) {
        log("ACTION: insertCard(" + cardNumber + ") in state: " + currentState.getName());
        currentState.insertCard(cardNumber);
    }

    public void enterPin(String pin) {
        log("ACTION: enterPin(****) in state: " + currentState.getName());
        currentState.enterPin(pin);
    }

    public void requestCash(int amount) {
        log("ACTION: requestCash(" + amount + ") in state: " + currentState.getName());
        currentState.requestCash(amount);
    }

    public void ejectCard() {
        log("ACTION: ejectCard() in state: " + currentState.getName());
        currentState.ejectCard();
    }

    // ---------------------------------------------------------------
    // State transition — called by State classes, not by clients
    // Package-private so only classes in the same package can call it
    // ---------------------------------------------------------------

    void setState(ATMState newState) {
        log("TRANSITION: " + currentState.getName() + " --> " + newState.getName());
        this.currentState = newState;
    }

    // ---------------------------------------------------------------
    // State factory accessors — States use these to navigate
    // ---------------------------------------------------------------

    ATMState getIdleState()          { return idleState; }
    ATMState getHasCardState()       { return hasCardState; }
    ATMState getAuthenticatedState() { return authenticatedState; }
    ATMState getDispensingState()    { return dispensingState; }
    ATMState getOutOfServiceState()  { return outOfServiceState; }

    // ---------------------------------------------------------------
    // Context data accessors — package-private for State classes
    // ---------------------------------------------------------------

    void setInsertedCard(String cardNumber) {
        this.insertedCardNumber = cardNumber;
        this.pinAttemptCount = 0; // reset on new card
    }

    String getInsertedCard() {
        return insertedCardNumber;
    }

    void setAuthenticatedAccount(String accountId) {
        this.authenticatedAccountId = accountId;
    }

    String getAuthenticatedAccount() {
        return authenticatedAccountId;
    }

    void clearSessionData() {
        this.insertedCardNumber = null;
        this.authenticatedAccountId = null;
        this.pinAttemptCount = 0;
    }

    int getCashAvailable() {
        return cashAvailable;
    }

    void dispenseCash(int amount) {
        if (amount > cashAvailable) {
            throw new IllegalStateException(
                "Cannot dispense $" + amount + "; only $" + cashAvailable + " available"
            );
        }
        this.cashAvailable -= amount;
        log("DISPENSE: $" + amount + " dispensed. Remaining cash: $" + cashAvailable);

        // If cash runs out, transition to OUT_OF_SERVICE after this dispense
        if (cashAvailable == 0) {
            log("WARNING: ATM cash depleted. Transitioning to OUT_OF_SERVICE.");
            setState(outOfServiceState);
        }
    }

    int incrementAndGetPinAttempts() {
        return ++pinAttemptCount;
    }

    // ---------------------------------------------------------------
    // Public query methods
    // ---------------------------------------------------------------

    public String getCurrentStateName() {
        return currentState.getName();
    }

    public int getCashAvailablePublic() {
        return cashAvailable;
    }

    public List<String> getAuditLog() {
        return Collections.unmodifiableList(auditLog);
    }

    public boolean isOutOfService() {
        return currentState == outOfServiceState;
    }

    // ---------------------------------------------------------------
    // Admin operations
    // ---------------------------------------------------------------

    public void replenishCash(int amount, String technicianId) {
        if (amount <= 0) throw new IllegalArgumentException("Replenish amount must be positive");
        log("REPLENISH: Technician " + technicianId + " added $" + amount);
        this.cashAvailable += amount;
        if (currentState == outOfServiceState && cashAvailable > 0) {
            setState(idleState);
        }
    }

    // ---------------------------------------------------------------
    // Internal helpers
    // ---------------------------------------------------------------

    private void log(String message) {
        String entry = "[" + System.currentTimeMillis() + "] " + message;
        auditLog.add(entry);
        System.out.println(entry);
    }
}
```

### IdleState.java

```java
package atm.state;

import atm.ATMMachine;
import atm.ATMState;
import atm.exception.InvalidOperationException;

/**
 * IdleState: No card is inserted. The ATM is waiting.
 *
 * Valid operations:  insertCard
 * Invalid operations: enterPin, requestCash, ejectCard
 *
 * Transitions: IDLE --> HAS_CARD (on successful card insertion)
 */
public class IdleState implements ATMState {

    private final ATMMachine atm;

    public IdleState(ATMMachine atm) {
        this.atm = atm;
    }

    @Override
    public void insertCard(String cardNumber) {
        if (cardNumber == null || cardNumber.isBlank()) {
            System.out.println("[IDLE] Card read failed — no card number detected.");
            return;
        }

        System.out.println("[IDLE] Card inserted: " + cardNumber);
        atm.setInsertedCard(cardNumber);
        atm.setState(atm.getHasCardState());
        System.out.println("[IDLE] Please enter your PIN.");
    }

    @Override
    public void enterPin(String pin) {
        throw new InvalidOperationException(getName(), "enterPin");
    }

    @Override
    public void requestCash(int amount) {
        throw new InvalidOperationException(getName(), "requestCash");
    }

    @Override
    public void ejectCard() {
        System.out.println("[IDLE] No card to eject.");
        // Not an error — user may press eject button by mistake
    }

    @Override
    public String getName() {
        return "IDLE";
    }
}
```

### HasCardState.java

```java
package atm.state;

import atm.ATMMachine;
import atm.ATMState;
import atm.exception.InvalidOperationException;

/**
 * HasCardState: A card has been inserted and is awaiting PIN entry.
 *
 * Valid operations:  enterPin, ejectCard
 * Invalid operations: insertCard (already has one), requestCash (not authenticated)
 *
 * Transitions:
 *   HAS_CARD --> AUTHENTICATED  (valid PIN entered)
 *   HAS_CARD --> IDLE           (card ejected, or 3 PIN failures)
 *
 * Key behaviors:
 *   - Tracks PIN attempt count; locks the card after 3 failures
 *   - In production, PIN would be validated against a remote auth service
 */
public class HasCardState implements ATMState {

    private static final int MAX_PIN_ATTEMPTS = 3;

    // In production this would call a card validation service
    private static final String CORRECT_PIN = "1234";

    private final ATMMachine atm;

    public HasCardState(ATMMachine atm) {
        this.atm = atm;
    }

    @Override
    public void insertCard(String cardNumber) {
        System.out.println("[HAS_CARD] A card is already inserted. Please complete the current transaction.");
        throw new InvalidOperationException(getName(), "insertCard");
    }

    @Override
    public void enterPin(String pin) {
        if (pin == null || pin.isBlank()) {
            System.out.println("[HAS_CARD] PIN cannot be empty.");
            return;
        }

        int attempts = atm.incrementAndGetPinAttempts();

        if (CORRECT_PIN.equals(pin)) {
            System.out.println("[HAS_CARD] PIN accepted. Authentication successful.");

            // Derive account ID from card number — in production, this comes
            // from the card validation service response
            String accountId = "ACC-" + atm.getInsertedCard();
            atm.setAuthenticatedAccount(accountId);
            atm.setState(atm.getAuthenticatedState());
            System.out.println("[HAS_CARD] Welcome! Account: " + accountId);
        } else {
            System.out.println("[HAS_CARD] Incorrect PIN. Attempt " + attempts + " of " + MAX_PIN_ATTEMPTS);

            if (attempts >= MAX_PIN_ATTEMPTS) {
                System.out.println("[HAS_CARD] Maximum PIN attempts exceeded. Card will be retained for security.");
                // Card is retained (not returned) — security measure
                atm.clearSessionData();
                atm.setState(atm.getIdleState());
            }
        }
    }

    @Override
    public void requestCash(int amount) {
        System.out.println("[HAS_CARD] Please authenticate before requesting cash.");
        throw new InvalidOperationException(getName(), "requestCash");
    }

    @Override
    public void ejectCard() {
        System.out.println("[HAS_CARD] Card ejected. Transaction cancelled.");
        atm.clearSessionData();
        atm.setState(atm.getIdleState());
    }

    @Override
    public String getName() {
        return "HAS_CARD";
    }
}
```

### AuthenticatedState.java

```java
package atm.state;

import atm.ATMMachine;
import atm.ATMState;
import atm.exception.InvalidOperationException;

/**
 * AuthenticatedState: The user has been authenticated and may perform
 * account operations.
 *
 * Valid operations:  requestCash, ejectCard
 * Invalid operations: insertCard (already has card), enterPin (already authed)
 *
 * Transitions:
 *   AUTHENTICATED --> DISPENSING   (cash requested, valid amount)
 *   AUTHENTICATED --> IDLE         (card ejected / transaction cancelled)
 *   AUTHENTICATED --> OUT_OF_SERVICE (if cash runs out mid-operation)
 */
public class AuthenticatedState implements ATMState {

    private final ATMMachine atm;

    public AuthenticatedState(ATMMachine atm) {
        this.atm = atm;
    }

    @Override
    public void insertCard(String cardNumber) {
        System.out.println("[AUTHENTICATED] A card is already active. Please complete your transaction.");
        throw new InvalidOperationException(getName(), "insertCard");
    }

    @Override
    public void enterPin(String pin) {
        System.out.println("[AUTHENTICATED] Already authenticated. No need to enter PIN again.");
        // Not an error — user may be confused, but no state change
    }

    @Override
    public void requestCash(int amount) {
        if (amount <= 0) {
            System.out.println("[AUTHENTICATED] Invalid amount: $" + amount + ". Amount must be positive.");
            return;
        }

        if (amount % 10 != 0) {
            System.out.println("[AUTHENTICATED] Amount must be a multiple of $10. Requested: $" + amount);
            return;
        }

        int available = atm.getCashAvailable();
        if (amount > available) {
            System.out.println("[AUTHENTICATED] Insufficient ATM cash. Requested: $" + amount
                + ", Available: $" + available);
            System.out.println("[AUTHENTICATED] Please request a smaller amount or visit another ATM.");
            return;
        }

        System.out.println("[AUTHENTICATED] Processing withdrawal of $" + amount + " for account: "
            + atm.getAuthenticatedAccount());

        // Transition to DISPENSING — cash has been committed
        atm.setState(atm.getDispensingState());

        // Perform the actual dispense — this may trigger OUT_OF_SERVICE
        // transition inside the ATMMachine if cash hits zero
        atm.dispenseCash(amount);

        System.out.println("[AUTHENTICATED] Please take your $" + amount + ".");

        // After successful dispense, return card and go back to IDLE
        // (unless out of service was triggered by dispenseCash)
        if (!atm.isOutOfService()) {
            finishTransaction();
        }
    }

    @Override
    public void ejectCard() {
        System.out.println("[AUTHENTICATED] Card ejected. Session ended. Goodbye!");
        atm.clearSessionData();
        atm.setState(atm.getIdleState());
    }

    @Override
    public String getName() {
        return "AUTHENTICATED";
    }

    private void finishTransaction() {
        System.out.println("[AUTHENTICATED] Transaction complete. Please take your card.");
        atm.clearSessionData();
        atm.setState(atm.getIdleState());
    }
}
```

### DispensingState.java

```java
package atm.state;

import atm.ATMMachine;
import atm.ATMState;
import atm.exception.InvalidOperationException;

/**
 * DispensingState: The ATM is actively dispensing cash.
 *
 * No user operations are valid during this state. This state represents
 * the mechanical operation of the cash dispenser — it exists so that
 * if the dispenser jams or fails, the system is in a well-defined state
 * rather than mid-transition.
 *
 * In an async system, this state would persist until the dispenser
 * sends a completion/failure callback. This state can transition
 * to OUT_OF_SERVICE if a dispenser hardware fault is detected.
 *
 * Transitions:
 *   DISPENSING --> IDLE           (dispense complete, card returned)
 *   DISPENSING --> OUT_OF_SERVICE (hardware fault detected)
 *
 * Note: In this synchronous implementation, the ATMMachine drives
 * the DISPENSING --> IDLE transition immediately after dispenseCash()
 * completes. In a real system with async hardware, DispensingState
 * would have a notifyDispenserComplete() method.
 */
public class DispensingState implements ATMState {

    private final ATMMachine atm;

    public DispensingState(ATMMachine atm) {
        this.atm = atm;
    }

    @Override
    public void insertCard(String cardNumber) {
        System.out.println("[DISPENSING] Cannot insert card — transaction in progress.");
        throw new InvalidOperationException(getName(), "insertCard");
    }

    @Override
    public void enterPin(String pin) {
        System.out.println("[DISPENSING] Cannot enter PIN — transaction in progress.");
        throw new InvalidOperationException(getName(), "enterPin");
    }

    @Override
    public void requestCash(int amount) {
        System.out.println("[DISPENSING] Cannot request cash — already dispensing.");
        throw new InvalidOperationException(getName(), "requestCash");
    }

    @Override
    public void ejectCard() {
        System.out.println("[DISPENSING] Cannot eject card — transaction in progress. Please wait.");
        throw new InvalidOperationException(getName(), "ejectCard");
    }

    /**
     * Called by hardware integration layer when a dispenser jam is detected.
     * Transitions to OUT_OF_SERVICE and notifies the service team.
     */
    public void notifyHardwareFault(String faultCode) {
        System.out.println("[DISPENSING] Hardware fault detected: " + faultCode);
        System.out.println("[DISPENSING] Transitioning to OUT_OF_SERVICE. Please contact support.");
        atm.clearSessionData();
        atm.setState(atm.getOutOfServiceState());
    }

    @Override
    public String getName() {
        return "DISPENSING";
    }
}
```

### OutOfServiceState.java

```java
package atm.state;

import atm.ATMMachine;
import atm.ATMState;
import atm.exception.InvalidOperationException;

/**
 * OutOfServiceState: The ATM is unavailable — either cash is depleted,
 * a hardware fault occurred, or it is undergoing maintenance.
 *
 * No user operations are available. A technician must replenish cash
 * and/or clear the fault condition via ATMMachine.replenishCash().
 *
 * Transitions:
 *   OUT_OF_SERVICE --> IDLE  (cash replenished by technician)
 *
 * Design note: This is a "sink" state from the user's perspective.
 * Only administrative operations can escape it.
 */
public class OutOfServiceState implements ATMState {

    private final ATMMachine atm;

    public OutOfServiceState(ATMMachine atm) {
        this.atm = atm;
    }

    @Override
    public void insertCard(String cardNumber) {
        System.out.println("[OUT_OF_SERVICE] This ATM is currently out of service. Please use another ATM.");
    }

    @Override
    public void enterPin(String pin) {
        System.out.println("[OUT_OF_SERVICE] This ATM is currently out of service.");
    }

    @Override
    public void requestCash(int amount) {
        System.out.println("[OUT_OF_SERVICE] This ATM is currently out of service.");
    }

    @Override
    public void ejectCard() {
        // If a card is somehow stuck in OUT_OF_SERVICE, attempt to eject
        if (atm.getInsertedCard() != null) {
            System.out.println("[OUT_OF_SERVICE] Emergency card ejection. Please contact your bank.");
            atm.clearSessionData();
        } else {
            System.out.println("[OUT_OF_SERVICE] No card to eject.");
        }
    }

    @Override
    public String getName() {
        return "OUT_OF_SERVICE";
    }
}
```

### ATMDemo.java — Entry Point and Integration Test

```java
package atm;

import atm.exception.InvalidOperationException;

/**
 * Demonstrates the ATM State Machine through several scenarios:
 *   1. Happy path: insert card → PIN → withdraw cash → eject
 *   2. Invalid PIN with lockout
 *   3. Insufficient funds
 *   4. Cash depletion leading to OUT_OF_SERVICE
 *   5. Invalid operations in wrong state
 *   6. Technician replenish
 */
public class ATMDemo {

    public static void main(String[] args) {
        System.out.println("=".repeat(70));
        System.out.println("SCENARIO 1: Happy Path — Successful Cash Withdrawal");
        System.out.println("=".repeat(70));
        happyPath();

        System.out.println("\n" + "=".repeat(70));
        System.out.println("SCENARIO 2: PIN Lockout after 3 Failed Attempts");
        System.out.println("=".repeat(70));
        pinLockout();

        System.out.println("\n" + "=".repeat(70));
        System.out.println("SCENARIO 3: Insufficient ATM Cash");
        System.out.println("=".repeat(70));
        insufficientFunds();

        System.out.println("\n" + "=".repeat(70));
        System.out.println("SCENARIO 4: Cash Depletion → OUT_OF_SERVICE → Replenish");
        System.out.println("=".repeat(70));
        cashDepletion();

        System.out.println("\n" + "=".repeat(70));
        System.out.println("SCENARIO 5: Invalid Operations in Wrong States");
        System.out.println("=".repeat(70));
        invalidOperations();
    }

    private static void happyPath() {
        ATMMachine atm = new ATMMachine(500);
        System.out.println("State: " + atm.getCurrentStateName()); // IDLE

        atm.insertCard("CARD-4567");
        System.out.println("State: " + atm.getCurrentStateName()); // HAS_CARD

        atm.enterPin("1234");
        System.out.println("State: " + atm.getCurrentStateName()); // AUTHENTICATED

        atm.requestCash(100);
        System.out.println("State: " + atm.getCurrentStateName()); // IDLE (after dispense)
        System.out.println("Remaining ATM cash: $" + atm.getCashAvailablePublic()); // $400
    }

    private static void pinLockout() {
        ATMMachine atm = new ATMMachine(500);
        atm.insertCard("CARD-9999");

        atm.enterPin("0000"); // Wrong
        atm.enterPin("1111"); // Wrong
        atm.enterPin("2222"); // Wrong — card retained, back to IDLE

        System.out.println("State after lockout: " + atm.getCurrentStateName()); // IDLE
    }

    private static void insufficientFunds() {
        ATMMachine atm = new ATMMachine(50); // Only $50 in ATM
        atm.insertCard("CARD-1234");
        atm.enterPin("1234");

        atm.requestCash(200); // More than available — graceful error, stays AUTHENTICATED
        System.out.println("State: " + atm.getCurrentStateName()); // AUTHENTICATED

        atm.requestCash(50); // Exactly available — succeeds
        System.out.println("State: " + atm.getCurrentStateName()); // OUT_OF_SERVICE (cash depleted)
    }

    private static void cashDepletion() {
        ATMMachine atm = new ATMMachine(100);
        atm.insertCard("CARD-0001");
        atm.enterPin("1234");
        atm.requestCash(100); // Depletes cash — transitions to OUT_OF_SERVICE

        System.out.println("State: " + atm.getCurrentStateName()); // OUT_OF_SERVICE

        // User tries to use the ATM
        atm.insertCard("CARD-1111");  // Gets polite out-of-service message

        // Technician replenishes
        atm.replenishCash(1000, "TECH-42");
        System.out.println("State after replenish: " + atm.getCurrentStateName()); // IDLE

        // ATM is back in service
        atm.insertCard("CARD-2222");
        System.out.println("State: " + atm.getCurrentStateName()); // HAS_CARD
        atm.ejectCard();
    }

    private static void invalidOperations() {
        ATMMachine atm = new ATMMachine(500);

        // Try entering PIN when no card inserted
        try {
            atm.enterPin("1234");
        } catch (InvalidOperationException e) {
            System.out.println("Caught expected exception: " + e.getMessage());
        }

        // Try requesting cash when no card inserted
        try {
            atm.requestCash(100);
        } catch (InvalidOperationException e) {
            System.out.println("Caught expected exception: " + e.getMessage());
        }

        // Insert card and try inserting another
        atm.insertCard("CARD-FIRST");
        try {
            atm.insertCard("CARD-SECOND");
        } catch (InvalidOperationException e) {
            System.out.println("Caught expected exception: " + e.getMessage());
        }
    }
}
```

---

## 6. Implementation 2: Vending Machine

### VendingMachineState.java — State Interface

```java
package vendingmachine;

import java.util.Map;

public interface VendingMachineState {

    void insertMoney(int amount);
    void selectProduct(String productCode);
    void dispense();
    void cancel();
    String getName();
}
```

### Product.java — Domain Model

```java
package vendingmachine;

/**
 * Represents a product slot in the vending machine.
 */
public class Product {

    private final String code;
    private final String name;
    private final int priceInCents;
    private int quantity;

    public Product(String code, String name, int priceInCents, int quantity) {
        this.code          = code;
        this.name          = name;
        this.priceInCents  = priceInCents;
        this.quantity      = quantity;
    }

    public String getCode()        { return code; }
    public String getName()        { return name; }
    public int getPriceInCents()   { return priceInCents; }
    public int getQuantity()       { return quantity; }
    public boolean isAvailable()   { return quantity > 0; }

    public void decrementQuantity() {
        if (quantity <= 0) throw new IllegalStateException("Cannot decrement — product is out of stock");
        this.quantity--;
    }

    @Override
    public String toString() {
        return String.format("[%s] %s - $%.2f (%d left)", code, name, priceInCents / 100.0, quantity);
    }
}
```

### VendingMachine.java — Context

```java
package vendingmachine;

import vendingmachine.state.*;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

/**
 * VendingMachine is the Context in the State pattern.
 *
 * It pre-creates all state instances and holds a reference to
 * the current state. State classes call back into the context
 * via package-private methods to trigger transitions.
 */
public class VendingMachine {

    // States (created once, reused — Flyweight optimization)
    private final VendingMachineState idleState;
    private final VendingMachineState hasMoneyState;
    private final VendingMachineState productSelectedState;
    private final VendingMachineState dispensingState;
    private final VendingMachineState outOfStockState;

    // Context data
    private VendingMachineState currentState;
    private int insertedAmountCents;
    private String selectedProductCode;
    private final Map<String, Product> inventory;

    public VendingMachine() {
        this.idleState             = new IdleState(this);
        this.hasMoneyState         = new HasMoneyState(this);
        this.productSelectedState  = new ProductSelectedState(this);
        this.dispensingState       = new DispensingVMState(this);
        this.outOfStockState       = new OutOfStockState(this);

        this.inventory             = new HashMap<>();
        this.insertedAmountCents   = 0;
        this.selectedProductCode   = null;

        this.currentState = idleState;
    }

    // ---------------------------------------------------------------
    // Public API — delegates to state
    // ---------------------------------------------------------------

    public void insertMoney(int cents) {
        System.out.println("\n[VM] insertMoney(" + cents + " cents) in state: " + currentState.getName());
        currentState.insertMoney(cents);
    }

    public void selectProduct(String productCode) {
        System.out.println("[VM] selectProduct(" + productCode + ") in state: " + currentState.getName());
        currentState.selectProduct(productCode);
    }

    public void dispense() {
        System.out.println("[VM] dispense() in state: " + currentState.getName());
        currentState.dispense();
    }

    public void cancel() {
        System.out.println("[VM] cancel() in state: " + currentState.getName());
        currentState.cancel();
    }

    // ---------------------------------------------------------------
    // State navigation — called by State classes
    // ---------------------------------------------------------------

    void setState(VendingMachineState state) {
        System.out.println("[VM] State transition: " + currentState.getName() + " --> " + state.getName());
        this.currentState = state;
    }

    // ---------------------------------------------------------------
    // State accessors for transitions
    // ---------------------------------------------------------------

    VendingMachineState getIdleState()            { return idleState; }
    VendingMachineState getHasMoneyState()        { return hasMoneyState; }
    VendingMachineState getProductSelectedState() { return productSelectedState; }
    VendingMachineState getDispensingState()      { return dispensingState; }
    VendingMachineState getOutOfStockState()      { return outOfStockState; }

    // ---------------------------------------------------------------
    // Context data access — package-private
    // ---------------------------------------------------------------

    void addInsertedAmount(int cents) {
        this.insertedAmountCents += cents;
        System.out.println("[VM] Total inserted: " + insertedAmountCents + " cents");
    }

    int getInsertedAmountCents() {
        return insertedAmountCents;
    }

    void setSelectedProduct(String productCode) {
        this.selectedProductCode = productCode;
    }

    String getSelectedProductCode() {
        return selectedProductCode;
    }

    Product getProduct(String code) {
        return inventory.get(code);
    }

    boolean hasProduct(String code) {
        Product p = inventory.get(code);
        return p != null && p.isAvailable();
    }

    boolean allProductsOutOfStock() {
        return inventory.values().stream().noneMatch(Product::isAvailable);
    }

    int refundAndReset() {
        int refund = this.insertedAmountCents;
        this.insertedAmountCents = 0;
        this.selectedProductCode = null;
        System.out.println("[VM] Session reset. Refund: " + refund + " cents.");
        return refund;
    }

    void dispenseSelectedProduct() {
        Product product = inventory.get(selectedProductCode);
        if (product == null || !product.isAvailable()) {
            throw new IllegalStateException("Cannot dispense: product unavailable");
        }
        System.out.println("[VM] Dispensing: " + product.getName());
        product.decrementQuantity();

        int change = insertedAmountCents - product.getPriceInCents();
        System.out.println("[VM] Change returned: " + change + " cents");

        refundAndReset();
    }

    // ---------------------------------------------------------------
    // Inventory management
    // ---------------------------------------------------------------

    public void addProduct(Product product) {
        inventory.put(product.getCode(), product);
        System.out.println("[VM] Stocked: " + product);

        // If machine was out of stock, transition back to idle
        if (currentState == outOfStockState && !allProductsOutOfStock()) {
            setState(idleState);
        }
    }

    public void printInventory() {
        System.out.println("\n--- Inventory ---");
        inventory.values().forEach(System.out::println);
        System.out.println("-----------------");
    }

    public String getCurrentStateName() {
        return currentState.getName();
    }
}
```

### IdleState.java (Vending Machine)

```java
package vendingmachine.state;

import vendingmachine.VendingMachine;
import vendingmachine.VendingMachineState;

public class IdleState implements VendingMachineState {

    private final VendingMachine vm;

    public IdleState(VendingMachine vm) { this.vm = vm; }

    @Override
    public void insertMoney(int cents) {
        if (cents <= 0) {
            System.out.println("[IDLE] Invalid amount. Please insert a positive amount.");
            return;
        }
        vm.addInsertedAmount(cents);
        vm.setState(vm.getHasMoneyState());
        System.out.println("[IDLE] Money inserted. Please select a product.");
    }

    @Override
    public void selectProduct(String productCode) {
        System.out.println("[IDLE] Please insert money before selecting a product.");
    }

    @Override
    public void dispense() {
        System.out.println("[IDLE] Please insert money and select a product first.");
    }

    @Override
    public void cancel() {
        System.out.println("[IDLE] Nothing to cancel.");
    }

    @Override
    public String getName() { return "IDLE"; }
}
```

### HasMoneyState.java (Vending Machine)

```java
package vendingmachine.state;

import vendingmachine.Product;
import vendingmachine.VendingMachine;
import vendingmachine.VendingMachineState;

public class HasMoneyState implements VendingMachineState {

    private final VendingMachine vm;

    public HasMoneyState(VendingMachine vm) { this.vm = vm; }

    @Override
    public void insertMoney(int cents) {
        if (cents <= 0) {
            System.out.println("[HAS_MONEY] Invalid amount.");
            return;
        }
        vm.addInsertedAmount(cents);
        System.out.println("[HAS_MONEY] Additional money accepted. Total: " + vm.getInsertedAmountCents() + " cents.");
    }

    @Override
    public void selectProduct(String productCode) {
        if (!vm.hasProduct(productCode)) {
            System.out.println("[HAS_MONEY] Product '" + productCode + "' is unavailable or out of stock.");
            return;
        }

        Product product = vm.getProduct(productCode);
        int inserted = vm.getInsertedAmountCents();

        if (inserted < product.getPriceInCents()) {
            System.out.println("[HAS_MONEY] Insufficient funds. "
                + product.getName() + " costs " + product.getPriceInCents() + " cents. "
                + "You have inserted " + inserted + " cents. "
                + "Please insert " + (product.getPriceInCents() - inserted) + " more cents.");
            return;
        }

        System.out.println("[HAS_MONEY] Product selected: " + product.getName()
            + " for " + product.getPriceInCents() + " cents.");
        vm.setSelectedProduct(productCode);
        vm.setState(vm.getProductSelectedState());
    }

    @Override
    public void dispense() {
        System.out.println("[HAS_MONEY] Please select a product first.");
    }

    @Override
    public void cancel() {
        int refund = vm.refundAndReset();
        System.out.println("[HAS_MONEY] Transaction cancelled. Refunding " + refund + " cents.");
        vm.setState(vm.getIdleState());
    }

    @Override
    public String getName() { return "HAS_MONEY"; }
}
```

### ProductSelectedState.java (Vending Machine)

```java
package vendingmachine.state;

import vendingmachine.VendingMachine;
import vendingmachine.VendingMachineState;

public class ProductSelectedState implements VendingMachineState {

    private final VendingMachine vm;

    public ProductSelectedState(VendingMachine vm) { this.vm = vm; }

    @Override
    public void insertMoney(int cents) {
        // Allow adding more money even after selection (user wants to buy multiple,
        // or wants to reconsider). In this simple model we just accumulate.
        vm.addInsertedAmount(cents);
        System.out.println("[PRODUCT_SELECTED] Additional money accepted.");
    }

    @Override
    public void selectProduct(String productCode) {
        // Allow changing product selection
        System.out.println("[PRODUCT_SELECTED] Changing product selection to: " + productCode);
        vm.setState(vm.getHasMoneyState());
        vm.setSelectedProduct(null);
        // Delegate the actual selection to HAS_MONEY state
        vm.selectProduct(productCode);
    }

    @Override
    public void dispense() {
        System.out.println("[PRODUCT_SELECTED] Dispensing product...");
        vm.setState(vm.getDispensingState());
        vm.dispenseSelectedProduct();

        // Check if all products are now out of stock
        if (vm.allProductsOutOfStock()) {
            System.out.println("[PRODUCT_SELECTED] All products out of stock. Machine going out of service.");
            vm.setState(vm.getOutOfStockState());
        } else {
            vm.setState(vm.getIdleState());
        }
    }

    @Override
    public void cancel() {
        int refund = vm.refundAndReset();
        System.out.println("[PRODUCT_SELECTED] Transaction cancelled. Refunding " + refund + " cents.");
        vm.setState(vm.getIdleState());
    }

    @Override
    public String getName() { return "PRODUCT_SELECTED"; }
}
```

### DispensingVMState.java and OutOfStockState.java

```java
package vendingmachine.state;

import vendingmachine.VendingMachine;
import vendingmachine.VendingMachineState;

public class DispensingVMState implements VendingMachineState {

    private final VendingMachine vm;

    public DispensingVMState(VendingMachine vm) { this.vm = vm; }

    @Override
    public void insertMoney(int cents) {
        System.out.println("[DISPENSING] Please wait — transaction in progress.");
    }

    @Override
    public void selectProduct(String productCode) {
        System.out.println("[DISPENSING] Please wait — transaction in progress.");
    }

    @Override
    public void dispense() {
        System.out.println("[DISPENSING] Already dispensing.");
    }

    @Override
    public void cancel() {
        System.out.println("[DISPENSING] Cannot cancel — dispense already in progress.");
    }

    @Override
    public String getName() { return "DISPENSING"; }
}
```

```java
package vendingmachine.state;

import vendingmachine.VendingMachine;
import vendingmachine.VendingMachineState;

public class OutOfStockState implements VendingMachineState {

    private final VendingMachine vm;

    public OutOfStockState(VendingMachine vm) { this.vm = vm; }

    @Override
    public void insertMoney(int cents) {
        System.out.println("[OUT_OF_STOCK] Machine is out of stock. Returning " + cents + " cents.");
    }

    @Override
    public void selectProduct(String productCode) {
        System.out.println("[OUT_OF_STOCK] All products are out of stock.");
    }

    @Override
    public void dispense() {
        System.out.println("[OUT_OF_STOCK] Nothing to dispense.");
    }

    @Override
    public void cancel() {
        // If any money was inserted before machine went out of stock, refund it
        int refund = vm.refundAndReset();
        if (refund > 0) {
            System.out.println("[OUT_OF_STOCK] Refunding " + refund + " cents.");
        } else {
            System.out.println("[OUT_OF_STOCK] Nothing to cancel.");
        }
    }

    @Override
    public String getName() { return "OUT_OF_STOCK"; }
}
```

### VendingMachineDemo.java

```java
package vendingmachine;

public class VendingMachineDemo {

    public static void main(String[] args) {
        VendingMachine vm = new VendingMachine();

        // Stock the machine
        vm.addProduct(new Product("A1", "Cola",    150, 2));
        vm.addProduct(new Product("A2", "Water",    75, 1));
        vm.addProduct(new Product("B1", "Chips",   200, 1));
        vm.printInventory();

        System.out.println("\n--- Happy Path ---");
        vm.insertMoney(100);
        vm.insertMoney(75);
        vm.selectProduct("A1"); // Cola at $1.50 — have $1.75
        vm.dispense();          // Dispense, return $0.25 change

        System.out.println("\n--- Insufficient Funds ---");
        vm.insertMoney(50);
        vm.selectProduct("B1"); // Chips at $2.00 — only have $0.50
        vm.insertMoney(150);    // Top up
        vm.selectProduct("B1"); // Now $2.00 — exact
        vm.dispense();

        System.out.println("\n--- Cancel Transaction ---");
        vm.insertMoney(200);
        vm.selectProduct("A1");
        vm.cancel();            // Refund $2.00, back to IDLE

        System.out.println("\n--- Last Item Purchase → Out of Stock ---");
        vm.insertMoney(75);
        vm.selectProduct("A2"); // Water at $0.75 — last one
        vm.dispense();          // After this, only Cola is left
        vm.printInventory();

        System.out.println("\n--- Out of Stock Behavior ---");
        // Deplete last item
        vm.insertMoney(200);
        vm.selectProduct("A1"); // Last cola
        vm.dispense();          // Machine now empty → OUT_OF_STOCK

        vm.insertMoney(100);    // Rejected gracefully
        System.out.println("Final state: " + vm.getCurrentStateName()); // OUT_OF_STOCK

        System.out.println("\n--- Restock ---");
        vm.addProduct(new Product("A1", "Cola", 150, 5));
        System.out.println("State after restock: " + vm.getCurrentStateName()); // IDLE
    }
}
```

---

## 7. State Transition Diagrams

### ATM State Transition Diagram

```
                    ┌─────────────────────────────────────┐
                    │           replenishCash()            │
                    │         (by technician)              │
                    ▼                                      │
         ┌──────────────────┐                    ┌──────────────────────┐
         │                  │                    │                      │
    ──►  │      IDLE        │                    │    OUT_OF_SERVICE    │
         │                  │                    │                      │
         └──────────────────┘                    └──────────────────────┘
                  │                                        ▲
                  │ insertCard(cardNumber)                  │
                  ▼                                         │
         ┌──────────────────┐                              │
         │                  │                              │
         │    HAS_CARD      │                              │
         │                  │                              │
         └──────────────────┘                              │
           │          │                                    │
           │          │ enterPin(pin) [valid]               │
           │          ▼                                    │
           │ ┌──────────────────┐                          │
           │ │                  │    dispenseCash()         │
           │ │  AUTHENTICATED   │──── (cash == 0) ─────────┘
           │ │                  │
           │ └──────────────────┘
           │          │
           │          │ requestCash(amount)
           │          ▼
           │ ┌──────────────────┐   dispense complete    ┌──────────────┐
           │ │                  │ ─────────────────────► │   back to    │
           │ │   DISPENSING     │                        │     IDLE     │
           │ │                  │ ─────────────────────► │              │
           │ └──────────────────┘   hardwareFault()      └──────────────┘
           │                             │
           │ ejectCard()                 ▼
           └──────────────────► OUT_OF_SERVICE
              [any state]

  PIN failure × 3:  HAS_CARD ──────────────────────────────► IDLE
  ejectCard():      HAS_CARD / AUTHENTICATED ───────────────► IDLE
```

### Vending Machine State Transition Diagram

```
                        ┌──────────────────────────────────┐
                        │         addProduct()             │
                        │    (by technician / restocking)  │
                        ▼                                  │
              ┌──────────────────┐              ┌──────────────────────┐
         ──►  │      IDLE        │              │    OUT_OF_STOCK      │
              └──────────────────┘              └──────────────────────┘
                       │                                   ▲
                       │ insertMoney(cents)                │
                       ▼                                   │
              ┌──────────────────┐                         │
              │                  │  cancel()               │
              │   HAS_MONEY      │ ────────────────────────┤  (refund)
              │                  │                         │
              └──────────────────┘                         │
                       │                                   │
                       │ selectProduct(code)               │
                       │ [amount >= price]                 │
                       ▼                                   │
              ┌──────────────────────┐                     │
              │                      │  cancel()           │
              │  PRODUCT_SELECTED    │ ────────────────────┤  (refund)
              │                      │                     │
              └──────────────────────┘                     │
                       │                                   │
                       │ dispense()                        │
                       ▼                                   │
              ┌──────────────────┐                         │
              │                  │  complete (stock left)  │
              │   DISPENSING     │ ──────────────────────► IDLE
              │                  │  complete (no stock)
              └──────────────────┘ ──────────────────────► OUT_OF_STOCK ──┘

  insertMoney() in HAS_MONEY: accumulates (stays in HAS_MONEY)
  selectProduct() in PRODUCT_SELECTED: allows product change → back to HAS_MONEY
```

---

## 8. Finite State Machine (FSM) Design Principles

The State design pattern is an object-oriented implementation of a **Finite State Machine (FSM)**. Understanding FSM theory helps you design state patterns correctly.

### Formal FSM Definition

An FSM is a 5-tuple: **M = (Q, Σ, δ, q₀, F)**

| Symbol | Meaning | ATM Example |
|---|---|---|
| Q | Finite set of states | {IDLE, HAS_CARD, AUTHENTICATED, DISPENSING, OUT_OF_SERVICE} |
| Σ | Input alphabet (events) | {insertCard, enterPin, requestCash, ejectCard} |
| δ | Transition function δ: Q × Σ → Q | e.g., δ(IDLE, insertCard) = HAS_CARD |
| q₀ | Initial state | IDLE |
| F | Set of accepting states | {IDLE} (transaction complete) |

### Key Design Decisions When Implementing an FSM

**1. Where do transitions live — in states or in the context?**

- **State-owned transitions** (used in both examples above): Each state class calls `context.setState(context.getXxxState())`. Transitions are co-located with the behavior that triggers them. This is more cohesive but couples states to one another via the context.
- **Context-owned transitions**: The context has a `transitionTable` (a `Map<StateClass, Map<Event, StateClass>>`). States signal events back to the context, which drives the transition. This is more centralized and testable in isolation but requires an event enum.

**2. Accepted Events vs. Ignored Events vs. Error Events**

- Some inputs in some states are **accepted** (trigger a transition + action)
- Some are **silently ignored** (e.g., ejectCard in IDLE — no harm, no transition)
- Some are **errors** (throw `InvalidOperationException` — invalid user action)

Deciding which is which is a domain modeling decision, not a technical one.

**3. Entry and Exit Actions**

Many FSM implementations support `onEntry()` and `onExit()` hooks on states. Add these to the `ATMState` interface when you need to:
- Start/stop timers (e.g., session timeout in HAS_CARD)
- Log audit events consistently on every transition
- Animate UI elements

```java
public interface ATMState {
    // ... existing methods ...

    /**
     * Called once when entering this state.
     * Default: no-op (implementations override as needed).
     */
    default void onEntry() {}

    /**
     * Called once just before leaving this state.
     * Default: no-op.
     */
    default void onExit() {}
}
```

**4. Guard Conditions**

Some transitions should only occur when a condition is met. In the ATM, `requestCash()` only transitions to `DISPENSING` if `amount <= cashAvailable`. These are **guard conditions** — they are checked before the transition fires.

**5. History States**

Some FSMs need to remember the previous state so they can return to it after an interruption. For example, if an ATM goes into `MAINTENANCE` mode and then back online, it might need to resume from `AUTHENTICATED` if a session was interrupted. This is the **History State** concept from UML statecharts.

```java
// Context example with history
private ATMState historyState;

void setState(ATMState newState) {
    this.historyState = this.currentState; // Save history
    this.currentState = newState;
}

void returnToHistoryState() {
    if (historyState != null) setState(historyState);
}
```

**6. Hierarchical States (Statecharts)**

Complex systems use **composite states** — a super-state that contains sub-states. For example, `AUTHENTICATED` could contain sub-states `SELECTING_AMOUNT`, `CONFIRMING`, `AWAITING_DISPENSE`. UML statecharts support this directly. In Java, implement with composition: the `AuthenticatedState` itself maintains an inner state machine.

---

## 9. Real-World Framework Examples

### Spring State Machine

Spring State Machine is a production-grade FSM framework for Java/Spring Boot applications. It implements the State pattern at a framework level with full support for hierarchical states, entry/exit actions, guards, and persistence.

```java
// Spring State Machine configuration for the ATM
@Configuration
@EnableStateMachine
public class ATMStateMachineConfig
        extends StateMachineConfigurerAdapter<ATMStates, ATMEvents> {

    // State enum
    public enum ATMStates {
        IDLE, HAS_CARD, AUTHENTICATED, DISPENSING, OUT_OF_SERVICE
    }

    // Event enum (the "input alphabet" Σ)
    public enum ATMEvents {
        CARD_INSERTED, PIN_ACCEPTED, PIN_REJECTED, CASH_REQUESTED,
        CASH_DISPENSED, CARD_EJECTED, FAULT_DETECTED, CASH_REPLENISHED
    }

    @Override
    public void configure(StateMachineStateConfigurer<ATMStates, ATMEvents> states)
            throws Exception {
        states
            .withStates()
                .initial(ATMStates.IDLE)
                .state(ATMStates.HAS_CARD)
                .state(ATMStates.AUTHENTICATED)
                .state(ATMStates.DISPENSING)
                .end(ATMStates.OUT_OF_SERVICE);
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<ATMStates, ATMEvents> transitions)
            throws Exception {
        transitions
            .withExternal()
                .source(ATMStates.IDLE).target(ATMStates.HAS_CARD)
                .event(ATMEvents.CARD_INSERTED)
                .and()
            .withExternal()
                .source(ATMStates.HAS_CARD).target(ATMStates.AUTHENTICATED)
                .event(ATMEvents.PIN_ACCEPTED)
                .and()
            .withExternal()
                .source(ATMStates.HAS_CARD).target(ATMStates.IDLE)
                .event(ATMEvents.CARD_EJECTED)
                .and()
            .withExternal()
                .source(ATMStates.AUTHENTICATED).target(ATMStates.DISPENSING)
                .event(ATMEvents.CASH_REQUESTED)
                // Guard: only transition if amount <= available
                .guard(context -> {
                    int amount = (int) context.getExtendedState()
                                              .getVariables()
                                              .get("requestedAmount");
                    int available = (int) context.getExtendedState()
                                                 .getVariables()
                                                 .get("cashAvailable");
                    return amount <= available;
                })
                .and()
            .withExternal()
                .source(ATMStates.DISPENSING).target(ATMStates.IDLE)
                .event(ATMEvents.CASH_DISPENSED)
                .and()
            .withExternal()
                .source(ATMStates.DISPENSING).target(ATMStates.OUT_OF_SERVICE)
                .event(ATMEvents.FAULT_DETECTED);
    }
}
```

Benefits Spring State Machine provides over hand-rolled State pattern:
- Persistent state via `StateMachineRuntimePersister` (survives restarts)
- Built-in audit trail and history
- Visual state machine diagrams via Spring Shell
- Distributed state machine support (Hazelcast, Redis)
- Testing support with `StateMachineTestPlan`

### TCP Connection States

The TCP protocol's connection lifecycle is a textbook FSM. The Linux kernel implements it as a state machine:

```
CLOSED → LISTEN → SYN_RECEIVED → ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
                              ↗
CLOSED → SYN_SENT ───────────
```

Each TCP socket is a context object. The kernel's `tcp_rcv_state_process()` function implements the transition table — functionally equivalent to the State pattern, though implemented in C with a switch statement (a pragmatic choice for kernel code where object allocation overhead matters).

### Workflow Engines (Camunda / Activiti)

Workflow engines model business processes as FSMs:

- **Activities** (tasks, service calls) = actions within a state
- **Gateways** = guard conditions on transitions  
- **Events** (message, timer, signal) = input alphabet elements

Camunda's BPMN engine stores the current "token" position in the workflow graph — effectively the current state reference — and drives transitions based on events. This is the State pattern at enterprise scale, supporting thousands of concurrent state machine instances with persistence, compensation, and distributed coordination.

### Java Enum-Based State Pattern (Alternative Implementation)

For simple, closed-world FSMs where states will never be added by subclassing:

```java
public enum OrderState {

    PLACED {
        @Override public OrderState confirm() { return CONFIRMED; }
        @Override public OrderState cancel()  { return CANCELLED; }
        @Override public OrderState ship()    { throw new InvalidOperationException("PLACED", "ship"); }
    },
    CONFIRMED {
        @Override public OrderState confirm() { return this; } // idempotent
        @Override public OrderState cancel()  { return CANCELLED; }
        @Override public OrderState ship()    { return SHIPPED; }
    },
    SHIPPED {
        @Override public OrderState confirm() { throw new InvalidOperationException("SHIPPED", "confirm"); }
        @Override public OrderState cancel()  { throw new InvalidOperationException("SHIPPED", "cancel"); }
        @Override public OrderState ship()    { return this; }
    },
    CANCELLED {
        @Override public OrderState confirm() { throw new InvalidOperationException("CANCELLED", "confirm"); }
        @Override public OrderState cancel()  { return this; }
        @Override public OrderState ship()    { throw new InvalidOperationException("CANCELLED", "ship"); }
    };

    public abstract OrderState confirm();
    public abstract OrderState cancel();
    public abstract OrderState ship();
}
```

This is a compact form of the State pattern using Java enum's per-constant method bodies. It is closed to extension (can't add a state without modifying the enum) but very concise for stable, enumerable domains.

---

## 10. Trade-offs

### Pros

| Benefit | Explanation |
|---|---|
| **Single Responsibility** | Each state class handles exactly one state's behavior. Easy to understand in isolation. |
| **Open/Closed Principle** | Adding a new state = adding a new class + updating transitions. Existing states are untouched. |
| **Eliminates conditionals** | No switch/if-else soup. Logic is distributed to where it belongs. |
| **State as a first-class citizen** | States can hold their own data (e.g., retry counters). This is impossible with enum-based switches. |
| **Explicit transitions** | Every legal state change is documented in code. Illegal transitions throw exceptions. |
| **Testability** | Each state class can be unit tested in isolation by mocking the context. |
| **Audit/observability** | The `setState()` method is a natural hook for logging, metrics, and alerts. |

### Cons

| Cost | Explanation |
|---|---|
| **Class proliferation** | N states = N classes. For simple machines, this is over-engineering. |
| **Inter-class coupling** | State classes reference each other's types via the context's factory methods. A state must know what states it can transition to. |
| **Context complexity** | The context must expose enough of its internals for state classes to operate. This often means package-private methods that feel like violations of encapsulation. |
| **Distributed logic** | Transition logic is spread across all state classes. Understanding the full machine requires reading all of them. A transition table (even a comment) can mitigate this. |
| **Stale state references** | If state instances are pre-created as Flyweights (as in both examples), they must not hold per-session mutable data — that data must live in the context. This is a subtle design constraint that trips up new implementors. |

---

## 11. Common Interview Discussion Points

### 1. State vs. Strategy: The Core Distinction

The most common interview follow-up. Both patterns use a family of interchangeable objects behind an interface. The critical differences:

- **Strategy objects are independent** — they do not know about each other. A `SortStrategy` doesn't know that `QuickSort` exists.
- **State objects know about each other** — `HasCardState` transitions to `AuthenticatedState`. States form a graph.
- **Strategy is chosen by the client** at construction or configuration time. `Context ctx = new Context(new ConcreteStrategy())`.
- **State transitions itself** from inside. The client just calls `context.doOperation()`.
- **Strategy doesn't change the context object's identity** — it configures it. **State changes the context's apparent behavior** — it transforms it.

### 2. Who Owns the Transitions?

A key design decision: should transitions be driven by the state classes (decentralized) or by a transition table in the context (centralized)?

- **Decentralized** (used in both examples): States call `context.setState(...)`. Transitions are co-located with the behavior that triggers them. More cohesive, easier to reason about locally.
- **Centralized** (Spring State Machine approach): The context has an event dispatch loop. States raise events; the context looks up the target state in a table. Better for visualization, testing, and persistence. Required for hierarchical/parallel states.

### 3. State Instance Reuse (Flyweight Integration)

In both implementations above, state instances are created once and reused. This is important for two reasons:
1. **No allocation on state transitions** — important in high-frequency systems
2. **Flyweight correctness** — because states are reused, they must be **stateless** (per-session data lives in the context). This is easy to violate accidentally and worth calling out.

### 4. Thread Safety in State Machines

In concurrent systems, state transitions must be atomic. Two critical scenarios:
- **TOCTOU (Time-of-Check-Time-of-Use)**: Thread A checks `if (state == AUTHENTICATED)`, then Thread B ejects the card, then Thread A transitions to DISPENSING based on stale check.
- **Solution**: Guard `setState()` with a `ReentrantLock` or `synchronized`. Or use `AtomicReference<ATMState>` with compare-and-swap for lock-free transitions.

### 5. The State Pattern in Production Systems

Interviewers like to hear real examples:
- **Order management systems**: Orders flow through PLACED → CONFIRMED → FULFILLING → SHIPPED → DELIVERED → CLOSED
- **Payment processing**: INITIATED → PROCESSING → AUTHORIZED → CAPTURED → SETTLED (or FAILED/REFUNDED at any point)
- **Game AI**: NPC behaviors: IDLE → PATROLLING → ALERTED → CHASING → ATTACKING → FLEEING
- **CI/CD pipelines**: QUEUED → RUNNING → PASSING/FAILING → SUCCEEDED/FAILED

---

## 12. Interview Questions and Model Answers

---

**Q1. What is the State design pattern and what problem does it solve?**

**Model Answer:**

The State pattern allows an object to change its behavior when its internal state changes, making it appear to change class. It solves the **conditional explosion problem** in stateful systems.

Without the pattern, a class whose behavior depends on state grows a switch or if-else chain in every method. If the class has M methods and N states, the code has O(M × N) conditional branches all scattered throughout the class. Adding a new state means editing every method. Adding a new method means writing a full switch covering every state. This violates the Open/Closed Principle and the Single Responsibility Principle.

The State pattern instead creates one class per state, each implementing a common interface. Each class handles exactly one state's behavior. Adding a new state adds one class and updates only the states that can transition to it. Adding a new operation adds one method to the interface and one implementation per state class — a linear change, not quadratic.

The fundamental insight is that state-dependent behavior is really a matrix of (state, operation) pairs. The pattern converts rows of that matrix (per-operation switch blocks) into columns (per-state classes), which aligns with how objects are organized: one object = one state = one column.

---

**Q2. How is the State pattern different from the Strategy pattern?**

**Model Answer:**

Both patterns use a family of interchangeable algorithm objects behind a common interface, and from a code structure perspective they look identical. The differences are semantic and behavioral:

**Lifecycle and ownership of transitions:**
- In Strategy, the *client* chooses and injects the strategy. A `Sorter` is constructed with `new Sorter(new QuickSortStrategy())`. The strategy does not change itself.
- In State, the *state object itself* (or the context) drives transitions. A state calls `context.setState(context.getNextState())`. The client just calls `context.doOperation()`.

**Awareness between siblings:**
- Strategy objects are completely independent — `QuickSort` does not know `MergeSort` exists.
- State objects form a directed graph — `HasCardState` must know about `AuthenticatedState` and `IdleState` to transition to them.

**Semantic intent:**
- Strategy captures a *choice* — which algorithm to use for this operation.
- State captures a *lifecycle* — what phase is the object in, and what is legal in that phase.

**Rule of thumb:** If the object's strategy never changes during its lifetime, it's a Strategy. If the object evolves through a sequence of states with different behaviors and legal operations, it's a State machine.

---

**Q3. In your ATM implementation, state instances are pre-created and reused. What are the implications of this design?**

**Model Answer:**

Pre-creating state instances is a **Flyweight optimization** on top of the State pattern. The implications are:

**Memory and performance benefits:**
- No object allocation on each state transition — important in high-throughput systems (millions of ATM transactions per day)
- No GC pressure from short-lived state objects
- States can be compared by reference (`currentState == idleState`) rather than by type, which is faster

**Critical design constraint — states must be stateless:**
- Because state instances are shared across all "sessions" (there is only one `IdleState` instance for the ATM), they cannot hold any per-session mutable data.
- Any data that varies per session (the inserted card number, the authenticated account ID, the PIN attempt count) must live in the context (`ATMMachine`), not in the state.
- Violating this leads to subtle bugs where session data from one session leaks into another.

**When NOT to use flyweight states:**
- If different state instances need different initial configurations (e.g., a retry limit that differs by card type), pre-created states cannot accommodate this. You would instead create a new state instance per session and pass configuration in the constructor.

In practice, the flyweight approach is correct for most state machines because states represent *roles* the context plays, not per-session data holders.

---

**Q4. How would you make the ATM state machine thread-safe?**

**Model Answer:**

The core concurrency risk in any state machine is the **TOCTOU (Time-of-Check-Time-of-Use)** race condition. In the ATM:

- Thread A reads `currentState == AUTHENTICATED` and decides to dispense $100
- Thread B calls `ejectCard()` and transitions to IDLE, wiping session data
- Thread A calls `dispenseCash()` on stale state — cash is dispensed with no authenticated user

**Solution 1 — Synchronized transitions with ReentrantLock:**

```java
private final ReentrantLock stateLock = new ReentrantLock();

public void requestCash(int amount) {
    stateLock.lock();
    try {
        currentState.requestCash(amount); // read + transition is atomic
    } finally {
        stateLock.unlock();
    }
}
```

All public methods acquire the lock. Since `requestCash()` calls `setState()` inside the lock, the entire check-and-transition is atomic.

**Solution 2 — AtomicReference for lock-free CAS transitions:**

```java
private final AtomicReference<ATMState> currentState = new AtomicReference<>(idleState);

// In HasCardState.enterPin():
boolean transitioned = atm.compareAndSetState(hasCardState, authenticatedState);
if (!transitioned) {
    // Another thread already changed the state — handle gracefully
}
```

CAS-based transitions are non-blocking but require careful handling of failed CAS operations.

**Solution 3 — Single-threaded event queue:**

Route all state machine events through a `LinkedBlockingQueue`. A single dedicated thread consumes the queue and drives the state machine. No locking needed because only one thread touches the state machine. This is the actor model approach (used by Akka, Vert.x). It is the cleanest solution for high-throughput state machines.

In a real ATM, the event queue approach is ideal — the ATM has one card slot and one human user; true parallelism in user interactions is impossible. The queue simplifies reasoning dramatically.

---

**Q5. How would you add a new "MAINTENANCE" state to the ATM without modifying existing state classes?**

**Model Answer:**

This is the Open/Closed Principle test. With the State pattern, adding a new state is purely additive:

**Step 1 — Create `MaintenanceState.java`:**

```java
package atm.state;

import atm.ATMMachine;
import atm.ATMState;

public class MaintenanceState implements ATMState {

    private final ATMMachine atm;
    private final String technicianId;

    public MaintenanceState(ATMMachine atm, String technicianId) {
        this.atm = atm;
        this.technicianId = technicianId;
    }

    @Override
    public void insertCard(String cardNumber) {
        System.out.println("[MAINTENANCE] ATM is in maintenance mode. Please use another ATM.");
    }

    @Override
    public void enterPin(String pin) {
        System.out.println("[MAINTENANCE] Not available during maintenance.");
    }

    @Override
    public void requestCash(int amount) {
        System.out.println("[MAINTENANCE] Not available during maintenance.");
    }

    @Override
    public void ejectCard() {
        System.out.println("[MAINTENANCE] No card to eject during maintenance.");
    }

    public void completeMaintenance() {
        System.out.println("[MAINTENANCE] Maintenance complete by " + technicianId);
        atm.setState(atm.getIdleState());
    }

    @Override
    public String getName() { return "MAINTENANCE"; }
}
```

**Step 2 — Add factory method to `ATMMachine`:**

```java
public MaintenanceState enterMaintenanceMode(String technicianId) {
    MaintenanceState maintenanceState = new MaintenanceState(this, technicianId);
    setState(maintenanceState);
    return maintenanceState;
}
```

**What we did NOT modify:**
- `IdleState`, `HasCardState`, `AuthenticatedState`, `DispensingState`, `OutOfServiceState` — untouched
- The `ATMState` interface — untouched

The only modification to existing code is adding one factory method to `ATMMachine`, which is a reasonable evolution. If even that is unacceptable (strict OCP), the transition to MAINTENANCE can be driven by an admin API that directly calls `setState()`, keeping existing classes completely untouched.

This is the concrete benefit the State pattern provides over switch statements: adding the MAINTENANCE state to the anti-pattern would require editing every switch block in every method.

---

**Q6. What is the difference between a State machine's "internal" vs. "external" transitions?**

**Model Answer:**

This distinction comes from UML statecharts:

**External transition:** The FSM exits the current state (fires `onExit()`), moves to the target state (fires `onEntry()`), and executes any transition action. Even if the source and target states are the same (self-transition), both exit and entry hooks fire.

```
AUTHENTICATED --[requestCash / amount invalid]--> AUTHENTICATED
// fires AUTHENTICATED.onExit() then AUTHENTICATED.onEntry()
// use case: reset a timeout timer on any activity
```

**Internal transition:** The FSM remains in the current state, executing only the transition action without firing `onExit()` or `onEntry()`. Used for operations that don't change the "phase" of the object.

```
HAS_MONEY --[insertMoney / accumulate]--> (self, internal)
// stays in HAS_MONEY, does NOT fire onExit/onEntry
// use case: accumulating additional money doesn't restart any timers
```

In a hand-rolled State implementation, internal transitions are implicitly what happens when a state method executes logic without calling `setState()`. External self-transitions would need to explicitly call `setState(currentState)`, which then triggers the logging/hook logic in `setState()`.

This distinction matters when you have entry/exit actions (e.g., starting a session timeout timer in `HAS_CARD.onEntry()`, stopping it in `HAS_CARD.onExit()`). An internal transition would not restart the timer; an external self-transition would.

---

**Q7. When would you choose a lookup-table (transition table) approach over the OOP State pattern?**

**Model Answer:**

The OOP State pattern and the transition table approach represent two ends of a spectrum. The right choice depends on the domain:

**Use OOP State pattern when:**
- Each state's behavior is complex and involves significant business logic
- States need to hold per-session or per-state data (e.g., retry counters)
- The FSM is embedded in an OOP system where behavior feels natural as methods
- You need polymorphic dispatch — different state classes from a library can be plugged in

**Use a transition table when:**
- The FSM has many states and few operations, and most behavior is simple (just transition and log)
- You need runtime mutability — users or configuration can add/modify transitions
- You need to serialize/deserialize the machine's definition (tables are data; classes are code)
- You're building a rule engine, workflow engine, or BPMN interpreter where the transitions ARE the product

**Use a hybrid (Spring State Machine / Camunda) when:**
- You need persistence, distributed state, parallel regions, or hierarchical states
- Visualization and tooling are required (business analysts need to read the machine)
- Compliance requires an auditable, externally-verifiable state machine definition

**Example transition table for the ATM:**

```java
Map<Class<? extends ATMState>, Map<String, Class<? extends ATMState>>> transitionTable = Map.of(
    IdleState.class,          Map.of("insertCard",   HasCardState.class),
    HasCardState.class,       Map.of("enterPin",     AuthenticatedState.class,
                                     "ejectCard",    IdleState.class),
    AuthenticatedState.class, Map.of("requestCash",  DispensingState.class,
                                     "ejectCard",    IdleState.class),
    DispensingState.class,    Map.of("complete",     IdleState.class,
                                     "fault",        OutOfServiceState.class)
);
```

This table can be read from a database, loaded from YAML, or modified at runtime — capabilities no class-based FSM can match. The trade-off is that behavior (what happens during a state) must still live somewhere, typically as callbacks/lambdas associated with each state in the table.

---

**Q8. In the vending machine, why does `ProductSelectedState.selectProduct()` not directly implement the re-selection logic, and instead delegates back to `HasMoneyState`?**

**Model Answer:**

This reveals an important principle: **states should not duplicate each other's logic**.

The selection logic in `HasMoneyState.selectProduct()` handles several cases:
- Validates the product code exists
- Checks if it is in stock
- Validates that inserted amount >= product price
- Updates the selected product in context
- Transitions to `PRODUCT_SELECTED`

If `ProductSelectedState.selectProduct()` duplicated this logic, we would violate DRY. Any change to selection logic (e.g., adding a loyalty discount check) would need to be updated in two places.

Instead, `ProductSelectedState.selectProduct()` does the minimal state-appropriate work:
1. Clears the current selection (because re-selecting means the prior selection is void)
2. Transitions back to `HAS_MONEY` state
3. Calls `vm.selectProduct(productCode)` — which dispatches to `HasMoneyState.selectProduct()` because the machine is now in `HAS_MONEY` state

This is a demonstration of the **Law of Demeter applied to state machines**: states should not reach into each other's logic. Instead, they route through the context's public API, which dispatches to the current state.

There is a subtle risk here: if `setState()` followed by `vm.selectProduct()` could be interrupted (in a concurrent system), the intermediate `HAS_MONEY` state would briefly be visible to other threads. In a production system with the single-threaded event queue approach, this is not a problem. With locks, both operations need to be in the same critical section.

---

## 13. Comparison with Related Patterns

### State vs. Strategy

```
PROPERTY              │ STATE                        │ STRATEGY
──────────────────────┼──────────────────────────────┼──────────────────────────────
Primary intent        │ Alter behavior as object      │ Select algorithm at runtime
                      │ progresses through a lifecycle│
──────────────────────┼──────────────────────────────┼──────────────────────────────
Who drives change?    │ The state object itself       │ The client/context constructor
                      │ (via context.setState())      │
──────────────────────┼──────────────────────────────┼──────────────────────────────
Sibling awareness     │ States know each other        │ Strategies are independent
                      │ (form a transition graph)     │
──────────────────────┼──────────────────────────────┼──────────────────────────────
Number of strategies  │ Usually fixed (closed set)    │ Extensible, open set
                      │                               │
──────────────────────┼──────────────────────────────┼──────────────────────────────
Context data          │ Significant — context holds   │ Minimal — strategy is usually
                      │ session state; states are     │ stateless or holds config
                      │ usually flyweights            │
──────────────────────┼──────────────────────────────┼──────────────────────────────
Domain model          │ Lifecycle / workflow          │ Pluggable algorithm / policy
──────────────────────┼──────────────────────────────┼──────────────────────────────
Example               │ ATM, TCP connection,          │ Sorting algorithm, payment
                      │ order lifecycle               │ gateway, compression codec
```

### State vs. Chain of Responsibility

```
PROPERTY              │ STATE                        │ CHAIN OF RESPONSIBILITY
──────────────────────┼──────────────────────────────┼──────────────────────────────
Purpose               │ Object changes behavior       │ Request passes along a chain
                      │ based on internal phase       │ of handlers until handled
──────────────────────┼──────────────────────────────┼──────────────────────────────
Structure             │ One active state at a time    │ Linear chain of handlers
──────────────────────┼──────────────────────────────┼──────────────────────────────
Request routing       │ Context delegates to current  │ Handler either handles or
                      │ state only                    │ passes to next in chain
──────────────────────┼──────────────────────────────┼──────────────────────────────
Handler knowledge     │ State knows sibling states    │ Handler knows only its
                      │ (via context factory methods) │ successor
──────────────────────┼──────────────────────────────┼──────────────────────────────
State of object       │ Changes fundamentally between │ Object doesn't change state
                      │ states — different legal ops  │ as request traverses chain
──────────────────────┼──────────────────────────────┼──────────────────────────────
Example               │ ATM in IDLE vs AUTHENTICATED  │ HTTP middleware pipeline,
                      │ behaves completely differently │ logging filter chain
```

### State vs. Command

- **Command** encapsulates a request as an object for queuing, logging, or undoing
- **State** encapsulates state-dependent behavior
- They combine naturally: a `Command` object is queued and dispatched to the current `State`'s handler. This is the event-driven state machine pattern used by Spring State Machine and Akka FSM.

### State vs. Observer

- **Observer** broadcasts change notifications to multiple interested parties
- **State** changes the behavior of a single object
- They combine naturally: the context fires `stateChanged(oldState, newState)` events that observers (UI, audit log, monitoring system) can subscribe to, without the state classes needing to know about the observers.

```java
// Combining State + Observer in the context
void setState(ATMState newState) {
    ATMState oldState = this.currentState;
    this.currentState = newState;

    // Observer notification — states don't know about this
    stateChangeListeners.forEach(l -> l.onStateChange(oldState, newState));
}
```

---

## Summary

The State design pattern converts a stateful object's conditional explosion into a clean family of collaborating objects, one per state. It excels when:

1. An object has non-trivial state-dependent behavior (3+ states)
2. New states will be added over the object's lifetime
3. Each state's behavior is complex enough to deserve encapsulation

The key implementation decisions are:
- **Where transitions live** (in states vs. in a centralized table)
- **Whether state instances are flyweights** (reused, stateless) or per-session (rich, stateful)
- **How invalid operations are handled** (exception vs. silent ignore vs. log-and-ignore)
- **How thread safety is achieved** (lock, CAS, or single-threaded event queue)

The pattern is not free — it adds N classes for N states and couples them via the context's factory methods. For simple machines with 2 states, a boolean field is sufficient. For complex, evolving lifecycles with business-critical correctness requirements, the State pattern pays for itself within the first change request.
