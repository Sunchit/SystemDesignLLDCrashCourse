# Capstone Project 5: Multi-Tenant SaaS Billing and Subscription Platform

## Project Overview

A multi-tenant SaaS billing and subscription platform is the operational backbone of any software-as-a-service business. Every tenant (a paying customer organization) shares the same application infrastructure, but experiences the platform as if it were their own isolated system. The billing engine must track what each tenant consumes, enforce plan limits, generate accurate invoices, collect payments, and handle plan changes mid-cycle with mathematically correct proration.

This system serves three audiences. Platform operators need to onboard tenants, configure plans, and monitor revenue. Tenant administrators need to understand their usage, upgrade or downgrade their subscription, and retrieve invoices. The automated billing scheduler needs to run renewals, detect payment failures, and suspend delinquent accounts without human intervention.

The complexity is significant. Plans are not static: a tenant might upgrade from FREE to PRO on day 15 of a 30-day billing cycle, requiring a credit for unused days on the old plan and a charge for remaining days on the new plan. Feature access must be enforced at every API call without hardcoding plan logic into every service method. Usage events arrive continuously and must trigger alerts when approaching limits. Every mutation to tenant state — plan changes, suspensions, invoice generation — must be written to an immutable audit log. All of this must be testable in isolation, meaning real payment gateways and real databases must be replaceable with fakes.

The design uses five Gang-of-Four patterns working together: Strategy isolates plan feature sets from tenant service logic, Decorator layers auditing and feature gating onto the tenant service without modifying it, Observer decouples usage metering alerts from the recording path, Builder constructs invoices with a readable fluent API, and Facade presents the entire platform through a single coherent entry point.

---

## Learning Objectives

1. Apply the **Strategy pattern** to make plan feature sets swappable at runtime without modifying calling code.
2. Apply the **Decorator pattern** to layer cross-cutting concerns (auditing, feature gating) onto a service without subclassing or modifying the original implementation.
3. Apply the **Observer pattern** to decouple usage metering side-effects (alerts, audit writes) from the core recording logic.
4. Apply the **Builder pattern** to construct complex Invoice objects with multiple optional line items and computed totals.
5. Apply the **Facade pattern** to present a cohesive API over a web of collaborating services.
6. Implement **proration arithmetic** with daily rates, handling mid-cycle plan upgrades and downgrades correctly.
7. Reason about **multi-tenancy isolation**: per-tenant data partitioning, feature enforcement, and audit trails.
8. Design for **testability**: every external dependency (payment gateway, clock) is injectable, enabling deterministic unit tests.

---

## Domain Model

**Tenant** is the root aggregate. It represents an organization that has signed up for the platform. A tenant has a unique ID, a human-readable name, a current status (ACTIVE, SUSPENDED, TRIAL, CANCELLED), a plan type (FREE, PRO, ENTERPRISE), a billing cycle (MONTHLY, ANNUAL), and an optional trial end date. Everything else in the system is scoped to a tenant.

**Subscription** tracks the formal billing relationship for a tenant: which plan they are on, when it started, and when the next billing date falls. A tenant has exactly one active subscription at any time.

**PlanFeatureSet** describes what a tenant on a given plan can do: which features are enabled, how many users they can have, how much storage and how many API calls per month, and what the base price is. This is a Strategy — there is one implementation per plan tier.

**UsageRecord** is an append-only record of a metered event: a tenant consumed some quantity of a metric (API calls, storage bytes, user count) at a specific timestamp. Usage records are never deleted; historical data supports billing and analytics.

**Invoice** is the document sent to a tenant for a billing period. It contains one or more **InvoiceLineItem** entries, each with a description, quantity, unit price, and computed total. The invoice carries a subtotal (sum of line items), a tax amount, and a grand total. Invoices transition from PENDING to PAID or FAILED.

**AuditLogEntry** is an immutable record of a significant platform event: who did what to which tenant and when. Every state-changing operation must produce an audit entry. Entries are never modified after creation.

**PaymentResult** is the response from a payment gateway: a payment ID, status (SUCCESS, FAILED, PENDING, REFUNDED), amount, timestamp, and an optional error message on failure.

**Money** is a value object pairing a BigDecimal amount with a currency code. Arithmetic operations (add, subtract, multiply) produce new Money instances; none mutate in place. Currency mismatch throws an exception.

**UsageLimit** pairs a UsageMetric with a maximum allowed value, or marks the limit as unlimited. It can compute percentage used and detect exceedance.

**DateRange** is a value object representing an inclusive start and exclusive end date. It supports containment checks and overlap day counting, which proration arithmetic depends on.

---

## Class Diagram

```
+------------------+          +-------------------+
|     Tenant       |1       1 |   Subscription    |
|------------------|----------|-------------------|
| tenantId: String |          | subscriptionId    |
| name: String     |          | tenantId: String  |
| status: enum     |          | planType: enum    |
| planType: enum   |          | startDate         |
| billingCycle     |          | nextBillingDate   |
| trialEndDate     |          | status: String    |
| createdAt        |          +-------------------+
+------------------+
         |1
         |
         |*
+------------------+       +----------------------+
|  UsageRecord     |       |       Invoice        |
|------------------|       |----------------------|
| tenantId: String |       | invoiceId: String    |
| metric: enum     |       | tenantId: String     |
| value: long      |       | billingPeriod        |
| recordedAt       |       | lineItems: List<>    |
+------------------+       | subtotal: Money      |
                           | tax: Money           |
                           | total: Money         |
                           | generatedAt          |
                           | status: String       |
                           +----------------------+
                                    |*
                           +----------------------+
                           |   InvoiceLineItem    |
                           |----------------------|
                           | description: String  |
                           | quantity: int        |
                           | unitPrice: Money     |
                           | total: Money         |
                           +----------------------+

+------------------+       +----------------------+
| AuditLogEntry    |       |    PaymentResult     |
|------------------|       |----------------------|
| entryId: String  |       | paymentId: String    |
| tenantId: String |       | status: enum         |
| action: enum     |       | amount: Money        |
| actor: String    |       | processedAt          |
| details: Map<>   |       | errorMessage: String |
| timestamp        |       +----------------------+
+------------------+

<<interface>>                <<interface>>
PlanFeatureSet               TenantService
getPlanType()                createTenant()
isFeatureEnabled()           getTenant()
getUsageLimit()              suspendTenant()
getBasePrice()               reactivateTenant()
getMaxUsers()                updatePlan()
getMaxStorageBytes()         getAllTenants()
getMaxApiCallsPerMonth()          |
getEnabledFeatures()              |implements
      |implements           TenantServiceImpl
FreePlanFeatureSet           FeatureGatedTenantService (decorator)
ProPlanFeatureSet            AuditedTenantService (decorator)
EnterprisePlanFeatureSet

<<interface>>
UsageEventListener
onUsageRecorded()
      |implements
UsageLimitAlertListener
UsageAuditListener

<<interface>>
PaymentGateway
charge()
refund()
      |implements
StripePaymentGateway
MockPaymentGateway

Value Objects:
  Money(amount: BigDecimal, currency: String)
  UsageLimit(metric, limit, unlimited)
  DateRange(startDate, endDate)

Facade:
  SaaSPlatform
    - auditLogService: InMemoryAuditLogService
    - tenantService: TenantService (decorated chain)
    - meteringService: UsageMeteringService
    - billingService: BillingService
    - trialManager: TrialManager
```

---

## Design Patterns Applied

### 1. Strategy — PlanFeatureSet

**Classes involved:** `PlanFeatureSet` (interface), `FreePlanFeatureSet`, `ProPlanFeatureSet`, `EnterprisePlanFeatureSet`, `PlanFeatureSetFactory`, `BillingService`, `UsageMeteringService`.

**Reason chosen:** Plan feature sets differ in every dimension — price, limits, enabled features — but the code that consumes them (billing, metering, feature checking) needs to treat all plans uniformly. Strategy defines a stable interface over variable behavior, satisfying the Open/Closed Principle: adding an ULTRA_ENTERPRISE plan requires writing one new class and registering it in the factory, with zero changes to billing or metering logic.

**Impact of removal:** Without Strategy, `BillingService.generateInvoice` would contain a switch statement over `PlanType`. Adding a new plan means editing the switch. Each new conditional branch increases the risk of breaking existing paths, and the billing logic becomes tightly coupled to plan definitions.

### 2. Decorator — TenantService

**Classes involved:** `TenantService` (interface), `TenantServiceImpl`, `AuditedTenantService`, `FeatureGatedTenantService`.

**Reason chosen:** Auditing and feature gating are cross-cutting concerns that should not live inside the core service. Decorator wraps the core service in layers, each adding one responsibility. The layers compose in any order and are independently testable. A test for audit logging passes a mock delegate and a real `AuditLogService`; it never touches `TenantServiceImpl`.

**Impact of removal:** Without Decorator, `TenantServiceImpl` would contain audit writes and feature gate checks inline. Every new cross-cutting requirement (rate limiting, caching, metrics) adds more code to the same class. The class violates Single Responsibility and becomes impossible to test individual concerns in isolation.

### 3. Observer — UsageMeteringService

**Classes involved:** `UsageEventListener` (interface), `UsageMeteringService`, `UsageLimitAlertListener`, `UsageAuditListener`.

**Reason chosen:** When usage is recorded, the metering service should not know what happens next. Today the platform sends an alert and writes an audit entry. Tomorrow it might push a webhook or trigger an automatic upgrade prompt. Observer lets new listeners be added without modifying `UsageMeteringService`.

**Impact of removal:** Without Observer, `UsageMeteringService.recordUsage` would call alert and audit logic directly. Adding a new reaction to usage events means editing the metering service — violating Open/Closed — and the class grows a dependency on every downstream system.

### 4. Builder — InvoiceBuilder

**Classes involved:** `InvoiceBuilder`, `Invoice`, `InvoiceLineItem`.

**Reason chosen:** An Invoice has several optional components: a base charge, zero or more overage charges, an optional proration adjustment, and a configurable tax rate. Constructing it with a single constructor would require either telescoping constructors or a constructor with many nullable parameters. Builder makes the construction readable and validates completeness before object creation.

**Impact of removal:** Without Builder, `BillingService` would build Invoice objects by directly constructing and mutating them, spreading invoice assembly logic across multiple methods and making it easy to forget to compute totals or leave required fields unset.

### 5. Facade — SaaSPlatform

**Classes involved:** `SaaSPlatform`, all service classes.

**Reason chosen:** The platform has many collaborating services: tenant management, metering, billing, audit logging, trial management, payment processing. Client code (demos, controllers, integration tests) should not need to know which service to call for each operation. Facade wires up the dependency graph in one place and exposes a simple, task-oriented API.

**Impact of removal:** Without Facade, every caller would need to construct and wire all services, know the correct call order (e.g., update tenant plan before generating proration invoice), and duplicate that wiring across tests and controllers.

---

## Complete Java Implementation

### Enums

```java
public enum PlanType {
    FREE, PRO, ENTERPRISE
}

public enum TenantStatus {
    ACTIVE, SUSPENDED, TRIAL, CANCELLED
}

public enum BillingCycle {
    MONTHLY, ANNUAL
}

public enum FeatureKey {
    SSO, ADVANCED_ANALYTICS, CUSTOM_DOMAIN, API_ACCESS,
    PRIORITY_SUPPORT, AUDIT_LOGS, CUSTOM_ROLES, WEBHOOKS
}

public enum UsageMetric {
    API_CALLS, STORAGE_BYTES, USER_COUNT
}

public enum PaymentStatus {
    SUCCESS, FAILED, PENDING, REFUNDED
}

public enum AuditAction {
    TENANT_CREATED, PLAN_UPGRADED, PLAN_DOWNGRADED,
    INVOICE_GENERATED, PAYMENT_PROCESSED, FEATURE_ACCESSED,
    TENANT_SUSPENDED
}
```

### Value Objects

```java
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Objects;

public final class Money implements Comparable<Money> {
    private final BigDecimal amount;
    private final String currency;

    private Money(BigDecimal amount, String currency) {
        Objects.requireNonNull(amount, "amount must not be null");
        Objects.requireNonNull(currency, "currency must not be null");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            // Allow negative for credit notes / proration credits
            // Validation: just ensure it's not NaN-equivalent
        }
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    public static Money of(BigDecimal amount, String currency) {
        return new Money(amount, currency);
    }

    public static Money of(String amount, String currency) {
        return new Money(new BigDecimal(amount), currency);
    }

    public static Money zero(String currency) {
        return new Money(BigDecimal.ZERO, currency);
    }

    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        requireSameCurrency(other);
        return new Money(this.amount.subtract(other.amount), this.currency);
    }

    public Money multiply(double factor) {
        BigDecimal result = this.amount.multiply(BigDecimal.valueOf(factor));
        return new Money(result, this.currency);
    }

    public Money multiply(long factor) {
        BigDecimal result = this.amount.multiply(BigDecimal.valueOf(factor));
        return new Money(result, this.currency);
    }

    public boolean isZero() {
        return this.amount.compareTo(BigDecimal.ZERO) == 0;
    }

    public boolean isNegative() {
        return this.amount.compareTo(BigDecimal.ZERO) < 0;
    }

    public BigDecimal getAmount() {
        return amount;
    }

    public String getCurrency() {
        return currency;
    }

    private void requireSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Cannot operate on different currencies: " + this.currency + " vs " + other.currency);
        }
    }

    @Override
    public int compareTo(Money other) {
        requireSameCurrency(other);
        return this.amount.compareTo(other.amount);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money money = (Money) o;
        return amount.compareTo(money.amount) == 0 && currency.equals(money.currency);
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

```java
import java.util.Objects;

public final class UsageLimit {
    private final UsageMetric metric;
    private final long limit;
    private final boolean unlimited;

    private UsageLimit(UsageMetric metric, long limit, boolean unlimited) {
        this.metric = Objects.requireNonNull(metric, "metric must not be null");
        this.limit = limit;
        this.unlimited = unlimited;
    }

    public static UsageLimit unlimited(UsageMetric metric) {
        return new UsageLimit(metric, Long.MAX_VALUE, true);
    }

    public static UsageLimit of(UsageMetric metric, long limit) {
        if (limit < 0) {
            throw new IllegalArgumentException("Limit cannot be negative: " + limit);
        }
        return new UsageLimit(metric, limit, false);
    }

    public UsageMetric getMetric() {
        return metric;
    }

    public long getLimit() {
        return limit;
    }

    public boolean isUnlimited() {
        return unlimited;
    }

    public boolean isExceeded(long currentUsage) {
        if (unlimited) return false;
        return currentUsage > limit;
    }

    public double getPercentageUsed(long currentUsage) {
        if (unlimited) return 0.0;
        if (limit == 0) return currentUsage > 0 ? 100.0 : 0.0;
        return (double) currentUsage / limit * 100.0;
    }

    @Override
    public String toString() {
        if (unlimited) {
            return "UsageLimit{metric=" + metric + ", limit=UNLIMITED}";
        }
        return "UsageLimit{metric=" + metric + ", limit=" + limit + "}";
    }
}
```

```java
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.util.Objects;

public final class DateRange {
    private final LocalDate startDate;
    private final LocalDate endDate;

    public DateRange(LocalDate startDate, LocalDate endDate) {
        Objects.requireNonNull(startDate, "startDate must not be null");
        Objects.requireNonNull(endDate, "endDate must not be null");
        if (startDate.isAfter(endDate)) {
            throw new IllegalArgumentException(
                "startDate " + startDate + " cannot be after endDate " + endDate);
        }
        this.startDate = startDate;
        this.endDate = endDate;
    }

    public LocalDate getStartDate() {
        return startDate;
    }

    public LocalDate getEndDate() {
        return endDate;
    }

    public boolean contains(LocalDate date) {
        Objects.requireNonNull(date, "date must not be null");
        return !date.isBefore(startDate) && !date.isAfter(endDate);
    }

    public long overlapDays(DateRange other) {
        Objects.requireNonNull(other, "other must not be null");
        LocalDate overlapStart = this.startDate.isAfter(other.startDate) ? this.startDate : other.startDate;
        LocalDate overlapEnd = this.endDate.isBefore(other.endDate) ? this.endDate : other.endDate;
        if (overlapStart.isAfter(overlapEnd)) return 0;
        return ChronoUnit.DAYS.between(overlapStart, overlapEnd);
    }

    public long totalDays() {
        return ChronoUnit.DAYS.between(startDate, endDate);
    }

    @Override
    public String toString() {
        return "DateRange{" + startDate + " to " + endDate + ", days=" + totalDays() + "}";
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof DateRange)) return false;
        DateRange other = (DateRange) o;
        return startDate.equals(other.startDate) && endDate.equals(other.endDate);
    }

    @Override
    public int hashCode() {
        return Objects.hash(startDate, endDate);
    }
}
```

### Domain Entities

```java
import java.time.LocalDate;
import java.time.LocalDateTime;

public class Tenant {
    private final String tenantId;
    private final String name;
    private TenantStatus status;
    private PlanType planType;
    private final BillingCycle billingCycle;
    private LocalDate trialEndDate;
    private final LocalDateTime createdAt;

    public Tenant(String tenantId, String name, TenantStatus status, PlanType planType,
                  BillingCycle billingCycle, LocalDate trialEndDate, LocalDateTime createdAt) {
        this.tenantId = tenantId;
        this.name = name;
        this.status = status;
        this.planType = planType;
        this.billingCycle = billingCycle;
        this.trialEndDate = trialEndDate;
        this.createdAt = createdAt;
    }

    public String getTenantId() { return tenantId; }
    public String getName() { return name; }
    public TenantStatus getStatus() { return status; }
    public PlanType getPlanType() { return planType; }
    public BillingCycle getBillingCycle() { return billingCycle; }
    public LocalDate getTrialEndDate() { return trialEndDate; }
    public LocalDateTime getCreatedAt() { return createdAt; }

    public void setStatus(TenantStatus status) { this.status = status; }
    public void setPlanType(PlanType planType) { this.planType = planType; }
    public void setTrialEndDate(LocalDate trialEndDate) { this.trialEndDate = trialEndDate; }

    public boolean isInTrial() {
        return status == TenantStatus.TRIAL
            && trialEndDate != null
            && !LocalDate.now().isAfter(trialEndDate);
    }

    public boolean isActive() {
        return status == TenantStatus.ACTIVE || status == TenantStatus.TRIAL;
    }

    @Override
    public String toString() {
        return "Tenant{id=" + tenantId + ", name=" + name + ", status=" + status
            + ", plan=" + planType + ", cycle=" + billingCycle + "}";
    }
}
```

```java
import java.time.LocalDate;

public class Subscription {
    private final String subscriptionId;
    private final String tenantId;
    private PlanType planType;
    private final LocalDate startDate;
    private LocalDate nextBillingDate;
    private String status;

    public Subscription(String subscriptionId, String tenantId, PlanType planType,
                        LocalDate startDate, LocalDate nextBillingDate, String status) {
        this.subscriptionId = subscriptionId;
        this.tenantId = tenantId;
        this.planType = planType;
        this.startDate = startDate;
        this.nextBillingDate = nextBillingDate;
        this.status = status;
    }

    public String getSubscriptionId() { return subscriptionId; }
    public String getTenantId() { return tenantId; }
    public PlanType getPlanType() { return planType; }
    public LocalDate getStartDate() { return startDate; }
    public LocalDate getNextBillingDate() { return nextBillingDate; }
    public String getStatus() { return status; }

    public void setPlanType(PlanType planType) { this.planType = planType; }
    public void setNextBillingDate(LocalDate nextBillingDate) { this.nextBillingDate = nextBillingDate; }
    public void setStatus(String status) { this.status = status; }

    @Override
    public String toString() {
        return "Subscription{id=" + subscriptionId + ", tenantId=" + tenantId
            + ", plan=" + planType + ", nextBilling=" + nextBillingDate + ", status=" + status + "}";
    }
}
```

```java
import java.time.LocalDateTime;

public class UsageRecord {
    private final String tenantId;
    private final UsageMetric metric;
    private final long value;
    private final LocalDateTime recordedAt;

    public UsageRecord(String tenantId, UsageMetric metric, long value, LocalDateTime recordedAt) {
        this.tenantId = tenantId;
        this.metric = metric;
        this.value = value;
        this.recordedAt = recordedAt;
    }

    public String getTenantId() { return tenantId; }
    public UsageMetric getMetric() { return metric; }
    public long getValue() { return value; }
    public LocalDateTime getRecordedAt() { return recordedAt; }

    @Override
    public String toString() {
        return "UsageRecord{tenantId=" + tenantId + ", metric=" + metric
            + ", value=" + value + ", recordedAt=" + recordedAt + "}";
    }
}
```

```java
public class InvoiceLineItem {
    private final String description;
    private final int quantity;
    private final Money unitPrice;
    private final Money total;

    public InvoiceLineItem(String description, int quantity, Money unitPrice) {
        this.description = description;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
        this.total = unitPrice.multiply((long) quantity);
    }

    public String getDescription() { return description; }
    public int getQuantity() { return quantity; }
    public Money getUnitPrice() { return unitPrice; }
    public Money getTotal() { return total; }

    @Override
    public String toString() {
        return "  " + description + " (qty=" + quantity + " x " + unitPrice + ") = " + total;
    }
}
```

```java
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class Invoice {
    private final String invoiceId;
    private final String tenantId;
    private final DateRange billingPeriod;
    private final List<InvoiceLineItem> lineItems;
    private Money subtotal;
    private Money tax;
    private Money total;
    private final LocalDateTime generatedAt;
    private String status;

    public Invoice(String invoiceId, String tenantId, DateRange billingPeriod,
                   List<InvoiceLineItem> lineItems, Money subtotal, Money tax,
                   Money total, LocalDateTime generatedAt, String status) {
        this.invoiceId = invoiceId;
        this.tenantId = tenantId;
        this.billingPeriod = billingPeriod;
        this.lineItems = new ArrayList<>(lineItems);
        this.subtotal = subtotal;
        this.tax = tax;
        this.total = total;
        this.generatedAt = generatedAt;
        this.status = status;
    }

    public String getInvoiceId() { return invoiceId; }
    public String getTenantId() { return tenantId; }
    public DateRange getBillingPeriod() { return billingPeriod; }
    public List<InvoiceLineItem> getLineItems() { return Collections.unmodifiableList(lineItems); }
    public Money getSubtotal() { return subtotal; }
    public Money getTax() { return tax; }
    public Money getTotal() { return total; }
    public LocalDateTime getGeneratedAt() { return generatedAt; }
    public String getStatus() { return status; }

    public void setStatus(String status) { this.status = status; }

    public void addLineItem(InvoiceLineItem item) {
        lineItems.add(item);
        subtotal = subtotal.add(item.getTotal());
        Money newTax = subtotal.multiply(tax.isZero() ? 0.0 : tax.getAmount().doubleValue() / subtotal.subtract(item.getTotal()).getAmount().doubleValue());
        // Recompute tax properly: keep tax rate implied from existing ratio or just re-add
        // For simplicity, we recompute total as subtotal + existing tax
        total = subtotal.add(tax);
    }

    public void recomputeWithTaxRate(double taxRate) {
        this.tax = subtotal.multiply(taxRate);
        this.total = subtotal.add(tax);
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("Invoice{id=").append(invoiceId)
          .append(", tenant=").append(tenantId)
          .append(", period=").append(billingPeriod)
          .append(", status=").append(status).append("}\n");
        for (InvoiceLineItem item : lineItems) {
            sb.append(item.toString()).append("\n");
        }
        sb.append("  Subtotal: ").append(subtotal).append("\n");
        sb.append("  Tax:      ").append(tax).append("\n");
        sb.append("  TOTAL:    ").append(total);
        return sb.toString();
    }
}
```

```java
import java.time.LocalDateTime;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

public class AuditLogEntry {
    private final String entryId;
    private final String tenantId;
    private final AuditAction action;
    private final String actor;
    private final Map<String, String> details;
    private final LocalDateTime timestamp;

    public AuditLogEntry(String entryId, String tenantId, AuditAction action,
                         String actor, Map<String, String> details, LocalDateTime timestamp) {
        this.entryId = entryId;
        this.tenantId = tenantId;
        this.action = action;
        this.actor = actor;
        this.details = Collections.unmodifiableMap(new HashMap<>(details));
        this.timestamp = timestamp;
    }

    public String getEntryId() { return entryId; }
    public String getTenantId() { return tenantId; }
    public AuditAction getAction() { return action; }
    public String getActor() { return actor; }
    public Map<String, String> getDetails() { return details; }
    public LocalDateTime getTimestamp() { return timestamp; }

    @Override
    public String toString() {
        return "AuditLogEntry{id=" + entryId + ", tenant=" + tenantId
            + ", action=" + action + ", actor=" + actor
            + ", details=" + details + ", timestamp=" + timestamp + "}";
    }
}
```

```java
import java.time.LocalDateTime;

public class PaymentResult {
    private final String paymentId;
    private final PaymentStatus status;
    private final Money amount;
    private final LocalDateTime processedAt;
    private final String errorMessage;

    private PaymentResult(String paymentId, PaymentStatus status, Money amount,
                          LocalDateTime processedAt, String errorMessage) {
        this.paymentId = paymentId;
        this.status = status;
        this.amount = amount;
        this.processedAt = processedAt;
        this.errorMessage = errorMessage;
    }

    public static PaymentResult success(String paymentId, Money amount) {
        return new PaymentResult(paymentId, PaymentStatus.SUCCESS, amount,
            LocalDateTime.now(), null);
    }

    public static PaymentResult failure(String errorMessage, Money amount) {
        return new PaymentResult(null, PaymentStatus.FAILED, amount,
            LocalDateTime.now(), errorMessage);
    }

    public String getPaymentId() { return paymentId; }
    public PaymentStatus getStatus() { return status; }
    public Money getAmount() { return amount; }
    public LocalDateTime getProcessedAt() { return processedAt; }
    public String getErrorMessage() { return errorMessage; }

    public boolean isSuccessful() {
        return status == PaymentStatus.SUCCESS;
    }

    @Override
    public String toString() {
        if (isSuccessful()) {
            return "PaymentResult{status=SUCCESS, paymentId=" + paymentId
                + ", amount=" + amount + ", processedAt=" + processedAt + "}";
        }
        return "PaymentResult{status=FAILED, amount=" + amount
            + ", error=" + errorMessage + ", processedAt=" + processedAt + "}";
    }
}
```

### Strategy Pattern — Plan Feature Sets

```java
import java.util.List;

public interface PlanFeatureSet {
    PlanType getPlanType();
    boolean isFeatureEnabled(FeatureKey feature);
    UsageLimit getUsageLimit(UsageMetric metric);
    Money getBasePrice(BillingCycle cycle);
    int getMaxUsers();
    long getMaxStorageBytes();
    long getMaxApiCallsPerMonth();
    List<FeatureKey> getEnabledFeatures();
}
```

```java
import java.math.BigDecimal;
import java.util.Arrays;
import java.util.List;

public class FreePlanFeatureSet implements PlanFeatureSet {
    private static final List<FeatureKey> ENABLED_FEATURES = Arrays.asList(FeatureKey.API_ACCESS);
    private static final int MAX_USERS = 5;
    private static final long MAX_STORAGE_BYTES = 1L * 1024 * 1024 * 1024; // 1 GB
    private static final long MAX_API_CALLS = 1_000L;

    @Override
    public PlanType getPlanType() {
        return PlanType.FREE;
    }

    @Override
    public boolean isFeatureEnabled(FeatureKey feature) {
        return ENABLED_FEATURES.contains(feature);
    }

    @Override
    public UsageLimit getUsageLimit(UsageMetric metric) {
        switch (metric) {
            case API_CALLS:    return UsageLimit.of(UsageMetric.API_CALLS, MAX_API_CALLS);
            case STORAGE_BYTES: return UsageLimit.of(UsageMetric.STORAGE_BYTES, MAX_STORAGE_BYTES);
            case USER_COUNT:   return UsageLimit.of(UsageMetric.USER_COUNT, MAX_USERS);
            default: throw new IllegalArgumentException("Unknown metric: " + metric);
        }
    }

    @Override
    public Money getBasePrice(BillingCycle cycle) {
        return Money.zero("USD");
    }

    @Override
    public int getMaxUsers() {
        return MAX_USERS;
    }

    @Override
    public long getMaxStorageBytes() {
        return MAX_STORAGE_BYTES;
    }

    @Override
    public long getMaxApiCallsPerMonth() {
        return MAX_API_CALLS;
    }

    @Override
    public List<FeatureKey> getEnabledFeatures() {
        return ENABLED_FEATURES;
    }
}
```

```java
import java.math.BigDecimal;
import java.util.Arrays;
import java.util.List;

public class ProPlanFeatureSet implements PlanFeatureSet {
    private static final List<FeatureKey> ENABLED_FEATURES = Arrays.asList(
        FeatureKey.API_ACCESS,
        FeatureKey.ADVANCED_ANALYTICS,
        FeatureKey.CUSTOM_DOMAIN,
        FeatureKey.PRIORITY_SUPPORT,
        FeatureKey.WEBHOOKS
    );
    private static final int MAX_USERS = 50;
    private static final long MAX_STORAGE_BYTES = 50L * 1024 * 1024 * 1024; // 50 GB
    private static final long MAX_API_CALLS = 100_000L;

    @Override
    public PlanType getPlanType() {
        return PlanType.PRO;
    }

    @Override
    public boolean isFeatureEnabled(FeatureKey feature) {
        return ENABLED_FEATURES.contains(feature);
    }

    @Override
    public UsageLimit getUsageLimit(UsageMetric metric) {
        switch (metric) {
            case API_CALLS:    return UsageLimit.of(UsageMetric.API_CALLS, MAX_API_CALLS);
            case STORAGE_BYTES: return UsageLimit.of(UsageMetric.STORAGE_BYTES, MAX_STORAGE_BYTES);
            case USER_COUNT:   return UsageLimit.of(UsageMetric.USER_COUNT, MAX_USERS);
            default: throw new IllegalArgumentException("Unknown metric: " + metric);
        }
    }

    @Override
    public Money getBasePrice(BillingCycle cycle) {
        switch (cycle) {
            case MONTHLY: return Money.of(new BigDecimal("29.99"), "USD");
            case ANNUAL:  return Money.of(new BigDecimal("299.99"), "USD");
            default: throw new IllegalArgumentException("Unknown billing cycle: " + cycle);
        }
    }

    @Override
    public int getMaxUsers() {
        return MAX_USERS;
    }

    @Override
    public long getMaxStorageBytes() {
        return MAX_STORAGE_BYTES;
    }

    @Override
    public long getMaxApiCallsPerMonth() {
        return MAX_API_CALLS;
    }

    @Override
    public List<FeatureKey> getEnabledFeatures() {
        return ENABLED_FEATURES;
    }
}
```

```java
import java.math.BigDecimal;
import java.util.Arrays;
import java.util.List;

public class EnterprisePlanFeatureSet implements PlanFeatureSet {
    private static final List<FeatureKey> ENABLED_FEATURES = Arrays.asList(FeatureKey.values());
    private static final int MAX_USERS = Integer.MAX_VALUE;
    private static final long MAX_STORAGE_BYTES = Long.MAX_VALUE;
    private static final long MAX_API_CALLS = Long.MAX_VALUE;

    @Override
    public PlanType getPlanType() {
        return PlanType.ENTERPRISE;
    }

    @Override
    public boolean isFeatureEnabled(FeatureKey feature) {
        return true;
    }

    @Override
    public UsageLimit getUsageLimit(UsageMetric metric) {
        return UsageLimit.unlimited(metric);
    }

    @Override
    public Money getBasePrice(BillingCycle cycle) {
        switch (cycle) {
            case MONTHLY: return Money.of(new BigDecimal("299.99"), "USD");
            case ANNUAL:  return Money.of(new BigDecimal("2999.99"), "USD");
            default: throw new IllegalArgumentException("Unknown billing cycle: " + cycle);
        }
    }

    @Override
    public int getMaxUsers() {
        return MAX_USERS;
    }

    @Override
    public long getMaxStorageBytes() {
        return MAX_STORAGE_BYTES;
    }

    @Override
    public long getMaxApiCallsPerMonth() {
        return MAX_API_CALLS;
    }

    @Override
    public List<FeatureKey> getEnabledFeatures() {
        return ENABLED_FEATURES;
    }
}
```

```java
public class PlanFeatureSetFactory {
    private static final FreePlanFeatureSet FREE = new FreePlanFeatureSet();
    private static final ProPlanFeatureSet PRO = new ProPlanFeatureSet();
    private static final EnterprisePlanFeatureSet ENTERPRISE = new EnterprisePlanFeatureSet();

    private PlanFeatureSetFactory() {
        // Utility class — prevent instantiation
    }

    public static PlanFeatureSet getFeatureSet(PlanType planType) {
        switch (planType) {
            case FREE:       return FREE;
            case PRO:        return PRO;
            case ENTERPRISE: return ENTERPRISE;
            default: throw new IllegalArgumentException("Unknown plan type: " + planType);
        }
    }
}
```

### Observer Pattern — Usage Metering

```java
public interface UsageEventListener {
    void onUsageRecorded(String tenantId, UsageMetric metric, long newTotal, UsageLimit limit);
}
```

```java
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

public class UsageMeteringService {
    // tenantId -> metric -> list of records
    private final Map<String, Map<UsageMetric, List<UsageRecord>>> usageRecords;
    private final List<UsageEventListener> listeners;
    private final Function<String, PlanType> tenantPlanLookup;

    public UsageMeteringService(Function<String, PlanType> tenantPlanLookup) {
        this.usageRecords = new HashMap<>();
        this.listeners = new ArrayList<>();
        this.tenantPlanLookup = tenantPlanLookup;
    }

    public void subscribe(UsageEventListener listener) {
        listeners.add(listener);
    }

    public void unsubscribe(UsageEventListener listener) {
        listeners.remove(listener);
    }

    public void recordUsage(String tenantId, UsageMetric metric, long value) {
        UsageRecord record = new UsageRecord(tenantId, metric, value, LocalDateTime.now());
        usageRecords
            .computeIfAbsent(tenantId, k -> new HashMap<>())
            .computeIfAbsent(metric, k -> new ArrayList<>())
            .add(record);

        long newTotal = getCurrentMonthUsage(tenantId, metric);
        PlanType planType = tenantPlanLookup.apply(tenantId);
        PlanFeatureSet featureSet = PlanFeatureSetFactory.getFeatureSet(planType);
        UsageLimit limit = featureSet.getUsageLimit(metric);

        for (UsageEventListener listener : new ArrayList<>(listeners)) {
            listener.onUsageRecorded(tenantId, metric, newTotal, limit);
        }
    }

    public long getTotalUsage(String tenantId, UsageMetric metric, DateRange period) {
        Map<UsageMetric, List<UsageRecord>> tenantRecords = usageRecords.get(tenantId);
        if (tenantRecords == null) return 0L;
        List<UsageRecord> metricRecords = tenantRecords.get(metric);
        if (metricRecords == null) return 0L;
        return metricRecords.stream()
            .filter(r -> {
                LocalDate recordDate = r.getRecordedAt().toLocalDate();
                return period.contains(recordDate);
            })
            .mapToLong(UsageRecord::getValue)
            .sum();
    }

    public boolean isWithinLimit(String tenantId, UsageMetric metric) {
        long currentUsage = getCurrentMonthUsage(tenantId, metric);
        PlanType planType = tenantPlanLookup.apply(tenantId);
        PlanFeatureSet featureSet = PlanFeatureSetFactory.getFeatureSet(planType);
        UsageLimit limit = featureSet.getUsageLimit(metric);
        return !limit.isExceeded(currentUsage);
    }

    public long getCurrentMonthUsage(String tenantId, UsageMetric metric) {
        LocalDate now = LocalDate.now();
        LocalDate monthStart = now.withDayOfMonth(1);
        LocalDate monthEnd = now.withDayOfMonth(now.lengthOfMonth());
        DateRange currentMonth = new DateRange(monthStart, monthEnd);
        return getTotalUsage(tenantId, metric, currentMonth);
    }

    public List<UsageRecord> getAllRecords(String tenantId, UsageMetric metric) {
        Map<UsageMetric, List<UsageRecord>> tenantRecords = usageRecords.get(tenantId);
        if (tenantRecords == null) return Collections.emptyList();
        List<UsageRecord> metricRecords = tenantRecords.get(metric);
        if (metricRecords == null) return Collections.emptyList();
        return Collections.unmodifiableList(metricRecords);
    }
}
```

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class UsageLimitAlertListener implements UsageEventListener {
    private static final double ALERT_THRESHOLD_PERCENT = 80.0;
    private final List<String> alerts;

    public UsageLimitAlertListener() {
        this.alerts = new ArrayList<>();
    }

    @Override
    public void onUsageRecorded(String tenantId, UsageMetric metric, long newTotal, UsageLimit limit) {
        if (limit.isUnlimited()) return;
        double percentUsed = limit.getPercentageUsed(newTotal);
        if (percentUsed >= ALERT_THRESHOLD_PERCENT) {
            String alert = String.format(
                "[ALERT] Tenant '%s' has used %.1f%% of %s limit (%d / %d)",
                tenantId, percentUsed, metric, newTotal, limit.getLimit());
            alerts.add(alert);
            System.out.println(alert);
        }
    }

    public List<String> getAlerts() {
        return Collections.unmodifiableList(alerts);
    }

    public void clearAlerts() {
        alerts.clear();
    }
}
```

```java
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

public class UsageAuditListener implements UsageEventListener {
    private final AuditLogService auditLogService;

    public UsageAuditListener(AuditLogService auditLogService) {
        this.auditLogService = auditLogService;
    }

    @Override
    public void onUsageRecorded(String tenantId, UsageMetric metric, long newTotal, UsageLimit limit) {
        if (!limit.isUnlimited() && limit.isExceeded(newTotal)) {
            Map<String, String> details = new HashMap<>();
            details.put("metric", metric.name());
            details.put("currentUsage", String.valueOf(newTotal));
            details.put("limit", String.valueOf(limit.getLimit()));
            details.put("percentUsed", String.format("%.1f%%", limit.getPercentageUsed(newTotal)));

            AuditLogEntry entry = new AuditLogEntry(
                UUID.randomUUID().toString(),
                tenantId,
                AuditAction.FEATURE_ACCESSED,
                "system",
                details,
                LocalDateTime.now()
            );
            auditLogService.log(entry);
        }
    }
}
```

### Decorator Pattern — Tenant Service

```java
public class FeatureNotEnabledException extends RuntimeException {
    private final FeatureKey feature;
    private final String tenantId;

    public FeatureNotEnabledException(FeatureKey feature, String tenantId) {
        super("Feature " + feature + " is not enabled for tenant " + tenantId);
        this.feature = feature;
        this.tenantId = tenantId;
    }

    public FeatureKey getFeature() { return feature; }
    public String getTenantId() { return tenantId; }
}
```

```java
import java.util.List;

public interface TenantService {
    Tenant createTenant(String name, PlanType planType, BillingCycle billingCycle);
    Tenant getTenant(String tenantId);
    void suspendTenant(String tenantId, String actor);
    void reactivateTenant(String tenantId, String actor);
    void updatePlan(String tenantId, PlanType newPlan, String actor);
    List<Tenant> getAllTenants();
}
```

```java
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

public class TenantServiceImpl implements TenantService {
    private final Map<String, Tenant> tenants = new HashMap<>();

    @Override
    public Tenant createTenant(String name, PlanType planType, BillingCycle billingCycle) {
        String tenantId = UUID.randomUUID().toString();
        Tenant tenant = new Tenant(tenantId, name, TenantStatus.ACTIVE, planType,
            billingCycle, null, LocalDateTime.now());
        tenants.put(tenantId, tenant);
        return tenant;
    }

    @Override
    public Tenant getTenant(String tenantId) {
        Tenant tenant = tenants.get(tenantId);
        if (tenant == null) {
            throw new IllegalArgumentException("Tenant not found: " + tenantId);
        }
        return tenant;
    }

    @Override
    public void suspendTenant(String tenantId, String actor) {
        Tenant tenant = getTenant(tenantId);
        tenant.setStatus(TenantStatus.SUSPENDED);
    }

    @Override
    public void reactivateTenant(String tenantId, String actor) {
        Tenant tenant = getTenant(tenantId);
        tenant.setStatus(TenantStatus.ACTIVE);
    }

    @Override
    public void updatePlan(String tenantId, PlanType newPlan, String actor) {
        Tenant tenant = getTenant(tenantId);
        tenant.setPlanType(newPlan);
    }

    @Override
    public List<Tenant> getAllTenants() {
        return Collections.unmodifiableList(new ArrayList<>(tenants.values()));
    }
}
```

```java
import java.util.List;
import java.util.function.Function;

public class FeatureGatedTenantService implements TenantService {
    private final TenantService delegate;
    private final Function<String, PlanFeatureSet> featureSetLookup;

    public FeatureGatedTenantService(TenantService delegate,
                                     Function<String, PlanFeatureSet> featureSetLookup) {
        this.delegate = delegate;
        this.featureSetLookup = featureSetLookup;
    }

    @Override
    public Tenant createTenant(String name, PlanType planType, BillingCycle billingCycle) {
        return delegate.createTenant(name, planType, billingCycle);
    }

    @Override
    public Tenant getTenant(String tenantId) {
        return delegate.getTenant(tenantId);
    }

    @Override
    public void suspendTenant(String tenantId, String actor) {
        // Example gate: suspending a tenant requires the actor's plan to have AUDIT_LOGS
        // In practice this gate is on the actor's tenant, but here we check the target tenant's plan
        // to demonstrate the decorator. Real systems would check actor permissions.
        PlanFeatureSet featureSet = featureSetLookup.apply(tenantId);
        if (!featureSet.isFeatureEnabled(FeatureKey.AUDIT_LOGS)) {
            // Suspension is still allowed — but log that audit logs are not available
            System.out.println("[FEATURE-GATE] Note: tenant " + tenantId
                + " does not have AUDIT_LOGS; suspension recorded without full audit trail.");
        }
        delegate.suspendTenant(tenantId, actor);
    }

    @Override
    public void reactivateTenant(String tenantId, String actor) {
        delegate.reactivateTenant(tenantId, actor);
    }

    @Override
    public void updatePlan(String tenantId, PlanType newPlan, String actor) {
        PlanFeatureSet currentFeatureSet = featureSetLookup.apply(tenantId);
        System.out.println("[FEATURE-GATE] Plan upgrade gate checked for tenant " + tenantId
            + ": current=" + currentFeatureSet.getPlanType() + " -> new=" + newPlan);
        delegate.updatePlan(tenantId, newPlan, actor);
    }

    @Override
    public List<Tenant> getAllTenants() {
        return delegate.getAllTenants();
    }
}
```

```java
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

public class AuditedTenantService implements TenantService {
    private final TenantService delegate;
    private final AuditLogService auditLogService;

    private static final int FREE_ORDINAL = 0;
    private static final int PRO_ORDINAL = 1;
    private static final int ENTERPRISE_ORDINAL = 2;

    public AuditedTenantService(TenantService delegate, AuditLogService auditLogService) {
        this.delegate = delegate;
        this.auditLogService = auditLogService;
    }

    @Override
    public Tenant createTenant(String name, PlanType planType, BillingCycle billingCycle) {
        Tenant tenant = delegate.createTenant(name, planType, billingCycle);
        Map<String, String> details = new HashMap<>();
        details.put("name", name);
        details.put("planType", planType.name());
        details.put("billingCycle", billingCycle.name());
        auditLogService.log(new AuditLogEntry(
            UUID.randomUUID().toString(),
            tenant.getTenantId(),
            AuditAction.TENANT_CREATED,
            "system",
            details,
            LocalDateTime.now()
        ));
        return tenant;
    }

    @Override
    public Tenant getTenant(String tenantId) {
        return delegate.getTenant(tenantId);
    }

    @Override
    public void suspendTenant(String tenantId, String actor) {
        delegate.suspendTenant(tenantId, actor);
        Map<String, String> details = new HashMap<>();
        details.put("suspendedBy", actor);
        auditLogService.log(new AuditLogEntry(
            UUID.randomUUID().toString(),
            tenantId,
            AuditAction.TENANT_SUSPENDED,
            actor,
            details,
            LocalDateTime.now()
        ));
    }

    @Override
    public void reactivateTenant(String tenantId, String actor) {
        delegate.reactivateTenant(tenantId, actor);
        Map<String, String> details = new HashMap<>();
        details.put("reactivatedBy", actor);
        auditLogService.log(new AuditLogEntry(
            UUID.randomUUID().toString(),
            tenantId,
            AuditAction.TENANT_CREATED,  // Using TENANT_CREATED as closest available for reactivation
            actor,
            details,
            LocalDateTime.now()
        ));
    }

    @Override
    public void updatePlan(String tenantId, PlanType newPlan, String actor) {
        Tenant tenant = delegate.getTenant(tenantId);
        PlanType oldPlan = tenant.getPlanType();
        delegate.updatePlan(tenantId, newPlan, actor);

        AuditAction action = planOrdinal(newPlan) > planOrdinal(oldPlan)
            ? AuditAction.PLAN_UPGRADED
            : AuditAction.PLAN_DOWNGRADED;

        Map<String, String> details = new HashMap<>();
        details.put("oldPlan", oldPlan.name());
        details.put("newPlan", newPlan.name());
        details.put("changedBy", actor);
        auditLogService.log(new AuditLogEntry(
            UUID.randomUUID().toString(),
            tenantId,
            action,
            actor,
            details,
            LocalDateTime.now()
        ));
    }

    @Override
    public List<Tenant> getAllTenants() {
        return delegate.getAllTenants();
    }

    private int planOrdinal(PlanType plan) {
        switch (plan) {
            case FREE:       return 0;
            case PRO:        return 1;
            case ENTERPRISE: return 2;
            default: return -1;
        }
    }
}
```

### Payment Gateway — Strategy/Adapter

```java
public interface PaymentGateway {
    PaymentResult charge(String tenantId, Money amount, String description);
    PaymentResult refund(String paymentId, Money amount);
}
```

```java
import java.util.UUID;

public class StripePaymentGateway implements PaymentGateway {

    @Override
    public PaymentResult charge(String tenantId, Money amount, String description) {
        String paymentId = "pi_" + UUID.randomUUID().toString().replace("-", "");
        System.out.println("[STRIPE] Charging " + amount + " for tenant " + tenantId
            + ": " + description + " -> paymentId=" + paymentId);
        return PaymentResult.success(paymentId, amount);
    }

    @Override
    public PaymentResult refund(String paymentId, Money amount) {
        String refundId = "re_" + UUID.randomUUID().toString().replace("-", "");
        System.out.println("[STRIPE] Refunding " + amount + " for paymentId=" + paymentId
            + " -> refundId=" + refundId);
        return PaymentResult.success(refundId, amount);
    }
}
```

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

public class MockPaymentGateway implements PaymentGateway {
    private final boolean shouldFail;
    private final List<PaymentResult> chargeHistory;

    public MockPaymentGateway(boolean shouldFail) {
        this.shouldFail = shouldFail;
        this.chargeHistory = new ArrayList<>();
    }

    @Override
    public PaymentResult charge(String tenantId, Money amount, String description) {
        PaymentResult result;
        if (shouldFail) {
            result = PaymentResult.failure("Simulated payment failure for testing", amount);
        } else {
            String paymentId = "mock_pi_" + UUID.randomUUID().toString().replace("-", "");
            result = PaymentResult.success(paymentId, amount);
        }
        chargeHistory.add(result);
        System.out.println("[MOCK-GATEWAY] charge(" + tenantId + ", " + amount + "): " + result.getStatus());
        return result;
    }

    @Override
    public PaymentResult refund(String paymentId, Money amount) {
        PaymentResult result;
        if (shouldFail) {
            result = PaymentResult.failure("Simulated refund failure for testing", amount);
        } else {
            String refundId = "mock_re_" + UUID.randomUUID().toString().replace("-", "");
            result = PaymentResult.success(refundId, amount);
        }
        chargeHistory.add(result);
        System.out.println("[MOCK-GATEWAY] refund(" + paymentId + ", " + amount + "): " + result.getStatus());
        return result;
    }

    public List<PaymentResult> getChargeHistory() {
        return Collections.unmodifiableList(chargeHistory);
    }

    public PaymentResult getLastCharge() {
        if (chargeHistory.isEmpty()) return null;
        return chargeHistory.get(chargeHistory.size() - 1);
    }

    public void reset() {
        chargeHistory.clear();
    }
}
```

### Billing Engine

```java
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

public class ProrationCalculator {

    public Money calculateProration(PlanFeatureSet oldPlan, PlanFeatureSet newPlan,
                                    LocalDate changeDate, DateRange billingPeriod) {
        long totalDays = billingPeriod.totalDays();
        if (totalDays == 0) return Money.zero("USD");

        long daysRemaining = getDaysRemaining(changeDate, billingPeriod);
        if (daysRemaining <= 0) return Money.zero("USD");

        Money oldMonthlyPrice = oldPlan.getBasePrice(BillingCycle.MONTHLY);
        Money newMonthlyPrice = newPlan.getBasePrice(BillingCycle.MONTHLY);

        Money proratedCredit = getProratedAmount(oldMonthlyPrice, totalDays, daysRemaining);
        Money proratedCharge = getProratedAmount(newMonthlyPrice, totalDays, daysRemaining);

        // netAmount = charge for new plan - credit for old plan
        // If upgrading: net is positive (customer pays more)
        // If downgrading: net is negative (customer receives credit)
        return proratedCharge.subtract(proratedCredit);
    }

    public long getDaysRemaining(LocalDate changeDate, DateRange period) {
        if (changeDate.isAfter(period.getEndDate())) return 0;
        LocalDate effectiveStart = changeDate.isBefore(period.getStartDate())
            ? period.getStartDate()
            : changeDate;
        return ChronoUnit.DAYS.between(effectiveStart, period.getEndDate());
    }

    public Money getProratedAmount(Money fullPrice, long daysInPeriod, long daysRemaining) {
        if (daysInPeriod == 0) return Money.zero("USD");
        double ratio = (double) daysRemaining / daysInPeriod;
        return fullPrice.multiply(ratio);
    }
}
```

```java
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

public class InvoiceBuilder {
    private String tenantId;
    private DateRange billingPeriod;
    private final List<InvoiceLineItem> lineItems = new ArrayList<>();
    private double taxRate = 0.0;

    public InvoiceBuilder forTenant(String tenantId) {
        this.tenantId = tenantId;
        return this;
    }

    public InvoiceBuilder forPeriod(DateRange billingPeriod) {
        this.billingPeriod = billingPeriod;
        return this;
    }

    public InvoiceBuilder addBaseCharge(PlanFeatureSet featureSet, BillingCycle cycle) {
        Money price = featureSet.getBasePrice(cycle);
        String description = featureSet.getPlanType().name() + " Plan Base Charge ("
            + cycle.name().toLowerCase() + ")";
        lineItems.add(new InvoiceLineItem(description, 1, price));
        return this;
    }

    public InvoiceBuilder addOverageCharge(UsageMetric metric, long overage, Money ratePerUnit) {
        if (overage <= 0) return this;
        String description = metric.name() + " Overage (" + overage + " units over limit)";
        lineItems.add(new InvoiceLineItem(description, (int) Math.min(overage, Integer.MAX_VALUE), ratePerUnit));
        return this;
    }

    public InvoiceBuilder addProrationAdjustment(Money adjustment, String description) {
        // Proration adjustment can be negative (credit) — represent as 1 unit at adjustment price
        lineItems.add(new InvoiceLineItem(description, 1, adjustment));
        return this;
    }

    public InvoiceBuilder withTaxRate(double taxRate) {
        this.taxRate = taxRate;
        return this;
    }

    public Invoice build() {
        validate();
        Money subtotal = Money.zero("USD");
        for (InvoiceLineItem item : lineItems) {
            subtotal = subtotal.add(item.getTotal());
        }
        Money tax = subtotal.isNegative() ? Money.zero("USD") : subtotal.multiply(taxRate);
        Money total = subtotal.add(tax);

        return new Invoice(
            UUID.randomUUID().toString(),
            tenantId,
            billingPeriod,
            lineItems,
            subtotal,
            tax,
            total,
            LocalDateTime.now(),
            "PENDING"
        );
    }

    private void validate() {
        if (tenantId == null || tenantId.isEmpty()) {
            throw new IllegalStateException("InvoiceBuilder: tenantId is required");
        }
        if (billingPeriod == null) {
            throw new IllegalStateException("InvoiceBuilder: billingPeriod is required");
        }
    }
}
```

```java
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.HashMap;
import java.util.Map;

public class BillingService {
    private static final Money OVERAGE_API_RATE = Money.of(new BigDecimal("0.001"), "USD");
    private static final Money OVERAGE_STORAGE_RATE_PER_GB = Money.of(new BigDecimal("0.10"), "USD");
    private static final long BYTES_PER_GB = 1024L * 1024 * 1024;

    private final TenantService tenantService;
    private final UsageMeteringService meteringService;
    private final PaymentGateway paymentGateway;
    private final AuditLogService auditLogService;
    private final ProrationCalculator prorationCalculator;

    public BillingService(TenantService tenantService,
                          UsageMeteringService meteringService,
                          PaymentGateway paymentGateway,
                          AuditLogService auditLogService,
                          ProrationCalculator prorationCalculator) {
        this.tenantService = tenantService;
        this.meteringService = meteringService;
        this.paymentGateway = paymentGateway;
        this.auditLogService = auditLogService;
        this.prorationCalculator = prorationCalculator;
    }

    public Invoice generateInvoice(String tenantId, DateRange billingPeriod) {
        Tenant tenant = tenantService.getTenant(tenantId);
        PlanFeatureSet featureSet = PlanFeatureSetFactory.getFeatureSet(tenant.getPlanType());

        InvoiceBuilder builder = new InvoiceBuilder()
            .forTenant(tenantId)
            .forPeriod(billingPeriod)
            .addBaseCharge(featureSet, tenant.getBillingCycle());

        // Check API call overages
        long apiUsage = meteringService.getTotalUsage(tenantId, UsageMetric.API_CALLS, billingPeriod);
        UsageLimit apiLimit = featureSet.getUsageLimit(UsageMetric.API_CALLS);
        if (!apiLimit.isUnlimited() && apiUsage > apiLimit.getLimit()) {
            long overage = apiUsage - apiLimit.getLimit();
            builder.addOverageCharge(UsageMetric.API_CALLS, overage, OVERAGE_API_RATE);
        }

        // Check storage overages
        long storageUsage = meteringService.getTotalUsage(tenantId, UsageMetric.STORAGE_BYTES, billingPeriod);
        UsageLimit storageLimit = featureSet.getUsageLimit(UsageMetric.STORAGE_BYTES);
        if (!storageLimit.isUnlimited() && storageUsage > storageLimit.getLimit()) {
            long overageBytes = storageUsage - storageLimit.getLimit();
            long overageGB = (overageBytes + BYTES_PER_GB - 1) / BYTES_PER_GB; // ceiling division
            builder.addOverageCharge(UsageMetric.STORAGE_BYTES, overageGB, OVERAGE_STORAGE_RATE_PER_GB);
        }

        Invoice invoice = builder.build();

        Map<String, String> details = new HashMap<>();
        details.put("invoiceId", invoice.getInvoiceId());
        details.put("total", invoice.getTotal().toString());
        details.put("period", billingPeriod.toString());
        auditLogService.log(buildAuditEntry(tenantId, AuditAction.INVOICE_GENERATED, "system", details));

        return invoice;
    }

    public PaymentResult processPayment(Invoice invoice) {
        PaymentResult result = paymentGateway.charge(
            invoice.getTenantId(),
            invoice.getTotal(),
            "Invoice " + invoice.getInvoiceId() + " for period " + invoice.getBillingPeriod()
        );

        if (result.isSuccessful()) {
            invoice.setStatus("PAID");
        } else {
            invoice.setStatus("FAILED");
        }

        Map<String, String> details = new HashMap<>();
        details.put("invoiceId", invoice.getInvoiceId());
        details.put("paymentStatus", result.getStatus().name());
        details.put("amount", result.getAmount().toString());
        if (!result.isSuccessful()) {
            details.put("error", result.getErrorMessage());
        }
        auditLogService.log(buildAuditEntry(invoice.getTenantId(),
            AuditAction.PAYMENT_PROCESSED, "system", details));

        return result;
    }

    public Invoice handleUpgrade(String tenantId, PlanType newPlan, String actor) {
        Tenant tenant = tenantService.getTenant(tenantId);
        PlanType oldPlan = tenant.getPlanType();
        PlanFeatureSet oldFeatureSet = PlanFeatureSetFactory.getFeatureSet(oldPlan);
        PlanFeatureSet newFeatureSet = PlanFeatureSetFactory.getFeatureSet(newPlan);

        LocalDate today = LocalDate.now();
        LocalDate monthStart = today.withDayOfMonth(1);
        LocalDate monthEnd = today.withDayOfMonth(today.lengthOfMonth());
        DateRange billingPeriod = new DateRange(monthStart, monthEnd);

        Money proratedAmount = prorationCalculator.calculateProration(
            oldFeatureSet, newFeatureSet, today, billingPeriod);

        InvoiceBuilder builder = new InvoiceBuilder()
            .forTenant(tenantId)
            .forPeriod(billingPeriod)
            .addProrationAdjustment(proratedAmount,
                "Proration adjustment: " + oldPlan + " -> " + newPlan
                + " (effective " + today + ")");

        Invoice prorationInvoice = builder.build();

        PaymentResult paymentResult = processPayment(prorationInvoice);

        if (paymentResult.isSuccessful()) {
            tenantService.updatePlan(tenantId, newPlan, actor);
        }

        return prorationInvoice;
    }

    public void handleDowngrade(String tenantId, PlanType newPlan, String actor) {
        Tenant tenant = tenantService.getTenant(tenantId);
        PlanType oldPlan = tenant.getPlanType();
        PlanFeatureSet oldFeatureSet = PlanFeatureSetFactory.getFeatureSet(oldPlan);
        PlanFeatureSet newFeatureSet = PlanFeatureSetFactory.getFeatureSet(newPlan);

        LocalDate today = LocalDate.now();
        LocalDate monthStart = today.withDayOfMonth(1);
        LocalDate monthEnd = today.withDayOfMonth(today.lengthOfMonth());
        DateRange billingPeriod = new DateRange(monthStart, monthEnd);

        Money proratedAmount = prorationCalculator.calculateProration(
            oldFeatureSet, newFeatureSet, today, billingPeriod);

        // Downgrade credit is negative — just log as a credit note
        System.out.println("[BILLING] Downgrade credit for tenant " + tenantId
            + ": " + proratedAmount + " (credit will be applied to next invoice)");

        // Apply plan change immediately (in production this would be scheduled for next billing date)
        tenantService.updatePlan(tenantId, newPlan, actor);

        Map<String, String> details = new HashMap<>();
        details.put("oldPlan", oldPlan.name());
        details.put("newPlan", newPlan.name());
        details.put("creditAmount", proratedAmount.toString());
        auditLogService.log(buildAuditEntry(tenantId, AuditAction.PLAN_DOWNGRADED, actor, details));
    }

    public Invoice processRenewal(String tenantId) {
        Tenant tenant = tenantService.getTenant(tenantId);
        LocalDate today = LocalDate.now();
        LocalDate monthStart = today.withDayOfMonth(1);
        LocalDate monthEnd = today.withDayOfMonth(today.lengthOfMonth());
        DateRange billingPeriod = new DateRange(monthStart, monthEnd);

        Invoice invoice = generateInvoice(tenantId, billingPeriod);
        PaymentResult result = processPayment(invoice);

        if (!result.isSuccessful()) {
            System.out.println("[BILLING] Payment failed for tenant " + tenantId
                + " — suspending account. Error: " + result.getErrorMessage());
            tenantService.suspendTenant(tenantId, "billing-system");
        }

        return invoice;
    }

    private AuditLogEntry buildAuditEntry(String tenantId, AuditAction action,
                                           String actor, Map<String, String> details) {
        return new AuditLogEntry(
            java.util.UUID.randomUUID().toString(),
            tenantId,
            action,
            actor,
            details,
            java.time.LocalDateTime.now()
        );
    }
}
```

### Audit Service

```java
import java.util.List;

public interface AuditLogService {
    void log(AuditLogEntry entry);
    List<AuditLogEntry> getLogsForTenant(String tenantId);
    List<AuditLogEntry> getLogsForAction(AuditAction action);
    List<AuditLogEntry> getAllLogs();
}
```

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.stream.Collectors;

public class InMemoryAuditLogService implements AuditLogService {
    private final List<AuditLogEntry> entries = new ArrayList<>();

    @Override
    public void log(AuditLogEntry entry) {
        if (entry == null) return;
        entries.add(entry);
        System.out.println("[AUDIT] " + entry.getTimestamp()
            + " | " + entry.getAction()
            + " | tenant=" + entry.getTenantId()
            + " | actor=" + entry.getActor()
            + " | details=" + entry.getDetails());
    }

    @Override
    public List<AuditLogEntry> getLogsForTenant(String tenantId) {
        return entries.stream()
            .filter(e -> tenantId.equals(e.getTenantId()))
            .collect(Collectors.toList());
    }

    @Override
    public List<AuditLogEntry> getLogsForAction(AuditAction action) {
        return entries.stream()
            .filter(e -> action == e.getAction())
            .collect(Collectors.toList());
    }

    @Override
    public List<AuditLogEntry> getAllLogs() {
        return Collections.unmodifiableList(entries);
    }
}
```

### Trial Management

```java
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

public class TrialManager {
    private static final int TRIAL_DURATION_DAYS = 14;

    private final Map<String, LocalDate> trialEndDates;
    private final TenantService tenantService;
    private final AuditLogService auditLogService;

    public TrialManager(TenantService tenantService, AuditLogService auditLogService) {
        this.trialEndDates = new HashMap<>();
        this.tenantService = tenantService;
        this.auditLogService = auditLogService;
    }

    public void startTrial(String tenantId, PlanType planType) {
        LocalDate trialEnd = LocalDate.now().plusDays(TRIAL_DURATION_DAYS);
        trialEndDates.put(tenantId, trialEnd);

        Tenant tenant = tenantService.getTenant(tenantId);
        tenant.setStatus(TenantStatus.TRIAL);
        tenant.setTrialEndDate(trialEnd);

        Map<String, String> details = new HashMap<>();
        details.put("planType", planType.name());
        details.put("trialEndDate", trialEnd.toString());
        auditLogService.log(new AuditLogEntry(
            UUID.randomUUID().toString(),
            tenantId,
            AuditAction.TENANT_CREATED,
            "system",
            details,
            LocalDateTime.now()
        ));

        System.out.println("[TRIAL] Started trial for tenant " + tenantId
            + " on plan " + planType + " until " + trialEnd);
    }

    public boolean isInTrial(String tenantId) {
        LocalDate trialEnd = trialEndDates.get(tenantId);
        if (trialEnd == null) return false;
        return !LocalDate.now().isAfter(trialEnd);
    }

    public long daysRemaining(String tenantId) {
        if (!isInTrial(tenantId)) return 0;
        LocalDate trialEnd = trialEndDates.get(tenantId);
        long days = ChronoUnit.DAYS.between(LocalDate.now(), trialEnd);
        return Math.max(0, days);
    }

    public void expireTrial(String tenantId) {
        Tenant tenant = tenantService.getTenant(tenantId);
        if (tenant.getStatus() == TenantStatus.TRIAL) {
            // Downgrade to FREE if they haven't upgraded
            tenant.setPlanType(PlanType.FREE);
            tenant.setStatus(TenantStatus.ACTIVE);
            trialEndDates.remove(tenantId);

            Map<String, String> details = new HashMap<>();
            details.put("reason", "trial_expired");
            details.put("convertedToPlan", PlanType.FREE.name());
            auditLogService.log(new AuditLogEntry(
                UUID.randomUUID().toString(),
                tenantId,
                AuditAction.PLAN_DOWNGRADED,
                "system",
                details,
                LocalDateTime.now()
            ));

            System.out.println("[TRIAL] Trial expired for tenant " + tenantId
                + " — converted to FREE plan.");
        }
    }
}
```

### SaaS Platform Facade

```java
import java.util.List;

public class SaaSPlatform {
    private final InMemoryAuditLogService auditLogService;
    private final TenantServiceImpl coreService;
    private final TenantService tenantService; // decorated chain
    private final UsageMeteringService meteringService;
    private final BillingService billingService;
    private final TrialManager trialManager;
    private final UsageLimitAlertListener alertListener;

    public SaaSPlatform(boolean useRealGateway) {
        // Wire up audit logging
        this.auditLogService = new InMemoryAuditLogService();

        // Wire up core tenant service with decorator chain
        this.coreService = new TenantServiceImpl();
        AuditedTenantService auditedService = new AuditedTenantService(coreService, auditLogService);
        this.tenantService = new FeatureGatedTenantService(
            auditedService,
            tenantId -> {
                try {
                    Tenant t = coreService.getTenant(tenantId);
                    return PlanFeatureSetFactory.getFeatureSet(t.getPlanType());
                } catch (IllegalArgumentException e) {
                    return PlanFeatureSetFactory.getFeatureSet(PlanType.FREE);
                }
            }
        );

        // Wire up usage metering
        this.meteringService = new UsageMeteringService(
            tenantId -> {
                try {
                    return coreService.getTenant(tenantId).getPlanType();
                } catch (IllegalArgumentException e) {
                    return PlanType.FREE;
                }
            }
        );
        this.alertListener = new UsageLimitAlertListener();
        UsageAuditListener auditListener = new UsageAuditListener(auditLogService);
        meteringService.subscribe(alertListener);
        meteringService.subscribe(auditListener);

        // Wire up payment gateway
        PaymentGateway paymentGateway = useRealGateway
            ? new StripePaymentGateway()
            : new MockPaymentGateway(false);

        // Wire up billing
        ProrationCalculator prorationCalculator = new ProrationCalculator();
        this.billingService = new BillingService(
            tenantService, meteringService, paymentGateway, auditLogService, prorationCalculator);

        // Wire up trial management
        this.trialManager = new TrialManager(tenantService, auditLogService);
    }

    public Tenant onboardTenant(String name, PlanType plan, BillingCycle cycle) {
        return tenantService.createTenant(name, plan, cycle);
    }

    public void startTrial(String tenantId) {
        Tenant tenant = coreService.getTenant(tenantId);
        trialManager.startTrial(tenantId, tenant.getPlanType());
    }

    public void recordUsage(String tenantId, UsageMetric metric, long value) {
        meteringService.recordUsage(tenantId, metric, value);
    }

    public boolean checkFeatureAccess(String tenantId, FeatureKey feature) {
        Tenant tenant = coreService.getTenant(tenantId);
        PlanFeatureSet featureSet = PlanFeatureSetFactory.getFeatureSet(tenant.getPlanType());
        boolean enabled = featureSet.isFeatureEnabled(feature);

        java.util.Map<String, String> details = new java.util.HashMap<>();
        details.put("feature", feature.name());
        details.put("enabled", String.valueOf(enabled));
        auditLogService.log(new AuditLogEntry(
            java.util.UUID.randomUUID().toString(),
            tenantId,
            AuditAction.FEATURE_ACCESSED,
            "api",
            details,
            java.time.LocalDateTime.now()
        ));
        return enabled;
    }

    public Invoice upgradePlan(String tenantId, PlanType newPlan, String actor) {
        return billingService.handleUpgrade(tenantId, newPlan, actor);
    }

    public Invoice generateMonthlyInvoice(String tenantId, DateRange period) {
        return billingService.generateInvoice(tenantId, period);
    }

    public List<AuditLogEntry> getAuditLog(String tenantId) {
        return auditLogService.getLogsForTenant(tenantId);
    }

    public Tenant getTenant(String tenantId) {
        return coreService.getTenant(tenantId);
    }

    public TrialManager getTrialManager() {
        return trialManager;
    }

    public UsageLimitAlertListener getAlertListener() {
        return alertListener;
    }

    public UsageMeteringService getMeteringService() {
        return meteringService;
    }
}
```

### Demo

```java
import java.time.LocalDate;
import java.util.List;

public class SaaSPlatformDemo {

    public static void main(String[] args) {
        System.out.println("=================================================");
        System.out.println("  Multi-Tenant SaaS Platform Demo");
        System.out.println("=================================================");

        // Step 1: Create platform with mock payment gateway (success mode)
        SaaSPlatform platform = new SaaSPlatform(false);

        // Step 2: Onboard 3 tenants
        System.out.println("\n--- Onboarding Tenants ---");
        Tenant acmeCorp = platform.onboardTenant("Acme Corp", PlanType.PRO, BillingCycle.MONTHLY);
        Tenant startupXYZ = platform.onboardTenant("StartupXYZ", PlanType.FREE, BillingCycle.MONTHLY);
        Tenant globalBank = platform.onboardTenant("GlobalBank", PlanType.ENTERPRISE, BillingCycle.ANNUAL);

        System.out.println("Onboarded: " + acmeCorp);
        System.out.println("Onboarded: " + startupXYZ);
        System.out.println("Onboarded: " + globalBank);

        // Step 3: Start trial for Acme Corp
        System.out.println("\n--- Trial Management ---");
        platform.startTrial(acmeCorp.getTenantId());
        long trialDays = platform.getTrialManager().daysRemaining(acmeCorp.getTenantId());
        System.out.println("Acme Corp trial days remaining: " + trialDays);
        System.out.println("Acme Corp in trial: " + platform.getTrialManager().isInTrial(acmeCorp.getTenantId()));

        // Step 4: Record usage
        System.out.println("\n--- Recording Usage ---");

        // Acme Corp (PRO): 85,000 API calls (85% of 100K limit — should trigger 80% alert)
        System.out.println("\n[Recording 85,000 API calls for Acme Corp (PRO, limit=100K)]");
        platform.recordUsage(acmeCorp.getTenantId(), UsageMetric.API_CALLS, 85_000);

        // StartupXYZ (FREE): 850 API calls (85% of 1K limit — should trigger alert)
        System.out.println("\n[Recording 850 API calls for StartupXYZ (FREE, limit=1K)]");
        platform.recordUsage(startupXYZ.getTenantId(), UsageMetric.API_CALLS, 850);

        // GlobalBank (ENTERPRISE): 5,000,000 API calls (unlimited — no alert)
        System.out.println("\n[Recording 5,000,000 API calls for GlobalBank (ENTERPRISE, unlimited)]");
        platform.recordUsage(globalBank.getTenantId(), UsageMetric.API_CALLS, 5_000_000);

        // Step 5: Check feature access
        System.out.println("\n--- Feature Access Checks ---");
        boolean startupSSO = platform.checkFeatureAccess(startupXYZ.getTenantId(), FeatureKey.SSO);
        System.out.println("StartupXYZ (FREE) has SSO: " + startupSSO);

        boolean acmeAnalytics = platform.checkFeatureAccess(acmeCorp.getTenantId(), FeatureKey.ADVANCED_ANALYTICS);
        System.out.println("Acme Corp (PRO) has ADVANCED_ANALYTICS: " + acmeAnalytics);

        boolean bankCustomRoles = platform.checkFeatureAccess(globalBank.getTenantId(), FeatureKey.CUSTOM_ROLES);
        System.out.println("GlobalBank (ENTERPRISE) has CUSTOM_ROLES: " + bankCustomRoles);

        // Step 6: Generate invoice for StartupXYZ for current month
        System.out.println("\n--- Invoice Generation for StartupXYZ (FREE plan) ---");
        LocalDate now = LocalDate.now();
        DateRange currentMonth = new DateRange(
            now.withDayOfMonth(1),
            now.withDayOfMonth(now.lengthOfMonth())
        );
        Invoice startupInvoice = platform.generateMonthlyInvoice(startupXYZ.getTenantId(), currentMonth);
        System.out.println(startupInvoice);

        // Step 7: Upgrade StartupXYZ from FREE to PRO mid-month
        System.out.println("\n--- Plan Upgrade: StartupXYZ FREE -> PRO ---");
        Invoice prorationInvoice = platform.upgradePlan(
            startupXYZ.getTenantId(), PlanType.PRO, "admin@startupxyz.com");
        System.out.println("Proration Invoice:");
        System.out.println(prorationInvoice);

        // Step 8: StartupXYZ (now PRO) checks SSO access
        System.out.println("\n--- Feature Access After Upgrade ---");
        boolean startupSSOAfterUpgrade = platform.checkFeatureAccess(startupXYZ.getTenantId(), FeatureKey.SSO);
        System.out.println("StartupXYZ (now PRO) has SSO: " + startupSSOAfterUpgrade);

        // Step 9: Print full audit logs
        System.out.println("\n--- Audit Log: Acme Corp ---");
        List<AuditLogEntry> acmeLogs = platform.getAuditLog(acmeCorp.getTenantId());
        System.out.println("Total audit entries for Acme Corp: " + acmeLogs.size());
        for (AuditLogEntry entry : acmeLogs) {
            System.out.println("  " + entry.getTimestamp().toLocalDate()
                + " | " + entry.getAction()
                + " | actor=" + entry.getActor());
        }

        System.out.println("\n--- Audit Log: StartupXYZ ---");
        List<AuditLogEntry> startupLogs = platform.getAuditLog(startupXYZ.getTenantId());
        System.out.println("Total audit entries for StartupXYZ: " + startupLogs.size());
        for (AuditLogEntry entry : startupLogs) {
            System.out.println("  " + entry.getTimestamp().toLocalDate()
                + " | " + entry.getAction()
                + " | actor=" + entry.getActor());
        }

        System.out.println("\n--- Audit Log: GlobalBank ---");
        List<AuditLogEntry> bankLogs = platform.getAuditLog(globalBank.getTenantId());
        System.out.println("Total audit entries for GlobalBank: " + bankLogs.size());
        for (AuditLogEntry entry : bankLogs) {
            System.out.println("  " + entry.getTimestamp().toLocalDate()
                + " | " + entry.getAction()
                + " | actor=" + entry.getActor());
        }

        // Step 10: Usage summary
        System.out.println("\n--- Usage Summary ---");
        long acmeApiUsage = platform.getMeteringService()
            .getCurrentMonthUsage(acmeCorp.getTenantId(), UsageMetric.API_CALLS);
        long startupApiUsage = platform.getMeteringService()
            .getCurrentMonthUsage(startupXYZ.getTenantId(), UsageMetric.API_CALLS);
        long bankApiUsage = platform.getMeteringService()
            .getCurrentMonthUsage(globalBank.getTenantId(), UsageMetric.API_CALLS);

        System.out.printf("Acme Corp API calls this month:   %,d%n", acmeApiUsage);
        System.out.printf("StartupXYZ API calls this month:  %,d%n", startupApiUsage);
        System.out.printf("GlobalBank API calls this month:  %,d%n", bankApiUsage);

        System.out.println("\n--- Alerts Fired ---");
        List<String> alerts = platform.getAlertListener().getAlerts();
        System.out.println("Total alerts fired: " + alerts.size());
        for (String alert : alerts) {
            System.out.println("  " + alert);
        }

        // Step 11: Current tenant states
        System.out.println("\n--- Final Tenant States ---");
        System.out.println(platform.getTenant(acmeCorp.getTenantId()));
        System.out.println(platform.getTenant(startupXYZ.getTenantId()));
        System.out.println(platform.getTenant(globalBank.getTenantId()));

        System.out.println("\n=================================================");
        System.out.println("  Demo Complete");
        System.out.println("=================================================");
    }
}
```

---

## Unit Tests Design

### PlanFeatureSetTest (6 tests)

1. `freePlanShouldEnableOnlyApiAccess` — Assert that `FreePlanFeatureSet.isFeatureEnabled(API_ACCESS)` returns true, and `isFeatureEnabled(SSO)`, `isFeatureEnabled(ADVANCED_ANALYTICS)`, and all other features return false.

2. `proPlanShouldEnableExactlyFiveFeatures` — Assert that `ProPlanFeatureSet.getEnabledFeatures()` has size 5 and contains API_ACCESS, ADVANCED_ANALYTICS, CUSTOM_DOMAIN, PRIORITY_SUPPORT, WEBHOOKS. Assert SSO and CUSTOM_ROLES are not in the list.

3. `enterprisePlanShouldEnableAllFeatures` — Assert that `EnterprisePlanFeatureSet.isFeatureEnabled(feature)` returns true for every value in `FeatureKey.values()`.

4. `freePlanBasePriceShouldBeZeroForBothCycles` — Assert that `getBasePrice(MONTHLY).isZero()` and `getBasePrice(ANNUAL).isZero()` for the FREE plan.

5. `proPlanApiCallLimitShouldBe100K` — Assert that `ProPlanFeatureSet.getUsageLimit(API_CALLS).getLimit()` equals 100,000 and `isUnlimited()` is false.

6. `enterprisePlanAllLimitsShouldBeUnlimited` — Assert that `getUsageLimit(API_CALLS).isUnlimited()`, `getUsageLimit(STORAGE_BYTES).isUnlimited()`, and `getUsageLimit(USER_COUNT).isUnlimited()` all return true for ENTERPRISE.

### ProrationCalculatorTest (5 tests)

1. `upgradeMidMonthShouldChargePositiveNetAmount` — Given a 30-day billing period, upgrading from FREE ($0) to PRO ($29.99) on day 15 should produce a positive proration amount (approximately $14.99).

2. `downgradeMidMonthShouldReturnNegativeNetAmount` — Given a 30-day billing period, downgrading from PRO ($29.99) to FREE ($0) on day 15 should produce a negative or zero proration amount (credit).

3. `changeDateAtStartOfPeriodShouldChargeFullMonth` — When `changeDate` equals `billingPeriod.startDate`, `getDaysRemaining` should return `billingPeriod.totalDays()`, and proration should equal the full price difference.

4. `changeDateAtEndOfPeriodShouldChargeZero` — When `changeDate` equals `billingPeriod.endDate`, `getDaysRemaining` should return 0 and `calculateProration` should return zero.

5. `getProratedAmountShouldComputeCorrectDailyRate` — For a $30.00 monthly price over 30 days with 10 days remaining, `getProratedAmount` should return $10.00 (exactly $1/day × 10 days).

### UsageMeteringServiceTest (5 tests)

1. `recordUsageShouldNotifyAllListeners` — Register two mock listeners, call `recordUsage`, assert both listeners received `onUsageRecorded` with the correct tenantId, metric, and newTotal.

2. `getTotalUsageShouldSumOnlyRecordsWithinPeriod` — Record usage on day 1, day 15, and day 35 of a given month. Call `getTotalUsage` with a DateRange spanning only that month. Assert the total sums only the day-1 and day-15 records, not day-35.

3. `isWithinLimitShouldReturnFalseWhenOverLimit` — For a FREE tenant with a 1,000 API call limit, record 1,001 calls, then assert `isWithinLimit(tenantId, API_CALLS)` returns false.

4. `alertListenerShouldFireAtEightyPercentThreshold` — Register a `UsageLimitAlertListener`, record exactly 800 API calls for a FREE tenant (limit=1000, so 80%), assert `getAlerts()` has size 1 containing the tenant ID.

5. `unsubscribedListenerShouldNotReceiveEvents` — Subscribe then unsubscribe a mock listener. Record usage. Assert the listener received no calls after unsubscription.

### InvoiceBuilderTest (4 tests)

1. `buildWithOnlyBaseChargeShouldComputeCorrectTotal` — Build an invoice with only a PRO plan base charge (MONTHLY = $29.99) and no tax. Assert `subtotal` equals $29.99, `tax` equals $0.00, `total` equals $29.99.

2. `buildWithTaxRateShouldComputeTaxCorrectly` — Build an invoice with a $100.00 base charge and a 10% tax rate. Assert `tax` equals $10.00 and `total` equals $110.00.

3. `buildWithoutTenantIdShouldThrowIllegalStateException` — Call `build()` without calling `forTenant()`. Assert that `IllegalStateException` is thrown.

4. `addOverageChargeShouldSkipWhenOverageIsZero` — Call `addOverageCharge(API_CALLS, 0, rate)`. Assert the resulting invoice has exactly one line item (the base charge) and zero overage line items.

### BillingServiceTest (5 tests)

1. `generateInvoiceShouldIncludeBaseChargeForPlanType` — For a PRO tenant with zero usage, `generateInvoice` should return an invoice with one line item whose description contains "PRO" and total equals $29.99.

2. `generateInvoiceShouldAddOverageLineItemWhenApiCallsExceedLimit` — For a FREE tenant (limit=1000 API calls), record 1,500 calls, then call `generateInvoice`. Assert the invoice has two line items: base charge ($0.00) and an overage charge for 500 calls at $0.001 = $0.50.

3. `processPaymentShouldMarkInvoicePaidOnSuccess` — Use a `MockPaymentGateway(false)` (success mode), call `processPayment`, assert the invoice status becomes "PAID" and the returned `PaymentResult.isSuccessful()` is true.

4. `processPaymentShouldMarkInvoiceFailedOnGatewayFailure` — Use a `MockPaymentGateway(true)` (failure mode), call `processPayment`, assert invoice status becomes "FAILED" and `PaymentResult.isSuccessful()` is false.

5. `processRenewalWithFailedPaymentShouldSuspendTenant` — Set up a billing service with a failing gateway. Call `processRenewal`. Assert the tenant's status is SUSPENDED after the method returns.

### FeatureGatedTenantServiceTest (4 tests)

1. `updatePlanShouldLogFeatureGateCheckBeforeDelegating` — Capture stdout, call `updatePlan`. Assert that the console output contains "FEATURE-GATE" and the plan change was still applied (delegate was called).

2. `getAllTenantsShouldDelegateToCoreServiceWithoutGating` — Call `getAllTenants` on the decorated service. Assert the result equals what the delegate would return (no gate applied).

3. `createTenantShouldDelegateAndReturnTenantFromDelegate` — Call `createTenant`. Assert the returned tenant has the correct name and plan type from the delegate's result.

4. `suspendTenantOnFreePlanShouldLogWarningAboutMissingAuditLogs` — Capture stdout, suspend a FREE tenant (no AUDIT_LOGS feature), assert console output contains "does not have AUDIT_LOGS".

### TrialManagerTest (4 tests)

1. `startTrialShouldSetTenantStatusToTrial` — Call `startTrial`. Assert `getTenant(tenantId).getStatus()` is TRIAL and `getTrialEndDate()` is 14 days from today.

2. `isInTrialShouldReturnFalseForNonTrialTenant` — For a tenant on which `startTrial` was never called, assert `isInTrial` returns false.

3. `daysRemainingShouldReturn14OnFirstDayOfTrial` — Immediately after calling `startTrial`, assert `daysRemaining` returns 14 (or 13 depending on same-day semantics — document which).

4. `expireTrialShouldConvertTenantToFreeActiveStatus` — Call `startTrial`, then immediately call `expireTrial`. Assert tenant status is ACTIVE and planType is FREE.

---

## Design Decisions

### 1. Strategy for Plan Features Over Inheritance

The most natural first instinct for plan tiers is inheritance: a `Plan` base class with `FreePlan`, `ProPlan`, and `EnterprisePlan` subclasses. This breaks down immediately when billing logic asks "what is the price for this plan?" — every calling class must cast or use `instanceof`. More critically, the Open/Closed Principle states that software entities should be open for extension but closed for modification. With Strategy, adding an ULTRA_ENTERPRISE plan means writing one new class that implements `PlanFeatureSet` and registering it in `PlanFeatureSetFactory`. Zero existing code changes. With inheritance, every switch statement or cast in the codebase must be updated.

Strategy also enables runtime swappability. A promotional campaign that temporarily gives FREE accounts PRO features for a weekend can be implemented by injecting a `PromotionalPlanFeatureSet` without touching billing, metering, or tenant services. This would be impossible with a fixed inheritance hierarchy.

### 2. Decorator for Feature Gating Over Inline Conditionals

The alternative to Decorator is adding auditing and feature gating directly into `TenantServiceImpl`. This violates Single Responsibility: the class would need to understand authentication tokens, audit schemas, and feature flags in addition to managing tenants. Every new cross-cutting concern (rate limiting, request caching, distributed tracing) adds another dependency.

Decorator solves this by treating the service as a pipeline. `AuditedTenantService` wraps `TenantServiceImpl` and adds audit writes after every mutation. `FeatureGatedTenantService` wraps `AuditedTenantService` and adds gate checks before certain mutations. Each layer is independently testable: an audit test passes a mock `TenantService` and a real `AuditLogService`, never touching the core implementation. The layers compose in different orders for different contexts: a background billing job might bypass the feature gate layer while keeping auditing.

### 3. Observer for Usage Metering

When `UsageMeteringService.recordUsage` is called, the immediate reaction today is: check the limit, fire an alert if over 80%, write an audit entry. Tomorrow it might also push a webhook, trigger an automatic upgrade prompt, or update a billing forecast. With direct method calls, each new reaction requires modifying `recordUsage`. With Observer, new reactions are added by implementing `UsageEventListener` and calling `subscribe()`. The metering service never changes.

There is a subtle correctness concern: listeners are notified after the record is stored, not before. This means a listener that throws an exception would not prevent the usage from being recorded, which is correct behavior — usage accounting must not be blocked by a failing alerting system.

### 4. Proration Arithmetic: Daily Rate Approach

The system computes proration by dividing the monthly price by the number of days in the billing period, then multiplying by the number of days remaining after the change date. This is a daily rate approach. The tradeoff is that February gives each day a higher value than January (28 days vs 31 days), so upgrading on February 14 results in a slightly different proration than upgrading on January 14 despite both being mid-month. Annual billing requires a different calculation: the annual price is divided by 365 (or 366 in a leap year) to compute a daily rate.

The alternative is a 30-day month assumption regardless of actual month length. This is simpler but creates user-visible inconsistency: customers on annual plans notice they pay slightly more in February than expected. Most SaaS platforms use the actual-days approach for transparency, accepting the minor month-to-month variation.

Leap years are handled automatically by `ChronoUnit.DAYS.between()`, which uses the actual calendar. An annual plan renewed in a leap year has 366 days, so each day's value is slightly lower.

### 5. Extending to Annual Billing

The `BillingCycle` enum already has `ANNUAL`. Each `PlanFeatureSet` already returns different prices for monthly vs annual billing cycles. The `generateInvoice` method passes `tenant.getBillingCycle()` to `addBaseCharge`, so annual tenants automatically get annual pricing.

What changes for annual billing is the proration calculation. A plan change mid-annual-cycle involves a much larger credit/charge than mid-monthly. The `ProrationCalculator` is agnostic to billing cycle — it takes a `DateRange` and a `changeDate` and computes days remaining, so passing an annual `DateRange` (365 days) produces correct results without modification. The `processRenewal` method would need to set the `DateRange` to the annual billing period rather than the current month; this is a one-line change at the call site.

### 6. Adding a New Feature Flag

To add a new feature flag (e.g., `AI_ASSISTANT`), the change procedure is:
1. Add `AI_ASSISTANT` to the `FeatureKey` enum.
2. Update each `PlanFeatureSet` implementation to handle the new key — three classes: `FreePlanFeatureSet` (returns false), `ProPlanFeatureSet` (returns false or true depending on business decision), `EnterprisePlanFeatureSet` (returns true, as it enables all features).
3. No changes are required to `TenantService`, `BillingService`, `UsageMeteringService`, `AuditedTenantService`, `FeatureGatedTenantService`, or any other class.

The cost is O(n plans): each plan implementation must handle the new key. This is acceptable because the number of plans is small and fixed, and each implementation is a focused, easily-audited change. The alternative — a central feature matrix Map — would be O(1) to add a flag but O(n) harder to audit because the matrix would grow unboundedly and lose the compile-time check that every plan declares a policy for every feature.

---

## SOLID Principles Checklist

### S — Single Responsibility Principle

A class should have only one reason to change.

`TenantServiceImpl` has one reason to change: tenant storage and retrieval logic. It does not know about auditing, feature gating, billing, or usage metering. When the audit schema changes, only `AuditedTenantService` and `AuditLogEntry` change. When a new payment gateway is added, only `PaymentGateway` implementations change. `UsageMeteringService` is responsible for recording usage and notifying listeners — it does not decide what to do when limits are exceeded; that responsibility belongs to the listeners.

### O — Open/Closed Principle

Software entities should be open for extension but closed for modification.

Adding a new plan tier requires creating a new `PlanFeatureSet` implementation and adding one case to `PlanFeatureSetFactory`. No existing service, billing engine, or metering code changes. Adding a new usage event reaction requires writing a new `UsageEventListener` and registering it with `UsageMeteringService` — the service itself does not change. Adding a new cross-cutting concern to `TenantService` requires writing a new decorator — `TenantServiceImpl` does not change.

### L — Liskov Substitution Principle

Subtypes must be substitutable for their base types without altering correctness.

All three `PlanFeatureSet` implementations can be substituted wherever a `PlanFeatureSet` is expected. `BillingService.generateInvoice` calls `featureSet.getBasePrice(cycle)` — it does not matter which implementation is passed; the behavior is always correct for that plan. All three `TenantService` decorators can be substituted for each other (or for `TenantServiceImpl`) — a caller that holds a `TenantService` reference does not know and does not care which layer it is talking to.

### I — Interface Segregation Principle

Clients should not be forced to depend on interfaces they do not use.

`PlanFeatureSet` is split from billing concerns (`getBasePrice`) which are split from usage concerns (via `getUsageLimit`). `UsageEventListener` has exactly one method — implementors are not forced to implement alert logic if they only need audit logic. `PaymentGateway` has only `charge` and `refund` — it does not include subscription management or invoice generation methods that payment gateways don't own. `AuditLogService` is a dedicated interface for audit concerns, not bundled into `TenantService`.

### D — Dependency Inversion Principle

High-level modules should depend on abstractions, not concretions.

`BillingService` depends on `TenantService` (interface), `PaymentGateway` (interface), and `AuditLogService` (interface) — all injected via constructor. It never references `TenantServiceImpl`, `StripePaymentGateway`, or `InMemoryAuditLogService` directly. This makes it trivial to swap `StripePaymentGateway` for `MockPaymentGateway` in tests. `UsageMeteringService` depends on a `Function<String, PlanType>` lambda rather than a concrete service class, which is the most flexible form of dependency inversion: any code that can look up a plan type by tenant ID satisfies the contract. `SaaSPlatform` is the only class that names concrete implementations — this is the Composition Root pattern, centralizing wiring in one place so the rest of the system never knows which implementations are live.
