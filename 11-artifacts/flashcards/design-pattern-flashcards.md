# Design Patterns — Flashcard Set

> 54 cards — 3 per pattern across 18 patterns (GoF Creational, Structural, Behavioural). Work one pattern at a time: Intent → When to Use → Structure/Contrast.

---

## Singleton

---
**Q1-A (Intent):** What is the intent of the Singleton pattern?

**A:** Singleton ensures that a class has exactly one instance and provides a global access point to that instance. The class itself controls creation, typically through a private constructor and a static factory method. It is used when exactly one object is needed to coordinate actions across the system — a shared resource like a configuration registry, a connection pool, or a logger.

---

---
**Q1-B (When to Use):** When should you use the Singleton pattern? Give 2–3 concrete scenarios.

**A:**
- A **configuration manager** that reads from a file or environment once and serves values throughout the app's lifetime.
- A **thread pool** or **connection pool** where multiple copies would exhaust resources or create inconsistency.
- A **logger** where all parts of the application must write to the same output stream.

Avoid Singleton when it is used merely for convenience to avoid passing dependencies — that is a sign the dependency should be injected instead.

---

---
**Q1-C (Structure):** What are the key implementation concerns for a thread-safe Singleton in Java?

**A:** Three common approaches:
- **Eager initialisation:** `private static final Singleton INSTANCE = new Singleton();` — thread-safe by classloader guarantee, but initialises even if never used.
- **Double-checked locking:** `volatile` field + synchronised block only on first access — lazy and thread-safe, but verbose.
- **Initialization-on-demand holder:** a private static inner class holds the instance; the JVM guarantees classloading is thread-safe. This is the preferred idiom — lazy, thread-safe, and concise.

---

---

## Factory Method

---
**Q2-A (Intent):** What is the intent of the Factory Method pattern?

**A:** Factory Method defines an interface for creating an object, but lets subclasses decide which class to instantiate. The creator class defers object creation to subclasses by declaring an abstract factory method that returns the product type. This makes the code open for extension (add a new product by adding a new creator subclass) while keeping the client closed to modification.

---

---
**Q2-B (When to Use):** When should you use the Factory Method pattern? Give 2–3 concrete scenarios.

**A:**
- A **UI framework** where the base `Dialog` class handles layout but subclasses (`WindowsDialog`, `WebDialog`) create platform-specific `Button` instances.
- A **logging library** where the base logger defers creation of the concrete log appender to subclasses.
- A **document editor** where the base `Application` class opens documents but subclasses decide what kind of `Document` object to create.

Use it when the exact type of product to create is not known at the creator's compile time.

---

---
**Q2-C (Contrast):** How does Factory Method differ from Simple Factory (Static Factory)?

**A:** A **Simple Factory** (not a GoF pattern) is a single class with a static method that creates objects based on a parameter. It is easy to understand but violates OCP — adding a new type means modifying the factory. A **Factory Method** uses inheritance: the factory method is defined in an abstract base class and overridden in subclasses. Adding a new product requires a new subclass, not modifying existing code. Factory Method is more extensible; Simple Factory is simpler for small, stable sets of types.

---

---

## Abstract Factory

---
**Q3-A (Intent):** What is the intent of the Abstract Factory pattern?

**A:** Abstract Factory provides an interface for creating *families* of related or dependent objects without specifying their concrete classes. Rather than a single factory method, it groups multiple factory methods into a cohesive interface, guaranteeing that the created objects are compatible with each other. The client works only through the abstract factory interface and never instantiates concrete products directly.

---

---
**Q3-B (When to Use):** When should you use the Abstract Factory pattern? Give 2–3 concrete scenarios.

**A:**
- A **cross-platform UI toolkit** where a `GUIFactory` interface produces a family of related widgets — `Button`, `Checkbox`, `TextField` — with `WindowsFactory` and `MacFactory` implementing the full family consistently.
- A **database access layer** that creates families of related objects (`Connection`, `Command`, `Reader`) for different databases (MySQL, PostgreSQL) ensuring all objects come from the same vendor.
- A **theme engine** where selecting a "Dark" or "Light" theme produces a matching set of colors, fonts, and icons.

---

---
**Q3-C (Contrast):** How does Abstract Factory differ from Factory Method?

**A:** **Factory Method** uses a single method overridden by subclasses to produce one product. The variation is in *inheritance* — subclasses decide the type. **Abstract Factory** uses an interface with *multiple factory methods* that together produce a family of related products. The variation is in *composition* — clients receive a concrete factory object and call its methods. As a rule: if you need to produce one type of product, start with Factory Method; if you need to guarantee consistency across multiple related products, use Abstract Factory.

---

---

## Builder

---
**Q4-A (Intent):** What is the intent of the Builder pattern?

**A:** Builder separates the construction of a complex object from its representation, allowing the same construction process to produce different representations. A `Builder` interface declares steps for building parts of the product; a `Director` (optional) defines the sequence of steps; a `ConcreteBuilder` implements the steps and assembles the product. Clients call steps in sequence and retrieve the finished product from the builder.

---

---
**Q4-B (When to Use):** When should you use the Builder pattern? Give 2–3 concrete scenarios.

**A:**
- Constructing a **complex configuration object** (e.g., an HTTP request with headers, body, timeout, auth) where many fields are optional and a telescoping-constructor anti-pattern would result in an unusable API.
- Building **different representations of the same structure** — e.g., constructing a `Meal` as either fast-food or fine-dining using the same construction steps.
- Creating **immutable objects** with many fields: the Builder accumulates state, then `build()` calls the private constructor, ensuring the object is never in a partially-constructed state.

---

---
**Q4-C (Contrast):** How does Builder differ from Abstract Factory?

**A:** **Abstract Factory** creates families of related objects in a *single step* — you call a factory method and immediately receive a complete product. **Builder** constructs a *single complex object* step by step — you call multiple methods to configure parts before retrieving the finished product. Use Abstract Factory when you need to produce compatible families of objects; use Builder when you need fine-grained control over the construction sequence of one complex object with many optional or combinatorial properties.

---

---

## Prototype

---
**Q5-A (Intent):** What is the intent of the Prototype pattern?

**A:** Prototype specifies the kinds of objects to create using a prototypical instance and creates new objects by cloning that prototype. Instead of instantiating a class from scratch, you copy an existing object. This decouples client code from the concrete classes of objects it needs and enables creating new objects that are pre-configured with specific state.

---

---
**Q5-B (When to Use):** When should you use the Prototype pattern? Give 2–3 concrete scenarios.

**A:**
- A **game engine** where many enemies share the same base configuration; instantiating from scratch each time is expensive, so the engine clones a pre-configured prototype.
- A **document editor** that needs to copy-paste a complex graphical object (with nested components, styles, and history) without knowing its concrete type.
- A **configuration system** where you take a working configuration template and clone it to produce slightly modified variants, avoiding repeated initialisation from defaults.

---

---
**Q5-C (Structure):** What is the difference between a shallow clone and a deep clone, and which does Prototype require?

**A:** A **shallow clone** copies the object's fields as-is — primitive values are copied, but object references point to the same shared instances. A **deep clone** recursively copies all referenced objects, producing a fully independent copy. Prototype requires **deep cloning** when the copied object contains mutable references, otherwise mutating the clone's nested objects would also mutate the original. In Java, `Object.clone()` performs a shallow copy by default; deep cloning must be implemented manually or via serialisation.

---

---

## Adapter

---
**Q6-A (Intent):** What is the intent of the Adapter pattern?

**A:** Adapter converts the interface of a class into another interface that clients expect. It lets two incompatible interfaces work together without modifying either. The Adapter wraps the adaptee and translates calls made through the target interface into calls the adaptee understands. It is the structural equivalent of a real-world power plug adapter.

---

---
**Q6-B (When to Use):** When should you use the Adapter pattern? Give 2–3 concrete scenarios.

**A:**
- Integrating a **third-party library** whose interface does not match your system's expected interface — wrap the library in an Adapter without modifying either.
- Making a **legacy class** work with new code that expects a different interface, without refactoring the legacy class.
- Allowing **multiple external payment providers** (Stripe, PayPal) to be called through a single uniform `PaymentGateway` interface by wrapping each in its own Adapter.

---

---
**Q6-C (Contrast):** How does Adapter differ from Facade?

**A:** **Adapter** makes two *incompatible* interfaces compatible — it is about translation, and the target interface is typically pre-defined by the client. **Facade** creates a new *simplified* interface over a complex subsystem — it is about simplification, and the interface is designed fresh. An Adapter does not simplify; it translates one-to-one. A Facade may internally call many subsystem classes. A single class can serve as both (a simplified adapter over a complex legacy API), but the intents are distinct.

---

---

## Bridge

---
**Q7-A (Intent):** What is the intent of the Bridge pattern?

**A:** Bridge decouples an abstraction from its implementation so that the two can vary independently. Rather than creating a large inheritance hierarchy where each combination of abstraction and implementation is its own class, Bridge separates them into two parallel hierarchies connected by composition (the abstraction holds a reference to an implementation object). This avoids a combinatorial explosion of subclasses.

---

---
**Q7-B (When to Use):** When should you use the Bridge pattern? Give 2–3 concrete scenarios.

**A:**
- A **shape drawing library** where `Shape` (abstraction) can be `Circle` or `Square`, and the `Renderer` (implementation) can be `VectorRenderer` or `RasterRenderer` — 2×2 = 4 combinations handled by 4 classes instead of 4 subclasses.
- **Cross-platform device remotes** where the remote control abstraction is independent of the device type (TV, Radio) it controls.
- A **messaging system** where the `Message` abstraction (short/long) is separated from the `MessageSender` implementation (email/SMS), so each evolves independently.

---

---
**Q7-C (Contrast):** How does Bridge differ from Adapter?

**A:** **Adapter** is typically applied to *existing, incompatible* classes to make them work together — it is a retrospective fix. **Bridge** is typically designed *upfront* to prevent a class hierarchy from becoming unwieldy — it is a proactive structure. Adapter focuses on interface translation; Bridge focuses on separating two dimensions of variation into two independent hierarchies. Bridge uses composition from the start; Adapter wraps an existing concrete class.

---

---

## Composite

---
**Q8-A (Intent):** What is the intent of the Composite pattern?

**A:** Composite composes objects into tree structures to represent part-whole hierarchies. It lets clients treat individual objects (leaves) and compositions of objects (composites) uniformly through a common interface. This means client code does not need to differentiate between a single element and a group of elements — it calls the same method on both.

---

---
**Q8-B (When to Use):** When should you use the Composite pattern? Give 2–3 concrete scenarios.

**A:**
- A **file system** where `File` and `Directory` implement the same `FileSystemNode` interface; iterating, searching, or computing sizes works identically on both.
- A **UI component tree** where `Button`, `TextInput`, and `Panel` all implement `Component`; a `Panel` can contain other components (including other Panels) and renders the full tree.
- A **bill of materials** in manufacturing where a product is made of sub-assemblies, which are made of parts — `getPrice()` on any node recursively sums the whole tree.

---

---
**Q8-C (Structure):** What are the key participants in the Composite pattern?

**A:**
- **Component** — the common interface (or abstract class) declaring the operation(s) all nodes support (e.g., `render()`, `getPrice()`).
- **Leaf** — a primitive element with no children. Implements Component directly.
- **Composite** — a container element that holds a list of Component children. Implements Component by delegating to each child.
- **Client** — works through the Component interface only, unaware of whether it holds a Leaf or Composite.

The key design decision: whether the Component interface includes child-management methods (`add`, `remove`) — making the interface less clean but more transparent — or whether only Composite exposes them.

---

---

## Decorator

---
**Q9-A (Intent):** What is the intent of the Decorator pattern?

**A:** Decorator attaches additional responsibilities to an object dynamically at runtime, as an alternative to subclassing. A Decorator wraps the component it decorates, implements the same interface, and adds behaviour before or after delegating to the wrapped object. Multiple decorators can be layered (stacked) in any combination, producing a flexible and composable extension mechanism.

---

---
**Q9-B (When to Use):** When should you use the Decorator pattern? Give 2–3 concrete scenarios.

**A:**
- **Java I/O streams:** `BufferedInputStream` decorates `FileInputStream` to add buffering; `GZIPInputStream` adds compression — each wrapper adds one capability without altering the others.
- Adding cross-cutting concerns like **logging, caching, or authentication** to service methods without subclassing each service class.
- A **coffee ordering system** where a base `Coffee` is decorated with `Milk`, `Sugar`, `Whip` — each decorator adds to cost and description, with any combination possible.

---

---
**Q9-C (Contrast):** How does Decorator differ from Inheritance as a way to extend behaviour?

**A:** **Inheritance** extends behaviour at compile time — the set of behaviours is fixed by the class hierarchy and every instance of the subclass has all extensions. **Decorator** extends at runtime — behaviours are added by wrapping objects dynamically, and different instances can have different combinations. Inheritance leads to a class explosion for many combinations (e.g., `MilkCoffee`, `SugarCoffee`, `MilkSugarCoffee`); Decorator scales with O(N) classes instead of O(2^N). Decorator favours composition over inheritance.

---

---

## Facade

---
**Q10-A (Intent):** What is the intent of the Facade pattern?

**A:** Facade provides a simplified interface to a complex subsystem. It does not add new capability; it hides the complexity of the subsystem behind a clean, unified entry point. Clients interact with the Facade instead of directly with the many subsystem classes. The subsystem classes remain accessible directly for clients that need full control, but most clients only need the simplified interface.

---

---
**Q10-B (When to Use):** When should you use the Facade pattern? Give 2–3 concrete scenarios.

**A:**
- A **home theatre system** where starting a movie requires turning on the projector, amplifier, DVD player, and dimming the lights — a `HomeTheatreFacade.watchMovie()` method hides all those steps.
- A **compiler** that exposes a single `compile(file)` method, hiding the scanner, parser, semantic analyser, and code generator used internally.
- An **order processing API** that internally calls inventory, payment, shipping, and notification services — clients call `OrderFacade.placeOrder()` rather than coordinating each service.

---

---
**Q10-C (Structure):** What are the key participants in the Facade pattern?

**A:**
- **Facade** — the entry point. Knows which subsystem classes handle a request and delegates client calls to the appropriate subsystem objects.
- **Subsystem classes** — implement actual functionality. They have no knowledge of the Facade; they receive calls from it just like from any other client.
- **Client** — calls the Facade instead of directly calling subsystem objects.

The Facade does not encapsulate subsystems (they remain accessible); it just makes the common use case simpler. A system can have multiple Facades for different "views" of the same subsystem.

---

---

## Flyweight

---
**Q11-A (Intent):** What is the intent of the Flyweight pattern?

**A:** Flyweight uses sharing to efficiently support a large number of fine-grained objects. It separates an object's state into **intrinsic state** (shared, context-independent — stored in the flyweight) and **extrinsic state** (unique per context — passed in by the client at call time). By sharing intrinsic state across many objects, Flyweight dramatically reduces memory consumption when large numbers of similar objects are needed.

---

---
**Q11-B (When to Use):** When should you use the Flyweight pattern? Give 2–3 concrete scenarios.

**A:**
- A **text editor** rendering millions of character objects: the character's font and glyph shape are intrinsic (shared); the position on the page is extrinsic (passed per render call).
- A **particle system** in a game with thousands of similar particles: the sprite and physics properties are intrinsic; the position and velocity are extrinsic.
- A **forest in a game world** with thousands of trees: the tree's mesh and texture are intrinsic (shared by tree type); the position, scale, and colour tint are extrinsic.

---

---
**Q11-C (Structure):** What are intrinsic and extrinsic state in Flyweight, and who is responsible for each?

**A:** **Intrinsic state** is the part of an object's data that is *independent of context* and can be shared across many instances (e.g., a character's glyph bitmap). It is stored inside the Flyweight object. **Extrinsic state** is *context-dependent* and varies between uses (e.g., a character's position on screen). It is never stored in the Flyweight — the **client** computes and passes it on each operation. The `FlyweightFactory` maintains a cache (usually a map) of existing Flyweights keyed by intrinsic state, returning a shared instance rather than creating a new one.

---

---

## Proxy

---
**Q12-A (Intent):** What is the intent of the Proxy pattern?

**A:** Proxy provides a surrogate or placeholder for another object to control access to it. The Proxy implements the same interface as the real subject and intercepts calls before (or after) forwarding them. It adds a layer of indirection that can be used for access control, lazy initialisation, logging, caching, remote communication, or reference counting — without the client knowing whether it is talking to the real object or a proxy.

---

---
**Q12-B (When to Use):** When should you use the Proxy pattern? Give 2–3 concrete scenarios.

**A:**
- **Virtual Proxy (lazy loading):** a `HeavyDocumentProxy` loads the actual `HeavyDocument` only when the client first accesses its content, saving resources on startup.
- **Protection Proxy (access control):** a `SecureFileProxy` checks the caller's permissions before delegating file operations to the real `File` object.
- **Remote Proxy (network transparency):** a stub in a remote procedure call (RPC) system makes a remote service appear as a local object, handling serialisation and network communication transparently.

---

---
**Q12-C (Contrast):** How does Proxy differ from Decorator?

**A:** Both wrap an object and implement the same interface, but their *intent* differs. **Proxy** controls *access* to the real subject — it may delay creation, restrict callers, add logging, or hide remote location. The client often does not know there is a proxy. **Decorator** adds *new behaviour* (stacked responsibilities) to an object at runtime — it is always explicit and composable. A protection proxy restricts; a decorator enhances. A caching proxy may look like a decorator (adds caching) but its purpose is transparent access optimisation, not feature extension.

---

---

## Chain of Responsibility

---
**Q13-A (Intent):** What is the intent of the Chain of Responsibility pattern?

**A:** Chain of Responsibility passes a request along a chain of handlers. Each handler decides whether to process the request or pass it to the next handler in the chain. This decouples the sender of a request from its receivers, allowing more than one object to handle the request, and letting you compose processing pipelines at runtime without hard-coding which handler handles which request.

---

---
**Q13-B (When to Use):** When should you use the Chain of Responsibility pattern? Give 2–3 concrete scenarios.

**A:**
- **HTTP middleware/filter pipelines:** authentication, rate-limiting, logging, and routing are separate handlers chained together; each processes (or short-circuits) the request in order.
- **Event handling in UI frameworks:** a mouse click event propagates from the deepest child widget upward through parent containers until one handles it.
- **Log level routing:** a logger chain passes a log entry through `DEBUG → INFO → WARN → ERROR` handlers; each handler processes entries at its level and above.

---

---
**Q13-C (Structure):** What are the key participants in the Chain of Responsibility pattern?

**A:**
- **Handler** — defines the interface for handling requests and (optionally) holds a reference to the next handler in the chain.
- **ConcreteHandler** — handles requests it is responsible for; either processes the request or forwards it to the next handler.
- **Client** — initiates the request to the first handler in the chain.

Key design choices: (1) whether each handler passes the request explicitly or the chain framework does it automatically; (2) whether processing stops at the first handler that can handle it or continues through all handlers (variation used in event pipelines).

---

---

## Command

---
**Q14-A (Intent):** What is the intent of the Command pattern?

**A:** Command encapsulates a request as an object, thereby allowing you to parameterise clients with different requests, queue or log requests, and support undoable operations. The request is turned into a standalone object containing all information about the request — the receiver, the method to call, and the parameters. This decouples the object that invokes an operation from the one that knows how to perform it.

---

---
**Q14-B (When to Use):** When should you use the Command pattern? Give 2–3 concrete scenarios.

**A:**
- **Undo/redo stacks** in editors: each user action is a Command with `execute()` and `undo()` methods; undo walks the stack backward calling `undo()`.
- **Job/task queues:** Commands are serialised and placed on a queue; workers dequeue and execute them, potentially on different threads or machines.
- **GUI button/menu actions:** a `Button` receives a `Command` object; clicking it calls `command.execute()` — the button does not know what action it triggers.

---

---
**Q14-C (Contrast):** How does Command differ from Strategy?

**A:** Both encapsulate behaviour in an object, but their intent differs. **Command** encapsulates a *request as an object* — it represents a specific action with a specific receiver. It is designed for deferred execution, queuing, logging, and undo. **Strategy** encapsulates an *interchangeable algorithm* — it represents a family of algorithms and lets clients choose among them at runtime. A Command is typically called once and may be stored; a Strategy is selected and repeatedly applied. Commands are about *what to do*; Strategies are about *how to do it*.

---

---

## Observer

---
**Q15-A (Intent):** What is the intent of the Observer pattern?

**A:** Observer defines a one-to-many dependency between objects: when one object (the subject/publisher) changes state, all its dependents (observers/subscribers) are notified and updated automatically. This allows loose coupling between the subject and its observers — the subject knows only that it has a list of observers implementing a common interface; it does not know their concrete types.

---

---
**Q15-B (When to Use):** When should you use the Observer pattern? Give 2–3 concrete scenarios.

**A:**
- **Event systems in UI frameworks:** a button (subject) notifies all registered click listeners (observers) when clicked — the button does not know what the listeners do.
- **Stock ticker:** a `StockMarket` subject notifies registered `StockDisplay`, `AlertSystem`, and `TradingBot` observers whenever a price changes.
- **Domain events in DDD:** an `Order` entity publishes an `OrderPlaced` event; inventory, notification, and billing observers react independently without the order knowing about them.

---

---
**Q15-C (Structure):** What are the key participants in the Observer pattern, and what is the risk of naive implementation?

**A:**
- **Subject (Publisher)** — maintains a list of observers; provides `subscribe()`, `unsubscribe()`, and `notify()` methods.
- **Observer (Subscriber)** — defines the `update()` interface called by the subject.
- **ConcreteSubject** — stores state; calls `notify()` when state changes.
- **ConcreteObserver** — implements `update()` to react to subject changes.

**Risk:** Memory leaks from observers that are never unregistered (the "lapsed listener" problem). Also, naively calling all observers synchronously from `notify()` can cause ordering issues, infinite loops if an observer modifies the subject, or performance bottlenecks if observers are slow. Use weak references or an explicit `unsubscribe()` contract to mitigate.

---

---

## State

---
**Q16-A (Intent):** What is the intent of the State pattern?

**A:** State allows an object to alter its behaviour when its internal state changes. The object appears to change its class. Rather than using large conditionals to manage state-dependent behaviour, State externalises each state into its own class. The context object delegates state-specific behaviour to the current state object, which may also trigger state transitions.

---

---
**Q16-B (When to Use):** When should you use the State pattern? Give 2–3 concrete scenarios.

**A:**
- A **vending machine** with states `Idle`, `HasMoney`, `Dispensing`, `OutOfStock` — each state handles button presses differently, and transitions between states are encapsulated in state classes.
- A **TCP connection** with states `Closed`, `Listening`, `Established` — `open()`, `close()`, and `acknowledge()` behave differently depending on the current state.
- An **order workflow** with states `Draft`, `Submitted`, `Approved`, `Shipped` — `cancel()` and `approve()` are valid only in certain states, and the logic for each state lives in its own class.

---

---
**Q16-C (Contrast):** How does State differ from Strategy?

**A:** Both delegate behaviour to a separate class through a common interface, but the *intent* and *lifecycle* differ. **Strategy** lets the *client* choose an algorithm; the strategy is typically set once and does not change unless the client explicitly swaps it. **State** lets the *object itself* transition between states as part of its normal lifecycle — the current state may decide the next state. State objects are often aware of each other (to trigger transitions); Strategy objects are independent alternatives unaware of each other.

---

---

## Strategy

---
**Q17-A (Intent):** What is the intent of the Strategy pattern?

**A:** Strategy defines a family of algorithms, encapsulates each one in a separate class, and makes them interchangeable. The client depends on an abstract strategy interface and is given a concrete implementation at runtime or configuration time. This lets the algorithm vary independently from the clients that use it, and eliminates conditional logic for choosing between algorithms.

---

---
**Q17-B (When to Use):** When should you use the Strategy pattern? Give 2–3 concrete scenarios.

**A:**
- A **sorting library** that supports multiple sorting algorithms (`QuickSort`, `MergeSort`, `HeapSort`) — the client passes the preferred strategy; switching algorithms requires no change to client code.
- **Payment processing** where `CreditCardStrategy`, `PayPalStrategy`, and `CryptoStrategy` each implement `PaymentStrategy.pay()` — the checkout service works with any strategy injected at runtime.
- **Route planning** in a map application with strategies for `WalkingRoute`, `CyclingRoute`, `DrivingRoute` — users pick the strategy; the `Navigator` class uses it without knowing the specifics.

---

---
**Q17-C (Structure):** What are the key participants in the Strategy pattern?

**A:**
- **Strategy** — the abstract interface (or functional interface in Java 8+) declaring the algorithm method (e.g., `sort(data)`, `pay(amount)`).
- **ConcreteStrategy** — one per algorithm variant; implements the Strategy interface with its specific algorithm.
- **Context** — holds a reference to a Strategy object; delegates the algorithm call to the strategy rather than implementing it directly. The Context may allow the strategy to be changed at runtime via a setter.
- **Client** — creates the concrete strategy and passes it to the context (or configures it via DI).

---

---

## Template Method

---
**Q18-A (Intent):** What is the intent of the Template Method pattern?

**A:** Template Method defines the skeleton of an algorithm in a base class, deferring some steps to subclasses. The base class implements the invariant parts of the algorithm and declares abstract (or overridable) "hook" methods for the parts that vary. Subclasses override the hooks to implement their specific variation without changing the overall algorithm structure. This is a classic example of the Hollywood Principle: "Don't call us, we'll call you."

---

---
**Q18-B (When to Use):** When should you use the Template Method pattern? Give 2–3 concrete scenarios.

**A:**
- A **data processing pipeline** with steps `readData()`, `processData()`, `writeOutput()` — the base class controls the sequence; subclasses implement `processData()` for CSV vs. JSON sources.
- **JUnit test lifecycle:** `setUp()`, `test()`, `tearDown()` — the framework calls these in a fixed order; test authors override `setUp` and `tearDown` without controlling when they are called.
- A **game AI framework** where `takeTurn()` calls `collectResources()`, `buildStructures()`, `attack()` in sequence — each AI faction subclass implements its own strategy for each step.

---

---
**Q18-C (Contrast):** How does Template Method differ from Strategy?

**A:** Both address algorithm variation, but through different mechanisms. **Template Method** uses *inheritance* — the skeleton is in the base class; subclasses fill in the blanks by overriding methods. The variation is baked in at compile time. **Strategy** uses *composition* — the algorithm is encapsulated in a separate object; the context receives it at runtime. Template Method is simpler but less flexible (you cannot change the algorithm at runtime without changing the object's class). Strategy is more flexible and testable since strategies are independent objects that can be mocked or swapped.

---

---

## Study Tips

1. **Use the Intent card as an anchor.** Before testing yourself on When to Use and Structure, nail the one-sentence intent. If you cannot state the pattern's purpose in one sentence, the other cards will not stick. Return to the Intent card until it flows naturally.

2. **Build a contrast map.** Several patterns are commonly confused: Adapter vs. Facade, Decorator vs. Proxy, State vs. Strategy, Factory Method vs. Abstract Factory, Command vs. Strategy. After completing the deck, make a single-page table of these pairs with one differentiating sentence each. Review the table at the start of each session.

3. **Code one pattern per day.** After studying a pattern's three cards, spend 20 minutes implementing it from scratch in your language of choice for a domain you know. The act of writing the participants — Context, Interface, ConcreteImplementation — and watching them fit together is what moves knowledge from recognition to production.

4. **Apply spaced repetition by category.** After the first full pass, rate each card: Easy (review in 1 week), Medium (review in 3 days), Hard (review tomorrow). Free tools like Anki let you digitise these cards and automate the schedule — paste the Q/A pairs directly into Anki for a persistent study deck.
