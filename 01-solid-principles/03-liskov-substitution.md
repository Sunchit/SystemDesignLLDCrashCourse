# Liskov Substitution Principle (LSP)

> SOLID Principle 3 of 5 — Part of the "01-solid-principles" module

---

## Table of Contents

1. [The Formal Definition](#1-the-formal-definition)
2. [What It Means in Plain English](#2-what-it-means-in-plain-english)
3. [The "Is-A" Trap: Structural vs. Behavioral Subtyping](#3-the-is-a-trap-structural-vs-behavioral-subtyping)
4. [Design by Contract: Preconditions, Postconditions, Invariants](#4-design-by-contract-preconditions-postconditions-invariants)
5. [The Rectangle/Square Problem — Complete Java Example](#5-the-rectanglesquare-problem--complete-java-example)
6. [Other Classic LSP Violations](#6-other-classic-lsp-violations)
7. [How to Detect LSP Violations in Code Review](#7-how-to-detect-lsp-violations-in-code-review)
8. [Behavioral vs. Structural Subtyping — A Deeper Dive](#8-behavioral-vs-structural-subtyping--a-deeper-dive)
9. [Design by Contract in Java Practice](#9-design-by-contract-in-java-practice)
10. [LSP and Interfaces — Interfaces as Behavioral Contracts](#10-lsp-and-interfaces--interfaces-as-behavioral-contracts)
11. [Interview Questions and Model Answers](#11-interview-questions-and-model-answers)
12. [Assignment: BankAccount Hierarchy](#12-assignment-bankaccount-hierarchy)

---

## 1. The Formal Definition

Barbara Liskov introduced this principle in her 1987 keynote address "Data Abstraction and Hierarchy" and later formalized it with Jeannette Wing in their 1994 paper "A Behavioral Notion of Subtyping." The formal statement:

> **"If S is a subtype of T, then objects of type T may be replaced with objects of type S without altering any of the desirable properties of the program."**

In notation:

```
For all x: T, Q(x) => Q(S(x))
where Q is a provable property of objects of type T
```

This is not just an object-oriented design guideline. It is a mathematical theorem about type systems. Liskov was defining what it means for one type to be a legitimate subtype of another — not just syntactically (Java will let you compile it) but semantically (the program still works correctly when you swap types).

---

## 2. What It Means in Plain English

Imagine you have a function that accepts a `Payment` object. It calls `payment.process()`, checks that a receipt is returned, and logs the result. You are the caller. You do not know or care whether the actual runtime object is a `CreditCardPayment`, a `BankTransferPayment`, or a `WalletPayment`.

**LSP says:** Every subtype must be substitutable for its parent without the caller having to know which subtype it got, and without the program breaking any of the behaviors the caller reasonably expected from the parent type.

Put differently: if a subclass forces any of its callers to handle it differently from its parent — through exceptions it shouldn't throw, behaviors it silently changes, or state it refuses to maintain — that subclass violates LSP.

LSP is ultimately about **trust**. Callers trust that a subtype will honor the behavioral contract established by its parent. When a subtype breaks that trust, callers must start defensively checking what they actually got, which makes the code brittle and the type hierarchy untrustworthy.

A violation almost always reveals one of two problems:

1. The "is-a" relationship used to justify inheritance is geometrically or taxonomically correct but **behaviorally incorrect** in the context of the program.
2. The parent type was designed with weaker assumptions than the child type can honor, meaning the parent's contract is too broad for the child to fulfill honestly.

---

## 3. The "Is-A" Trap: Structural vs. Behavioral Subtyping

### The Geometry Trap

In Euclidean geometry, a square is a rectangle. A square has four right angles and four sides — it satisfies every geometric property that defines a rectangle, with the additional constraint that all sides are equal. "Square is-a Rectangle" is mathematically valid.

In a Java program, however, "is-a" means something different. It means: **can a Square object be used anywhere a Rectangle object is expected, without the program breaking?** The answer, as you will see shortly, is no.

This is the is-a trap. Developers learn early in their careers that inheritance models "is-a" relationships. They see a Square and a Rectangle, recognize the geometric truth, and reach for `class Square extends Rectangle`. This compiles. It even seems to run. But it introduces a subtle violation that will cause test failures and production bugs later.

### Structural Subtyping

Structural subtyping asks only: **does the subtype have all the methods of the parent?** Java's type system performs this check. If `Square` extends `Rectangle` and doesn't declare any new abstract methods, Java considers it a valid subtype. The `instanceof` check passes. You can pass a `Square` anywhere a `Rectangle` is expected.

This is what Java's compiler verifies. It is necessary but not sufficient.

### Behavioral Subtyping

Behavioral subtyping asks a harder question: **does the subtype honor every behavioral guarantee that clients of the parent type rely on?**

If `Rectangle` guarantees that calling `setWidth(5)` changes only the width while the height remains unchanged, then any `Rectangle` subtype must honor that guarantee. A `Square` that overrides `setWidth(5)` to also change the height does not honor that guarantee. It is structurally a subtype but behaviorally not.

The distinction matters because real software does not call methods through the compiler. It calls them through object references at runtime. And at runtime, the caller has a behavioral expectation — the contract established by the parent type — which the subtype may silently violate.

**Summary:**

| Question | Structural Subtyping | Behavioral Subtyping |
|---|---|---|
| What does it check? | Method signatures present? | Behavioral contracts honored? |
| Who enforces it? | Java compiler | Programmer discipline |
| Is it sufficient for LSP? | No | Yes |
| Rectangle/Square: passes? | Yes | No |

---

## 4. Design by Contract: Preconditions, Postconditions, Invariants

Bertrand Meyer's Design by Contract (DbC) gives us precise language for describing what a method promises and what it demands. LSP can be expressed precisely using these three constructs:

### 4.1 Preconditions

A precondition is a condition that must be true **before** a method is called — something the caller must guarantee.

**Example:** `withdraw(double amount)` might have the precondition `amount > 0`.

**LSP rule on preconditions:** A subtype **cannot strengthen** preconditions.

If the parent accepts `amount > 0`, the subtype cannot require `amount > 100`. A caller written for the parent type will call `withdraw(50)` believing it is a valid call. If the subtype rejects it, the substitution is broken.

Put differently: the subtype must accept at least everything the parent accepts. It may be more lenient (accept `amount >= 0`) but never more restrictive.

### 4.2 Postconditions

A postcondition is a condition that must be true **after** a method returns — something the method guarantees to the caller.

**Example:** `deposit(double amount)` might guarantee that `balance_after = balance_before + amount`.

**LSP rule on postconditions:** A subtype **cannot weaken** postconditions.

If the parent guarantees that balance increases by exactly `amount`, the subtype cannot guarantee only that balance increases by at most `amount`. Callers written for the parent rely on the full guarantee. Receiving less breaks them.

Put differently: the subtype must deliver at least everything the parent delivers. It may deliver more (stronger guarantees) but never less.

### 4.3 Invariants

An invariant is a condition that must always be true of an object throughout its lifetime — regardless of which method was called or when.

**Example:** A `BankAccount` might maintain the invariant `balance >= 0` (no overdraft allowed).

**LSP rule on invariants:** A subtype **must maintain all class invariants** of the parent.

If the parent guarantees the object is always in a valid, non-negative-balance state, the subtype cannot allow the balance to go negative under any circumstance — even in methods the parent never called.

### Summary Table

| Contract Element | Parent defines | Subtype rule | Violation |
|---|---|---|---|
| Precondition | What caller must provide | Cannot strengthen (demand more) | Subtype rejects inputs parent accepted |
| Postcondition | What method guarantees | Cannot weaken (deliver less) | Subtype returns less than parent promised |
| Invariant | Object state always true | Must maintain all parent invariants | Subtype allows invalid object state |

---

## 5. The Rectangle/Square Problem — Complete Java Example

### 5.1 The Naive (Broken) Design

```java
// Rectangle.java
public class Rectangle {

    protected double width;
    protected double height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    public void setWidth(double width) {
        this.width = width;
    }

    public void setHeight(double height) {
        this.height = height;
    }

    public double getWidth() {
        return width;
    }

    public double getHeight() {
        return height;
    }

    public double area() {
        return width * height;
    }

    @Override
    public String toString() {
        return String.format("Rectangle[width=%.1f, height=%.1f, area=%.1f]",
                width, height, area());
    }
}
```

```java
// Square.java — LSP VIOLATION
public class Square extends Rectangle {

    public Square(double side) {
        super(side, side);
    }

    // Overriding setWidth to keep sides equal — this is the violation.
    // The parent's postcondition guarantees: after setWidth(w), width == w and height is UNCHANGED.
    // This override silently changes height as well, breaking that postcondition.
    @Override
    public void setWidth(double width) {
        this.width = width;
        this.height = width;  // violates parent postcondition
    }

    // Same violation: parent guarantees height changes and width is UNCHANGED.
    @Override
    public void setHeight(double height) {
        this.width = height;  // violates parent postcondition
        this.height = height;
    }

    @Override
    public String toString() {
        return String.format("Square[side=%.1f, area=%.1f]", width, area());
    }
}
```

```java
// ShapeResizer.java — client code written for Rectangle
public class ShapeResizer {

    /**
     * This method is written against the Rectangle contract.
     * It assumes: after setWidth(w), width == w and height is unchanged.
     * That is a reasonable assumption — it is exactly what Rectangle guarantees.
     */
    public static void stretchWidth(Rectangle rect, double newWidth) {
        double originalHeight = rect.getHeight();
        rect.setWidth(newWidth);

        // Post-call assertion: height must be unchanged
        // This assertion holds for Rectangle but FAILS for Square.
        assert rect.getHeight() == originalHeight
                : "Height changed unexpectedly! Expected: " + originalHeight
                + " but got: " + rect.getHeight();

        System.out.println("After stretching width to " + newWidth + ": " + rect);
    }

    public static void main(String[] args) {
        System.out.println("--- Testing with Rectangle ---");
        Rectangle rect = new Rectangle(4.0, 6.0);
        System.out.println("Before: " + rect);
        stretchWidth(rect, 10.0);
        // Expected: Rectangle[width=10.0, height=6.0, area=60.0]
        // Actual:   Rectangle[width=10.0, height=6.0, area=60.0]  -- CORRECT

        System.out.println();

        System.out.println("--- Testing with Square (LSP Violation) ---");
        Rectangle square = new Square(5.0);  // polymorphic reference — legal in Java
        System.out.println("Before: " + square);
        stretchWidth(square, 10.0);
        // Expected by caller: Square[side=5.0] with width stretched to 10 but height still 5
        // Actual: Square[side=10.0, area=100.0] -- WRONG: height changed too!
        // The assertion above would fire if assertions are enabled (-ea JVM flag).
    }
}
```

**Run this with JVM assertions enabled:**

```bash
javac Rectangle.java Square.java ShapeResizer.java
java -ea ShapeResizer
```

Output:
```
--- Testing with Rectangle ---
Before: Rectangle[width=4.0, height=6.0, area=24.0]
After stretching width to 10.0: Rectangle[width=10.0, height=6.0, area=60.0]

--- Testing with Square (LSP Violation) ---
Before: Square[side=5.0, area=25.0]
Exception in thread "main" java.lang.AssertionError: Height changed unexpectedly!
Expected: 5.0 but got: 10.0
```

The assertion proves the violation: code that was correct for `Rectangle` breaks when given a `Square`.

### 5.2 A More Realistic Failure

Real-world violations do not always show up in unit tests. They appear in production, in edge cases, in code written by a different team months later:

```java
// RectanglePrinter.java — another client, written later by a different team
public class RectanglePrinter {

    /**
     * Prints a "border" version of the shape: a rectangle frame using '*' characters.
     * Scales the shape to fit a given width boundary, preserving height.
     */
    public static void printBorder(Rectangle shape, double targetWidth) {
        shape.setWidth(targetWidth);
        // Assumes height is now unchanged — prints accordingly
        System.out.printf("Border: width=%.0f, height=%.0f%n",
                shape.getWidth(), shape.getHeight());
    }

    public static void main(String[] args) {
        Rectangle r = new Rectangle(3, 7);
        printBorder(r, 10);
        // Output: Border: width=10, height=7  -- correct

        Rectangle s = new Square(5);
        printBorder(s, 10);
        // Output: Border: width=10, height=10 -- WRONG: height was not supposed to change
        // This code has a bug that only appears with Square, not Rectangle.
        // The developer who wrote printBorder had no idea Square existed.
    }
}
```

This is the real damage LSP violations cause: **bugs that only appear in production when a new subtype is introduced**, written by a team that had no knowledge of the subtype at the time they wrote client code. Their code was perfectly correct against the parent's contract.

### 5.3 The Fix: Correct Design

There are several correct approaches. The right choice depends on what the domain actually needs.

**Option A: Separate classes with a common interface (preferred)**

```java
// Shape.java — behavioral contract, no mutable state assumptions
public interface Shape {
    double area();
    double perimeter();
}
```

```java
// Rectangle.java — standalone, no inheritance relationship to Square
public class Rectangle implements Shape {

    private double width;
    private double height;

    public Rectangle(double width, double height) {
        if (width <= 0 || height <= 0) {
            throw new IllegalArgumentException("Dimensions must be positive");
        }
        this.width = width;
        this.height = height;
    }

    public void setWidth(double width) {
        if (width <= 0) throw new IllegalArgumentException("Width must be positive");
        this.width = width;
        // Postcondition: this.width == width, this.height is unchanged
    }

    public void setHeight(double height) {
        if (height <= 0) throw new IllegalArgumentException("Height must be positive");
        this.height = height;
        // Postcondition: this.height == height, this.width is unchanged
    }

    public double getWidth() { return width; }
    public double getHeight() { return height; }

    @Override
    public double area() { return width * height; }

    @Override
    public double perimeter() { return 2 * (width + height); }

    @Override
    public String toString() {
        return String.format("Rectangle[width=%.1f, height=%.1f]", width, height);
    }
}
```

```java
// Square.java — standalone, no inheritance from Rectangle
public class Square implements Shape {

    private double side;

    public Square(double side) {
        if (side <= 0) throw new IllegalArgumentException("Side must be positive");
        this.side = side;
    }

    public void setSide(double side) {
        if (side <= 0) throw new IllegalArgumentException("Side must be positive");
        this.side = side;
        // Postcondition: this.side == side (both dimensions change together — that is the contract of Square)
    }

    public double getSide() { return side; }

    @Override
    public double area() { return side * side; }

    @Override
    public double perimeter() { return 4 * side; }

    @Override
    public String toString() {
        return String.format("Square[side=%.1f]", side);
    }
}
```

```java
// ShapeResizer.java — fixed client code
public class ShapeResizer {

    /**
     * Now explicitly accepts a Rectangle. There is no ambiguity.
     * Square is not a Rectangle in this design, so it cannot be passed here.
     * The compiler enforces what LSP should have enforced by design.
     */
    public static void stretchWidth(Rectangle rect, double newWidth) {
        double originalHeight = rect.getHeight();
        rect.setWidth(newWidth);

        // This assertion now provably holds for all valid inputs
        assert rect.getHeight() == originalHeight : "Internal error: height changed";

        System.out.println("After stretching: " + rect);
    }

    /**
     * This method works for any Shape — it only uses area() and perimeter(),
     * which every Shape correctly implements.
     */
    public static void printShapeInfo(Shape shape) {
        System.out.printf("Area: %.2f, Perimeter: %.2f%n",
                shape.area(), shape.perimeter());
    }

    public static void main(String[] args) {
        Rectangle rect = new Rectangle(4.0, 6.0);
        stretchWidth(rect, 10.0);

        Square square = new Square(5.0);
        // stretchWidth(square, 10.0); -- compile error: Square is not a Rectangle
        // This is correct. The compiler now catches the conceptual mismatch.

        printShapeInfo(rect);   // works
        printShapeInfo(square); // works
    }
}
```

**Option B: Immutable value objects (sometimes appropriate)**

If shapes in your domain are immutable value objects (common in graphics or geometry libraries), there is no mutation contract to violate. Immutability sidesteps the mutation contract problem entirely:

```java
public final class Rectangle implements Shape {
    private final double width;
    private final double height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    // No setters — width and height can never change after construction
    public double getWidth() { return width; }
    public double getHeight() { return height; }

    // To "resize", return a new object — functional style
    public Rectangle withWidth(double newWidth) {
        return new Rectangle(newWidth, this.height);
    }

    @Override
    public double area() { return width * height; }

    @Override
    public double perimeter() { return 2 * (width + height); }
}

public final class Square implements Shape {
    private final double side;

    public Square(double side) {
        this.side = side;
    }

    public Square withSide(double newSide) {
        return new Square(newSide);
    }

    @Override
    public double area() { return side * side; }

    @Override
    public double perimeter() { return 4 * side; }
}
```

With immutable objects, a `Square` can safely extend `Rectangle` from a geometric standpoint only if both are immutable, because there is no mutation contract to violate. Even then, the `withWidth` method returning a `Rectangle` from a `Square` call is semantically awkward, so separate classes remain cleaner.

---

## 6. Other Classic LSP Violations

### 6.1 The Bird/Ostrich Problem — Unexpected Exceptions

```java
// Bird.java
public abstract class Bird {

    private String name;

    public Bird(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    /**
     * Contract: fly() causes the bird to become airborne.
     * Postcondition: the bird is in flight after this call.
     * No subtype should throw an exception here — if this method exists
     * on Bird, callers expect every Bird to be capable of flying.
     */
    public abstract void fly();

    public abstract void eat();
}
```

```java
// Eagle.java — honors the contract
public class Eagle extends Bird {

    public Eagle() {
        super("Eagle");
    }

    @Override
    public void fly() {
        System.out.println("Eagle soars at altitude using thermal updrafts");
        // Postcondition met: bird is airborne
    }

    @Override
    public void eat() {
        System.out.println("Eagle hunts fish and small mammals");
    }
}
```

```java
// Ostrich.java — LSP VIOLATION: throws where parent guarantees behavior
public class Ostrich extends Bird {

    public Ostrich() {
        super("Ostrich");
    }

    @Override
    public void fly() {
        // LSP VIOLATION: parent contract says fly() causes flight.
        // Ostrich throws instead. Any caller iterating a List<Bird>
        // and calling fly() will crash when it hits an Ostrich.
        throw new UnsupportedOperationException("Ostriches cannot fly");
    }

    @Override
    public void eat() {
        System.out.println("Ostrich grazes on plants and seeds");
    }
}
```

```java
// BirdTrainer.java — client that breaks with Ostrich
public class BirdTrainer {

    public static void trainFlock(List<Bird> flock) {
        for (Bird bird : flock) {
            bird.eat();
            bird.fly(); // Crashes at runtime when bird is an Ostrich
        }
    }

    public static void main(String[] args) {
        List<Bird> flock = new ArrayList<>();
        flock.add(new Eagle());
        flock.add(new Ostrich()); // compiles fine, breaks at runtime

        trainFlock(flock); // UnsupportedOperationException on Ostrich.fly()
    }
}
```

**The fix: restructure the hierarchy to reflect behavioral reality:**

```java
// Bird.java — only what ALL birds can do
public abstract class Bird {
    private String name;

    public Bird(String name) { this.name = name; }
    public String getName() { return name; }
    public abstract void eat();
    public abstract void move(); // all birds can move — on foot, wing, or swim
}
```

```java
// FlyingBird.java — behavioral contract for birds that fly
public abstract class FlyingBird extends Bird {

    public FlyingBird(String name) { super(name); }

    /**
     * Contract: every FlyingBird can fly. No exception will be thrown.
     * Postcondition: the bird is airborne after this call.
     */
    public abstract void fly();

    @Override
    public void move() {
        fly(); // FlyingBirds move by flying by default
    }
}
```

```java
// Eagle.java — correctly extends FlyingBird
public class Eagle extends FlyingBird {

    public Eagle() { super("Eagle"); }

    @Override
    public void fly() {
        System.out.println("Eagle soars on thermal updrafts");
    }

    @Override
    public void eat() {
        System.out.println("Eagle hunts fish");
    }
}
```

```java
// Ostrich.java — correctly extends Bird (not FlyingBird)
public class Ostrich extends Bird {

    public Ostrich() { super("Ostrich"); }

    @Override
    public void move() {
        System.out.println("Ostrich runs at up to 70 km/h");
    }

    @Override
    public void eat() {
        System.out.println("Ostrich grazes on plants");
    }
}
```

```java
// BirdTrainer.java — fixed: operates on Bird for general tasks, FlyingBird for flight training
public class BirdTrainer {

    public static void feedAll(List<Bird> birds) {
        for (Bird bird : birds) {
            bird.eat(); // safe for all birds
        }
    }

    public static void conductFlightTraining(List<FlyingBird> flyingBirds) {
        for (FlyingBird bird : flyingBirds) {
            bird.fly(); // safe: only FlyingBirds are here, and they all can fly
        }
    }

    public static void main(String[] args) {
        List<Bird> allBirds = new ArrayList<>();
        allBirds.add(new Eagle());
        allBirds.add(new Ostrich());
        feedAll(allBirds); // works for all

        List<FlyingBird> flyingBirds = new ArrayList<>();
        flyingBirds.add(new Eagle());
        // flyingBirds.add(new Ostrich()); -- compile error: Ostrich is not a FlyingBird
        conductFlightTraining(flyingBirds); // works correctly
    }
}
```

Now the type system reflects behavioral reality. `Ostrich` cannot be accidentally passed to flight-training code because it is not a `FlyingBird`. The compiler catches what was previously a runtime crash.

---

### 6.2 Read-Only Collection That Extends a Mutable Collection

This is a real violation pattern that appears in standard library designs. Consider a naive attempt:

```java
// MutableList.java — parent contract
public interface MutableList<T> {

    /**
     * Adds element to the list.
     * Postcondition: size increases by 1, element is retrievable at the last index.
     */
    void add(T element);

    /**
     * Removes element at index.
     * Postcondition: size decreases by 1, subsequent elements shift left.
     */
    void remove(int index);

    T get(int index);
    int size();
}
```

```java
// ReadOnlyList.java — LSP VIOLATION
// This "is-a" MutableList but refuses to honor the mutation contract.
public class ReadOnlyList<T> implements MutableList<T> {

    private final List<T> internal;

    public ReadOnlyList(List<T> source) {
        this.internal = Collections.unmodifiableList(new ArrayList<>(source));
    }

    @Override
    public void add(T element) {
        // LSP VIOLATION: parent postcondition says element is added.
        // This subtype throws instead — any caller adding to a "MutableList"
        // will crash at runtime if handed a ReadOnlyList.
        throw new UnsupportedOperationException("This list is read-only");
    }

    @Override
    public void remove(int index) {
        throw new UnsupportedOperationException("This list is read-only");
    }

    @Override
    public T get(int index) { return internal.get(index); }

    @Override
    public int size() { return internal.size(); }
}
```

```java
// Client code that breaks
public class DataProcessor {

    public static void appendResults(MutableList<String> list, List<String> results) {
        for (String result : results) {
            list.add(result); // Caller trusts the MutableList contract
        }
    }

    public static void main(String[] args) {
        MutableList<String> safe = new ArrayListWrapper<>();
        appendResults(safe, Arrays.asList("a", "b")); // works

        MutableList<String> readOnly = new ReadOnlyList<>(Arrays.asList("x", "y"));
        appendResults(readOnly, Arrays.asList("a", "b")); // UnsupportedOperationException at runtime
    }
}
```

**The fix: separate read and write contracts into distinct interfaces:**

```java
// ReadableList.java — read-only contract
public interface ReadableList<T> {
    T get(int index);
    int size();
    boolean isEmpty();
}
```

```java
// MutableList.java — extends readable with write operations
public interface MutableList<T> extends ReadableList<T> {
    void add(T element);
    void remove(int index);
}
```

```java
// ReadOnlyList.java — correctly implements only what it can honor
public class ReadOnlyList<T> implements ReadableList<T> {
    private final List<T> internal;

    public ReadOnlyList(List<T> source) {
        this.internal = Collections.unmodifiableList(new ArrayList<>(source));
    }

    @Override
    public T get(int index) { return internal.get(index); }

    @Override
    public int size() { return internal.size(); }

    @Override
    public boolean isEmpty() { return internal.isEmpty(); }
    // No add() or remove() — not promised, not present
}
```

```java
// DataProcessor.java — fixed: method signature now enforces the real requirement
public class DataProcessor {

    // Requires a MutableList — ReadOnlyList cannot be passed here at all
    public static void appendResults(MutableList<String> list, List<String> results) {
        for (String result : results) {
            list.add(result); // safe: MutableList guarantees this works
        }
    }

    // Accepts any readable list — works for both mutable and read-only
    public static void printAll(ReadableList<String> list) {
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
}
```

Now a `ReadOnlyList` cannot be accidentally passed to `appendResults`. The type system prevents the misuse at compile time.

---

### 6.3 Weakened Postcondition — Silent Data Corruption

```java
// DataRepository.java — parent contract
public abstract class DataRepository {

    /**
     * Saves entity and returns the persisted entity with its generated ID populated.
     * Postcondition: returned entity has a non-null, non-empty ID.
     * Postcondition: entity is retrievable by that ID immediately after this call.
     */
    public abstract Entity save(Entity entity);
}
```

```java
// InMemoryRepository.java — honors postconditions
public class InMemoryRepository extends DataRepository {

    private final Map<String, Entity> store = new HashMap<>();

    @Override
    public Entity save(Entity entity) {
        String id = UUID.randomUUID().toString();
        entity.setId(id);
        store.put(id, entity);
        return entity;
        // Postcondition met: entity has non-null ID, is retrievable by ID
    }
}
```

```java
// CachingRepository.java — LSP VIOLATION: weakened postcondition
public class CachingRepository extends DataRepository {

    private final DataRepository delegate;
    private final List<Entity> pendingWrites = new ArrayList<>();

    public CachingRepository(DataRepository delegate) {
        this.delegate = delegate;
    }

    @Override
    public Entity save(Entity entity) {
        // Buffers the write — entity is NOT immediately persisted.
        // Postcondition violated: entity is NOT retrievable by ID immediately after this call.
        // Callers expect immediate persistence. They will read stale or missing data.
        pendingWrites.add(entity);
        entity.setId("pending-" + System.currentTimeMillis()); // fake ID
        return entity;
    }

    public void flush() {
        for (Entity e : pendingWrites) {
            delegate.save(e);
        }
        pendingWrites.clear();
    }
}
```

```java
// Service that relies on the postcondition
public class OrderService {

    private final DataRepository repo;

    public OrderService(DataRepository repo) { this.repo = repo; }

    public void placeOrder(Order order) {
        Entity saved = repo.save(order.toEntity());
        // Relies on postcondition: saved.getId() is valid and the entity is persisted
        notifyFulfillment(saved.getId()); // sends ID to fulfillment service
        // With CachingRepository, ID is "pending-12345" — fulfillment will fail to find the order
    }
}
```

The fix is not to extend `DataRepository` with a caching subtype that changes the semantics of `save`. Instead, either:
- Use composition (wrap `DataRepository` without extending it)
- Make the async/eventual-consistency behavior part of the declared contract with a different interface (e.g., `AsyncDataRepository`)

---

## 7. How to Detect LSP Violations in Code Review

Use these specific signals when reviewing code for LSP compliance:

### 7.1 `instanceof` or Type Checking in Client Code

```java
// RED FLAG: client code that checks which subtype it got
public void processPayment(Payment payment) {
    if (payment instanceof CryptoCurrencyPayment) {
        // Special handling — why? Because CryptoCurrencyPayment doesn't honor Payment's contract?
        ((CryptoCurrencyPayment) payment).waitForConfirmation();
    }
    payment.process();
}
```

If client code needs to check the concrete type of an object to decide how to use it, the hierarchy is broken. Either the parent contract is too narrow (missing a needed method), or the subtype is not a true behavioral subtype.

### 7.2 `UnsupportedOperationException` in Overridden Methods

Any override that throws `UnsupportedOperationException` (or any runtime exception the parent does not declare) is almost always an LSP violation. The parent promised this operation. The subtype refusing it breaks the contract.

```java
// RED FLAG
@Override
public void setReadDeadline(Duration deadline) {
    throw new UnsupportedOperationException("Not supported by this stream type");
}
```

### 7.3 Empty Method Bodies That Silently Do Nothing

An override that does nothing silently violates postconditions:

```java
// RED FLAG: postcondition says "log is written." Silent no-op violates it.
@Override
public void writeAuditLog(AuditEvent event) {
    // intentionally empty — we decided this subtype doesn't need logging
}
```

This is arguably worse than throwing an exception because the caller has no idea the operation failed. The program continues without the side effect that the contract promised.

### 7.4 Overrides That Ignore or Bypass Invariants

```java
// Parent invariant: balance >= 0
public abstract class Account {
    protected double balance;

    public void withdraw(double amount) {
        if (amount > balance) throw new IllegalArgumentException("Insufficient funds");
        balance -= amount;
        // Invariant maintained: balance >= 0
    }
}

// RED FLAG: subtype bypass
public class DeveloperTestAccount extends Account {
    @Override
    public void withdraw(double amount) {
        balance -= amount; // no check — balance can go negative. Invariant broken.
    }
}
```

### 7.5 Return Values That Are Subtypes of What the Parent Declares

This is actually valid (covariant return types are LSP-compliant), but watch for the inverse — returning a wider type than declared or silently returning null when the parent guarantees a non-null return:

```java
// RED FLAG: parent guarantees non-null, subtype returns null
@Override
public Customer findById(String id) {
    // returns null when not found — parent contract says "returns a Customer"
    // Callers will NPE. Use Optional<Customer> instead if absence is possible.
    return cache.get(id);
}
```

### Summary: LSP Violation Detection Checklist

| Signal | What to look for |
|---|---|
| `instanceof` checks in client | Type-specific handling in polymorphic code |
| `UnsupportedOperationException` | Overrides that refuse inherited operations |
| Empty overrides | Silent no-ops where side effects were promised |
| Invariant bypass | Overrides that skip validation the parent enforced |
| Stronger preconditions | Overrides that reject inputs the parent accepted |
| Null returns | Returning null where parent guaranteed non-null |
| Narrower checked exceptions | Subtype declares checked exceptions the parent doesn't |

---

## 8. Behavioral vs. Structural Subtyping — A Deeper Dive

### What Java's Type System Actually Checks

Java enforces structural subtyping through its type system. When you write `class Square extends Rectangle`, Java verifies:

1. `Square` has all methods `Rectangle` declares (either inherited or overridden)
2. Method signatures are compatible (parameter types, return types, declared exceptions)
3. Access modifiers are not narrowed (can make public method more public, not less)

Java does **not** verify:

1. Whether overridden methods preserve the parent's behavioral intent
2. Whether postconditions are maintained
3. Whether invariants are upheld
4. Whether the override does what callers of the parent type expect

### The Specification/Implementation Gap

Behavioral subtyping requires that you reason about **specifications** (what a method promises), not just **implementations** (what code it contains). Java has no built-in specification language. This is why LSP violations are a design discipline problem, not a compiler problem.

Consider:

```java
// SpecificationExample.java
public abstract class SortedCollection<T extends Comparable<T>> {

    /**
     * Adds element to the collection.
     * Postcondition: element is present in the collection.
     * Postcondition: all elements in the collection are in sorted order.
     * Invariant: collection is always sorted (verifiable after any mutation).
     */
    public abstract void add(T element);

    public abstract T getMin();
    public abstract T getMax();
}
```

A subclass that overrides `add()` to insert at the end without sorting breaks the postcondition "all elements are in sorted order." Java compiles it. No runtime exception fires immediately. But `getMin()` and `getMax()` will return wrong answers. The behavioral contract is violated invisibly.

### When Structural Subtyping Is Sufficient

Structural subtyping is sufficient when the parent type has no meaningful behavioral contracts beyond method signatures — for example, simple data carriers (DTOs), marker interfaces, or types where any implementation is equally valid.

```java
// Serializable is a marker interface — no behavioral contract beyond "this type declares it can be serialized"
// Any class can implement it regardless of internal structure.
public class OrderDTO implements Serializable {
    private String orderId;
    private double amount;
    // No methods to override — structural subtyping is all that matters here
}
```

### When Behavioral Subtyping Is Required

Behavioral subtyping is required whenever:

1. The parent type establishes a contract that callers rely on (method pre/postconditions)
2. The parent type enforces invariants that callers assume hold throughout
3. The subtype is used polymorphically through the parent reference (the whole point of polymorphism)
4. Side effects are promised (writes to a log, updates to state, notifications sent)

In practice: any time you use inheritance for polymorphism (not just code reuse), behavioral subtyping is required.

---

## 9. Design by Contract in Java Practice

Java does not have a built-in DbC framework, but you can approximate it at increasing levels of rigor:

### Level 1: Javadoc Comments (Documentation Only)

```java
/**
 * Transfers amount from this account to the target account.
 *
 * @param target   the recipient account; must not be null
 * @param amount   the transfer amount; must be positive
 *
 * @pre  amount > 0
 * @pre  this.getBalance() >= amount
 * @pre  target != null
 *
 * @post this.getBalance() == old(this.getBalance()) - amount
 * @post target.getBalance() == old(target.getBalance()) + amount
 * @post The total balance across both accounts is conserved (no money created/destroyed)
 *
 * @throws IllegalArgumentException if amount <= 0
 * @throws InsufficientFundsException if this.getBalance() < amount
 */
public void transfer(BankAccount target, double amount) {
    // ...
}
```

This is the minimum. At least it communicates the contract to developers.

### Level 2: Guard Clauses and Assertions

```java
public void transfer(BankAccount target, double amount) {
    // Preconditions — validated at runtime in production
    if (target == null) throw new IllegalArgumentException("Target account must not be null");
    if (amount <= 0) throw new IllegalArgumentException("Transfer amount must be positive, got: " + amount);
    if (this.balance < amount) throw new InsufficientFundsException(
            "Insufficient funds: balance=" + this.balance + ", requested=" + amount);

    double thisBalanceBefore = this.balance;
    double targetBalanceBefore = target.balance;

    this.balance -= amount;
    target.balance += amount;

    // Postconditions — enabled with -ea in dev/test, disabled in production
    assert this.balance == thisBalanceBefore - amount
            : "Postcondition violated: this balance incorrect after transfer";
    assert target.balance == targetBalanceBefore + amount
            : "Postcondition violated: target balance incorrect after transfer";
    assert (this.balance + target.balance) == (thisBalanceBefore + targetBalanceBefore)
            : "Invariant violated: total balance not conserved";
}
```

Using Java's `assert` for postconditions is idiomatic. Enable with `-ea` in development and test environments. Guard clauses for preconditions should stay in production (they protect against caller bugs).

### Level 3: Test-Enforced Contracts

The most practical approach for teams: encode contracts as unit tests that any subtype must pass.

```java
// AccountContractTest.java — abstract test class that encodes the contract
// Any concrete Account subtype should extend this test to verify LSP compliance.
public abstract class AccountContractTest {

    // Subclasses provide the specific Account implementation under test
    protected abstract BankAccount createAccount(double initialBalance);

    @Test
    public void transfer_reducesSourceBalanceByAmount() {
        BankAccount source = createAccount(1000.0);
        BankAccount target = createAccount(200.0);
        source.transfer(target, 300.0);
        assertEquals(700.0, source.getBalance(), 0.001,
                "Source balance should decrease by transfer amount");
    }

    @Test
    public void transfer_increasesTargetBalanceByAmount() {
        BankAccount source = createAccount(1000.0);
        BankAccount target = createAccount(200.0);
        source.transfer(target, 300.0);
        assertEquals(500.0, target.getBalance(), 0.001,
                "Target balance should increase by transfer amount");
    }

    @Test
    public void transfer_conservesTotalBalance() {
        BankAccount source = createAccount(1000.0);
        BankAccount target = createAccount(200.0);
        double totalBefore = source.getBalance() + target.getBalance();
        source.transfer(target, 300.0);
        double totalAfter = source.getBalance() + target.getBalance();
        assertEquals(totalBefore, totalAfter, 0.001,
                "Total balance must be conserved across transfer");
    }

    @Test
    public void transfer_throwsOnNegativeAmount() {
        BankAccount source = createAccount(1000.0);
        BankAccount target = createAccount(200.0);
        assertThrows(IllegalArgumentException.class,
                () -> source.transfer(target, -50.0),
                "Transfer with negative amount must throw IllegalArgumentException");
    }

    @Test
    public void transfer_throwsOnInsufficientFunds() {
        BankAccount source = createAccount(100.0);
        BankAccount target = createAccount(200.0);
        assertThrows(InsufficientFundsException.class,
                () -> source.transfer(target, 500.0),
                "Transfer exceeding balance must throw InsufficientFundsException");
    }
}
```

```java
// SavingsAccountContractTest.java — verifies SavingsAccount is a valid LSP subtype
public class SavingsAccountContractTest extends AccountContractTest {

    @Override
    protected BankAccount createAccount(double initialBalance) {
        return new SavingsAccount(initialBalance, 0.05); // 5% interest rate
    }

    // Savings-specific tests go here, beyond the inherited contract tests
    @Test
    public void applyInterest_increasesBalanceByRate() {
        SavingsAccount account = new SavingsAccount(1000.0, 0.05);
        account.applyMonthlyInterest();
        assertEquals(1050.0, account.getBalance(), 0.001);
    }
}
```

If `SavingsAccount` violates any of the inherited contract tests, it violates LSP. This is the most practical and maintainable approach for enforcing LSP in a codebase.

---

## 10. LSP and Interfaces — Interfaces as Behavioral Contracts

In Java, interfaces are the primary tool for defining behavioral contracts. When designed well, they make LSP violations harder to introduce.

### Interfaces Define Behavioral Expectations, Not Just Method Lists

An interface is a promise. Every class that implements it promises to honor not just the method signatures, but the behavioral intent behind them.

```java
// PaymentProcessor.java — interface as behavioral contract
public interface PaymentProcessor {

    /**
     * Processes the payment.
     *
     * Precondition:  payment.getAmount() > 0
     * Precondition:  payment.getCurrency() is a supported ISO 4217 code
     *
     * Postcondition: if returned status is APPROVED, funds are reserved
     * Postcondition: if returned status is DECLINED, no funds are reserved
     * Postcondition: returned PaymentResult is never null
     * Postcondition: returned PaymentResult.transactionId is non-null on APPROVED status
     *
     * This method must not throw checked exceptions — use the status field to communicate failure.
     * Runtime exceptions are permitted only for programming errors (null arguments, etc.)
     */
    PaymentResult process(Payment payment);

    /**
     * Returns whether this processor supports the given currency.
     * Postcondition: return value is consistent — same currency always returns same result
     *               for the lifetime of this processor instance.
     */
    boolean supportsCurrency(String currencyCode);
}
```

```java
// CreditCardProcessor.java — LSP-compliant implementation
public class CreditCardProcessor implements PaymentProcessor {

    private final Set<String> supportedCurrencies = Set.of("USD", "EUR", "GBP");

    @Override
    public PaymentResult process(Payment payment) {
        // Preconditions enforced
        Objects.requireNonNull(payment, "Payment must not be null");
        if (payment.getAmount() <= 0)
            throw new IllegalArgumentException("Amount must be positive");

        // Business logic...
        boolean approved = chargeCard(payment);

        // Postcondition: return non-null PaymentResult with appropriate status
        if (approved) {
            return PaymentResult.approved(UUID.randomUUID().toString(), payment.getAmount());
        } else {
            return PaymentResult.declined("Insufficient credit");
        }
        // Never returns null — postcondition met
    }

    @Override
    public boolean supportsCurrency(String currencyCode) {
        return supportedCurrencies.contains(currencyCode);
    }

    private boolean chargeCard(Payment payment) {
        // ... network call to card network
        return true;
    }
}
```

```java
// LoggingPaymentProcessor.java — decorator, also LSP-compliant
// This is the Decorator pattern applied correctly: it extends behavior without violating contracts.
public class LoggingPaymentProcessor implements PaymentProcessor {

    private final PaymentProcessor delegate;
    private final Logger logger = LoggerFactory.getLogger(LoggingPaymentProcessor.class);

    public LoggingPaymentProcessor(PaymentProcessor delegate) {
        this.delegate = Objects.requireNonNull(delegate);
    }

    @Override
    public PaymentResult process(Payment payment) {
        logger.info("Processing payment: amount={}, currency={}",
                payment.getAmount(), payment.getCurrency());

        PaymentResult result = delegate.process(payment);

        logger.info("Payment result: status={}, transactionId={}",
                result.getStatus(), result.getTransactionId());

        return result; // postconditions inherited from delegate, still met
    }

    @Override
    public boolean supportsCurrency(String currencyCode) {
        return delegate.supportsCurrency(currencyCode); // delegates, postcondition preserved
    }
}
```

### Interface Segregation Enables Correct LSP Compliance

When interfaces are too broad, implementors are forced to implement methods they cannot honor honestly. This is why Interface Segregation Principle (ISP) and LSP are closely related:

```java
// Overly broad interface — forces LSP violations
public interface Worker {
    void work();
    void eat();      // robots don't eat
    void sleep();    // robots don't sleep
    void getPaycheck(); // interns may not get paid
}

// Robot is forced to implement eat() and sleep() — either throws or does nothing.
// Either is an LSP violation if callers expect these to have meaningful effects.
public class Robot implements Worker {
    @Override
    public void work() { System.out.println("Robot working"); }

    @Override
    public void eat() {
        throw new UnsupportedOperationException("Robots don't eat"); // LSP violation
    }

    @Override
    public void sleep() { /* no-op */ } // Silent LSP violation

    @Override
    public void getPaycheck() {
        throw new UnsupportedOperationException("Robots don't get paid"); // LSP violation
    }
}
```

```java
// Correctly segregated interfaces
public interface Workable {
    void work();
}

public interface Feedable {
    void eat();
}

public interface Restable {
    void sleep();
}

public interface Compensable {
    void getPaycheck();
}

// Robot only implements what it can honor
public class Robot implements Workable {
    @Override
    public void work() { System.out.println("Robot working"); }
    // No eat(), sleep(), or getPaycheck() — no LSP violation risk
}

// Human employee implements everything appropriate
public class Employee implements Workable, Feedable, Restable, Compensable {
    @Override
    public void work() { System.out.println("Employee working"); }

    @Override
    public void eat() { System.out.println("Employee eating lunch"); }

    @Override
    public void sleep() { System.out.println("Employee resting"); }

    @Override
    public void getPaycheck() { System.out.println("Employee receiving paycheck"); }
}
```

**Key insight:** The narrower your interfaces, the easier it is to build LSP-compliant hierarchies. A class that implements a narrow interface only promises what it can genuinely deliver.

---

## 11. Interview Questions and Model Answers

### Q1: What is the Liskov Substitution Principle in your own words?

**Model Answer:**

LSP says that if you have code written to work with a base type, that code should work correctly when given any subtype without modification. It is not just about the subclass having the same method names — it is about the subclass honoring the same behavioral contracts: accepting at least the inputs the parent accepts, delivering at least the guarantees the parent delivers, and maintaining the same invariants the parent maintains.

The practical implication: if client code ever has to check what concrete subtype it received in order to handle it correctly, the hierarchy is broken.

---

### Q2: Why is the Rectangle/Square relationship an LSP violation even though mathematically a square is a rectangle?

**Model Answer:**

In mathematics, "is-a" is about properties — and every property of a rectangle (four right angles, opposite sides equal) holds for a square. But in software, "is-a" through inheritance means behavioral substitutability — can I give you a Square everywhere a Rectangle is expected without breaking the program?

The Rectangle type establishes a behavioral contract through its setter methods: `setWidth(w)` changes only the width; `setHeight(h)` changes only the height. These are independent mutations. Code written for Rectangle relies on this: "I'll set the width to 10, and I expect the height to remain exactly what it was."

A Square cannot honor this contract. If you call `setWidth(10)` on a Square, the height must also change to 10 (because a square must have equal sides). The Square silently changes a value the caller expected to be unchanged. This breaks the postcondition of `setWidth`.

The geometric "is-a" and the behavioral "is-a" diverge here. LSP cares about the behavioral relationship, not the geometric one.

---

### Q3: What are preconditions, postconditions, and invariants, and what are LSP's rules for each?

**Model Answer:**

These come from Bertrand Meyer's Design by Contract:

- **Precondition**: what the caller must guarantee before calling a method. Example: `withdraw(amount)` requires `amount > 0`. LSP says a subtype **cannot strengthen preconditions** — it cannot demand more from callers than the parent did. If callers were told `amount > 0` is sufficient, a subtype that requires `amount > 100` will reject valid calls and break the substitution.

- **Postcondition**: what the method guarantees to the caller after it returns. Example: `save(entity)` returns the entity with a non-null ID. LSP says a subtype **cannot weaken postconditions** — it cannot deliver less than the parent promised. Callers rely on the full guarantee; receiving less breaks them.

- **Invariant**: a condition that must always hold for all objects of a type, throughout their lifetime. Example: `balance >= 0` for a BankAccount. LSP says a subtype **must maintain all parent invariants** — if the parent guarantees the object is always in a valid state, the subtype cannot allow the object to reach an invalid state through any of its methods.

---

### Q4: How do you detect LSP violations during code review?

**Model Answer:**

There are several specific signals:

First, `instanceof` or type-casting in client code. If a method that accepts a base type needs to check `instanceof SubtypeX` to handle it specially, the hierarchy is broken — either the subtype doesn't honor the parent's contract, or the parent's contract is missing something.

Second, `UnsupportedOperationException` or unexpected runtime exceptions in overridden methods. The parent promised the operation works. The subtype refusing it is a clear contract violation.

Third, empty method overrides that silently do nothing. The parent's postcondition says a side effect occurred. The no-op override means it did not. Callers will behave incorrectly assuming the side effect happened.

Fourth, overrides that skip validation the parent enforced. If the parent's `withdraw()` checks `balance >= amount` and the subtype's override skips that check, the invariant `balance >= 0` can be violated.

Fifth, return values that can be null when the parent declares they cannot be. Any caller that does not null-check (because the parent contract said it was unnecessary) will throw a NullPointerException.

---

### Q5: Distinguish between behavioral subtyping and structural subtyping with a concrete example.

**Model Answer:**

Structural subtyping asks: does the subtype have the same method signatures as the parent? Java's compiler enforces this. A `Square` that extends `Rectangle` and provides all required methods passes structural subtyping.

Behavioral subtyping asks: does the subtype honor all behavioral contracts of the parent, not just have the same method names? This is what LSP requires, and Java's compiler does not check it — it requires programmer discipline.

Concrete example: `Square extends Rectangle` passes structural subtyping — both have `setWidth`, `setHeight`, `getWidth`, `getHeight`, `area`. But it fails behavioral subtyping because `Rectangle.setWidth(w)` has the postcondition "width changes to w, height is unchanged," and `Square.setWidth(w)` violates that postcondition by changing height as well.

You can compile `Square extends Rectangle`. You cannot substitute a `Square` for a `Rectangle` in code that relies on independent dimension mutation. The structural check passes; the behavioral check fails.

---

### Q6: Can you give an example of an LSP violation that does not involve inheritance — that is, in an interface implementation?

**Model Answer:**

Yes, and this is common. An interface defines a behavioral contract. Any implementation that fails to honor that contract violates LSP, even though there is no class inheritance involved.

The most common real-world example is a "read-only" implementation of a mutable collection interface. If `Collection<T>` has an `add()` method with the postcondition "element is added," an implementation that throws `UnsupportedOperationException` from `add()` violates LSP. Callers that treat the object as a `Collection` and call `add()` will crash at runtime.

Another example: a `Logger` interface with a `log()` method where the behavioral contract says "the message is persisted." A `NullLogger` that does nothing seems harmless, but if callers rely on the log being written (for auditing, compliance, or debugging), the silent no-op breaks the behavioral contract. This is an LSP violation if the interface does not explicitly permit "no-op implementations."

The fix in both cases is to design the interface to reflect only what all implementations can honestly honor, and to separate concerns at the interface level.

---

### Q7: How does LSP relate to the Open/Closed Principle?

**Model Answer:**

They are deeply complementary. The Open/Closed Principle says classes should be open for extension (new subtypes) and closed for modification (existing code does not change when new subtypes are added). The mechanism that makes this possible is substitutability — the ability to introduce a new subtype without touching client code.

If your subtypes violate LSP, OCP breaks down. When you introduce a new subtype that does not honor the parent's contract, client code starts failing. The "fix" is to add `instanceof` checks or special cases in the client — which is exactly the modification that OCP says should never be necessary.

In other words: LSP is the precondition for OCP. You cannot have extensible, modification-free code if the new types you introduce are not true behavioral subtypes of the types they extend. LSP ensures that "open for extension" actually works in practice.

---

### Q8: When is it acceptable to override a method and have it throw an exception?

**Model Answer:**

There are limited cases where an override throwing an exception is LSP-compliant:

1. The parent method declares the exception. If the parent's `process()` declares `throws ProcessingException`, a subtype throwing `ProcessingException` is consistent with the contract. Callers already handle it.

2. The exception represents an input validation failure for a precondition the parent also enforces. If the parent throws `IllegalArgumentException` for null inputs, the subtype throwing `IllegalArgumentException` for null inputs is not a new violation — it is the same contract.

3. The parent interface or abstract class explicitly documents that no-op or exception-throwing implementations are permitted. Some interfaces are designed to be optionally implemented.

What is never LSP-compliant: throwing an exception that the parent method does not declare, for conditions where the parent would have returned normally. If the parent's contract says `findById(id)` returns a `Customer` (possibly via Optional if designed that way), a subtype that throws `DatabaseUnavailableException` as a runtime exception when the parent would have returned a result is a violation — callers have no reason to expect or handle this exception.

---

### Q9: How would you design a type hierarchy for a fleet of vehicles (cars, trucks, motorcycles, boats, helicopters) in an LSP-compliant way?

**Model Answer:**

Start by identifying what behavioral contracts are truly shared across all vehicles versus what is specific to subgroups.

A `Vehicle` base type can declare: `startEngine()`, `stopEngine()`, `refuel()`, `getMaxSpeed()`. Every vehicle in the fleet has an engine, can be fueled, and has a speed. These are contracts all implementations can honor.

Now, flying versus non-flying is a behavioral split. Create a `FlyingVehicle` interface with `takeOff()`, `land()`, `getMaxAltitude()`. Only helicopters implement this. No LSP violations arise because nothing in the `Vehicle` hierarchy mentions flight.

Road versus water is another split. Create `RoadVehicle` with `steer()` (returns a `SteeringAngle`) and `WaterVehicle` with `navigate(Heading)`. Motorcycles, cars, and trucks implement `RoadVehicle`. Boats implement `WaterVehicle`. A hovercraft could implement both.

Never put a `fly()` method in `Vehicle` to then have car implementations throw `UnsupportedOperationException`. That is the exact pattern LSP forbids. The type hierarchy should reflect capability boundaries, not the broadest possible supertype with optional methods.

The result is a set of narrow, honest interfaces. Client code that manages the entire fleet uses `Vehicle`. Code that plans flight routes uses `FlyingVehicle`. Code that handles road logistics uses `RoadVehicle`. No `instanceof` checks needed anywhere.

---

### Q10: In your experience, what is the most common source of LSP violations in production codebases?

**Model Answer:**

In production codebases, the most common source is retrofitting — taking an existing class hierarchy designed for one set of requirements and extending it to cover a new requirement that does not fully fit the original behavioral contract.

The pattern looks like this: there is a well-designed `Notification` class with `send(User user)`. It works for email and SMS. Then a new requirement arrives for "in-app notifications," which are stored in a database rather than sent over a network, and which do not have the concept of a delivery receipt. A developer creates `InAppNotification extends Notification`, overrides `send()` to write to the database, and returns a fake receipt. The postcondition "recipient receives the notification immediately" may not hold if the user is offline. The invariant "receipt is non-null and meaningful" may be violated.

The correct response is to evaluate whether `InAppNotification` is genuinely a behavioral subtype of `Notification` or whether it only shares some attributes. Often the right move is to extract an interface (`Notifiable`) that covers only what is common, and have separate implementations rather than an inheritance chain.

The discipline required is to question every `extends` relationship and ask: "Can I substitute this subtype for its parent in every context where the parent is used, and will all the contracts hold?" If the answer is no for any realistic context, composition or interface extraction is better than inheritance.

---

## 12. Assignment: BankAccount Hierarchy

### Context

You are reviewing a pull request for a banking system. The developer has built the following class hierarchy. Your task is to identify all LSP violations, explain why each is a violation, and redesign the hierarchy to be LSP-compliant.

### The Problematic Code

```java
// BankAccount.java — the parent class
public class BankAccount {

    protected String accountId;
    protected String ownerId;
    protected double balance;
    protected boolean active;

    public BankAccount(String accountId, String ownerId, double initialBalance) {
        this.accountId = accountId;
        this.ownerId = ownerId;
        this.balance = initialBalance;
        this.active = true;
    }

    /**
     * Deposits amount into the account.
     * Precondition:  amount > 0
     * Postcondition: balance increases by amount
     * Postcondition: returns updated balance
     * Invariant:     balance >= 0 at all times
     */
    public double deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Deposit amount must be positive");
        balance += amount;
        return balance;
    }

    /**
     * Withdraws amount from the account.
     * Precondition:  amount > 0
     * Precondition:  amount <= balance (sufficient funds)
     * Postcondition: balance decreases by amount
     * Postcondition: returns updated balance
     * Invariant:     balance >= 0 at all times
     */
    public double withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Withdrawal amount must be positive");
        if (amount > balance) throw new InsufficientFundsException("Insufficient funds");
        balance -= amount;
        return balance;
    }

    /**
     * Transfers amount to target account.
     * Precondition:  amount > 0
     * Precondition:  target != null
     * Postcondition: this.balance decreases by amount
     * Postcondition: target.balance increases by amount
     * Postcondition: total balance (this + target) is unchanged
     */
    public void transfer(BankAccount target, double amount) {
        this.withdraw(amount);
        target.deposit(amount);
    }

    public double getBalance() { return balance; }
    public String getAccountId() { return accountId; }
    public boolean isActive() { return active; }

    public void close() {
        this.active = false;
    }
}
```

```java
// SavingsAccount.java — first subclass
public class SavingsAccount extends BankAccount {

    private double interestRate;
    private int withdrawalCount;
    private static final int MAX_MONTHLY_WITHDRAWALS = 6; // regulatory limit

    public SavingsAccount(String accountId, String ownerId,
                          double initialBalance, double interestRate) {
        super(accountId, ownerId, initialBalance);
        this.interestRate = interestRate;
        this.withdrawalCount = 0;
    }

    @Override
    public double withdraw(double amount) {
        // CANDIDATE VIOLATION A: strengthened precondition
        if (withdrawalCount >= MAX_MONTHLY_WITHDRAWALS) {
            throw new WithdrawalLimitExceededException(
                    "Maximum monthly withdrawals (" + MAX_MONTHLY_WITHDRAWALS + ") exceeded");
        }
        double result = super.withdraw(amount);
        withdrawalCount++;
        return result;
    }

    public void applyMonthlyInterest() {
        double interest = balance * interestRate;
        balance += interest;
    }

    public void resetMonthlyWithdrawalCount() {
        withdrawalCount = 0;
    }
}
```

```java
// FixedDepositAccount.java — second subclass
public class FixedDepositAccount extends BankAccount {

    private final LocalDate maturityDate;
    private boolean matured;

    public FixedDepositAccount(String accountId, String ownerId,
                               double initialBalance, LocalDate maturityDate) {
        super(accountId, ownerId, initialBalance);
        this.maturityDate = maturityDate;
        this.matured = false;
    }

    @Override
    public double withdraw(double amount) {
        // CANDIDATE VIOLATION B: exception not declared by parent
        if (!matured) {
            throw new PrematureWithdrawalException(
                    "Cannot withdraw before maturity date: " + maturityDate);
        }
        return super.withdraw(amount);
    }

    @Override
    public double deposit(double amount) {
        // CANDIDATE VIOLATION C: deposit rejected after account is locked
        if (matured) {
            throw new UnsupportedOperationException("Cannot deposit into a matured fixed deposit");
        }
        return super.deposit(amount);
    }

    public void checkAndSetMaturity() {
        if (LocalDate.now().isAfter(maturityDate) || LocalDate.now().isEqual(maturityDate)) {
            this.matured = true;
        }
    }

    public boolean isMatured() { return matured; }
}
```

```java
// AccountManager.java — client code that uses the hierarchy
public class AccountManager {

    private final List<BankAccount> accounts = new ArrayList<>();

    public void addAccount(BankAccount account) {
        accounts.add(account);
    }

    /**
     * Attempts to withdraw from all accounts (e.g., for a monthly fee deduction).
     */
    public void deductMonthlyFee(double fee) {
        for (BankAccount account : accounts) {
            if (account.getBalance() >= fee) {
                account.withdraw(fee); // This may throw unexpectedly depending on subtype
            }
        }
    }

    /**
     * Moves funds from one account to another (e.g., automatic savings transfer).
     */
    public void autoTransfer(BankAccount source, BankAccount destination, double amount) {
        source.transfer(destination, amount); // May throw unexpectedly for FixedDepositAccount
    }
}
```

---

### Part 1: Identify All LSP Violations

Analyze the code above and answer the following questions for each violation:

1. Which class, which method, which specific line is the violation?
2. Which LSP rule does it break (precondition strengthening, postcondition weakening, invariant violation, unexpected exception)?
3. Write the test case that demonstrates the violation — a test that passes for `BankAccount` but fails for the subtype.
4. What is the real-world consequence if this code reaches production?

Use the table format below:

| # | Class | Method | Violation Type | Description |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |

---

### Part 2: Redesign the Hierarchy

Redesign the `BankAccount` hierarchy so that all LSP violations are resolved. Your redesign must:

1. Keep the core concept of a `BankAccount` with deposit, withdraw, and transfer operations
2. Correctly represent `SavingsAccount` behavior (interest, withdrawal limits) without LSP violations
3. Correctly represent `FixedDepositAccount` behavior (lock-in period, maturity) without LSP violations
4. Ensure `AccountManager.deductMonthlyFee()` can work correctly for any account type it accepts
5. Write the abstract contract test class (`AccountContractTest`) and show that both `SavingsAccount` and `FixedDepositAccount` pass it

Hints to consider:
- Does `FixedDepositAccount` belong in the same hierarchy as `BankAccount`? What operations does it genuinely share?
- For `SavingsAccount`, the monthly withdrawal limit is a real-world regulatory constraint. Is this truly a precondition strengthening, or is it a feature that client code should be aware of? How does the answer change your design?
- Should `AccountManager.deductMonthlyFee()` accept `BankAccount`, or should it accept a narrower interface?

---

### Part 3: Extension Question

A new requirement arrives: a `JointAccount` where two owners must both approve withdrawals above a threshold amount. Withdrawals below the threshold work like a normal `BankAccount` withdrawal.

1. Does `JointAccount extends BankAccount` violate LSP? Explain your reasoning carefully.
2. How would you design `JointAccount` so that it is LSP-compliant?
3. What changes, if any, are required in `AccountManager` to support `JointAccount`?

---

### Expected Deliverables

- Java source files for all redesigned classes and interfaces
- `AccountContractTest.java` (abstract test class encoding the contract)
- `SavingsAccountContractTest.java` and `FixedDepositAccountContractTest.java` (concrete test subclasses)
- A brief design note (in comments at the top of your main file) explaining each design decision and which LSP rule it satisfies

---

### Evaluation Rubric

| Criterion | Points |
|---|---|
| Correctly identifies all three violations with precise explanation | 30 |
| Writes test cases that demonstrate each violation | 20 |
| Redesigned hierarchy is LSP-compliant (verified by contract tests passing) | 25 |
| `AccountManager.deductMonthlyFee()` works correctly for all account types | 10 |
| Extension question: correct LSP analysis and compliant JointAccount design | 15 |

---

## Summary

The Liskov Substitution Principle is one of the most frequently misunderstood principles in object-oriented design. Its surface statement — subtypes should be substitutable for their supertypes — sounds obvious. The depth is in understanding what "substitutable" means: not just that the method signatures match, but that the full behavioral contract is honored.

The key rules to internalize:

1. **Preconditions cannot be strengthened** in subtypes. Do not demand more from callers than the parent did.
2. **Postconditions cannot be weakened** in subtypes. Do not deliver less than the parent promised.
3. **Invariants must be maintained**. If the parent guaranteed the object is always in a valid state, every subtype must maintain that guarantee.
4. **No unexpected exceptions**. If the parent's method does not declare an exception, the subtype must not throw one the caller has no reason to expect.
5. **No silent no-ops**. If the parent's method promises a side effect, a silent no-op override is a postcondition violation.

The practical test: can you write code against the parent type, then swap in a subtype, and have the code work correctly without modification? If the answer is no, the hierarchy violates LSP.

When LSP is violated, the consequences compound over time: `instanceof` checks proliferate, defensive null checks appear everywhere, tests become fragile, and the type hierarchy becomes untrustworthy. When LSP holds, polymorphism works correctly, the Open/Closed Principle becomes achievable, and new subtypes can be introduced without touching existing code.

The design tools for achieving LSP:
- Prefer interfaces over abstract classes for defining contracts (easier to keep narrow)
- Use interface segregation to keep contracts narrow enough that all implementations can honor them honestly
- Encode behavioral contracts as abstract test classes that all subtypes must pass
- Question every `extends` relationship: ask whether the subtype can honor every behavioral contract of the parent, not just whether it has the same method signatures
- Prefer composition over inheritance when the behavioral contract cannot be fully honored

---

*Next: [04-interface-segregation.md — The Interface Segregation Principle](./04-interface-segregation.md)*

*Previous: [02-open-closed.md — The Open/Closed Principle](./02-open-closed.md)*
