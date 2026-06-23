# Capstone Project 4: Event-Driven Workflow Engine (Staff Engineer Level)

## Project Overview

An enterprise-grade workflow engine that executes multi-step workflows in an event-driven manner. Workflows are defined via a fluent Java DSL, executed step by step, and support compensation (undo) for failure scenarios using the Saga pattern.

**Learning Objectives:**
- Apply Builder + Fluent API for expressive workflow DSL
- Apply Command pattern for executable, compensatable steps
- Apply Observer pattern with an in-process event bus
- Apply Chain of Responsibility for step executors
- Apply Strategy for retry policies
- Apply Memento for serializable workflow context
- Apply Saga pattern for distributed transaction compensation

**Difficulty:** Staff Engineer Level  
**Estimated Time:** 6-10 hours  
**Topics Covered:** Builder, Command, Observer, Chain of Responsibility, Strategy, Memento, Saga

---

## Requirements

### Functional Requirements
1. **Workflow DSL:** Fluent builder to define a sequence of steps with branching
2. **Step types:** TaskStep (sync), DecisionStep (conditional branch), ParallelStep (concurrent), LoopStep (repeat N times or while condition), SubWorkflowStep (nested workflow)
3. **Event-driven execution:** Each step completion publishes a `StepCompletedEvent`; the engine subscribes and triggers the next step
4. **Long-running workflows:** `WorkflowContext` is serializable between steps to simulate persistence
5. **Compensation:** Each step declares a `compensate()` method; on failure the engine rolls back completed steps in reverse order
6. **Workflow versioning:** `WorkflowDefinition` has a version field; instances record which version they executed
7. **Monitoring hooks:** `WorkflowLifecycleListener` called on start/complete/fail/compensate

### Non-Functional Requirements
1. Decouple step execution from orchestration via event bus (no direct method chaining)
2. Retry policy is per-step, not global
3. Context snapshot/restore enables resumable workflows
4. Thread-safe event bus for parallel step execution

---

## Class Diagram

```
WorkflowDefinition
  id, name, version
  List<WorkflowStep>
  startStepId
  Map<stepId → nextStepId>
  Map<stepId → failureStepId>
  Builder (fluent)

WorkflowInstance
  instanceId
  definitionId, version
  WorkflowStatus
  WorkflowContext
  List<StepExecution>

WorkflowContext (Memento)
  Map<String,Object> data
  snapshot() → ContextSnapshot
  restore(ContextSnapshot)

WorkflowEngine
  start(definition, initialData)
  resume(instance)
  ──────────────────────────────
  uses WorkflowEventBus
  uses StepExecutor
  uses CompensationManager

WorkflowEventBus (Observer)
  publish(WorkflowEvent)
  subscribe(listener)
  unsubscribe(listener)

WorkflowStep (Command interface)
  execute(WorkflowContext) → StepResult
  compensate(WorkflowContext)
  getRetryPolicy() → RetryPolicy

AbstractWorkflowStep
  ↑
  ├─ TaskStep        (wraps Function<Context,StepResult>)
  ├─ DecisionStep    (Predicate → trueStepId / falseStepId)
  ├─ ParallelStep    (List<WorkflowStep> + ExecutorService)
  ├─ LoopStep        (inner step, maxIterations, continueCondition)
  └─ SubWorkflowStep (nested WorkflowDefinition)

CompensationManager (Saga)
  push(stepId, step)
  executeCompensation(context) → reverses completed steps

RetryPolicy (Strategy)
  ├─ NoRetryPolicy
  └─ ExponentialBackoffRetryPolicy
```

---

## Design Patterns Applied

| Pattern | Where Applied | Benefit |
|---|---|---|
| Builder + Fluent API | `WorkflowDefinition.Builder` | Readable, compile-time-safe workflow definition |
| Command | `WorkflowStep.execute()` and `compensate()` | Steps are objects; can be queued, retried, reversed |
| Observer | `WorkflowEventBus` + `WorkflowEventListener` | Engine reacts to events without polling |
| Chain of Responsibility | `StepExecutor` applies retry, then delegates to step | Each concern handled at one layer |
| Strategy | `RetryPolicy` implementations | Swap retry logic without changing executor |
| Memento | `WorkflowContext.snapshot()` / `restore()` | Serialize state for resumable long-running workflows |
| Saga | `CompensationManager` reverses completed steps | Eventual consistency via compensating transactions |

---

## Complete Java Implementation

### Enumerations

```java
// StepStatus.java
package com.example.workflow;

/**
 * Lifecycle status of a single step execution.
 */
public enum StepStatus {
    PENDING,
    RUNNING,
    COMPLETED,
    FAILED,
    COMPENSATED,
    SKIPPED
}
```

```java
// WorkflowStatus.java
package com.example.workflow;

/**
 * Lifecycle status of a workflow instance.
 */
public enum WorkflowStatus {
    CREATED,
    RUNNING,
    COMPLETED,
    FAILED,
    COMPENSATING,
    COMPENSATED
}
```

```java
// WorkflowEventType.java
package com.example.workflow;

/**
 * All event types published to the WorkflowEventBus.
 */
public enum WorkflowEventType {
    WORKFLOW_STARTED,
    STEP_STARTED,
    STEP_COMPLETED,
    STEP_FAILED,
    WORKFLOW_COMPLETED,
    WORKFLOW_FAILED,
    COMPENSATION_STARTED,
    STEP_COMPENSATED,
    COMPENSATION_COMPLETED
}
```

### Workflow Context (Memento Pattern)

```java
// ContextSnapshot.java
package com.example.workflow;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

/**
 * Immutable snapshot of a WorkflowContext at a point in time.
 * Implements the Memento pattern: stores state without exposing internals.
 */
public final class ContextSnapshot {
    private final Map<String, Object> data;
    private final long snapshotTimestampMs;

    public ContextSnapshot(Map<String, Object> data) {
        this.data = Collections.unmodifiableMap(new HashMap<>(data));
        this.snapshotTimestampMs = System.currentTimeMillis();
    }

    public Map<String, Object> getData() {
        return data;
    }

    public long getSnapshotTimestampMs() {
        return snapshotTimestampMs;
    }

    @Override
    public String toString() {
        return "ContextSnapshot{data=" + data + ", ts=" + snapshotTimestampMs + "}";
    }
}
```

```java
// WorkflowContext.java
package com.example.workflow;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

/**
 * Mutable key-value context passed between steps.
 * Implements Memento via snapshot() and restore().
 *
 * Steps read input from context and write output back to context so that
 * subsequent steps can consume results without tight coupling.
 */
public class WorkflowContext {

    private final Map<String, Object> data = new HashMap<>();

    public WorkflowContext() {}

    public WorkflowContext(Map<String, Object> initialData) {
        if (initialData != null) {
            this.data.putAll(initialData);
        }
    }

    public void addData(String key, Object value) {
        if (key == null) throw new IllegalArgumentException("Context key cannot be null");
        data.put(key, value);
    }

    @SuppressWarnings("unchecked")
    public <T> Optional<T> getData(String key) {
        return Optional.ofNullable((T) data.get(key));
    }

    @SuppressWarnings("unchecked")
    public <T> T getDataOrThrow(String key) {
        Object value = data.get(key);
        if (value == null) {
            throw new IllegalStateException("Required context key not found: " + key);
        }
        return (T) value;
    }

    public boolean hasKey(String key) {
        return data.containsKey(key);
    }

    public Map<String, Object> getAllData() {
        return Collections.unmodifiableMap(data);
    }

    /**
     * Memento: capture current state as an immutable snapshot.
     */
    public ContextSnapshot snapshot() {
        return new ContextSnapshot(data);
    }

    /**
     * Memento: restore state from a previously captured snapshot.
     */
    public void restore(ContextSnapshot snapshot) {
        data.clear();
        data.putAll(snapshot.getData());
    }

    @Override
    public String toString() {
        return "WorkflowContext{data=" + data + "}";
    }
}
```

### Events and Event Bus

```java
// WorkflowEvent.java
package com.example.workflow;

import java.time.Instant;

/**
 * Immutable event published to the WorkflowEventBus.
 * Carries a snapshot of context at the moment the event fired.
 */
public final class WorkflowEvent {
    private final String workflowInstanceId;
    private final String stepId;
    private final WorkflowEventType eventType;
    private final Instant timestamp;
    private final ContextSnapshot contextSnapshot;
    private final String errorMessage;

    private WorkflowEvent(Builder builder) {
        this.workflowInstanceId = builder.workflowInstanceId;
        this.stepId = builder.stepId;
        this.eventType = builder.eventType;
        this.timestamp = Instant.now();
        this.contextSnapshot = builder.contextSnapshot;
        this.errorMessage = builder.errorMessage;
    }

    public String getWorkflowInstanceId() { return workflowInstanceId; }
    public String getStepId() { return stepId; }
    public WorkflowEventType getEventType() { return eventType; }
    public Instant getTimestamp() { return timestamp; }
    public ContextSnapshot getContextSnapshot() { return contextSnapshot; }
    public String getErrorMessage() { return errorMessage; }

    @Override
    public String toString() {
        return "WorkflowEvent{instanceId=" + workflowInstanceId
                + ", stepId=" + stepId
                + ", type=" + eventType
                + ", ts=" + timestamp
                + (errorMessage != null ? ", error=" + errorMessage : "")
                + "}";
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String workflowInstanceId;
        private String stepId;
        private WorkflowEventType eventType;
        private ContextSnapshot contextSnapshot;
        private String errorMessage;

        public Builder workflowInstanceId(String id) { this.workflowInstanceId = id; return this; }
        public Builder stepId(String stepId) { this.stepId = stepId; return this; }
        public Builder eventType(WorkflowEventType type) { this.eventType = type; return this; }
        public Builder contextSnapshot(ContextSnapshot snapshot) { this.contextSnapshot = snapshot; return this; }
        public Builder errorMessage(String msg) { this.errorMessage = msg; return this; }
        public WorkflowEvent build() {
            if (workflowInstanceId == null) throw new IllegalStateException("workflowInstanceId required");
            if (eventType == null) throw new IllegalStateException("eventType required");
            return new WorkflowEvent(this);
        }
    }
}
```

```java
// WorkflowEventListener.java
package com.example.workflow;

/**
 * Observer interface for receiving workflow events from the event bus.
 */
@FunctionalInterface
public interface WorkflowEventListener {
    void onEvent(WorkflowEvent event);
}
```

```java
// WorkflowEventBus.java
package com.example.workflow;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Simple synchronous in-process event bus.
 *
 * Uses CopyOnWriteArrayList so that subscribe/unsubscribe during event
 * dispatching does not cause ConcurrentModificationException.
 *
 * For async dispatch, listeners would be called via an ExecutorService;
 * that is left as an extension exercise.
 */
public class WorkflowEventBus {

    private final List<WorkflowEventListener> listeners = new CopyOnWriteArrayList<>();

    public void subscribe(WorkflowEventListener listener) {
        if (listener == null) throw new IllegalArgumentException("listener cannot be null");
        listeners.add(listener);
    }

    public void unsubscribe(WorkflowEventListener listener) {
        listeners.remove(listener);
    }

    public void publish(WorkflowEvent event) {
        if (event == null) throw new IllegalArgumentException("event cannot be null");
        for (WorkflowEventListener listener : listeners) {
            try {
                listener.onEvent(event);
            } catch (Exception e) {
                // Isolate listener failures so one bad listener does not block others
                System.err.println("[EventBus] Listener threw exception for event "
                        + event.getEventType() + ": " + e.getMessage());
            }
        }
    }

    public int listenerCount() {
        return listeners.size();
    }
}
```

### Lifecycle Listener

```java
// WorkflowLifecycleListener.java
package com.example.workflow;

/**
 * High-level lifecycle monitoring hooks.
 * Implement this to add logging, metrics, or alerting without touching engine code.
 */
public interface WorkflowLifecycleListener {
    void onWorkflowStarted(String instanceId, String definitionName, String version);
    void onWorkflowCompleted(String instanceId, WorkflowContext finalContext);
    void onWorkflowFailed(String instanceId, String failedStepId, Throwable cause);
    void onStepCompleted(String instanceId, String stepId, StepResult result);
    void onStepFailed(String instanceId, String stepId, Throwable cause);
    void onCompensationStarted(String instanceId);
    void onCompensationCompleted(String instanceId);
}
```

```java
// LoggingLifecycleListener.java
package com.example.workflow;

/**
 * Simple lifecycle listener that prints all events to stdout.
 * Useful for development and demos.
 */
public class LoggingLifecycleListener implements WorkflowLifecycleListener {

    @Override
    public void onWorkflowStarted(String instanceId, String definitionName, String version) {
        System.out.printf("[LIFECYCLE] Workflow STARTED   | instance=%s | name=%s | version=%s%n",
                instanceId, definitionName, version);
    }

    @Override
    public void onWorkflowCompleted(String instanceId, WorkflowContext finalContext) {
        System.out.printf("[LIFECYCLE] Workflow COMPLETED  | instance=%s | context=%s%n",
                instanceId, finalContext);
    }

    @Override
    public void onWorkflowFailed(String instanceId, String failedStepId, Throwable cause) {
        System.out.printf("[LIFECYCLE] Workflow FAILED     | instance=%s | failedStep=%s | cause=%s%n",
                instanceId, failedStepId, cause.getMessage());
    }

    @Override
    public void onStepCompleted(String instanceId, String stepId, StepResult result) {
        System.out.printf("[LIFECYCLE] Step COMPLETED      | instance=%s | step=%s | output=%s%n",
                instanceId, stepId, result.getOutput());
    }

    @Override
    public void onStepFailed(String instanceId, String stepId, Throwable cause) {
        System.out.printf("[LIFECYCLE] Step FAILED         | instance=%s | step=%s | cause=%s%n",
                instanceId, stepId, cause.getMessage());
    }

    @Override
    public void onCompensationStarted(String instanceId) {
        System.out.printf("[LIFECYCLE] COMPENSATION STARTED | instance=%s%n", instanceId);
    }

    @Override
    public void onCompensationCompleted(String instanceId) {
        System.out.printf("[LIFECYCLE] COMPENSATION DONE   | instance=%s%n", instanceId);
    }
}
```

### Step Result

```java
// StepResult.java
package com.example.workflow;

/**
 * Outcome of executing a single step.
 * Use the static factory methods rather than constructors.
 */
public final class StepResult {

    private final boolean success;
    private final Object output;
    private final String errorMessage;
    private final Throwable cause;

    private StepResult(boolean success, Object output, String errorMessage, Throwable cause) {
        this.success = success;
        this.output = output;
        this.errorMessage = errorMessage;
        this.cause = cause;
    }

    public static StepResult success() {
        return new StepResult(true, null, null, null);
    }

    public static StepResult success(Object output) {
        return new StepResult(true, output, null, null);
    }

    public static StepResult failure(String errorMessage) {
        return new StepResult(false, null, errorMessage, null);
    }

    public static StepResult failure(String errorMessage, Throwable cause) {
        return new StepResult(false, null, errorMessage, cause);
    }

    public boolean isSuccess() { return success; }
    public Object getOutput() { return output; }
    public String getErrorMessage() { return errorMessage; }
    public Throwable getCause() { return cause; }

    @Override
    public String toString() {
        if (success) {
            return "StepResult{SUCCESS, output=" + output + "}";
        } else {
            return "StepResult{FAILURE, error=" + errorMessage + "}";
        }
    }
}
```

### Retry Policies

```java
// RetryPolicy.java
package com.example.workflow.retry;

/**
 * Strategy interface for step retry behavior.
 * Called by StepExecutor after each failed attempt.
 */
public interface RetryPolicy {
    /**
     * Return true if the step should be retried after this attempt.
     * @param attempt 1-based attempt number (1 = first try, 2 = first retry)
     * @param cause   exception thrown by the step
     */
    boolean shouldRetry(int attempt, Exception cause);

    /**
     * Return the delay in milliseconds before the next retry attempt.
     * @param attempt 1-based attempt number of the attempt that just failed
     */
    long getDelayMs(int attempt);
}
```

```java
// NoRetryPolicy.java
package com.example.workflow.retry;

/**
 * No-op retry: always fails immediately with no retry.
 */
public class NoRetryPolicy implements RetryPolicy {

    @Override
    public boolean shouldRetry(int attempt, Exception cause) {
        return false;
    }

    @Override
    public long getDelayMs(int attempt) {
        return 0L;
    }
}
```

```java
// ExponentialBackoffRetryPolicy.java
package com.example.workflow.retry;

/**
 * Retries up to maxAttempts times with exponential backoff.
 *
 * Delay formula: min(baseDelayMs * 2^(attempt-1), maxDelayMs)
 *
 * Example with baseDelayMs=100, maxDelayMs=10000, maxAttempts=5:
 *   Attempt 1 fails → delay 100ms  → retry
 *   Attempt 2 fails → delay 200ms  → retry
 *   Attempt 3 fails → delay 400ms  → retry
 *   Attempt 4 fails → delay 800ms  → retry
 *   Attempt 5 fails → no more retries
 */
public class ExponentialBackoffRetryPolicy implements RetryPolicy {

    private final int maxAttempts;
    private final long baseDelayMs;
    private final long maxDelayMs;

    public ExponentialBackoffRetryPolicy(int maxAttempts, long baseDelayMs, long maxDelayMs) {
        if (maxAttempts < 1) throw new IllegalArgumentException("maxAttempts must be >= 1");
        if (baseDelayMs < 0) throw new IllegalArgumentException("baseDelayMs must be >= 0");
        if (maxDelayMs < baseDelayMs) throw new IllegalArgumentException("maxDelayMs must be >= baseDelayMs");
        this.maxAttempts = maxAttempts;
        this.baseDelayMs = baseDelayMs;
        this.maxDelayMs = maxDelayMs;
    }

    @Override
    public boolean shouldRetry(int attempt, Exception cause) {
        return attempt < maxAttempts;
    }

    @Override
    public long getDelayMs(int attempt) {
        // attempt is 1-based; shift is 0-based so use (attempt - 1)
        long exponentialDelay = baseDelayMs * (1L << (attempt - 1));
        return Math.min(exponentialDelay, maxDelayMs);
    }

    public int getMaxAttempts() { return maxAttempts; }
    public long getBaseDelayMs() { return baseDelayMs; }
    public long getMaxDelayMs() { return maxDelayMs; }
}
```

### Step Interfaces and Implementations

```java
// WorkflowStep.java
package com.example.workflow.step;

import com.example.workflow.StepResult;
import com.example.workflow.WorkflowContext;
import com.example.workflow.retry.RetryPolicy;

/**
 * Command interface: each step knows how to execute itself and how to compensate.
 */
public interface WorkflowStep {
    String getId();
    String getName();
    StepResult execute(WorkflowContext context) throws Exception;
    void compensate(WorkflowContext context);
    RetryPolicy getRetryPolicy();
}
```

```java
// AbstractWorkflowStep.java
package com.example.workflow.step;

import com.example.workflow.WorkflowContext;
import com.example.workflow.retry.NoRetryPolicy;
import com.example.workflow.retry.RetryPolicy;

/**
 * Base class providing id, name, and default no-op compensation.
 * Subclasses override execute() and optionally compensate().
 */
public abstract class AbstractWorkflowStep implements WorkflowStep {

    private final String id;
    private final String name;
    private final RetryPolicy retryPolicy;

    protected AbstractWorkflowStep(String id, String name, RetryPolicy retryPolicy) {
        if (id == null || id.isBlank()) throw new IllegalArgumentException("step id cannot be blank");
        if (name == null || name.isBlank()) throw new IllegalArgumentException("step name cannot be blank");
        this.id = id;
        this.name = name;
        this.retryPolicy = retryPolicy != null ? retryPolicy : new NoRetryPolicy();
    }

    protected AbstractWorkflowStep(String id, String name) {
        this(id, name, new NoRetryPolicy());
    }

    @Override
    public String getId() { return id; }

    @Override
    public String getName() { return name; }

    @Override
    public RetryPolicy getRetryPolicy() { return retryPolicy; }

    /**
     * Default compensation is a no-op.
     * Override in steps that need to roll back side effects.
     */
    @Override
    public void compensate(WorkflowContext context) {
        System.out.printf("[COMPENSATE] Default no-op compensation for step '%s'%n", name);
    }
}
```

```java
// TaskStep.java
package com.example.workflow.step;

import com.example.workflow.StepResult;
import com.example.workflow.WorkflowContext;
import com.example.workflow.retry.RetryPolicy;

import java.util.function.Function;

/**
 * Wraps a plain Java lambda or method reference as a workflow step.
 * The compensationAction is also a lambda so compensation logic stays
 * co-located with the step definition.
 *
 * Usage:
 *   TaskStep.<String>builder()
 *       .id("validate")
 *       .name("Validate Order")
 *       .action(ctx -> StepResult.success("validated"))
 *       .compensation(ctx -> ctx.addData("inventory:reserved", false))
 *       .build()
 */
public class TaskStep extends AbstractWorkflowStep {

    private final Function<WorkflowContext, StepResult> action;
    private final java.util.function.Consumer<WorkflowContext> compensationAction;

    private TaskStep(Builder builder) {
        super(builder.id, builder.name, builder.retryPolicy);
        if (builder.action == null) throw new IllegalStateException("action cannot be null");
        this.action = builder.action;
        this.compensationAction = builder.compensationAction;
    }

    @Override
    public StepResult execute(WorkflowContext context) throws Exception {
        return action.apply(context);
    }

    @Override
    public void compensate(WorkflowContext context) {
        if (compensationAction != null) {
            compensationAction.accept(context);
        } else {
            super.compensate(context);
        }
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String id;
        private String name;
        private RetryPolicy retryPolicy;
        private Function<WorkflowContext, StepResult> action;
        private java.util.function.Consumer<WorkflowContext> compensationAction;

        public Builder id(String id) { this.id = id; return this; }
        public Builder name(String name) { this.name = name; return this; }
        public Builder retryPolicy(RetryPolicy retryPolicy) { this.retryPolicy = retryPolicy; return this; }
        public Builder action(Function<WorkflowContext, StepResult> action) { this.action = action; return this; }
        public Builder compensation(java.util.function.Consumer<WorkflowContext> compensationAction) {
            this.compensationAction = compensationAction;
            return this;
        }
        public TaskStep build() { return new TaskStep(this); }
    }
}
```

```java
// DecisionStep.java
package com.example.workflow.step;

import com.example.workflow.StepResult;
import com.example.workflow.WorkflowContext;
import com.example.workflow.retry.RetryPolicy;

import java.util.function.Predicate;

/**
 * Conditional branching step.
 * Evaluates a Predicate<WorkflowContext>; publishes the trueStepId or falseStepId
 * as its output so the WorkflowEngine knows which branch to follow.
 *
 * The step itself does not navigate — it produces an output that the engine reads.
 */
public class DecisionStep extends AbstractWorkflowStep {

    private final Predicate<WorkflowContext> condition;
    private final String trueStepId;
    private final String falseStepId;

    public DecisionStep(String id, String name, Predicate<WorkflowContext> condition,
                        String trueStepId, String falseStepId) {
        super(id, name);
        if (condition == null) throw new IllegalArgumentException("condition cannot be null");
        if (trueStepId == null) throw new IllegalArgumentException("trueStepId cannot be null");
        if (falseStepId == null) throw new IllegalArgumentException("falseStepId cannot be null");
        this.condition = condition;
        this.trueStepId = trueStepId;
        this.falseStepId = falseStepId;
    }

    public DecisionStep(String id, String name, RetryPolicy retryPolicy,
                        Predicate<WorkflowContext> condition,
                        String trueStepId, String falseStepId) {
        super(id, name, retryPolicy);
        this.condition = condition;
        this.trueStepId = trueStepId;
        this.falseStepId = falseStepId;
    }

    @Override
    public StepResult execute(WorkflowContext context) {
        boolean result = condition.test(context);
        String nextStepId = result ? trueStepId : falseStepId;
        // Store decision outcome in context so the engine can read it
        context.addData("__decision__" + getId(), nextStepId);
        return StepResult.success(nextStepId);
    }

    public String getTrueStepId() { return trueStepId; }
    public String getFalseStepId() { return falseStepId; }
}
```

```java
// ParallelStep.java
package com.example.workflow.step;

import com.example.workflow.StepResult;
import com.example.workflow.WorkflowContext;
import com.example.workflow.retry.RetryPolicy;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

/**
 * Executes a list of steps concurrently using an ExecutorService.
 *
 * All parallel steps share the same WorkflowContext reference.
 * Callers must ensure sub-steps write to distinct context keys to avoid races.
 *
 * The parallel step succeeds only if ALL sub-steps succeed.
 * On any failure, it returns a failure result with the first error message.
 */
public class ParallelStep extends AbstractWorkflowStep {

    private final List<WorkflowStep> parallelSteps;
    private final ExecutorService executorService;

    public ParallelStep(String id, String name, List<WorkflowStep> parallelSteps) {
        super(id, name);
        if (parallelSteps == null || parallelSteps.isEmpty()) {
            throw new IllegalArgumentException("parallelSteps cannot be null or empty");
        }
        this.parallelSteps = new ArrayList<>(parallelSteps);
        this.executorService = Executors.newFixedThreadPool(
                Math.min(parallelSteps.size(), Runtime.getRuntime().availableProcessors()));
    }

    public ParallelStep(String id, String name, RetryPolicy retryPolicy,
                        List<WorkflowStep> parallelSteps, ExecutorService executorService) {
        super(id, name, retryPolicy);
        this.parallelSteps = new ArrayList<>(parallelSteps);
        this.executorService = executorService;
    }

    @Override
    public StepResult execute(WorkflowContext context) throws Exception {
        List<Callable<StepResult>> tasks = new ArrayList<>();
        for (WorkflowStep step : parallelSteps) {
            tasks.add(() -> step.execute(context));
        }

        List<Future<StepResult>> futures = executorService.invokeAll(tasks);

        List<String> errors = new ArrayList<>();
        for (int i = 0; i < futures.size(); i++) {
            Future<StepResult> future = futures.get(i);
            try {
                StepResult result = future.get();
                if (!result.isSuccess()) {
                    errors.add(parallelSteps.get(i).getName() + ": " + result.getErrorMessage());
                }
            } catch (Exception e) {
                errors.add(parallelSteps.get(i).getName() + ": " + e.getMessage());
            }
        }

        if (!errors.isEmpty()) {
            return StepResult.failure("Parallel steps failed: " + String.join("; ", errors));
        }
        return StepResult.success("All " + parallelSteps.size() + " parallel steps completed");
    }

    @Override
    public void compensate(WorkflowContext context) {
        // Compensate all parallel steps in reverse order
        for (int i = parallelSteps.size() - 1; i >= 0; i--) {
            try {
                parallelSteps.get(i).compensate(context);
            } catch (Exception e) {
                System.err.printf("[ParallelStep] Compensation failed for sub-step '%s': %s%n",
                        parallelSteps.get(i).getName(), e.getMessage());
            }
        }
    }

    public List<WorkflowStep> getParallelSteps() { return parallelSteps; }
}
```

```java
// LoopStep.java
package com.example.workflow.step;

import com.example.workflow.StepResult;
import com.example.workflow.WorkflowContext;
import com.example.workflow.retry.RetryPolicy;

import java.util.function.Predicate;

/**
 * Repeats an inner WorkflowStep up to maxIterations times, or while a
 * continueCondition evaluates to true, whichever limit is reached first.
 *
 * The iteration count is stored in context under "__loop_<id>_iterations"
 * so subsequent steps can read how many times the loop ran.
 */
public class LoopStep extends AbstractWorkflowStep {

    private final WorkflowStep innerStep;
    private final int maxIterations;
    private final Predicate<WorkflowContext> continueCondition;

    /**
     * Loop up to maxIterations times unconditionally.
     */
    public LoopStep(String id, String name, WorkflowStep innerStep, int maxIterations) {
        super(id, name);
        this.innerStep = innerStep;
        this.maxIterations = maxIterations;
        this.continueCondition = ctx -> true;
    }

    /**
     * Loop while condition is true, up to maxIterations as a safety cap.
     */
    public LoopStep(String id, String name, RetryPolicy retryPolicy,
                    WorkflowStep innerStep, int maxIterations,
                    Predicate<WorkflowContext> continueCondition) {
        super(id, name, retryPolicy);
        this.innerStep = innerStep;
        this.maxIterations = maxIterations;
        this.continueCondition = continueCondition != null ? continueCondition : ctx -> true;
    }

    @Override
    public StepResult execute(WorkflowContext context) throws Exception {
        int iterationsCompleted = 0;
        for (int i = 0; i < maxIterations; i++) {
            if (!continueCondition.test(context)) {
                break;
            }
            StepResult result = innerStep.execute(context);
            if (!result.isSuccess()) {
                return StepResult.failure(
                        "Loop failed at iteration " + (i + 1) + ": " + result.getErrorMessage());
            }
            iterationsCompleted++;
        }
        context.addData("__loop_" + getId() + "_iterations", iterationsCompleted);
        return StepResult.success("Completed " + iterationsCompleted + " iterations");
    }

    @Override
    public void compensate(WorkflowContext context) {
        innerStep.compensate(context);
    }

    public WorkflowStep getInnerStep() { return innerStep; }
    public int getMaxIterations() { return maxIterations; }
}
```

```java
// SubWorkflowStep.java
package com.example.workflow.step;

import com.example.workflow.StepResult;
import com.example.workflow.WorkflowContext;
import com.example.workflow.WorkflowDefinition;
import com.example.workflow.WorkflowInstance;
import com.example.workflow.WorkflowStatus;
import com.example.workflow.retry.RetryPolicy;

/**
 * Executes a nested WorkflowDefinition as a single step within the parent workflow.
 *
 * The sub-workflow shares the same WorkflowContext, so data flows naturally
 * between parent and child workflows.
 *
 * Uses a separate WorkflowEngine instance to avoid re-entrancy issues.
 */
public class SubWorkflowStep extends AbstractWorkflowStep {

    private final WorkflowDefinition subWorkflow;

    public SubWorkflowStep(String id, String name, WorkflowDefinition subWorkflow) {
        super(id, name);
        if (subWorkflow == null) throw new IllegalArgumentException("subWorkflow cannot be null");
        this.subWorkflow = subWorkflow;
    }

    public SubWorkflowStep(String id, String name, RetryPolicy retryPolicy,
                           WorkflowDefinition subWorkflow) {
        super(id, name, retryPolicy);
        this.subWorkflow = subWorkflow;
    }

    @Override
    public StepResult execute(WorkflowContext context) {
        // Create a dedicated engine for the sub-workflow to avoid shared state
        com.example.workflow.WorkflowEngine subEngine = new com.example.workflow.WorkflowEngine();
        WorkflowInstance subInstance = subEngine.start(subWorkflow, context.getAllData());

        if (subInstance.getStatus() == WorkflowStatus.COMPLETED) {
            // Merge sub-workflow outputs back into parent context
            subInstance.getContext().getAllData().forEach(context::addData);
            return StepResult.success("Sub-workflow '" + subWorkflow.getName() + "' completed");
        } else {
            return StepResult.failure("Sub-workflow '" + subWorkflow.getName()
                    + "' failed with status: " + subInstance.getStatus());
        }
    }

    public WorkflowDefinition getSubWorkflow() { return subWorkflow; }
}
```

### Workflow Definition and Instance

```java
// StepExecution.java
package com.example.workflow;

import java.time.Instant;

/**
 * Audit record for a single step execution attempt within a workflow instance.
 */
public class StepExecution {

    private final String stepId;
    private StepStatus status;
    private final Instant startTime;
    private Instant endTime;
    private int attemptNumber;
    private String errorMessage;

    public StepExecution(String stepId, int attemptNumber) {
        this.stepId = stepId;
        this.status = StepStatus.PENDING;
        this.startTime = Instant.now();
        this.attemptNumber = attemptNumber;
    }

    public void markRunning() {
        this.status = StepStatus.RUNNING;
    }

    public void markCompleted() {
        this.status = StepStatus.COMPLETED;
        this.endTime = Instant.now();
    }

    public void markFailed(String errorMessage) {
        this.status = StepStatus.FAILED;
        this.endTime = Instant.now();
        this.errorMessage = errorMessage;
    }

    public void markCompensated() {
        this.status = StepStatus.COMPENSATED;
        this.endTime = Instant.now();
    }

    public void markSkipped() {
        this.status = StepStatus.SKIPPED;
        this.endTime = Instant.now();
    }

    public String getStepId() { return stepId; }
    public StepStatus getStatus() { return status; }
    public Instant getStartTime() { return startTime; }
    public Instant getEndTime() { return endTime; }
    public int getAttemptNumber() { return attemptNumber; }
    public String getErrorMessage() { return errorMessage; }

    public long getDurationMs() {
        if (endTime == null) return -1;
        return endTime.toEpochMilli() - startTime.toEpochMilli();
    }

    @Override
    public String toString() {
        return "StepExecution{stepId=" + stepId + ", status=" + status
                + ", attempt=" + attemptNumber + ", durationMs=" + getDurationMs()
                + (errorMessage != null ? ", error=" + errorMessage : "") + "}";
    }
}
```

```java
// WorkflowDefinition.java
package com.example.workflow;

import com.example.workflow.step.WorkflowStep;

import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * Immutable description of a workflow: its steps and transition rules.
 *
 * Transitions are stored as:
 *   nextStepMap: stepId → nextStepId (happy path)
 *   failureStepMap: stepId → failureHandlerStepId (optional error path)
 *
 * The Builder enforces that every referenced step id exists in the step registry.
 */
public final class WorkflowDefinition {

    private final String id;
    private final String name;
    private final String version;
    private final List<WorkflowStep> steps;
    private final Map<String, WorkflowStep> stepById;
    private final String startStepId;
    private final Map<String, String> nextStepMap;
    private final Map<String, String> failureStepMap;

    private WorkflowDefinition(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.version = builder.version;
        this.steps = Collections.unmodifiableList(new ArrayList<>(builder.steps));
        this.stepById = Collections.unmodifiableMap(new HashMap<>(builder.stepById));
        this.startStepId = builder.startStepId;
        this.nextStepMap = Collections.unmodifiableMap(new HashMap<>(builder.nextStepMap));
        this.failureStepMap = Collections.unmodifiableMap(new HashMap<>(builder.failureStepMap));
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getVersion() { return version; }
    public List<WorkflowStep> getSteps() { return steps; }
    public String getStartStepId() { return startStepId; }

    public WorkflowStep getStep(String stepId) {
        return stepById.get(stepId);
    }

    public String getNextStepId(String stepId) {
        return nextStepMap.get(stepId);
    }

    public String getFailureStepId(String stepId) {
        return failureStepMap.get(stepId);
    }

    public boolean hasStep(String stepId) {
        return stepById.containsKey(stepId);
    }

    @Override
    public String toString() {
        return "WorkflowDefinition{id=" + id + ", name=" + name
                + ", version=" + version + ", steps=" + steps.size() + "}";
    }

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private String id;
        private String name;
        private String version = "1.0";
        private final List<WorkflowStep> steps = new ArrayList<>();
        private final Map<String, WorkflowStep> stepById = new LinkedHashMap<>();
        private String startStepId;
        private final Map<String, String> nextStepMap = new HashMap<>();
        private final Map<String, String> failureStepMap = new HashMap<>();
        private String lastAddedStepId;

        public Builder id(String id) { this.id = id; return this; }
        public Builder name(String name) { this.name = name; return this; }
        public Builder version(String version) { this.version = version; return this; }

        /**
         * Add a step to the workflow. The first step added is automatically set as startStepId.
         */
        public Builder step(WorkflowStep step) {
            if (step == null) throw new IllegalArgumentException("step cannot be null");
            if (stepById.containsKey(step.getId())) {
                throw new IllegalArgumentException("Duplicate step id: " + step.getId());
            }
            steps.add(step);
            stepById.put(step.getId(), step);
            if (startStepId == null) {
                startStepId = step.getId();
            }
            lastAddedStepId = step.getId();
            return this;
        }

        /**
         * Connect the last added step to the given next step id.
         */
        public Builder then(String nextStepId) {
            if (lastAddedStepId == null) throw new IllegalStateException("No step added yet");
            nextStepMap.put(lastAddedStepId, nextStepId);
            return this;
        }

        /**
         * Connect the last added step to an explicit next step id (same as then, explicit source).
         */
        public Builder then(String fromStepId, String toStepId) {
            nextStepMap.put(fromStepId, toStepId);
            return this;
        }

        /**
         * Define a failure handler step for the given step id.
         */
        public Builder onFailure(String stepId, String failureHandlerStepId) {
            failureStepMap.put(stepId, failureHandlerStepId);
            return this;
        }

        /**
         * Override the automatically detected start step.
         */
        public Builder startAt(String stepId) {
            this.startStepId = stepId;
            return this;
        }

        public WorkflowDefinition build() {
            if (id == null || id.isBlank()) throw new IllegalStateException("Workflow id is required");
            if (name == null || name.isBlank()) throw new IllegalStateException("Workflow name is required");
            if (steps.isEmpty()) throw new IllegalStateException("Workflow must have at least one step");
            if (startStepId == null) throw new IllegalStateException("Workflow startStepId is required");
            // Validate all nextStepMap and failureStepMap references
            for (Map.Entry<String, String> entry : nextStepMap.entrySet()) {
                if (!stepById.containsKey(entry.getKey())) {
                    throw new IllegalStateException("nextStepMap references unknown source step: " + entry.getKey());
                }
                if (!stepById.containsKey(entry.getValue())) {
                    throw new IllegalStateException("nextStepMap references unknown target step: " + entry.getValue());
                }
            }
            return new WorkflowDefinition(this);
        }
    }
}
```

```java
// WorkflowInstance.java
package com.example.workflow;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

/**
 * Runtime instance of a workflow definition.
 * Tracks status, mutable context, and full execution history.
 */
public class WorkflowInstance {

    private final String instanceId;
    private final String definitionId;
    private final String version;
    private WorkflowStatus status;
    private final WorkflowContext context;
    private final List<StepExecution> executionHistory;
    private final Instant createdAt;
    private Instant completedAt;

    public WorkflowInstance(String definitionId, String version, WorkflowContext context) {
        this.instanceId = UUID.randomUUID().toString();
        this.definitionId = definitionId;
        this.version = version;
        this.status = WorkflowStatus.CREATED;
        this.context = context;
        this.executionHistory = new ArrayList<>();
        this.createdAt = Instant.now();
    }

    public String getInstanceId() { return instanceId; }
    public String getDefinitionId() { return definitionId; }
    public String getVersion() { return version; }
    public WorkflowStatus getStatus() { return status; }
    public WorkflowContext getContext() { return context; }
    public List<StepExecution> getExecutionHistory() { return Collections.unmodifiableList(executionHistory); }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getCompletedAt() { return completedAt; }

    public void setStatus(WorkflowStatus status) {
        this.status = status;
        if (status == WorkflowStatus.COMPLETED || status == WorkflowStatus.FAILED
                || status == WorkflowStatus.COMPENSATED) {
            this.completedAt = Instant.now();
        }
    }

    public void addStepExecution(StepExecution execution) {
        executionHistory.add(execution);
    }

    public StepExecution getLatestExecution(String stepId) {
        for (int i = executionHistory.size() - 1; i >= 0; i--) {
            if (executionHistory.get(i).getStepId().equals(stepId)) {
                return executionHistory.get(i);
            }
        }
        return null;
    }

    @Override
    public String toString() {
        return "WorkflowInstance{instanceId=" + instanceId
                + ", definitionId=" + definitionId
                + ", version=" + version
                + ", status=" + status
                + ", steps=" + executionHistory.size() + "}";
    }
}
```

### Compensation Manager

```java
// CompensationManager.java
package com.example.workflow;

import com.example.workflow.step.WorkflowStep;

import java.util.ArrayDeque;
import java.util.Deque;

/**
 * Saga pattern: tracks completed steps on a stack and compensates them
 * in LIFO order (reverse completion order) on failure.
 *
 * This implements the "backward recovery" variant of the Saga pattern:
 * compensating transactions undo the effects of already-completed steps.
 */
public class CompensationManager {

    // Stack of (stepId, step) pairs in completion order
    private final Deque<CompensationEntry> completedSteps = new ArrayDeque<>();

    /**
     * Register a successfully completed step for potential compensation.
     */
    public void push(String stepId, WorkflowStep step) {
        completedSteps.push(new CompensationEntry(stepId, step));
    }

    /**
     * Execute compensation for all registered steps in reverse order.
     * Compensation failures are logged but do not stop the rollback chain.
     */
    public void executeCompensation(WorkflowContext context, WorkflowEventBus eventBus,
                                     String instanceId) {
        System.out.printf("[CompensationManager] Starting compensation for %d steps (LIFO order)%n",
                completedSteps.size());

        while (!completedSteps.isEmpty()) {
            CompensationEntry entry = completedSteps.pop();
            System.out.printf("[CompensationManager] Compensating step '%s'%n", entry.stepId);
            try {
                entry.step.compensate(context);

                eventBus.publish(WorkflowEvent.builder()
                        .workflowInstanceId(instanceId)
                        .stepId(entry.stepId)
                        .eventType(WorkflowEventType.STEP_COMPENSATED)
                        .contextSnapshot(context.snapshot())
                        .build());

                System.out.printf("[CompensationManager] Step '%s' compensated successfully%n",
                        entry.stepId);
            } catch (Exception e) {
                System.err.printf("[CompensationManager] Compensation failed for step '%s': %s%n",
                        entry.stepId, e.getMessage());
                // Continue compensating remaining steps
            }
        }
    }

    public int pendingCompensationCount() {
        return completedSteps.size();
    }

    public void clear() {
        completedSteps.clear();
    }

    private static final class CompensationEntry {
        final String stepId;
        final WorkflowStep step;

        CompensationEntry(String stepId, WorkflowStep step) {
            this.stepId = stepId;
            this.step = step;
        }
    }
}
```

### Step Executor

```java
// StepExecutor.java
package com.example.workflow;

import com.example.workflow.retry.RetryPolicy;
import com.example.workflow.step.WorkflowStep;

/**
 * Executes a single step with retry logic.
 *
 * Chain of Responsibility role: applies retry behavior, then delegates
 * the actual execution to the step.
 *
 * The executor is stateless and reusable across workflow instances.
 */
public class StepExecutor {

    /**
     * Execute the given step against the context, applying the step's retry policy.
     *
     * @return StepResult from the step (success or final failure after retries)
     */
    public StepResult execute(WorkflowStep step, WorkflowContext context) {
        RetryPolicy retryPolicy = step.getRetryPolicy();
        int attempt = 1;
        Exception lastException = null;

        while (true) {
            try {
                System.out.printf("[StepExecutor] Executing step '%s' (attempt %d)%n",
                        step.getName(), attempt);
                StepResult result = step.execute(context);
                if (result.isSuccess()) {
                    return result;
                }
                // Step returned a failure result (not an exception)
                if (!retryPolicy.shouldRetry(attempt, new RuntimeException(result.getErrorMessage()))) {
                    return result;
                }
                System.out.printf("[StepExecutor] Step '%s' returned failure, will retry (attempt %d)%n",
                        step.getName(), attempt);
            } catch (Exception e) {
                lastException = e;
                System.out.printf("[StepExecutor] Step '%s' threw exception: %s%n",
                        step.getName(), e.getMessage());
                if (!retryPolicy.shouldRetry(attempt, e)) {
                    return StepResult.failure(
                            "Step '" + step.getName() + "' failed after " + attempt + " attempt(s): "
                                    + e.getMessage(), e);
                }
            }

            long delayMs = retryPolicy.getDelayMs(attempt);
            if (delayMs > 0) {
                System.out.printf("[StepExecutor] Waiting %dms before retry%n", delayMs);
                try {
                    Thread.sleep(delayMs);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    return StepResult.failure("Step '" + step.getName() + "' interrupted during retry wait");
                }
            }
            attempt++;
        }
    }
}
```

### Workflow Engine

```java
// WorkflowEngine.java
package com.example.workflow;

import com.example.workflow.step.DecisionStep;
import com.example.workflow.step.WorkflowStep;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

/**
 * The main orchestrator.
 *
 * Execution model:
 *   1. Create a WorkflowInstance
 *   2. Look up the current step from WorkflowDefinition
 *   3. Delegate execution to StepExecutor (handles retry)
 *   4. Publish events to WorkflowEventBus
 *   5. If success: push step to CompensationManager, advance to next step
 *   6. If failure: trigger compensation via CompensationManager
 *   7. Notify WorkflowLifecycleListeners at each phase
 *
 * The engine is synchronous by default. Async execution can be layered
 * on top using a task queue and the resume(WorkflowInstance) method.
 */
public class WorkflowEngine {

    private final WorkflowEventBus eventBus;
    private final StepExecutor stepExecutor;
    private final List<WorkflowLifecycleListener> lifecycleListeners;

    public WorkflowEngine() {
        this.eventBus = new WorkflowEventBus();
        this.stepExecutor = new StepExecutor();
        this.lifecycleListeners = new ArrayList<>();
    }

    public WorkflowEngine(WorkflowEventBus eventBus) {
        this.eventBus = eventBus;
        this.stepExecutor = new StepExecutor();
        this.lifecycleListeners = new ArrayList<>();
    }

    public void addLifecycleListener(WorkflowLifecycleListener listener) {
        lifecycleListeners.add(listener);
        // Also subscribe to raw event bus
        eventBus.subscribe(event -> {
            // lifecycle listener gets higher-level callbacks; raw events handled below
        });
    }

    public WorkflowEventBus getEventBus() { return eventBus; }

    /**
     * Start a new workflow instance from its definition and initial data.
     *
     * @param definition   the workflow to execute
     * @param initialData  seed data for the WorkflowContext
     * @return the completed (or failed) WorkflowInstance
     */
    public WorkflowInstance start(WorkflowDefinition definition, Map<String, Object> initialData) {
        WorkflowContext context = new WorkflowContext(initialData);
        WorkflowInstance instance = new WorkflowInstance(
                definition.getId(), definition.getVersion(), context);

        instance.setStatus(WorkflowStatus.RUNNING);

        notifyWorkflowStarted(instance.getInstanceId(), definition.getName(), definition.getVersion());

        publishEvent(WorkflowEvent.builder()
                .workflowInstanceId(instance.getInstanceId())
                .stepId(null)
                .eventType(WorkflowEventType.WORKFLOW_STARTED)
                .contextSnapshot(context.snapshot())
                .build());

        executeWorkflow(definition, instance);
        return instance;
    }

    /**
     * Resume a previously paused workflow instance.
     * The instance must have status RUNNING and a non-empty execution history
     * indicating the last completed step.
     */
    public WorkflowInstance resume(WorkflowDefinition definition, WorkflowInstance instance) {
        if (instance.getStatus() != WorkflowStatus.RUNNING) {
            throw new IllegalStateException("Can only resume a RUNNING workflow instance, got: "
                    + instance.getStatus());
        }
        executeWorkflow(definition, instance);
        return instance;
    }

    // ------------------------------------------------------------------
    // Internal execution loop
    // ------------------------------------------------------------------

    private void executeWorkflow(WorkflowDefinition definition, WorkflowInstance instance) {
        CompensationManager compensationManager = new CompensationManager();
        String currentStepId = determineStartStep(definition, instance);

        while (currentStepId != null) {
            WorkflowStep step = definition.getStep(currentStepId);
            if (step == null) {
                failWorkflow(instance, compensationManager, currentStepId,
                        new IllegalStateException("Step not found in definition: " + currentStepId));
                return;
            }

            // Publish STEP_STARTED
            StepExecution execution = new StepExecution(currentStepId, 1);
            execution.markRunning();
            instance.addStepExecution(execution);

            publishEvent(WorkflowEvent.builder()
                    .workflowInstanceId(instance.getInstanceId())
                    .stepId(currentStepId)
                    .eventType(WorkflowEventType.STEP_STARTED)
                    .contextSnapshot(instance.getContext().snapshot())
                    .build());

            // Execute with retry
            StepResult result = stepExecutor.execute(step, instance.getContext());

            if (result.isSuccess()) {
                execution.markCompleted();
                compensationManager.push(currentStepId, step);

                publishEvent(WorkflowEvent.builder()
                        .workflowInstanceId(instance.getInstanceId())
                        .stepId(currentStepId)
                        .eventType(WorkflowEventType.STEP_COMPLETED)
                        .contextSnapshot(instance.getContext().snapshot())
                        .build());

                notifyStepCompleted(instance.getInstanceId(), currentStepId, result);

                // Advance to next step
                currentStepId = resolveNextStep(definition, step, currentStepId, instance.getContext());

            } else {
                execution.markFailed(result.getErrorMessage());

                publishEvent(WorkflowEvent.builder()
                        .workflowInstanceId(instance.getInstanceId())
                        .stepId(currentStepId)
                        .eventType(WorkflowEventType.STEP_FAILED)
                        .contextSnapshot(instance.getContext().snapshot())
                        .errorMessage(result.getErrorMessage())
                        .build());

                Throwable cause = result.getCause() != null
                        ? result.getCause()
                        : new RuntimeException(result.getErrorMessage());
                notifyStepFailed(instance.getInstanceId(), currentStepId, cause);

                failWorkflow(instance, compensationManager, currentStepId, cause);
                return;
            }
        }

        // All steps completed successfully
        instance.setStatus(WorkflowStatus.COMPLETED);

        publishEvent(WorkflowEvent.builder()
                .workflowInstanceId(instance.getInstanceId())
                .stepId(null)
                .eventType(WorkflowEventType.WORKFLOW_COMPLETED)
                .contextSnapshot(instance.getContext().snapshot())
                .build());

        notifyWorkflowCompleted(instance.getInstanceId(), instance.getContext());
    }

    private String determineStartStep(WorkflowDefinition definition, WorkflowInstance instance) {
        List<StepExecution> history = instance.getExecutionHistory();
        if (history.isEmpty()) {
            return definition.getStartStepId();
        }
        // For resume: find the last completed step and return its successor
        for (int i = history.size() - 1; i >= 0; i--) {
            StepExecution exec = history.get(i);
            if (exec.getStatus() == StepStatus.COMPLETED) {
                return definition.getNextStepId(exec.getStepId());
            }
        }
        return definition.getStartStepId();
    }

    /**
     * Determines the next step id after a successful step execution.
     * Handles DecisionStep branching by reading the decision output from context.
     */
    private String resolveNextStep(WorkflowDefinition definition, WorkflowStep step,
                                    String currentStepId, WorkflowContext context) {
        if (step instanceof DecisionStep) {
            // DecisionStep stores its chosen branch in context
            DecisionStep decisionStep = (DecisionStep) step;
            return context.<String>getData("__decision__" + currentStepId)
                    .orElse(definition.getNextStepId(currentStepId));
        }
        return definition.getNextStepId(currentStepId);
    }

    private void failWorkflow(WorkflowInstance instance, CompensationManager compensationManager,
                               String failedStepId, Throwable cause) {
        instance.setStatus(WorkflowStatus.FAILED);

        publishEvent(WorkflowEvent.builder()
                .workflowInstanceId(instance.getInstanceId())
                .stepId(failedStepId)
                .eventType(WorkflowEventType.WORKFLOW_FAILED)
                .contextSnapshot(instance.getContext().snapshot())
                .errorMessage(cause.getMessage())
                .build());

        notifyWorkflowFailed(instance.getInstanceId(), failedStepId, cause);

        // Begin Saga compensation
        if (compensationManager.pendingCompensationCount() > 0) {
            instance.setStatus(WorkflowStatus.COMPENSATING);

            publishEvent(WorkflowEvent.builder()
                    .workflowInstanceId(instance.getInstanceId())
                    .stepId(null)
                    .eventType(WorkflowEventType.COMPENSATION_STARTED)
                    .contextSnapshot(instance.getContext().snapshot())
                    .build());

            notifyCompensationStarted(instance.getInstanceId());
            compensationManager.executeCompensation(instance.getContext(), eventBus,
                    instance.getInstanceId());
            instance.setStatus(WorkflowStatus.COMPENSATED);
            notifyCompensationCompleted(instance.getInstanceId());
        }
    }

    // ------------------------------------------------------------------
    // Lifecycle notification helpers
    // ------------------------------------------------------------------

    private void publishEvent(WorkflowEvent event) {
        eventBus.publish(event);
    }

    private void notifyWorkflowStarted(String instanceId, String name, String version) {
        for (WorkflowLifecycleListener l : lifecycleListeners) {
            try { l.onWorkflowStarted(instanceId, name, version); }
            catch (Exception e) { System.err.println("[Engine] Listener error: " + e.getMessage()); }
        }
    }

    private void notifyWorkflowCompleted(String instanceId, WorkflowContext ctx) {
        for (WorkflowLifecycleListener l : lifecycleListeners) {
            try { l.onWorkflowCompleted(instanceId, ctx); }
            catch (Exception e) { System.err.println("[Engine] Listener error: " + e.getMessage()); }
        }
    }

    private void notifyWorkflowFailed(String instanceId, String stepId, Throwable cause) {
        for (WorkflowLifecycleListener l : lifecycleListeners) {
            try { l.onWorkflowFailed(instanceId, stepId, cause); }
            catch (Exception e) { System.err.println("[Engine] Listener error: " + e.getMessage()); }
        }
    }

    private void notifyStepCompleted(String instanceId, String stepId, StepResult result) {
        for (WorkflowLifecycleListener l : lifecycleListeners) {
            try { l.onStepCompleted(instanceId, stepId, result); }
            catch (Exception e) { System.err.println("[Engine] Listener error: " + e.getMessage()); }
        }
    }

    private void notifyStepFailed(String instanceId, String stepId, Throwable cause) {
        for (WorkflowLifecycleListener l : lifecycleListeners) {
            try { l.onStepFailed(instanceId, stepId, cause); }
            catch (Exception e) { System.err.println("[Engine] Listener error: " + e.getMessage()); }
        }
    }

    private void notifyCompensationStarted(String instanceId) {
        for (WorkflowLifecycleListener l : lifecycleListeners) {
            try { l.onCompensationStarted(instanceId); }
            catch (Exception e) { System.err.println("[Engine] Listener error: " + e.getMessage()); }
        }
    }

    private void notifyCompensationCompleted(String instanceId) {
        for (WorkflowLifecycleListener l : lifecycleListeners) {
            try { l.onCompensationCompleted(instanceId); }
            catch (Exception e) { System.err.println("[Engine] Listener error: " + e.getMessage()); }
        }
    }
}
```

### Demo: 5-Step Order Processing Workflow

```java
// WorkflowEngineDemo.java
package com.example.workflow.demo;

import com.example.workflow.LoggingLifecycleListener;
import com.example.workflow.StepResult;
import com.example.workflow.WorkflowDefinition;
import com.example.workflow.WorkflowEngine;
import com.example.workflow.WorkflowInstance;
import com.example.workflow.WorkflowStatus;
import com.example.workflow.retry.ExponentialBackoffRetryPolicy;
import com.example.workflow.step.TaskStep;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * Demonstrates a 5-step Order Processing workflow:
 *
 *   ValidateOrder → ReserveInventory → ChargePayment → SendConfirmation → UpdateAnalytics
 *
 * Scenario 1: Happy path — all steps succeed
 * Scenario 2: PaymentFailure — ChargePayment fails, compensates ReserveInventory
 */
public class WorkflowEngineDemo {

    public static void main(String[] args) {
        System.out.println("╔══════════════════════════════════════════════════╗");
        System.out.println("║   Event-Driven Workflow Engine — Demo            ║");
        System.out.println("╚══════════════════════════════════════════════════╝\n");

        runHappyPathScenario();
        System.out.println("\n" + "=".repeat(60) + "\n");
        runPaymentFailureScenario();
    }

    // ------------------------------------------------------------------
    // Scenario 1: Happy Path
    // ------------------------------------------------------------------
    private static void runHappyPathScenario() {
        System.out.println("SCENARIO 1: Happy Path — all steps succeed\n");

        WorkflowDefinition definition = buildOrderWorkflow(false);
        WorkflowEngine engine = new WorkflowEngine();
        engine.addLifecycleListener(new LoggingLifecycleListener());

        Map<String, Object> initialData = new HashMap<>();
        initialData.put("orderId", "ORD-1001");
        initialData.put("customerId", "CUST-42");
        initialData.put("itemId", "ITEM-SKU-9988");
        initialData.put("quantity", 2);
        initialData.put("totalAmount", 149.99);

        WorkflowInstance instance = engine.start(definition, initialData);

        System.out.println("\nFinal instance status: " + instance.getStatus());
        System.out.println("Execution history:");
        instance.getExecutionHistory().forEach(e ->
                System.out.println("  " + e.getStepId() + " → " + e.getStatus()
                        + " (" + e.getDurationMs() + "ms)"));
    }

    // ------------------------------------------------------------------
    // Scenario 2: Payment Failure with Saga Compensation
    // ------------------------------------------------------------------
    private static void runPaymentFailureScenario() {
        System.out.println("SCENARIO 2: Payment Failure — compensation of ReserveInventory\n");

        WorkflowDefinition definition = buildOrderWorkflow(true); // true = simulate payment failure
        WorkflowEngine engine = new WorkflowEngine();
        engine.addLifecycleListener(new LoggingLifecycleListener());

        Map<String, Object> initialData = new HashMap<>();
        initialData.put("orderId", "ORD-1002");
        initialData.put("customerId", "CUST-99");
        initialData.put("itemId", "ITEM-SKU-0042");
        initialData.put("quantity", 1);
        initialData.put("totalAmount", 299.00);
        initialData.put("paymentMethod", "CARD-DECLINED");

        WorkflowInstance instance = engine.start(definition, initialData);

        System.out.println("\nFinal instance status: " + instance.getStatus());
        System.out.println("Execution history:");
        instance.getExecutionHistory().forEach(e ->
                System.out.println("  " + e.getStepId() + " → " + e.getStatus()
                        + (e.getErrorMessage() != null ? " [" + e.getErrorMessage() + "]" : "")
                        + " (" + e.getDurationMs() + "ms)"));

        // Verify compensation ran
        boolean inventoryCompensated = instance.getContext()
                .<Boolean>getData("inventory:released")
                .orElse(false);
        System.out.println("\nInventory compensation ran: " + inventoryCompensated);
        System.out.println("Expected workflow status COMPENSATED: "
                + (instance.getStatus() == WorkflowStatus.COMPENSATED));
    }

    // ------------------------------------------------------------------
    // Workflow Definition Builder
    // ------------------------------------------------------------------
    private static WorkflowDefinition buildOrderWorkflow(boolean simulatePaymentFailure) {

        // Step 1: Validate Order
        TaskStep validateOrder = TaskStep.builder()
                .id("validateOrder")
                .name("Validate Order")
                .action(ctx -> {
                    String orderId = ctx.<String>getData("orderId").orElse("unknown");
                    Integer quantity = ctx.<Integer>getData("quantity").orElse(0);
                    System.out.println("  [Step] Validating order " + orderId
                            + " for quantity " + quantity);
                    if (quantity <= 0) {
                        return StepResult.failure("Order quantity must be positive");
                    }
                    ctx.addData("order:validated", true);
                    ctx.addData("order:validatedAt", System.currentTimeMillis());
                    return StepResult.success("Order validated");
                })
                .compensation(ctx -> {
                    System.out.println("  [Compensate] Marking order as cancelled");
                    ctx.addData("order:cancelled", true);
                })
                .build();

        // Step 2: Reserve Inventory
        AtomicBoolean inventoryReserved = new AtomicBoolean(false);
        TaskStep reserveInventory = TaskStep.builder()
                .id("reserveInventory")
                .name("Reserve Inventory")
                .retryPolicy(new ExponentialBackoffRetryPolicy(3, 50, 500))
                .action(ctx -> {
                    String itemId = ctx.<String>getData("itemId").orElse("unknown");
                    Integer quantity = ctx.<Integer>getData("quantity").orElse(0);
                    System.out.println("  [Step] Reserving " + quantity + "x " + itemId);
                    ctx.addData("inventory:reserved", true);
                    ctx.addData("inventory:reservationId", "RES-" + System.currentTimeMillis());
                    inventoryReserved.set(true);
                    return StepResult.success("Inventory reserved");
                })
                .compensation(ctx -> {
                    String reservationId = ctx.<String>getData("inventory:reservationId")
                            .orElse("unknown");
                    System.out.println("  [Compensate] Releasing inventory reservation: "
                            + reservationId);
                    ctx.addData("inventory:reserved", false);
                    ctx.addData("inventory:released", true);
                    inventoryReserved.set(false);
                })
                .build();

        // Step 3: Charge Payment (may fail in scenario 2)
        TaskStep chargePayment = TaskStep.builder()
                .id("chargePayment")
                .name("Charge Payment")
                .action(ctx -> {
                    String paymentMethod = ctx.<String>getData("paymentMethod").orElse("CARD-OK");
                    Double amount = ctx.<Double>getData("totalAmount").orElse(0.0);
                    System.out.println("  [Step] Charging $" + amount + " via " + paymentMethod);
                    if (simulatePaymentFailure || "CARD-DECLINED".equals(paymentMethod)) {
                        return StepResult.failure("Payment declined: insufficient funds");
                    }
                    ctx.addData("payment:charged", true);
                    ctx.addData("payment:transactionId", "TXN-" + System.currentTimeMillis());
                    return StepResult.success("Payment charged");
                })
                .compensation(ctx -> {
                    String txnId = ctx.<String>getData("payment:transactionId").orElse("none");
                    System.out.println("  [Compensate] Refunding transaction: " + txnId);
                    ctx.addData("payment:refunded", true);
                })
                .build();

        // Step 4: Send Confirmation Email
        TaskStep sendConfirmation = TaskStep.builder()
                .id("sendConfirmation")
                .name("Send Confirmation Email")
                .action(ctx -> {
                    String customerId = ctx.<String>getData("customerId").orElse("unknown");
                    String orderId = ctx.<String>getData("orderId").orElse("unknown");
                    System.out.println("  [Step] Sending confirmation to customer "
                            + customerId + " for order " + orderId);
                    ctx.addData("email:sent", true);
                    ctx.addData("email:sentAt", System.currentTimeMillis());
                    return StepResult.success("Confirmation sent");
                })
                .compensation(ctx -> {
                    System.out.println("  [Compensate] Sending cancellation email");
                    ctx.addData("email:cancellationSent", true);
                })
                .build();

        // Step 5: Update Analytics (fire and forget — no compensation needed)
        TaskStep updateAnalytics = TaskStep.builder()
                .id("updateAnalytics")
                .name("Update Analytics")
                .action(ctx -> {
                    String orderId = ctx.<String>getData("orderId").orElse("unknown");
                    System.out.println("  [Step] Recording analytics event for order " + orderId);
                    ctx.addData("analytics:recorded", true);
                    return StepResult.success("Analytics updated");
                })
                .build();

        return WorkflowDefinition.builder()
                .id("order-processing-workflow")
                .name("Order Processing Workflow")
                .version("2.1")
                .step(validateOrder).then("reserveInventory")
                .step(reserveInventory).then("chargePayment")
                .step(chargePayment).then("sendConfirmation")
                .step(sendConfirmation).then("updateAnalytics")
                .step(updateAnalytics)
                .build();
    }
}
```

---

## Unit Tests Design

### WorkflowContextTest

```java
// WorkflowContextTest.java
package com.example.workflow;

import org.junit.jupiter.api.Test;
import java.util.HashMap;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class WorkflowContextTest {

    @Test
    void givenAddData_whenGetData_thenReturnsValue() {
        WorkflowContext context = new WorkflowContext();
        context.addData("key", "value");
        assertEquals("value", context.<String>getData("key").orElse(null));
    }

    @Test
    void givenSnapshot_whenRestoreToEmpty_thenDataIsRestoredExactly() {
        WorkflowContext context = new WorkflowContext();
        context.addData("a", 1);
        context.addData("b", "hello");
        ContextSnapshot snapshot = context.snapshot();

        WorkflowContext restored = new WorkflowContext();
        restored.restore(snapshot);

        assertEquals(1, restored.<Integer>getData("a").orElse(0));
        assertEquals("hello", restored.<String>getData("b").orElse(null));
    }

    @Test
    void givenSnapshotThenMutate_whenRestoreSnapshot_thenMutationsReverted() {
        WorkflowContext context = new WorkflowContext();
        context.addData("counter", 0);
        ContextSnapshot snapshot = context.snapshot();

        context.addData("counter", 99);
        context.addData("extra", "should not persist");
        context.restore(snapshot);

        assertEquals(0, context.<Integer>getData("counter").orElse(-1));
        assertFalse(context.hasKey("extra"));
    }

    @Test
    void givenNullKey_whenAddData_thenThrowsIllegalArgumentException() {
        WorkflowContext context = new WorkflowContext();
        assertThrows(IllegalArgumentException.class, () -> context.addData(null, "value"));
    }
}
```

### TaskStepTest

```java
// TaskStepTest.java
package com.example.workflow.step;

import com.example.workflow.StepResult;
import com.example.workflow.WorkflowContext;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class TaskStepTest {

    @Test
    void givenSuccessfulAction_whenExecute_thenReturnsSuccess() throws Exception {
        TaskStep step = TaskStep.builder()
                .id("s1").name("Step 1")
                .action(ctx -> StepResult.success("done"))
                .build();
        WorkflowContext context = new WorkflowContext();
        StepResult result = step.execute(context);
        assertTrue(result.isSuccess());
        assertEquals("done", result.getOutput());
    }

    @Test
    void givenFailingAction_whenExecute_thenReturnsFailure() throws Exception {
        TaskStep step = TaskStep.builder()
                .id("s2").name("Step 2")
                .action(ctx -> StepResult.failure("something went wrong"))
                .build();
        WorkflowContext context = new WorkflowContext();
        StepResult result = step.execute(context);
        assertFalse(result.isSuccess());
        assertEquals("something went wrong", result.getErrorMessage());
    }

    @Test
    void givenCompensationAction_whenCompensate_thenContextMutated() throws Exception {
        TaskStep step = TaskStep.builder()
                .id("s3").name("Step 3")
                .action(ctx -> { ctx.addData("reserved", true); return StepResult.success(); })
                .compensation(ctx -> ctx.addData("reserved", false))
                .build();
        WorkflowContext context = new WorkflowContext();
        step.execute(context);
        assertTrue((Boolean) context.<Boolean>getData("reserved").orElse(false));
        step.compensate(context);
        assertFalse((Boolean) context.<Boolean>getData("reserved").orElse(true));
    }
}
```

### DecisionStepTest

```java
// DecisionStepTest.java
package com.example.workflow.step;

import com.example.workflow.StepResult;
import com.example.workflow.WorkflowContext;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class DecisionStepTest {

    @Test
    void givenTrueCondition_whenExecute_thenOutputIsTrueStepId() throws Exception {
        DecisionStep step = new DecisionStep("d1", "Check Premium",
                ctx -> ctx.<Boolean>getData("isPremium").orElse(false),
                "premiumStep", "standardStep");
        WorkflowContext context = new WorkflowContext();
        context.addData("isPremium", true);
        StepResult result = step.execute(context);
        assertTrue(result.isSuccess());
        assertEquals("premiumStep", result.getOutput());
    }

    @Test
    void givenFalseCondition_whenExecute_thenOutputIsFalseStepId() throws Exception {
        DecisionStep step = new DecisionStep("d2", "Check Premium",
                ctx -> ctx.<Boolean>getData("isPremium").orElse(false),
                "premiumStep", "standardStep");
        WorkflowContext context = new WorkflowContext();
        context.addData("isPremium", false);
        StepResult result = step.execute(context);
        assertTrue(result.isSuccess());
        assertEquals("standardStep", result.getOutput());
    }

    @Test
    void givenExecute_whenDecisionMade_thenContextStoresDecisionKey() throws Exception {
        DecisionStep step = new DecisionStep("d3", "Route Order",
                ctx -> true, "express", "standard");
        WorkflowContext context = new WorkflowContext();
        step.execute(context);
        assertEquals("express", context.<String>getData("__decision__d3").orElse(null));
    }
}
```

### CompensationManagerTest

```java
// CompensationManagerTest.java
package com.example.workflow;

import com.example.workflow.step.TaskStep;
import com.example.workflow.step.WorkflowStep;
import org.junit.jupiter.api.Test;
import java.util.concurrent.atomic.AtomicInteger;
import static org.junit.jupiter.api.Assertions.*;

class CompensationManagerTest {

    @Test
    void givenMultipleSteps_whenCompensate_thenLIFOOrder() {
        CompensationManager manager = new CompensationManager();
        StringBuilder order = new StringBuilder();

        WorkflowStep s1 = TaskStep.builder().id("s1").name("S1")
                .action(ctx -> StepResult.success())
                .compensation(ctx -> order.append("C1 "))
                .build();
        WorkflowStep s2 = TaskStep.builder().id("s2").name("S2")
                .action(ctx -> StepResult.success())
                .compensation(ctx -> order.append("C2 "))
                .build();
        WorkflowStep s3 = TaskStep.builder().id("s3").name("S3")
                .action(ctx -> StepResult.success())
                .compensation(ctx -> order.append("C3 "))
                .build();

        manager.push("s1", s1);
        manager.push("s2", s2);
        manager.push("s3", s3);

        WorkflowContext context = new WorkflowContext();
        WorkflowEventBus bus = new WorkflowEventBus();
        manager.executeCompensation(context, bus, "test-instance");

        assertEquals("C3 C2 C1 ", order.toString());
    }

    @Test
    void givenNoSteps_whenCompensate_thenNothingHappens() {
        CompensationManager manager = new CompensationManager();
        WorkflowContext context = new WorkflowContext();
        WorkflowEventBus bus = new WorkflowEventBus();
        // Should not throw
        assertDoesNotThrow(() -> manager.executeCompensation(context, bus, "test"));
    }

    @Test
    void givenPushedSteps_whenPendingCount_thenReturnsCorrectCount() {
        CompensationManager manager = new CompensationManager();
        WorkflowStep step = TaskStep.builder().id("s").name("S")
                .action(ctx -> StepResult.success()).build();
        manager.push("s", step);
        manager.push("s2", step);
        assertEquals(2, manager.pendingCompensationCount());
    }

    @Test
    void givenCompensationThrows_whenExecute_thenContinuesWithRemainingSteps() {
        CompensationManager manager = new CompensationManager();
        AtomicInteger compensatedCount = new AtomicInteger(0);

        WorkflowStep badStep = TaskStep.builder().id("bad").name("Bad")
                .action(ctx -> StepResult.success())
                .compensation(ctx -> { throw new RuntimeException("Compensation exploded"); })
                .build();
        WorkflowStep goodStep = TaskStep.builder().id("good").name("Good")
                .action(ctx -> StepResult.success())
                .compensation(ctx -> compensatedCount.incrementAndGet())
                .build();

        manager.push("good", goodStep); // pushed first
        manager.push("bad", badStep);   // pushed second, compensated first (LIFO)

        WorkflowContext context = new WorkflowContext();
        WorkflowEventBus bus = new WorkflowEventBus();
        // Should not throw despite badStep compensation failing
        assertDoesNotThrow(() -> manager.executeCompensation(context, bus, "test"));
        // goodStep compensation should still have run
        assertEquals(1, compensatedCount.get());
    }
}
```

### WorkflowEngineTest

```java
// WorkflowEngineTest.java
package com.example.workflow;

import com.example.workflow.step.DecisionStep;
import com.example.workflow.step.TaskStep;
import org.junit.jupiter.api.Test;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.atomic.AtomicBoolean;
import static org.junit.jupiter.api.Assertions.*;

class WorkflowEngineTest {

    private WorkflowDefinition singleStepWorkflow(boolean succeed) {
        TaskStep step = TaskStep.builder()
                .id("only")
                .name("Only Step")
                .action(ctx -> succeed
                        ? StepResult.success("ok")
                        : StepResult.failure("forced failure"))
                .build();
        return WorkflowDefinition.builder()
                .id("single").name("Single Step WF").step(step).build();
    }

    @Test
    void givenSuccessfulWorkflow_whenStart_thenStatusIsCompleted() {
        WorkflowEngine engine = new WorkflowEngine();
        WorkflowInstance instance = engine.start(singleStepWorkflow(true), new HashMap<>());
        assertEquals(WorkflowStatus.COMPLETED, instance.getStatus());
        assertEquals(1, instance.getExecutionHistory().size());
        assertEquals(StepStatus.COMPLETED, instance.getExecutionHistory().get(0).getStatus());
    }

    @Test
    void givenFailingStep_whenStart_thenStatusIsCompensated() {
        WorkflowEngine engine = new WorkflowEngine();
        WorkflowInstance instance = engine.start(singleStepWorkflow(false), new HashMap<>());
        // No steps completed before failure, so no compensation needed
        assertEquals(WorkflowStatus.FAILED, instance.getStatus());
    }

    @Test
    void givenMultiStepWorkflow_whenStart_thenAllStepsInHistory() {
        TaskStep s1 = TaskStep.builder().id("s1").name("S1")
                .action(ctx -> StepResult.success()).build();
        TaskStep s2 = TaskStep.builder().id("s2").name("S2")
                .action(ctx -> StepResult.success()).build();
        TaskStep s3 = TaskStep.builder().id("s3").name("S3")
                .action(ctx -> StepResult.success()).build();

        WorkflowDefinition def = WorkflowDefinition.builder()
                .id("multi").name("Multi Step WF")
                .step(s1).then("s2")
                .step(s2).then("s3")
                .step(s3)
                .build();

        WorkflowEngine engine = new WorkflowEngine();
        WorkflowInstance instance = engine.start(def, new HashMap<>());

        assertEquals(WorkflowStatus.COMPLETED, instance.getStatus());
        assertEquals(3, instance.getExecutionHistory().size());
    }

    @Test
    void givenFailureInMiddle_whenCompensationConfigured_thenCompletedStepsCompensated() {
        AtomicBoolean step1Compensated = new AtomicBoolean(false);

        TaskStep s1 = TaskStep.builder().id("s1").name("S1")
                .action(ctx -> { ctx.addData("s1done", true); return StepResult.success(); })
                .compensation(ctx -> step1Compensated.set(true))
                .build();
        TaskStep s2 = TaskStep.builder().id("s2").name("S2")
                .action(ctx -> StepResult.failure("s2 always fails"))
                .build();

        WorkflowDefinition def = WorkflowDefinition.builder()
                .id("comp-test").name("Compensation Test WF")
                .step(s1).then("s2")
                .step(s2)
                .build();

        WorkflowEngine engine = new WorkflowEngine();
        WorkflowInstance instance = engine.start(def, new HashMap<>());

        assertEquals(WorkflowStatus.COMPENSATED, instance.getStatus());
        assertTrue(step1Compensated.get(), "s1 should have been compensated");
    }

    @Test
    void givenDecisionStep_whenConditionTrue_thenFollowsTrueBranch() {
        DecisionStep decision = new DecisionStep("decide", "Route",
                ctx -> ctx.<Boolean>getData("isPremium").orElse(false),
                "premiumStep", "standardStep");

        TaskStep premiumStep = TaskStep.builder().id("premiumStep").name("Premium")
                .action(ctx -> { ctx.addData("branch", "premium"); return StepResult.success(); })
                .build();
        TaskStep standardStep = TaskStep.builder().id("standardStep").name("Standard")
                .action(ctx -> { ctx.addData("branch", "standard"); return StepResult.success(); })
                .build();

        WorkflowDefinition def = WorkflowDefinition.builder()
                .id("branch-test").name("Branch WF")
                .step(decision).then("decide", "premiumStep")
                .step(premiumStep)
                .step(standardStep)
                .build();

        WorkflowEngine engine = new WorkflowEngine();
        Map<String, Object> input = new HashMap<>();
        input.put("isPremium", true);
        WorkflowInstance instance = engine.start(def, input);

        assertEquals(WorkflowStatus.COMPLETED, instance.getStatus());
        assertEquals("premium",
                instance.getContext().<String>getData("branch").orElse("none"));
    }
}
```

### ExponentialBackoffRetryPolicyTest

```java
// ExponentialBackoffRetryPolicyTest.java
package com.example.workflow.retry;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ExponentialBackoffRetryPolicyTest {

    @Test
    void givenAttemptBelowMax_whenShouldRetry_thenReturnsTrue() {
        ExponentialBackoffRetryPolicy policy =
                new ExponentialBackoffRetryPolicy(3, 100, 10000);
        assertTrue(policy.shouldRetry(1, new RuntimeException("err")));
        assertTrue(policy.shouldRetry(2, new RuntimeException("err")));
    }

    @Test
    void givenAttemptAtMax_whenShouldRetry_thenReturnsFalse() {
        ExponentialBackoffRetryPolicy policy =
                new ExponentialBackoffRetryPolicy(3, 100, 10000);
        // maxAttempts=3, so attempt=3 is the last → no retry
        assertFalse(policy.shouldRetry(3, new RuntimeException("err")));
    }

    @Test
    void givenBaseDelayAndAttempt_whenGetDelayMs_thenExponentialDelayRespectsCap() {
        ExponentialBackoffRetryPolicy policy =
                new ExponentialBackoffRetryPolicy(10, 100, 1000);
        assertEquals(100L, policy.getDelayMs(1));   // 100 * 2^0 = 100
        assertEquals(200L, policy.getDelayMs(2));   // 100 * 2^1 = 200
        assertEquals(400L, policy.getDelayMs(3));   // 100 * 2^2 = 400
        assertEquals(800L, policy.getDelayMs(4));   // 100 * 2^3 = 800
        assertEquals(1000L, policy.getDelayMs(5));  // 100 * 2^4 = 1600 → capped at 1000
        assertEquals(1000L, policy.getDelayMs(6));  // capped
    }
}
```

---

## Extension Exercise: TimerStep

The following shows how to add a `TimerStep` that waits N seconds then continues, without any changes to `WorkflowEngine`.

**Principle:** The engine only knows about `WorkflowStep`. Adding a new step type is a pure extension (Open/Closed Principle).

```java
// TimerStep.java
package com.example.workflow.step;

import com.example.workflow.StepResult;
import com.example.workflow.WorkflowContext;
import com.example.workflow.retry.NoRetryPolicy;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.CountDownLatch;

/**
 * A step that pauses execution for a specified duration then continues.
 *
 * Implementation notes:
 *   1. Extends AbstractWorkflowStep — no changes to WorkflowEngine needed.
 *   2. Uses ScheduledExecutorService to schedule the continuation.
 *   3. A CountDownLatch makes execute() block until the timer fires,
 *      so the engine sees synchronous step completion.
 *   4. Publishes StepCompletedEvent implicitly via a successful StepResult.
 *
 * For true async long-running timers (e.g., "wait 24 hours"), use a persistent
 * timer store (database + scheduler job) rather than blocking a thread.
 * The compensation for a TimerStep is a no-op because time cannot be reversed.
 */
public class TimerStep extends AbstractWorkflowStep {

    private final long waitSeconds;
    private final ScheduledExecutorService scheduler;

    /**
     * @param id          step id
     * @param name        human-readable name
     * @param waitSeconds how many seconds to wait before completing
     */
    public TimerStep(String id, String name, long waitSeconds) {
        super(id, name, new NoRetryPolicy());
        if (waitSeconds < 0) throw new IllegalArgumentException("waitSeconds must be >= 0");
        this.waitSeconds = waitSeconds;
        this.scheduler = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "timer-step-" + id);
            t.setDaemon(true);
            return t;
        });
    }

    @Override
    public StepResult execute(WorkflowContext context) throws Exception {
        if (waitSeconds == 0) {
            return StepResult.success("Timer completed immediately (0s wait)");
        }

        System.out.printf("[TimerStep] '%s' waiting for %d second(s)...%n", getName(), waitSeconds);

        CountDownLatch latch = new CountDownLatch(1);

        scheduler.schedule(() -> {
            System.out.printf("[TimerStep] '%s' timer fired after %d second(s)%n",
                    getName(), waitSeconds);
            latch.countDown();
        }, waitSeconds, TimeUnit.SECONDS);

        boolean completed = latch.await(waitSeconds + 5, TimeUnit.SECONDS);

        if (!completed) {
            return StepResult.failure("Timer step '" + getName() + "' timed out waiting for scheduler");
        }

        context.addData("__timer__" + getId() + "_firedAt", System.currentTimeMillis());
        return StepResult.success("Timer step '" + getName() + "' completed after " + waitSeconds + "s");
    }

    @Override
    public void compensate(WorkflowContext context) {
        // Time cannot be reversed; compensation is a no-op
        System.out.printf("[TimerStep] No compensation available for timer '%s'%n", getName());
    }

    public void shutdown() {
        scheduler.shutdown();
    }

    public long getWaitSeconds() { return waitSeconds; }
}
```

**How to use TimerStep in a workflow with zero changes to WorkflowEngine:**

```java
// TimerStepUsageExample.java — compile and run standalone
package com.example.workflow.demo;

import com.example.workflow.StepResult;
import com.example.workflow.WorkflowDefinition;
import com.example.workflow.WorkflowEngine;
import com.example.workflow.WorkflowInstance;
import com.example.workflow.step.TaskStep;
import com.example.workflow.step.TimerStep;

import java.util.HashMap;

public class TimerStepUsageExample {

    public static void main(String[] args) {
        TaskStep step1 = TaskStep.builder()
                .id("start").name("Start Processing")
                .action(ctx -> { ctx.addData("startTime", System.currentTimeMillis());
                                 return StepResult.success(); })
                .build();

        TimerStep timer = new TimerStep("cooldown", "Cooldown Timer", 2); // 2-second wait

        TaskStep step3 = TaskStep.builder()
                .id("finish").name("Finish Processing")
                .action(ctx -> {
                    long start = ctx.<Long>getData("startTime").orElse(0L);
                    long elapsed = System.currentTimeMillis() - start;
                    System.out.println("Elapsed time: " + elapsed + "ms (should be ~2000ms)");
                    return StepResult.success();
                })
                .build();

        WorkflowDefinition def = WorkflowDefinition.builder()
                .id("timer-demo").name("Timer Demo Workflow")
                .step(step1).then("cooldown")
                .step(timer).then("finish")
                .step(step3)
                .build();

        WorkflowEngine engine = new WorkflowEngine();  // Zero changes to engine
        WorkflowInstance instance = engine.start(def, new HashMap<>());

        System.out.println("Final status: " + instance.getStatus());
        timer.shutdown();
    }
}
```

**Key design points of this extension:**
1. `TimerStep` extends `AbstractWorkflowStep` — the only contract the engine requires.
2. `execute()` blocks synchronously using `CountDownLatch`. The engine sees a normal `StepResult.success()` return, which triggers the `STEP_COMPLETED` event exactly as any other step.
3. `compensate()` is a no-op — time cannot be undone.
4. The `ScheduledExecutorService` is owned by the step, not shared. This allows independent timer management.
5. Zero lines of `WorkflowEngine` were changed.

---

## Performance and Architecture Considerations

### Event Bus: Sync vs Async Dispatch

The current `WorkflowEventBus` dispatches synchronously (listeners are called inline). This is correct for:
- Steps that are fast and non-blocking
- Tests (deterministic execution order)
- Single-threaded orchestration

For production long-running workflows, replace synchronous dispatch with an `ExecutorService`-backed async bus:

```java
// Async publish (replace the for-loop in WorkflowEventBus.publish):
for (WorkflowEventListener listener : listeners) {
    asyncExecutor.submit(() -> {
        try { listener.onEvent(event); }
        catch (Exception e) { /* log and continue */ }
    });
}
```

This decouples step execution latency from listener processing time.

### Compensation and Idempotency

Each compensating action must be **idempotent**: if compensation is called twice (e.g., due to a system restart during rollback), it must not double-refund or double-release. Add an idempotency check using a compensation id stored in context:

```java
.compensation(ctx -> {
    if (ctx.<Boolean>getData("inventory:released").orElse(false)) {
        return; // already compensated
    }
    releaseInventory(ctx.getDataOrThrow("inventory:reservationId"));
    ctx.addData("inventory:released", true);
})
```

### Workflow Persistence

For truly long-running workflows (hours/days), `WorkflowInstance` must be serialized to a durable store between steps. The `ContextSnapshot` (Memento) already provides this: serialize `snapshot.getData()` to JSON via Jackson, store in a database, and restore via `context.restore(snapshot)` when the next step is triggered by a scheduler or message consumer.

### Thread Safety of WorkflowContext in ParallelStep

`ParallelStep` runs sub-steps concurrently on a shared `WorkflowContext`. Since `HashMap` is not thread-safe, production use should either:
1. Give each parallel step its own isolated `WorkflowContext` and merge results after all complete.
2. Replace `HashMap` in `WorkflowContext` with `ConcurrentHashMap`.

Option 1 (isolated sub-contexts) is preferred because it eliminates races without requiring synchronization on every context access.

---

## Interview Questions

1. **Why does `DecisionStep` store its output in the context rather than returning the next step id directly from `execute()`?**
   The `WorkflowStep` interface's `execute()` returns a `StepResult` — a value object. If it returned a step id, the interface would be coupled to the engine's routing concern. Storing the decision in context respects the Command pattern: the step produces output, the engine reads it and routes accordingly.

2. **The Saga pattern compensates in LIFO order. Why is LIFO correct for distributed transactions?**
   Each compensating transaction undoes the most recent side effect first. If step B depended on step A's output, reversing A before B would leave the system in an inconsistent state (e.g., releasing inventory before cancelling the reservation that referenced it). LIFO ensures each compensation targets a state the system actually reached.

3. **How would you support parallel compensation (compensate all failed parallel branches concurrently)?**
   Replace the sequential `while` loop in `CompensationManager.executeCompensation` with an `ExecutorService.invokeAll` call, submitting each `step.compensate(context)` as a `Callable`. Collect failures and report them together. Note that shared context writes during parallel compensation reintroduce the thread-safety concern from the ParallelStep discussion.

4. **Why is `WorkflowDefinition` immutable but `WorkflowInstance` mutable?**
   A `WorkflowDefinition` is a template used to create many instances. Mutating it would corrupt all in-flight instances. A `WorkflowInstance` is the runtime state — its status, context, and history must evolve as the workflow progresses. This follows the Prototype/Template distinction: immutable blueprint, mutable execution record.

5. **How would you add workflow persistence without changing `WorkflowEngine`?**
   Implement `WorkflowEventListener` and register it with `WorkflowEventBus`. On every `STEP_COMPLETED` event, serialize `event.getContextSnapshot()` and the current `StepExecution` to a database. On `WORKFLOW_STARTED`, create a row. On `WORKFLOW_COMPLETED` or `WORKFLOW_COMPENSATED`, mark the row terminal. The engine knows nothing about persistence — it only publishes events.
