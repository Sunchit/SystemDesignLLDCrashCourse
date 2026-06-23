# Module 07 — Machine Coding Round Preparation

> A machine coding round is not a harder coding interview. It is a different kind of interview. You are not being asked to solve a puzzle — you are being asked to show how you work.

---

## Introduction

The machine coding round is the LLD interview in execution form. Where a whiteboard design round tests whether you can reason about a system, a machine coding round tests whether you can build one — in real time, under pressure, with an interviewer watching.

The distinguishing feature of this round is that nothing you produce is theoretical. The code runs. The classes exist. The interviewer can call your methods. When you say "the parking lot can have multiple floors," they can ask you to add a second floor and watch what changes. When you say "payment is extensible," they can ask you to add UPI support in ten minutes.

This is harder than a whiteboard, not because the problems are harder, but because the gap between design intention and working code is exposed. A lot of engineers who can sketch a clean class diagram freeze when they have to make the actual design decisions — which constructor to call, where the null check lives, what the method signature actually looks like when you write it.

This module prepares you for that. You will learn how to structure your time, how to make design decisions at speed, what interviewers actually score, and how to approach each of the 20 most common machine coding problems.

---

## Section 1 — What is a Machine Coding Round?

### Format

A machine coding round typically runs **60 to 90 minutes**. You work on your own laptop with your own IDE, your own keyboard shortcuts, and access to documentation (Stack Overflow, Javadoc, and similar resources are generally permitted — just not pre-written solutions).

The output is a working program. Not a design document. Not pseudocode. Runnable code that an interviewer can inspect, ask about, and extend.

Some companies run the round in a shared coding environment like CoderPad or a shared screen. Others let you work entirely locally and share your screen. Either way, the interviewer can see your screen. They are watching how you think, not just what you produce.

**Key constraints**:
- No internet access to pre-built solutions or GitHub repositories
- You must explain every design decision when asked
- The code must compile and run
- You are expected to handle the core use case end-to-end

### What Interviewers Evaluate

Interviewers in a machine coding round are not just checking whether the code works. They are running a mental rubric against everything they see. Here is what that rubric looks like, in detail:

**1. Requirement Understanding (10 points)**
- Did you ask the right clarifying questions before writing a line?
- Did you correctly identify what is in scope vs. out of scope?
- Did you make reasonable assumptions and state them explicitly?
- Did you avoid building things that weren't asked for?

Poor engineers start coding immediately. Average engineers ask one or two surface-level questions. Strong engineers spend 5-10 minutes with the interviewer walking through requirements, explicit assumptions, and scope boundaries.

**2. Design Quality (25 points)**
- Are the core entities correctly identified?
- Are responsibilities correctly distributed across classes?
- Is the class hierarchy clean — no god classes, no anemic models?
- Are interfaces used where behavior varies?
- Are design patterns applied where they reduce complexity?
- Is the design open for the extensions the interviewer is likely to ask?

**3. Code Quality (20 points)**
- Are names clear and accurate — classes, methods, variables?
- Are methods short and focused on a single responsibility?
- Is the code readable without running it?
- Are there magic numbers, hard-coded strings, or other code smells?
- Is the code organized in a logical, navigable structure?

**4. Correctness (20 points)**
- Does the core use case work?
- Does it handle edge cases (null inputs, empty collections, boundary conditions)?
- Is the logic correct for the primary scenarios?
- Are there any obvious bugs or off-by-one errors?

**5. Object-Oriented Design (15 points)**
- Encapsulation: is internal state protected?
- Abstraction: are implementation details hidden behind clean interfaces?
- Inheritance/Composition: is the hierarchy appropriate?
- Polymorphism: does the code exploit polymorphism where it reduces conditionals?

**6. Extensibility (10 points)**
- Can you add a new vehicle type without modifying existing vehicle code?
- Can you add a new payment method without touching the booking flow?
- Is the design closed for modification but open for extension?
- When asked "what if we needed X?", can you trace exactly where X would go without touching four other things?

**Total: 100 points. Passing threshold at most companies: 65-75 points.**

Interviewers do not always score mechanically — but they do hold all of these dimensions in their head. An engineer who produces beautiful OOP but a non-working program will fail. An engineer who produces working code with a 400-line god class will also fail. You need both.

### Companies That Use Machine Coding Rounds

Machine coding rounds are standard at most Indian product companies and are increasingly common globally:

- **Flipkart**: One of the originators of the format. 90 minutes, strict rubric, usually parking lot, cab booking, or chess.
- **Swiggy**: 60 minutes, real-world delivery-domain problems. Food delivery, order management.
- **Dunzo**: Strong focus on extensibility. They will ask for extensions during the round.
- **CRED**: Design quality weighted heavily. Clean code and OOP are explicitly scored.
- **Razorpay**: Payment domain problems. Rate limiters, transaction flows, fraud detection.
- **PhonePe**: Financial system problems, high emphasis on edge case handling.
- **Meesho**: Order and inventory management.
- **Zepto**: Inventory and delivery-slot management.
- **Groww**: Portfolio, transaction, and financial instrument modeling.
- **Nykaa**: Product catalog, cart, and order systems.
- **Zomato**: Food delivery, restaurant management, rider assignment.
- **Urban Company**: Service booking, provider management, availability slots.
- **Amazon (India)**: Machine coding is part of the SDE-2 bar. Order management and recommendation systems.
- **Google (some teams)**: Used for SWE L4/L5 at specific offices, less standardized globally.

---

## Section 2 — The RADIO Framework for Machine Coding

The RADIO framework is a five-step mental model for executing a machine coding round. It is not a rigid process — it is a checklist of concerns to address before you consider yourself done with each phase. Engineers who skip steps fail. Engineers who execute all five in order consistently produce strong outputs.

**R — Requirements**
**A — Abstractions**
**D — Design**
**I — Implementation**
**O — Open for extension**

Work through these in order. Do not start writing code until you have finished R and A. Do not move to I without a clear D. Reserve time for O.

---

### R — Requirements Clarification

The first thing you do in a machine coding round is not open your IDE. It is talk to the interviewer.

Requirements clarification serves two purposes. First, it ensures you build the right thing. A parking lot problem can mean a simple single-floor lot with three vehicle types, or it can mean a multi-floor, multi-entry-point lot with a dynamic pricing engine. Building the wrong system is a hard failure regardless of code quality.

Second, it signals to the interviewer that you think before you code. This is a strong positive signal. Interviewers take note of engineers who ask clear, structured questions.

**What to ask**:

1. **Scale and scope**: "Are we designing for a single parking lot or a chain? Single floor or multiple? What vehicle types should I support?"
2. **Core use cases**: "What are the three or four operations this system must perform? Park a vehicle, unpark, calculate fee — what else?"
3. **Constraints and rules**: "Is pricing by duration only, or also by vehicle type? Are there reserved spots? Is a spot exclusive to one vehicle type?"
4. **What to exclude**: "Should I handle payment processing, or just fee calculation? Do I need persistence? User authentication?"
5. **Extension hints**: "Are there features you are likely to ask me to extend in the second half?" (Some interviewers will tell you. This is valuable.)

**What to assume when the interviewer says "use your judgment"**:

- In-memory data storage (no database, no file system)
- Single-threaded execution unless concurrency is explicitly relevant
- Input validation is necessary but not the focus
- The simplest reasonable interpretation of an ambiguous requirement
- Standard vehicle types (motorcycle, car, truck) unless otherwise specified

Always state your assumptions out loud: "I'm going to assume this is a single-floor parking lot with three vehicle types and duration-based pricing. I'll ignore payment processing. Does that work for you?"

**What not to do**:
- Do not ask questions the problem statement already answered
- Do not ask questions to stall while you think — state that you are thinking instead
- Do not ask so many questions that the interviewer loses patience
- Do not skip requirements and assume you know what they want

---

### A — Abstractions

Before you open your IDE, identify the core entities and their relationships on paper or in your head. This is the step that prevents you from building a god class called `ParkingLotManager` that does everything.

Abstractions emerge from the requirements. The process is:

1. Extract the nouns from your requirements — these are candidate entities
2. Extract the verbs — these are candidate behaviors (methods)
3. Group behaviors with the entities they logically belong to
4. Identify which entities vary (these need interfaces or abstract classes)
5. Identify which entities are composed of other entities

**Example: Parking Lot**

Requirements: "Design a parking lot that can park vehicles of type motorcycle, car, and truck. Each floor has spots of different sizes. Calculate fee based on duration."

Nouns: ParkingLot, Floor, ParkingSpot, Vehicle, Motorcycle, Car, Truck, Ticket, Fee
Verbs: park, unpark, findSpot, calculateFee, issueTicket, generateReceipt

Grouping behaviors:
- ParkingLot: park, unpark, findAvailableSpot
- Floor: getAvailableSpots, getSpotByNumber
- ParkingSpot: park, unpark, isAvailable, canFitVehicle
- FeeCalculator: calculateFee (varies by strategy — this is the interface)
- Vehicle: getType, getLicensePlate (varies by subtype)

Entities that vary: Vehicle (by type), FeeCalculator (by pricing strategy), ParkingSpot (by size)

Relationships:
- ParkingLot has many Floors (composition)
- Floor has many ParkingSpots (composition)
- ParkingSpot holds at most one Vehicle (association)
- Ticket references a Vehicle and a ParkingSpot (association)
- ParkingLot uses a FeeCalculator (dependency injection)

This mapping takes five minutes. It prevents thirty minutes of refactoring later.

---

### D — Design

With abstractions identified, produce a concrete design: key interfaces, key classes, primary method signatures. You do not need a complete UML diagram — you need enough clarity to code without stopping to make structural decisions.

**Key interfaces to define first**:

```java
// Every vehicle type implements this
public interface Vehicle {
    VehicleType getType();
    String getLicensePlate();
}

// Every spot size strategy implements this
public interface ParkingSpot {
    boolean isAvailable();
    boolean canFit(Vehicle vehicle);
    void park(Vehicle vehicle);
    void unpark();
    SpotSize getSize();
}

// Fee calculation is pluggable
public interface FeeCalculator {
    double calculateFee(Ticket ticket);
}
```

**Key classes to sketch**:

```java
public class ParkingLot {
    private final List<Floor> floors;
    private final FeeCalculator feeCalculator;
    private final Map<String, Ticket> activeTickets;

    public Ticket park(Vehicle vehicle) { ... }
    public Receipt unpark(String ticketId) { ... }
    private Optional<ParkingSpot> findAvailableSpot(Vehicle vehicle) { ... }
}

public class Ticket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final LocalDateTime entryTime;
    // no setters except for exit time
}
```

At this point you have answered the critical design questions:
- Where does parking logic live? (ParkingLot, delegating to Floor)
- Where does fee logic live? (FeeCalculator interface, injected)
- Where does spot-fit logic live? (ParkingSpot.canFit)
- What is immutable? (Ticket: once issued, entry time is fixed)

This takes another five minutes. You now have a design contract you can implement incrementally.

---

### I — Implementation

Implementation is where the code gets written. The guiding principle is: **get something working first, then improve it**.

This means:
1. Implement the happy path for the most important use case first
2. Make it compilable and runnable as quickly as possible
3. Then add edge cases, error handling, and secondary use cases
4. Refactor for clarity as you go — but do not refactor at the expense of coverage

**The incremental approach in practice**:

First pass — make parking work:

```java
public class ParkingLot {
    private final List<Floor> floors;
    private final FeeCalculator feeCalculator;
    private final Map<String, Ticket> activeTickets = new HashMap<>();

    public ParkingLot(List<Floor> floors, FeeCalculator feeCalculator) {
        this.floors = floors;
        this.feeCalculator = feeCalculator;
    }

    public Ticket park(Vehicle vehicle) {
        ParkingSpot spot = findAvailableSpot(vehicle)
            .orElseThrow(() -> new ParkingLotFullException("No spot available for " + vehicle.getType()));

        spot.park(vehicle);
        Ticket ticket = new Ticket(generateTicketId(), vehicle, spot, LocalDateTime.now());
        activeTickets.put(ticket.getTicketId(), ticket);
        return ticket;
    }

    public Receipt unpark(String ticketId) {
        Ticket ticket = activeTickets.remove(ticketId);
        if (ticket == null) {
            throw new InvalidTicketException("Ticket not found: " + ticketId);
        }
        ticket.setExitTime(LocalDateTime.now());
        ticket.getSpot().unpark();
        double fee = feeCalculator.calculateFee(ticket);
        return new Receipt(ticket, fee);
    }

    private Optional<ParkingSpot> findAvailableSpot(Vehicle vehicle) {
        return floors.stream()
            .flatMap(floor -> floor.getSpots().stream())
            .filter(spot -> spot.isAvailable() && spot.canFit(vehicle))
            .findFirst();
    }

    private String generateTicketId() {
        return UUID.randomUUID().toString();
    }
}
```

Second pass — implement FeeCalculator:

```java
public class HourlyFeeCalculator implements FeeCalculator {
    private final Map<VehicleType, Double> hourlyRates;

    public HourlyFeeCalculator(Map<VehicleType, Double> hourlyRates) {
        this.hourlyRates = hourlyRates;
    }

    @Override
    public double calculateFee(Ticket ticket) {
        long hours = ChronoUnit.HOURS.between(ticket.getEntryTime(), ticket.getExitTime());
        long billableHours = Math.max(1, hours); // minimum 1 hour
        double ratePerHour = hourlyRates.getOrDefault(ticket.getVehicle().getType(), 10.0);
        return billableHours * ratePerHour;
    }
}
```

Third pass — implement vehicle hierarchy:

```java
public enum VehicleType { MOTORCYCLE, CAR, TRUCK }

public abstract class BaseVehicle implements Vehicle {
    private final String licensePlate;
    private final VehicleType type;

    protected BaseVehicle(String licensePlate, VehicleType type) {
        this.licensePlate = Objects.requireNonNull(licensePlate, "License plate cannot be null");
        this.type = type;
    }

    @Override public String getLicensePlate() { return licensePlate; }
    @Override public VehicleType getType() { return type; }
}

public class Car extends BaseVehicle {
    public Car(String licensePlate) { super(licensePlate, VehicleType.CAR); }
}
```

This incremental approach means that at any point during the interview, you have working code. If the interviewer stops you at 45 minutes, you have a partial but functional system — not 45 minutes of uncompilable scaffolding.

---

### O — Open for Extension

In the last 10-15 minutes, demonstrate that your design is open for extension. This is not about adding features — it is about showing that adding features would be cheap.

Do this in two ways:

**1. Verbal walkthrough**

"If you asked me to add a dynamic pricing strategy — charge more during peak hours — I would create a `PeakHourFeeCalculator` that implements `FeeCalculator`. Nothing in `ParkingLot` changes. Nothing in `Ticket` changes. I inject the new calculator at construction time."

**2. Brief code sketch**

```java
// Adding surge pricing requires zero changes to existing code
public class SurgePricingCalculator implements FeeCalculator {
    private final FeeCalculator baseFeeCalculator;
    private final SurgeTimeProvider surgeTimeProvider;
    private final double surgeMultiplier;

    public SurgePricingCalculator(FeeCalculator baseFeeCalculator,
                                   SurgeTimeProvider surgeTimeProvider,
                                   double surgeMultiplier) {
        this.baseFeeCalculator = baseFeeCalculator;
        this.surgeTimeProvider = surgeTimeProvider;
        this.surgeMultiplier = surgeMultiplier;
    }

    @Override
    public double calculateFee(Ticket ticket) {
        double baseFee = baseFeeCalculator.calculateFee(ticket);
        return surgeTimeProvider.isSurgeTime(ticket.getEntryTime())
            ? baseFee * surgeMultiplier
            : baseFee;
    }
}
```

This is the Decorator pattern applied to fee calculation. The interviewer sees that you understand extensibility as a design property, not just a word.

Extension demonstrations to prepare for common problems:
- Parking lot: new vehicle type, new fee strategy, new spot allocation strategy
- Chess: new piece type, new game variant (960/Fischer)
- Order management: new order status, new discount rule, new fulfillment channel
- Chat application: new message type, new notification channel

---

## Section 3 — Time Management

Time management in a machine coding round is not optional. Engineers who run out of time with half a system implemented fail, regardless of how good the half is. Here is a precise time budget for a 90-minute round.

### First 10 Minutes: Requirements + High-Level Design

Do not open your IDE. Sit at the keyboard and talk.

- **Minutes 0-5**: Requirements clarification. Ask your questions. State your assumptions. Get alignment.
- **Minutes 5-8**: Entity identification. Name your core entities, their responsibilities, and key relationships out loud.
- **Minutes 8-10**: Interface skeleton. Identify the two or three interfaces that will anchor your design. Name the key classes.

At the 10-minute mark, the interviewer should know: what you are building, what the core entities are, and what interfaces you will define. You have not written a line of Java.

### Next 60 Minutes: Implementation — Priority Order

Not all code is equal. Implement in this order:

**Priority 1 (Minutes 10-30): Core domain model**
- Core entities with their fields and basic methods
- The interfaces that will hold the design together
- The primary happy-path use case, end to end

At the 30-minute mark, you should have the primary use case working.

**Priority 2 (Minutes 30-50): Secondary use cases + error handling**
- The second most important use case (e.g., unparking after parking)
- Null checks and guard clauses on public methods
- Exception types for failure scenarios
- Edge cases that are obvious from the requirements

At the 50-minute mark, the system should handle all stated requirements.

**Priority 3 (Minutes 50-70): Polish + secondary features**
- Clean up naming and method structure
- Add any secondary features that were in scope but deferred
- Add a simple main() or test method that demonstrates the system working end-to-end

At the 70-minute mark, you should have a complete, demonstrable system.

### Last 10-20 Minutes: Refactor + Extensibility

- Review for obvious code smells: long methods, repeated logic, bad names
- Demonstrate one extension verbally and/or in code
- Walk the interviewer through the design at a high level
- Invite questions about design decisions

### What to Do If Running Out of Time

If you hit the 60-minute mark and are not done, stop and triage:

1. **What is missing?** Make a list. Tell the interviewer: "I have X and Y complete. Z is not implemented — here is where it would go."
2. **Add stubs for the missing pieces.** A method with a clear signature and a `// TODO: implement fee calculation` comment is better than no method.
3. **Demonstrate what works.** Run the happy path. Show the interviewer the output.
4. **Do not panic-code.** Code written in the last five minutes under panic usually introduces bugs and makes the existing code worse. Stop, assess, and communicate.

Never sacrifice design quality for coverage. A well-designed incomplete system scores better than a complete system with a 400-line god class.

---

## Section 4 — The Incremental Design Strategy

The single most important tactical advice for machine coding rounds: **start with a working MVP, then layer in complexity**.

This sounds obvious. In practice, engineers do the opposite. They try to design the complete, correct system upfront — every interface, every edge case, every design pattern — and then start coding. Forty-five minutes in, they have a beautiful half-implemented system that does not compile.

### The MVP First Principle

Your first implementation target is the smallest possible system that performs the core function end-to-end. For a parking lot:

```
MVP: Park a car. Retrieve the car. Calculate a flat fee.
```

That's it. No floors. No vehicle types. No dynamic pricing. No reservations. A `ParkingLot` with a fixed number of spots, one vehicle type, one fee. Code that works.

The moment you have working MVP code, you are safe. You can extend from there. You can get interrupted. You have something to demonstrate.

### Layering In Complexity

After MVP, add complexity in order of:
1. **Correctness requirements**: Multiple vehicle types, multiple floors, fee by duration
2. **Robustness requirements**: Edge cases, error handling, validation
3. **Design quality improvements**: Extract interfaces, apply patterns, improve names
4. **Extension demonstrations**: Show what adding a new feature would look like

Each layer should produce runnable code. Do not start layer 2 until layer 1 compiles and runs.

### Never Get Stuck on Perfect Design

The most dangerous trap in a machine coding round is the pursuit of the perfect design before any code is written.

Signs you are in this trap:
- You have been thinking for 20 minutes and written no code
- You are debating between two class hierarchies that are both fine
- You are adding interfaces "just in case"
- You have a design for features that were not in the requirements

The rule: **make a decision and move**. If you are debating whether `ParkingSpot` should be an interface or an abstract class, pick one, write a comment noting the tradeoff, and implement. If the interviewer objects to your choice, you can discuss it. But you cannot be evaluated on code you never wrote.

Good design is not about finding the one correct answer. It is about making defensible decisions quickly. Two reasonable designs are often equivalent in quality. Speed of decision-making under uncertainty is itself a senior-engineer skill.

---

## Section 5 — Common Evaluation Rubrics

Here is each evaluation dimension explained in depth, with what strong and weak performance looks like.

### Correctness: Does It Work?

**What it means**: The code compiles without errors. The primary use cases execute without exceptions. The output is what the requirements specified.

**Strong performance**:
- Core use case works end to end
- Secondary use cases work
- Edge cases are handled without crashes (empty lot, double-unpark attempt, invalid ticket ID)
- A `main()` method or demonstration class shows it working

**Weak performance**:
- Code does not compile
- NullPointerExceptions on normal inputs
- Incorrect output (wrong fee calculation, wrong spot assignment)
- Features claimed in design that do not exist in code

**Common correctness bugs**:
```java
// Bug: not removing the vehicle from the spot on unpark
public Receipt unpark(String ticketId) {
    Ticket ticket = activeTickets.remove(ticketId);
    // Missing: ticket.getSpot().unpark();
    double fee = feeCalculator.calculateFee(ticket);
    return new Receipt(ticket, fee);
}

// Bug: integer division truncating hours
long hours = (exitTime.toEpochSecond() - entryTime.toEpochSecond()) / 3600; // correct
// vs.
int hours = (int)((exitTime.toEpochSecond() - entryTime.toEpochSecond()) / 3600); // may truncate
```

---

### Code Quality: Is It Clean?

**What it means**: The code is readable. Names are accurate and consistent. Methods are focused. There are no obvious code smells.

**Strong performance**:
```java
// Clear method name, single responsibility, no magic numbers
private static final int MINIMUM_BILLABLE_HOURS = 1;

private long calculateBillableHours(Ticket ticket) {
    long actualHours = ChronoUnit.HOURS.between(ticket.getEntryTime(), ticket.getExitTime());
    return Math.max(actualHours, MINIMUM_BILLABLE_HOURS);
}
```

**Weak performance**:
```java
// Magic number, unclear variable names, multiple responsibilities in one method
public double calc(Ticket t) {
    long h = (t.getEx().toEpochSecond() - t.getEn().toEpochSecond()) / 3600;
    if (h < 1) h = 1;
    double f = 0;
    if (t.getV().getType() == 0) f = h * 20;
    else if (t.getV().getType() == 1) f = h * 40;
    else f = h * 60;
    return f;
}
```

The second version works. It scores poorly. Interviewers see it as a signal that you write code you cannot maintain.

**Key naming rules for machine coding**:
- Classes are nouns: `ParkingLot`, `Ticket`, `FeeCalculator`
- Methods are verbs: `park()`, `calculateFee()`, `findAvailableSpot()`
- Booleans describe a state: `isAvailable()`, `canFit()`, `hasCapacity()`
- No abbreviations: `tkt` instead of `ticket` is unacceptable
- No generic names: `Manager`, `Handler`, `Processor`, `Helper` are red flags unless justified

---

### OOP Design: Are the Abstractions Right?

**What it means**: Classes have single responsibilities. The hierarchy is correct. Interfaces are used where behavior varies. There are no god classes.

**Strong performance — Strategy pattern for fee calculation**:
```java
// FeeCalculator is an interface because pricing strategy varies
public interface FeeCalculator {
    double calculateFee(Ticket ticket);
}

// Each strategy is a separate class
public class HourlyFeeCalculator implements FeeCalculator { ... }
public class DailyCapFeeCalculator implements FeeCalculator { ... }
public class FlatRateFeeCalculator implements FeeCalculator { ... }

// ParkingLot does not know which strategy is being used
public class ParkingLot {
    private final FeeCalculator feeCalculator; // injected

    public ParkingLot(List<Floor> floors, FeeCalculator feeCalculator) {
        this.feeCalculator = feeCalculator;
    }
}
```

**Weak performance — hardcoded conditional**:
```java
public class ParkingLot {
    public double calculateFee(Ticket ticket, String pricingType) {
        if (pricingType.equals("hourly")) {
            // hourly logic
        } else if (pricingType.equals("daily")) {
            // daily logic
        }
        // adding a new pricing type requires modifying this method
    }
}
```

The first version passes OCP. The second fails it. The interviewer knows the difference.

**OOP red flags**:
- Any class over 200 lines is suspect
- Any method over 30 lines is suspect
- `instanceof` checks in core logic usually indicate missing polymorphism
- Public fields (except for simple value objects)
- Static state that is not truly global

---

### Extensibility: Can It Be Extended?

**What it means**: Adding a new variant of an existing concept requires writing new code, not modifying existing code.

**The extension test**: For any entity that has types (vehicle types, spot types, fee strategies, order statuses), adding a new type should require:
1. Creating a new class or enum value
2. Registering it (if necessary — factory or config)
3. Zero changes to existing logic classes

**Strong extensibility — adding truck support**:

```java
// Step 1: Create the new vehicle class
public class Truck extends BaseVehicle {
    public Truck(String licensePlate) {
        super(licensePlate, VehicleType.TRUCK);
    }
}

// Step 2: Add to the enum
public enum VehicleType { MOTORCYCLE, CAR, TRUCK }

// Step 3: Spot compatibility is handled in LargeSpot
public class LargeSpot extends BaseParkingSpot {
    @Override
    public boolean canFit(Vehicle vehicle) {
        return true; // large spots fit all vehicle types
    }
}

// No changes to ParkingLot, Floor, Ticket, or FeeCalculator
```

**Weak extensibility — requiring modification**:
```java
// Adding truck requires editing this switch in ParkingLot
private boolean canVehicleFitInSpot(Vehicle vehicle, SpotType spotType) {
    switch (vehicle.getType()) {
        case MOTORCYCLE: return spotType == SpotType.SMALL || spotType == SpotType.MEDIUM || spotType == SpotType.LARGE;
        case CAR: return spotType == SpotType.MEDIUM || spotType == SpotType.LARGE;
        // Adding TRUCK requires editing this switch
    }
    return false;
}
```

Move this logic to the `ParkingSpot.canFit(Vehicle)` method and the problem disappears.

---

### Testability: Could You Unit Test It?

**What it means**: The key classes can be tested in isolation. Dependencies are injected, not created internally. Pure functions are separated from I/O.

**Testable design**:
```java
// FeeCalculator can be tested without a ParkingLot
@Test
public void calculateFee_minimumOneHour_whenParkedLessThanOneHour() {
    FeeCalculator calculator = new HourlyFeeCalculator(Map.of(VehicleType.CAR, 40.0));
    Ticket ticket = createTicket(VehicleType.CAR,
        LocalDateTime.of(2024, 1, 1, 10, 0),
        LocalDateTime.of(2024, 1, 1, 10, 30)); // 30 minutes

    double fee = calculator.calculateFee(ticket);

    assertEquals(40.0, fee, 0.001); // minimum 1 hour = 40.0
}
```

**Untestable design**:
```java
public class ParkingLot {
    public double calculateFee(String ticketId) {
        // fee calculation is buried inside ParkingLot
        // requires a full ParkingLot instance to test
        // requires a ticket to have been issued by this lot
    }
}
```

Signs of testable design:
- Dependencies are constructor-injected
- Core algorithms are in pure methods with no side effects
- Enums and value objects are used instead of strings for type-safe parameters
- Exception types are specific and documented

---

### Error Handling: Does It Handle Edge Cases?

**What it means**: The system does not crash on invalid input. Errors are communicated clearly. Illegal states are prevented.

**Strong error handling**:
```java
public Ticket park(Vehicle vehicle) {
    Objects.requireNonNull(vehicle, "Vehicle cannot be null");
    Objects.requireNonNull(vehicle.getLicensePlate(), "License plate cannot be null");

    if (activeTickets.values().stream()
            .anyMatch(t -> t.getVehicle().getLicensePlate().equals(vehicle.getLicensePlate()))) {
        throw new VehicleAlreadyParkedException(
            "Vehicle " + vehicle.getLicensePlate() + " is already parked");
    }

    return findAvailableSpot(vehicle)
        .map(spot -> {
            spot.park(vehicle);
            Ticket ticket = new Ticket(generateTicketId(), vehicle, spot, LocalDateTime.now());
            activeTickets.put(ticket.getTicketId(), ticket);
            return ticket;
        })
        .orElseThrow(() -> new ParkingLotFullException(
            "No available spot for vehicle type: " + vehicle.getType()));
}
```

**Define custom exception types**:
```java
public class ParkingLotFullException extends RuntimeException {
    public ParkingLotFullException(String message) { super(message); }
}

public class InvalidTicketException extends RuntimeException {
    public InvalidTicketException(String message) { super(message); }
}

public class VehicleAlreadyParkedException extends RuntimeException {
    public VehicleAlreadyParkedException(String message) { super(message); }
}
```

Custom exceptions serve two purposes: they communicate intent clearly, and they allow callers to handle specific failure modes without catching `Exception`.

**Edge cases to always consider**:
- Null inputs on every public method
- Empty collections (empty parking lot, no available spots)
- Duplicate operations (park the same vehicle twice, unpark a vehicle not parked here)
- Boundary conditions (lot at exactly full capacity, fee for zero duration)
- Invalid IDs (ticket ID that does not exist)

---

## Section 6 — 20 Machine Coding Questions

These are the 20 problems that appear most frequently in machine coding rounds at Indian and global product companies. Every problem is listed with its core complexity (what makes it genuinely hard), the key design pattern to apply, an estimated completion time for a well-prepared engineer, and a difficulty level.

Practice all 20. For each, do the following before looking at any solution:
1. Spend 5 minutes on requirements clarification (ask questions out loud)
2. Identify core entities and their relationships
3. Sketch the key interfaces
4. Set a timer and implement

| # | Problem | Core Complexity / Challenge | Key Design Pattern | Time Estimate | Difficulty |
|---|---|---|---|---|---|
| 1 | **Parking Lot** | Matching vehicle types to spot sizes across multiple floors; clean fee calculation abstraction | Strategy (fee), Composite (lot/floor/spot) | 60 min | Easy |
| 2 | **Snake Game** | Collision detection; growing body representation; clean game loop separation from rendering | Observer (game events), State (game state) | 75 min | Medium |
| 3 | **Cricket Scoreboard** | Tracking ball-by-ball state across overs, wickets, partnerships; multiple views of the same data | Observer (scoreboard update), Builder (match setup) | 60 min | Medium |
| 4 | **File System** | Recursive tree structure where files and directories share an interface; path resolution | Composite (file/directory), Iterator (tree traversal) | 70 min | Medium |
| 5 | **Elevator System** | Multi-elevator coordination; request scheduling; direction and floor state management | State (elevator direction), Strategy (scheduling algorithm) | 90 min | Hard |
| 6 | **Chess Game** | 16 piece types with distinct movement rules; turn management; check detection; immutable board state | Strategy (move validation per piece), Command (move history for undo) | 90 min | Hard |
| 7 | **Online Judge** | Submission lifecycle (queued, running, judged); multiple language support; test case execution isolation | State (submission status), Strategy (per-language executor), Observer (result notification) | 90 min | Hard |
| 8 | **Task Scheduler** | Priority queues; recurring tasks; dependency resolution between tasks | Strategy (scheduling policy), Observer (completion notification), Builder (task definition) | 75 min | Medium |
| 9 | **Order Management System** | Order state machine (placed, confirmed, shipped, delivered, cancelled); multiple actor types | State (order lifecycle), Observer (status notification), Strategy (fulfillment channel) | 75 min | Medium |
| 10 | **Social Media Feed** | Post creation, follow graph, feed generation; ranking/sorting strategy; pagination | Observer (feed update on post), Strategy (feed ranking), Iterator (paginated feed) | 80 min | Medium |
| 11 | **Hotel Management** | Room type matrix; booking conflict detection; check-in/check-out lifecycle; pricing by room class | Strategy (pricing), State (room status), Composite (hotel/floor/room) | 75 min | Medium |
| 12 | **Cab Booking** | Driver-rider matching; ride state machine; fare calculation with surge; real-time driver location | State (ride lifecycle), Strategy (matching algorithm + fare), Observer (driver location update) | 90 min | Hard |
| 13 | **Movie Ticket Booking** | Seat locking during selection; concurrent booking prevention; show-seat-theater hierarchy | Composite (theater/screen/seat), State (seat status), Strategy (seat selection) | 70 min | Medium |
| 14 | **Inventory System** | Stock level tracking across warehouses; reorder triggers; reservation vs. deduction | Observer (low-stock alerts), Strategy (reorder policy), State (inventory reservation) | 65 min | Easy |
| 15 | **Chat Application** | User-to-user and group messaging; message delivery states; notification fan-out | Observer (message delivery), Mediator (group chat coordination), State (message status) | 80 min | Medium |
| 16 | **Tic Tac Toe / Connect 4** | Generalized win-condition check for any board size; player abstraction for human vs. AI | Strategy (win detection, player move), Template Method (game loop) | 45 min | Easy |
| 17 | **ATM Machine** | State-driven transaction flow (idle, card inserted, PIN verified, transaction, dispensing); concurrent access safety | State (ATM transaction state), Command (transaction types), Chain of Responsibility (auth steps) | 80 min | Medium |
| 18 | **Food Delivery** | Restaurant-order-rider three-way coordination; delivery state machine; ETA calculation | State (delivery lifecycle), Observer (status updates), Strategy (rider assignment) | 90 min | Hard |
| 19 | **Cache with LRU Eviction** | O(1) get and put with O(1) eviction; doubly linked list + hashmap combination; thread-safety concerns | Decorator (layered cache), Iterator (eviction order) | 50 min | Easy |
| 20 | **Rate Limiter** | Multiple algorithms (token bucket, sliding window, fixed window); per-client limit enforcement; thread-safe counters | Strategy (rate limiting algorithm), Decorator (layered limiting), Flyweight (per-client state) | 60 min | Medium |

### Notes on the Table

**Time estimates** are for a well-prepared engineer who has practiced similar problems. On your first attempt at a new problem, add 20-30 minutes.

**Difficulty levels** are relative to machine coding round standards:
- Easy: Core complexity is moderate, clean design is achievable in 60 minutes
- Medium: Requires a non-trivial design decision or a well-known pattern applied correctly
- Hard: Multiple interacting subsystems, non-trivial state management, or genuinely tricky edge cases

**Which to practice first**: Start with Parking Lot (1), Tic Tac Toe (16), and Cache with LRU (19). These three collectively cover the most common patterns, are achievable within 60 minutes, and give you strong signal on your baseline level.

**Which are asked most frequently**: Parking Lot, Chess, ATM, Elevator, and Order Management System appear in over 70% of machine coding rounds at the companies listed in Section 1.

---

## Section 7 — Common Mistakes in Machine Coding

These are the mistakes that cause strong engineers to fail machine coding rounds. Not one-off bugs — systematic patterns that interviewers recognize and score against.

### Mistake 1: Starting to Code Without Designing

**What it looks like**: The engineer opens their IDE within two minutes of receiving the problem and starts typing class names. By the 30-minute mark, they have a lot of code but no clear architecture. They start restructuring. By the end, there is a partially-refactored system with two competing designs layered on top of each other.

**Why it happens**: Coding feels like progress. Design feels like stalling. Engineers who are anxious about time take false comfort in seeing code appear.

**Why it fails**: Every structural mistake you make before you have a design costs you 3x as much to fix. Moving `calculateFee` from `ParkingLot` to `FeeCalculator` after you have written ten methods that call `parkingLot.calculateFee()` is a 20-minute refactor. Deciding upfront that `FeeCalculator` is a separate interface takes 30 seconds.

**The fix**: Force yourself to spend 10 minutes on requirements and design before opening your IDE. If you feel the urge to start coding before then, it is a sign you are anxious, not a sign you are ready. Experienced interviewers actively look for this discipline.

---

### Mistake 2: Over-Engineering from the Start

**What it looks like**: The engineer designs a beautiful abstract framework with six interfaces, four abstract classes, and a factory that produces factories. Forty minutes in, they have not implemented any actual parking logic.

**Why it happens**: Engineers who have studied design patterns want to apply them. Every problem looks like a nail when you are holding a Strategy-Observer-Command hammer. There is also a misunderstanding that more abstraction equals better design.

**Why it fails**: Over-engineering produces systems that are hard to implement, hard to understand, and frequently wrong. It signals that the engineer cannot calibrate complexity to requirements. It also guarantees you run out of time.

**The test**: For every interface you define, ask: "Does this have two concrete implementations right now, or a strong reason to expect two soon?" If the answer is no, you probably do not need the interface yet. A `FeeCalculator` interface is justified because pricing strategies clearly vary. A `ParkingLotConfigurationProvider` interface is almost certainly premature.

**The fix**: Start with concrete classes. Extract an interface when you have two implementations or when you are injecting a dependency you want to swap in tests. Apply patterns when they solve a clear pain point, not because the problem is "the right shape" for a pattern.

---

### Mistake 3: Not Handling Edge Cases

**What it looks like**: The engineer builds a clean system that works perfectly on the happy path. The interviewer asks: "What happens if I try to unpark a vehicle that was never parked here?" The engineer's code throws a `NullPointerException`.

**Why it happens**: Engineers focus on the happy path because the requirements describe the happy path. Edge cases feel like extensions that can be added later.

**Why it fails**: Edge case handling is part of correctness. A system that crashes on invalid input is not a working system. More importantly, how you handle edge cases reveals whether you think defensively — a key senior-engineer skill.

**The fix**: After implementing each public method's happy path, immediately ask: "What are the three ways this can be called incorrectly?" Add null checks. Add duplicate-state checks. Add boundary checks. Use `Objects.requireNonNull` for parameters that cannot be null. Throw specific, named exceptions.

```java
public Ticket park(Vehicle vehicle) {
    // Always validate inputs first
    Objects.requireNonNull(vehicle, "Vehicle cannot be null");

    // Check for duplicate parking
    boolean alreadyParked = activeTickets.values().stream()
        .anyMatch(t -> t.getVehicle().getLicensePlate().equals(vehicle.getLicensePlate()));
    if (alreadyParked) {
        throw new VehicleAlreadyParkedException(vehicle.getLicensePlate());
    }

    // Then proceed with happy path
    ...
}
```

---

### Mistake 4: Poor Naming Conventions

**What it looks like**: Classes named `Manager`, `Handler`, `Processor`, or `Util`. Methods named `process()`, `handle()`, `doStuff()`. Variables named `temp`, `x`, `obj`. Abbreviations everywhere: `pkg`, `tkt`, `cust`.

**Why it happens**: Engineers type fast and pick the first name that comes to mind. Under time pressure, naming feels less important than getting the logic right.

**Why it fails**: Poor naming is the single most visible signal of code quality. An interviewer can evaluate your naming in 30 seconds by glancing at your class list. A `VehicleAssignmentStrategy` is immediately understood. A `ParkingHandler` is a guess. Beyond interview optics, poor names force readers to read method bodies to understand what a method does — which is exactly what good names are supposed to prevent.

**Specific naming failures and their fixes**:

| Bad Name | Problem | Better Name |
|---|---|---|
| `ParkingManager` | "Manager" means nothing specific | `ParkingLot` or `ParkingService` |
| `process(Vehicle v)` | What does "process" mean? | `park(Vehicle vehicle)` |
| `getData()` | Get what data? | `getAvailableSpots()` |
| `calc(Ticket t)` | Abbreviation + no return type hint | `calculateFee(Ticket ticket)` |
| `flag` | Boolean with no semantic meaning | `isAvailable`, `hasCapacity` |
| `list` | No indication of what it contains | `availableSpots`, `activeTickets` |
| `obj` | Could be anything | Name it by what it actually is |

**The naming test**: Read the method signature. Can you tell what the method does without reading its body? If not, rename it.

---

### Mistake 5: Missing Encapsulation

**What it looks like**: Public fields everywhere. Mutable state exposed directly. Internal collections returned by reference. No invariants enforced.

**Why it happens**: Getters and setters for everything feel like "good OOP." They are not. A class with a public `List<ParkingSpot> spots` field has no encapsulation — it just has a slightly longer path to break the object.

**Why it fails**: Missing encapsulation means the class's invariants can be violated from outside. A `Ticket` whose `entryTime` can be set to any value at any time is not a reliable record of when a vehicle entered. A `ParkingSpot` whose `vehicle` field is public can be put into an inconsistent state (vehicle set without updating the availability flag).

**The encapsulation rules**:

```java
// Bad: public mutable field
public class ParkingSpot {
    public Vehicle vehicle; // anyone can set this
    public boolean available; // now inconsistent with vehicle
}

// Bad: returning internal mutable collection
public class Floor {
    private List<ParkingSpot> spots = new ArrayList<>();

    public List<ParkingSpot> getSpots() {
        return spots; // caller can clear() or add() to the internal list
    }
}

// Good: return unmodifiable view
public class Floor {
    private final List<ParkingSpot> spots;

    public List<ParkingSpot> getSpots() {
        return Collections.unmodifiableList(spots);
    }
}

// Good: encapsulate state transition
public class ParkingSpot {
    private Vehicle parkedVehicle;
    private boolean available = true;

    public void park(Vehicle vehicle) {
        if (!available) throw new SpotAlreadyOccupiedException(getId());
        this.parkedVehicle = vehicle;
        this.available = false;
    }

    public void unpark() {
        this.parkedVehicle = null;
        this.available = true;
    }

    public boolean isAvailable() { return available; }
    public Optional<Vehicle> getParkedVehicle() { return Optional.ofNullable(parkedVehicle); }
}
```

The key insight: a class is not encapsulated because it has private fields. It is encapsulated because its state can only be modified through methods that enforce invariants.

---

### Mistake 6: Tight Coupling

**What it looks like**: `ParkingLot` creates its own `HourlyFeeCalculator` with `new`. `OrderService` creates its own `EmailNotifier` with `new`. Classes reference concrete implementations, not interfaces.

**Why it happens**: Using `new` is the simplest way to get an object. It feels natural. The coupling is invisible until you try to test or extend the code.

**Why it fails**: Tightly coupled classes cannot be tested in isolation. Adding a new fee strategy requires modifying `ParkingLot`. Changing the notification channel requires modifying `OrderService`. The system is closed for extension and the test suite is impossible to write without setting up the entire dependency chain.

**The coupling test**: For every `new SomeConcreteClass()` inside a class that is not a factory, ask: "Would I ever want to swap this for a different implementation in a test or a future feature?" If yes, inject it.

```java
// Tightly coupled — ParkingLot owns the calculator decision
public class ParkingLot {
    private final FeeCalculator feeCalculator = new HourlyFeeCalculator(); // hardcoded

    // Testing with a different fee strategy requires changing this class
}

// Loosely coupled — caller decides the calculator
public class ParkingLot {
    private final FeeCalculator feeCalculator;

    public ParkingLot(List<Floor> floors, FeeCalculator feeCalculator) {
        this.floors = floors;
        this.feeCalculator = feeCalculator; // injected
    }

    // Testing: inject a MockFeeCalculator
    // Production: inject HourlyFeeCalculator or SurgePricingCalculator
    // Extension: inject any new FeeCalculator without changing this class
}
```

Dependency injection — through constructors, not setters — is the single most effective technique for reducing coupling in machine coding problems. Use it consistently on any dependency that represents a varying behavior.

---

## Summary: The Machine Coding Checklist

Before you submit or call yourself done, run through this checklist:

**Requirements**
- [ ] Core use cases identified and stated
- [ ] Assumptions made explicit and confirmed
- [ ] Scope boundaries defined (what is in, what is out)

**Design**
- [ ] Core entities identified
- [ ] Responsibilities clearly distributed
- [ ] Key interfaces defined for varying behavior
- [ ] No god classes (no class over 200 lines that does more than one thing)

**Implementation**
- [ ] Core use case works end to end
- [ ] Public methods validate their inputs
- [ ] Edge cases handled with specific exceptions
- [ ] No magic numbers (use named constants)
- [ ] No abbreviations in names
- [ ] Dependencies injected, not created internally

**Extensibility**
- [ ] Can articulate where a new variant of each entity type would go
- [ ] Adding a new variant does not require modifying existing logic classes
- [ ] At least one extension demonstrated verbally or in code

**Demonstration**
- [ ] `main()` or test class shows the system working end to end
- [ ] Output is readable and confirms the system is correct

---

*Proceed to [`07-machine-coding/questions/`](./questions/) for the full problem statements, and to [`07-machine-coding/solutions/`](./solutions/) for annotated reference implementations.*
