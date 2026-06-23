# Designing for Extensibility and Scalability at the Code Level

**Module:** Senior Engineer Design Thinking  
**Audience:** Engineers preparing for Senior SWE / Staff SWE roles  
**Language:** Java

---

## Table of Contents

1. [What Extensibility Means at the Code Level](#1-what-extensibility-means-at-the-code-level)
2. [Extension Points: The Toolkit](#2-extension-points-the-toolkit)
3. [Open/Closed Principle at Scale](#3-openclosed-principle-at-scale)
4. [Plugin Architecture Design](#4-plugin-architecture-design)
5. [Configuration-Driven Behavior](#5-configuration-driven-behavior)
6. [Backward Compatibility](#6-backward-compatibility)
7. [Senior vs Mid-Level Thinking: Comparison Table](#7-senior-vs-mid-level-thinking-comparison-table)
8. [Common Mistakes](#8-common-mistakes)
9. [Interview Perspective](#9-interview-perspective)

---

## 1. What Extensibility Means at the Code Level

### 1.1 Extensibility vs. Scalability: Clarifying the Terms

These two words are used interchangeably in casual conversation and almost always incorrectly. They solve different problems.

**Infrastructure scalability** is about handling more load: adding nodes, partitioning databases, tuning thread pools, introducing caches. It is an operational and architectural concern. A system can be infinitely scalable at the infrastructure level and still be a complete mess to change.

**Code-level scalability** is about how well the codebase handles the growth of requirements: new features, new business rules, new integrations, new team members. It is a software design concern. A system scales at the code level when the cost of adding the tenth feature is not ten times the cost of adding the first.

**Extensibility** is a specific dimension of code-level scalability. A system is extensible when new behavior can be added without modifying existing, tested code. The operative word is "without modifying." Extensibility is not about making code shorter. It is about making future change cheaper and safer.

Concretely:
- **Not extensible:** adding a new payment method requires modifying `PaymentProcessor`, `PaymentValidator`, `InvoiceGenerator`, and three downstream services.
- **Extensible:** adding a new payment method means implementing one interface and registering one class. No existing class is touched.

### 1.2 Why Extensibility Matters in Production Systems

**Cost of change.** Every time an engineer modifies an existing class to add new behavior, they incur risk: existing tests may break, edge cases in the original code may be disrupted, and the diff becomes harder to review. Over a codebase lifetime, this compounds. Senior engineers understand that the biggest long-term cost in software is not writing new code but changing existing code safely.

**Team velocity.** In a team of ten engineers, two may work on payment processing while three work on notifications. If both teams need to modify the same `OrderService` class to add their features, you get merge conflicts, coordination overhead, and a bottleneck at code review. Extensibility is a team scaling strategy as much as a technical one.

**Avoiding rewrites.** Most rewrites happen because a system was designed for specific cases rather than for a class of cases. "We need to support three more report formats and the current design can only handle CSV" is how rewrites begin. Extensibility defers that conversation indefinitely by design.

**Regression risk.** Modifying existing code requires re-running its test suite and trusting that tests cover all paths. Extensions added without touching existing code carry far lower regression risk because the existing code does not change.

### 1.3 The Coupling-Cohesion Tradeoff

These two forces govern every extensibility decision.

**Cohesion** measures how closely related the responsibilities within a single module are. A class that does exactly one thing and does it completely has high cohesion.

**Coupling** measures how much one module knows about or depends on another. Low coupling means modules can change independently.

The goal is high cohesion and low coupling simultaneously. This is harder than it sounds because the natural way humans write code tends toward low cohesion (doing many related things in one place) and high coupling (directly instantiating dependencies).

The tradeoff appears when introducing extension points: every abstraction you introduce (interface, abstract class, event, plugin contract) adds indirection. Indirection reduces coupling but adds complexity. The question is always: does the complexity of this abstraction pay for itself over the expected lifetime of the system?

A concrete rule: introduce an abstraction only when you have at least two concrete cases or when a third case is clearly foreseeable within the planning horizon. One concrete case does not justify an abstract class. Two concrete cases almost always do.

---

## 2. Extension Points: The Toolkit

An extension point is a place in a system where new behavior can be added without modifying existing code. Designing a system for extensibility means deliberately placing these points at the right seams.

### 2.1 Abstract Classes as Extension Points: Template Method Pattern

The Template Method pattern defines the skeleton of an algorithm in a base class and lets subclasses fill in specific steps. The invariant structure lives in the base class; the variant behavior lives in the subclasses.

Use this when:
- Multiple implementations share a fixed sequence of steps
- You want to enforce that certain steps always execute (e.g., logging, metrics, cleanup)
- The variant parts are clearly identifiable and bounded

```java
/**
 * Abstract base class defining the fixed pipeline for data ingestion.
 * Subclasses override specific steps to vary behavior without changing the overall flow.
 */
public abstract class DataIngestionPipeline {

    private static final Logger log = LoggerFactory.getLogger(DataIngestionPipeline.class);

    /**
     * Template method — defines the invariant sequence.
     * Marked final so subclasses cannot reorder or skip steps.
     */
    public final IngestionResult execute(IngestionRequest request) {
        log.info("Starting ingestion pipeline for source={}", request.getSourceId());
        long startMs = System.currentTimeMillis();

        try {
            RawPayload raw = fetch(request);
            ValidatedPayload validated = validate(raw);
            TransformedPayload transformed = transform(validated);
            PersistenceResult persisted = persist(transformed);
            onSuccess(persisted);
            return IngestionResult.success(persisted.getRecordCount());
        } catch (ValidationException e) {
            onValidationFailure(request, e);
            return IngestionResult.failure("VALIDATION_FAILED", e.getMessage());
        } catch (Exception e) {
            onUnexpectedFailure(request, e);
            return IngestionResult.failure("PIPELINE_ERROR", e.getMessage());
        } finally {
            long elapsedMs = System.currentTimeMillis() - startMs;
            log.info("Pipeline completed in {}ms for source={}", elapsedMs, request.getSourceId());
        }
    }

    /** Fetch raw data from the source. Each subclass knows its source protocol. */
    protected abstract RawPayload fetch(IngestionRequest request);

    /** Validate the raw payload against source-specific rules. */
    protected abstract ValidatedPayload validate(RawPayload raw) throws ValidationException;

    /** Transform the validated payload into the canonical internal model. */
    protected abstract TransformedPayload transform(ValidatedPayload validated);

    /** Persist the transformed payload. Default: writes to the primary data store. */
    protected PersistenceResult persist(TransformedPayload transformed) {
        // Default implementation — subclasses may override if they need custom persistence
        return PrimaryDataStore.getInstance().write(transformed);
    }

    /** Hook called on successful completion. Default: no-op. Override for custom notifications. */
    protected void onSuccess(PersistenceResult result) {}

    /** Hook called when validation fails. Default: logs the error. */
    protected void onValidationFailure(IngestionRequest request, ValidationException e) {
        log.warn("Validation failed for source={}: {}", request.getSourceId(), e.getMessage());
    }

    /** Hook called on unexpected pipeline failure. Default: logs the error. */
    protected void onUnexpectedFailure(IngestionRequest request, Exception e) {
        log.error("Unexpected failure for source={}", request.getSourceId(), e);
    }
}
```

Adding a new data source means creating a new subclass. No existing code is modified:

```java
public class S3CsvIngestionPipeline extends DataIngestionPipeline {

    private final S3Client s3Client;
    private final CsvSchemaRegistry schemaRegistry;

    public S3CsvIngestionPipeline(S3Client s3Client, CsvSchemaRegistry schemaRegistry) {
        this.s3Client = s3Client;
        this.schemaRegistry = schemaRegistry;
    }

    @Override
    protected RawPayload fetch(IngestionRequest request) {
        String bucket = request.getParam("bucket");
        String key = request.getParam("key");
        byte[] content = s3Client.getObject(bucket, key);
        return RawPayload.of(content, ContentType.CSV);
    }

    @Override
    protected ValidatedPayload validate(RawPayload raw) throws ValidationException {
        CsvSchema schema = schemaRegistry.getSchema(raw.getSchemaId());
        return CsvValidator.validate(raw, schema);
    }

    @Override
    protected TransformedPayload transform(ValidatedPayload validated) {
        return CsvTransformer.toCanonical(validated);
    }

    @Override
    protected void onSuccess(PersistenceResult result) {
        MetricsRegistry.counter("ingestion.s3.success").increment();
    }
}
```

### 2.2 Interfaces as Contracts

Interfaces define what a collaborator must be able to do, without saying anything about how it does it. This is the most fundamental extension point in Java.

The key discipline: keep interfaces narrow. An interface with twelve methods is not a contract — it is a specification for a specific class, expressed as an interface. A narrow interface like `MessageSender` with one method is a genuine contract that many classes can fulfil.

```java
/**
 * Contract for sending messages through any channel.
 * Implementors: EmailSender, SmsSender, SlackSender, PushNotificationSender.
 */
public interface MessageSender {
    /**
     * Send a message to the specified recipient.
     *
     * @param message the message to send
     * @throws MessageDeliveryException if the message cannot be delivered
     */
    void send(Message message) throws MessageDeliveryException;

    /**
     * Returns the channel identifier for this sender.
     * Used for routing and logging.
     */
    String getChannelName();
}
```

### 2.3 Hooks and Lifecycle Callbacks

Hooks are optional extension points — methods that default to a no-op but can be overridden or registered to inject behavior at specific lifecycle moments. They appear in frameworks (Spring's `BeanPostProcessor`, Hibernate's entity listeners) and are equally valuable in application code.

```java
/**
 * Listener interface for order lifecycle events.
 * Implementors receive callbacks at key moments in the order lifecycle.
 * Register implementations via OrderEventRegistry.
 */
public interface OrderLifecycleListener {
    default void onOrderCreated(Order order) {}
    default void onOrderConfirmed(Order order) {}
    default void onOrderShipped(Order order, ShipmentDetails shipment) {}
    default void onOrderCancelled(Order order, CancellationReason reason) {}
    default void onOrderCompleted(Order order) {}
}

public class OrderService {

    private final List<OrderLifecycleListener> listeners;
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository, List<OrderLifecycleListener> listeners) {
        this.orderRepository = orderRepository;
        this.listeners = new ArrayList<>(listeners);
    }

    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.from(request);
        orderRepository.save(order);
        listeners.forEach(l -> l.onOrderCreated(order));
        return order;
    }

    public void confirmOrder(String orderId) {
        Order order = orderRepository.findByIdOrThrow(orderId);
        order.confirm();
        orderRepository.save(order);
        listeners.forEach(l -> l.onOrderConfirmed(order));
    }

    public void cancelOrder(String orderId, CancellationReason reason) {
        Order order = orderRepository.findByIdOrThrow(orderId);
        order.cancel(reason);
        orderRepository.save(order);
        listeners.forEach(l -> l.onOrderCancelled(order, reason));
    }
}
```

Adding fraud detection, inventory adjustment, and email notification independently:

```java
public class FraudCheckListener implements OrderLifecycleListener {
    private final FraudDetectionService fraudService;

    @Override
    public void onOrderCreated(Order order) {
        FraudRiskScore score = fraudService.assess(order);
        if (score.isHighRisk()) {
            order.flagForReview();
        }
    }
}

public class InventoryReservationListener implements OrderLifecycleListener {
    private final InventoryService inventoryService;

    @Override
    public void onOrderConfirmed(Order order) {
        inventoryService.reserveItems(order.getLineItems());
    }

    @Override
    public void onOrderCancelled(Order order, CancellationReason reason) {
        inventoryService.releaseReservation(order.getLineItems());
    }
}
```

### 2.4 Event-Driven Extension: Observer and Event Bus

The observer pattern and event bus achieve the same goal as lifecycle listeners but at a lower coupling level — the publisher does not hold references to subscribers. The publisher fires an event into a bus; subscribers register interest in specific event types. Publisher and subscriber never see each other's types.

```java
// --- Event types ---

public abstract class DomainEvent {
    private final String eventId;
    private final Instant occurredAt;

    protected DomainEvent() {
        this.eventId = UUID.randomUUID().toString();
        this.occurredAt = Instant.now();
    }

    public String getEventId() { return eventId; }
    public Instant getOccurredAt() { return occurredAt; }
}

public class OrderCreatedEvent extends DomainEvent {
    private final Order order;
    public OrderCreatedEvent(Order order) { this.order = order; }
    public Order getOrder() { return order; }
}

// --- Event bus ---

public class DomainEventBus {
    private final Map<Class<?>, List<Consumer<DomainEvent>>> handlers = new ConcurrentHashMap<>();

    @SuppressWarnings("unchecked")
    public <T extends DomainEvent> void subscribe(Class<T> eventType, Consumer<T> handler) {
        handlers.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
                .add((Consumer<DomainEvent>) handler);
    }

    public void publish(DomainEvent event) {
        List<Consumer<DomainEvent>> eventHandlers = handlers.getOrDefault(event.getClass(), Collections.emptyList());
        for (Consumer<DomainEvent> handler : eventHandlers) {
            try {
                handler.accept(event);
            } catch (Exception e) {
                // Handlers are isolated — one failure does not prevent others
                log.error("Handler failed for event={}", event.getEventId(), e);
            }
        }
    }
}

// --- Publisher (knows nothing about subscribers) ---

public class OrderService {
    private final DomainEventBus eventBus;

    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.from(request);
        orderRepository.save(order);
        eventBus.publish(new OrderCreatedEvent(order));
        return order;
    }
}

// --- Subscribers (know nothing about each other or the publisher) ---

public class WelcomeEmailSubscriber {
    public WelcomeEmailSubscriber(DomainEventBus bus, EmailService emailService) {
        bus.subscribe(OrderCreatedEvent.class, event -> {
            emailService.sendOrderConfirmation(event.getOrder().getCustomerEmail(), event.getOrder());
        });
    }
}

public class AnalyticsSubscriber {
    public AnalyticsSubscriber(DomainEventBus bus, AnalyticsClient analyticsClient) {
        bus.subscribe(OrderCreatedEvent.class, event -> {
            analyticsClient.track("order_created", Map.of(
                "orderId", event.getOrder().getId(),
                "totalAmount", event.getOrder().getTotalAmount()
            ));
        });
    }
}
```

### 2.5 Full Example: Extensible Data Processing Pipeline with Pluggable Stages

This example brings together the toolkit into a cohesive design: a pipeline where stages are pluggable, ordered, and independently testable.

```java
// --- Stage contract ---

/**
 * A single stage in a data processing pipeline.
 * Each stage receives a processing context, may transform it, and passes it forward.
 *
 * @param <T> the type of payload flowing through the pipeline
 */
public interface PipelineStage<T> {
    /**
     * Execute this stage.
     *
     * @param context mutable context carrying the payload and metadata
     * @throws StageExecutionException if this stage cannot complete successfully
     */
    void execute(PipelineContext<T> context) throws StageExecutionException;

    /** Human-readable name for logging and metrics. */
    String getStageName();

    /**
     * Whether pipeline execution should halt if this stage fails.
     * Defaults to true — most stages are critical path.
     */
    default boolean isRequired() {
        return true;
    }
}

// --- Pipeline context ---

public class PipelineContext<T> {
    private T payload;
    private final Map<String, Object> attributes = new HashMap<>();
    private final List<String> executedStages = new ArrayList<>();
    private boolean aborted = false;

    public PipelineContext(T initialPayload) {
        this.payload = initialPayload;
    }

    public T getPayload() { return payload; }
    public void setPayload(T payload) { this.payload = payload; }

    public void setAttribute(String key, Object value) { attributes.put(key, value); }

    @SuppressWarnings("unchecked")
    public <V> V getAttribute(String key) { return (V) attributes.get(key); }

    public void markStageExecuted(String stageName) { executedStages.add(stageName); }
    public List<String> getExecutedStages() { return Collections.unmodifiableList(executedStages); }

    public void abort() { this.aborted = true; }
    public boolean isAborted() { return aborted; }
}

// --- Pipeline executor ---

public class DataProcessingPipeline<T> {

    private static final Logger log = LoggerFactory.getLogger(DataProcessingPipeline.class);

    private final String pipelineName;
    private final List<PipelineStage<T>> stages;

    private DataProcessingPipeline(Builder<T> builder) {
        this.pipelineName = builder.pipelineName;
        this.stages = Collections.unmodifiableList(new ArrayList<>(builder.stages));
    }

    public PipelineResult execute(T payload) {
        PipelineContext<T> context = new PipelineContext<>(payload);
        log.info("Starting pipeline={} with {} stages", pipelineName, stages.size());

        for (PipelineStage<T> stage : stages) {
            if (context.isAborted()) {
                log.info("Pipeline={} aborted before stage={}", pipelineName, stage.getStageName());
                break;
            }
            try {
                long start = System.currentTimeMillis();
                stage.execute(context);
                context.markStageExecuted(stage.getStageName());
                long elapsed = System.currentTimeMillis() - start;
                log.debug("Stage={} completed in {}ms", stage.getStageName(), elapsed);
            } catch (StageExecutionException e) {
                log.error("Stage={} failed: {}", stage.getStageName(), e.getMessage());
                if (stage.isRequired()) {
                    return PipelineResult.failure(stage.getStageName(), e.getMessage(), context.getExecutedStages());
                }
                // Optional stage: log and continue
                log.warn("Optional stage={} failed, continuing pipeline", stage.getStageName());
            }
        }

        return PipelineResult.success(context.getExecutedStages());
    }

    public static <T> Builder<T> builder(String pipelineName) {
        return new Builder<>(pipelineName);
    }

    public static class Builder<T> {
        private final String pipelineName;
        private final List<PipelineStage<T>> stages = new ArrayList<>();

        private Builder(String pipelineName) {
            this.pipelineName = pipelineName;
        }

        public Builder<T> addStage(PipelineStage<T> stage) {
            stages.add(stage);
            return this;
        }

        public DataProcessingPipeline<T> build() {
            if (stages.isEmpty()) {
                throw new IllegalStateException("Pipeline must have at least one stage");
            }
            return new DataProcessingPipeline<>(this);
        }
    }
}

// --- Concrete stages ---

public class DeduplicationStage implements PipelineStage<EventBatch> {
    private final DeduplicationCache cache;

    public DeduplicationStage(DeduplicationCache cache) {
        this.cache = cache;
    }

    @Override
    public void execute(PipelineContext<EventBatch> context) throws StageExecutionException {
        EventBatch batch = context.getPayload();
        List<Event> deduped = batch.getEvents().stream()
            .filter(event -> !cache.hasProcessed(event.getEventId()))
            .collect(Collectors.toList());
        cache.markProcessed(batch.getEvents().stream()
            .map(Event::getEventId)
            .collect(Collectors.toList()));
        context.setPayload(batch.withEvents(deduped));
        context.setAttribute("dedupedCount", batch.size() - deduped.size());
    }

    @Override
    public String getStageName() { return "deduplication"; }
}

public class SchemaValidationStage implements PipelineStage<EventBatch> {
    private final SchemaRegistry schemaRegistry;

    @Override
    public void execute(PipelineContext<EventBatch> context) throws StageExecutionException {
        EventBatch batch = context.getPayload();
        List<ValidationError> errors = new ArrayList<>();
        for (Event event : batch.getEvents()) {
            Schema schema = schemaRegistry.getSchema(event.getSchemaVersion());
            errors.addAll(schema.validate(event));
        }
        if (!errors.isEmpty()) {
            throw new StageExecutionException("Schema validation failed: " + errors.size() + " errors");
        }
    }

    @Override
    public String getStageName() { return "schema-validation"; }
}

public class EnrichmentStage implements PipelineStage<EventBatch> {
    private final UserProfileService profileService;

    @Override
    public void execute(PipelineContext<EventBatch> context) {
        EventBatch enriched = context.getPayload().getEvents().stream()
            .map(event -> event.withUserProfile(profileService.getProfile(event.getUserId())))
            .collect(EventBatch.collector());
        context.setPayload(enriched);
    }

    @Override
    public String getStageName() { return "enrichment"; }

    @Override
    public boolean isRequired() { return false; } // Enrichment failure is non-fatal
}

// --- Usage ---

DataProcessingPipeline<EventBatch> pipeline = DataProcessingPipeline.<EventBatch>builder("event-ingestion")
    .addStage(new DeduplicationStage(deduplicationCache))
    .addStage(new SchemaValidationStage(schemaRegistry))
    .addStage(new EnrichmentStage(userProfileService))
    .addStage(new PersistenceStage(eventStore))
    .build();

PipelineResult result = pipeline.execute(incomingBatch);
```

Adding a new stage — say, geo-IP resolution — means creating one class implementing `PipelineStage<EventBatch>` and adding it to the builder. Nothing existing is touched.

---

## 3. Open/Closed Principle at Scale

### 3.1 What OCP Actually Means in Production

The textbook formulation — "open for extension, closed for modification" — is often taught with inheritance examples that mislead. In production systems, OCP is primarily achieved through composition and dependency injection, not inheritance.

The practical meaning: the core business logic of a system should not need to be modified every time a new variant of behavior is needed. "Core business logic" typically means the orchestration layer — the class that coordinates steps — not the leaf implementations.

When you add a new discount type and only need to write a new `DiscountStrategy` class without touching `PriceCalculator`, `CheckoutService`, or `OrderValidator`, OCP is working.

When you add a new discount type and need to add an `if` branch to `PriceCalculator.calculateDiscount()`, OCP is violated.

### 3.2 When Violating OCP Is the Right Call

OCP has a cost: every extension point adds indirection. If your system has three discount types and you are confident it will never have a fourth, the overhead of a full strategy pattern — interface, registry, injection — may exceed its value.

Violate OCP consciously when:
- The number of variants is small and provably fixed (e.g., days of the week)
- The variant logic is a single line and lives in a lookup table or enum
- The abstraction required to make it OCP-compliant is significantly more complex than the problem it solves
- You are in a prototype or early discovery phase where the requirements are changing rapidly

The discipline is to violate it consciously and document why, not to stumble into the violation.

### 3.3 Strategy Pattern as OCP Enabler

The strategy pattern extracts a family of algorithms behind an interface and makes them injectable. The context class (the one that previously contained the algorithm logic) becomes the orchestrator; it calls a strategy without knowing which one.

### 3.4 Full Example: Pricing Engine Closed for Modification, Open for Extension

```java
// --- Discount strategy contract ---

/**
 * A discount strategy encapsulates a single discount rule.
 * The pricing engine applies all applicable strategies and accumulates discounts.
 */
public interface DiscountStrategy {
    /**
     * Determine whether this strategy applies to the given order.
     */
    boolean isApplicable(Order order, CustomerProfile customer);

    /**
     * Calculate the discount amount for the given order.
     * Called only if isApplicable returns true.
     *
     * @return discount amount as a non-negative value in the order's currency
     */
    Money calculateDiscount(Order order, CustomerProfile customer);

    /**
     * Human-readable name for audit logging.
     */
    String getStrategyName();

    /**
     * Priority for application order. Lower number = applied first.
     * Use when discount strategies must be applied in a specific sequence.
     */
    default int getPriority() {
        return 100;
    }
}

// --- Pricing engine: closed for modification ---

/**
 * Calculates the final price for an order by applying all applicable discount strategies.
 * This class is intentionally closed for modification. New discount rules are introduced
 * by adding DiscountStrategy implementations and registering them here.
 */
public class PricingEngine {

    private static final Logger log = LoggerFactory.getLogger(PricingEngine.class);

    private final List<DiscountStrategy> strategies;

    public PricingEngine(List<DiscountStrategy> strategies) {
        // Sort by priority to ensure deterministic application order
        this.strategies = strategies.stream()
            .sorted(Comparator.comparingInt(DiscountStrategy::getPriority))
            .collect(Collectors.toUnmodifiableList());
    }

    /**
     * Calculate the final price for an order.
     * All applicable strategies are evaluated; discounts are accumulated.
     * A discount cannot reduce the price below zero.
     */
    public PricingResult calculatePrice(Order order, CustomerProfile customer) {
        Money basePrice = order.getBasePrice();
        List<AppliedDiscount> appliedDiscounts = new ArrayList<>();

        for (DiscountStrategy strategy : strategies) {
            if (strategy.isApplicable(order, customer)) {
                Money discount = strategy.calculateDiscount(order, customer);
                appliedDiscounts.add(new AppliedDiscount(strategy.getStrategyName(), discount));
                log.debug("Applied discount strategy={} amount={}", strategy.getStrategyName(), discount);
            }
        }

        Money totalDiscount = appliedDiscounts.stream()
            .map(AppliedDiscount::getAmount)
            .reduce(Money.ZERO, Money::add);

        Money finalPrice = basePrice.subtract(totalDiscount).max(Money.ZERO);

        return new PricingResult(basePrice, finalPrice, totalDiscount, appliedDiscounts);
    }
}

// --- Concrete strategies: open for extension ---

public class LoyaltyTierDiscountStrategy implements DiscountStrategy {

    private static final Map<LoyaltyTier, BigDecimal> TIER_RATES = Map.of(
        LoyaltyTier.SILVER, new BigDecimal("0.05"),
        LoyaltyTier.GOLD, new BigDecimal("0.10"),
        LoyaltyTier.PLATINUM, new BigDecimal("0.15")
    );

    @Override
    public boolean isApplicable(Order order, CustomerProfile customer) {
        return customer.getLoyaltyTier() != LoyaltyTier.STANDARD;
    }

    @Override
    public Money calculateDiscount(Order order, CustomerProfile customer) {
        BigDecimal rate = TIER_RATES.getOrDefault(customer.getLoyaltyTier(), BigDecimal.ZERO);
        return order.getBasePrice().multiply(rate);
    }

    @Override
    public String getStrategyName() { return "LOYALTY_TIER"; }

    @Override
    public int getPriority() { return 10; }
}

public class BulkOrderDiscountStrategy implements DiscountStrategy {

    private static final int MINIMUM_ITEMS = 10;
    private static final BigDecimal DISCOUNT_RATE = new BigDecimal("0.08");

    @Override
    public boolean isApplicable(Order order, CustomerProfile customer) {
        return order.getTotalItemCount() >= MINIMUM_ITEMS;
    }

    @Override
    public Money calculateDiscount(Order order, CustomerProfile customer) {
        return order.getBasePrice().multiply(DISCOUNT_RATE);
    }

    @Override
    public String getStrategyName() { return "BULK_ORDER"; }

    @Override
    public int getPriority() { return 20; }
}

public class PromotionalCodeDiscountStrategy implements DiscountStrategy {

    private final PromotionalCodeRepository codeRepository;

    public PromotionalCodeDiscountStrategy(PromotionalCodeRepository codeRepository) {
        this.codeRepository = codeRepository;
    }

    @Override
    public boolean isApplicable(Order order, CustomerProfile customer) {
        return order.getPromotionalCode() != null
            && codeRepository.isValid(order.getPromotionalCode());
    }

    @Override
    public Money calculateDiscount(Order order, CustomerProfile customer) {
        PromotionalCode code = codeRepository.findByCode(order.getPromotionalCode());
        return code.calculateDiscount(order.getBasePrice());
    }

    @Override
    public String getStrategyName() { return "PROMOTIONAL_CODE"; }

    @Override
    public int getPriority() { return 30; }
}

// Adding a new first-time customer discount requires ONLY this class:
public class FirstTimePurchaseDiscountStrategy implements DiscountStrategy {

    private static final Money FIXED_DISCOUNT = Money.of(new BigDecimal("10.00"), Currency.USD);

    @Override
    public boolean isApplicable(Order order, CustomerProfile customer) {
        return customer.getTotalOrderCount() == 0;
    }

    @Override
    public Money calculateDiscount(Order order, CustomerProfile customer) {
        return FIXED_DISCOUNT.min(order.getBasePrice());
    }

    @Override
    public String getStrategyName() { return "FIRST_TIME_PURCHASE"; }

    @Override
    public int getPriority() { return 5; } // Apply before tier discounts
}

// --- Wiring (e.g. in a DI container) ---

List<DiscountStrategy> strategies = List.of(
    new FirstTimePurchaseDiscountStrategy(),
    new LoyaltyTierDiscountStrategy(),
    new BulkOrderDiscountStrategy(),
    new PromotionalCodeDiscountStrategy(codeRepository)
);
PricingEngine pricingEngine = new PricingEngine(strategies);
```

Notice that `PricingEngine` has not changed across four strategies and will not change when the fifth is added. The class is stable.

---

## 4. Plugin Architecture Design

A plugin architecture takes the strategy pattern one step further: plugins are discovered and loaded at runtime, often from external JARs. The core system defines contracts; plugin authors implement them in isolation.

### 4.1 Plugin Registration and Discovery

There are two models:

**Explicit registration:** Plugins are instantiated and registered in code or configuration. Simple and debuggable; requires recompilation or at least configuration changes to add a plugin.

**Automatic discovery via ServiceLoader:** Plugins declare themselves in `META-INF/services` descriptor files. The core system discovers them at startup without any code change. This is the Java-standard mechanism and is used by JDBC drivers, logging frameworks, and many Java ecosystem tools.

### 4.2 Plugin Contracts

The contract is an interface (or set of interfaces) that plugin authors implement. Keep contracts minimal. Every method you add to the contract is a burden on every plugin author and a source of incompatibility when the contract evolves.

### 4.3 Plugin Lifecycle Management

Plugins with resource ownership need lifecycle hooks: initialize on load, destroy on shutdown. A `PluginLifecycle` interface gives plugins this hook without requiring the core system to know anything about what resources each plugin manages.

### 4.4 Full Example: Report Generation System with Pluggable Formatters

```java
// --- Plugin contract ---

/**
 * Plugin contract for report formatters.
 * Implement this interface to add a new output format to the report generation system.
 *
 * <p>Implementations are discovered via Java's ServiceLoader mechanism.
 * To register a formatter plugin, create the file:
 * {@code META-INF/services/com.example.reports.ReportFormatter}
 * containing the fully qualified class name of your implementation.
 */
public interface ReportFormatter {

    /**
     * The MIME type produced by this formatter.
     * Used to match format requests and to set Content-Type in API responses.
     * Example: "application/json", "text/csv", "application/pdf"
     */
    String getMimeType();

    /**
     * The short format name used in API parameters and configuration.
     * Must be lowercase and contain only alphanumeric characters and hyphens.
     * Example: "json", "csv", "pdf", "excel"
     */
    String getFormatName();

    /**
     * Format the report data into a byte array in this formatter's output format.
     *
     * @param reportData the data to format
     * @param options    format-specific options (e.g., delimiter for CSV, paper size for PDF)
     * @return the formatted report as bytes
     * @throws FormattingException if the data cannot be formatted
     */
    byte[] format(ReportData reportData, FormatterOptions options) throws FormattingException;

    /**
     * Called once when the formatter is loaded. Implementors may initialize connections,
     * load templates, or allocate resources here.
     * Default: no-op.
     */
    default void initialize(FormatterConfig config) {}

    /**
     * Called on application shutdown. Release resources here.
     * Default: no-op.
     */
    default void destroy() {}
}

// --- Plugin registry ---

public class FormatterRegistry {

    private static final Logger log = LoggerFactory.getLogger(FormatterRegistry.class);

    private final Map<String, ReportFormatter> formattersByName = new ConcurrentHashMap<>();

    /**
     * Loads and registers all formatters discovered via ServiceLoader.
     * Called once at application startup.
     */
    public void loadPlugins(FormatterConfig config) {
        ServiceLoader<ReportFormatter> loader = ServiceLoader.load(ReportFormatter.class);
        for (ReportFormatter formatter : loader) {
            String name = formatter.getFormatName();
            log.info("Discovered formatter plugin: name={} type={}", name, formatter.getClass().getName());
            formatter.initialize(config);
            formattersByName.put(name, formatter);
        }
        log.info("Loaded {} formatter plugins", formattersByName.size());
    }

    /**
     * Register a formatter explicitly (for testing or programmatic registration).
     */
    public void register(ReportFormatter formatter, FormatterConfig config) {
        formatter.initialize(config);
        formattersByName.put(formatter.getFormatName(), formatter);
    }

    public ReportFormatter getFormatter(String formatName) {
        ReportFormatter formatter = formattersByName.get(formatName);
        if (formatter == null) {
            throw new UnsupportedFormatException("No formatter found for format: " + formatName
                + ". Available formats: " + formattersByName.keySet());
        }
        return formatter;
    }

    public Set<String> getSupportedFormats() {
        return Collections.unmodifiableSet(formattersByName.keySet());
    }

    public void shutdownAll() {
        formattersByName.values().forEach(f -> {
            try {
                f.destroy();
            } catch (Exception e) {
                log.error("Error destroying formatter={}", f.getFormatName(), e);
            }
        });
    }
}

// --- Report generation service: knows nothing about specific formats ---

public class ReportGenerationService {

    private final FormatterRegistry formatterRegistry;
    private final ReportDataService reportDataService;

    public ReportGenerationService(FormatterRegistry formatterRegistry,
                                   ReportDataService reportDataService) {
        this.formatterRegistry = formatterRegistry;
        this.reportDataService = reportDataService;
    }

    public GeneratedReport generateReport(ReportRequest request) throws FormattingException {
        ReportData data = reportDataService.buildReportData(request);
        ReportFormatter formatter = formatterRegistry.getFormatter(request.getFormatName());
        FormatterOptions options = FormatterOptions.from(request.getFormatOptions());

        byte[] content = formatter.format(data, options);

        return GeneratedReport.builder()
            .content(content)
            .mimeType(formatter.getMimeType())
            .formatName(formatter.getFormatName())
            .sizeBytes(content.length)
            .generatedAt(Instant.now())
            .build();
    }
}

// --- Built-in formatter implementations ---

public class JsonReportFormatter implements ReportFormatter {

    private final ObjectMapper objectMapper;

    public JsonReportFormatter() {
        this.objectMapper = new ObjectMapper()
            .enable(SerializationFeature.INDENT_OUTPUT)
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .registerModule(new JavaTimeModule());
    }

    @Override
    public String getMimeType() { return "application/json"; }

    @Override
    public String getFormatName() { return "json"; }

    @Override
    public byte[] format(ReportData data, FormatterOptions options) throws FormattingException {
        try {
            return objectMapper.writeValueAsBytes(data);
        } catch (JsonProcessingException e) {
            throw new FormattingException("JSON serialization failed", e);
        }
    }
}

public class CsvReportFormatter implements ReportFormatter {

    @Override
    public String getMimeType() { return "text/csv"; }

    @Override
    public String getFormatName() { return "csv"; }

    @Override
    public byte[] format(ReportData data, FormatterOptions options) throws FormattingException {
        String delimiter = options.getString("delimiter", ",");
        boolean includeHeaders = options.getBoolean("includeHeaders", true);

        StringBuilder csv = new StringBuilder();
        if (includeHeaders && !data.getColumns().isEmpty()) {
            csv.append(String.join(delimiter, data.getColumns())).append("\n");
        }
        for (List<String> row : data.getRows()) {
            csv.append(row.stream()
                .map(cell -> escapeCsvField(cell, delimiter))
                .collect(Collectors.joining(delimiter)))
                .append("\n");
        }
        return csv.toString().getBytes(StandardCharsets.UTF_8);
    }

    private String escapeCsvField(String field, String delimiter) {
        if (field == null) return "";
        if (field.contains(delimiter) || field.contains("\"") || field.contains("\n")) {
            return "\"" + field.replace("\"", "\"\"") + "\"";
        }
        return field;
    }
}

// --- Plugin loaded without recompilation via ServiceLoader ---
// In the plugin JAR, the PDF formatter is registered in:
// META-INF/services/com.example.reports.ReportFormatter
// Content: com.example.reports.plugins.pdf.PdfReportFormatter

public class PdfReportFormatter implements ReportFormatter {

    private PdfTemplate template;

    @Override
    public String getMimeType() { return "application/pdf"; }

    @Override
    public String getFormatName() { return "pdf"; }

    @Override
    public void initialize(FormatterConfig config) {
        String templatePath = config.getString("pdf.template.path", "templates/default.jrxml");
        this.template = PdfTemplate.load(templatePath);
    }

    @Override
    public byte[] format(ReportData data, FormatterOptions options) throws FormattingException {
        try {
            return template.render(data, options);
        } catch (TemplateRenderException e) {
            throw new FormattingException("PDF rendering failed", e);
        }
    }

    @Override
    public void destroy() {
        if (template != null) {
            template.close();
        }
    }
}
```

With `ServiceLoader`, adding the PDF formatter plugin JAR to the classpath is sufficient. No change to `ReportGenerationService`, `FormatterRegistry`, or any other class in the core system.

---

## 5. Configuration-Driven Behavior

### 5.1 When to Use Config vs. Code

The decision boundary: does this variation require logic, or just data?

**Use code (strategies, plugins) when:**
- The variant requires different algorithms or workflows
- The variant introduces new dependencies or resources
- The variant has its own error handling and edge cases
- The behavior cannot be expressed as a simple value

**Use configuration when:**
- The variant is a value (threshold, rate, URL, label)
- The behavior change is a switch between already-coded paths
- The change needs to be made by operations without a deploy cycle
- The change needs to be reversible quickly (feature flags)

A common mistake: encoding business logic in configuration that grows so complex it becomes an informal programming language. YAML with embedded conditionals is still code — it is just code with no type safety, no IDE support, and no tests. When your configuration reaches that complexity, move the logic back into code and use configuration only for the data.

### 5.2 Feature Flags at the Code Level

Feature flags are a code-level construct, not just an infrastructure one. At the code level, a feature flag is a named boolean (or enum) that gates a code path. The implementation discipline:

- Flags should be checked in exactly one place per feature
- Flag evaluation logic should be in a dedicated service, not scattered across the codebase
- Flags must be removed when the feature is fully rolled out — flag debt is real debt
- Never nest flags: `if (flagA && flagB)` is a maintenance hazard

### 5.3 Full Example: Notification System Driven by Configuration

```java
// --- Channel contract ---

public interface NotificationChannel {
    /**
     * Attempt to deliver the notification through this channel.
     *
     * @return true if delivered successfully, false if delivery failed but should not halt
     * @throws NotificationException if delivery failed and should halt processing
     */
    boolean deliver(Notification notification) throws NotificationException;

    String getChannelName();
}

// --- Feature flag service ---

public class FeatureFlagService {

    private final FeatureFlagConfig config;

    public FeatureFlagService(FeatureFlagConfig config) {
        this.config = config;
    }

    public boolean isEnabled(String flagName) {
        return config.getBoolean(flagName, false);
    }

    public boolean isEnabledForUser(String flagName, String userId) {
        if (!isEnabled(flagName)) return false;
        // Percentage rollout: stable hash ensures the same user always gets the same result
        double rolloutPercentage = config.getDouble(flagName + ".rollout.percentage", 100.0);
        int userHash = Math.abs(userId.hashCode() % 100);
        return userHash < rolloutPercentage;
    }
}

// --- Notification configuration model ---

public class NotificationChannelConfig {
    private final String channelName;
    private final boolean enabled;
    private final int maxRetries;
    private final Duration retryDelay;
    private final int priority; // lower = higher priority

    // constructors, getters omitted for brevity
}

// --- Notification service: behavior driven by configuration ---

public class NotificationService {

    private static final Logger log = LoggerFactory.getLogger(NotificationService.class);

    private final Map<String, NotificationChannel> channels;
    private final NotificationRoutingConfig routingConfig;
    private final FeatureFlagService featureFlagService;

    public NotificationService(List<NotificationChannel> channelList,
                               NotificationRoutingConfig routingConfig,
                               FeatureFlagService featureFlagService) {
        this.channels = channelList.stream()
            .collect(Collectors.toMap(NotificationChannel::getChannelName, c -> c));
        this.routingConfig = routingConfig;
        this.featureFlagService = featureFlagService;
    }

    public NotificationResult send(NotificationRequest request) {
        List<NotificationChannelConfig> eligibleChannels = resolveChannels(request);
        List<DeliveryRecord> deliveries = new ArrayList<>();

        for (NotificationChannelConfig channelConfig : eligibleChannels) {
            NotificationChannel channel = channels.get(channelConfig.getChannelName());
            if (channel == null) {
                log.warn("Channel configured but not registered: {}", channelConfig.getChannelName());
                continue;
            }

            DeliveryRecord record = deliverWithRetry(channel, request.toNotification(), channelConfig);
            deliveries.add(record);
        }

        return new NotificationResult(deliveries);
    }

    /**
     * Resolve which channels to use for this request.
     * Channels are enabled/disabled via configuration and feature flags.
     */
    private List<NotificationChannelConfig> resolveChannels(NotificationRequest request) {
        return routingConfig.getChannelsForCategory(request.getCategory())
            .stream()
            .filter(c -> c.isEnabled())
            .filter(c -> isChannelEnabledForUser(c, request.getUserId()))
            .sorted(Comparator.comparingInt(NotificationChannelConfig::getPriority))
            .collect(Collectors.toList());
    }

    private boolean isChannelEnabledForUser(NotificationChannelConfig config, String userId) {
        String flagName = "notification.channel." + config.getChannelName();
        // If no feature flag exists for this channel, treat as enabled
        if (!featureFlagService.isEnabled(flagName + ".enabled")) {
            return true; // Default: enabled if not flagged
        }
        return featureFlagService.isEnabledForUser(flagName, userId);
    }

    private DeliveryRecord deliverWithRetry(NotificationChannel channel,
                                             Notification notification,
                                             NotificationChannelConfig config) {
        int attempts = 0;
        Exception lastException = null;

        while (attempts <= config.getMaxRetries()) {
            try {
                boolean delivered = channel.deliver(notification);
                if (delivered) {
                    return DeliveryRecord.success(channel.getChannelName(), attempts + 1);
                }
                return DeliveryRecord.softFailure(channel.getChannelName(), attempts + 1, "Channel returned false");
            } catch (NotificationException e) {
                lastException = e;
                attempts++;
                if (attempts <= config.getMaxRetries()) {
                    log.warn("Delivery attempt {} failed for channel={}, retrying in {}",
                        attempts, channel.getChannelName(), config.getRetryDelay());
                    sleep(config.getRetryDelay());
                }
            }
        }

        log.error("Delivery failed after {} attempts for channel={}", attempts, channel.getChannelName());
        return DeliveryRecord.failure(channel.getChannelName(), attempts, lastException);
    }

    private void sleep(Duration duration) {
        try {
            Thread.sleep(duration.toMillis());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

// --- Configuration (e.g., YAML loaded at startup) ---
// notification.routing:
//   ORDER_CONFIRMATION:
//     channels:
//       - name: email
//         enabled: true
//         maxRetries: 3
//         retryDelay: 2s
//         priority: 1
//       - name: push
//         enabled: true
//         maxRetries: 1
//         retryDelay: 1s
//         priority: 2
//       - name: sms
//         enabled: false    <-- SMS disabled globally; no code change needed
//         maxRetries: 2
//         retryDelay: 5s
//         priority: 3
//
// feature.flags:
//   notification.channel.push.enabled: true
//   notification.channel.push.rollout.percentage: 20.0  <-- 20% rollout

// --- Channel implementations ---

public class EmailNotificationChannel implements NotificationChannel {
    private final EmailClient emailClient;
    private final EmailTemplateEngine templateEngine;

    @Override
    public boolean deliver(Notification notification) throws NotificationException {
        try {
            String html = templateEngine.render(notification.getTemplateId(), notification.getParams());
            emailClient.send(EmailMessage.builder()
                .to(notification.getRecipientEmail())
                .subject(notification.getSubject())
                .htmlBody(html)
                .build());
            return true;
        } catch (EmailClientException e) {
            throw new NotificationException("Email delivery failed", e);
        }
    }

    @Override
    public String getChannelName() { return "email"; }
}

public class SmsNotificationChannel implements NotificationChannel {
    private final SmsGatewayClient smsClient;

    @Override
    public boolean deliver(Notification notification) throws NotificationException {
        try {
            smsClient.sendSms(notification.getRecipientPhone(), notification.getShortMessage());
            return true;
        } catch (SmsGatewayException e) {
            throw new NotificationException("SMS delivery failed", e);
        }
    }

    @Override
    public String getChannelName() { return "sms"; }
}
```

Enabling SMS requires a single configuration change. Adding a new channel (Slack, WhatsApp) requires implementing `NotificationChannel` and adding the channel name to the routing configuration. `NotificationService` never changes.

---

## 6. Backward Compatibility

### 6.1 Why Backward Compatibility Is a Senior-Level Concern

Junior engineers think about the feature they are adding. Mid-level engineers think about the feature and the tests. Senior engineers think about the feature, the tests, and what happens to every caller of the code they are changing — including callers they cannot see.

In a monolith, "callers you cannot see" means other teams' code in the same repository. In a service-oriented architecture, it means other services, mobile clients, browser clients, and third-party integrations. In an open-source library, it means millions of users across a version spectrum you cannot control.

Breaking backward compatibility has a concrete cost: it forces every caller to update at the same time you release, or it silently breaks them at runtime. In a company running hundreds of microservices, a single incompatible change in a shared library can block dozens of teams' deployments simultaneously.

### 6.2 Semantic Versioning Applied to Library Design

Semantic versioning (MAJOR.MINOR.PATCH) is a communication protocol between library authors and consumers.

- **PATCH** (1.0.x): bug fixes only. Zero behavior changes for any caller following the documented contract.
- **MINOR** (1.x.0): additive changes only. New methods, new classes, new optional parameters. Existing callers are unaffected.
- **MAJOR** (x.0.0): breaking changes. Method signatures changed, classes removed, behaviors altered. Callers must explicitly opt in by upgrading the major version.

In practice, senior engineers treat every change to a public API as a potential MAJOR version bump and ask: "Is there any way to make this change without breaking existing callers?" Only when the answer is definitively no should a breaking change proceed.

### 6.3 Deprecation Strategy

Deprecation is not just adding `@Deprecated`. It is a three-step communication:

1. Mark the old method `@Deprecated` in the current release
2. Javadoc the replacement and the version when the old method will be removed
3. Keep the old method functional — it should still work, just flag a warning
4. Remove the method only in a MAJOR version bump, never sooner

```java
/**
 * @deprecated since 2.1.0. Use {@link #sendNotification(NotificationRequest)} instead.
 *             This method will be removed in version 3.0.0.
 *             Migration: wrap your parameters in a {@link NotificationRequest}:
 *             {@code sendNotification(NotificationRequest.of(userId, message, channel)) }
 */
@Deprecated(since = "2.1.0", forRemoval = true)
public boolean sendMessage(String userId, String message) {
    return sendNotification(NotificationRequest.of(userId, message, "email")).isSuccessful();
}
```

### 6.4 The Additive-Only Rule

The single most important backward compatibility principle: never remove or change the signature of a public method. Only add.

Why it exists: any class that extends or calls your public API compiles against it. If you remove a method or change its parameters, their code will no longer compile — or worse, will compile but silently call the wrong overload. The JVM's dynamic dispatch makes signature-changing failures particularly insidious.

Corollary: be conservative about what you make public. Every public method is a commitment. `public` means "I guarantee this interface forever." Prefer package-private until you know something must be public.

### 6.5 API Evolution Patterns

**Additive changes:** Add new methods, new parameters with defaults, new subclasses. Never remove or alter.

**Method overloading for new parameters:**
```java
// v1
public Report generateReport(String reportType, Date startDate, Date endDate);

// v2: add filter parameter without breaking v1 callers
public Report generateReport(String reportType, Date startDate, Date endDate);
public Report generateReport(String reportType, Date startDate, Date endDate, ReportFilter filter);
```

**Default methods in interfaces:** Java 8+ allows default method implementations in interfaces. Adding a default method to an interface does not break existing implementors. This is the canonical pattern for evolving interface contracts.

**Versioned types:** For request/response types, create parallel versioned classes rather than mutating existing ones.

### 6.6 Full Example: Evolving UserService API Across v1 and v2 Without Breaking Callers

```java
// =============================================================
// VERSION 1 — Initial release
// =============================================================

public interface UserService {
    /**
     * Retrieve a user by their unique identifier.
     *
     * @param userId the user's unique identifier
     * @return the user, or null if not found
     */
    User getUserById(String userId);

    /**
     * Create a new user account.
     *
     * @param request the creation request
     * @return the created user with assigned ID
     * @throws UserAlreadyExistsException if the email is already registered
     */
    User createUser(CreateUserRequest request) throws UserAlreadyExistsException;

    /**
     * Search for users by email address.
     *
     * @param email the email to search for
     * @return list of matching users (may be empty, never null)
     */
    List<User> searchByEmail(String email);
}

// =============================================================
// VERSION 2 — Requirements: Optional return type, pagination,
// multi-field search, audit metadata. Cannot break v1 callers.
// =============================================================

public interface UserService {

    /**
     * Retrieve a user by their unique identifier.
     *
     * @deprecated since 2.0.0. Use {@link #findUserById(String)} for Optional semantics.
     *             Returns null if not found; prefer the Optional variant.
     *             This method will be removed in 3.0.0.
     */
    @Deprecated(since = "2.0.0", forRemoval = true)
    User getUserById(String userId);

    /**
     * Retrieve a user by their unique identifier.
     *
     * @param userId the user's unique identifier
     * @return the user wrapped in Optional, empty if not found
     */
    Optional<User> findUserById(String userId);

    /**
     * Create a new user account.
     * This method is unchanged from v1; all v1 callers continue to work.
     */
    User createUser(CreateUserRequest request) throws UserAlreadyExistsException;

    /**
     * Create a user and return extended metadata (new in v2).
     * V1 callers that only need the User object continue to use createUser.
     */
    UserCreationResult createUserWithAudit(CreateUserRequest request) throws UserAlreadyExistsException;

    /**
     * Search for users by email address.
     *
     * @deprecated since 2.0.0. Use {@link #searchUsers(UserSearchRequest)} for pagination support.
     *             This method will be removed in 3.0.0.
     */
    @Deprecated(since = "2.0.0", forRemoval = true)
    List<User> searchByEmail(String email);

    /**
     * Search for users using a structured request (new in v2).
     * Supports multi-field search, pagination, and sorting.
     */
    PagedResult<User> searchUsers(UserSearchRequest request);

    /**
     * Deactivate a user account (new in v2).
     * Not present in v1 — v1 callers simply do not call this method.
     */
    void deactivateUser(String userId, DeactivationReason reason);

    /**
     * Check whether a user exists (new in v2).
     * Provided as a default method to avoid breaking existing implementations.
     */
    default boolean userExists(String userId) {
        return findUserById(userId).isPresent();
    }
}

// --- V2 implementation: supports both v1 and v2 callers ---

public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final AuditLogger auditLogger;

    public UserServiceImpl(UserRepository userRepository, AuditLogger auditLogger) {
        this.userRepository = userRepository;
        this.auditLogger = auditLogger;
    }

    @Override
    @Deprecated
    public User getUserById(String userId) {
        // Delegates to the v2 method; v1 callers transparently get v2 behavior
        return findUserById(userId).orElse(null);
    }

    @Override
    public Optional<User> findUserById(String userId) {
        return userRepository.findById(userId);
    }

    @Override
    public User createUser(CreateUserRequest request) throws UserAlreadyExistsException {
        return createUserWithAudit(request).getUser();
    }

    @Override
    public UserCreationResult createUserWithAudit(CreateUserRequest request) throws UserAlreadyExistsException {
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new UserAlreadyExistsException(request.getEmail());
        }
        User user = userRepository.save(User.from(request));
        AuditEntry audit = auditLogger.log(AuditAction.USER_CREATED, user.getId());
        return new UserCreationResult(user, audit);
    }

    @Override
    @Deprecated
    public List<User> searchByEmail(String email) {
        // Delegates to v2 method; maintains backward compatibility
        return searchUsers(UserSearchRequest.builder()
            .email(email)
            .page(0)
            .pageSize(Integer.MAX_VALUE)
            .build())
            .getContent();
    }

    @Override
    public PagedResult<User> searchUsers(UserSearchRequest request) {
        return userRepository.search(request);
    }

    @Override
    public void deactivateUser(String userId, DeactivationReason reason) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        user.deactivate(reason);
        userRepository.save(user);
        auditLogger.log(AuditAction.USER_DEACTIVATED, userId, Map.of("reason", reason.name()));
    }
}

// --- V1 callers: unchanged, continue to compile and work ---

public class LegacyUserController {
    private final UserService userService; // still works with UserServiceImpl

    public UserResponse getUser(String userId) {
        User user = userService.getUserById(userId); // deprecated but still works
        if (user == null) return UserResponse.notFound();
        return UserResponse.of(user);
    }

    public List<UserResponse> searchUsers(String email) {
        return userService.searchByEmail(email) // deprecated but still works
            .stream()
            .map(UserResponse::of)
            .collect(Collectors.toList());
    }
}

// --- V2 callers: use the new API ---

public class UserController {
    private final UserService userService;

    public UserResponse getUser(String userId) {
        return userService.findUserById(userId)
            .map(UserResponse::of)
            .orElse(UserResponse.notFound());
    }

    public PagedResponse<UserResponse> searchUsers(UserSearchParams params) {
        UserSearchRequest request = UserSearchRequest.from(params);
        PagedResult<User> result = userService.searchUsers(request);
        return PagedResponse.of(result.map(UserResponse::of));
    }
}
```

Both callers share the same `UserServiceImpl` instance. V1 callers are not broken; they see deprecation warnings at compile time, giving them a clear signal and migration path. V2 callers use the improved API.

---

## 7. Senior vs Mid-Level Thinking: Comparison Table

| Dimension | Mid-Level Decision | Senior Decision | Why It Matters |
|---|---|---|---|
| **Adding a new variant** | Add an `if/else if` block to the existing method | Create a new strategy or plugin implementation | The `if/else` grows without bound; each addition is a regression risk to all other branches. The strategy is isolated. |
| **Coupling to dependencies** | Directly instantiate `new EmailService()` inside the business class | Declare `MessageSender` interface; inject implementation via constructor | Direct instantiation makes the class impossible to test without real infrastructure and impossible to extend without modification. |
| **Backward compatibility** | Rename/modify a public method when requirements change | Add a new method alongside the old one; deprecate the old method with migration docs | Modifying a public method breaks every caller invisibly at compile time or silently at runtime. |
| **Configuration vs. code** | Hard-code thresholds (retry count = 3, timeout = 5s) as constants | Make thresholds configurable via application properties loaded at startup | Hard-coded values require a recompile and deploy to change; config changes can be done by ops without a code cycle. |
| **API surface area** | Make everything public because "it might be useful" | Default to package-private; promote to public only on demonstrated need | Every public method is a permanent commitment. Widening access is always possible; narrowing it is a breaking change. |
| **Abstraction timing** | Abstract on the first case "just in case" | Abstract at the second concrete case; the third makes the abstraction mandatory | Premature abstraction produces wrong abstractions. Two cases reveal the real commonality; one case is a guess. |
| **Error handling in extensible systems** | Catch all exceptions from plugins and swallow them silently | Isolate plugin errors; catch, log with context, and decide per-plugin whether to fail-fast or continue | Swallowed errors make production debugging impossible. Isolation ensures one bad plugin cannot corrupt the pipeline for all. |
| **Interface design** | Create a `UserManager` interface with 15 methods covering all user operations | Segregate into `UserReader`, `UserWriter`, `UserSearcher`; compose as needed | Large interfaces impose an all-or-nothing implementation burden. Small interfaces compose cleanly and allow partial mocking in tests. |
| **Extension point placement** | Design for current requirements; add extension later when needed | Identify seams (format, transport, rule evaluation) at design time; place contracts there | Retrofitting extension points requires touching existing code, which is the risk the pattern is meant to prevent. |
| **Naming of extension contracts** | Name after implementation: `EmailSenderStrategy`, `CsvFormatterPlugin` | Name after behavior/role: `NotificationChannel`, `ReportFormatter` | Implementation-named contracts constrain thinking. Behavior-named contracts invite broader implementations. |
| **Feature flag lifecycle** | Add a flag at rollout; forget to remove it afterward | Track flag creation with a removal deadline; treat flag removal as a required follow-up ticket | Flag accumulation produces tangled conditionals. Each flag that is never removed is permanent code complexity. |
| **Dependency on concrete types** | Accept `ArrayList<User>` as a parameter type | Accept `List<User>` or `Collection<User>` | Accepting a concrete type prevents callers from passing other list implementations and reveals an irrelevant implementation detail. |

---

## 8. Common Mistakes

### 8.1 Over-Engineering Extensibility (YAGNI Violation)

The most common senior engineer mistake is applying every pattern to every problem. Building a plugin registry, factory, strategy, and event bus for a system that has one client, one deployment, and three use cases is over-engineering. The code becomes harder to read, harder to debug, and harder to onboard new engineers.

YAGNI — You Aren't Gonna Need It — is not a license to write rigid code. It is a constraint on complexity: add extensibility when you have evidence it will be needed, not based on speculation.

Evidence you need extensibility:
- There are already two concrete cases in the current requirements
- The product roadmap explicitly calls for a third case in the next quarter
- The team context makes it likely (a platform team serving multiple consumer teams)

Speculation that does not justify extensibility:
- "We might add more payment types someday"
- "It would be cool if this were pluggable"
- "I've seen this pattern work well elsewhere"

### 8.2 Wrong Abstraction: Premature Generalization

Premature generalization is worse than no generalization because it locks you into the wrong abstraction. When you extract an interface from a single implementation, you are guessing at what the contract should be. When the second implementation arrives, it reveals that the contract was wrong and you must refactor both — more work than having waited.

The canonical symptom: an interface with methods that make sense for the original implementation but must be stubbed out with `throw new UnsupportedOperationException()` in the second.

```java
// Bad: abstracted from one implementation, second implementation reveals the mismatch
public interface ReportExporter {
    void setPageSize(PageSize pageSize);        // PDF concept; meaningless for CSV
    void setFontFamily(String fontFamily);     // PDF concept; meaningless for JSON
    void setDelimiter(char delimiter);         // CSV concept; meaningless for PDF
    byte[] export(ReportData data);
}

// Better: narrow interface with options bag
public interface ReportFormatter {
    byte[] format(ReportData data, FormatterOptions options);
    String getFormatName();
}
```

### 8.3 God Interfaces

An interface with more than five or six methods is almost always a design smell. It means one of two things: the interface has multiple responsibilities (split it) or the abstraction was designed from a single implementation (narrow it).

God interfaces cause:
- High implementation burden: every new formatter must implement 15 methods
- Test complexity: mocking 15 methods to test one behavior
- Coupling: changing the interface requires updating all implementations simultaneously

Apply Interface Segregation: split the god interface into narrow role interfaces. Callers depend only on the roles they use.

### 8.4 When to NOT Design for Extensibility

- **Scripts and one-off tools:** A script that runs once a month to generate a report does not need a plugin architecture. It needs to work.
- **Truly leaf-level logic:** A utility method that formats a date string does not need an extension point. It has one behavior; that behavior is correct.
- **Early stage / prototype:** When the requirements are changing every week, extensibility investments will be wasted when the direction pivots. Write the simplest thing that works; refactor toward extensibility when the design stabilizes.
- **Performance-critical paths:** Polymorphism and indirection have a cost. In tight loops processing millions of records, a direct call is faster than a virtual dispatch through an interface. Profile before you abstract.

The guiding principle: extensibility is a multiplier. It multiplies velocity when the design is correct and it multiplies complexity when the design is wrong. Apply it deliberately.

---

## 9. Interview Perspective

### 9.1 How to Demonstrate Extensibility Thinking in a Machine Coding Interview

Machine coding interviews typically give you 60-90 minutes to implement a system from a problem statement. Interviewers evaluating for senior roles are not looking for the most feature-complete solution. They are looking for evidence that you understand design at a professional level.

**Start with clarifying questions that reveal extensibility awareness:**
- "Should this support additional payment methods in the future, or are these three the complete set?"
- "Is the report format list fixed, or should the system accommodate new formats without code changes?"
- "Will the pricing rules be maintained by engineers or by a business configuration team?"

These questions signal to the interviewer that you are thinking about the system's life beyond the interview.

**Use named patterns but implement them, do not just name-drop them:**
Wrong: "I'll use the strategy pattern here." Then write a single class with three if-else branches.
Right: "I'll use the strategy pattern here." Then create `DiscountStrategy` interface, write two implementations, and wire them through a list.

**Show the seam explicitly:** When you define an interface, explain what it enables. "By defining `ReportFormatter` as an interface here, adding a PDF formatter in the future means implementing this interface and registering one class — nothing else changes."

**Write the wiring code:** Many candidates define the interface but leave the injection as a hand-wave. Write the constructor that takes `List<DiscountStrategy>`. Show how new strategies are added. The wiring is where the extensibility claim is proven.

**Demonstrate backward compatibility awareness if you evolve an API:**
If the problem asks you to extend existing functionality, ask: "Should I assume existing callers of this method are still working code I cannot change?" If yes, add a new method alongside the existing one rather than modifying it.

### 9.2 What Interviewers Look For When They Say "Make It Extensible"

"Make it extensible" in an interview is a specific challenge, not a vague aspiration. It means:

1. **Identify the variation point.** Where does the behavior change between cases? If you cannot name it precisely, you cannot design for it.

2. **Define a contract for that variation.** An interface or abstract class with a minimal, well-named contract.

3. **Centralize selection logic.** A registry, factory, or DI wiring point that maps a discriminator (format name, channel type, discount category) to an implementation. This logic should live in one place.

4. **Show that adding a new case requires only a new implementation.** Walk through the exercise: "To add a WhatsApp channel, I implement `NotificationChannel` and register it here. No existing class changes."

5. **Avoid the anti-pattern of showing extensibility by adding a second if-else branch.** Some candidates interpret "make it extensible" as "I should add another else-if for a hypothetical third case." That is not extensibility — that is continuation of the problem the pattern solves.

**The senior signal interviewers remember:** The candidate instinctively named the contract after the behavior role, not the implementation. They made the existing classes stable while showing clearly where future cases would live. They asked who owns the extension point — engineers or configuration — and let that answer drive the design.

---

*This document is part of the Senior Engineer Design Thinking module. Proceed to 02-identifying-abstractions.md for the next topic.*
