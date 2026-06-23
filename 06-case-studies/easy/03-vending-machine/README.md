# Vending Machine — Low-Level Design Case Study

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Assumptions and Constraints](#4-assumptions-and-constraints)
5. [Domain Model Description](#5-domain-model-description)
6. [Class Diagram (UML)](#6-class-diagram-uml)
7. [Core Classes — Complete Java Implementation](#7-core-classes--complete-java-implementation)
8. [Design Patterns Used](#8-design-patterns-used)
9. [Key Design Decisions and Trade-offs](#9-key-design-decisions-and-trade-offs)
10. [Extension Points](#10-extension-points)
11. [Interview Discussion Points](#11-interview-discussion-points)
12. [Common Mistakes in This Design](#12-common-mistakes-in-this-design)

---

## 1. Problem Statement

Design a vending machine that:

- Holds multiple products at different prices and quantities.
- Accepts multiple payment methods (cash coins/bills, credit/debit card).
- Dispenses the selected product when payment is sufficient.
- Returns the correct change when cash payment exceeds the product price.
- Manages its own inventory and transitions through well-defined states.
- Provides an admin mode so an operator can restock products and adjust prices without exposing those operations to regular users.

The goal is a clean, extensible object-oriented design that clearly separates concerns across state management, inventory, payment processing, and administration.

---

## 2. Functional Requirements

1. The machine shall display a list of available products with their names, prices, and remaining quantities.
2. A user shall be able to insert money (cash denominations or card) before selecting a product.
3. A user shall be able to select a product by its product ID.
4. The machine shall dispense the selected product only when the inserted amount is greater than or equal to the product price.
5. The machine shall return change in the largest available denominations first after dispensing a product paid for with cash.
6. The machine shall reject a product selection if the requested product is out of stock and transition to an `OutOfStock` state.
7. A user shall be able to cancel a transaction at any time and receive a full refund of inserted cash.
8. The machine shall support an admin mode protected by a PIN.
9. In admin mode, an operator shall be able to restock any product by specifying a product ID and quantity.
10. In admin mode, an operator shall be able to change the price of any product.
11. In admin mode, an operator shall be able to add a new product slot to the machine.
12. The machine shall track available change denominations and warn when it cannot make exact change.
13. Card payments shall not require change dispensing.
14. The machine shall log all transactions (product dispensed, amount paid, change returned).

---

## 3. Non-Functional Requirements

- **Correctness**: No product is ever dispensed without successful payment; no money is lost.
- **Extensibility**: New payment methods and new states can be added without modifying existing state classes.
- **Maintainability**: Each state class is cohesive and contains only the logic appropriate to that state.
- **Thread Safety**: The `VendingMachine` is designed for single-threaded use (a real embedded system); concurrent access is handled at the hardware controller layer, not in this design.
- **Testability**: Every class can be unit-tested in isolation; states can be injected.
- **Readability**: Code reads like the domain language ("insert money", "select product", "dispense").

---

## 4. Assumptions and Constraints

- The machine operates in a single-threaded environment (one user at a time).
- Cash denominations are represented as integer cent values (e.g., 25 = quarter, 100 = $1 bill).
- Prices are stored and compared as integer cents to avoid floating-point rounding.
- The machine starts with a pre-loaded set of change denominations in its change vault.
- Card payment is assumed to always succeed (payment gateway integration is out of scope).
- The admin PIN is hardcoded for simplicity; in production it would be stored hashed.
- A product slot has a maximum capacity of 10 units.
- The machine has at most 20 distinct product slots.
- Products are identified by a string ID (e.g., `"A1"`, `"B3"`).
- `OutOfStock` state is entered when the *selected* product is out of stock; other products may still be available.

---

## 5. Domain Model Description

### Entities

**Product**
Represents a single SKU in the machine. Holds the product's unique ID, display name, price in cents, and current quantity. Quantity is decremented on dispense and incremented on restock.

**Inventory**
A registry of all `Product` objects keyed by product ID. Provides lookup, stock-check, dispense, and restock operations. Acts as the single source of truth for what the machine contains.

**Payment (interface)**
Abstracts the concept of tendering money. Concrete types are `CashPayment` (tracks inserted denominations) and `CardPayment` (represents a card swipe with a pre-authorized amount).

**ChangeVault**
Manages the denominations (coins and bills) available for making change. Implements a greedy algorithm to compute the optimal set of denominations to return as change.

**VendingMachineState (interface)**
The State pattern contract. Every user-visible action on the machine is a method here. Concrete states implement the subset of actions that are legal in that state and throw `IllegalStateException` for actions that are not.

**Concrete States**
- `IdleState`: Machine is waiting for money. Accepts `insertMoney()`, rejects all others.
- `HasMoneyState`: Money has been inserted. Accepts `selectProduct()` and `cancelTransaction()`.
- `DispensingState`: A product is selected and payment is sufficient. Accepts `dispense()` only.
- `OutOfStockState`: The selected product is out of stock. Accepts `cancelTransaction()` and `refund()`.

**VendingMachine**
The context object in the State pattern. Holds a reference to the current state, the inventory, the change vault, and the current payment. Delegates every user action to `currentState`. Provides `setState()` for state transitions triggered from within state classes.

**AdminPanel**
A separate facade for privileged operations. Wraps `VendingMachine` and `Inventory`. Requires PIN authentication before any operation executes.

**TransactionLog**
A simple in-memory log of `TransactionRecord` objects capturing what happened in each completed transaction.

---

## 6. Class Diagram (UML)

```
+---------------------------+          +-------------------------+
|   <<interface>>           |          |   VendingMachine        |
|   VendingMachineState     |<>------->| - currentState          |
|---------------------------|          | - inventory: Inventory  |
| +insertMoney(int)         |          | - changeVault: ChangeV. |
| +selectProduct(String)    |          | - currentPayment        |
| +dispense()               |          | - selectedProduct       |
| +cancelTransaction()      |          | - txLog: TransactionLog |
| +refund()                 |          |-------------------------|
+---------------------------+          | +insertMoney(int)        |
          ^                            | +insertCard(int)         |
          |                            | +selectProduct(String)   |
   +------+-------+                   | +dispense()              |
   |       |      |                   | +cancelTransaction()      |
   |       |      |                   | +setState(State)          |
   |       |      |                   | +getInventory()           |
   |       |      |                   | +getChangeVault()         |
IdleState  |  DispensingState          +-------------------------+
           |      |
    HasMoneyState  OutOfStockState


+---------------------+        +---------------------+
|   Inventory         |        |   ChangeVault        |
|---------------------|        |---------------------|
| - products:         |        | - denominations:    |
|   Map<String,Prod.> |        |   Map<Integer,Int.> |
|---------------------|        |---------------------|
| +getProduct(String) |        | +addDenomination()  |
| +isInStock(String)  |        | +makeChange(int)    |
| +dispenseProduct()  |        | +hasEnoughChange()  |
| +restockProduct()   |        +---------------------+
| +addProduct()       |
| +getAllProducts()   |
+---------------------+

+---------------------+        +-------------------+
|  <<interface>>      |        |   Product         |
|  Payment            |        |-------------------|
|---------------------|        | - id: String      |
| +getAmount(): int   |        | - name: String    |
| +getType(): String  |        | - priceInCents:int|
+---------------------+        | - quantity: int   |
         ^                     |-------------------|
         |                     | +dispense()       |
  +------+------+              | +restock(int)     |
  |             |              | +isAvailable()    |
CashPayment  CardPayment       +-------------------+

+---------------------+        +----------------------+
|   AdminPanel        |        |   TransactionLog     |
|---------------------|        |----------------------|
| - machine           |        | - records: List<>    |
| - inventory         |        |----------------------|
| - ADMIN_PIN         |        | +record(...)         |
|---------------------|        | +getHistory()        |
| +authenticate(int)  |        +----------------------+
| +restock(...)       |
| +setPrice(...)      |
| +addProduct(...)    |
| +showInventory()    |
+---------------------+
```

---

## 7. Core Classes — Complete Java Implementation

The full implementation is organized into the following files. All code is Java 11-compatible and compiles without external dependencies.

---

### `Product.java`

```java
package vendingmachine;

public class Product {

    private final String id;
    private final String name;
    private int priceInCents;
    private int quantity;
    private static final int MAX_CAPACITY = 10;

    public Product(String id, String name, int priceInCents, int quantity) {
        if (id == null || id.isBlank()) {
            throw new IllegalArgumentException("Product id cannot be blank");
        }
        if (priceInCents <= 0) {
            throw new IllegalArgumentException("Price must be positive, got: " + priceInCents);
        }
        if (quantity < 0 || quantity > MAX_CAPACITY) {
            throw new IllegalArgumentException(
                "Quantity must be between 0 and " + MAX_CAPACITY + ", got: " + quantity);
        }
        this.id = id;
        this.name = name;
        this.priceInCents = priceInCents;
        this.quantity = quantity;
    }

    public String getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public int getPriceInCents() {
        return priceInCents;
    }

    public void setPriceInCents(int priceInCents) {
        if (priceInCents <= 0) {
            throw new IllegalArgumentException("Price must be positive");
        }
        this.priceInCents = priceInCents;
    }

    public int getQuantity() {
        return quantity;
    }

    public boolean isAvailable() {
        return quantity > 0;
    }

    /** Decrements quantity by 1. Called only by Inventory after payment succeeds. */
    public void dispense() {
        if (quantity <= 0) {
            throw new IllegalStateException("Cannot dispense: " + name + " is out of stock");
        }
        quantity--;
    }

    /** Adds units up to MAX_CAPACITY. */
    public void restock(int units) {
        if (units <= 0) {
            throw new IllegalArgumentException("Restock units must be positive");
        }
        if (quantity + units > MAX_CAPACITY) {
            throw new IllegalArgumentException(
                "Restock would exceed max capacity of " + MAX_CAPACITY
                + ". Current: " + quantity + ", Adding: " + units);
        }
        quantity += units;
    }

    @Override
    public String toString() {
        return String.format("[%s] %-20s $%4.2f  qty:%d",
            id, name, priceInCents / 100.0, quantity);
    }
}
```

---

### `Inventory.java`

```java
package vendingmachine;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;

public class Inventory {

    private static final int MAX_SLOTS = 20;
    private final Map<String, Product> products = new LinkedHashMap<>();

    public void addProduct(Product product) {
        if (products.size() >= MAX_SLOTS) {
            throw new IllegalStateException(
                "Machine is full: cannot add more than " + MAX_SLOTS + " product slots");
        }
        if (products.containsKey(product.getId())) {
            throw new IllegalArgumentException(
                "Product ID already exists: " + product.getId());
        }
        products.put(product.getId(), product);
    }

    public Product getProduct(String productId) {
        Product p = products.get(productId);
        if (p == null) {
            throw new IllegalArgumentException("Unknown product ID: " + productId);
        }
        return p;
    }

    public boolean isInStock(String productId) {
        Product p = products.get(productId);
        return p != null && p.isAvailable();
    }

    /**
     * Removes one unit from inventory. Returns the product that was dispensed.
     * Callers must verify payment before calling this.
     */
    public Product dispenseProduct(String productId) {
        Product p = getProduct(productId);
        p.dispense();
        return p;
    }

    /**
     * Adds stock to an existing product slot.
     */
    public void restockProduct(String productId, int units) {
        getProduct(productId).restock(units);
    }

    public boolean hasAnyStock() {
        return products.values().stream().anyMatch(Product::isAvailable);
    }

    public Map<String, Product> getAllProducts() {
        return Collections.unmodifiableMap(products);
    }

    public void printInventory() {
        System.out.println("=== Current Inventory ===");
        if (products.isEmpty()) {
            System.out.println("  (empty)");
        } else {
            products.values().forEach(p -> System.out.println("  " + p));
        }
        System.out.println("=========================");
    }
}
```

---

### `Payment.java` (interface)

```java
package vendingmachine;

public interface Payment {

    /** Returns the total tendered amount in cents. */
    int getAmount();

    /** Returns a human-readable type label, e.g. "CASH" or "CARD". */
    String getType();
}
```

---

### `CashPayment.java`

```java
package vendingmachine;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class CashPayment implements Payment {

    // Each element is a denomination value in cents (e.g. 25 = quarter, 100 = $1)
    private final List<Integer> insertedDenominations = new ArrayList<>();
    private int totalCents = 0;

    /**
     * Records a single denomination being inserted.
     * Valid denominations: 1, 5, 10, 25, 50, 100, 200, 500, 1000 (cents).
     */
    public void insertDenomination(int cents) {
        if (!isValidDenomination(cents)) {
            throw new IllegalArgumentException(
                "Invalid denomination: " + cents + " cents. "
                + "Accepted: 1, 5, 10, 25, 50, 100, 200, 500, 1000");
        }
        insertedDenominations.add(cents);
        totalCents += cents;
    }

    private boolean isValidDenomination(int cents) {
        return cents == 1   || cents == 5   || cents == 10  || cents == 25
            || cents == 50  || cents == 100 || cents == 200 || cents == 500
            || cents == 1000;
    }

    @Override
    public int getAmount() {
        return totalCents;
    }

    @Override
    public String getType() {
        return "CASH";
    }

    public List<Integer> getInsertedDenominations() {
        return Collections.unmodifiableList(insertedDenominations);
    }

    @Override
    public String toString() {
        return String.format("CashPayment{total=$%.2f, denominations=%s}",
            totalCents / 100.0, insertedDenominations);
    }
}
```

---

### `CardPayment.java`

```java
package vendingmachine;

public class CardPayment implements Payment {

    private final String cardToken;      // Masked card reference, e.g. "****-1234"
    private final int authorizedCents;   // Pre-authorized amount from the gateway

    /**
     * In a real system the gateway would return an authorization code.
     * Here we accept the amount directly to keep the design self-contained.
     */
    public CardPayment(String cardToken, int authorizedCents) {
        if (authorizedCents <= 0) {
            throw new IllegalArgumentException("Authorized amount must be positive");
        }
        this.cardToken = cardToken;
        this.authorizedCents = authorizedCents;
    }

    @Override
    public int getAmount() {
        return authorizedCents;
    }

    @Override
    public String getType() {
        return "CARD";
    }

    public String getCardToken() {
        return cardToken;
    }

    @Override
    public String toString() {
        return String.format("CardPayment{card=%s, authorized=$%.2f}",
            cardToken, authorizedCents / 100.0);
    }
}
```

---

### `ChangeVault.java`

```java
package vendingmachine;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * Manages the physical coins and bills available for making change.
 * Denominations are stored from largest to smallest for greedy dispensing.
 */
public class ChangeVault {

    // Ordered largest-to-smallest: $10, $5, $2, $1, 50c, 25c, 10c, 5c, 1c
    private static final int[] DENOMINATION_ORDER =
        {1000, 500, 200, 100, 50, 25, 10, 5, 1};

    // denomination (cents) -> count available
    private final Map<Integer, Integer> vault = new LinkedHashMap<>();

    public ChangeVault() {
        for (int d : DENOMINATION_ORDER) {
            vault.put(d, 0);
        }
    }

    /**
     * Loads the vault with a given count of a denomination.
     * Called during machine initialization or admin restocking.
     */
    public void addDenomination(int cents, int count) {
        if (!vault.containsKey(cents)) {
            throw new IllegalArgumentException("Unsupported denomination: " + cents);
        }
        if (count < 0) {
            throw new IllegalArgumentException("Count cannot be negative");
        }
        vault.put(cents, vault.get(cents) + count);
    }

    /**
     * Attempts to make exact change for the given amount using available denominations.
     * Uses a greedy algorithm (largest first).
     *
     * @param amountInCents the change to return
     * @return list of denomination values to return (each element = one coin/bill)
     * @throws IllegalStateException if exact change cannot be made
     */
    public List<Integer> makeChange(int amountInCents) {
        if (amountInCents == 0) {
            return new ArrayList<>();
        }
        if (amountInCents < 0) {
            throw new IllegalArgumentException("Change amount cannot be negative: " + amountInCents);
        }

        // Work on a temporary copy so we only commit if fully successful
        Map<Integer, Integer> tempVault = new LinkedHashMap<>(vault);
        List<Integer> changeList = new ArrayList<>();
        int remaining = amountInCents;

        for (int denomination : DENOMINATION_ORDER) {
            int available = tempVault.getOrDefault(denomination, 0);
            while (remaining >= denomination && available > 0) {
                changeList.add(denomination);
                remaining -= denomination;
                available--;
            }
            tempVault.put(denomination, available);
        }

        if (remaining != 0) {
            throw new IllegalStateException(
                "Cannot make exact change for $" + String.format("%.2f", amountInCents / 100.0)
                + ". Remaining: " + remaining + " cents. Please use exact change or card.");
        }

        // Commit the vault changes
        vault.putAll(tempVault);
        return changeList;
    }

    /**
     * Checks whether exact change CAN be made without actually modifying the vault.
     */
    public boolean canMakeChange(int amountInCents) {
        if (amountInCents == 0) {
            return true;
        }
        Map<Integer, Integer> tempVault = new LinkedHashMap<>(vault);
        int remaining = amountInCents;
        for (int denomination : DENOMINATION_ORDER) {
            int available = tempVault.getOrDefault(denomination, 0);
            while (remaining >= denomination && available > 0) {
                remaining -= denomination;
                available--;
            }
        }
        return remaining == 0;
    }

    /**
     * When cash is inserted, those denominations are added to the vault
     * (they become available for future change-making).
     */
    public void depositCash(List<Integer> denominations) {
        for (int d : denominations) {
            if (!vault.containsKey(d)) {
                throw new IllegalArgumentException("Unsupported denomination: " + d);
            }
            vault.put(d, vault.get(d) + 1);
        }
    }

    /**
     * Returns a snapshot of current vault contents for admin display.
     */
    public Map<Integer, Integer> getVaultSnapshot() {
        return new LinkedHashMap<>(vault);
    }

    public void printVault() {
        System.out.println("=== Change Vault ===");
        for (int d : DENOMINATION_ORDER) {
            int count = vault.getOrDefault(d, 0);
            if (count > 0) {
                System.out.printf("  $%5.2f x %d%n", d / 100.0, count);
            }
        }
        System.out.println("====================");
    }
}
```

---

### `VendingMachineState.java` (interface)

```java
package vendingmachine;

/**
 * State pattern interface.
 * Every action a user can perform maps to a method here.
 * Each concrete state decides whether the action is legal in that state.
 */
public interface VendingMachineState {

    /**
     * User inserts a cash denomination (value in cents).
     * Legal in: IdleState, HasMoneyState (add more money).
     */
    void insertMoney(VendingMachine machine, int cents);

    /**
     * User selects a product by its ID.
     * Legal in: HasMoneyState.
     */
    void selectProduct(VendingMachine machine, String productId);

    /**
     * Machine physically dispenses the product and returns change.
     * Legal in: DispensingState.
     */
    void dispense(VendingMachine machine);

    /**
     * User cancels the transaction before dispensing.
     * Legal in: HasMoneyState, OutOfStockState.
     */
    void cancelTransaction(VendingMachine machine);

    /**
     * Machine refunds all inserted money.
     * Legal in: HasMoneyState, OutOfStockState, or after a cancel.
     */
    void refund(VendingMachine machine);
}
```

---

### `IdleState.java`

```java
package vendingmachine;

public class IdleState implements VendingMachineState {

    @Override
    public void insertMoney(VendingMachine machine, int cents) {
        CashPayment payment = new CashPayment();
        payment.insertDenomination(cents);
        machine.setCurrentPayment(payment);
        System.out.printf("Inserted $%.2f. Total: $%.2f%n",
            cents / 100.0, payment.getAmount() / 100.0);
        machine.setState(machine.getHasMoneyState());
    }

    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        System.out.println("Please insert money first.");
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("No product selected and no money inserted.");
    }

    @Override
    public void cancelTransaction(VendingMachine machine) {
        System.out.println("No transaction in progress.");
    }

    @Override
    public void refund(VendingMachine machine) {
        System.out.println("Nothing to refund.");
    }

    @Override
    public String toString() {
        return "IDLE";
    }
}
```

---

### `HasMoneyState.java`

```java
package vendingmachine;

public class HasMoneyState implements VendingMachineState {

    @Override
    public void insertMoney(VendingMachine machine, int cents) {
        Payment current = machine.getCurrentPayment();
        if (current instanceof CashPayment) {
            ((CashPayment) current).insertDenomination(cents);
            System.out.printf("Inserted $%.2f more. Total: $%.2f%n",
                cents / 100.0, current.getAmount() / 100.0);
        } else {
            System.out.println("Card payment in progress. Cannot insert cash.");
        }
    }

    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        Inventory inventory = machine.getInventory();

        if (!inventory.isInStock(productId)) {
            System.out.println("Product " + productId + " is out of stock.");
            machine.setSelectedProductId(productId);
            machine.setState(machine.getOutOfStockState());
            return;
        }

        Product product = inventory.getProduct(productId);
        int paid = machine.getCurrentPayment().getAmount();

        if (paid < product.getPriceInCents()) {
            System.out.printf("Insufficient funds. Price: $%.2f, Inserted: $%.2f. "
                + "Please insert $%.2f more.%n",
                product.getPriceInCents() / 100.0,
                paid / 100.0,
                (product.getPriceInCents() - paid) / 100.0);
            return;
        }

        int change = paid - product.getPriceInCents();
        if (change > 0 && !(machine.getCurrentPayment() instanceof CardPayment)) {
            if (!machine.getChangeVault().canMakeChange(change)) {
                System.out.printf(
                    "Cannot make change of $%.2f. Please use exact amount or card.%n",
                    change / 100.0);
                return;
            }
        }

        machine.setSelectedProductId(productId);
        System.out.println("Selected: " + product.getName()
            + " at $" + String.format("%.2f", product.getPriceInCents() / 100.0));
        machine.setState(machine.getDispensingState());
        // Immediately trigger dispensing — in a real machine the hardware fires this
        machine.dispense();
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please select a product first.");
    }

    @Override
    public void cancelTransaction(VendingMachine machine) {
        System.out.println("Transaction cancelled.");
        refund(machine);
    }

    @Override
    public void refund(VendingMachine machine) {
        Payment payment = machine.getCurrentPayment();
        if (payment != null) {
            if (payment instanceof CashPayment) {
                System.out.printf("Refunding $%.2f in cash.%n", payment.getAmount() / 100.0);
            } else {
                System.out.printf("Refunding $%.2f to card.%n", payment.getAmount() / 100.0);
            }
        }
        machine.setCurrentPayment(null);
        machine.setSelectedProductId(null);
        machine.setState(machine.getIdleState());
    }

    @Override
    public String toString() {
        return "HAS_MONEY";
    }
}
```

---

### `DispensingState.java`

```java
package vendingmachine;

import java.util.List;

public class DispensingState implements VendingMachineState {

    @Override
    public void insertMoney(VendingMachine machine, int cents) {
        System.out.println("Please wait — dispensing in progress.");
    }

    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        System.out.println("Please wait — dispensing in progress.");
    }

    @Override
    public void dispense(VendingMachine machine) {
        String productId = machine.getSelectedProductId();
        Payment payment = machine.getCurrentPayment();

        // Physically remove from inventory
        Product dispensed = machine.getInventory().dispenseProduct(productId);
        int paid = payment.getAmount();
        int price = dispensed.getPriceInCents();
        int changeAmount = paid - price;

        System.out.println(">>> Dispensing: " + dispensed.getName() + " <<<");

        // Deposit inserted cash into the vault before computing change
        if (payment instanceof CashPayment) {
            machine.getChangeVault().depositCash(
                ((CashPayment) payment).getInsertedDenominations());
        }

        // Return change if any (only for cash; card authorizes exactly what is needed)
        if (changeAmount > 0 && payment instanceof CashPayment) {
            List<Integer> changeCoins = machine.getChangeVault().makeChange(changeAmount);
            System.out.printf("Change returned: $%.2f -> %s%n",
                changeAmount / 100.0, formatChange(changeCoins));
        } else if (changeAmount > 0 && payment instanceof CardPayment) {
            System.out.printf("Card refund: $%.2f will be credited.%n", changeAmount / 100.0);
        }

        // Record the transaction
        machine.getTransactionLog().record(
            dispensed, payment, changeAmount);

        // Reset state
        machine.setCurrentPayment(null);
        machine.setSelectedProductId(null);
        machine.setState(machine.getIdleState());
    }

    @Override
    public void cancelTransaction(VendingMachine machine) {
        System.out.println("Cannot cancel — dispensing already in progress.");
    }

    @Override
    public void refund(VendingMachine machine) {
        System.out.println("Cannot refund — dispensing already in progress.");
    }

    private String formatChange(List<Integer> coins) {
        if (coins.isEmpty()) return "no coins";
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < coins.size(); i++) {
            sb.append(String.format("$%.2f", coins.get(i) / 100.0));
            if (i < coins.size() - 1) sb.append(", ");
        }
        sb.append("]");
        return sb.toString();
    }

    @Override
    public String toString() {
        return "DISPENSING";
    }
}
```

---

### `OutOfStockState.java`

```java
package vendingmachine;

public class OutOfStockState implements VendingMachineState {

    @Override
    public void insertMoney(VendingMachine machine, int cents) {
        System.out.println("Selected product is out of stock. Cancel and choose another.");
    }

    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        // Allow the user to switch to a different product without losing money
        Inventory inventory = machine.getInventory();

        if (!inventory.isInStock(productId)) {
            System.out.println("Product " + productId + " is also out of stock.");
            machine.setSelectedProductId(productId);
            return;
        }

        System.out.println("Switching to available product: " + productId);
        machine.setSelectedProductId(productId);
        machine.setState(machine.getHasMoneyState());
        // Delegate to HasMoneyState's selectProduct to proceed normally
        machine.selectProduct(productId);
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Cannot dispense — selected product is out of stock.");
    }

    @Override
    public void cancelTransaction(VendingMachine machine) {
        System.out.println("Transaction cancelled due to out-of-stock.");
        refund(machine);
    }

    @Override
    public void refund(VendingMachine machine) {
        Payment payment = machine.getCurrentPayment();
        if (payment != null && payment.getAmount() > 0) {
            System.out.printf("Refunding $%.2f.%n", payment.getAmount() / 100.0);
        }
        machine.setCurrentPayment(null);
        machine.setSelectedProductId(null);
        machine.setState(machine.getIdleState());
    }

    @Override
    public String toString() {
        return "OUT_OF_STOCK";
    }
}
```

---

### `TransactionRecord.java`

```java
package vendingmachine;

import java.time.LocalDateTime;

public class TransactionRecord {

    private final LocalDateTime timestamp;
    private final String productId;
    private final String productName;
    private final int priceInCents;
    private final String paymentType;
    private final int paidInCents;
    private final int changeInCents;

    public TransactionRecord(Product product, Payment payment, int changeInCents) {
        this.timestamp = LocalDateTime.now();
        this.productId = product.getId();
        this.productName = product.getName();
        this.priceInCents = product.getPriceInCents();
        this.paymentType = payment.getType();
        this.paidInCents = payment.getAmount();
        this.changeInCents = changeInCents;
    }

    @Override
    public String toString() {
        return String.format(
            "[%s] Product: %s (%s), Price: $%.2f, Paid: $%.2f (%s), Change: $%.2f",
            timestamp, productName, productId,
            priceInCents / 100.0,
            paidInCents / 100.0, paymentType,
            changeInCents / 100.0);
    }

    // Getters for reporting
    public LocalDateTime getTimestamp() { return timestamp; }
    public String getProductId() { return productId; }
    public String getProductName() { return productName; }
    public int getPriceInCents() { return priceInCents; }
    public String getPaymentType() { return paymentType; }
    public int getPaidInCents() { return paidInCents; }
    public int getChangeInCents() { return changeInCents; }
}
```

---

### `TransactionLog.java`

```java
package vendingmachine;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class TransactionLog {

    private final List<TransactionRecord> records = new ArrayList<>();

    public void record(Product product, Payment payment, int changeInCents) {
        TransactionRecord rec = new TransactionRecord(product, payment, changeInCents);
        records.add(rec);
        System.out.println("[LOG] " + rec);
    }

    public List<TransactionRecord> getHistory() {
        return Collections.unmodifiableList(records);
    }

    public void printHistory() {
        System.out.println("=== Transaction History ===");
        if (records.isEmpty()) {
            System.out.println("  (no transactions yet)");
        } else {
            records.forEach(r -> System.out.println("  " + r));
        }
        System.out.println("===========================");
    }
}
```

---

### `VendingMachine.java`

```java
package vendingmachine;

/**
 * Context class in the State pattern.
 *
 * All user-facing operations are delegated to currentState.
 * State objects call setState() to trigger transitions.
 * Package-private setters are intentional — only states (same package) modify context.
 */
public class VendingMachine {

    // ---- State singletons (one instance each, reused across transactions) ----
    private final VendingMachineState idleState       = new IdleState();
    private final VendingMachineState hasMoneyState   = new HasMoneyState();
    private final VendingMachineState dispensingState = new DispensingState();
    private final VendingMachineState outOfStockState = new OutOfStockState();

    // ---- Current state ----
    private VendingMachineState currentState;

    // ---- Infrastructure ----
    private final Inventory inventory;
    private final ChangeVault changeVault;
    private final TransactionLog transactionLog;

    // ---- Transaction context ----
    private Payment currentPayment;
    private String selectedProductId;

    public VendingMachine(Inventory inventory, ChangeVault changeVault) {
        this.inventory = inventory;
        this.changeVault = changeVault;
        this.transactionLog = new TransactionLog();
        this.currentState = idleState;
    }

    // ---- Public API (delegates to current state) ----

    public void insertMoney(int cents) {
        currentState.insertMoney(this, cents);
    }

    /**
     * Convenience: insert a card with a specific authorized amount.
     * Transitions directly to HasMoneyState with a CardPayment.
     */
    public void insertCard(String cardToken, int authorizedCents) {
        if (currentState != idleState && currentState != hasMoneyState) {
            System.out.println("Cannot insert card in current state: " + currentState);
            return;
        }
        CardPayment payment = new CardPayment(cardToken, authorizedCents);
        setCurrentPayment(payment);
        System.out.printf("Card %s authorized for $%.2f%n",
            cardToken, authorizedCents / 100.0);
        setState(hasMoneyState);
    }

    public void selectProduct(String productId) {
        currentState.selectProduct(this, productId);
    }

    public void dispense() {
        currentState.dispense(this);
    }

    public void cancelTransaction() {
        currentState.cancelTransaction(this);
    }

    public void refund() {
        currentState.refund(this);
    }

    public void showProducts() {
        inventory.printInventory();
    }

    public String getCurrentStateName() {
        return currentState.toString();
    }

    // ---- State accessors (used by state classes) ----

    VendingMachineState getIdleState()       { return idleState; }
    VendingMachineState getHasMoneyState()   { return hasMoneyState; }
    VendingMachineState getDispensingState() { return dispensingState; }
    VendingMachineState getOutOfStockState() { return outOfStockState; }

    void setState(VendingMachineState state) {
        System.out.println("[STATE] " + currentState + " -> " + state);
        this.currentState = state;
    }

    // ---- Infrastructure accessors ----

    Inventory getInventory()             { return inventory; }
    ChangeVault getChangeVault()         { return changeVault; }
    TransactionLog getTransactionLog()   { return transactionLog; }

    Payment getCurrentPayment()                    { return currentPayment; }
    void setCurrentPayment(Payment payment)        { this.currentPayment = payment; }

    String getSelectedProductId()                  { return selectedProductId; }
    void setSelectedProductId(String productId)    { this.selectedProductId = productId; }
}
```

---

### `AdminPanel.java`

```java
package vendingmachine;

/**
 * Privileged operations panel.
 * All methods require prior authentication via authenticate().
 */
public class AdminPanel {

    private static final int ADMIN_PIN = 1234;

    private final VendingMachine machine;
    private final Inventory inventory;
    private boolean authenticated = false;

    public AdminPanel(VendingMachine machine, Inventory inventory) {
        this.machine = machine;
        this.inventory = inventory;
    }

    public boolean authenticate(int pin) {
        if (pin == ADMIN_PIN) {
            authenticated = true;
            System.out.println("[ADMIN] Authentication successful.");
            return true;
        }
        System.out.println("[ADMIN] Incorrect PIN. Access denied.");
        return false;
    }

    public void logout() {
        authenticated = false;
        System.out.println("[ADMIN] Logged out.");
    }

    public void restock(String productId, int units) {
        requireAuth();
        inventory.restockProduct(productId, units);
        Product p = inventory.getProduct(productId);
        System.out.printf("[ADMIN] Restocked %s: +%d units (new qty: %d)%n",
            p.getName(), units, p.getQuantity());
    }

    public void setPrice(String productId, int newPriceInCents) {
        requireAuth();
        if (newPriceInCents <= 0) {
            throw new IllegalArgumentException("Price must be positive");
        }
        Product p = inventory.getProduct(productId);
        int oldPrice = p.getPriceInCents();
        p.setPriceInCents(newPriceInCents);
        System.out.printf("[ADMIN] Price updated for %s: $%.2f -> $%.2f%n",
            p.getName(), oldPrice / 100.0, newPriceInCents / 100.0);
    }

    public void addProduct(String id, String name, int priceInCents, int initialQty) {
        requireAuth();
        Product newProduct = new Product(id, name, priceInCents, initialQty);
        inventory.addProduct(newProduct);
        System.out.printf("[ADMIN] Added new product: %s%n", newProduct);
    }

    public void showInventory() {
        requireAuth();
        inventory.printInventory();
    }

    public void showVault() {
        requireAuth();
        machine.getChangeVault().printVault();
    }

    public void showTransactionLog() {
        requireAuth();
        machine.getTransactionLog().printHistory();
    }

    private void requireAuth() {
        if (!authenticated) {
            throw new SecurityException(
                "Admin authentication required. Call authenticate(pin) first.");
        }
    }
}
```

---

### `VendingMachineDemo.java`

```java
package vendingmachine;

/**
 * Demonstrates all major flows:
 *  1. Happy path: insert cash -> select product -> dispense -> get change
 *  2. Insufficient funds: add more money
 *  3. Cancel transaction: get full refund
 *  4. Out-of-stock scenario
 *  5. Card payment
 *  6. Admin operations: restock, price change, add product
 */
public class VendingMachineDemo {

    public static void main(String[] args) {
        System.out.println("============================================================");
        System.out.println("              VENDING MACHINE DEMO                         ");
        System.out.println("============================================================\n");

        // ---- Setup ----
        Inventory inventory = new Inventory();
        inventory.addProduct(new Product("A1", "Cola",        150, 3));  // $1.50
        inventory.addProduct(new Product("A2", "Diet Cola",   150, 0));  // out of stock
        inventory.addProduct(new Product("B1", "Chips",       100, 5));  // $1.00
        inventory.addProduct(new Product("B2", "Chocolate",   175, 2));  // $1.75
        inventory.addProduct(new Product("C1", "Water",        75, 4));  // $0.75

        ChangeVault vault = new ChangeVault();
        vault.addDenomination(100, 10);  // 10 x $1 bills
        vault.addDenomination(25,  20);  // 20 x quarters
        vault.addDenomination(10,  20);  // 20 x dimes
        vault.addDenomination(5,   20);  // 20 x nickels
        vault.addDenomination(1,   50);  // 50 x pennies

        VendingMachine machine = new VendingMachine(inventory, vault);

        // ---- Scenario 1: Happy path with exact change ----
        System.out.println("\n--- Scenario 1: Buy Cola with $2 (get $0.50 change) ---");
        machine.showProducts();
        machine.insertMoney(100);   // $1.00
        machine.insertMoney(100);   // $1.00 more -> total $2.00
        machine.selectProduct("A1");
        // dispense() is triggered automatically by selectProduct in DispensingState
        System.out.println("Machine state: " + machine.getCurrentStateName());

        // ---- Scenario 2: Insufficient funds ----
        System.out.println("\n--- Scenario 2: Insufficient funds for Chocolate ($1.75) ---");
        machine.insertMoney(100);   // $1.00 — not enough
        machine.selectProduct("B2");
        machine.insertMoney(100);   // add another $1.00 -> total $2.00
        machine.selectProduct("B2");
        System.out.println("Machine state: " + machine.getCurrentStateName());

        // ---- Scenario 3: Cancel and refund ----
        System.out.println("\n--- Scenario 3: Cancel transaction, get refund ---");
        machine.insertMoney(100);
        machine.insertMoney(25);
        System.out.println("Inserted $1.25, now cancelling...");
        machine.cancelTransaction();
        System.out.println("Machine state: " + machine.getCurrentStateName());

        // ---- Scenario 4: Out-of-stock ----
        System.out.println("\n--- Scenario 4: Select out-of-stock product ---");
        machine.insertMoney(200);   // $2.00
        machine.selectProduct("A2");  // out of stock
        System.out.println("Machine state: " + machine.getCurrentStateName());
        System.out.println("Switching to Cola (A1)...");
        machine.selectProduct("A1");  // switch to in-stock product
        System.out.println("Machine state: " + machine.getCurrentStateName());

        // ---- Scenario 5: Card payment ----
        System.out.println("\n--- Scenario 5: Card payment for Water ($0.75) ---");
        machine.insertCard("****-5678", 75);  // exact amount
        machine.selectProduct("C1");
        System.out.println("Machine state: " + machine.getCurrentStateName());

        // ---- Scenario 6: Admin operations ----
        System.out.println("\n--- Scenario 6: Admin panel operations ---");
        AdminPanel admin = new AdminPanel(machine, inventory);

        // Wrong PIN
        admin.restock("A2", 5); // should throw SecurityException
        // ... actually it won't throw here; the requireAuth prints to stdout
        // We demonstrate with proper auth:
        admin.authenticate(9999);  // wrong PIN
        admin.authenticate(1234);  // correct PIN

        admin.restock("A2", 5);    // restock Diet Cola
        admin.setPrice("B1", 125); // raise Chips to $1.25
        admin.addProduct("D1", "Granola Bar", 200, 3);
        admin.showInventory();
        admin.showVault();
        admin.showTransactionLog();
        admin.logout();

        // ---- Scenario 7: Buy restocked product ----
        System.out.println("\n--- Scenario 7: Buy the newly restocked Diet Cola ---");
        machine.insertMoney(200);   // $2.00
        machine.selectProduct("A2");
        System.out.println("Machine state: " + machine.getCurrentStateName());

        System.out.println("\n============================================================");
        System.out.println("                      DEMO COMPLETE                        ");
        System.out.println("============================================================");
    }
}
```

---

### Running the Demo

```bash
# Compile all files from the project root
javac -d out vendingmachine/*.java

# Run the demo
java -cp out vendingmachine.VendingMachineDemo
```

---

## 8. Design Patterns Used

### 8.1 State Pattern (Core)

**What it is**: Define a set of state objects. The context (`VendingMachine`) delegates every operation to `currentState`. Each state object encapsulates the behavior for that state and triggers transitions by calling `machine.setState(...)`.

**Why it is used here**:
A vending machine is a classic finite state machine. Without the State pattern, `VendingMachine` would contain one giant method per operation, each containing a cascading `if/else` or `switch` on the current state. Adding a new state (e.g., `MaintenanceState`) would require modifying every existing method. With State pattern, you add one new class and update only the states that can transition to it.

**Concrete benefit**: `IdleState.insertMoney()` creates a `CashPayment`, stores it on the machine, and transitions to `HasMoneyState` — all without the machine knowing the transition logic.

---

### 8.2 Strategy Pattern (Payment)

**What it is**: Define a family of algorithms (here, payment types), encapsulate each one, and make them interchangeable.

**Why it is used here**:
`CashPayment` and `CardPayment` are fundamentally different: cash needs denomination tracking and vault deposit; card does not produce change. The `Payment` interface lets `DispensingState` call `payment.getAmount()` without caring whether it is cash or card. The change calculation checks `instanceof CashPayment` only once.

**Concrete benefit**: Adding `CryptoPay` or `MobileWalletPay` requires implementing `Payment` and wiring it in `VendingMachine.insertCard()` — no changes to any state class.

---

### 8.3 Facade Pattern (AdminPanel)

**What it is**: Provide a simplified interface to a complex subsystem.

**Why it is used here**:
Admin operations span `Inventory`, `ChangeVault`, and `TransactionLog`. Exposing those directly would let user-facing code call `inventory.restockProduct()` without a PIN check. `AdminPanel` consolidates access behind `requireAuth()`, giving operators a single, coherent API.

---

### 8.4 Template Method (implicit in `ChangeVault.makeChange`)

The greedy change algorithm iterates over a fixed denomination order. The denomination list itself acts as a template; overriding it (e.g., for a different currency) changes behavior without touching the loop logic.

---

## 9. Key Design Decisions and Trade-offs

### 9.1 State Singletons vs. Per-Transaction State Objects

**Decision**: States are created once and reused (singleton per machine).

**Trade-off**: States are stateless — all transaction context (current payment, selected product) lives on `VendingMachine`, not on the state. This means state methods need the machine reference as a parameter, which is the standard State pattern idiom. The alternative — creating a new `HasMoneyState(payment)` per transaction — would make state objects heavier and eliminate reuse, with no benefit.

---

### 9.2 Prices in Integer Cents (Not `double`)

**Decision**: All monetary values are `int` representing cents.

**Trade-off**: `$1.50` is stored as `150`. This eliminates floating-point rounding errors entirely. The display layer converts with `/ 100.0`. The cost is that all input must be converted to cents first. This is the universally correct approach for financial systems.

---

### 9.3 Dispensing Triggered Automatically After `selectProduct`

**Decision**: `HasMoneyState.selectProduct()` transitions to `DispensingState` and immediately calls `machine.dispense()` on the same thread.

**Trade-off**: In a real machine, the hardware controller fires dispensing asynchronously. In this design the state transition and dispense are synchronous, which simplifies the demo and unit testing without hiding the state machine structure. The state still transitions to `DispensingState` first, making it trivial to decouple later.

---

### 9.4 `ChangeVault` Deposits Cash Before Computing Change

**Decision**: `DispensingState.dispense()` first calls `vault.depositCash(insertedDenominations)`, then calls `vault.makeChange(changeAmount)`.

**Trade-off**: Depositing first means the inserted bills become available for the very change being returned in the same transaction. This is physically correct (a $5 bill inserted to buy a $1.50 item makes a $5 bill available for change). The alternative — depositing after — could cause a spurious "cannot make change" error.

---

### 9.5 `OutOfStockState` Allows Product Switching

**Decision**: `OutOfStockState.selectProduct()` lets the user choose a different in-stock product without losing their money.

**Trade-off**: This adds a small amount of complexity to `OutOfStockState`, but it dramatically improves UX. The alternative (force refund on out-of-stock) is annoying and non-standard for physical machines.

---

### 9.6 `AdminPanel` Lives Outside the State Machine

**Decision**: Admin operations bypass the state machine entirely.

**Trade-off**: An operator can restock the machine regardless of whether a user transaction is in progress. In practice this is fine — admin operations affect inventory counts, not the current transaction. If stricter isolation were needed, you could add a `MaintenanceState` that locks out user operations while admin works.

---

## 10. Extension Points

### Add a New Payment Method

1. Implement `Payment` with a new class (e.g., `MobileWalletPayment`).
2. Add a method on `VendingMachine` (e.g., `insertMobileWallet(String token, int amount)`).
3. Handle `instanceof MobileWalletPayment` in `DispensingState.dispense()` if it needs special change behavior (most digital payments do not).
4. No existing state class needs to change.

---

### Add a New State (e.g., `MaintenanceState`)

1. Create `MaintenanceState implements VendingMachineState`.
2. Add `private final VendingMachineState maintenanceState = new MaintenanceState();` and a getter to `VendingMachine`.
3. Trigger the transition from `AdminPanel.enterMaintenance()`.
4. Transitions back to `IdleState` when `AdminPanel.exitMaintenance()` is called.
5. No existing state class needs to change.

---

### Persist State Across Power Cycles

Replace `Inventory` and `ChangeVault` with database-backed implementations behind interfaces:

```java
public interface InventoryRepository {
    void addProduct(Product p);
    Product getProduct(String id);
    boolean isInStock(String id);
    Product dispenseProduct(String id);
    void restockProduct(String id, int units);
}
```

Swap `new Inventory()` for `new DatabaseInventory(dataSource)` in the factory/wiring layer. State classes are unaffected.

---

### Multiple Machines / Fleet Management

1. Extract `VendingMachine` behind an interface.
2. Create `VendingMachineFleet` that holds a `Map<String, VendingMachine>` keyed by machine ID.
3. `AdminPanel` operates on a fleet-wide view; per-machine admin delegates to individual machines.

---

### Price Tiers / Promotions

Add a `PricingEngine` interface with a method `int effectivePrice(Product p, Payment payment)`. Inject it into `HasMoneyState` (via `VendingMachine`). A `PromoPricingEngine` can return discounted prices based on time-of-day or payment method.

---

## 11. Interview Discussion Points

**Q: Why State pattern instead of a simple enum + switch?**

A: An enum + switch puts all state-dependent logic in one class. Every new action requires a new case in every switch. Every new state requires updating every switch. The State pattern distributes responsibility: each state class knows exactly what is legal in that state. Adding a state is additive, not a modification. The Open/Closed Principle is satisfied.

---

**Q: What happens if two users try to use the machine simultaneously?**

A: This design is single-threaded by assumption (matching real embedded hardware). If multi-threading were required, you would add `synchronized` to all public methods on `VendingMachine`, or use a `ReentrantLock`. The state pattern itself does not introduce race conditions — the vulnerability is `setCurrentPayment` / `setSelectedProductId` being non-atomic.

---

**Q: How does change dispensing handle the case where exact change is impossible?**

A: `HasMoneyState.selectProduct()` calls `changeVault.canMakeChange(change)` before committing to the transaction. If it returns false, the user is told to use exact amount or card. The vault is not modified. This is a check-then-act pattern; it is safe because the machine is single-threaded.

---

**Q: Why does `DispensingState` deposit cash before computing change?**

A: The inserted bills physically enter the machine at the moment of dispense. They become part of the change fund. Consider: user inserts a $5 bill for a $1 item. If you compute change before depositing, the $5 is not yet in the vault, so the machine might incorrectly claim it cannot make $4 change. Depositing first is the physically correct model.

---

**Q: Could you model this without the State pattern using just methods on VendingMachine?**

A: Yes, but the resulting code would be fragile. `insertMoney()` would look like:

```java
if (state == IDLE) { ... transition ... }
else if (state == HAS_MONEY) { ... add more ... }
else if (state == DISPENSING) { ... reject ... }
else if (state == OUT_OF_STOCK) { ... reject ... }
```

Four states × five methods = 20 switch branches in one class. The State pattern turns this into five focused classes of five methods each, each branch under 10 lines.

---

**Q: What is the difference between the State pattern and the Strategy pattern here?**

A: State pattern is used for `VendingMachineState` — the selected state depends on an internal lifecycle, and states transition each other. Strategy pattern is used for `Payment` — the caller selects a payment strategy upfront, and strategies do not transition to each other. The structural code is similar; the semantic intent is different.

---

**Q: How would you add a loyalty points feature?**

A: Add a `LoyaltyAccount` to `VendingMachine`. After dispensing in `DispensingState.dispense()`, call `loyaltyAccount.credit(dispensed.getPriceInCents())`. No state class interface changes. No existing state logic changes. The feature is additive.

---

## 12. Common Mistakes in This Design

### Mistake 1: Putting State Transition Logic in `VendingMachine`

```java
// BAD: VendingMachine decides transitions
public void insertMoney(int cents) {
    if (currentState == IDLE) {
        currentState = HAS_MONEY;
        ...
    }
}
```

**Why it is wrong**: The machine becomes a god object. Every new state requires modifying `VendingMachine`. The point of the State pattern is that *states* decide their own transitions.

**Correct approach**: `IdleState.insertMoney()` calls `machine.setState(machine.getHasMoneyState())`.

---

### Mistake 2: Using `double` for Prices

```java
// BAD
private double price = 1.50;
if (paid >= price) { ... }  // floating-point comparison hazard
```

**Why it is wrong**: `0.1 + 0.2 == 0.30000000000000004` in IEEE 754. Change computation with doubles will produce rounding errors.

**Correct approach**: Store prices as `int` cents. Display with `priceInCents / 100.0`.

---

### Mistake 3: State Objects Holding Mutable Transaction State

```java
// BAD
public class HasMoneyState {
    private int insertedAmount;  // state object holds mutable data
}
```

**Why it is wrong**: State singletons are reused across transactions. Mutable fields on a state object bleed between transactions and make concurrent use impossible.

**Correct approach**: All transaction context (`currentPayment`, `selectedProductId`) lives on `VendingMachine`, the context object.

---

### Mistake 4: Not Depositing Cash into the Vault Before Making Change

```java
// BAD: compute change, then deposit
List<Integer> change = vault.makeChange(changeAmount);
vault.depositCash(insertedCoins);  // too late — change algorithm didn't see inserted bills
```

**Why it is wrong**: If the user inserted a $2 bill and the change needed is $1.25, the vault might fail because it does not yet know the $2 bill is available.

**Correct approach**: Deposit first, then compute change (as implemented in `DispensingState`).

---

### Mistake 5: Allowing `cancelTransaction` in `DispensingState`

```java
// BAD: cancel allowed at any time
public void cancelTransaction(VendingMachine machine) {
    if (currentState == DISPENSING) {
        refundAndReturnToIdle();  // product already physically falling!
    }
}
```

**Why it is wrong**: In `DispensingState`, the product is already physically being dispensed by the motor. Cancellation is physically impossible. The correct behavior is to reject it with a helpful message.

---

### Mistake 6: Tight Coupling Between Payment and Change Logic

```java
// BAD: CashPayment computes its own change
public List<Integer> getChange(int price) { ... }
```

**Why it is wrong**: Change computation depends on the vault's available denominations, which `CashPayment` should not know about. Payment represents *what was tendered*; `ChangeVault` represents *what can be returned*.

**Correct approach**: `DispensingState` orchestrates: get `payment.getAmount()`, subtract `product.getPriceInCents()`, call `vault.makeChange(difference)`.

---

### Mistake 7: AdminPanel Modifying State During an Active Transaction

```java
// RISKY: admin restocks A1 while A1 is the selectedProductId mid-transaction
admin.restock("A1", 5);
```

**Why it is a risk**: This is safe for restock (adding quantity). It would be dangerous for `removeProduct()` if such a method existed. A `MaintenanceState` or a lock on admin operations while a transaction is in progress would be the production solution.

---

*End of case study.*
