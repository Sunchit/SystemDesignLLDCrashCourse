# Composition vs Inheritance

> "Favor object composition over class inheritance."
> — Gang of Four, *Design Patterns: Elements of Reusable Object-Oriented Software* (1994)

This is one of the most cited and least understood pieces of OOP wisdom. This chapter explains exactly why the GoF said this, when they are wrong to follow it blindly, and how to make the call in practice.

---

## Table of Contents

1. [What Is Inheritance?](#1-what-is-inheritance)
2. [What Is Composition?](#2-what-is-composition)
3. [Why GoF Said "Favor Composition"](#3-why-gof-said-favor-composition)
4. [The Fragile Base Class Problem](#4-the-fragile-base-class-problem)
5. [The Deadly Diamond of Death](#5-the-deadly-diamond-of-death)
6. [When Inheritance IS Appropriate](#6-when-inheritance-is-appropriate)
7. [Composition Benefits in Depth](#7-composition-benefits-in-depth)
8. [The Mixin Pattern as Middle Ground](#8-the-mixin-pattern-as-middle-ground)
9. [Case Study: Duck Simulation](#9-case-study-duck-simulation)
10. [Case Study: Shape Drawing System](#10-case-study-shape-drawing-system)
11. [Decision Framework](#11-decision-framework)
12. [Interview Questions with Model Answers](#12-interview-questions-with-model-answers)

---

## 1. What Is Inheritance?

Inheritance is a mechanism where a child class (subclass) derives behavior and state from a parent class (superclass). In Java:

```java
public class Animal {
    protected String name;

    public void eat() {
        System.out.println(name + " is eating");
    }

    public void breathe() {
        System.out.println(name + " is breathing");
    }
}

public class Dog extends Animal {
    public void bark() {
        System.out.println(name + " says Woof!");
    }
}

// Dog inherits eat() and breathe() from Animal
Dog dog = new Dog();
dog.eat();    // inherited
dog.bark();   // own behavior
```

Inheritance creates an **is-a** relationship: a Dog **is an** Animal.

---

## 2. What Is Composition?

Composition is a mechanism where a class achieves behavior by holding references to other objects and delegating work to them. The class **has** the behavior rather than **being** a type that carries it.

```java
public class Engine {
    public void start() { System.out.println("Engine started"); }
    public void stop()  { System.out.println("Engine stopped"); }
}

public class Car {
    private final Engine engine; // Car HAS an Engine

    public Car(Engine engine) {
        this.engine = engine;
    }

    public void startCar() {
        engine.start(); // delegate to composed object
    }

    public void stopCar() {
        engine.stop();
    }
}
```

Composition creates a **has-a** relationship: a Car **has an** Engine.

---

## 3. Why GoF Said "Favor Composition"

The GoF identified two problems with heavy inheritance:

### Problem 1: Inheritance Breaks Encapsulation

When you subclass, you gain access to the internals of the parent. The subclass is tightly coupled to the parent's implementation — not just its interface. This means:

- Changes to the parent can silently break subclasses
- The parent cannot freely refactor without checking all subclasses
- The parent's implementation decisions become part of the subclass's contract

With composition, the composed object exposes only its public interface. The class holding it knows nothing about the implementation. When the composed object changes internally, the holder is unaffected.

### Problem 2: Inheritance Hierarchies Are Static

Class relationships are fixed at compile time. A `Dog` is always an `Animal`. You cannot change this at runtime. With composition, you can swap components at runtime:

```java
// Inheritance: behavior fixed at compile time
class Dog extends Animal { /* always barks */ }

// Composition: behavior configurable at runtime
class Robot {
    private SoundBehavior sound;

    public void setSound(SoundBehavior sound) {
        this.sound = sound; // swap behavior at runtime
    }

    public void makeSound() {
        sound.makeSound();
    }
}
```

### Problem 3: Deep Hierarchies Are Hard to Reason About

When you call a method, you must trace through multiple inheritance levels to understand what actually executes. The behavior of a class is spread across the hierarchy.

---

## 4. The Fragile Base Class Problem

The fragile base class problem occurs when a correct change to the base class breaks a subclass. This happens because subclasses depend on the *implementation details* of the base class, not just its interface.

```java
public class InstrumentedList<E> extends ArrayList<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

    public int getAddCount() { return addCount; }
}

// Test:
InstrumentedList<String> list = new InstrumentedList<>();
list.addAll(List.of("a", "b", "c"));
System.out.println(list.getAddCount()); // Expected: 3. Actual: 6!
```

Why? Because `ArrayList.addAll()` internally calls `add()` for each element. Our overridden `add()` increments the counter once, and our overridden `addAll()` increments it again — double-counting.

We wrote correct code in our subclass, but it broke because we depended on the internal implementation of `ArrayList`. When the base class changes its internal `addAll()` behavior, our subclass breaks — even though our subclass code is unchanged.

**The Fix: Composition**
```java
public class InstrumentedList<E> {
    private int addCount = 0;
    private final List<E> delegate; // compose instead of inherit

    public InstrumentedList(List<E> delegate) {
        this.delegate = delegate;
    }

    public boolean add(E e) {
        addCount++;
        return delegate.add(e);
    }

    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return delegate.addAll(c);
    }

    public int getAddCount() { return addCount; }
}

// Now addAll does not call our add() — it calls delegate.addAll()
// Count is always correct regardless of how ArrayList implements addAll internally
InstrumentedList<String> list = new InstrumentedList<>(new ArrayList<>());
list.addAll(List.of("a", "b", "c"));
System.out.println(list.getAddCount()); // Always 3
```

This example is from Joshua Bloch's *Effective Java* (Item 18) and is considered the canonical demonstration of the fragile base class problem.

---

## 5. The Deadly Diamond of Death

The diamond problem occurs in languages that support multiple inheritance. Java does not allow multiple class inheritance, which avoids this problem for classes. But it is worth understanding because Java does allow multiple interface inheritance, and interface default methods can reproduce it.

```
        Flyable
       /        \
    Bird       Airplane
       \        /
        FlyingBird?  <-- which fly() is inherited?
```

If `Bird` and `Airplane` both provide a different implementation of `fly()`, a class inheriting from both faces an ambiguity: which implementation does it use?

**Java's Solution: No Multiple Class Inheritance**

Java resolves this by simply disallowing it. You cannot extend two classes.

**Java's Default Method Diamond**

Java 8 introduced default methods in interfaces, which can create a limited diamond problem:

```java
public interface Flyable {
    default String fly() { return "Flying generically"; }
}

public interface Bird extends Flyable {
    default String fly() { return "Bird flying with wings"; }
}

public interface Airplane extends Flyable {
    default String fly() { return "Airplane flying with engines"; }
}

// This compiles only if you override fly() explicitly
public class FlyingMachine implements Bird, Airplane {
    @Override
    public String fly() {
        // Must resolve ambiguity explicitly
        return Bird.super.fly(); // or Airplane.super.fly()
    }
}
```

Java forces you to resolve the ambiguity explicitly. The compiler will refuse to compile `FlyingMachine` unless it overrides `fly()`.

---

## 6. When Inheritance IS Appropriate

Despite all the problems above, inheritance is sometimes the right choice. The GoF said **favor** composition — not "always use composition."

Inheritance is appropriate when:

### The relationship is a genuine is-a relationship

The subclass truly is a more specific kind of the superclass — not just has similar behavior.

```java
// Genuine is-a: a SavingsAccount IS a BankAccount
public abstract class BankAccount {
    protected double balance;

    public abstract void applyMonthlyFee();

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance += amount;
    }

    public void withdraw(double amount) {
        if (amount > balance) throw new IllegalStateException("Insufficient funds");
        balance -= amount;
    }
}

public class SavingsAccount extends BankAccount {
    private static final double INTEREST_RATE = 0.04;
    private static final double MONTHLY_FEE = 5.0;

    @Override
    public void applyMonthlyFee() {
        balance -= MONTHLY_FEE;
        balance += balance * INTEREST_RATE / 12; // monthly interest
    }
}
```

A `SavingsAccount` has all the behaviors of a `BankAccount` plus account-specific behavior. The Liskov Substitution Principle holds: you can use a `SavingsAccount` everywhere a `BankAccount` is expected.

### When you are extending a framework-designed hierarchy

JUnit extends `TestCase`. Spring MVC controllers extend `AbstractController`. These are designed for extension. The base class knows it will be subclassed.

### When the subclass needs access to the superclass's protected state

If the child genuinely needs internal state of the parent that cannot be exposed via a public interface without breaking encapsulation for all other callers, inheritance may be justified.

### The litmus test: the Liskov Substitution Principle

If LSP holds — if you can substitute the subclass anywhere the superclass is used without affecting correctness — inheritance is likely valid. If LSP breaks, inheritance is wrong.

---

## 7. Composition Benefits in Depth

### Flexibility: Multiple Behaviors

A class can compose multiple behaviors:

```java
// A class can have many behaviors via composition
public class FileProcessor {
    private final FileReader reader;
    private final FileValidator validator;
    private final FileTransformer transformer;
    private final FileWriter writer;

    // Can mix and match implementations at construction time
    public FileProcessor(FileReader reader, FileValidator validator,
                         FileTransformer transformer, FileWriter writer) {
        this.reader = reader;
        this.validator = validator;
        this.transformer = transformer;
        this.writer = writer;
    }

    public void process(String path) {
        FileData data = reader.read(path);
        validator.validate(data);
        FileData transformed = transformer.transform(data);
        writer.write(transformed);
    }
}
```

With inheritance, getting multiple behaviors requires multiple inheritance or deep hierarchies. With composition, you simply hold multiple references.

### Testability: Easy Mocking

Composed objects can be replaced with mocks or stubs in tests:

```java
// With composition — test is clean and isolated
@Test
void testProcess() {
    FileReader mockReader = mock(FileReader.class);
    FileValidator mockValidator = mock(FileValidator.class);
    FileTransformer mockTransformer = mock(FileTransformer.class);
    FileWriter mockWriter = mock(FileWriter.class);

    when(mockReader.read(anyString())).thenReturn(new FileData("content"));
    when(mockTransformer.transform(any())).thenReturn(new FileData("transformed"));

    FileProcessor processor = new FileProcessor(mockReader, mockValidator,
                                                mockTransformer, mockWriter);
    processor.process("test.txt");

    verify(mockWriter).write(any());
}
```

With inheritance, the subclass carries all the behavior of the parent — testing just the subclass behavior requires either a partial mock or a fully initialized parent.

### Runtime Behavior Change (Strategy Pattern)

```java
public class SortingService {
    private SortingStrategy strategy;

    public SortingService(SortingStrategy strategy) {
        this.strategy = strategy;
    }

    // Swap strategy at runtime
    public void setStrategy(SortingStrategy strategy) {
        this.strategy = strategy;
    }

    public void sort(List<Integer> data) {
        strategy.sort(data);
    }
}

public interface SortingStrategy {
    void sort(List<Integer> data);
}

public class QuickSort implements SortingStrategy {
    @Override
    public void sort(List<Integer> data) { /* quicksort */ }
}

public class MergeSort implements SortingStrategy {
    @Override
    public void sort(List<Integer> data) { /* mergesort */ }
}

// Switch strategy at runtime based on data size
SortingService service = new SortingService(new QuickSort());
if (data.size() > 1_000_000) {
    service.setStrategy(new MergeSort());
}
service.sort(data);
```

---

## 8. The Mixin Pattern as Middle Ground

Java does not have true mixins, but interface default methods allow a mixin-like pattern: adding reusable behavior to a class without inheritance.

```java
// Mixin interface: adds logging capability to any class
public interface Loggable {
    default void log(String message) {
        System.out.printf("[%s] %s: %s%n",
            LocalDateTime.now(),
            getClass().getSimpleName(),
            message);
    }
}

// Mixin interface: adds timing capability
public interface Timed {
    default long measureTime(Runnable operation) {
        long start = System.currentTimeMillis();
        operation.run();
        return System.currentTimeMillis() - start;
    }
}

// A class mixes in both behaviors without using up its single inheritance slot
public class OrderService implements Loggable, Timed {

    public void processOrder(Order order) {
        log("Processing order " + order.getId());
        long elapsed = measureTime(() -> {
            // actual order processing
        });
        log("Order processed in " + elapsed + "ms");
    }
}
```

Limitations of this approach:
- Default methods cannot access instance state of the implementing class
- They add to the public interface, potentially polluting the class API
- Best used for cross-cutting concerns that are genuinely stateless

---

## 9. Case Study: Duck Simulation

This is the canonical example from *Head First Design Patterns*.

### The Broken Inheritance Approach

```java
// Starting point
public abstract class Duck {
    public void quack() { System.out.println("Quack!"); }
    public void swim()  { System.out.println("Swimming..."); }
    public abstract void display();
}

// Problem: rubber ducks don't quack. Penguins don't fly.
// Adding fly() to Duck breaks RubberDuck
public abstract class Duck {
    public void quack() { System.out.println("Quack!"); }
    public void swim()  { System.out.println("Swimming..."); }
    public void fly()   { System.out.println("Flying!"); } // MISTAKE
    public abstract void display();
}

public class RubberDuck extends Duck {
    @Override
    public void quack() { System.out.println("Squeak!"); } // ok
    @Override
    public void fly()   { /* do nothing */ }               // LSP violation
    @Override
    public void display() { System.out.println("Rubber Duck"); }
}

public class Penguin extends Duck {
    @Override
    public void fly() { throw new UnsupportedOperationException(); } // LSP violation
    @Override
    public void display() { System.out.println("Penguin"); }
}
```

The hierarchy is broken because not all ducks fly, and not all ducks quack the same way. Inheriting fly() forces every subclass to deal with it, even when the behavior is wrong.

### The Composition Solution

```java
// Behaviors as interfaces
public interface QuackBehavior {
    void quack();
}

public interface FlyBehavior {
    void fly();
}

// Implementations
public class Quack implements QuackBehavior {
    public void quack() { System.out.println("Quack!"); }
}

public class Squeak implements QuackBehavior {
    public void quack() { System.out.println("Squeak!"); }
}

public class MuteQuack implements QuackBehavior {
    public void quack() { /* silence */ }
}

public class FlyWithWings implements FlyBehavior {
    public void fly() { System.out.println("Flying with wings!"); }
}

public class FlyNoWay implements FlyBehavior {
    public void fly() { /* cannot fly */ }
}

public class FlyWithRocket implements FlyBehavior {
    public void fly() { System.out.println("Flying with rocket booster!"); }
}

// Duck composes behaviors instead of inheriting them
public abstract class Duck {
    protected QuackBehavior quackBehavior;
    protected FlyBehavior flyBehavior;

    public Duck(QuackBehavior quackBehavior, FlyBehavior flyBehavior) {
        this.quackBehavior = quackBehavior;
        this.flyBehavior = flyBehavior;
    }

    public void performQuack() { quackBehavior.quack(); }
    public void performFly()   { flyBehavior.fly(); }
    public void swim()         { System.out.println("Swimming..."); }
    public abstract void display();

    // Behavior can change at runtime!
    public void setFlyBehavior(FlyBehavior fb)     { this.flyBehavior = fb; }
    public void setQuackBehavior(QuackBehavior qb) { this.quackBehavior = qb; }
}

public class MallardDuck extends Duck {
    public MallardDuck() {
        super(new Quack(), new FlyWithWings());
    }
    @Override
    public void display() { System.out.println("Mallard Duck"); }
}

public class RubberDuck extends Duck {
    public RubberDuck() {
        super(new Squeak(), new FlyNoWay()); // correct behavior, no override needed
    }
    @Override
    public void display() { System.out.println("Rubber Duck"); }
}

// Runtime behavior change
Duck mallard = new MallardDuck();
mallard.performFly();                        // Flying with wings!
mallard.setFlyBehavior(new FlyWithRocket());
mallard.performFly();                        // Flying with rocket booster!
```

Now adding a new flying behavior requires only a new `FlyBehavior` implementation. No existing Duck class needs to change. This is the Open/Closed Principle achieved through composition.

---

## 10. Case Study: Shape Drawing System

### Inheritance Approach (Limited)

```java
public abstract class Shape {
    private Color color;

    public Shape(Color color) { this.color = color; }
    public Color getColor()   { return color; }

    public abstract double area();
    public abstract double perimeter();
    public abstract void draw(); // Problem: couples rendering to the shape
}

public class Circle extends Shape {
    private double radius;

    public Circle(Color color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double area()      { return Math.PI * radius * radius; }
    @Override
    public double perimeter() { return 2 * Math.PI * radius; }
    @Override
    public void draw()        { /* draws circle to screen — what screen? */ }
}
```

Problem: `draw()` is baked into `Shape`. What if you need to render to a screen, a PDF, and an SVG file? You would need `drawToScreen()`, `drawToPdf()`, `drawToSvg()` — or a parallel hierarchy of `PdfCircle`, `ScreenCircle`, `SvgCircle`.

### Composition Approach (Flexible)

```java
// Shape is pure geometry — no rendering knowledge
public abstract class Shape {
    public abstract double area();
    public abstract double perimeter();
    public abstract void accept(ShapeRenderer renderer); // visitor pattern hook
}

public class Circle extends Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    public double getRadius()    { return radius; }

    @Override
    public double area()      { return Math.PI * radius * radius; }
    @Override
    public double perimeter() { return 2 * Math.PI * radius; }
    @Override
    public void accept(ShapeRenderer renderer) { renderer.render(this); }
}

public class Rectangle extends Shape {
    private final double width, height;
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    public double getWidth()  { return width; }
    public double getHeight() { return height; }

    @Override
    public double area()      { return width * height; }
    @Override
    public double perimeter() { return 2 * (width + height); }
    @Override
    public void accept(ShapeRenderer renderer) { renderer.render(this); }
}

// Rendering as a separate concern — composed, not inherited
public interface ShapeRenderer {
    void render(Circle circle);
    void render(Rectangle rectangle);
}

public class ScreenRenderer implements ShapeRenderer {
    @Override
    public void render(Circle c) {
        System.out.printf("Drawing circle with radius=%.2f on screen%n", c.getRadius());
    }
    @Override
    public void render(Rectangle r) {
        System.out.printf("Drawing %sx%s rectangle on screen%n",
            r.getWidth(), r.getHeight());
    }
}

public class PdfRenderer implements ShapeRenderer {
    @Override
    public void render(Circle c) {
        System.out.printf("Rendering circle to PDF%n");
    }
    @Override
    public void render(Rectangle r) {
        System.out.printf("Rendering rectangle to PDF%n");
    }
}

// Canvas composes shapes and a renderer
public class Canvas {
    private final List<Shape> shapes = new ArrayList<>();
    private ShapeRenderer renderer;

    public Canvas(ShapeRenderer renderer) {
        this.renderer = renderer;
    }

    public void add(Shape shape)                 { shapes.add(shape); }
    public void setRenderer(ShapeRenderer r)     { this.renderer = r; }

    public void draw() {
        shapes.forEach(shape -> shape.accept(renderer));
    }
}

// Usage
Canvas canvas = new Canvas(new ScreenRenderer());
canvas.add(new Circle(5.0));
canvas.add(new Rectangle(3.0, 4.0));
canvas.draw();

// Switch renderer at runtime — no shape code changes
canvas.setRenderer(new PdfRenderer());
canvas.draw();
```

New rendering target? Add a new `ShapeRenderer` implementation. New shape? Add a new `Shape` subclass. Neither change touches the other side.

---

## 11. Decision Framework

Use this framework when deciding between inheritance and composition:

```
1. Is it a genuine is-a relationship?
   - Ask: "Is [subclass] always a kind of [superclass] in every context?"
   - Ask: "Can I substitute [subclass] everywhere [superclass] is used without breaking correctness?" (LSP)
   - If NO to either -> use composition

2. Does the subclass need all of the superclass's public interface?
   - If the subclass must override methods to throw UnsupportedOperationException -> use composition

3. Will you ever need multiple parents?
   - Composition supports multiple behaviors naturally -> use composition

4. Do you need to change behavior at runtime?
   - Inheritance is static -> use composition

5. Is the base class designed for extension?
   - Check: does it have protected methods? abstract methods? javadoc saying "subclass to extend"?
   - If YES -> inheritance may be appropriate

6. Is the superclass from a third-party library you don't control?
   - If YES -> prefer composition (fragile base class risk is highest here)

DEFAULT: When in doubt, start with composition.
```

---

## 12. Interview Questions with Model Answers

**Q1. Why does the Gang of Four recommend favoring composition over inheritance?**

The GoF identified two core problems with heavy inheritance. First, inheritance breaks encapsulation — subclasses depend on the implementation details of the parent class, not just its interface, so a change to the parent can break subclasses even when the subclass code is unchanged. Second, inheritance is static — class relationships are fixed at compile time, making runtime behavior changes impossible. Composition avoids both: the composed object's implementation is hidden behind its interface, and components can be swapped at runtime.

**Q2. Explain the fragile base class problem with a concrete example.**

When you extend `ArrayList` and override both `add()` and `addAll()`, you find that your `addAll()` override double-counts additions because `ArrayList.addAll()` internally calls `add()`. Your subclass code is correct in isolation, but it breaks because it depends on how the parent internally implements a method — an implementation detail that is not part of the contract. The fix is composition: hold a reference to the list and delegate, so the internal implementation of the delegate is irrelevant.

**Q3. When is inheritance the right choice?**

When there is a genuine is-a relationship where the Liskov Substitution Principle holds — meaning you can substitute the subclass everywhere the superclass is used without affecting correctness. A `SavingsAccount` is a `BankAccount`. A `MallardDuck` is a `Duck`. Inheritance is also appropriate when extending a framework-designed hierarchy where the base class has abstract methods explicitly inviting subclassing.

**Q4. What is the deadly diamond of death and how does Java handle it?**

The diamond problem occurs in multiple inheritance when a class inherits from two classes that share a common ancestor, and both intermediate classes override a method — creating ambiguity about which implementation is used. Java avoids this by not allowing multiple class inheritance. However, Java 8 interface default methods can reproduce a limited version: if two interfaces provide a default method with the same signature, an implementing class must explicitly override it and choose which to use with `InterfaceName.super.method()`.

**Q5. How does composition improve testability?**

With composition, you inject dependencies through the constructor. In tests, you replace those dependencies with mocks or stubs. The class under test operates on the mock, not the real dependency. The test is isolated: it only exercises the logic of the class under test, not its dependencies. With inheritance, the subclass carries all the behavior of the parent. Testing just the subclass in isolation requires either a partial mock of the parent or running the full parent initialization.

**Q6. What is the Strategy pattern and how does it embody composition over inheritance?**

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. The context class holds a reference to a strategy interface rather than inheriting from an algorithm class. This allows the algorithm to vary independently from the class that uses it. It solves the problem of wanting to select an algorithm at runtime — something inheritance cannot do. The Duck example demonstrates this: `FlyBehavior` is the strategy, and the Duck composes it rather than inheriting a fixed fly() behavior.

**Q7. Can you give an example where using inheritance led to a design problem and how composition would fix it?**

The classic case is `java.util.Stack` extending `java.util.Vector`. A Stack should expose only push, pop, and peek. But because it inherits Vector, it also exposes `add(int index, E element)`, `remove(int index)`, and other random-access methods that violate the LIFO contract of a Stack. You can insert into the middle of a "stack." The fix would have been composition: `Stack` holds a `Vector` internally and exposes only the stack operations. This is documented as a design mistake in the Java standard library.

**Q8. What is the difference between an is-a and a has-a relationship? How do you distinguish them in practice?**

Is-a means the subtype is a specialization of the supertype and upholds all of the supertype's contracts (LSP). A Circle is a Shape. A SavingsAccount is a BankAccount. Has-a means a class contains and uses another class. A Car has an Engine. A University has Students. The practical test: if you remove the parent type from an is-a subclass, the subclass loses its fundamental identity. If you remove a component from a has-a class, the class can still exist — it just loses a capability.

**Q9. What are deep inheritance hierarchies and why are they problematic?**

Deep hierarchies (3+ levels) make behavior hard to trace because method dispatch traverses multiple levels. To understand what a method does, you must read 4 or more files. Changes at any level can have unexpected effects on all descendants. Testing becomes harder because setting up a leaf class requires satisfying all ancestor constructors. The GoF recommends keeping hierarchies shallow (no more than 2 levels in most cases) and using composition for the behavior variation.

**Q10. What is the Mixin pattern in Java and what are its limitations?**

A mixin is a way to add reusable behavior to a class without inheritance. In Java, interface default methods can act as mixins — a class implements the interface and gets the default behavior without extending a class. This is useful for cross-cutting concerns like logging or timing. Limitations: default methods cannot access instance state of the implementing class, they add methods to the public interface (polluting the API), and they can create diamond-method conflicts that must be resolved explicitly. They work best for stateless, utility-like behaviors.
