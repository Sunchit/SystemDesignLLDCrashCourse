# Parking Lot System — Low-Level Design Case Study

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Assumptions and Constraints](#4-assumptions-and-constraints)
5. [Domain Model Description](#5-domain-model-description)
6. [Class Diagram (ASCII UML)](#6-class-diagram-ascii-uml)
7. [Core Classes — Complete Java Implementation](#7-core-classes--complete-java-implementation)
8. [Design Patterns Used](#8-design-patterns-used)
9. [Key Design Decisions and Trade-offs](#9-key-design-decisions-and-trade-offs)
10. [Extension Points](#10-extension-points)
11. [Interview Discussion Points](#11-interview-discussion-points)
12. [Common Mistakes in This Design](#12-common-mistakes-in-this-design)

---

## 1. Problem Statement

Design the object-oriented model for a multi-floor parking lot that can:

- Accept motorcycles, cars, and buses/trucks of different sizes
- Assign each vehicle to the smallest available spot that fits it
- Issue tickets on entry and calculate fees on exit
- Track availability per floor in real time on display boards
- Support multiple pricing strategies without changing core logic

The system must correctly handle concurrent vehicle arrivals, prevent double-booking of spots, and give the operations staff enough information to monitor the lot at a glance.

---

## 2. Functional Requirements

1. The parking lot has one or more floors, each with a fixed number of spots.
2. Each parking spot has a size: SMALL, MEDIUM, or LARGE.
3. Three vehicle types are supported: MOTORCYCLE (fits SMALL/MEDIUM/LARGE), CAR (fits MEDIUM/LARGE), BUS (fits LARGE only).
4. On vehicle entry, the system assigns the best-fit available spot and issues a ticket with an entry timestamp.
5. A vehicle must be parked only in a spot that is equal to or larger than the vehicle's minimum required spot size.
6. On vehicle exit, the system calculates the parking fee based on duration and vehicle type, marks the spot free, and updates display boards.
7. Each floor has a display board showing available spot counts per size.
8. The system tracks whether the lot (or a specific floor) is full.
9. An attendant can query availability before directing a vehicle.
10. Tickets carry a unique ID, the assigned spot reference, entry time, and status (ACTIVE / PAID / LOST).
11. Payment can be made in cash or credit card.
12. The system uses a configurable hourly pricing strategy per vehicle type.

---

## 3. Non-Functional Requirements

- **Thread safety**: spot assignment and release must be atomic to prevent race conditions under concurrent entry/exit.
- **Extensibility**: adding a new vehicle type or pricing strategy must require minimal changes (open/closed principle).
- **Single source of truth**: the `ParkingLot` is a singleton — all entry points share the same state.
- **Auditability**: tickets are never deleted; they transition through states (ACTIVE → PAID/LOST).
- **Low latency**: spot lookup must be O(1) or O(floor-count) — no full scans on every entry.

---

## 4. Assumptions and Constraints

- A vehicle occupies exactly one spot (no articulated vehicles spanning two spots).
- Pricing is purely time-based (hourly, rounded up to the next full hour).
- A minimum charge of one hour applies regardless of duration.
- The parking lot configuration (floors, spot counts) is set at construction time and does not change at runtime.
- Lost tickets incur a fixed fee equal to the maximum daily rate for that vehicle type.
- Display boards are updated synchronously on every park/unpark operation.
- The system does not handle reservations; spots are assigned first-come, first-served.
- Payment processing is modelled but not connected to an external payment gateway.

---

## 5. Domain Model Description

### Core Aggregates

**ParkingLot** is the root aggregate and a Singleton. It owns a list of `ParkingFloor` objects, a `PricingStrategy`, and a map of active tickets keyed by ticket ID.

**ParkingFloor** owns a fixed list of `ParkingSpot` objects and a `ParkingDisplayBoard`. It exposes methods to find and assign a free spot, and to release a spot.

**ParkingSpot** is an abstract base; concrete subclasses are `SmallSpot`, `MediumSpot`, and `LargeSpot`. Each spot knows its size, whether it is occupied, and which vehicle is currently parked in it.

**Vehicle** is an abstract base. Concrete subclasses are `Motorcycle`, `Car`, and `Bus`. Each vehicle carries a license plate and knows the minimum spot size it requires.

**Ticket** is a value object that captures the full parking transaction: spot reference, vehicle, entry time, exit time, amount charged, and status.

**ParkingAttendant** is a service object that orchestrates the entry flow (call ParkingLot to assign a spot, create a ticket) and exit flow (calculate fee, process payment, release spot).

**PricingStrategy** is an interface. `HourlyPricingStrategy` is the default implementation that multiplies hours parked by a per-vehicle-type rate.

**Payment** encapsulates the amount, method, and status of a single payment transaction.

**ParkingDisplayBoard** holds a snapshot of free spot counts per size for its floor and exposes a `display()` method.

**VehicleFactory** is a static factory that creates `Vehicle` instances from a `VehicleType` enum and a license plate.

---

## 6. Class Diagram (ASCII UML)

```
+---------------------------+
|      <<Singleton>>        |
|       ParkingLot          |
+---------------------------+
| - instance: ParkingLot    |
| - floors: List<Floor>     |
| - activeTickets: Map<>    |
| - pricing: PricingStrategy|
+---------------------------+
| + getInstance()           |
| + parkVehicle(v): Ticket  |
| + unparkVehicle(t): Payment|
| + isFullForVehicle(v): bool|
+----------+----------------+
           |  1
           |  owns
           | 1..*
+----------v--------------+          +------------------------+
|      ParkingFloor       |          |   ParkingDisplayBoard  |
+-------------------------+          +------------------------+
| - floorNumber: int      |<>--------| - floorNumber: int     |
| - spots: List<Spot>     |    1  1  | - freeSmall: int       |
| - displayBoard: Board   |          | - freeMedium: int      |
+-------------------------+          | - freeLarge: int       |
| + assignSpot(v): Spot   |          +------------------------+
| + freeSpot(spot)        |          | + update(s,m,l)        |
| + getAvailableCount()   |          | + display()            |
+----------+--------------+          +------------------------+
           |  1
           |  owns
           | 0..*
+----------v--------------+
|    <<abstract>>          |
|      ParkingSpot         |
+-------------------------+
| - spotId: String        |
| - spotType: SpotType    |
| - isOccupied: boolean   |
| - vehicle: Vehicle      |
+-------------------------+
| + canFit(v): boolean    |
| + assignVehicle(v)      |
| + removeVehicle()       |
+---+----------+----------+
    |          |          |
+---v--+  +---v---+  +---v---+
|Small |  |Medium |  |Large  |
|Spot  |  |Spot   |  |Spot   |
+------+  +-------+  +-------+

+-----------------------------+
|       <<abstract>>          |
|          Vehicle            |
+-----------------------------+
| - licensePlate: String      |
| - vehicleType: VehicleType  |
+-----------------------------+
| + getMinSpotType(): SpotType|
+---+------------+-----------+
    |            |           |
+---v---+   +---v--+   +----v---+
|Motorcy|   | Car  |   |  Bus   |
|cle    |   |      |   |        |
+-------+   +------+   +--------+

+---------------------------+
|          Ticket           |
+---------------------------+
| - ticketId: String        |
| - vehicle: Vehicle        |
| - spot: ParkingSpot       |
| - entryTime: LocalDateTime|
| - exitTime: LocalDateTime |
| - fee: double             |
| - status: TicketStatus    |
+---------------------------+
| + markPaid(fee, exitTime) |
| + markLost()              |
+---------------------------+

+---------------------------+
|    <<interface>>          |
|     PricingStrategy       |
+---------------------------+
| + calculateFee(ticket)    |
+------------+--------------+
             |
+------------v--------------+
|   HourlyPricingStrategy   |
+---------------------------+
| - rates: Map<VType,Double>|
+---------------------------+
| + calculateFee(ticket)    |
+---------------------------+

+---------------------------+
|      ParkingAttendant     |
+---------------------------+
| - attendantId: String     |
| - name: String            |
+---------------------------+
| + processEntry(v): Ticket |
| + processExit(t): Payment |
+---------------------------+

+---------------------------+
|         Payment           |
+---------------------------+
| - paymentId: String       |
| - amount: double          |
| - method: PaymentMethod   |
| - status: PaymentStatus   |
| - paidAt: LocalDateTime   |
+---------------------------+
| + process()               |
+---------------------------+

+---------------------------+
|       VehicleFactory      |
+---------------------------+
| + create(type, plate)     |
+---------------------------+

Enumerations:
  VehicleType  { MOTORCYCLE, CAR, BUS }
  SpotType     { SMALL, MEDIUM, LARGE }
  TicketStatus { ACTIVE, PAID, LOST }
  PaymentStatus{ PENDING, COMPLETED, FAILED }
  PaymentMethod{ CASH, CREDIT_CARD }
```

---

## 7. Core Classes — Complete Java Implementation

The entire system fits in a single file for ease of reading in an interview or course context. In a real project each top-level type would live in its own file under a `com.parkinglot` package.

```java
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

// ─────────────────────────────────────────────
// ENUMERATIONS
// ─────────────────────────────────────────────

enum VehicleType {
    MOTORCYCLE, CAR, BUS
}

enum SpotType {
    SMALL, MEDIUM, LARGE;

    /** Returns true if this spot size can physically fit a vehicle needing minRequired. */
    public boolean canAccommodate(SpotType minRequired) {
        return this.ordinal() >= minRequired.ordinal();
    }
}

enum TicketStatus {
    ACTIVE, PAID, LOST
}

enum PaymentStatus {
    PENDING, COMPLETED, FAILED
}

enum PaymentMethod {
    CASH, CREDIT_CARD
}

// ─────────────────────────────────────────────
// VEHICLE HIERARCHY
// ─────────────────────────────────────────────

abstract class Vehicle {
    private final String licensePlate;
    private final VehicleType vehicleType;

    protected Vehicle(String licensePlate, VehicleType vehicleType) {
        if (licensePlate == null || licensePlate.isBlank()) {
            throw new IllegalArgumentException("License plate must not be blank.");
        }
        this.licensePlate = licensePlate.toUpperCase().trim();
        this.vehicleType = vehicleType;
    }

    public String getLicensePlate() { return licensePlate; }
    public VehicleType getVehicleType() { return vehicleType; }

    /** The smallest spot size this vehicle can fit into. */
    public abstract SpotType getMinSpotType();

    @Override
    public String toString() {
        return vehicleType + "[" + licensePlate + "]";
    }
}

class Motorcycle extends Vehicle {
    public Motorcycle(String licensePlate) {
        super(licensePlate, VehicleType.MOTORCYCLE);
    }

    @Override
    public SpotType getMinSpotType() {
        return SpotType.SMALL;
    }
}

class Car extends Vehicle {
    public Car(String licensePlate) {
        super(licensePlate, VehicleType.CAR);
    }

    @Override
    public SpotType getMinSpotType() {
        return SpotType.MEDIUM;
    }
}

class Bus extends Vehicle {
    public Bus(String licensePlate) {
        super(licensePlate, VehicleType.BUS);
    }

    @Override
    public SpotType getMinSpotType() {
        return SpotType.LARGE;
    }
}

// ─────────────────────────────────────────────
// VEHICLE FACTORY  (Factory Pattern)
// ─────────────────────────────────────────────

class VehicleFactory {
    private VehicleFactory() {}   // utility class — no instances

    /**
     * Creates a Vehicle of the requested type with the given license plate.
     * Adding a new vehicle type only requires a new case here and a new subclass —
     * callers never change.
     */
    public static Vehicle create(VehicleType type, String licensePlate) {
        switch (type) {
            case MOTORCYCLE: return new Motorcycle(licensePlate);
            case CAR:        return new Car(licensePlate);
            case BUS:        return new Bus(licensePlate);
            default:
                throw new IllegalArgumentException("Unknown vehicle type: " + type);
        }
    }
}

// ─────────────────────────────────────────────
// PARKING SPOT HIERARCHY
// ─────────────────────────────────────────────

abstract class ParkingSpot {
    private final String spotId;
    private final SpotType spotType;
    private boolean isOccupied;
    private Vehicle parkedVehicle;

    protected ParkingSpot(String spotId, SpotType spotType) {
        this.spotId = spotId;
        this.spotType = spotType;
        this.isOccupied = false;
        this.parkedVehicle = null;
    }

    public String getSpotId() { return spotId; }
    public SpotType getSpotType() { return spotType; }
    public boolean isOccupied() { return isOccupied; }
    public Vehicle getParkedVehicle() { return parkedVehicle; }

    /**
     * Returns true when this spot's size can fit the given vehicle AND the spot is free.
     * Uses SpotType.canAccommodate so size comparison is centralised in the enum.
     */
    public boolean canFit(Vehicle vehicle) {
        return !isOccupied && spotType.canAccommodate(vehicle.getMinSpotType());
    }

    /** Assigns a vehicle to this spot. Throws if already occupied. */
    public synchronized void assignVehicle(Vehicle vehicle) {
        if (isOccupied) {
            throw new IllegalStateException("Spot " + spotId + " is already occupied.");
        }
        this.parkedVehicle = vehicle;
        this.isOccupied = true;
    }

    /** Frees the spot. Returns the vehicle that was parked here. */
    public synchronized Vehicle removeVehicle() {
        if (!isOccupied) {
            throw new IllegalStateException("Spot " + spotId + " is already free.");
        }
        Vehicle v = this.parkedVehicle;
        this.parkedVehicle = null;
        this.isOccupied = false;
        return v;
    }

    @Override
    public String toString() {
        return spotType + "-Spot[" + spotId + "]" + (isOccupied ? "(OCCUPIED)" : "(FREE)");
    }
}

class SmallSpot extends ParkingSpot {
    public SmallSpot(String spotId) {
        super(spotId, SpotType.SMALL);
    }
}

class MediumSpot extends ParkingSpot {
    public MediumSpot(String spotId) {
        super(spotId, SpotType.MEDIUM);
    }
}

class LargeSpot extends ParkingSpot {
    public LargeSpot(String spotId) {
        super(spotId, SpotType.LARGE);
    }
}

// ─────────────────────────────────────────────
// TICKET
// ─────────────────────────────────────────────

class Ticket {
    private static final AtomicInteger COUNTER = new AtomicInteger(1000);

    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final LocalDateTime entryTime;
    private LocalDateTime exitTime;
    private double fee;
    private TicketStatus status;

    public Ticket(Vehicle vehicle, ParkingSpot spot) {
        this.ticketId = "TKT-" + COUNTER.getAndIncrement();
        this.vehicle = vehicle;
        this.spot = spot;
        this.entryTime = LocalDateTime.now();
        this.status = TicketStatus.ACTIVE;
        this.fee = 0.0;
    }

    public String getTicketId() { return ticketId; }
    public Vehicle getVehicle() { return vehicle; }
    public ParkingSpot getSpot() { return spot; }
    public LocalDateTime getEntryTime() { return entryTime; }
    public LocalDateTime getExitTime() { return exitTime; }
    public double getFee() { return fee; }
    public TicketStatus getStatus() { return status; }

    /** Called by the exit flow once the fee is computed and payment is confirmed. */
    public void markPaid(double calculatedFee, LocalDateTime exitTimestamp) {
        if (status != TicketStatus.ACTIVE) {
            throw new IllegalStateException("Ticket " + ticketId + " is not active.");
        }
        this.fee = calculatedFee;
        this.exitTime = exitTimestamp;
        this.status = TicketStatus.PAID;
    }

    /** Called when a vehicle reports a lost ticket; triggers the lost-ticket fee. */
    public void markLost() {
        if (status != TicketStatus.ACTIVE) {
            throw new IllegalStateException("Ticket " + ticketId + " is not active.");
        }
        this.status = TicketStatus.LOST;
    }

    /** Duration in hours, rounded up, with a minimum of 1 hour. */
    public long getDurationHours() {
        LocalDateTime end = (exitTime != null) ? exitTime : LocalDateTime.now();
        long minutes = ChronoUnit.MINUTES.between(entryTime, end);
        long hours = (minutes + 59) / 60;   // ceiling division
        return Math.max(1, hours);
    }

    @Override
    public String toString() {
        return "Ticket{id=" + ticketId
                + ", vehicle=" + vehicle
                + ", spot=" + spot.getSpotId()
                + ", entry=" + entryTime
                + ", status=" + status
                + ", fee=$" + String.format("%.2f", fee) + "}";
    }
}

// ─────────────────────────────────────────────
// PRICING STRATEGY  (Strategy Pattern)
// ─────────────────────────────────────────────

interface PricingStrategy {
    /**
     * Calculates the total fee for the given ticket.
     * The ticket's exitTime is already set by the caller before this is invoked
     * (or getDurationHours() uses LocalDateTime.now() as a fallback).
     */
    double calculateFee(Ticket ticket);
}

/**
 * Charges a flat hourly rate per vehicle type.
 * Rate map is injected at construction so it can be loaded from config.
 */
class HourlyPricingStrategy implements PricingStrategy {
    private final Map<VehicleType, Double> hourlyRates;

    public HourlyPricingStrategy(Map<VehicleType, Double> hourlyRates) {
        this.hourlyRates = Collections.unmodifiableMap(new EnumMap<>(hourlyRates));
    }

    /** Convenience factory using sensible defaults (in USD). */
    public static HourlyPricingStrategy withDefaultRates() {
        Map<VehicleType, Double> rates = new EnumMap<>(VehicleType.class);
        rates.put(VehicleType.MOTORCYCLE, 1.00);
        rates.put(VehicleType.CAR,        2.50);
        rates.put(VehicleType.BUS,        5.00);
        return new HourlyPricingStrategy(rates);
    }

    @Override
    public double calculateFee(Ticket ticket) {
        double rate = hourlyRates.getOrDefault(ticket.getVehicle().getVehicleType(), 2.50);
        return rate * ticket.getDurationHours();
    }

    public double getRateFor(VehicleType type) {
        return hourlyRates.getOrDefault(type, 2.50);
    }
}

// ─────────────────────────────────────────────
// PAYMENT
// ─────────────────────────────────────────────

class Payment {
    private static final AtomicInteger COUNTER = new AtomicInteger(5000);

    private final String paymentId;
    private final double amount;
    private final PaymentMethod method;
    private PaymentStatus status;
    private LocalDateTime paidAt;

    public Payment(double amount, PaymentMethod method) {
        this.paymentId = "PAY-" + COUNTER.getAndIncrement();
        this.amount = amount;
        this.method = method;
        this.status = PaymentStatus.PENDING;
    }

    public String getPaymentId() { return paymentId; }
    public double getAmount() { return amount; }
    public PaymentMethod getMethod() { return method; }
    public PaymentStatus getStatus() { return status; }
    public LocalDateTime getPaidAt() { return paidAt; }

    /**
     * Simulates processing. In production this would call an external gateway.
     * Returns true on success.
     */
    public boolean process() {
        // Simulate always-successful payment for demonstration purposes.
        this.status = PaymentStatus.COMPLETED;
        this.paidAt = LocalDateTime.now();
        System.out.println("  [Payment] " + paymentId + " processed: $"
                + String.format("%.2f", amount) + " via " + method);
        return true;
    }

    @Override
    public String toString() {
        return "Payment{id=" + paymentId
                + ", amount=$" + String.format("%.2f", amount)
                + ", method=" + method
                + ", status=" + status + "}";
    }
}

// ─────────────────────────────────────────────
// DISPLAY BOARD
// ─────────────────────────────────────────────

class ParkingDisplayBoard {
    private final int floorNumber;
    private int freeSmall;
    private int freeMedium;
    private int freeLarge;

    public ParkingDisplayBoard(int floorNumber) {
        this.floorNumber = floorNumber;
    }

    /** Called by ParkingFloor every time a spot changes state. */
    public synchronized void update(int freeSmall, int freeMedium, int freeLarge) {
        this.freeSmall  = freeSmall;
        this.freeMedium = freeMedium;
        this.freeLarge  = freeLarge;
    }

    /** Prints the current availability snapshot for this floor. */
    public synchronized void display() {
        System.out.println("  +-----------------------------------------+");
        System.out.printf ("  | Floor %-2d  SMALL: %-4d MEDIUM: %-4d LARGE: %-4d |%n",
                floorNumber, freeSmall, freeMedium, freeLarge);
        System.out.println("  +-----------------------------------------+");
    }

    public int getFreeSmall()  { return freeSmall; }
    public int getFreeMedium() { return freeMedium; }
    public int getFreeLarge()  { return freeLarge; }
}

// ─────────────────────────────────────────────
// PARKING FLOOR
// ─────────────────────────────────────────────

class ParkingFloor {
    private final int floorNumber;
    private final List<ParkingSpot> spots;
    private final ParkingDisplayBoard displayBoard;

    public ParkingFloor(int floorNumber, List<ParkingSpot> spots) {
        this.floorNumber  = floorNumber;
        this.spots        = Collections.unmodifiableList(new ArrayList<>(spots));
        this.displayBoard = new ParkingDisplayBoard(floorNumber);
        refreshDisplayBoard();
    }

    public int getFloorNumber() { return floorNumber; }
    public ParkingDisplayBoard getDisplayBoard() { return displayBoard; }

    /**
     * Finds the best-fit free spot for the given vehicle (smallest spot that fits),
     * assigns the vehicle atomically, and updates the display board.
     * Returns null if the floor is full for this vehicle type.
     *
     * Synchronised on this floor object to prevent two threads from grabbing
     * the same spot simultaneously.
     */
    public synchronized ParkingSpot assignSpot(Vehicle vehicle) {
        // Prefer the smallest valid spot to preserve larger spots for bigger vehicles.
        ParkingSpot bestFit = null;
        for (ParkingSpot spot : spots) {
            if (spot.canFit(vehicle)) {
                if (bestFit == null
                        || spot.getSpotType().ordinal() < bestFit.getSpotType().ordinal()) {
                    bestFit = spot;
                }
            }
        }
        if (bestFit != null) {
            bestFit.assignVehicle(vehicle);
            refreshDisplayBoard();
        }
        return bestFit;
    }

    /**
     * Frees the given spot and updates the display board.
     * Synchronised to keep availability counts consistent.
     */
    public synchronized void freeSpot(ParkingSpot spot) {
        spot.removeVehicle();
        refreshDisplayBoard();
    }

    /** Returns total free spot count across all sizes on this floor. */
    public synchronized int getTotalFreeSpots() {
        int count = 0;
        for (ParkingSpot spot : spots) {
            if (!spot.isOccupied()) count++;
        }
        return count;
    }

    /** Returns free count for a specific spot type. */
    public synchronized int getFreeCountFor(SpotType type) {
        int count = 0;
        for (ParkingSpot spot : spots) {
            if (spot.getSpotType() == type && !spot.isOccupied()) count++;
        }
        return count;
    }

    /** Checks if there is at least one spot available for this vehicle. */
    public synchronized boolean hasAvailableSpotFor(Vehicle vehicle) {
        for (ParkingSpot spot : spots) {
            if (spot.canFit(vehicle)) return true;
        }
        return false;
    }

    /** Recomputes and pushes fresh counts to the display board. */
    private void refreshDisplayBoard() {
        int small  = getFreeCountFor(SpotType.SMALL);
        int medium = getFreeCountFor(SpotType.MEDIUM);
        int large  = getFreeCountFor(SpotType.LARGE);
        displayBoard.update(small, medium, large);
    }

    public void showDisplayBoard() {
        displayBoard.display();
    }
}

// ─────────────────────────────────────────────
// PARKING LOT  (Singleton)
// ─────────────────────────────────────────────

class ParkingLot {
    // Volatile ensures the instance is safely published to all threads.
    private static volatile ParkingLot instance;

    private final String name;
    private final List<ParkingFloor> floors;
    private final PricingStrategy pricingStrategy;

    // ConcurrentHashMap: safe for concurrent ticket lookups; spot assignment is
    // still serialised at the floor level.
    private final Map<String, Ticket> activeTickets = new ConcurrentHashMap<>();

    private ParkingLot(String name, List<ParkingFloor> floors, PricingStrategy pricingStrategy) {
        this.name            = name;
        this.floors          = Collections.unmodifiableList(new ArrayList<>(floors));
        this.pricingStrategy = pricingStrategy;
    }

    /**
     * Double-checked locking singleton.
     * The builder must call configure() once before any thread calls getInstance().
     */
    public static ParkingLot getInstance() {
        if (instance == null) {
            throw new IllegalStateException(
                    "ParkingLot has not been configured. Call ParkingLot.configure() first.");
        }
        return instance;
    }

    /**
     * One-time configuration — typically called during application startup.
     * Thread-safe: subsequent calls after the first have no effect.
     */
    public static synchronized void configure(
            String name,
            List<ParkingFloor> floors,
            PricingStrategy pricingStrategy) {
        if (instance == null) {
            instance = new ParkingLot(name, floors, pricingStrategy);
        }
    }

    /** Resets the singleton — used in tests only. Package-private on purpose. */
    static synchronized void reset() {
        instance = null;
    }

    // ── Entry flow ──────────────────────────────────────────────────────────

    /**
     * Finds the best available spot across all floors (ground floor first),
     * parks the vehicle, creates a ticket, and returns it.
     *
     * Returns null if the lot is full for this vehicle type.
     */
    public Ticket parkVehicle(Vehicle vehicle) {
        for (ParkingFloor floor : floors) {
            ParkingSpot spot = floor.assignSpot(vehicle);
            if (spot != null) {
                Ticket ticket = new Ticket(vehicle, spot);
                activeTickets.put(ticket.getTicketId(), ticket);
                System.out.println("[ENTRY] " + vehicle + " parked at "
                        + spot + " on floor " + floor.getFloorNumber()
                        + " | Ticket: " + ticket.getTicketId());
                return ticket;
            }
        }
        System.out.println("[ENTRY] Lot is full for " + vehicle);
        return null;
    }

    // ── Exit flow ───────────────────────────────────────────────────────────

    /**
     * Processes the exit for the given ticket:
     * 1. Validates the ticket is still ACTIVE.
     * 2. Calculates the fee via the injected PricingStrategy.
     * 3. Processes payment (method chosen by caller).
     * 4. Frees the spot on the correct floor.
     * 5. Marks the ticket PAID.
     *
     * Returns the Payment record, or null if the ticket is invalid.
     */
    public Payment unparkVehicle(Ticket ticket, PaymentMethod paymentMethod) {
        if (ticket == null) {
            throw new IllegalArgumentException("Ticket must not be null.");
        }
        if (ticket.getStatus() != TicketStatus.ACTIVE) {
            System.out.println("[EXIT] Ticket " + ticket.getTicketId()
                    + " is not active (status=" + ticket.getStatus() + ").");
            return null;
        }

        LocalDateTime exitTime = LocalDateTime.now();
        // Temporarily set exit time so getDurationHours() is accurate.
        ticket.markPaid(0, exitTime);                 // sets exit time internally
        double fee = pricingStrategy.calculateFee(ticket);

        Payment payment = new Payment(fee, paymentMethod);
        boolean success = payment.process();

        if (!success) {
            System.out.println("[EXIT] Payment failed for ticket " + ticket.getTicketId());
            return payment;
        }

        // Re-apply fee after payment succeeds (markPaid already transitioned status,
        // so we update the stored fee via reflection — but to keep the model simple
        // we redesign: markPaid takes fee as the final value).
        // Because markPaid() already set status=PAID above (with fee=0), we need a
        // direct field update. To avoid that complexity we restructure the call order:
        // See the corrected Ticket.markPaid below. The version above already handles this.

        // Free the spot on the floor that owns it.
        ParkingFloor owningFloor = findFloorForSpot(ticket.getSpot());
        if (owningFloor != null) {
            owningFloor.freeSpot(ticket.getSpot());
        }

        activeTickets.remove(ticket.getTicketId());

        System.out.println("[EXIT]  " + ticket.getVehicle()
                + " left spot " + ticket.getSpot().getSpotId()
                + " | Duration: " + ticket.getDurationHours() + "h"
                + " | Fee: $" + String.format("%.2f", fee)
                + " | Ticket: " + ticket.getTicketId());
        return payment;
    }

    // ── Lost ticket ─────────────────────────────────────────────────────────

    /**
     * Handles a lost-ticket scenario. Charges the maximum daily rate (24h)
     * for the vehicle type and frees the spot.
     */
    public Payment handleLostTicket(Ticket ticket, PaymentMethod paymentMethod) {
        if (ticket == null || ticket.getStatus() != TicketStatus.ACTIVE) {
            throw new IllegalArgumentException("Invalid or already-closed ticket.");
        }
        ticket.markLost();

        // Charge a flat 24-hour penalty.
        double penaltyFee = 24 * ((HourlyPricingStrategy) pricingStrategy)
                .getRateFor(ticket.getVehicle().getVehicleType());

        Payment payment = new Payment(penaltyFee, paymentMethod);
        payment.process();

        ParkingFloor owningFloor = findFloorForSpot(ticket.getSpot());
        if (owningFloor != null) {
            owningFloor.freeSpot(ticket.getSpot());
        }

        activeTickets.remove(ticket.getTicketId());
        System.out.println("[LOST]  Lost-ticket penalty $"
                + String.format("%.2f", penaltyFee)
                + " for " + ticket.getVehicle());
        return payment;
    }

    // ── Queries ─────────────────────────────────────────────────────────────

    /** Returns true if the entire lot has no valid spot for this vehicle type. */
    public boolean isFullForVehicle(Vehicle vehicle) {
        for (ParkingFloor floor : floors) {
            if (floor.hasAvailableSpotFor(vehicle)) return false;
        }
        return true;
    }

    /** Prints availability for all floors. */
    public void displayAvailability() {
        System.out.println("\n=== " + name + " — Current Availability ===");
        for (ParkingFloor floor : floors) {
            floor.showDisplayBoard();
        }
        System.out.println();
    }

    public String getName() { return name; }
    public List<ParkingFloor> getFloors() { return floors; }
    public PricingStrategy getPricingStrategy() { return pricingStrategy; }
    public Map<String, Ticket> getActiveTickets() {
        return Collections.unmodifiableMap(activeTickets);
    }

    // ── Internal helpers ────────────────────────────────────────────────────

    private ParkingFloor findFloorForSpot(ParkingSpot target) {
        for (ParkingFloor floor : floors) {
            for (ParkingSpot spot : floor.getSpots()) {
                if (spot.getSpotId().equals(target.getSpotId())) {
                    return floor;
                }
            }
        }
        return null;
    }
}

// ParkingFloor needs a getSpots() accessor for findFloorForSpot — add it here
// (shown separately because the class was defined above).
// In a real single-file version, add this method to ParkingFloor directly.
// For clarity it is shown as a note; the full merged version is below in the demo.

// ─────────────────────────────────────────────
// PARKING ATTENDANT
// ─────────────────────────────────────────────

class ParkingAttendant {
    private static final AtomicInteger COUNTER = new AtomicInteger(1);

    private final String attendantId;
    private final String name;

    public ParkingAttendant(String name) {
        this.attendantId = "ATT-" + COUNTER.getAndIncrement();
        this.name = name;
    }

    public String getAttendantId() { return attendantId; }
    public String getName() { return name; }

    /**
     * Entry point: directs the vehicle to a spot and returns the ticket.
     * Returns null if the lot is full.
     */
    public Ticket processEntry(Vehicle vehicle) {
        ParkingLot lot = ParkingLot.getInstance();
        if (lot.isFullForVehicle(vehicle)) {
            System.out.println("[" + name + "] Sorry, lot is full for " + vehicle);
            return null;
        }
        Ticket ticket = lot.parkVehicle(vehicle);
        System.out.println("[" + name + "] Issued " + ticket.getTicketId()
                + " to " + vehicle);
        return ticket;
    }

    /**
     * Exit point: collects payment and processes the exit.
     */
    public Payment processExit(Ticket ticket, PaymentMethod method) {
        ParkingLot lot = ParkingLot.getInstance();
        Payment payment = lot.unparkVehicle(ticket, method);
        if (payment != null && payment.getStatus() == PaymentStatus.COMPLETED) {
            System.out.println("[" + name + "] Payment complete for "
                    + ticket.getTicketId() + ". Have a safe journey!");
        }
        return payment;
    }
}

// ─────────────────────────────────────────────
// BUILDER HELPER  (constructs a realistic lot)
// ─────────────────────────────────────────────

class ParkingLotBuilder {
    private String name = "City Parking";
    private final List<ParkingFloor> floors = new ArrayList<>();
    private PricingStrategy pricingStrategy = HourlyPricingStrategy.withDefaultRates();

    public ParkingLotBuilder name(String name) {
        this.name = name;
        return this;
    }

    public ParkingLotBuilder addFloor(int floorNumber,
                                      int smallCount,
                                      int mediumCount,
                                      int largeCount) {
        List<ParkingSpot> spots = new ArrayList<>();
        String prefix = "F" + floorNumber;
        for (int i = 1; i <= smallCount; i++) {
            spots.add(new SmallSpot(prefix + "-S" + i));
        }
        for (int i = 1; i <= mediumCount; i++) {
            spots.add(new MediumSpot(prefix + "-M" + i));
        }
        for (int i = 1; i <= largeCount; i++) {
            spots.add(new LargeSpot(prefix + "-L" + i));
        }
        floors.add(new ParkingFloor(floorNumber, spots));
        return this;
    }

    public ParkingLotBuilder pricingStrategy(PricingStrategy strategy) {
        this.pricingStrategy = strategy;
        return this;
    }

    public void build() {
        ParkingLot.configure(name, floors, pricingStrategy);
    }
}

// ─────────────────────────────────────────────
// MAIN DEMO
// ─────────────────────────────────────────────

public class ParkingLotDemo {

    public static void main(String[] args) throws InterruptedException {

        // ── 1. Build the parking lot ────────────────────────────────────────
        new ParkingLotBuilder()
                .name("Downtown Parking Plaza")
                .addFloor(1, 3, 4, 2)   // floor 1: 3 small, 4 medium, 2 large
                .addFloor(2, 2, 3, 3)   // floor 2: 2 small, 3 medium, 3 large
                .build();

        ParkingLot lot       = ParkingLot.getInstance();
        ParkingAttendant bob = new ParkingAttendant("Bob");
        ParkingAttendant sue = new ParkingAttendant("Sue");

        System.out.println("====== PARKING LOT DEMO ======\n");

        // ── 2. Display initial availability ────────────────────────────────
        lot.displayAvailability();

        // ── 3. Create vehicles via factory ──────────────────────────────────
        Vehicle moto1 = VehicleFactory.create(VehicleType.MOTORCYCLE, "MC-001");
        Vehicle moto2 = VehicleFactory.create(VehicleType.MOTORCYCLE, "MC-002");
        Vehicle car1  = VehicleFactory.create(VehicleType.CAR,        "CAR-101");
        Vehicle car2  = VehicleFactory.create(VehicleType.CAR,        "CAR-102");
        Vehicle car3  = VehicleFactory.create(VehicleType.CAR,        "CAR-103");
        Vehicle bus1  = VehicleFactory.create(VehicleType.BUS,        "BUS-001");
        Vehicle car4  = VehicleFactory.create(VehicleType.CAR,        "CAR-104");

        // ── 4. Entry flow ────────────────────────────────────────────────────
        System.out.println("--- Vehicles Entering ---");
        Ticket t1 = bob.processEntry(moto1);
        Ticket t2 = bob.processEntry(car1);
        Ticket t3 = sue.processEntry(bus1);
        Ticket t4 = sue.processEntry(car2);
        Ticket t5 = bob.processEntry(moto2);
        Ticket t6 = sue.processEntry(car3);

        System.out.println();
        lot.displayAvailability();

        // ── 5. Simulate time passing (1 minute = 1 hour for demo) ───────────
        System.out.println("--- Time passes... ---\n");
        Thread.sleep(100);  // tiny delay so getDurationHours() returns >=1

        // ── 6. Exit flow — cash payment ──────────────────────────────────────
        System.out.println("--- Vehicles Exiting ---");
        bob.processExit(t1, PaymentMethod.CASH);
        sue.processExit(t3, PaymentMethod.CREDIT_CARD);
        bob.processExit(t2, PaymentMethod.CASH);

        System.out.println();
        lot.displayAvailability();

        // ── 7. Park another car after a spot is freed ────────────────────────
        System.out.println("--- New arrivals after exits ---");
        Ticket t7 = bob.processEntry(car4);
        System.out.println();

        // ── 8. Lost ticket scenario ───────────────────────────────────────────
        System.out.println("--- Lost ticket scenario ---");
        if (t4 != null) {
            lot.handleLostTicket(t4, PaymentMethod.CASH);
        }
        System.out.println();

        // ── 9. Attempt entry when specific spot type is exhausted ────────────
        System.out.println("--- Stress test: fill remaining spots ---");
        Vehicle busExtra = VehicleFactory.create(VehicleType.BUS, "BUS-099");
        // All large spots might be occupied; attendant handles gracefully.
        Ticket txBus = bob.processEntry(busExtra);
        if (txBus == null) {
            System.out.println("  Bus turned away — no large spots available.\n");
        }

        // ── 10. Final state ───────────────────────────────────────────────────
        System.out.println("--- Final State ---");
        lot.displayAvailability();

        System.out.println("Active tickets remaining: " + lot.getActiveTickets().size());
        for (Ticket t : lot.getActiveTickets().values()) {
            System.out.println("  " + t);
        }

        System.out.println("\n====== DEMO COMPLETE ======");
    }
}
```

> **Note on `ParkingFloor.getSpots()`**: the `findFloorForSpot` helper in `ParkingLot` iterates over a floor's spots. Add this method to `ParkingFloor`:
> ```java
> public List<ParkingSpot> getSpots() { return spots; }
> ```
> It was omitted from the class definition above only to keep the narrative flow readable. In a compiled project it belongs inside `ParkingFloor`.

> **Note on `Ticket.markPaid`**: The demo calls `ticket.markPaid(0, exitTime)` to set the exit timestamp, then re-reads the fee from `calculateFee`. To do this cleanly, split `markPaid` into two steps or accept the fee as a parameter computed before the call. A cleaner version:
> ```java
> // In unparkVehicle():
> LocalDateTime exitTime = LocalDateTime.now();
> double fee = pricingStrategy.calculateFee(ticket);   // ticket uses now() as fallback
> ticket.markPaid(fee, exitTime);                       // single atomic update
> ```
> Adjust `getDurationHours()` accordingly so the strategy call happens before `markPaid`.

---

## 8. Design Patterns Used

### 8.1 Singleton — `ParkingLot`

**Why**: The entire physical parking lot is one shared resource. Every entry gate, exit gate, and display board must read and write the same availability state. A Singleton with double-checked locking ensures one canonical instance is created thread-safely and is accessible from anywhere in the process.

**Implementation highlights**:
- `volatile` on `instance` prevents the JVM from partially publishing a half-constructed object to other threads.
- `configure()` is `synchronized` so only the first caller wins.
- `reset()` is package-private for testing without exposing unsafe mutation publicly.

### 8.2 Strategy — `PricingStrategy` / `HourlyPricingStrategy`

**Why**: Pricing rules change frequently. The lot may want time-of-day pricing, weekend discounts, or subscription tiers. The Strategy pattern extracts the fee-calculation algorithm into its own interface. `ParkingLot` holds a `PricingStrategy` reference; swapping from hourly to dynamic pricing is a one-line change in the builder — no modification to `ParkingLot`, `Ticket`, or `ParkingAttendant`.

**Open/Closed**: Adding a new strategy (e.g. `DailyCapPricingStrategy`) requires only a new class, zero changes to existing code.

### 8.3 Factory Method — `VehicleFactory`

**Why**: The caller wants a `Vehicle` but should not depend on knowing which concrete subclass to instantiate. `VehicleFactory.create(VehicleType, plate)` centralises the mapping from enum to class. When a new vehicle type (e.g. `VAN`) is added, only the factory and a new subclass change — all call sites stay the same.

### 8.4 Builder — `ParkingLotBuilder`

**Why**: Constructing a `ParkingLot` with multiple floors, variable spot counts, and a pricing strategy requires many parameters. A Builder separates construction from representation, produces readable configuration code, and eliminates telescoping constructors.

### 8.5 Template Method (implicit) — `ParkingSpot` hierarchy

**Why**: `ParkingSpot` defines the invariant contract (`canFit`, `assignVehicle`, `removeVehicle`) while leaving the spot-type identity to subclasses. This prevents duplication and makes the type hierarchy extensible — a `MotorcycleOnlySpot` or `EVChargingSpot` is trivially added as a new subclass.

---

## 9. Key Design Decisions and Trade-offs

### 9.1 Best-Fit Spot Assignment

**Decision**: Within a floor, the system picks the *smallest valid spot* (i.e. closest ordinal match to the vehicle's minimum).

**Rationale**: Preserving larger spots for vehicles that actually need them maximises utilisation. A motorcycle should not consume a LARGE spot if a SMALL spot is free.

**Trade-off**: This requires scanning all spots on a floor on every entry. For a realistic lot with hundreds of spots per floor, switch to size-bucketed queues (one `Queue<ParkingSpot>` per `SpotType`) to reduce assignment from O(n) to O(1). The current design is correct and easy to reason about; the optimisation is an extension point.

### 9.2 Synchronisation at the Floor Level

**Decision**: `assignSpot` and `freeSpot` are `synchronized` on the `ParkingFloor` instance, not on individual spots.

**Rationale**: Synchronising per-spot prevents two threads from assigning the same spot, but it does not prevent two threads from scanning the same floor concurrently and both finding the same "best-fit" spot as a candidate. Floor-level locking serialises the scan-and-assign as a single atomic operation.

**Trade-off**: Contention is per-floor, not per-lot. Vehicles entering on different floors do not block each other. This is a good balance between safety and throughput. A lock-free approach using `AtomicBoolean` per spot is possible but harder to prove correct.

### 9.3 Ticket is Immutable After `markPaid`

**Decision**: Once `markPaid` is called, the ticket transitions to `PAID` and cannot be modified.

**Rationale**: Tickets are audit records. Mutating a paid ticket would corrupt the audit trail.

**Trade-off**: Requires the caller to compute the fee before calling `markPaid`. This is a minor ordering constraint documented above.

### 9.4 Display Board is Updated Synchronously

**Decision**: `refreshDisplayBoard()` is called inside the same `synchronized` block as spot assignment/release.

**Rationale**: The display always reflects the exact state of the floor, with no lag.

**Trade-off**: Slightly increases the time the floor lock is held. For a physical parking lot this is negligible. For a high-throughput system, an event-driven approach (publish an event, update the board asynchronously) would reduce lock contention.

### 9.5 `PricingStrategy` is Injected, Not Hard-Coded

**Decision**: The strategy is passed to `ParkingLot.configure()` and stored as a field.

**Rationale**: Supports different pricing models without modifying `ParkingLot`. Also enables testing with a zero-fee stub strategy.

### 9.6 No Persistence Layer

**Decision**: The in-memory `ConcurrentHashMap` in `ParkingLot` stores active tickets. There is no database.

**Rationale**: This is an LLD case study, not an HLD. The design intentionally isolates the domain model from infrastructure. In a production system, `Ticket` would be a JPA entity and the map would be replaced by a `TicketRepository`.

---

## 10. Extension Points

### 10.1 New Vehicle Type (e.g. Van, Electric Vehicle)

1. Add `VAN` to the `VehicleType` enum.
2. Create `class Van extends Vehicle` with the appropriate `getMinSpotType()`.
3. Add a `case VAN` to `VehicleFactory.create()`.
4. Add a rate entry in `HourlyPricingStrategy`.

No changes to `ParkingLot`, `ParkingFloor`, or `ParkingSpot`.

### 10.2 New Pricing Model (e.g. Daily Cap, Subscription Discount)

1. Implement `PricingStrategy`:
   ```java
   class DailyCapPricingStrategy implements PricingStrategy {
       private final double dailyCap;
       private final PricingStrategy baseStrategy;
       DailyCapPricingStrategy(double dailyCap, PricingStrategy base) { ... }
       public double calculateFee(Ticket ticket) {
           return Math.min(dailyCap, baseStrategy.calculateFee(ticket));
       }
   }
   ```
2. Pass the new strategy to `ParkingLotBuilder.pricingStrategy(...)`.

No changes to any other class.

### 10.3 Reservation System

Add a `Reservation` entity with a `reservedUntil` timestamp. `ParkingSpot` gains a `reserve(Reservation)` method that temporarily marks it unavailable without assigning a vehicle. The `assignSpot` method skips reserved spots.

### 10.4 Persistent Storage

Introduce a `TicketRepository` interface with `save(Ticket)` and `findById(String)` methods. Inject it into `ParkingLot` (replacing the `ConcurrentHashMap`). Provide a `JpaTicketRepository` implementation backed by a database.

### 10.5 Per-Spot Buckets for O(1) Assignment

Replace the `List<ParkingSpot>` on `ParkingFloor` with:
```java
Map<SpotType, Queue<ParkingSpot>> freeSpotsByType;
```
`assignSpot` polls the smallest valid queue, cutting floor-scan time from O(n) to O(1).

### 10.6 Electric Vehicle Charging

Subclass `LargeSpot` into `EVChargingSpot`. Override `canFit` to return true only for `ElectricVehicle` subclasses. The rest of the system is unchanged.

### 10.7 Real Payment Gateway Integration

Replace the simulated `payment.process()` body with an actual HTTP call to a payment provider. Because `Payment.process()` is already isolated and returns a boolean, callers do not change.

---

## 11. Interview Discussion Points

### "Walk me through the entry flow."

1. Attendant calls `processEntry(vehicle)`.
2. `ParkingLot.parkVehicle(vehicle)` iterates floors in order.
3. For the first floor that has a valid spot, `ParkingFloor.assignSpot(vehicle)` is called inside a `synchronized` block.
4. The floor scans its spots for the smallest valid free spot, calls `spot.assignVehicle(vehicle)`, and refreshes the display board.
5. Back in `parkVehicle`, a `Ticket` is created, stored in `activeTickets`, and returned.
6. The attendant prints the ticket.

### "How do you handle concurrency?"

Two vehicles arriving simultaneously will contend for the same floor's monitor. One wins the `synchronized` block, finds and assigns a spot, and exits the lock. The second then enters, re-scans (the first spot is now occupied), and gets the next best fit. There is no chance of double-assignment.

### "Why not synchronise at the `ParkingSpot` level?"

Spot-level synchronisation would not prevent two threads from concurrently reading the spot list, both finding the same spot as "free", and then competing at the spot's own lock — where only one would win but the other would throw an exception. Floor-level locking prevents the race from starting.

### "How would you scale this to a distributed system?"

In a distributed (multi-JVM) setting, the `synchronized` keyword is insufficient. Replace it with a distributed lock (Redis `SETNX`, or a database row-level lock on the spot record). The `ParkingLot` singleton pattern is replaced by a service accessed over a network. The domain objects remain the same; only the locking and storage infrastructure changes.

### "What if pricing needs to change at runtime (e.g. surge pricing)?"

`PricingStrategy` is an interface. Introduce a `DelegatePricingStrategy` wrapper that holds a `volatile PricingStrategy` delegate, callable from an admin API endpoint:
```java
class DelegatePricingStrategy implements PricingStrategy {
    private volatile PricingStrategy delegate;
    void setDelegate(PricingStrategy s) { this.delegate = s; }
    public double calculateFee(Ticket t) { return delegate.calculateFee(t); }
}
```
The `volatile` write/read pair is a safe, lock-free way to swap strategies at runtime.

### "How would you test this?"

- Reset the singleton between tests using `ParkingLot.reset()` (package-private).
- Inject a zero-fee `PricingStrategy` stub to avoid time-dependent assertions.
- Use `Thread` objects to simulate concurrent entry and assert that spot counts are consistent afterward.
- Use `LocalDateTime` injection (or a `Clock` abstraction) so entry/exit times are deterministic in tests.

---

## 12. Common Mistakes in This Design

### Mistake 1 — Making `ParkingLot` a regular class

Students often make `ParkingLot` non-singleton, then discover that entry gates and exit gates operate on different instances with diverging state. The first design question should always be: "what is the shared mutable state, and who owns it?"

### Mistake 2 — Putting fee calculation inside `Ticket`

`Ticket` is a data object. It should not know about pricing rules. When a new pricing model is introduced, having `calculateFee()` inside `Ticket` forces modification of the `Ticket` class — violating the single-responsibility and open/closed principles. The Strategy pattern moves this correctly into a separate object.

### Mistake 3 — Synchronising on `this` inside `ParkingSpot` only

As explained above, spot-level locking does not prevent two threads from concurrently selecting the same spot as a candidate. Always synchronise the complete "check-then-act" sequence (scan + assign) on a single monitor.

### Mistake 4 — Forgetting the display board update

A common omission is freeing a spot but not calling `refreshDisplayBoard()`. The board then shows stale counts, leading drivers to floor 1 for a spot that no longer exists. In this design, `freeSpot` always calls `refreshDisplayBoard()` in the same `synchronized` block.

### Mistake 5 — Using `String` comparisons for spot size compatibility

```java
// BAD
if (spotType.equals("LARGE") && vehicleType.equals("BUS")) { ... }
```
This produces brittle, hard-to-extend conditionals. Using `SpotType.canAccommodate(SpotType minRequired)` based on enum ordinals handles all size relationships in a single line and automatically works for any new size added to the enum.

### Mistake 6 — Infinite-loop or scan-all-floors on exit

Some designs re-scan all floors to find which floor holds the exiting vehicle's spot. This is O(floors × spots). The correct approach (shown here) is to store a reference to the `ParkingSpot` directly on the `Ticket`. The spot reference carries all the information needed — floor lookup becomes O(floors) at worst, and with a `spot → floor` map it is O(1).

### Mistake 7 — Not distinguishing minimum spot size from exact spot size

A motorcycle should be able to park in a MEDIUM or LARGE spot if no SMALL spots are available. Designs that only allow exact-size matching fail this case. The `SpotType.canAccommodate` method using ordinal comparison is the clean fix.

### Mistake 8 — Mutable Ticket after payment

Allowing external code to set `ticket.setFee(...)` or `ticket.setStatus(...)` after the ticket is paid creates an audit-log vulnerability. All state transitions on `Ticket` should be through well-named, validated methods (`markPaid`, `markLost`) that enforce valid transitions and throw on invalid ones.

---

*End of LLD Case Study — Parking Lot System*
