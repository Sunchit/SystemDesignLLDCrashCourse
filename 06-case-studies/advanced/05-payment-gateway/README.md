# Payment Gateway — Advanced LLD Case Study

## Table of Contents
1. [Problem Statement](#1-problem-statement)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Design Goals and Constraints](#4-design-goals-and-constraints)
5. [Architecture / Class Diagram](#5-architecture--class-diagram)
6. [Complete Java Implementation](#6-complete-java-implementation)
7. [Design Patterns Used](#7-design-patterns-used)
8. [Key Design Decisions and Trade-offs](#8-key-design-decisions-and-trade-offs)
9. [Extension Points](#9-extension-points)
10. [Production Considerations](#10-production-considerations)

---

## 1. Problem Statement

A modern e-commerce platform needs a **Payment Gateway** that abstracts over multiple upstream payment providers (Stripe, PayPal, Razorpay) and exposes a single, consistent API to the rest of the platform. The gateway must handle the full lifecycle of a payment — from initiation through settlement — including asynchronous status updates from providers via webhooks, refunds, fraud screening, and idempotency to prevent double charges.

**Core challenge:** Payment systems deal with distributed, eventually-consistent flows where money moves across network boundaries. Failures can leave transactions in ambiguous states. The design must be correct under partial failures, safe against duplicate requests, auditable for compliance, and extensible as new providers and payment methods are added.

---

## 2. Functional Requirements

| # | Requirement |
|---|-------------|
| FR-1 | Accept payments via Credit Card, Debit Card, UPI, Net Banking, and Wallet |
| FR-2 | Route payments through Stripe, PayPal, or Razorpay based on configuration/rules |
| FR-3 | Enforce idempotency — same `idempotencyKey` must never produce two charges |
| FR-4 | Expose a refund API supporting full and partial refunds |
| FR-5 | Receive and process provider webhooks for async status updates |
| FR-6 | Run fraud-detection hooks before processing each payment |
| FR-7 | Enforce per-user and per-merchant transaction limits |
| FR-8 | Tokenize card data — raw PAN never stored after authorisation |
| FR-9 | Produce a settlement reconciliation report for each business day |
| FR-10 | Emit domain events so downstream systems (order service, notification service) react to payment outcomes |

---

## 3. Non-Functional Requirements

| Attribute | Target |
|-----------|--------|
| Availability | 99.99 % (< 53 min/year downtime) |
| Latency (p99) | < 2 s for synchronous payment response |
| Throughput | 10 000 TPS peak |
| Durability | Zero loss of committed transaction records |
| Security | PCI-DSS Level 1 design; no raw PAN at rest |
| Idempotency window | 24 hours |
| Auditability | Immutable audit log for every state transition |
| Scalability | Horizontal scaling of gateway nodes; stateless design |

---

## 4. Design Goals and Constraints

### Goals
- **Provider agnosticism:** Adding a new payment provider requires only one new class, zero changes to existing code.
- **Correctness over availability:** In ambiguous failure scenarios, prefer safe (non-charge) outcomes.
- **Auditability:** Every state transition is logged with actor, timestamp, and reason.
- **Testability:** Business logic is pure and dependency-injected; no static state.

### Constraints
- Raw card numbers (PAN) must never be stored in application memory beyond the tokenisation call.
- The gateway must be stateless between requests; all state lives in the DB and cache.
- Fraud checks are synchronous and blocking for the MVP; async is an extension point.
- Idempotency keys expire after 24 hours to bound the deduplication cache size.

---

## 5. Architecture / Class Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CLIENT / ORDER SERVICE                              │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ HTTP
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PaymentGatewayFacade  (Facade)                        │
│   initiatePayment()  refundPayment()  getTransaction()  handleWebhook()      │
└──────┬──────────────────┬─────────────────────┬────────────────────────────┘
       │                  │                      │
       ▼                  ▼                      ▼
┌─────────────┐  ┌─────────────────┐  ┌──────────────────────┐
│Idempotency  │  │  FraudCheck     │  │  WebhookProcessor    │
│  Service    │  │  Chain (CoR)    │  │  (Observer/Events)   │
└─────────────┘  │                 │  └──────────────────────┘
                 │ VelocityCheck   │
                 │ AmountLimitCheck│
                 │ BlacklistCheck  │
                 └────────┬────────┘
                          │ pass
                          ▼
              ┌───────────────────────┐
              │   PaymentProcessor    │
              │  (orchestrates flow)  │
              └───────┬───────────────┘
                      │ selects provider
                      ▼
         ┌────────────────────────────┐
         │  PaymentProviderStrategy   │  <<interface>>
         │  + charge()                │
         │  + refund()                │
         │  + verifyWebhook()         │
         └────┬──────────┬────────────┘
              │          │           │
    ┌─────────┘    ┌─────┘    ┌──────┘
    ▼              ▼           ▼
┌────────┐  ┌──────────┐  ┌────────────┐
│ Stripe │  │  PayPal  │  │  Razorpay  │
│Provider│  │ Provider │  │  Provider  │
└────────┘  └──────────┘  └────────────┘

         ┌─────────────────────────────┐
         │     Transaction (State)     │
         │  INITIATED → PROCESSING     │
         │  PROCESSING → SUCCESS       │
         │  PROCESSING → FAILED        │
         │  SUCCESS    → REFUNDED      │
         └─────────────────────────────┘

         ┌─────────────────────────────┐
         │   TokenisationService       │
         │   + tokenise(CardData)      │
         │   + detokenise(token)       │
         └─────────────────────────────┘

         ┌─────────────────────────────┐
         │   SettlementService         │
         │   + reconcile(date)         │
         └─────────────────────────────┘

         ┌─────────────────────────────┐
         │   PaymentEventPublisher     │
         │   (Observer — publishes to  │
         │    Kafka / in-proc bus)     │
         └─────────────────────────────┘
```

### Package Structure

```
com.payment.gateway
├── api
│   └── PaymentGatewayFacade.java
├── model
│   ├── Transaction.java
│   ├── TransactionStatus.java
│   ├── PaymentMethod.java
│   ├── PaymentRequest.java
│   ├── PaymentResult.java
│   ├── RefundRequest.java
│   ├── RefundResult.java
│   ├── CardData.java
│   └── WebhookPayload.java
├── provider
│   ├── PaymentProviderStrategy.java
│   ├── ProviderType.java
│   ├── StripePaymentProvider.java
│   ├── PayPalPaymentProvider.java
│   ├── RazorpayPaymentProvider.java
│   └── PaymentProviderRegistry.java
├── fraud
│   ├── FraudCheckHandler.java
│   ├── FraudCheckRequest.java
│   ├── FraudCheckResult.java
│   ├── VelocityCheckHandler.java
│   ├── AmountLimitCheckHandler.java
│   └── BlacklistCheckHandler.java
├── state
│   ├── TransactionState.java
│   ├── InitiatedState.java
│   ├── ProcessingState.java
│   ├── SuccessState.java
│   ├── FailedState.java
│   └── RefundedState.java
├── service
│   ├── PaymentProcessor.java
│   ├── IdempotencyService.java
│   ├── TokenisationService.java
│   ├── WebhookProcessor.java
│   ├── SettlementService.java
│   └── AuditLogService.java
├── event
│   ├── PaymentEvent.java
│   ├── PaymentEventPublisher.java
│   └── PaymentEventListener.java
└── exception
    ├── PaymentException.java
    ├── DuplicatePaymentException.java
    ├── FraudDetectedException.java
    ├── ProviderException.java
    └── InvalidStateTransitionException.java
```

---

## 6. Complete Java Implementation

### 6.1 Domain Model

```java
// com/payment/gateway/model/TransactionStatus.java
package com.payment.gateway.model;

public enum TransactionStatus {
    INITIATED,
    PROCESSING,
    SUCCESS,
    FAILED,
    REFUNDED;

    public boolean isTerminal() {
        return this == SUCCESS || this == FAILED || this == REFUNDED;
    }

    public boolean canTransitionTo(TransactionStatus next) {
        return switch (this) {
            case INITIATED   -> next == PROCESSING || next == FAILED;
            case PROCESSING  -> next == SUCCESS || next == FAILED;
            case SUCCESS     -> next == REFUNDED;
            case FAILED, REFUNDED -> false;
        };
    }
}
```

```java
// com/payment/gateway/model/PaymentMethod.java
package com.payment.gateway.model;

public enum PaymentMethod {
    CREDIT_CARD,
    DEBIT_CARD,
    UPI,
    NET_BANKING,
    WALLET
}
```

```java
// com/payment/gateway/model/CardData.java
package com.payment.gateway.model;

/**
 * Sensitive card data — held in memory only during the tokenisation call.
 * Never serialised to disk or logs. Fields are cleared after use.
 */
public final class CardData {

    private final char[] pan;       // Primary Account Number
    private final char[] cvv;
    private final String expiryMonth;
    private final String expiryYear;
    private final String cardholderName;

    public CardData(char[] pan, char[] cvv,
                    String expiryMonth, String expiryYear,
                    String cardholderName) {
        this.pan           = pan.clone();
        this.cvv           = cvv.clone();
        this.expiryMonth   = expiryMonth;
        this.expiryYear    = expiryYear;
        this.cardholderName = cardholderName;
    }

    public char[] getPan()  { return pan.clone(); }
    public char[] getCvv()  { return cvv.clone(); }
    public String getExpiryMonth()    { return expiryMonth; }
    public String getExpiryYear()     { return expiryYear; }
    public String getCardholderName() { return cardholderName; }

    /** Zero-out sensitive fields after use. */
    public void destroy() {
        java.util.Arrays.fill(pan, '\0');
        java.util.Arrays.fill(cvv, '\0');
    }
}
```

```java
// com/payment/gateway/model/PaymentRequest.java
package com.payment.gateway.model;

import java.math.BigDecimal;
import java.util.Currency;
import java.util.Objects;

public final class PaymentRequest {

    private final String        idempotencyKey;
    private final String        merchantId;
    private final String        customerId;
    private final BigDecimal    amount;
    private final Currency      currency;
    private final PaymentMethod paymentMethod;
    private final String        paymentToken;   // tokenised card / UPI VPA / etc.
    private final String        description;
    private final ProviderHint  providerHint;   // nullable — let gateway decide

    private PaymentRequest(Builder b) {
        this.idempotencyKey = Objects.requireNonNull(b.idempotencyKey, "idempotencyKey");
        this.merchantId     = Objects.requireNonNull(b.merchantId,     "merchantId");
        this.customerId     = Objects.requireNonNull(b.customerId,     "customerId");
        this.amount         = Objects.requireNonNull(b.amount,         "amount");
        this.currency       = Objects.requireNonNull(b.currency,       "currency");
        this.paymentMethod  = Objects.requireNonNull(b.paymentMethod,  "paymentMethod");
        this.paymentToken   = Objects.requireNonNull(b.paymentToken,   "paymentToken");
        this.description    = b.description;
        this.providerHint   = b.providerHint;

        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
    }

    public String        getIdempotencyKey() { return idempotencyKey; }
    public String        getMerchantId()     { return merchantId; }
    public String        getCustomerId()     { return customerId; }
    public BigDecimal    getAmount()         { return amount; }
    public Currency      getCurrency()       { return currency; }
    public PaymentMethod getPaymentMethod()  { return paymentMethod; }
    public String        getPaymentToken()   { return paymentToken; }
    public String        getDescription()    { return description; }
    public ProviderHint  getProviderHint()   { return providerHint; }

    public enum ProviderHint { STRIPE, PAYPAL, RAZORPAY }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String idempotencyKey;
        private String merchantId;
        private String customerId;
        private BigDecimal amount;
        private Currency currency;
        private PaymentMethod paymentMethod;
        private String paymentToken;
        private String description;
        private ProviderHint providerHint;

        public Builder idempotencyKey(String v)  { this.idempotencyKey = v; return this; }
        public Builder merchantId(String v)       { this.merchantId     = v; return this; }
        public Builder customerId(String v)       { this.customerId     = v; return this; }
        public Builder amount(BigDecimal v)       { this.amount         = v; return this; }
        public Builder currency(Currency v)       { this.currency       = v; return this; }
        public Builder paymentMethod(PaymentMethod v) { this.paymentMethod = v; return this; }
        public Builder paymentToken(String v)     { this.paymentToken   = v; return this; }
        public Builder description(String v)      { this.description    = v; return this; }
        public Builder providerHint(ProviderHint v) { this.providerHint = v; return this; }

        public PaymentRequest build() { return new PaymentRequest(this); }
    }
}
```

```java
// com/payment/gateway/model/PaymentResult.java
package com.payment.gateway.model;

import java.math.BigDecimal;
import java.time.Instant;

public record PaymentResult(
        String          transactionId,
        TransactionStatus status,
        BigDecimal      amount,
        String          currency,
        String          providerReference,
        String          errorCode,
        String          errorMessage,
        Instant         timestamp
) {
    public boolean isSuccess() { return status == TransactionStatus.SUCCESS; }

    public static PaymentResult success(String txnId, BigDecimal amount,
                                        String currency, String providerRef) {
        return new PaymentResult(txnId, TransactionStatus.SUCCESS, amount,
                currency, providerRef, null, null, Instant.now());
    }

    public static PaymentResult failure(String txnId, BigDecimal amount,
                                        String currency, String code, String msg) {
        return new PaymentResult(txnId, TransactionStatus.FAILED, amount,
                currency, null, code, msg, Instant.now());
    }
}
```

```java
// com/payment/gateway/model/RefundRequest.java
package com.payment.gateway.model;

import java.math.BigDecimal;
import java.util.Objects;

public final class RefundRequest {

    private final String     idempotencyKey;
    private final String     originalTransactionId;
    private final BigDecimal refundAmount;   // null = full refund
    private final String     reason;
    private final String     requestedBy;

    public RefundRequest(String idempotencyKey, String originalTransactionId,
                         BigDecimal refundAmount, String reason, String requestedBy) {
        this.idempotencyKey        = Objects.requireNonNull(idempotencyKey);
        this.originalTransactionId = Objects.requireNonNull(originalTransactionId);
        this.refundAmount          = refundAmount; // nullable → full refund
        this.reason                = reason;
        this.requestedBy           = requestedBy;
    }

    public String     getIdempotencyKey()        { return idempotencyKey; }
    public String     getOriginalTransactionId() { return originalTransactionId; }
    public BigDecimal getRefundAmount()          { return refundAmount; }
    public String     getReason()                { return reason; }
    public String     getRequestedBy()           { return requestedBy; }
    public boolean    isFullRefund()             { return refundAmount == null; }
}
```

```java
// com/payment/gateway/model/RefundResult.java
package com.payment.gateway.model;

import java.math.BigDecimal;
import java.time.Instant;

public record RefundResult(
        String     refundId,
        String     originalTransactionId,
        BigDecimal refundedAmount,
        boolean    success,
        String     errorMessage,
        Instant    timestamp
) {}
```

```java
// com/payment/gateway/model/Transaction.java
package com.payment.gateway.model;

import com.payment.gateway.state.TransactionState;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.Currency;
import java.util.UUID;

/**
 * Aggregate root for a payment transaction.
 * State transitions are validated by the embedded TransactionState.
 */
public final class Transaction {

    private final String          id;
    private final String          idempotencyKey;
    private final String          merchantId;
    private final String          customerId;
    private final BigDecimal      amount;
    private final Currency        currency;
    private final PaymentMethod   paymentMethod;
    private final String          paymentToken;
    private final ProviderType    provider;
    private       TransactionStatus status;
    private       TransactionState  state;         // State-pattern delegate
    private       String          providerReference;
    private       String          failureCode;
    private       String          failureMessage;
    private final Instant         createdAt;
    private       Instant         updatedAt;
    private       BigDecimal      refundedAmount;

    public Transaction(String idempotencyKey, String merchantId, String customerId,
                       BigDecimal amount, Currency currency, PaymentMethod paymentMethod,
                       String paymentToken, ProviderType provider,
                       TransactionState initialState) {
        this.id             = UUID.randomUUID().toString();
        this.idempotencyKey = idempotencyKey;
        this.merchantId     = merchantId;
        this.customerId     = customerId;
        this.amount         = amount;
        this.currency       = currency;
        this.paymentMethod  = paymentMethod;
        this.paymentToken   = paymentToken;
        this.provider       = provider;
        this.status         = TransactionStatus.INITIATED;
        this.state          = initialState;
        this.createdAt      = Instant.now();
        this.updatedAt      = this.createdAt;
        this.refundedAmount = BigDecimal.ZERO;
    }

    // ── State transitions (delegated to State objects) ──────────────────────

    public void startProcessing() {
        state.startProcessing(this);
    }

    public void markSuccess(String providerRef) {
        state.markSuccess(this, providerRef);
    }

    public void markFailed(String code, String message) {
        state.markFailed(this, code, message);
    }

    public void markRefunded(BigDecimal refundAmount) {
        state.markRefunded(this, refundAmount);
    }

    // ── Internal setters called by State objects ─────────────────────────────

    public void applyStatus(TransactionStatus newStatus, TransactionState newState) {
        if (!this.status.canTransitionTo(newStatus)) {
            throw new com.payment.gateway.exception.InvalidStateTransitionException(
                    id, this.status, newStatus);
        }
        this.status    = newStatus;
        this.state     = newState;
        this.updatedAt = Instant.now();
    }

    public void setProviderReference(String ref)   { this.providerReference = ref; }
    public void setFailureCode(String code)         { this.failureCode       = code; }
    public void setFailureMessage(String msg)       { this.failureMessage    = msg; }
    public void addRefundedAmount(BigDecimal amt)   {
        this.refundedAmount = this.refundedAmount.add(amt);
    }

    // ── Accessors ─────────────────────────────────────────────────────────────

    public String            getId()               { return id; }
    public String            getIdempotencyKey()   { return idempotencyKey; }
    public String            getMerchantId()       { return merchantId; }
    public String            getCustomerId()       { return customerId; }
    public BigDecimal        getAmount()           { return amount; }
    public Currency          getCurrency()         { return currency; }
    public PaymentMethod     getPaymentMethod()    { return paymentMethod; }
    public String            getPaymentToken()     { return paymentToken; }
    public ProviderType      getProvider()         { return provider; }
    public TransactionStatus getStatus()           { return status; }
    public String            getProviderReference(){ return providerReference; }
    public String            getFailureCode()      { return failureCode; }
    public String            getFailureMessage()   { return failureMessage; }
    public Instant           getCreatedAt()        { return createdAt; }
    public Instant           getUpdatedAt()        { return updatedAt; }
    public BigDecimal        getRefundedAmount()   { return refundedAmount; }

    public BigDecimal getRemainingRefundableAmount() {
        return amount.subtract(refundedAmount);
    }
}
```

```java
// com/payment/gateway/model/WebhookPayload.java
package com.payment.gateway.model;

import java.time.Instant;
import java.util.Map;

public record WebhookPayload(
        String              providerId,
        String              eventType,
        String              providerReference,
        TransactionStatus   reportedStatus,
        Map<String, String> rawHeaders,
        String              rawBody,
        Instant             receivedAt
) {}
```

```java
// com/payment/gateway/provider/ProviderType.java
package com.payment.gateway.provider;

public enum ProviderType {
    STRIPE, PAYPAL, RAZORPAY
}
```

### 6.2 Exception Hierarchy

```java
// com/payment/gateway/exception/PaymentException.java
package com.payment.gateway.exception;

public class PaymentException extends RuntimeException {
    private final String errorCode;

    public PaymentException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public PaymentException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}
```

```java
// com/payment/gateway/exception/DuplicatePaymentException.java
package com.payment.gateway.exception;

public class DuplicatePaymentException extends PaymentException {
    private final String existingTransactionId;

    public DuplicatePaymentException(String idempotencyKey, String existingTxnId) {
        super("DUPLICATE_PAYMENT",
              "Payment with idempotency key already processed: " + idempotencyKey);
        this.existingTransactionId = existingTxnId;
    }

    public String getExistingTransactionId() { return existingTransactionId; }
}
```

```java
// com/payment/gateway/exception/FraudDetectedException.java
package com.payment.gateway.exception;

public class FraudDetectedException extends PaymentException {
    private final String ruleTriggered;

    public FraudDetectedException(String ruleTriggered, String details) {
        super("FRAUD_DETECTED", "Payment blocked by fraud rule [" + ruleTriggered + "]: " + details);
        this.ruleTriggered = ruleTriggered;
    }

    public String getRuleTriggered() { return ruleTriggered; }
}
```

```java
// com/payment/gateway/exception/ProviderException.java
package com.payment.gateway.exception;

import com.payment.gateway.provider.ProviderType;

public class ProviderException extends PaymentException {
    private final ProviderType provider;

    public ProviderException(ProviderType provider, String code, String message) {
        super(code, "[" + provider + "] " + message);
        this.provider = provider;
    }

    public ProviderException(ProviderType provider, String code, String message, Throwable cause) {
        super(code, "[" + provider + "] " + message, cause);
        this.provider = provider;
    }

    public ProviderType getProvider() { return provider; }
}
```

```java
// com/payment/gateway/exception/InvalidStateTransitionException.java
package com.payment.gateway.exception;

import com.payment.gateway.model.TransactionStatus;

public class InvalidStateTransitionException extends PaymentException {
    public InvalidStateTransitionException(String txnId,
                                           TransactionStatus from,
                                           TransactionStatus to) {
        super("INVALID_STATE_TRANSITION",
              "Transaction " + txnId + " cannot move from " + from + " to " + to);
    }
}
```

### 6.3 State Pattern — Transaction Lifecycle

```java
// com/payment/gateway/state/TransactionState.java
package com.payment.gateway.state;

import com.payment.gateway.model.Transaction;
import java.math.BigDecimal;

/**
 * State interface for the Transaction state machine.
 * Each concrete state permits only valid transitions.
 */
public interface TransactionState {
    void startProcessing(Transaction txn);
    void markSuccess(Transaction txn, String providerRef);
    void markFailed(Transaction txn, String code, String message);
    void markRefunded(Transaction txn, BigDecimal refundAmount);
}
```

```java
// com/payment/gateway/state/InitiatedState.java
package com.payment.gateway.state;

import com.payment.gateway.exception.InvalidStateTransitionException;
import com.payment.gateway.model.Transaction;
import com.payment.gateway.model.TransactionStatus;
import java.math.BigDecimal;

public final class InitiatedState implements TransactionState {

    @Override
    public void startProcessing(Transaction txn) {
        txn.applyStatus(TransactionStatus.PROCESSING, new ProcessingState());
    }

    @Override
    public void markSuccess(Transaction txn, String providerRef) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.INITIATED, TransactionStatus.SUCCESS);
    }

    @Override
    public void markFailed(Transaction txn, String code, String message) {
        txn.setFailureCode(code);
        txn.setFailureMessage(message);
        txn.applyStatus(TransactionStatus.FAILED, new FailedState());
    }

    @Override
    public void markRefunded(Transaction txn, BigDecimal refundAmount) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.INITIATED, TransactionStatus.REFUNDED);
    }
}
```

```java
// com/payment/gateway/state/ProcessingState.java
package com.payment.gateway.state;

import com.payment.gateway.exception.InvalidStateTransitionException;
import com.payment.gateway.model.Transaction;
import com.payment.gateway.model.TransactionStatus;
import java.math.BigDecimal;

public final class ProcessingState implements TransactionState {

    @Override
    public void startProcessing(Transaction txn) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.PROCESSING, TransactionStatus.PROCESSING);
    }

    @Override
    public void markSuccess(Transaction txn, String providerRef) {
        txn.setProviderReference(providerRef);
        txn.applyStatus(TransactionStatus.SUCCESS, new SuccessState());
    }

    @Override
    public void markFailed(Transaction txn, String code, String message) {
        txn.setFailureCode(code);
        txn.setFailureMessage(message);
        txn.applyStatus(TransactionStatus.FAILED, new FailedState());
    }

    @Override
    public void markRefunded(Transaction txn, BigDecimal refundAmount) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.PROCESSING, TransactionStatus.REFUNDED);
    }
}
```

```java
// com/payment/gateway/state/SuccessState.java
package com.payment.gateway.state;

import com.payment.gateway.exception.InvalidStateTransitionException;
import com.payment.gateway.model.Transaction;
import com.payment.gateway.model.TransactionStatus;
import java.math.BigDecimal;

public final class SuccessState implements TransactionState {

    @Override
    public void startProcessing(Transaction txn) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.SUCCESS, TransactionStatus.PROCESSING);
    }

    @Override
    public void markSuccess(Transaction txn, String providerRef) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.SUCCESS, TransactionStatus.SUCCESS);
    }

    @Override
    public void markFailed(Transaction txn, String code, String message) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.SUCCESS, TransactionStatus.FAILED);
    }

    @Override
    public void markRefunded(Transaction txn, BigDecimal refundAmount) {
        if (refundAmount == null) {
            refundAmount = txn.getRemainingRefundableAmount();
        }
        if (refundAmount.compareTo(txn.getRemainingRefundableAmount()) > 0) {
            throw new com.payment.gateway.exception.PaymentException(
                    "REFUND_EXCEEDS_AMOUNT",
                    "Refund amount " + refundAmount + " exceeds refundable amount "
                    + txn.getRemainingRefundableAmount());
        }
        txn.addRefundedAmount(refundAmount);
        // Partial refund keeps transaction in SUCCESS; full refund moves to REFUNDED
        if (txn.getRemainingRefundableAmount().compareTo(BigDecimal.ZERO) == 0) {
            txn.applyStatus(TransactionStatus.REFUNDED, new RefundedState());
        }
    }
}
```

```java
// com/payment/gateway/state/FailedState.java
package com.payment.gateway.state;

import com.payment.gateway.exception.InvalidStateTransitionException;
import com.payment.gateway.model.Transaction;
import com.payment.gateway.model.TransactionStatus;
import java.math.BigDecimal;

public final class FailedState implements TransactionState {

    @Override
    public void startProcessing(Transaction txn) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.FAILED, TransactionStatus.PROCESSING);
    }

    @Override
    public void markSuccess(Transaction txn, String providerRef) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.FAILED, TransactionStatus.SUCCESS);
    }

    @Override
    public void markFailed(Transaction txn, String code, String message) {
        // Idempotent: already failed, ignore duplicate failure signals
    }

    @Override
    public void markRefunded(Transaction txn, BigDecimal refundAmount) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.FAILED, TransactionStatus.REFUNDED);
    }
}
```

```java
// com/payment/gateway/state/RefundedState.java
package com.payment.gateway.state;

import com.payment.gateway.exception.InvalidStateTransitionException;
import com.payment.gateway.model.Transaction;
import com.payment.gateway.model.TransactionStatus;
import java.math.BigDecimal;

public final class RefundedState implements TransactionState {

    @Override
    public void startProcessing(Transaction txn) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.REFUNDED, TransactionStatus.PROCESSING);
    }

    @Override
    public void markSuccess(Transaction txn, String providerRef) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.REFUNDED, TransactionStatus.SUCCESS);
    }

    @Override
    public void markFailed(Transaction txn, String code, String message) {
        throw new InvalidStateTransitionException(txn.getId(),
                TransactionStatus.REFUNDED, TransactionStatus.FAILED);
    }

    @Override
    public void markRefunded(Transaction txn, BigDecimal refundAmount) {
        throw new com.payment.gateway.exception.PaymentException(
                "ALREADY_REFUNDED",
                "Transaction " + txn.getId() + " is already fully refunded");
    }
}
```

### 6.4 Provider Strategy

```java
// com/payment/gateway/provider/PaymentProviderStrategy.java
package com.payment.gateway.provider;

import com.payment.gateway.model.*;

/**
 * Strategy interface — each payment provider implements this.
 * Switching providers requires no change to calling code.
 */
public interface PaymentProviderStrategy {

    ProviderType getProviderType();

    /**
     * Charge the customer.
     * @return provider-assigned reference ID
     * @throws com.payment.gateway.exception.ProviderException on upstream failure
     */
    String charge(Transaction transaction);

    /**
     * Refund a previously-charged transaction.
     * @param transaction  the original transaction
     * @param refundAmount amount to refund (null = full)
     * @return provider-assigned refund reference ID
     */
    String refund(Transaction transaction, java.math.BigDecimal refundAmount);

    /**
     * Verify that the incoming webhook payload is authentic (HMAC / signature check).
     */
    boolean verifyWebhookSignature(WebhookPayload payload);

    /**
     * Whether this provider supports the given payment method.
     */
    boolean supports(PaymentMethod method);
}
```

```java
// com/payment/gateway/provider/StripePaymentProvider.java
package com.payment.gateway.provider;

import com.payment.gateway.exception.ProviderException;
import com.payment.gateway.model.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.math.BigDecimal;
import java.nio.charset.StandardCharsets;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.util.EnumSet;
import java.util.Set;
import java.util.UUID;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

public final class StripePaymentProvider implements PaymentProviderStrategy {

    private static final Logger log = LoggerFactory.getLogger(StripePaymentProvider.class);
    private static final Set<PaymentMethod> SUPPORTED_METHODS = EnumSet.of(
            PaymentMethod.CREDIT_CARD, PaymentMethod.DEBIT_CARD, PaymentMethod.WALLET);

    private final String apiKey;
    private final String webhookSecret;

    public StripePaymentProvider(String apiKey, String webhookSecret) {
        this.apiKey        = apiKey;
        this.webhookSecret = webhookSecret;
    }

    @Override
    public ProviderType getProviderType() { return ProviderType.STRIPE; }

    @Override
    public String charge(Transaction transaction) {
        log.info("Stripe charge: txn={} amount={} {}",
                transaction.getId(), transaction.getAmount(), transaction.getCurrency());
        try {
            // In production: call stripe-java SDK
            // PaymentIntentCreateParams params = PaymentIntentCreateParams.builder()
            //     .setAmount(transaction.getAmount().multiply(BigDecimal.valueOf(100)).longValue())
            //     .setCurrency(transaction.getCurrency().getCurrencyCode().toLowerCase())
            //     .setPaymentMethod(transaction.getPaymentToken())
            //     .setConfirm(true)
            //     .build();
            // PaymentIntent intent = PaymentIntent.create(params);
            // return intent.getId();

            // Simulated for this design document
            String providerRef = "pi_stripe_" + UUID.randomUUID().toString().replace("-", "");
            log.info("Stripe charge success: providerRef={}", providerRef);
            return providerRef;
        } catch (Exception e) {
            throw new ProviderException(ProviderType.STRIPE, "STRIPE_CHARGE_FAILED",
                    "Stripe charge failed: " + e.getMessage(), e);
        }
    }

    @Override
    public String refund(Transaction transaction, BigDecimal refundAmount) {
        log.info("Stripe refund: txn={} providerRef={} amount={}",
                transaction.getId(), transaction.getProviderReference(), refundAmount);
        try {
            // In production: call stripe-java SDK
            // RefundCreateParams params = RefundCreateParams.builder()
            //     .setPaymentIntent(transaction.getProviderReference())
            //     .setAmount(refundAmount.multiply(BigDecimal.valueOf(100)).longValue())
            //     .build();
            // Refund refund = Refund.create(params);
            // return refund.getId();

            String refundRef = "re_stripe_" + UUID.randomUUID().toString().replace("-", "");
            log.info("Stripe refund success: refundRef={}", refundRef);
            return refundRef;
        } catch (Exception e) {
            throw new ProviderException(ProviderType.STRIPE, "STRIPE_REFUND_FAILED",
                    "Stripe refund failed: " + e.getMessage(), e);
        }
    }

    @Override
    public boolean verifyWebhookSignature(WebhookPayload payload) {
        try {
            String sigHeader = payload.rawHeaders().get("Stripe-Signature");
            if (sigHeader == null) return false;

            // Parse t= and v1= from Stripe-Signature header
            String timestamp = null;
            String signature = null;
            for (String part : sigHeader.split(",")) {
                if (part.startsWith("t="))  timestamp = part.substring(2);
                if (part.startsWith("v1=")) signature = part.substring(3);
            }
            if (timestamp == null || signature == null) return false;

            String signedPayload = timestamp + "." + payload.rawBody();
            String expected = computeHmacSha256(webhookSecret, signedPayload);
            return expected.equals(signature);
        } catch (Exception e) {
            log.warn("Stripe webhook signature verification failed", e);
            return false;
        }
    }

    @Override
    public boolean supports(PaymentMethod method) {
        return SUPPORTED_METHODS.contains(method);
    }

    private String computeHmacSha256(String key, String data)
            throws NoSuchAlgorithmException, InvalidKeyException {
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
        byte[] hash = mac.doFinal(data.getBytes(StandardCharsets.UTF_8));
        StringBuilder sb = new StringBuilder();
        for (byte b : hash) sb.append(String.format("%02x", b));
        return sb.toString();
    }
}
```

```java
// com/payment/gateway/provider/PayPalPaymentProvider.java
package com.payment.gateway.provider;

import com.payment.gateway.exception.ProviderException;
import com.payment.gateway.model.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.math.BigDecimal;
import java.util.EnumSet;
import java.util.Set;
import java.util.UUID;

public final class PayPalPaymentProvider implements PaymentProviderStrategy {

    private static final Logger log = LoggerFactory.getLogger(PayPalPaymentProvider.class);
    private static final Set<PaymentMethod> SUPPORTED_METHODS = EnumSet.of(
            PaymentMethod.CREDIT_CARD, PaymentMethod.DEBIT_CARD,
            PaymentMethod.WALLET, PaymentMethod.NET_BANKING);

    private final String clientId;
    private final String clientSecret;
    private final String webhookId;

    public PayPalPaymentProvider(String clientId, String clientSecret, String webhookId) {
        this.clientId     = clientId;
        this.clientSecret = clientSecret;
        this.webhookId    = webhookId;
    }

    @Override
    public ProviderType getProviderType() { return ProviderType.PAYPAL; }

    @Override
    public String charge(Transaction transaction) {
        log.info("PayPal charge: txn={} amount={} {}",
                transaction.getId(), transaction.getAmount(), transaction.getCurrency());
        try {
            // In production: use PayPal Orders v2 API
            // POST /v2/checkout/orders  with capture=immediate
            String providerRef = "PAYPAL-" + UUID.randomUUID().toString().toUpperCase();
            log.info("PayPal charge success: providerRef={}", providerRef);
            return providerRef;
        } catch (Exception e) {
            throw new ProviderException(ProviderType.PAYPAL, "PAYPAL_CHARGE_FAILED",
                    "PayPal charge failed: " + e.getMessage(), e);
        }
    }

    @Override
    public String refund(Transaction transaction, BigDecimal refundAmount) {
        log.info("PayPal refund: txn={} amount={}", transaction.getId(), refundAmount);
        try {
            // In production: POST /v2/payments/captures/{captureId}/refund
            String refundRef = "PAYPAL-REFUND-" + UUID.randomUUID().toString().toUpperCase();
            log.info("PayPal refund success: refundRef={}", refundRef);
            return refundRef;
        } catch (Exception e) {
            throw new ProviderException(ProviderType.PAYPAL, "PAYPAL_REFUND_FAILED",
                    "PayPal refund failed: " + e.getMessage(), e);
        }
    }

    @Override
    public boolean verifyWebhookSignature(WebhookPayload payload) {
        // PayPal recommends calling their verify-webhook-signature API endpoint
        // POST /v1/notifications/verify-webhook-signature
        // For brevity we check for the presence of standard PayPal headers
        String certUrl   = payload.rawHeaders().get("PAYPAL-CERT-URL");
        String transmId  = payload.rawHeaders().get("PAYPAL-TRANSMISSION-ID");
        String transmSig = payload.rawHeaders().get("PAYPAL-TRANSMISSION-SIG");
        boolean headersPresent = certUrl != null && transmId != null && transmSig != null;
        if (!headersPresent) {
            log.warn("PayPal webhook missing required headers");
            return false;
        }
        // In production: make the verification API call with the webhookId
        return true;
    }

    @Override
    public boolean supports(PaymentMethod method) {
        return SUPPORTED_METHODS.contains(method);
    }
}
```

```java
// com/payment/gateway/provider/RazorpayPaymentProvider.java
package com.payment.gateway.provider;

import com.payment.gateway.exception.ProviderException;
import com.payment.gateway.model.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.math.BigDecimal;
import java.nio.charset.StandardCharsets;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.util.EnumSet;
import java.util.Set;
import java.util.UUID;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

public final class RazorpayPaymentProvider implements PaymentProviderStrategy {

    private static final Logger log = LoggerFactory.getLogger(RazorpayPaymentProvider.class);
    private static final Set<PaymentMethod> SUPPORTED_METHODS = EnumSet.allOf(PaymentMethod.class);

    private final String keyId;
    private final String keySecret;

    public RazorpayPaymentProvider(String keyId, String keySecret) {
        this.keyId     = keyId;
        this.keySecret = keySecret;
    }

    @Override
    public ProviderType getProviderType() { return ProviderType.RAZORPAY; }

    @Override
    public String charge(Transaction transaction) {
        log.info("Razorpay charge: txn={} method={} amount={}",
                transaction.getId(), transaction.getPaymentMethod(), transaction.getAmount());
        try {
            // In production: use Razorpay Java SDK
            // JSONObject orderRequest = new JSONObject();
            // orderRequest.put("amount", amountInPaise);
            // orderRequest.put("currency", "INR");
            // orderRequest.put("payment_capture", 1);
            // Order order = razorpayClient.orders.create(orderRequest);
            String providerRef = "pay_rzp_" + UUID.randomUUID().toString().replace("-", "");
            log.info("Razorpay charge success: providerRef={}", providerRef);
            return providerRef;
        } catch (Exception e) {
            throw new ProviderException(ProviderType.RAZORPAY, "RAZORPAY_CHARGE_FAILED",
                    "Razorpay charge failed: " + e.getMessage(), e);
        }
    }

    @Override
    public String refund(Transaction transaction, BigDecimal refundAmount) {
        log.info("Razorpay refund: txn={} amount={}", transaction.getId(), refundAmount);
        try {
            // In production: POST /v1/payments/{paymentId}/refund
            String refundRef = "rfnd_rzp_" + UUID.randomUUID().toString().replace("-", "");
            log.info("Razorpay refund success: refundRef={}", refundRef);
            return refundRef;
        } catch (Exception e) {
            throw new ProviderException(ProviderType.RAZORPAY, "RAZORPAY_REFUND_FAILED",
                    "Razorpay refund failed: " + e.getMessage(), e);
        }
    }

    @Override
    public boolean verifyWebhookSignature(WebhookPayload payload) {
        try {
            String receivedSig = payload.rawHeaders().get("X-Razorpay-Signature");
            if (receivedSig == null) return false;
            String expected = computeHmacSha256(keySecret, payload.rawBody());
            return expected.equals(receivedSig);
        } catch (Exception e) {
            log.warn("Razorpay webhook signature verification failed", e);
            return false;
        }
    }

    @Override
    public boolean supports(PaymentMethod method) {
        return SUPPORTED_METHODS.contains(method);
    }

    private String computeHmacSha256(String key, String data)
            throws NoSuchAlgorithmException, InvalidKeyException {
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
        byte[] hash = mac.doFinal(data.getBytes(StandardCharsets.UTF_8));
        StringBuilder sb = new StringBuilder();
        for (byte b : hash) sb.append(String.format("%02x", b));
        return sb.toString();
    }
}
```

```java
// com/payment/gateway/provider/PaymentProviderRegistry.java
package com.payment.gateway.provider;

import com.payment.gateway.model.PaymentMethod;

import java.util.*;

/**
 * Registry holding all registered providers.
 * Provider selection uses a simple rule: prefer hint, then find first
 * provider supporting the requested PaymentMethod.
 */
public final class PaymentProviderRegistry {

    private final Map<ProviderType, PaymentProviderStrategy> providers = new EnumMap<>(ProviderType.class);

    public PaymentProviderRegistry(List<PaymentProviderStrategy> providerList) {
        for (PaymentProviderStrategy p : providerList) {
            providers.put(p.getProviderType(), p);
        }
    }

    public PaymentProviderStrategy getProvider(ProviderType type) {
        PaymentProviderStrategy p = providers.get(type);
        if (p == null) throw new IllegalArgumentException("No provider registered: " + type);
        return p;
    }

    /**
     * Select provider: use hint if present and supports the method;
     * otherwise pick first registered provider that supports the method.
     */
    public PaymentProviderStrategy select(PaymentMethod method,
                                          com.payment.gateway.model.PaymentRequest.ProviderHint hint) {
        if (hint != null) {
            ProviderType hintType = switch (hint) {
                case STRIPE   -> ProviderType.STRIPE;
                case PAYPAL   -> ProviderType.PAYPAL;
                case RAZORPAY -> ProviderType.RAZORPAY;
            };
            PaymentProviderStrategy candidate = providers.get(hintType);
            if (candidate != null && candidate.supports(method)) return candidate;
        }
        return providers.values().stream()
                .filter(p -> p.supports(method))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException(
                        "No provider supports payment method: " + method));
    }

    public Collection<PaymentProviderStrategy> allProviders() {
        return Collections.unmodifiableCollection(providers.values());
    }
}
```

### 6.5 Fraud Detection — Chain of Responsibility

```java
// com/payment/gateway/fraud/FraudCheckRequest.java
package com.payment.gateway.fraud;

import com.payment.gateway.model.PaymentMethod;
import com.payment.gateway.model.PaymentRequest;
import java.math.BigDecimal;

public record FraudCheckRequest(
        String        customerId,
        String        merchantId,
        BigDecimal    amount,
        String        currency,
        PaymentMethod paymentMethod,
        String        ipAddress,
        String        deviceFingerprint
) {
    public static FraudCheckRequest from(PaymentRequest req, String ipAddress,
                                         String deviceFingerprint) {
        return new FraudCheckRequest(
                req.getCustomerId(), req.getMerchantId(), req.getAmount(),
                req.getCurrency().getCurrencyCode(), req.getPaymentMethod(),
                ipAddress, deviceFingerprint);
    }
}
```

```java
// com/payment/gateway/fraud/FraudCheckResult.java
package com.payment.gateway.fraud;

public record FraudCheckResult(
        boolean passed,
        String  ruleName,
        String  reason,
        double  riskScore   // 0.0–1.0
) {
    public static FraudCheckResult pass() {
        return new FraudCheckResult(true, null, null, 0.0);
    }

    public static FraudCheckResult block(String ruleName, String reason, double riskScore) {
        return new FraudCheckResult(false, ruleName, reason, riskScore);
    }
}
```

```java
// com/payment/gateway/fraud/FraudCheckHandler.java
package com.payment.gateway.fraud;

/**
 * Chain of Responsibility node for fraud checks.
 */
public abstract class FraudCheckHandler {

    private FraudCheckHandler next;

    public FraudCheckHandler setNext(FraudCheckHandler next) {
        this.next = next;
        return next;
    }

    /**
     * Execute this handler's check; if it passes, delegate to the next handler.
     */
    public final FraudCheckResult handle(FraudCheckRequest request) {
        FraudCheckResult result = check(request);
        if (!result.passed()) return result;
        if (next != null) return next.handle(request);
        return FraudCheckResult.pass();
    }

    protected abstract FraudCheckResult check(FraudCheckRequest request);
}
```

```java
// com/payment/gateway/fraud/VelocityCheckHandler.java
package com.payment.gateway.fraud;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.time.Instant;
import java.util.Map;

/**
 * Blocks customers who exceed N transactions per minute (sliding window via in-memory counter).
 * In production this would be backed by Redis with TTL.
 */
public final class VelocityCheckHandler extends FraudCheckHandler {

    private static final int    MAX_TXN_PER_MINUTE = 10;
    private static final String RULE_NAME          = "VELOCITY_CHECK";

    // customerId -> (windowStart, count)
    private final Map<String, long[]> windowMap = new ConcurrentHashMap<>();

    @Override
    protected FraudCheckResult check(FraudCheckRequest request) {
        long now = Instant.now().toEpochMilli();
        long windowMs = 60_000L;

        long[] window = windowMap.compute(request.customerId(), (k, v) -> {
            if (v == null || (now - v[0]) > windowMs) return new long[]{now, 1};
            v[1]++;
            return v;
        });

        if (window[1] > MAX_TXN_PER_MINUTE) {
            return FraudCheckResult.block(RULE_NAME,
                    "Customer " + request.customerId() + " exceeded " + MAX_TXN_PER_MINUTE
                    + " transactions/minute (count=" + window[1] + ")", 0.9);
        }
        return FraudCheckResult.pass();
    }
}
```

```java
// com/payment/gateway/fraud/AmountLimitCheckHandler.java
package com.payment.gateway.fraud;

import java.math.BigDecimal;
import java.util.Map;

/**
 * Enforces per-merchant and global per-transaction amount limits.
 */
public final class AmountLimitCheckHandler extends FraudCheckHandler {

    private static final String    RULE_NAME            = "AMOUNT_LIMIT_CHECK";
    private static final BigDecimal GLOBAL_LIMIT        = new BigDecimal("100000.00");

    // merchantId -> max single-transaction limit
    private final Map<String, BigDecimal> merchantLimits;

    public AmountLimitCheckHandler(Map<String, BigDecimal> merchantLimits) {
        this.merchantLimits = Map.copyOf(merchantLimits);
    }

    @Override
    protected FraudCheckResult check(FraudCheckRequest request) {
        BigDecimal limit = merchantLimits.getOrDefault(request.merchantId(), GLOBAL_LIMIT);
        if (request.amount().compareTo(limit) > 0) {
            return FraudCheckResult.block(RULE_NAME,
                    "Transaction amount " + request.amount()
                    + " exceeds limit " + limit + " for merchant " + request.merchantId(), 0.7);
        }
        return FraudCheckResult.pass();
    }
}
```

```java
// com/payment/gateway/fraud/BlacklistCheckHandler.java
package com.payment.gateway.fraud;

import java.util.Set;

/**
 * Blocks transactions from blacklisted customers or merchants.
 * In production the blacklist is loaded from a DB / Redis set with periodic refresh.
 */
public final class BlacklistCheckHandler extends FraudCheckHandler {

    private static final String RULE_NAME = "BLACKLIST_CHECK";

    private final Set<String> blacklistedCustomers;
    private final Set<String> blacklistedMerchants;

    public BlacklistCheckHandler(Set<String> blacklistedCustomers,
                                 Set<String> blacklistedMerchants) {
        this.blacklistedCustomers = Set.copyOf(blacklistedCustomers);
        this.blacklistedMerchants = Set.copyOf(blacklistedMerchants);
    }

    @Override
    protected FraudCheckResult check(FraudCheckRequest request) {
        if (blacklistedCustomers.contains(request.customerId())) {
            return FraudCheckResult.block(RULE_NAME,
                    "Customer is blacklisted: " + request.customerId(), 1.0);
        }
        if (blacklistedMerchants.contains(request.merchantId())) {
            return FraudCheckResult.block(RULE_NAME,
                    "Merchant is blacklisted: " + request.merchantId(), 1.0);
        }
        return FraudCheckResult.pass();
    }
}
```

### 6.6 Events — Observer Pattern

```java
// com/payment/gateway/event/PaymentEvent.java
package com.payment.gateway.event;

import com.payment.gateway.model.TransactionStatus;
import java.math.BigDecimal;
import java.time.Instant;

public record PaymentEvent(
        String            eventId,
        String            eventType,       // PAYMENT_SUCCESS, PAYMENT_FAILED, REFUND_INITIATED, etc.
        String            transactionId,
        String            customerId,
        String            merchantId,
        BigDecimal        amount,
        String            currency,
        TransactionStatus status,
        String            errorCode,
        Instant           occurredAt
) {
    public static PaymentEvent of(String eventType, String transactionId,
                                  String customerId, String merchantId,
                                  BigDecimal amount, String currency,
                                  TransactionStatus status, String errorCode) {
        return new PaymentEvent(
                java.util.UUID.randomUUID().toString(),
                eventType, transactionId, customerId, merchantId,
                amount, currency, status, errorCode, Instant.now());
    }
}
```

```java
// com/payment/gateway/event/PaymentEventListener.java
package com.payment.gateway.event;

@FunctionalInterface
public interface PaymentEventListener {
    void onEvent(PaymentEvent event);
}
```

```java
// com/payment/gateway/event/PaymentEventPublisher.java
package com.payment.gateway.event;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * Thread-safe event publisher.
 * Listeners are notified asynchronously on a dedicated thread pool.
 * In production this would publish to Kafka; listeners consume from topics.
 */
public final class PaymentEventPublisher {

    private static final Logger log = LoggerFactory.getLogger(PaymentEventPublisher.class);

    private final List<PaymentEventListener> listeners = new CopyOnWriteArrayList<>();
    private final ExecutorService            executor;

    public PaymentEventPublisher(int listenerThreads) {
        this.executor = Executors.newFixedThreadPool(listenerThreads,
                Thread.ofVirtual().name("event-publisher-", 0).factory());
    }

    public void register(PaymentEventListener listener) {
        listeners.add(listener);
    }

    public void publish(PaymentEvent event) {
        log.debug("Publishing event: {} txn={}", event.eventType(), event.transactionId());
        for (PaymentEventListener listener : listeners) {
            executor.submit(() -> {
                try {
                    listener.onEvent(event);
                } catch (Exception e) {
                    log.error("Listener {} threw exception for event {}",
                            listener.getClass().getSimpleName(), event.eventId(), e);
                }
            });
        }
    }

    public void shutdown() {
        executor.shutdown();
    }
}
```

### 6.7 Core Services

```java
// com/payment/gateway/service/IdempotencyService.java
package com.payment.gateway.service;

import com.payment.gateway.exception.DuplicatePaymentException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Prevents duplicate charges for the same idempotency key.
 * Backed by Redis in production (key → txnId, TTL = 24 h).
 * This in-process implementation is suitable for single-node or test scenarios.
 */
public final class IdempotencyService {

    private static final Logger   log    = LoggerFactory.getLogger(IdempotencyService.class);
    private static final Duration WINDOW = Duration.ofHours(24);

    // key → [transactionId, expiryEpochMs]
    private final ConcurrentHashMap<String, String[]> store = new ConcurrentHashMap<>();

    /**
     * Assert uniqueness. Throws DuplicatePaymentException if key was seen before.
     */
    public void assertUnique(String idempotencyKey) {
        String[] entry = store.get(idempotencyKey);
        if (entry != null) {
            long expiry = Long.parseLong(entry[1]);
            if (Instant.now().toEpochMilli() < expiry) {
                throw new DuplicatePaymentException(idempotencyKey, entry[0]);
            }
            // Expired — evict and allow re-use
            store.remove(idempotencyKey);
        }
    }

    /**
     * Record that this key has been used.
     */
    public void record(String idempotencyKey, String transactionId) {
        long expiry = Instant.now().plus(WINDOW).toEpochMilli();
        store.put(idempotencyKey, new String[]{transactionId, String.valueOf(expiry)});
        log.debug("Idempotency key recorded: key={} txn={}", idempotencyKey, transactionId);
    }

    /**
     * Periodic cleanup of expired entries (call from a scheduler).
     */
    public void evictExpired() {
        long now = Instant.now().toEpochMilli();
        store.entrySet().removeIf(e -> Long.parseLong(e.getValue()[1]) < now);
    }
}
```

```java
// com/payment/gateway/service/TokenisationService.java
package com.payment.gateway.service;

import com.payment.gateway.model.CardData;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import java.security.SecureRandom;
import java.util.Base64;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Tokenises raw card data into an opaque token.
 * The token maps to an AES-GCM–encrypted blob stored in a separate vault DB.
 *
 * PCI note: in production, this service runs in an isolated network segment;
 * only the vault service holds the encryption key.
 */
public final class TokenisationService {

    private static final Logger log        = LoggerFactory.getLogger(TokenisationService.class);
    private static final int    GCM_TAG    = 128;
    private static final int    GCM_IV_LEN = 12;

    private final SecretKey masterKey;
    private final SecureRandom rng = new SecureRandom();

    // token → ciphertext (in production: Vault / HSM-backed DB)
    private final Map<String, String> tokenStore = new ConcurrentHashMap<>();

    public TokenisationService() {
        try {
            KeyGenerator kg = KeyGenerator.getInstance("AES");
            kg.init(256, rng);
            this.masterKey = kg.generateKey();
        } catch (Exception e) {
            throw new ExceptionInInitializerError("Failed to initialise tokenisation key: " + e);
        }
    }

    /**
     * Tokenise card data. Raw PAN is zeroed after encryption.
     * @return opaque token (safe to store anywhere)
     */
    public String tokenise(CardData card) {
        try {
            String pan = new String(card.getPan());
            byte[] iv  = new byte[GCM_IV_LEN];
            rng.nextBytes(iv);

            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.ENCRYPT_MODE, masterKey, new GCMParameterSpec(GCM_TAG, iv));
            byte[] cipherText = cipher.doFinal(pan.getBytes());

            byte[] ivAndCipher = new byte[iv.length + cipherText.length];
            System.arraycopy(iv, 0, ivAndCipher, 0, iv.length);
            System.arraycopy(cipherText, 0, ivAndCipher, iv.length, cipherText.length);

            String token = "tok_" + UUID.randomUUID().toString().replace("-", "");
            tokenStore.put(token, Base64.getEncoder().encodeToString(ivAndCipher));

            card.destroy(); // zero raw PAN
            log.info("Card tokenised successfully, token={}", token);
            return token;
        } catch (Exception e) {
            card.destroy();
            throw new com.payment.gateway.exception.PaymentException(
                    "TOKENISATION_FAILED", "Failed to tokenise card: " + e.getMessage(), e);
        }
    }

    /**
     * Detokenise — used only by the charge path inside the gateway's trusted zone.
     * Never expose raw PAN outside of this method.
     */
    public String detokenise(String token) {
        String encoded = tokenStore.get(token);
        if (encoded == null) {
            throw new com.payment.gateway.exception.PaymentException(
                    "TOKEN_NOT_FOUND", "Token not found: " + token);
        }
        try {
            byte[] ivAndCipher = Base64.getDecoder().decode(encoded);
            byte[] iv          = new byte[GCM_IV_LEN];
            byte[] cipherText  = new byte[ivAndCipher.length - GCM_IV_LEN];
            System.arraycopy(ivAndCipher, 0, iv, 0, GCM_IV_LEN);
            System.arraycopy(ivAndCipher, GCM_IV_LEN, cipherText, 0, cipherText.length);

            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.DECRYPT_MODE, masterKey, new GCMParameterSpec(GCM_TAG, iv));
            return new String(cipher.doFinal(cipherText));
        } catch (Exception e) {
            throw new com.payment.gateway.exception.PaymentException(
                    "DETOKENISATION_FAILED", "Failed to detokenise: " + e.getMessage(), e);
        }
    }
}
```

```java
// com/payment/gateway/service/AuditLogService.java
package com.payment.gateway.service;

import com.payment.gateway.model.Transaction;
import com.payment.gateway.model.TransactionStatus;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Immutable audit trail for every state transition.
 * In production: append-only table in PostgreSQL / Cassandra with no UPDATE/DELETE grants.
 */
public final class AuditLogService {

    private static final Logger log = LoggerFactory.getLogger(AuditLogService.class);

    public record AuditEntry(
            String            entryId,
            String            transactionId,
            TransactionStatus fromStatus,
            TransactionStatus toStatus,
            String            actor,
            String            reason,
            Instant           timestamp
    ) {}

    // In production: DB write-only; here we use an in-memory list for illustration
    private final List<AuditEntry> log_ = new CopyOnWriteArrayList<>();

    public void record(Transaction txn, TransactionStatus from,
                       TransactionStatus to, String actor, String reason) {
        AuditEntry entry = new AuditEntry(
                UUID.randomUUID().toString(),
                txn.getId(), from, to, actor, reason, Instant.now());
        log_.add(entry);
        log.info("AUDIT txn={} {}→{} actor={} reason={}", txn.getId(), from, to, actor, reason);
    }

    public List<AuditEntry> getEntriesForTransaction(String transactionId) {
        return log_.stream()
                .filter(e -> e.transactionId().equals(transactionId))
                .sorted(Comparator.comparing(AuditEntry::timestamp))
                .toList();
    }
}
```

```java
// com/payment/gateway/service/PaymentProcessor.java
package com.payment.gateway.service;

import com.payment.gateway.event.PaymentEvent;
import com.payment.gateway.event.PaymentEventPublisher;
import com.payment.gateway.exception.FraudDetectedException;
import com.payment.gateway.exception.ProviderException;
import com.payment.gateway.fraud.*;
import com.payment.gateway.model.*;
import com.payment.gateway.provider.*;
import com.payment.gateway.state.InitiatedState;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.math.BigDecimal;
import java.util.Currency;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Orchestrates the end-to-end payment flow:
 *  1. Fraud checks (CoR)
 *  2. Persist INITIATED transaction
 *  3. Transition to PROCESSING
 *  4. Call provider (Strategy)
 *  5. Transition to SUCCESS / FAILED
 *  6. Publish event (Observer)
 */
public final class PaymentProcessor {

    private static final Logger log = LoggerFactory.getLogger(PaymentProcessor.class);

    private final PaymentProviderRegistry providerRegistry;
    private final FraudCheckHandler       fraudChain;
    private final IdempotencyService      idempotencyService;
    private final AuditLogService         auditLogService;
    private final PaymentEventPublisher   eventPublisher;

    // In production: backed by DB (JPA/JDBC)
    private final Map<String, Transaction> transactionStore = new ConcurrentHashMap<>();

    public PaymentProcessor(PaymentProviderRegistry providerRegistry,
                            FraudCheckHandler fraudChain,
                            IdempotencyService idempotencyService,
                            AuditLogService auditLogService,
                            PaymentEventPublisher eventPublisher) {
        this.providerRegistry  = providerRegistry;
        this.fraudChain        = fraudChain;
        this.idempotencyService = idempotencyService;
        this.auditLogService   = auditLogService;
        this.eventPublisher    = eventPublisher;
    }

    public PaymentResult process(PaymentRequest request,
                                 String ipAddress, String deviceFingerprint) {
        // Step 1: Idempotency check
        idempotencyService.assertUnique(request.getIdempotencyKey());

        // Step 2: Fraud check chain
        FraudCheckRequest fraudReq = FraudCheckRequest.from(request, ipAddress, deviceFingerprint);
        FraudCheckResult  fraudRes = fraudChain.handle(fraudReq);
        if (!fraudRes.passed()) {
            throw new FraudDetectedException(fraudRes.ruleName(), fraudRes.reason());
        }

        // Step 3: Select provider
        PaymentProviderStrategy provider = providerRegistry.select(
                request.getPaymentMethod(), request.getProviderHint());

        // Step 4: Create transaction in INITIATED state
        Transaction txn = new Transaction(
                request.getIdempotencyKey(),
                request.getMerchantId(),
                request.getCustomerId(),
                request.getAmount(),
                request.getCurrency(),
                request.getPaymentMethod(),
                request.getPaymentToken(),
                provider.getProviderType(),
                new InitiatedState());

        transactionStore.put(txn.getId(), txn);
        idempotencyService.record(request.getIdempotencyKey(), txn.getId());
        auditLogService.record(txn, null, TransactionStatus.INITIATED, "SYSTEM", "Payment initiated");

        // Step 5: Transition to PROCESSING
        TransactionStatus prev = txn.getStatus();
        txn.startProcessing();
        auditLogService.record(txn, prev, TransactionStatus.PROCESSING, "SYSTEM", "Sent to provider");

        // Step 6: Call provider
        try {
            String providerRef = provider.charge(txn);
            prev = txn.getStatus();
            txn.markSuccess(providerRef);
            auditLogService.record(txn, prev, TransactionStatus.SUCCESS, "PROVIDER",
                    "Charge accepted: " + providerRef);

            PaymentEvent event = PaymentEvent.of("PAYMENT_SUCCESS", txn.getId(),
                    txn.getCustomerId(), txn.getMerchantId(), txn.getAmount(),
                    txn.getCurrency().getCurrencyCode(), TransactionStatus.SUCCESS, null);
            eventPublisher.publish(event);

            return PaymentResult.success(txn.getId(), txn.getAmount(),
                    txn.getCurrency().getCurrencyCode(), providerRef);

        } catch (ProviderException e) {
            log.error("Provider charge failed for txn={}: {}", txn.getId(), e.getMessage());
            prev = txn.getStatus();
            txn.markFailed(e.getErrorCode(), e.getMessage());
            auditLogService.record(txn, prev, TransactionStatus.FAILED, "PROVIDER", e.getMessage());

            PaymentEvent event = PaymentEvent.of("PAYMENT_FAILED", txn.getId(),
                    txn.getCustomerId(), txn.getMerchantId(), txn.getAmount(),
                    txn.getCurrency().getCurrencyCode(), TransactionStatus.FAILED, e.getErrorCode());
            eventPublisher.publish(event);

            return PaymentResult.failure(txn.getId(), txn.getAmount(),
                    txn.getCurrency().getCurrencyCode(), e.getErrorCode(), e.getMessage());
        }
    }

    public RefundResult refund(RefundRequest request) {
        Transaction txn = transactionStore.get(request.getOriginalTransactionId());
        if (txn == null) {
            throw new com.payment.gateway.exception.PaymentException(
                    "TRANSACTION_NOT_FOUND",
                    "Transaction not found: " + request.getOriginalTransactionId());
        }

        // Idempotency for refund requests
        idempotencyService.assertUnique(request.getIdempotencyKey());

        BigDecimal refundAmount = request.isFullRefund()
                ? txn.getRemainingRefundableAmount()
                : request.getRefundAmount();

        PaymentProviderStrategy provider = providerRegistry.getProvider(txn.getProvider());

        try {
            String refundRef = provider.refund(txn, refundAmount);
            TransactionStatus prev = txn.getStatus();
            txn.markRefunded(refundAmount);
            auditLogService.record(txn, prev, txn.getStatus(), request.getRequestedBy(),
                    "Refund processed: " + refundRef + " amount=" + refundAmount);

            idempotencyService.record(request.getIdempotencyKey(), txn.getId());

            PaymentEvent event = PaymentEvent.of("REFUND_PROCESSED", txn.getId(),
                    txn.getCustomerId(), txn.getMerchantId(), refundAmount,
                    txn.getCurrency().getCurrencyCode(), txn.getStatus(), null);
            eventPublisher.publish(event);

            return new RefundResult(refundRef, txn.getId(), refundAmount,
                    true, null, java.time.Instant.now());

        } catch (ProviderException e) {
            log.error("Refund failed for txn={}: {}", txn.getId(), e.getMessage());
            return new RefundResult(null, txn.getId(), refundAmount,
                    false, e.getMessage(), java.time.Instant.now());
        }
    }

    public Transaction getTransaction(String transactionId) {
        Transaction txn = transactionStore.get(transactionId);
        if (txn == null) {
            throw new com.payment.gateway.exception.PaymentException(
                    "TRANSACTION_NOT_FOUND", "Transaction not found: " + transactionId);
        }
        return txn;
    }

    public Map<String, Transaction> getAllTransactions() {
        return java.util.Collections.unmodifiableMap(transactionStore);
    }
}
```

```java
// com/payment/gateway/service/WebhookProcessor.java
package com.payment.gateway.service;

import com.payment.gateway.event.PaymentEvent;
import com.payment.gateway.event.PaymentEventPublisher;
import com.payment.gateway.model.*;
import com.payment.gateway.provider.PaymentProviderRegistry;
import com.payment.gateway.provider.PaymentProviderStrategy;
import com.payment.gateway.provider.ProviderType;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Processes inbound webhooks from payment providers.
 * Responsibilities:
 *  - Verify signature (delegates to the provider)
 *  - Locate the transaction by providerReference
 *  - Apply the status update
 *  - Publish a domain event
 *
 * Designed to be idempotent: replaying the same webhook is safe.
 */
public final class WebhookProcessor {

    private static final Logger log = LoggerFactory.getLogger(WebhookProcessor.class);

    private final PaymentProcessor        paymentProcessor;
    private final PaymentProviderRegistry providerRegistry;
    private final AuditLogService         auditLogService;
    private final PaymentEventPublisher   eventPublisher;

    // providerReference → transactionId (in production: DB index)
    private final Map<String, String> providerRefIndex = new ConcurrentHashMap<>();

    public WebhookProcessor(PaymentProcessor paymentProcessor,
                            PaymentProviderRegistry providerRegistry,
                            AuditLogService auditLogService,
                            PaymentEventPublisher eventPublisher) {
        this.paymentProcessor  = paymentProcessor;
        this.providerRegistry  = providerRegistry;
        this.auditLogService   = auditLogService;
        this.eventPublisher    = eventPublisher;
    }

    public void registerProviderReference(String providerRef, String transactionId) {
        providerRefIndex.put(providerRef, transactionId);
    }

    public void process(ProviderType providerType, WebhookPayload payload) {
        PaymentProviderStrategy provider = providerRegistry.getProvider(providerType);

        // 1. Verify authenticity
        if (!provider.verifyWebhookSignature(payload)) {
            log.warn("Rejected webhook from {} — invalid signature. eventType={}",
                    providerType, payload.eventType());
            throw new com.payment.gateway.exception.PaymentException(
                    "INVALID_WEBHOOK_SIGNATURE",
                    "Webhook signature verification failed for provider: " + providerType);
        }

        // 2. Look up the transaction
        String transactionId = providerRefIndex.get(payload.providerReference());
        if (transactionId == null) {
            log.warn("No transaction found for providerReference={}", payload.providerReference());
            // Return 200 to provider to avoid retries; log for investigation
            return;
        }

        Transaction txn = paymentProcessor.getTransaction(transactionId);
        TransactionStatus currentStatus = txn.getStatus();
        TransactionStatus targetStatus  = payload.reportedStatus();

        // 3. Idempotent: ignore webhook if already in target state
        if (currentStatus == targetStatus) {
            log.debug("Webhook idempotent — txn={} already in status {}", transactionId, currentStatus);
            return;
        }

        // 4. Apply the transition
        try {
            TransactionStatus prev = txn.getStatus();
            switch (targetStatus) {
                case SUCCESS  -> txn.markSuccess(payload.providerReference());
                case FAILED   -> txn.markFailed("PROVIDER_ASYNC_FAILED",
                        "Provider reported failure via webhook");
                default       -> log.warn("Unhandled webhook target status: {}", targetStatus);
            }
            auditLogService.record(txn, prev, txn.getStatus(), "WEBHOOK:" + providerType,
                    "Async status update from provider webhook");

            // 5. Publish event
            String eventType = targetStatus == TransactionStatus.SUCCESS
                    ? "PAYMENT_SUCCESS" : "PAYMENT_FAILED";
            PaymentEvent event = PaymentEvent.of(eventType, txn.getId(),
                    txn.getCustomerId(), txn.getMerchantId(), txn.getAmount(),
                    txn.getCurrency().getCurrencyCode(), txn.getStatus(), txn.getFailureCode());
            eventPublisher.publish(event);

        } catch (com.payment.gateway.exception.InvalidStateTransitionException e) {
            log.error("Webhook triggered invalid state transition for txn={}: {}",
                    transactionId, e.getMessage());
        }
    }
}
```

```java
// com/payment/gateway/service/SettlementService.java
package com.payment.gateway.service;

import com.payment.gateway.model.Transaction;
import com.payment.gateway.model.TransactionStatus;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.ZoneOffset;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Daily settlement reconciliation.
 * Produces a per-merchant summary of settled and refunded amounts.
 *
 * In production:
 * - Triggered by a cron job at end of business day
 * - Cross-referenced with provider settlement files (CSV/SFTP)
 * - Discrepancies trigger alerts and manual review
 */
public final class SettlementService {

    private static final Logger log = LoggerFactory.getLogger(SettlementService.class);

    public record MerchantSettlementSummary(
            String     merchantId,
            LocalDate  settlementDate,
            int        successCount,
            int        refundCount,
            BigDecimal grossAmount,
            BigDecimal refundedAmount,
            BigDecimal netAmount,
            String     currency
    ) {}

    public record SettlementReport(
            LocalDate                       reportDate,
            List<MerchantSettlementSummary> summaries,
            int                             totalTransactions,
            BigDecimal                      totalGross,
            BigDecimal                      totalRefunded,
            BigDecimal                      totalNet
    ) {}

    private final PaymentProcessor paymentProcessor;

    public SettlementService(PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }

    public SettlementReport reconcile(LocalDate date) {
        log.info("Starting settlement reconciliation for date={}", date);

        long dayStartMs = date.atStartOfDay(ZoneOffset.UTC).toInstant().toEpochMilli();
        long dayEndMs   = date.plusDays(1).atStartOfDay(ZoneOffset.UTC).toInstant().toEpochMilli();

        List<Transaction> dayTransactions = paymentProcessor.getAllTransactions().values().stream()
                .filter(t -> {
                    long ts = t.getCreatedAt().toEpochMilli();
                    return ts >= dayStartMs && ts < dayEndMs;
                })
                .toList();

        // Group by merchantId
        Map<String, List<Transaction>> byMerchant = dayTransactions.stream()
                .collect(Collectors.groupingBy(Transaction::getMerchantId));

        List<MerchantSettlementSummary> summaries = new ArrayList<>();
        BigDecimal totalGross     = BigDecimal.ZERO;
        BigDecimal totalRefunded  = BigDecimal.ZERO;

        for (Map.Entry<String, List<Transaction>> entry : byMerchant.entrySet()) {
            String merchantId    = entry.getKey();
            List<Transaction> ts = entry.getValue();

            List<Transaction> successful = ts.stream()
                    .filter(t -> t.getStatus() == TransactionStatus.SUCCESS
                              || t.getStatus() == TransactionStatus.REFUNDED)
                    .toList();

            BigDecimal gross    = successful.stream()
                    .map(Transaction::getAmount)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

            BigDecimal refunded = successful.stream()
                    .map(Transaction::getRefundedAmount)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

            BigDecimal net = gross.subtract(refunded);

            String currency = ts.isEmpty() ? "USD"
                    : ts.get(0).getCurrency().getCurrencyCode();

            int refundCount = (int) ts.stream()
                    .filter(t -> t.getRefundedAmount().compareTo(BigDecimal.ZERO) > 0)
                    .count();

            summaries.add(new MerchantSettlementSummary(
                    merchantId, date, successful.size(), refundCount,
                    gross, refunded, net, currency));

            totalGross    = totalGross.add(gross);
            totalRefunded = totalRefunded.add(refunded);
        }

        BigDecimal totalNet = totalGross.subtract(totalRefunded);
        SettlementReport report = new SettlementReport(
                date, summaries, dayTransactions.size(), totalGross, totalRefunded, totalNet);

        log.info("Settlement complete for {}: txns={} gross={} refunded={} net={}",
                date, dayTransactions.size(), totalGross, totalRefunded, totalNet);
        return report;
    }
}
```

### 6.8 Facade — Client-Facing API

```java
// com/payment/gateway/api/PaymentGatewayFacade.java
package com.payment.gateway.api;

import com.payment.gateway.model.*;
import com.payment.gateway.provider.ProviderType;
import com.payment.gateway.service.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.LocalDate;

/**
 * Facade — single entry point for all payment operations.
 *
 * Clients interact only with this class; internal subsystems are hidden.
 * This decouples clients from implementation changes (e.g. swapping a provider,
 * changing the fraud chain) without any API-level impact.
 */
public final class PaymentGatewayFacade {

    private static final Logger log = LoggerFactory.getLogger(PaymentGatewayFacade.class);

    private final PaymentProcessor   paymentProcessor;
    private final WebhookProcessor   webhookProcessor;
    private final SettlementService  settlementService;
    private final TokenisationService tokenisationService;

    public PaymentGatewayFacade(PaymentProcessor paymentProcessor,
                                 WebhookProcessor webhookProcessor,
                                 SettlementService settlementService,
                                 TokenisationService tokenisationService) {
        this.paymentProcessor    = paymentProcessor;
        this.webhookProcessor    = webhookProcessor;
        this.settlementService   = settlementService;
        this.tokenisationService = tokenisationService;
    }

    /**
     * Initiate a payment. The caller must have already obtained a payment token
     * via {@link #tokeniseCard(CardData)} for card-based payments.
     */
    public PaymentResult initiatePayment(PaymentRequest request,
                                          String ipAddress,
                                          String deviceFingerprint) {
        log.info("initiatePayment: idempotencyKey={} merchant={} customer={} amount={} {}",
                request.getIdempotencyKey(), request.getMerchantId(), request.getCustomerId(),
                request.getAmount(), request.getCurrency());
        return paymentProcessor.process(request, ipAddress, deviceFingerprint);
    }

    /**
     * Tokenise raw card data. Should be called from the PCI-scoped SDK in the client.
     * Raw PAN is zeroed immediately after encryption.
     */
    public String tokeniseCard(CardData cardData) {
        return tokenisationService.tokenise(cardData);
    }

    /**
     * Request a full or partial refund.
     */
    public RefundResult refundPayment(RefundRequest request) {
        log.info("refundPayment: originalTxn={} amount={}",
                request.getOriginalTransactionId(), request.getRefundAmount());
        return paymentProcessor.refund(request);
    }

    /**
     * Retrieve transaction details.
     */
    public Transaction getTransaction(String transactionId) {
        return paymentProcessor.getTransaction(transactionId);
    }

    /**
     * Handle an inbound webhook from a payment provider.
     * The HTTP layer should pass raw headers and body for signature verification.
     */
    public void handleWebhook(ProviderType providerType, WebhookPayload payload) {
        log.info("handleWebhook: provider={} eventType={} providerRef={}",
                providerType, payload.eventType(), payload.providerReference());
        webhookProcessor.process(providerType, payload);
    }

    /**
     * Run daily settlement reconciliation.
     */
    public SettlementService.SettlementReport reconcile(LocalDate date) {
        return settlementService.reconcile(date);
    }
}
```

### 6.9 Wiring — Dependency Injection / Bootstrap

```java
// com/payment/gateway/PaymentGatewayBootstrap.java
package com.payment.gateway;

import com.payment.gateway.api.PaymentGatewayFacade;
import com.payment.gateway.event.PaymentEventPublisher;
import com.payment.gateway.fraud.*;
import com.payment.gateway.provider.*;
import com.payment.gateway.service.*;

import java.math.BigDecimal;
import java.util.*;

/**
 * Manual DI bootstrap — in production replace with Spring @Configuration.
 * Demonstrates how all components are assembled.
 */
public final class PaymentGatewayBootstrap {

    public static PaymentGatewayFacade build(GatewayConfig config) {

        // ── Providers ──────────────────────────────────────────────────────
        StripePaymentProvider stripe = new StripePaymentProvider(
                config.stripeApiKey(), config.stripeWebhookSecret());

        PayPalPaymentProvider paypal = new PayPalPaymentProvider(
                config.paypalClientId(), config.paypalClientSecret(), config.paypalWebhookId());

        RazorpayPaymentProvider razorpay = new RazorpayPaymentProvider(
                config.razorpayKeyId(), config.razorpayKeySecret());

        PaymentProviderRegistry registry = new PaymentProviderRegistry(
                List.of(stripe, paypal, razorpay));

        // ── Fraud chain ────────────────────────────────────────────────────
        Map<String, BigDecimal> merchantLimits = config.merchantTransactionLimits();
        Set<String> blacklistedCustomers = config.blacklistedCustomers();
        Set<String> blacklistedMerchants = config.blacklistedMerchants();

        BlacklistCheckHandler  blacklist  = new BlacklistCheckHandler(blacklistedCustomers, blacklistedMerchants);
        AmountLimitCheckHandler amountLimit = new AmountLimitCheckHandler(merchantLimits);
        VelocityCheckHandler   velocity   = new VelocityCheckHandler();

        // Chain: velocity → amount → blacklist
        velocity.setNext(amountLimit).setNext(blacklist);

        // ── Infrastructure services ────────────────────────────────────────
        IdempotencyService    idempotency = new IdempotencyService();
        AuditLogService       auditLog    = new AuditLogService();
        PaymentEventPublisher publisher   = new PaymentEventPublisher(4);
        TokenisationService   tokeniser   = new TokenisationService();

        // ── Core processor ─────────────────────────────────────────────────
        PaymentProcessor processor = new PaymentProcessor(
                registry, velocity, idempotency, auditLog, publisher);

        WebhookProcessor webhookProcessor = new WebhookProcessor(
                processor, registry, auditLog, publisher);

        SettlementService settlementService = new SettlementService(processor);

        // ── Register example event listeners (Observer) ────────────────────
        publisher.register(event -> {
            // Notify order service
            System.out.printf("[OrderService] Received event: %s txn=%s%n",
                    event.eventType(), event.transactionId());
        });
        publisher.register(event -> {
            // Send user notification
            System.out.printf("[NotificationService] Sending %s notification for customer=%s%n",
                    event.eventType(), event.customerId());
        });

        return new PaymentGatewayFacade(processor, webhookProcessor, settlementService, tokeniser);
    }

    public record GatewayConfig(
            String stripeApiKey,
            String stripeWebhookSecret,
            String paypalClientId,
            String paypalClientSecret,
            String paypalWebhookId,
            String razorpayKeyId,
            String razorpayKeySecret,
            Map<String, BigDecimal> merchantTransactionLimits,
            Set<String> blacklistedCustomers,
            Set<String> blacklistedMerchants
    ) {}
}
```

### 6.10 Usage Example / Integration Test

```java
// com/payment/gateway/PaymentGatewayIntegrationTest.java
package com.payment.gateway;

import com.payment.gateway.api.PaymentGatewayFacade;
import com.payment.gateway.model.*;
import com.payment.gateway.provider.ProviderType;
import com.payment.gateway.service.SettlementService;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.*;

public final class PaymentGatewayIntegrationTest {

    public static void main(String[] args) {

        // 1. Bootstrap gateway
        PaymentGatewayBootstrap.GatewayConfig config = new PaymentGatewayBootstrap.GatewayConfig(
                "sk_test_stripe_key",
                "whsec_stripe_secret",
                "paypal-client-id",
                "paypal-client-secret",
                "paypal-webhook-id",
                "rzp_test_key_id",
                "rzp_test_key_secret",
                Map.of("merchant_001", new BigDecimal("50000.00")),
                Set.of("blacklisted_cust_001"),
                Set.of()
        );
        PaymentGatewayFacade gateway = PaymentGatewayBootstrap.build(config);

        // 2. Tokenise a card
        CardData card = new CardData(
                "4111111111111111".toCharArray(),
                "123".toCharArray(),
                "12", "2028", "John Doe");
        String token = gateway.tokeniseCard(card);
        System.out.println("Token: " + token);

        // 3. Initiate a payment
        PaymentRequest request = PaymentRequest.builder()
                .idempotencyKey(UUID.randomUUID().toString())
                .merchantId("merchant_001")
                .customerId("customer_001")
                .amount(new BigDecimal("1500.00"))
                .currency(Currency.getInstance("USD"))
                .paymentMethod(PaymentMethod.CREDIT_CARD)
                .paymentToken(token)
                .description("Order #ORD-9912")
                .providerHint(PaymentRequest.ProviderHint.STRIPE)
                .build();

        PaymentResult result = gateway.initiatePayment(request, "192.168.1.1", "device_fp_abc");
        System.out.println("Payment status: " + result.status() + " txnId=" + result.transactionId());

        // 4. Idempotency check — same key should throw DuplicatePaymentException
        try {
            gateway.initiatePayment(request, "192.168.1.1", "device_fp_abc");
        } catch (com.payment.gateway.exception.DuplicatePaymentException e) {
            System.out.println("Caught duplicate: " + e.getMessage());
        }

        // 5. Partial refund
        RefundRequest refundReq = new RefundRequest(
                UUID.randomUUID().toString(),
                result.transactionId(),
                new BigDecimal("500.00"),
                "Customer requested partial refund",
                "agent_007");

        RefundResult refundResult = gateway.refundPayment(refundReq);
        System.out.println("Refund success: " + refundResult.success()
                + " amount=" + refundResult.refundedAmount());

        // 6. Simulate provider webhook
        Map<String, String> headers = new HashMap<>();
        headers.put("Stripe-Signature", "t=12345,v1=fakesig");
        WebhookPayload webhook = new WebhookPayload(
                "stripe",
                "payment_intent.succeeded",
                result.providerReference(),
                TransactionStatus.SUCCESS,
                headers,
                "{\"event\":\"payment_intent.succeeded\"}",
                java.time.Instant.now()
        );
        // Note: signature won't verify without real HMAC; skipped for demo
        System.out.println("Webhook processing skipped (demo mode — no real HMAC)");

        // 7. Settlement
        SettlementService.SettlementReport report = gateway.reconcile(LocalDate.now());
        System.out.println("Settlement for " + report.reportDate()
                + ": totalGross=" + report.totalGross()
                + " totalRefunded=" + report.totalRefunded()
                + " totalNet=" + report.totalNet());
    }
}
```

---

## 7. Design Patterns Used

### 7.1 Strategy — Payment Provider Pluggability

`PaymentProviderStrategy` is the strategy interface. `StripePaymentProvider`, `PayPalPaymentProvider`, and `RazorpayPaymentProvider` are concrete strategies. `PaymentProviderRegistry` acts as the context that selects and holds strategies.

**Why Strategy here?** The charge and refund logic differs entirely per provider (different SDKs, different auth, different error shapes). Strategy isolates this variance behind a stable interface. Adding Braintree means adding one class and one registration line — zero changes to `PaymentProcessor`.

### 7.2 State — Transaction Lifecycle

Each `TransactionStatus` has a corresponding `TransactionState` implementation. The `Transaction` aggregate delegates all lifecycle calls (`startProcessing`, `markSuccess`, `markFailed`, `markRefunded`) to the current state object, which validates the transition and swaps itself out.

**Why State here?** Lifecycle rules are complex and grow over time (e.g. adding `DISPUTED`, `ON_HOLD`). Encoding transitions in the state objects means invalid transitions are structurally impossible — the compiler enforces them, not runtime if-chains.

### 7.3 Chain of Responsibility — Fraud Checks

`FraudCheckHandler` is the abstract handler. `VelocityCheckHandler`, `AmountLimitCheckHandler`, and `BlacklistCheckHandler` are concrete handlers linked at startup. Each handler either blocks the payment or delegates to the next handler.

**Why CoR here?** Fraud rules are numerous, independently maintainable, and ordered by cost (cheap checks first). CoR lets operations add, remove, or reorder rules without touching other checks. Each handler is unit-testable in isolation.

### 7.4 Observer — Payment Events

`PaymentEventPublisher` maintains a list of `PaymentEventListener` instances. After every state change, `PaymentProcessor` publishes a `PaymentEvent`. Listeners (order service, notification service) react without the processor knowing about them.

**Why Observer here?** Payment outcomes must trigger actions in multiple downstream systems. Hard-coding these calls in `PaymentProcessor` creates coupling. Observer decouples producer from consumers; in production the publisher sends to Kafka and consumers are separate microservices.

### 7.5 Facade — Simplified Client API

`PaymentGatewayFacade` exposes five methods and hides the ten+ internal classes. Clients never import anything from `service`, `fraud`, `state`, or `provider` packages.

**Why Facade here?** The internal design will evolve (e.g. provider selection logic changes, fraud chain is externalised to a rule engine). Clients are insulated from all of that. The facade is the only API contract that matters.

---

## 8. Key Design Decisions and Trade-offs

### 8.1 Synchronous vs. Asynchronous Charging

**Decision:** The `charge()` call is synchronous for the MVP. The provider call happens inline.

**Trade-off:** Simple and predictable for the client, but ties up a thread per in-flight payment. At 10 000 TPS this requires a large thread pool or virtual threads (Java 21+, used here via `Thread.ofVirtual()`). The `WebhookProcessor` handles the async confirmation path for providers (like Razorpay) that return a "pending" status synchronously and confirm later via webhook.

### 8.2 Idempotency Window of 24 Hours

**Decision:** Idempotency keys expire after 24 hours.

**Trade-off:** Prevents the deduplication store from growing unboundedly. Risk: a retried request after 24 hours will not be deduplicated. Acceptable because payment retries across days are rare and can be caught by amount + customer velocity checks.

### 8.3 Immutable Audit Log

**Decision:** `AuditLogService` is append-only with no delete or update path.

**Trade-off:** Requires more storage over time. Benefit: full traceability, required for PCI-DSS compliance. In production this is an append-only table with `INSERT`-only grants and periodic archival to cold storage (S3 Glacier).

### 8.4 Token Storage in Gateway vs. External Vault

**Decision:** `TokenisationService` manages its own AES-GCM key for the case study. In production this is replaced by HashiCorp Vault or AWS KMS.

**Trade-off:** Self-managed crypto is simpler to demo but creates key-management risk. An HSM or cloud KMS provides hardware-backed key security, automatic rotation, and audit logs of every decrypt call.

### 8.5 In-Memory State vs. Database

**Decision:** `transactionStore` and `idempotencyService` use `ConcurrentHashMap` in this case study.

**Trade-off:** Easy to follow and test, but not persistent or distributed. In production: PostgreSQL for transactions (ACID, audit log), Redis for idempotency keys (TTL support), and the transaction store is a JPA `Repository`. The gateway nodes are then fully stateless.

### 8.6 Fraud Chain Synchronous and Blocking

**Decision:** All fraud checks run inline before the provider call.

**Trade-off:** Adds latency for every payment. Benefit: a payment that fails fraud checks never reaches the provider, so no partial charges occur. The velocity check uses an in-memory counter; at scale this moves to a Redis atomic increment.

---

## 9. Extension Points

### 9.1 New Payment Provider

1. Implement `PaymentProviderStrategy`.
2. Register it in `PaymentProviderRegistry`.
3. Add the corresponding `ProviderType` enum value and `ProviderHint`.
4. No changes to `PaymentProcessor`, `WebhookProcessor`, or any fraud handler.

### 9.2 New Payment Method

1. Add the value to `PaymentMethod`.
2. Update `supports()` in any provider that handles it.
3. No business-logic changes.

### 9.3 New Fraud Rule

1. Extend `FraudCheckHandler` and implement `check()`.
2. Insert the handler in the chain in `PaymentGatewayBootstrap`.
3. No changes to existing handlers.

### 9.4 New Transaction State (e.g. DISPUTED)

1. Add `DISPUTED` to `TransactionStatus` and update `canTransitionTo()`.
2. Create `DisputedState implements TransactionState`.
3. Add `markDisputed()` to `Transaction` delegating to the state.
4. Update `SuccessState.markDisputed()` to allow the transition.
5. No changes to `PaymentProcessor` except calling `txn.markDisputed()`.

### 9.5 Async Fraud Screening

Replace the synchronous `fraudChain.handle(request)` call with a `CompletableFuture`-based pipeline. The payment stays in `INITIATED` state until the async result arrives, then transitions to `PROCESSING`. No changes to the chain handlers themselves.

### 9.6 Kafka-Backed Event Publisher

Replace the in-process `PaymentEventPublisher` with a Kafka producer:

```java
public class KafkaPaymentEventPublisher extends PaymentEventPublisher {
    private final KafkaProducer<String, PaymentEvent> producer;

    @Override
    public void publish(PaymentEvent event) {
        producer.send(new ProducerRecord<>("payment-events", event.transactionId(), event));
    }
}
```

Consumers (order service, notification service) become Kafka consumer groups with independent scaling and retry semantics.

---

## 10. Production Considerations

### 10.1 Distributed Idempotency

Replace `ConcurrentHashMap` with Redis `SET NX PX` (set if not exists, with TTL):

```
SET idempotency:{key} {txnId} NX PX 86400000
```

This ensures idempotency across multiple gateway nodes. The `NX` flag makes the set atomic — only one node wins the race to process a given key.

### 10.2 Database Schema

```sql
-- Core transaction table (append-mostly; no updates on amount/customer)
CREATE TABLE transactions (
    id                 UUID PRIMARY KEY,
    idempotency_key    VARCHAR(255) NOT NULL UNIQUE,
    merchant_id        VARCHAR(100) NOT NULL,
    customer_id        VARCHAR(100) NOT NULL,
    amount             NUMERIC(18, 4) NOT NULL,
    currency           CHAR(3) NOT NULL,
    payment_method     VARCHAR(50) NOT NULL,
    payment_token      VARCHAR(255) NOT NULL,
    provider           VARCHAR(50) NOT NULL,
    status             VARCHAR(50) NOT NULL,
    provider_reference VARCHAR(255),
    failure_code       VARCHAR(100),
    failure_message    TEXT,
    refunded_amount    NUMERIC(18, 4) NOT NULL DEFAULT 0,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Audit log: no UPDATE or DELETE grants on this table
CREATE TABLE transaction_audit_log (
    entry_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID NOT NULL REFERENCES transactions(id),
    from_status    VARCHAR(50),
    to_status      VARCHAR(50) NOT NULL,
    actor          VARCHAR(100) NOT NULL,
    reason         TEXT,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transactions_idempotency ON transactions(idempotency_key);
CREATE INDEX idx_transactions_customer    ON transactions(customer_id, created_at DESC);
CREATE INDEX idx_transactions_merchant    ON transactions(merchant_id, created_at DESC);
CREATE INDEX idx_transactions_provider_ref ON transactions(provider_reference);
```

### 10.3 PCI-DSS Compliance Design

| Control | Implementation |
|---------|----------------|
| No raw PAN storage | `CardData.destroy()` zeroes the `char[]` immediately after `tokenise()` |
| Encryption at rest | AES-256-GCM; keys in HSM / AWS KMS, not in application config |
| Encryption in transit | TLS 1.3 enforced on all provider API calls; mTLS between gateway and vault |
| Access control | Only the vault service holds decryption keys; gateway uses tokens |
| Audit logging | Append-only audit table; `INSERT`-only DB grant for application user |
| Network isolation | Tokenisation service in isolated VPC segment; no direct internet access |
| Pen testing | Quarterly external pen test; annual PCI QSA assessment |

### 10.4 Observability

```java
// Metrics (Micrometer / Prometheus)
Counter.builder("payment.initiated").tags("method", method.name()).register(registry);
Counter.builder("payment.success").tags("provider", provider.name()).register(registry);
Counter.builder("payment.failed").tags("provider", provider.name(), "code", errorCode).register(registry);
Timer.builder("payment.processing.duration").tags("provider", provider.name()).register(registry);
Counter.builder("fraud.blocked").tags("rule", ruleName).register(registry);

// Structured logs (JSON with correlation ID)
MDC.put("transactionId", txn.getId());
MDC.put("merchantId", txn.getMerchantId());
MDC.put("correlationId", request.getIdempotencyKey());
log.info("Payment processed");
```

Key dashboards:
- Payment success rate per provider (alert if < 98 %)
- p50/p95/p99 processing latency
- Fraud block rate per rule (sudden spike = misconfigured rule or attack)
- Refund rate per merchant (spike = potential fraud or product issue)
- Webhook processing lag

### 10.5 Circuit Breaker for Providers

Wrap each provider call with Resilience4j:

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
        .failureRateThreshold(50)
        .waitDurationInOpenState(Duration.ofSeconds(30))
        .slidingWindowSize(20)
        .build();

CircuitBreaker cb = CircuitBreakerRegistry.of(config)
        .circuitBreaker("stripe");

String providerRef = CircuitBreaker.decorateSupplier(cb,
        () -> provider.charge(txn)).get();
```

When Stripe's circuit is open, the registry falls back to the next supporting provider automatically (if configured), or returns a graceful degraded response.

### 10.6 Security Headers and Rate Limiting for Webhook Endpoints

```java
// Spring Security / Filter example
@Component
public class WebhookSecurityFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        // 1. Rate limit by source IP (provider IP ranges whitelisted)
        String remoteIp = request.getRemoteAddr();
        if (!isWhitelistedProviderIp(remoteIp)) {
            response.sendError(403, "Forbidden source IP");
            return;
        }
        // 2. Body size limit (prevent payload bombs)
        if (request.getContentLengthLong() > 1_048_576) { // 1 MB
            response.sendError(413, "Payload too large");
            return;
        }
        chain.doFilter(request, response);
    }
}
```

### 10.7 Retry and Exactly-Once Semantics

Payments use an optimistic idempotency model:
- Client retries with the same `idempotencyKey`.
- If the first attempt succeeded (even partially), the retry returns the original result.
- If the first attempt failed, the key is not recorded, so the retry is treated as a fresh attempt.

For webhooks:
- Return `HTTP 200` immediately after signature verification (before processing).
- Process asynchronously with at-least-once delivery.
- Idempotent state machine (replaying SUCCESS on an already-SUCCESS transaction is a no-op).

### 10.8 Graceful Shutdown

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    log.info("Shutting down payment gateway...");
    eventPublisher.shutdown();      // drain event queue
    // Allow in-flight HTTP requests to complete (managed by server)
    log.info("Shutdown complete");
}));
```

---

*End of Case Study*
