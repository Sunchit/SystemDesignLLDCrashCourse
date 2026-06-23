# ATM System — Low-Level Design Case Study

> **Difficulty**: Medium  
> **Primary Pattern**: State Machine (State Pattern)  
> **Secondary Patterns**: Strategy, Template Method, Singleton  
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

Design the software for a standalone ATM machine. The ATM must allow bank customers to insert their card, authenticate with a PIN, and perform financial transactions (balance inquiry, cash withdrawal, deposit, and fund transfer). The system must enforce security (PIN lockout after repeated failures), daily withdrawal limits, and maintain a complete transaction history. The ATM communicates with the bank's backend over a network, which must be abstracted so it can be swapped between real and mock implementations.

The design should model the ATM's lifecycle as an explicit state machine so that invalid operations in a given state (e.g., withdrawing cash before authenticating) are rejected at the state level, not through scattered `if` checks throughout the codebase.

---

## 2. Functional Requirements

1. The ATM shall accept a customer's card (identified by card number).
2. The ATM shall prompt the customer to enter a PIN.
3. The ATM shall validate the PIN against the stored hash; allow at most **3 attempts** before locking the card.
4. Upon successful authentication, the customer shall be able to:
   - 4a. Check the balance of any linked account.
   - 4b. Withdraw cash (limited by available denominations, account balance, and daily withdrawal limit).
   - 4c. Deposit cash or cheques.
   - 4d. Transfer funds to another account.
5. The ATM shall support two account types: **Checking** and **Savings**.
6. The ATM shall dispense cash using available denominations (e.g., $100, $50, $20, $10).
7. The ATM shall record every transaction with type, amount, timestamp, balance before, and balance after.
8. The customer shall be able to view the last N transactions.
9. The ATM shall eject the card at the end of each session (either by customer choice or on error/lockout).
10. A locked card shall not be usable until it is unlocked by bank staff (out of scope for this design).

---

## 3. Non-Functional Requirements

| Quality Attribute | Requirement |
|---|---|
| **Security** | PINs are never stored or transmitted in plain text; SHA-256 hash used |
| **Security** | Card locked after 3 consecutive failed PIN attempts |
| **Security** | Session automatically invalidated on card ejection |
| **Correctness** | All monetary amounts stored as `long` cents to avoid floating-point errors |
| **Reliability** | Network failures return a typed result; the ATM does not crash |
| **Maintainability** | New states can be added without modifying existing state classes (Open/Closed) |
| **Testability** | Bank network interaction hidden behind an interface; fully mockable |
| **Auditability** | Every transaction written to an immutable `Transaction` record |

---

## 4. Assumptions and Constraints

- **Single currency**: All amounts are in USD cents (e.g., `500` = $5.00).
- **Single card per session**: The ATM handles one customer session at a time.
- **Card storage**: Cards and accounts are pre-populated (no card issuance logic in scope).
- **PIN storage**: We store a SHA-256 hash of the PIN. Salt is omitted for simplicity; a production system would use bcrypt/Argon2 with a per-card salt.
- **Cash denominations**: $100, $50, $20, $10 bills. The dispenser uses a greedy algorithm (largest first).
- **Daily limit**: $1,000 per day. The limit resets at midnight (tracked per card, not per account).
- **Transaction history**: Stored in-memory (a real system would persist to a database).
- **Thread safety**: Not in scope; a single-threaded model is assumed (one session at a time per ATM).
- **Network**: The `BankNetworkService` is the only external dependency; all bank operations go through it.
- **Deposit**: For simplicity, deposit immediately credits the account (no hold logic).

---

## 5. Domain Model Description

The domain centers on three clusters:

### ATM Hardware Cluster
- **ATM**: The top-level machine. Owns a `CardReader`, `CashDispenser`, `Screen`, and `Keypad`. Delegates all behavioral logic to a `ATMState` implementation.
- **CardReader**: Accepts or ejects a physical card. Holds the currently inserted `Card`.
- **CashDispenser**: Tracks available denominations and physically dispenses bills.
- **Screen / Keypad**: I/O devices (simulated in this design as simple print/read wrappers).

### State Machine Cluster
- **ATMState** (interface): The contract every concrete state must implement. Operations that are illegal in a state throw `IllegalStateException`.
- **IdleState**: Waiting for a card. Only `insertCard()` is valid.
- **CardInsertedState**: Card is in the reader. Only `enterPin()` is valid.
- **PINEnteredState**: PIN has been submitted; waiting for bank response.
- **AuthenticatedState**: Customer is verified. Financial operations are valid.
- **DispensingState**: Cash dispensing is in progress. Only `dispenseCash()` is valid (system-driven).

### Banking Cluster
- **Account** (abstract): Holds `accountId`, `balance`, `transactions`. Subclassed by `CheckingAccount` and `SavingsAccount`.
- **Card**: Links a physical card number to a hashed PIN and an `accountId`.
- **Transaction**: Immutable value object recording a financial event.
- **BankNetworkService** (interface): Abstraction for all bank-side operations. `MockBankNetworkService` provides an in-memory implementation.

---

## 6. Class Diagram

```
+---------------------------+          +----------------------+
|          ATM              |          |      ATMState        |
|---------------------------|          |    <<interface>>     |
| - currentState: ATMState  |<>------->|----------------------|
| - cardReader: CardReader  |          | + insertCard(...)    |
| - cashDispenser:          |          | + enterPin(...)      |
|     CashDispenser         |          | + checkBalance(...)  |
| - bankService:            |          | + withdraw(...)      |
|     BankNetworkService    |          | + deposit(...)       |
| - screen: Screen          |          | + transfer(...)      |
| - keypad: Keypad          |          | + ejectCard(...)     |
|---------------------------|          +----------^-----------+
| + insertCard(card)        |                     |
| + enterPin(pin)           |         implements by:
| + checkBalance()          |    +----------------+----------------+
| + withdraw(amount)        |    |       |         |               |
| + deposit(amount)         | IdleState CardInserted PINEntered  Authenticated
| + transfer(to, amount)    |           State       State        State
| + ejectCard()             |                                       |
+---------------------------+                               DispensingState

+-------------------+       +-----------------------+
|    CardReader     |       |    CashDispenser       |
|-------------------|       |-----------------------|
| - card: Card      |       | - denominations:      |
|-------------------|       |     Map<Integer,Int>  |
| + insertCard(c)   |       |-----------------------|
| + ejectCard()     |       | + canDispense(amt)    |
| + getCard()       |       | + dispense(amt)       |
+-------------------+       | + getAvailableCash()  |
                            +-----------------------+

+---------------------------+       +----------------------+
|          Card             |       |       Account        |
|---------------------------|       |    <<abstract>>      |
| - cardNumber: String      |       |----------------------|
| - pinHash: String         |       | - accountId: String  |
| - linkedAccountId: String |       | - balance: long      |
| - locked: boolean         |       | - transactions:List  |
| - failedAttempts: int     |       |----------------------|
|---------------------------|       | + credit(amount)     |
| + validatePin(pin): bool  |       | + debit(amount)      |
| + incrementFailedAttempts |       | + getBalance()       |
| + lock()                  |       | + getTransactions()  |
| + isLocked()              |       +----------^-----------+
| + resetFailedAttempts()   |                  |
+---------------------------+         +---------+---------+
                                      |                   |
                               CheckingAccount     SavingsAccount

+---------------------------+       +---------------------------+
|       Transaction         |       |   BankNetworkService      |
|---------------------------|       |      <<interface>>        |
| - transactionId: String   |       |---------------------------|
| - type: TransactionType   |       | + validatePin(...):Result |
| - amount: long            |       | + getAccount(...):Account |
| - timestamp: Instant      |       | + debit(...):Result       |
| - balanceBefore: long     |       | + credit(...):Result      |
| - balanceAfter: long      |       | + transfer(...):Result    |
+---------------------------+       | + getTransactionHistory() |
                                    +---------------------------+
                                               ^
                                               |
                                  MockBankNetworkService
```

---

## 7. Core Classes — Complete Java Implementation

### 7.1 TransactionType Enum

```java
package atm;

public enum TransactionType {
    BALANCE_INQUIRY,
    WITHDRAWAL,
    DEPOSIT,
    TRANSFER_OUT,
    TRANSFER_IN
}
```

---

### 7.2 Transaction (Immutable Value Object)

```java
package atm;

import java.time.Instant;
import java.util.UUID;

public final class Transaction {

    private final String transactionId;
    private final TransactionType type;
    private final long amountCents;       // always positive
    private final Instant timestamp;
    private final long balanceBeforeCents;
    private final long balanceAfterCents;
    private final String referenceAccountId; // non-null for transfers

    public Transaction(TransactionType type,
                       long amountCents,
                       long balanceBeforeCents,
                       long balanceAfterCents,
                       String referenceAccountId) {
        this.transactionId       = UUID.randomUUID().toString();
        this.type                = type;
        this.amountCents         = amountCents;
        this.timestamp           = Instant.now();
        this.balanceBeforeCents  = balanceBeforeCents;
        this.balanceAfterCents   = balanceAfterCents;
        this.referenceAccountId  = referenceAccountId;
    }

    public String getTransactionId()        { return transactionId; }
    public TransactionType getType()        { return type; }
    public long getAmountCents()            { return amountCents; }
    public Instant getTimestamp()           { return timestamp; }
    public long getBalanceBeforeCents()     { return balanceBeforeCents; }
    public long getBalanceAfterCents()      { return balanceAfterCents; }
    public String getReferenceAccountId()   { return referenceAccountId; }

    @Override
    public String toString() {
        return String.format("[%s] %s: $%.2f  (before=$%.2f  after=$%.2f)%s",
                timestamp,
                type,
                amountCents / 100.0,
                balanceBeforeCents / 100.0,
                balanceAfterCents / 100.0,
                referenceAccountId != null ? "  ref=" + referenceAccountId : "");
    }
}
```

---

### 7.3 Account Hierarchy

```java
package atm;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public abstract class Account {

    private final String accountId;
    private final String ownerId;
    private long balanceCents;
    private final List<Transaction> transactions;

    protected Account(String accountId, String ownerId, long initialBalanceCents) {
        if (initialBalanceCents < 0) {
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        this.accountId    = accountId;
        this.ownerId      = ownerId;
        this.balanceCents = initialBalanceCents;
        this.transactions = new ArrayList<>();
    }

    public String getAccountId() { return accountId; }
    public String getOwnerId()   { return ownerId; }
    public long getBalanceCents() { return balanceCents; }

    /**
     * Credit the account (deposit or incoming transfer).
     * Records a transaction and updates balance.
     */
    public void credit(long amountCents, TransactionType type, String referenceAccountId) {
        if (amountCents <= 0) {
            throw new IllegalArgumentException("Credit amount must be positive");
        }
        long before = balanceCents;
        balanceCents += amountCents;
        transactions.add(new Transaction(type, amountCents, before, balanceCents, referenceAccountId));
    }

    /**
     * Debit the account (withdrawal or outgoing transfer).
     * Validates sufficient funds first; subclasses can add overdraft rules.
     */
    public void debit(long amountCents, TransactionType type, String referenceAccountId) {
        if (amountCents <= 0) {
            throw new IllegalArgumentException("Debit amount must be positive");
        }
        validateSufficientFunds(amountCents);
        long before = balanceCents;
        balanceCents -= amountCents;
        transactions.add(new Transaction(type, amountCents, before, balanceCents, referenceAccountId));
    }

    /**
     * Hook for subclasses to override overdraft policies.
     */
    protected void validateSufficientFunds(long amountCents) {
        if (amountCents > balanceCents) {
            throw new InsufficientFundsException(
                    "Requested $" + (amountCents / 100.0) +
                    " but available balance is $" + (balanceCents / 100.0));
        }
    }

    public List<Transaction> getTransactions() {
        return Collections.unmodifiableList(transactions);
    }

    public List<Transaction> getLastNTransactions(int n) {
        int size  = transactions.size();
        int start = Math.max(0, size - n);
        return Collections.unmodifiableList(transactions.subList(start, size));
    }

    @Override
    public String toString() {
        return getClass().getSimpleName() + "[" + accountId + "] balance=$" + (balanceCents / 100.0);
    }
}
```

```java
package atm;

public class CheckingAccount extends Account {

    // Checking accounts allow a small overdraft up to this limit (in cents)
    private static final long OVERDRAFT_LIMIT_CENTS = 10000L; // $100

    public CheckingAccount(String accountId, String ownerId, long initialBalanceCents) {
        super(accountId, ownerId, initialBalanceCents);
    }

    @Override
    protected void validateSufficientFunds(long amountCents) {
        if (amountCents > getBalanceCents() + OVERDRAFT_LIMIT_CENTS) {
            throw new InsufficientFundsException(
                    "Requested $" + (amountCents / 100.0) +
                    " exceeds balance + overdraft limit ($" +
                    ((getBalanceCents() + OVERDRAFT_LIMIT_CENTS) / 100.0) + ")");
        }
    }
}
```

```java
package atm;

public class SavingsAccount extends Account {

    // Savings accounts do not permit any overdraft
    public SavingsAccount(String accountId, String ownerId, long initialBalanceCents) {
        super(accountId, ownerId, initialBalanceCents);
    }

    // validateSufficientFunds inherited from Account — no overdraft allowed
}
```

---

### 7.4 InsufficientFundsException

```java
package atm;

public class InsufficientFundsException extends RuntimeException {
    public InsufficientFundsException(String message) {
        super(message);
    }
}
```

---

### 7.5 Card

```java
package atm;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

/**
 * Represents a physical ATM card.
 *
 * Security note: In production, PINs should be hashed with bcrypt/Argon2
 * with a per-card salt. SHA-256 is used here for simplicity and clarity.
 * The raw PIN is never stored after hashing.
 */
public class Card {

    private static final int MAX_FAILED_ATTEMPTS = 3;

    private final String cardNumber;
    private final String pinHash;          // SHA-256 hex of raw PIN
    private final String linkedAccountId;
    private boolean locked;
    private int failedAttempts;

    public Card(String cardNumber, String rawPin, String linkedAccountId) {
        this.cardNumber        = cardNumber;
        this.pinHash           = sha256(rawPin);
        this.linkedAccountId   = linkedAccountId;
        this.locked            = false;
        this.failedAttempts    = 0;
    }

    public String getCardNumber()      { return cardNumber; }
    public String getLinkedAccountId() { return linkedAccountId; }
    public boolean isLocked()          { return locked; }
    public int getFailedAttempts()     { return failedAttempts; }

    /**
     * Validates the supplied PIN against the stored hash.
     * On failure, increments the attempt counter and locks the card
     * if the maximum is reached.
     *
     * @return true if the PIN matches AND the card is not locked
     */
    public boolean validatePin(String rawPin) {
        if (locked) {
            return false;
        }
        boolean matches = sha256(rawPin).equals(pinHash);
        if (matches) {
            resetFailedAttempts();
        } else {
            incrementFailedAttempts();
        }
        return matches;
    }

    public void incrementFailedAttempts() {
        failedAttempts++;
        if (failedAttempts >= MAX_FAILED_ATTEMPTS) {
            lock();
        }
    }

    public void lock() {
        this.locked = true;
        System.out.println("[SECURITY] Card " + cardNumber + " has been locked after " +
                failedAttempts + " failed PIN attempt(s).");
    }

    public void resetFailedAttempts() {
        this.failedAttempts = 0;
    }

    /** Unlocked by bank staff (not ATM logic). */
    public void unlock() {
        this.locked        = false;
        this.failedAttempts = 0;
    }

    private static String sha256(String input) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            StringBuilder hex = new StringBuilder();
            for (byte b : hash) {
                hex.append(String.format("%02x", b));
            }
            return hex.toString();
        } catch (NoSuchAlgorithmException e) {
            // SHA-256 is guaranteed by the Java spec; this cannot happen
            throw new RuntimeException("SHA-256 not available", e);
        }
    }

    @Override
    public String toString() {
        return "Card[" + cardNumber + ", locked=" + locked + "]";
    }
}
```

---

### 7.6 CashDispenser

```java
package atm;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Models the physical cash dispensing unit of an ATM.
 *
 * Denominations are stored in descending order so the greedy algorithm
 * naturally picks the largest bill first, minimizing the number of bills.
 */
public class CashDispenser {

    // Key = denomination in cents (e.g. 10000 = $100), Value = count
    private final LinkedHashMap<Integer, Integer> denominations;

    public CashDispenser() {
        denominations = new LinkedHashMap<>();
        // Descending denomination order
        denominations.put(10000, 20);  // $100 x 20 = $2,000
        denominations.put(5000,  20);  // $50  x 20 = $1,000
        denominations.put(2000,  20);  // $20  x 20 = $  400
        denominations.put(1000,  20);  // $10  x 20 = $  200
    }

    /**
     * Returns the total cash available in the dispenser in cents.
     */
    public long getAvailableCashCents() {
        long total = 0;
        for (Map.Entry<Integer, Integer> entry : denominations.entrySet()) {
            total += (long) entry.getKey() * entry.getValue();
        }
        return total;
    }

    /**
     * Returns true if the exact amount can be dispensed with current stock.
     * Uses greedy (largest first); the amount must be an exact match.
     */
    public boolean canDispense(long amountCents) {
        if (amountCents <= 0) {
            return false;
        }
        long remaining = amountCents;
        for (Map.Entry<Integer, Integer> entry : denominations.entrySet()) {
            int denom = entry.getKey();
            int count = entry.getValue();
            int needed = (int) Math.min(remaining / denom, count);
            remaining -= (long) needed * denom;
            if (remaining == 0) {
                return true;
            }
        }
        return remaining == 0;
    }

    /**
     * Physically dispenses the given amount.
     * Deducts the appropriate bills from inventory.
     * Throws IllegalArgumentException if the amount cannot be dispensed.
     *
     * @return a map of denomination (cents) → bills dispensed, for receipt printing
     */
    public Map<Integer, Integer> dispense(long amountCents) {
        if (!canDispense(amountCents)) {
            throw new IllegalArgumentException(
                    "Cannot dispense $" + (amountCents / 100.0) +
                    " with available denominations and stock");
        }
        Map<Integer, Integer> dispensed = new LinkedHashMap<>();
        long remaining = amountCents;

        for (Map.Entry<Integer, Integer> entry : denominations.entrySet()) {
            int denom = entry.getKey();
            int count = entry.getValue();
            int needed = (int) Math.min(remaining / denom, count);
            if (needed > 0) {
                dispensed.put(denom, needed);
                denominations.put(denom, count - needed);
                remaining -= (long) needed * denom;
            }
            if (remaining == 0) {
                break;
            }
        }

        System.out.println("[DISPENSER] Dispensed: " + formatDispensed(dispensed));
        return dispensed;
    }

    /**
     * Refills a denomination (performed by ATM technician).
     */
    public void refill(int denominationCents, int count) {
        denominations.merge(denominationCents, count, Integer::sum);
    }

    private String formatDispensed(Map<Integer, Integer> dispensed) {
        StringBuilder sb = new StringBuilder();
        for (Map.Entry<Integer, Integer> entry : dispensed.entrySet()) {
            sb.append(entry.getValue()).append(" x $").append(entry.getKey() / 100).append("  ");
        }
        return sb.toString().trim();
    }
}
```

---

### 7.7 BankNetworkService Interface and Result Type

```java
package atm;

/**
 * Generic result wrapper for all bank network calls.
 * Allows callers to distinguish success from typed failures
 * without using exceptions for control flow.
 */
public class NetworkResult {

    public enum Status { SUCCESS, INSUFFICIENT_FUNDS, ACCOUNT_NOT_FOUND, CARD_LOCKED, NETWORK_ERROR }

    private final Status status;
    private final String message;

    private NetworkResult(Status status, String message) {
        this.status  = status;
        this.message = message;
    }

    public static NetworkResult success() {
        return new NetworkResult(Status.SUCCESS, "OK");
    }

    public static NetworkResult failure(Status status, String message) {
        return new NetworkResult(status, message);
    }

    public boolean isSuccess() { return status == Status.SUCCESS; }
    public Status getStatus()  { return status; }
    public String getMessage() { return message; }

    @Override
    public String toString() {
        return "NetworkResult[" + status + ": " + message + "]";
    }
}
```

```java
package atm;

import java.util.List;

/**
 * Abstracts all communication between the ATM and the bank's back-end system.
 *
 * Every method returns a NetworkResult so callers can handle failures
 * without catching exceptions from network I/O.
 */
public interface BankNetworkService {

    /** Validate the PIN for a given card number. Returns SUCCESS or CARD_LOCKED / NETWORK_ERROR. */
    NetworkResult validatePin(String cardNumber, String rawPin);

    /**
     * Fetch the account linked to a card.
     * Returns null on the account object if not found (check NetworkResult first).
     */
    AccountLookupResult getAccount(String accountId);

    /** Debit (withdraw) an amount from the account. */
    NetworkResult debit(String accountId, long amountCents, TransactionType type, String referenceId);

    /** Credit (deposit) an amount to the account. */
    NetworkResult credit(String accountId, long amountCents, TransactionType type, String referenceId);

    /** Transfer amountCents from one account to another atomically. */
    NetworkResult transfer(String fromAccountId, String toAccountId, long amountCents);

    /** Retrieve transaction history for an account. */
    List<Transaction> getTransactionHistory(String accountId, int limit);

    // -------------------------------------------------------------------------
    // Helper wrapper returned by getAccount()
    // -------------------------------------------------------------------------
    class AccountLookupResult {
        private final NetworkResult result;
        private final Account account;

        public AccountLookupResult(NetworkResult result, Account account) {
            this.result  = result;
            this.account = account;
        }

        public NetworkResult getResult()  { return result; }
        public Account getAccount()       { return account; }
    }
}
```

---

### 7.8 MockBankNetworkService

```java
package atm;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * In-memory implementation of BankNetworkService used for testing and demos.
 * Stores accounts and cards in local maps — no actual network calls.
 */
public class MockBankNetworkService implements BankNetworkService {

    private final Map<String, Account> accounts = new HashMap<>();
    private final Map<String, Card>    cards     = new HashMap<>();

    // -------------------------------------------------------------------------
    // Seed methods (called at setup time)
    // -------------------------------------------------------------------------

    public void addAccount(Account account) {
        accounts.put(account.getAccountId(), account);
    }

    public void addCard(Card card) {
        cards.put(card.getCardNumber(), card);
    }

    // -------------------------------------------------------------------------
    // BankNetworkService implementation
    // -------------------------------------------------------------------------

    @Override
    public NetworkResult validatePin(String cardNumber, String rawPin) {
        Card card = cards.get(cardNumber);
        if (card == null) {
            return NetworkResult.failure(NetworkResult.Status.ACCOUNT_NOT_FOUND,
                    "Card not found: " + cardNumber);
        }
        if (card.isLocked()) {
            return NetworkResult.failure(NetworkResult.Status.CARD_LOCKED,
                    "Card is locked");
        }
        boolean valid = card.validatePin(rawPin);
        if (valid) {
            return NetworkResult.success();
        }
        // Card.validatePin() has already incremented the counter and possibly locked
        if (card.isLocked()) {
            return NetworkResult.failure(NetworkResult.Status.CARD_LOCKED,
                    "Card is now locked after too many failed attempts");
        }
        return NetworkResult.failure(NetworkResult.Status.NETWORK_ERROR,
                "Incorrect PIN. Attempts remaining: " +
                (Card.MAX_FAILED_ATTEMPTS_PUBLIC - card.getFailedAttempts()));
    }

    @Override
    public AccountLookupResult getAccount(String accountId) {
        Account account = accounts.get(accountId);
        if (account == null) {
            return new AccountLookupResult(
                    NetworkResult.failure(NetworkResult.Status.ACCOUNT_NOT_FOUND,
                            "Account not found: " + accountId),
                    null);
        }
        return new AccountLookupResult(NetworkResult.success(), account);
    }

    @Override
    public NetworkResult debit(String accountId, long amountCents,
                               TransactionType type, String referenceId) {
        Account account = accounts.get(accountId);
        if (account == null) {
            return NetworkResult.failure(NetworkResult.Status.ACCOUNT_NOT_FOUND,
                    "Account not found: " + accountId);
        }
        try {
            account.debit(amountCents, type, referenceId);
            return NetworkResult.success();
        } catch (InsufficientFundsException e) {
            return NetworkResult.failure(NetworkResult.Status.INSUFFICIENT_FUNDS, e.getMessage());
        }
    }

    @Override
    public NetworkResult credit(String accountId, long amountCents,
                                TransactionType type, String referenceId) {
        Account account = accounts.get(accountId);
        if (account == null) {
            return NetworkResult.failure(NetworkResult.Status.ACCOUNT_NOT_FOUND,
                    "Account not found: " + accountId);
        }
        account.credit(amountCents, type, referenceId);
        return NetworkResult.success();
    }

    @Override
    public NetworkResult transfer(String fromAccountId, String toAccountId, long amountCents) {
        Account from = accounts.get(fromAccountId);
        Account to   = accounts.get(toAccountId);

        if (from == null) {
            return NetworkResult.failure(NetworkResult.Status.ACCOUNT_NOT_FOUND,
                    "Source account not found: " + fromAccountId);
        }
        if (to == null) {
            return NetworkResult.failure(NetworkResult.Status.ACCOUNT_NOT_FOUND,
                    "Destination account not found: " + toAccountId);
        }
        try {
            from.debit(amountCents,  TransactionType.TRANSFER_OUT, toAccountId);
            to.credit(amountCents,   TransactionType.TRANSFER_IN,  fromAccountId);
            return NetworkResult.success();
        } catch (InsufficientFundsException e) {
            return NetworkResult.failure(NetworkResult.Status.INSUFFICIENT_FUNDS, e.getMessage());
        }
    }

    @Override
    public List<Transaction> getTransactionHistory(String accountId, int limit) {
        Account account = accounts.get(accountId);
        if (account == null) {
            return new ArrayList<>();
        }
        return account.getLastNTransactions(limit);
    }
}
```

> **Note**: `Card.MAX_FAILED_ATTEMPTS` needs to be package-accessible. Add the following to `Card.java`:
> ```java
> // Add this constant alongside MAX_FAILED_ATTEMPTS in Card.java
> public static final int MAX_FAILED_ATTEMPTS_PUBLIC = MAX_FAILED_ATTEMPTS;
> ```

---

### 7.9 ATMState Interface

```java
package atm;

/**
 * The State interface for the ATM state machine.
 *
 * Every method corresponds to a customer-visible or system-driven action.
 * Implementations must:
 *  - Execute the action if it is valid in this state.
 *  - Throw IllegalStateException with a clear message if it is not valid.
 *  - Transition the ATM to the next appropriate state upon success.
 */
public interface ATMState {

    void insertCard(ATM atm, Card card);

    void enterPin(ATM atm, String rawPin);

    void checkBalance(ATM atm);

    void withdraw(ATM atm, long amountCents);

    void deposit(ATM atm, long amountCents);

    void transfer(ATM atm, String toAccountId, long amountCents);

    void ejectCard(ATM atm);
}
```

---

### 7.10 IdleState

```java
package atm;

/**
 * The ATM is waiting for a card.
 * The only valid operation is insertCard().
 */
public class IdleState implements ATMState {

    @Override
    public void insertCard(ATM atm, Card card) {
        if (card == null) {
            throw new IllegalArgumentException("Card cannot be null");
        }
        if (card.isLocked()) {
            atm.getScreen().display("Card is locked. Please contact your bank.");
            return;
        }
        atm.getCardReader().insertCard(card);
        atm.getScreen().display("Card accepted. Please enter your PIN.");
        atm.setCurrentState(new CardInsertedState());
    }

    @Override
    public void enterPin(ATM atm, String rawPin) {
        throw new IllegalStateException("Please insert a card before entering a PIN.");
    }

    @Override
    public void checkBalance(ATM atm) {
        throw new IllegalStateException("Please insert a card and authenticate first.");
    }

    @Override
    public void withdraw(ATM atm, long amountCents) {
        throw new IllegalStateException("Please insert a card and authenticate first.");
    }

    @Override
    public void deposit(ATM atm, long amountCents) {
        throw new IllegalStateException("Please insert a card and authenticate first.");
    }

    @Override
    public void transfer(ATM atm, String toAccountId, long amountCents) {
        throw new IllegalStateException("Please insert a card and authenticate first.");
    }

    @Override
    public void ejectCard(ATM atm) {
        atm.getScreen().display("No card inserted.");
        // No state transition needed; already idle
    }
}
```

---

### 7.11 CardInsertedState

```java
package atm;

/**
 * A card has been inserted.
 * The only valid operation is enterPin().
 */
public class CardInsertedState implements ATMState {

    @Override
    public void insertCard(ATM atm, Card card) {
        throw new IllegalStateException("A card is already inserted. Please eject it first.");
    }

    @Override
    public void enterPin(ATM atm, String rawPin) {
        Card card = atm.getCardReader().getCard();
        if (card == null) {
            throw new IllegalStateException("No card in reader — this should not happen in CardInsertedState.");
        }
        atm.getScreen().display("Validating PIN...");
        atm.setCurrentState(new PINEnteredState(rawPin));
        // Immediately trigger validation within the new state
        atm.getCurrentState().enterPin(atm, rawPin);
    }

    @Override
    public void checkBalance(ATM atm) {
        throw new IllegalStateException("Please enter your PIN before performing transactions.");
    }

    @Override
    public void withdraw(ATM atm, long amountCents) {
        throw new IllegalStateException("Please enter your PIN before performing transactions.");
    }

    @Override
    public void deposit(ATM atm, long amountCents) {
        throw new IllegalStateException("Please enter your PIN before performing transactions.");
    }

    @Override
    public void transfer(ATM atm, String toAccountId, long amountCents) {
        throw new IllegalStateException("Please enter your PIN before performing transactions.");
    }

    @Override
    public void ejectCard(ATM atm) {
        atm.getScreen().display("Card ejected.");
        atm.getCardReader().ejectCard();
        atm.setCurrentState(new IdleState());
    }
}
```

---

### 7.12 PINEnteredState

```java
package atm;

/**
 * PIN has been submitted and is being validated against the bank.
 * This is a transient state — it transitions to Authenticated or back to Idle
 * (with card ejected) after processing.
 *
 * enterPin() is called once more by CardInsertedState to trigger actual validation.
 */
public class PINEnteredState implements ATMState {

    private final String rawPin;

    public PINEnteredState(String rawPin) {
        this.rawPin = rawPin;
    }

    @Override
    public void insertCard(ATM atm, Card card) {
        throw new IllegalStateException("Cannot insert card while validating PIN.");
    }

    @Override
    public void enterPin(ATM atm, String pin) {
        Card card = atm.getCardReader().getCard();
        NetworkResult result = atm.getBankService().validatePin(card.getCardNumber(), pin);

        if (result.isSuccess()) {
            // Load the account for the session
            String accountId = card.getLinkedAccountId();
            BankNetworkService.AccountLookupResult lookup = atm.getBankService().getAccount(accountId);
            if (!lookup.getResult().isSuccess()) {
                atm.getScreen().display("Could not load account. Please contact your bank.");
                atm.getCardReader().ejectCard();
                atm.setCurrentState(new IdleState());
                return;
            }
            atm.setCurrentAccount(lookup.getAccount());
            atm.getScreen().display("Authentication successful. Welcome!");
            atm.setCurrentState(new AuthenticatedState());
        } else if (result.getStatus() == NetworkResult.Status.CARD_LOCKED) {
            atm.getScreen().display("Your card has been locked. Please contact your bank.");
            atm.getCardReader().ejectCard();
            atm.setCurrentState(new IdleState());
        } else {
            atm.getScreen().display("Incorrect PIN. " + result.getMessage());
            // Return to CardInsertedState to allow retry (card is still in reader)
            atm.setCurrentState(new CardInsertedState());
        }
    }

    @Override
    public void checkBalance(ATM atm) {
        throw new IllegalStateException("PIN validation in progress. Please wait.");
    }

    @Override
    public void withdraw(ATM atm, long amountCents) {
        throw new IllegalStateException("PIN validation in progress. Please wait.");
    }

    @Override
    public void deposit(ATM atm, long amountCents) {
        throw new IllegalStateException("PIN validation in progress. Please wait.");
    }

    @Override
    public void transfer(ATM atm, String toAccountId, long amountCents) {
        throw new IllegalStateException("PIN validation in progress. Please wait.");
    }

    @Override
    public void ejectCard(ATM atm) {
        atm.getScreen().display("Card ejected.");
        atm.getCardReader().ejectCard();
        atm.setCurrentState(new IdleState());
    }
}
```

---

### 7.13 AuthenticatedState

```java
package atm;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;

/**
 * The customer is fully authenticated.
 * All financial operations are available.
 */
public class AuthenticatedState implements ATMState {

    // Daily withdrawal limit in cents ($1,000)
    private static final long DAILY_LIMIT_CENTS = 100000L;

    // Per-session tracking — in a real system, this would be in a persistent store
    // keyed by cardNumber + date, so it survives ATM restarts.
    private long withdrawnTodayCents = 0L;

    @Override
    public void insertCard(ATM atm, Card card) {
        throw new IllegalStateException("A session is already active. Please eject your card first.");
    }

    @Override
    public void enterPin(ATM atm, String rawPin) {
        throw new IllegalStateException("Already authenticated.");
    }

    @Override
    public void checkBalance(ATM atm) {
        Account account = atm.getCurrentAccount();
        long balanceCents = account.getBalanceCents();
        atm.getScreen().display(String.format("Account: %s | Balance: $%.2f",
                account.getAccountId(), balanceCents / 100.0));
    }

    @Override
    public void withdraw(ATM atm, long amountCents) {
        if (amountCents <= 0) {
            atm.getScreen().display("Withdrawal amount must be positive.");
            return;
        }

        // Check daily limit
        if (withdrawnTodayCents + amountCents > DAILY_LIMIT_CENTS) {
            long remaining = DAILY_LIMIT_CENTS - withdrawnTodayCents;
            atm.getScreen().display(String.format(
                    "Daily limit exceeded. You may withdraw up to $%.2f more today.",
                    remaining / 100.0));
            return;
        }

        // Check dispenser can cover the amount
        if (!atm.getCashDispenser().canDispense(amountCents)) {
            atm.getScreen().display("ATM cannot dispense that amount with available denominations.");
            return;
        }

        // Check account balance via bank
        NetworkResult result = atm.getBankService().debit(
                atm.getCurrentAccount().getAccountId(),
                amountCents,
                TransactionType.WITHDRAWAL,
                null);

        if (!result.isSuccess()) {
            atm.getScreen().display("Withdrawal failed: " + result.getMessage());
            return;
        }

        // Transition to dispensing state — physically dispense the cash
        atm.setCurrentState(new DispensingState(amountCents, this));
        atm.getCurrentState().withdraw(atm, amountCents);

        // After dispensing, we return to authenticated for further transactions
        withdrawnTodayCents += amountCents;
    }

    @Override
    public void deposit(ATM atm, long amountCents) {
        if (amountCents <= 0) {
            atm.getScreen().display("Deposit amount must be positive.");
            return;
        }
        NetworkResult result = atm.getBankService().credit(
                atm.getCurrentAccount().getAccountId(),
                amountCents,
                TransactionType.DEPOSIT,
                null);

        if (result.isSuccess()) {
            atm.getScreen().display(String.format("Deposit of $%.2f accepted. New balance: $%.2f",
                    amountCents / 100.0,
                    atm.getCurrentAccount().getBalanceCents() / 100.0));
        } else {
            atm.getScreen().display("Deposit failed: " + result.getMessage());
        }
    }

    @Override
    public void transfer(ATM atm, String toAccountId, long amountCents) {
        if (amountCents <= 0) {
            atm.getScreen().display("Transfer amount must be positive.");
            return;
        }
        if (toAccountId == null || toAccountId.isBlank()) {
            atm.getScreen().display("Destination account ID is required.");
            return;
        }

        String fromAccountId = atm.getCurrentAccount().getAccountId();
        if (fromAccountId.equals(toAccountId)) {
            atm.getScreen().display("Cannot transfer to the same account.");
            return;
        }

        NetworkResult result = atm.getBankService().transfer(fromAccountId, toAccountId, amountCents);

        if (result.isSuccess()) {
            atm.getScreen().display(String.format(
                    "Transferred $%.2f to account %s. New balance: $%.2f",
                    amountCents / 100.0,
                    toAccountId,
                    atm.getCurrentAccount().getBalanceCents() / 100.0));
        } else {
            atm.getScreen().display("Transfer failed: " + result.getMessage());
        }
    }

    @Override
    public void ejectCard(ATM atm) {
        atm.getScreen().display("Thank you. Card ejected. Have a great day!");
        atm.getCardReader().ejectCard();
        atm.setCurrentAccount(null);
        atm.setCurrentState(new IdleState());
    }

    public long getWithdrawnTodayCents() {
        return withdrawnTodayCents;
    }
}
```

---

### 7.14 DispensingState

```java
package atm;

import java.util.Map;

/**
 * The ATM is physically dispensing cash.
 * This state is system-driven (not directly triggered by user input after the
 * withdrawal was already authorized in AuthenticatedState).
 *
 * Once dispensing completes (or fails), the machine returns to AuthenticatedState.
 */
public class DispensingState implements ATMState {

    private final long amountCents;
    private final AuthenticatedState returnState;

    public DispensingState(long amountCents, AuthenticatedState returnState) {
        this.amountCents = amountCents;
        this.returnState = returnState;
    }

    @Override
    public void insertCard(ATM atm, Card card) {
        throw new IllegalStateException("Cash dispensing in progress.");
    }

    @Override
    public void enterPin(ATM atm, String rawPin) {
        throw new IllegalStateException("Cash dispensing in progress.");
    }

    @Override
    public void checkBalance(ATM atm) {
        throw new IllegalStateException("Cash dispensing in progress.");
    }

    @Override
    public void withdraw(ATM atm, long amount) {
        try {
            Map<Integer, Integer> dispensed = atm.getCashDispenser().dispense(amountCents);
            atm.getScreen().display(String.format(
                    "Please collect $%.2f. %s",
                    amountCents / 100.0,
                    formatDispensed(dispensed)));
        } catch (IllegalArgumentException e) {
            // Dispenser cannot fulfill — this should have been caught in AuthenticatedState,
            // but we handle it defensively here.
            atm.getScreen().display("Dispensing error: " + e.getMessage());
            // Reversal would be needed in a real system; omitted for clarity
        } finally {
            // Always return to authenticated state after dispensing attempt
            atm.setCurrentState(returnState);
        }
    }

    @Override
    public void deposit(ATM atm, long amountCents) {
        throw new IllegalStateException("Cash dispensing in progress.");
    }

    @Override
    public void transfer(ATM atm, String toAccountId, long amountCents) {
        throw new IllegalStateException("Cash dispensing in progress.");
    }

    @Override
    public void ejectCard(ATM atm) {
        // Emergency eject — return card, go idle, dispenser stops mid-cycle
        atm.getScreen().display("Session ended. Card ejected.");
        atm.getCardReader().ejectCard();
        atm.setCurrentAccount(null);
        atm.setCurrentState(new IdleState());
    }

    private String formatDispensed(Map<Integer, Integer> dispensed) {
        StringBuilder sb = new StringBuilder("Bills: ");
        for (Map.Entry<Integer, Integer> entry : dispensed.entrySet()) {
            sb.append(entry.getValue()).append("x$").append(entry.getKey() / 100).append(" ");
        }
        return sb.toString().trim();
    }
}
```

---

### 7.15 CardReader

```java
package atm;

/**
 * Represents the physical card reader slot of the ATM.
 */
public class CardReader {

    private Card currentCard;

    public void insertCard(Card card) {
        if (currentCard != null) {
            throw new IllegalStateException("A card is already in the reader.");
        }
        this.currentCard = card;
        System.out.println("[CARD READER] Card inserted: " + card.getCardNumber());
    }

    public Card getCard() {
        return currentCard;
    }

    public void ejectCard() {
        if (currentCard != null) {
            System.out.println("[CARD READER] Card ejected: " + currentCard.getCardNumber());
        }
        this.currentCard = null;
    }
}
```

---

### 7.16 Screen and Keypad (I/O Abstractions)

```java
package atm;

/**
 * Represents the ATM display screen.
 * In a real system this would drive actual hardware or a GUI component.
 */
public class Screen {

    public void display(String message) {
        System.out.println("[SCREEN] " + message);
    }

    public void displayMenu(String... options) {
        System.out.println("[SCREEN] --- MENU ---");
        for (int i = 0; i < options.length; i++) {
            System.out.println("[SCREEN] " + (i + 1) + ". " + options[i]);
        }
    }
}
```

```java
package atm;

import java.util.Scanner;

/**
 * Represents the ATM keypad.
 * In a real system this would receive hardware input events.
 * The Scanner is used only for interactive demos; unit tests bypass this entirely.
 */
public class Keypad {

    private final Scanner scanner;

    public Keypad() {
        this.scanner = new Scanner(System.in);
    }

    public String readLine(String prompt) {
        System.out.print("[KEYPAD PROMPT] " + prompt + " > ");
        return scanner.nextLine().trim();
    }

    public long readAmountCents(String prompt) {
        System.out.print("[KEYPAD PROMPT] " + prompt + " (enter dollars, e.g. 100) > ");
        try {
            double dollars = Double.parseDouble(scanner.nextLine().trim());
            return Math.round(dollars * 100);
        } catch (NumberFormatException e) {
            return 0L;
        }
    }
}
```

---

### 7.17 ATM — The Context Class

```java
package atm;

/**
 * The ATM class is the State Pattern "Context".
 *
 * It:
 *  - Holds a reference to the current ATMState
 *  - Delegates every public operation to the current state
 *  - Provides accessors so states can read/modify ATM internals
 *    without exposing them publicly
 *
 * ATM does NOT contain any business logic — it only routes calls.
 */
public class ATM {

    private ATMState currentState;
    private final CardReader cardReader;
    private final CashDispenser cashDispenser;
    private final BankNetworkService bankService;
    private final Screen screen;
    private final Keypad keypad;

    // Set by PINEnteredState after successful authentication; cleared on ejectCard
    private Account currentAccount;

    public ATM(BankNetworkService bankService) {
        this.bankService    = bankService;
        this.cardReader     = new CardReader();
        this.cashDispenser  = new CashDispenser();
        this.screen         = new Screen();
        this.keypad         = new Keypad();
        this.currentState   = new IdleState();   // start in Idle
    }

    // -------------------------------------------------------------------------
    // Public API — all delegate to the current state
    // -------------------------------------------------------------------------

    public void insertCard(Card card) {
        currentState.insertCard(this, card);
    }

    public void enterPin(String rawPin) {
        currentState.enterPin(this, rawPin);
    }

    public void checkBalance() {
        currentState.checkBalance(this);
    }

    public void withdraw(long amountCents) {
        currentState.withdraw(this, amountCents);
    }

    public void deposit(long amountCents) {
        currentState.deposit(this, amountCents);
    }

    public void transfer(String toAccountId, long amountCents) {
        currentState.transfer(this, toAccountId, amountCents);
    }

    public void ejectCard() {
        currentState.ejectCard(this);
    }

    // -------------------------------------------------------------------------
    // Package-private accessors for state classes
    // -------------------------------------------------------------------------

    ATMState getCurrentState()            { return currentState; }
    void setCurrentState(ATMState state)  { this.currentState = state; }

    CardReader getCardReader()            { return cardReader; }
    CashDispenser getCashDispenser()      { return cashDispenser; }
    BankNetworkService getBankService()   { return bankService; }
    Screen getScreen()                    { return screen; }
    Keypad getKeypad()                    { return keypad; }

    Account getCurrentAccount()           { return currentAccount; }
    void setCurrentAccount(Account a)     { this.currentAccount = a; }
}
```

---

### 7.18 Main Demo

```java
package atm;

import java.util.List;

/**
 * End-to-end demo:
 *  Insert card → validate PIN (including wrong attempt) → check balance →
 *  withdraw $200 → transfer $100 → view history → eject card
 */
public class ATMDemo {

    public static void main(String[] args) {

        // ------------------------------------------------------------------
        // 1. Set up the mock bank with two accounts and a card
        // ------------------------------------------------------------------
        MockBankNetworkService bankService = new MockBankNetworkService();

        SavingsAccount savings   = new SavingsAccount("ACC-001", "Alice", 150000L); // $1,500.00
        CheckingAccount checking = new CheckingAccount("ACC-002", "Bob",  80000L);  // $800.00

        Card aliceCard = new Card("4111111111111111", "1234", "ACC-001");

        bankService.addAccount(savings);
        bankService.addAccount(checking);
        bankService.addCard(aliceCard);

        // ------------------------------------------------------------------
        // 2. Create the ATM
        // ------------------------------------------------------------------
        ATM atm = new ATM(bankService);

        System.out.println("\n============================================================");
        System.out.println("  ATM SESSION DEMO");
        System.out.println("============================================================\n");

        // ------------------------------------------------------------------
        // 3. Insert card
        // ------------------------------------------------------------------
        System.out.println("--- Step 1: Insert card ---");
        atm.insertCard(aliceCard);

        // ------------------------------------------------------------------
        // 4. Enter wrong PIN (should count as failed attempt)
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 2: Enter wrong PIN ---");
        atm.enterPin("9999");

        // ------------------------------------------------------------------
        // 5. Enter correct PIN
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 3: Enter correct PIN ---");
        atm.enterPin("1234");

        // ------------------------------------------------------------------
        // 6. Check balance
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 4: Check balance ---");
        atm.checkBalance();

        // ------------------------------------------------------------------
        // 7. Withdraw $200
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 5: Withdraw $200 ---");
        atm.withdraw(20000L);  // 20000 cents = $200.00

        // ------------------------------------------------------------------
        // 8. Check balance again to confirm deduction
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 6: Check balance after withdrawal ---");
        atm.checkBalance();

        // ------------------------------------------------------------------
        // 9. Transfer $100 to Bob's checking account
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 7: Transfer $100 to ACC-002 ---");
        atm.transfer("ACC-002", 10000L);  // 10000 cents = $100.00

        // ------------------------------------------------------------------
        // 10. View transaction history
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 8: Transaction history (last 5) ---");
        List<Transaction> history = bankService.getTransactionHistory("ACC-001", 5);
        for (Transaction tx : history) {
            System.out.println("  " + tx);
        }

        // ------------------------------------------------------------------
        // 11. Try to withdraw more than daily limit (to demo enforcement)
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 9: Attempt to exceed daily limit ($900 more) ---");
        atm.withdraw(90000L);  // Total would be $200 + $900 = $1,100 > $1,000 daily limit

        // ------------------------------------------------------------------
        // 12. Eject card
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 10: Eject card ---");
        atm.ejectCard();

        // ------------------------------------------------------------------
        // 13. Verify machine is idle — any operation should now fail gracefully
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 11: Try to check balance after session (should fail) ---");
        try {
            atm.checkBalance();
        } catch (IllegalStateException e) {
            System.out.println("[EXPECTED] " + e.getMessage());
        }

        // ------------------------------------------------------------------
        // 14. Demo card lockout (3 wrong PINs)
        // ------------------------------------------------------------------
        System.out.println("\n--- Step 12: Demo card lockout ---");
        Card bobCard = new Card("5500005555555559", "5678", "ACC-002");
        bankService.addCard(bobCard);

        atm.insertCard(bobCard);
        atm.enterPin("0000");  // wrong
        atm.enterPin("1111");  // wrong
        // Third attempt locks the card — insertCard returns to Idle
        atm.enterPin("2222");  // wrong — card is now locked and ejected

        // Trying to use the locked card again
        System.out.println("\n--- Step 13: Try to use locked card ---");
        atm.insertCard(bobCard); // Should display "Card is locked"

        System.out.println("\n============================================================");
        System.out.println("  DEMO COMPLETE");
        System.out.println("============================================================");
    }
}
```

---

## 8. Design Patterns Used

### 8.1 State Pattern (Core)

**What it is**: Encapsulates each state of the ATM in its own class. The ATM context object holds a reference to the current state and delegates all operations to it.

**Why used here**:
- An ATM has 5 distinct behavioral modes. Without the State pattern, every method (withdraw, deposit, etc.) would contain a giant `if/switch` on the current mode — fragile, hard to test, and violates Single Responsibility.
- With State, adding a new state (e.g., `MaintenanceState`) requires only a new class; no existing state class needs to change.
- Invalid operations throw `IllegalStateException` inside the state itself, so the protection is co-located with the behavior definition.

**Classes**: `ATMState`, `IdleState`, `CardInsertedState`, `PINEnteredState`, `AuthenticatedState`, `DispensingState`

---

### 8.2 Template Method Pattern (Account hierarchy)

**What it is**: `Account` defines the algorithm for `debit()` (validate funds → update balance → record transaction) and provides a hook — `validateSufficientFunds()` — that subclasses override.

**Why used here**:
- `CheckingAccount` needs overdraft tolerance; `SavingsAccount` does not. The debit workflow is otherwise identical.
- Avoids code duplication while allowing each account type to plug in its own validation rule.

**Classes**: `Account` (template), `CheckingAccount`, `SavingsAccount` (hooks)

---

### 8.3 Strategy Pattern (BankNetworkService)

**What it is**: `BankNetworkService` is an interface. The ATM is programmed to the interface, not the implementation. At runtime you inject `MockBankNetworkService` (tests/demo) or a real `HttpBankNetworkService` (production).

**Why used here**:
- The ATM's behavior must not change based on whether it talks to a local mock or a real bank host.
- Enables unit testing without a real network.
- Enables swapping bank protocols (REST → message queue) with zero ATM code changes.

**Classes**: `BankNetworkService` (strategy interface), `MockBankNetworkService` (concrete strategy)

---

### 8.4 Value Object Pattern (Transaction)

**What it is**: `Transaction` is immutable — all fields are set in the constructor and there are no setters.

**Why used here**:
- Financial records must never be mutated after creation (audit trail integrity).
- Immutable objects are inherently thread-safe.

---

## 9. Key Design Decisions and Trade-offs

### Decision 1: Amounts stored as `long` cents, not `double`

**Decision**: All monetary amounts (balance, withdrawal, etc.) are stored and computed as `long` integers representing cents.

**Why**: `double` cannot represent most decimal fractions exactly (e.g., `0.1 + 0.2 != 0.3` in IEEE 754). For money, this causes silent rounding errors. Using `long` cents makes all arithmetic exact.

**Trade-off**: Display formatting requires dividing by 100. All callers must agree on the "cents" convention.

---

### Decision 2: PIN stored as SHA-256 hash, not plain text

**Decision**: `Card` stores `sha256(rawPin)`. The raw PIN is discarded after construction.

**Why**: Storing plain-text PINs is a critical security flaw. Even if the in-memory store is compromised, the attacker cannot reverse a hash to get the original PIN.

**Trade-off chosen for clarity**: SHA-256 without a salt is used. A production system would use bcrypt, scrypt, or Argon2 with a random per-card salt to prevent rainbow table attacks. This is called out explicitly as a design note in `Card.java`.

---

### Decision 3: State transitions happen inside state classes

**Decision**: Each state is responsible for calling `atm.setCurrentState(newState)`. The ATM context does not decide transitions.

**Why**: The transition logic is part of the behavioral contract of each state. If the ATM managed transitions, it would need to understand what every state does — defeating the purpose of the pattern.

**Trade-off**: States need access to `atm` to call `setCurrentState`. This makes them slightly coupled to the ATM class, but the coupling is one-directional and confined to the state package.

---

### Decision 4: DispensingState is entered by AuthenticatedState

**Decision**: The `withdraw()` method in `AuthenticatedState` validates all pre-conditions (daily limit, balance, dispenser), debits the account, then transitions to `DispensingState` to perform the physical dispense.

**Why**: Separates "authorization" (business rules) from "fulfillment" (hardware action). If the dispenser jams, only the `DispensingState` is affected; the debit and business logic are already clean.

**Trade-off**: In a real system, the debit must be reversed if dispensing fails (a two-phase commit or saga pattern). This reversal is omitted here but is noted as a required extension.

---

### Decision 5: NetworkResult wrapper instead of exceptions for network calls

**Decision**: `BankNetworkService` methods return `NetworkResult` rather than throwing exceptions on failure.

**Why**: Network failures (insufficient funds, account not found) are expected, not exceptional. Using exceptions for control flow of expected outcomes is a code smell and makes the happy path harder to read.

**Trade-off**: Callers must explicitly check `result.isSuccess()`. Forgetting to check is a risk — in a real system, a linting rule or sealed-type Result would enforce handling.

---

### Decision 6: Daily limit tracked per session in AuthenticatedState

**Decision**: `withdrawnTodayCents` is a field on `AuthenticatedState`, not persisted.

**Why**: Keeps the in-memory demo self-contained.

**Trade-off**: This resets to zero on every session. A production system must persist the daily total in a database keyed by `(cardNumber, date)` and query it at the start of each withdrawal.

---

## 10. Extension Points

### 10.1 Adding a new state (e.g., MaintenanceMode)

1. Create `MaintenanceState implements ATMState`.
2. Override methods that are valid in maintenance (e.g., `ejectCard()`, a new `refillCash()` admin action).
3. Throw `IllegalStateException` for all customer-facing operations.
4. Add a trigger (e.g., technician key switch) in `IdleState` that transitions to `MaintenanceState`.
5. **No existing state class needs modification.**

---

### 10.2 Adding a new account type (e.g., CreditAccount)

1. Extend `Account`.
2. Override `validateSufficientFunds()` to allow spending up to the credit limit.
3. Register it in `MockBankNetworkService` (or the real bank service).
4. **No ATM or state class needs modification.**

---

### 10.3 Replacing the bank network implementation

1. Implement `BankNetworkService` (e.g., `HttpBankNetworkService` using REST, or `IsoBankNetworkService` using ISO 8583 messages).
2. Inject the new implementation into the `ATM` constructor.
3. **Zero ATM logic changes.**

---

### 10.4 Adding a receipt printer

1. Create a `ReceiptPrinter` class with a `print(Transaction tx)` method.
2. Add it as a field on `ATM`.
3. Call it from `AuthenticatedState.withdraw()` (and `deposit()`, `transfer()`) after a successful operation.
4. **No state interface changes needed** — this is an additive change.

---

### 10.5 Persisting daily withdrawal limits

1. Create a `DailyLimitRepository` interface with `getWithdrawnToday(cardNumber): long` and `addWithdrawal(cardNumber, amount)`.
2. Inject it into `AuthenticatedState` (or into the ATM and passed to the state).
3. Replace the in-memory `withdrawnTodayCents` field with calls to the repository.

---

### 10.6 Adding multi-factor authentication (OTP)

1. Add `sendOTP(ATM atm)` to `ATMState` (default: throw `IllegalStateException`).
2. Add `OTPSentState` between `PINEnteredState` and `AuthenticatedState`.
3. `PINEnteredState` transitions to `OTPSentState` after PIN success.
4. `OTPSentState` validates the OTP and transitions to `AuthenticatedState`.
5. **Existing states do not change behavior.**

---

## 11. Interview Discussion Points

### "Why use the State pattern instead of a big switch statement?"

The State pattern converts a switch-on-mode anti-pattern into polymorphism. With a switch, adding a new state means editing the ATM class and potentially every method. With State, the ATM class never changes — you only add new state classes. The switch also concentrates all behavioral logic in one place, making it harder to test individual states in isolation.

---

### "How would you make this thread-safe for multiple concurrent sessions?"

Today's design is single-threaded per ATM. For a multi-session ATM (e.g., contactless transactions in parallel), you would:
- Make `currentState` an `AtomicReference<ATMState>` and use compare-and-swap for state transitions.
- Use a `ReentrantLock` around `CashDispenser.dispense()` since it modifies shared hardware state.
- Move `currentAccount` and `withdrawnTodayCents` into a `Session` object keyed by session ID.
- Use optimistic locking on account balance updates in the bank service.

---

### "What happens if the machine crashes between debiting the account and dispensing cash?"

This is the classic distributed systems problem of a two-phase operation. The customer is charged but receives no cash.

**Solutions**:
1. **Idempotency tokens**: Generate a unique withdrawal ID before debiting. If dispensing fails, the customer can re-present the token; the bank checks if the token was already settled.
2. **Saga pattern**: Record a pending withdrawal event before debiting. If dispensing fails, a compensating transaction (credit) is issued automatically.
3. **Transaction log**: The ATM writes to a local write-ahead log. On restart, it replays the log and either dispenses or reverses.

This is a high-value interview discussion because it shows you understand the difference between a ACID database transaction and a distributed two-phase operation.

---

### "How do you prevent a customer from withdrawing the same money twice (replay attack)?"

Every withdrawal generates a unique `transactionId`. The bank service must check for duplicate `transactionId` values before processing. If the ATM re-submits (e.g., after a network timeout), the bank recognizes the duplicate and returns the original result without debiting again.

---

### "What if the PIN is brute-forced at the network level?"

`Card.validatePin()` enforces a 3-attempt lockout locally. However, a sophisticated attacker might send forged `validatePin` calls to the `BankNetworkService` directly. Additional defenses:
1. Rate limiting at the bank network layer.
2. Requiring a challenge-response (the ATM sends a random nonce; the PIN response must include a MAC over the nonce).
3. Hardware Security Module (HSM) at the ATM that never exposes the raw PIN to software at all — the PIN is encrypted by the ATM's HSM and decrypted only inside the bank's HSM.

---

### "Why is the balance stored as `long` cents and not `BigDecimal`?"

Both are valid. `long` is simpler and avoids heap allocation for every arithmetic operation. It works for currencies where the smallest unit is 1 cent. `BigDecimal` is necessary if you need arbitrary precision (e.g., cryptocurrency, interest calculations with fractional cents). For a standard bank ATM dealing in whole cents, `long` is idiomatic and sufficient.

---

### "Walk me through a withdrawal. What classes are touched?"

1. User calls `atm.withdraw(20000)`.
2. `ATM.withdraw()` delegates to `currentState.withdraw(atm, 20000)`.
3. `AuthenticatedState.withdraw()` checks the daily limit.
4. Calls `atm.getCashDispenser().canDispense(20000)`.
5. Calls `atm.getBankService().debit("ACC-001", 20000, WITHDRAWAL, null)`.
6. `MockBankNetworkService.debit()` calls `account.debit(20000, WITHDRAWAL, null)`.
7. `Account.debit()` calls `validateSufficientFunds(20000)`, updates balance, appends a `Transaction`.
8. `AuthenticatedState` transitions ATM to `new DispensingState(20000, this)`.
9. Calls `atm.getCurrentState().withdraw(atm, 20000)` — `DispensingState.withdraw()` executes.
10. `DispensingState.withdraw()` calls `atm.getCashDispenser().dispense(20000)`.
11. `CashDispenser.dispense()` updates denomination counts and prints a receipt line.
12. `DispensingState` transitions ATM back to `returnState` (the original `AuthenticatedState`).

---

## 12. Common Mistakes in This Design

### Mistake 1: Storing the raw PIN

**What students do**: `this.pin = rawPin;` in `Card.java`.

**Why it's wrong**: Any memory dump, log statement, or debug print that includes a `Card` object leaks the customer's PIN. The PIN should be hashed immediately and the raw value discarded.

**Fix**: Hash on construction (as shown in `Card.java` above). Never expose a `getRawPin()` method.

---

### Mistake 2: Putting all state logic in the ATM class

**What students do**: A single `ATM` class with a `String currentState` field and `if (currentState.equals("AUTHENTICATED")) { ... }` in every method.

**Why it's wrong**: This is the "God Object" anti-pattern. The ATM class balloons to hundreds of lines of branching logic. Adding a new state requires touching every method. Impossible to unit-test individual states in isolation.

**Fix**: State pattern — each state in its own class.

---

### Mistake 3: Using `double` for monetary amounts

**What students do**: `private double balanceDollars;`

**Why it's wrong**: `double` arithmetic is imprecise for decimals. `0.1 + 0.2` evaluates to `0.30000000000000004`. Over many transactions this causes phantom losses or gains.

**Fix**: Use `long` cents or `BigDecimal`.

---

### Mistake 4: Validating the PIN in the ATM class instead of delegating to the bank service

**What students do**: `if (card.getPin().equals(enteredPin)) { authenticate(); }`

**Why it's wrong**:
1. Requires the raw PIN to be readable from `Card`, creating a leak risk.
2. Bypasses the bank's authority — the bank should be the source of truth for authentication.
3. Prevents adding server-side fraud detection.

**Fix**: Route PIN validation through `BankNetworkService.validatePin()` as shown.

---

### Mistake 5: Ignoring the dispensing failure case

**What students do**: Debit the account, then call `dispenser.dispense()` — and ignore the possibility that `dispense()` throws an exception.

**Why it's wrong**: If the dispenser fails, the customer is charged but receives no cash. This is a real-world ATM bug that has caused significant customer harm.

**Fix**: At minimum, document the reversal as a known gap and point to the saga/two-phase pattern as the correct solution. Show that you understand the risk even if the full solution is out of scope for an interview.

---

### Mistake 6: Allowing any state to freely call `atm.setCurrentState()`

**What students do**: Expose `setCurrentState()` as `public` on `ATM`, then call it from everywhere including client code.

**Why it's wrong**: Breaks the invariant that only state classes manage transitions. External code could put the ATM in an invalid state (e.g., `AuthenticatedState` with no card in the reader).

**Fix**: Make `setCurrentState()` package-private (as shown) so only classes in the `atm` package can call it. In Java 9+, use module-info to further restrict access.

---

### Mistake 7: Not resetting session state on card ejection

**What students do**: The ATM ejects the card and goes to `IdleState` but forgets to null out `currentAccount`.

**Why it's wrong**: If the next customer inserts their card and the bank lookup fails before overwriting `currentAccount`, the ATM might briefly display the previous customer's balance.

**Fix**: Always call `atm.setCurrentAccount(null)` on every `ejectCard()` path (as shown in `AuthenticatedState.ejectCard()` and `DispensingState.ejectCard()`).

---

### Mistake 8: Per-session daily limit tracking

**What students do**: Track `withdrawnTodayCents` inside `AuthenticatedState` without persisting it.

**Why it's wrong**: The limit resets every time the customer opens a new session (re-inserts the card). A customer can bypass the $1,000 daily limit by making ten sessions of $100 each.

**Fix**: Persist the daily total in a durable store (database or the bank server), keyed by `(cardNumber, date)`.

---

*End of ATM System LLD Case Study*
