# Bridge Design Pattern

> "Decouple an abstraction from its implementation so that the two can vary independently."
> — Gang of Four, *Design Patterns: Elements of Reusable Object-Oriented Software*

---

## Table of Contents

1. [Intent](#1-intent)
2. [The Cartesian Product Problem — Class Explosion](#2-the-cartesian-product-problem--class-explosion)
3. [Real-World Analogy](#3-real-world-analogy)
4. [UML Class Diagram](#4-uml-class-diagram)
5. [When to Use](#5-when-to-use)
6. [When NOT to Use](#6-when-not-to-use)
7. [Java Implementation: Shape & Color](#7-java-implementation-shape--color)
8. [Java Implementation: Remote Control & Device](#8-java-implementation-remote-control--device)
9. [Java Implementation: Notification System](#9-java-implementation-notification-system)
10. [Real-World Examples in the Java Ecosystem](#10-real-world-examples-in-the-java-ecosystem)
11. [Bridge vs Adapter](#11-bridge-vs-adapter)
12. [Bridge vs Strategy](#12-bridge-vs-strategy)
13. [Bridge vs Abstract Factory](#13-bridge-vs-abstract-factory)
14. [Trade-offs](#14-trade-offs)
15. [Common Pitfalls and Gotchas](#15-common-pitfalls-and-gotchas)
16. [Interview Discussion Points](#16-interview-discussion-points)
17. [Interview Questions with Model Answers](#17-interview-questions-with-model-answers)

---

## 1. Intent

The Bridge pattern is a **structural design pattern** that separates an abstraction from its implementation by placing them in two distinct class hierarchies that are connected by a composition relationship — the "bridge."

### The Core Idea

In object-oriented programming, inheritance is a powerful mechanism for code reuse. However, when a class can vary along **two independent dimensions**, modeling both dimensions through inheritance creates a combinatorial explosion of subclasses. The Bridge pattern breaks this coupling by introducing an interface for one dimension and composing it into the abstraction hierarchy of the other.

The pattern achieves two primary goals:

1. **Decoupling abstraction from implementation**: The abstraction does not directly implement its behavior. Instead, it holds a reference to an implementor interface and delegates work to it. The concrete abstraction shapes the high-level logic; the concrete implementor provides the platform- or technology-specific details.

2. **Independent variability**: Because the two hierarchies are separate, you can extend the abstraction side (add new shapes, new remote controls, new notification types) without touching the implementation side, and vice versa. A new `BlueColor` implementor does not require any change to `Circle` or `Square`. A new `Circle` abstraction does not require any change to `RedColor` or `GreenColor`.

### What "Abstraction" and "Implementation" Mean Here

These terms are used in the GoF sense, not in the Java language sense:

- **Abstraction** does not mean an `abstract` class or `interface` keyword. It means the high-level control layer — the part of the design that a client interacts with.
- **Implementation** does not mean the concrete class that `implements` an interface. It means the low-level platform layer — the part that does the actual work underneath.

A `RemoteControl` is an abstraction. A `TV` is an implementation. The remote defines the high-level operations (`volumeUp`, `volumeDown`, `channelUp`). The TV device knows the actual hardware-level mechanism to carry those operations out.

---

## 2. The Cartesian Product Problem — Class Explosion

This is the foundational motivation for the Bridge pattern and must be understood deeply before looking at any code.

### Scenario: Shapes and Colors

Suppose you are designing a drawing library. You have shapes and colors. Using naive inheritance, you extend each shape with a color:

```
Shape (abstract)
├── RedCircle
├── GreenCircle
├── BlueCircle
├── RedSquare
├── GreenSquare
└── BlueSquare
```

This is the **Cartesian product problem**. With M shapes and N colors, you need M × N classes. Adding a fourth color `Yellow` requires 2 new classes (`YellowCircle`, `YellowSquare`). Adding a third shape `Triangle` requires 3 new classes (`RedTriangle`, `GreenTriangle`, `BlueTriangle`).

The class count explodes:

| Shapes \ Colors | Red | Green | Blue | Yellow |
|-----------------|-----|-------|------|--------|
| Circle          |  1  |   1   |   1  |    1   |
| Square          |  1  |   1   |   1  |    1   |
| Triangle        |  1  |   1   |   1  |    1   |

For 3 shapes and 4 colors: **12 classes**, each with largely duplicated logic.

### The Bridge Solution

Instead of creating a class for every combination, the Bridge pattern uses composition:

- **M abstraction classes** (shapes): `Circle`, `Square`, `Triangle`
- **N implementor classes** (colors): `RedColor`, `GreenColor`, `BlueColor`, `YellowColor`
- **Total classes**: M + N (3 + 4 = 7, instead of 12)

Adding a new color: **1 new class**. Adding a new shape: **1 new class**.

This is the mathematical leverage of the Bridge pattern: it converts a multiplicative relationship (M × N) into an additive one (M + N).

---

## 3. Real-World Analogy

### The Universal Remote Control

Consider a universal TV remote control. The remote is the **abstraction** — it provides buttons like `Power`, `Volume Up`, `Volume Down`, `Channel Up`, `Channel Down`. A user thinks in terms of these operations.

The **TV brand** is the implementation — Samsung TV communicates over one proprietary protocol, Sony TV over another, LG TV over another. The remote does not care which TV it is pointed at. You can swap the TV (change the implementation) without getting a new remote. You can upgrade to a universal remote (change or extend the abstraction) without buying a new TV.

The "bridge" is the infrared or Bluetooth communication interface that the remote and every TV agrees on. This is the `Device` interface in code.

This is a physical example of programming to an interface rather than to a concrete class — and of decoupling two independently varying hierarchies through a shared contract.

### The Electrical Plug Analogy

A power plug (abstraction) defines a standard interface: two or three prongs delivering voltage and current. A power outlet (implementation) provides the physical socket. You can plug a laptop, a lamp, or a phone charger (different abstractions) into the same outlet. You can also travel to a different country (different implementation — different voltage, socket shape, using an adapter) without redesigning your laptop.

---

## 4. UML Class Diagram

### General Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT                                      │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │ uses
                                  ▼
          ┌───────────────────────────────────────┐
          │           Abstraction                 │
          │───────────────────────────────────────│
          │  # implementor: Implementor           │
          │───────────────────────────────────────│
          │  + Abstraction(impl: Implementor)     │
          │  + operation(): void                  │
          └───────────────┬───────────────────────┘
                          │                    ◆ (composition)
               ┌──────────┴───────────┐        │
               │                     │         │ references
               ▼                     ▼         ▼
  ┌─────────────────────┐  ┌─────────────────────────────────┐
  │  RefinedAbstraction │  │         <<interface>>           │
  │─────────────────────│  │          Implementor            │
  │  + operation(): void│  │─────────────────────────────────│
  │  + extendedOp(): ..│  │  + operationImpl(): void        │
  └─────────────────────┘  └────────────────┬────────────────┘
                                            │
                             ┌──────────────┴──────────────┐
                             │                             │
                             ▼                             ▼
               ┌─────────────────────────┐  ┌─────────────────────────┐
               │  ConcreteImplementorA   │  │  ConcreteImplementorB   │
               │─────────────────────────│  │─────────────────────────│
               │  + operationImpl(): void│  │  + operationImpl(): void│
               └─────────────────────────┘  └─────────────────────────┘
```

### Shape/Color Specific Diagram

```
          ┌─────────────────────────────────────────────────────┐
          │               Shape (Abstraction)                   │
          │─────────────────────────────────────────────────────│
          │  # color: Color                                     │
          │─────────────────────────────────────────────────────│
          │  + Shape(color: Color)                              │
          │  + draw(): void  (abstract)                         │
          └────────────────┬────────────────────────────────────┘
                           │
             ┌─────────────┴──────────────┐
             ▼                            ▼
 ┌───────────────────────┐  ┌─────────────────────────────┐
 │  Circle               │  │  Square                     │
 │  (RefinedAbstraction) │  │  (RefinedAbstraction)       │
 │───────────────────────│  │─────────────────────────────│
 │  - radius: double     │  │  - side: double             │
 │───────────────────────│  │─────────────────────────────│
 │  + draw(): void       │  │  + draw(): void             │
 └───────────────────────┘  └─────────────────────────────┘

          ┌─────────────────────────────────────────────────────┐
          │                <<interface>>                        │
          │                   Color  (Implementor)              │
          │─────────────────────────────────────────────────────│
          │  + applyColor(): String                             │
          └────────────────┬────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
 ┌────────────┐   ┌─────────────────┐  ┌──────────────┐
 │  RedColor  │   │   GreenColor    │  │  BlueColor   │
 │────────────│   │─────────────────│  │──────────────│
 │+applyColor │   │  +applyColor()  │  │+applyColor() │
 │  (): String│   │      : String   │  │    : String  │
 └────────────┘   └─────────────────┘  └──────────────┘
```

---

## 5. When to Use

Use the Bridge pattern when:

1. **You want to avoid a permanent binding between abstraction and implementation.** The implementation should be selectable or switchable at runtime — for example, choosing between a file-based logger and a database logger without changing any calling code.

2. **Both abstractions and implementations should be extensible through subclassing.** You have two independent dimensions of variation and want to combine them freely rather than creating every combination.

3. **Changes to the implementation of an abstraction should have no impact on clients.** The client code should not be recompiled when a new `ConcreteImplementor` is added or an existing one is modified.

4. **You have a proliferating class hierarchy.** If you find yourself creating classes like `WindowsXMLLogger`, `LinuxXMLLogger`, `WindowsJSONLogger`, `LinuxJSONLogger`, you have identified two dimensions (OS and format) and the Bridge pattern will reduce the explosion to `XMLLogger`, `JSONLogger`, `WindowsSystem`, `LinuxSystem`.

5. **You need to share an implementation among multiple objects.** Several abstraction objects can share a single implementor instance (for example, multiple remote controls pointing at the same TV).

6. **You want to hide implementation details completely.** The implementation classes can be package-private; clients only ever see the abstraction interface.

---

## 6. When NOT to Use

Avoid the Bridge pattern when:

1. **There is only one implementation and it is unlikely to change.** The extra indirection adds cognitive overhead with no payoff. YAGNI applies: do not introduce the Bridge pattern speculatively.

2. **The abstraction and implementation are naturally stable.** If the two dimensions do not vary independently, a simple class hierarchy is clearer.

3. **You need maximum performance on a hot path.** The extra virtual dispatch through the implementor interface adds a small but real overhead. In latency-critical inner loops, direct method calls are faster.

4. **The team is unfamiliar with structural patterns.** The Bridge pattern requires some design sophistication to understand. In a junior team, a simpler composition approach with a clear name may communicate intent better.

5. **The problem is a single variation axis.** If only the implementation varies and the abstraction is fixed (always the same type), the Strategy pattern is a simpler and more widely understood fit. Bridge is specifically for two independently varying hierarchies.

---

## 7. Java Implementation: Shape & Color

This is the canonical Bridge example. `Shape` is the abstraction hierarchy; `Color` is the implementor hierarchy.

```java
package com.course.patterns.bridge.shape;

/**
 * Implementor interface — defines the contract for the color rendering hierarchy.
 *
 * <p>All concrete color implementations must provide an {@code applyColor()} method
 * that returns an ANSI-styled string representation of the color. This interface
 * is the "bridge" between the shape abstraction and the color implementation.
 *
 * <p>The Color hierarchy can evolve completely independently of the Shape hierarchy.
 */
public interface Color {

    /**
     * Returns a display representation annotated with this color.
     *
     * @param shape a textual description of the shape being colored
     * @return a color-annotated representation string
     */
    String applyColor(String shape);

    /**
     * Returns the human-readable name of this color, suitable for logging.
     *
     * @return color name, e.g., "Red", "Green", "Blue"
     */
    String getColorName();
}
```

```java
package com.course.patterns.bridge.shape;

/**
 * ConcreteImplementor — Red color rendering.
 *
 * <p>Uses ANSI escape codes so output is visibly colored when run in a
 * terminal that supports ANSI sequences (standard on Linux/macOS; requires
 * Windows Terminal or ANSICON on Windows).
 */
public class RedColor implements Color {

    private static final String ANSI_RED   = "[31m";
    private static final String ANSI_RESET = "[0m";

    @Override
    public String applyColor(String shape) {
        return ANSI_RED + "[RED]  " + shape + ANSI_RESET;
    }

    @Override
    public String getColorName() {
        return "Red";
    }
}
```

```java
package com.course.patterns.bridge.shape;

/**
 * ConcreteImplementor — Green color rendering.
 */
public class GreenColor implements Color {

    private static final String ANSI_GREEN = "[32m";
    private static final String ANSI_RESET = "[0m";

    @Override
    public String applyColor(String shape) {
        return ANSI_GREEN + "[GREEN] " + shape + ANSI_RESET;
    }

    @Override
    public String getColorName() {
        return "Green";
    }
}
```

```java
package com.course.patterns.bridge.shape;

/**
 * ConcreteImplementor — Blue color rendering.
 */
public class BlueColor implements Color {

    private static final String ANSI_BLUE  = "[34m";
    private static final String ANSI_RESET = "[0m";

    @Override
    public String applyColor(String shape) {
        return ANSI_BLUE + "[BLUE]  " + shape + ANSI_RESET;
    }

    @Override
    public String getColorName() {
        return "Blue";
    }
}
```

```java
package com.course.patterns.bridge.shape;

/**
 * Abstraction — the base class for all shapes.
 *
 * <p>This class holds a reference to a {@link Color} implementor and delegates
 * the rendering logic to it. Subclasses define the geometry; the Color implementor
 * defines the rendering. The two hierarchies are decoupled and can vary independently.
 *
 * <p>Note that the {@code color} field is {@code protected} so that subclasses can
 * pass shape-specific descriptions down to the implementor.
 */
public abstract class Shape {

    /**
     * The bridge to the Color implementor hierarchy.
     *
     * <p>Declared {@code protected} so that {@link RefinedAbstraction} subclasses
     * can compose the color rendering into their own draw logic. In a stricter design,
     * this could be {@code private} with a helper method {@code applyColor(String)} on
     * the abstraction.
     */
    protected final Color color;

    /**
     * Constructs a Shape with the given color implementor.
     *
     * @param color the color rendering strategy; must not be {@code null}
     * @throws IllegalArgumentException if {@code color} is {@code null}
     */
    protected Shape(Color color) {
        if (color == null) {
            throw new IllegalArgumentException("Color implementor must not be null");
        }
        this.color = color;
    }

    /**
     * Renders this shape to a string.
     *
     * <p>Subclasses must implement this method, typically by calling
     * {@code color.applyColor(geometryDescription)} to produce the final output.
     *
     * @return a rendered string representation of this shape with its color
     */
    public abstract String draw();

    /**
     * Returns the area of this shape.
     *
     * @return area in square units
     */
    public abstract double area();
}
```

```java
package com.course.patterns.bridge.shape;

/**
 * RefinedAbstraction — a circle with a given radius.
 *
 * <p>Defines the circle-specific geometry (radius, circumference, area) and
 * delegates color rendering entirely to the {@link Color} implementor. The Circle
 * class knows nothing about how colors are rendered — it only knows its own geometry.
 */
public class Circle extends Shape {

    private final double radius;

    /**
     * Constructs a Circle with the given radius and color.
     *
     * @param radius positive radius in units
     * @param color  color implementor to use for rendering
     * @throws IllegalArgumentException if {@code radius} is non-positive
     */
    public Circle(double radius, Color color) {
        super(color);
        if (radius <= 0) {
            throw new IllegalArgumentException("Radius must be positive, got: " + radius);
        }
        this.radius = radius;
    }

    /**
     * Renders this circle by delegating color application to the implementor.
     *
     * @return colored string representation of the circle
     */
    @Override
    public String draw() {
        String description = String.format("Circle(radius=%.2f, area=%.2f)", radius, area());
        return color.applyColor(description);
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }

    public double getRadius() {
        return radius;
    }
}
```

```java
package com.course.patterns.bridge.shape;

/**
 * RefinedAbstraction — a square with a given side length.
 *
 * <p>Knows only its own geometry. Color rendering is entirely delegated to the
 * bridge ({@link Color} implementor).
 */
public class Square extends Shape {

    private final double side;

    /**
     * Constructs a Square with the given side length and color.
     *
     * @param side  positive side length in units
     * @param color color implementor to use for rendering
     * @throws IllegalArgumentException if {@code side} is non-positive
     */
    public Square(double side, Color color) {
        super(color);
        if (side <= 0) {
            throw new IllegalArgumentException("Side must be positive, got: " + side);
        }
        this.side = side;
    }

    @Override
    public String draw() {
        String description = String.format("Square(side=%.2f, area=%.2f)", side, area());
        return color.applyColor(description);
    }

    @Override
    public double area() {
        return side * side;
    }

    public double getSide() {
        return side;
    }
}
```

```java
package com.course.patterns.bridge.shape;

/**
 * Demonstrates the Bridge pattern with shapes and colors.
 *
 * <p>Key observation: 2 shapes × 3 colors = 6 combinations, but only 5 classes total.
 * Adding a fourth color requires exactly ONE new class (YellowColor).
 * Adding a third shape requires exactly ONE new class (Triangle).
 */
public class ShapeBridgeDemo {

    public static void main(String[] args) {
        // Each shape is freely combined with any color at construction time.
        // No RedCircle, GreenCircle, BlueSquare classes needed.

        Shape redCircle   = new Circle(5.0,  new RedColor());
        Shape greenCircle = new Circle(5.0,  new GreenColor());
        Shape blueSquare  = new Square(4.0,  new BlueColor());
        Shape greenSquare = new Square(4.0,  new GreenColor());

        System.out.println("=== Bridge Pattern: Shape & Color Demo ===");
        System.out.println(redCircle.draw());
        System.out.println(greenCircle.draw());
        System.out.println(blueSquare.draw());
        System.out.println(greenSquare.draw());

        // Runtime swapping — the implementor can even change at runtime
        // if the field is not final. This shows decoupling.
        System.out.println("\n=== Runtime Color Substitution ===");
        Color[] colors = { new RedColor(), new GreenColor(), new BlueColor() };
        Shape   shape  = new Circle(3.0, colors[0]);
        // In a mutable variant, color could be swapped here.
        // Here we demonstrate construction-time combination.
        for (Color c : colors) {
            Shape s = new Circle(3.0, c);
            System.out.println(s.draw());
        }
    }
}
```

---

## 8. Java Implementation: Remote Control & Device

This is the GoF's own canonical example domain. The abstraction hierarchy is `RemoteControl`; the implementor hierarchy is `Device`.

```java
package com.course.patterns.bridge.remote;

/**
 * Implementor interface — Device.
 *
 * <p>Defines the low-level operations that every device must support.
 * The remote control abstraction talks only to this interface, never to
 * a specific device class such as {@code SamsungTV} or {@code SonyRadio}.
 *
 * <p>A device implementation knows the hardware-specific protocol (HDMI-CEC,
 * infrared, Bluetooth, etc.). The remote control does not care which protocol
 * is used — it only calls the methods below.
 */
public interface Device {

    /** Returns {@code true} if the device is currently powered on. */
    boolean isEnabled();

    /** Switches the device on or off. */
    void enable();

    /** Switches the device off. */
    void disable();

    /**
     * Returns the current volume level.
     *
     * @return volume in the range [0, 100]
     */
    int getVolume();

    /**
     * Sets the volume level.
     *
     * @param volume target volume in the range [0, 100]
     * @throws IllegalArgumentException if volume is out of range
     */
    void setVolume(int volume);

    /**
     * Returns the current channel number.
     *
     * @return current channel, always &gt;= 1
     */
    int getChannel();

    /**
     * Sets the channel.
     *
     * @param channel target channel number, must be &gt;= 1
     */
    void setChannel(int channel);

    /** Returns the human-readable name of this device. */
    String getName();

    /** Returns a full status string suitable for logging or display. */
    String getStatus();
}
```

```java
package com.course.patterns.bridge.remote;

/**
 * ConcreteImplementor — Television.
 *
 * <p>Models a television set with volume, channel, and power state.
 * In a real system, this class would contain the native driver calls
 * (e.g., HDMI-CEC API, V4L2, or a vendor SDK). Here it uses in-memory state.
 */
public class TV implements Device {

    private static final int MIN_VOLUME  = 0;
    private static final int MAX_VOLUME  = 100;
    private static final int MIN_CHANNEL = 1;

    private boolean on      = false;
    private int     volume  = 30;
    private int     channel = 1;

    @Override
    public boolean isEnabled() {
        return on;
    }

    @Override
    public void enable() {
        System.out.println("TV: powering ON");
        on = true;
    }

    @Override
    public void disable() {
        System.out.println("TV: powering OFF");
        on = false;
    }

    @Override
    public int getVolume() {
        return volume;
    }

    @Override
    public void setVolume(int volume) {
        if (volume < MIN_VOLUME || volume > MAX_VOLUME) {
            throw new IllegalArgumentException(
                "TV volume must be in [" + MIN_VOLUME + ", " + MAX_VOLUME + "], got: " + volume);
        }
        System.out.printf("TV: setting volume to %d%n", volume);
        this.volume = volume;
    }

    @Override
    public int getChannel() {
        return channel;
    }

    @Override
    public void setChannel(int channel) {
        if (channel < MIN_CHANNEL) {
            throw new IllegalArgumentException("TV channel must be >= " + MIN_CHANNEL);
        }
        System.out.printf("TV: tuning to channel %d%n", channel);
        this.channel = channel;
    }

    @Override
    public String getName() {
        return "Samsung QLED TV";
    }

    @Override
    public String getStatus() {
        return String.format("TV[power=%s, volume=%d, channel=%d]",
                             on ? "ON" : "OFF", volume, channel);
    }
}
```

```java
package com.course.patterns.bridge.remote;

/**
 * ConcreteImplementor — Radio.
 *
 * <p>Differs from TV in that "channel" is interpreted as a frequency
 * band index, and the volume range is [0, 10] (radio hardware uses a
 * different scale). This shows that two ConcreteImplementors can have
 * different internal behavior behind the same Device interface.
 */
public class Radio implements Device {

    private boolean on      = false;
    private int     volume  = 5;
    private int     channel = 1;  // band index: 1=FM1, 2=FM2, ...

    @Override
    public boolean isEnabled() { return on; }

    @Override
    public void enable() {
        System.out.println("Radio: powering ON");
        on = true;
    }

    @Override
    public void disable() {
        System.out.println("Radio: powering OFF");
        on = false;
    }

    @Override
    public int getVolume() { return volume; }

    @Override
    public void setVolume(int volume) {
        // Radio uses scale 0-10, but we accept 0-100 and rescale
        this.volume = Math.min(10, Math.max(0, volume / 10));
        System.out.printf("Radio: volume set to %d (internal: %d)%n", volume, this.volume);
    }

    @Override
    public int getChannel() { return channel; }

    @Override
    public void setChannel(int channel) {
        if (channel < 1) throw new IllegalArgumentException("Radio channel must be >= 1");
        System.out.printf("Radio: switching to band %d%n", channel);
        this.channel = channel;
    }

    @Override
    public String getName() { return "Sony FM Radio"; }

    @Override
    public String getStatus() {
        return String.format("Radio[power=%s, volume=%d, band=%d]",
                             on ? "ON" : "OFF", volume, channel);
    }
}
```

```java
package com.course.patterns.bridge.remote;

/**
 * ConcreteImplementor — DVD Player.
 *
 * <p>Demonstrates that a device may have operations that differ significantly
 * in semantics from a TV. "Channel" maps to a disc track number.
 */
public class DVDPlayer implements Device {

    private boolean on      = false;
    private int     volume  = 20;
    private int     track   = 1;

    @Override
    public boolean isEnabled() { return on; }

    @Override
    public void enable() {
        System.out.println("DVD Player: powering ON, disc loading...");
        on = true;
    }

    @Override
    public void disable() {
        System.out.println("DVD Player: ejecting disc, powering OFF");
        on = false;
    }

    @Override
    public int getVolume() { return volume; }

    @Override
    public void setVolume(int volume) {
        this.volume = Math.min(100, Math.max(0, volume));
        System.out.printf("DVD Player: volume = %d%n", this.volume);
    }

    @Override
    public int getChannel() { return track; }  // "channel" = track number

    @Override
    public void setChannel(int channel) {
        this.track = Math.max(1, channel);
        System.out.printf("DVD Player: jumping to track %d%n", this.track);
    }

    @Override
    public String getName() { return "Panasonic DVD Player"; }

    @Override
    public String getStatus() {
        return String.format("DVD[power=%s, volume=%d, track=%d]",
                             on ? "ON" : "OFF", volume, track);
    }
}
```

```java
package com.course.patterns.bridge.remote;

/**
 * Abstraction — BasicRemote.
 *
 * <p>The basic remote control provides fundamental operations: power toggle,
 * volume control, and channel navigation. It delegates all work to the
 * {@link Device} implementor. It has no idea whether it is controlling a TV,
 * a Radio, or a DVD player.
 *
 * <p>The {@link Device} reference is the "bridge" — the link between the
 * abstraction hierarchy and the implementor hierarchy.
 */
public class BasicRemote {

    /** The bridge to the Device implementor. */
    protected final Device device;

    /**
     * Constructs a BasicRemote wired to the given device.
     *
     * @param device the device to control; must not be {@code null}
     */
    public BasicRemote(Device device) {
        if (device == null) throw new IllegalArgumentException("Device must not be null");
        this.device = device;
    }

    /**
     * Toggles the power state of the device.
     */
    public void power() {
        System.out.printf("Remote -> %s: POWER%n", device.getName());
        if (device.isEnabled()) {
            device.disable();
        } else {
            device.enable();
        }
    }

    /**
     * Increases the device volume by a fixed step.
     */
    public void volumeUp() {
        int newVolume = Math.min(100, device.getVolume() + 10);
        System.out.printf("Remote -> %s: VOLUME UP to %d%n", device.getName(), newVolume);
        device.setVolume(newVolume);
    }

    /**
     * Decreases the device volume by a fixed step.
     */
    public void volumeDown() {
        int newVolume = Math.max(0, device.getVolume() - 10);
        System.out.printf("Remote -> %s: VOLUME DOWN to %d%n", device.getName(), newVolume);
        device.setVolume(newVolume);
    }

    /**
     * Advances to the next channel.
     */
    public void channelUp() {
        int next = device.getChannel() + 1;
        System.out.printf("Remote -> %s: CHANNEL UP to %d%n", device.getName(), next);
        device.setChannel(next);
    }

    /**
     * Goes back to the previous channel.
     */
    public void channelDown() {
        int prev = Math.max(1, device.getChannel() - 1);
        System.out.printf("Remote -> %s: CHANNEL DOWN to %d%n", device.getName(), prev);
        device.setChannel(prev);
    }

    /**
     * Prints the current device status.
     */
    public void status() {
        System.out.println("Status: " + device.getStatus());
    }
}
```

```java
package com.course.patterns.bridge.remote;

/**
 * RefinedAbstraction — AdvancedRemote.
 *
 * <p>Extends the BasicRemote with additional operations: mute and direct
 * channel selection. This is a refined abstraction — it builds on the
 * bridge without touching the device implementations at all.
 *
 * <p>Notice that adding this new class required zero changes to TV, Radio,
 * or DVDPlayer. That is the power of the Bridge pattern.
 */
public class AdvancedRemote extends BasicRemote {

    /**
     * Constructs an AdvancedRemote wired to the given device.
     *
     * @param device the device to control
     */
    public AdvancedRemote(Device device) {
        super(device);
    }

    /**
     * Mutes the device by setting its volume to zero.
     *
     * <p>A real implementation might track the pre-mute volume to restore it.
     */
    public void mute() {
        System.out.printf("AdvancedRemote -> %s: MUTE%n", device.getName());
        device.setVolume(0);
    }

    /**
     * Jumps directly to the specified channel.
     *
     * @param channel target channel number; must be &gt;= 1
     */
    public void setChannel(int channel) {
        System.out.printf("AdvancedRemote -> %s: SET CHANNEL to %d%n",
                          device.getName(), channel);
        device.setChannel(channel);
    }

    /**
     * Sets volume to a precise level rather than incrementally.
     *
     * @param level volume level in [0, 100]
     */
    public void setVolume(int level) {
        System.out.printf("AdvancedRemote -> %s: SET VOLUME to %d%n",
                          device.getName(), level);
        device.setVolume(level);
    }
}
```

```java
package com.course.patterns.bridge.remote;

/**
 * Demonstrates the Remote/Device Bridge.
 *
 * <p>Key observations:
 * <ul>
 *   <li>One BasicRemote controls any Device implementation without modification.</li>
 *   <li>AdvancedRemote adds new operations without touching any Device class.</li>
 *   <li>Any future device (SteamingBox, GameConsole) plugs in with zero remote changes.</li>
 * </ul>
 */
public class RemoteBridgeDemo {

    public static void main(String[] args) {
        System.out.println("=== Basic Remote + TV ===");
        Device      tv          = new TV();
        BasicRemote basicTvRemote = new BasicRemote(tv);
        basicTvRemote.power();      // Turn on
        basicTvRemote.volumeUp();
        basicTvRemote.channelUp();
        basicTvRemote.status();

        System.out.println("\n=== Advanced Remote + Radio ===");
        Device         radio         = new Radio();
        AdvancedRemote advancedRadio = new AdvancedRemote(radio);
        advancedRadio.power();          // Turn on
        advancedRadio.setVolume(70);
        advancedRadio.setChannel(3);
        advancedRadio.mute();
        advancedRadio.status();

        System.out.println("\n=== Advanced Remote + DVD Player ===");
        Device         dvd      = new DVDPlayer();
        AdvancedRemote dvdCtrl  = new AdvancedRemote(dvd);
        dvdCtrl.power();
        dvdCtrl.setChannel(5);    // Jump to track 5
        dvdCtrl.setVolume(50);
        dvdCtrl.status();
    }
}
```

---

## 9. Java Implementation: Notification System

A production-grade example showing a notification system where the abstraction (notification type) and the implementation (message formatter) vary independently.

```java
package com.course.patterns.bridge.notification;

/**
 * Implementor interface — MessageFormatter.
 *
 * <p>Defines how a notification message is formatted for a given channel.
 * A plain-text formatter works for SMS and email text. An HTML formatter
 * works for email with rich clients. A JSON formatter works for REST webhook
 * push notifications. Each can vary independently of the notification type.
 */
public interface MessageFormatter {

    /**
     * Formats a notification message.
     *
     * @param subject the subject or title of the notification
     * @param body    the detailed body content
     * @param sender  the sender identifier (email, phone, service name)
     * @return the fully formatted message string
     */
    String format(String subject, String body, String sender);

    /**
     * Returns the MIME content type of the formatted output.
     *
     * @return MIME type string, e.g., "text/plain", "text/html", "application/json"
     */
    String contentType();
}
```

```java
package com.course.patterns.bridge.notification;

/**
 * ConcreteImplementor — PlainTextFormatter.
 *
 * <p>Produces a simple plain-text message suitable for SMS, terminal output,
 * or plain-text email clients.
 */
public class PlainTextFormatter implements MessageFormatter {

    @Override
    public String format(String subject, String body, String sender) {
        return String.format(
            "From: %s%n" +
            "Subject: %s%n" +
            "----%n" +
            "%s%n" +
            "----%n",
            sender, subject, body
        );
    }

    @Override
    public String contentType() {
        return "text/plain";
    }
}
```

```java
package com.course.patterns.bridge.notification;

/**
 * ConcreteImplementor — HtmlFormatter.
 *
 * <p>Produces an HTML-formatted message suitable for rich email clients
 * or web-hook endpoints that render HTML.
 */
public class HtmlFormatter implements MessageFormatter {

    @Override
    public String format(String subject, String body, String sender) {
        return String.format(
            "<!DOCTYPE html><html><body>%n" +
            "  <h2>%s</h2>%n" +
            "  <p><em>From: %s</em></p>%n" +
            "  <hr/>%n" +
            "  <p>%s</p>%n" +
            "</body></html>",
            escapeHtml(subject), escapeHtml(sender), escapeHtml(body)
        );
    }

    @Override
    public String contentType() {
        return "text/html";
    }

    private String escapeHtml(String raw) {
        if (raw == null) return "";
        return raw.replace("&", "&amp;")
                  .replace("<", "&lt;")
                  .replace(">", "&gt;")
                  .replace("\"", "&quot;");
    }
}
```

```java
package com.course.patterns.bridge.notification;

/**
 * ConcreteImplementor — JsonFormatter.
 *
 * <p>Produces a JSON payload suitable for push notification APIs (FCM, APNs)
 * or generic REST webhook integrations.
 */
public class JsonFormatter implements MessageFormatter {

    @Override
    public String format(String subject, String body, String sender) {
        return String.format(
            "{%n" +
            "  \"from\": \"%s\",%n" +
            "  \"subject\": \"%s\",%n" +
            "  \"body\": \"%s\"%n" +
            "}",
            escapeJson(sender), escapeJson(subject), escapeJson(body)
        );
    }

    @Override
    public String contentType() {
        return "application/json";
    }

    private String escapeJson(String raw) {
        if (raw == null) return "";
        return raw.replace("\\", "\\\\")
                  .replace("\"", "\\\"")
                  .replace("\n", "\\n")
                  .replace("\r", "\\r");
    }
}
```

```java
package com.course.patterns.bridge.notification;

/**
 * Abstraction — Notification.
 *
 * <p>The base class for all notification types. Each notification type
 * (Email, SMS, Push) knows its own channel-specific logic (e.g., how to
 * address a recipient, what transport to use), but delegates message
 * formatting entirely to the {@link MessageFormatter} implementor.
 *
 * <p>This allows any notification type to use any formatter freely:
 * an EmailNotification can send plain-text or HTML; an SMSNotification
 * always uses plain-text; a PushNotification always uses JSON.
 */
public abstract class Notification {

    /** The bridge to the MessageFormatter implementor hierarchy. */
    protected final MessageFormatter formatter;

    /**
     * Constructs a Notification with the given formatter.
     *
     * @param formatter the message formatter to use; must not be {@code null}
     */
    protected Notification(MessageFormatter formatter) {
        if (formatter == null) {
            throw new IllegalArgumentException("Formatter must not be null");
        }
        this.formatter = formatter;
    }

    /**
     * Sends a notification to the given recipient.
     *
     * @param recipient the target address (email, phone number, device token, etc.)
     * @param subject   the notification subject or title
     * @param body      the notification body text
     */
    public abstract void send(String recipient, String subject, String body);

    /**
     * Returns the channel type name, e.g., "Email", "SMS", "Push".
     *
     * @return channel name
     */
    public abstract String getChannelName();
}
```

```java
package com.course.patterns.bridge.notification;

/**
 * RefinedAbstraction — EmailNotification.
 *
 * <p>Sends a notification over the email channel using SMTP (simulated here).
 * The email channel supports both plain-text and HTML formatters, making it
 * the only abstraction that meaningfully uses all three formatters.
 */
public class EmailNotification extends Notification {

    private final String senderAddress;

    /**
     * Constructs an EmailNotification.
     *
     * @param formatter     the message formatter ({@link PlainTextFormatter}
     *                      or {@link HtmlFormatter} are the natural choices)
     * @param senderAddress the "From" email address
     */
    public EmailNotification(MessageFormatter formatter, String senderAddress) {
        super(formatter);
        this.senderAddress = senderAddress;
    }

    @Override
    public void send(String recipient, String subject, String body) {
        String formatted = formatter.format(subject, body, senderAddress);
        System.out.printf("=== Sending EMAIL to %s ===%n", recipient);
        System.out.printf("Content-Type: %s%n", formatter.contentType());
        System.out.println(formatted);
        // In production: smtpClient.send(recipient, formatted, formatter.contentType());
    }

    @Override
    public String getChannelName() {
        return "Email";
    }
}
```

```java
package com.course.patterns.bridge.notification;

/**
 * RefinedAbstraction — SMSNotification.
 *
 * <p>Sends a notification via SMS. SMS has a strict character limit; this
 * class enforces truncation. It always uses plain-text formatting because
 * SMS clients cannot render HTML or JSON.
 *
 * <p>Note: Even though SMS is fixed to plain-text in practice, the formatter
 * is injected rather than hard-coded, preserving the Bridge structure and
 * making the class unit-testable with a mock formatter.
 */
public class SMSNotification extends Notification {

    private static final int MAX_SMS_LENGTH = 160;
    private final String senderPhone;

    /**
     * Constructs an SMSNotification.
     *
     * @param formatter   the message formatter (typically {@link PlainTextFormatter})
     * @param senderPhone the sender's phone number or short code
     */
    public SMSNotification(MessageFormatter formatter, String senderPhone) {
        super(formatter);
        this.senderPhone = senderPhone;
    }

    @Override
    public void send(String recipient, String subject, String body) {
        String formatted = formatter.format(subject, body, senderPhone);
        if (formatted.length() > MAX_SMS_LENGTH) {
            formatted = formatted.substring(0, MAX_SMS_LENGTH - 3) + "...";
        }
        System.out.printf("=== Sending SMS to %s ===%n", recipient);
        System.out.println(formatted);
        // In production: twilioClient.messages().create(recipient, senderPhone, formatted);
    }

    @Override
    public String getChannelName() {
        return "SMS";
    }
}
```

```java
package com.course.patterns.bridge.notification;

/**
 * RefinedAbstraction — PushNotification.
 *
 * <p>Sends a push notification via FCM or APNs (Firebase Cloud Messaging /
 * Apple Push Notification service). Push notifications require a JSON payload,
 * so {@link JsonFormatter} is the natural implementation choice.
 *
 * <p>The {@code priority} field is a push-specific concern — it has no
 * analog in Email or SMS. This shows that RefinedAbstractions can carry
 * extra state not present in the base Abstraction class.
 */
public class PushNotification extends Notification {

    /** Push notification delivery priority. */
    public enum Priority { LOW, NORMAL, HIGH }

    private final String  appId;
    private final Priority priority;

    /**
     * Constructs a PushNotification.
     *
     * @param formatter the message formatter (typically {@link JsonFormatter})
     * @param appId     the application identifier registered with FCM/APNs
     * @param priority  the delivery priority for the push gateway
     */
    public PushNotification(MessageFormatter formatter, String appId, Priority priority) {
        super(formatter);
        this.appId    = appId;
        this.priority = priority;
    }

    @Override
    public void send(String recipient, String subject, String body) {
        String payload = formatter.format(subject, body, appId);
        System.out.printf("=== Sending PUSH to device token %s (priority=%s) ===%n",
                          recipient, priority);
        System.out.println(payload);
        // In production: fcmClient.send(recipient, payload, priority.name());
    }

    @Override
    public String getChannelName() {
        return "Push";
    }

    public Priority getPriority() {
        return priority;
    }
}
```

```java
package com.course.patterns.bridge.notification;

/**
 * Demonstrates the Notification / Formatter Bridge.
 *
 * <p>Three notification types (Email, SMS, Push) can each use any of the
 * three formatters (PlainText, HTML, JSON). That is 3 × 3 = 9 combinations,
 * but only 6 classes total (3 + 3). Any new notification type or formatter
 * requires exactly one new class.
 */
public class NotificationBridgeDemo {

    public static void main(String[] args) {
        String subject = "Your order #1234 has shipped";
        String body    = "Expected delivery: Tomorrow by 8pm. Track at: https://track.example.com/1234";

        // Email with HTML formatting
        Notification htmlEmail = new EmailNotification(
            new HtmlFormatter(), "noreply@shop.example.com");
        htmlEmail.send("customer@example.com", subject, body);

        // Email with plain-text formatting (for plain-text email clients)
        Notification textEmail = new EmailNotification(
            new PlainTextFormatter(), "noreply@shop.example.com");
        textEmail.send("customer@example.com", subject, body);

        // SMS with plain-text formatting
        Notification sms = new SMSNotification(
            new PlainTextFormatter(), "+18005551234");
        sms.send("+19175559876", subject, body);

        // Push notification with JSON formatting
        Notification push = new PushNotification(
            new JsonFormatter(), "com.example.shop",
            PushNotification.Priority.HIGH);
        push.send("fcm-device-token-abc123xyz", subject, body);
    }
}
```

---

## 10. Real-World Examples in the Java Ecosystem

### 10.1 JDBC — The Most Famous Bridge in Java

JDBC (Java Database Connectivity) is a textbook Bridge pattern implementation.

```
Abstraction hierarchy:            Implementor hierarchy:
────────────────────────          ────────────────────────
java.sql.Connection               java.sql.Driver
java.sql.Statement                ├── com.mysql.jdbc.Driver
java.sql.PreparedStatement        ├── org.postgresql.Driver
java.sql.ResultSet                ├── oracle.jdbc.OracleDriver
                                  └── com.microsoft.sqlserver.jdbc...
```

- **Abstraction**: `java.sql.Connection`, `java.sql.Statement`, `java.sql.PreparedStatement` — the high-level API that application code writes against.
- **Implementor**: `java.sql.Driver` — the interface that every database vendor implements.
- **Bridge**: When you call `DriverManager.getConnection(url, user, pass)`, the manager finds the registered `Driver` implementation and delegates connection creation to it.

The application code never imports `com.mysql.jdbc` directly. It only uses `java.sql.*`. Swapping from MySQL to PostgreSQL requires only a JAR swap and a connection URL change — not a single line of application code changes. This is the Bridge pattern at architectural scale.

### 10.2 SLF4J — Logging Facade

SLF4J (Simple Logging Facade for Java) is the Bridge pattern applied to logging frameworks.

```
Abstraction:            Implementor:
────────────            ────────────
org.slf4j.Logger        slf4j-log4j12-*.jar  →  Log4j 1.x
org.slf4j.LoggerFactory slf4j-log4j2-*.jar   →  Log4j 2.x
                        logback-classic-*.jar →  Logback
                        slf4j-jdk14-*.jar     →  java.util.logging
                        slf4j-simple-*.jar    →  Simple stdout logger
```

The application writes `Logger log = LoggerFactory.getLogger(MyClass.class)` and calls `log.info(...)`, `log.debug(...)`. The underlying logging framework is a runtime decision made through the classpath. The abstraction (SLF4J API) and the implementation (logging framework) vary completely independently. This allows a library author to use SLF4J without forcing a specific logging framework on library users.

### 10.3 AWT/Swing Peer System

Java AWT (Abstract Window Toolkit) uses the Bridge pattern to provide a cross-platform UI:

```
Abstraction (java.awt):      Implementor (sun.awt.XX):
────────────────────────     ─────────────────────────
java.awt.Component           sun.awt.windows.WComponentPeer   (Windows)
java.awt.Button              sun.awt.X11.XButtonPeer          (Linux/X11)
java.awt.TextField           sun.awt.mac.CTextFieldPeer       (macOS)
```

Every `java.awt.Button` has an associated "peer" — a native OS widget. When you call `button.setLabel("OK")`, AWT delegates to the platform-specific peer, which calls the Win32, X11, or Cocoa API. The AWT abstraction hierarchy and the platform-native peer hierarchy evolve independently. When Apple releases a new macOS version, only the Mac peer implementations change — the `Button`, `TextField`, `Frame` classes are unaffected.

### 10.4 java.sql.Driver — Design Upfront

`java.sql.Driver` is not retrofitted. It was **designed as the Bridge implementor interface from day one** (JDK 1.1, 1997). This is the critical distinction from the Adapter pattern: the Driver interface was created with the explicit intent of allowing multiple database implementations to plug in. No specific database driver class was being "adapted" — a clean implementor contract was specified, and vendors implemented it.

---

## 11. Bridge vs Adapter

This is the most important comparison in the Bridge pattern's surrounding design space. Getting this wrong in an interview is a red flag for interviewers.

### The Critical Distinction: Design Intent and Timing

| Dimension             | Bridge                              | Adapter                              |
|-----------------------|-------------------------------------|--------------------------------------|
| **Intent**            | Decouples two hierarchies upfront   | Makes an incompatible interface work |
| **Timing**            | Designed before implementation      | Applied after the fact               |
| **Problem solved**    | Prevent class explosion (M × N)     | Interface incompatibility            |
| **Interface designed for**| Future extensibility           | An existing class that can't change  |
| **Number of classes** | Reduces total class count           | Adds one wrapper class               |
| **Awareness**         | Abstraction knows about Implementor | Adaptee knows nothing about Adapter  |

### Bridge: Designed Upfront

```java
// Bridge: The Color interface was designed as the implementor from the start.
// Shape was designed to take a Color — there is no "legacy" Color class being wrapped.
public interface Color {        // Designed as the implementor interface
    String applyColor(String shape);
}
public abstract class Shape {
    protected final Color color;  // Composition is intentional from the design phase
    protected Shape(Color color) { this.color = color; }
}
```

### Adapter: Retrofitted

```java
// Adapter: LegacyPrinter exists independently. It was not designed for the
// modern Printer interface. The adapter wraps it to make it compatible.

// Pre-existing, unchangeable class (from a library or legacy system):
public class LegacyPrinter {
    public void oldPrint(String text, int copies) {
        for (int i = 0; i < copies; i++) System.out.println(text);
    }
}

// Modern interface that the rest of the system expects:
public interface Printer {
    void print(String document);
}

// Adapter: wraps LegacyPrinter to satisfy the Printer interface.
// This is NOT a Bridge — it's an Adapter because:
// 1. LegacyPrinter existed BEFORE the Printer interface was defined.
// 2. We are solving an incompatibility problem, not a class explosion problem.
// 3. We are wrapping a specific class, not designing a pluggable implementor hierarchy.
public class LegacyPrinterAdapter implements Printer {
    private final LegacyPrinter legacy;
    public LegacyPrinterAdapter(LegacyPrinter legacy) { this.legacy = legacy; }

    @Override
    public void print(String document) {
        legacy.oldPrint(document, 1);  // Translates the modern call to the legacy API
    }
}
```

### Structural Similarity

Both patterns use composition. A Bridge's abstraction holds a reference to an implementor; an Adapter holds a reference to an adaptee. Structurally, they can look identical. **The difference is entirely in design intent and the problem being solved.**

A useful heuristic: if you drew the two class hierarchies on a whiteboard *before* writing any code, you are designing a Bridge. If you are solving an existing incompatibility between two already-written classes, you are retrofitting an Adapter.

---

## 12. Bridge vs Strategy

Bridge and Strategy are the patterns most often confused with each other. They share the same structural implementation (an object holds a reference to an interface; the interface has multiple implementations) but differ significantly in intent, scope, and design context.

### Structural Similarity

```java
// Bridge: Shape holds a Color reference
public abstract class Shape {
    protected final Color color;  // color = implementor
}

// Strategy: Sorter holds an Algorithm reference
public class Sorter {
    private SortAlgorithm algorithm;  // algorithm = strategy
    public void setAlgorithm(SortAlgorithm a) { this.algorithm = a; }
}
```

Both have an object that holds a reference to an interface. If you look only at the Java code structure, they are identical.

### Intent Differences

| Dimension              | Bridge                                          | Strategy                                        |
|------------------------|--------------------------------------------------|--------------------------------------------------|
| **Scale of variation** | Two full class hierarchies                       | One dimension: the behavior of one operation     |
| **Abstraction**        | Has its own class hierarchy                      | Usually a single concrete class                  |
| **Implementor**        | Is a full hierarchy (multiple implementations)  | Is a family of interchangeable algorithms        |
| **Problem**            | Prevent M × N class explosion                   | Make an algorithm interchangeable at runtime     |
| **GoF Category**       | Structural                                       | Behavioral                                       |
| **Primary concern**    | How objects are composed/structured              | How objects communicate and behave               |

### When the Difference Matters

- **Bridge**: Use when you have two dimensions of variation and want to prevent class explosion. You are making a structural composition decision upfront.
- **Strategy**: Use when you have a single dimension of variable behavior — one algorithm that can be swapped. The context class is not itself part of a hierarchy.

In practice, a Strategy with a multi-level context class hierarchy begins to look like a Bridge. The distinction blurs at the edges, but the design intent remains different. A Bridge says "these two dimensions are genuinely independent and should vary separately." A Strategy says "this one behavior should be pluggable."

---

## 13. Bridge vs Abstract Factory

### Comparison

| Dimension          | Bridge                                    | Abstract Factory                              |
|--------------------|-------------------------------------------|-----------------------------------------------|
| **Primary purpose**| Decouple two varying hierarchies          | Create families of related objects            |
| **Pattern type**   | Structural                                | Creational                                    |
| **Focus**          | Object structure and composition          | Object creation                               |
| **Implementation** | Abstraction holds implementor by reference| Factory creates and returns implementor objects|

### How They Work Together

Abstract Factory is often used to **instantiate** a Bridge pattern's concrete implementors. The factory decides which `ConcreteImplementor` to create based on the environment (OS, configuration, profile); the Bridge then uses that implementor for its lifetime.

```java
// Abstract Factory creates the right Color implementor based on runtime config
public interface ColorFactory {
    Color createColor();
}

public class HighContrastColorFactory implements ColorFactory {
    @Override
    public Color createColor() {
        return new HighContrastColor();  // A ConcreteImplementor for accessibility mode
    }
}

// Bridge then uses the factory to get the right implementor
Shape shape = new Circle(5.0, colorFactory.createColor());
```

This is a natural pairing: Abstract Factory handles the creation concern; Bridge handles the structural composition concern.

---

## 14. Trade-offs

### Advantages

**1. Eliminates class explosion**
The most concrete and measurable benefit. A design with M abstractions and N implementations needs only M + N classes instead of M × N. For large M and N, this is a dramatic reduction in codebase complexity.

**2. Open/Closed Principle compliance**
Both hierarchies can be extended without modifying existing classes. A new `ConcreteImplementor` (e.g., `YellowColor`) requires no changes to any `Shape` class. A new `RefinedAbstraction` (e.g., `Triangle`) requires no changes to any `Color` class.

**3. Single Responsibility Principle compliance**
The abstraction is responsible for high-level logic; the implementor is responsible for low-level details. These are genuinely distinct responsibilities, and the Bridge keeps them in separate class hierarchies.

**4. Runtime implementation switching**
If the implementor is not declared `final`, it can be swapped at runtime. A `Device` can be reassigned after construction. This supports scenarios like hot-pluggable modules, A/B testing of implementations, and feature flags at the object level.

**5. Hiding implementation details**
The implementor hierarchy can be in a separate package or module, fully hidden from clients. Clients only see the abstraction. This is analogous to how JDBC hides all vendor-specific driver code.

### Disadvantages

**1. Increased design complexity**
The Bridge pattern introduces at minimum two interfaces or abstract classes and at least four concrete classes where a simpler design might use one or two classes. For simple problems, this overhead is not justified.

**2. Indirection overhead**
Every call from the abstraction to the implementor passes through a virtual dispatch. On a hot path (inner loop processing millions of items), this overhead can be measurable. Profile before applying the pattern on performance-critical code.

**3. Must be designed upfront**
Unlike Adapter (which is retrofitted), the Bridge pattern requires you to identify the two independent variation axes at design time. Retrofitting Bridge into an existing inheritance hierarchy is difficult and often requires significant refactoring.

**4. Not always intuitive**
The terminology ("abstraction," "implementor") conflicts with Java language terminology (`abstract`, `implements`). New team members often find the pattern confusing until they internalize the GoF definitions. Code documentation must be explicit.

**5. Overkill for two-implementation scenarios**
If you know there will be exactly two implementations and the abstraction will never extend, a simple strategy pattern or even a boolean flag is simpler and clearer.

---

## 15. Common Pitfalls and Gotchas

### Pitfall 1: Conflating "Abstraction" with Java's `abstract`

The most common misconception. A Bridge `Abstraction` can be a concrete class. It does not need the Java `abstract` keyword. The GoF use "abstraction" to mean "the high-level control layer," not the Java language construct. Do not let the terminology confuse the design.

### Pitfall 2: Designing Bridge When Adapter Is the Right Answer

If you are integrating a third-party library with an incompatible interface, use Adapter. If you are designing a new API that must support multiple underlying engines from the start, use Bridge. Applying Bridge when Adapter is appropriate adds unnecessary complexity.

### Pitfall 3: Making the Implementor Too Coarse

If the `Implementor` interface is too broad (exposes too many unrelated methods), concrete implementations will have large stubs returning `UnsupportedOperationException`. Split the implementor into narrower interfaces following the Interface Segregation Principle.

### Pitfall 4: Making the Implementor Too Fine-Grained

Conversely, if every single method call is a separate implementor interface, the bridge becomes a leaky abstraction. The abstraction should be able to express one complete operation through a small number of implementor calls.

### Pitfall 5: Forgetting That Both Sides Can Have Hierarchies

A common error is to use Bridge but only extend one side. The pattern's full power comes from extending both the abstraction hierarchy (new notification types) and the implementor hierarchy (new formatters) independently. If only one side ever changes, consider whether Strategy is a simpler fit.

### Pitfall 6: Circular Dependencies

If an `Implementor` method returns objects of the `Abstraction` type, or if the `Abstraction` has methods that take `Implementor`-hierarchy arguments, you introduce coupling that defeats the Bridge's purpose. Keep the dependency arrow one-directional: Abstraction → Implementor.

### Pitfall 7: Thread Safety of Shared Implementors

If multiple abstraction instances share a single implementor instance (e.g., two remote controls for the same TV), the implementor must be thread-safe if accessed from multiple threads. This is easy to overlook when the implementor looks like a simple value object.

---

## 16. Interview Discussion Points

### Key Points to Make

1. **Lead with the problem, not the pattern.** Explain the Cartesian product problem first. The pattern is the solution; interviewers want to see that you understand the problem it solves.

2. **Be precise about terminology.** "Abstraction" in Bridge does not mean Java `abstract`. State this explicitly to show depth.

3. **Contrast with Adapter decisively.** The designed-upfront vs retrofitted distinction is the single most important differentiator. Interviewers specifically probe this.

4. **Cite JDBC.** It is the most famous real-world Java Bridge and every Java developer is expected to know JDBC. Mentioning it signals practical knowledge beyond textbook patterns.

5. **Know the trade-offs.** A candidate who only lists advantages sounds junior. Knowing when NOT to use the pattern is senior-engineer thinking.

6. **Relate to SOLID.** Bridge enforces OCP (add implementations without modifying abstractions) and SRP (separate high-level and low-level concerns into distinct hierarchies).

---

## 17. Interview Questions with Model Answers

### Q1: What is the Bridge design pattern and what problem does it solve?

**Model Answer:**

The Bridge pattern is a structural design pattern that separates an abstraction from its implementation by placing them in two distinct class hierarchies connected by composition. The abstraction holds a reference to the implementor interface and delegates low-level work to it.

The fundamental problem it solves is the Cartesian product class explosion. When a class can vary along two independent dimensions — say, shapes can vary by geometry and by color — modeling both dimensions through inheritance produces M × N subclasses. With three shapes and four colors, that is twelve classes. Adding a color requires three new classes; adding a shape requires four new classes.

The Bridge converts this multiplicative relationship to an additive one: M abstraction classes plus N implementor classes, for M + N total. Adding a color now requires one new class. Adding a shape requires one new class. The two hierarchies are extensible independently.

The second problem it solves is binding an abstraction permanently to its implementation. If a `Shape` subclass directly contains color-specific rendering logic, you cannot swap the color at runtime or reuse the shape with a different color without creating a new class. The Bridge's composition approach allows the implementor to be chosen or swapped freely, including at runtime.

---

### Q2: What is the difference between Bridge and Adapter?

**Model Answer:**

This is the most important distinction in this space, and it is fundamentally about **design intent and timing**, not structure.

The Bridge pattern is designed upfront. When you recognize that a class can vary along two independent dimensions, you deliberately design a `Implementor` interface for one dimension and compose it into the `Abstraction` class hierarchy. Both the abstraction and the implementor interface are created as part of the same design session, specifically to allow independent extension of both hierarchies. No pre-existing classes are being wrapped.

The Adapter pattern is retrofitted. An Adapter wraps an existing, incompatible class to make it conform to an interface that a client expects. The adaptee existed before the adapter interface was defined. The adapter translates calls from the new interface to the old one. You are solving an incompatibility problem between two independently developed pieces of code.

Structurally, both patterns use composition — an object holds a reference to another object's interface. If you look only at the Java code, they can be identical. The distinction is in why and when the pattern was introduced.

A practical heuristic: if you drew both class hierarchies on a whiteboard before writing a single line of implementation code, you are designing a Bridge. If you are solving an existing incompatibility, you are writing an Adapter.

JDBC illustrates both concepts: the `java.sql.Driver` interface is a Bridge implementor interface designed upfront by the JDK team. Individual driver adapters (like `com.mysql.jdbc.Driver`) implement that interface. The original MySQL JDBC driver was written to implement `java.sql.Driver` — it was not adapted from a pre-existing interface. This makes the JDBC architecture a Bridge, not a collection of Adapters.

---

### Q3: How does the Bridge pattern relate to the SOLID principles?

**Model Answer:**

The Bridge pattern is one of the strongest examples of multiple SOLID principles applied simultaneously.

**Open/Closed Principle**: Both hierarchies in a Bridge are closed for modification but open for extension. To support a new color, you write a new `Color` implementation class and do not touch `Circle`, `Square`, or any other shape class. To support a new shape, you subclass `Shape` without touching any color class. The system is extensible in two dimensions without modifying existing code.

**Single Responsibility Principle**: The abstraction class is responsible for high-level coordination (what the operation means conceptually). The implementor class is responsible for low-level execution (how the operation is carried out on a specific platform or technology). Splitting these into separate class hierarchies gives each class a single reason to change: shape classes change when geometry logic changes; color classes change when rendering technology changes.

**Dependency Inversion Principle**: The abstraction depends on the `Implementor` interface (an abstraction), not on any `ConcreteImplementor` (a concrete class). This inverts the usual dependency from "high-level depends on low-level" to "high-level depends on abstraction; low-level implements abstraction."

**Interface Segregation Principle**: The `Implementor` interface should be as narrow as necessary. If different abstractions need different subsets of implementor operations, define narrower implementor interfaces for each rather than one fat interface.

**Liskov Substitution Principle**: All `ConcreteImplementor` classes must be substitutable for the `Implementor` interface without breaking the abstraction's invariants. If `RedColor.applyColor()` throws an exception in cases where `GreenColor.applyColor()` does not, the `Shape` class cannot rely on uniform behavior.

---

### Q4: Can you give a real-world Java ecosystem example of the Bridge pattern?

**Model Answer:**

The most famous and widely deployed Bridge pattern in Java is JDBC.

The `java.sql` package defines the abstraction hierarchy: `Connection`, `Statement`, `PreparedStatement`, `ResultSet`, `CallableStatement`. These are the high-level database operations that application code writes against.

The `java.sql.Driver` interface defines the implementor. Every database vendor provides a `Driver` implementation: MySQL provides `com.mysql.cj.jdbc.Driver`, PostgreSQL provides `org.postgresql.Driver`, Oracle provides `oracle.jdbc.OracleDriver`. These classes implement the low-level, vendor-specific communication protocols (MySQL wire protocol, PostgreSQL wire protocol, Oracle Net).

The bridge is the `DriverManager`. When application code calls `DriverManager.getConnection("jdbc:mysql://localhost/db", user, pass)`, the manager inspects the URL prefix, finds the registered MySQL `Driver` implementation, and delegates connection creation to it. The returned `Connection` object is a MySQL-specific implementation behind the standard `java.sql.Connection` interface.

Application code never imports `com.mysql.cj.jdbc.*`. If you switch from MySQL to PostgreSQL, you change the JAR on the classpath and change the connection URL from `jdbc:mysql://` to `jdbc:postgresql://`. Zero lines of application code change. This is the Bridge pattern delivering on its promise of independent variability: the abstraction hierarchy (`java.sql` API) and the implementation hierarchy (database drivers) evolve completely independently.

SLF4J is a second example with the same structure: `org.slf4j.Logger` is the abstraction; the binding JARs (logback-classic, log4j2, jul-to-slf4j) are the implementors. Library code uses `LoggerFactory.getLogger()` and the binding is resolved at deployment time.

---

### Q5: How do you decide between Bridge and Strategy patterns when they look structurally identical?

**Model Answer:**

This is one of the subtler questions in design pattern discourse, because at the Java source-code level, the two patterns are structurally indistinguishable: both involve an object holding a reference to an interface, and the interface having multiple implementations.

The distinction is in scope, intent, and the presence of an abstraction hierarchy.

**Bridge** is chosen when:
- You have two genuinely independent dimensions of variation that form their own class hierarchies.
- The "context" class (the abstraction) is not a single concrete class but the root of its own hierarchy (Circle, Square, Triangle are all abstractions; RedColor, GreenColor, BlueColor are all implementors).
- The goal is to prevent M × N class explosion.
- The problem is structural: how objects are composed.

**Strategy** is chosen when:
- You have one dimension of variation — one operation or algorithm that must be interchangeable.
- The context is usually a single concrete class (a `Sorter` that has a pluggable `SortAlgorithm`).
- The goal is to make one behavior swappable at runtime.
- The problem is behavioral: how an object communicates and what it does.

A useful diagnostic question: "Can I draw two separate, extensible class hierarchies on a whiteboard, one for the abstraction side and one for the implementation side, where each hierarchy has more than one class and both are expected to grow?" If yes, Bridge. If you have only one class on the "context" side and only a family of algorithm variants on the other side, it is Strategy.

In practice, a Strategy that grows to have a multi-level context class hierarchy has evolved into a Bridge. The intent distinction becomes most important during the design phase, when you decide which pattern to apply before writing code.

---

### Q6: What are the trade-offs of using the Bridge pattern?

**Model Answer:**

The Bridge pattern's benefits are real but come with genuine costs that must be weighed honestly.

**On the benefit side**: The pattern eliminates class explosion, which is a concrete, measurable reduction in codebase size and complexity. Adding a new dimension variant (new color, new formatter) requires exactly one new class and zero modifications to existing code, which is a textbook OCP win. The implementor hierarchy can be independently developed, tested, and packaged — even by different teams or third-party vendors, as the JDBC ecosystem demonstrates.

**On the cost side**:

*Design complexity*: The Bridge pattern requires identifying two independent variation axes at design time. This requires anticipating future extension directions, which is not always possible or desirable (YAGNI). If the system never needs more than one implementor, the extra indirection is pure overhead.

*Cognitive overhead*: The GoF terminology conflicts with Java language keywords. A new team member reads "Abstraction" and thinks `abstract` class; they read "Implementor" and think `implements`. Extensive documentation is needed to prevent misunderstanding.

*Virtual dispatch cost*: Every call from the abstraction to the implementor is a virtual method call. On modern JVMs, the JIT typically inlines monomorphic call sites, reducing this to zero overhead. But polymorphic call sites (multiple implementations used simultaneously) incur a real dispatch cost. On a hot path processing millions of records, this can be significant.

*Upfront design requirement*: Unlike Adapter (which can be retrofitted), Bridge requires upfront design. Retrofitting Bridge into a flat inheritance hierarchy is a significant refactoring effort, often requiring breaking changes to existing APIs.

My rule of thumb: use Bridge when you can identify two genuinely independent variation axes at design time, and when you expect both axes to grow over the lifetime of the project. Do not apply it speculatively.

---

### Q7: How would you implement the Bridge pattern in a notification system where notification channels (Email, SMS, Push) and message formats (plain-text, HTML, JSON) must vary independently?

**Model Answer:**

This is a clean two-dimensional problem:
- **Axis 1 (abstraction)**: Notification channel — Email, SMS, Push. Each channel has channel-specific logic: email has a sender address and SMTP transport; SMS has a phone number and character limit; Push has a device token and priority.
- **Axis 2 (implementor)**: Message format — PlainText, HTML, JSON. Each formatter knows how to format a subject, body, and sender into a channel-agnostic string.

Without Bridge, combining 3 channels with 3 formats yields 9 classes: `HtmlEmailNotification`, `JsonEmailNotification`, `PlainEmailNotification`, `HtmlSMSNotification` (doesn't make sense semantically, but the class must exist to complete the hierarchy), and so on.

With Bridge, the design is:
- `MessageFormatter` interface (implementor) with `format(subject, body, sender): String` and `contentType(): String`.
- `PlainTextFormatter`, `HtmlFormatter`, `JsonFormatter` — three ConcreteImplementors.
- `Notification` abstract class (abstraction) holding a `final MessageFormatter formatter`.
- `EmailNotification`, `SMSNotification`, `PushNotification` — three RefinedAbstractions.

Total: 6 classes instead of 9. Adding a fourth formatter (Markdown, for Slack) requires one new class. Adding a fourth channel (InApp) requires one new class. The multiplicative growth is broken.

A key design consideration: `SMSNotification` in practice always uses plain-text (SMS cannot render HTML), so pairing it with `HtmlFormatter` is semantically wrong even if structurally valid. This is where the Bridge's flexibility must be balanced with business logic constraints. You can enforce the valid pairings through a factory method or builder rather than leaving them to caller discretion, while still preserving the Bridge structure for testability and future extension.

---

### Q8: What happens if the two hierarchies in a Bridge pattern are not truly independent?

**Model Answer:**

This is an insightful edge-case question that tests whether the candidate understands the pattern's preconditions, not just its mechanics.

If the two hierarchies are not truly independent, the Bridge pattern starts to break down in two ways.

**First**, the `Implementor` interface becomes too coupled to the specific needs of one `Abstraction` subclass. For example, if `AdvancedRemote` needs a method on `Device` that `BasicRemote` never uses — say `getSubtitleTrack()` — and if that method only makes sense for `TV` but not for `Radio`, the `Device` interface now has a method that `Radio` must stub out. This violates ISP and indicates that the dimensions are not as independent as originally assumed. The fix is either to use interface segregation (a separate `VideoDevice` extends `Device` with the extra method) or to reconsider whether Bridge is the right pattern.

**Second**, if an `Abstraction` subclass needs to cast the `Implementor` reference to a specific `ConcreteImplementor` type to access methods not on the interface, the Bridge has failed. For example, if `AdvancedRemote` does `((TV) device).get4KResolution()`, the coupling between abstraction and implementor is back, and the bridge provides no decoupling benefit.

The diagnostic: if every `RefinedAbstraction` works correctly with every `ConcreteImplementor` through the `Implementor` interface alone, the dimensions are truly independent and Bridge is appropriate. If certain combinations are semantically invalid or require downcasting, consider whether the dimensions need to be split differently, or whether a different structural pattern (Decorator, Composite) better models the relationship.

---

*End of Bridge Design Pattern — Chapter 06*
