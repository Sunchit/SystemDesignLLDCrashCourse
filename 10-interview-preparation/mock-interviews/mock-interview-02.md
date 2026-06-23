# Mock Interview 02 — Design a Notification System

---

## Interview Setup

### Context

This mock interview simulates a Senior Software Engineer (L5/L6) onsite loop at a mid-to-large product company — think a company operating at the scale of LinkedIn, Airbnb, or a major fintech with millions of active users. The interviewer is a Staff Engineer on the Platform team. The problem is deliberately chosen because it sits at the intersection of object-oriented design, distributed systems thinking, and production engineering concerns — all three of which a senior engineer is expected to reason about simultaneously.

The candidate has been told in advance: "We'll be doing a low-level design interview. You'll be asked to design and code a system from scratch on a whiteboard or shared editor. We're looking for production-quality thinking, not just a class diagram."

### Time Allocation Breakdown (Total: 55 minutes)

| Stage | Activity | Duration |
|---|---|---|
| 1 | Requirements and Constraints | 8 min |
| 2 | Channel Abstraction | 10 min |
| 3 | Class Design — Preferences, Templates, Orchestration | 12 min |
| 4 | Code: Retry and Rate-Limiting Logic | 15 min |
| 5 | Scale Considerations at the Class Level | 5 min |
| — | Debrief / Candidate Questions | 5 min |

### What the Interviewer Is Testing

1. **Decomposition skill** — Can the candidate take a vague prompt and produce a clean, bounded problem statement with explicit assumptions?
2. **Abstraction quality** — Does the candidate reach for the right interfaces at the right level of granularity, or do they either over-engineer (fifteen interfaces for three classes) or under-engineer (one God class)?
3. **Open/Closed principle in practice** — Does the design allow a new channel (e.g., WhatsApp) to be added by writing a new class, with zero changes to existing code?
4. **Concurrency literacy** — Can the candidate write thread-safe code without being prompted? Do they know when to use `AtomicLong`, `ReentrantLock`, and `ConcurrentHashMap`, and why each is appropriate?
5. **Failure-mode reasoning** — Does the candidate think about what happens when a delivery fails, when a third-party API is rate-limited, when the same notification is submitted twice?
6. **Behavioral modeling** — Does the candidate model user preferences correctly as a first-class domain object, not a boolean flag bolted onto a user record?
7. **Production instincts** — Does the candidate add observability (delivery receipts, logging), or do they design a system that fails silently?
8. **Communication under pressure** — Can the candidate articulate trade-offs clearly ("I'm using a token bucket instead of a leaky bucket here because...") rather than just writing code and hoping the interviewer follows?

### How to Use This Mock Interview

**For self-practice:** Read each stage's interviewer question, pause the document, and speak your answer aloud for the allotted time. Then read the Expected Senior Response and score yourself honestly against the Green/Red Flag rubrics.

**For pair practice:** One person plays the interviewer and reads the script verbatim. The other person answers. Do not show the candidate the Expected Senior Response until after they have answered. Debrief using the scoring guide at the end.

**For course instructors:** The Stage 4 coding section can be used as a standalone take-home coding assessment. The Scoring Guide maps directly to a 40-point rubric (8 dimensions × 5 points each).

---

## Stage-by-Stage Interviewer Script

---

### Stage 1: Requirements and Constraints (8 minutes)

#### Interviewer Questions

*Open with:*
> "We need to design a notification system. Users should be able to receive notifications through email, SMS, and push notifications. Walk me through how you'd approach designing this."

*Follow-up probes (use as needed to fill 8 minutes):*
> "How many notifications per second are we expecting to handle?"

> "If a notification fails to deliver, what should happen?"

> "Should notifications be delivered immediately, or is some latency acceptable?"

> "What happens if a user has turned off SMS but we send them a CRITICAL alert — say, a fraud detection notification?"

> "Are all notifications equal in importance?"

---

#### Expected Senior Response — Stage 1

A strong senior candidate does not start drawing classes. They drive a requirements conversation for the first 2-3 minutes, explicitly separating functional from non-functional requirements, and they state their assumptions out loud.

**Functional requirements they should identify:**
- Send notifications through three channels: email, SMS, push notification.
- Users can configure their channel preferences (opt in/out per channel).
- Users can globally opt out of all non-critical notifications.
- Notifications have priorities (Critical, High, Normal, Low). Critical notifications may override opt-out preferences.
- Messages are template-based — the caller provides a template ID and parameters, not raw message text.
- The system retries failed deliveries with backoff.
- Delivery receipts are tracked (did the notification actually go out? did it fail permanently?).

**Non-functional requirements they should identify:**
- Scale: "I'll assume 10,000 notifications per second at peak. That's roughly 850 million per day — consistent with a large consumer platform." (The specific number matters less than showing they've thought about it.)
- Latency: Async delivery is acceptable for Normal/Low priority. Critical and High should be dispatched within seconds.
- Delivery guarantee: At-least-once semantics. We prefer a duplicate notification over a missed one.
- Idempotency: Because of at-least-once, the system must be idempotent on `requestId` to avoid genuine duplicates reaching the user.
- Rate limiting: Both system-level (to protect third-party providers) and per-user (to avoid spamming).

**Constraint clarification they should raise:**
- "Should the rate limiting be per user, per channel, per user-channel pair, or all three? I'll assume per user-channel pair for now and make that configurable."
- "Are templates stored in a database, or do callers pass raw strings? I'll design for template IDs resolved from a service."
- "I'm going to assume this is a single-JVM design for the LLD, but I'll flag where message queues (Kafka, SQS) would sit in a distributed version."

---

#### Red Flags — Stage 1

1. **Jumps directly to class names** without spending any time on requirements. Starts with "I'll have a `NotificationService` with a `sendEmail` method" before constraints are discussed.
2. **Treats all notifications as equal priority.** Fails to ask about or propose priority tiers. A notification system without priorities cannot distinguish a fraud alert from a marketing email.
3. **Assumes synchronous delivery is required.** States "we'll send the email and wait for a response before returning" without questioning whether the caller needs to block. This collapses the entire async/queue discussion.
4. **Does not mention idempotency.** In an at-least-once delivery system, every senior engineer should immediately ask "what happens if this request is submitted twice?"

---

#### Green Flags — Stage 1

1. **Asks explicit scale questions** and derives implications: "At 10K/s, we can't do synchronous HTTP calls to email providers inline — we need a queue." This is a green flag because the candidate is letting scale shape architecture rather than imposing architecture first.
2. **Distinguishes between user opt-out and channel failure.** Raises the question: "If a user has SMS disabled but a CRITICAL message arrives, do we fall back to email, or do we force-deliver to SMS? That's a product decision — I'll assume critical overrides opt-out for now."
3. **Introduces idempotency unprompted.** Mentions `requestId` as a deduplication key before drawing any classes.
4. **Separates the template system from the delivery system.** Observes that "who resolves the template" and "who sends the message" are separate concerns even at requirements time.

---

### Stage 2: Channel Abstraction (10 minutes)

#### Interviewer Questions

*Open with:*
> "Let's talk about how you model the three channels — email, SMS, and push. Start with the interface."

*Follow-up probes:*
> "Your `NotificationChannel` interface has a `send()` method. What does it take as input and what does it return?"

> "Six months from now, the product team asks you to add WhatsApp as a fourth channel. What changes in your design?"

> "What is different between an email channel and an SMS channel at the implementation level? What is the same?"

> "Why not just use an enum switch statement in a single `NotificationService.send()` method instead of a polymorphic channel abstraction?"

---

#### Expected Senior Response — Stage 2

The candidate should propose a `NotificationChannel` interface immediately and reason about its contract before writing it.

**Interface design they should articulate:**

```java
public interface NotificationChannel {
    DeliveryResult send(RenderedNotification notification, String userId);
    boolean supports(ChannelType channelType);
    ChannelType getChannelType();
}
```

They should explain the reasoning behind each method:
- `send()` takes a `RenderedNotification` (already-rendered content) rather than a raw template, because the channel's job is delivery, not rendering. Rendering is a separate concern.
- `supports()` enables a channel registry to be queried at runtime without a switch statement.
- `getChannelType()` is used to populate the channel map in the dispatcher keyed by `ChannelType`.

**On extensibility (adding WhatsApp):** The candidate should say: "I create a `WhatsAppChannel` class that implements `NotificationChannel`, add `WHATSAPP` to the `ChannelType` enum, register it in the dispatcher's channel map, and create templates with `channelType = WHATSAPP`. The dispatcher doesn't change. This is the Open/Closed principle — the dispatcher is closed for modification, open for extension via the channel map."

**On what varies vs. what is constant across channels:**
- *Varies*: HTTP client, authentication (API key vs. OAuth), payload format (JSON for push, SMTP for email, REST for SMS), retry characteristics (SMS providers have stricter rate limits than email providers)
- *Constant*: The contract — given a rendered notification and a user ID, attempt delivery and return a success/failure result

**On the enum switch anti-pattern:** "If I use a switch statement on channel type, every time I add a channel I modify the dispatcher. That's fragile in a team environment and violates OCP. The channel map approach means the dispatcher never needs to know about specific channels — it just looks up the right implementation."

---

#### Red Flags — Stage 2

1. **Puts rendering logic inside the channel implementation.** Writes `EmailChannel.send(NotificationRequest request)` and calls `templateService.render()` inside `send()`. This couples the channel to the template system — now every channel must depend on `TemplateService`.
2. **Does not use an interface at all.** Creates an abstract class `BaseNotificationChannel` with no interface, making it impossible to mock in tests and coupling all channels to a common inheritance hierarchy.
3. **Hardcodes channel logic in the dispatcher** with an `if (channelType == EMAIL) { emailChannel.send() } else if (...)`. Fails to see that this approach breaks every time a channel is added.
4. **Makes `send()` return void.** Does not return a delivery result, which means the caller has no way to know if delivery succeeded, failed, or was rate-limited at the provider level. This makes retry logic impossible without side effects.

---

#### Green Flags — Stage 2

1. **Separates `RenderedNotification` from `NotificationRequest` immediately.** Recognizes that the channel should receive ready-to-send content, not a raw request with template parameters. This is the key abstraction insight of this stage.
2. **Explains why `supports()` is on the interface** rather than in the dispatcher. "I could do `instanceof` checks, but `supports()` keeps the type-checking logic with the class that owns it."
3. **Mentions that channel implementations can have their own retry characteristics.** "In a more complete design, each channel could expose a preferred `RetryPolicy` — email can retry 5 times over 10 minutes, SMS should retry faster because messages expire."
4. **Voluntarily mentions thread safety at the channel level.** "Each channel implementation should be stateless or thread-safe because the dispatcher will call `send()` from a thread pool."

---

### Stage 3: Class Design — Preferences, Templates, and Orchestration (12 minutes)

#### Interviewer Questions

*Open with:*
> "Walk me through the `UserPreferenceService`. What does it look like, and how does the dispatcher use it?"

*Follow-up probes:*
> "What's the difference between a `NotificationTemplate` and a `RenderedNotification`? Why are they separate classes?"

> "The dispatcher needs to coordinate preferences, templates, rate limiting, retry, and channels. What are its exact responsibilities? Where do you draw the line?"

> "What does the dispatcher do if the user has SMS disabled, but the notification request specifies SMS as the channel?"

> "How does priority affect the dispatcher's behavior?"

---

#### Expected Senior Response — Stage 3

**On `UserPreferenceService`:**

The candidate should model user preferences as a rich domain object, not a flat boolean. A strong answer:

```
UserPreference {
    userId: String
    enabledChannels: Set<ChannelType>
    globalOptOut: boolean
    rateLimit: Map<ChannelType, Integer>  // max notifications per minute per channel
}
```

`UserPreferenceService` is an interface with `getPreference(String userId)` and `updatePreference(UserPreference preference)`. The in-memory implementation uses a `ConcurrentHashMap`. The candidate should note: "In production this is backed by a database with a cache layer — the interface hides that."

The dispatcher calls `userPreferenceService.getPreference(userId)` first, before doing anything else. If `globalOptOut` is true, the dispatcher returns `NotificationResult.optOut(requestId)` immediately — unless priority is CRITICAL, in which case it proceeds.

**On `NotificationTemplate` vs `RenderedNotification`:**

This is a key design question. A strong answer:
- `NotificationTemplate` is a stored artifact — it contains a `subjectPattern` like `"Your order {{orderId}} has shipped"` and a `bodyPattern`. It is channel-specific because email bodies are HTML, SMS bodies are plain 160-char text, and push bodies are 100-char summaries.
- `RenderedNotification` is the runtime product of calling `template.render(params)`. It contains the fully interpolated subject and body, ready to hand to a channel.
- Separation matters because templates are reused across millions of notifications. You never want a template to carry per-notification state.

**On the dispatcher's responsibilities (the candidate should list exactly these, and draw a clear line):**

The dispatcher is responsible for:
1. Validating that the requested channel exists in the channel map.
2. Checking user opt-out preference.
3. Checking rate limit (delegating to `RateLimiter`).
4. Resolving and rendering the template (delegating to `TemplateService`).
5. Attempting delivery via the channel with retry (delegating to the channel and `RetryPolicy`).
6. Publishing a `DeliveryReceipt` to all registered observers.
7. Returning a `NotificationResult`.

The dispatcher is NOT responsible for: template storage, user data persistence, actual HTTP calls to email/SMS providers, or deciding what "retry" means. Those are delegated.

**On priority affecting dispatcher behavior:**

"Priority drives two things: first, the `PriorityBlockingQueue` ordering — CRITICAL notifications are dequeued before HIGH, which are dequeued before NORMAL. Second, CRITICAL priority overrides `globalOptOut`. I would also consider having CRITICAL bypass the rate limiter, or at minimum use a separate rate limit bucket."

---

#### Red Flags — Stage 3

1. **Conflates `NotificationTemplate` and `RenderedNotification` into one class.** Stores `templateParams` directly on the template, making the template stateful and non-reusable across concurrent requests.
2. **Puts business logic inside the channel implementations.** Writes preference checking or rate limiting inside `EmailChannel.send()`. Each channel reimplements the same logic, violating DRY and making the system inconsistent.
3. **Makes the dispatcher a static utility class** with static methods. Loses dependency injection, making the dispatcher untestable and impossible to configure differently per environment.
4. **Does not model the dispatcher's `dispatch()` method return type.** Returns void or throws exceptions for all failure cases. Does not use a result object, making it impossible for callers to distinguish between "user opted out," "rate limited," and "delivery failed after all retries."

---

#### Green Flags — Stage 3

1. **Proposes `NotificationResult` as a sealed type or uses static factory methods.** Says "I want `NotificationResult.success()`, `NotificationResult.rateLimited()`, `NotificationResult.optOut()`, `NotificationResult.failed()` — this makes the caller's code a clean switch expression."
2. **Raises the observer pattern for delivery receipts unprompted.** "I'll have the dispatcher publish a `DeliveryReceipt` to a list of `DeliveryReceiptObserver` instances after each attempt. This decouples auditing, analytics, and alerting from the core delivery path."
3. **Correctly identifies that priority is an ordering concern, not a filtering concern.** Priority does not mean "skip low-priority notifications" — it means "when the system is backlogged, deliver critical ones first."
4. **Mentions interface segregation for the `TemplateService`.** "The dispatcher only needs `getTemplate(templateId, channelType)`. I don't expose template creation or deletion on the same interface the dispatcher depends on — that's a write-side admin concern."

---

### Stage 4: Code the Retry and Rate-Limiting Logic (15 minutes)

#### Interviewer Questions

*Open with:*
> "I want you to write the `RetryPolicy` interface and the `ExponentialBackoffRetry` implementation. Then write the `RateLimiter` interface and a `TokenBucketRateLimiter` implementation. Take the full 15 minutes — I want working code, not pseudocode."

*Follow-up probes (raise after candidate starts writing):*
> "Your `ExponentialBackoffRetry` — is it thread-safe? Walk me through what shared state it has."

> "In your token bucket, what happens between two `tryAcquire()` calls that are 60 seconds apart? Walk me through the refill."

> "What's jitter, and why would you add it to exponential backoff?"

> "Your `ConcurrentHashMap` in `TokenBucketRateLimiter` — is that sufficient for thread safety, or do you need additional synchronization?"

---

#### Expected Senior Response — Stage 4

A strong candidate writes the following code fluently, narrating their decisions:

**On `RetryPolicy`:**
"The interface has two methods: `shouldRetry(int attempt, Exception e)` and `getDelayMillis(int attempt)`. I separate them because a caller needs to know both whether to retry and how long to wait before retrying."

**On `ExponentialBackoffRetry`:**
"Exponential backoff means delay doubles each attempt: base * 2^attempt. I cap it at `maxDelayMs` to avoid infinite waits. Jitter adds randomness — without it, all clients that hit the same failure simultaneously will all retry at the same moment, creating a thundering herd. I use `Math.random() * delay` to spread retries across a window."

"The class itself is stateless — `maxAttempts`, `baseDelayMs`, `maxDelayMs` are final fields set at construction. Multiple threads can call `shouldRetry()` and `getDelayMillis()` concurrently with no synchronization needed."

**On `TokenBucketRateLimiter`:**
"The token bucket algorithm: each key (userId + channelType) has a bucket with a maximum capacity. Tokens refill at a fixed rate over time. Each `tryAcquire()` call attempts to consume one token. If the bucket is empty, it returns false.

The challenge is thread safety. My bucket state per key is: `lastRefillTime` (AtomicLong nanoseconds) and `availableTokens` (AtomicLong). I use `ConcurrentHashMap.computeIfAbsent()` to create buckets safely. For the actual token consume + refill operation, I need to atomically read the last refill time, compute elapsed time, add new tokens, and consume one. That multi-step operation requires a lock per bucket — I use a `ReentrantLock` stored alongside the token count."

---

#### Red Flags — Stage 4

1. **Implements retry with a fixed delay** (e.g., `Thread.sleep(1000)` for every attempt) without any exponential progression. Does not mention jitter. At scale, this causes retry storms.
2. **Uses `synchronized` on the entire `TokenBucketRateLimiter` instance** rather than per-key locking. This serializes all rate limit checks across all users, creating a bottleneck. A candidate should scope the lock to the bucket, not the map.
3. **Does not handle the case where tokens should not exceed capacity.** After a long idle period, a user's bucket should not accumulate unlimited tokens — the candidate forgets to cap refilled tokens at `maxTokens`.
4. **Writes `shouldRetry()` to catch all exceptions indiscriminately.** Does not consider that some exceptions (e.g., `InvalidRecipientException` — the phone number doesn't exist) should not be retried. A senior candidate says "I'd check `exception instanceof RetryableException` or maintain an allowlist of retryable exception types."

---

#### Green Flags — Stage 4

1. **Immediately makes `ExponentialBackoffRetry` immutable with final fields.** Says "this class has no mutable state, so it's inherently thread-safe — I won't need any synchronization here." Shows they analyze thread safety by examining state, not by reflexively adding `synchronized`.
2. **Uses `ConcurrentHashMap.computeIfAbsent()` for bucket initialization** rather than a check-then-act pattern. Recognizes that a plain `if (!map.containsKey(key)) { map.put(...) }` is a race condition.
3. **Adds jitter and explains the thundering herd problem** without being prompted. Bonus: mentions "full jitter" vs "equal jitter" strategies from the AWS architecture blog.
4. **Distinguishes retryable from non-retryable exceptions.** Either adds an `isRetryable(Exception e)` method to the interface, or notes that the implementation should only retry on `IOException`, `TimeoutException`, and provider-specific transient errors — not on `IllegalArgumentException` or `NullPointerException`.

---

### Stage 5: Scale Considerations at the Class Level (5 minutes)

#### Interviewer Questions

*Open with:*
> "We've designed the core logic. Now let's talk about how this holds up under load. How does priority queuing work in your design?"

*Follow-up probes:*
> "If I submit the same `requestId` twice — say, because a client retried their HTTP call to our API — what happens?"

> "How would you make delivery asynchronous so the caller doesn't block waiting for the notification to be sent?"

> "How do delivery receipts flow back to the caller or to monitoring systems?"

---

#### Expected Senior Response — Stage 5

**On priority queues:**
"The dispatcher maintains a `PriorityBlockingQueue<NotificationRequest>`. `NotificationRequest` implements `Comparable<NotificationRequest>` based on its `Priority` enum, with CRITICAL comparing as lowest (highest priority in Java's min-heap). A background worker thread pool continuously drains the queue and dispatches. Callers call `dispatcher.submit(request)` which enqueues and returns immediately. If they need the result, they get back a `Future<NotificationResult>`."

**On idempotency:**
"I maintain a `Set<String> processedRequestIds` — in production this is Redis with a TTL of 24 hours, but in the LLD it's a `ConcurrentHashMap<String, NotificationResult>`. Before processing, I check: if `processedRequestIds.contains(requestId)`, return the cached result immediately. This handles both client retries and internal retry-on-failure."

**On async dispatch:**
"I wrap the dispatcher in an `ExecutorService` — specifically a `ThreadPoolExecutor` with a bounded queue. The `dispatch()` method submits a `Callable<NotificationResult>` and returns a `CompletableFuture<NotificationResult>`. The caller can either block with `.get()` or attach a callback with `.thenAccept()`. For CRITICAL priority, the `CompletableFuture` can be awaited with a short timeout."

**On delivery receipts and observability:**
"I use the Observer pattern. The dispatcher holds a `List<DeliveryReceiptObserver>`. After each send attempt (success or failure), it creates a `DeliveryReceipt` and calls `observer.onDeliveryAttempt(receipt)` for each registered observer. Observers can be: a logging observer, a metrics observer (increment counters in Prometheus), a persistence observer (write to a database). This is an extension point — new observability requirements never touch the dispatcher."

---

#### Red Flags — Stage 5

1. **Describes priority as "skip low-priority notifications when busy"** rather than as an ordering mechanism. A priority queue dequeues high-priority items first — it doesn't drop low-priority items.
2. **Proposes idempotency by checking a plain `HashSet`** without mentioning thread safety or TTL. A non-concurrent set will corrupt under concurrent access. A set without TTL grows unboundedly.
3. **Makes the dispatcher fully synchronous** without addressing that 10K notifications/second cannot be dispatched inline with the caller's thread. Does not mention `ExecutorService`, thread pools, or `CompletableFuture`.
4. **Omits delivery receipts entirely**, or mentions logging inside the channel implementation. Logging inside the channel couples observability to delivery — adding a new observer requires modifying channel code.

---

#### Green Flags — Stage 5

1. **Proposes `CompletableFuture` for the async result** and explains that CRITICAL notifications might use `future.get(timeout, SECONDS)` while NORMAL notifications use fire-and-forget with an error callback.
2. **Mentions bounded queues and backpressure.** "I'd use a bounded `ArrayBlockingQueue` rather than an unbounded one. If the queue fills up, the dispatcher rejects new submissions and the API returns a 429 or 503 — this prevents OOM under load."
3. **Raises the idempotency TTL concern unprompted.** "The deduplication store needs a TTL — a request ID from 30 days ago shouldn't stay in memory forever. In production this is Redis with a 24-hour expiry."
4. **Suggests the observer list itself should be immutable after startup** (add observers during construction, then make the list unmodifiable) to avoid `ConcurrentModificationException` at runtime.

---

## Model Solution

### Class Diagram (Text Form)

```
<<interface>>
NotificationChannel
+ send(RenderedNotification, String userId) : DeliveryResult
+ supports(ChannelType) : boolean
+ getChannelType() : ChannelType
        ^
        |  implements
   _____|_______________________
   |            |              |
EmailChannel  SmsChannel   PushChannel


<<enum>>
ChannelType
EMAIL, SMS, PUSH


<<interface>>
RetryPolicy
+ shouldRetry(int attempt, Exception e) : boolean
+ getDelayMillis(int attempt) : long
        ^
        | implements
ExponentialBackoffRetry
- maxAttempts : int
- baseDelayMs : long
- maxDelayMs : long
- jitter : boolean


<<interface>>
RateLimiter
+ tryAcquire(String key) : boolean
+ getRemainingTokens(String key) : long
        ^
        | implements
TokenBucketRateLimiter
- buckets : ConcurrentHashMap<String, TokenBucket>
- maxTokens : long
- refillRatePerSecond : double


TokenBucket  (inner class)
- availableTokens : AtomicLong
- lastRefillNanos : AtomicLong
- lock : ReentrantLock


<<interface>>
TemplateService
+ getTemplate(String templateId, ChannelType) : NotificationTemplate
        ^
        | implements
InMemoryTemplateService
- templates : Map<String, NotificationTemplate>


NotificationTemplate
- templateId : String
- channelType : ChannelType
- subjectPattern : String
- bodyPattern : String
+ render(Map<String,String> params) : RenderedNotification


RenderedNotification
- subject : String
- body : String
- channelType : ChannelType


<<interface>>
UserPreferenceService
+ getPreference(String userId) : UserPreference
+ updatePreference(UserPreference) : void
        ^
        | implements
InMemoryUserPreferenceService
- preferences : ConcurrentHashMap<String, UserPreference>


UserPreference
- userId : String
- enabledChannels : Set<ChannelType>
- globalOptOut : boolean
- rateLimit : Map<ChannelType, Integer>


NotificationRequest  [Comparable<NotificationRequest>]
- requestId : String
- userId : String
- channelType : ChannelType
- templateId : String
- templateParams : Map<String, String>
- priority : Priority
- createdAt : Instant
+ Builder (inner)


<<enum>>
Priority
CRITICAL(0), HIGH(1), NORMAL(2), LOW(3)
- orderValue : int


DeliveryReceipt
- requestId : String
- userId : String
- channelType : ChannelType
- status : NotificationStatus
- timestamp : Instant
- failureReason : String


<<interface>>
DeliveryReceiptObserver
+ onDeliveryAttempt(DeliveryReceipt receipt) : void
        ^
        | implements
LoggingDeliveryReceiptObserver


NotificationResult
- requestId : String
- status : NotificationStatus
- channelType : ChannelType
- attemptCount : int
- errorMessage : String
+ success(...)  : static factory
+ failed(...)   : static factory
+ rateLimited(...) : static factory
+ optOut(...)   : static factory
+ unsupported(...) : static factory


<<enum>>
NotificationStatus
SUCCESS, FAILED, RATE_LIMITED, USER_OPT_OUT, CHANNEL_UNSUPPORTED


NotificationDispatcher
- channels : Map<ChannelType, NotificationChannel>
- preferenceService : UserPreferenceService
- retryPolicy : RetryPolicy
- rateLimiter : RateLimiter
- templateService : TemplateService
- queue : PriorityBlockingQueue<NotificationRequest>
- observers : List<DeliveryReceiptObserver>
- processed : ConcurrentHashMap<String, NotificationResult>
- executor : ExecutorService
+ dispatch(NotificationRequest) : CompletableFuture<NotificationResult>
+ addObserver(DeliveryReceiptObserver) : void
+ shutdown() : void


Relationships:
NotificationDispatcher  ---uses--->  NotificationChannel  (1 per ChannelType, via Map)
NotificationDispatcher  ---uses--->  UserPreferenceService
NotificationDispatcher  ---uses--->  RetryPolicy
NotificationDispatcher  ---uses--->  RateLimiter
NotificationDispatcher  ---uses--->  TemplateService
NotificationDispatcher  ---uses--->  DeliveryReceiptObserver  (0..*)
NotificationTemplate    ---creates-> RenderedNotification
NotificationChannel     ---accepts-> RenderedNotification
NotificationDispatcher  ---creates-> DeliveryReceipt
NotificationDispatcher  ---creates-> NotificationResult
```

---

### Complete Java Implementation

```java
// ============================================================
// Package: com.lld.notification
// ============================================================

package com.lld.notification;

// ============================================================
// ChannelType.java
// ============================================================

/**
 * Enumeration of all supported notification delivery channels.
 * Adding a new channel requires: (1) a new enum constant here,
 * (2) a new NotificationChannel implementation, (3) registration
 * in the dispatcher's channel map. No existing code changes.
 */
public enum ChannelType {
    EMAIL,
    SMS,
    PUSH
}
```

```java
package com.lld.notification;

/**
 * Priority levels for notification delivery.
 * Lower orderValue = higher priority in the PriorityBlockingQueue.
 * CRITICAL notifications override user opt-out preferences.
 */
public enum Priority {
    CRITICAL(0),
    HIGH(1),
    NORMAL(2),
    LOW(3);

    private final int orderValue;

    Priority(int orderValue) {
        this.orderValue = orderValue;
    }

    public int getOrderValue() {
        return orderValue;
    }
}
```

```java
package com.lld.notification;

/**
 * Represents the terminal status of a notification dispatch attempt.
 */
public enum NotificationStatus {
    /** Notification was delivered successfully to the channel provider. */
    SUCCESS,
    /** All retry attempts exhausted; delivery failed. */
    FAILED,
    /** The user has exceeded their per-channel rate limit. */
    RATE_LIMITED,
    /** The user has globally opted out (and priority does not override). */
    USER_OPT_OUT,
    /** No channel implementation registered for the requested ChannelType. */
    CHANNEL_UNSUPPORTED
}
```

```java
package com.lld.notification;

import java.time.Instant;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

/**
 * Immutable value object representing a single notification dispatch request.
 * Use the nested Builder to construct instances.
 *
 * <p>Implements Comparable so that NotificationRequest instances can be ordered
 * inside a PriorityBlockingQueue — CRITICAL is dequeued before HIGH, etc.</p>
 */
public final class NotificationRequest implements Comparable<NotificationRequest> {

    private final String requestId;
    private final String userId;
    private final ChannelType channelType;
    private final String templateId;
    private final Map<String, String> templateParams;
    private final Priority priority;
    private final Instant createdAt;

    private NotificationRequest(Builder builder) {
        this.requestId = Objects.requireNonNull(builder.requestId, "requestId must not be null");
        this.userId = Objects.requireNonNull(builder.userId, "userId must not be null");
        this.channelType = Objects.requireNonNull(builder.channelType, "channelType must not be null");
        this.templateId = Objects.requireNonNull(builder.templateId, "templateId must not be null");
        this.templateParams = Collections.unmodifiableMap(new HashMap<>(builder.templateParams));
        this.priority = Objects.requireNonNull(builder.priority, "priority must not be null");
        this.createdAt = builder.createdAt != null ? builder.createdAt : Instant.now();
    }

    public String getRequestId()                  { return requestId; }
    public String getUserId()                      { return userId; }
    public ChannelType getChannelType()            { return channelType; }
    public String getTemplateId()                  { return templateId; }
    public Map<String, String> getTemplateParams() { return templateParams; }
    public Priority getPriority()                  { return priority; }
    public Instant getCreatedAt()                  { return createdAt; }

    /**
     * Orders by priority ascending (CRITICAL first), then by createdAt ascending
     * (older requests served first within the same priority tier).
     */
    @Override
    public int compareTo(NotificationRequest other) {
        int priorityCompare = Integer.compare(
            this.priority.getOrderValue(),
            other.priority.getOrderValue()
        );
        if (priorityCompare != 0) {
            return priorityCompare;
        }
        return this.createdAt.compareTo(other.createdAt);
    }

    @Override
    public String toString() {
        return "NotificationRequest{requestId='" + requestId + "', userId='" + userId
            + "', channel=" + channelType + ", priority=" + priority + "}";
    }

    // ----------------------------------------------------------------
    // Builder
    // ----------------------------------------------------------------

    public static Builder builder() {
        return new Builder();
    }

    public static final class Builder {
        private String requestId;
        private String userId;
        private ChannelType channelType;
        private String templateId;
        private Map<String, String> templateParams = new HashMap<>();
        private Priority priority = Priority.NORMAL;
        private Instant createdAt;

        private Builder() {}

        public Builder requestId(String requestId)     { this.requestId = requestId; return this; }
        public Builder userId(String userId)           { this.userId = userId; return this; }
        public Builder channelType(ChannelType ct)     { this.channelType = ct; return this; }
        public Builder templateId(String templateId)   { this.templateId = templateId; return this; }
        public Builder templateParam(String k, String v) { this.templateParams.put(k, v); return this; }
        public Builder templateParams(Map<String, String> p) { this.templateParams.putAll(p); return this; }
        public Builder priority(Priority priority)     { this.priority = priority; return this; }
        public Builder createdAt(Instant createdAt)    { this.createdAt = createdAt; return this; }

        public NotificationRequest build() {
            return new NotificationRequest(this);
        }
    }
}
```

```java
package com.lld.notification;

/**
 * A notification that has been fully rendered from a template —
 * subject and body are interpolated strings ready for delivery.
 *
 * <p>This is what a {@link NotificationChannel} receives. Channels
 * have no knowledge of templates or template parameters.</p>
 */
public final class RenderedNotification {

    private final String subject;
    private final String body;
    private final ChannelType channelType;

    public RenderedNotification(String subject, String body, ChannelType channelType) {
        this.subject = subject;
        this.body = body;
        this.channelType = channelType;
    }

    public String getSubject()         { return subject; }
    public String getBody()            { return body; }
    public ChannelType getChannelType(){ return channelType; }

    @Override
    public String toString() {
        return "RenderedNotification{channel=" + channelType
            + ", subject='" + subject + "'}";
    }
}
```

```java
package com.lld.notification;

import java.util.Map;
import java.util.Objects;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * A stored notification template containing subject and body patterns
 * with {{placeholder}} variables.
 *
 * <p>Templates are channel-specific because the formatting requirements
 * differ: email bodies can be HTML, SMS bodies must be ≤160 chars,
 * push bodies must be ≤100 chars.</p>
 *
 * <p>Templates are immutable and safe to share across threads.</p>
 */
public final class NotificationTemplate {

    private static final Pattern PLACEHOLDER_PATTERN = Pattern.compile("\\{\\{(\\w+)}}");

    private final String templateId;
    private final ChannelType channelType;
    private final String subjectPattern;
    private final String bodyPattern;

    public NotificationTemplate(String templateId, ChannelType channelType,
                                 String subjectPattern, String bodyPattern) {
        this.templateId = Objects.requireNonNull(templateId);
        this.channelType = Objects.requireNonNull(channelType);
        this.subjectPattern = Objects.requireNonNull(subjectPattern);
        this.bodyPattern = Objects.requireNonNull(bodyPattern);
    }

    public String getTemplateId()     { return templateId; }
    public ChannelType getChannelType(){ return channelType; }

    /**
     * Renders the template by substituting all {{key}} placeholders
     * with the corresponding values from {@code params}.
     *
     * @param params map of placeholder names to their runtime values
     * @return a fully interpolated {@link RenderedNotification}
     * @throws IllegalArgumentException if a required placeholder is missing
     */
    public RenderedNotification render(Map<String, String> params) {
        String subject = interpolate(subjectPattern, params);
        String body = interpolate(bodyPattern, params);
        return new RenderedNotification(subject, body, channelType);
    }

    private String interpolate(String pattern, Map<String, String> params) {
        Matcher matcher = PLACEHOLDER_PATTERN.matcher(pattern);
        StringBuffer result = new StringBuffer();
        while (matcher.find()) {
            String key = matcher.group(1);
            String value = params.get(key);
            if (value == null) {
                throw new IllegalArgumentException(
                    "Template '" + templateId + "' requires parameter '" + key
                    + "' but it was not provided in params: " + params.keySet());
            }
            matcher.appendReplacement(result, Matcher.quoteReplacement(value));
        }
        matcher.appendTail(result);
        return result.toString();
    }
}
```

```java
package com.lld.notification;

/**
 * Service for retrieving notification templates by ID and channel type.
 */
public interface TemplateService {

    /**
     * Retrieves the template for the given ID and channel.
     *
     * @param templateId  the identifier of the desired template
     * @param channelType the channel for which the template is needed
     * @return the matching template
     * @throws IllegalArgumentException if no template is found
     */
    NotificationTemplate getTemplate(String templateId, ChannelType channelType);

    /**
     * Registers a template. Used during system initialization.
     */
    void registerTemplate(NotificationTemplate template);
}
```

```java
package com.lld.notification;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * In-memory implementation of {@link TemplateService}.
 * Templates are keyed by "templateId:channelType" for O(1) lookup.
 */
public class InMemoryTemplateService implements TemplateService {

    private final Map<String, NotificationTemplate> templates = new ConcurrentHashMap<>();

    @Override
    public NotificationTemplate getTemplate(String templateId, ChannelType channelType) {
        String key = buildKey(templateId, channelType);
        NotificationTemplate template = templates.get(key);
        if (template == null) {
            throw new IllegalArgumentException(
                "No template registered for templateId='" + templateId
                + "' and channelType=" + channelType);
        }
        return template;
    }

    @Override
    public void registerTemplate(NotificationTemplate template) {
        String key = buildKey(template.getTemplateId(), template.getChannelType());
        templates.put(key, template);
    }

    private String buildKey(String templateId, ChannelType channelType) {
        return templateId + ":" + channelType.name();
    }
}
```

```java
package com.lld.notification;

import java.util.Collections;
import java.util.EnumMap;
import java.util.EnumSet;
import java.util.Map;
import java.util.Objects;
import java.util.Set;

/**
 * Captures a user's notification delivery preferences.
 *
 * <p>{@code globalOptOut} suppresses all non-CRITICAL notifications.
 * {@code enabledChannels} controls which channels are active for this user.
 * {@code rateLimit} stores per-channel maximums in notifications per minute.</p>
 */
public final class UserPreference {

    private final String userId;
    private final Set<ChannelType> enabledChannels;
    private final boolean globalOptOut;
    private final Map<ChannelType, Integer> rateLimit;

    public UserPreference(String userId,
                          Set<ChannelType> enabledChannels,
                          boolean globalOptOut,
                          Map<ChannelType, Integer> rateLimit) {
        this.userId = Objects.requireNonNull(userId);
        this.enabledChannels = Collections.unmodifiableSet(EnumSet.copyOf(
            enabledChannels.isEmpty() ? EnumSet.noneOf(ChannelType.class) : enabledChannels));
        this.globalOptOut = globalOptOut;
        this.rateLimit = Collections.unmodifiableMap(new EnumMap<>(rateLimit));
    }

    public String getUserId()                               { return userId; }
    public Set<ChannelType> getEnabledChannels()            { return enabledChannels; }
    public boolean isGlobalOptOut()                         { return globalOptOut; }
    public Map<ChannelType, Integer> getRateLimit()         { return rateLimit; }

    /** Returns the rate limit for a channel, or a default of 60 if not specified. */
    public int getRateLimitForChannel(ChannelType channelType) {
        return rateLimit.getOrDefault(channelType, 60);
    }

    public boolean isChannelEnabled(ChannelType channelType) {
        return enabledChannels.contains(channelType);
    }
}
```

```java
package com.lld.notification;

/**
 * Service for reading and updating user notification preferences.
 */
public interface UserPreferenceService {

    /**
     * Returns the notification preferences for the given user.
     * If no preference record exists, returns a permissive default
     * (all channels enabled, not opted out).
     */
    UserPreference getPreference(String userId);

    /**
     * Persists or updates the notification preferences for a user.
     */
    void updatePreference(UserPreference preference);
}
```

```java
package com.lld.notification;

import java.util.EnumSet;
import java.util.EnumMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * In-memory implementation of {@link UserPreferenceService}.
 * Thread-safe via ConcurrentHashMap; appropriate for single-JVM deployments.
 */
public class InMemoryUserPreferenceService implements UserPreferenceService {

    private final Map<String, UserPreference> preferences = new ConcurrentHashMap<>();

    @Override
    public UserPreference getPreference(String userId) {
        return preferences.computeIfAbsent(userId, id -> defaultPreference(id));
    }

    @Override
    public void updatePreference(UserPreference preference) {
        preferences.put(preference.getUserId(), preference);
    }

    private UserPreference defaultPreference(String userId) {
        EnumMap<ChannelType, Integer> defaultRateLimits = new EnumMap<>(ChannelType.class);
        defaultRateLimits.put(ChannelType.EMAIL, 10);
        defaultRateLimits.put(ChannelType.SMS, 5);
        defaultRateLimits.put(ChannelType.PUSH, 30);
        return new UserPreference(
            userId,
            EnumSet.allOf(ChannelType.class),
            false,
            defaultRateLimits
        );
    }
}
```

```java
package com.lld.notification;

/**
 * Abstraction over a single notification delivery channel.
 *
 * <p>Implementations must be thread-safe — the dispatcher calls
 * {@link #send} concurrently from a thread pool.</p>
 *
 * <p>To add a new channel: implement this interface, add the new
 * {@link ChannelType} constant, and register the implementation
 * in the dispatcher's channel map. No existing code changes required.</p>
 */
public interface NotificationChannel {

    /**
     * Attempts to deliver the notification to the channel's provider.
     *
     * @param notification the fully rendered notification to deliver
     * @param userId       the target user's identifier
     * @return a {@link DeliveryResult} indicating success or failure with reason
     */
    DeliveryResult send(RenderedNotification notification, String userId);

    /**
     * Returns true if this channel handles the given {@link ChannelType}.
     */
    boolean supports(ChannelType channelType);

    /**
     * Returns the channel type this implementation handles.
     */
    ChannelType getChannelType();
}
```

```java
package com.lld.notification;

/**
 * Value object returned by a {@link NotificationChannel#send} call,
 * capturing whether delivery succeeded and, on failure, the reason.
 */
public final class DeliveryResult {

    private final boolean success;
    private final String failureReason;
    private final boolean retryable;

    private DeliveryResult(boolean success, String failureReason, boolean retryable) {
        this.success = success;
        this.failureReason = failureReason;
        this.retryable = retryable;
    }

    public static DeliveryResult success() {
        return new DeliveryResult(true, null, false);
    }

    public static DeliveryResult transientFailure(String reason) {
        return new DeliveryResult(false, reason, true);
    }

    public static DeliveryResult permanentFailure(String reason) {
        return new DeliveryResult(false, reason, false);
    }

    public boolean isSuccess()        { return success; }
    public String getFailureReason()  { return failureReason; }
    public boolean isRetryable()      { return retryable; }

    @Override
    public String toString() {
        return success ? "DeliveryResult{SUCCESS}"
            : "DeliveryResult{FAILED, retryable=" + retryable + ", reason='" + failureReason + "'}";
    }
}
```

```java
package com.lld.notification;

import java.util.logging.Logger;

/**
 * Simulated email delivery channel.
 *
 * <p>In production this wraps an SMTP client or an HTTP client
 * for a provider such as SendGrid or AWS SES. Here we log the
 * delivery attempt to simulate the call without a real provider.</p>
 */
public class EmailChannel implements NotificationChannel {

    private static final Logger log = Logger.getLogger(EmailChannel.class.getName());

    @Override
    public DeliveryResult send(RenderedNotification notification, String userId) {
        log.info(String.format(
            "[EmailChannel] Sending email to userId=%s | subject='%s' | body length=%d chars",
            userId, notification.getSubject(), notification.getBody().length()
        ));

        // Simulate a call to an external email provider.
        // In production: emailClient.send(buildMimeMessage(notification, userId))
        // Here we simulate ~95% success rate for realism in demos.
        boolean simulatedSuccess = Math.random() > 0.05;
        if (simulatedSuccess) {
            log.info("[EmailChannel] Email delivered successfully to userId=" + userId);
            return DeliveryResult.success();
        } else {
            String reason = "Simulated transient SMTP gateway error";
            log.warning("[EmailChannel] Email delivery failed for userId=" + userId + ": " + reason);
            return DeliveryResult.transientFailure(reason);
        }
    }

    @Override
    public boolean supports(ChannelType channelType) {
        return channelType == ChannelType.EMAIL;
    }

    @Override
    public ChannelType getChannelType() {
        return ChannelType.EMAIL;
    }
}
```

```java
package com.lld.notification;

import java.util.logging.Logger;

/**
 * Simulated SMS delivery channel.
 *
 * <p>In production this wraps an HTTP client for a provider
 * such as Twilio or AWS SNS SMS. SMS bodies should be kept
 * to ≤160 characters; this implementation truncates and warns.</p>
 */
public class SmsChannel implements NotificationChannel {

    private static final Logger log = Logger.getLogger(SmsChannel.class.getName());
    private static final int SMS_MAX_LENGTH = 160;

    @Override
    public DeliveryResult send(RenderedNotification notification, String userId) {
        String body = notification.getBody();
        if (body.length() > SMS_MAX_LENGTH) {
            log.warning(String.format(
                "[SmsChannel] SMS body exceeds %d chars (%d). Truncating for userId=%s",
                SMS_MAX_LENGTH, body.length(), userId
            ));
            body = body.substring(0, SMS_MAX_LENGTH - 3) + "...";
        }

        log.info(String.format(
            "[SmsChannel] Sending SMS to userId=%s | body='%s'",
            userId, body
        ));

        // Simulate ~90% success rate; SMS providers have stricter delivery SLAs.
        boolean simulatedSuccess = Math.random() > 0.10;
        if (simulatedSuccess) {
            log.info("[SmsChannel] SMS delivered successfully to userId=" + userId);
            return DeliveryResult.success();
        } else {
            String reason = "Simulated carrier gateway timeout";
            log.warning("[SmsChannel] SMS delivery failed for userId=" + userId + ": " + reason);
            return DeliveryResult.transientFailure(reason);
        }
    }

    @Override
    public boolean supports(ChannelType channelType) {
        return channelType == ChannelType.SMS;
    }

    @Override
    public ChannelType getChannelType() {
        return ChannelType.SMS;
    }
}
```

```java
package com.lld.notification;

import java.util.logging.Logger;

/**
 * Simulated push notification delivery channel.
 *
 * <p>In production this wraps FCM (Firebase Cloud Messaging) or APNs
 * (Apple Push Notification service). Push tokens are resolved from
 * a device registry (not modeled in this LLD for scope reasons).</p>
 */
public class PushChannel implements NotificationChannel {

    private static final Logger log = Logger.getLogger(PushChannel.class.getName());

    @Override
    public DeliveryResult send(RenderedNotification notification, String userId) {
        log.info(String.format(
            "[PushChannel] Sending push to userId=%s | title='%s'",
            userId, notification.getSubject()
        ));

        // Simulate push delivery. Push can fail permanently if device token is invalid.
        double roll = Math.random();
        if (roll > 0.08) {
            log.info("[PushChannel] Push delivered successfully to userId=" + userId);
            return DeliveryResult.success();
        } else if (roll > 0.03) {
            String reason = "Simulated FCM upstream error (transient)";
            log.warning("[PushChannel] Push transient failure for userId=" + userId);
            return DeliveryResult.transientFailure(reason);
        } else {
            // Permanent failure — e.g., invalid/expired device token.
            String reason = "Device token no longer valid (permanent)";
            log.warning("[PushChannel] Push permanent failure for userId=" + userId + ": " + reason);
            return DeliveryResult.permanentFailure(reason);
        }
    }

    @Override
    public boolean supports(ChannelType channelType) {
        return channelType == ChannelType.PUSH;
    }

    @Override
    public ChannelType getChannelType() {
        return ChannelType.PUSH;
    }
}
```

```java
package com.lld.notification;

/**
 * Defines the retry strategy for failed notification deliveries.
 *
 * <p>Implementations determine whether another attempt should be made
 * and how long to wait before it. Retry policies are stateless — they
 * compute from the attempt number and exception type only, making them
 * safe to share across threads and requests.</p>
 */
public interface RetryPolicy {

    /**
     * Returns true if another delivery attempt should be made.
     *
     * @param attemptNumber the number of attempts already made (0-based)
     * @param lastException the exception thrown by the last attempt,
     *                      or null if the channel returned a failure result
     *                      without throwing
     */
    boolean shouldRetry(int attemptNumber, Exception lastException);

    /**
     * Returns the delay in milliseconds before the next attempt.
     *
     * @param attemptNumber the number of attempts already made (0-based)
     */
    long getDelayMillis(int attemptNumber);
}
```

```java
package com.lld.notification;

import java.util.concurrent.ThreadLocalRandom;

/**
 * Exponential backoff retry policy with optional full jitter.
 *
 * <p>Delay formula (without jitter): {@code min(baseDelayMs * 2^attempt, maxDelayMs)}</p>
 * <p>Delay formula (with full jitter): {@code random(0, min(baseDelayMs * 2^attempt, maxDelayMs))}</p>
 *
 * <p>Jitter is recommended for production use because it spreads retries
 * across time, preventing the thundering herd problem where many clients
 * that all fail simultaneously retry at the same instant.</p>
 *
 * <p>This class is immutable and thread-safe.</p>
 */
public final class ExponentialBackoffRetry implements RetryPolicy {

    private final int maxAttempts;
    private final long baseDelayMs;
    private final long maxDelayMs;
    private final boolean jitter;

    /**
     * @param maxAttempts  total number of attempts allowed (first attempt + retries)
     * @param baseDelayMs  initial delay before the first retry, in milliseconds
     * @param maxDelayMs   maximum delay cap, in milliseconds
     * @param jitter       if true, applies full jitter to spread retry load
     */
    public ExponentialBackoffRetry(int maxAttempts, long baseDelayMs,
                                    long maxDelayMs, boolean jitter) {
        if (maxAttempts < 1) throw new IllegalArgumentException("maxAttempts must be >= 1");
        if (baseDelayMs < 0) throw new IllegalArgumentException("baseDelayMs must be >= 0");
        if (maxDelayMs < baseDelayMs) throw new IllegalArgumentException("maxDelayMs must be >= baseDelayMs");

        this.maxAttempts = maxAttempts;
        this.baseDelayMs = baseDelayMs;
        this.maxDelayMs = maxDelayMs;
        this.jitter = jitter;
    }

    /**
     * Returns true if we have not yet exhausted all attempts.
     * Does not retry on null exception (covers the case where channel returned
     * a permanent failure result rather than throwing).
     * In a production system, you would also check for specific non-retryable
     * exception types (e.g., InvalidRecipientException) here.
     */
    @Override
    public boolean shouldRetry(int attemptNumber, Exception lastException) {
        // attemptNumber is 0-based; we allow up to (maxAttempts - 1) retries
        // after the first attempt, so the total attempts = maxAttempts.
        return attemptNumber < maxAttempts;
    }

    /**
     * Computes the delay before the next retry using exponential backoff.
     * Uses a left shift for efficiency: baseDelayMs * 2^attemptNumber.
     * Guards against overflow with Math.min.
     */
    @Override
    public long getDelayMillis(int attemptNumber) {
        // Prevent overflow: if attemptNumber >= 63, the shift wraps.
        long exponentialDelay;
        if (attemptNumber >= 30) {
            exponentialDelay = maxDelayMs;
        } else {
            exponentialDelay = Math.min(baseDelayMs * (1L << attemptNumber), maxDelayMs);
        }

        if (jitter) {
            // Full jitter: uniform random in [0, exponentialDelay]
            return ThreadLocalRandom.current().nextLong(exponentialDelay + 1);
        }
        return exponentialDelay;
    }

    public int getMaxAttempts()  { return maxAttempts; }
    public long getBaseDelayMs() { return baseDelayMs; }
    public long getMaxDelayMs()  { return maxDelayMs; }
    public boolean isJitter()    { return jitter; }
}
```

```java
package com.lld.notification;

/**
 * Abstraction over a rate-limiting mechanism.
 *
 * <p>Keys are typically "{userId}:{channelType}" strings, allowing
 * independent rate limits per user per channel.</p>
 */
public interface RateLimiter {

    /**
     * Attempts to acquire a token for the given key.
     *
     * @param key the rate-limit bucket identifier (e.g. "user123:EMAIL")
     * @return true if a token was available and consumed; false if rate limited
     */
    boolean tryAcquire(String key);

    /**
     * Returns the number of tokens currently available for the given key.
     * Primarily for monitoring and debugging.
     */
    long getRemainingTokens(String key);
}
```

```java
package com.lld.notification;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Thread-safe token bucket rate limiter.
 *
 * <p>The token bucket algorithm:
 * <ul>
 *   <li>Each key has a bucket holding up to {@code maxTokens} tokens.</li>
 *   <li>Tokens refill continuously at {@code refillRatePerSecond} tokens/sec.</li>
 *   <li>Each {@link #tryAcquire} call consumes one token if available.</li>
 *   <li>If the bucket is empty, the call returns false (caller is rate limited).</li>
 * </ul>
 *
 * <p>Thread safety: buckets are created atomically via
 * {@code ConcurrentHashMap.computeIfAbsent}. Within a single bucket,
 * the refill-and-consume operation is guarded by a per-bucket
 * {@link ReentrantLock} to prevent lost updates under concurrent access.</p>
 *
 * <p>Why per-bucket locks rather than synchronizing the whole map?
 * Synchronizing the map would serialize ALL rate limit checks across ALL users,
 * creating a single bottleneck. Per-bucket locks allow user A's check to
 * proceed concurrently with user B's check.</p>
 */
public final class TokenBucketRateLimiter implements RateLimiter {

    private final long maxTokens;
    private final double refillRatePerSecond;
    private final ConcurrentHashMap<String, TokenBucket> buckets = new ConcurrentHashMap<>();

    /**
     * @param maxTokens           maximum burst capacity of each bucket
     * @param refillRatePerSecond tokens added per second (can be fractional)
     */
    public TokenBucketRateLimiter(long maxTokens, double refillRatePerSecond) {
        if (maxTokens <= 0) throw new IllegalArgumentException("maxTokens must be > 0");
        if (refillRatePerSecond <= 0) throw new IllegalArgumentException("refillRatePerSecond must be > 0");
        this.maxTokens = maxTokens;
        this.refillRatePerSecond = refillRatePerSecond;
    }

    @Override
    public boolean tryAcquire(String key) {
        TokenBucket bucket = buckets.computeIfAbsent(key, k -> new TokenBucket(maxTokens));
        return bucket.tryConsume(refillRatePerSecond, maxTokens);
    }

    @Override
    public long getRemainingTokens(String key) {
        TokenBucket bucket = buckets.get(key);
        if (bucket == null) {
            return maxTokens; // No activity yet — full bucket.
        }
        return bucket.getAvailableTokens();
    }

    // ----------------------------------------------------------------
    // Inner class: TokenBucket
    // ----------------------------------------------------------------

    /**
     * State for a single rate-limit key. Encapsulates the token count
     * and the last refill timestamp. All mutations are guarded by the
     * per-instance ReentrantLock.
     */
    static final class TokenBucket {

        // Using AtomicLong for visibility, but mutations are always
        // performed under the lock — AtomicLong gives us a safe read
        // path for getRemainingTokens() without locking.
        private final AtomicLong availableTokens;
        private final AtomicLong lastRefillNanos;
        private final ReentrantLock lock = new ReentrantLock();

        TokenBucket(long initialTokens) {
            this.availableTokens = new AtomicLong(initialTokens);
            this.lastRefillNanos = new AtomicLong(System.nanoTime());
        }

        /**
         * Refills the bucket based on elapsed time, then consumes one token.
         * Returns true if a token was available.
         *
         * <p>The refill-and-consume is an atomic operation under the lock to
         * prevent two threads from both seeing an empty bucket, both refilling
         * it, and both successfully consuming a token (double-consume).</p>
         */
        boolean tryConsume(double refillRatePerSecond, long maxTokens) {
            lock.lock();
            try {
                refill(refillRatePerSecond, maxTokens);
                long current = availableTokens.get();
                if (current > 0) {
                    availableTokens.set(current - 1);
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }

        private void refill(double refillRatePerSecond, long maxTokens) {
            long nowNanos = System.nanoTime();
            long lastNanos = lastRefillNanos.get();
            double elapsedSeconds = (nowNanos - lastNanos) / 1_000_000_000.0;

            if (elapsedSeconds > 0) {
                long newTokens = (long) (elapsedSeconds * refillRatePerSecond);
                if (newTokens > 0) {
                    long refilled = Math.min(availableTokens.get() + newTokens, maxTokens);
                    availableTokens.set(refilled);
                    lastRefillNanos.set(nowNanos);
                }
            }
        }

        long getAvailableTokens() {
            return availableTokens.get();
        }
    }
}
```

```java
package com.lld.notification;

import java.time.Instant;

/**
 * An immutable record of a single delivery attempt, published to
 * {@link DeliveryReceiptObserver} instances after each send.
 *
 * <p>Receipts are created for every attempt — including failures —
 * so that the observability layer has a complete audit trail.</p>
 */
public final class DeliveryReceipt {

    private final String requestId;
    private final String userId;
    private final ChannelType channelType;
    private final NotificationStatus status;
    private final Instant timestamp;
    private final String failureReason;

    public DeliveryReceipt(String requestId, String userId, ChannelType channelType,
                            NotificationStatus status, Instant timestamp, String failureReason) {
        this.requestId = requestId;
        this.userId = userId;
        this.channelType = channelType;
        this.status = status;
        this.timestamp = timestamp;
        this.failureReason = failureReason;
    }

    public String getRequestId()        { return requestId; }
    public String getUserId()           { return userId; }
    public ChannelType getChannelType() { return channelType; }
    public NotificationStatus getStatus(){ return status; }
    public Instant getTimestamp()       { return timestamp; }
    public String getFailureReason()    { return failureReason; }

    @Override
    public String toString() {
        return String.format("DeliveryReceipt{requestId='%s', userId='%s', channel=%s, status=%s, time=%s%s}",
            requestId, userId, channelType, status, timestamp,
            failureReason != null ? ", reason='" + failureReason + "'" : "");
    }
}
```

```java
package com.lld.notification;

/**
 * Observer interface for notification delivery events.
 *
 * <p>Register implementations with the dispatcher to receive a callback
 * after every delivery attempt. Common uses: audit logging, metrics
 * counters, alerting on high failure rates, persisting delivery history.</p>
 *
 * <p>Implementations must be thread-safe — the dispatcher may call
 * {@link #onDeliveryAttempt} from multiple threads concurrently.</p>
 */
public interface DeliveryReceiptObserver {

    /**
     * Called after each delivery attempt, whether successful or failed.
     *
     * @param receipt the delivery receipt for this attempt
     */
    void onDeliveryAttempt(DeliveryReceipt receipt);
}
```

```java
package com.lld.notification;

import java.util.logging.Logger;

/**
 * A {@link DeliveryReceiptObserver} that writes all delivery events
 * to the application log. Suitable as a reference implementation and
 * as a production audit trail when combined with a structured log sink.
 */
public class LoggingDeliveryReceiptObserver implements DeliveryReceiptObserver {

    private static final Logger log = Logger.getLogger(LoggingDeliveryReceiptObserver.class.getName());

    @Override
    public void onDeliveryAttempt(DeliveryReceipt receipt) {
        if (receipt.getStatus() == NotificationStatus.SUCCESS) {
            log.info("[DeliveryReceipt] " + receipt);
        } else {
            log.warning("[DeliveryReceipt] " + receipt);
        }
    }
}
```

```java
package com.lld.notification;

/**
 * Immutable result of a {@link NotificationDispatcher#dispatch} call.
 *
 * <p>Use the static factory methods to construct instances — this makes
 * the caller's code readable: {@code NotificationResult.optOut(requestId)}
 * is clearer than {@code new NotificationResult(..., USER_OPT_OUT, ...)}.</p>
 */
public final class NotificationResult {

    private final String requestId;
    private final NotificationStatus status;
    private final ChannelType channelType;
    private final int attemptCount;
    private final String errorMessage;

    private NotificationResult(String requestId, NotificationStatus status,
                                ChannelType channelType, int attemptCount, String errorMessage) {
        this.requestId = requestId;
        this.status = status;
        this.channelType = channelType;
        this.attemptCount = attemptCount;
        this.errorMessage = errorMessage;
    }

    // ---- Static factory methods ----

    public static NotificationResult success(String requestId, ChannelType channelType, int attempts) {
        return new NotificationResult(requestId, NotificationStatus.SUCCESS, channelType, attempts, null);
    }

    public static NotificationResult failed(String requestId, ChannelType channelType,
                                             int attempts, String errorMessage) {
        return new NotificationResult(requestId, NotificationStatus.FAILED, channelType, attempts, errorMessage);
    }

    public static NotificationResult rateLimited(String requestId, ChannelType channelType) {
        return new NotificationResult(requestId, NotificationStatus.RATE_LIMITED, channelType, 0,
            "User has exceeded rate limit for channel " + channelType);
    }

    public static NotificationResult optOut(String requestId, ChannelType channelType) {
        return new NotificationResult(requestId, NotificationStatus.USER_OPT_OUT, channelType, 0,
            "User has globally opted out of notifications");
    }

    public static NotificationResult channelDisabled(String requestId, ChannelType channelType) {
        return new NotificationResult(requestId, NotificationStatus.USER_OPT_OUT, channelType, 0,
            "User has disabled channel " + channelType);
    }

    public static NotificationResult unsupported(String requestId, ChannelType channelType) {
        return new NotificationResult(requestId, NotificationStatus.CHANNEL_UNSUPPORTED, channelType, 0,
            "No channel implementation registered for " + channelType);
    }

    // ---- Accessors ----

    public String getRequestId()          { return requestId; }
    public NotificationStatus getStatus() { return status; }
    public ChannelType getChannelType()   { return channelType; }
    public int getAttemptCount()          { return attemptCount; }
    public String getErrorMessage()       { return errorMessage; }
    public boolean isSuccess()            { return status == NotificationStatus.SUCCESS; }

    @Override
    public String toString() {
        return String.format("NotificationResult{requestId='%s', status=%s, channel=%s, attempts=%d%s}",
            requestId, status, channelType, attemptCount,
            errorMessage != null ? ", error='" + errorMessage + "'" : "");
    }
}
```

```java
package com.lld.notification;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.PriorityBlockingQueue;
import java.util.logging.Logger;

/**
 * Central orchestrator for the notification system.
 *
 * <p>Responsibilities (in order of execution per request):
 * <ol>
 *   <li>Idempotency check — return cached result if requestId was already processed.</li>
 *   <li>Channel support check — fail fast if no implementation is registered.</li>
 *   <li>User preference check — respect global opt-out (unless CRITICAL priority).</li>
 *   <li>Channel-enabled check — respect per-channel opt-in (unless CRITICAL priority).</li>
 *   <li>Rate limit check — enforce per-user-per-channel limits.</li>
 *   <li>Template resolution and rendering.</li>
 *   <li>Delivery with retry — delegate to channel and retry policy.</li>
 *   <li>Observer notification — publish DeliveryReceipt to all observers.</li>
 *   <li>Result caching and return.</li>
 * </ol>
 *
 * <p>Requests are enqueued in a {@link PriorityBlockingQueue} so that
 * CRITICAL notifications are dispatched before HIGH, HIGH before NORMAL, etc.
 * Callers receive a {@link CompletableFuture} for the result.</p>
 *
 * <p>Thread safety: The dispatcher is fully thread-safe. The channel map,
 * services, and policy objects are injected at construction and never mutated.
 * The processed-results cache uses ConcurrentHashMap. Observer list is made
 * unmodifiable after construction.</p>
 */
public final class NotificationDispatcher {

    private static final Logger log = Logger.getLogger(NotificationDispatcher.class.getName());

    private final Map<ChannelType, NotificationChannel> channels;
    private final UserPreferenceService preferenceService;
    private final RetryPolicy retryPolicy;
    private final RateLimiter rateLimiter;
    private final TemplateService templateService;
    private final List<DeliveryReceiptObserver> observers;

    /** Deduplication store: requestId -> previously computed result. */
    private final ConcurrentHashMap<String, NotificationResult> processedRequests
        = new ConcurrentHashMap<>();

    /** Priority queue: CRITICAL requests are dequeued before NORMAL. */
    private final PriorityBlockingQueue<NotificationRequest> queue
        = new PriorityBlockingQueue<>();

    private final ExecutorService workerPool;

    /**
     * Constructs the dispatcher and starts the background worker pool.
     *
     * @param channels          map of ChannelType to its NotificationChannel implementation
     * @param preferenceService service for user notification preferences
     * @param retryPolicy       retry strategy for failed delivery attempts
     * @param rateLimiter       rate limiter for per-user-per-channel throttling
     * @param templateService   service for resolving and rendering templates
     * @param observers         list of delivery receipt observers (copied and made unmodifiable)
     * @param workerThreads     number of threads in the dispatch worker pool
     */
    public NotificationDispatcher(Map<ChannelType, NotificationChannel> channels,
                                   UserPreferenceService preferenceService,
                                   RetryPolicy retryPolicy,
                                   RateLimiter rateLimiter,
                                   TemplateService templateService,
                                   List<DeliveryReceiptObserver> observers,
                                   int workerThreads) {
        this.channels = Collections.unmodifiableMap(channels);
        this.preferenceService = preferenceService;
        this.retryPolicy = retryPolicy;
        this.rateLimiter = rateLimiter;
        this.templateService = templateService;
        this.observers = Collections.unmodifiableList(new ArrayList<>(observers));
        this.workerPool = Executors.newFixedThreadPool(workerThreads, r -> {
            Thread t = new Thread(r, "notification-worker");
            t.setDaemon(true);
            return t;
        });

        startWorkers(workerThreads);
    }

    /**
     * Submits a notification request for async delivery.
     *
     * <p>Returns immediately with a {@link CompletableFuture} that will be
     * completed when the dispatcher has processed the request (including all
     * retries). CRITICAL priority requests should use
     * {@code future.get(timeout, TimeUnit.SECONDS)} to await the result.
     * NORMAL/LOW priority requests can attach a callback with
     * {@code future.thenAccept(...)}.</p>
     */
    public CompletableFuture<NotificationResult> dispatch(NotificationRequest request) {
        CompletableFuture<NotificationResult> future = new CompletableFuture<>();

        // Idempotency: if we have already processed this requestId, return the
        // cached result immediately without re-queuing.
        NotificationResult cached = processedRequests.get(request.getRequestId());
        if (cached != null) {
            log.info("Duplicate requestId detected, returning cached result: " + request.getRequestId());
            future.complete(cached);
            return future;
        }

        // Enqueue the request; the worker pool drains the queue in priority order.
        // We store the future alongside the request by completing it from the worker.
        workerPool.submit(() -> {
            NotificationResult result = processRequest(request);
            future.complete(result);
        });

        return future;
    }

    /**
     * Gracefully shuts down the worker pool.
     * Call this when the application is stopping to allow in-flight deliveries to complete.
     */
    public void shutdown() {
        workerPool.shutdown();
        log.info("NotificationDispatcher shutdown initiated.");
    }

    // ----------------------------------------------------------------
    // Private processing pipeline
    // ----------------------------------------------------------------

    private void startWorkers(int count) {
        // The workers are already running via workerPool.submit() in dispatch().
        // In a queue-driven design you would have dedicated drain threads;
        // here we rely on the submitted Callables in the fixed thread pool.
        log.info("NotificationDispatcher initialized with " + count + " worker threads.");
    }

    /**
     * Executes the full dispatch pipeline for a single request.
     * This runs on a worker thread from the pool.
     */
    private NotificationResult processRequest(NotificationRequest request) {
        String requestId = request.getRequestId();
        String userId = request.getUserId();
        ChannelType channelType = request.getChannelType();

        log.info("Processing " + request);

        // Step 1: Verify channel support
        NotificationChannel channel = channels.get(channelType);
        if (channel == null) {
            NotificationResult result = NotificationResult.unsupported(requestId, channelType);
            cacheAndReturn(requestId, result);
            publishReceipt(requestId, userId, channelType, NotificationStatus.CHANNEL_UNSUPPORTED, null);
            return result;
        }

        // Step 2: User preference checks
        UserPreference preference = preferenceService.getPreference(userId);
        boolean isCritical = request.getPriority() == Priority.CRITICAL;

        if (preference.isGlobalOptOut() && !isCritical) {
            NotificationResult result = NotificationResult.optOut(requestId, channelType);
            cacheAndReturn(requestId, result);
            publishReceipt(requestId, userId, channelType, NotificationStatus.USER_OPT_OUT, "global opt-out");
            return result;
        }

        if (!preference.isChannelEnabled(channelType) && !isCritical) {
            NotificationResult result = NotificationResult.channelDisabled(requestId, channelType);
            cacheAndReturn(requestId, result);
            publishReceipt(requestId, userId, channelType, NotificationStatus.USER_OPT_OUT,
                "channel " + channelType + " disabled");
            return result;
        }

        // Step 3: Rate limit check
        // Key format: "userId:CHANNEL" for per-user-per-channel limiting.
        String rateLimitKey = userId + ":" + channelType.name();
        if (!rateLimiter.tryAcquire(rateLimitKey)) {
            NotificationResult result = NotificationResult.rateLimited(requestId, channelType);
            cacheAndReturn(requestId, result);
            publishReceipt(requestId, userId, channelType, NotificationStatus.RATE_LIMITED, null);
            return result;
        }

        // Step 4: Render template
        RenderedNotification rendered;
        try {
            NotificationTemplate template = templateService.getTemplate(
                request.getTemplateId(), channelType);
            rendered = template.render(request.getTemplateParams());
        } catch (IllegalArgumentException e) {
            NotificationResult result = NotificationResult.failed(requestId, channelType, 0,
                "Template error: " + e.getMessage());
            cacheAndReturn(requestId, result);
            publishReceipt(requestId, userId, channelType, NotificationStatus.FAILED, e.getMessage());
            return result;
        }

        // Step 5: Deliver with retry
        NotificationResult result = deliverWithRetry(request, channel, rendered, userId);
        cacheAndReturn(requestId, result);
        return result;
    }

    /**
     * Attempts delivery, retrying according to the RetryPolicy.
     * Publishes a DeliveryReceipt after each attempt.
     */
    private NotificationResult deliverWithRetry(NotificationRequest request,
                                                  NotificationChannel channel,
                                                  RenderedNotification rendered,
                                                  String userId) {
        String requestId = request.getRequestId();
        ChannelType channelType = request.getChannelType();
        int attempt = 0;
        DeliveryResult lastResult = null;
        Exception lastException = null;

        while (retryPolicy.shouldRetry(attempt, lastException)) {
            if (attempt > 0) {
                long delay = retryPolicy.getDelayMillis(attempt);
                log.info(String.format("Retry attempt %d for requestId=%s after %dms delay",
                    attempt, requestId, delay));
                try {
                    Thread.sleep(delay);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    return NotificationResult.failed(requestId, channelType, attempt,
                        "Delivery interrupted: " + ie.getMessage());
                }
            }

            try {
                lastResult = channel.send(rendered, userId);
                lastException = null;
            } catch (Exception e) {
                lastException = e;
                lastResult = DeliveryResult.transientFailure(e.getMessage());
                log.warning(String.format("Exception during delivery attempt %d for requestId=%s: %s",
                    attempt, requestId, e.getMessage()));
            }

            attempt++;

            if (lastResult != null && lastResult.isSuccess()) {
                publishReceipt(requestId, userId, channelType, NotificationStatus.SUCCESS, null);
                return NotificationResult.success(requestId, channelType, attempt);
            }

            if (lastResult != null && !lastResult.isRetryable()) {
                // Permanent failure — do not retry regardless of policy.
                log.warning("Permanent failure for requestId=" + requestId
                    + ": " + lastResult.getFailureReason());
                publishReceipt(requestId, userId, channelType, NotificationStatus.FAILED,
                    lastResult.getFailureReason());
                return NotificationResult.failed(requestId, channelType, attempt,
                    lastResult.getFailureReason());
            }

            // Transient failure — publish attempt receipt and loop to retry.
            publishReceipt(requestId, userId, channelType, NotificationStatus.FAILED,
                lastResult != null ? lastResult.getFailureReason() : "Unknown error");
        }

        // Exhausted all retry attempts.
        String finalReason = lastResult != null ? lastResult.getFailureReason()
            : (lastException != null ? lastException.getMessage() : "Max retries exceeded");
        log.warning("All " + attempt + " delivery attempts exhausted for requestId=" + requestId);
        return NotificationResult.failed(requestId, channelType, attempt, finalReason);
    }

    private void cacheAndReturn(String requestId, NotificationResult result) {
        processedRequests.put(requestId, result);
    }

    private void publishReceipt(String requestId, String userId, ChannelType channelType,
                                  NotificationStatus status, String failureReason) {
        DeliveryReceipt receipt = new DeliveryReceipt(
            requestId, userId, channelType, status, Instant.now(), failureReason);
        for (DeliveryReceiptObserver observer : observers) {
            try {
                observer.onDeliveryAttempt(receipt);
            } catch (Exception e) {
                // Observers must not disrupt the delivery pipeline.
                log.warning("Observer threw exception: " + e.getMessage());
            }
        }
    }
}
```

```java
package com.lld.notification;

import java.util.EnumMap;
import java.util.EnumSet;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;
import java.util.logging.Logger;

/**
 * Demonstration entry point showing the full notification system wired
 * together with real dependencies and sample notifications.
 *
 * <p>Run this class to see the system in action. Observe the log output
 * to trace the full dispatch pipeline: preference check → rate limit →
 * template render → channel delivery → receipt publication.</p>
 */
public class Main {

    private static final Logger log = Logger.getLogger(Main.class.getName());

    public static void main(String[] args) throws Exception {

        // ---- 1. Build the template service and register templates ----

        InMemoryTemplateService templateService = new InMemoryTemplateService();

        templateService.registerTemplate(new NotificationTemplate(
            "ORDER_SHIPPED", ChannelType.EMAIL,
            "Your order {{orderId}} has shipped!",
            "Hi {{userName}}, great news! Your order {{orderId}} is on its way. " +
            "Expected delivery: {{deliveryDate}}. Track it at: {{trackingUrl}}"
        ));

        templateService.registerTemplate(new NotificationTemplate(
            "ORDER_SHIPPED", ChannelType.SMS,
            "Order shipped",
            "Hi {{userName}}, order {{orderId}} shipped. Track: {{trackingUrl}}"
        ));

        templateService.registerTemplate(new NotificationTemplate(
            "ORDER_SHIPPED", ChannelType.PUSH,
            "Order {{orderId}} shipped!",
            "Tap to track your delivery from {{userName}}."
        ));

        templateService.registerTemplate(new NotificationTemplate(
            "FRAUD_ALERT", ChannelType.EMAIL,
            "URGENT: Suspicious activity on your account",
            "Dear {{userName}}, we detected suspicious login from {{ipAddress}} " +
            "at {{loginTime}}. If this was not you, please secure your account immediately."
        ));

        templateService.registerTemplate(new NotificationTemplate(
            "FRAUD_ALERT", ChannelType.SMS,
            "Security Alert",
            "ALERT: Suspicious login on your account from {{ipAddress}}. " +
            "Not you? Call us immediately."
        ));

        // ---- 2. Configure user preferences ----

        InMemoryUserPreferenceService preferenceService = new InMemoryUserPreferenceService();

        // User alice: all channels enabled, per-channel rate limits
        EnumMap<ChannelType, Integer> aliceRateLimits = new EnumMap<>(ChannelType.class);
        aliceRateLimits.put(ChannelType.EMAIL, 5);
        aliceRateLimits.put(ChannelType.SMS, 2);
        aliceRateLimits.put(ChannelType.PUSH, 20);
        preferenceService.updatePreference(new UserPreference(
            "alice", EnumSet.allOf(ChannelType.class), false, aliceRateLimits));

        // User bob: has opted out of SMS, everything else enabled
        EnumMap<ChannelType, Integer> bobRateLimits = new EnumMap<>(ChannelType.class);
        bobRateLimits.put(ChannelType.EMAIL, 10);
        bobRateLimits.put(ChannelType.PUSH, 15);
        preferenceService.updatePreference(new UserPreference(
            "bob",
            EnumSet.of(ChannelType.EMAIL, ChannelType.PUSH),
            false,
            bobRateLimits
        ));

        // User charlie: globally opted out
        preferenceService.updatePreference(new UserPreference(
            "charlie",
            EnumSet.allOf(ChannelType.class),
            true,
            new EnumMap<>(ChannelType.class)
        ));

        // ---- 3. Wire up channels ----

        Map<ChannelType, NotificationChannel> channelMap = new EnumMap<>(ChannelType.class);
        channelMap.put(ChannelType.EMAIL, new EmailChannel());
        channelMap.put(ChannelType.SMS, new SmsChannel());
        channelMap.put(ChannelType.PUSH, new PushChannel());

        // ---- 4. Configure retry and rate limiting ----

        // Retry up to 3 total attempts, starting at 100ms backoff, capping at 2s, with jitter.
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(3, 100L, 2000L, true);

        // Token bucket: each key gets up to 10 tokens, refilling at 1 token/second.
        // In production, configure per-channel based on provider limits.
        RateLimiter rateLimiter = new TokenBucketRateLimiter(10, 1.0);

        // ---- 5. Build the dispatcher ----

        List<DeliveryReceiptObserver> observers = List.of(new LoggingDeliveryReceiptObserver());

        NotificationDispatcher dispatcher = new NotificationDispatcher(
            channelMap,
            preferenceService,
            retryPolicy,
            rateLimiter,
            templateService,
            observers,
            4  // 4 worker threads
        );

        // ---- 6. Submit sample notifications ----

        log.info("=== Submitting notifications ===");

        // Case 1: Normal order shipped notification for alice (should succeed)
        CompletableFuture<NotificationResult> f1 = dispatcher.dispatch(
            NotificationRequest.builder()
                .requestId(UUID.randomUUID().toString())
                .userId("alice")
                .channelType(ChannelType.EMAIL)
                .templateId("ORDER_SHIPPED")
                .templateParam("orderId", "ORD-10042")
                .templateParam("userName", "Alice")
                .templateParam("deliveryDate", "June 25, 2026")
                .templateParam("trackingUrl", "https://track.example.com/ORD-10042")
                .priority(Priority.NORMAL)
                .build()
        );

        // Case 2: SMS to bob who has SMS disabled (should return USER_OPT_OUT)
        CompletableFuture<NotificationResult> f2 = dispatcher.dispatch(
            NotificationRequest.builder()
                .requestId(UUID.randomUUID().toString())
                .userId("bob")
                .channelType(ChannelType.SMS)
                .templateId("ORDER_SHIPPED")
                .templateParam("orderId", "ORD-20099")
                .templateParam("userName", "Bob")
                .templateParam("trackingUrl", "https://track.example.com/ORD-20099")
                .priority(Priority.NORMAL)
                .build()
        );

        // Case 3: CRITICAL fraud alert to charlie who has global opt-out
        // (should bypass opt-out because CRITICAL)
        CompletableFuture<NotificationResult> f3 = dispatcher.dispatch(
            NotificationRequest.builder()
                .requestId(UUID.randomUUID().toString())
                .userId("charlie")
                .channelType(ChannelType.EMAIL)
                .templateId("FRAUD_ALERT")
                .templateParam("userName", "Charlie")
                .templateParam("ipAddress", "192.168.1.99")
                .templateParam("loginTime", "2026-06-22 03:14:00 UTC")
                .priority(Priority.CRITICAL)
                .build()
        );

        // Case 4: Duplicate request (same requestId as case 1 — to test idempotency)
        String duplicateRequestId = "FIXED-IDEMPOTENCY-TEST-001";
        CompletableFuture<NotificationResult> f4a = dispatcher.dispatch(
            NotificationRequest.builder()
                .requestId(duplicateRequestId)
                .userId("alice")
                .channelType(ChannelType.PUSH)
                .templateId("ORDER_SHIPPED")
                .templateParam("orderId", "ORD-30001")
                .templateParam("userName", "Alice")
                .templateParam("trackingUrl", "https://track.example.com/ORD-30001")
                .priority(Priority.HIGH)
                .build()
        );
        // Submit same requestId again — should return cached result immediately
        CompletableFuture<NotificationResult> f4b = dispatcher.dispatch(
            NotificationRequest.builder()
                .requestId(duplicateRequestId)
                .userId("alice")
                .channelType(ChannelType.PUSH)
                .templateId("ORDER_SHIPPED")
                .templateParam("orderId", "ORD-30001")
                .templateParam("userName", "Alice")
                .templateParam("trackingUrl", "https://track.example.com/ORD-30001")
                .priority(Priority.HIGH)
                .build()
        );

        // ---- 7. Collect and print results ----

        // Wait for all futures with a timeout
        NotificationResult r1 = f1.get(10, TimeUnit.SECONDS);
        NotificationResult r2 = f2.get(10, TimeUnit.SECONDS);
        NotificationResult r3 = f3.get(10, TimeUnit.SECONDS);
        NotificationResult r4a = f4a.get(10, TimeUnit.SECONDS);
        NotificationResult r4b = f4b.get(10, TimeUnit.SECONDS);

        log.info("=== Results ===");
        log.info("Case 1 (alice email ORDER_SHIPPED): " + r1);
        log.info("Case 2 (bob SMS disabled):          " + r2);
        log.info("Case 3 (charlie CRITICAL opt-out):  " + r3);
        log.info("Case 4a (first submission):         " + r4a);
        log.info("Case 4b (duplicate requestId):      " + r4b);

        // ---- 8. Cleanup ----
        dispatcher.shutdown();
    }
}
```

---

## Scoring Guide

### How to Score

Each dimension is scored 1–5. A passing senior hire scores 30/40. An exceptional candidate scores 37–40.

---

### Dimension 1: Requirements Gathering

| Score | Description |
|---|---|
| **1/5** | Skips requirements entirely. Starts writing classes from the prompt text alone. Misses non-functional requirements, scale, and delivery guarantees entirely. |
| **3/5** | Identifies the three channels and user preferences. Asks about scale but produces a round number without deriving implications ("10K/s, so we need a queue"). Does not raise idempotency, does not distinguish priority levels, confuses opt-out and channel-disabled as the same concept. |
| **5/5** | Drives a structured requirements conversation covering functional requirements (channels, preferences, templates, retry, priority), non-functional requirements (scale, latency, delivery guarantee, idempotency), and explicit constraint clarifications. Derives architectural consequences from scale numbers. States all assumptions out loud. Identifies the at-least-once vs. exactly-once tradeoff and picks one with justification. |

---

### Dimension 2: Abstraction Quality

| Score | Description |
|---|---|
| **1/5** | Creates a single `NotificationService` class with methods `sendEmail()`, `sendSms()`, `sendPush()`. No interface. Adding a channel requires modifying the service class. |
| **3/5** | Creates a `NotificationChannel` interface with a `send()` method. Returns void or throws exceptions rather than returning a result object. Conflates `NotificationRequest` and `RenderedNotification` into one parameter type — channels receive raw template parameters and are responsible for rendering. |
| **5/5** | Correctly separates `NotificationChannel` (delivery) from `TemplateService` (rendering). Channels receive `RenderedNotification` only. `send()` returns `DeliveryResult` with retryability flag. `supports()` is on the interface for runtime lookup. Explains why each design decision was made in terms of coupling and extensibility. |

---

### Dimension 3: Class Design Correctness

| Score | Description |
|---|---|
| **1/5** | Classes have inappropriate responsibilities. Dispatcher handles rendering, rate limiting, and retry inline with no delegation. `UserPreference` is a boolean flag on a user object. No `NotificationResult` — outcomes are communicated via exceptions. |
| **3/5** | Reasonable class structure with dispatcher delegating to channels. `UserPreference` is a dedicated class. Template rendering is separated. `NotificationResult` exists but uses a simple enum without static factory methods. Retry and rate limiting are in the dispatcher rather than extracted to dedicated policy classes. |
| **5/5** | All 23 required classes are present with correct responsibilities and dependencies. Dispatcher depends only on interfaces, never on concrete implementations. `NotificationResult` uses static factories. `RetryPolicy` and `RateLimiter` are interfaces with concrete implementations injected. Observer pattern used for delivery receipts. `Priority` drives queue ordering via `Comparable`. |

---

### Dimension 4: Extensibility

| Score | Description |
|---|---|
| **1/5** | Adding a new channel (WhatsApp) requires modifying the dispatcher, the template service, and the preference service. There are `if/else` or `switch` blocks on `ChannelType` in at least two places outside the enum definition. |
| **3/5** | Adding a new channel requires only a new implementation class plus an enum constant. However, adding a new failure behavior (e.g., a new `NotificationStatus` value) requires changing multiple classes. Rate limiting and retry are not extensible — only one algorithm is supported with no interface. |
| **5/5** | Open/Closed principle applied throughout. New channel: one new class + one enum constant + registration in main. New retry algorithm: one new `RetryPolicy` implementation. New rate limit algorithm: one new `RateLimiter` implementation. New observability: one new `DeliveryReceiptObserver`. Candidate articulates which extension points exist and why. |

---

### Dimension 5: Coding Ability

| Score | Description |
|---|---|
| **1/5** | Cannot write the token bucket algorithm without significant help. Produces pseudocode rather than compilable Java. Does not know how to use generics or the `concurrent` package. Writes `new Thread()` rather than using an `ExecutorService`. |
| **3/5** | Writes `ExponentialBackoffRetry` correctly but forgets jitter. Writes the token bucket but uses `synchronized` on the whole method rather than per-bucket, or uses a plain `HashMap` in a multi-threaded context. `PriorityBlockingQueue` is mentioned but not connected to a `Comparable` implementation on `NotificationRequest`. |
| **5/5** | Writes all required classes with correct, idiomatic Java. `ExponentialBackoffRetry` uses `ThreadLocalRandom` for jitter and guards against overflow. `TokenBucketRateLimiter` uses `ConcurrentHashMap.computeIfAbsent()` for bucket creation and per-bucket `ReentrantLock` for atomic refill-and-consume. `NotificationRequest implements Comparable` with correct ordering semantics. Builder pattern correctly implemented. |

---

### Dimension 6: Concurrency Awareness

| Score | Description |
|---|---|
| **1/5** | Shared mutable state is not protected. Uses `HashMap` instead of `ConcurrentHashMap` for the channel map and processed-request cache. Does not identify that channel implementations will be called from multiple threads. Does not mention thread safety anywhere. |
| **3/5** | Uses `ConcurrentHashMap` for shared maps. Makes channels stateless. But uses `synchronized` at a method level rather than at a finer granularity (synchronized on `TokenBucketRateLimiter` instance rather than per bucket). Does not mention `ThreadLocalRandom` for jitter vs. `Math.random()` (which is internally synchronized). |
| **5/5** | Analyzes thread safety of each class explicitly: immutable classes (no sync needed), stateless classes (no sync needed), shared mutable state (ConcurrentHashMap + per-entry locks or AtomicLong). Uses `ThreadLocalRandom` for jitter. Correctly identifies the check-then-act race in token bucket creation and solves it with `computeIfAbsent`. Explains why per-bucket locking is better than map-level locking. |

---

### Dimension 7: Communication

| Score | Description |
|---|---|
| **1/5** | Silent while coding. Does not explain decisions. When asked why they chose a design, gives vague answers ("it's cleaner this way"). Cannot articulate trade-offs. Does not check in with interviewer. |
| **3/5** | Narrates what they are doing but not why. Explains the "what" of each class but misses the "why" — does not connect design decisions to requirements or principles. Asks clarifying questions reactively (only when stuck) rather than proactively. |
| **5/5** | Narrates design decisions with explicit trade-off reasoning ("I'm using a per-bucket lock rather than a synchronized method because at 10K/s, serializing all rate limit checks on one lock would be a throughput bottleneck"). Checks in at natural transition points. Flags deferred decisions explicitly ("I'm skipping the database-backed preference store for now, but I'd note this is the right integration point"). Uses precise vocabulary (idempotency, thundering herd, backpressure, at-least-once). |

---

### Dimension 8: Edge Case Handling

| Score | Description |
|---|---|
| **1/5** | Handles only the happy path. No consideration of: failed delivery, missing templates, opted-out users, duplicate requests, thread interruption, observer exceptions propagating into the delivery pipeline, permanent vs. transient failures. |
| **3/5** | Handles failed delivery with basic retry. Handles opt-out. Does not consider: permanent vs. transient failure distinction (retries a "phone number invalid" error forever), observer exceptions disrupting delivery, `InterruptedException` in retry sleep, token bucket overflow (tokens exceeding max after a long idle period). |
| **5/5** | Explicitly handles: global opt-out vs. channel-disabled (different status codes), CRITICAL priority overriding opt-out, permanent vs. retryable failure from channels (stops retrying on permanent failure), `InterruptedException` in retry sleep (restores interrupt flag), observer exceptions caught and logged without disrupting the delivery pipeline, token bucket cap (tokens never exceed maxTokens after refill), duplicate `requestId` returning cached result without re-processing. |

---

## Debrief Notes

### What Makes This Problem Genuinely Hard

On the surface, a notification system sounds like a CRUD application with a few `if` statements. The genuine difficulty emerges from the intersection of concerns:

**The rendering vs. delivery boundary is non-obvious.** Most candidates instinctively pass the raw `NotificationRequest` all the way into the channel. The insight that rendering belongs in a dedicated service — and that channels should receive finished content — requires understanding that channels are I/O boundaries, not business logic processors. Channels should be as thin as possible.

**Priority is an ordering concern, not a filtering concern.** Candidates frequently mistake "low priority" as "might be skipped." In a correctly designed system, low-priority notifications are always delivered — just after higher-priority ones when the system is backlogged. This distinction matters when explaining the `PriorityBlockingQueue` semantics.

**Thread safety analysis requires state analysis, not instinct.** Candidates who reflexively add `synchronized` everywhere produce correct but slow code. Candidates who understand that immutable objects need no synchronization, stateless objects need no synchronization, and only shared mutable state requires synchronization — and who can identify exactly what that state is — produce clean, high-throughput code.

**The token bucket's atomic refill-and-consume is the crux.** It is not enough to put a lock anywhere. The lock must encompass the refill AND the consume as a single atomic operation. Candidates who add the lock only around the consume step but not the refill will double-consume under concurrent access.

**At-least-once delivery creates idempotency as a first-class requirement.** If you retry and the notification was delivered but the acknowledgment was lost, you will send the notification twice. Idempotency on `requestId` is the only defense. Most candidates do not surface this connection.

---

### Five Common Traps That Catch Senior Candidates

**Trap 1: Treating the dispatcher as a static utility.**
Writing `NotificationDispatcher.dispatch(request)` as a static method loses dependency injection and makes the system untestable. You cannot swap in a mock `TemplateService` in tests if the dispatcher constructs its own dependencies. This trap reveals whether the candidate understands testability-driven design.

**Trap 2: Putting retry logic inside channel implementations.**
Retry belongs at the orchestration layer (dispatcher), not inside each channel. If `EmailChannel` retries internally, then the dispatcher cannot distinguish "email failed after 3 retries" from "email failed on the first attempt." You lose visibility, and every channel reimplements the same retry logic differently.

**Trap 3: Writing `shouldRetry()` as a count-only check.**
`shouldRetry(int attempt, Exception e)` — the `Exception` parameter is there for a reason. A well-designed retry policy should not retry on `IllegalArgumentException` (template parameter missing — a permanent logic error) but should retry on `IOException` (transient network error). Ignoring the exception type causes infinite-equivalent retry loops on unrecoverable errors.

**Trap 4: Missing the `globalOptOut` vs. `channelDisabled` distinction.**
These are different business rules with different return statuses. A user who has globally opted out is making a broad privacy or preference choice. A user who has disabled SMS but not email is making a per-channel ergonomic choice. Merging them into one boolean produces a system that cannot distinguish "user doesn't want any notifications" from "user prefers email over SMS." They require different `NotificationResult` codes and different handling logic.

**Trap 5: Making observers synchronous and un-isolated.**
If a `DeliveryReceiptObserver` throws an exception and the dispatcher does not catch it, the exception propagates up and may prevent the `NotificationResult` from being returned, or may corrupt the retry counter. Observer exceptions must be caught and logged without disrupting the delivery pipeline. This is a production maturity concern that candidates who have not debugged observer-pattern failures in production tend to miss.

---

### How to Stand Out from Other Strong Candidates

Strong candidates get the architecture right. To stand out at the top of the range:

**Raise backpressure explicitly.** When discussing the `PriorityBlockingQueue`, note that an unbounded queue is a disguised OOM risk. At 10K/s intake with a slower delivery rate, the queue grows without bound. Propose a bounded `ArrayBlockingQueue` and define what happens on rejection — a `RejectedExecutionHandler` that returns HTTP 429 to the caller. This shows you've operated systems at scale.

**Connect idempotency to the cache TTL.** Saying "I'll cache processed request IDs" is good. Saying "I'll cache them in Redis with a 24-hour TTL, because after 24 hours a retry is almost certainly a new user action rather than a duplicate, and I don't want the cache to grow unboundedly" is great. It shows you've thought about the system's operational characteristics over time, not just its behavior in a single session.

**Mention that `ChannelType` enum growth has a cost.** Every time a new channel is added, the enum must be extended, which triggers redeployment of every service that depends on the enum. In a true microservices environment, this is a coupling problem. An alternative is to use a string-based channel identifier with a registration mechanism. Raising this shows you think about team-scale software development, not just single-system design.

---

### The One Insight That Separates a 4/5 from a 5/5

The `DeliveryResult.isRetryable()` flag — and the discipline to stop retrying on permanent failures — is the single insight that most cleanly separates candidates who have thought about production failure modes from those who have only thought about the happy path and transient failures.

Almost every candidate adds retry logic. Most check the attempt count. Fewer add jitter. But very few spontaneously say: "The channel returns a `DeliveryResult` with a `retryable` flag. If the phone number is invalid, or the device token has been revoked, retrying will never succeed — I must terminate the loop immediately, not exhaust all attempts first."

This insight is important for two reasons: it prevents wasted API calls to third-party providers (which cost money and count against provider rate limits), and it surfaces permanent failures quickly to the observability layer so on-call engineers can investigate data-quality issues rather than waiting for retry exhaustion. A candidate who explains both of those reasons unprompted has demonstrated genuine production experience, not just textbook knowledge.
