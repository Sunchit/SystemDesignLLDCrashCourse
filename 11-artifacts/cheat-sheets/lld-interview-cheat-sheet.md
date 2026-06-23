# LLD Interview Day Cheat Sheet

> **Purpose:** Review in 15 minutes the morning of an interview. Dense by design — every line is load-bearing.

---

## 1. The 5-Step LLD Framework

| Step | Time | What You Do |
|---|---|---|
| **Step 1:** Clarify Requirements | 5 min | Ask scoping questions. Pin down scale, actors, constraints. No code yet. |
| **Step 2:** Identify Core Entities | 5 min | Extract nouns → classes. List attributes. Rough class names on the whiteboard. |
| **Step 3:** Define Relationships & Interfaces | 10 min | Draw associations, inheritance, composition. Define public method signatures. |
| **Step 4:** Apply Design Patterns | 10 min | Name the patterns you're using and justify. Add enums, factories, observers. |
| **Step 5:** Edge Cases & Extensibility | 5 min | Thread safety, nulls, error states, SOLID review, future feature hooks. |

### Step 1 — Clarifying Questions to Ask

- Who are the actors? (user roles, external systems)
- What are the primary use cases / workflows?
- What scale? (1 user vs. 1 million concurrent)
- Any persistence? (in-memory is fine to state explicitly)
- Mobile/web/API — or pure object model?
- Any SLA or performance constraints?

### Step 2 — Nouns → Classes Heuristic

```
Problem statement → underline nouns → shortlist non-redundant ones → assign as class or enum
Verbs → methods | Adjectives → fields or strategy variants
```

### Step 3 — Relationships Checklist

- [ ] Is-A (inheritance) vs. Has-A (composition) — prefer composition
- [ ] Identify interfaces (behavior contracts) vs. abstract classes (partial implementations)
- [ ] Mark cardinality: 1:1, 1:N, M:N
- [ ] Identify which associations need a separate association class (e.g., `Booking` for User ↔ Hotel)

### Step 4 — Pattern Application Checklist

- [ ] Object creation complex? → Builder / Factory
- [ ] Multiple behaviors that swap? → Strategy / State
- [ ] Notify multiple parties? → Observer
- [ ] Add features without modifying? → Decorator / Visitor
- [ ] Simplify a complex subsystem API? → Facade
- [ ] Tree structure? → Composite

### Step 5 — Extensibility Prompts

- "If we add a new type of X, what changes?" (Open/Closed check)
- "Can this run in multiple threads?" (thread-safety check)
- "What happens when the input is null/empty/malformed?"

---

## 2. Questions to Always Ask at the Start

1. **What are the primary actors?** (Who uses the system?)
2. **What are the top 3 use cases?** (Focus scope — most LLD problems have 5-10 features; design for the core 3.)
3. **In-memory or persistent?** (Clarifies whether you need repository/DAO layer.)
4. **Single-machine or distributed?** (Sets thread-safety expectations.)
5. **Any existing APIs I must integrate with?** (Flags Adapter need.)
6. **Do we need concurrency support?** (Locks, atomic ops, thread-safe collections.)
7. **What are the extension points expected in the future?** (Drives interface choices.)
8. **Are there SLA/performance constraints?** (Caching, Flyweight, lazy loading considerations.)
9. **Any specific design patterns or principles you want applied?** (Interviewer preference signal.)
10. **Should I handle payments/auth, or treat them as black boxes?** (Scopes the problem hard.)

---

## 3. Common Patterns Per Problem Type

| Problem Type | Core Pattern(s) | Supporting Patterns | Key Entities |
|---|---|---|---|
| **Parking Lot** | Strategy (pricing), State (spot/vehicle state) | Factory (vehicle types), Singleton (ParkingLot) | `ParkingLot`, `ParkingSpot`, `Vehicle`, `Ticket`, `PricingStrategy` |
| **Library System** | State (book availability), Observer (reservation notify) | Factory (membership types), Iterator (catalog search) | `Book`, `Member`, `Loan`, `Catalog`, `Reservation` |
| **Hotel Booking** | State (room/booking states), Strategy (pricing/discount) | Builder (booking request), Facade (booking subsystem) | `Hotel`, `Room`, `Booking`, `Guest`, `PricingStrategy` |
| **Elevator System** | State (elevator state), Strategy (dispatch algorithm) | Observer (floor requests), Singleton (ElevatorController) | `Elevator`, `ElevatorController`, `Request`, `DispatchStrategy` |
| **Chess / Board Game** | Command (move + undo), State (game state) | Composite (board), Factory (piece creation) | `Board`, `Piece`, `Move`, `Player`, `GameController` |
| **Ride Share (Uber)** | Observer (driver matching), Strategy (pricing/routing) | State (trip state), Factory (ride types) | `Driver`, `Rider`, `Trip`, `Location`, `PricingStrategy` |
| **ATM** | State (ATM states: idle/card-inserted/PIN/transaction), Command (transactions) | Singleton (ATM), Chain of Responsibility (validation) | `ATM`, `Card`, `Account`, `Transaction`, `ATMState` |
| **Notification System** | Observer (pub-sub), Strategy (channel: email/SMS/push) | Factory (notification types), Decorator (retry/throttle) | `NotificationService`, `Channel`, `Subscriber`, `Notification` |
| **Shopping Cart** | Composite (cart + line items), Strategy (discount/promo) | Observer (inventory update), Builder (order) | `Cart`, `LineItem`, `Product`, `DiscountStrategy`, `Order` |
| **File System** | Composite (file + directory tree), Visitor (search/size calc) | Iterator (traversal), Decorator (permissions/compression) | `FileSystemNode`, `File`, `Directory`, `Permission` |

---

## 4. Time-Box Guide — 45-Minute Interview

```
00:00 – 05:00  | REQUIREMENTS   | Ask your 10 questions. Write actors + top use cases on whiteboard.
05:00 – 10:00  | ENTITIES       | Extract nouns. Sketch 5–8 class names. List 2–3 attributes each.
10:00 – 20:00  | RELATIONSHIPS  | Draw associations. Define 2–3 key interfaces. Clarify inheritance.
20:00 – 30:00  | PATTERNS       | Apply and name patterns. Draw sequence diagram for main flow.
30:00 – 35:00  | CODE           | Implement 1–2 core classes fully. Show one interface + impl.
35:00 – 42:00  | EDGE CASES     | Thread safety, error states, extensibility discussion.
42:00 – 45:00  | BUFFER / Q&A   | Summarize design, invite feedback.
```

> **Rule:** Never start coding before minute 20. Sketching first always scores higher than diving into code.

---

## 5. Red Flags to Avoid

1. **Starting with code immediately** — no class diagram or entity discussion first.
2. **Using `public` fields** on domain classes — breaks encapsulation.
3. **God class** — one class doing everything (booking + pricing + notification + storage).
4. **`instanceof` chains** — replace with polymorphism (Strategy, State, Visitor).
5. **Concrete dependencies everywhere** — depend on interfaces, not implementations.
6. **No interfaces defined** — shows you don't think in contracts.
7. **Ignoring thread safety** when concurrency is implied (booking systems, ATMs, ride-share).
8. **Switch/if-else on type/state** — this is the State or Strategy pattern screaming at you.
9. **Anemic domain model** — classes that are pure data bags with no behavior.
10. **Not naming your patterns** — saying "I'll have a class that creates X" instead of "I'll use a Factory Method here."

---

## 6. Quick UML Notation Reference (Text-Based)

```
CLASS RELATIONSHIPS
──────────────────────────────────────────────
A ──────> B        Association      A has a reference to B
A ──────▷ B        Dependency       A uses B temporarily (method param)
A ────── B         Composition      A owns B; B cannot exist without A  (filled diamond)
A ─ ─ ─▷ B        Aggregation      A has B; B can exist independently  (open diamond)
A ──────▶ B        Inheritance      A extends B (solid arrow, open triangle head)
A ─ ─ ─▶ B        Realization      A implements B (dashed arrow, open triangle head)

MULTIPLICITY
──────────────────────────────────────────────
1           Exactly one
0..1        Zero or one
*  / 0..*   Zero or more
1..*        One or more
m..n        Specified range

VISIBILITY (in class box)
──────────────────────────────────────────────
+  public
-  private
#  protected
~  package-private

SEQUENCE DIAGRAM
──────────────────────────────────────────────
A ──────────────> B   :message()   Synchronous call
A - - - - - - -> B   :reply       Return / async
[ loop / alt / opt ]              Combined fragments (guard in [])
```

---

## 7. Design Smell → Pattern Mapping

| Design Smell | Pattern to Apply | Why It Fixes It |
|---|---|---|
| Switch/if-else on object type | **Strategy** | Each branch becomes a concrete strategy; open for extension |
| Switch/if-else on state | **State** | State-specific behavior moves into state objects |
| Subclass explosion to add features | **Decorator** | Wrap instead of inherit; combine behaviors at runtime |
| Client coupled to `new ConcreteClass()` everywhere | **Factory Method / Abstract Factory** | Centralize and abstract object creation |
| Multi-param constructor (> 4 params, many optional) | **Builder** | Named, readable, validated construction |
| Repeated `null` checks across codebase | **Null Object** (variant of Strategy) | Replace null with inert no-op object |
| One class notifying many others via direct method calls | **Observer** | Decouple publisher from subscribers |
| Deep inheritance tree for a cross-cutting concern (logging, auth) | **Proxy / Decorator** | Add cross-cutting behavior without touching hierarchy |
| Multiple subsystems called in fixed sequence everywhere | **Facade** | Encapsulate orchestration in single entry point |
| Complex tree traversal logic scattered across codebase | **Visitor** or **Iterator** | Centralize traversal; separate algorithm from structure |
| Many similar small objects consuming memory | **Flyweight** | Share intrinsic state; pass extrinsic state as argument |
| Hardcoded algorithm with no swap point | **Strategy + Dependency Injection** | Extract algorithm; inject variant |
| Object copied but state restoration is fragile | **Memento** | Formalize snapshot/restore contract |
| Validation / auth chain as nested if-else | **Chain of Responsibility** | Decouple handlers; each handles or forwards |

---

## 8. Top 10 Must-Know Patterns for Interviews (Ranked by Frequency)

| Rank | Pattern | 1-Line Description | Prototypical Interview Use Case |
|---|---|---|---|
| 1 | **Strategy** | Swap algorithms at runtime behind a common interface | Pricing engine, discount calculator, sort strategy |
| 2 | **Observer** | Notify many subscribers when one object changes | Notification system, event bus, UI model updates |
| 3 | **Factory Method** | Let subclass / factory decide what concrete type to create | Vehicle/payment/notification type creation |
| 4 | **Singleton** | Guarantee one instance with global access | ParkingLot manager, configuration, logger |
| 5 | **State** | Object's behavior changes based on internal state | Elevator, ATM, booking workflow, order lifecycle |
| 6 | **Decorator** | Wrap object to add behavior without modifying class | Middleware stack, I/O streams, auth + logging |
| 7 | **Builder** | Step-by-step construction of complex objects | Order, booking request, report configuration |
| 8 | **Command** | Encapsulate operation as object for undo/queue/log | Chess moves, text editor, job scheduler |
| 9 | **Composite** | Treat leaf and container uniformly via common interface | File system, UI hierarchy, org chart |
| 10 | **Chain of Responsibility** | Pass request through chain until one handler consumes it | Validation pipeline, escalation chain, servlet filters |

---

## 9. Machine Coding Tips (Live Coding Rounds)

1. **Compile in your head before you type.** Sketch the class structure in comments first (`// class Booking`, `// fields`, `// methods`) then fill in. Never delete-and-retype — it signals anxiety.

2. **Name things like a senior engineer.** Use full, intention-revealing names: `calculateParkingFee()` not `calc()`, `VehicleType` enum not magic strings, `BookingStatus` enum not `int status`.

3. **Code the interface before the implementation.** Write `interface PaymentStrategy { double calculate(Booking b); }` before `CashPaymentStrategy`. It proves you think in abstractions.

4. **Handle the happy path first, then add guards.** Complete the primary flow end-to-end, then add `null` checks, `IllegalArgumentException`, and edge conditions. A working skeleton beats a half-defended mess.

5. **Narrate your trade-offs out loud.** "I'm using `synchronized` here for simplicity — in production I'd use `ReentrantReadWriteLock` to allow concurrent reads." Interviewers reward awareness of trade-offs over perfect code.
