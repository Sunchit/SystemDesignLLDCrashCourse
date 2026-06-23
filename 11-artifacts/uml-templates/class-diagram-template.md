# UML Class Diagram Reference and Templates

---

## Section 1: Class Notation Template

### Standard 3-Compartment Class Box

```
+-----------------------------+
|        ClassName            |  <- Compartment 1: Class Name (bold, centered)
+-----------------------------+
| - attribute1: Type          |  <- Compartment 2: Attributes
| # attribute2: Type          |
| + attribute3: Type          |
| ~ attribute4: Type          |
+-----------------------------+
| + method1(): ReturnType     |  <- Compartment 3: Methods
| - method2(arg: Type): void  |
| # method3(): Type           |
+-----------------------------+
```

**Compartment 1 — Class Name:** Centered. If abstract, italicize the name or annotate with `{abstract}`. If interface, prefix with `<<interface>>` stereotype.

**Compartment 2 — Attributes:** Format: `visibility name: Type [= defaultValue]`

**Compartment 3 — Methods:** Format: `visibility name(param: Type, ...): ReturnType`

---

### Visibility Symbols

| Symbol | Visibility      | Java Equivalent | Accessible From                            |
|--------|-----------------|-----------------|--------------------------------------------|
| `+`    | Public          | `public`        | Anywhere                                   |
| `-`    | Private         | `private`       | Declaring class only                       |
| `#`    | Protected       | `protected`     | Declaring class + subclasses + same package|
| `~`    | Package-private | (default)       | Classes in the same package                |

---

### Abstract Class

```
+-----------------------------+
|   <<abstract>>              |
|   *AbstractShape*           |  <- Name in italics or prefixed
+-----------------------------+
| # color: String             |
+-----------------------------+
| + draw(): void {abstract}   |  <- Abstract method: italicize or mark {abstract}
| + getColor(): String        |
+-----------------------------+
```

### Interface

```
+-----------------------------+
|       <<interface>>         |
|         Drawable            |
+-----------------------------+
|                             |  <- Attributes compartment usually empty
+-----------------------------+
| + draw(): void              |
| + resize(factor: double)    |
+-----------------------------+
```

### Concrete Class

```
+-----------------------------+
|           Circle            |
+-----------------------------+
| - radius: double            |
| - CENTER: Point {final}     |
+-----------------------------+
| + draw(): void              |
| + area(): double            |
+-----------------------------+
```

---

### Member Modifiers

| Modifier   | UML Notation                        | Example                                     |
|------------|-------------------------------------|---------------------------------------------|
| Static     | Underline the member                | <u>`+ getInstance(): Singleton`</u> (underlined) or note `{static}` |
| Abstract   | Italicize or append `{abstract}`    | *`+ draw(): void`* or `+ draw(): void {abstract}` |
| Final      | Append `{frozen}` or `{final}`      | `- MAX_SIZE: int {final}`                   |

In ASCII diagrams where underline/italic are unavailable, use annotations:

```
| + getInstance(): Singleton  {static}  |
| + draw(): void              {abstract}|
| - MAX_SIZE: int = 100       {final}   |
```

---

## Section 2: All Relationship Types

### 1. Association

**Arrow:**
```
ClassA -----------------> ClassB
         association
```
Plain line with optional open arrowhead indicating navigation direction.

**Formal meaning:** A structural relationship where one class has a reference to another. Objects have independent lifecycles.

**When to use:** `ClassA` uses or knows about `ClassB`, but neither owns the other. Examples: `Student` references `Course`; `Driver` references `Car`.

**Java equivalent:**
```java
class Student {
    private Course enrolledCourse; // association: Student knows Course
}
```

**Multiplicity examples:**

| Notation | Meaning                        |
|----------|--------------------------------|
| `1`      | Exactly one                    |
| `0..1`   | Zero or one (optional)         |
| `*`      | Zero or more                   |
| `1..*`   | One or more                    |
| `0..*`   | Zero or more (explicit form)   |

```
Student "1" --------> "0..*" Course
```

---

### 2. Aggregation

**Arrow:**
```
ClassA <>-------------> ClassB
  (whole)               (part)
```
Open (hollow) diamond at the "whole" end; plain line or open arrow at the "part" end.

**Formal meaning:** A "has-a" relationship where the part can exist independently of the whole. The part has a lifecycle independent of the aggregate.

**When to use:** The container and contained objects have independent lifetimes. Destroying the whole does NOT destroy the parts. Example: `Department` aggregates `Professors` — professors can exist without a department.

**Java equivalent:**
```java
class Department {
    private List<Professor> professors; // passed in from outside
    Department(List<Professor> professors) {
        this.professors = professors;
    }
}
```

---

### 3. Composition

**Arrow:**
```
ClassA [filled<>]-------> ClassB
  (whole)                  (part)
```
Filled (solid) diamond at the "whole" end.

**Formal meaning:** A strong "owns-a" relationship. The part cannot exist without the whole. The whole is responsible for the lifecycle of its parts — when the whole is destroyed, its parts are destroyed.

**When to use:** When the part has no meaningful existence outside the container. Example: `House` composes `Rooms` — a room cannot exist without a house.

**Java equivalent:**
```java
class House {
    private final List<Room> rooms; // created inside, owned by House
    House() {
        this.rooms = new ArrayList<>();
        rooms.add(new Room("Kitchen")); // House creates Room
    }
}
```

**Aggregation vs Composition — key distinction:**

| Aspect          | Aggregation                        | Composition                        |
|-----------------|------------------------------------|------------------------------------|
| Diamond         | Hollow `<>`                        | Filled `[<>]`                      |
| Part lifecycle  | Independent of whole               | Tied to whole                      |
| Ownership       | Shared or external                 | Exclusive, whole creates parts     |
| Java signal     | Part injected via constructor/setter | Part instantiated inside the class |
| Destroy whole?  | Parts survive                      | Parts are destroyed                |

---

### 4. Inheritance / Generalization

**Arrow:**
```
SubClass -------|>  SuperClass
           (open triangle arrowhead at superclass end)
```

**Formal meaning:** An "is-a" relationship. The subclass inherits all attributes and methods of the superclass and can override behavior.

**When to use:** True "is-a" relationship where substitution (Liskov Substitution Principle) holds. Example: `Dog` extends `Animal`.

**Java equivalent:**
```java
class Dog extends Animal {
    @Override
    public void speak() { System.out.println("Woof"); }
}
```

---

### 5. Realization / Implementation

**Arrow:**
```
ConcreteClass - - - - -|> <<interface>>
              (dashed line, open triangle at interface end)
```

**Formal meaning:** The class commits to fulfilling the contract defined by the interface.

**When to use:** When a concrete class implements an interface. Example: `ArrayList` implements `List`.

**Java equivalent:**
```java
class Circle implements Drawable {
    @Override
    public void draw() { /* render circle */ }
}
```

---

### 6. Dependency

**Arrow:**
```
ClassA  - - - - - ->  ClassB
             (dashed open arrow)
```

**Formal meaning:** A weak, transient usage relationship. `ClassA` uses `ClassB` but does not hold a persistent reference. `ClassB` typically appears as a method parameter, local variable, or return type.

**When to use:** When `ClassA` uses `ClassB` only temporarily within a method — not as a field. Example: `OrderService` depends on `EmailService` to send a confirmation.

**Java equivalent:**
```java
class OrderService {
    public void placeOrder(Order o, EmailService email) { // dependency
        email.send(o.getCustomerEmail(), "Order confirmed");
    }
}
```

---

### Summary: All Arrow Symbols

```
Association:     A ──────────────> B        (solid line, open arrow)
Aggregation:     A <>─────────────  B        (solid line, hollow diamond at A)
Composition:     A [<>]───────────  B        (solid line, filled diamond at A)
Inheritance:     A ──────────────|> B        (solid line, open triangle at B)
Realization:     A - - - - - - - |> B        (dashed line, open triangle at B)
Dependency:      A - - - - - - - -> B        (dashed line, open arrow at B)
```

---

## Section 3: Common Design Patterns in UML

---

### 1. Strategy Pattern

**Description:** Defines a family of algorithms, encapsulates each one, and makes them interchangeable. The client holds a reference to a strategy interface and delegates behavior to it at runtime.

**Key insight:** Replaces conditional branching (`if/else`, `switch`) on behavior type with polymorphic dispatch. Behavior can be swapped without modifying the context class.

```
+----------------------+          +------------------------+
|      Context         |          |   <<interface>>        |
+----------------------+          |     SortStrategy       |
| - strategy:          |--------> +------------------------+
|     SortStrategy     |  uses    | + sort(data: int[]):   |
+----------------------+          |     void               |
| + setStrategy(s:     |          +------------------------+
|     SortStrategy)    |                    ^
| + executeSort(data:  |                    |
|     int[]): void     |         +----------+----------+
+----------------------+         |                     |
                        +------------------+  +------------------+
                        | BubbleSortStrategy|  | QuickSortStrategy|
                        +------------------+  +------------------+
                        | + sort(data:     |  | + sort(data:     |
                        |     int[]): void |  |     int[]): void |
                        +------------------+  +------------------+
```

**Relationships:**
- `Context` → `SortStrategy`: Association (holds reference)
- `BubbleSortStrategy`, `QuickSortStrategy` → `SortStrategy`: Realization

---

### 2. Observer Pattern

**Description:** Defines a one-to-many dependency between objects. When the subject (publisher) changes state, all registered observers (subscribers) are notified automatically.

**Key insight:** Decouples the subject from its observers. The subject only knows the `Observer` interface — it never references concrete observer types directly.

```
+-------------------------+          +----------------------+
|    <<interface>>        |          |   <<interface>>      |
|       Subject           |          |      Observer        |
+-------------------------+          +----------------------+
| + attach(o: Observer)   |          | + update(): void     |
| + detach(o: Observer)   |          +----------------------+
| + notify(): void        |                    ^
+-------------------------+                    |
           ^                       +-----------+-----------+
           |              +------------------+ +------------------+
+---------------------+   | ConcreteObserverA| | ConcreteObserverB|
|   ConcreteSubject   |   +------------------+ +------------------+
+---------------------+   | - subject:       | | - subject:       |
| - state: int        |   |   ConcreteSubject| |   ConcreteSubject|
| - observers:        |   +------------------+ +------------------+
|   List<Observer>    |-->| + update(): void | | + update(): void |
+---------------------+   +------------------+ +------------------+
| + getState(): int   |
| + setState(s: int)  |
| + notify(): void    |
+---------------------+
```

**Relationships:**
- `ConcreteSubject` → `Observer`: Association with multiplicity `1..*`
- `ConcreteObserverA/B` → `Observer`: Realization
- `ConcreteSubject` → `Subject`: Realization

---

### 3. Factory Method Pattern

**Description:** Defines an interface for creating an object but lets subclasses decide which class to instantiate. The factory method defers instantiation to subclasses.

**Key insight:** Removes the dependency of the creator on concrete product types. Adding a new product requires only a new `ConcreteCreator` subclass — the client code referencing `Creator` is unchanged.

```
+-----------------------------+          +-----------------------------+
|     <<abstract>>            |          |      <<interface>>          |
|          Creator            |          |           Product           |
+-----------------------------+          +-----------------------------+
| + createProduct(): Product  |--------> | + operation(): void         |
| + someOperation(): void     |          +-----------------------------+
+-----------------------------+                        ^
               ^                                       |
               |                          +------------+------------+
  +------------------+          +-----------------+ +-----------------+
  | ConcreteCreatorA |          | ConcreteProductA| | ConcreteProductB|
  +------------------+          +-----------------+ +-----------------+
  | + createProduct()|          | + operation():  | | + operation():  |
  |   : Product      |          |     void        | |     void        |
  +------------------+          +-----------------+ +-----------------+
  | ConcreteCreatorB |
  +------------------+
  | + createProduct()|
  |   : Product      |
  +------------------+
```

**Relationships:**
- `Creator` → `Product`: Dependency (factory method returns `Product`)
- `ConcreteCreatorA/B` → `Creator`: Inheritance
- `ConcreteProductA/B` → `Product`: Realization

---

### 4. Builder Pattern

**Description:** Separates the construction of a complex object from its representation, allowing the same construction process to create different representations.

**Key insight:** Eliminates telescoping constructors. The `Director` controls the construction sequence; the `Builder` handles the "how". The client gets a clean, validated object without knowing its internal assembly.

```
+----------------------------+           +-----------------------------+
|          Director          |           |      <<interface>>          |
+----------------------------+           |           Builder           |
| - builder: Builder         |---------> +-----------------------------+
+----------------------------+           | + buildPartA(): Builder     |
| + Director(b: Builder)     |           | + buildPartB(): Builder     |
| + construct(): Product     |           | + buildPartC(): Builder     |
+----------------------------+           | + build(): Product          |
                                         +-----------------------------+
                                                      ^
                                                      |
                                        +-----------------------------+
                                        |      ConcreteBuilder        |
                                        +-----------------------------+
                                        | - product: Product          |
                                        +-----------------------------+
                                        | + buildPartA(): Builder     |
                                        | + buildPartB(): Builder     |
                                        | + buildPartC(): Builder     |
                                        | + build(): Product          |
                                        +-----------------------------+
                                                      |
                                                      | creates
                                                      v
                                        +-----------------------------+
                                        |           Product           |
                                        +-----------------------------+
                                        | - partA: String             |
                                        | - partB: int                |
                                        | - partC: boolean            |
                                        +-----------------------------+
```

**Relationships:**
- `Director` → `Builder`: Association
- `ConcreteBuilder` → `Builder`: Realization
- `ConcreteBuilder` → `Product`: Composition (creates and owns)

---

### 5. Decorator Pattern

**Description:** Attaches additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

**Key insight:** Each decorator wraps a component, forwards the core call, and adds behavior before/after. Multiple decorators can be stacked. The client code sees only the `Component` interface — the wrapping is invisible.

```
+---------------------------+
|      <<interface>>        |
|         Component         |
+---------------------------+
| + operation(): String     |
+---------------------------+
           ^
           |
    +------+------+
    |             |
+------------------+    +-----------------------------+
| ConcreteComponent|    |     <<abstract>>            |
+------------------+    |       Decorator             |
| + operation():   |    +-----------------------------+
|     String       |    | - wrappee: Component        |
+------------------+    +-----------------------------+
                        | + Decorator(c: Component)   |
                        | + operation(): String       |
                        +-----------------------------+
                                     ^
                                     |
                        +------------+------------+
               +------------------+   +------------------+
               | ConcreteDecoratorA|  | ConcreteDecoratorB|
               +------------------+   +------------------+
               | + operation():   |   | + operation():   |
               |     String       |   |     String       |
               +------------------+   +------------------+
```

**Relationships:**
- `ConcreteComponent` → `Component`: Realization
- `Decorator` → `Component`: Realization (is-a Component)
- `Decorator` → `Component`: Association (has-a Component as wrappee)
- `ConcreteDecoratorA/B` → `Decorator`: Inheritance

---

### 6. Composite Pattern

**Description:** Composes objects into tree structures to represent part-whole hierarchies. Clients treat individual objects and compositions uniformly through the same interface.

**Key insight:** The `Composite` class and `Leaf` class both implement the same `Component` interface. Client code operates on `Component` references without knowing if it holds a leaf or a subtree. Enables recursive tree traversal.

```
+-----------------------------+
|      <<interface>>          |
|          Component          |
+-----------------------------+
| + operation(): void         |
| + add(c: Component): void   |
| + remove(c: Component): void|
| + getChild(i: int): Component|
+-----------------------------+
              ^
              |
     +--------+--------+
     |                 |
+----------+    +---------------------+
|   Leaf   |    |      Composite      |
+----------+    +---------------------+
| + operation():|    | - children:          |
|     void      |    |   List<Component>    |
+----------+    +---------------------+
                | + operation(): void  |
                | + add(c: Component) |
                | + remove(c: Component)|
                | + getChild(i: int)  |
                +---------------------+
                |                     |
                | (self-referential)  |
                | Composite <>------> Component
                +---------------------+
```

**Relationships:**
- `Leaf` → `Component`: Realization
- `Composite` → `Component`: Realization
- `Composite` → `Component`: Aggregation (holds a list of children, which may be Leafs or other Composites)

---

## Section 4: Whiteboard/Interview UML Tips

### How to Draw UML Quickly in Under 5 Minutes

**Step 1 — Identify entities (30 seconds):** Ask: what are the nouns in the problem? Each noun is a candidate class. Write class names in rough boxes. Do not fill in fields yet.

**Step 2 — Identify relationships (60 seconds):** For each pair of classes, ask: does A own B? use B? extend B? implement B? Draw arrows immediately. Use labels if the relationship is not obvious.

**Step 3 — Fill in key attributes (60 seconds):** Add only the fields that drive your design — IDs, state flags, foreign-key-style references. Skip getters/setters.

**Step 4 — Add important methods (60 seconds):** Add only the methods that show contract or behavior. Focus on public interface, not implementation details.

**Step 5 — Annotate multiplicities (30 seconds):** Add `1`, `*`, `1..*` to all associations. Interviewers specifically check this.

**Step 6 — Review and clean up (30 seconds):** Check for missing abstractions (should any group of classes share an interface?), missing aggregation/composition distinctions, and orphaned classes.

---

### Acceptable Shortcuts in Interviews

- **Skip getter/setter methods** — interviewers know they exist. Only show substantive behavior.
- **Omit method bodies** — write signature only: `+ placeOrder(order: Order): Receipt`
- **Use abbreviated types** — `String`, `int`, `List<T>` are fine. Avoid fully qualified Java names.
- **Merge small utility classes** — if a class has only data and no behavior, you may describe it as an attribute type instead of a full box.
- **Approximate arrow style** — if you cannot draw a filled diamond on a whiteboard, write `[comp]` or `[agg]` on the line.
- **Use `<<interface>>` or `<<abstract>>` stereotypes** instead of visual italics.
- **Partial diagrams are acceptable** — draw the core domain model fully; note "…and Notification subsystem follows similar pattern."

---

### What to Always Label (Never Skip)

- **Class names** — every box must have a name.
- **Relationship type** — never leave a line unlabeled if its type is ambiguous (especially aggregation vs composition).
- **Multiplicity** — both ends of every association. `1` is not assumed; write it explicitly.
- **Interface stereotypes** — mark `<<interface>>` and `<<abstract>>` explicitly.
- **Key method signatures** on interfaces and abstract classes — this is where the contract lives.
- **Navigation direction** — add arrowheads to indicate which class holds a reference to which.

---

### Common Mistakes Candidates Make in Interview UML

1. **Confusing aggregation and composition.** Drawing a hollow diamond when the part's lifecycle is wholly owned by the container (should be composition), or drawing a filled diamond when the part is shared. Default rule: if you `new` it inside the constructor, it is composition.

2. **Omitting multiplicities.** Saying "a Cart has Items" without specifying `1..*` loses critical design information.

3. **Making everything an inheritance relationship.** Inheritance is the strongest coupling. Prefer composition over inheritance. Overuse of `extends` in a diagram signals poor design instinct.

4. **Missing the interface layer.** Drawing concrete-to-concrete dependencies directly instead of programming to an interface. Every polymorphic relationship needs an abstraction (interface or abstract class) at the top.

5. **Circular dependencies between packages/layers.** Controller → Service → Repository is fine. Service → Controller is a design error — interviewers look for this.

6. **Conflating dependency with association.** If `ClassA` only uses `ClassB` in a single method argument, it is a dependency (dashed arrow), not an association (solid line with field).

7. **God classes.** Drawing one class with 20 attributes signals failure to decompose the domain. Break it up.

8. **Forgetting return types on key methods.** Writing `+ placeOrder()` without a return type is incomplete. At minimum specify `void` or the key return type.

---

## Section 5: Five Complete System Class Diagram Templates

---

### 1. Library Management System

**Key Classes/Interfaces:**

| Class/Interface    | Responsibility                                           |
|--------------------|----------------------------------------------------------|
| `Library`          | Root aggregate; manages catalog and members              |
| `Book`             | Represents a title (not a physical copy)                 |
| `BookItem`         | A physical copy of a `Book` with its own barcode/status  |
| `Member`           | Library patron with borrowing privileges                 |
| `Librarian`        | Staff member who processes loans and returns             |
| `Account`          | Tracks a member's loan history and fines                 |
| `Loan`             | Records a single borrowing transaction                   |
| `Reservation`      | Holds a book for a member                               |
| `Catalog`          | Searchable index of all books                           |
| `BookStatus`       | Enum: AVAILABLE, BORROWED, RESERVED, LOST               |

**ASCII Class Diagram:**

```
+--------------------+        +-------------------------+
|      Library       |        |         Catalog         |
+--------------------+        +-------------------------+
| - name: String     |<>----->| - books: List<Book>     |
| - address: String  | 1    1 +-------------------------+
+--------------------+        | + searchByTitle(t: String): List<Book>|
| + getCatalog():    |        | + searchByAuthor(a: String): List<Book>|
|   Catalog          |        | + searchByISBN(isbn: String): Book    |
| + getMembers():    |        +-------------------------+
|   List<Member>     |
+--------------------+
        |
        | 1
        |  manages
        | *
+--------------------+        +-------------------------+
|      Member        |        |          Book           |
+--------------------+        +-------------------------+
| - memberId: String |        | - isbn: String          |
| - name: String     |        | - title: String         |
| - email: String    |        | - author: String        |
| - account: Account |<>---+  | - subject: String       |
+--------------------+     |  | - publisher: String     |
| + requestBook(b:   |     |  +-------------------------+
|   BookItem): Loan  |     |  | + getAvailableCopies(): |
| + returnBook(l:    |     |  |   List<BookItem>        |
|   Loan): void      |     |  +-------------------------+
| + reserveBook(b:   |     |          | 1
|   Book): Reservation|    |          | composed of
+--------------------+     |          | 1..*
        |                  |  +-------------------------+
        | 1                |  |        BookItem          |
        | places           |  +-------------------------+
        | *                |  | - barcode: String       |
+--------------------+     |  | - status: BookStatus    |
|       Loan         |     |  | - dueDate: LocalDate    |
+--------------------+     |  +-------------------------+
| - loanId: String   |     |  | + checkout(): void      |
| - bookItem:        |     |  | + returnItem(): void    |
|   BookItem         |---->|  +-------------------------+
| - member: Member   |     |
| - issueDate:       |     |  +-------------------------+
|   LocalDate        |     |  |        Account          |
| - dueDate:         |     +->+-------------------------+
|   LocalDate        |        | - totalBooksCheckedOut: |
| - returnDate:      |        |   int                   |
|   LocalDate        |        | - totalFine: double     |
+--------------------+        | - loans: List<Loan>     |
| + calculateFine(): |        +-------------------------+
|   double           |        | + addFine(amt: double)  |
+--------------------+        | + payFine(amt: double)  |
                              +-------------------------+

+---------------------+       +-------------------------+
|     Librarian       |       |       Reservation       |
+---------------------+       +-------------------------+
| - staffId: String   |       | - reservationId: String |
| - name: String      |       | - book: Book            |
+---------------------+       | - member: Member        |
| + issueBook(m:      |       | - creationDate:LocalDate|
|   Member, b:        |       | - status: ResvStatus    |
|   BookItem): Loan   |       +-------------------------+
| + returnBook(l:     |       | + cancel(): void        |
|   Loan): void       |       +-------------------------+
| + renewLoan(l: Loan)|
+---------------------+
```

**Key relationships:**
- `Library` <>---> `Catalog` : Composition (1 to 1)
- `Book` <>---> `BookItem` : Composition (1 to 1..*)
- `Member` <>---> `Account` : Composition (1 to 1)
- `Loan` ------> `BookItem` : Association (many to 1)
- `Loan` ------> `Member` : Association (many to 1)
- `Reservation` ------> `Member` : Association
- `Reservation` ------> `Book` : Association

---

### 2. Parking Lot System

**Key Classes/Interfaces:**

| Class/Interface     | Responsibility                                       |
|---------------------|------------------------------------------------------|
| `ParkingLot`        | Top-level aggregate managing floors and entrances    |
| `ParkingFloor`      | One level of the lot, contains spots                 |
| `ParkingSpot`       | Single stall with type and occupancy status          |
| `Vehicle`           | Abstract base for all vehicle types                  |
| `Car/Bike/Truck`    | Concrete vehicle types                               |
| `ParkingTicket`     | Issued at entry, resolved at exit                    |
| `EntrancePanel`     | Entry gate — prints tickets                          |
| `ExitPanel`         | Exit gate — collects fees                           |
| `ParkingRate`       | Strategy for fee calculation                        |
| `Payment`           | Abstract payment; subclassed by Cash/Card           |
| `SpotType`          | Enum: COMPACT, LARGE, HANDICAPPED, MOTORCYCLE       |
| `SpotStatus`        | Enum: FREE, OCCUPIED, RESERVED, OUT_OF_ORDER        |

**ASCII Class Diagram:**

```
+-------------------------+
|       ParkingLot        |
+-------------------------+
| - lotId: String         |
| - name: String          |
| - address: String       |
| - floors:               |
|   List<ParkingFloor>    |
| - entrances:            |
|   List<EntrancePanel>   |
| - exits:                |
|   List<ExitPanel>       |
+-------------------------+
| + getAvailableSpots(    |
|   v: Vehicle):          |
|   List<ParkingSpot>     |
| + isFull(): boolean     |
+-------------------------+
        | 1
        | <>
        | 1..*
+-------------------------+          +-------------------------+
|      ParkingFloor       |          |       ParkingSpot       |
+-------------------------+          +-------------------------+
| - floorNumber: int      | 1  <>--> | - spotId: String        |
| - spots:                | 1..*     | - spotType: SpotType    |
|   List<ParkingSpot>     |          | - status: SpotStatus    |
+-------------------------+          | - vehicle: Vehicle      |
| + getFreeSpots(         |          +-------------------------+
|   type: SpotType):      |          | + assign(v: Vehicle)    |
|   List<ParkingSpot>     |          | + free(): void          |
| + getFloorStatus():     |          | + canFit(v: Vehicle):   |
|   Map<SpotType, Integer>|          |   boolean               |
+-------------------------+          +-------------------------+

+-------------------------+          +-------------------------+
|    <<abstract>>         |          |      ParkingTicket      |
|        Vehicle          |          +-------------------------+
+-------------------------+          | - ticketId: String      |
| - licenseNo: String     |          | - vehicle: Vehicle      |
| - vehicleType: VehicleType|        | - spot: ParkingSpot     |
+-------------------------+          | - entryTime: Instant    |
| + getType(): VehicleType|          | - exitTime: Instant     |
+-------------------------+          | - fee: double           |
         ^                           | - status: TicketStatus  |
    +----|----+                       +-------------------------+
    |         |                      | + calculateFee(         |
+-------+ +-------+                  |   rate: ParkingRate):   |
|  Car  | | Bike  |                  |   double                |
+-------+ +-------+                  +-------------------------+
| + getType() | + getType()|
+-------+ +-------+

+-------------------------+          +-------------------------+
|     EntrancePanel       |          |       ExitPanel         |
+-------------------------+          +-------------------------+
| - id: String            |          | - id: String            |
+-------------------------+          +-------------------------+
| + printTicket(          |          | + processExit(          |
|   v: Vehicle):          |          |   ticket: ParkingTicket,|
|   ParkingTicket         |          |   payment: Payment):    |
+-------------------------+          |   Receipt               |
                                     +-------------------------+

+---------------------------+        +-------------------------+
|    <<interface>>          |        |   <<abstract>>          |
|       ParkingRate         |        |        Payment          |
+---------------------------+        +-------------------------+
| + calculateFee(           |        | - amount: double        |
|   ticket: ParkingTicket): |        +-------------------------+
|   double                  |        | + processPayment():     |
+---------------------------+        |   boolean               |
         ^                           +-------------------------+
    +----|----+                              ^
    |         |                        +----+----+
+----------+ +----------+          +--------+ +----------+
|HourlyRate| |FlatRate  |          | Cash   | | Card     |
+----------+ +----------+          +--------+ +----------+
```

**Key relationships:**
- `ParkingLot` <>---> `ParkingFloor` : Composition (1 to 1..*)
- `ParkingFloor` <>---> `ParkingSpot` : Composition (1 to 1..*)
- `ParkingSpot` ------> `Vehicle` : Association (0..1 to 1)
- `ParkingTicket` ------> `Vehicle` : Association
- `ParkingTicket` ------> `ParkingSpot` : Association
- `Car/Bike` ---|> `Vehicle` : Inheritance
- `HourlyRate/FlatRate` ---|> `ParkingRate` : Realization
- `Cash/Card` ---|> `Payment` : Inheritance

---

### 3. Hotel Booking System

**Key Classes/Interfaces:**

| Class/Interface     | Responsibility                                       |
|---------------------|------------------------------------------------------|
| `Hotel`             | Aggregate root managing rooms and bookings           |
| `Room`              | Physical room with type, status, price               |
| `RoomType`          | Enum: SINGLE, DOUBLE, SUITE, DELUXE                  |
| `RoomStatus`        | Enum: AVAILABLE, BOOKED, OCCUPIED, MAINTENANCE       |
| `Guest`             | Person making reservations                          |
| `Reservation`       | Links guest to room for a date range                |
| `ReservationStatus` | Enum: CONFIRMED, CANCELLED, CHECKED_IN, CHECKED_OUT  |
| `CheckIn`           | Records actual arrival event                        |
| `CheckOut`          | Records departure and triggers billing               |
| `Invoice`           | Bill for a reservation including extra charges      |
| `RoomService`       | Optional in-room order tied to a reservation        |
| `HotelStaff`        | Employee who processes check-ins/outs               |
| `SearchCriteria`    | Value object for room search parameters             |

**ASCII Class Diagram:**

```
+---------------------------+
|          Hotel            |
+---------------------------+
| - hotelId: String         |
| - name: String            |
| - address: Address        |
| - rooms: List<Room>       |
| - staff: List<HotelStaff> |
+---------------------------+
| + searchRooms(            |
|   criteria: SearchCriteria|
|   ): List<Room>           |
| + makeReservation(g:      |
|   Guest, r: Room,         |
|   from/to: LocalDate):    |
|   Reservation             |
+---------------------------+
        |1               |1
        |<>              |<>
        |1..*            |1..*
+------------------+   +------------------+
|       Room       |   |    HotelStaff    |
+------------------+   +------------------+
| - roomNumber:    |   | - staffId: String|
|   String         |   | - name: String   |
| - floor: int     |   | - role: StaffRole|
| - type: RoomType |   +------------------+
| - status:        |   | + processCheckIn |
|   RoomStatus     |   |   (r: Reservation|
| - pricePerNight: |   |   ): CheckIn     |
|   double         |   | + processCheckOut|
+------------------+   |   (r: Reservation|
| + isAvailable(   |   |   ): CheckOut    |
|   from: LocalDate|   +------------------+
|   to: LocalDate):|
|   boolean        |
+------------------+
        |
        | 1
        | (referenced by)
        | *
+---------------------------+         +---------------------------+
|        Reservation        |         |           Guest           |
+---------------------------+         +---------------------------+
| - reservationId: String   |         | - guestId: String         |
| - guest: Guest            |-------->| - name: String            |
| - room: Room              |         | - email: String           |
| - checkInDate: LocalDate  |         | - phone: String           |
| - checkOutDate: LocalDate |         | - loyaltyPoints: int      |
| - status: ResvStatus      |         +---------------------------+
| - invoice: Invoice        |         | + getReservationHistory() |
+---------------------------+         |   : List<Reservation>     |
| + cancel(): void          |         +---------------------------+
| + modify(from: LocalDate, |
|   to: LocalDate): void    |
+---------------------------+
        | 1
        | <>
        | 1
+---------------------------+
|         Invoice           |
+---------------------------+
| - invoiceId: String       |
| - lineItems:              |
|   List<InvoiceItem>       |
| - totalAmount: double     |
| - paymentStatus:          |
|   PaymentStatus           |
+---------------------------+
| + addCharge(item:         |
|   InvoiceItem): void      |
| + calculateTotal(): double|
| + pay(p: Payment): Receipt|
+---------------------------+
        ^
        |
+---------------------------+
|        RoomService        |
+---------------------------+
| - orderId: String         |
| - reservation: Reservation|
| - items: List<ServiceItem>|
| - orderTime: Instant      |
+---------------------------+
| + addToInvoice(           |
|   inv: Invoice): void     |
+---------------------------+
```

**Key relationships:**
- `Hotel` <>---> `Room` : Composition (1 to 1..*)
- `Hotel` <>---> `HotelStaff` : Aggregation (1 to 1..*)
- `Reservation` ------> `Guest` : Association (many to 1)
- `Reservation` ------> `Room` : Association (many to 1)
- `Reservation` <>---> `Invoice` : Composition (1 to 1)
- `RoomService` ------> `Reservation` : Association

---

### 4. Ride Sharing System (Uber)

**Key Classes/Interfaces:**

| Class/Interface      | Responsibility                                          |
|----------------------|---------------------------------------------------------|
| `RideSharingApp`     | Facade / top-level orchestrator                        |
| `Rider`              | Person requesting a ride                               |
| `Driver`             | Person providing a ride                                |
| `Vehicle`            | Car associated with a driver                           |
| `Trip`               | Core domain object representing one ride               |
| `TripStatus`         | Enum: REQUESTED, DRIVER_ASSIGNED, EN_ROUTE, COMPLETED, CANCELLED |
| `Location`           | Value object: latitude, longitude                      |
| `Route`              | Planned path from origin to destination                |
| `TripRequest`        | Rider's request with pickup, dropoff, type             |
| `FareCalculator`     | Interface for pricing strategy                        |
| `Payment`            | Abstract payment (CreditCard, Wallet, Cash)           |
| `Rating`             | Feedback from rider or driver post-trip               |
| `DriverMatcher`      | Interface: algorithm to find nearest available driver  |
| `Notification`       | Pushed to rider/driver on state changes               |

**ASCII Class Diagram:**

```
+---------------------------+
|       RideSharingApp      |
+---------------------------+
| - matcher: DriverMatcher  |
| - fareCalc: FareCalculator|
+---------------------------+
| + requestRide(r: Rider,   |
|   req: TripRequest):Trip  |
| + cancelTrip(t: Trip):void|
+---------------------------+
        |
        | uses
        v
+---------------------------+         +---------------------------+
|    <<interface>>          |         |    <<interface>>          |
|     DriverMatcher         |         |     FareCalculator        |
+---------------------------+         +---------------------------+
| + findDriver(             |         | + calculate(              |
|   req: TripRequest):      |         |   trip: Trip): double     |
|   Driver                  |         +---------------------------+
+---------------------------+                  ^
        ^                                      |
+------------------+               +-----------+-----------+
| NearestMatcher   |          +----------+   +-------------+
+------------------+          |SurgeFare |   | StandardFare|
| + findDriver()   |          +----------+   +-------------+
+------------------+

+---------------------------+         +---------------------------+
|          Rider            |         |          Driver           |
+---------------------------+         +---------------------------+
| - riderId: String         |         | - driverId: String        |
| - name: String            |         | - name: String            |
| - phone: String           |         | - licenseNo: String       |
| - rating: double          |         | - rating: double          |
| - paymentMethod: Payment  |         | - status: DriverStatus    |
| - location: Location      |         | - currentLocation:        |
+---------------------------+         |   Location                |
| + requestRide(            |         | - vehicle: Vehicle        |
|   req: TripRequest): Trip |         +---------------------------+
| + rateDriver(             |         | + acceptTrip(t: Trip):void|
|   d: Driver,              |         | + updateLocation(         |
|   r: Rating): void        |         |   l: Location): void      |
+---------------------------+         | + completeTrip(): void    |
                                      +---------------------------+
                                               | 1
                                               | <>
                                               | 1
                                      +---------------------------+
                                      |          Vehicle          |
                                      +---------------------------+
                                      | - plateNumber: String     |
                                      | - model: String           |
                                      | - capacity: int           |
                                      | - vehicleType: VehicleType|
                                      +---------------------------+

+---------------------------+
|           Trip            |
+---------------------------+
| - tripId: String          |
| - rider: Rider            |
| - driver: Driver          |
| - pickupLocation: Location|
| - dropoffLocation: Location|
| - route: Route            |
| - status: TripStatus      |
| - requestTime: Instant    |
| - startTime: Instant      |
| - endTime: Instant        |
| - fare: double            |
+---------------------------+
| + assignDriver(           |
|   d: Driver): void        |
| + start(): void           |
| + complete(): void        |
| + cancel(): void          |
+---------------------------+
        | 1
        | <>
        | 1
+---------------------------+
|          Payment          |
+---------------------------+
| - amount: double          |
| - status: PaymentStatus   |
+---------------------------+
| + process(): boolean      |
+---------------------------+
         ^
    +----+----+
    |         |
+--------+ +--------+
|CreditCard| |Wallet |
+--------+ +--------+

+---------------------------+
|          Rating           |
+---------------------------+
| - ratingId: String        |
| - trip: Trip              |
| - ratedBy: String         |  (riderId or driverId)
| - score: int              |  (1-5)
| - comment: String         |
+---------------------------+
```

**Key relationships:**
- `RideSharingApp` --> `DriverMatcher` : Association (holds interface ref)
- `RideSharingApp` --> `FareCalculator` : Association
- `Trip` ------> `Rider` : Association (many to 1)
- `Trip` ------> `Driver` : Association (many to 1)
- `Driver` <>---> `Vehicle` : Composition (1 to 1)
- `Trip` <>---> `Payment` : Composition (1 to 1)
- `NearestMatcher` ---|> `DriverMatcher` : Realization
- `SurgeFare/StandardFare` ---|> `FareCalculator` : Realization
- `CreditCard/Wallet` ---|> `Payment` : Inheritance

---

### 5. ATM System

**Key Classes/Interfaces:**

| Class/Interface     | Responsibility                                         |
|---------------------|--------------------------------------------------------|
| `ATM`               | Physical machine; orchestrates all operations         |
| `CardReader`        | Reads magnetic stripe / chip data                    |
| `KeyPad`            | PIN entry device                                     |
| `Screen`            | Displays UI state                                    |
| `CashDispenser`     | Holds and dispenses physical cash                    |
| `ReceiptPrinter`    | Prints transaction receipts                          |
| `NetworkInterface`  | Communicates with bank's backend                    |
| `BankAccount`       | Customer's account on the bank side                  |
| `Card`              | Debit/credit card data (number, expiry)              |
| `Transaction`       | Abstract: Withdrawal, Deposit, BalanceInquiry, Transfer |
| `ATMState`          | Interface for State pattern: Idle, HasCard, Authenticated, Dispensing |
| `Session`           | Active authenticated session for one card use        |
| `TransactionResult` | Value object returned after each transaction attempt  |

**ASCII Class Diagram:**

```
+-------------------------------+
|             ATM               |
+-------------------------------+
| - atmId: String               |
| - location: String            |
| - currentState: ATMState      |
| - session: Session            |
| - cardReader: CardReader      |
| - keyPad: KeyPad              |
| - screen: Screen              |
| - cashDispenser: CashDispenser|
| - printer: ReceiptPrinter     |
| - network: NetworkInterface   |
+-------------------------------+
| + insertCard(c: Card): void   |
| + enterPin(pin: String): void |
| + selectTransaction(          |
|   type: TxType): void         |
| + cancel(): void              |
+-------------------------------+
        | 1 <>
        |         to hardware components (composition)
        |
+---------------+  +---------------+  +---------------------+
|  CardReader   |  |  CashDispenser|  |   ReceiptPrinter    |
+---------------+  +---------------+  +---------------------+
| + readCard(): |  | - cashLevel:  |  | + print(receipt:    |
|   Card        |  |   Map<int,int>|  |   String): void     |
| + ejectCard() |  +---------------+  +---------------------+
+---------------+  | + dispense(   |
                   |   amount:int):
                   |   boolean     |
                   | + getCashLevel|
                   |   (): int     |
                   +---------------+

+-------------------------------+
|        <<interface>>          |
|           ATMState            |
+-------------------------------+
| + insertCard(atm: ATM,        |
|   card: Card): void           |
| + enterPin(atm: ATM,          |
|   pin: String): void          |
| + selectTransaction(          |
|   atm: ATM,                   |
|   type: TxType): void         |
| + cancel(atm: ATM): void      |
+-------------------------------+
             ^
    +--------+---------+--------+
    |        |                  |
+----------+ +------------------+ +-----------------+
| IdleState| |HasCardState      | |AuthenticatedState|
+----------+ +------------------+ +-----------------+
(inserts card)  (validates PIN)   (selects transaction)

+-------------------------------+         +-------------------------------+
|           Session             |         |       NetworkInterface        |
+-------------------------------+         +-------------------------------+
| - sessionId: String           |         +-------------------------------+
| - card: Card                  |         | + validateCard(c: Card,       |
| - authenticated: boolean      |         |   pin: String): boolean       |
| - transactions:               |         | + getBalance(                 |
|   List<Transaction>           |         |   acctNo: String): double     |
| - startTime: Instant          |         | + debit(acctNo: String,       |
+-------------------------------+         |   amount: double):            |
| + authenticate(               |         |   TransactionResult           |
|   pin: String): boolean       |         | + credit(acctNo: String,      |
| + end(): void                 |         |   amount: double):            |
+-------------------------------+         |   TransactionResult           |
                                          +-------------------------------+

+-------------------------------+
|     <<abstract>>              |
|         Transaction           |
+-------------------------------+
| - transactionId: String       |
| - accountNumber: String       |
| - timestamp: Instant          |
| - amount: double              |
| - result: TransactionResult   |
+-------------------------------+
| + execute(net:                |
|   NetworkInterface):          |
|   TransactionResult           |
+-------------------------------+
             ^
    +--------+---------+--------+
    |                  |        |
+------------+ +----------+ +----------+
| Withdrawal | | Deposit  | | Balance  |
|            | |          | | Inquiry  |
+------------+ +----------+ +----------+
| + execute()| |+ execute()| |+ execute()|
+------------+ +----------+ +----------+

+-------------------------------+
|             Card              |
+-------------------------------+
| - cardNumber: String          |
| - holderName: String          |
| - expiryDate: YearMonth       |
| - cardType: CardType          |
| - associatedAccountNo: String |
+-------------------------------+
```

**Key relationships:**
- `ATM` <>---> `CardReader`, `CashDispenser`, `ReceiptPrinter`, `KeyPad`, `Screen` : Composition (1 to 1 each)
- `ATM` --> `ATMState` : Association (current state, changes at runtime)
- `ATM` <>---> `Session` : Composition (0..1 active session at a time)
- `ATM` --> `NetworkInterface` : Association
- `Session` -->  `Card` : Association
- `Session` <>---> `Transaction` : Composition (1 to 0..*)
- `IdleState`, `HasCardState`, `AuthenticatedState` ---|> `ATMState` : Realization
- `Withdrawal`, `Deposit`, `BalanceInquiry` ---|> `Transaction` : Inheritance
- `Transaction` --> `NetworkInterface` : Dependency (uses in `execute()`)
```
