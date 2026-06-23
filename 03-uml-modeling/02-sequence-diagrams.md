# Sequence Diagrams

> Sequence diagrams show **how objects interact over time** to fulfill a specific scenario. Where class diagrams answer "what exists," sequence diagrams answer "what happens."

---

## Table of Contents

1. [Core Notation](#1-core-notation)
2. [Message Types](#2-message-types)
3. [Combined Fragments](#3-combined-fragments)
4. [Example: User Login Flow](#4-example-user-login-flow)
5. [Example: ATM Withdrawal Flow](#5-example-atm-withdrawal-flow)
6. [Example: E-Commerce Order Placement](#6-example-e-commerce-order-placement)
7. [How to Sketch Quickly in Interviews](#7-how-to-sketch-quickly-in-interviews)
8. [Exercises](#8-exercises)

---

## 1. Core Notation

### Lifelines

A lifeline represents a participant (object, component, actor) in the interaction. It consists of a box at the top with the participant's name, and a vertical dashed line running down representing time.

```
 :Client        :Server       :Database
    |               |               |
    |               |               |    <- time flows downward
    |               |               |
    |               |               |
```

**Naming convention:** `:ClassName` for anonymous instances, `objectName:ClassName` for named instances.

```
browser:Browser   webServer:HttpServer   db:Database
```

### Activation Boxes

An activation box (thin rectangle on the lifeline) shows when a participant is actively executing.

```
:Client                   :Server
   |                          |
   |                     +---------+
   |   request --------> |         |    <- Server is active during this period
   |                     |         |
   |   response <------- |         |
   |                     +---------+
   |
```

### Object Creation and Destruction

**Creation:** Arrow pointing to the lifeline box (not the dashed line).
**Destruction:** An X at the bottom of the lifeline.

```
:Creator                              new :CreatedObject
   |                                          |
   |---- create() --------------------------> |   <- arrow to box
   |                                          |
   |                                         [X]  <- destroyed
```

---

## 2. Message Types

### Synchronous Message (Solid arrow with filled arrowhead)

The sender waits for the receiver to complete before continuing.

```
Caller                  Callee
  |                       |
  |----> methodCall() --> |
  |                  +----|
  |                  |    |   <- Callee executes
  |                  +----|
  |<--- return value ---- |
  |
```

In text diagrams: `---->`

### Asynchronous Message (Solid arrow with open arrowhead)

The sender does not wait — it continues immediately after sending.

```
Publisher              MessageQueue
    |                       |
    |---> publish(msg) ---> |   <- does not wait
    |                       |
    | (continues immediately)
```

In text diagrams: `---->>` or `- - ->>`

### Return Message (Dashed arrow)

Shows the return value of a synchronous call explicitly. Often implied and omitted for clarity.

```
Controller              Service
    |                      |
    |----> getUser(id) --> |
    |                      |
    |<--- : User ----------|   <- dashed = return
```

In text diagrams: `<----` or `<- - -`

### Self-Call

A message from a lifeline to itself.

```
:OrderService
      |
 +----|
 |    |----> validate() ----|
 |    |<---                  |
 |    |
 +----|
```

---

## 3. Combined Fragments

Combined fragments represent control flow in sequence diagrams.

### alt — Alternative (if/else)

```
:Client                    :AuthService
   |                            |
   | login(user, pwd) --------> |
   |                            |
   |     alt [credentials valid]|
   |     |--------------------------|
   |     | <--- :AuthToken -------- |
   |     |--------------------------|
   |     alt [credentials invalid]  |
   |     |--------------------------|
   |     | <--- AuthException ------ |
   |     |--------------------------|
```

### opt — Optional (if, no else)

```
:OrderService               :EmailService
      |                           |
      | [opt: email notifications enabled]
      |-----------------------------------|
      | sendConfirmation(order) --------> |
      |-----------------------------------|
```

### loop — Loop

```
:BatchProcessor               :ItemProcessor
      |                              |
      | [loop: for each item in batch]|
      |-------------------------------|
      | processItem(item) ----------> |
      | <--- ProcessResult ---------- |
      |-------------------------------|
```

### par — Parallel Execution

```
:OrderService     :EmailService     :SMSService
      |                 |                 |
      | par             |                 |
      |-----------------|-----------------|
      | sendEmail() --> |                 |
      |                 |                 |
      | sendSMS() ----------------------> |
      |-----------------|-----------------|
      | (both execute in parallel)
```

### ref — Reference to Another Sequence Diagram

```
:Client              :Server
   |                    |
   | ref: Authentication|
   |......................|   <- refers to another diagram
   |                    |
   | doActualWork() --> |
```

---

## 4. Example: User Login Flow

**Scenario:** A user logs in via a web browser. The server validates credentials against the database, creates a session, and returns an authentication token.

```
Browser          WebServer        AuthService       UserDB         SessionStore
   |                 |                 |               |                |
   | POST /login     |                 |               |                |
   | {user, pwd} --> |                 |               |                |
   |             +---|                 |               |                |
   |             | validateRequest()   |               |                |
   |             +---|                 |               |                |
   |                 | authenticate(   |               |                |
   |                 | user, pwd) ---> |               |                |
   |                 |             +---|               |                |
   |                 |             | findUser(user) -> |                |
   |                 |             |               +---|                |
   |                 |             |               | query             |
   |                 |             |               +---|                |
   |                 |             |                   |                |
   |   alt [user exists AND pwd matches]               |                |
   |   ------------------------------------------------|----------------|
   |                 |             | <-- :User --------|                |
   |                 |             |                                    |
   |                 |             | createSession(userId) -----------> |
   |                 |             | <-- sessionId:String ------------- |
   |                 |             |                                    |
   |                 | <-- :AuthToken (sessionId, expiry) ----------    |
   |                 |                                                   |
   | HTTP 200        |                                                   |
   | {token} <------ |                                                   |
   |                 |                                                   |
   |   alt [user not found OR pwd wrong]                                |
   |   ------------------------------------------------                 |
   |                 | <-- AuthException --                             |
   |                 |                                                   |
   | HTTP 401        |                                                   |
   | {error} <------ |                                                   |
   |   ------------------------------------------------                 |
```

**Java implementation sketched from diagram:**

```java
// WebServer layer (Controller)
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest request) {
    try {
        AuthToken token = authService.authenticate(
            request.getUsername(), request.getPassword());
        return ResponseEntity.ok(token);
    } catch (AuthenticationException e) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse(e.getMessage()));
    }
}

// AuthService
public class AuthService {
    private final UserRepository userRepository;
    private final SessionStore sessionStore;
    private final PasswordEncoder passwordEncoder;

    public AuthToken authenticate(String username, String password) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new AuthenticationException("Invalid credentials"));

        if (!passwordEncoder.matches(password, user.getPasswordHash())) {
            throw new AuthenticationException("Invalid credentials");
        }

        String sessionId = sessionStore.createSession(user.getId());
        return new AuthToken(sessionId, LocalDateTime.now().plusHours(24));
    }
}
```

---

## 5. Example: ATM Withdrawal Flow

**Scenario:** A customer inserts their card, enters PIN, selects withdrawal amount, and receives cash.

```
Customer       ATM          BankServer        AccountDB
   |            |                |                |
   | insertCard |                |                |
   | ---------> |                |                |
   |            |                |                |
   | enterPin() |                |                |
   | ---------> |                |                |
   |            | validatePin(   |                |
   |            | cardNo, pin) ->|                |
   |            |                | findAccount(   |
   |            |                | cardNo) -----> |
   |            |                | <-- :Account --|
   |            |                |                |
   |  alt [PIN correct]         |                |
   |  -------------------------------------------|
   |            | <-- PinValid - |                |
   |            |                |                |
   | selectAmount(500) -------> |                |
   |            |                |                |
   |            | withdraw(      |                |
   |            | accountId,500)->               |
   |            |                | getBalance(    |
   |            |                | accountId) --> |
   |            |                | <-- balance:1200|
   |            |                |                |
   |  alt [balance >= amount]   |                |
   |  ------------------------------------------|
   |            |                | deductBalance( |
   |            |                | accountId,500)->|
   |            |                | <-- OK --------|
   |            |                |                |
   |            | <-- WithdrawalSuccess ----------|
   |            |                |                |
   | dispenseCash(500) --------> |               |
   | printReceipt() -----------> |               |
   |  ------------------------------------------|
   |  alt [balance < amount]                     |
   |  ------------------------------------------|
   |            | <-- InsufficientFundsException  |
   |            |                                 |
   | displayError("Insufficient funds") -------> |
   |  ------------------------------------------|
   |  ------------------------------------------|
   |  alt [PIN wrong]                            |
   |  ------------------------------------------|
   |            | <-- PinInvalid -- |             |
   | displayError("Wrong PIN") --------> |       |
   |  ------------------------------------------|
```

**Java implementation:**

```java
public class ATM {
    private final BankService bankService;
    private final CashDispenser cashDispenser;
    private final ReceiptPrinter printer;
    private String insertedCardNumber;

    public void insertCard(String cardNumber) {
        this.insertedCardNumber = cardNumber;
        displayMessage("Please enter your PIN");
    }

    public PinValidationResult validatePin(String pin) {
        return bankService.validatePin(insertedCardNumber, pin);
    }

    public WithdrawalResult withdraw(String accountId, double amount) {
        try {
            bankService.withdraw(accountId, amount);
            cashDispenser.dispense(amount);
            printer.printReceipt(accountId, amount, LocalDateTime.now());
            return WithdrawalResult.success(amount);
        } catch (InsufficientFundsException e) {
            displayMessage("Insufficient funds. Available: " + e.getAvailableBalance());
            return WithdrawalResult.failure("Insufficient funds");
        }
    }
}

public class BankService {
    private final AccountRepository accountRepository;

    public PinValidationResult validatePin(String cardNumber, String pin) {
        Account account = accountRepository.findByCardNumber(cardNumber)
            .orElseThrow(() -> new CardNotFoundException("Card not found"));
        boolean valid = account.validatePin(pin);
        return valid ? PinValidationResult.valid(account.getId())
                     : PinValidationResult.invalid();
    }

    public void withdraw(String accountId, double amount) {
        Account account = accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));

        if (account.getBalance() < amount) {
            throw new InsufficientFundsException(account.getBalance(), amount);
        }

        account.deduct(amount);
        accountRepository.save(account);
    }
}
```

---

## 6. Example: E-Commerce Order Placement

**Scenario:** A customer adds items to a cart, proceeds to checkout, enters payment, and the system confirms the order with inventory updates and an email.

```
Customer    OrderController   OrderService  InventoryService PaymentGateway EmailService
   |              |                |               |               |             |
   | POST /checkout|               |               |               |             |
   | {items, card} -> |            |               |               |             |
   |              | placeOrder(   |               |               |             |
   |              | request) ---> |               |               |             |
   |              |            +--|               |               |             |
   |              |            | validateItems()  |               |             |
   |              |            +--|               |               |             |
   |              |               |               |               |             |
   |              |               | loop [for each item]          |             |
   |              |               |-------------------------------|             |
   |              |               | checkStock(   |               |             |
   |              |               | itemId, qty)->|               |             |
   |              |               | <-- stockOk --|               |             |
   |              |               |-------------------------------|             |
   |              |               |                               |             |
   |              |  alt [all items in stock]                     |             |
   |              |  ---------------------------------------------|             |
   |              |               | chargeCard(   |               |             |
   |              |               | cardToken,    |               |             |
   |              |               | total) ---------------------->|             |
   |              |               | <-- PaymentResult ------------|             |
   |              |               |                               |             |
   |              |    alt [payment successful]                   |             |
   |              |    --------------------------------------------|            |
   |              |               | loop [for each item]          |             |
   |              |               |-------------------------------|             |
   |              |               | reserveStock( |               |             |
   |              |               | itemId, qty)->|               |             |
   |              |               | <-- OK --------|               |             |
   |              |               |-------------------------------|             |
   |              |               |                               |             |
   |              |               | sendConfirmation(order) -----------------> |
   |              |               | <-- emailSent --------------------------------|
   |              |               |                               |             |
   |              | <-- :Order (CONFIRMED) --|                    |             |
   |              |                          |                    |             |
   | HTTP 201     |                          |                    |             |
   | {orderId} <--|                          |                    |             |
   |              |    alt [payment failed]  |                    |             |
   |              |    ------------------    |                    |             |
   |              |               | <-- PaymentException          |             |
   |              | <-- PaymentFailed --|   |                    |             |
   | HTTP 402 <---|                    |                    |             |
   |              |  alt [item out of stock] |               |             |
   |              |  -------------------------------------------            |
   |              |               | <-- OutOfStockException                  |
   |              | <-- OutOfStock--|                           |             |
   | HTTP 409 <---|                 |                           |             |
   |              |  -------------------------------------------            |
```

**Java implementation:**

```java
@RestController
public class OrderController {
    private final OrderService orderService;

    @PostMapping("/checkout")
    public ResponseEntity<?> placeOrder(@RequestBody CheckoutRequest request) {
        try {
            Order order = orderService.placeOrder(request);
            return ResponseEntity.status(HttpStatus.CREATED)
                .body(new OrderResponse(order.getId(), order.getStatus()));
        } catch (OutOfStockException e) {
            return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(new ErrorResponse("Item out of stock: " + e.getItemId()));
        } catch (PaymentDeclinedException e) {
            return ResponseEntity.status(HttpStatus.PAYMENT_REQUIRED)
                .body(new ErrorResponse("Payment declined: " + e.getReason()));
        }
    }
}

@Service
public class OrderService {
    private final InventoryService inventoryService;
    private final PaymentGateway paymentGateway;
    private final EmailService emailService;
    private final OrderRepository orderRepository;

    @Transactional
    public Order placeOrder(CheckoutRequest request) {
        // Validate all items are in stock
        for (OrderItem item : request.getItems()) {
            if (!inventoryService.checkStock(item.getProductId(), item.getQuantity())) {
                throw new OutOfStockException(item.getProductId());
            }
        }

        // Calculate total
        Money total = calculateTotal(request.getItems());

        // Charge payment
        PaymentResult result = paymentGateway.charge(request.getPaymentToken(), total);
        if (!result.isSuccessful()) {
            throw new PaymentDeclinedException(result.getDeclineReason());
        }

        // Reserve inventory
        for (OrderItem item : request.getItems()) {
            inventoryService.reserve(item.getProductId(), item.getQuantity());
        }

        // Create and save order
        Order order = new Order(request.getCustomerId(), request.getItems(), total);
        order.setStatus(OrderStatus.CONFIRMED);
        Order saved = orderRepository.save(order);

        // Send confirmation (async — best effort)
        try {
            emailService.sendOrderConfirmation(saved);
        } catch (EmailException e) {
            // Log but don't fail the order — email is not critical path
            log.warn("Failed to send order confirmation for order {}", saved.getId(), e);
        }

        return saved;
    }
}
```

---

## 7. How to Sketch Quickly in Interviews

### The 4-Step Process (5 minutes)

**Step 1: List the participants (30 seconds)**
Write them left-to-right from initiator to downstream. Usually: Actor/Client → Controller/API → Service → External Services

```
User  |  Browser  |  AuthController  |  AuthService  |  UserRepo  |  SessionStore
```

**Step 2: Identify the happy path (2 minutes)**
Trace the main success scenario as a sequence of numbered arrows:

```
1. User submits credentials to Browser
2. Browser POST /login to AuthController
3. AuthController calls AuthService.authenticate(user, pwd)
4. AuthService calls UserRepo.findByUsername(user)
5. UserRepo returns User
6. AuthService creates session in SessionStore
7. AuthService returns AuthToken to AuthController
8. AuthController returns HTTP 200 to Browser
9. Browser shows logged-in state to User
```

**Step 3: Add the error paths as alt fragments (1 minute)**
Identify the 2-3 main failure modes and box them.

**Step 4: Draw the arrows (2 minutes)**
Convert the numbered list into a diagram with arrows and lifelines.

### Shorthand Text Notation

When time is very short, number the calls:

```
1: Browser -> AuthController: POST /login {username, pwd}
2: AuthController -> AuthService: authenticate(username, pwd)
3: AuthService -> UserRepo: findByUsername(username)
4: UserRepo --> AuthService: :User
   [if found and pwd matches]
5: AuthService -> SessionStore: createSession(userId)
6: SessionStore --> AuthService: sessionId
7: AuthService --> AuthController: :AuthToken
8: AuthController --> Browser: HTTP 200 {token}
   [else]
5': AuthService --> AuthController: AuthException
8': AuthController --> Browser: HTTP 401 {error}
```

This communicates the sequence clearly without formal diagram notation.

### What Interviewers Look For

1. **Correct direction of arrows** — from caller to callee
2. **Separate layers** — controller, service, repository distinct
3. **Error cases covered** — at least one alt/opt fragment
4. **Return arrows shown or clearly implied** — especially for synchronous calls
5. **Realistic participant names** — use real service/class names, not generic "Database"

---

## 8. Exercises

### Exercise 1: Password Reset Flow

**Scenario:** A user requests a password reset. The system sends an email with a reset link. The user clicks the link and sets a new password.

**Participants:** User, Browser, PasswordResetController, PasswordResetService, UserRepository, TokenStore, EmailService

**Draw the sequence for:**
1. The request reset flow
2. The verify token and set new password flow

**Solution outline:**

```
=== Flow 1: Request Reset ===

User --> Browser: click "Forgot Password"
Browser --> PasswordResetController: POST /password-reset {email}
PasswordResetController --> PasswordResetService: requestReset(email)
PasswordResetService --> UserRepository: findByEmail(email)
  [opt: user exists]
  UserRepository --> PasswordResetService: :User
  PasswordResetService -> PasswordResetService: generateToken()
  PasswordResetService --> TokenStore: storeToken(token, userId, expiresAt)
  PasswordResetService --> EmailService: sendResetEmail(email, token)
  [end opt]
PasswordResetController --> Browser: HTTP 200 "If email exists, check inbox"
(Note: always return 200 to prevent email enumeration attacks)

=== Flow 2: Set New Password ===

User --> Browser: click reset link (contains token)
Browser --> PasswordResetController: POST /password-reset/confirm {token, newPassword}
PasswordResetController --> PasswordResetService: confirmReset(token, newPassword)
PasswordResetService --> TokenStore: getToken(token)
  [alt: token valid and not expired]
  TokenStore --> PasswordResetService: :ResetToken (userId, expiresAt)
  PasswordResetService --> UserRepository: findById(userId)
  UserRepository --> PasswordResetService: :User
  PasswordResetService -> PasswordResetService: hashPassword(newPassword)
  PasswordResetService --> UserRepository: updatePassword(userId, hashedPwd)
  PasswordResetService --> TokenStore: invalidateToken(token)
  PasswordResetController --> Browser: HTTP 200 "Password reset successful"
  [alt: token expired or invalid]
  TokenStore --> PasswordResetService: null or expired token
  PasswordResetService -> PasswordResetController: TokenExpiredException
  PasswordResetController --> Browser: HTTP 400 "Reset link has expired"
```

---

### Exercise 2: Online Shopping Cart Checkout

**Scenario:** A guest user adds items to a cart, applies a coupon code, and checks out.

**Draw the complete sequence including:**
- Item addition with stock validation
- Coupon validation and discount calculation
- Checkout with payment processing
- Order creation

---

### Exercise 3: Microservices Order Creation

**Scenario:** In a microservices architecture, an Order Service creates an order, publishes an event to a message broker, and downstream services (Inventory Service, Notification Service) react asynchronously.

**Draw the sequence showing:**
- Synchronous REST call from client to Order Service
- Synchronous call from Order Service to Payment Service
- Asynchronous event publication to Kafka
- Async consumption by Inventory Service and Notification Service

**Highlight the difference between synchronous arrows (filled) and asynchronous arrows (open).**

**Solution outline:**

```
Client     OrderService    PaymentService    Kafka      InventoryService  NotificationSvc
  |             |                |             |               |                 |
  | POST/orders |                |             |               |                 |
  | ----------> |                |             |               |                 |
  |             | chargeCard() ->|             |               |                 |
  |             | <-- success -- |             |               |                 |
  |             |                |             |               |                 |
  |             | createOrder()  |             |               |                 |
  |             |                |             |               |                 |
  |             |-->> publish(OrderCreated) -> |               |                 |
  |             |    (async, does not wait)    |               |                 |
  |             |                |             |               |                 |
  | 201 Created |                |             |               |                 |
  | <---------- |                |             |               |                 |
  |             |                |             |               |                 |
  |             |                |             | -->> consume  |                 |
  |             |                |             | (async) ----> |                 |
  |             |                |             |               | reserveStock()  |
  |             |                |             |               |                 |
  |             |                |             | -->> consume -----------------> |
  |             |                |             | (async)       |  sendEmail()    |
```

The double arrows (`-->>`) indicate asynchronous messages.
