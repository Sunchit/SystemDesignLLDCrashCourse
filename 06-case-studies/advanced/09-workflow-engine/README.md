# Workflow Engine — Advanced LLD Case Study

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

Modern enterprise software relies heavily on multi-step business processes: employee onboarding, order fulfillment, document approval, CI/CD pipelines, data-processing ETL jobs, and more. These processes share a common need: **orchestrated, stateful execution of a graph of tasks** with branching, parallelism, error recovery, and auditing.

Building these workflows ad-hoc—with spaghetti if/else chains or hardcoded state machines—leads to brittle, untestable code. The goal is a **reusable, extensible Workflow Engine** that:

- Expresses complex business processes declaratively as a tree of composable steps.
- Executes those steps with proper lifecycle management (start, pause, resume, cancel).
- Handles parallel independent work, conditional branches, loops, manual approvals, and automated tasks uniformly.
- Records every state transition in an immutable audit trail.
- Supports retries and configurable error-handling policies without polluting business logic.
- Allows versioned workflow definitions so running instances are not broken by definition changes.

---

## 2. Functional Requirements

| # | Requirement |
|---|-------------|
| FR-01 | Define a **workflow** as an ordered/structured collection of steps. |
| FR-02 | Support step types: **Automated**, **ManualApproval**, **Conditional**, **Parallel**, **Loop**. |
| FR-03 | Support workflow lifecycle: **start**, **pause**, **resume**, **cancel**. |
| FR-04 | Implement step state machine: `PENDING → RUNNING → COMPLETED / FAILED / SKIPPED`. |
| FR-05 | Support **conditional branching** — evaluate a predicate on `WorkflowContext` and choose a branch. |
| FR-06 | Execute independent steps **in parallel** using `CompletableFuture` / `ExecutorService`. |
| FR-07 | **Error handling**: per-step retry policy, skip-on-failure, and fail-workflow policies. |
| FR-08 | **Workflow versioning** — a definition carries a version; running instances track which version they use. |
| FR-09 | Immutable **audit trail** — every step transition is logged with timestamp, actor, and result. |
| FR-10 | **Observer / event** system — subscribers receive lifecycle events (step started, step failed, workflow completed, etc.). |
| FR-11 | **Context propagation** — steps read/write a shared typed key-value store (the `WorkflowContext`). |
| FR-12 | **Builder API** — fluent DSL for constructing workflow definitions. |

---

## 3. Non-Functional Requirements

| # | Requirement | Target |
|---|-------------|--------|
| NFR-01 | Thread safety for concurrent workflow instances | All shared state guarded |
| NFR-02 | Parallel step execution latency overhead | < 5 ms over sequential baseline |
| NFR-03 | Audit log append throughput | > 10 000 events/sec |
| NFR-04 | Workflow definition immutability after build | Enforced at compile time |
| NFR-05 | Step execution isolation | Failures in one parallel branch must not corrupt sibling branches |
| NFR-06 | Pluggable persistence | Engine core has no I/O dependency |
| NFR-07 | Java version | Java 17+ (records, sealed classes, switch expressions) |

---

## 4. Design Goals and Constraints

### Goals
- **Separation of concerns**: definition vs. execution vs. observation vs. persistence are distinct layers.
- **Composability**: every step type implements the same `Step` interface, enabling recursive nesting (Composite pattern).
- **Testability**: inject an `ExecutorService`, `Clock`, and observer — the engine has no static state.
- **Fail-safety**: an unchecked exception in a step handler must never corrupt the engine's internal state machine.
- **Openness for extension, closed for modification**: new step types or error-handling strategies are addable without touching existing code.

### Constraints
- No external framework dependencies (no Spring, no Quartz) — pure Java 17.
- The engine is **in-process**; distribution/clustering is a deployment concern.
- `WorkflowDefinition` is **immutable** once built; runtime state lives in `WorkflowInstance`.
- Parallel branches share the same `WorkflowContext` — callers must use thread-safe value types or coordinate via the context's `ConcurrentHashMap`.

---

## 5. Architecture / Class Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         com.workflow.engine                              │
│                                                                         │
│  ┌──────────────────────┐       ┌──────────────────────────────────┐    │
│  │  WorkflowDefinition  │──────▶│           Step (interface)        │    │
│  │  - id: String        │  1..* │  + getId(): String                │    │
│  │  - version: int      │       │  + getName(): String              │    │
│  │  - rootStep: Step    │       │  + execute(ctx, instance): void   │    │
│  └──────────────────────┘       │  + getErrorPolicy(): ErrorPolicy  │    │
│           │                     └──────────────────────────────────┘    │
│           │ builds                         △                             │
│           │                   ┌────────────┼────────────┐               │
│  ┌────────▼───────────┐       │            │            │               │
│  │  WorkflowBuilder   │  ┌────┴──────┐ ┌──┴──────┐ ┌───┴──────────┐   │
│  │  (fluent DSL)      │  │Automated  │ │Manual   │ │Sequential    │   │
│  └────────────────────┘  │Step       │ │Approval │ │Step          │   │
│                           └───────────┘ │Step     │ │(children[])  │   │
│  ┌──────────────────────┐               └─────────┘ └──────────────┘   │
│  │  WorkflowEngine      │                                               │
│  │  - execute()         │           ┌───────────────┐                   │
│  │  - pause()           │           │ ParallelStep  │                   │
│  │  - resume()          │           │ (children[])  │                   │
│  │  - cancel()          │           └───────────────┘                   │
│  └──────────┬───────────┘                                               │
│             │ creates/manages                                            │
│  ┌──────────▼───────────┐       ┌──────────────────────────────────┐    │
│  │  WorkflowInstance    │──────▶│        WorkflowContext           │    │
│  │  - instanceId        │       │  - variables: ConcurrentHashMap  │    │
│  │  - definitionId      │       │  - get/put/require               │    │
│  │  - definitionVersion │       └──────────────────────────────────┘    │
│  │  - status            │                                               │
│  │  - auditLog          │       ┌──────────────────────────────────┐    │
│  └──────────────────────┘       │         AuditLog                 │    │
│                                 │  - entries: List<AuditEntry>     │    │
│  ┌──────────────────────┐       │  - append(entry)                 │    │
│  │  WorkflowEventBus    │       └──────────────────────────────────┘    │
│  │  - subscribe()       │                                               │
│  │  - publish()         │       ┌──────────────────────────────────┐    │
│  └──────────────────────┘       │      ErrorHandlerChain           │    │
│                                 │  - handlers: List<ErrorHandler>  │    │
│  ┌──────────────────────┐       │  - handle(ctx, step, ex)         │    │
│  │  StepState (sealed)  │       └──────────────────────────────────┘    │
│  │  PENDING/RUNNING/    │                                               │
│  │  COMPLETED/FAILED/   │                                               │
│  │  SKIPPED/PAUSED      │                                               │
│  └──────────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────────┘

Composite Hierarchy:
  Step
  ├── AbstractStep (base state machine)
  │   ├── AutomatedStep      (Command pattern: StepCommand)
  │   ├── ManualApprovalStep (blocks until signal or timeout)
  │   ├── ConditionalStep    (evaluates Predicate<WorkflowContext>)
  │   ├── LoopStep           (repeats child while condition holds)
  │   ├── SequentialStep     (Composite: runs children in order)
  │   └── ParallelStep       (Composite: runs children via CompletableFuture)

Error Handler Chain (Chain of Responsibility):
  RetryErrorHandler → SkipErrorHandler → FailWorkflowErrorHandler

Observer:
  WorkflowEventBus  ←  WorkflowEvent (sealed record hierarchy)
                    →  WorkflowEventListener (functional interface)
```

---

## 6. Complete Java Implementation

### 6.1 Package Structure

```
com.workflow.engine
├── core
│   ├── Step.java
│   ├── AbstractStep.java
│   ├── StepState.java
│   ├── StepResult.java
│   ├── StepCommand.java
│   ├── WorkflowDefinition.java
│   ├── WorkflowInstance.java
│   ├── WorkflowContext.java
│   ├── WorkflowStatus.java
│   └── WorkflowEngine.java
├── steps
│   ├── AutomatedStep.java
│   ├── ManualApprovalStep.java
│   ├── ConditionalStep.java
│   ├── SequentialStep.java
│   ├── ParallelStep.java
│   └── LoopStep.java
├── error
│   ├── ErrorPolicy.java
│   ├── ErrorHandler.java
│   ├── ErrorHandlerChain.java
│   ├── RetryErrorHandler.java
│   ├── SkipErrorHandler.java
│   └── FailWorkflowErrorHandler.java
├── audit
│   ├── AuditEntry.java
│   └── AuditLog.java
├── event
│   ├── WorkflowEvent.java
│   ├── WorkflowEventListener.java
│   └── WorkflowEventBus.java
└── builder
    └── WorkflowBuilder.java
```

---

### 6.2 Core — StepState

```java
package com.workflow.engine.core;

/**
 * Sealed interface representing all legal states a step can occupy.
 * Using a sealed interface + records gives us exhaustive pattern matching
 * in switch expressions (Java 17+).
 */
public sealed interface StepState
        permits StepState.Pending, StepState.Running, StepState.Completed,
                StepState.Failed, StepState.Skipped, StepState.Paused {

    /** A terminal state is one from which no further transition is possible. */
    default boolean isTerminal() {
        return switch (this) {
            case Completed c -> true;
            case Failed    f -> true;
            case Skipped   s -> true;
            default          -> false;
        };
    }

    default boolean isRunning() {
        return this instanceof Running;
    }

    // ── Concrete states ────────────────────────────────────────────────

    record Pending()   implements StepState {}
    record Running()   implements StepState {}
    record Completed() implements StepState {}
    record Failed(String reason, Throwable cause) implements StepState {}
    record Skipped(String reason) implements StepState {}
    record Paused()    implements StepState {}

    // ── Factory helpers ────────────────────────────────────────────────

    static StepState pending()   { return new Pending(); }
    static StepState running()   { return new Running(); }
    static StepState completed() { return new Completed(); }
    static StepState paused()    { return new Paused(); }

    static StepState failed(String reason, Throwable cause) {
        return new Failed(reason, cause);
    }

    static StepState failed(Throwable cause) {
        return new Failed(cause.getMessage(), cause);
    }

    static StepState skipped(String reason) {
        return new Skipped(reason);
    }
}
```

---

### 6.3 Core — StepResult

```java
package com.workflow.engine.core;

import java.time.Instant;
import java.util.Optional;

/**
 * Immutable value returned by every step execution.
 * Carries the final state, optional output, and wall-clock timing.
 */
public record StepResult(
        String stepId,
        StepState state,
        Instant startedAt,
        Instant finishedAt,
        Optional<Object> output,
        Optional<Throwable> error
) {

    public static StepResult success(String stepId, Instant start, Instant end, Object output) {
        return new StepResult(stepId, StepState.completed(), start, end,
                Optional.ofNullable(output), Optional.empty());
    }

    public static StepResult success(String stepId, Instant start, Instant end) {
        return success(stepId, start, end, null);
    }

    public static StepResult failure(String stepId, Instant start, Instant end, Throwable ex) {
        return new StepResult(stepId, StepState.failed(ex), start, end,
                Optional.empty(), Optional.of(ex));
    }

    public static StepResult skipped(String stepId, Instant start, String reason) {
        return new StepResult(stepId, StepState.skipped(reason), start, start,
                Optional.empty(), Optional.empty());
    }

    public boolean isSuccess() { return state instanceof StepState.Completed; }
    public boolean isFailure() { return state instanceof StepState.Failed; }
    public boolean isSkipped() { return state instanceof StepState.Skipped; }

    public long durationMillis() {
        return finishedAt.toEpochMilli() - startedAt.toEpochMilli();
    }
}
```

---

### 6.4 Core — WorkflowContext

```java
package com.workflow.engine.core;

import java.util.Collections;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Shared, thread-safe key-value store passed through an entire workflow execution.
 *
 * Design notes:
 * - Values are stored as Object; callers use typed get() helpers to avoid casting.
 * - ConcurrentHashMap provides safe concurrent access for parallel steps.
 * - Snapshot returns an unmodifiable view for audit/debugging purposes.
 */
public final class WorkflowContext {

    private final String instanceId;
    private final ConcurrentHashMap<String, Object> variables;

    public WorkflowContext(String instanceId) {
        this.instanceId = instanceId;
        this.variables  = new ConcurrentHashMap<>();
    }

    /** Deep-copy constructor used when forking a context for a branch. */
    private WorkflowContext(String instanceId, Map<String, Object> initial) {
        this.instanceId = instanceId;
        this.variables  = new ConcurrentHashMap<>(initial);
    }

    // ── Write ──────────────────────────────────────────────────────────

    public void put(String key, Object value) {
        if (key == null) throw new IllegalArgumentException("Context key must not be null");
        if (value == null) {
            variables.remove(key);
        } else {
            variables.put(key, value);
        }
    }

    // ── Read ───────────────────────────────────────────────────────────

    @SuppressWarnings("unchecked")
    public <T> Optional<T> get(String key, Class<T> type) {
        Object val = variables.get(key);
        if (val == null) return Optional.empty();
        if (!type.isInstance(val)) {
            throw new ClassCastException(
                    "Context key '" + key + "' holds " + val.getClass().getName()
                    + " but requested " + type.getName());
        }
        return Optional.of(type.cast(val));
    }

    @SuppressWarnings("unchecked")
    public <T> T require(String key, Class<T> type) {
        return get(key, type).orElseThrow(() ->
                new IllegalStateException("Required context key '" + key + "' is missing"));
    }

    public boolean contains(String key) {
        return variables.containsKey(key);
    }

    // ── Utility ────────────────────────────────────────────────────────

    /** Returns an unmodifiable snapshot — safe to pass to audit/logging. */
    public Map<String, Object> snapshot() {
        return Collections.unmodifiableMap(variables);
    }

    /** Fork creates a copy — useful for conditional branches that should not pollute the main context. */
    public WorkflowContext fork(String branchId) {
        return new WorkflowContext(instanceId + "/" + branchId, variables);
    }

    public String getInstanceId() { return instanceId; }

    @Override
    public String toString() {
        return "WorkflowContext{instanceId='" + instanceId + "', variables=" + variables + "}";
    }
}
```

---

### 6.5 Core — WorkflowStatus

```java
package com.workflow.engine.core;

/**
 * Top-level lifecycle status of a WorkflowInstance.
 */
public enum WorkflowStatus {
    /** Created but not yet started. */
    CREATED,
    /** Currently executing. */
    RUNNING,
    /** Execution suspended; can be resumed. */
    PAUSED,
    /** All steps completed successfully. */
    COMPLETED,
    /** At least one step failed and no recovery policy handled it. */
    FAILED,
    /** Externally cancelled before completion. */
    CANCELLED
}
```

---

### 6.6 Core — ErrorPolicy

```java
package com.workflow.engine.error;

/**
 * Declarative error policy attached to each step.
 *
 * @param maxRetries    number of automatic retry attempts (0 = no retry)
 * @param retryDelayMs  delay between retries in milliseconds
 * @param skipOnFailure if true and all retries are exhausted, mark step SKIPPED
 *                      instead of failing the workflow
 * @param failWorkflow  if true (and skipOnFailure=false), the whole workflow
 *                      transitions to FAILED on step failure
 */
public record ErrorPolicy(
        int maxRetries,
        long retryDelayMs,
        boolean skipOnFailure,
        boolean failWorkflow
) {

    /** Strict: no retries, fail the whole workflow immediately. */
    public static final ErrorPolicy FAIL_FAST =
            new ErrorPolicy(0, 0, false, true);

    /** Lenient: no retries, just skip this step and continue. */
    public static final ErrorPolicy SKIP_ON_FAILURE =
            new ErrorPolicy(0, 0, true, false);

    /** Retry up to 3 times with 500 ms delay, then fail the workflow. */
    public static final ErrorPolicy RETRY_3_THEN_FAIL =
            new ErrorPolicy(3, 500, false, true);

    /** Retry up to 3 times, then skip (never fails the workflow). */
    public static final ErrorPolicy RETRY_3_THEN_SKIP =
            new ErrorPolicy(3, 500, true, false);

    public ErrorPolicy {
        if (maxRetries < 0) throw new IllegalArgumentException("maxRetries must be >= 0");
        if (retryDelayMs < 0) throw new IllegalArgumentException("retryDelayMs must be >= 0");
    }
}
```

---

### 6.7 Core — Step interface

```java
package com.workflow.engine.core;

import com.workflow.engine.error.ErrorPolicy;

/**
 * Root abstraction for all workflow steps.
 *
 * Every concrete step — leaf or composite — implements this interface.
 * This is the "Component" role in the Composite pattern and the "Command"
 * role in the Command pattern.
 *
 * Thread-safety contract: implementations must be stateless with respect to
 * execution; all mutable runtime state is stored in WorkflowInstance.
 */
public interface Step {

    /** Unique identifier within the workflow definition. */
    String getId();

    /** Human-readable name for display and audit. */
    String getName();

    /**
     * Execute this step.
     *
     * @param context  shared workflow context (thread-safe)
     * @param instance the running workflow instance (for pause/cancel checks)
     * @return a StepResult capturing the outcome
     */
    StepResult execute(WorkflowContext context, WorkflowInstance instance);

    /**
     * Policy controlling retry / skip / fail behavior on execution error.
     * Default: fail the workflow immediately.
     */
    default ErrorPolicy getErrorPolicy() {
        return ErrorPolicy.FAIL_FAST;
    }

    /**
     * Optional human-readable description.
     */
    default String getDescription() { return ""; }
}
```

---

### 6.8 Core — AbstractStep

```java
package com.workflow.engine.core;

import com.workflow.engine.audit.AuditEntry;
import com.workflow.engine.error.ErrorPolicy;
import com.workflow.engine.event.WorkflowEvent;

import java.time.Instant;
import java.util.Objects;
import java.util.concurrent.atomic.AtomicReference;

/**
 * Base class wiring together:
 * - State machine transitions (PENDING → RUNNING → terminal)
 * - Audit logging before/after execution
 * - Event publishing (step started / completed / failed / skipped)
 * - Pause/cancel interlock — checks the instance status before proceeding
 *
 * Concrete step types override doExecute() and are never concerned with
 * the machinery above.
 */
public abstract class AbstractStep implements Step {

    protected final String id;
    protected final String name;
    protected final String description;
    protected final ErrorPolicy errorPolicy;

    // Per-execution state — reset each time execute() is called.
    private final AtomicReference<StepState> state =
            new AtomicReference<>(StepState.pending());

    protected AbstractStep(String id, String name, String description, ErrorPolicy errorPolicy) {
        this.id          = Objects.requireNonNull(id,          "Step id must not be null");
        this.name        = Objects.requireNonNull(name,        "Step name must not be null");
        this.description = description != null ? description : "";
        this.errorPolicy = Objects.requireNonNull(errorPolicy, "ErrorPolicy must not be null");
    }

    @Override public String getId()          { return id; }
    @Override public String getName()        { return name; }
    @Override public String getDescription() { return description; }
    @Override public ErrorPolicy getErrorPolicy() { return errorPolicy; }

    // ── Template method ────────────────────────────────────────────────

    @Override
    public final StepResult execute(WorkflowContext context, WorkflowInstance instance) {
        // 1. Honor pause/cancel before starting
        if (instance.isCancelled()) {
            return skipped(context, instance, "Workflow cancelled");
        }
        if (instance.isPaused()) {
            // Block until resumed or cancelled
            instance.awaitResumeOrCancel();
            if (instance.isCancelled()) {
                return skipped(context, instance, "Workflow cancelled during pause");
            }
        }

        // 2. Transition to RUNNING
        transitionTo(StepState.running());
        Instant startedAt = Instant.now();
        instance.getAuditLog().append(
                AuditEntry.stepStarted(id, name, startedAt, context.snapshot()));
        instance.getEventBus().publish(
                WorkflowEvent.stepStarted(instance.getInstanceId(), id, name, startedAt));

        // 3. Delegate to subclass
        StepResult result;
        try {
            result = doExecute(context, instance);
        } catch (Exception ex) {
            result = StepResult.failure(id, startedAt, Instant.now(), ex);
        }

        // 4. Record terminal state
        Instant finishedAt = Instant.now();
        transitionTo(result.state());
        instance.getAuditLog().append(
                AuditEntry.stepFinished(id, name, finishedAt, result));
        if (result.isSuccess()) {
            instance.getEventBus().publish(
                    WorkflowEvent.stepCompleted(instance.getInstanceId(), id, name, finishedAt));
        } else if (result.isFailure()) {
            instance.getEventBus().publish(
                    WorkflowEvent.stepFailed(instance.getInstanceId(), id, name, finishedAt,
                            result.error().orElse(null)));
        } else if (result.isSkipped()) {
            instance.getEventBus().publish(
                    WorkflowEvent.stepSkipped(instance.getInstanceId(), id, name, finishedAt));
        }

        return result;
    }

    /**
     * Subclasses implement actual step logic here.
     * They may throw any exception — AbstractStep wraps it into a StepResult.
     */
    protected abstract StepResult doExecute(WorkflowContext context, WorkflowInstance instance);

    // ── Helpers ────────────────────────────────────────────────────────

    protected StepResult skipped(WorkflowContext context, WorkflowInstance instance, String reason) {
        Instant now = Instant.now();
        transitionTo(StepState.skipped(reason));
        instance.getAuditLog().append(AuditEntry.stepSkipped(id, name, now, reason));
        instance.getEventBus().publish(
                WorkflowEvent.stepSkipped(instance.getInstanceId(), id, name, now));
        return StepResult.skipped(id, now, reason);
    }

    protected void transitionTo(StepState newState) {
        state.set(newState);
    }

    public StepState getCurrentState() {
        return state.get();
    }
}
```

---

### 6.9 Core — WorkflowDefinition

```java
package com.workflow.engine.core;

import java.time.Instant;
import java.util.Objects;

/**
 * Immutable descriptor of a workflow.
 *
 * A WorkflowDefinition is created once via WorkflowBuilder and never mutated.
 * Multiple WorkflowInstances can share the same definition concurrently.
 *
 * version is incremented each time the definition is intentionally changed
 * so that running instances can be associated with the exact definition
 * they started from.
 */
public final class WorkflowDefinition {

    private final String id;
    private final String name;
    private final String description;
    private final int    version;
    private final Step   rootStep;
    private final Instant createdAt;

    WorkflowDefinition(String id, String name, String description, int version, Step rootStep) {
        this.id          = Objects.requireNonNull(id);
        this.name        = Objects.requireNonNull(name);
        this.description = description != null ? description : "";
        this.version     = version;
        this.rootStep    = Objects.requireNonNull(rootStep);
        this.createdAt   = Instant.now();
    }

    public String getId()          { return id; }
    public String getName()        { return name; }
    public String getDescription() { return description; }
    public int    getVersion()     { return version; }
    public Step   getRootStep()    { return rootStep; }
    public Instant getCreatedAt()  { return createdAt; }

    @Override
    public String toString() {
        return "WorkflowDefinition{id='" + id + "', version=" + version + ", name='" + name + "'}";
    }
}
```

---

### 6.10 Core — WorkflowInstance

```java
package com.workflow.engine.core;

import com.workflow.engine.audit.AuditLog;
import com.workflow.engine.event.WorkflowEventBus;

import java.time.Instant;
import java.util.Objects;
import java.util.UUID;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicReference;

/**
 * Mutable runtime counterpart to an immutable WorkflowDefinition.
 *
 * One instance is created per workflow execution. It carries:
 * - Status (CREATED → RUNNING → COMPLETED / FAILED / CANCELLED / PAUSED)
 * - The WorkflowContext (shared data bag)
 * - The AuditLog (append-only event journal)
 * - The EventBus (observer notifications)
 * - Pause/resume/cancel synchronization primitives
 */
public final class WorkflowInstance {

    private final String                     instanceId;
    private final String                     definitionId;
    private final int                        definitionVersion;
    private final WorkflowContext            context;
    private final AuditLog                   auditLog;
    private final WorkflowEventBus           eventBus;
    private final AtomicReference<WorkflowStatus> status;
    private final Instant                    createdAt;

    /**
     * Pause/resume: when paused, a new latch is installed;
     * resume() counts it down; awaitResumeOrCancel() blocks on it.
     */
    private volatile CountDownLatch pauseLatch;

    public WorkflowInstance(String definitionId, int definitionVersion,
                            WorkflowContext context, WorkflowEventBus eventBus) {
        this.instanceId        = UUID.randomUUID().toString();
        this.definitionId      = Objects.requireNonNull(definitionId);
        this.definitionVersion = definitionVersion;
        this.context           = Objects.requireNonNull(context);
        this.eventBus          = Objects.requireNonNull(eventBus);
        this.auditLog          = new AuditLog(instanceId);
        this.status            = new AtomicReference<>(WorkflowStatus.CREATED);
        this.createdAt         = Instant.now();
    }

    // ── Lifecycle transitions ──────────────────────────────────────────

    public boolean start() {
        return status.compareAndSet(WorkflowStatus.CREATED, WorkflowStatus.RUNNING);
    }

    public boolean pause() {
        if (status.compareAndSet(WorkflowStatus.RUNNING, WorkflowStatus.PAUSED)) {
            pauseLatch = new CountDownLatch(1);
            return true;
        }
        return false;
    }

    public boolean resume() {
        if (status.compareAndSet(WorkflowStatus.PAUSED, WorkflowStatus.RUNNING)) {
            CountDownLatch latch = pauseLatch;
            if (latch != null) latch.countDown();
            return true;
        }
        return false;
    }

    public boolean cancel() {
        WorkflowStatus prev = status.getAndSet(WorkflowStatus.CANCELLED);
        if (prev == WorkflowStatus.PAUSED) {
            CountDownLatch latch = pauseLatch;
            if (latch != null) latch.countDown(); // unblock waiting steps
        }
        return prev != WorkflowStatus.CANCELLED;
    }

    public void complete() {
        status.set(WorkflowStatus.COMPLETED);
    }

    public void fail() {
        status.set(WorkflowStatus.FAILED);
    }

    // ── Pause interlock ────────────────────────────────────────────────

    /**
     * Blocks the calling thread until the instance is resumed or cancelled.
     * Steps call this at the top of their execution loop.
     */
    public void awaitResumeOrCancel() {
        CountDownLatch latch = pauseLatch;
        if (latch != null) {
            try {
                latch.await();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                cancel();
            }
        }
    }

    // ── Queries ────────────────────────────────────────────────────────

    public boolean isRunning()    { return status.get() == WorkflowStatus.RUNNING; }
    public boolean isPaused()     { return status.get() == WorkflowStatus.PAUSED; }
    public boolean isCancelled()  { return status.get() == WorkflowStatus.CANCELLED; }
    public boolean isTerminated() {
        WorkflowStatus s = status.get();
        return s == WorkflowStatus.COMPLETED
            || s == WorkflowStatus.FAILED
            || s == WorkflowStatus.CANCELLED;
    }

    // ── Accessors ──────────────────────────────────────────────────────

    public String             getInstanceId()        { return instanceId; }
    public String             getDefinitionId()      { return definitionId; }
    public int                getDefinitionVersion() { return definitionVersion; }
    public WorkflowContext    getContext()            { return context; }
    public AuditLog           getAuditLog()          { return auditLog; }
    public WorkflowEventBus   getEventBus()          { return eventBus; }
    public WorkflowStatus     getStatus()            { return status.get(); }
    public Instant            getCreatedAt()         { return createdAt; }
}
```

---

### 6.11 Audit — AuditEntry & AuditLog

```java
package com.workflow.engine.audit;

import com.workflow.engine.core.StepResult;

import java.time.Instant;
import java.util.Map;

/**
 * Immutable record of a single event in the workflow audit trail.
 */
public record AuditEntry(
        String      entryId,
        String      instanceId,
        String      stepId,
        String      stepName,
        AuditEvent  event,
        Instant     timestamp,
        Map<String, Object> contextSnapshot,
        String      detail
) {

    public enum AuditEvent {
        STEP_STARTED, STEP_COMPLETED, STEP_FAILED, STEP_SKIPPED,
        WORKFLOW_STARTED, WORKFLOW_COMPLETED, WORKFLOW_FAILED,
        WORKFLOW_PAUSED, WORKFLOW_RESUMED, WORKFLOW_CANCELLED
    }

    // ── Static factories ───────────────────────────────────────────────

    public static AuditEntry stepStarted(String stepId, String stepName,
                                          Instant ts, Map<String, Object> snapshot) {
        return new AuditEntry(java.util.UUID.randomUUID().toString(), null,
                stepId, stepName, AuditEvent.STEP_STARTED, ts, snapshot, "");
    }

    public static AuditEntry stepFinished(String stepId, String stepName,
                                           Instant ts, StepResult result) {
        AuditEvent ev = result.isSuccess() ? AuditEvent.STEP_COMPLETED
                      : result.isSkipped() ? AuditEvent.STEP_SKIPPED
                      : AuditEvent.STEP_FAILED;
        String detail = result.error()
                .map(Throwable::getMessage)
                .orElse("");
        return new AuditEntry(java.util.UUID.randomUUID().toString(), null,
                stepId, stepName, ev, ts, Map.of(), detail);
    }

    public static AuditEntry stepSkipped(String stepId, String stepName,
                                          Instant ts, String reason) {
        return new AuditEntry(java.util.UUID.randomUUID().toString(), null,
                stepId, stepName, AuditEvent.STEP_SKIPPED, ts, Map.of(), reason);
    }

    public static AuditEntry workflowEvent(AuditEvent event, Instant ts, String detail) {
        return new AuditEntry(java.util.UUID.randomUUID().toString(), null,
                null, null, event, ts, Map.of(), detail);
    }
}
```

```java
package com.workflow.engine.audit;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * Append-only, thread-safe audit journal for a single WorkflowInstance.
 *
 * Uses a ReadWriteLock so many readers (monitoring threads) can proceed
 * concurrently while a single writer appends an entry.
 */
public final class AuditLog {

    private final String instanceId;
    private final List<AuditEntry> entries;
    private final ReadWriteLock lock;

    public AuditLog(String instanceId) {
        this.instanceId = instanceId;
        this.entries    = new ArrayList<>();
        this.lock       = new ReentrantReadWriteLock();
    }

    public void append(AuditEntry entry) {
        lock.writeLock().lock();
        try {
            entries.add(entry);
        } finally {
            lock.writeLock().unlock();
        }
    }

    /** Returns an unmodifiable snapshot of all entries at this moment. */
    public List<AuditEntry> getEntries() {
        lock.readLock().lock();
        try {
            return Collections.unmodifiableList(new ArrayList<>(entries));
        } finally {
            lock.readLock().unlock();
        }
    }

    public String getInstanceId() { return instanceId; }

    public void print() {
        getEntries().forEach(e ->
                System.out.printf("[AUDIT] %s | %-22s | step=%-30s | %s%n",
                        e.timestamp(), e.event(), e.stepId(), e.detail()));
    }
}
```

---

### 6.12 Event — WorkflowEvent (sealed)

```java
package com.workflow.engine.event;

import java.time.Instant;

/**
 * Sealed hierarchy of all events the engine can emit.
 *
 * Using sealed records gives exhaustive pattern matching for event handlers
 * and makes it impossible to add an event type without updating all switches.
 */
public sealed interface WorkflowEvent
        permits WorkflowEvent.WorkflowStarted,
                WorkflowEvent.WorkflowCompleted,
                WorkflowEvent.WorkflowFailed,
                WorkflowEvent.WorkflowPaused,
                WorkflowEvent.WorkflowResumed,
                WorkflowEvent.WorkflowCancelled,
                WorkflowEvent.StepStarted,
                WorkflowEvent.StepCompleted,
                WorkflowEvent.StepFailed,
                WorkflowEvent.StepSkipped {

    String instanceId();
    Instant occurredAt();

    // ── Workflow-level events ──────────────────────────────────────────
    record WorkflowStarted   (String instanceId, String definitionId, Instant occurredAt) implements WorkflowEvent {}
    record WorkflowCompleted (String instanceId, Instant occurredAt)                      implements WorkflowEvent {}
    record WorkflowFailed    (String instanceId, String reason, Instant occurredAt)       implements WorkflowEvent {}
    record WorkflowPaused    (String instanceId, Instant occurredAt)                      implements WorkflowEvent {}
    record WorkflowResumed   (String instanceId, Instant occurredAt)                      implements WorkflowEvent {}
    record WorkflowCancelled (String instanceId, Instant occurredAt)                      implements WorkflowEvent {}

    // ── Step-level events ──────────────────────────────────────────────
    record StepStarted   (String instanceId, String stepId, String stepName, Instant occurredAt)                       implements WorkflowEvent {}
    record StepCompleted (String instanceId, String stepId, String stepName, Instant occurredAt)                       implements WorkflowEvent {}
    record StepFailed    (String instanceId, String stepId, String stepName, Instant occurredAt, Throwable cause)      implements WorkflowEvent {}
    record StepSkipped   (String instanceId, String stepId, String stepName, Instant occurredAt)                       implements WorkflowEvent {}

    // ── Static factories ───────────────────────────────────────────────

    static WorkflowEvent workflowStarted(String instanceId, String defId) {
        return new WorkflowStarted(instanceId, defId, Instant.now());
    }
    static WorkflowEvent workflowCompleted(String instanceId) {
        return new WorkflowCompleted(instanceId, Instant.now());
    }
    static WorkflowEvent workflowFailed(String instanceId, String reason) {
        return new WorkflowFailed(instanceId, reason, Instant.now());
    }
    static WorkflowEvent workflowPaused(String instanceId) {
        return new WorkflowPaused(instanceId, Instant.now());
    }
    static WorkflowEvent workflowResumed(String instanceId) {
        return new WorkflowResumed(instanceId, Instant.now());
    }
    static WorkflowEvent workflowCancelled(String instanceId) {
        return new WorkflowCancelled(instanceId, Instant.now());
    }
    static WorkflowEvent stepStarted(String instanceId, String stepId, String stepName, Instant ts) {
        return new StepStarted(instanceId, stepId, stepName, ts);
    }
    static WorkflowEvent stepCompleted(String instanceId, String stepId, String stepName, Instant ts) {
        return new StepCompleted(instanceId, stepId, stepName, ts);
    }
    static WorkflowEvent stepFailed(String instanceId, String stepId, String stepName, Instant ts, Throwable cause) {
        return new StepFailed(instanceId, stepId, stepName, ts, cause);
    }
    static WorkflowEvent stepSkipped(String instanceId, String stepId, String stepName, Instant ts) {
        return new StepSkipped(instanceId, stepId, stepName, ts);
    }
}
```

---

### 6.13 Event — WorkflowEventListener & WorkflowEventBus

```java
package com.workflow.engine.event;

/**
 * Functional interface for event subscribers.
 * Implementations must be non-blocking; heavy processing should be
 * dispatched to a separate executor inside the implementation.
 */
@FunctionalInterface
public interface WorkflowEventListener {
    void onEvent(WorkflowEvent event);
}
```

```java
package com.workflow.engine.event;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Simple synchronous event bus.
 *
 * Listeners are invoked on the same thread that publishes the event.
 * CopyOnWriteArrayList provides safe concurrent registration and iteration
 * without holding a lock during dispatch.
 *
 * For high-throughput scenarios, replace dispatch with an async queue
 * (e.g., a single-threaded ExecutorService per subscriber).
 */
public final class WorkflowEventBus {

    private final List<WorkflowEventListener> listeners = new CopyOnWriteArrayList<>();

    public void subscribe(WorkflowEventListener listener) {
        if (listener != null) listeners.add(listener);
    }

    public void unsubscribe(WorkflowEventListener listener) {
        listeners.remove(listener);
    }

    public void publish(WorkflowEvent event) {
        if (event == null) return;
        for (WorkflowEventListener listener : listeners) {
            try {
                listener.onEvent(event);
            } catch (Exception ex) {
                // A bad listener must never disrupt the engine
                System.err.println("[EventBus] Listener threw exception: " + ex.getMessage());
            }
        }
    }
}
```

---

### 6.14 Error Handling — Chain of Responsibility

```java
package com.workflow.engine.error;

import com.workflow.engine.core.Step;
import com.workflow.engine.core.StepResult;
import com.workflow.engine.core.WorkflowContext;
import com.workflow.engine.core.WorkflowInstance;

/**
 * Handler interface in the Chain of Responsibility for step errors.
 * Each handler either handles the error (returns a recovery StepResult)
 * or delegates to the next handler.
 */
public interface ErrorHandler {
    /**
     * @param step     the step that failed
     * @param context  the current workflow context
     * @param instance the running instance
     * @param cause    the exception that caused the failure
     * @param next     the next handler in the chain (null if this is the last)
     * @return a StepResult representing the recovery outcome
     */
    StepResult handle(Step step, WorkflowContext context, WorkflowInstance instance,
                      Throwable cause, ErrorHandler next);
}
```

```java
package com.workflow.engine.error;

import com.workflow.engine.core.Step;
import com.workflow.engine.core.StepResult;
import com.workflow.engine.core.WorkflowContext;
import com.workflow.engine.core.WorkflowInstance;

import java.time.Instant;

/**
 * Retries the step up to maxRetries times with a configurable delay.
 * Delegates to the next handler if all retries are exhausted.
 */
public final class RetryErrorHandler implements ErrorHandler {

    @Override
    public StepResult handle(Step step, WorkflowContext context,
                             WorkflowInstance instance, Throwable cause,
                             ErrorHandler next) {
        ErrorPolicy policy = step.getErrorPolicy();
        int maxRetries     = policy.maxRetries();
        long delayMs       = policy.retryDelayMs();

        if (maxRetries <= 0) {
            return next != null
                    ? next.handle(step, context, instance, cause, null)
                    : StepResult.failure(step.getId(), Instant.now(), Instant.now(), cause);
        }

        Throwable lastCause = cause;
        for (int attempt = 1; attempt <= maxRetries; attempt++) {
            System.out.printf("[Retry] Step '%s' attempt %d/%d after %dms%n",
                    step.getName(), attempt, maxRetries, delayMs);
            if (delayMs > 0) {
                try {
                    Thread.sleep(delayMs);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
            StepResult result = step.execute(context, instance);
            if (result.isSuccess()) {
                System.out.printf("[Retry] Step '%s' succeeded on attempt %d%n",
                        step.getName(), attempt);
                return result;
            }
            lastCause = result.error().orElse(lastCause);
        }

        // All retries exhausted — delegate to next handler
        return next != null
                ? next.handle(step, context, instance, lastCause, null)
                : StepResult.failure(step.getId(), Instant.now(), Instant.now(), lastCause);
    }
}
```

```java
package com.workflow.engine.error;

import com.workflow.engine.core.Step;
import com.workflow.engine.core.StepResult;
import com.workflow.engine.core.WorkflowContext;
import com.workflow.engine.core.WorkflowInstance;

import java.time.Instant;

/**
 * If the step's error policy says skipOnFailure, marks the step SKIPPED
 * and allows the workflow to continue. Otherwise delegates to next handler.
 */
public final class SkipErrorHandler implements ErrorHandler {

    @Override
    public StepResult handle(Step step, WorkflowContext context,
                             WorkflowInstance instance, Throwable cause,
                             ErrorHandler next) {
        if (step.getErrorPolicy().skipOnFailure()) {
            System.out.printf("[Skip] Step '%s' skipped due to failure: %s%n",
                    step.getName(), cause.getMessage());
            return StepResult.skipped(step.getId(), Instant.now(),
                    "Skipped after failure: " + cause.getMessage());
        }
        return next != null
                ? next.handle(step, context, instance, cause, null)
                : StepResult.failure(step.getId(), Instant.now(), Instant.now(), cause);
    }
}
```

```java
package com.workflow.engine.error;

import com.workflow.engine.core.Step;
import com.workflow.engine.core.StepResult;
import com.workflow.engine.core.WorkflowContext;
import com.workflow.engine.core.WorkflowInstance;

import java.time.Instant;

/**
 * Terminal handler: transitions the workflow to FAILED state.
 * This is always the last handler in the chain.
 */
public final class FailWorkflowErrorHandler implements ErrorHandler {

    @Override
    public StepResult handle(Step step, WorkflowContext context,
                             WorkflowInstance instance, Throwable cause,
                             ErrorHandler next) {
        System.err.printf("[FailWorkflow] Step '%s' caused workflow '%s' to fail: %s%n",
                step.getName(), instance.getInstanceId(), cause.getMessage());
        instance.fail();
        return StepResult.failure(step.getId(), Instant.now(), Instant.now(), cause);
    }
}
```

```java
package com.workflow.engine.error;

import com.workflow.engine.core.Step;
import com.workflow.engine.core.StepResult;
import com.workflow.engine.core.WorkflowContext;
import com.workflow.engine.core.WorkflowInstance;

import java.util.ArrayList;
import java.util.List;

/**
 * Assembles the error-handling chain and invokes it.
 *
 * Default chain (built by ErrorHandlerChain.defaultChain()):
 *   RetryErrorHandler → SkipErrorHandler → FailWorkflowErrorHandler
 *
 * The chain is constructed once and reused across all step executions.
 */
public final class ErrorHandlerChain {

    private final List<ErrorHandler> handlers;

    public ErrorHandlerChain(List<ErrorHandler> handlers) {
        if (handlers == null || handlers.isEmpty()) {
            throw new IllegalArgumentException("ErrorHandlerChain requires at least one handler");
        }
        this.handlers = new ArrayList<>(handlers);
    }

    public static ErrorHandlerChain defaultChain() {
        return new ErrorHandlerChain(List.of(
                new RetryErrorHandler(),
                new SkipErrorHandler(),
                new FailWorkflowErrorHandler()
        ));
    }

    /**
     * Executes the chain for the given failure.
     * Builds a recursive delegation structure from the handler list.
     */
    public StepResult handle(Step step, WorkflowContext context,
                             WorkflowInstance instance, Throwable cause) {
        return invokeChain(0, step, context, instance, cause);
    }

    private StepResult invokeChain(int index, Step step, WorkflowContext context,
                                   WorkflowInstance instance, Throwable cause) {
        if (index >= handlers.size()) {
            // Safeguard — should not happen if FailWorkflowErrorHandler is last
            instance.fail();
            return StepResult.failure(step.getId(),
                    java.time.Instant.now(), java.time.Instant.now(), cause);
        }
        ErrorHandler current = handlers.get(index);
        ErrorHandler next    = (index + 1 < handlers.size())
                ? (s, ctx, inst, ex, n) -> invokeChain(index + 1, s, ctx, inst, ex)
                : null;
        return current.handle(step, context, instance, cause, next);
    }
}
```

---

### 6.15 Steps — AutomatedStep

```java
package com.workflow.engine.steps;

import com.workflow.engine.core.*;
import com.workflow.engine.error.ErrorHandlerChain;
import com.workflow.engine.error.ErrorPolicy;

import java.time.Instant;
import java.util.Objects;
import java.util.function.Function;

/**
 * A leaf step that executes a programmatic action (a Function<WorkflowContext, Object>).
 *
 * This is the "Command" pattern's ConcreteCommand — the lambda/method-reference
 * passed at construction time IS the command body.
 *
 * Error handling is delegated to the configured ErrorHandlerChain.
 */
public final class AutomatedStep extends AbstractStep {

    private final Function<WorkflowContext, Object> command;
    private final ErrorHandlerChain errorHandlerChain;

    public AutomatedStep(String id, String name, String description,
                         ErrorPolicy errorPolicy,
                         Function<WorkflowContext, Object> command) {
        super(id, name, description, errorPolicy);
        this.command           = Objects.requireNonNull(command, "Command must not be null");
        this.errorHandlerChain = ErrorHandlerChain.defaultChain();
    }

    /** Convenience constructor: no error policy override, fail-fast default. */
    public AutomatedStep(String id, String name, Function<WorkflowContext, Object> command) {
        this(id, name, "", ErrorPolicy.FAIL_FAST, command);
    }

    @Override
    protected StepResult doExecute(WorkflowContext context, WorkflowInstance instance) {
        Instant start = Instant.now();
        try {
            Object output = command.apply(context);
            return StepResult.success(getId(), start, Instant.now(), output);
        } catch (Exception ex) {
            // Pass to error-handler chain ONLY if caused by the command itself
            // (AbstractStep already wraps broader exceptions before doExecute)
            return errorHandlerChain.handle(this, context, instance, ex);
        }
    }
}
```

---

### 6.16 Steps — ManualApprovalStep

```java
package com.workflow.engine.steps;

import com.workflow.engine.core.*;
import com.workflow.engine.error.ErrorPolicy;

import java.time.Instant;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * Pauses workflow execution until a human actor approves or rejects the step,
 * or until a timeout expires.
 *
 * Usage in a running workflow:
 *   approvalStep.signal(true, "manager@corp.com");  // approve
 *   approvalStep.signal(false, "manager@corp.com"); // reject
 *
 * Thread-safety: approve/reject signals are sent from external threads
 * (e.g., a web-request handler). The latch ensures safe handoff.
 */
public final class ManualApprovalStep extends AbstractStep {

    private final long    timeoutMs;
    private final CountDownLatch latch       = new CountDownLatch(1);
    private final AtomicBoolean  approved    = new AtomicBoolean(false);
    private volatile String      actor       = "unknown";
    private volatile String      comment     = "";

    public ManualApprovalStep(String id, String name, String description, long timeoutMs) {
        super(id, name, description, ErrorPolicy.FAIL_FAST);
        this.timeoutMs = timeoutMs > 0 ? timeoutMs : Long.MAX_VALUE;
    }

    public ManualApprovalStep(String id, String name) {
        this(id, name, "Manual approval required", 0);
    }

    /**
     * Called by an external actor to approve or reject this step.
     *
     * @param isApproved true=approve, false=reject
     * @param actorId    identifier of the approving/rejecting party
     */
    public void signal(boolean isApproved, String actorId) {
        signal(isApproved, actorId, "");
    }

    public void signal(boolean isApproved, String actorId, String comment) {
        this.approved.set(isApproved);
        this.actor   = actorId != null ? actorId : "unknown";
        this.comment = comment != null ? comment : "";
        latch.countDown();
    }

    @Override
    protected StepResult doExecute(WorkflowContext context, WorkflowInstance instance) {
        Instant start = Instant.now();
        System.out.printf("[ManualApproval] Step '%s' is waiting for approval (timeout=%dms)%n",
                getName(), timeoutMs);

        try {
            boolean signalled = latch.await(timeoutMs, TimeUnit.MILLISECONDS);

            if (!signalled) {
                // Timeout expired — treat as rejection by policy
                Throwable timeout = new RuntimeException(
                        "Approval timeout after " + timeoutMs + "ms for step: " + getName());
                return StepResult.failure(getId(), start, Instant.now(), timeout);
            }

            if (approved.get()) {
                System.out.printf("[ManualApproval] Step '%s' APPROVED by %s: %s%n",
                        getName(), actor, comment);
                context.put("approval." + getId() + ".actor",   actor);
                context.put("approval." + getId() + ".comment", comment);
                return StepResult.success(getId(), start, Instant.now(), "Approved by " + actor);
            } else {
                System.out.printf("[ManualApproval] Step '%s' REJECTED by %s: %s%n",
                        getName(), actor, comment);
                Throwable rejection = new RuntimeException(
                        "Step '" + getName() + "' rejected by " + actor + ": " + comment);
                return StepResult.failure(getId(), start, Instant.now(), rejection);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return StepResult.failure(getId(), start, Instant.now(),
                    new RuntimeException("Interrupted while awaiting approval"));
        }
    }
}
```

---

### 6.17 Steps — SequentialStep (Composite)

```java
package com.workflow.engine.steps;

import com.workflow.engine.core.*;
import com.workflow.engine.error.ErrorHandlerChain;
import com.workflow.engine.error.ErrorPolicy;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

/**
 * Composite step that executes its children in order.
 *
 * If a child step fails:
 * - The ErrorHandlerChain is consulted (retry / skip / fail-workflow).
 * - If the chain resolves the failure (skip), execution continues.
 * - If the chain marks the workflow FAILED, this step returns FAILED immediately.
 *
 * This is the "Composite" node in the Composite pattern.
 */
public final class SequentialStep extends AbstractStep {

    private final List<Step>        children;
    private final ErrorHandlerChain errorHandlerChain;

    private SequentialStep(String id, String name, String description,
                           ErrorPolicy errorPolicy, List<Step> children) {
        super(id, name, description, errorPolicy);
        this.children          = List.copyOf(children);
        this.errorHandlerChain = ErrorHandlerChain.defaultChain();
    }

    public static Builder builder(String id, String name) {
        return new Builder(id, name);
    }

    @Override
    protected StepResult doExecute(WorkflowContext context, WorkflowInstance instance) {
        Instant start = Instant.now();

        for (Step child : children) {
            if (instance.isCancelled()) {
                return StepResult.skipped(getId(), Instant.now(), "Workflow cancelled");
            }

            StepResult childResult = child.execute(context, instance);

            if (childResult.isFailure()) {
                // Delegate to the error handler chain
                StepResult recovered = errorHandlerChain.handle(
                        child, context, instance,
                        childResult.error().orElse(new RuntimeException("Step failed")));

                if (instance.isTerminated() && !instance.isRunning()) {
                    // Workflow was failed by the chain
                    return StepResult.failure(getId(), start, Instant.now(),
                            recovered.error().orElse(new RuntimeException("Child step failed")));
                }
                // If we're still running, the chain resolved it (skip) — continue
            }
        }

        return StepResult.success(getId(), start, Instant.now());
    }

    public List<Step> getChildren() { return children; }

    // ── Builder ────────────────────────────────────────────────────────

    public static final class Builder {
        private final String id;
        private final String name;
        private String       description = "";
        private ErrorPolicy  errorPolicy = ErrorPolicy.FAIL_FAST;
        private final List<Step> children = new ArrayList<>();

        private Builder(String id, String name) {
            this.id   = Objects.requireNonNull(id);
            this.name = Objects.requireNonNull(name);
        }

        public Builder description(String d)    { this.description = d; return this; }
        public Builder errorPolicy(ErrorPolicy p){ this.errorPolicy = p; return this; }
        public Builder step(Step s)             { children.add(s);  return this; }

        public SequentialStep build() {
            if (children.isEmpty()) throw new IllegalStateException("SequentialStep needs at least one child");
            return new SequentialStep(id, name, description, errorPolicy, children);
        }
    }
}
```

---

### 6.18 Steps — ParallelStep

```java
package com.workflow.engine.steps;

import com.workflow.engine.core.*;
import com.workflow.engine.error.ErrorHandlerChain;
import com.workflow.engine.error.ErrorPolicy;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.*;
import java.util.stream.Collectors;

/**
 * Composite step that executes all children concurrently using CompletableFuture.
 *
 * Key design decisions:
 * - Each child runs on the provided ExecutorService (injected for testability).
 * - All children share the same WorkflowContext (ConcurrentHashMap guarantees safety).
 * - Waits for ALL children before returning — either allOf completes or one fails.
 * - If ANY child fails and failFast=true, remaining futures are cancelled.
 * - If ANY child fails and failFast=false, collects all results then evaluates.
 */
public final class ParallelStep extends AbstractStep {

    private final List<Step>        children;
    private final ExecutorService   executor;
    private final boolean           failFast;
    private final ErrorHandlerChain errorHandlerChain;

    private ParallelStep(String id, String name, String description,
                         ErrorPolicy errorPolicy, List<Step> children,
                         ExecutorService executor, boolean failFast) {
        super(id, name, description, errorPolicy);
        this.children          = List.copyOf(children);
        this.executor          = Objects.requireNonNull(executor);
        this.failFast          = failFast;
        this.errorHandlerChain = ErrorHandlerChain.defaultChain();
    }

    public static Builder builder(String id, String name, ExecutorService executor) {
        return new Builder(id, name, executor);
    }

    @Override
    protected StepResult doExecute(WorkflowContext context, WorkflowInstance instance) {
        Instant start = Instant.now();

        // Submit all children as CompletableFuture tasks
        List<CompletableFuture<StepResult>> futures = children.stream()
                .map(child -> CompletableFuture.supplyAsync(
                        () -> child.execute(context, instance), executor))
                .collect(Collectors.toList());

        List<StepResult> results = new ArrayList<>();

        if (failFast) {
            // Cancel remaining on first failure
            try {
                CompletableFuture<Void> allOf = CompletableFuture.allOf(
                        futures.toArray(new CompletableFuture[0]));
                allOf.get(); // blocks until all complete or one throws
                for (CompletableFuture<StepResult> f : futures) {
                    results.add(f.get());
                }
            } catch (ExecutionException ex) {
                futures.forEach(f -> f.cancel(true));
                Throwable cause = ex.getCause() != null ? ex.getCause() : ex;
                return errorHandlerChain.handle(this, context, instance, cause);
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
                futures.forEach(f -> f.cancel(true));
                return StepResult.failure(getId(), start, Instant.now(), ex);
            }
        } else {
            // Collect all, even if some fail
            for (CompletableFuture<StepResult> future : futures) {
                try {
                    results.add(future.get());
                } catch (ExecutionException ex) {
                    Throwable cause = ex.getCause() != null ? ex.getCause() : ex;
                    results.add(StepResult.failure(
                            "unknown", start, Instant.now(), cause));
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                    results.add(StepResult.failure(
                            "unknown", start, Instant.now(), ex));
                }
            }
        }

        // Evaluate collected results
        List<StepResult> failures = results.stream()
                .filter(StepResult::isFailure)
                .collect(Collectors.toList());

        if (failures.isEmpty()) {
            return StepResult.success(getId(), start, Instant.now(),
                    results.size() + " parallel steps completed");
        }

        // Some children failed — use error handler chain
        Throwable firstCause = failures.get(0).error()
                .orElse(new RuntimeException("Parallel child step failed"));
        return errorHandlerChain.handle(this, context, instance, firstCause);
    }

    public List<Step> getChildren() { return children; }

    // ── Builder ────────────────────────────────────────────────────────

    public static final class Builder {
        private final String          id;
        private final String          name;
        private final ExecutorService executor;
        private String                description = "";
        private ErrorPolicy           errorPolicy = ErrorPolicy.FAIL_FAST;
        private boolean               failFast    = true;
        private final List<Step>      children    = new ArrayList<>();

        private Builder(String id, String name, ExecutorService executor) {
            this.id       = Objects.requireNonNull(id);
            this.name     = Objects.requireNonNull(name);
            this.executor = Objects.requireNonNull(executor);
        }

        public Builder description(String d)     { this.description = d; return this; }
        public Builder errorPolicy(ErrorPolicy p) { this.errorPolicy = p; return this; }
        public Builder failFast(boolean ff)      { this.failFast    = ff; return this; }
        public Builder step(Step s)              { children.add(s);  return this; }

        public ParallelStep build() {
            if (children.isEmpty()) throw new IllegalStateException("ParallelStep needs at least one child");
            return new ParallelStep(id, name, description, errorPolicy, children, executor, failFast);
        }
    }
}
```

---

### 6.19 Steps — ConditionalStep

```java
package com.workflow.engine.steps;

import com.workflow.engine.core.*;
import com.workflow.engine.error.ErrorPolicy;

import java.time.Instant;
import java.util.Objects;
import java.util.Optional;
import java.util.function.Predicate;

/**
 * Evaluates a Predicate<WorkflowContext> and executes one of two branches:
 * - thenStep  if the predicate evaluates to true
 * - elseStep  if the predicate evaluates to false (optional)
 *
 * Implements conditional branching — "if/else" in the workflow DSL.
 *
 * The predicate is evaluated lazily at execution time so it reads the
 * actual runtime values written by preceding steps.
 */
public final class ConditionalStep extends AbstractStep {

    private final Predicate<WorkflowContext> condition;
    private final Step                       thenStep;
    private final Optional<Step>             elseStep;

    public ConditionalStep(String id, String name, String description,
                           Predicate<WorkflowContext> condition,
                           Step thenStep, Step elseStep) {
        super(id, name, description, ErrorPolicy.FAIL_FAST);
        this.condition = Objects.requireNonNull(condition, "Condition must not be null");
        this.thenStep  = Objects.requireNonNull(thenStep,  "Then-step must not be null");
        this.elseStep  = Optional.ofNullable(elseStep);
    }

    /** Constructor without else branch. */
    public ConditionalStep(String id, String name,
                           Predicate<WorkflowContext> condition, Step thenStep) {
        this(id, name, "Conditional: " + name, condition, thenStep, null);
    }

    @Override
    protected StepResult doExecute(WorkflowContext context, WorkflowInstance instance) {
        Instant start    = Instant.now();
        boolean condVal  = condition.test(context);

        System.out.printf("[Conditional] Step '%s' condition evaluated to: %b%n",
                getName(), condVal);

        if (condVal) {
            return thenStep.execute(context, instance);
        } else {
            return elseStep
                    .map(step -> step.execute(context, instance))
                    .orElseGet(() -> StepResult.skipped(getId(), start,
                            "No else branch defined; condition was false"));
        }
    }

    public Predicate<WorkflowContext> getCondition() { return condition; }
    public Step getThenStep()                        { return thenStep; }
    public Optional<Step> getElseStep()              { return elseStep; }
}
```

---

### 6.20 Steps — LoopStep

```java
package com.workflow.engine.steps;

import com.workflow.engine.core.*;
import com.workflow.engine.error.ErrorHandlerChain;
import com.workflow.engine.error.ErrorPolicy;

import java.time.Instant;
import java.util.Objects;
import java.util.function.Predicate;

/**
 * Repeats its body step while the condition holds (while-loop semantics).
 *
 * Safety features:
 * - maxIterations guards against infinite loops.
 * - Checks instance.isCancelled() on every iteration.
 * - The condition is evaluated BEFORE executing the body (pre-test loop).
 *
 * The body step can be any Step — including a SequentialStep or ParallelStep —
 * enabling complex repeated sub-workflows.
 */
public final class LoopStep extends AbstractStep {

    private final Predicate<WorkflowContext> condition;
    private final Step                       body;
    private final int                        maxIterations;
    private final ErrorHandlerChain          errorHandlerChain;

    public LoopStep(String id, String name, String description,
                    Predicate<WorkflowContext> condition, Step body, int maxIterations) {
        super(id, name, description, ErrorPolicy.FAIL_FAST);
        this.condition         = Objects.requireNonNull(condition);
        this.body              = Objects.requireNonNull(body);
        this.maxIterations     = maxIterations > 0 ? maxIterations : Integer.MAX_VALUE;
        this.errorHandlerChain = ErrorHandlerChain.defaultChain();
    }

    public LoopStep(String id, String name,
                    Predicate<WorkflowContext> condition, Step body) {
        this(id, name, "Loop: " + name, condition, body, 1000);
    }

    @Override
    protected StepResult doExecute(WorkflowContext context, WorkflowInstance instance) {
        Instant start     = Instant.now();
        int     iteration = 0;

        while (!instance.isCancelled() && condition.test(context)) {
            if (iteration >= maxIterations) {
                Throwable ex = new RuntimeException(
                        "LoopStep '" + getName() + "' exceeded maxIterations=" + maxIterations);
                return errorHandlerChain.handle(this, context, instance, ex);
            }

            System.out.printf("[Loop] Step '%s' iteration %d%n", getName(), iteration + 1);

            StepResult bodyResult = body.execute(context, instance);

            if (bodyResult.isFailure()) {
                StepResult recovered = errorHandlerChain.handle(
                        body, context, instance,
                        bodyResult.error().orElse(new RuntimeException("Loop body failed")));
                if (instance.isTerminated()) {
                    return StepResult.failure(getId(), start, Instant.now(),
                            recovered.error().orElse(new RuntimeException("Loop body failed")));
                }
                // Skip resolved — continue next iteration
            }

            iteration++;
        }

        System.out.printf("[Loop] Step '%s' completed after %d iterations%n",
                getName(), iteration);
        return StepResult.success(getId(), start, Instant.now(),
                "Completed " + iteration + " iterations");
    }

    public Predicate<WorkflowContext> getCondition() { return condition; }
    public Step getBody()                            { return body; }
    public int  getMaxIterations()                   { return maxIterations; }
}
```

---

### 6.21 Builder — WorkflowBuilder

```java
package com.workflow.engine.builder;

import com.workflow.engine.core.*;
import com.workflow.engine.error.ErrorPolicy;
import com.workflow.engine.steps.*;

import java.util.Objects;
import java.util.concurrent.ExecutorService;
import java.util.function.Function;
import java.util.function.Predicate;

/**
 * Fluent builder for WorkflowDefinition.
 *
 * Design: WorkflowBuilder owns the root SequentialStep.Builder; callers
 * add steps to it via the fluent API.  When build() is called, the root
 * step is sealed and wrapped in an immutable WorkflowDefinition.
 *
 * Thread-safety: builders are NOT thread-safe; a single thread constructs
 * and then calls build() once.
 */
public final class WorkflowBuilder {

    private final String  id;
    private       String  name;
    private       String  description = "";
    private       int     version     = 1;
    private final SequentialStep.Builder rootBuilder;

    private WorkflowBuilder(String id, String name) {
        this.id          = Objects.requireNonNull(id,   "Workflow id must not be null");
        this.name        = Objects.requireNonNull(name, "Workflow name must not be null");
        this.rootBuilder = SequentialStep.builder("root." + id, "Root of " + name);
    }

    // ── Factory ────────────────────────────────────────────────────────

    public static WorkflowBuilder create(String id, String name) {
        return new WorkflowBuilder(id, name);
    }

    // ── Workflow metadata ──────────────────────────────────────────────

    public WorkflowBuilder description(String desc)  { this.description = desc; return this; }
    public WorkflowBuilder version(int v)            { this.version = v;        return this; }

    // ── Automated step ─────────────────────────────────────────────────

    public WorkflowBuilder automated(String id, String name,
                                     Function<WorkflowContext, Object> command) {
        rootBuilder.step(new AutomatedStep(id, name, command));
        return this;
    }

    public WorkflowBuilder automated(String id, String name, ErrorPolicy policy,
                                     Function<WorkflowContext, Object> command) {
        rootBuilder.step(new AutomatedStep(id, name, "", policy, command));
        return this;
    }

    // ── Manual approval step ───────────────────────────────────────────

    public WorkflowBuilder manualApproval(ManualApprovalStep step) {
        rootBuilder.step(step);
        return this;
    }

    // ── Sequential sub-workflow ────────────────────────────────────────

    public WorkflowBuilder sequential(SequentialStep step) {
        rootBuilder.step(step);
        return this;
    }

    // ── Parallel step ──────────────────────────────────────────────────

    public WorkflowBuilder parallel(ParallelStep step) {
        rootBuilder.step(step);
        return this;
    }

    // ── Conditional step ───────────────────────────────────────────────

    public WorkflowBuilder condition(String id, String name,
                                     Predicate<WorkflowContext> condition,
                                     Step thenStep) {
        rootBuilder.step(new ConditionalStep(id, name, condition, thenStep));
        return this;
    }

    public WorkflowBuilder condition(String id, String name,
                                     Predicate<WorkflowContext> condition,
                                     Step thenStep, Step elseStep) {
        rootBuilder.step(new ConditionalStep(id, name, name, condition, thenStep, elseStep));
        return this;
    }

    // ── Loop step ──────────────────────────────────────────────────────

    public WorkflowBuilder loop(String id, String name,
                                Predicate<WorkflowContext> condition, Step body) {
        rootBuilder.step(new LoopStep(id, name, condition, body));
        return this;
    }

    public WorkflowBuilder loop(String id, String name, int maxIterations,
                                Predicate<WorkflowContext> condition, Step body) {
        rootBuilder.step(new LoopStep(id, name, name, condition, body, maxIterations));
        return this;
    }

    // ── Raw step ───────────────────────────────────────────────────────

    public WorkflowBuilder step(Step step) {
        rootBuilder.step(step);
        return this;
    }

    // ── Build ──────────────────────────────────────────────────────────

    public WorkflowDefinition build() {
        SequentialStep root = rootBuilder.build();
        return new WorkflowDefinition(id, name, description, version, root);
    }
}
```

---

### 6.22 Core — WorkflowEngine

```java
package com.workflow.engine.core;

import com.workflow.engine.audit.AuditEntry;
import com.workflow.engine.event.WorkflowEvent;
import com.workflow.engine.event.WorkflowEventBus;
import com.workflow.engine.event.WorkflowEventListener;

import java.time.Instant;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Orchestrates workflow execution.
 *
 * Responsibilities:
 * 1. Create and track WorkflowInstances.
 * 2. Execute the root step of the definition on the calling thread (or hand off).
 * 3. Transition the instance's WorkflowStatus based on root-step outcome.
 * 4. Emit workflow-level events (started, completed, failed, cancelled).
 * 5. Expose pause/resume/cancel operations keyed by instanceId.
 *
 * The engine itself is stateless with respect to business logic — it only
 * manages the lifecycle machinery and delegates execution to the step tree.
 */
public final class WorkflowEngine {

    private final WorkflowEventBus                     globalEventBus;
    private final Map<String, WorkflowInstance>        instances;

    public WorkflowEngine() {
        this(new WorkflowEventBus());
    }

    public WorkflowEngine(WorkflowEventBus globalEventBus) {
        this.globalEventBus = Objects.requireNonNull(globalEventBus);
        this.instances      = new ConcurrentHashMap<>();
    }

    // ── Registration ───────────────────────────────────────────────────

    public void subscribe(WorkflowEventListener listener) {
        globalEventBus.subscribe(listener);
    }

    // ── Execution ──────────────────────────────────────────────────────

    /**
     * Synchronously executes the workflow.
     * Blocks until the root step tree completes, fails, or is cancelled.
     *
     * @param definition the workflow to run
     * @param context    pre-populated context (may be empty)
     * @return the resulting WorkflowInstance (inspect status and auditLog)
     */
    public WorkflowInstance execute(WorkflowDefinition definition, WorkflowContext context) {
        Objects.requireNonNull(definition, "WorkflowDefinition must not be null");
        Objects.requireNonNull(context,    "WorkflowContext must not be null");

        WorkflowInstance instance = new WorkflowInstance(
                definition.getId(),
                definition.getVersion(),
                context,
                globalEventBus);

        instances.put(instance.getInstanceId(), instance);

        if (!instance.start()) {
            throw new IllegalStateException("Could not start workflow instance: " + instance.getInstanceId());
        }

        instance.getAuditLog().append(
                AuditEntry.workflowEvent(AuditEntry.AuditEvent.WORKFLOW_STARTED,
                        Instant.now(), definition.toString()));
        globalEventBus.publish(
                WorkflowEvent.workflowStarted(instance.getInstanceId(), definition.getId()));

        System.out.printf("%n[Engine] Starting workflow '%s' v%d (instance=%s)%n",
                definition.getName(), definition.getVersion(), instance.getInstanceId());

        try {
            StepResult result = definition.getRootStep().execute(context, instance);

            if (instance.isCancelled()) {
                finalizeCancelled(instance);
            } else if (result.isFailure() || instance.getStatus() == WorkflowStatus.FAILED) {
                finalizeFailure(instance, result);
            } else {
                finalizeSuccess(instance);
            }
        } catch (Exception ex) {
            // Catastrophic engine error — mark failed
            instance.fail();
            instance.getAuditLog().append(
                    AuditEntry.workflowEvent(AuditEntry.AuditEvent.WORKFLOW_FAILED,
                            Instant.now(), ex.getMessage()));
            globalEventBus.publish(
                    WorkflowEvent.workflowFailed(instance.getInstanceId(), ex.getMessage()));
            System.err.println("[Engine] Unexpected engine error: " + ex.getMessage());
        }

        return instance;
    }

    // ── Control ────────────────────────────────────────────────────────

    public boolean pause(String instanceId) {
        WorkflowInstance instance = getInstance(instanceId);
        boolean paused = instance.pause();
        if (paused) {
            instance.getAuditLog().append(
                    AuditEntry.workflowEvent(AuditEntry.AuditEvent.WORKFLOW_PAUSED,
                            Instant.now(), "Paused by external request"));
            globalEventBus.publish(WorkflowEvent.workflowPaused(instanceId));
        }
        return paused;
    }

    public boolean resume(String instanceId) {
        WorkflowInstance instance = getInstance(instanceId);
        boolean resumed = instance.resume();
        if (resumed) {
            instance.getAuditLog().append(
                    AuditEntry.workflowEvent(AuditEntry.AuditEvent.WORKFLOW_RESUMED,
                            Instant.now(), "Resumed by external request"));
            globalEventBus.publish(WorkflowEvent.workflowResumed(instanceId));
        }
        return resumed;
    }

    public boolean cancel(String instanceId) {
        WorkflowInstance instance = getInstance(instanceId);
        boolean cancelled = instance.cancel();
        if (cancelled) {
            instance.getAuditLog().append(
                    AuditEntry.workflowEvent(AuditEntry.AuditEvent.WORKFLOW_CANCELLED,
                            Instant.now(), "Cancelled by external request"));
            globalEventBus.publish(WorkflowEvent.workflowCancelled(instanceId));
        }
        return cancelled;
    }

    public WorkflowInstance getInstance(String instanceId) {
        WorkflowInstance inst = instances.get(instanceId);
        if (inst == null) throw new IllegalArgumentException(
                "No workflow instance found for id: " + instanceId);
        return inst;
    }

    // ── Private helpers ────────────────────────────────────────────────

    private void finalizeSuccess(WorkflowInstance instance) {
        instance.complete();
        instance.getAuditLog().append(
                AuditEntry.workflowEvent(AuditEntry.AuditEvent.WORKFLOW_COMPLETED,
                        Instant.now(), "All steps completed successfully"));
        globalEventBus.publish(WorkflowEvent.workflowCompleted(instance.getInstanceId()));
        System.out.printf("[Engine] Workflow '%s' COMPLETED%n", instance.getInstanceId());
    }

    private void finalizeFailure(WorkflowInstance instance, StepResult result) {
        instance.fail();
        String reason = result.error().map(Throwable::getMessage).orElse("Unknown failure");
        instance.getAuditLog().append(
                AuditEntry.workflowEvent(AuditEntry.AuditEvent.WORKFLOW_FAILED,
                        Instant.now(), reason));
        globalEventBus.publish(
                WorkflowEvent.workflowFailed(instance.getInstanceId(), reason));
        System.out.printf("[Engine] Workflow '%s' FAILED: %s%n",
                instance.getInstanceId(), reason);
    }

    private void finalizeCancelled(WorkflowInstance instance) {
        instance.getAuditLog().append(
                AuditEntry.workflowEvent(AuditEntry.AuditEvent.WORKFLOW_CANCELLED,
                        Instant.now(), "Cancelled during execution"));
        globalEventBus.publish(WorkflowEvent.workflowCancelled(instance.getInstanceId()));
        System.out.printf("[Engine] Workflow '%s' CANCELLED%n", instance.getInstanceId());
    }
}
```

---

### 6.23 End-to-End Example — Employee Onboarding Workflow

This example models a realistic HR onboarding workflow:

1. **Provision user account** (automated)
2. **Send welcome email** (automated)
3. **IT equipment setup** — three tasks run in parallel: laptop, badge, software
4. **Conditional**: if the employee is a manager, enqueue manager training; otherwise standard training
5. **Manager approval of onboarding completion** (manual approval, 30-second timeout for demo)
6. **Loop**: check background-check status until cleared (up to 5 iterations)

```java
package com.workflow.engine.example;

import com.workflow.engine.builder.WorkflowBuilder;
import com.workflow.engine.core.*;
import com.workflow.engine.error.ErrorPolicy;
import com.workflow.engine.event.WorkflowEvent;
import com.workflow.engine.steps.*;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

public final class EmployeeOnboardingExample {

    public static void main(String[] args) throws Exception {

        // ── Shared infrastructure ──────────────────────────────────────
        ExecutorService executor = Executors.newFixedThreadPool(4);
        WorkflowEngine  engine   = new WorkflowEngine();

        // Subscribe a console logger to all workflow events
        engine.subscribe(event -> System.out.printf(
                "  [EVENT] %s%n", describeEvent(event)));

        // ── Manual approval step (created outside builder so we can signal it) ──
        ManualApprovalStep approvalStep = new ManualApprovalStep(
                "approve-onboarding", "Manager Approval", "Manager must approve completion", 30_000);

        // Background thread simulates the manager clicking "Approve" after 2 seconds
        Thread approvalThread = Thread.ofVirtual().start(() -> {
            try {
                Thread.sleep(2_000);
                System.out.println("  [MANAGER] Approving onboarding...");
                approvalStep.signal(true, "manager@corp.com", "Looks good!");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        // ── Background check counter (simulates external polling) ──────
        AtomicInteger bgCheckAttempts = new AtomicInteger(0);

        // ── Workflow definition ────────────────────────────────────────
        WorkflowDefinition onboarding = WorkflowBuilder
                .create("employee-onboarding", "Employee Onboarding")
                .description("Full onboarding workflow for new hires")
                .version(2)

                // Step 1: Provision user account
                .automated("provision-account", "Provision User Account",
                        ErrorPolicy.RETRY_3_THEN_FAIL,
                        ctx -> {
                            String empId = ctx.require("employeeId", String.class);
                            System.out.println("  [Step] Provisioning account for " + empId);
                            simulateWork(300);
                            ctx.put("accountUsername", empId.toLowerCase() + "@corp.com");
                            return "account-" + empId;
                        })

                // Step 2: Send welcome email
                .automated("send-welcome-email", "Send Welcome Email",
                        ctx -> {
                            String username = ctx.require("accountUsername", String.class);
                            System.out.println("  [Step] Sending welcome email to " + username);
                            simulateWork(100);
                            ctx.put("welcomeEmailSent", true);
                            return "email-sent";
                        })

                // Step 3: IT equipment setup (3 tasks in parallel)
                .parallel(ParallelStep.builder("it-setup", "IT Equipment Setup", executor)
                        .description("Provision laptop, badge, and software in parallel")
                        .failFast(false) // collect all results even if one fails
                        .step(new AutomatedStep("provision-laptop", "Provision Laptop",
                                ctx -> {
                                    System.out.println("  [Parallel] Provisioning laptop...");
                                    simulateWork(500);
                                    ctx.put("laptopReady", true);
                                    return "laptop-ready";
                                }))
                        .step(new AutomatedStep("provision-badge", "Issue Security Badge",
                                ctx -> {
                                    System.out.println("  [Parallel] Issuing security badge...");
                                    simulateWork(300);
                                    ctx.put("badgeReady", true);
                                    return "badge-issued";
                                }))
                        .step(new AutomatedStep("install-software", "Install Software Suite",
                                "", ErrorPolicy.RETRY_3_THEN_SKIP,
                                ctx -> {
                                    System.out.println("  [Parallel] Installing software...");
                                    simulateWork(700);
                                    ctx.put("softwareReady", true);
                                    return "software-installed";
                                }))
                        .build())

                // Step 4: Conditional — manager vs. IC training path
                .condition("training-path", "Assign Training Path",
                        ctx -> Boolean.TRUE.equals(ctx.get("isManager", Boolean.class).orElse(false)),
                        // Then (manager path)
                        SequentialStep.builder("manager-training", "Manager Training Track")
                                .step(new AutomatedStep("leadership-101", "Leadership 101",
                                        ctx -> {
                                            System.out.println("  [Cond/Then] Enrolling in Leadership 101");
                                            simulateWork(200);
                                            ctx.put("trainingTrack", "manager");
                                            return "enrolled";
                                        }))
                                .step(new AutomatedStep("budget-training", "Budget Management",
                                        ctx -> {
                                            System.out.println("  [Cond/Then] Enrolling in Budget Management");
                                            simulateWork(150);
                                            return "enrolled";
                                        }))
                                .build(),
                        // Else (IC path)
                        new AutomatedStep("ic-training", "IC Onboarding Training",
                                ctx -> {
                                    System.out.println("  [Cond/Else] Enrolling in IC Onboarding");
                                    simulateWork(200);
                                    ctx.put("trainingTrack", "individual-contributor");
                                    return "enrolled";
                                }))

                // Step 5: Manual approval
                .manualApproval(approvalStep)

                // Step 6: Loop — poll background check until cleared (max 5 iterations)
                .loop("background-check-poll", "Background Check Polling", 5,
                        ctx -> {
                            boolean cleared = ctx.get("bgCheckCleared", Boolean.class).orElse(false);
                            return !cleared;
                        },
                        new AutomatedStep("check-bg-status", "Check Background Status",
                                ctx -> {
                                    int attempt = bgCheckAttempts.incrementAndGet();
                                    System.out.println("  [Loop] Background check poll attempt " + attempt);
                                    simulateWork(400);
                                    if (attempt >= 3) {
                                        // Cleared on 3rd attempt
                                        ctx.put("bgCheckCleared", true);
                                        System.out.println("  [Loop] Background check CLEARED");
                                    }
                                    return "attempt-" + attempt;
                                }))

                .build();

        // ── Initial context ────────────────────────────────────────────
        WorkflowContext ctx = new WorkflowContext("onboarding-" + System.currentTimeMillis());
        ctx.put("employeeId", "EMP-9042");
        ctx.put("isManager",  false);  // change to true to exercise manager branch

        // ── Execute ────────────────────────────────────────────────────
        System.out.println("=".repeat(70));
        System.out.println("  STARTING: Employee Onboarding Workflow");
        System.out.println("=".repeat(70));

        WorkflowInstance instance = engine.execute(onboarding, ctx);

        approvalThread.join();

        // ── Results ────────────────────────────────────────────────────
        System.out.println();
        System.out.println("=".repeat(70));
        System.out.printf("  RESULT: %s%n", instance.getStatus());
        System.out.println("=".repeat(70));
        System.out.println("  Final context values:");
        instance.getContext().snapshot().forEach((k, v) ->
                System.out.printf("    %-30s = %s%n", k, v));
        System.out.println();
        System.out.println("  Audit Trail:");
        instance.getAuditLog().print();

        executor.shutdown();
    }

    // ── Helpers ────────────────────────────────────────────────────────

    private static void simulateWork(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }

    private static String describeEvent(WorkflowEvent event) {
        return switch (event) {
            case WorkflowEvent.WorkflowStarted   e -> "WORKFLOW_STARTED   instance=" + e.instanceId();
            case WorkflowEvent.WorkflowCompleted  e -> "WORKFLOW_COMPLETED instance=" + e.instanceId();
            case WorkflowEvent.WorkflowFailed     e -> "WORKFLOW_FAILED    instance=" + e.instanceId() + " reason=" + e.reason();
            case WorkflowEvent.WorkflowPaused     e -> "WORKFLOW_PAUSED    instance=" + e.instanceId();
            case WorkflowEvent.WorkflowResumed    e -> "WORKFLOW_RESUMED   instance=" + e.instanceId();
            case WorkflowEvent.WorkflowCancelled  e -> "WORKFLOW_CANCELLED instance=" + e.instanceId();
            case WorkflowEvent.StepStarted        e -> "STEP_STARTED       step=" + e.stepName();
            case WorkflowEvent.StepCompleted      e -> "STEP_COMPLETED     step=" + e.stepName();
            case WorkflowEvent.StepFailed         e -> "STEP_FAILED        step=" + e.stepName();
            case WorkflowEvent.StepSkipped        e -> "STEP_SKIPPED       step=" + e.stepName();
        };
    }
}
```

---

### 6.24 StepCommand Interface (Command Pattern Formalization)

```java
package com.workflow.engine.core;

/**
 * Formalizes the Command pattern for step execution.
 *
 * AutomatedStep wraps a StepCommand (or a Function<WorkflowContext, Object>
 * which is functionally equivalent). The separation allows richer commands
 * that carry metadata (e.g., timeouts, idempotency keys) beyond a plain lambda.
 */
@FunctionalInterface
public interface StepCommand {

    /**
     * Execute the command.
     *
     * @param context the workflow's shared context
     * @return an arbitrary output value stored in the step result
     * @throws Exception any exception signals step failure
     */
    Object execute(WorkflowContext context) throws Exception;

    /**
     * Optional name for logging and audit. Default: "anonymous-command".
     */
    default String commandName() { return "anonymous-command"; }
}
```

---

## 7. Design Patterns Used

### 7.1 Composite Pattern
**Where**: `Step` hierarchy — `SequentialStep`, `ParallelStep`, `ConditionalStep`, `LoopStep` all implement `Step` and contain child `Step` objects. The engine's root call is `definition.getRootStep().execute(...)` — it has no knowledge of the tree's depth or composition.

**Benefit**: New composite step types (e.g., `RaceStep` — first child to complete wins) require zero changes to the engine or existing step types.

### 7.2 State Pattern
**Where**: `StepState` (sealed interface with `Pending`, `Running`, `Completed`, `Failed`, `Skipped`, `Paused` records). `WorkflowStatus` enum governs the instance-level state machine. `AbstractStep.transitionTo()` drives state changes.

**Benefit**: Exhaustive `switch` expressions in Java 17 catch missing state cases at compile time. State-specific behavior (e.g., `isTerminal()`) is co-located with the state itself.

### 7.3 Command Pattern
**Where**: `StepCommand` interface and its use in `AutomatedStep`. The `Function<WorkflowContext, Object>` lambda passed to `AutomatedStep` is the ConcreteCommand. The step itself is the Invoker; the `WorkflowEngine` is the Client.

**Benefit**: Decouples "what to execute" from "when and how to execute it." Enables command queuing, logging, undo (if added), and idempotency key attachment without modifying callers.

### 7.4 Chain of Responsibility
**Where**: `ErrorHandler` → `RetryErrorHandler` → `SkipErrorHandler` → `FailWorkflowErrorHandler` assembled by `ErrorHandlerChain`.

**Benefit**: Error-handling policies are independently testable. New policies (e.g., `AlertErrorHandler` that sends a PagerDuty alert) insert cleanly without touching existing handlers.

### 7.5 Builder Pattern
**Where**: `WorkflowBuilder` (fluent DSL for `WorkflowDefinition`), `SequentialStep.Builder`, `ParallelStep.Builder`.

**Benefit**: Enforces valid construction (e.g., at least one child step), separates "configuration time" from "execution time," and produces an immutable product.

### 7.6 Observer Pattern (Event Bus)
**Where**: `WorkflowEventBus` + `WorkflowEventListener`. Every lifecycle transition publishes a `WorkflowEvent` to the bus. Observers (audit writers, metrics collectors, notification senders) subscribe.

**Benefit**: The engine emits events without knowing anything about the consumers. A Prometheus metrics subscriber, a Slack notifier, and a database auditor all register independently.

### 7.7 Template Method Pattern
**Where**: `AbstractStep.execute()` is the template — it handles pause/cancel checks, audit logging, event publishing, and state transitions. Subclasses override `doExecute()` for their specific logic.

**Benefit**: Eliminates boilerplate duplication across all six step types while guaranteeing consistent lifecycle behavior regardless of step type.

---

## 8. Key Design Decisions and Trade-offs

### 8.1 Sealed Interfaces for StepState and WorkflowEvent

**Decision**: Use Java 17 sealed interfaces with records.

**Why**: Exhaustive `switch` expressions — if a new state/event is added, the compiler forces all switch sites to handle it. Records give value semantics (equality, `toString`, `hashCode`) for free.

**Trade-off**: Sealed types prevent external extension. External callers cannot add new `StepState` subtypes, which is intentional for this core abstraction but may be limiting for plugin scenarios.

---

### 8.2 Synchronous Execution Model

**Decision**: `WorkflowEngine.execute()` blocks the calling thread until the workflow completes.

**Why**: Simplicity, debuggability, and test friendliness. The caller always gets back a fully-resolved `WorkflowInstance`. Async execution is opt-in: callers wrap `engine.execute(...)` in a `CompletableFuture.supplyAsync(...)` themselves.

**Trade-off**: Long-running workflows tie up a thread. In high-concurrency scenarios, callers should run workflows on a virtual thread (`Thread.ofVirtual()`) — Java 21 makes this cheap. Alternatively, add an `executeAsync()` overload.

---

### 8.3 Shared WorkflowContext in Parallel Steps

**Decision**: Parallel branches share the same `WorkflowContext` backed by `ConcurrentHashMap`.

**Why**: Branches often need to read inputs written by preceding sequential steps. A forked copy per branch (via `context.fork()`) would prevent result aggregation.

**Trade-off**: Parallel steps that write to the SAME key create a data race on the semantic value (last writer wins). Documentation and convention (write distinct keys per branch) is the mitigation. For stronger guarantees, use `context.fork()` per branch and merge afterward.

---

### 8.4 ErrorHandlerChain — Separation of Retry, Skip, and Fail

**Decision**: Three distinct handler types vs. a single `switch` block.

**Why**: Each handler is independently testable. The chain can be customized per-engine or even per-step by passing a different chain. New handlers (e.g., `CompensationErrorHandler` for saga-style rollback) insert without modifying existing handlers.

**Trade-off**: Slightly more indirection than an inline `if/else`. The chain adds ~1 microsecond overhead per error handling invocation — acceptable for workflow-scale latencies.

---

### 8.5 Immutable WorkflowDefinition

**Decision**: `WorkflowDefinition` is fully immutable post-construction; the `Step` tree is composed of stateless step objects.

**Why**: Multiple concurrent `WorkflowInstance` objects can safely share one definition. No synchronization is needed on the definition itself. Versioning is trivially correct — version N and version N+1 can coexist.

**Trade-off**: Step objects cannot cache per-execution state. All mutable state flows through `WorkflowContext` and `WorkflowInstance`. This is the correct design but requires discipline from step authors.

---

### 8.6 ManualApprovalStep Uses CountDownLatch

**Decision**: `ManualApprovalStep` blocks its execution thread on a `CountDownLatch` until `signal()` is called from an external thread.

**Why**: Simple, correct, and requires no external coordination infrastructure. The latch is created fresh for each step execution.

**Trade-off**: Ties up a thread for potentially long durations. Mitigate by running the workflow on a virtual thread. For truly long-lived approvals (days), use an external persistence mechanism (database row) and resume the workflow via a separate HTTP callback.

---

### 8.7 Loop Safety via maxIterations

**Decision**: `LoopStep` accepts an explicit `maxIterations` cap (default 1000).

**Why**: Prevents infinite loops caused by a condition that never becomes false due to a bug. The engine fails the step (via error handler chain) when the cap is hit rather than hanging forever.

**Trade-off**: Legitimate long-running loops (e.g., polling for days) need a high cap. The cap is configurable, so this is a documentation concern rather than a hard limit.

---

## 9. Extension Points

### 9.1 New Step Types
Implement `AbstractStep` or directly `Step`. Examples:
- `TimeoutStep` — wraps another step with a deadline.
- `RaceStep` — like `ParallelStep` but returns as soon as the first child completes.
- `SubWorkflowStep` — embeds one `WorkflowDefinition` inside another.
- `HttpCallStep` — makes an HTTP request and writes the response to context.

```java
// Example: TimeoutStep wrapping any other Step
public final class TimeoutStep extends AbstractStep {
    private final Step    delegate;
    private final long    timeoutMs;
    private final ExecutorService executor;

    public TimeoutStep(String id, String name, Step delegate,
                       long timeoutMs, ExecutorService executor) {
        super(id, name, "Timeout wrapper: " + name, ErrorPolicy.FAIL_FAST);
        this.delegate  = delegate;
        this.timeoutMs = timeoutMs;
        this.executor  = executor;
    }

    @Override
    protected StepResult doExecute(WorkflowContext context, WorkflowInstance instance) {
        Instant start = Instant.now();
        var future = CompletableFuture.supplyAsync(
                () -> delegate.execute(context, instance), executor);
        try {
            return future.get(timeoutMs, java.util.concurrent.TimeUnit.MILLISECONDS);
        } catch (java.util.concurrent.TimeoutException e) {
            future.cancel(true);
            return StepResult.failure(getId(), start, Instant.now(),
                    new RuntimeException("Step '" + getName() + "' timed out after " + timeoutMs + "ms"));
        } catch (Exception e) {
            return StepResult.failure(getId(), start, Instant.now(), e);
        }
    }
}
```

### 9.2 New Error Handlers
Implement `ErrorHandler` and insert into `ErrorHandlerChain`:
- `AlertErrorHandler` — sends PagerDuty/Slack alert before delegating.
- `CompensationErrorHandler` — invokes a rollback step for saga patterns.
- `DeadLetterErrorHandler` — publishes the failed step context to a queue.

### 9.3 Persistent Workflow State
Implement a `WorkflowInstanceRepository` and call it after each step completion:

```java
public interface WorkflowInstanceRepository {
    void save(WorkflowInstance instance);
    Optional<WorkflowInstance> findById(String instanceId);
    List<WorkflowInstance> findByStatus(WorkflowStatus status);
}
```

The engine would load a suspended instance from the repository when resuming after a JVM restart.

### 9.4 Custom Event Listeners
`WorkflowEventListener` is a functional interface. Any lambda or class implementing `onEvent(WorkflowEvent)` can subscribe:

```java
// Metrics listener
engine.subscribe(event -> {
    if (event instanceof WorkflowEvent.StepCompleted sc) {
        metricsRegistry.counter("workflow.step.completed",
                "step", sc.stepName()).increment();
    }
});

// Slack notifier
engine.subscribe(event -> {
    if (event instanceof WorkflowEvent.WorkflowFailed wf) {
        slackClient.sendAlert("#alerts", "Workflow failed: " + wf.instanceId()
                + " — " + wf.reason());
    }
});
```

### 9.5 Distributed Locking for Pause/Resume
Replace `CountDownLatch` in `WorkflowInstance` with a distributed lock (Redis `SETNX` or ZooKeeper ephemeral node) to support pause/resume across JVM restarts or multiple nodes.

### 9.6 Workflow Scheduling
Wrap `WorkflowEngine.execute()` with a scheduler:

```java
public final class ScheduledWorkflowEngine {
    private final WorkflowEngine       engine;
    private final ScheduledExecutorService scheduler;

    public void schedule(WorkflowDefinition def, WorkflowContext ctx,
                         long delayMs, TimeUnit unit) {
        scheduler.schedule(() -> engine.execute(def, ctx), delayMs, unit);
    }

    public void scheduleRecurring(WorkflowDefinition def,
                                  java.util.function.Supplier<WorkflowContext> ctxFactory,
                                  long periodMs, TimeUnit unit) {
        scheduler.scheduleAtFixedRate(
                () -> engine.execute(def, ctxFactory.get()), 0, periodMs, unit);
    }
}
```

---

## 10. Production Considerations

### 10.1 Thread Pool Sizing
`ParallelStep` accepts an injected `ExecutorService`. In production:
- Use a **bounded** thread pool to avoid resource exhaustion when many parallel workflows execute simultaneously.
- Separate pools for I/O-heavy steps vs. CPU-heavy steps.
- Java 21 virtual threads (`Executors.newVirtualThreadPerTaskExecutor()`) are ideal for I/O-bound parallel steps.

```java
// Production executor configuration
ExecutorService workflowExecutor = new ThreadPoolExecutor(
    4,                              // corePoolSize
    Runtime.getRuntime().availableProcessors() * 2,  // maxPoolSize
    60L, TimeUnit.SECONDS,          // keepAlive
    new LinkedBlockingQueue<>(1000), // bounded queue
    new ThreadFactory() {
        private final AtomicInteger count = new AtomicInteger();
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "workflow-worker-" + count.incrementAndGet());
            t.setDaemon(true);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // back-pressure: caller executes if queue full
);
```

### 10.2 Workflow Persistence and Durability
For long-running workflows (hours/days):
- Serialize `WorkflowInstance` state to a database after each step completion.
- Use an event-sourced approach: replay the `AuditLog` to reconstruct state on recovery.
- Idempotency keys on `AutomatedStep` prevent double-execution during replay.

### 10.3 Observability
- **Metrics**: emit `step.duration`, `step.failure.count`, `workflow.active.count` via Micrometer.
- **Tracing**: propagate a trace ID through `WorkflowContext` and attach it to each span.
- **Structured logging**: `AuditLog.getEntries()` serializes to JSON for log aggregation (ELK, Splunk).

### 10.4 Error Budget and Alerting
- Alert on `step.failure.rate > 1%` or `workflow.failed.count > threshold`.
- Dead-letter failed workflows to a queue for manual inspection.
- SLA tracking: if `workflow.duration > p99_threshold`, raise an alert.

### 10.5 Security
- **Authorization**: `ManualApprovalStep.signal()` should verify the actor's permission before accepting the signal. Integrate with an `AuthorizationService` that checks ACLs.
- **Context sanitization**: do not log sensitive values (passwords, PII) from `WorkflowContext`. Use a masking filter on `AuditEntry.contextSnapshot`.
- **Input validation**: validate context values at `WorkflowEngine.execute()` entry using a `WorkflowValidator` that runs against the definition's declared input schema.

### 10.6 Testing Strategy

```java
// Unit test example — AutomatedStep with retry
@Test
void automatedStepShouldRetryAndSucceed() {
    AtomicInteger attempts = new AtomicInteger(0);
    AutomatedStep step = new AutomatedStep("s1", "Flaky Step",
            "", ErrorPolicy.RETRY_3_THEN_FAIL,
            ctx -> {
                if (attempts.incrementAndGet() < 3) {
                    throw new RuntimeException("Transient failure");
                }
                return "success";
            });

    WorkflowContext  ctx      = new WorkflowContext("test-instance");
    WorkflowEventBus bus      = new WorkflowEventBus();
    WorkflowInstance instance = new WorkflowInstance("def-1", 1, ctx, bus);
    instance.start();

    StepResult result = step.execute(ctx, instance);

    assertThat(result.isSuccess()).isTrue();
    assertThat(attempts.get()).isEqualTo(3);
}

// Integration test — full parallel workflow
@Test
void parallelStepShouldExecuteAllChildrenConcurrently() throws Exception {
    ExecutorService executor = Executors.newFixedThreadPool(3);
    List<String>    order    = new CopyOnWriteArrayList<>();

    ParallelStep parallel = ParallelStep
            .builder("par", "Parallel", executor)
            .step(new AutomatedStep("a", "A", ctx -> { order.add("A"); return null; }))
            .step(new AutomatedStep("b", "B", ctx -> { order.add("B"); return null; }))
            .step(new AutomatedStep("c", "C", ctx -> { order.add("C"); return null; }))
            .build();

    WorkflowDefinition def = WorkflowBuilder.create("test", "Test")
            .parallel(parallel).build();

    WorkflowEngine engine = new WorkflowEngine();
    WorkflowInstance result = engine.execute(def, new WorkflowContext("t1"));

    assertThat(result.getStatus()).isEqualTo(WorkflowStatus.COMPLETED);
    assertThat(order).containsExactlyInAnyOrder("A", "B", "C");
    executor.shutdown();
}
```

### 10.7 Workflow Versioning Strategy
- Store `definitionVersion` in `WorkflowInstance` at creation time.
- Running instances (in-flight) always execute against the version they started on.
- New instances pick up the latest version automatically via `WorkflowBuilder.version(N)`.
- Maintain a `WorkflowDefinitionRegistry` (Map<String, Map<Integer, WorkflowDefinition>>) for version lookup during recovery.

```java
public final class WorkflowDefinitionRegistry {
    private final ConcurrentHashMap<String, ConcurrentHashMap<Integer, WorkflowDefinition>>
            store = new ConcurrentHashMap<>();

    public void register(WorkflowDefinition def) {
        store.computeIfAbsent(def.getId(), k -> new ConcurrentHashMap<>())
             .put(def.getVersion(), def);
    }

    public WorkflowDefinition getLatest(String defId) {
        var versions = store.get(defId);
        if (versions == null) throw new IllegalArgumentException("Unknown definition: " + defId);
        return versions.entrySet().stream()
                .max(Map.Entry.comparingByKey())
                .map(Map.Entry::getValue)
                .orElseThrow();
    }

    public WorkflowDefinition getVersion(String defId, int version) {
        var versions = store.getOrDefault(defId, new ConcurrentHashMap<>());
        var def = versions.get(version);
        if (def == null) throw new IllegalArgumentException(
                "Definition " + defId + " v" + version + " not found");
        return def;
    }
}
```

### 10.8 Graceful Shutdown
```java
// Register a JVM shutdown hook
Runtime.getRuntime().addShutdownHook(Thread.ofVirtual().unstarted(() -> {
    System.out.println("[Engine] Shutdown initiated — cancelling active workflows...");
    engine.getActiveInstanceIds().forEach(id -> {
        engine.cancel(id);
        System.out.println("[Engine] Cancelled workflow instance: " + id);
    });
    executor.shutdown();
    try {
        if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
            executor.shutdownNow();
        }
    } catch (InterruptedException e) {
        executor.shutdownNow();
        Thread.currentThread().interrupt();
    }
}));
```

---

## Summary

This case study demonstrates how combining six design patterns produces a workflow engine that is:

| Quality | How Achieved |
|---------|-------------|
| **Extensible** | Composite + open `Step` interface; new step types require no engine changes |
| **Testable** | Injected `ExecutorService`, `EventBus`, no static state; all collaborators mockable |
| **Observable** | Observer (EventBus) + append-only AuditLog give full execution visibility |
| **Resilient** | Chain of Responsibility isolates error-handling from business logic |
| **Type-safe** | Sealed interfaces + records + exhaustive switch for all states and events |
| **Thread-safe** | `ConcurrentHashMap`, `AtomicReference`, `CountDownLatch`, `CopyOnWriteArrayList` used deliberately |
| **Versionable** | Immutable `WorkflowDefinition` with explicit version field; instances track their version |

The architecture scales from simple three-step scripts to hundred-step enterprise processes without structural changes — just new leaf or composite `Step` implementations registered via the builder DSL.
