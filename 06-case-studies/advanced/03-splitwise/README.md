# Splitwise — Advanced LLD Case Study

---

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

Splitwise is an expense-splitting application that allows groups of people (friends, roommates, colleagues) to record shared expenses and track who owes whom. The core pain point it solves: when a group of people share costs over time, tracking individual debts manually becomes unwieldy. Splitwise maintains a running ledger and at any point can tell you exactly what each person owes each other person.

The interesting engineering challenges are:

- **Split semantics**: An expense can be divided in four fundamentally different ways (equal, exact dollar amounts, percentage of total, or weighted shares), each with its own validation rules.
- **Debt graph minimisation**: Instead of tracking every individual transaction, the system must compute the minimum number of settlements required to clear all debts — a classic graph problem.
- **Consistency**: Adding an expense atomically updates all affected balances.
- **Observability**: Users must be notified when expenses are added or settled.

---

## 2. Functional Requirements

1. A user can register with a name and email address.
2. Users can add other users as friends.
3. Users can create named groups and add members to groups.
4. A user can create an expense specifying: total amount, description, the user who paid, and the list of users among whom the expense is split.
5. The system supports four split types:
   - **Equal** — amount divided equally among all participants.
   - **Exact** — each participant's share is specified as a fixed dollar amount; shares must sum to the total.
   - **Percentage** — each participant's share is specified as a percentage; percentages must sum to 100.
   - **Shares** — each participant is assigned a weight; the payer distributes the total proportionally to each weight.
6. An expense can be attached to a group or recorded between individuals (no group required).
7. The system computes and updates running balances after every expense or settlement.
8. A user can view their balance with every other user they have a financial relationship with.
9. A user can record a settlement (a direct cash payment from one user to another).
10. The system can compute a simplified debt graph for a group — producing the minimum number of transactions required to settle all debts.
11. A user can view the full history of expenses for a group or for a 1-on-1 relationship.
12. Participants are notified when an expense that involves them is created.
13. Participants are notified when a settlement involving them is recorded.

---

## 3. Non-Functional Requirements

- **Correctness**: Every split strategy must validate its inputs and throw a descriptive exception on violation. Balances must remain consistent after every write.
- **Extensibility**: Adding a new split type must require adding one class only — no changes to existing classes (Open/Closed Principle).
- **Testability**: The split strategies and debt simplifier must be independently unit-testable with no framework dependencies.
- **Thread safety**: The service layer must handle concurrent expense additions without corrupting balance state.
- **Auditability**: Every expense and settlement is immutable once recorded; the history is append-only.
- **Precision**: Monetary arithmetic uses `BigDecimal` throughout — no `double` or `float` for money.

---

## 4. Assumptions

- Currency is single and homogeneous (no multi-currency support in this design).
- All amounts are non-negative.
- A user cannot owe themselves money — the payer is automatically excluded from owing their own share.
- Percentages are represented as values in the range (0, 100] and must sum to exactly 100 (within a configurable epsilon).
- The debt simplification algorithm operates on a snapshot of balances at request time; it is advisory, not authoritative — actual settlements are recorded independently.
- Group membership is flat (no sub-groups).
- Expense amounts are stored and compared with a precision of 2 decimal places (cents).
- Notification delivery is best-effort (fire-and-forget); a failed notification does not roll back an expense.
- User IDs and group IDs are UUIDs generated at creation time.

---

## 5. Domain Model

```
User
  - id: UUID
  - name: String
  - email: String
  - friends: Set<User>

Group
  - id: UUID
  - name: String
  - members: List<User>
  - expenses: List<Expense>

Expense
  - id: UUID
  - description: String
  - totalAmount: BigDecimal
  - paidBy: User
  - splits: List<ExpenseSplit>
  - splitType: SplitType
  - group: Group (nullable)
  - createdAt: LocalDateTime

ExpenseSplit
  - user: User
  - amount: BigDecimal        ← always the resolved dollar amount owed

Balance
  - owedBy: User             ← the debtor
  - owedTo: User             ← the creditor
  - amount: BigDecimal       ← always positive; direction encoded by owedBy/owedTo

Settlement
  - id: UUID
  - paidBy: User
  - paidTo: User
  - amount: BigDecimal
  - settledAt: LocalDateTime

BalanceSheet
  - balances: Map<UserPair, BigDecimal>
  Derived view; recomputed from expense + settlement history.

SplitStrategy (interface)
  - computeSplits(expense, participants, rawInputs) → List<ExpenseSplit>
  - validate(expense, participants, rawInputs)

DebtSimplifier
  - simplify(balances) → List<Transaction>
  Uses net-balance + two-heap greedy algorithm.
```

---

## 6. Class Diagram (ASCII UML)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SplitwiseService                            │
│  + registerUser()  + createGroup()  + addExpense()                  │
│  + recordSettlement()  + getBalances()  + simplifyDebts()           │
└────────────────────────────┬────────────────────────────────────────┘
                             │ uses
          ┌──────────────────┼──────────────────────┐
          ▼                  ▼                       ▼
   ┌─────────────┐   ┌──────────────┐      ┌────────────────┐
   │    User     │   │    Group     │      │  BalanceSheet  │
   │─────────────│   │──────────────│      │────────────────│
   │ id: UUID    │   │ id: UUID     │      │ update()       │
   │ name        │   │ name         │      │ getBalance()   │
   │ email       │   │ members      │      │ getAllBalances()│
   │ friends     │   │ expenses     │      └────────────────┘
   └─────────────┘   └──────────────┘
          │                  │
          │                  │ contains
          ▼                  ▼
   ┌─────────────────────────────────┐
   │             Expense             │
   │─────────────────────────────────│
   │ id, description, totalAmount    │
   │ paidBy: User                    │
   │ splits: List<ExpenseSplit>      │
   │ splitType: SplitType (enum)     │
   │ group: Group (nullable)         │
   │ createdAt: LocalDateTime        │
   └──────────────┬──────────────────┘
                  │ has-many
                  ▼
          ┌───────────────────┐
          │   ExpenseSplit    │
          │───────────────────│
          │ user: User        │
          │ amount: BigDecimal│
          └───────────────────┘

   ┌─────────────────────────────────────────┐
   │          <<interface>>                  │
   │           SplitStrategy                 │
   │─────────────────────────────────────────│
   │ + validate(SplitRequest): void          │
   │ + computeSplits(SplitRequest):          │
   │       List<ExpenseSplit>                │
   └──────┬──────────┬────────────┬──────────┘
          │          │            │          │
          ▼          ▼            ▼          ▼
   ┌──────────┐ ┌─────────┐ ┌──────────┐ ┌────────┐
   │  Equal   │ │  Exact  │ │Percentage│ │Shares  │
   │ Strategy │ │Strategy │ │ Strategy │ │Strategy│
   └──────────┘ └─────────┘ └──────────┘ └────────┘

   ┌───────────────────────────────────┐
   │          DebtSimplifier           │
   │───────────────────────────────────│
   │ + simplify(Map<UserPair,BigDec>)  │
   │       : List<Transaction>         │
   │ - computeNetBalances()            │
   │ - greedySettle()                  │
   └───────────────────────────────────┘

   ┌──────────────────────────────────────┐
   │      <<interface>>                   │
   │      NotificationObserver            │
   │──────────────────────────────────────│
   │ + onExpenseAdded(Expense): void      │
   │ + onSettlementRecorded(Settlement)   │
   └──────────┬───────────────────────────┘
              │
    ┌─────────┴──────────┐
    ▼                    ▼
┌──────────────┐  ┌─────────────────────┐
│EmailNotifier │  │ PushNotifier        │
└──────────────┘  └─────────────────────┘
```

---

## 7. Complete Java Implementation

### 7.1 Enums and Constants

```java
package splitwise.model;

/**
 * Supported split types. Each value maps to a concrete SplitStrategy implementation.
 */
public enum SplitType {
    EQUAL,
    EXACT,
    PERCENTAGE,
    SHARES
}
```

```java
package splitwise.model;

import java.math.BigDecimal;

/**
 * Application-wide monetary constants.
 */
public final class MoneyConstants {

    private MoneyConstants() {}

    /** Scale used for all stored monetary values (cents precision). */
    public static final int SCALE = 2;

    /** Rounding mode for all monetary arithmetic. */
    public static final java.math.RoundingMode ROUNDING = java.math.RoundingMode.HALF_UP;

    /** Epsilon for floating-point percentage comparisons. */
    public static final BigDecimal PERCENTAGE_EPSILON = new BigDecimal("0.001");

    /** The value 100 as BigDecimal for percentage checks. */
    public static final BigDecimal HUNDRED = new BigDecimal("100");
}
```

### 7.2 Exception Classes

```java
package splitwise.exception;

/**
 * Thrown when a split request is invalid — e.g., percentages do not sum to 100,
 * or exact amounts do not sum to the expense total.
 */
public class InvalidSplitException extends RuntimeException {

    public InvalidSplitException(String message) {
        super(message);
    }

    public InvalidSplitException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

```java
package splitwise.exception;

/**
 * Thrown when a referenced entity (User, Group, Expense) is not found.
 */
public class EntityNotFoundException extends RuntimeException {

    public EntityNotFoundException(String entityType, String id) {
        super(entityType + " not found with id: " + id);
    }
}
```

```java
package splitwise.exception;

/**
 * Thrown when a business rule is violated — e.g., settling more than owed,
 * or adding a member who is already in the group.
 */
public class BusinessRuleViolationException extends RuntimeException {

    public BusinessRuleViolationException(String message) {
        super(message);
    }
}
```

### 7.3 Core Domain Models

```java
package splitwise.model;

import java.util.Collections;
import java.util.HashSet;
import java.util.Objects;
import java.util.Set;
import java.util.UUID;

/**
 * Represents a registered user in the system.
 * Immutable after construction except for the friends set (managed via service layer).
 */
public class User {

    private final String id;
    private final String name;
    private final String email;
    // Friends are stored by reference; the set is mutated only through SplitwiseService.
    private final Set<User> friends;

    public User(String name, String email) {
        this.id = UUID.randomUUID().toString();
        this.name = Objects.requireNonNull(name, "name must not be null");
        this.email = Objects.requireNonNull(email, "email must not be null");
        this.friends = new HashSet<>();
    }

    // Package-private mutator — only SplitwiseService calls this.
    void addFriend(User other) {
        friends.add(other);
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public Set<User> getFriends() { return Collections.unmodifiableSet(friends); }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        return id.equals(((User) o).id);
    }

    @Override
    public int hashCode() { return id.hashCode(); }

    @Override
    public String toString() {
        return "User{id='" + id + "', name='" + name + "'}";
    }
}
```

```java
package splitwise.model;

import java.math.BigDecimal;
import java.util.Objects;

/**
 * Represents one participant's computed share of an expense.
 * Always stored as the resolved dollar amount — regardless of the original split type.
 */
public class ExpenseSplit {

    private final User user;
    private final BigDecimal amount;

    public ExpenseSplit(User user, BigDecimal amount) {
        this.user = Objects.requireNonNull(user, "user must not be null");
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Split amount must be >= 0, got: " + amount);
        }
        this.amount = amount;
    }

    public User getUser() { return user; }
    public BigDecimal getAmount() { return amount; }

    @Override
    public String toString() {
        return "ExpenseSplit{user=" + user.getName() + ", amount=" + amount + "}";
    }
}
```

```java
package splitwise.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.UUID;

/**
 * An immutable record of a shared expense.
 * Once created, an Expense cannot be modified — amendments are new expenses with adjustments.
 */
public class Expense {

    private final String id;
    private final String description;
    private final BigDecimal totalAmount;
    private final User paidBy;
    private final List<ExpenseSplit> splits;
    private final SplitType splitType;
    private final Group group;           // null for non-group expenses
    private final LocalDateTime createdAt;

    public Expense(
            String description,
            BigDecimal totalAmount,
            User paidBy,
            List<ExpenseSplit> splits,
            SplitType splitType,
            Group group) {

        this.id = UUID.randomUUID().toString();
        this.description = Objects.requireNonNull(description, "description must not be null");
        this.paidBy = Objects.requireNonNull(paidBy, "paidBy must not be null");
        this.splits = Collections.unmodifiableList(
                Objects.requireNonNull(splits, "splits must not be null"));
        this.splitType = Objects.requireNonNull(splitType, "splitType must not be null");
        this.group = group;
        this.createdAt = LocalDateTime.now();

        if (totalAmount == null || totalAmount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("totalAmount must be positive, got: " + totalAmount);
        }
        this.totalAmount = totalAmount;
    }

    public String getId() { return id; }
    public String getDescription() { return description; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public User getPaidBy() { return paidBy; }
    public List<ExpenseSplit> getSplits() { return splits; }
    public SplitType getSplitType() { return splitType; }
    public Group getGroup() { return group; }
    public LocalDateTime getCreatedAt() { return createdAt; }

    @Override
    public String toString() {
        return "Expense{id='" + id + "', description='" + description +
               "', totalAmount=" + totalAmount + ", paidBy=" + paidBy.getName() + "}";
    }
}
```

```java
package splitwise.model;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.UUID;

/**
 * A named collection of users who share expenses together.
 * The expenses list is append-only and grows as expenses are added through the service layer.
 */
public class Group {

    private final String id;
    private final String name;
    private final List<User> members;
    private final List<Expense> expenses;
    private final LocalDateTime createdAt;

    public Group(String name, List<User> initialMembers) {
        this.id = UUID.randomUUID().toString();
        this.name = Objects.requireNonNull(name, "name must not be null");
        this.members = new ArrayList<>(Objects.requireNonNull(initialMembers, "initialMembers must not be null"));
        this.expenses = new ArrayList<>();
        this.createdAt = LocalDateTime.now();
    }

    // Package-private — mutated only via SplitwiseService
    void addMember(User user) {
        members.add(user);
    }

    // Package-private — mutated only via SplitwiseService
    void addExpense(Expense expense) {
        expenses.add(expense);
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public List<User> getMembers() { return Collections.unmodifiableList(members); }
    public List<Expense> getExpenses() { return Collections.unmodifiableList(expenses); }
    public LocalDateTime getCreatedAt() { return createdAt; }

    @Override
    public String toString() {
        return "Group{id='" + id + "', name='" + name + "', members=" + members.size() + "}";
    }
}
```

```java
package splitwise.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Objects;
import java.util.UUID;

/**
 * Immutable record of a cash settlement between two users.
 * Recording a settlement reduces the balance between paidBy and paidTo by amount.
 */
public class Settlement {

    private final String id;
    private final User paidBy;
    private final User paidTo;
    private final BigDecimal amount;
    private final LocalDateTime settledAt;

    public Settlement(User paidBy, User paidTo, BigDecimal amount) {
        this.id = UUID.randomUUID().toString();
        this.paidBy = Objects.requireNonNull(paidBy, "paidBy must not be null");
        this.paidTo = Objects.requireNonNull(paidTo, "paidTo must not be null");
        this.settledAt = LocalDateTime.now();

        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Settlement amount must be positive, got: " + amount);
        }
        this.amount = amount;
    }

    public String getId() { return id; }
    public User getPaidBy() { return paidBy; }
    public User getPaidTo() { return paidTo; }
    public BigDecimal getAmount() { return amount; }
    public LocalDateTime getSettledAt() { return settledAt; }

    @Override
    public String toString() {
        return "Settlement{" + paidBy.getName() + " → " + paidTo.getName() +
               " $" + amount + " at " + settledAt + "}";
    }
}
```

```java
package splitwise.model;

import java.util.Objects;

/**
 * An ordered pair of users used as a Map key for balance storage.
 * The canonical form always places the lexicographically smaller id first,
 * ensuring that (Alice, Bob) and (Bob, Alice) resolve to the same key.
 */
public final class UserPair {

    private final String firstId;
    private final String secondId;

    public UserPair(User a, User b) {
        Objects.requireNonNull(a, "a must not be null");
        Objects.requireNonNull(b, "b must not be null");
        if (a.getId().compareTo(b.getId()) <= 0) {
            this.firstId = a.getId();
            this.secondId = b.getId();
        } else {
            this.firstId = b.getId();
            this.secondId = a.getId();
        }
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof UserPair)) return false;
        UserPair that = (UserPair) o;
        return firstId.equals(that.firstId) && secondId.equals(that.secondId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(firstId, secondId);
    }

    @Override
    public String toString() {
        return "UserPair(" + firstId + ", " + secondId + ")";
    }
}
```

```java
package splitwise.model;

import java.math.BigDecimal;
import java.util.Objects;

/**
 * A single directed settlement recommendation produced by the DebtSimplifier.
 * Not persisted — it is a suggestion returned to the caller.
 */
public class Transaction {

    private final User from;
    private final User to;
    private final BigDecimal amount;

    public Transaction(User from, User to, BigDecimal amount) {
        this.from = Objects.requireNonNull(from);
        this.to = Objects.requireNonNull(to);
        this.amount = Objects.requireNonNull(amount);
    }

    public User getFrom() { return from; }
    public User getTo() { return to; }
    public BigDecimal getAmount() { return amount; }

    @Override
    public String toString() {
        return from.getName() + " → " + to.getName() + ": $" + amount;
    }
}
```

### 7.4 Split Strategy — Interface and Request Object

```java
package splitwise.strategy;

import splitwise.model.ExpenseSplit;
import splitwise.model.User;

import java.math.BigDecimal;
import java.util.List;
import java.util.Map;

/**
 * Encapsulates the algorithm for dividing an expense's total among participants.
 *
 * CONTRACT:
 *   1. validate() MUST be called before computeSplits() — it throws InvalidSplitException on bad input.
 *   2. computeSplits() MUST return exactly one ExpenseSplit per participant in the request.
 *   3. The sum of all returned ExpenseSplit amounts MUST equal the totalAmount (within rounding tolerance).
 *   4. The payer's own share, if any, is included in the splits — the caller decides how to interpret it.
 *
 * Strategy pattern: each implementing class is a self-contained algorithm. Adding a new split type
 * means adding one new class — no existing code changes.
 */
public interface SplitStrategy {

    /**
     * Validates the split request inputs.
     *
     * @param request the fully populated split request
     * @throws splitwise.exception.InvalidSplitException if inputs are invalid
     */
    void validate(SplitRequest request);

    /**
     * Computes per-participant shares given the validated request.
     *
     * @param request validated split request
     * @return list of ExpenseSplit, one per participant
     */
    List<ExpenseSplit> computeSplits(SplitRequest request);
}
```

```java
package splitwise.strategy;

import splitwise.model.User;

import java.math.BigDecimal;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Objects;

/**
 * Bundles all inputs required by a SplitStrategy into one value object.
 *
 * Fields:
 *   totalAmount   — the full expense amount to be split
 *   participants  — ordered list of users who share this expense (may include payer)
 *   exactAmounts  — used by EXACT split: userId → dollar amount
 *   percentages   — used by PERCENTAGE split: userId → percentage (0-100)
 *   shares        — used by SHARES split: userId → integer weight
 *
 * Fields irrelevant to the chosen strategy are ignored by that strategy's implementation.
 */
public class SplitRequest {

    private final BigDecimal totalAmount;
    private final List<User> participants;
    private final Map<String, BigDecimal> exactAmounts;
    private final Map<String, BigDecimal> percentages;
    private final Map<String, Integer> shares;

    private SplitRequest(Builder builder) {
        this.totalAmount = Objects.requireNonNull(builder.totalAmount, "totalAmount required");
        this.participants = Collections.unmodifiableList(
                Objects.requireNonNull(builder.participants, "participants required"));
        this.exactAmounts = builder.exactAmounts != null
                ? Collections.unmodifiableMap(builder.exactAmounts)
                : Collections.emptyMap();
        this.percentages = builder.percentages != null
                ? Collections.unmodifiableMap(builder.percentages)
                : Collections.emptyMap();
        this.shares = builder.shares != null
                ? Collections.unmodifiableMap(builder.shares)
                : Collections.emptyMap();
    }

    public BigDecimal getTotalAmount() { return totalAmount; }
    public List<User> getParticipants() { return participants; }
    public Map<String, BigDecimal> getExactAmounts() { return exactAmounts; }
    public Map<String, BigDecimal> getPercentages() { return percentages; }
    public Map<String, Integer> getShares() { return shares; }

    // ── Builder ─────────────────────────────────────────────────────────────

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private BigDecimal totalAmount;
        private List<User> participants;
        private Map<String, BigDecimal> exactAmounts;
        private Map<String, BigDecimal> percentages;
        private Map<String, Integer> shares;

        public Builder totalAmount(BigDecimal totalAmount) {
            this.totalAmount = totalAmount;
            return this;
        }

        public Builder participants(List<User> participants) {
            this.participants = participants;
            return this;
        }

        public Builder exactAmounts(Map<String, BigDecimal> exactAmounts) {
            this.exactAmounts = exactAmounts;
            return this;
        }

        public Builder percentages(Map<String, BigDecimal> percentages) {
            this.percentages = percentages;
            return this;
        }

        public Builder shares(Map<String, Integer> shares) {
            this.shares = shares;
            return this;
        }

        public SplitRequest build() { return new SplitRequest(this); }
    }
}
```

### 7.5 Split Strategy Implementations

```java
package splitwise.strategy;

import splitwise.exception.InvalidSplitException;
import splitwise.model.ExpenseSplit;
import splitwise.model.MoneyConstants;
import splitwise.model.User;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.ArrayList;
import java.util.List;

/**
 * EQUAL split strategy.
 *
 * Each participant pays (totalAmount / participantCount), with any rounding remainder
 * assigned to the first participant in the list (deterministic, auditable).
 *
 * Example:
 *   Total = $100, participants = [Alice, Bob, Charlie]
 *   Base share = $33.33
 *   Remainder = $100.00 - ($33.33 * 3) = $0.01
 *   Alice gets $33.34, Bob and Charlie get $33.33 each.
 *
 * Validation:
 *   - At least one participant must be provided.
 */
public class EqualSplitStrategy implements SplitStrategy {

    @Override
    public void validate(SplitRequest request) {
        if (request.getParticipants() == null || request.getParticipants().isEmpty()) {
            throw new InvalidSplitException("EQUAL split requires at least one participant.");
        }
    }

    @Override
    public List<ExpenseSplit> computeSplits(SplitRequest request) {
        validate(request);

        List<User> participants = request.getParticipants();
        int count = participants.size();
        BigDecimal total = request.getTotalAmount();

        // Integer-divide at cent level to avoid floating-point drift.
        BigDecimal base = total.divide(new BigDecimal(count), MoneyConstants.SCALE, MoneyConstants.ROUNDING);
        BigDecimal remainder = total.subtract(base.multiply(new BigDecimal(count)));

        List<ExpenseSplit> result = new ArrayList<>(count);
        for (int i = 0; i < count; i++) {
            BigDecimal share = (i == 0) ? base.add(remainder) : base;
            result.add(new ExpenseSplit(participants.get(i), share));
        }
        return result;
    }
}
```

```java
package splitwise.strategy;

import splitwise.exception.InvalidSplitException;
import splitwise.model.ExpenseSplit;
import splitwise.model.MoneyConstants;
import splitwise.model.User;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

/**
 * EXACT split strategy.
 *
 * Each participant's share is specified as a precise dollar amount.
 * The amounts must be provided for every participant and must sum to totalAmount.
 *
 * Example:
 *   Total = $100, participants = [Alice, Bob, Charlie]
 *   Inputs: Alice=$50, Bob=$30, Charlie=$20  → valid (sum=100)
 *   Inputs: Alice=$50, Bob=$30, Charlie=$25  → invalid (sum=105 ≠ 100)
 *
 * Validation:
 *   - An exact amount must be provided for every participant.
 *   - All amounts must be positive.
 *   - Amounts must sum exactly to totalAmount.
 */
public class ExactSplitStrategy implements SplitStrategy {

    @Override
    public void validate(SplitRequest request) {
        List<User> participants = request.getParticipants();
        if (participants == null || participants.isEmpty()) {
            throw new InvalidSplitException("EXACT split requires at least one participant.");
        }

        BigDecimal sum = BigDecimal.ZERO;
        for (User user : participants) {
            BigDecimal amount = request.getExactAmounts().get(user.getId());
            if (amount == null) {
                throw new InvalidSplitException(
                        "EXACT split: missing amount for user '" + user.getName() + "' (id=" + user.getId() + ").");
            }
            if (amount.compareTo(BigDecimal.ZERO) < 0) {
                throw new InvalidSplitException(
                        "EXACT split: amount for user '" + user.getName() + "' must be >= 0, got " + amount + ".");
            }
            sum = sum.add(amount);
        }

        if (sum.setScale(MoneyConstants.SCALE, MoneyConstants.ROUNDING)
                .compareTo(request.getTotalAmount().setScale(MoneyConstants.SCALE, MoneyConstants.ROUNDING)) != 0) {
            throw new InvalidSplitException(
                    "EXACT split: amounts sum to " + sum + " but totalAmount is " + request.getTotalAmount() + ".");
        }
    }

    @Override
    public List<ExpenseSplit> computeSplits(SplitRequest request) {
        validate(request);

        List<ExpenseSplit> result = new ArrayList<>(request.getParticipants().size());
        for (User user : request.getParticipants()) {
            BigDecimal amount = request.getExactAmounts().get(user.getId());
            result.add(new ExpenseSplit(user, amount.setScale(MoneyConstants.SCALE, MoneyConstants.ROUNDING)));
        }
        return result;
    }
}
```

```java
package splitwise.strategy;

import splitwise.exception.InvalidSplitException;
import splitwise.model.ExpenseSplit;
import splitwise.model.MoneyConstants;
import splitwise.model.User;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.ArrayList;
import java.util.List;

/**
 * PERCENTAGE split strategy.
 *
 * Each participant's share is expressed as a percentage of totalAmount.
 * Percentages must be in (0, 100] and must sum to exactly 100 (within epsilon).
 * Rounding remainder is assigned to the first participant.
 *
 * Example:
 *   Total = $200, participants = [Alice, Bob]
 *   Inputs: Alice=60%, Bob=40%
 *   Alice = $200 * 0.60 = $120.00, Bob = $200 * 0.40 = $80.00
 *
 * Validation:
 *   - A percentage must be provided for every participant.
 *   - Each percentage must be in (0, 100].
 *   - All percentages must sum to 100 ± PERCENTAGE_EPSILON.
 */
public class PercentageSplitStrategy implements SplitStrategy {

    @Override
    public void validate(SplitRequest request) {
        List<User> participants = request.getParticipants();
        if (participants == null || participants.isEmpty()) {
            throw new InvalidSplitException("PERCENTAGE split requires at least one participant.");
        }

        BigDecimal sum = BigDecimal.ZERO;
        for (User user : participants) {
            BigDecimal pct = request.getPercentages().get(user.getId());
            if (pct == null) {
                throw new InvalidSplitException(
                        "PERCENTAGE split: missing percentage for user '" + user.getName() + "'.");
            }
            if (pct.compareTo(BigDecimal.ZERO) <= 0 || pct.compareTo(MoneyConstants.HUNDRED) > 0) {
                throw new InvalidSplitException(
                        "PERCENTAGE split: percentage for '" + user.getName() +
                        "' must be in (0, 100], got " + pct + ".");
            }
            sum = sum.add(pct);
        }

        BigDecimal diff = sum.subtract(MoneyConstants.HUNDRED).abs();
        if (diff.compareTo(MoneyConstants.PERCENTAGE_EPSILON) > 0) {
            throw new InvalidSplitException(
                    "PERCENTAGE split: percentages sum to " + sum + ", must sum to 100.");
        }
    }

    @Override
    public List<ExpenseSplit> computeSplits(SplitRequest request) {
        validate(request);

        List<User> participants = request.getParticipants();
        BigDecimal total = request.getTotalAmount();
        List<ExpenseSplit> result = new ArrayList<>(participants.size());

        BigDecimal allocated = BigDecimal.ZERO;
        for (int i = 0; i < participants.size(); i++) {
            User user = participants.get(i);
            BigDecimal pct = request.getPercentages().get(user.getId());

            BigDecimal share;
            if (i == participants.size() - 1) {
                // Last participant gets whatever is left — absorbs rounding errors.
                share = total.subtract(allocated).setScale(MoneyConstants.SCALE, MoneyConstants.ROUNDING);
            } else {
                share = total.multiply(pct)
                             .divide(MoneyConstants.HUNDRED, MoneyConstants.SCALE, MoneyConstants.ROUNDING);
                allocated = allocated.add(share);
            }
            result.add(new ExpenseSplit(user, share));
        }
        return result;
    }
}
```

```java
package splitwise.strategy;

import splitwise.exception.InvalidSplitException;
import splitwise.model.ExpenseSplit;
import splitwise.model.MoneyConstants;
import splitwise.model.User;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

/**
 * SHARES split strategy.
 *
 * Each participant is assigned an integer weight (share count). The total amount is
 * distributed proportionally to the weights. This is useful for "Alice should pay twice
 * as much as Bob" scenarios without specifying exact amounts.
 *
 * Example:
 *   Total = $300, participants = [Alice=1, Bob=2, Charlie=3]
 *   Total shares = 6
 *   Alice = $300 * (1/6) = $50.00
 *   Bob   = $300 * (2/6) = $100.00
 *   Charlie = $300 * (3/6) = $150.00
 *
 * Rounding: Last participant absorbs any penny-level remainder.
 *
 * Validation:
 *   - A share count must be provided for every participant.
 *   - Each share count must be >= 1.
 */
public class SharesSplitStrategy implements SplitStrategy {

    @Override
    public void validate(SplitRequest request) {
        List<User> participants = request.getParticipants();
        if (participants == null || participants.isEmpty()) {
            throw new InvalidSplitException("SHARES split requires at least one participant.");
        }

        for (User user : participants) {
            Integer share = request.getShares().get(user.getId());
            if (share == null) {
                throw new InvalidSplitException(
                        "SHARES split: missing share count for user '" + user.getName() + "'.");
            }
            if (share < 1) {
                throw new InvalidSplitException(
                        "SHARES split: share count for '" + user.getName() + "' must be >= 1, got " + share + ".");
            }
        }
    }

    @Override
    public List<ExpenseSplit> computeSplits(SplitRequest request) {
        validate(request);

        List<User> participants = request.getParticipants();
        BigDecimal total = request.getTotalAmount();

        int totalShares = participants.stream()
                .mapToInt(u -> request.getShares().get(u.getId()))
                .sum();

        BigDecimal totalSharesBD = new BigDecimal(totalShares);
        List<ExpenseSplit> result = new ArrayList<>(participants.size());

        BigDecimal allocated = BigDecimal.ZERO;
        for (int i = 0; i < participants.size(); i++) {
            User user = participants.get(i);
            int userShares = request.getShares().get(user.getId());

            BigDecimal share;
            if (i == participants.size() - 1) {
                share = total.subtract(allocated).setScale(MoneyConstants.SCALE, MoneyConstants.ROUNDING);
            } else {
                share = total.multiply(new BigDecimal(userShares))
                             .divide(totalSharesBD, MoneyConstants.SCALE, MoneyConstants.ROUNDING);
                allocated = allocated.add(share);
            }
            result.add(new ExpenseSplit(user, share));
        }
        return result;
    }
}
```

### 7.6 Strategy Factory

```java
package splitwise.strategy;

import splitwise.model.SplitType;

import java.util.EnumMap;
import java.util.Map;

/**
 * Factory that maps SplitType → SplitStrategy.
 * Strategies are stateless singletons — safe to reuse across requests.
 *
 * To add a new split type:
 *   1. Add value to SplitType enum.
 *   2. Create a new SplitStrategy implementation.
 *   3. Register it here.
 *   That is the only change required — nothing else in the system needs to know.
 */
public class SplitStrategyFactory {

    private static final Map<SplitType, SplitStrategy> REGISTRY = new EnumMap<>(SplitType.class);

    static {
        REGISTRY.put(SplitType.EQUAL, new EqualSplitStrategy());
        REGISTRY.put(SplitType.EXACT, new ExactSplitStrategy());
        REGISTRY.put(SplitType.PERCENTAGE, new PercentageSplitStrategy());
        REGISTRY.put(SplitType.SHARES, new SharesSplitStrategy());
    }

    private SplitStrategyFactory() {}

    public static SplitStrategy getStrategy(SplitType type) {
        SplitStrategy strategy = REGISTRY.get(type);
        if (strategy == null) {
            throw new IllegalArgumentException("No strategy registered for split type: " + type);
        }
        return strategy;
    }
}
```

### 7.7 Balance Sheet

```java
package splitwise.service;

import splitwise.model.Expense;
import splitwise.model.ExpenseSplit;
import splitwise.model.MoneyConstants;
import splitwise.model.Settlement;
import splitwise.model.User;
import splitwise.model.UserPair;

import java.math.BigDecimal;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Maintains a running ledger of who owes whom.
 *
 * Internal storage:
 *   balances: Map&lt;UserPair, BigDecimal&gt;
 *
 * Interpretation convention:
 *   The value is ALWAYS positive. The sign is encoded externally by recording
 *   which user is the creditor and which is the debtor in a separate structure.
 *
 *   We use a simpler directional map:
 *     netBalance: Map&lt;String, Map&lt;String, BigDecimal&gt;&gt;
 *     netBalance.get(debtorId).get(creditorId) = amount debtor owes creditor
 *
 *   This is easier to reason about than a signed canonical pair map.
 *
 * Thread safety: all mutating methods are synchronized on this instance.
 */
public class BalanceSheet {

    // netBalance[debtor][creditor] = amount debtor owes creditor (always positive)
    private final Map<String, Map<String, BigDecimal>> netBalance = new ConcurrentHashMap<>();

    /**
     * Updates balances after a new expense is recorded.
     *
     * For each split:
     *   If split.user != payer → split.user owes payer split.amount
     */
    public synchronized void applyExpense(Expense expense) {
        User payer = expense.getPaidBy();
        for (ExpenseSplit split : expense.getSplits()) {
            User debtor = split.getUser();
            if (debtor.equals(payer)) {
                // Payer's own share — no debt created.
                continue;
            }
            addDebt(debtor, payer, split.getAmount());
        }
    }

    /**
     * Updates balances after a settlement is recorded.
     * paidBy reduces their debt to paidTo by amount.
     */
    public synchronized void applySettlement(Settlement settlement) {
        reduceDebt(settlement.getPaidBy(), settlement.getPaidTo(), settlement.getAmount());
    }

    /**
     * Returns how much userId1 owes userId2 (positive) or is owed by userId2 (negative).
     * Zero if no financial relationship exists.
     */
    public BigDecimal getBalance(String userId1, String userId2) {
        BigDecimal owes = getDebt(userId1, userId2);
        BigDecimal isOwed = getDebt(userId2, userId1);
        return owes.subtract(isOwed);
    }

    /**
     * Returns the complete ledger as an unmodifiable snapshot.
     * Key: debtor user id. Value: map of creditor id → amount owed.
     */
    public Map<String, Map<String, BigDecimal>> getNetBalance() {
        // Deep-copy to prevent external mutation
        Map<String, Map<String, BigDecimal>> copy = new HashMap<>();
        for (Map.Entry<String, Map<String, BigDecimal>> entry : netBalance.entrySet()) {
            copy.put(entry.getKey(), Collections.unmodifiableMap(new HashMap<>(entry.getValue())));
        }
        return Collections.unmodifiableMap(copy);
    }

    // ── Private helpers ──────────────────────────────────────────────────────

    private void addDebt(User debtor, User creditor, BigDecimal amount) {
        // Check if creditor already owes debtor — net off first.
        BigDecimal reverseDebt = getDebt(creditor, debtor);
        if (reverseDebt.compareTo(BigDecimal.ZERO) > 0) {
            if (reverseDebt.compareTo(amount) >= 0) {
                // Fully absorbed by reverse debt.
                setDebt(creditor, debtor, reverseDebt.subtract(amount));
                return;
            } else {
                // Partially absorbed — clear reverse, add remainder as new debt.
                setDebt(creditor, debtor, BigDecimal.ZERO);
                amount = amount.subtract(reverseDebt);
            }
        }
        BigDecimal existing = getDebt(debtor, creditor);
        setDebt(debtor, creditor, existing.add(amount));
    }

    private void reduceDebt(User debtor, User creditor, BigDecimal amount) {
        BigDecimal existing = getDebt(debtor, creditor);
        BigDecimal newBalance = existing.subtract(amount);
        if (newBalance.compareTo(BigDecimal.ZERO) < 0) {
            // Over-payment: flip direction — creditor now owes debtor the excess.
            setDebt(debtor, creditor, BigDecimal.ZERO);
            setDebt(creditor, debtor, newBalance.abs());
        } else {
            setDebt(debtor, creditor, newBalance);
        }
    }

    private BigDecimal getDebt(User debtor, User creditor) {
        return netBalance
                .getOrDefault(debtor.getId(), Collections.emptyMap())
                .getOrDefault(creditor.getId(), BigDecimal.ZERO);
    }

    private void setDebt(User debtor, User creditor, BigDecimal amount) {
        netBalance
                .computeIfAbsent(debtor.getId(), k -> new ConcurrentHashMap<>())
                .put(creditor.getId(), amount.setScale(MoneyConstants.SCALE, MoneyConstants.ROUNDING));
    }
}
```

### 7.8 Debt Simplifier

```java
package splitwise.service;

import splitwise.model.MoneyConstants;
import splitwise.model.Transaction;
import splitwise.model.User;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.PriorityQueue;

/**
 * Computes the minimum number of transactions required to settle all debts
 * within a group of users, using a greedy net-balance algorithm.
 *
 * ── Algorithm Overview ────────────────────────────────────────────────────────
 *
 * STEP 1 — Compute net balance per user:
 *   For each user, sum all amounts others owe them (credits) minus all amounts
 *   they owe others (debits).
 *
 *   net[u] > 0  → u is owed money overall (creditor)
 *   net[u] < 0  → u owes money overall (debtor)
 *   net[u] = 0  → u is settled
 *
 * STEP 2 — Greedy settlement with two heaps:
 *   - maxCreditors: max-heap of (user, net) for users with net > 0
 *   - maxDebtors:   max-heap of (user, |net|) for users with net < 0 (absolute value)
 *
 *   While both heaps are non-empty:
 *     1. Pop the largest creditor C and largest debtor D.
 *     2. amount = min(C.net, D.net)
 *     3. Record: D pays C amount.
 *     4. Reduce C.net and D.net by amount.
 *     5. If either has residual > 0, push back.
 *
 * ── Concrete Example ─────────────────────────────────────────────────────────
 *
 *   Users: Alice (A), Bob (B), Charlie (C), Dave (D)
 *
 *   Raw debts after expenses:
 *     B owes A: $30
 *     C owes A: $20
 *     C owes B: $10
 *     D owes B: $40
 *
 *   Net balances:
 *     A: +$30 +$20            = +$50  (creditor)
 *     B: -$30 +$10 +$40 -$10 = +$10  (creditor) [B owes A $30, is owed by C $10 and D $40]
 *        Actually: B receives from C $10 + D $40 = $50, owes A $30 → net = +$20
 *     Wait — let me recompute properly:
 *       A net: B owes A $30, C owes A $20 → A is owed $50 → net[A] = +50
 *       B net: C owes B $10, D owes B $40, B owes A $30 → net[B] = +10 - 30 = -20
 *         Actually B receives 10+40=50, owes 30 → net[B] = +20
 *       C net: owes A $20, owes B $10 → net[C] = -30
 *       D net: owes B $40 → net[D] = -40
 *
 *   Sum check: 50 + 20 - 30 - 40 = 0 ✓
 *
 *   Heaps:
 *     creditors (max-heap by net):  A(50), B(20)
 *     debtors   (max-heap by |net|): D(40), C(30)
 *
 *   Iteration 1: C=A(50), D=D(40), amount=40 → "D pays A $40"
 *     A residual = 10, D residual = 0 → push A(10) back
 *
 *   Iteration 2: C=B(20), D=C(30), amount=20 → "C pays B $20"
 *     B residual = 0, C residual = 10 → push C(10) back
 *
 *   Iteration 3: C=A(10), D=C(10), amount=10 → "C pays A $10"
 *     Both residual = 0 → done
 *
 *   Result: 3 transactions (vs 4 original debts).
 *
 * ── Complexity ────────────────────────────────────────────────────────────────
 *   Time:  O(n log n) where n = number of users with non-zero net balance.
 *          Each user is pushed/popped from a heap at most twice (once as debtor, once residual).
 *   Space: O(n) for the heaps.
 *
 *   Note: This greedy algorithm is optimal for minimising transaction count when
 *   transaction amounts are continuous (real numbers). For integer amounts it remains
 *   near-optimal but the NP-hard subset-sum variant applies only when we additionally
 *   want to reuse existing debt amounts exactly.
 */
public class DebtSimplifier {

    /**
     * Produces the minimum-transaction settlement plan for the given balance snapshot.
     *
     * @param users         all users in scope (must cover all ids in netBalance)
     * @param netBalance    snapshot from BalanceSheet.getNetBalance()
     *                      netBalance[debtorId][creditorId] = amount debtor owes creditor
     * @return ordered list of Transactions (from, to, amount)
     */
    public List<Transaction> simplify(
            List<User> users,
            Map<String, Map<String, BigDecimal>> netBalance) {

        // Build a userId → User lookup.
        Map<String, User> userMap = new HashMap<>();
        for (User u : users) {
            userMap.put(u.getId(), u);
        }

        // STEP 1: Compute net balance per user.
        Map<String, BigDecimal> net = computeNetBalances(userMap, netBalance);

        // STEP 2: Separate into creditors (net > 0) and debtors (net < 0).
        // Max-heap for creditors: highest positive net first.
        PriorityQueue<Map.Entry<String, BigDecimal>> creditorHeap = new PriorityQueue<>(
                Comparator.<Map.Entry<String, BigDecimal>, BigDecimal>
                        comparing(Map.Entry::getValue).reversed());

        // Max-heap for debtors: highest |debt| first (stored as positive values).
        PriorityQueue<Map.Entry<String, BigDecimal>> debtorHeap = new PriorityQueue<>(
                Comparator.<Map.Entry<String, BigDecimal>, BigDecimal>
                        comparing(Map.Entry::getValue).reversed());

        for (Map.Entry<String, BigDecimal> entry : net.entrySet()) {
            int cmp = entry.getValue().compareTo(BigDecimal.ZERO);
            if (cmp > 0) {
                creditorHeap.offer(Map.entry(entry.getKey(), entry.getValue()));
            } else if (cmp < 0) {
                // Store absolute value in heap.
                debtorHeap.offer(Map.entry(entry.getKey(), entry.getValue().negate()));
            }
            // Zero net → already settled, skip.
        }

        // STEP 3: Greedy settlement.
        List<Transaction> result = new ArrayList<>();

        while (!creditorHeap.isEmpty() && !debtorHeap.isEmpty()) {
            Map.Entry<String, BigDecimal> creditorEntry = creditorHeap.poll();
            Map.Entry<String, BigDecimal> debtorEntry = debtorHeap.poll();

            String creditorId = creditorEntry.getKey();
            String debtorId = debtorEntry.getKey();
            BigDecimal creditorNet = creditorEntry.getValue();
            BigDecimal debtorNet = debtorEntry.getValue(); // absolute value

            BigDecimal amount = creditorNet.min(debtorNet)
                                           .setScale(MoneyConstants.SCALE, MoneyConstants.ROUNDING);

            User from = userMap.get(debtorId);
            User to = userMap.get(creditorId);
            result.add(new Transaction(from, to, amount));

            BigDecimal creditorResidual = creditorNet.subtract(amount);
            BigDecimal debtorResidual = debtorNet.subtract(amount);

            if (creditorResidual.compareTo(BigDecimal.ZERO) > 0) {
                creditorHeap.offer(Map.entry(creditorId, creditorResidual));
            }
            if (debtorResidual.compareTo(BigDecimal.ZERO) > 0) {
                debtorHeap.offer(Map.entry(debtorId, debtorResidual));
            }
        }

        return result;
    }

    // ── Private helpers ──────────────────────────────────────────────────────

    /**
     * Collapses the debtor→creditor matrix into a single net value per user.
     * net[u] = (total owed to u by others) - (total u owes to others)
     */
    private Map<String, BigDecimal> computeNetBalances(
            Map<String, User> userMap,
            Map<String, Map<String, BigDecimal>> netBalance) {

        Map<String, BigDecimal> net = new HashMap<>();

        // Initialise all known users to zero.
        for (String uid : userMap.keySet()) {
            net.put(uid, BigDecimal.ZERO);
        }

        for (Map.Entry<String, Map<String, BigDecimal>> debtorEntry : netBalance.entrySet()) {
            String debtorId = debtorEntry.getKey();
            for (Map.Entry<String, BigDecimal> creditorEntry : debtorEntry.getValue().entrySet()) {
                String creditorId = creditorEntry.getKey();
                BigDecimal amount = creditorEntry.getValue();

                if (amount.compareTo(BigDecimal.ZERO) <= 0) continue;

                // Debtor's net decreases.
                net.merge(debtorId, amount.negate(), BigDecimal::add);
                // Creditor's net increases.
                net.merge(creditorId, amount, BigDecimal::add);
            }
        }

        return net;
    }
}
```

### 7.9 Notification Observer

```java
package splitwise.notification;

import splitwise.model.Expense;
import splitwise.model.Settlement;

/**
 * Observer interface for expense and settlement events.
 * Implementations handle delivery via different channels (email, push, SMS, etc.).
 *
 * The Observer pattern is used here because:
 *   - Multiple notification channels must be supported simultaneously.
 *   - The service layer should not depend on any specific notification technology.
 *   - Channels can be added/removed at runtime without changing the service.
 */
public interface NotificationObserver {

    /**
     * Called synchronously after an expense is successfully recorded.
     * Implementations should be fast and non-blocking; heavy work should be offloaded
     * to a background thread or message queue.
     */
    void onExpenseAdded(Expense expense);

    /**
     * Called synchronously after a settlement is successfully recorded.
     */
    void onSettlementRecorded(Settlement settlement);
}
```

```java
package splitwise.notification;

import splitwise.model.Expense;
import splitwise.model.ExpenseSplit;
import splitwise.model.Settlement;

import java.util.logging.Logger;

/**
 * Console/log-based notification implementation.
 * In production this would integrate with an SMTP service or email queue.
 */
public class EmailNotificationObserver implements NotificationObserver {

    private static final Logger log = Logger.getLogger(EmailNotificationObserver.class.getName());

    @Override
    public void onExpenseAdded(Expense expense) {
        for (ExpenseSplit split : expense.getSplits()) {
            if (split.getUser().equals(expense.getPaidBy())) continue;
            log.info(String.format(
                    "[EMAIL] To: %s <%s> — %s added expense '%s'. You owe $%s to %s.",
                    split.getUser().getName(),
                    split.getUser().getEmail(),
                    expense.getPaidBy().getName(),
                    expense.getDescription(),
                    split.getAmount(),
                    expense.getPaidBy().getName()));
        }
    }

    @Override
    public void onSettlementRecorded(Settlement settlement) {
        log.info(String.format(
                "[EMAIL] To: %s <%s> — %s has paid you $%s.",
                settlement.getPaidTo().getName(),
                settlement.getPaidTo().getEmail(),
                settlement.getPaidBy().getName(),
                settlement.getAmount()));
        log.info(String.format(
                "[EMAIL] To: %s <%s> — Your payment of $%s to %s has been recorded.",
                settlement.getPaidBy().getName(),
                settlement.getPaidBy().getEmail(),
                settlement.getAmount(),
                settlement.getPaidTo().getName()));
    }
}
```

```java
package splitwise.notification;

import splitwise.model.Expense;
import splitwise.model.ExpenseSplit;
import splitwise.model.Settlement;

import java.util.logging.Logger;

/**
 * Push notification stub implementation.
 * In production this would call FCM / APNs / a push gateway.
 */
public class PushNotificationObserver implements NotificationObserver {

    private static final Logger log = Logger.getLogger(PushNotificationObserver.class.getName());

    @Override
    public void onExpenseAdded(Expense expense) {
        for (ExpenseSplit split : expense.getSplits()) {
            log.info(String.format(
                    "[PUSH] → device(%s): New expense '%s' — you owe $%s",
                    split.getUser().getId(),
                    expense.getDescription(),
                    split.getAmount()));
        }
    }

    @Override
    public void onSettlementRecorded(Settlement settlement) {
        log.info(String.format(
                "[PUSH] → device(%s): %s paid you $%s",
                settlement.getPaidTo().getId(),
                settlement.getPaidBy().getName(),
                settlement.getAmount()));
    }
}
```

### 7.10 SplitwiseService — Facade / Orchestrator

```java
package splitwise.service;

import splitwise.exception.BusinessRuleViolationException;
import splitwise.exception.EntityNotFoundException;
import splitwise.exception.InvalidSplitException;
import splitwise.model.Expense;
import splitwise.model.ExpenseSplit;
import splitwise.model.Group;
import splitwise.model.Settlement;
import splitwise.model.SplitType;
import splitwise.model.Transaction;
import splitwise.model.User;
import splitwise.notification.NotificationObserver;
import splitwise.strategy.SplitRequest;
import splitwise.strategy.SplitStrategy;
import splitwise.strategy.SplitStrategyFactory;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Central service and entry point for all Splitwise operations.
 *
 * Responsibilities:
 *   - User and group lifecycle management
 *   - Expense recording (delegates split computation to SplitStrategy)
 *   - Settlement recording
 *   - Balance queries (delegates to BalanceSheet)
 *   - Debt simplification (delegates to DebtSimplifier)
 *   - Notification dispatch (delegates to registered NotificationObserver list)
 *
 * This class is the Facade: callers only interact with SplitwiseService and never
 * need to know about strategies, balance sheets, or simplifier internals.
 *
 * Thread safety: user and group stores use ConcurrentHashMap; expense/settlement
 * operations synchronise on the BalanceSheet.
 */
public class SplitwiseService {

    // ── In-memory stores (replace with repository layer in production) ───────
    private final Map<String, User> users = new ConcurrentHashMap<>();
    private final Map<String, Group> groups = new ConcurrentHashMap<>();
    private final List<Expense> allExpenses = Collections.synchronizedList(new ArrayList<>());
    private final List<Settlement> allSettlements = Collections.synchronizedList(new ArrayList<>());

    // ── Supporting services ───────────────────────────────────────────────────
    private final BalanceSheet balanceSheet = new BalanceSheet();
    private final DebtSimplifier debtSimplifier = new DebtSimplifier();
    private final List<NotificationObserver> observers = new ArrayList<>();

    // ── Observer management ───────────────────────────────────────────────────

    public void registerObserver(NotificationObserver observer) {
        observers.add(observer);
    }

    public void removeObserver(NotificationObserver observer) {
        observers.remove(observer);
    }

    // ── User operations ───────────────────────────────────────────────────────

    /**
     * Registers a new user. Email must be unique (enforced here for correctness;
     * production would use a database unique constraint).
     */
    public User registerUser(String name, String email) {
        boolean emailExists = users.values().stream()
                .anyMatch(u -> u.getEmail().equalsIgnoreCase(email));
        if (emailExists) {
            throw new BusinessRuleViolationException("Email already registered: " + email);
        }
        User user = new User(name, email);
        users.put(user.getId(), user);
        return user;
    }

    /**
     * Returns a user by ID, throwing EntityNotFoundException if absent.
     */
    public User getUser(String userId) {
        User user = users.get(userId);
        if (user == null) throw new EntityNotFoundException("User", userId);
        return user;
    }

    /**
     * Creates a bidirectional friend relationship between two users.
     */
    public void addFriend(String userId1, String userId2) {
        User u1 = getUser(userId1);
        User u2 = getUser(userId2);
        u1.addFriend(u2);
        u2.addFriend(u1);
    }

    // ── Group operations ──────────────────────────────────────────────────────

    /**
     * Creates a named group with an initial set of member user IDs.
     */
    public Group createGroup(String name, List<String> memberUserIds) {
        List<User> members = new ArrayList<>();
        for (String uid : memberUserIds) {
            members.add(getUser(uid));
        }
        Group group = new Group(name, members);
        groups.put(group.getId(), group);
        return group;
    }

    /**
     * Returns a group by ID, throwing EntityNotFoundException if absent.
     */
    public Group getGroup(String groupId) {
        Group group = groups.get(groupId);
        if (group == null) throw new EntityNotFoundException("Group", groupId);
        return group;
    }

    /**
     * Adds a user to an existing group. Idempotent — silently skips if already a member.
     */
    public void addMemberToGroup(String groupId, String userId) {
        Group group = getGroup(groupId);
        User user = getUser(userId);
        boolean alreadyMember = group.getMembers().stream()
                .anyMatch(m -> m.getId().equals(userId));
        if (!alreadyMember) {
            group.addMember(user);
        }
    }

    // ── Expense operations ────────────────────────────────────────────────────

    /**
     * Records an expense and updates balances.
     *
     * @param description    human-readable label
     * @param totalAmount    full amount of the expense
     * @param paidByUserId   ID of the user who paid
     * @param participantIds IDs of all users who share the expense (must include payer
     *                       if payer should have a share)
     * @param splitType      how the amount is divided
     * @param splitRequest   pre-built SplitRequest (caller sets the type-specific fields)
     * @param groupId        optional — null for non-group expenses
     * @return the created Expense
     */
    public Expense addExpense(
            String description,
            BigDecimal totalAmount,
            String paidByUserId,
            List<String> participantIds,
            SplitType splitType,
            SplitRequest splitRequest,
            String groupId) {

        User payer = getUser(paidByUserId);
        List<User> participants = new ArrayList<>();
        for (String uid : participantIds) {
            participants.add(getUser(uid));
        }

        Group group = (groupId != null) ? getGroup(groupId) : null;

        // Validate all participants are group members if a group is specified.
        if (group != null) {
            for (User participant : participants) {
                boolean isMember = group.getMembers().stream()
                        .anyMatch(m -> m.getId().equals(participant.getId()));
                if (!isMember) {
                    throw new BusinessRuleViolationException(
                            "User " + participant.getName() + " is not a member of group " + group.getName());
                }
            }
        }

        // Delegate split computation to the appropriate strategy.
        SplitStrategy strategy = SplitStrategyFactory.getStrategy(splitType);
        // Rebuild request with resolved participants (in case caller passed partial request).
        SplitRequest resolvedRequest = SplitRequest.builder()
                .totalAmount(totalAmount)
                .participants(participants)
                .exactAmounts(splitRequest.getExactAmounts())
                .percentages(splitRequest.getPercentages())
                .shares(splitRequest.getShares())
                .build();

        List<ExpenseSplit> splits = strategy.computeSplits(resolvedRequest);

        Expense expense = new Expense(description, totalAmount, payer, splits, splitType, group);
        allExpenses.add(expense);

        if (group != null) {
            group.addExpense(expense);
        }

        // Update the balance sheet atomically.
        balanceSheet.applyExpense(expense);

        // Notify all observers (fire-and-forget).
        notifyExpenseAdded(expense);

        return expense;
    }

    /**
     * Convenience overload for expenses not attached to any group.
     */
    public Expense addExpense(
            String description,
            BigDecimal totalAmount,
            String paidByUserId,
            List<String> participantIds,
            SplitType splitType,
            SplitRequest splitRequest) {
        return addExpense(description, totalAmount, paidByUserId, participantIds,
                          splitType, splitRequest, null);
    }

    // ── Settlement operations ─────────────────────────────────────────────────

    /**
     * Records that paidByUserId has paid paidToUserId the given amount.
     * The balance between them is reduced accordingly.
     */
    public Settlement recordSettlement(String paidByUserId, String paidToUserId, BigDecimal amount) {
        User paidBy = getUser(paidByUserId);
        User paidTo = getUser(paidToUserId);

        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Settlement amount must be positive.");
        }

        Settlement settlement = new Settlement(paidBy, paidTo, amount);
        allSettlements.add(settlement);

        balanceSheet.applySettlement(settlement);
        notifySettlementRecorded(settlement);

        return settlement;
    }

    // ── Balance queries ───────────────────────────────────────────────────────

    /**
     * Returns a signed balance: positive means userId1 owes userId2,
     * negative means userId2 owes userId1.
     */
    public BigDecimal getBalance(String userId1, String userId2) {
        return balanceSheet.getBalance(userId1, userId2);
    }

    /**
     * Returns the full internal balance ledger as a snapshot.
     * Key: debtor user id → map of creditor id → amount owed.
     */
    public Map<String, Map<String, BigDecimal>> getAllBalances() {
        return balanceSheet.getNetBalance();
    }

    // ── Debt simplification ───────────────────────────────────────────────────

    /**
     * Computes the minimum set of transactions required to settle all debts
     * among the members of a group.
     */
    public List<Transaction> simplifyGroupDebts(String groupId) {
        Group group = getGroup(groupId);
        return debtSimplifier.simplify(group.getMembers(), balanceSheet.getNetBalance());
    }

    /**
     * Computes the minimum set of transactions for a custom list of users.
     */
    public List<Transaction> simplifyDebts(List<String> userIds) {
        List<User> userList = new ArrayList<>();
        for (String uid : userIds) {
            userList.add(getUser(uid));
        }
        return debtSimplifier.simplify(userList, balanceSheet.getNetBalance());
    }

    // ── Expense history ───────────────────────────────────────────────────────

    /**
     * Returns all expenses for a group in chronological order.
     */
    public List<Expense> getGroupExpenses(String groupId) {
        Group group = getGroup(groupId);
        return group.getExpenses(); // already chronological (append-only list)
    }

    /**
     * Returns all expenses that involve both userId1 and userId2 (either as payer or participant).
     */
    public List<Expense> getSharedExpenses(String userId1, String userId2) {
        List<Expense> result = new ArrayList<>();
        for (Expense expense : allExpenses) {
            boolean involvesUser1 = expense.getPaidBy().getId().equals(userId1) ||
                    expense.getSplits().stream().anyMatch(s -> s.getUser().getId().equals(userId1));
            boolean involvesUser2 = expense.getPaidBy().getId().equals(userId2) ||
                    expense.getSplits().stream().anyMatch(s -> s.getUser().getId().equals(userId2));
            if (involvesUser1 && involvesUser2) {
                result.add(expense);
            }
        }
        return Collections.unmodifiableList(result);
    }

    /**
     * Returns all settlements involving a given user.
     */
    public List<Settlement> getSettlements(String userId) {
        List<Settlement> result = new ArrayList<>();
        for (Settlement s : allSettlements) {
            if (s.getPaidBy().getId().equals(userId) || s.getPaidTo().getId().equals(userId)) {
                result.add(s);
            }
        }
        return Collections.unmodifiableList(result);
    }

    // ── Private helpers ───────────────────────────────────────────────────────

    private void notifyExpenseAdded(Expense expense) {
        for (NotificationObserver observer : observers) {
            try {
                observer.onExpenseAdded(expense);
            } catch (Exception e) {
                // Notification failure must not roll back the expense.
                System.err.println("Notification observer failed: " + e.getMessage());
            }
        }
    }

    private void notifySettlementRecorded(Settlement settlement) {
        for (NotificationObserver observer : observers) {
            try {
                observer.onSettlementRecorded(settlement);
            } catch (Exception e) {
                System.err.println("Notification observer failed: " + e.getMessage());
            }
        }
    }
}
```

### 7.11 End-to-End Demo

```java
package splitwise;

import splitwise.model.Expense;
import splitwise.model.Group;
import splitwise.model.Settlement;
import splitwise.model.SplitType;
import splitwise.model.Transaction;
import splitwise.model.User;
import splitwise.notification.EmailNotificationObserver;
import splitwise.notification.PushNotificationObserver;
import splitwise.service.SplitwiseService;
import splitwise.strategy.SplitRequest;

import java.math.BigDecimal;
import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.logging.ConsoleHandler;
import java.util.logging.Level;
import java.util.logging.Logger;
import java.util.logging.SimpleFormatter;

/**
 * End-to-end demonstration of the Splitwise system.
 *
 * Scenario:
 *   Four users — Alice, Bob, Charlie, Dave — share a trip.
 *   Multiple expenses are recorded with different split types.
 *   Balances are displayed, debts are simplified, and settlements are recorded.
 */
public class SplitwiseDemo {

    public static void main(String[] args) {
        // Configure logging to print INFO to console cleanly.
        Logger rootLogger = Logger.getLogger("");
        rootLogger.setLevel(Level.INFO);
        ConsoleHandler handler = new ConsoleHandler();
        handler.setLevel(Level.INFO);
        handler.setFormatter(new SimpleFormatter());
        rootLogger.addHandler(handler);

        SplitwiseService service = new SplitwiseService();

        // Register notification observers.
        service.registerObserver(new EmailNotificationObserver());
        service.registerObserver(new PushNotificationObserver());

        // ── Step 1: Register users ─────────────────────────────────────────────
        section("STEP 1: Register Users");
        User alice   = service.registerUser("Alice",   "alice@example.com");
        User bob     = service.registerUser("Bob",     "bob@example.com");
        User charlie = service.registerUser("Charlie", "charlie@example.com");
        User dave    = service.registerUser("Dave",    "dave@example.com");

        System.out.printf("Registered: %s, %s, %s, %s%n",
                alice.getName(), bob.getName(), charlie.getName(), dave.getName());

        // ── Step 2: Add friends ────────────────────────────────────────────────
        section("STEP 2: Add Friends");
        service.addFriend(alice.getId(), bob.getId());
        service.addFriend(alice.getId(), charlie.getId());
        service.addFriend(bob.getId(), dave.getId());
        System.out.println("Alice's friends: " +
                alice.getFriends().stream().map(User::getName).toList());

        // ── Step 3: Create group ───────────────────────────────────────────────
        section("STEP 3: Create Group 'Road Trip 2024'");
        Group group = service.createGroup("Road Trip 2024",
                Arrays.asList(alice.getId(), bob.getId(), charlie.getId(), dave.getId()));
        System.out.println("Group created: " + group.getName() +
                " with " + group.getMembers().size() + " members");

        // ── Step 4: EQUAL split — Hotel ($400, paid by Alice) ─────────────────
        section("STEP 4: Add Expense — Hotel (EQUAL split, paid by Alice)");
        SplitRequest equalRequest = SplitRequest.builder()
                .totalAmount(new BigDecimal("400.00"))
                .participants(Arrays.asList(alice, bob, charlie, dave))
                .build();

        Expense hotel = service.addExpense(
                "Hotel (2 nights)",
                new BigDecimal("400.00"),
                alice.getId(),
                Arrays.asList(alice.getId(), bob.getId(), charlie.getId(), dave.getId()),
                SplitType.EQUAL,
                equalRequest,
                group.getId());

        System.out.println("Expense: " + hotel.getDescription() + " — $" + hotel.getTotalAmount());
        hotel.getSplits().forEach(s ->
                System.out.printf("  %s owes $%s%n", s.getUser().getName(), s.getAmount()));

        // ── Step 5: EXACT split — Petrol ($90, paid by Bob) ───────────────────
        section("STEP 5: Add Expense — Petrol (EXACT split, paid by Bob)");
        // Alice=$30, Bob=$20 (his own share), Charlie=$25, Dave=$15
        Map<String, BigDecimal> exactAmounts = new HashMap<>();
        exactAmounts.put(alice.getId(),   new BigDecimal("30.00"));
        exactAmounts.put(bob.getId(),     new BigDecimal("20.00"));
        exactAmounts.put(charlie.getId(), new BigDecimal("25.00"));
        exactAmounts.put(dave.getId(),    new BigDecimal("15.00"));

        SplitRequest exactRequest = SplitRequest.builder()
                .totalAmount(new BigDecimal("90.00"))
                .participants(Arrays.asList(alice, bob, charlie, dave))
                .exactAmounts(exactAmounts)
                .build();

        Expense petrol = service.addExpense(
                "Petrol",
                new BigDecimal("90.00"),
                bob.getId(),
                Arrays.asList(alice.getId(), bob.getId(), charlie.getId(), dave.getId()),
                SplitType.EXACT,
                exactRequest,
                group.getId());

        System.out.println("Expense: " + petrol.getDescription() + " — $" + petrol.getTotalAmount());
        petrol.getSplits().forEach(s ->
                System.out.printf("  %s owes $%s%n", s.getUser().getName(), s.getAmount()));

        // ── Step 6: PERCENTAGE split — Dinner ($200, paid by Charlie) ─────────
        section("STEP 6: Add Expense — Dinner (PERCENTAGE split, paid by Charlie)");
        // Alice=40%, Bob=30%, Charlie=20%, Dave=10%
        Map<String, BigDecimal> percentages = new HashMap<>();
        percentages.put(alice.getId(),   new BigDecimal("40"));
        percentages.put(bob.getId(),     new BigDecimal("30"));
        percentages.put(charlie.getId(), new BigDecimal("20"));
        percentages.put(dave.getId(),    new BigDecimal("10"));

        SplitRequest pctRequest = SplitRequest.builder()
                .totalAmount(new BigDecimal("200.00"))
                .participants(Arrays.asList(alice, bob, charlie, dave))
                .percentages(percentages)
                .build();

        Expense dinner = service.addExpense(
                "Dinner at Steakhouse",
                new BigDecimal("200.00"),
                charlie.getId(),
                Arrays.asList(alice.getId(), bob.getId(), charlie.getId(), dave.getId()),
                SplitType.PERCENTAGE,
                pctRequest,
                group.getId());

        System.out.println("Expense: " + dinner.getDescription() + " — $" + dinner.getTotalAmount());
        dinner.getSplits().forEach(s ->
                System.out.printf("  %s owes $%s%n", s.getUser().getName(), s.getAmount()));

        // ── Step 7: SHARES split — Activities ($150, paid by Dave) ────────────
        section("STEP 7: Add Expense — Activities (SHARES split, paid by Dave)");
        // Alice=3 shares, Bob=2 shares, Charlie=2 shares, Dave=3 shares
        // Total shares = 10; $150/10 = $15 per share
        Map<String, Integer> shares = new HashMap<>();
        shares.put(alice.getId(),   3);
        shares.put(bob.getId(),     2);
        shares.put(charlie.getId(), 2);
        shares.put(dave.getId(),    3);

        SplitRequest sharesRequest = SplitRequest.builder()
                .totalAmount(new BigDecimal("150.00"))
                .participants(Arrays.asList(alice, bob, charlie, dave))
                .shares(shares)
                .build();

        Expense activities = service.addExpense(
                "Activities & Tours",
                new BigDecimal("150.00"),
                dave.getId(),
                Arrays.asList(alice.getId(), bob.getId(), charlie.getId(), dave.getId()),
                SplitType.SHARES,
                sharesRequest,
                group.getId());

        System.out.println("Expense: " + activities.getDescription() + " — $" + activities.getTotalAmount());
        activities.getSplits().forEach(s ->
                System.out.printf("  %s owes $%s (%d shares)%n",
                        s.getUser().getName(), s.getAmount(), shares.get(s.getUser().getId())));

        // ── Step 8: Show raw balances ──────────────────────────────────────────
        section("STEP 8: Current Balances");
        printBalance(service, alice, bob,    "Alice owes Bob");
        printBalance(service, alice, charlie, "Alice owes Charlie");
        printBalance(service, alice, dave,    "Alice owes Dave");
        printBalance(service, bob,   charlie, "Bob owes Charlie");
        printBalance(service, bob,   dave,    "Bob owes Dave");
        printBalance(service, charlie, dave,  "Charlie owes Dave");

        // ── Step 9: Simplify debts ────────────────────────────────────────────
        section("STEP 9: Simplified Debt Transactions");
        List<Transaction> simplified = service.simplifyGroupDebts(group.getId());
        if (simplified.isEmpty()) {
            System.out.println("All debts are settled!");
        } else {
            simplified.forEach(t ->
                    System.out.printf("  %s pays %s $%s%n",
                            t.getFrom().getName(), t.getTo().getName(), t.getAmount()));
        }
        System.out.println("Minimum transactions needed: " + simplified.size());

        // ── Step 10: Record a settlement ──────────────────────────────────────
        section("STEP 10: Record Settlement — Bob pays Alice $70");
        Settlement settlement = service.recordSettlement(
                bob.getId(), alice.getId(), new BigDecimal("70.00"));
        System.out.println("Settlement recorded: " + settlement);

        // ── Step 11: Show updated balances after settlement ───────────────────
        section("STEP 11: Updated Balances After Settlement");
        printBalance(service, alice, bob, "Alice-Bob balance");
        printBalance(service, alice, charlie, "Alice-Charlie balance");

        // ── Step 12: Show group expense history ───────────────────────────────
        section("STEP 12: Group Expense History");
        List<Expense> history = service.getGroupExpenses(group.getId());
        System.out.println("Total expenses in group: " + history.size());
        history.forEach(e -> System.out.printf(
                "  [%s] %s — $%s — paid by %s — split type: %s%n",
                e.getCreatedAt().toLocalDate(), e.getDescription(),
                e.getTotalAmount(), e.getPaidBy().getName(), e.getSplitType()));

        // ── Step 13: Simplify debts after settlement ──────────────────────────
        section("STEP 13: Simplified Debts After Settlement");
        List<Transaction> simplifiedAfter = service.simplifyGroupDebts(group.getId());
        simplifiedAfter.forEach(t ->
                System.out.printf("  %s pays %s $%s%n",
                        t.getFrom().getName(), t.getTo().getName(), t.getAmount()));
        System.out.println("Minimum transactions needed: " + simplifiedAfter.size());

        // ── Step 14: Demonstrate validation error ─────────────────────────────
        section("STEP 14: Validate Error Handling — Bad Percentage Split");
        try {
            Map<String, BigDecimal> badPcts = new HashMap<>();
            badPcts.put(alice.getId(), new BigDecimal("60"));
            badPcts.put(bob.getId(),   new BigDecimal("50")); // sum = 110, not 100

            SplitRequest badRequest = SplitRequest.builder()
                    .totalAmount(new BigDecimal("100.00"))
                    .participants(Arrays.asList(alice, bob))
                    .percentages(badPcts)
                    .build();

            service.addExpense("Bad Expense", new BigDecimal("100.00"),
                    alice.getId(), Arrays.asList(alice.getId(), bob.getId()),
                    SplitType.PERCENTAGE, badRequest);

        } catch (splitwise.exception.InvalidSplitException e) {
            System.out.println("Caught expected exception: " + e.getMessage());
        }

        section("DEMO COMPLETE");
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private static void printBalance(SplitwiseService service, User u1, User u2, String label) {
        BigDecimal balance = service.getBalance(u1.getId(), u2.getId());
        if (balance.compareTo(BigDecimal.ZERO) > 0) {
            System.out.printf("  %s: %s owes %s $%s%n", label, u1.getName(), u2.getName(), balance);
        } else if (balance.compareTo(BigDecimal.ZERO) < 0) {
            System.out.printf("  %s: %s owes %s $%s%n", label, u2.getName(), u1.getName(), balance.negate());
        } else {
            System.out.printf("  %s: settled ($0.00)%n", label);
        }
    }

    private static void section(String title) {
        System.out.println();
        System.out.println("══════════════════════════════════════════════════");
        System.out.println("  " + title);
        System.out.println("══════════════════════════════════════════════════");
    }
}
```

### 7.12 Package Structure Summary

```
splitwise/
├── model/
│   ├── SplitType.java
│   ├── MoneyConstants.java
│   ├── User.java
│   ├── Group.java
│   ├── Expense.java
│   ├── ExpenseSplit.java
│   ├── Settlement.java
│   ├── UserPair.java
│   └── Transaction.java
├── strategy/
│   ├── SplitStrategy.java            ← interface
│   ├── SplitRequest.java             ← value object / builder
│   ├── SplitStrategyFactory.java     ← factory
│   ├── EqualSplitStrategy.java
│   ├── ExactSplitStrategy.java
│   ├── PercentageSplitStrategy.java
│   └── SharesSplitStrategy.java
├── service/
│   ├── BalanceSheet.java
│   ├── DebtSimplifier.java
│   └── SplitwiseService.java         ← facade
├── notification/
│   ├── NotificationObserver.java     ← interface
│   ├── EmailNotificationObserver.java
│   └── PushNotificationObserver.java
├── exception/
│   ├── InvalidSplitException.java
│   ├── EntityNotFoundException.java
│   └── BusinessRuleViolationException.java
└── SplitwiseDemo.java
```

---

## 8. Design Patterns Used

### 8.1 Strategy Pattern — Split Types

**What it is**: Define a family of algorithms, encapsulate each one, and make them interchangeable. Clients can select an algorithm at runtime without altering the code that uses it.

**Why it's the right fit here**: The four split types (Equal, Exact, Percentage, Shares) are semantically identical at the service level — they all receive an amount and a list of participants, and they all return a list of `ExpenseSplit` objects. The only thing that differs is the computation and its validation rules. Without Strategy, the service would contain a giant switch statement:

```java
// Without Strategy — anti-pattern
if (splitType == EQUAL) { ... }
else if (splitType == EXACT) { validateExact(); ... }
else if (splitType == PERCENTAGE) { validatePercentage(); ... }
else if (splitType == SHARES) { ... }
```

Every new split type means editing the service class, risking regressions. With Strategy:

```java
// With Strategy — adding a new type = one new class, zero changes elsewhere
SplitStrategy strategy = SplitStrategyFactory.getStrategy(splitType);
List<ExpenseSplit> splits = strategy.computeSplits(request);
```

**Key design point — self-validating strategies**: Each strategy owns its validation. `validate()` is called inside `computeSplits()`, so the strategy is always in a consistent state. The service does not need to know what "valid PERCENTAGE input" means — it delegates that knowledge entirely.

**Key design point — the `SplitRequest` value object**: Rather than passing six parameters to each strategy method, `SplitRequest` bundles all possible inputs. Irrelevant fields (e.g., `percentages` for an EQUAL split) are simply ignored. This keeps the interface stable when new fields are added.

**Key design point — stateless strategies**: All four strategy classes are stateless singletons registered in `SplitStrategyFactory`. This means they can be shared across threads without synchronisation.

### 8.2 Observer Pattern — Notifications

**What it is**: Define a one-to-many dependency so that when one object changes state, all its dependents are notified automatically.

**Why it's the right fit here**: The service layer must not be coupled to any specific notification technology. Email, push, SMS, webhooks — these should be pluggable, and multiple channels should operate simultaneously. The Observer pattern achieves exactly this:

```java
// SplitwiseService knows only about the NotificationObserver interface
for (NotificationObserver observer : observers) {
    observer.onExpenseAdded(expense);
}
```

New notification channels are added by implementing `NotificationObserver` and calling `service.registerObserver(new SmsNotificationObserver())`. The service itself never changes.

**Key design point — fire-and-forget with fault isolation**: Notification failures are caught and logged but do not roll back the expense. The `try/catch` around each observer call ensures one bad observer does not prevent others from running and does not corrupt the business transaction.

### 8.3 Facade Pattern — SplitwiseService

**What it is**: Provide a simplified interface to a complex subsystem.

**Why it's appropriate**: Callers should not need to know about `BalanceSheet`, `DebtSimplifier`, `SplitStrategyFactory`, or individual strategy classes. `SplitwiseService` is the single entry point that orchestrates all of these — callers make one call to `addExpense()` and the facade handles strategy selection, validation, persistence, balance update, and notification dispatch.

### 8.4 Factory Pattern — SplitStrategyFactory

**What it is**: Centralise object creation behind a static factory method.

**Why it's appropriate**: The mapping from `SplitType` enum to concrete strategy class is encapsulated in one place. When a new split type is added, the factory is the only place that needs to change (beyond the new strategy class itself). The `EnumMap` provides O(1) lookup and compile-time safety.

### 8.5 Builder Pattern — SplitRequest

**What it is**: Separate the construction of a complex object from its representation.

**Why it's appropriate**: `SplitRequest` has several optional fields that vary by split type. A constructor with all fields would be unwieldy and error-prone. The builder makes intent explicit at call sites:

```java
SplitRequest.builder()
    .totalAmount(new BigDecimal("200.00"))
    .participants(participants)
    .percentages(percentages)
    .build();
```

### 8.6 Value Object Pattern — UserPair, Transaction, ExpenseSplit

These classes are immutable, equality-by-value objects that encapsulate a concept without identity. `UserPair` provides a canonical, order-independent key for balance maps. `Transaction` is a pure data carrier for simplification results.

---

## 9. Key Design Decisions and Trade-offs

### 9.1 Split Strategy Design — The Core Interview Challenge

The central design challenge is that four semantically different algorithms must be pluggable at the call site. The key decisions:

**Decision: Strategy validates its own input.**
Alternative: the service validates before delegating. This is inferior because it requires the service to know the rules of each strategy, defeating the purpose of encapsulation. Each strategy class is the authority on what constitutes valid input for itself.

**Decision: `computeSplits` calls `validate` internally.**
This makes misuse impossible — a caller cannot forget to validate. The cost is that `validate` runs twice if the caller explicitly calls it first (e.g., in unit tests). This is a minor and acceptable cost.

**Decision: Rounding remainder goes to the first (or last) participant, deterministically.**
Alternatives: random participant (non-deterministic, hard to audit), round-robin rotation (stateful). Deterministic rounding is easiest to explain to users and to reproduce in tests. The last-participant approach is used for PERCENTAGE and SHARES because by the time we reach the last person, we know the exact remainder — this gives us a mathematically correct total.

**Decision: Percentages stored as 0–100 (not 0–1).**
User-facing input is always 0–100 (matching how humans think about percentages). Internal arithmetic multiplies by total and divides by 100. This avoids the common off-by-factor-of-100 bug when mixing representations.

**Decision: BigDecimal for all monetary values.**
`double` cannot represent most decimal fractions exactly. For financial systems, even sub-cent errors accumulate. `BigDecimal` with a fixed scale of 2 and `HALF_UP` rounding is the only correct approach in Java. The cost is slightly more verbose arithmetic.

### 9.2 Debt Simplification Algorithm

**Decision: Greedy net-balance algorithm.**
Alternative: enumerate all possible subsets of debts (NP-hard). The greedy approach is O(n log n) and produces optimal results when debt amounts are continuous. In practice, with integer cents, it produces near-optimal results. The difference between optimal and near-optimal for a typical group of 10 people is at most 1-2 extra transactions — a worthwhile trade for polynomial-time complexity.

**Decision: Simplification is advisory, not authoritative.**
The simplified transaction list is a recommendation. Actual settlements are recorded independently via `recordSettlement()`. This is intentional: users may settle in any order, or partially, or in amounts that differ from the recommendation. The balance sheet always reflects actual recorded settlements.

**Decision: Two separate heaps for creditors and debtors.**
Using a single sorted list would require rescanning after each iteration. Two heaps give O(log n) insert and O(log n) extract, keeping the overall algorithm at O(n log n).

**Decision: Net balances are positive-only in the balance sheet.**
The balance sheet uses `netBalance[debtor][creditor]` = positive amount. This avoids sign confusion when reading and updating values. The direction is encoded structurally (which user is the key, which is the inner key) rather than by sign. Signed approaches are compact but error-prone.

### 9.3 BalanceSheet Thread Safety

**Decision: Synchronise on the BalanceSheet instance.**
All mutating methods (`applyExpense`, `applySettlement`) are `synchronized`. This prevents concurrent expense additions from corrupting the balance state. The trade-off is that this is a coarse lock — a heavily loaded production system would need a per-UserPair lock or an optimistic concurrency approach (e.g., compare-and-swap). For the LLD scope, instance-level synchronisation is correct and simple.

**Decision: Automatic debt netting in `addDebt`.**
When user B owes A $50 and A owes B $30, the system stores one entry: B owes A $20, rather than two cross-entries. This keeps the balance sheet compact and makes queries simple. The netting logic runs inside `addDebt` transparently.

### 9.4 Immutability of Expenses and Settlements

**Decision: Expense and Settlement are immutable after creation.**
This means the audit trail cannot be accidentally mutated. Amendments are modelled as new expenses (e.g., a negative correction expense). This is the standard accounting approach and aligns with regulatory requirements in production. The trade-off is that amending an expense requires creating two records instead of one update — a minor overhead.

### 9.5 No Persistent Identity for BalanceSheet

**Decision: Balance is recomputed from the running ledger, not stored separately.**
In a production system the balance would be stored and updated incrementally. Here, the `BalanceSheet` is an in-memory accumulator that is initialised empty and updated with each expense/settlement. If the JVM restarts, the balances are lost (because we have no persistence layer). In production, the BalanceSheet would be backed by a database and replayed from the event log on startup — or more likely, the balance would be a derived view in the DB.

---

## 10. Extension Points

### 10.1 New Split Type

1. Add a value to `SplitType` enum.
2. Create `MySplitStrategy implements SplitStrategy`.
3. Register in `SplitStrategyFactory.REGISTRY`.

No other changes. All existing tests remain green. This is a textbook Open/Closed implementation.

### 10.2 New Notification Channel

Implement `NotificationObserver` and register with `service.registerObserver(...)`. The service is completely agnostic to the channel.

### 10.3 Persistence Layer

Replace the in-memory `ConcurrentHashMap` stores in `SplitwiseService` with repository interfaces:

```java
interface UserRepository {
    void save(User user);
    Optional<User> findById(String id);
    Optional<User> findByEmail(String email);
}
```

The service depends on the interface; concrete implementations use JPA, DynamoDB, Redis, etc.

### 10.4 Recurring Expenses

Add a `RecurringExpense` that wraps a `SplitRequest` and a `ScheduleDefinition` (cron expression, frequency). A scheduler service reads recurring definitions and calls `SplitwiseService.addExpense()` at the right time.

### 10.5 Multi-Currency Support

Introduce a `Currency` field on `Expense`. Add a `CurrencyConverter` interface:

```java
interface CurrencyConverter {
    BigDecimal convert(BigDecimal amount, Currency from, Currency to);
}
```

`BalanceSheet.applyExpense` converts all splits to a base currency before updating balances. The Strategy pattern still works — strategies compute splits in the expense's native currency, and conversion happens at the service layer.

### 10.6 Expense Categories and Reporting

Add a `Category` enum to `Expense`. Extend `getGroupExpenses` to accept an optional `Category` filter. Add a `ReportService` that groups expenses by category and produces summaries (total spend per person per category).

### 10.7 Audit Log / Event Sourcing

Replace the mutable `BalanceSheet` with an event log (append-only `ExpenseCreatedEvent`, `SettlementRecordedEvent`). The `BalanceSheet` becomes a read model rebuilt by replaying events. This enables time-travel queries ("what were the balances on Oct 1?") and is the standard production architecture for financial systems.

---

## 11. Interview Discussion Points

### 11.1 "Walk me through how adding a new expense works."

The call flow is:
1. `SplitwiseService.addExpense()` resolves `User` and `Group` objects from ID stores.
2. Validates group membership if a group is specified.
3. Calls `SplitStrategyFactory.getStrategy(splitType)` — O(1) enum map lookup.
4. Builds a `SplitRequest` with all inputs.
5. Calls `strategy.computeSplits(request)` — this internally calls `validate()` first.
6. Constructs an immutable `Expense` object with the computed splits.
7. Appends the expense to the group's list and the global list.
8. Calls `balanceSheet.applyExpense(expense)` — synchronized, updates net balances.
9. Calls `notifyExpenseAdded(expense)` — dispatches to all registered observers.

The critical observation: steps 4–8 are atomic from the caller's perspective (single-threaded within the lock). Notification happens after the balance update succeeds, so a notification failure cannot cause inconsistency.

### 11.2 "How does the simplification algorithm work?"

Walk through the two-phase approach:

**Phase 1 — Net balances**: Collapse the full debtor→creditor matrix into one number per person. This loses information about who-owes-whom in the individual sense, but preserves the total financial position of each person.

**Phase 2 — Greedy two-heap**: Always pair the largest creditor with the largest debtor. They transact the minimum of the two amounts. Whoever has residual goes back in the heap. This greedy choice is locally optimal (each iteration creates exactly one transaction) and globally optimal for continuous amounts.

**Why is it optimal?** Because the minimum number of transactions to resolve n non-zero net balances is exactly n-1 (you resolve one balance each iteration, and at least one party reaches zero balance per step). The greedy approach achieves this.

**Time complexity**: O(n log n) — computing net balances is O(E) where E = number of expense-participant pairs; the heap operations across all iterations are O(n log n) where n = users with non-zero balance.

### 11.3 "How would you handle the case where percentages don't add up to exactly 100 due to floating point?"

We use `BigDecimal` for all monetary values, which eliminates floating-point representation errors. For percentages, we compare against 100 using an epsilon of 0.001 (i.e., within one-tenth of a percent). This handles cases where a user enters 33.33% for each of three participants (sum = 99.99%, within epsilon). The last participant absorbs the rounding difference — not through epsilon tolerance, but through the "last participant gets the remainder" rule in `computeSplits`.

### 11.4 "What if two expenses are added simultaneously?"

The `BalanceSheet.applyExpense` method is `synchronized`. Concurrent calls will queue up and execute serially. The `Expense` object itself is constructed before the lock is acquired, so construction can happen in parallel — only the balance update is serialised. In production, you would replace this with database-level transactions (e.g., `UPDATE balances SET amount = amount + ? WHERE debtor = ? AND creditor = ?` with optimistic locking or row-level locking).

### 11.5 "How does the payer's own share work?"

The payer can appear in the participant list (e.g., Alice pays $400 and is also split equally among 4 people). In that case, Alice's share is $100. Alice owes herself $100... which is nonsensical. In `BalanceSheet.applyExpense`, we explicitly skip the split where `split.getUser().equals(payer)`. So Alice's $100 self-share simply creates no debt entry. The net effect: the other three each owe Alice $100, totalling $300 of incoming debt to Alice (not $400), which is correct — Alice effectively paid $100 for herself out of pocket.

### 11.6 "How would you support settling debts between multiple groups?"

Currently `simplifyGroupDebts` scopes the algorithm to a single group's members. To simplify across multiple groups (e.g., a user is in a travel group and a household group), you would call `simplifyDebts(userIds)` with the union of user IDs across all groups. The algorithm is user-list-agnostic — it works on any subset of the global balance sheet.

### 11.7 "What's the trade-off between storing raw debts vs. net balances?"

**Raw debts** (one entry per expense-participant pair): Full audit trail, but balance queries require scanning all expense history — O(E) per query. Balances are computed at read time.

**Net balances** (aggregated ledger, our approach): Balance queries are O(1). But you lose the per-expense breakdown in the balance sheet (still available from the expense list). This is the correct choice for a system where balance lookups are frequent (every page load) and expense creation is less frequent.

A production system would maintain both: the expense ledger for audit, and a balance cache for read performance, kept in sync via database transactions.

### 11.8 "How would you ensure no money is lost or created due to rounding?"

The rounding strategy is: compute n-1 shares using division+rounding, then assign the remainder to the last participant. This guarantees that the sum of all splits is exactly equal to `totalAmount`. The `ExactSplitStrategy` validates this by summing the user-provided amounts and comparing to `totalAmount` — if they differ by even one cent, an exception is thrown.

---

## 12. Production Considerations

### 12.1 Persistence

Replace in-memory maps with a relational database (PostgreSQL recommended for ACID guarantees). Schema:

```sql
users        (id UUID PK, name, email UNIQUE, created_at)
groups       (id UUID PK, name, created_at)
group_members (group_id FK, user_id FK)
expenses     (id UUID PK, description, total_amount NUMERIC(12,2),
               paid_by_user_id FK, split_type, group_id FK, created_at)
expense_splits (expense_id FK, user_id FK, amount NUMERIC(12,2))
settlements  (id UUID PK, paid_by_user_id FK, paid_to_user_id FK,
               amount NUMERIC(12,2), settled_at)
balances     (debtor_user_id FK, creditor_user_id FK, amount NUMERIC(12,2),
               PRIMARY KEY(debtor_user_id, creditor_user_id))
```

The `balances` table is updated atomically with each expense insertion using a database transaction that spans both the `expenses`/`expense_splits` insert and the `balances` upserts.

### 12.2 API Layer

Expose a REST API with the following endpoints:

```
POST   /users                        → register user
POST   /users/{id}/friends           → add friend
POST   /groups                       → create group
POST   /groups/{id}/members          → add member
POST   /expenses                     → add expense (body: type, amount, payer, participants, split inputs)
GET    /expenses?groupId=&userId=    → expense history
POST   /settlements                  → record settlement
GET    /balances?userId=             → get balances for a user
GET    /groups/{id}/simplify         → simplified debts for group
```

### 12.3 Concurrency at Scale

Replace the `synchronized` balance update with database-level row locking:

```sql
BEGIN;
  SELECT amount FROM balances WHERE debtor = $1 AND creditor = $2 FOR UPDATE;
  UPDATE balances SET amount = amount + $3 WHERE debtor = $1 AND creditor = $2;
COMMIT;
```

Or use an idempotent event-sourcing approach where balance is a projection from the immutable event log.

### 12.4 Notification Reliability

Replace synchronous observer dispatch with an outbox pattern:

1. Write the expense and a `notifications_pending` record in the same database transaction.
2. A background worker reads `notifications_pending` and dispatches via email gateway / push gateway.
3. On success, mark the notification as sent. On failure, retry with exponential backoff.

This ensures notifications are delivered at-least-once even if the application crashes after writing the expense but before dispatching.

### 12.5 Idempotency

All expense creation requests should include a client-generated `idempotencyKey`. The server rejects (or deduplicates) duplicate requests with the same key within a time window. This prevents double-charging when clients retry on network timeout.

### 12.6 Decimal Precision

Store monetary amounts as `NUMERIC(12, 2)` in the database. Never use `FLOAT` or `DOUBLE PRECISION` for money in SQL — they introduce the same floating-point hazards as Java `double`. Java-side `BigDecimal` maps cleanly to SQL `NUMERIC`.

### 12.7 Observability

Add structured logging and metrics:

- Log every expense creation and settlement with correlation IDs.
- Emit metrics: `expense.created.count`, `expense.created.amount` (histogram), `settlement.count`, `debt.simplification.transactions` (gauge), `split.strategy.calls` (by type).
- Alert on balance inconsistency (sum of all net balances should always be zero — a non-zero sum indicates a bug).

### 12.8 Testing Strategy

- **Unit tests**: Each `SplitStrategy` tested in isolation with normal cases, edge cases (single participant, zero remainder, rounding), and invalid inputs.
- **Unit tests**: `DebtSimplifier` tested against known examples (verify transaction count = optimal, verify all balances are resolved).
- **Unit tests**: `BalanceSheet` tested for correct netting, over-payment handling, and concurrent modifications.
- **Integration tests**: `SplitwiseService` with all four split types, verifying that balances after expense creation match expected values.
- **Property-based tests**: For all strategies, verify that `sum(splits) == totalAmount` for any valid input (using a property testing library like jqwik).
- **Mutation tests**: Run PIT to ensure test suite catches intentional mutations in the arithmetic.

### 12.9 Security

- Authenticate all API calls (JWT or OAuth 2.0).
- Authorise: a user can only view expenses and balances for groups they belong to.
- Prevent a user from creating an expense on behalf of a payer who is a different user without that payer's consent (flag for explicit approval workflow).
- Rate-limit the expense creation endpoint to prevent abuse.
- Log all financial mutations to an immutable audit log with user identity and timestamp.

### 12.10 Scale Characteristics

For a system at Splitwise scale (~10M active users):

- Balance reads are hot; cache them in Redis with write-through on each settlement/expense.
- Groups are bounded in size (typical Splitwise groups are 2–20 people), so the simplification algorithm never processes more than ~20 users — runtime is negligible.
- Expense history can grow unboundedly; paginate all list endpoints, and archive expenses older than N years to cold storage.
- Shard the database by user ID for write scalability; cross-shard group queries use a read replica fan-out pattern.
