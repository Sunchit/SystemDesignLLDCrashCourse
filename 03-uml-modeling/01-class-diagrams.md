# Class Diagrams

> Class diagrams show the **static structure** of a system: what classes exist, what data they hold, what operations they perform, and how they relate to each other.

---

## Table of Contents

1. [Class Notation](#1-class-notation)
2. [Relationships](#2-relationships)
3. [Multiplicity](#3-multiplicity)
4. [Abstract Classes and Interfaces in UML](#4-abstract-classes-and-interfaces-in-uml)
5. [Example: E-Commerce Domain](#5-example-e-commerce-domain)
6. [Example: Library Management System](#6-example-library-management-system)
7. [Example: Parking Lot System](#7-example-parking-lot-system)
8. [How to Draw Class Diagrams in Interviews](#8-how-to-draw-class-diagrams-in-interviews)
9. [Common Mistakes](#9-common-mistakes)
10. [Exercises](#10-exercises)

---

## 1. Class Notation

A class is drawn as a rectangle divided into three compartments:

```
+---------------------------+
|       ClassName           |  <- 1. Name compartment
+---------------------------+
| - privateField: Type      |  <- 2. Attribute compartment
| # protectedField: Type    |
| + publicField: Type       |
| ~ packageField: Type      |
+---------------------------+
| + publicMethod(): Type    |  <- 3. Operation compartment
| - privateMethod(): void   |
| # protectedMethod(): Type |
+---------------------------+
```

### Visibility Modifiers

| Symbol | Visibility | Java Keyword |
|--------|-----------|--------------|
| `+` | Public | `public` |
| `-` | Private | `private` |
| `#` | Protected | `protected` |
| `~` | Package | (default) |

### Attribute Syntax

```
visibility name : type [multiplicity] = defaultValue
```

Examples:
```
- name    : String
- age     : int = 0
- tags    : String[*]
- status  : OrderStatus = PENDING
```

### Operation Syntax

```
visibility name(param: type, ...) : returnType
```

Examples:
```
+ calculateTotal() : double
+ findById(id: long) : Optional<Order>
- validateAge(age: int) : boolean
```

### Static Members

Underlined in UML:
```
+ MAX_SIZE : int = 100     (underlined = static)
+ getInstance() : Singleton (underlined = static)
```

### Abstract Classes and Methods

Class name and abstract method names are written in *italics* in formal UML. In text-based notation, use `<<abstract>>` or `{abstract}`.

---

## 2. Relationships

### 2.1 Association

**Meaning:** Class A has a reference to class B. The association is a "uses" relationship.

**Java:** A field of type B in class A.

**Notation:** A solid line, optionally with an open arrowhead showing direction.

```
+----------+           +----------+
|  Driver  |---------->|   Car    |
+----------+           +----------+

A Driver is associated with a Car.
The arrow shows: Driver knows about Car (not necessarily vice versa).
```

**Bidirectional association** (no arrowhead, or arrowheads on both ends):
```
+----------+           +----------+
|  Student |<--------->|  Course  |
+----------+           +----------+

Students know about their courses; courses know about their students.
```

### 2.2 Aggregation

**Meaning:** A "whole-part" relationship where the part can exist independently of the whole. If the whole is destroyed, the part survives.

**Java:** The part is created outside the whole and passed in (usually via constructor).

**Notation:** A solid line with an open diamond on the whole's end.

```
+------------+           +----------+
| University |<>-------->| Student  |
+------------+           +----------+

A University has Students.
If the University closes, the Students still exist.
Diamond is on the "whole" (University) side.
```

```java
public class University {
    private List<Student> students; // aggregation — students passed in, not owned

    public University(List<Student> students) {
        this.students = students; // University doesn't create students
    }
}
```

### 2.3 Composition

**Meaning:** A "whole-part" relationship where the part cannot exist independently of the whole. If the whole is destroyed, the part is destroyed too.

**Java:** The part is created by and owned by the whole (typically instantiated in the constructor).

**Notation:** A solid line with a **filled** diamond on the whole's end.

```
+---------+           +---------+
|  House  |<#>------->|  Room   |
+---------+           +---------+

A House is composed of Rooms.
A Room cannot exist without a House.
Filled diamond is on the "whole" (House) side.
```

```java
public class House {
    private final List<Room> rooms; // composition — House creates and owns Rooms

    public House(int numberOfRooms) {
        this.rooms = new ArrayList<>();
        for (int i = 0; i < numberOfRooms; i++) {
            this.rooms.add(new Room("Room " + (i + 1))); // House creates Rooms
        }
    }
}
```

### Aggregation vs Composition: The Key Question

**Ask:** "If I delete the container, does the contained object also stop existing?"

- If YES → Composition (filled diamond)
- If NO → Aggregation (open diamond)

| Scenario | Relationship |
|----------|-------------|
| Order → OrderItem | Composition (items don't exist without the order) |
| Department → Employee | Aggregation (employees survive the department) |
| Library → Book | Aggregation (books exist independently of the library) |
| Car → Engine | Composition (a car's engine is specific to that car) |

### 2.4 Inheritance (Generalization)

**Meaning:** Is-a relationship. The subclass is a specialization of the superclass.

**Java:** `extends`

**Notation:** A solid line with an open (unfilled) triangle/arrowhead pointing to the superclass.

```
          +--------+
          | Animal |
          +--------+
              |>
         _____|_____
        |           |
    +-------+   +-------+
    |  Dog  |   |  Cat  |
    +-------+   +-------+

Arrow points FROM child TO parent (child extends parent).
```

### 2.5 Realization (Interface Implementation)

**Meaning:** A class implements an interface contract.

**Java:** `implements`

**Notation:** A dashed line with an open triangle/arrowhead pointing to the interface.

```
<<interface>>
+----------+
| Flyable  |
+----------+
| + fly()  |
+----------+
     |>
  (dashed)
     |
+-------+
| Eagle |
+-------+

Dashed arrow: Eagle realizes (implements) Flyable.
```

### 2.6 Dependency

**Meaning:** Class A uses class B temporarily — B appears as a method parameter, return type, or local variable — but A does not hold a permanent reference to B.

**Java:** B is a method parameter or local variable, not a field.

**Notation:** A dashed line with a regular open arrowhead.

```
+--------+   - - - ->  +-----------+
| Mailer |              | EmailData |
+--------+              +-----------+

Mailer uses EmailData in its send() method but doesn't hold a reference.
```

### Relationship Summary

```
Association:   A ————————> B    (A has field of type B)
Aggregation:   A <>————————> B  (A has B, B can survive without A)
Composition:   A <#>———————> B  (A owns B, B cannot survive without A)
Inheritance:   A ————————|> B   (A extends B, solid arrow)
Realization:   A - - - - -|> B  (A implements B, dashed arrow)
Dependency:    A - - - - -> B   (A uses B temporarily)
```

---

## 3. Multiplicity

Multiplicity notation specifies how many instances participate in a relationship.

| Notation | Meaning |
|----------|---------|
| `1` | Exactly one |
| `0..1` | Zero or one (optional) |
| `*` or `0..*` | Zero or more |
| `1..*` | One or more |
| `2..4` | Between two and four |
| `n` | Exactly n |

**Placement:** Multiplicity goes at each end of the relationship line, near the class whose count it describes.

```
+----------+  1      *  +-----------+
|  Order   |<#>---------|  OrderItem|
+----------+             +-----------+

One Order has many (zero or more) OrderItems.
Each OrderItem belongs to exactly one Order.
```

```
+---------+  1      0..1  +----------+
|  User   |<>-------------|  Profile |
+---------+               +----------+

One User has zero or one Profile.
Each Profile belongs to exactly one User.
```

```
+----------+  *        *  +----------+
|  Student |<>------------|  Course  |
+----------+              +----------+

Many Students can enroll in many Courses.
```

---

## 4. Abstract Classes and Interfaces in UML

### Abstract Classes

```
+----------------------+
| <<abstract>>         |     <- stereotype label
| Shape                |     (or: class name in italics)
+----------------------+
| # color: Color       |
+----------------------+
| + area(): double     |     <- abstract method (italics or {abstract})
| + draw(): void       |
+----------------------+
```

### Interfaces

```
+----------------------+
| <<interface>>        |     <- interface stereotype
| Serializable         |
+----------------------+
|                      |     <- usually no attributes
+----------------------+
| + serialize(): byte[]|
| + deserialize(b:byte[]) |
+----------------------+
```

### Enum

```
+----------------------+
| <<enumeration>>      |
| OrderStatus          |
+----------------------+
| PENDING              |
| CONFIRMED            |
| SHIPPED              |
| DELIVERED            |
| CANCELLED            |
+----------------------+
```

---

## 5. Example: E-Commerce Domain

```
                      +-------------------+
                      | <<enumeration>>   |
                      | OrderStatus       |
                      +-------------------+
                      | PENDING           |
                      | CONFIRMED         |
                      | SHIPPED           |
                      | DELIVERED         |
                      | CANCELLED         |
                      +-------------------+

+------------+                                     +---------------+
|  Customer  |                                     |    Product    |
+------------+                                     +---------------+
|- id: long  |                                     |- id: long     |
|- name: Str |                                     |- name: String |
|- email: Str|                                     |- price: Money |
+------------+                                     |- stock: int   |
| +getOrders()|                                    +---------------+
+------------+                                     |+getPrice():Money|
      |1                                           +---------------+
      |                                                   |
      | places                                            | contains
      |                                                   |
      |*                                           --------
+-------------------+   1         *   +-----------+
|      Order        |<#>————————————|>| OrderItem |
+-------------------+                 +-----------+
|- id: long         |                 |- quantity:int|
|- createdAt: LDT   |                 |- unitPrice:Money|
|- status:OrderStatus|                +-----------+
|- total: Money     |                 |+getSubtotal():Money|
+-------------------+                 +-----------+
|+addItem(i:OrderItem)|
|+calculateTotal():Money|
|+cancel(): void    |
+-------------------+
         |1
         | ships to
         |0..1
+-------------------+
|    ShippingAddress|
+-------------------+
|- street: String   |
|- city: String     |
|- zipCode: String  |
|- country: String  |
+-------------------+
         |
         | paid via
         |
+-------------------+
| <<abstract>>      |
|   Payment         |
+-------------------+
|- amount: Money    |
|- paidAt: LDT      |
+-------------------+
|+{abstract}process()|
+-------------------+
         |>
    _____|_____
   |           |
+--------+  +--------+
|CreditCard| |PayPal  |
+--------+  +--------+
|- cardNo: String| |- email:String|
+--------+  +--------+
|+process()| |+process()|
+--------+  +--------+
```

**Java mapping:**

```java
public class Customer {
    private long id;
    private String name;
    private String email;
    private List<Order> orders; // 1-to-many association
}

public class Order {
    private long id;
    private Customer customer;        // many-to-one
    private List<OrderItem> items;    // composition (1 to *)
    private ShippingAddress address;  // composition (1 to 0..1)
    private Payment payment;          // association
    private OrderStatus status;
    private Money total;

    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

public class OrderItem {
    private Product product;   // association (many-to-1)
    private int quantity;
    private Money unitPrice;

    public Money getSubtotal() {
        return unitPrice.multiply(quantity);
    }
}

public abstract class Payment {
    private Money amount;
    private LocalDateTime paidAt;

    public abstract void process();
}

public class CreditCardPayment extends Payment {
    private String cardNumber;

    @Override
    public void process() { /* charge card */ }
}
```

---

## 6. Example: Library Management System

```
+------------------+        +------------------+
| <<interface>>    |        | <<interface>>    |
| Searchable       |        | Borrowable       |
+------------------+        +------------------+
|+search(q:String):|        |+borrow(m:Member):|
|  List<LibItem>   |        |+returnItem():void|
+------------------+        +------------------+
         |>                          |>
    (dashed)                     (dashed)
         |                           |
+----------------------------+
|         Library            |
+----------------------------+
|- name: String              |
|- address: String           |
+----------------------------+
|+addBook(b:Book): void      |
|+search(q:String): List     |
|+issueLoan(m:Member,b:Book) |
+----------------------------+
     |1              |1
     |               |
     | has            | manages
     |*              |*
+----------+    +-----------+
|   Book   |    |   Member  |
+----------+    +-----------+
|- isbn: Str|    |- id: long  |
|- title:Str|    |- name: Str |
|- author:Str|   |- email: Str|
|- copies:int|   +------------+
+----------+    |+getLoanCount()|
|+isAvailable()| +-----------+
+----------+
     |1
     |
     | has many
     |*
+-------------+
| BookCopy    |
+-------------+
|- copyId: Str|
|- condition  |
|  : Condition|
+-------------+
|+isAvailable()|
+-------------+
     |
     | issued as
     |
+-------------+
|    Loan     |
+-------------+
|- issueDate:LD|
|- dueDate: LD |
|- returnDate:LD|
+-------------+
|+isOverdue():bool|
|+getDaysLate():int|
+-------------+
```

**Java mapping:**

```java
public interface Searchable {
    List<LibraryItem> search(String query);
}

public interface Borrowable {
    void borrow(Member member);
    void returnItem();
}

public class Library implements Searchable {
    private final String name;
    private final List<Book> catalog = new ArrayList<>();    // composition
    private final List<Member> members = new ArrayList<>();  // composition

    @Override
    public List<LibraryItem> search(String query) {
        return catalog.stream()
            .filter(book -> book.getTitle().toLowerCase().contains(query.toLowerCase()))
            .collect(Collectors.toList());
    }

    public Loan issueLoan(Member member, Book book) {
        BookCopy copy = book.getAvailableCopy()
            .orElseThrow(() -> new NoCopiesAvailableException(book.getIsbn()));
        Loan loan = new Loan(member, copy, LocalDate.now(), LocalDate.now().plusDays(14));
        copy.setLoaned(true);
        return loan;
    }
}

public class Loan {
    private final Member member;
    private final BookCopy copy;
    private final LocalDate issueDate;
    private final LocalDate dueDate;
    private LocalDate returnDate;

    public boolean isOverdue() {
        LocalDate checkDate = returnDate != null ? returnDate : LocalDate.now();
        return checkDate.isAfter(dueDate);
    }

    public int getDaysLate() {
        if (!isOverdue()) return 0;
        LocalDate checkDate = returnDate != null ? returnDate : LocalDate.now();
        return (int) ChronoUnit.DAYS.between(dueDate, checkDate);
    }
}
```

---

## 7. Example: Parking Lot System

```
+---------------------+
|     ParkingLot      |
+---------------------+
|- name: String       |
|- address: String    |
|- capacity: int      |
+---------------------+
|+getAvailableSpots():int|
|+findSpot(v:Vehicle):ParkingSpot|
|+issueTicket(v:Vehicle):Ticket|
|+processPayment(t:Ticket,p:Payment):Receipt|
+---------------------+
     |1             |1
     |               |
     |<#>            |<#>
     |*              |*
+---------------+  +---------------+
| ParkingFloor  |  |  EntryPanel   |
+---------------+  +---------------+
|- floorNumber:int|  |- panelId: String|
+---------------+  +---------------+
     |1              |
     |<#>            |
     |*
+------------------+
|   ParkingSpot    |
+------------------+
|- spotNumber: int |
|- type: SpotType  |
|- isOccupied: bool|
+------------------+
|+occupy(v:Vehicle)|
|+vacate(): void   |
+------------------+

+--------------------+
| <<enumeration>>    |
| SpotType           |
+--------------------+
| COMPACT            |
| REGULAR            |
| LARGE              |
| HANDICAPPED        |
| EV_CHARGING        |
+--------------------+

+------------------+        +------------------+
| <<abstract>>     |        | <<abstract>>     |
|    Vehicle       |        |    Payment       |
+------------------+        +------------------+
|- licensePlate:Str|        |- amount: double  |
|- type: VehicleType|       |- paidAt: LDT     |
+------------------+        +------------------+
|+getType():VehicleType|    |+{abstract}        |
+------------------+        |processPayment()   |
       |>                   +------------------+
  _____|_____                      |>
 |     |     |               ______|______
+---+ +--+ +----+          |             |
|Car| |SUV| |Truck|     +--------+  +----------+
+---+ +--+ +----+     |  Cash  |  | CreditCard|
                       +--------+  +----------+

+------------------+
|     Ticket       |
+------------------+
|- ticketId: String|
|- issuedAt: LDT   |
|- exitedAt: LDT   |
|- vehicle: Vehicle|
|- spot: ParkingSpot|
+------------------+
|+getDuration():Duration|
|+calculateFee():double|
+------------------+
```

**Java mapping:**

```java
public class ParkingLot {
    private final String name;
    private final List<ParkingFloor> floors;  // composition
    private final List<EntryPanel> panels;    // composition
    private final PricingStrategy pricing;    // aggregation (injected)

    public ParkingLot(String name, int numberOfFloors, int spotsPerFloor,
                      PricingStrategy pricing) {
        this.name = name;
        this.pricing = pricing;
        this.floors = new ArrayList<>();
        for (int i = 1; i <= numberOfFloors; i++) {
            floors.add(new ParkingFloor(i, spotsPerFloor));
        }
        this.panels = new ArrayList<>();
    }

    public Optional<ParkingSpot> findAvailableSpot(VehicleType type) {
        return floors.stream()
            .flatMap(floor -> floor.getSpots().stream())
            .filter(spot -> !spot.isOccupied() && spot.getType().accepts(type))
            .findFirst();
    }

    public Ticket issueTicket(Vehicle vehicle) {
        ParkingSpot spot = findAvailableSpot(vehicle.getType())
            .orElseThrow(() -> new ParkingLotFullException("No available spots"));
        spot.occupy(vehicle);
        return new Ticket(UUID.randomUUID().toString(), vehicle, spot, LocalDateTime.now());
    }

    public Receipt processExit(Ticket ticket, Payment payment) {
        Duration duration = ticket.getDuration();
        double fee = pricing.calculate(duration, ticket.getSpot().getType());
        payment.process(fee);
        ticket.getSpot().vacate();
        ticket.setExitedAt(LocalDateTime.now());
        return new Receipt(ticket, fee, payment);
    }
}

public class ParkingSpot {
    private final int spotNumber;
    private final SpotType type;
    private boolean occupied;
    private Vehicle currentVehicle;

    public void occupy(Vehicle vehicle) {
        if (occupied) throw new IllegalStateException("Spot already occupied");
        this.currentVehicle = vehicle;
        this.occupied = true;
    }

    public void vacate() {
        this.currentVehicle = null;
        this.occupied = false;
    }

    public boolean isOccupied() { return occupied; }
    public SpotType getType()   { return type; }
}

public class Ticket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final LocalDateTime issuedAt;
    private LocalDateTime exitedAt;

    public Duration getDuration() {
        LocalDateTime end = exitedAt != null ? exitedAt : LocalDateTime.now();
        return Duration.between(issuedAt, end);
    }
}

public abstract class Vehicle {
    private final String licensePlate;
    private final VehicleType type;

    public abstract VehicleType getType();
}

public class Car extends Vehicle {
    @Override
    public VehicleType getType() { return VehicleType.COMPACT; }
}
```

---

## 8. How to Draw Class Diagrams in Interviews

### Step-by-Step Process (5-10 minutes)

**Step 1: Identify the nouns (2 minutes)**
Read the problem statement. Underline every noun — each is a potential class.
"Design a parking lot system" → ParkingLot, Floor, Spot, Vehicle, Ticket, Payment, Receipt

**Step 2: Filter to meaningful domain classes (1 minute)**
Remove obvious non-entities (words like "system", "design") and group synonyms.

**Step 3: Identify attributes for each class (1 minute)**
Just the important ones — not getters/setters, not trivial fields.

**Step 4: Identify relationships (2-3 minutes)**
For each pair of classes: does one have the other? Is one a kind of the other? Does one create the other?

**Step 5: Add multiplicity (30 seconds)**
Check each relationship: how many on each side?

**Step 6: Identify abstract classes and interfaces (1 minute)**
Is there common behavior? Can callers work through an abstraction?

### Text Notation for Whiteboard/Interview

When you cannot draw arrows easily, use text notation:

```
[ParkingLot] 1 ---|<#>|--- * [ParkingFloor]
[ParkingFloor] 1 ---|<#>|--- * [ParkingSpot]
[Vehicle] ---|>  [Car, Truck, Motorcycle]
[Payment] <<abstract>> ---|> [CashPayment, CardPayment]
[Ticket] ---> [ParkingSpot]   (association)
[Ticket] ---> [Vehicle]       (association)
```

### What Interviewers Look For

1. **Correct use of composition vs aggregation** — this shows you understand ownership and lifecycle
2. **Interfaces/abstract classes** — this shows you think in terms of abstractions
3. **No god classes** — classes should be focused, not 20-field monsters
4. **Sensible multiplicity** — 1-to-many, many-to-many where appropriate
5. **Enumerations** — finite sets of values should be enums, not strings

---

## 9. Common Mistakes

### Mistake 1: Arrows in wrong direction
```
// Wrong: inheritance arrow from parent to child
Animal <|---- Dog    (wrong direction)

// Correct: arrow from child to parent
Animal <|---- Dog
 ^(open triangle at parent end)
```

### Mistake 2: Confusing aggregation and composition
- If you create it inside the class → composition (filled diamond)
- If you receive it from outside → aggregation (open diamond)

### Mistake 3: Every relationship is association
Not all relationships are the same. Distinguish: does the lifecycle of B depend on A? (composition vs aggregation). Is A a type of B? (inheritance vs association).

### Mistake 4: Showing implementation details
A class diagram shows design intent, not code. Skip trivial getters/setters. Show meaningful attributes and important operations only.

### Mistake 5: Missing the <<interface>> stereotype
When you draw an interface, mark it with `<<interface>>` so it's clear. Without it, it looks like a class with no attributes.

---

## 10. Exercises

### Exercise 1: Hotel Booking System

**Problem:** Design the class diagram for a hotel booking system. A hotel has multiple rooms of different types (single, double, suite). Guests can make reservations for specific date ranges. Each reservation must be paid for. The hotel offers seasonal pricing.

**Guidance:**
- Identify at least 6 classes
- Show at least one abstract class
- Show composition, aggregation, and association
- Include an enumeration

**Solution:**

```
+-------------------+
| <<enumeration>>   |        +-------------------+
| RoomType          |        | <<enumeration>>   |
+-------------------+        | ReservationStatus |
| SINGLE            |        +-------------------+
| DOUBLE            |        | PENDING           |
| SUITE             |        | CONFIRMED         |
| PENTHOUSE         |        | CHECKED_IN        |
+-------------------+        | CHECKED_OUT       |
                             | CANCELLED         |
                             +-------------------+

+------------------+
|      Hotel       |
+------------------+
|- name: String    |
|- address: Address|
|- starRating: int |
+------------------+
|+getAvailableRooms(checkIn,checkOut): List<Room>|
|+makeReservation(guest, room, dates): Reservation|
+------------------+
      |1             |1
      |<#>           |<#>
      |*             |*
+----------+      +----------+
|   Room   |      |   Staff  |
+----------+      +----------+
|- number:Str|    |- id: long |
|- floor: int|    |- name: Str|
|- type: RoomType|+----------+
|- ratePerNight:double|
+----------+
|+isAvailable(checkIn,checkOut): bool|
+----------+
      |1
      |
      | occupied by
      |*
+-----------------+
|   Reservation   |
+-----------------+
|- id: String     |
|- checkIn: LD    |
|- checkOut: LD   |
|- status: ReservationStatus|
|- totalAmount: double|
+-----------------+
|+getDurationNights(): int|
|+cancel(): void  |
+-----------------+
      |*            |*
      | made by     | includes
      |1            |1
+-----------+  +-----------+
|   Guest   |  |  Payment  |
+-----------+  +-----------+
|- id: long |  | <<abstract>>|
|- name: Str|  |- amount: double|
|- email:Str|  +-----------+
+-----------+  |+process() |
               +-----------+
                    |>
               _____|_____
              |           |
         +--------+  +----------+
         |  Cash  |  |CreditCard|
         +--------+  +----------+
```

```java
public class Hotel {
    private final String name;
    private final Address address;
    private final List<Room> rooms;     // composition

    public List<Room> getAvailableRooms(LocalDate checkIn, LocalDate checkOut) {
        return rooms.stream()
            .filter(room -> room.isAvailable(checkIn, checkOut))
            .collect(Collectors.toList());
    }

    public Reservation makeReservation(Guest guest, Room room,
                                       LocalDate checkIn, LocalDate checkOut) {
        if (!room.isAvailable(checkIn, checkOut)) {
            throw new RoomNotAvailableException("Room " + room.getNumber() + " is not available");
        }
        Reservation reservation = new Reservation(
            UUID.randomUUID().toString(), guest, room, checkIn, checkOut);
        room.addReservation(reservation);
        return reservation;
    }
}

public class Reservation {
    private final String id;
    private final Guest guest;
    private final Room room;
    private final LocalDate checkIn;
    private final LocalDate checkOut;
    private ReservationStatus status;
    private Payment payment;

    public int getDurationNights() {
        return (int) ChronoUnit.DAYS.between(checkIn, checkOut);
    }

    public double calculateTotal(PricingPolicy pricing) {
        return getDurationNights() * pricing.getRatePerNight(room, checkIn);
    }

    public void cancel() {
        if (status == ReservationStatus.CHECKED_IN) {
            throw new IllegalStateException("Cannot cancel an active check-in");
        }
        this.status = ReservationStatus.CANCELLED;
    }
}
```

---

### Exercise 2: Online Banking System

**Problem:** Design the class diagram for an online banking system. Users have accounts (checking, savings, credit). They can transfer funds, view transactions, and pay bills. An account has a transaction history.

**Solution sketch:**

```
[User] 1 ---<>--- * [Account]         (aggregation: user can have multiple accounts)
[Account] <<abstract>> |> [CheckingAccount, SavingsAccount, CreditAccount]
[Account] 1 ---<#>--- * [Transaction]  (composition: transactions belong to account)
[Transaction] has type [TransactionType] <<enumeration>>
[Transfer] is a [Transaction] that links two accounts
[BillPayment] is a [Transaction] to an [Biller]
```

---

### Exercise 3: Restaurant Order System

**Problem:** A restaurant has multiple tables. Customers sit at a table and place orders. An order contains menu items. Each menu item belongs to a category. The kitchen receives orders and marks them as prepared.

**Solution sketch:**

```
[Restaurant] 1 ---<#>--- * [Table]
[Table] 1 --- 0..1 [Order]           (a table has an active order or none)
[Order] 1 ---<#>--- * [OrderItem]    (composition)
[OrderItem] * ---> 1 [MenuItem]      (association: items reference menu)
[MenuItem] * ---> 1 [MenuCategory]   (association: items belong to category)
[Order] has [OrderStatus] <<enumeration>>
[Kitchen] uses [Order]               (dependency)
```

These are the starting points — extend them with full class details as practice.
