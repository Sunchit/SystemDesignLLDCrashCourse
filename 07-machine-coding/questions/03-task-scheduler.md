# Machine Coding: Task Scheduler

## Problem Statement

> You are asked to design and implement a **Task Scheduling System** — similar to a simplified version of Quartz Scheduler or Unix cron.
>
> The system must allow clients to schedule work units (tasks) that execute either **once at a specific future time** or **repeatedly on a cron schedule**. Tasks carry a priority so that when multiple tasks are due at the same instant the higher-priority one runs first. Tasks can **depend on other tasks** (Task B may not start until Task A completes successfully). If a task fails, it should be **retried** a configurable number of times with exponential backoff. Any task can be **cancelled** before it runs.
>
> Implement the core in Java. You do not need a persistent store; an in-memory solution is fine. Focus on clean abstractions, thread safety, and extensibility.

---

## Functional Requirements

| # | Requirement |
|---|-------------|
| FR-1 | **One-time scheduling** — execute a task once at a caller-specified `Instant`. |
| FR-2 | **Recurring scheduling** — execute a task repeatedly according to a cron expression (5-field: `minute hour dayOfMonth month dayOfWeek`). |
| FR-3 | **Priority-based execution** — when two or more tasks are ready at the same scheduled time, execute the one with the highest numeric priority first (higher number = higher priority). |
| FR-4 | **Task cancellation** — a caller can cancel a pending or recurring task by ID; running tasks are not interrupted but no further executions are scheduled. |
| FR-5 | **Task dependencies** — a task may declare one or more prerequisite task IDs; it must not start until all prerequisites have status `COMPLETED`. Circular dependencies must be detected at registration time and rejected. |
| FR-6 | **Retry on failure** — a task whose `Callable` throws an exception is retried up to `maxRetries` times; the delay between retries follows configurable exponential backoff (`initialDelay * 2^attempt`). After exhausting retries the task transitions to `FAILED`. |

---

## Non-Functional Requirements / Assumptions

- **In-memory only**: No persistence layer; all state lives in JVM memory.
- **Single JVM**: No distributed coordination.
- **Thread safety**: The scheduler will be called from multiple threads concurrently.
- **Cron granularity**: The cron parser supports minute-level granularity (no seconds field). The 5 fields are: `minute hour dayOfMonth month dayOfWeek`.
- **Cron wildcards**: Supports `*` (any), `,` (list), `-` (range), and `/` (step) in each field.
- **Dependency resolution**: Dependencies must form a DAG; cycles are rejected. A dependent task is checked for readiness each time a prerequisite completes.
- **Retry backoff**: `delay(attempt) = initialDelayMs * 2^attempt`, capped at a configurable maximum.
- **Observer notification**: Completion/failure events are broadcast to registered listeners asynchronously to avoid slowing the executor thread.
- **Clock abstraction**: The scheduler accepts a `Clock` so tests can inject a fixed clock.
- **Graceful shutdown**: `shutdown()` waits for running tasks to finish (up to a configurable timeout) and cancels all pending futures.

---

## Entity Identification (High-Level Design)

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Client Code                                │
└────────────────────────────┬────────────────────────────────────────┘
                             │ schedule() / cancel()
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        TaskScheduler                                │
│                                                                     │
│  ┌──────────────────┐   ┌────────────────────────┐                 │
│  │ TaskRegistry     │   │ DependencyGraph         │                 │
│  │ (ConcurrentMap)  │   │ (adjacency list + topo) │                 │
│  └──────────────────┘   └────────────────────────┘                 │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │             ScheduledExecutorService                         │   │
│  │   fires trigger at scheduled time → enqueues into           │   │
│  │             PriorityBlockingQueue<TaskWrapper>               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   TaskExecutor thread pool                   │   │
│  │  drains PriorityBlockingQueue → checks deps → runs task →   │   │
│  │  handles retry (RetryPolicy) → fires TaskEventListener      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘

Key entities
────────────
TaskStatus          enum: PENDING / RUNNING / COMPLETED / FAILED / CANCELLED
Task                id, name, priority, status, action (Callable<Void>),
                    retryCount, maxRetries, dependencies
RecurringTask       extends Task — adds cronExpression, nextExecutionTime()
RetryPolicy         interface — compute next retry delay (Strategy pattern)
ExponentialBackoff  implements RetryPolicy
DependencyGraph     DAG stored as Map<String,Set<String>>;
                    topological sort at registration time
TaskEventListener   interface — onComplete / onFailure (Observer pattern)
TaskScheduler       façade — schedule(), cancel(), addListener(), shutdown()
SimpleCronParser    parses 5-field cron; computes next ZonedDateTime after now
```

---

## Complete Java Implementation

### 1. `TaskStatus.java`

```java
package scheduler;

public enum TaskStatus {
    PENDING,
    RUNNING,
    COMPLETED,
    FAILED,
    CANCELLED
}
```

---

### 2. `RetryPolicy.java` — Strategy interface

```java
package scheduler;

/**
 * Strategy interface for computing retry delays.
 * Implementations decide whether a retry should be attempted and how long to wait.
 */
public interface RetryPolicy {

    /**
     * Returns the delay in milliseconds before the next attempt,
     * or -1 if no further retries should be made.
     *
     * @param attemptNumber 1-based attempt number (first retry = 1)
     * @param maxRetries    maximum number of retries configured on the task
     */
    long delayMillis(int attemptNumber, int maxRetries);
}
```

---

### 3. `ExponentialBackoffRetryPolicy.java`

```java
package scheduler;

/**
 * Exponential backoff: delay = initialDelayMs * 2^(attempt-1), capped at maxDelayMs.
 * Returns -1 when attemptNumber > maxRetries.
 */
public class ExponentialBackoffRetryPolicy implements RetryPolicy {

    private final long initialDelayMs;
    private final long maxDelayMs;

    public ExponentialBackoffRetryPolicy(long initialDelayMs, long maxDelayMs) {
        if (initialDelayMs <= 0) throw new IllegalArgumentException("initialDelayMs must be > 0");
        if (maxDelayMs < initialDelayMs) throw new IllegalArgumentException("maxDelayMs must be >= initialDelayMs");
        this.initialDelayMs = initialDelayMs;
        this.maxDelayMs = maxDelayMs;
    }

    @Override
    public long delayMillis(int attemptNumber, int maxRetries) {
        if (attemptNumber > maxRetries) return -1L;
        // shift left by (attemptNumber - 1) is equivalent to multiplying by 2^(attempt-1)
        long shift = (long) Math.min(attemptNumber - 1, 62); // prevent overflow
        long delay = initialDelayMs << shift;
        return Math.min(delay, maxDelayMs);
    }
}
```

---

### 4. `TaskEventListener.java` — Observer interface

```java
package scheduler;

/**
 * Observer interface for task lifecycle events.
 * Implementations are called asynchronously after task completion or failure.
 */
public interface TaskEventListener {

    /** Called when a task finishes successfully. */
    void onComplete(Task task);

    /**
     * Called when a task fails after exhausting all retries.
     *
     * @param task  the failed task
     * @param cause the last exception thrown by the task action
     */
    void onFailure(Task task, Throwable cause);
}
```

---

### 5. `Task.java` — Core domain object (Builder pattern)

```java
package scheduler;

import java.util.Collections;
import java.util.HashSet;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.Callable;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;

/**
 * Represents a unit of work to be executed by the scheduler.
 *
 * <p>Use {@link Task.Builder} to construct instances.
 *
 * <p>Thread safety: status transitions are guarded by an AtomicReference.
 * retryCount is updated only within the executor thread that owns the task
 * execution, so an AtomicInteger is sufficient.
 */
public class Task implements Comparable<Task> {

    private final String id;
    private final String name;
    private final int priority;                        // higher = runs first
    private final Callable<Void> action;
    private final int maxRetries;
    private final RetryPolicy retryPolicy;
    private final Set<String> dependencies;            // prerequisite task IDs

    private final AtomicReference<TaskStatus> status;
    private final AtomicInteger retryCount;

    protected Task(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.priority = builder.priority;
        this.action = builder.action;
        this.maxRetries = builder.maxRetries;
        this.retryPolicy = builder.retryPolicy;
        this.dependencies = Collections.unmodifiableSet(new HashSet<>(builder.dependencies));
        this.status = new AtomicReference<>(TaskStatus.PENDING);
        this.retryCount = new AtomicInteger(0);
    }

    // ── accessors ──────────────────────────────────────────────────────────

    public String getId() { return id; }
    public String getName() { return name; }
    public int getPriority() { return priority; }
    public Callable<Void> getAction() { return action; }
    public int getMaxRetries() { return maxRetries; }
    public RetryPolicy getRetryPolicy() { return retryPolicy; }
    public Set<String> getDependencies() { return dependencies; }
    public TaskStatus getStatus() { return status.get(); }
    public int getRetryCount() { return retryCount.get(); }

    // ── status transitions ─────────────────────────────────────────────────

    /** Returns true if the CAS succeeded (i.e. the transition was valid). */
    public boolean transitionTo(TaskStatus expected, TaskStatus next) {
        return status.compareAndSet(expected, next);
    }

    /** Unconditional set — used only by the executor after acquiring ownership. */
    public void setStatus(TaskStatus next) { status.set(next); }

    public int incrementRetryCount() { return retryCount.incrementAndGet(); }

    // ── Comparable: higher priority tasks are "less than" (PriorityQueue is min-heap) ──

    @Override
    public int compareTo(Task other) {
        // Negate so that higher priority = smaller order (PriorityBlockingQueue is min-heap)
        return Integer.compare(other.priority, this.priority);
    }

    @Override
    public String toString() {
        return String.format("Task{id='%s', name='%s', priority=%d, status=%s, retries=%d/%d}",
                id, name, priority, status.get(), retryCount.get(), maxRetries);
    }

    // ── Builder ────────────────────────────────────────────────────────────

    public static Builder builder(String name, Callable<Void> action) {
        return new Builder(name, action);
    }

    public static final class Builder {
        private String id = UUID.randomUUID().toString();
        private final String name;
        private int priority = 0;
        private final Callable<Void> action;
        private int maxRetries = 0;
        private RetryPolicy retryPolicy = new ExponentialBackoffRetryPolicy(1000L, 60_000L);
        private final Set<String> dependencies = new HashSet<>();

        private Builder(String name, Callable<Void> action) {
            if (name == null || name.isBlank()) throw new IllegalArgumentException("name must not be blank");
            if (action == null) throw new IllegalArgumentException("action must not be null");
            this.name = name;
            this.action = action;
        }

        public Builder id(String id) { this.id = id; return this; }
        public Builder priority(int priority) { this.priority = priority; return this; }
        public Builder maxRetries(int maxRetries) { this.maxRetries = maxRetries; return this; }
        public Builder retryPolicy(RetryPolicy retryPolicy) { this.retryPolicy = retryPolicy; return this; }
        public Builder dependsOn(String taskId) { this.dependencies.add(taskId); return this; }
        public Builder dependsOn(String... taskIds) {
            for (String id : taskIds) this.dependencies.add(id);
            return this;
        }

        public Task build() { return new Task(this); }
    }
}
```

---

### 6. `SimpleCronParser.java`

```java
package scheduler;

import java.time.ZonedDateTime;
import java.time.temporal.ChronoField;
import java.util.ArrayList;
import java.util.List;
import java.util.TreeSet;

/**
 * Parses a 5-field cron expression and computes the next execution time after
 * a given base time.
 *
 * <pre>
 * ┌───────────── minute (0-59)
 * │ ┌───────────── hour (0-23)
 * │ │ ┌───────────── day of month (1-31)
 * │ │ │ ┌───────────── month (1-12)
 * │ │ │ │ ┌───────────── day of week (0-6, Sunday=0)
 * │ │ │ │ │
 * * * * * *
 * </pre>
 *
 * Supported meta-characters: {@code *}, {@code ,}, {@code -}, {@code /}.
 *
 * <p>Algorithm: advance from {@code base + 1 minute} in minute increments,
 * skipping whole hours/days/months when possible to avoid O(525600) scans
 * in the worst case. Capped at 4 years of lookahead to avoid infinite loops
 * on unsatisfiable expressions.
 */
public class SimpleCronParser {

    private final TreeSet<Integer> minutes;
    private final TreeSet<Integer> hours;
    private final TreeSet<Integer> daysOfMonth;
    private final TreeSet<Integer> months;
    private final TreeSet<Integer> daysOfWeek;

    /** Sunday=0 … Saturday=6 (matches classic cron). Java DayOfWeek is 1=Monday … 7=Sunday. */
    private static final int[] JAVA_DOW_TO_CRON = {-1, 1, 2, 3, 4, 5, 6, 0}; // index=java dow

    public SimpleCronParser(String expression) {
        String[] fields = expression.trim().split("\\s+");
        if (fields.length != 5) {
            throw new IllegalArgumentException(
                    "Cron expression must have exactly 5 fields: '" + expression + "'");
        }
        minutes     = parseField(fields[0], 0, 59);
        hours       = parseField(fields[1], 0, 23);
        daysOfMonth = parseField(fields[2], 1, 31);
        months      = parseField(fields[3], 1, 12);
        daysOfWeek  = parseField(fields[4], 0,  6);
    }

    /**
     * Returns the next scheduled instant that is strictly after {@code base}.
     *
     * @throws IllegalStateException if no valid time is found within 4 years
     */
    public ZonedDateTime nextExecution(ZonedDateTime base) {
        // Start from the next minute
        ZonedDateTime candidate = base.plusMinutes(1)
                .withSecond(0)
                .withNano(0);

        ZonedDateTime limit = base.plusYears(4);

        while (candidate.isBefore(limit)) {
            // Check month
            int candidateMonth = candidate.getMonthValue(); // 1-12
            if (!months.contains(candidateMonth)) {
                Integer nextMonth = months.ceiling(candidateMonth);
                if (nextMonth == null) {
                    // wrap to next year
                    candidate = candidate
                            .withMonth(months.first())
                            .withDayOfMonth(1)
                            .withHour(0)
                            .withMinute(0)
                            .plusYears(1);
                } else {
                    candidate = candidate
                            .withMonth(nextMonth)
                            .withDayOfMonth(1)
                            .withHour(0)
                            .withMinute(0);
                }
                continue;
            }

            // Check day of month and day of week
            int candidateDay = candidate.getDayOfMonth();
            int candidateCronDow = JAVA_DOW_TO_CRON[candidate.getDayOfWeek().getValue()];
            if (!daysOfMonth.contains(candidateDay) || !daysOfWeek.contains(candidateCronDow)) {
                candidate = candidate
                        .plusDays(1)
                        .withHour(0)
                        .withMinute(0);
                continue;
            }

            // Check hour
            int candidateHour = candidate.getHour();
            if (!hours.contains(candidateHour)) {
                Integer nextHour = hours.ceiling(candidateHour);
                if (nextHour == null) {
                    candidate = candidate.plusDays(1).withHour(0).withMinute(0);
                } else {
                    candidate = candidate.withHour(nextHour).withMinute(0);
                }
                continue;
            }

            // Check minute
            int candidateMinute = candidate.getMinute();
            if (!minutes.contains(candidateMinute)) {
                Integer nextMinute = minutes.ceiling(candidateMinute);
                if (nextMinute == null) {
                    candidate = candidate.plusHours(1).withMinute(0);
                } else {
                    candidate = candidate.withMinute(nextMinute);
                }
                continue;
            }

            // All fields match
            return candidate;
        }

        throw new IllegalStateException(
                "No valid execution time found within 4 years for expression. " +
                "Check the cron expression for unsatisfiable constraints.");
    }

    // ── field parser ──────────────────────────────────────────────────────

    /**
     * Expands a single cron field token (possibly containing {@code *}, {@code ,},
     * {@code -}, {@code /}) into the full set of matching integer values.
     */
    static TreeSet<Integer> parseField(String field, int min, int max) {
        TreeSet<Integer> result = new TreeSet<>();
        for (String part : field.split(",")) {
            part = part.trim();
            if (part.contains("/")) {
                // step expression: range/step  or  */step
                String[] stepParts = part.split("/", 2);
                int step = Integer.parseInt(stepParts[1]);
                if (step <= 0) throw new IllegalArgumentException("Step must be > 0: " + part);
                int rangeStart = min;
                int rangeEnd = max;
                if (!stepParts[0].equals("*")) {
                    if (stepParts[0].contains("-")) {
                        String[] rangeParts = stepParts[0].split("-", 2);
                        rangeStart = Integer.parseInt(rangeParts[0]);
                        rangeEnd   = Integer.parseInt(rangeParts[1]);
                    } else {
                        rangeStart = Integer.parseInt(stepParts[0]);
                    }
                }
                for (int v = rangeStart; v <= rangeEnd; v += step) {
                    checkBounds(v, min, max, field);
                    result.add(v);
                }
            } else if (part.contains("-")) {
                // range expression: low-high
                String[] rangeParts = part.split("-", 2);
                int low  = Integer.parseInt(rangeParts[0]);
                int high = Integer.parseInt(rangeParts[1]);
                for (int v = low; v <= high; v++) {
                    checkBounds(v, min, max, field);
                    result.add(v);
                }
            } else if (part.equals("*")) {
                for (int v = min; v <= max; v++) result.add(v);
            } else {
                int v = Integer.parseInt(part);
                checkBounds(v, min, max, field);
                result.add(v);
            }
        }
        return result;
    }

    private static void checkBounds(int value, int min, int max, String field) {
        if (value < min || value > max) {
            throw new IllegalArgumentException(
                    String.format("Value %d out of range [%d, %d] in field: %s", value, min, max, field));
        }
    }
}
```

---

### 7. `RecurringTask.java`

```java
package scheduler;

import java.time.ZonedDateTime;
import java.util.concurrent.Callable;

/**
 * A task that repeats according to a cron expression.
 * After each execution the scheduler calls {@link #advanceNextExecution(ZonedDateTime)}
 * to compute the next fire time.
 */
public class RecurringTask extends Task {

    private final String cronExpression;
    private final SimpleCronParser cronParser;
    private volatile ZonedDateTime nextExecutionTime;

    private RecurringTask(RecurringBuilder builder) {
        super(builder);
        this.cronExpression = builder.cronExpression;
        this.cronParser = new SimpleCronParser(cronExpression);
        this.nextExecutionTime = cronParser.nextExecution(builder.referenceTime);
    }

    public String getCronExpression() { return cronExpression; }

    public ZonedDateTime getNextExecutionTime() { return nextExecutionTime; }

    /**
     * Advances the next execution time to the first slot after {@code completedAt}.
     * Called by the scheduler after each successful (or failed-and-retried) run.
     */
    public void advanceNextExecution(ZonedDateTime completedAt) {
        this.nextExecutionTime = cronParser.nextExecution(completedAt);
    }

    @Override
    public String toString() {
        return String.format(
                "RecurringTask{id='%s', name='%s', cron='%s', next=%s, status=%s}",
                getId(), getName(), cronExpression, nextExecutionTime, getStatus());
    }

    // ── Builder ────────────────────────────────────────────────────────────

    public static RecurringBuilder recurringBuilder(String name,
                                                     Callable<Void> action,
                                                     String cronExpression) {
        return new RecurringBuilder(name, action, cronExpression);
    }

    public static final class RecurringBuilder extends Task.Builder {
        private final String cronExpression;
        private ZonedDateTime referenceTime = ZonedDateTime.now();

        private RecurringBuilder(String name, Callable<Void> action, String cronExpression) {
            super(name, action);
            if (cronExpression == null || cronExpression.isBlank()) {
                throw new IllegalArgumentException("cronExpression must not be blank");
            }
            this.cronExpression = cronExpression;
        }

        /** Override the reference time used to compute the first execution slot. */
        public RecurringBuilder startingFrom(ZonedDateTime referenceTime) {
            this.referenceTime = referenceTime;
            return this;
        }

        @Override
        public RecurringTask build() {
            return new RecurringTask(this);
        }
    }
}
```

---

### 8. `DependencyGraph.java`

```java
package scheduler;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Maintains a DAG of task dependencies.
 *
 * <p>When a new task is registered, its declared dependency set is added as
 * directed edges (dep → task). The graph verifies that no cycle is introduced.
 *
 * <p>Thread safety: mutations are synchronised; reads of the adjacency maps
 * use ConcurrentHashMap so concurrent status checks are safe.
 */
public class DependencyGraph {

    // edges: taskId → set of tasks that depend on it (downstream)
    private final Map<String, Set<String>> downstream = new ConcurrentHashMap<>();

    // edges: taskId → set of task IDs it depends on (upstream)
    private final Map<String, Set<String>> upstream = new ConcurrentHashMap<>();

    /**
     * Registers {@code task} and its declared dependencies.
     *
     * @throws IllegalArgumentException if a prerequisite task ID does not exist in the graph,
     *                                  or if adding this task would create a cycle.
     */
    public synchronized void register(Task task, Map<String, Task> registry) {
        String id = task.getId();
        upstream.put(id, new HashSet<>(task.getDependencies()));
        downstream.putIfAbsent(id, new HashSet<>());

        for (String depId : task.getDependencies()) {
            if (!registry.containsKey(depId)) {
                // Roll back and reject
                upstream.remove(id);
                throw new IllegalArgumentException(
                        "Unknown dependency '" + depId + "' for task '" + id + "'");
            }
            downstream.computeIfAbsent(depId, k -> new HashSet<>()).add(id);
        }

        // Cycle detection via DFS from the newly added node
        if (hasCycle()) {
            // Roll back
            for (String depId : task.getDependencies()) {
                Set<String> ds = downstream.get(depId);
                if (ds != null) ds.remove(id);
            }
            upstream.remove(id);
            throw new IllegalArgumentException(
                    "Adding task '" + id + "' would introduce a dependency cycle.");
        }
    }

    /**
     * Returns true if all declared upstream dependencies of {@code taskId} are COMPLETED.
     *
     * @param registry the live task registry to look up current statuses
     */
    public boolean allDependenciesMet(String taskId, Map<String, Task> registry) {
        Set<String> deps = upstream.getOrDefault(taskId, Collections.emptySet());
        for (String depId : deps) {
            Task dep = registry.get(depId);
            if (dep == null || dep.getStatus() != TaskStatus.COMPLETED) return false;
        }
        return true;
    }

    /**
     * Returns IDs of tasks that are waiting on {@code completedTaskId}.
     */
    public Set<String> getDownstream(String completedTaskId) {
        return Collections.unmodifiableSet(
                downstream.getOrDefault(completedTaskId, Collections.emptySet()));
    }

    /** Removes a task from the graph (e.g. after cancellation). */
    public synchronized void deregister(String taskId) {
        Set<String> deps = upstream.remove(taskId);
        downstream.remove(taskId);
        if (deps != null) {
            for (String depId : deps) {
                Set<String> ds = downstream.get(depId);
                if (ds != null) ds.remove(taskId);
            }
        }
    }

    // ── cycle detection ────────────────────────────────────────────────────

    /**
     * Kahn's algorithm (BFS-based topological sort).
     * Returns true if the current graph contains a cycle.
     */
    private boolean hasCycle() {
        // Compute in-degree for every node
        Map<String, Integer> inDegree = new HashMap<>();
        for (String node : upstream.keySet()) {
            inDegree.put(node, upstream.get(node).size());
        }

        Queue<String> queue = new ArrayDeque<>();
        for (Map.Entry<String, Integer> entry : inDegree.entrySet()) {
            if (entry.getValue() == 0) queue.add(entry.getKey());
        }

        int processed = 0;
        while (!queue.isEmpty()) {
            String node = queue.poll();
            processed++;
            for (String dependent : downstream.getOrDefault(node, Collections.emptySet())) {
                int deg = inDegree.merge(dependent, -1, Integer::sum);
                if (deg == 0) queue.add(dependent);
            }
        }
        return processed != upstream.size();
    }

    /**
     * Returns a valid topological ordering of all registered tasks,
     * or throws if a cycle exists (should not happen after registration guards).
     */
    public List<String> topologicalOrder() {
        Map<String, Integer> inDegree = new HashMap<>();
        for (String node : upstream.keySet()) {
            inDegree.put(node, upstream.get(node).size());
        }

        Queue<String> queue = new ArrayDeque<>();
        for (Map.Entry<String, Integer> entry : inDegree.entrySet()) {
            if (entry.getValue() == 0) queue.add(entry.getKey());
        }

        List<String> order = new ArrayList<>();
        while (!queue.isEmpty()) {
            String node = queue.poll();
            order.add(node);
            for (String dependent : downstream.getOrDefault(node, Collections.emptySet())) {
                int deg = inDegree.merge(dependent, -1, Integer::sum);
                if (deg == 0) queue.add(dependent);
            }
        }

        if (order.size() != upstream.size()) {
            throw new IllegalStateException("Cycle detected in dependency graph — this should have been caught at registration.");
        }
        return order;
    }
}
```

---

### 9. `TaskExecutor.java`

```java
package scheduler;

import java.time.ZonedDateTime;
import java.util.List;
import java.util.Map;
import java.util.concurrent.*;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * Drains a {@link PriorityBlockingQueue} of ready tasks and executes them.
 *
 * <p>Each task is executed on a dedicated thread from the worker pool. Retry
 * scheduling and dependency unblocking are handled here.
 */
public class TaskExecutor {

    private static final Logger log = Logger.getLogger(TaskExecutor.class.getName());

    private final ExecutorService workerPool;
    private final ScheduledExecutorService retryScheduler;
    private final PriorityBlockingQueue<Task> readyQueue;
    private final Map<String, Task> registry;
    private final DependencyGraph dependencyGraph;
    private final List<TaskEventListener> listeners;

    // Set to false on shutdown to stop the drain loop
    private volatile boolean running = true;
    private Thread drainThread;

    public TaskExecutor(ExecutorService workerPool,
                        ScheduledExecutorService retryScheduler,
                        PriorityBlockingQueue<Task> readyQueue,
                        Map<String, Task> registry,
                        DependencyGraph dependencyGraph,
                        List<TaskEventListener> listeners) {
        this.workerPool       = workerPool;
        this.retryScheduler   = retryScheduler;
        this.readyQueue       = readyQueue;
        this.registry         = registry;
        this.dependencyGraph  = dependencyGraph;
        this.listeners        = listeners;
    }

    /** Starts the drain loop on a dedicated thread. */
    public void start() {
        drainThread = new Thread(this::drainLoop, "task-executor-drain");
        drainThread.setDaemon(true);
        drainThread.start();
    }

    /** Signals the drain loop to stop. */
    public void stop() {
        running = false;
        if (drainThread != null) drainThread.interrupt();
    }

    // ── drain loop ─────────────────────────────────────────────────────────

    private void drainLoop() {
        while (running) {
            try {
                // Block until a task is available (poll with timeout to allow clean shutdown)
                Task task = readyQueue.poll(500, TimeUnit.MILLISECONDS);
                if (task == null) continue;

                TaskStatus status = task.getStatus();

                // Skip tasks that were cancelled while waiting in the queue
                if (status == TaskStatus.CANCELLED) continue;

                // Skip tasks whose dependencies are not yet met — re-queue after a short delay
                if (!dependencyGraph.allDependenciesMet(task.getId(), registry)) {
                    log.fine("Task " + task.getId() + " blocked by unmet dependencies; re-queuing.");
                    retryScheduler.schedule(
                            () -> readyQueue.offer(task),
                            500, TimeUnit.MILLISECONDS);
                    continue;
                }

                // Transition PENDING → RUNNING
                if (!task.transitionTo(TaskStatus.PENDING, TaskStatus.RUNNING)) {
                    // Task was cancelled or already running (shouldn't normally happen)
                    continue;
                }

                workerPool.submit(() -> executeTask(task));

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    // ── task execution ─────────────────────────────────────────────────────

    private void executeTask(Task task) {
        try {
            task.getAction().call();
            handleSuccess(task);
        } catch (Exception ex) {
            handleFailure(task, ex);
        }
    }

    private void handleSuccess(Task task) {
        if (task instanceof RecurringTask) {
            RecurringTask rt = (RecurringTask) task;
            rt.setStatus(TaskStatus.PENDING); // reset for next run
            rt.advanceNextExecution(ZonedDateTime.now());
            log.info("Recurring task '" + task.getName() + "' completed; next run at " + rt.getNextExecutionTime());
            // The main scheduler's trigger loop will re-schedule rt at nextExecutionTime.
        } else {
            task.setStatus(TaskStatus.COMPLETED);
            log.info("Task '" + task.getName() + "' completed successfully.");
        }

        // Notify listeners asynchronously
        List<TaskEventListener> snapshot = List.copyOf(listeners);
        workerPool.submit(() -> {
            for (TaskEventListener l : snapshot) {
                try { l.onComplete(task); } catch (Exception ex) {
                    log.log(Level.WARNING, "Listener threw during onComplete", ex);
                }
            }
        });

        // Unblock downstream tasks that were waiting on this task
        unblockDownstream(task.getId());
    }

    private void handleFailure(Task task, Exception cause) {
        int attempt = task.incrementRetryCount();
        long delay = task.getRetryPolicy().delayMillis(attempt, task.getMaxRetries());

        if (delay >= 0) {
            // There are retries remaining
            log.warning(String.format("Task '%s' failed (attempt %d/%d); retrying in %d ms. Error: %s",
                    task.getName(), attempt, task.getMaxRetries(), delay, cause.getMessage()));
            task.setStatus(TaskStatus.PENDING);
            retryScheduler.schedule(
                    () -> readyQueue.offer(task),
                    delay, TimeUnit.MILLISECONDS);
        } else {
            // Exhausted retries
            task.setStatus(TaskStatus.FAILED);
            log.severe(String.format("Task '%s' failed after %d retries. Marking FAILED.", task.getName(), attempt - 1));

            List<TaskEventListener> snapshot = List.copyOf(listeners);
            workerPool.submit(() -> {
                for (TaskEventListener l : snapshot) {
                    try { l.onFailure(task, cause); } catch (Exception ex) {
                        log.log(Level.WARNING, "Listener threw during onFailure", ex);
                    }
                }
            });
        }
    }

    /** Puts any downstream tasks back on the ready queue if all their deps are now met. */
    private void unblockDownstream(String completedTaskId) {
        for (String downstreamId : dependencyGraph.getDownstream(completedTaskId)) {
            Task downstream = registry.get(downstreamId);
            if (downstream != null
                    && downstream.getStatus() == TaskStatus.PENDING
                    && dependencyGraph.allDependenciesMet(downstreamId, registry)) {
                log.info("Dependency met; enqueuing task '" + downstream.getName() + "'.");
                readyQueue.offer(downstream);
            }
        }
    }
}
```

---

### 10. `TaskScheduler.java` — Main facade

```java
package scheduler;

import java.time.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * Central façade for the task scheduling system.
 *
 * <h3>Usage</h3>
 * <pre>{@code
 * TaskScheduler scheduler = TaskScheduler.builder().build();
 * scheduler.start();
 *
 * Task t1 = Task.builder("send-email", () -> { sendEmail(); return null; })
 *               .priority(5)
 *               .maxRetries(3)
 *               .build();
 * scheduler.scheduleAt(t1, Instant.now().plusSeconds(10));
 *
 * RecurringTask daily = RecurringTask.recurringBuilder("daily-report", () -> { generateReport(); return null; }, "0 8 * * *")
 *                                    .build();
 * scheduler.scheduleRecurring(daily);
 * }</pre>
 *
 * <h3>Thread model</h3>
 * <ul>
 *   <li>One {@link ScheduledExecutorService} triggers tasks at their scheduled time (low-frequency).</li>
 *   <li>One drain thread (inside {@link TaskExecutor}) moves tasks from the trigger to a
 *       {@link PriorityBlockingQueue} and dispatches to a worker pool.</li>
 *   <li>A fixed-size worker pool executes task actions.</li>
 *   <li>A small {@link ScheduledExecutorService} handles retry delays.</li>
 * </ul>
 */
public class TaskScheduler {

    private static final Logger log = Logger.getLogger(TaskScheduler.class.getName());

    // ── core infrastructure ────────────────────────────────────────────────

    private final ScheduledExecutorService triggerScheduler;
    private final ExecutorService workerPool;
    private final ScheduledExecutorService retryScheduler;
    private final PriorityBlockingQueue<Task> readyQueue;

    // ── state ──────────────────────────────────────────────────────────────

    /** All registered tasks, keyed by task ID. */
    private final Map<String, Task> registry = new ConcurrentHashMap<>();

    /** Scheduled futures keyed by task ID — used for cancellation. */
    private final Map<String, ScheduledFuture<?>> scheduledFutures = new ConcurrentHashMap<>();

    private final DependencyGraph dependencyGraph = new DependencyGraph();
    private final List<TaskEventListener> listeners = new CopyOnWriteArrayList<>();
    private final TaskExecutor taskExecutor;

    private final Clock clock;
    private volatile boolean started = false;
    private volatile boolean shutdown = false;

    // ── constructor ────────────────────────────────────────────────────────

    private TaskScheduler(Builder builder) {
        this.triggerScheduler = Executors.newScheduledThreadPool(builder.triggerThreads,
                r -> { Thread t = new Thread(r, "task-trigger"); t.setDaemon(true); return t; });
        this.workerPool = Executors.newFixedThreadPool(builder.workerThreads,
                r -> { Thread t = new Thread(r, "task-worker"); t.setDaemon(true); return t; });
        this.retryScheduler = Executors.newScheduledThreadPool(2,
                r -> { Thread t = new Thread(r, "task-retry"); t.setDaemon(true); return t; });
        this.readyQueue = new PriorityBlockingQueue<>();
        this.clock = builder.clock;

        this.taskExecutor = new TaskExecutor(
                workerPool, retryScheduler, readyQueue, registry, dependencyGraph, listeners);
    }

    // ── lifecycle ──────────────────────────────────────────────────────────

    public synchronized void start() {
        if (started) throw new IllegalStateException("Scheduler already started.");
        taskExecutor.start();
        started = true;
        log.info("TaskScheduler started.");
    }

    /**
     * Initiates an orderly shutdown. Running tasks are allowed to finish.
     * Pending triggers and retries are cancelled.
     *
     * @param timeoutSeconds maximum seconds to wait for running tasks to complete
     */
    public void shutdown(long timeoutSeconds) {
        if (shutdown) return;
        shutdown = true;
        log.info("TaskScheduler shutting down...");

        // Cancel all pending triggers
        scheduledFutures.values().forEach(f -> f.cancel(false));
        scheduledFutures.clear();

        taskExecutor.stop();
        triggerScheduler.shutdownNow();
        retryScheduler.shutdownNow();

        workerPool.shutdown();
        try {
            if (!workerPool.awaitTermination(timeoutSeconds, TimeUnit.SECONDS)) {
                workerPool.shutdownNow();
            }
        } catch (InterruptedException e) {
            workerPool.shutdownNow();
            Thread.currentThread().interrupt();
        }
        log.info("TaskScheduler shut down.");
    }

    // ── scheduling API ─────────────────────────────────────────────────────

    /**
     * Registers and schedules a one-time task to run at {@code executeAt}.
     *
     * @throws IllegalArgumentException if the task has unsatisfied dependency references
     *                                  or would create a dependency cycle
     * @throws IllegalStateException    if the scheduler has not been started
     */
    public void scheduleAt(Task task, Instant executeAt) {
        requireStarted();
        register(task);

        long delayMs = Math.max(0, Duration.between(clock.instant(), executeAt).toMillis());
        ScheduledFuture<?> future = triggerScheduler.schedule(
                () -> enqueueIfEligible(task), delayMs, TimeUnit.MILLISECONDS);
        scheduledFutures.put(task.getId(), future);

        log.info(String.format("Scheduled one-time task '%s' [%s] in %d ms.",
                task.getName(), task.getId(), delayMs));
    }

    /**
     * Registers and schedules a recurring task.
     * The first execution is determined by the task's cron parser.
     */
    public void scheduleRecurring(RecurringTask task) {
        requireStarted();
        register(task);
        scheduleTrigger(task);
        log.info(String.format("Scheduled recurring task '%s' [%s] with cron '%s'; first run at %s.",
                task.getName(), task.getId(), task.getCronExpression(), task.getNextExecutionTime()));
    }

    /**
     * Cancels a pending or recurring task.
     * If the task is currently running, it will complete but no further executions are scheduled.
     */
    public boolean cancel(String taskId) {
        Task task = registry.get(taskId);
        if (task == null) return false;

        TaskStatus current = task.getStatus();
        if (current == TaskStatus.COMPLETED
                || current == TaskStatus.FAILED
                || current == TaskStatus.CANCELLED) {
            return false;
        }

        task.setStatus(TaskStatus.CANCELLED);

        ScheduledFuture<?> future = scheduledFutures.remove(taskId);
        if (future != null) future.cancel(false);

        dependencyGraph.deregister(taskId);
        log.info("Cancelled task '" + task.getName() + "' [" + taskId + "].");
        return true;
    }

    // ── observer management ────────────────────────────────────────────────

    public void addListener(TaskEventListener listener) {
        listeners.add(listener);
    }

    public void removeListener(TaskEventListener listener) {
        listeners.remove(listener);
    }

    // ── query API ──────────────────────────────────────────────────────────

    public Optional<Task> getTask(String taskId) {
        return Optional.ofNullable(registry.get(taskId));
    }

    public TaskStatus getStatus(String taskId) {
        return Optional.ofNullable(registry.get(taskId))
                .map(Task::getStatus)
                .orElseThrow(() -> new NoSuchElementException("Unknown task: " + taskId));
    }

    public List<Task> getAllTasks() {
        return List.copyOf(registry.values());
    }

    // ── internals ─────────────────────────────────────────────────────────

    private void register(Task task) {
        registry.put(task.getId(), task);
        try {
            dependencyGraph.register(task, registry);
        } catch (IllegalArgumentException e) {
            registry.remove(task.getId());
            throw e;
        }
    }

    /**
     * Enqueues the task only if it is still PENDING and not cancelled.
     * For tasks that have unmet dependencies, the task is still enqueued —
     * the executor's drain loop will re-check and re-queue if needed.
     */
    private void enqueueIfEligible(Task task) {
        if (task.getStatus() == TaskStatus.PENDING) {
            readyQueue.offer(task);
        }
    }

    /**
     * Schedules the next trigger for a recurring task based on its {@code nextExecutionTime}.
     * Called after each completed recurring run to chain the next one.
     */
    private void scheduleTrigger(RecurringTask task) {
        if (task.getStatus() == TaskStatus.CANCELLED || shutdown) return;

        ZonedDateTime next = task.getNextExecutionTime();
        long delayMs = Math.max(0,
                Duration.between(ZonedDateTime.now(clock), next).toMillis());

        ScheduledFuture<?> future = triggerScheduler.schedule(() -> {
            enqueueIfEligible(task);
            // After the task finishes its current run, advanceNextExecution() is called
            // inside TaskExecutor.handleSuccess(). We need to re-schedule the trigger
            // once execution completes. We do that by adding a one-shot listener here.
            addTransientRecurringListener(task);
        }, delayMs, TimeUnit.MILLISECONDS);

        scheduledFutures.put(task.getId(), future);
    }

    /**
     * Adds a one-shot listener that re-schedules the recurring task trigger
     * after each execution completes or fails (the task itself decides whether
     * to keep running via its status in handleSuccess).
     */
    private void addTransientRecurringListener(RecurringTask task) {
        TaskEventListener transient_ = new TaskEventListener() {
            @Override
            public void onComplete(Task t) {
                if (t.getId().equals(task.getId())) {
                    listeners.remove(this);
                    scheduleTrigger(task);
                }
            }
            @Override
            public void onFailure(Task t, Throwable cause) {
                if (t.getId().equals(task.getId())) {
                    listeners.remove(this);
                    // Still re-schedule: recurring tasks continue even after a failed run
                    if (task.getStatus() != TaskStatus.CANCELLED) {
                        scheduleTrigger(task);
                    }
                }
            }
        };
        listeners.add(transient_);
    }

    private void requireStarted() {
        if (!started) throw new IllegalStateException("Scheduler has not been started. Call start() first.");
        if (shutdown) throw new IllegalStateException("Scheduler has been shut down.");
    }

    // ── Builder ────────────────────────────────────────────────────────────

    public static Builder builder() { return new Builder(); }

    public static final class Builder {
        private int triggerThreads = 2;
        private int workerThreads  = Runtime.getRuntime().availableProcessors();
        private Clock clock = Clock.systemDefaultZone();

        public Builder triggerThreads(int n) { this.triggerThreads = n; return this; }
        public Builder workerThreads(int n)  { this.workerThreads  = n; return this; }
        public Builder clock(Clock clock)    { this.clock = clock;       return this; }

        public TaskScheduler build() { return new TaskScheduler(this); }
    }
}
```

---

### 11. `TaskSchedulerDemo.java` — Integration demo

```java
package scheduler;

import java.time.*;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.logging.*;

/**
 * Demonstrates all six functional requirements in sequence.
 *
 * Run with: javac -d out src/scheduler/*.java && java -cp out scheduler.TaskSchedulerDemo
 */
public class TaskSchedulerDemo {

    private static final Logger log = Logger.getLogger(TaskSchedulerDemo.class.getName());

    public static void main(String[] args) throws InterruptedException {
        configureLogging();

        TaskScheduler scheduler = TaskScheduler.builder()
                .workerThreads(4)
                .build();

        CountDownLatch latch = new CountDownLatch(3); // wait for 3 tasks to complete

        // ── Observer ────────────────────────────────────────────────────────
        scheduler.addListener(new TaskEventListener() {
            @Override
            public void onComplete(Task t) {
                log.info("[LISTENER] Task COMPLETED: " + t);
                if (!(t instanceof RecurringTask)) latch.countDown();
            }
            @Override
            public void onFailure(Task t, Throwable cause) {
                log.warning("[LISTENER] Task FAILED: " + t + " — " + cause.getMessage());
                latch.countDown();
            }
        });

        scheduler.start();

        // ── FR-1: One-time task ──────────────────────────────────────────────
        Task oneTime = Task.builder("one-time-hello", () -> {
            log.info("==> One-time task executing!");
            return null;
        }).priority(3).build();

        scheduler.scheduleAt(oneTime, Instant.now().plusSeconds(2));

        // ── FR-3: Priority — two tasks at the same time; higher priority runs first ──
        AtomicInteger executionOrder = new AtomicInteger(0);

        Task lowPriority = Task.builder("low-priority", () -> {
            int order = executionOrder.incrementAndGet();
            log.info("==> Low-priority task executing (order=" + order + ")");
            return null;
        }).priority(1).build();

        Task highPriority = Task.builder("high-priority", () -> {
            int order = executionOrder.incrementAndGet();
            log.info("==> High-priority task executing (order=" + order + ")");
            return null;
        }).priority(10).build();

        Instant sameTime = Instant.now().plusSeconds(4);
        scheduler.scheduleAt(lowPriority, sameTime);
        scheduler.scheduleAt(highPriority, sameTime);

        // ── FR-5: Dependency — taskC runs only after taskA ────────────────────
        Task taskA = Task.builder("task-A", () -> {
            log.info("==> Task A executing (prerequisite)");
            return null;
        }).id("task-A").priority(5).build();

        Task taskB = Task.builder("task-B", () -> {
            log.info("==> Task B executing (depends on A)");
            return null;
        }).id("task-B").priority(5).dependsOn("task-A").build();

        scheduler.scheduleAt(taskA, Instant.now().plusSeconds(6));
        scheduler.scheduleAt(taskB, Instant.now().plusSeconds(6)); // same trigger time, but blocked by dep

        // ── FR-6: Retry — task that fails twice before succeeding ─────────────
        AtomicInteger attempts = new AtomicInteger(0);
        Task flaky = Task.builder("flaky-task", () -> {
            int n = attempts.incrementAndGet();
            if (n < 3) throw new RuntimeException("Simulated failure #" + n);
            log.info("==> Flaky task succeeded on attempt " + n);
            return null;
        })
        .maxRetries(3)
        .retryPolicy(new ExponentialBackoffRetryPolicy(500L, 5_000L))
        .build();

        scheduler.scheduleAt(flaky, Instant.now().plusSeconds(8));

        // ── FR-2: Recurring task (every minute in a real environment) ─────────
        // For demo, we use a 1-second polling cron-like setup via the scheduler.
        // Real cron minimum is 1 minute; we demonstrate the wiring here.
        RecurringTask recurring = RecurringTask.recurringBuilder(
                "recurring-report",
                () -> {
                    log.info("==> Recurring report executed at " + ZonedDateTime.now());
                    return null;
                },
                "* * * * *"  // every minute
        ).priority(2).build();

        scheduler.scheduleRecurring(recurring);
        log.info("Recurring task next run: " + recurring.getNextExecutionTime());

        // ── FR-4: Cancellation ────────────────────────────────────────────────
        Task toCancel = Task.builder("cancel-me", () -> {
            log.info("==> This should never print!");
            return null;
        }).build();

        scheduler.scheduleAt(toCancel, Instant.now().plusSeconds(20));
        boolean cancelled = scheduler.cancel(toCancel.getId());
        log.info("Cancellation of 'cancel-me' succeeded: " + cancelled);

        // ── Await completion ──────────────────────────────────────────────────
        boolean done = latch.await(30, TimeUnit.SECONDS);
        log.info("All expected tasks finished: " + done);

        // Print final statuses
        scheduler.getAllTasks().forEach(t -> log.info("Final status: " + t));

        scheduler.shutdown(10);
    }

    private static void configureLogging() {
        Logger root = Logger.getLogger("");
        root.setLevel(Level.INFO);
        ConsoleHandler ch = new ConsoleHandler();
        ch.setFormatter(new SimpleFormatter());
        ch.setLevel(Level.ALL);
        root.addHandler(ch);
    }
}
```

---

## Design Patterns Used

### 1. Strategy — `RetryPolicy`

The retry delay computation is fully decoupled from the executor. Clients can supply any `RetryPolicy` implementation (exponential backoff, fixed delay, no retry, jitter-based) without touching `TaskExecutor`.

```
RetryPolicy (interface)
    └── ExponentialBackoffRetryPolicy
    └── FixedDelayRetryPolicy          (extension point)
    └── JitterRetryPolicy              (extension point)
```

### 2. Observer — `TaskEventListener`

`TaskScheduler` maintains a `CopyOnWriteArrayList<TaskEventListener>`. After each task completes or permanently fails, all registered listeners are notified asynchronously via the worker pool. The transient recurring-task listener is a special case of the same pattern.

```
TaskEventListener (interface)
    ├── onComplete(Task)
    └── onFailure(Task, Throwable)
```

### 3. Builder — `Task.Builder` / `RecurringTask.RecurringBuilder`

Both `Task` and `RecurringTask` are immutable after construction. All optional fields have sensible defaults. The fluent builders enforce that mandatory fields (`name`, `action`) are provided at compile time.

### 4. Template Method (implicit) — `RecurringTask extends Task`

`RecurringTask` inherits the full lifecycle (status transitions, retry, observer notifications) from `Task` and only overrides the concern it owns: computing the next execution time via `advanceNextExecution`.

---

## Extension Points

| Extension | How to implement |
|-----------|-----------------|
| Persistent task store | Replace the in-memory `ConcurrentHashMap` registry with a `TaskRepository` interface; provide a JDBC or Redis implementation. Add a `restore()` method on startup to reload PENDING tasks. |
| Distributed scheduling | Replace `ScheduledExecutorService` triggers with a distributed timer (Redis Sorted Set, or Kafka scheduled messages). Use a leader-election lock (ZooKeeper, Redisson) to ensure only one node fires each trigger. |
| Second-level cron granularity | Add a 6th field to `SimpleCronParser` and adjust the increment unit in `nextExecution()` from 1 minute to 1 second. |
| Rate limiting / throttling | Add a `Semaphore` (or token-bucket) inside `TaskExecutor.drainLoop()` before offering to the worker pool. Expose a `maxConcurrency` setting on `Task`. |
| Dead-letter queue | In `TaskExecutor.handleFailure()`, after transitioning to `FAILED`, publish the task to a `DeadLetterQueue` (e.g., a separate `BlockingQueue` or a persistent store) for manual replay. |
| Metrics & observability | Add a `MeterRegistry` (Micrometer) to `TaskExecutor`; record execution duration, retry count, and queue depth as gauges/timers. |
| Dynamic cron expression update | Expose `updateCron(String taskId, String newExpression)` on `TaskScheduler`; cancel the existing trigger, update the `RecurringTask`, and call `scheduleTrigger()` again. |
| Web UI / REST API | Wrap `TaskScheduler` in a Spring Boot `@RestController`; expose endpoints for `POST /tasks`, `DELETE /tasks/{id}`, `GET /tasks/{id}/status`. |
| Custom thread factory / virtual threads | Pass a custom `ThreadFactory` via the builder; swap to `Thread.ofVirtual().factory()` (Java 21+) for I/O-bound tasks at scale. |
| Audit log | Implement `TaskEventListener` that writes to an append-only audit table; register it at startup. |
