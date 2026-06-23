# Inventory Management System — Low-Level Design Case Study

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

A retail or manufacturing company manages physical goods across one or more warehouses. Stock levels change continuously due to purchases from suppliers, sales to customers, inter-warehouse transfers, and periodic adjustments from damage or loss. Without a structured system the business faces stockouts (lost revenue), overstock (wasted capital), and invisible shrinkage (theft/damage).

**Goal:** Build an Inventory Management System that:

- Tracks every unit of every product across every warehouse location.
- Automatically alerts stakeholders and creates reorder purchase orders when stock falls below a threshold.
- Records every stock movement so the history is fully auditable.
- Supports reconciliation: comparing the system's expected count against a physical count.
- Provides reports on current stock levels, low-stock items, and movement history.

This is a pure domain/service layer design. Persistence (databases) and HTTP transport are intentionally outside scope — the design must be clean enough that either can be plugged in without restructuring core logic.

---

## 2. Functional Requirements

1. **Product catalog management** — Create products with a name, category, SKU, unit price, and arbitrary key/value variants (size, color, etc.).
2. **Warehouse management** — Register multiple warehouses, each with a unique ID, name, and physical location string.
3. **Add stock** — Record the arrival of goods (from a purchase or manual addition) into a specific warehouse, at a specific bin/shelf location.
4. **Remove stock** — Record the consumption of goods (sale, manual removal) from a specific warehouse, preventing negative stock.
5. **Transfer stock** — Move a quantity of a product from one warehouse to another atomically; records both a TRANSFER_OUT and a TRANSFER_IN movement.
6. **Reorder threshold alerts** — When stock of a product in a warehouse drops at or below the `reorderThreshold`, fire an alert to all registered listeners.
7. **Auto-reorder** — One alert listener can automatically create a draft purchase order to the product's primary supplier.
8. **Supplier management** — Register suppliers with contact details, lead time, and the set of products they supply.
9. **Purchase order lifecycle** — Create a PO (DRAFT → SENT → CONFIRMED → PARTIALLY_RECEIVED → RECEIVED or CANCELLED). Receiving a PO adds stock automatically.
10. **Stock audit / reconciliation** — Compare the system's recorded quantity against a provided physical count for each item; produce a discrepancy report.
11. **Reporting** — Generate: (a) current stock levels per warehouse, (b) movement history with optional filters, (c) low-stock report across all warehouses.
12. **Movement history** — Every stock change (purchase, sale, transfer, adjustment, loss) is persisted as an immutable `StockMovement` record.

---

## 3. Non-Functional Requirements

| Concern | Requirement |
|---|---|
| **Correctness** | No operation may produce a negative quantity; all mutations are guarded. |
| **Auditability** | Every stock change produces an immutable `StockMovement` record with a timestamp and reason. |
| **Extensibility** | Adding a new alert channel (e.g., PagerDuty) requires only a new `InventoryAlertListener` implementation. Changing reorder quantity logic requires only a new `ReorderStrategy` implementation. |
| **Thread safety** | `InventoryItem` quantity mutations are synchronized; the design is safe for concurrent usage within a single JVM. |
| **Testability** | All services accept interfaces; concrete dependencies are injected, enabling unit tests with mocks. |
| **Observability** | `StockMovement` records form a complete audit log that can be exported or queried. |

---

## 4. Assumptions and Constraints

- **Single JVM, in-memory storage** — Persistence is out of scope. All maps are `ConcurrentHashMap` for thread safety, but distributed consistency is not addressed.
- **One primary supplier per product** — A product can have many suppliers, but the auto-reorder listener picks the first registered supplier.
- **Quantities are integers** — Fractional units (e.g., kg sold by weight) are out of scope.
- **No authentication / authorization** — All callers are trusted.
- **No multi-currency** — All prices are in a single currency (BigDecimal).
- **Reorder strategy is per-product** — Each product carries its own `ReorderStrategy` instance; it can be changed at runtime.
- **Alerts are synchronous** — Listeners are invoked inline during `removeStock` / `transferStock`. Async dispatch is an extension point.
- **PO receipt is all-or-nothing per call** — Partial receipts can be modelled by calling `receivePartial` multiple times.

---

## 5. Domain Model Description

### Entities

**Product** — the central catalog item. Identified by a system-generated `productId` and a human-readable `sku`. Carries variants (e.g., `{"size": "M", "color": "Red"}`), unit price, and the `ReorderStrategy` to use when this product needs restocking.

**ProductVariant** — a named variant of a product (e.g., "Blue / XL"). In a full system each variant would have its own SKU; here it is modelled as a value object holding a variant map.

**Warehouse** — a physical location. Holds an `inventory` map keyed by `productId → InventoryItem`.

**InventoryItem** — the join between a product and a warehouse. Tracks `quantity`, `reorderThreshold`, and the default `reorderQuantity` used as input to the reorder strategy. Also stores the bin/shelf location string within the warehouse.

**StockMovement** — immutable audit record. Every call to `addStock`, `removeStock`, or `transferStock` creates one or two of these. Contains `movementId`, `StockMovementType`, `productId`, `warehouseId`, `quantity`, `timestamp`, and a free-text `reason`.

**Supplier** — a vendor. Holds contact info, average `leadTimeDays`, and a set of `productId`s it supplies.

**PurchaseOrder** — a request to a supplier for specific products and quantities. Tracks `PurchaseOrderStatus` through its lifecycle. Receiving the PO calls back into `InventoryService` to add stock.

### Value Objects

**Address / location** — stored as a plain `String` in `Warehouse` and `InventoryItem` for simplicity.

### Services

**InventoryService** — main orchestrator. Manages warehouses, products, stock mutations, and alert firing. Holds the `StockMovement` log.

**SupplierService** — manages supplier registration and the mapping of products to suppliers.

**PurchaseOrderService** — manages PO lifecycle. Calls `InventoryService.addStock` when a PO is received.

**AuditService** — accepts a map of `{productId → physicalCount}` for a warehouse and returns a `ReconciliationReport` listing discrepancies.

**ReportingService** — queries `InventoryService` and `StockMovement` log to produce structured reports.

### Patterns

- **Observer (publish-subscribe)** — `InventoryService` fires alert events; `InventoryAlertListener` implementations react.
- **Strategy** — `ReorderStrategy` determines how many units to reorder; plugged into each `Product`.

---

## 6. Class Diagram (UML)

```
+------------------+         uses         +---------------------+
|   InventoryService|-------------------->| ReorderStrategy     |<<interface>>
|                  |                      +---------------------+
|  + addStock()    |                      | + calculateReorder()|
|  + removeStock() |                      +---------------------+
|  + transfer()    |           impl by           |
|  + addListener() |    +----------------------+ |
+------------------+    | FixedQuantityReorder | |
         |              +----------------------+ |
         |              | EOQReorderStrategy   | |
         |              +----------------------+
         |
         | fires alert to (Observer)
         v
+-------------------------+
| InventoryAlertListener  |<<interface>>
+-------------------------+
| + onLowStock(event)     |
+-------------------------+
         ^
         |-- EmailAlertListener
         |-- SlackAlertListener
         |-- AutoReorderListener -----> PurchaseOrderService
                                              |
                                              v
                                     +------------------+
                                     | PurchaseOrder    |
                                     | - poId           |
                                     | - supplier       |
                                     | - items          |
                                     | - status         |
                                     +------------------+

+------------------+     1     *    +------------------+
|    Warehouse     |------------>   |  InventoryItem   |
| - warehouseId    |                | - product        |
| - name           |                | - quantity       |
| - location       |                | - reorderThreshold|
| - inventory      |                | - reorderQuantity|
+------------------+                | - binLocation    |
                                    +------------------+
                                             |
                                             | references
                                             v
                                    +------------------+
                                    |    Product       |
                                    | - productId      |
                                    | - name           |
                                    | - category       |
                                    | - sku            |
                                    | - variants       |
                                    | - unitPrice      |
                                    | - reorderStrategy|
                                    +------------------+
                                             |
                                             | has many
                                             v
                                    +------------------+
                                    | ProductVariant   |
                                    | - variantId      |
                                    | - attributes     |
                                    +------------------+

+------------------+     1     *    +------------------+
|    Supplier      |------------>   | PurchaseOrder    |
| - supplierId     |                | - poId           |
| - name           |                | - items          |
| - contact        |                | - status         |
| - leadTimeDays   |                | - createdAt      |
| - productIds     |                | - expectedDelivery|
+------------------+                +------------------+

+------------------+
|  StockMovement   |
| - movementId     |
| - type           |
| - productId      |
| - warehouseId    |
| - quantity       |
| - timestamp      |
| - reason         |
+------------------+

+------------------+
|  AuditService    |
| + reconcile()    |
+------------------+

+------------------+
| ReportingService |
| + stockLevels()  |
| + movements()    |
| + lowStock()     |
+------------------+
```

---

## 7. Core Classes — Complete Java Implementation

All classes below are production-quality and fully compilable. They use only the Java standard library (Java 11+).

---

### 7.1 Enums

```java
// StockMovementType.java
package inventory;

public enum StockMovementType {
    PURCHASE,        // goods received from supplier
    SALE,            // goods dispatched to customer
    TRANSFER_IN,     // goods received from another warehouse
    TRANSFER_OUT,    // goods sent to another warehouse
    ADJUSTMENT,      // manual positive/negative correction
    LOSS             // shrinkage, damage, expiry
}
```

```java
// PurchaseOrderStatus.java
package inventory;

public enum PurchaseOrderStatus {
    DRAFT,
    SENT,
    CONFIRMED,
    PARTIALLY_RECEIVED,
    RECEIVED,
    CANCELLED
}
```

---

### 7.2 Domain Models

```java
// ProductVariant.java
package inventory;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * A specific variant of a product (e.g., size=M, color=Red).
 * Immutable after construction.
 */
public class ProductVariant {

    private final String variantId;
    private final Map<String, String> attributes; // e.g. {"size":"M","color":"Red"}

    public ProductVariant(Map<String, String> attributes) {
        this.variantId = UUID.randomUUID().toString();
        this.attributes = Collections.unmodifiableMap(new HashMap<>(attributes));
    }

    public String getVariantId() {
        return variantId;
    }

    public Map<String, String> getAttributes() {
        return attributes;
    }

    public String getAttribute(String key) {
        return attributes.getOrDefault(key, "");
    }

    @Override
    public String toString() {
        return "ProductVariant{variantId='" + variantId + "', attributes=" + attributes + "}";
    }
}
```

```java
// Product.java
package inventory;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * A catalog item. Carries its own ReorderStrategy so different products can
 * use different restocking logic independently.
 */
public class Product {

    private final String productId;
    private String name;
    private String category;
    private final String sku;             // human-readable unique stock-keeping unit
    private BigDecimal unitPrice;
    private final List<ProductVariant> variants;
    private ReorderStrategy reorderStrategy;

    public Product(String productId, String name, String category, String sku, BigDecimal unitPrice) {
        if (productId == null || productId.isBlank()) {
            throw new IllegalArgumentException("productId must not be blank");
        }
        if (sku == null || sku.isBlank()) {
            throw new IllegalArgumentException("sku must not be blank");
        }
        this.productId = productId;
        this.name = name;
        this.category = category;
        this.sku = sku;
        this.unitPrice = unitPrice;
        this.variants = new ArrayList<>();
        this.reorderStrategy = new FixedQuantityReorderStrategy(); // safe default
    }

    public void addVariant(ProductVariant variant) {
        variants.add(variant);
    }

    public List<ProductVariant> getVariants() {
        return Collections.unmodifiableList(variants);
    }

    // Getters
    public String getProductId()          { return productId; }
    public String getName()               { return name; }
    public String getCategory()           { return category; }
    public String getSku()                { return sku; }
    public BigDecimal getUnitPrice()      { return unitPrice; }
    public ReorderStrategy getReorderStrategy() { return reorderStrategy; }

    // Setters for mutable fields
    public void setName(String name)              { this.name = name; }
    public void setCategory(String category)      { this.category = category; }
    public void setUnitPrice(BigDecimal price)    { this.unitPrice = price; }
    public void setReorderStrategy(ReorderStrategy strategy) {
        if (strategy == null) throw new IllegalArgumentException("strategy must not be null");
        this.reorderStrategy = strategy;
    }

    @Override
    public String toString() {
        return "Product{productId='" + productId + "', sku='" + sku + "', name='" + name
                + "', category='" + category + "', unitPrice=" + unitPrice + "}";
    }
}
```

```java
// InventoryItem.java
package inventory;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * The join entity between a Product and a Warehouse.
 * Quantity is managed via AtomicInteger for thread-safe increments/decrements.
 */
public class InventoryItem {

    private final Product product;
    private final AtomicInteger quantity;
    private int reorderThreshold;   // alert fires when quantity <= this value
    private int reorderQuantity;    // default input to the reorder strategy
    private String binLocation;     // shelf/bin string, e.g. "A-03-2"

    public InventoryItem(Product product, int initialQuantity, int reorderThreshold,
                         int reorderQuantity, String binLocation) {
        if (initialQuantity < 0) {
            throw new IllegalArgumentException("initialQuantity cannot be negative");
        }
        if (reorderThreshold < 0) {
            throw new IllegalArgumentException("reorderThreshold cannot be negative");
        }
        this.product = product;
        this.quantity = new AtomicInteger(initialQuantity);
        this.reorderThreshold = reorderThreshold;
        this.reorderQuantity = reorderQuantity;
        this.binLocation = binLocation;
    }

    /**
     * Adds the given quantity. Always succeeds (positive delta only).
     *
     * @param amount positive integer
     * @return the new quantity
     */
    public int addQuantity(int amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("amount to add must be positive, got: " + amount);
        }
        return quantity.addAndGet(amount);
    }

    /**
     * Removes the given quantity atomically.
     * Throws if the result would be negative (prevents overselling).
     *
     * @param amount positive integer
     * @return the new quantity
     * @throws InsufficientStockException when current quantity < amount
     */
    public synchronized int removeQuantity(int amount) throws InsufficientStockException {
        if (amount <= 0) {
            throw new IllegalArgumentException("amount to remove must be positive, got: " + amount);
        }
        int current = quantity.get();
        if (current < amount) {
            throw new InsufficientStockException(
                    "Insufficient stock for product '" + product.getSku() + "': "
                    + "requested=" + amount + ", available=" + current);
        }
        quantity.addAndGet(-amount);
        return quantity.get();
    }

    public boolean isBelowReorderThreshold() {
        return quantity.get() <= reorderThreshold;
    }

    // Getters
    public Product getProduct()        { return product; }
    public int getQuantity()           { return quantity.get(); }
    public int getReorderThreshold()   { return reorderThreshold; }
    public int getReorderQuantity()    { return reorderQuantity; }
    public String getBinLocation()     { return binLocation; }

    // Setters
    public void setReorderThreshold(int t) { this.reorderThreshold = t; }
    public void setReorderQuantity(int q)  { this.reorderQuantity = q; }
    public void setBinLocation(String loc) { this.binLocation = loc; }

    @Override
    public String toString() {
        return "InventoryItem{product=" + product.getSku()
                + ", qty=" + quantity.get()
                + ", threshold=" + reorderThreshold
                + ", bin='" + binLocation + "'}";
    }
}
```

```java
// InsufficientStockException.java
package inventory;

/**
 * Thrown when a remove or transfer operation would result in negative stock.
 */
public class InsufficientStockException extends Exception {

    public InsufficientStockException(String message) {
        super(message);
    }
}
```

```java
// StockMovement.java
package inventory;

import java.time.Instant;
import java.util.UUID;

/**
 * Immutable audit record of every quantity change.
 * Created by InventoryService; never mutated after construction.
 */
public class StockMovement {

    private final String movementId;
    private final StockMovementType type;
    private final String productId;
    private final String warehouseId;
    private final int quantity;          // always positive; direction implied by type
    private final Instant timestamp;
    private final String reason;

    public StockMovement(StockMovementType type, String productId, String warehouseId,
                         int quantity, String reason) {
        this.movementId  = UUID.randomUUID().toString();
        this.type        = type;
        this.productId   = productId;
        this.warehouseId = warehouseId;
        this.quantity    = quantity;
        this.timestamp   = Instant.now();
        this.reason      = reason;
    }

    public String getMovementId()   { return movementId; }
    public StockMovementType getType() { return type; }
    public String getProductId()    { return productId; }
    public String getWarehouseId()  { return warehouseId; }
    public int getQuantity()        { return quantity; }
    public Instant getTimestamp()   { return timestamp; }
    public String getReason()       { return reason; }

    @Override
    public String toString() {
        return "StockMovement{id='" + movementId + "', type=" + type
                + ", product='" + productId + "', warehouse='" + warehouseId
                + "', qty=" + quantity + ", at=" + timestamp + ", reason='" + reason + "'}";
    }
}
```

```java
// Warehouse.java
package inventory;

import java.util.Collections;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

/**
 * A physical storage location.
 * inventory is keyed by productId for O(1) lookup.
 */
public class Warehouse {

    private final String warehouseId;
    private String name;
    private String location;  // free-text address, e.g. "123 Logistics Park, Chicago IL"
    private final Map<String, InventoryItem> inventory; // productId -> InventoryItem

    public Warehouse(String warehouseId, String name, String location) {
        this.warehouseId = warehouseId;
        this.name        = name;
        this.location    = location;
        this.inventory   = new ConcurrentHashMap<>();
    }

    /**
     * Adds an InventoryItem for a product that is not yet stocked here.
     * Throws if the product is already registered in this warehouse.
     */
    public void registerProduct(InventoryItem item) {
        String productId = item.getProduct().getProductId();
        if (inventory.containsKey(productId)) {
            throw new IllegalStateException(
                    "Product '" + productId + "' is already registered in warehouse '" + warehouseId + "'");
        }
        inventory.put(productId, item);
    }

    public Optional<InventoryItem> getItem(String productId) {
        return Optional.ofNullable(inventory.get(productId));
    }

    public Map<String, InventoryItem> getAllItems() {
        return Collections.unmodifiableMap(inventory);
    }

    public boolean hasProduct(String productId) {
        return inventory.containsKey(productId);
    }

    public String getWarehouseId() { return warehouseId; }
    public String getName()        { return name; }
    public String getLocation()    { return location; }

    public void setName(String name)         { this.name = name; }
    public void setLocation(String location) { this.location = location; }

    @Override
    public String toString() {
        return "Warehouse{id='" + warehouseId + "', name='" + name + "', location='" + location + "'}";
    }
}
```

```java
// Supplier.java
package inventory;

import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

/**
 * A vendor that supplies products.
 */
public class Supplier {

    private final String supplierId;
    private String name;
    private String contactEmail;
    private String contactPhone;
    private int leadTimeDays;
    private final Set<String> suppliedProductIds; // productIds this supplier can provide

    public Supplier(String supplierId, String name, String contactEmail,
                    String contactPhone, int leadTimeDays) {
        this.supplierId         = supplierId;
        this.name               = name;
        this.contactEmail       = contactEmail;
        this.contactPhone       = contactPhone;
        this.leadTimeDays       = leadTimeDays;
        this.suppliedProductIds = new HashSet<>();
    }

    public void addProduct(String productId) {
        suppliedProductIds.add(productId);
    }

    public void removeProduct(String productId) {
        suppliedProductIds.remove(productId);
    }

    public boolean supplies(String productId) {
        return suppliedProductIds.contains(productId);
    }

    public Set<String> getSuppliedProductIds() {
        return Collections.unmodifiableSet(suppliedProductIds);
    }

    public String getSupplierId()    { return supplierId; }
    public String getName()          { return name; }
    public String getContactEmail()  { return contactEmail; }
    public String getContactPhone()  { return contactPhone; }
    public int getLeadTimeDays()     { return leadTimeDays; }

    public void setName(String n)              { this.name = n; }
    public void setContactEmail(String e)      { this.contactEmail = e; }
    public void setContactPhone(String p)      { this.contactPhone = p; }
    public void setLeadTimeDays(int d)         { this.leadTimeDays = d; }

    @Override
    public String toString() {
        return "Supplier{id='" + supplierId + "', name='" + name
                + "', leadTimeDays=" + leadTimeDays + "}";
    }
}
```

```java
// PurchaseOrderItem.java
package inventory;

/**
 * A line item in a PurchaseOrder.
 */
public class PurchaseOrderItem {

    private final String productId;
    private final int orderedQuantity;
    private int receivedQuantity;

    public PurchaseOrderItem(String productId, int orderedQuantity) {
        if (orderedQuantity <= 0) {
            throw new IllegalArgumentException("orderedQuantity must be positive");
        }
        this.productId       = productId;
        this.orderedQuantity = orderedQuantity;
        this.receivedQuantity = 0;
    }

    /** Returns how many units are still outstanding. */
    public int outstandingQuantity() {
        return orderedQuantity - receivedQuantity;
    }

    public boolean isFullyReceived() {
        return receivedQuantity >= orderedQuantity;
    }

    public void receiveUnits(int units) {
        if (units <= 0) throw new IllegalArgumentException("units must be positive");
        if (receivedQuantity + units > orderedQuantity) {
            throw new IllegalStateException("Cannot receive more units than ordered for product " + productId);
        }
        this.receivedQuantity += units;
    }

    public String getProductId()        { return productId; }
    public int getOrderedQuantity()     { return orderedQuantity; }
    public int getReceivedQuantity()    { return receivedQuantity; }

    @Override
    public String toString() {
        return "POItem{product='" + productId + "', ordered=" + orderedQuantity
                + ", received=" + receivedQuantity + "}";
    }
}
```

```java
// PurchaseOrder.java
package inventory;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

/**
 * Represents a procurement request sent to a Supplier.
 */
public class PurchaseOrder {

    private final String poId;
    private final Supplier supplier;
    private final String destinationWarehouseId;
    private final List<PurchaseOrderItem> items;
    private PurchaseOrderStatus status;
    private final Instant createdAt;
    private Instant expectedDelivery;
    private Instant receivedAt;

    public PurchaseOrder(Supplier supplier, String destinationWarehouseId,
                         List<PurchaseOrderItem> items) {
        this.poId                   = UUID.randomUUID().toString();
        this.supplier               = supplier;
        this.destinationWarehouseId = destinationWarehouseId;
        this.items                  = new ArrayList<>(items);
        this.status                 = PurchaseOrderStatus.DRAFT;
        this.createdAt              = Instant.now();
        this.expectedDelivery       = createdAt.plus(supplier.getLeadTimeDays(), ChronoUnit.DAYS);
    }

    /** Transition: DRAFT -> SENT */
    public void send() {
        requireStatus(PurchaseOrderStatus.DRAFT);
        this.status = PurchaseOrderStatus.SENT;
    }

    /** Transition: SENT -> CONFIRMED */
    public void confirm() {
        requireStatus(PurchaseOrderStatus.SENT);
        this.status = PurchaseOrderStatus.CONFIRMED;
    }

    /**
     * Marks a specific line item as received.
     * The caller must then add stock via InventoryService.
     * Transitions: CONFIRMED/PARTIALLY_RECEIVED -> PARTIALLY_RECEIVED or RECEIVED.
     */
    public void receiveItem(String productId, int units) {
        if (status != PurchaseOrderStatus.CONFIRMED && status != PurchaseOrderStatus.PARTIALLY_RECEIVED) {
            throw new IllegalStateException(
                    "PO must be CONFIRMED or PARTIALLY_RECEIVED to receive items; current=" + status);
        }
        PurchaseOrderItem lineItem = items.stream()
                .filter(i -> i.getProductId().equals(productId))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException(
                        "Product '" + productId + "' not found in PO " + poId));
        lineItem.receiveUnits(units);
        boolean allReceived = items.stream().allMatch(PurchaseOrderItem::isFullyReceived);
        if (allReceived) {
            this.status   = PurchaseOrderStatus.RECEIVED;
            this.receivedAt = Instant.now();
        } else {
            this.status = PurchaseOrderStatus.PARTIALLY_RECEIVED;
        }
    }

    /** Transition: DRAFT | SENT | CONFIRMED -> CANCELLED */
    public void cancel() {
        if (status == PurchaseOrderStatus.RECEIVED) {
            throw new IllegalStateException("Cannot cancel a fully received PO");
        }
        this.status = PurchaseOrderStatus.CANCELLED;
    }

    private void requireStatus(PurchaseOrderStatus required) {
        if (this.status != required) {
            throw new IllegalStateException(
                    "PO " + poId + " must be in status " + required + " but is " + status);
        }
    }

    public String getPoId()                        { return poId; }
    public Supplier getSupplier()                  { return supplier; }
    public String getDestinationWarehouseId()      { return destinationWarehouseId; }
    public List<PurchaseOrderItem> getItems()      { return Collections.unmodifiableList(items); }
    public PurchaseOrderStatus getStatus()         { return status; }
    public Instant getCreatedAt()                  { return createdAt; }
    public Instant getExpectedDelivery()           { return expectedDelivery; }
    public Instant getReceivedAt()                 { return receivedAt; }

    public void setExpectedDelivery(Instant d)     { this.expectedDelivery = d; }

    @Override
    public String toString() {
        return "PurchaseOrder{poId='" + poId + "', supplier=" + supplier.getName()
                + ", status=" + status + ", items=" + items + "}";
    }
}
```

---

### 7.3 Strategy Pattern — Reorder Quantity

```java
// ReorderStrategy.java
package inventory;

/**
 * Strategy interface for determining how many units to order when restocking.
 * Implement this to plug in different ordering algorithms.
 */
public interface ReorderStrategy {

    /**
     * Calculates the quantity to order.
     *
     * @param currentStock   current on-hand quantity
     * @param reorderQty     the item's default reorder quantity (used as a baseline)
     * @param product        the product being restocked (for price/demand data)
     * @return               number of units to order (always >= 1)
     */
    int calculateReorderQuantity(int currentStock, int reorderQty, Product product);
}
```

```java
// FixedQuantityReorderStrategy.java
package inventory;

/**
 * The simplest reorder strategy: always order the item's configured reorderQuantity.
 * Use this when demand is stable and storage costs are negligible.
 */
public class FixedQuantityReorderStrategy implements ReorderStrategy {

    @Override
    public int calculateReorderQuantity(int currentStock, int reorderQty, Product product) {
        if (reorderQty <= 0) {
            // Defensive: if no reorder quantity is configured, default to a sensible minimum
            return 10;
        }
        return reorderQty;
    }

    @Override
    public String toString() {
        return "FixedQuantityReorderStrategy";
    }
}
```

```java
// EOQReorderStrategy.java
package inventory;

/**
 * Economic Order Quantity (EOQ) reorder strategy.
 *
 * EOQ = sqrt( (2 * D * S) / H )
 *
 *   D = annual demand units (estimated here as reorderQty * 12, a monthly proxy)
 *   S = ordering cost per order (fixed at $50 for this implementation)
 *   H = annual holding cost per unit (estimated as 20% of unit price)
 *
 * The formula minimises total inventory cost (ordering cost + holding cost).
 * A minimum order of 1 is enforced.
 */
public class EOQReorderStrategy implements ReorderStrategy {

    private static final double ORDERING_COST_PER_ORDER = 50.0; // dollars per order placed
    private static final double HOLDING_COST_FRACTION   = 0.20; // 20% of unit price per year

    @Override
    public int calculateReorderQuantity(int currentStock, int reorderQty, Product product) {
        // Annual demand: treat reorderQty as approximate monthly demand
        double annualDemand = reorderQty * 12.0;

        double unitPrice = product.getUnitPrice().doubleValue();
        // If unit price is zero or unavailable, fall back to fixed quantity
        if (unitPrice <= 0) {
            return Math.max(1, reorderQty);
        }

        double holdingCostPerUnit = unitPrice * HOLDING_COST_FRACTION;
        // EOQ formula
        double eoq = Math.sqrt((2.0 * annualDemand * ORDERING_COST_PER_ORDER) / holdingCostPerUnit);

        int result = (int) Math.ceil(eoq);
        return Math.max(1, result);
    }

    @Override
    public String toString() {
        return "EOQReorderStrategy";
    }
}
```

---

### 7.4 Observer Pattern — Alerts

```java
// LowStockEvent.java
package inventory;

/**
 * Immutable event fired when a product's stock in a warehouse falls to or below
 * the reorder threshold.
 */
public class LowStockEvent {

    private final Product product;
    private final Warehouse warehouse;
    private final int currentQuantity;
    private final int reorderThreshold;
    private final int suggestedReorderQuantity;

    public LowStockEvent(Product product, Warehouse warehouse,
                         int currentQuantity, int reorderThreshold,
                         int suggestedReorderQuantity) {
        this.product                 = product;
        this.warehouse               = warehouse;
        this.currentQuantity         = currentQuantity;
        this.reorderThreshold        = reorderThreshold;
        this.suggestedReorderQuantity = suggestedReorderQuantity;
    }

    public Product getProduct()                { return product; }
    public Warehouse getWarehouse()            { return warehouse; }
    public int getCurrentQuantity()            { return currentQuantity; }
    public int getReorderThreshold()           { return reorderThreshold; }
    public int getSuggestedReorderQuantity()   { return suggestedReorderQuantity; }

    @Override
    public String toString() {
        return "LowStockEvent{product=" + product.getSku()
                + ", warehouse=" + warehouse.getName()
                + ", qty=" + currentQuantity + "/" + reorderThreshold
                + ", suggest=" + suggestedReorderQuantity + "}";
    }
}
```

```java
// InventoryAlertListener.java
package inventory;

/**
 * Observer interface. Implement to react to low-stock events.
 */
public interface InventoryAlertListener {

    /**
     * Called by InventoryService whenever stock falls to or below the threshold.
     *
     * @param event contains product, warehouse, and reorder suggestion
     */
    void onLowStock(LowStockEvent event);
}
```

```java
// EmailAlertListener.java
package inventory;

/**
 * Sends an email notification when stock is low.
 * In production replace System.out with a real email client (JavaMail / SES / SendGrid).
 */
public class EmailAlertListener implements InventoryAlertListener {

    private final String recipientEmail;

    public EmailAlertListener(String recipientEmail) {
        this.recipientEmail = recipientEmail;
    }

    @Override
    public void onLowStock(LowStockEvent event) {
        String subject = "[LOW STOCK ALERT] " + event.getProduct().getName()
                + " @ " + event.getWarehouse().getName();
        String body = String.format(
                "Product  : %s (SKU: %s)%n"
                + "Warehouse: %s%n"
                + "Current  : %d units%n"
                + "Threshold: %d units%n"
                + "Suggested reorder qty: %d units%n"
                + "Please raise a purchase order promptly.",
                event.getProduct().getName(),
                event.getProduct().getSku(),
                event.getWarehouse().getName(),
                event.getCurrentQuantity(),
                event.getReorderThreshold(),
                event.getSuggestedReorderQuantity());

        // Simulated send — replace with actual email dispatch
        System.out.println("=== EMAIL to " + recipientEmail + " ===");
        System.out.println("Subject: " + subject);
        System.out.println(body);
        System.out.println("==========================================");
    }
}
```

```java
// SlackAlertListener.java
package inventory;

/**
 * Posts a Slack message when stock is low.
 * In production replace System.out with an HTTP call to the Slack Incoming Webhooks API.
 */
public class SlackAlertListener implements InventoryAlertListener {

    private final String webhookChannel;

    public SlackAlertListener(String webhookChannel) {
        this.webhookChannel = webhookChannel;
    }

    @Override
    public void onLowStock(LowStockEvent event) {
        String message = String.format(
                ":warning: *Low Stock Alert* — `%s` in *%s*%n"
                + ">Current: %d | Threshold: %d | Suggest reorder: %d",
                event.getProduct().getSku(),
                event.getWarehouse().getName(),
                event.getCurrentQuantity(),
                event.getReorderThreshold(),
                event.getSuggestedReorderQuantity());

        // Simulated Slack post — replace with HTTP POST to Slack webhook URL
        System.out.println("=== SLACK [" + webhookChannel + "] ===");
        System.out.println(message);
        System.out.println("======================================");
    }
}
```

```java
// AutoReorderListener.java
package inventory;

/**
 * Observer that automatically creates a draft Purchase Order when stock is low.
 * Delegates PO creation to PurchaseOrderService.
 * Requires SupplierService to look up the product's supplier.
 */
public class AutoReorderListener implements InventoryAlertListener {

    private final SupplierService supplierService;
    private final PurchaseOrderService purchaseOrderService;

    public AutoReorderListener(SupplierService supplierService,
                               PurchaseOrderService purchaseOrderService) {
        this.supplierService      = supplierService;
        this.purchaseOrderService = purchaseOrderService;
    }

    @Override
    public void onLowStock(LowStockEvent event) {
        String productId    = event.getProduct().getProductId();
        String warehouseId  = event.getWarehouse().getWarehouseId();
        int reorderQty      = event.getSuggestedReorderQuantity();

        Supplier supplier = supplierService.getPrimarySupplier(productId).orElse(null);
        if (supplier == null) {
            System.out.println("[AutoReorder] No supplier found for product "
                    + event.getProduct().getSku() + " — skipping auto-reorder.");
            return;
        }

        // Check if there is already an open PO for this product to avoid duplicates
        boolean openPoExists = purchaseOrderService.hasOpenPurchaseOrder(productId, warehouseId);
        if (openPoExists) {
            System.out.println("[AutoReorder] Open PO already exists for product "
                    + event.getProduct().getSku() + " — skipping duplicate.");
            return;
        }

        PurchaseOrderItem lineItem = new PurchaseOrderItem(productId, reorderQty);
        PurchaseOrder po = purchaseOrderService.createPurchaseOrder(
                supplier, warehouseId, java.util.List.of(lineItem));

        System.out.println("[AutoReorder] Created PO " + po.getPoId()
                + " for " + reorderQty + " units of " + event.getProduct().getSku()
                + " from supplier " + supplier.getName());
    }
}
```

---

### 7.5 Services

```java
// InventoryService.java
package inventory;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.stream.Collectors;

/**
 * Main orchestrator for all inventory operations.
 *
 * Responsibilities:
 *   - Maintain the product catalog and warehouse registry.
 *   - Perform stock mutations (add, remove, transfer).
 *   - Fire LowStockEvents to registered InventoryAlertListeners.
 *   - Maintain an ordered StockMovement audit log.
 */
public class InventoryService {

    private final Map<String, Product>   products;    // productId -> Product
    private final Map<String, Warehouse> warehouses;  // warehouseId -> Warehouse
    private final List<StockMovement>    movements;   // ordered audit log
    private final List<InventoryAlertListener> alertListeners;

    public InventoryService() {
        this.products       = new ConcurrentHashMap<>();
        this.warehouses     = new ConcurrentHashMap<>();
        this.movements      = Collections.synchronizedList(new ArrayList<>());
        this.alertListeners = new CopyOnWriteArrayList<>();
    }

    // -------------------------------------------------------------------------
    // Catalog management
    // -------------------------------------------------------------------------

    /** Registers a product in the catalog. Throws if the productId already exists. */
    public void registerProduct(Product product) {
        if (products.containsKey(product.getProductId())) {
            throw new IllegalStateException(
                    "Product with id '" + product.getProductId() + "' is already registered.");
        }
        products.put(product.getProductId(), product);
    }

    public Optional<Product> findProduct(String productId) {
        return Optional.ofNullable(products.get(productId));
    }

    public Optional<Product> findProductBySku(String sku) {
        return products.values().stream()
                .filter(p -> p.getSku().equals(sku))
                .findFirst();
    }

    // -------------------------------------------------------------------------
    // Warehouse management
    // -------------------------------------------------------------------------

    /** Registers a warehouse. Throws if the warehouseId already exists. */
    public void registerWarehouse(Warehouse warehouse) {
        if (warehouses.containsKey(warehouse.getWarehouseId())) {
            throw new IllegalStateException(
                    "Warehouse with id '" + warehouse.getWarehouseId() + "' is already registered.");
        }
        warehouses.put(warehouse.getWarehouseId(), warehouse);
    }

    public Optional<Warehouse> findWarehouse(String warehouseId) {
        return Optional.ofNullable(warehouses.get(warehouseId));
    }

    /**
     * Adds a product to a warehouse inventory with initial configuration.
     * Call this before the first addStock for that product-warehouse combination.
     */
    public void registerProductInWarehouse(String productId, String warehouseId,
                                           int reorderThreshold, int reorderQuantity,
                                           String binLocation) {
        Product   product   = requireProduct(productId);
        Warehouse warehouse = requireWarehouse(warehouseId);
        InventoryItem item = new InventoryItem(product, 0, reorderThreshold, reorderQuantity, binLocation);
        warehouse.registerProduct(item);
    }

    // -------------------------------------------------------------------------
    // Stock mutations
    // -------------------------------------------------------------------------

    /**
     * Adds stock to a warehouse.
     * Creates a StockMovement of the given type (typically PURCHASE or ADJUSTMENT).
     *
     * @param productId   the product to add
     * @param warehouseId the target warehouse
     * @param quantity    positive number of units to add
     * @param type        movement type (PURCHASE, ADJUSTMENT, etc.)
     * @param reason      free-text reason for audit log
     * @return            the new on-hand quantity
     */
    public int addStock(String productId, String warehouseId, int quantity,
                        StockMovementType type, String reason) {
        InventoryItem item = requireInventoryItem(productId, warehouseId);
        int newQty = item.addQuantity(quantity);
        recordMovement(type, productId, warehouseId, quantity, reason);
        return newQty;
    }

    /**
     * Removes stock from a warehouse.
     * Fires a LowStockEvent if the resulting quantity is at or below the reorder threshold.
     *
     * @param productId   the product to remove
     * @param warehouseId the source warehouse
     * @param quantity    positive number of units to remove
     * @param type        movement type (SALE, LOSS, etc.)
     * @param reason      free-text reason for audit log
     * @return            the new on-hand quantity
     * @throws InsufficientStockException if current stock < quantity
     */
    public int removeStock(String productId, String warehouseId, int quantity,
                           StockMovementType type, String reason)
            throws InsufficientStockException {
        InventoryItem item = requireInventoryItem(productId, warehouseId);
        int newQty = item.removeQuantity(quantity);
        recordMovement(type, productId, warehouseId, quantity, reason);
        checkAndFireAlert(item, warehouses.get(warehouseId));
        return newQty;
    }

    /**
     * Transfers stock from one warehouse to another atomically.
     * Records a TRANSFER_OUT in the source warehouse and a TRANSFER_IN in the destination.
     * If the destination warehouse does not yet have the product, it is auto-registered
     * with zero threshold and the same bin as "INCOMING".
     *
     * @throws InsufficientStockException if source stock < quantity
     */
    public void transferStock(String productId, String fromWarehouseId,
                              String toWarehouseId, int quantity, String reason)
            throws InsufficientStockException {
        if (fromWarehouseId.equals(toWarehouseId)) {
            throw new IllegalArgumentException("Source and destination warehouses must differ.");
        }
        Warehouse from = requireWarehouse(fromWarehouseId);
        Warehouse to   = requireWarehouse(toWarehouseId);

        InventoryItem sourceItem = requireInventoryItem(productId, fromWarehouseId);

        // Auto-register in destination if not already present
        if (!to.hasProduct(productId)) {
            Product product = requireProduct(productId);
            InventoryItem destItem = new InventoryItem(product, 0, 0, 0, "INCOMING");
            to.registerProduct(destItem);
        }

        // Perform mutation — remove first to fail fast on insufficient stock
        sourceItem.removeQuantity(quantity);
        recordMovement(StockMovementType.TRANSFER_OUT, productId, fromWarehouseId, quantity, reason);

        to.getItem(productId).ifPresent(destItem -> {
            destItem.addQuantity(quantity);
            recordMovement(StockMovementType.TRANSFER_IN, productId, toWarehouseId, quantity, reason);
        });

        // Check reorder threshold for the source warehouse after transfer
        checkAndFireAlert(sourceItem, from);
    }

    // -------------------------------------------------------------------------
    // Querying
    // -------------------------------------------------------------------------

    public int getStockLevel(String productId, String warehouseId) {
        return requireInventoryItem(productId, warehouseId).getQuantity();
    }

    public List<StockMovement> getAllMovements() {
        return Collections.unmodifiableList(new ArrayList<>(movements));
    }

    public List<StockMovement> getMovementsForProduct(String productId) {
        return movements.stream()
                .filter(m -> m.getProductId().equals(productId))
                .collect(Collectors.toList());
    }

    public List<StockMovement> getMovementsForWarehouse(String warehouseId) {
        return movements.stream()
                .filter(m -> m.getWarehouseId().equals(warehouseId))
                .collect(Collectors.toList());
    }

    public Map<String, Warehouse> getAllWarehouses() {
        return Collections.unmodifiableMap(warehouses);
    }

    public Map<String, Product> getAllProducts() {
        return Collections.unmodifiableMap(products);
    }

    // -------------------------------------------------------------------------
    // Observer management
    // -------------------------------------------------------------------------

    public void addAlertListener(InventoryAlertListener listener) {
        alertListeners.add(listener);
    }

    public void removeAlertListener(InventoryAlertListener listener) {
        alertListeners.remove(listener);
    }

    // -------------------------------------------------------------------------
    // Private helpers
    // -------------------------------------------------------------------------

    private void recordMovement(StockMovementType type, String productId,
                                String warehouseId, int quantity, String reason) {
        movements.add(new StockMovement(type, productId, warehouseId, quantity, reason));
    }

    private void checkAndFireAlert(InventoryItem item, Warehouse warehouse) {
        if (item.isBelowReorderThreshold()) {
            Product product = item.getProduct();
            int suggestedQty = product.getReorderStrategy().calculateReorderQuantity(
                    item.getQuantity(), item.getReorderQuantity(), product);
            LowStockEvent event = new LowStockEvent(
                    product, warehouse,
                    item.getQuantity(), item.getReorderThreshold(),
                    suggestedQty);
            for (InventoryAlertListener listener : alertListeners) {
                try {
                    listener.onLowStock(event);
                } catch (Exception ex) {
                    // One bad listener must not prevent others from receiving the alert
                    System.err.println("[InventoryService] Alert listener " + listener.getClass().getSimpleName()
                            + " threw an exception: " + ex.getMessage());
                }
            }
        }
    }

    private Product requireProduct(String productId) {
        return findProduct(productId)
                .orElseThrow(() -> new IllegalArgumentException("Unknown productId: " + productId));
    }

    private Warehouse requireWarehouse(String warehouseId) {
        return findWarehouse(warehouseId)
                .orElseThrow(() -> new IllegalArgumentException("Unknown warehouseId: " + warehouseId));
    }

    private InventoryItem requireInventoryItem(String productId, String warehouseId) {
        Warehouse warehouse = requireWarehouse(warehouseId);
        return warehouse.getItem(productId)
                .orElseThrow(() -> new IllegalStateException(
                        "Product '" + productId + "' is not registered in warehouse '" + warehouseId + "'"));
    }
}
```

```java
// SupplierService.java
package inventory;

import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

/**
 * Manages supplier registration and the product-to-supplier mapping.
 */
public class SupplierService {

    private final Map<String, Supplier> suppliers; // supplierId -> Supplier

    public SupplierService() {
        this.suppliers = new ConcurrentHashMap<>();
    }

    public void registerSupplier(Supplier supplier) {
        if (suppliers.containsKey(supplier.getSupplierId())) {
            throw new IllegalStateException("Supplier '" + supplier.getSupplierId() + "' already registered.");
        }
        suppliers.put(supplier.getSupplierId(), supplier);
    }

    public Optional<Supplier> findSupplier(String supplierId) {
        return Optional.ofNullable(suppliers.get(supplierId));
    }

    /**
     * Returns the first supplier (alphabetical by supplierId) that supplies the given product.
     * Deterministic: always returns the same supplier for the same product given the same data.
     */
    public Optional<Supplier> getPrimarySupplier(String productId) {
        return suppliers.values().stream()
                .filter(s -> s.supplies(productId))
                .min(java.util.Comparator.comparing(Supplier::getSupplierId));
    }

    public List<Supplier> getSuppliersForProduct(String productId) {
        return suppliers.values().stream()
                .filter(s -> s.supplies(productId))
                .collect(Collectors.toList());
    }

    public Map<String, Supplier> getAllSuppliers() {
        return Collections.unmodifiableMap(suppliers);
    }
}
```

```java
// PurchaseOrderService.java
package inventory;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

/**
 * Manages the full Purchase Order lifecycle.
 * Calls back into InventoryService when goods are received to add stock.
 */
public class PurchaseOrderService {

    private final Map<String, PurchaseOrder> purchaseOrders; // poId -> PurchaseOrder
    private final InventoryService inventoryService;

    public PurchaseOrderService(InventoryService inventoryService) {
        this.purchaseOrders  = new ConcurrentHashMap<>();
        this.inventoryService = inventoryService;
    }

    /** Creates a draft PO and stores it. */
    public PurchaseOrder createPurchaseOrder(Supplier supplier, String destinationWarehouseId,
                                             List<PurchaseOrderItem> items) {
        PurchaseOrder po = new PurchaseOrder(supplier, destinationWarehouseId, items);
        purchaseOrders.put(po.getPoId(), po);
        return po;
    }

    /** Sends a DRAFT PO (transitions to SENT). */
    public void sendPurchaseOrder(String poId) {
        requirePO(poId).send();
    }

    /** Confirms a SENT PO (transitions to CONFIRMED). */
    public void confirmPurchaseOrder(String poId) {
        requirePO(poId).confirm();
    }

    /**
     * Records receipt of goods for a specific line item in the PO.
     * Automatically calls InventoryService.addStock to update on-hand quantity.
     *
     * @param poId      the purchase order
     * @param productId the line item product
     * @param units     number of units received in this shipment
     */
    public void receiveGoods(String poId, String productId, int units)
            throws InsufficientStockException {
        PurchaseOrder po = requirePO(poId);
        po.receiveItem(productId, units);

        // Ensure the product is registered in the destination warehouse
        String warehouseId = po.getDestinationWarehouseId();
        Warehouse warehouse = inventoryService.findWarehouse(warehouseId)
                .orElseThrow(() -> new IllegalArgumentException(
                        "Destination warehouse '" + warehouseId + "' not found"));

        if (!warehouse.hasProduct(productId)) {
            // Auto-register with no threshold/reorder so stock can be added
            inventoryService.registerProductInWarehouse(productId, warehouseId, 0, 0, "RECEIVING");
        }

        inventoryService.addStock(productId, warehouseId, units,
                StockMovementType.PURCHASE, "PO receipt: " + poId);

        System.out.println("[PurchaseOrderService] Received " + units + " units of product '"
                + productId + "' for PO " + poId + ". PO status: " + po.getStatus());
    }

    /** Cancels a PO that has not yet been fully received. */
    public void cancelPurchaseOrder(String poId) {
        requirePO(poId).cancel();
    }

    /**
     * Returns true if there is any PO for the given product/warehouse that is
     * DRAFT, SENT, CONFIRMED, or PARTIALLY_RECEIVED.
     * Used by AutoReorderListener to prevent duplicate orders.
     */
    public boolean hasOpenPurchaseOrder(String productId, String warehouseId) {
        return purchaseOrders.values().stream()
                .anyMatch(po ->
                        po.getDestinationWarehouseId().equals(warehouseId)
                        && po.getStatus() != PurchaseOrderStatus.RECEIVED
                        && po.getStatus() != PurchaseOrderStatus.CANCELLED
                        && po.getItems().stream().anyMatch(i -> i.getProductId().equals(productId)));
    }

    public Optional<PurchaseOrder> findPurchaseOrder(String poId) {
        return Optional.ofNullable(purchaseOrders.get(poId));
    }

    public List<PurchaseOrder> getAllPurchaseOrders() {
        return Collections.unmodifiableList(new ArrayList<>(purchaseOrders.values()));
    }

    public List<PurchaseOrder> getOpenPurchaseOrders() {
        return purchaseOrders.values().stream()
                .filter(po -> po.getStatus() != PurchaseOrderStatus.RECEIVED
                           && po.getStatus() != PurchaseOrderStatus.CANCELLED)
                .collect(Collectors.toList());
    }

    private PurchaseOrder requirePO(String poId) {
        return findPurchaseOrder(poId)
                .orElseThrow(() -> new IllegalArgumentException("Unknown PO id: " + poId));
    }
}
```

```java
// AuditService.java
package inventory;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;

/**
 * Performs stock reconciliation: compares the system's recorded quantity
 * against a physical count and produces a discrepancy report.
 */
public class AuditService {

    private final InventoryService inventoryService;

    public AuditService(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    /**
     * Reconciles the inventory for a warehouse.
     *
     * @param warehouseId   the warehouse being audited
     * @param physicalCounts map of productId -> units counted physically
     * @return              a ReconciliationReport detailing every discrepancy
     */
    public ReconciliationReport reconcile(String warehouseId, Map<String, Integer> physicalCounts) {
        Warehouse warehouse = inventoryService.findWarehouse(warehouseId)
                .orElseThrow(() -> new IllegalArgumentException("Unknown warehouse: " + warehouseId));

        List<DiscrepancyRecord> discrepancies = new ArrayList<>();
        int totalSystemQty   = 0;
        int totalPhysicalQty = 0;

        // Check every product tracked in the system for this warehouse
        for (Map.Entry<String, InventoryItem> entry : warehouse.getAllItems().entrySet()) {
            String productId    = entry.getKey();
            InventoryItem item  = entry.getValue();
            int systemQty       = item.getQuantity();
            int physicalQty     = physicalCounts.getOrDefault(productId, 0);
            int delta           = physicalQty - systemQty;

            totalSystemQty   += systemQty;
            totalPhysicalQty += physicalQty;

            if (delta != 0) {
                discrepancies.add(new DiscrepancyRecord(
                        productId,
                        item.getProduct().getSku(),
                        systemQty,
                        physicalQty,
                        delta));
            }
        }

        // Also flag products counted physically but not in the system
        for (Map.Entry<String, Integer> entry : physicalCounts.entrySet()) {
            String productId = entry.getKey();
            if (!warehouse.hasProduct(productId) && entry.getValue() > 0) {
                discrepancies.add(new DiscrepancyRecord(
                        productId, "UNKNOWN_SKU", 0, entry.getValue(), entry.getValue()));
            }
        }

        return new ReconciliationReport(
                warehouseId, Instant.now(),
                totalSystemQty, totalPhysicalQty,
                Collections.unmodifiableList(discrepancies));
    }

    /**
     * Applies the physical counts as adjustments to bring the system in line.
     * Creates ADJUSTMENT or LOSS movements for each discrepancy.
     *
     * @param report    the reconciliation report to apply
     */
    public void applyReconciliation(ReconciliationReport report)
            throws InsufficientStockException {
        for (DiscrepancyRecord dr : report.getDiscrepancies()) {
            String productId   = dr.getProductId();
            String warehouseId = report.getWarehouseId();
            int delta          = dr.getDelta(); // physicalQty - systemQty

            if (delta > 0) {
                // Physical count is higher than system — add as positive adjustment
                inventoryService.addStock(productId, warehouseId, delta,
                        StockMovementType.ADJUSTMENT,
                        "Reconciliation adjustment: found +" + delta + " extra units");
            } else if (delta < 0) {
                // Physical count is lower than system — record as loss
                inventoryService.removeStock(productId, warehouseId, Math.abs(delta),
                        StockMovementType.LOSS,
                        "Reconciliation adjustment: " + delta + " units missing");
            }
        }
    }

    // -------------------------------------------------------------------------
    // Inner value objects for the report
    // -------------------------------------------------------------------------

    public static class DiscrepancyRecord {
        private final String productId;
        private final String sku;
        private final int systemQuantity;
        private final int physicalQuantity;
        private final int delta; // physicalQty - systemQty; positive = surplus, negative = shortage

        public DiscrepancyRecord(String productId, String sku, int systemQuantity,
                                 int physicalQuantity, int delta) {
            this.productId       = productId;
            this.sku             = sku;
            this.systemQuantity  = systemQuantity;
            this.physicalQuantity = physicalQuantity;
            this.delta           = delta;
        }

        public String getProductId()       { return productId; }
        public String getSku()             { return sku; }
        public int getSystemQuantity()     { return systemQuantity; }
        public int getPhysicalQuantity()   { return physicalQuantity; }
        public int getDelta()              { return delta; }

        @Override
        public String toString() {
            String direction = delta > 0 ? "SURPLUS" : "SHORTAGE";
            return String.format("  %-30s system=%-5d physical=%-5d delta=%+d (%s)",
                    sku + " (" + productId + ")",
                    systemQuantity, physicalQuantity, delta, direction);
        }
    }

    public static class ReconciliationReport {
        private final String warehouseId;
        private final Instant auditedAt;
        private final int totalSystemQuantity;
        private final int totalPhysicalQuantity;
        private final List<DiscrepancyRecord> discrepancies;

        public ReconciliationReport(String warehouseId, Instant auditedAt,
                                    int totalSystemQuantity, int totalPhysicalQuantity,
                                    List<DiscrepancyRecord> discrepancies) {
            this.warehouseId           = warehouseId;
            this.auditedAt             = auditedAt;
            this.totalSystemQuantity   = totalSystemQuantity;
            this.totalPhysicalQuantity = totalPhysicalQuantity;
            this.discrepancies         = discrepancies;
        }

        public boolean hasDiscrepancies()              { return !discrepancies.isEmpty(); }
        public String getWarehouseId()                 { return warehouseId; }
        public Instant getAuditedAt()                  { return auditedAt; }
        public int getTotalSystemQuantity()            { return totalSystemQuantity; }
        public int getTotalPhysicalQuantity()          { return totalPhysicalQuantity; }
        public List<DiscrepancyRecord> getDiscrepancies() { return discrepancies; }

        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();
            sb.append("=== Reconciliation Report ===\n");
            sb.append("Warehouse  : ").append(warehouseId).append("\n");
            sb.append("Audited at : ").append(auditedAt).append("\n");
            sb.append("System total   : ").append(totalSystemQuantity).append("\n");
            sb.append("Physical total : ").append(totalPhysicalQuantity).append("\n");
            sb.append("Discrepancies  : ").append(discrepancies.size()).append("\n");
            discrepancies.forEach(d -> sb.append(d).append("\n"));
            sb.append("=============================");
            return sb.toString();
        }
    }
}
```

```java
// ReportingService.java
package inventory;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * Generates operational reports from InventoryService data.
 */
public class ReportingService {

    private final InventoryService inventoryService;

    public ReportingService(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    /**
     * Returns a snapshot of current stock levels for all products in a warehouse.
     */
    public List<StockLevelRecord> getStockLevels(String warehouseId) {
        Warehouse warehouse = inventoryService.findWarehouse(warehouseId)
                .orElseThrow(() -> new IllegalArgumentException("Unknown warehouse: " + warehouseId));

        return warehouse.getAllItems().values().stream()
                .map(item -> new StockLevelRecord(
                        item.getProduct().getProductId(),
                        item.getProduct().getSku(),
                        item.getProduct().getName(),
                        warehouseId,
                        warehouse.getName(),
                        item.getQuantity(),
                        item.getReorderThreshold(),
                        item.getBinLocation()))
                .sorted(Comparator.comparing(StockLevelRecord::getSku))
                .collect(Collectors.toList());
    }

    /**
     * Returns stock levels for all warehouses for a single product.
     */
    public List<StockLevelRecord> getStockLevelsForProduct(String productId) {
        List<StockLevelRecord> results = new ArrayList<>();
        for (Warehouse warehouse : inventoryService.getAllWarehouses().values()) {
            warehouse.getItem(productId).ifPresent(item ->
                    results.add(new StockLevelRecord(
                            productId,
                            item.getProduct().getSku(),
                            item.getProduct().getName(),
                            warehouse.getWarehouseId(),
                            warehouse.getName(),
                            item.getQuantity(),
                            item.getReorderThreshold(),
                            item.getBinLocation())));
        }
        return results;
    }

    /**
     * Returns all stock movement records, optionally filtered by product and warehouse.
     * Pass null for either filter to skip it.
     */
    public List<StockMovement> getMovementHistory(String productIdFilter, String warehouseIdFilter) {
        return inventoryService.getAllMovements().stream()
                .filter(m -> productIdFilter  == null || m.getProductId().equals(productIdFilter))
                .filter(m -> warehouseIdFilter == null || m.getWarehouseId().equals(warehouseIdFilter))
                .sorted(Comparator.comparing(StockMovement::getTimestamp))
                .collect(Collectors.toList());
    }

    /**
     * Returns all products that are at or below their reorder threshold across all warehouses.
     */
    public List<LowStockRecord> getLowStockReport() {
        List<LowStockRecord> results = new ArrayList<>();
        for (Warehouse warehouse : inventoryService.getAllWarehouses().values()) {
            for (InventoryItem item : warehouse.getAllItems().values()) {
                if (item.isBelowReorderThreshold()) {
                    Product product = item.getProduct();
                    int suggested = product.getReorderStrategy().calculateReorderQuantity(
                            item.getQuantity(), item.getReorderQuantity(), product);
                    results.add(new LowStockRecord(
                            product.getProductId(),
                            product.getSku(),
                            product.getName(),
                            warehouse.getWarehouseId(),
                            warehouse.getName(),
                            item.getQuantity(),
                            item.getReorderThreshold(),
                            suggested));
                }
            }
        }
        results.sort(Comparator.comparingInt(LowStockRecord::getCurrentQuantity));
        return results;
    }

    // -------------------------------------------------------------------------
    // Report record types
    // -------------------------------------------------------------------------

    public static class StockLevelRecord {
        private final String productId;
        private final String sku;
        private final String productName;
        private final String warehouseId;
        private final String warehouseName;
        private final int currentQuantity;
        private final int reorderThreshold;
        private final String binLocation;

        public StockLevelRecord(String productId, String sku, String productName,
                                String warehouseId, String warehouseName,
                                int currentQuantity, int reorderThreshold, String binLocation) {
            this.productId       = productId;
            this.sku             = sku;
            this.productName     = productName;
            this.warehouseId     = warehouseId;
            this.warehouseName   = warehouseName;
            this.currentQuantity = currentQuantity;
            this.reorderThreshold = reorderThreshold;
            this.binLocation     = binLocation;
        }

        public String getProductId()       { return productId; }
        public String getSku()             { return sku; }
        public String getProductName()     { return productName; }
        public String getWarehouseId()     { return warehouseId; }
        public String getWarehouseName()   { return warehouseName; }
        public int getCurrentQuantity()    { return currentQuantity; }
        public int getReorderThreshold()   { return reorderThreshold; }
        public String getBinLocation()     { return binLocation; }

        @Override
        public String toString() {
            return String.format("  %-15s %-25s qty=%-5d threshold=%-5d bin=%-10s [%s]",
                    sku, productName, currentQuantity, reorderThreshold, binLocation, warehouseName);
        }
    }

    public static class LowStockRecord {
        private final String productId;
        private final String sku;
        private final String productName;
        private final String warehouseId;
        private final String warehouseName;
        private final int currentQuantity;
        private final int reorderThreshold;
        private final int suggestedReorderQuantity;

        public LowStockRecord(String productId, String sku, String productName,
                              String warehouseId, String warehouseName,
                              int currentQuantity, int reorderThreshold,
                              int suggestedReorderQuantity) {
            this.productId                = productId;
            this.sku                      = sku;
            this.productName              = productName;
            this.warehouseId              = warehouseId;
            this.warehouseName            = warehouseName;
            this.currentQuantity          = currentQuantity;
            this.reorderThreshold         = reorderThreshold;
            this.suggestedReorderQuantity = suggestedReorderQuantity;
        }

        public String getProductId()              { return productId; }
        public String getSku()                    { return sku; }
        public String getProductName()            { return productName; }
        public String getWarehouseId()            { return warehouseId; }
        public String getWarehouseName()          { return warehouseName; }
        public int getCurrentQuantity()           { return currentQuantity; }
        public int getReorderThreshold()          { return reorderThreshold; }
        public int getSuggestedReorderQuantity()  { return suggestedReorderQuantity; }

        @Override
        public String toString() {
            return String.format("  [LOW] %-15s %-25s qty=%-3d threshold=%-3d suggest=%d [%s]",
                    sku, productName, currentQuantity, reorderThreshold,
                    suggestedReorderQuantity, warehouseName);
        }
    }
}
```

---

### 7.6 Main Demo

```java
// InventoryManagementDemo.java
package inventory;

import java.math.BigDecimal;
import java.util.List;
import java.util.Map;

/**
 * End-to-end demonstration of the Inventory Management System.
 *
 * Scenario:
 *   1. Register a product, warehouse, and supplier.
 *   2. Add initial stock via a purchase.
 *   3. Process a large sale that drops stock below the reorder threshold.
 *   4. Observe: email alert fires, Slack alert fires, auto-reorder creates a PO.
 *   5. Receive the PO goods (stock is replenished).
 *   6. Transfer some stock to a second warehouse.
 *   7. Run a stock audit with a deliberate discrepancy.
 *   8. Print reports: stock levels, movement history, low-stock report.
 */
public class InventoryManagementDemo {

    public static void main(String[] args) throws InsufficientStockException {

        // ---------------------------------------------------------------
        // Bootstrap services
        // ---------------------------------------------------------------
        InventoryService      inventoryService      = new InventoryService();
        SupplierService       supplierService       = new SupplierService();
        PurchaseOrderService  purchaseOrderService  = new PurchaseOrderService(inventoryService);
        AuditService          auditService          = new AuditService(inventoryService);
        ReportingService      reportingService      = new ReportingService(inventoryService);

        // ---------------------------------------------------------------
        // Wire up observers (alert listeners)
        // ---------------------------------------------------------------
        inventoryService.addAlertListener(new EmailAlertListener("warehouse-ops@company.com"));
        inventoryService.addAlertListener(new SlackAlertListener("#inventory-alerts"));
        inventoryService.addAlertListener(
                new AutoReorderListener(supplierService, purchaseOrderService));

        // ---------------------------------------------------------------
        // Step 1: Create a product
        // ---------------------------------------------------------------
        System.out.println("\n==== STEP 1: Register Product ====");
        Product tShirt = new Product("PROD-001", "Classic T-Shirt", "Apparel",
                                     "APP-TSHIRT-001", new BigDecimal("19.99"));
        tShirt.setReorderStrategy(new EOQReorderStrategy());

        tShirt.addVariant(new ProductVariant(Map.of("size", "S", "color", "White")));
        tShirt.addVariant(new ProductVariant(Map.of("size", "M", "color", "White")));
        tShirt.addVariant(new ProductVariant(Map.of("size", "L", "color", "White")));

        inventoryService.registerProduct(tShirt);
        System.out.println("Registered: " + tShirt);

        // A second product
        Product hoodie = new Product("PROD-002", "Zip Hoodie", "Apparel",
                                     "APP-HOODIE-001", new BigDecimal("49.99"));
        hoodie.setReorderStrategy(new FixedQuantityReorderStrategy());
        inventoryService.registerProduct(hoodie);
        System.out.println("Registered: " + hoodie);

        // ---------------------------------------------------------------
        // Step 2: Create warehouses
        // ---------------------------------------------------------------
        System.out.println("\n==== STEP 2: Register Warehouses ====");
        Warehouse warehouseA = new Warehouse("WH-001", "Main Distribution Center",
                                             "100 Logistics Blvd, Chicago IL 60601");
        Warehouse warehouseB = new Warehouse("WH-002", "West Coast Fulfillment",
                                             "500 Harbor St, Los Angeles CA 90001");
        inventoryService.registerWarehouse(warehouseA);
        inventoryService.registerWarehouse(warehouseB);
        System.out.println("Registered: " + warehouseA);
        System.out.println("Registered: " + warehouseB);

        // Register products in warehouses with thresholds
        // T-shirt: reorder when qty drops to 5, reorder 20 at a time
        inventoryService.registerProductInWarehouse("PROD-001", "WH-001", 5, 20, "A-01-3");
        inventoryService.registerProductInWarehouse("PROD-002", "WH-001", 3, 10, "A-02-1");

        // ---------------------------------------------------------------
        // Step 3: Register supplier and link to products
        // ---------------------------------------------------------------
        System.out.println("\n==== STEP 3: Register Supplier ====");
        Supplier fashionSupplier = new Supplier(
                "SUP-001", "FashionCo Wholesale",
                "orders@fashionco.com", "+1-800-555-0100", 7);
        fashionSupplier.addProduct("PROD-001");
        fashionSupplier.addProduct("PROD-002");
        supplierService.registerSupplier(fashionSupplier);
        System.out.println("Registered: " + fashionSupplier);

        // ---------------------------------------------------------------
        // Step 4: Add initial stock via a purchase
        // ---------------------------------------------------------------
        System.out.println("\n==== STEP 4: Add Initial Stock ====");
        int qtyAfterPurchase = inventoryService.addStock(
                "PROD-001", "WH-001", 50,
                StockMovementType.PURCHASE, "Initial stock from PO-INIT-001");
        System.out.println("T-Shirt stock after initial purchase: " + qtyAfterPurchase);

        inventoryService.addStock("PROD-002", "WH-001", 15,
                StockMovementType.PURCHASE, "Initial hoodie stock");
        System.out.println("Hoodie stock after initial purchase: "
                + inventoryService.getStockLevel("PROD-002", "WH-001"));

        // ---------------------------------------------------------------
        // Step 5: Process sales — eventually trigger the low-stock alert
        // ---------------------------------------------------------------
        System.out.println("\n==== STEP 5: Process Sales ====");

        // Sell 30 units — still above threshold (50 - 30 = 20)
        inventoryService.removeStock("PROD-001", "WH-001", 30,
                StockMovementType.SALE, "Online order batch #1001");
        System.out.println("T-Shirt stock after selling 30: "
                + inventoryService.getStockLevel("PROD-001", "WH-001"));

        // Sell 15 more — drops to 5 which equals the threshold, triggers alert + auto-reorder
        System.out.println("\n--- Selling 15 more (will hit reorder threshold) ---");
        int remainingAfterAlert = inventoryService.removeStock(
                "PROD-001", "WH-001", 15,
                StockMovementType.SALE, "Online order batch #1002");
        System.out.println("\nT-Shirt stock after selling 15 more: " + remainingAfterAlert);

        // ---------------------------------------------------------------
        // Step 6: View the auto-created Purchase Order
        // ---------------------------------------------------------------
        System.out.println("\n==== STEP 6: View Open Purchase Orders ====");
        List<PurchaseOrder> openPOs = purchaseOrderService.getOpenPurchaseOrders();
        openPOs.forEach(po -> System.out.println("  " + po));

        // Advance the PO through its lifecycle and receive goods
        if (!openPOs.isEmpty()) {
            PurchaseOrder autoPO = openPOs.get(0);
            String poId = autoPO.getPoId();

            System.out.println("\n==== STEP 7: Advance PO Lifecycle ====");
            purchaseOrderService.sendPurchaseOrder(poId);
            System.out.println("PO status after send: " + autoPO.getStatus());

            purchaseOrderService.confirmPurchaseOrder(poId);
            System.out.println("PO status after confirm: " + autoPO.getStatus());

            // Receive the goods — this calls inventoryService.addStock internally
            int reorderQty = autoPO.getItems().get(0).getOrderedQuantity();
            purchaseOrderService.receiveGoods(poId, "PROD-001", reorderQty);
            System.out.println("T-Shirt stock after receiving PO: "
                    + inventoryService.getStockLevel("PROD-001", "WH-001"));
            System.out.println("PO status after receipt: " + autoPO.getStatus());
        }

        // ---------------------------------------------------------------
        // Step 8: Transfer stock to second warehouse
        // ---------------------------------------------------------------
        System.out.println("\n==== STEP 8: Transfer Stock to West Coast Warehouse ====");
        inventoryService.transferStock("PROD-001", "WH-001", "WH-002", 10,
                "Replenish West Coast fulfillment center");
        System.out.println("WH-001 T-Shirt stock after transfer: "
                + inventoryService.getStockLevel("PROD-001", "WH-001"));
        System.out.println("WH-002 T-Shirt stock after transfer: "
                + inventoryService.getStockLevel("PROD-001", "WH-002"));

        // ---------------------------------------------------------------
        // Step 9: Run an audit — pretend the physical count found a discrepancy
        // ---------------------------------------------------------------
        System.out.println("\n==== STEP 9: Stock Audit / Reconciliation ====");
        int systemQty = inventoryService.getStockLevel("PROD-001", "WH-001");
        System.out.println("System quantity for T-Shirt in WH-001: " + systemQty);

        // Physical count found 2 fewer units (shrinkage/damage)
        Map<String, Integer> physicalCounts = Map.of(
                "PROD-001", systemQty - 2,
                "PROD-002", inventoryService.getStockLevel("PROD-002", "WH-001"));

        AuditService.ReconciliationReport report = auditService.reconcile("WH-001", physicalCounts);
        System.out.println(report);

        // Apply the reconciliation (write LOSS movements)
        auditService.applyReconciliation(report);
        System.out.println("T-Shirt stock after reconciliation: "
                + inventoryService.getStockLevel("PROD-001", "WH-001"));

        // ---------------------------------------------------------------
        // Step 10: Reports
        // ---------------------------------------------------------------
        System.out.println("\n==== STEP 10: Stock Level Report (WH-001) ====");
        reportingService.getStockLevels("WH-001").forEach(r -> System.out.println(r));

        System.out.println("\n==== STEP 10: Low Stock Report (All Warehouses) ====");
        List<ReportingService.LowStockRecord> lowStock = reportingService.getLowStockReport();
        if (lowStock.isEmpty()) {
            System.out.println("  No items below reorder threshold.");
        } else {
            lowStock.forEach(r -> System.out.println(r));
        }

        System.out.println("\n==== STEP 10: Movement History for T-Shirt ====");
        reportingService.getMovementHistory("PROD-001", null)
                .forEach(m -> System.out.printf("  [%s] %s  qty=%d  reason='%s'%n",
                        m.getTimestamp(), m.getType(), m.getQuantity(), m.getReason()));

        System.out.println("\n==== Demo Complete ====");
    }
}
```

---

## 8. Design Patterns Used

### 8.1 Observer Pattern

**Where:** `InventoryAlertListener` (observer interface) + `EmailAlertListener`, `SlackAlertListener`, `AutoReorderListener` (concrete observers) + `InventoryService` (subject/publisher).

**Why:**

The requirement is to notify multiple heterogeneous systems (email, Slack, auto-reorder) when a business event (low stock) occurs. The core `InventoryService` should not know about email clients, Slack webhooks, or PO creation logic — that would violate the Single Responsibility Principle and make it hard to add a new channel later.

Observer decouples the stock mutation logic from notification side effects. Adding PagerDuty alerts in the future means writing one new class that implements `InventoryAlertListener` and calling `addAlertListener()` — zero changes to `InventoryService`.

**Interaction sequence:**

```
InventoryService.removeStock()
   -> item.removeQuantity()
   -> checkAndFireAlert()
      -> LowStockEvent created
      -> for each InventoryAlertListener:
            EmailAlertListener.onLowStock(event)   // sends email
            SlackAlertListener.onLowStock(event)   // posts to Slack
            AutoReorderListener.onLowStock(event)  // creates PO via PurchaseOrderService
```

**Key implementation detail:** Listeners are stored in `CopyOnWriteArrayList`. This means `addAlertListener` / `removeAlertListener` can be called concurrently without blocking the notification loop. Each listener's exception is caught individually so one bad listener cannot silence the others.

---

### 8.2 Strategy Pattern

**Where:** `ReorderStrategy` (strategy interface) + `FixedQuantityReorderStrategy`, `EOQReorderStrategy` (concrete strategies), referenced by `Product`.

**Why:**

Different products need different restocking logic. A low-margin commodity benefits from bulk ordering (EOQ). A high-value, slow-moving item should have a fixed, conservative quantity. If reorder quantity were hardcoded in `InventoryService`, every new formula would require modifying a core service class — an Open/Closed Principle violation.

The strategy lives on the `Product` itself (not on the service), which is the right owner: it is the product's nature that determines the optimal order size, not the warehouse or the service.

**The EOQ formula:**

```
EOQ = sqrt( (2 * D * S) / H )

D = annual demand (units/year)
S = fixed cost per order placed ($)
H = annual holding cost per unit ($ per unit per year = unit_price * 20%)
```

At the EOQ, total ordering cost equals total holding cost, minimising combined inventory cost.

---

## 9. Key Design Decisions and Trade-offs

### Decision 1: Strategy on Product vs. on InventoryItem

**Chosen:** `ReorderStrategy` is a field of `Product`.

**Alternative:** `ReorderStrategy` could live on `InventoryItem` (per product-warehouse combination).

**Trade-off:** Putting it on `Product` means the same algorithm is used for all warehouses that stock the product. This is simpler and covers 90% of real-world cases (the product's economics do not change by location). If per-warehouse strategies are needed in the future, promote `reorderStrategy` from `Product` to `InventoryItem`.

---

### Decision 2: Synchronous alert dispatch

**Chosen:** `checkAndFireAlert` is called inline, blocking the `removeStock` call until all listeners complete.

**Alternative:** Dispatch to an `ExecutorService` / message queue asynchronously.

**Trade-off:** Synchronous is simpler to reason about and guarantees the alert is delivered before the caller proceeds. The downside is that a slow email server or Slack network call adds latency to `removeStock`. For a web application, async dispatch (e.g., via a bounded `BlockingQueue`) should be wired up, but the Observer interface contract does not change.

---

### Decision 3: Atomic transfer using synchronized removeQuantity

**Chosen:** `removeQuantity` on `InventoryItem` is `synchronized`. In `transferStock`, the source is decremented first; if it fails, no movement is recorded and the destination is never touched.

**Alternative:** Use a two-phase commit or a distributed lock.

**Trade-off:** For a single-JVM system, synchronized methods are sufficient and avoid complex failure modes. The "remove first" ordering ensures that overselling is impossible even if the JVM crashes between the two mutations (the source is protected). For multi-node deployment, this would need to be replaced with a database transaction or a distributed lock manager.

---

### Decision 4: Auto-register product in destination warehouse during transfer

**Chosen:** If the destination warehouse doesn't have the product registered, `transferStock` creates a zero-threshold `InventoryItem` automatically.

**Alternative:** Require explicit pre-registration and throw if missing.

**Trade-off:** Auto-registration is more ergonomic for operators but may create InventoryItems with unset thresholds (threshold=0 means alerts fire immediately after the transfer). The design documents this as an expected behavior that operators should correct after the first transfer.

---

### Decision 5: PO receipt triggers stock addition via callback

**Chosen:** `PurchaseOrderService.receiveGoods()` calls `InventoryService.addStock()` directly.

**Alternative:** The PO service emits an event and a separate handler calls addStock.

**Trade-off:** Direct call is simpler, but creates a circular-ish dependency (`InventoryService` uses `PurchaseOrderService` via `AutoReorderListener`, and `PurchaseOrderService` uses `InventoryService`). This is broken by passing `InventoryService` into `PurchaseOrderService`'s constructor (dependency injection), not the reverse. In a larger system, an event bus (or a domain event) would decouple the two services further.

---

### Decision 6: AtomicInteger for quantity in InventoryItem

**Chosen:** `AtomicInteger` for the backing field; `removeQuantity` is `synchronized` for the compare-then-decrement.

**Alternative:** Make the whole `InventoryItem` immutable and replace it atomically via `AtomicReference`.

**Trade-off:** `AtomicInteger.addAndGet` handles `addQuantity` lock-free. However, `removeQuantity` must be synchronized because it involves a non-atomic check-then-act (compare quantity, then decrement). An immutable-replace approach would work but requires every reader to hold a reference copy, which increases GC pressure and cognitive overhead.

---

## 10. Extension Points

### 10.1 Add a new alert channel (e.g., PagerDuty)

1. Create `PagerDutyAlertListener implements InventoryAlertListener`.
2. Implement `onLowStock(LowStockEvent event)` to call the PagerDuty API.
3. Register: `inventoryService.addAlertListener(new PagerDutyAlertListener(apiKey))`.

No changes to any existing class.

---

### 10.2 Add a new reorder strategy (e.g., seasonal demand)

1. Create `SeasonalReorderStrategy implements ReorderStrategy`.
2. Implement `calculateReorderQuantity(...)` using seasonal factors.
3. Assign: `product.setReorderStrategy(new SeasonalReorderStrategy(month -> factor[month]))`.

No changes to any existing class.

---

### 10.3 Add persistence

Replace the in-memory `ConcurrentHashMap` collections in each service with repository interfaces:

```java
interface ProductRepository {
    void save(Product p);
    Optional<Product> findById(String id);
    // ...
}
```

Services accept the repository interface. Provide a `JpaProductRepository` or `MongoProductRepository` as the concrete implementation. The service layer is unchanged.

---

### 10.4 Add REST API

Introduce a controller layer:

```java
@RestController
public class InventoryController {
    private final InventoryService inventoryService;
    // POST /stock/add, POST /stock/remove, POST /stock/transfer, ...
}
```

The service layer is unchanged.

---

### 10.5 Add per-warehouse reorder strategies

Promote `ReorderStrategy` from `Product` to `InventoryItem`:

```java
// In InventoryItem:
private ReorderStrategy reorderStrategy;
```

The `checkAndFireAlert` method in `InventoryService` already resolves strategy via `product.getReorderStrategy()` — change that one line to `item.getReorderStrategy()`. All other logic is unchanged.

---

### 10.6 Add CQRS / event sourcing

Replace `StockMovement` as a side-effect log with a primary event store. Reconstruct `InventoryItem.quantity` by replaying events. This fits naturally because `StockMovement` is already immutable and carries all information needed to rebuild state.

---

### 10.7 Add batch operations

Add `addStockBatch(List<StockAddRequest> requests)` to `InventoryService` that processes multiple items in a single transaction boundary (when persistence is added). The Observer notification is deferred until all mutations succeed.

---

## 11. Interview Discussion Points

### "Why does the Strategy live on Product rather than in InventoryService?"

The strategy determines how many units of *this product* to order, which is a property of the product's economics (price, demand, storage cost). `InventoryService` is about warehouse operations, not product economics. If the strategy were in `InventoryService`, you'd need a map of `productId -> strategy` inside a class that already has many responsibilities — a smell. Single Responsibility says the product owns its restocking rules.

---

### "What happens if two threads simultaneously try to sell the last unit?"

`removeQuantity` is `synchronized` on the `InventoryItem` instance. Thread B will block while Thread A completes its check-then-decrement. If Thread A takes the last unit, Thread B's check will find quantity 0 and throw `InsufficientStockException`. No negative stock can result. For a distributed system across JVMs, you'd use a database row-level lock or optimistic concurrency (compare-and-swap on a version field).

---

### "How would you handle partial PO delivery across multiple shipments?"

The current model handles this with `PurchaseOrderStatus.PARTIALLY_RECEIVED`. Call `purchaseOrderService.receiveGoods(poId, productId, partialQty)` once per shipment. The PO transitions to `PARTIALLY_RECEIVED` and stays open until all line items are fully received, at which point it auto-transitions to `RECEIVED`. You can call `receiveGoods` as many times as needed until the ordered quantity is exhausted.

---

### "How would you scale this to multiple data centers?"

The current design is single-JVM. To scale:
1. Extract each service behind an HTTP or gRPC API.
2. Replace in-memory maps with a distributed database (Postgres with row-level locking, or a document store like MongoDB).
3. Replace the synchronous Observer with an event bus (Kafka or SQS). `InventoryService` publishes a `LowStockEvent` message; consumer microservices (email, Slack, auto-reorder) subscribe.
4. Use idempotency keys on `StockMovement` to prevent double-counting if a message is redelivered.

---

### "How do you prevent double purchase orders from the AutoReorderListener?"

`PurchaseOrderService.hasOpenPurchaseOrder(productId, warehouseId)` checks for any non-terminal PO before creating a new one. `AutoReorderListener.onLowStock` calls this guard before creating a PO. In a concurrent environment, this check could race (two threads both see no open PO and both create one). The fix is to make the check-and-create atomic via a database unique constraint on `(productId, warehouseId, status NOT IN (RECEIVED, CANCELLED))` — at most one open PO per product per warehouse.

---

### "What is the time complexity of the key operations?"

| Operation | Complexity | Note |
|---|---|---|
| `addStock` / `removeStock` | O(1) | HashMap lookup by productId in Warehouse |
| `transferStock` | O(1) | Two warehouse lookups + two item lookups |
| `getStockLevel` | O(1) | Direct HashMap access |
| `getLowStockReport` | O(W * P) | W warehouses, P products each |
| `getMovementsForProduct` | O(M) | M = total movement records |
| `reconcile` | O(P) | P = products in warehouse |

---

### "What would you change if quantities were fractional (sold by weight)?"

Replace `int` with `BigDecimal` throughout `InventoryItem`. Change `AtomicInteger` to a plain `BigDecimal` field with a `synchronized` block for both add and remove (since there is no `AtomicBigDecimal` in the JDK). Update `InsufficientStockException` message formatting accordingly.

---

## 12. Common Mistakes in This Design

### Mistake 1: Mutating quantity without guarding against negative values

**Wrong approach:**
```java
// No check — allows negative stock
item.quantity -= requested;
```

**Why it's wrong:** A sale or transfer can bring quantity below zero if two threads race or if the caller passes a large number. The system loses integrity silently.

**Correct approach:** Synchronize the check-then-decrement:
```java
public synchronized int removeQuantity(int amount) throws InsufficientStockException {
    if (quantity.get() < amount) throw new InsufficientStockException(...);
    return quantity.addAndGet(-amount);
}
```

---

### Mistake 2: Putting alert logic directly inside InventoryService

**Wrong approach:**
```java
// Hard-coded in removeStock
if (item.isBelowReorderThreshold()) {
    emailService.sendEmail(...);
    slackService.post(...);
    purchaseOrderService.createPO(...);
}
```

**Why it's wrong:** Every new notification channel requires modifying `InventoryService`. The class becomes a God class with dependencies on email, Slack, and PO systems. Unit tests for `removeStock` must mock all three.

**Correct approach:** Observer pattern — `InventoryService` holds a list of `InventoryAlertListener` and calls `onLowStock(event)`. Each channel is its own implementation.

---

### Mistake 3: Placing ReorderStrategy as a static method or constant

**Wrong approach:**
```java
public static int calculateReorderQty(InventoryItem item) {
    return item.getReorderQuantity() * 2; // hardcoded multiplier
}
```

**Why it's wrong:** No ability to use different algorithms for different products. Changing the formula breaks all products at once.

**Correct approach:** Interface + concrete implementations, assigned per `Product`. Call `product.getReorderStrategy().calculateReorderQuantity(...)`.

---

### Mistake 4: Using HashMap instead of ConcurrentHashMap in services accessed by multiple threads

**Wrong approach:**
```java
private final Map<String, Product> products = new HashMap<>();
```

**Why it's wrong:** Concurrent reads and writes to `HashMap` can cause infinite loops (Java 7) or `ConcurrentModificationException` (Java 8+).

**Correct approach:** Use `ConcurrentHashMap` for the registry maps. Use `Collections.synchronizedList` or `CopyOnWriteArrayList` for the movements log.

---

### Mistake 5: Non-atomic transfer (remove from A, crash, never add to B)

**Wrong approach:**
```java
sourceItem.removeQuantity(qty);  // succeeds
// server crashes here — stock is lost
destItem.addQuantity(qty);       // never runs
```

**Why it's wrong:** Stock disappears from the system. This is a classic distributed systems problem (two generals / two-phase commit).

**Correct approach:** Wrap both mutations in a single database transaction. In the in-memory version, the design documents this as a known limitation and ensures the "remove first" ordering so that on a crash the source stock is correctly reduced (and can be detected on audit) rather than doubled.

---

### Mistake 6: Firing the alert inside a lock

**Wrong approach:**
```java
public synchronized int removeQuantity(int amount) throws InsufficientStockException {
    // ... mutation ...
    fireAlerts();  // calls external services while holding the lock
}
```

**Why it's wrong:** If an alert listener blocks (e.g., waiting for an SMTP response), the lock is held for seconds, blocking all other threads trying to modify any item in the warehouse.

**Correct approach:** Release the lock before firing alerts. In the design, `removeQuantity` just mutates the quantity. `InventoryService.removeStock` calls `checkAndFireAlert` *after* `removeQuantity` returns, outside the synchronized block.

---

### Mistake 7: Ignoring the case where a product has no supplier during auto-reorder

**Wrong approach:**
```java
Supplier supplier = supplierService.getPrimarySupplier(productId).get(); // throws NoSuchElementException
```

**Why it's wrong:** If no supplier is registered, the `Optional.get()` call crashes with an unhandled exception, and because alert listeners run in a loop, this stops subsequent listeners from firing.

**Correct approach:** Check `Optional.isPresent()` (or use `orElse(null)` with a null check) and log a warning rather than propagating an exception:
```java
Supplier supplier = supplierService.getPrimarySupplier(productId).orElse(null);
if (supplier == null) { System.out.println("[AutoReorder] No supplier found..."); return; }
```

---

### Mistake 8: Using == for String comparison in movement filtering

**Wrong approach:**
```java
.filter(m -> m.getProductId() == productIdFilter) // reference comparison
```

**Why it's wrong:** String literals and dynamically created Strings have different references. The filter will silently return an empty list when the strings are equal but not the same object.

**Correct approach:** Always use `.equals()`:
```java
.filter(m -> m.getProductId().equals(productIdFilter))
```

---

*End of Inventory Management System LLD Case Study*
