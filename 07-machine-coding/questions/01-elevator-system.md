# Machine Coding: Elevator System

**Difficulty:** Medium–Hard
**Time Limit:** 60–90 minutes (interview setting)
**Topics:** OOP, State Pattern, Scheduling Algorithms, Multi-entity Coordination

---

## 1. Problem Statement

You are designing the software for a **multi-elevator system** in a commercial building. The building has **N floors** and **M elevators**. Each elevator can travel between any floor and must serve passenger requests efficiently.

A **request** can originate from two sources:
1. **Hall call** — a passenger presses the UP or DOWN button on a floor panel.
2. **Car call** — a passenger inside an elevator presses a destination floor button.

Your system must:
- Accept hall calls and car calls.
- Dispatch the optimal elevator to serve hall calls.
- Move each elevator in an efficient manner so that it does not unnecessarily reverse direction.
- Handle all edge cases: elevator already at the requested floor, multiple simultaneous requests, idle elevators, etc.

The system will be evaluated on **correctness of elevator movement**, **efficiency of scheduling**, and **code quality** (clean OOP, separation of concerns, no God classes).

You do NOT need to implement a real-time simulation loop or threading. Implement the core scheduling and movement logic with a step-based model where each call to `step()` advances the simulation by one floor.

---

## 2. Functional Requirements

| # | Requirement |
|---|-------------|
| FR-1 | The system supports a configurable number of floors (minimum 2) and elevators (minimum 1). |
| FR-2 | A user on any floor can press UP or DOWN to request an elevator (hall call). |
| FR-3 | A user inside an elevator can press any floor button to set a destination (car call). |
| FR-4 | Each elevator moves one floor per `step()` call. |
| FR-5 | An elevator opens its doors when it arrives at a floor that has a pending stop. |
| FR-6 | The dispatcher assigns hall calls to the most suitable available elevator. |
| FR-7 | Each elevator uses the LOOK algorithm to process its queue: service all requests in the current direction first, then reverse. |
| FR-8 | An idle elevator at the correct floor serves a request immediately (no movement needed). |
| FR-9 | The system must correctly represent elevator states: IDLE, MOVING_UP, MOVING_DOWN, STOPPED (doors open). |
| FR-10 | The system must support querying the current state, current floor, and pending stops of any elevator. |

---

## 3. Assumptions

- **Floors are numbered 1 through N** (1-indexed, no basement by default).
- **One step = one floor of movement.** Processing requests (opening doors, boarding) is instantaneous in this model — it does not consume a step.
- **A hall call specifies a direction** (UP or DOWN). The elevator that responds will pick up passengers going that direction.
- **Car calls have no direction** — they are just destination floors pressed by someone already inside.
- **Dispatcher re-runs on every new hall call.** It does not proactively balance load between steps.
- **An elevator already moving in the right direction can pick up a hall call on the way** — the LOOK algorithm accounts for this.
- **No weight limits, no VIP floors, no maintenance mode** in the base implementation (extension points are discussed in Section 7).
- **Thread safety is not required** for the base implementation. The simulation is single-threaded and step-driven.
- **All elevators start at floor 1, IDLE.**

---

## 4. High-Level Design Walkthrough

### 4.1 Core Entities

```
ElevatorSystem
    ├── ElevatorController         (one per elevator)
    │       ├── Elevator           (physical state: floor, direction, doors)
    │       └── RequestQueue       (sorted stops for LOOK algorithm)
    ├── Dispatcher                 (assigns hall calls to best controller)
    └── Floor[]                    (floor metadata, optional panel buttons)
```

**Why separate `ElevatorController` from `Elevator`?**
The `Elevator` is a pure value object — it holds physical state (current floor, door status). The `ElevatorController` contains the scheduling intelligence (request queue, LOOK algorithm, step logic). This mirrors the real world: an elevator car is dumb hardware; the controller PCB runs the algorithm.

### 4.2 LOOK Algorithm

The LOOK algorithm (a variant of SCAN) works as follows:

1. While moving UP: service all stops at floors >= current floor, in ascending order.
2. When no more stops exist above (or in the current direction): reverse direction.
3. While moving DOWN: service all stops at floors <= current floor, in descending order.
4. When no stops remain: go IDLE.

Unlike SCAN (which goes all the way to the top/bottom floor), LOOK only travels as far as the highest/lowest pending request — hence the name "LOOK ahead before reversing."

**Data structure choice:** Two `TreeSet<Integer>` — one for upward stops, one for downward stops. This gives O(log n) insertion and O(1) access to the next stop in either direction (`first()` / `last()`).

### 4.3 Dispatcher Strategy

The dispatcher scores each elevator for a given hall call and picks the lowest-cost one:

| Scenario | Score |
|---|---|
| Elevator is IDLE at the requested floor | 0 (perfect) |
| Elevator is IDLE, not at the floor | distance to floor |
| Elevator moving toward the floor, same direction as request | distance to floor |
| Elevator moving toward the floor, opposite direction | distance + penalty |
| Elevator moving away from the floor | large penalty |

A lower score = better candidate. Ties broken by elevator ID (deterministic).

### 4.4 State Machine

```
         addRequest()                   step() reaches stop
IDLE ─────────────────► MOVING_UP/DOWN ──────────────────► STOPPED
  ▲                          │                                  │
  │                          │ step() no more stops             │ step() (doors close)
  └──────────────────────────┘◄─────────────────────────────────┘
```

---

## 5. Complete Java Implementation

```java
import java.util.*;

// ─────────────────────────────────────────────
// Direction.java
// ─────────────────────────────────────────────
enum Direction {
    UP, DOWN, NONE
}

// ─────────────────────────────────────────────
// ElevatorState.java
// ─────────────────────────────────────────────
enum ElevatorState {
    IDLE,         // No pending requests, doors closed
    MOVING_UP,    // Travelling upward
    MOVING_DOWN,  // Travelling downward
    STOPPED       // Arrived at a stop, doors open (1 step to close)
}

// ─────────────────────────────────────────────
// Request.java
// ─────────────────────────────────────────────
class Request {
    private final int floor;
    private final Direction direction; // NONE for car calls
    private final RequestType type;

    public enum RequestType {
        HALL_CALL,  // from floor panel
        CAR_CALL    // from inside elevator
    }

    // Hall call constructor
    public Request(int floor, Direction direction) {
        if (direction == Direction.NONE) {
            throw new IllegalArgumentException("Hall call must have a direction (UP or DOWN).");
        }
        this.floor = floor;
        this.direction = direction;
        this.type = RequestType.HALL_CALL;
    }

    // Car call constructor
    public Request(int floor) {
        this.floor = floor;
        this.direction = Direction.NONE;
        this.type = RequestType.CAR_CALL;
    }

    public int getFloor() { return floor; }
    public Direction getDirection() { return direction; }
    public RequestType getType() { return type; }

    @Override
    public String toString() {
        return type == RequestType.HALL_CALL
            ? String.format("HallCall(floor=%d, dir=%s)", floor, direction)
            : String.format("CarCall(floor=%d)", floor);
    }
}

// ─────────────────────────────────────────────
// Floor.java
// ─────────────────────────────────────────────
class Floor {
    private final int floorNumber;
    private boolean upButtonPressed;
    private boolean downButtonPressed;

    public Floor(int floorNumber) {
        this.floorNumber = floorNumber;
        this.upButtonPressed = false;
        this.downButtonPressed = false;
    }

    public int getFloorNumber() { return floorNumber; }

    public void pressUp()   { this.upButtonPressed = true; }
    public void pressDown() { this.downButtonPressed = true; }

    public void clearUp()   { this.upButtonPressed = false; }
    public void clearDown() { this.downButtonPressed = false; }

    public boolean isUpPressed()   { return upButtonPressed; }
    public boolean isDownPressed() { return downButtonPressed; }

    @Override
    public String toString() {
        return String.format("Floor %d [UP:%s DOWN:%s]",
            floorNumber,
            upButtonPressed ? "ON" : "off",
            downButtonPressed ? "ON" : "off");
    }
}

// ─────────────────────────────────────────────
// Elevator.java  — pure physical state
// ─────────────────────────────────────────────
class Elevator {
    private final int id;
    private int currentFloor;
    private ElevatorState state;
    private Direction direction;
    private boolean doorsOpen;

    public Elevator(int id, int startFloor) {
        this.id = id;
        this.currentFloor = startFloor;
        this.state = ElevatorState.IDLE;
        this.direction = Direction.NONE;
        this.doorsOpen = false;
    }

    public int getId()            { return id; }
    public int getCurrentFloor()  { return currentFloor; }
    public ElevatorState getState() { return state; }
    public Direction getDirection() { return direction; }
    public boolean isDoorsOpen()  { return doorsOpen; }

    public void setCurrentFloor(int floor) { this.currentFloor = floor; }
    public void setState(ElevatorState state) { this.state = state; }
    public void setDirection(Direction direction) { this.direction = direction; }
    public void setDoorsOpen(boolean open) { this.doorsOpen = open; }

    @Override
    public String toString() {
        return String.format("Elevator[id=%d floor=%d state=%s dir=%s doors=%s]",
            id, currentFloor, state, direction, doorsOpen ? "OPEN" : "closed");
    }
}

// ─────────────────────────────────────────────
// ElevatorController.java  — LOOK algorithm + scheduling
// ─────────────────────────────────────────────
class ElevatorController {

    private final Elevator elevator;

    /**
     * upStops: floors to visit while moving UP (ascending order).
     * downStops: floors to visit while moving DOWN (descending order).
     *
     * When a stop is added:
     *   - If the elevator is IDLE or MOVING_UP and the target floor > current → upStops
     *   - If the elevator is MOVING_DOWN and the target floor < current → downStops
     *   - If the target floor is in the "wrong" direction for the current pass,
     *     it goes into the opposite set so the LOOK algorithm picks it up on the return.
     *
     * This is the core of LOOK: we never mix up/down in the same set.
     */
    private final TreeSet<Integer> upStops   = new TreeSet<>();
    private final TreeSet<Integer> downStops = new TreeSet<>(Comparator.reverseOrder());

    // Tracks whether a STOPPED step needs to close doors before moving on.
    private boolean doorsJustOpened = false;

    public ElevatorController(int id, int startFloor) {
        this.elevator = new Elevator(id, startFloor);
    }

    public Elevator getElevator() { return elevator; }

    // ── Public API ────────────────────────────────────────────────────────────

    /**
     * Add a stop to this elevator's schedule.
     * Works for both hall calls (with direction hint) and car calls.
     */
    public void addRequest(Request request) {
        int targetFloor = request.getFloor();
        int current     = elevator.getCurrentFloor();
        ElevatorState state = elevator.getState();
        Direction dir   = elevator.getDirection();

        // Already here — open doors immediately on next step
        if (targetFloor == current) {
            elevator.setState(ElevatorState.STOPPED);
            elevator.setDoorsOpen(true);
            doorsJustOpened = true;
            System.out.printf("  [Elevator %d] Request for current floor %d — opening doors.%n",
                elevator.getId(), current);
            return;
        }

        // Decide which bucket this stop belongs to.
        // Rule: if this stop can be served in the current sweep, put it there.
        //       Otherwise, put it in the opposite bucket for the return sweep.
        if (state == ElevatorState.IDLE) {
            // Elevator not moving: choose direction based on where the target is
            if (targetFloor > current) {
                upStops.add(targetFloor);
            } else {
                downStops.add(targetFloor);
            }
        } else if (state == ElevatorState.MOVING_UP || (state == ElevatorState.STOPPED && dir == Direction.UP)) {
            if (targetFloor > current) {
                // Target is ahead in the current upward sweep
                upStops.add(targetFloor);
            } else {
                // Target is below — handle on the way back down
                downStops.add(targetFloor);
            }
        } else if (state == ElevatorState.MOVING_DOWN || (state == ElevatorState.STOPPED && dir == Direction.DOWN)) {
            if (targetFloor < current) {
                // Target is ahead in the current downward sweep
                downStops.add(targetFloor);
            } else {
                // Target is above — handle on the way back up
                upStops.add(targetFloor);
            }
        } else {
            // STOPPED with NONE direction (e.g., just opened from IDLE) — decide by position
            if (targetFloor > current) {
                upStops.add(targetFloor);
            } else {
                downStops.add(targetFloor);
            }
        }

        // Kick the elevator out of IDLE if it was waiting
        if (elevator.getState() == ElevatorState.IDLE) {
            assignNextDirection();
        }

        System.out.printf("  [Elevator %d] Queued stop at floor %d. upStops=%s downStops=%s%n",
            elevator.getId(), targetFloor, upStops, downStops);
    }

    /**
     * Advance the elevator simulation by one floor (one time unit).
     * Returns true if any action was taken (moved, opened/closed doors).
     */
    public boolean step() {
        switch (elevator.getState()) {

            case STOPPED:
                // Doors were open last step — close them and decide what to do next
                if (doorsJustOpened) {
                    doorsJustOpened = false;
                    // Doors stay open for this step (simulate passenger boarding)
                    System.out.printf("  [Elevator %d] Floor %d — doors open (passengers boarding).%n",
                        elevator.getId(), elevator.getCurrentFloor());
                    return true;
                }
                // Close doors and resume movement
                elevator.setDoorsOpen(false);
                System.out.printf("  [Elevator %d] Floor %d — doors closing.%n",
                    elevator.getId(), elevator.getCurrentFloor());
                // Now decide next move
                assignNextDirection();
                if (elevator.getState() == ElevatorState.IDLE) {
                    return true; // Nothing more to do
                }
                // Fall through to move on the same step? No — one action per step.
                return true;

            case MOVING_UP: {
                int nextFloor = elevator.getCurrentFloor() + 1;
                elevator.setCurrentFloor(nextFloor);
                System.out.printf("  [Elevator %d] Moving UP → floor %d%n",
                    elevator.getId(), nextFloor);

                // Check if this floor is a stop in the upward set
                if (upStops.contains(nextFloor)) {
                    upStops.remove(nextFloor);
                    elevator.setState(ElevatorState.STOPPED);
                    elevator.setDoorsOpen(true);
                    doorsJustOpened = true;
                    System.out.printf("  [Elevator %d] Arrived at floor %d — opening doors.%n",
                        elevator.getId(), nextFloor);
                }
                return true;
            }

            case MOVING_DOWN: {
                int nextFloor = elevator.getCurrentFloor() - 1;
                elevator.setCurrentFloor(nextFloor);
                System.out.printf("  [Elevator %d] Moving DOWN → floor %d%n",
                    elevator.getId(), nextFloor);

                // Check if this floor is a stop in the downward set
                if (downStops.contains(nextFloor)) {
                    downStops.remove(nextFloor);
                    elevator.setState(ElevatorState.STOPPED);
                    elevator.setDoorsOpen(true);
                    doorsJustOpened = true;
                    System.out.printf("  [Elevator %d] Arrived at floor %d — opening doors.%n",
                        elevator.getId(), nextFloor);
                }
                return true;
            }

            case IDLE:
            default:
                return false;
        }
    }

    /**
     * LOOK algorithm direction assignment.
     * Called after closing doors or when coming out of IDLE.
     *
     * Priority:
     *   1. Continue in the current direction if there are more stops that way.
     *   2. Reverse if the other direction has stops.
     *   3. Go IDLE if no stops remain anywhere.
     */
    private void assignNextDirection() {
        Direction currentDir = elevator.getDirection();
        int currentFloor = elevator.getCurrentFloor();

        if (currentDir == Direction.UP || currentDir == Direction.NONE) {
            // Try to keep going up first
            if (!upStops.isEmpty()) {
                elevator.setState(ElevatorState.MOVING_UP);
                elevator.setDirection(Direction.UP);
                return;
            }
            // No more up stops — check down
            if (!downStops.isEmpty()) {
                elevator.setState(ElevatorState.MOVING_DOWN);
                elevator.setDirection(Direction.DOWN);
                return;
            }
        } else if (currentDir == Direction.DOWN) {
            // Try to keep going down first
            if (!downStops.isEmpty()) {
                elevator.setState(ElevatorState.MOVING_DOWN);
                elevator.setDirection(Direction.DOWN);
                return;
            }
            // No more down stops — check up
            if (!upStops.isEmpty()) {
                elevator.setState(ElevatorState.MOVING_UP);
                elevator.setDirection(Direction.UP);
                return;
            }
        }

        // Absolutely nothing left to do
        elevator.setState(ElevatorState.IDLE);
        elevator.setDirection(Direction.NONE);
        System.out.printf("  [Elevator %d] All stops served — going IDLE at floor %d.%n",
            elevator.getId(), currentFloor);
    }

    /** Estimate the cost (steps) for this elevator to serve a hall call. Used by Dispatcher. */
    public int estimateCost(int targetFloor, Direction requestedDirection) {
        int current   = elevator.getCurrentFloor();
        ElevatorState state = elevator.getState();
        Direction dir = elevator.getDirection();
        int distance  = Math.abs(current - targetFloor);

        // Idle elevator: pure travel distance
        if (state == ElevatorState.IDLE) {
            return distance;
        }

        // Moving toward the target AND same direction as the request → can pick up on the way
        boolean movingTowardTarget =
            (dir == Direction.UP   && targetFloor >= current) ||
            (dir == Direction.DOWN && targetFloor <= current);

        boolean directionMatches =
            (dir == Direction.UP   && requestedDirection == Direction.UP) ||
            (dir == Direction.DOWN && requestedDirection == Direction.DOWN);

        if (movingTowardTarget && directionMatches) {
            return distance; // Best case: pick up on the way
        }

        if (movingTowardTarget && !directionMatches) {
            // We pass the floor but wrong direction for the passenger.
            // Passenger would need to wait for return sweep.
            // Rough estimate: finish current sweep + return.
            int furthestInCurrentDirection = getFurthestStopInCurrentDirection();
            int distToFurthest = Math.abs(furthestInCurrentDirection - current);
            int distFromFurthestToTarget = Math.abs(furthestInCurrentDirection - targetFloor);
            return distToFurthest + distFromFurthestToTarget + 20; // penalty for wrong direction
        }

        // Moving away from target — elevator will reverse at some point
        int furthestInCurrentDirection = getFurthestStopInCurrentDirection();
        int distToFurthest = Math.abs(furthestInCurrentDirection - current);
        int distFromFurthestToTarget = Math.abs(furthestInCurrentDirection - targetFloor);
        return distToFurthest + distFromFurthestToTarget + 50; // large penalty for moving away
    }

    private int getFurthestStopInCurrentDirection() {
        Direction dir = elevator.getDirection();
        if (dir == Direction.UP && !upStops.isEmpty()) {
            return upStops.last(); // TreeSet<Integer> ascending — last() is highest
        }
        if (dir == Direction.DOWN && !downStops.isEmpty()) {
            return downStops.last(); // TreeSet<>(reverseOrder()) — last() is lowest
        }
        return elevator.getCurrentFloor();
    }

    public boolean hasPendingStops() {
        return !upStops.isEmpty() || !downStops.isEmpty()
            || elevator.getState() == ElevatorState.STOPPED;
    }

    public String getQueueSummary() {
        return String.format("up=%s down=%s", upStops, new TreeSet<>(downStops));
    }
}

// ─────────────────────────────────────────────
// Dispatcher.java
// ─────────────────────────────────────────────
class Dispatcher {

    private final List<ElevatorController> controllers;

    public Dispatcher(List<ElevatorController> controllers) {
        this.controllers = Collections.unmodifiableList(new ArrayList<>(controllers));
    }

    /**
     * Assign a hall call to the best available elevator.
     * "Best" = lowest estimated cost to serve the request.
     */
    public ElevatorController dispatch(Request hallCall) {
        if (hallCall.getType() != Request.RequestType.HALL_CALL) {
            throw new IllegalArgumentException("Dispatcher only handles hall calls.");
        }

        ElevatorController best = null;
        int bestCost = Integer.MAX_VALUE;

        for (ElevatorController controller : controllers) {
            int cost = controller.estimateCost(hallCall.getFloor(), hallCall.getDirection());
            System.out.printf("  [Dispatcher] Elevator %d cost for %s: %d%n",
                controller.getElevator().getId(), hallCall, cost);
            if (cost < bestCost) {
                bestCost = cost;
                best = controller;
            }
        }

        System.out.printf("  [Dispatcher] Assigned %s → Elevator %d (cost %d)%n",
            hallCall, best.getElevator().getId(), bestCost);

        best.addRequest(hallCall);
        return best;
    }
}

// ─────────────────────────────────────────────
// ElevatorSystem.java  — facade / entry point
// ─────────────────────────────────────────────
class ElevatorSystem {

    private final int totalFloors;
    private final List<Floor> floors;
    private final List<ElevatorController> controllers;
    private final Dispatcher dispatcher;

    public ElevatorSystem(int totalFloors, int numElevators) {
        if (totalFloors < 2) throw new IllegalArgumentException("Need at least 2 floors.");
        if (numElevators < 1) throw new IllegalArgumentException("Need at least 1 elevator.");

        this.totalFloors = totalFloors;
        this.floors = new ArrayList<>(totalFloors);
        this.controllers = new ArrayList<>(numElevators);

        for (int i = 1; i <= totalFloors; i++) {
            floors.add(new Floor(i));
        }
        for (int i = 1; i <= numElevators; i++) {
            controllers.add(new ElevatorController(i, 1)); // all start at floor 1
        }

        this.dispatcher = new Dispatcher(controllers);
    }

    // ── Hall call from a floor panel ─────────────────────────────────────────

    public void hallCall(int floor, Direction direction) {
        validateFloor(floor);
        if (direction == Direction.NONE) {
            throw new IllegalArgumentException("Hall call direction must be UP or DOWN.");
        }
        Floor f = floors.get(floor - 1);
        if (direction == Direction.UP)   f.pressUp();
        else                              f.pressDown();

        System.out.printf("%n[System] Hall call: floor %d going %s%n", floor, direction);
        dispatcher.dispatch(new Request(floor, direction));
    }

    // ── Car call from inside a specific elevator ──────────────────────────────

    public void carCall(int elevatorId, int destinationFloor) {
        validateFloor(destinationFloor);
        ElevatorController controller = getController(elevatorId);
        System.out.printf("%n[System] Car call: Elevator %d → floor %d%n",
            elevatorId, destinationFloor);
        controller.addRequest(new Request(destinationFloor));
    }

    // ── Step all elevators forward by one time unit ───────────────────────────

    public void step() {
        System.out.println("\n[System] ── STEP ────────────────────────────────");
        for (ElevatorController controller : controllers) {
            controller.step();
        }
        printStatus();
    }

    public void stepN(int n) {
        for (int i = 0; i < n; i++) {
            step();
        }
    }

    // ── Status ───────────────────────────────────────────────────────────────

    public void printStatus() {
        System.out.println("[System] Status:");
        for (ElevatorController c : controllers) {
            Elevator e = c.getElevator();
            System.out.printf("  %s | queue: %s%n", e, c.getQueueSummary());
        }
    }

    // ── Helpers ──────────────────────────────────────────────────────────────

    private void validateFloor(int floor) {
        if (floor < 1 || floor > totalFloors) {
            throw new IllegalArgumentException(
                String.format("Floor %d is out of range [1, %d].", floor, totalFloors));
        }
    }

    private ElevatorController getController(int elevatorId) {
        return controllers.stream()
            .filter(c -> c.getElevator().getId() == elevatorId)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("No elevator with id: " + elevatorId));
    }

    public List<ElevatorController> getControllers() {
        return Collections.unmodifiableList(controllers);
    }
}

// ─────────────────────────────────────────────
// Main.java  — runnable demo
// ─────────────────────────────────────────────
public class Main {

    public static void main(String[] args) {
        System.out.println("=== Elevator System Demo ===");
        System.out.println("Building: 10 floors, 2 elevators\n");

        ElevatorSystem system = new ElevatorSystem(10, 2);
        system.printStatus();

        // ── Scenario 1: Simple hall call ──────────────────────────────────────
        System.out.println("\n\n>>> Scenario 1: Hall call from floor 5 going UP");
        system.hallCall(5, Direction.UP);
        // Takes 4 steps to get from floor 1 to floor 5
        system.stepN(4);  // moves to floor 5, doors open on arrival

        // Passenger boards and presses floor 8 (car call)
        System.out.println("\n>>> Passenger boards, presses floor 8");
        system.carCall(1, 8);
        system.stepN(5);  // closes doors (1) + moves 3 floors + opens doors (1)

        // ── Scenario 2: Multiple concurrent hall calls ────────────────────────
        System.out.println("\n\n>>> Scenario 2: Two simultaneous hall calls");
        // Elevator 1 is at floor 8; elevator 2 is idle at floor 1
        // Hall call from floor 3 going UP → dispatcher should prefer elevator 2 (closer)
        // Hall call from floor 9 going DOWN → dispatcher should prefer elevator 1 (closer)
        system.hallCall(3, Direction.UP);
        system.hallCall(9, Direction.DOWN);
        system.stepN(8);

        // ── Scenario 3: LOOK algorithm — requests on both sides ───────────────
        System.out.println("\n\n>>> Scenario 3: LOOK reversal");
        // Add requests that force elevator 1 to go up then come back down
        system.carCall(1, 10);  // go to floor 10
        system.carCall(1, 6);   // also need to stop at floor 6 on way up
        system.carCall(1, 2);   // then come back down to floor 2
        system.stepN(12);

        System.out.println("\n=== Demo complete ===");
    }
}
```

---

## 6. Step-by-Step Interview Walkthrough

This section explains how to **think through and present** this design during a 60–90-minute interview. The order matters — interviewers reward structured thinking, not just correct code.

### Step 1 — Clarify Requirements (5 minutes)

Before writing a single line, ask:

- How many floors? How many elevators?
- Is this real-time or step-based? (Step-based is far easier to reason about.)
- Do car calls and hall calls both need to be handled?
- What scheduling algorithm is expected? (LOOK, FCFS, SCAN?)
- Is thread safety required?

Interviewers reward candidates who frame scope before diving in.

### Step 2 — Identify Core Entities (5 minutes)

Narrate your reasoning:

> "The real world has a building, floors, elevator cars, a controller per car, and a central dispatcher. I want to model these directly so the code is self-documenting."

Draw or write out the entity map:
```
ElevatorSystem → Dispatcher → ElevatorController → Elevator
                                                 → RequestQueue (upStops, downStops)
Floor (hall panel state)
Request (hall call or car call)
```

Explain why `Elevator` and `ElevatorController` are separate: the car is hardware state; the controller is algorithm state.

### Step 3 — Define Enums and Value Objects (5 minutes)

Write `Direction`, `ElevatorState`, and `Request` first. These are small, uncontroversial, and give you a foundation:

- `Direction.UP`, `Direction.DOWN`, `Direction.NONE`
- `ElevatorState.IDLE`, `MOVING_UP`, `MOVING_DOWN`, `STOPPED`
- `Request` with both hall-call and car-call constructors

### Step 4 — Design the LOOK Algorithm (15 minutes)

This is the centerpiece. Explain the two-bucket approach:

> "I use two `TreeSet<Integer>` — one for upward stops sorted ascending, one for downward stops sorted in reverse. When moving up, I always pull from `upStops.first()` (the lowest floor above me). When moving down, I pull from `downStops.first()` (the highest floor below me, because it's reverse-sorted). When the up set empties, I switch to down, and vice versa."

Walk through a concrete trace before coding:
```
Elevator at floor 3, moving UP.
upStops   = [5, 7, 9]
downStops = [1]

Step 1: move to 4 (no stop)
Step 2: move to 5 → STOP, remove 5. upStops = [7, 9]
...
Step 4: move to 7 → STOP, remove 7. upStops = [9]
...
Step 6: move to 9 → STOP, remove 9. upStops = []
Now reverse: downStops = [1]
Step 7: move to 8
...
Step 15: move to 1 → STOP, remove 1. downStops = []
IDLE.
```

### Step 5 — Implement ElevatorController.step() (15 minutes)

Code the `step()` method with the switch on `ElevatorState`. Be explicit about the door model: arriving at a stop sets `doorsJustOpened = true`, and the NEXT call to `step()` is the "boarding" step. The step after that closes the doors and calls `assignNextDirection()`.

### Step 6 — Implement the Dispatcher (10 minutes)

Explain the cost heuristic:

> "I score each elevator by how many steps it would realistically take to serve this hall call. Idle elevators get score = distance. Moving elevators that are heading toward the floor in the right direction also get score = distance. Moving the wrong way or passing the floor get a penalty — either 20 or 50 extra steps — to make the dispatcher prefer other options."

Implement `estimateCost()` and the `dispatch()` loop.

### Step 7 — Wrap with ElevatorSystem (5 minutes)

The facade just wires everything together. `hallCall()` validates, updates `Floor` button state, and delegates to `Dispatcher`. `carCall()` looks up the controller and calls `addRequest()`. `step()` calls `step()` on all controllers.

### Step 8 — Write the Demo in Main (5 minutes)

Show three scenarios: simple hall call, simultaneous hall calls, and LOOK reversal. Running this mentally or on paper demonstrates that the algorithm is correct.

### What NOT to do in an interview

- Do not start coding `Main` first. Show design before code.
- Do not implement a `while(true)` game loop if the interviewer only asked for the scheduling logic.
- Do not make `ElevatorSystem` a singleton — it makes testing impossible.
- Do not put LOOK algorithm logic inside `Elevator` — that violates SRP. The `Elevator` is hardware state; put algorithm in `ElevatorController`.

---

## 7. Extension Points

### 7.1 Priority Floors (VIP / Emergency)

**Requirement:** Certain floors (e.g., floor 1 in a fire emergency, or a VIP penthouse) must be served before normal requests.

**Design change:**

1. Add a `boolean isPriority` field to `Request`.
2. Replace the `TreeSet<Integer>` stop sets with a `TreeSet<PrioritizedStop>` where `PrioritizedStop` has a floor and a boolean priority flag.
3. Override the `Comparator` so priority stops always sort to the front of the set, ahead of normal stops at the same or even closer floors.
4. In `ElevatorController.addRequest()`, if a priority request arrives, also interrupt a MOVING elevator — set `doorsJustOpened = false` and re-route.

```java
class PrioritizedStop implements Comparable<PrioritizedStop> {
    final int floor;
    final boolean priority;
    // priority stops sort before non-priority stops regardless of floor
    @Override
    public int compareTo(PrioritizedStop other) {
        if (this.priority != other.priority) return this.priority ? -1 : 1;
        return Integer.compare(this.floor, other.floor);
    }
}
```

### 7.2 Maintenance Mode

**Requirement:** A specific elevator can be taken out of service. It should finish its current run (if any), then park at a designated maintenance floor (usually 1) and stop accepting new requests.

**Design change:**

1. Add `MAINTENANCE` to the `ElevatorState` enum.
2. Add `setMaintenanceMode(boolean)` to `ElevatorController`. When enabled:
   - Stop accepting new `addRequest()` calls (throw `ElevatorUnavailableException` or silently re-dispatch).
   - Let the elevator finish current stops.
   - After the queue is empty, issue a car call to the maintenance floor.
   - Set state to `MAINTENANCE` when it arrives.
3. The `Dispatcher.dispatch()` method must skip controllers in `MAINTENANCE` state:

```java
for (ElevatorController controller : controllers) {
    if (controller.getElevator().getState() == ElevatorState.MAINTENANCE) continue;
    // ... cost calculation
}
```

4. For re-commissioning, add `clearMaintenanceMode()` which sets the state back to `IDLE` and allows requests again.

### 7.3 Weight Limits

**Requirement:** Each elevator has a maximum weight capacity (e.g., 1000 kg). If the elevator is at or above 80% capacity, it should not accept new hall calls (but should still serve car calls from passengers already inside).

**Design change:**

1. Add `maxWeightKg` and `currentWeightKg` to `Elevator`.
2. Add a `WeightSensor` interface that `Elevator` delegates to (enables mocking in tests):

```java
interface WeightSensor {
    int getCurrentWeightKg();
}
```

3. In `ElevatorController.addRequest()`, check the weight before accepting a hall call:

```java
if (request.getType() == Request.RequestType.HALL_CALL) {
    int currentWeight = elevator.getWeightSensor().getCurrentWeightKg();
    int maxWeight = elevator.getMaxWeightKg();
    if (currentWeight >= maxWeight * 0.8) {
        System.out.printf("  [Elevator %d] Near capacity (%d/%d kg) — rejecting hall call%n",
            elevator.getId(), currentWeight, maxWeight);
        return; // Dispatcher will try next best elevator
    }
}
```

4. The `Dispatcher` needs to handle the case where all elevators reject a hall call due to weight — add a fallback queue and retry on the next step.

---

## Quick Reference Card

| Concept | Implementation choice | Why |
|---|---|---|
| Elevator stop queue | `TreeSet<Integer>` (two: up/down) | O(log n) insert, O(1) next stop |
| Direction reversal | Two buckets, swap when one empties | Core of LOOK algorithm |
| Dispatcher scoring | Distance + direction penalties | Greedy, O(M) where M = num elevators |
| Door model | `doorsJustOpened` flag, 2-step open | Simple state machine, easy to extend |
| State machine | `ElevatorState` enum + switch in `step()` | Explicit, easy to add new states |
| Car vs hall call | `Request.RequestType` enum | Single `addRequest()` entry point |

---

*End of Problem — 01 Elevator System*
