# Design Pattern Selection Guide
## Capstone Reference — All 23 GoF Patterns

> **How to use this guide.** Read the decision tree first to find candidate patterns, cross-check the comparison matrix to confirm the fit, then use the interview cheat sheet before whiteboard sessions. The anti-pattern recognition section is the fastest way to spot which pattern a codebase is silently begging for.

---

## Table of Contents

1. [Decision Tree — Which Pattern Should I Use?](#1-decision-tree)
2. [Full Pattern Comparison Matrix (all 23 GoF)](#2-full-pattern-comparison-matrix)
3. [Behavioral Patterns Deep-Dive Comparison](#3-behavioral-patterns-deep-dive-comparison)
4. [Anti-Pattern Recognition Guide](#4-anti-pattern-recognition-guide)
5. [Common Pattern Combinations](#5-common-pattern-combinations)
6. [Patterns in Popular Java Frameworks](#6-patterns-in-popular-java-frameworks)
7. [Interview Cheat Sheet — Two-Liners for All 23 Patterns](#7-interview-cheat-sheet)
8. [10 Hardest Cross-Pattern Interview Questions](#8-hardest-cross-pattern-interview-questions)

---

## 1. Decision Tree

```
START: What is my primary design problem?
│
├─── "I need to CREATE objects" ──────────────────────────────── [CREATIONAL]
│     │
│     ├── "I want to hide which concrete class is instantiated"
│     │     ├── "One factory method, subclasses decide" ──────── Factory Method
│     │     └── "Families of related objects, no concrete refs" ─ Abstract Factory
│     │
│     ├── "I need to build complex objects step-by-step" ──────── Builder
│     │
│     ├── "Copying an existing object is cheaper than new()" ──── Prototype
│     │
│     └── "Exactly one instance must exist globally" ──────────── Singleton
│
├─── "I need to COMPOSE or WRAP objects" ─────────────────────── [STRUCTURAL]
│     │
│     ├── "Two incompatible interfaces must work together"
│     │     ├── "Wrap to convert interface" ─────────────────── Adapter
│     │     └── "Decouple abstraction from implementation" ───── Bridge
│     │
│     ├── "I need to add behavior without changing the class"
│     │     ├── "Stack wrappers at runtime, same interface" ──── Decorator
│     │     └── "Control/guard access to the real object" ─────── Proxy
│     │
│     ├── "I need to simplify a complex subsystem" ────────────── Facade
│     │
│     ├── "Tree structure of part-whole hierarchies" ──────────── Composite
│     │
│     └── "Share fine-grained objects to save memory" ─────────── Flyweight
│
└─── "I need to define HOW objects INTERACT / behave" ────────── [BEHAVIORAL]
      │
      ├── "Interchangeable algorithms at runtime" ────────────── Strategy
      │
      ├── "Notify many objects when one changes" ─────────────── Observer
      │
      ├── "Encapsulate a request as an object (undo/queue)" ──── Command
      │
      ├── "Object behaviour changes based on its own state" ──── State
      │
      ├── "Pass a request along a chain of handlers" ─────────── Chain of Responsibility
      │
      ├── "Skeleton algorithm in base class, steps overridden" ── Template Method
      │
      ├── "Centralise many-to-many object communication" ──────── Mediator
      │
      ├── "New operation on stable class hierarchy, no change" ── Visitor
      │
      ├── "Access elements of a collection sequentially" ──────── Iterator
      │
      ├── "One-to-many, lazy pull notification" ─────────────────
      │     └── (see Observer; pull model variant)
      │
      ├── "Define a grammar and interpret sentences" ──────────── Interpreter
      │
      └── "Capture & restore object state (snapshot)" ─────────── Memento
```

### Creational Sub-tree (Expanded)

```
Need to create objects?
│
├── Single vs Family?
│     ├── Single object
│     │     ├── Subclass decides type? ────────────── Factory Method
│     │     ├── Copy from prototype? ──────────────── Prototype
│     │     ├── Step-by-step construction? ─────────── Builder
│     │     └── Only one ever? ────────────────────── Singleton
│     └── Family of related objects
│           └── Platform/theme families? ────────────── Abstract Factory
│
└── Who knows the concrete type?
      ├── The subclass ────────────────────────────── Factory Method
      ├── A dedicated factory object ──────────────── Abstract Factory
      └── Nobody (caller passes a prototype) ──────── Prototype
```

### Structural Sub-tree (Expanded)

```
Need to compose or adapt?
│
├── Adapting an interface?
│     ├── Legacy class, new interface ──────────────── Adapter
│     └── Vary both abstraction and impl independently ─ Bridge
│
├── Adding responsibilities?
│     ├── Stack them at runtime? ──────────────────── Decorator
│     └── Control access (lazy init, security, cache)? ─ Proxy
│
├── Simplify for clients?
│     └── Hide subsystem complexity ────────────────── Facade
│
├── Tree structures?
│     └── Leaf and composite treated uniformly ──────── Composite
│
└── Memory optimisation?
      └── Many identical fine-grained objects ─────── Flyweight
```

### Behavioral Sub-tree (Expanded)

```
Need to manage object interaction or algorithm variation?
│
├── Algorithm varies?
│     ├── Whole algorithm swappable ───────────────── Strategy
│     ├── Skeleton fixed, steps vary ──────────────── Template Method
│     └── Behaviour varies by object state ─────────── State
│
├── Communication / event flow?
│     ├── One-to-many notification ────────────────── Observer
│     └── Many-to-many, centralised hub ───────────── Mediator
│
├── Request handling?
│     ├── Encapsulate + queue + undo ───────────────── Command
│     ├── Chain of handlers, first match wins ──────── Chain of Responsibility
│     └── Snapshot for undo/rollback ───────────────── Memento
│
├── Object traversal?
│     ├── Sequential access, hide structure ─────────── Iterator
│     ├── New operation across hierarchy ────────────── Visitor
│     └── Tree traversal with operation ────────────── Composite + Visitor
│
└── Domain language?
      └── Parse/evaluate grammar ──────────────────── Interpreter
```

---

## 2. Full Pattern Comparison Matrix

> Complexity rating: L = Low, M = Medium, H = High (implementation effort + cognitive overhead)

| Pattern | Category | Core Problem Solved | Key Participants | Inheritance vs Composition | Runtime vs Compile-time | Complexity | Reach for it when... |
|---|---|---|---|---|---|---|---|
| **Factory Method** | Creational | Decouple object creation from usage; let subclasses choose the class | Creator, ConcreteCreator, Product, ConcreteProduct | Inheritance (subclass overrides factory method) | Compile-time (type fixed per subclass) | L | You have a base class that should not depend on concrete products; new product types are added by subclassing |
| **Abstract Factory** | Creational | Create families of related objects without specifying concrete classes | AbstractFactory, ConcreteFactory, AbstractProduct, ConcreteProduct | Composition (factory object injected) | Runtime (swap factory to change family) | M | Platform/theme variants that must be consistent across many object types; switching UI toolkit or DB driver |
| **Builder** | Creational | Construct complex objects step-by-step; same process, different representations | Director, Builder, ConcreteBuilder, Product | Composition | Runtime | M | Object has many optional fields (telescoping constructor smell); construction sequence matters; immutable result |
| **Prototype** | Creational | Clone an existing object instead of constructing from scratch | Prototype (clone()), ConcretePrototype, Client | Composition | Runtime | L | Object creation is expensive; configuration templates; objects differ only in state at clone time |
| **Singleton** | Creational | Guarantee exactly one instance; global access point | Singleton | N/A | Compile-time | L | Shared resource with one logical instance (logger, config, thread pool); use sparingly — often an anti-pattern |
| **Adapter** | Structural | Convert one interface into another expected by the client | Target, Adapter, Adaptee, Client | Both (class adapter = inheritance; object adapter = composition) | Compile-time (class) / Runtime (object) | L | Integrating legacy or third-party code with an incompatible interface; wrapping external SDK |
| **Bridge** | Structural | Decouple abstraction from implementation so both can vary independently | Abstraction, RefinedAbstraction, Implementor, ConcreteImplementor | Composition | Runtime | M | Two orthogonal dimensions of variation (e.g., Shape × RenderingAPI); prevents n×m subclass explosion |
| **Composite** | Structural | Treat individual objects and compositions uniformly (tree structures) | Component, Leaf, Composite, Client | Composition + Recursive | Runtime | M | Tree/hierarchy: file system, UI widget tree, org chart, expression AST |
| **Decorator** | Structural | Add responsibilities to objects dynamically without subclassing | Component, ConcreteComponent, Decorator, ConcreteDecorator | Composition | Runtime | M | Stack optional features/behaviours: logging, encryption, compression, caching on a stream or service |
| **Facade** | Structural | Provide a simplified interface to a complex subsystem | Facade, Subsystem classes | Composition | Compile-time | L | Simplify entry point for a library/framework; hide internal coupling from clients; layered architecture |
| **Flyweight** | Structural | Share fine-grained objects to reduce memory when many instances are nearly identical | Flyweight, ConcreteFlyweight, FlyweightFactory, Context | Composition | Runtime | H | Thousands/millions of objects sharing intrinsic state; game entities, characters in a text editor |
| **Proxy** | Structural | Control access to an object (lazy init, protection, remote, caching) | Subject, RealSubject, Proxy | Composition | Runtime | M | Lazy loading heavy resources; access control; remote method invocation; caching layer |
| **Chain of Responsibility** | Behavioral | Pass a request along a handler chain; each handler decides to process or forward | Handler, ConcreteHandler, Client | Composition (linked list / chain) | Runtime | M | Event/exception handling pipelines; middleware; auth → validation → business logic chains |
| **Command** | Behavioral | Encapsulate a request as an object; supports undo, queue, log | Command, ConcreteCommand, Invoker, Receiver, Client | Composition | Runtime | M | Undo/redo; job queues; transactional operations; macro recording; GUI button actions |
| **Interpreter** | Behavioral | Define a grammar and interpreter for a language | AbstractExpression, TerminalExpression, NonterminalExpression, Context | Inheritance + Composition | Compile-time (grammar fixed) | H | Domain-specific languages; SQL parsers; regex engines; rule engines; expression evaluators |
| **Iterator** | Behavioral | Sequentially access elements of a collection without exposing the underlying structure | Iterator, ConcreteIterator, Aggregate, ConcreteAggregate | Composition | Runtime | L | Decouple traversal logic from collection; support multiple traversals simultaneously |
| **Mediator** | Behavioral | Centralise complex many-to-many communication between objects | Mediator, ConcreteMediator, Colleague | Composition | Runtime | M | UI dialog where many controls affect each other; chat system; air traffic control; event bus |
| **Memento** | Behavioral | Capture and externalise an object's internal state for later restoration | Originator, Memento, Caretaker | Composition | Runtime | M | Undo/redo stacks; save-game snapshots; transaction checkpoints; editor history |
| **Observer** | Behavioral | Define a one-to-many dependency so dependents are notified automatically | Subject (Observable), Observer, ConcreteSubject, ConcreteObserver | Composition | Runtime | M | Event systems; MVC (Model notifies View); reactive streams; publish-subscribe |
| **State** | Behavioral | Allow an object to alter its behaviour when its internal state changes | Context, State, ConcreteState | Composition | Runtime | M | Vending machine, order lifecycle, TCP connection, game character FSM; replace giant switch on state field |
| **Strategy** | Behavioral | Define a family of algorithms, encapsulate each, make them interchangeable | Context, Strategy, ConcreteStrategy | Composition | Runtime | L | Pluggable sort algorithms; payment methods; routing strategies; replace long if/else on type |
| **Template Method** | Behavioral | Define the skeleton of an algorithm in a base class; defer steps to subclasses | AbstractClass, ConcreteClass | Inheritance | Compile-time | L | Invariant workflow with variable steps; data parsers; game AI loops; report generators |
| **Visitor** | Behavioral | Add new operations to a class hierarchy without modifying the classes | Visitor, ConcreteVisitor, Element, ConcreteElement, ObjectStructure | Composition + Double Dispatch | Compile-time (hierarchy must be stable) | H | Compilers (AST operations); serialisation; linting rules; adding analytics to a stable domain model |

---

## 3. Behavioral Patterns Deep-Dive Comparison

The 8 core behavioral patterns most commonly tested at interview and encountered in production systems:

| Attribute | Strategy | Observer | Command | State | Chain of Responsibility | Template Method | Mediator | Visitor |
|---|---|---|---|---|---|---|---|---|
| **Runtime flexibility** | High — swap algorithm with no client change | High — add/remove observers at any time | High — queue, delay, cancel commands | High — add new states without changing context | High — reorder/add handlers dynamically | Low — subclasses fixed at compile-time | High — change wiring without touching colleagues | Low — hierarchy must be known at compile-time |
| **Undo support** | No | No | Yes — store commands, call undo() | No (but State can record history) | No | No | No | No |
| **Decoupling level** | Context decoupled from algorithm | Subject decoupled from observer type; observer decoupled from each other | Invoker decoupled from receiver; sender decoupled from timing | Context decoupled from state logic | Sender decoupled from handler; handlers decoupled from each other | Base class controls flow; steps are coupled via inheritance | All colleagues decoupled from each other; coupled only to mediator | Element hierarchy decoupled from operations |
| **Coupling direction** | Context → Strategy (one-way) | Subject → Observer interface; Observer may hold Subject ref | Invoker → Command interface; Command → Receiver | Context → State interface; State → Context (back-ref optional) | Handler → next Handler | AbstractClass → overridden hooks | Colleague → Mediator; Mediator → all Colleagues | Visitor → Element types; Element → Visitor interface |
| **Typical Java use case** | `Comparator<T>`, `javax.servlet.Filter`, payment processing | `java.util.EventListener`, Spring `ApplicationEvent`, `PropertyChangeListener` | GUI button actions, `javax.swing.Action`, job schedulers, CQRS | Order FSM, connection lifecycle, parser states | Servlet filter chain, Spring Security filters, logging pipelines | `java.io.InputStream`, `AbstractList`, JUnit `@Before`/`@After` lifecycle | Spring MVC `DispatcherServlet`, chat room, UI form validation | Compiler AST traversal, `javax.lang.model.element.ElementVisitor` |
| **When it breaks down** | Too many strategies becomes strategy explosion; consider combining with Factory | Too many observers cause memory leaks (strong refs); event ordering becomes unpredictable | Too many command classes; undo semantics are complex for composite operations | Too many states → many small classes; circular state transitions are hard to trace | Long chains have poor debuggability; request may reach no handler | Deep inheritance hierarchies; hard to override only part of a complex template | Mediator becomes a god object if it takes on too much logic | Cannot add new element types without modifying all visitors — open/closed inverted |
| **Key differentiator phrase** | "Pluggable algorithm" | "Push notification to subscribers" | "Request as first-class object" | "Same object, different behaviour per state" | "Who handles this? Pass it on." | "I define the steps, you fill them in" | "Don't call your colleagues, call me" | "New operation, no class change" |

---

## 4. Anti-Pattern Recognition Guide

Use this section to diagnose existing code that is missing a pattern. Each entry shows the code smell, the Java signal, and the prescription.

---

### Long if/else or switch on type/algorithm → **Strategy**

```java
// SMELL: selecting algorithm by type flag
if (sortType.equals("BUBBLE")) { bubbleSort(data); }
else if (sortType.equals("MERGE")) { mergeSort(data); }
else if (sortType.equals("QUICK")) { quickSort(data); }
// Every new algorithm requires editing this method.
```
**Prescription:** Extract a `SortStrategy` interface with a `sort()` method. Each algorithm becomes a `ConcreteStrategy`. The context receives a strategy object and delegates.

---

### Tight coupling between event producer and consumer → **Observer**

```java
// SMELL: producer directly calls every consumer
public void processOrder(Order o) {
    emailService.send(o);           // direct call
    inventoryService.update(o);     // direct call
    analyticsService.track(o);      // direct call
}
// Adding a new consumer requires modifying processOrder.
```
**Prescription:** Make `OrderProcessor` an `Observable`. Register `EmailObserver`, `InventoryObserver`, `AnalyticsObserver`. Producer fires `notifyObservers(event)`; each observer handles its own concern.

---

### No undo/redo, or operations need queuing/logging → **Command**

```java
// SMELL: UI button directly mutates document
button.addActionListener(e -> {
    document.deleteSelectedText();   // un-undoable, un-loggable
});
```
**Prescription:** Introduce `DeleteCommand implements Command { execute(); undo(); }`. The invoker (history stack) calls `execute()` and stores the command for `undo()`.

---

### Giant switch (or if/else) on an object's own state field → **State**

```java
// SMELL: behaviour branches on internal state throughout the class
public void handle(Request r) {
    switch (this.state) {
        case IDLE:    handleIdle(r);    break;
        case RUNNING: handleRunning(r); break;
        case PAUSED:  handlePaused(r);  break;
    }
}
// Every method has the same switch. Adding a state means editing all of them.
```
**Prescription:** Extract a `State` interface with a `handle(Context, Request)` method. Each case becomes a `ConcreteState`. The context delegates: `currentState.handle(this, r)`.

---

### Long if/else chain checking handler identity → **Chain of Responsibility**

```java
// SMELL: manual routing through handlers
if (request.type == AUTH)       { authHandler.process(request); }
else if (request.type == RATE)  { rateLimiter.process(request); }
else if (request.type == LOG)   { logger.process(request); }
// Fragile ordering, hard to extend.
```
**Prescription:** Each handler implements `Handler` with `setNext(Handler)` and `handle(Request)`. Build the chain at startup: `auth.setNext(rateLimiter).setNext(logger)`.

---

### Copy-pasted algorithmic skeleton across subclasses → **Template Method**

```java
// SMELL: DataParser, CsvParser, JsonParser all have:
// readFile() → parseHeader() → parseRows() → writeOutput()
// The structure is identical but the middle steps differ.
```
**Prescription:** `AbstractDataParser` implements the skeleton as a `final parse()` method calling `readFile()`, `parseHeader()`, `parseRows()`, `writeOutput()`. Subclasses override only the varying steps.

---

### Spaghetti object references — every object calls every other object directly → **Mediator**

```java
// SMELL: in a UI dialog
nameField.onChange(() -> submitButton.setEnabled(!nameField.isEmpty() && emailField.isValid()));
emailField.onChange(() -> submitButton.setEnabled(!nameField.isEmpty() && emailField.isValid()));
submitButton.onClick(() -> { nameField.disable(); emailField.disable(); });
// N objects, N*(N-1) references.
```
**Prescription:** Introduce `FormMediator`. Each control calls `mediator.notify(this, event)`. The mediator centralises all coordination logic.

---

### New methods needed on a stable class hierarchy without modifying classes → **Visitor**

```java
// SMELL: every time a new report format is needed, Shape hierarchy is opened
class Circle  { String toSvg() {...} String toPdf() {...} String toCanvas() {...} }
class Square  { String toSvg() {...} String toPdf() {...} String toCanvas() {...} }
// Hierarchy reopened for every new export format.
```
**Prescription:** Define `ShapeVisitor { visitCircle(Circle); visitSquare(Square); }`. Each export format is a `ConcreteVisitor`. Shapes implement `accept(ShapeVisitor v) { v.visit(this); }`.

---

### New subclass for every feature combination → **Decorator** (and/or **Strategy**)

```java
// SMELL: subclass explosion
class LoggingEmailService extends EmailService { ... }
class ThrottledEmailService extends EmailService { ... }
class LoggingThrottledEmailService extends LoggingEmailService { ... }
// n features = 2^n subclasses.
```
**Prescription:** `LoggingDecorator` and `ThrottlingDecorator` both implement `EmailService` and wrap another `EmailService`. Compose at runtime: `new LoggingDecorator(new ThrottlingDecorator(realEmailService))`.

---

### Expensive object creation / configuration templates → **Prototype**

```java
// SMELL: constructing complex configuration objects from scratch repeatedly
Config c = new Config();
c.setHost("prod.db.com"); c.setPort(5432); c.setPool(20); // ... 15 more fields
// Copy-constructing from a template is safer and faster.
```
**Prescription:** Implement `Cloneable` (or a `copy()` method). Store pre-configured prototypes in a registry. Callers clone a prototype and mutate only what differs.

---

### Snapshot / save-restore needed → **Memento**

```java
// SMELL: undo saves entire object reference (aliasing bug) or exposes internals
history.push(editor);  // pushing live object — undo corrupts history
```
**Prescription:** `Editor.save()` returns an opaque `EditorMemento`. `Caretaker` stores mementos. `Editor.restore(memento)` replaces internal state without exposing it.

---

### Complex object construction with many optional parameters → **Builder**

```java
// SMELL: telescoping constructor
new HttpRequest(url, method, body, headers, timeout, retries, followRedirects, proxy, ...);
// Or: a partially initialised object mutated by many setters — not thread safe.
```
**Prescription:** `HttpRequest.Builder` with fluent setters. `builder.url(u).method(GET).timeout(5000).build()` produces an immutable `HttpRequest`.

---

## 5. Common Pattern Combinations

Patterns rarely exist in isolation. These pairings solve compound problems and appear repeatedly in production systems.

---

### Command + Memento — Full Undo/Redo

**Problem:** Undo needs to reverse the effect of a command, but the command may have touched multiple objects.

**How they combine:**
- Each `Command.execute()` asks the target `Originator` for a `Memento` before mutating it.
- `Command.undo()` passes the saved `Memento` back to `Originator.restore()`.
- The `Caretaker` (history stack) holds the executed commands; pop to undo.

```
[UndoStack]
    │  push(command)
    ▼
[ConcreteCommand]
    ├── before = receiver.save()   // Memento captured
    ├── execute() → mutate receiver
    └── undo()   → receiver.restore(before)
```

**Where you see it:** Text editors (IntelliJ, VS Code), Photoshop, database migration tools.

---

### Strategy + Factory Method — Pluggable Algorithm Creation

**Problem:** Strategies must be instantiated based on runtime configuration without the client knowing concrete types.

**How they combine:**
- A `StrategyFactory.create(config)` returns the appropriate `Strategy` implementation.
- The client receives a `Strategy` interface — it never references concrete strategy classes.
- Adding a new strategy requires a new class and a factory registration, nothing else.

**Where you see it:** Payment gateway selection, sort algorithm selection, ML model routing.

---

### Observer + Mediator — Decoupled Event Bus with Central Hub

**Problem:** Components need to react to events, but the publisher/subscriber count grows and direct observer registration creates too many cross-references.

**How they combine:**
- The `Mediator` is the sole `Observer` registered with each event source.
- The mediator receives events and routes them to the appropriate colleague objects.
- Colleagues communicate only through the mediator — they never observe each other.

```
[Button] ──notifyObservers──▶ [FormMediator] ──▶ [TextField]
[CheckBox] ──────────────────▶ [FormMediator] ──▶ [SubmitButton]
```

**Where you see it:** Spring `ApplicationEventPublisher`, Redux store, UI frameworks.

---

### Composite + Visitor — Tree Operations Without Modifying Nodes

**Problem:** A tree of heterogeneous nodes (Composite structure) needs multiple independent operations applied to it (pretty-print, validate, serialise, measure).

**How they combine:**
- The `Composite` pattern provides the tree and the `accept(Visitor)` hook in every node.
- Each `Visitor` implements logic for each node type via overloaded `visit()` methods.
- The Composite node's `accept()` calls `visitor.visit(this)` then delegates to children.

**Where you see it:** Compiler AST (parse → type-check → optimise → codegen all as visitors), XML/JSON document processors, file system analysers.

---

### Template Method + Strategy — Fixed Skeleton, Pluggable Steps

**Problem:** An algorithm's overall shape is fixed (Template Method), but some steps are best varied by composition rather than inheritance (to allow runtime swapping).

**How they combine:**
- The `AbstractClass` implements the template (final method).
- Some steps call a `Strategy` object that is injected rather than overridden.
- Subclasses override simple hooks; complex variable steps delegate to strategies.

```java
abstract class DataProcessor {
    final void process() {
        data = load();
        data = transformer.transform(data);  // Strategy — injected
        save(data);
    }
    abstract Data load();
    abstract void save(Data d);
}
```

**Where you see it:** Spring `JdbcTemplate`, `RestTemplate`, `AbstractMessageConverterMethodArgumentResolver`.

---

### Decorator + Strategy — Composable Behaviour with Pluggable Algorithm

**Problem:** Multiple cross-cutting concerns (logging, caching, retry) each need to be composable wrappers, and each wrapper itself may select an algorithm at runtime.

**How they combine:**
- Each `Decorator` wraps the component interface and adds a concern.
- Inside each decorator, a `Strategy` governs how that concern executes (e.g., the `CacheDecorator` holds a `CacheEvictionStrategy`).

**Where you see it:** HTTP client libraries (OkHttp interceptors), service mesh sidecars.

---

### State + Singleton — Shared, Stateless State Objects

**Problem:** A `State` object carries no instance state of its own (all state lives in the `Context`). Creating a new state object on every transition wastes memory.

**How they combine:**
- Each `ConcreteState` class is a Singleton (or an enum constant).
- Transitions assign a reference to a shared Singleton state: `context.setState(RunningState.INSTANCE)`.
- Thread safety must be verified — if the state object is truly stateless, it is safe.

**Where you see it:** TCP connection state machine, enum-based FSMs in Java.

---

## 6. Patterns in Popular Java Frameworks

### Spring Framework

| Pattern | Where in Spring | Concrete Classes |
|---|---|---|
| **Singleton** | Every `@Bean` default scope | `DefaultSingletonBeanRegistry`, `BeanFactory` |
| **Factory Method** | `BeanFactory.getBean()` | `AbstractBeanFactory.doGetBean()`, `@Bean` methods in `@Configuration` |
| **Abstract Factory** | `ApplicationContext` variations | `AnnotationConfigApplicationContext`, `ClassPathXmlApplicationContext` |
| **Prototype** | `@Scope("prototype")` beans | `AbstractBeanFactory` with `SCOPE_PROTOTYPE` |
| **Builder** | Fluent config APIs | `MockMvcRequestBuilders`, `UriComponentsBuilder`, `RestTemplateBuilder`, `SecurityMockMvcRequestPostProcessors` |
| **Proxy** | AOP, transactions, security | `JdkDynamicAopProxy`, `CglibAopProxy`, `TransactionProxyFactoryBean` |
| **Decorator** | `HttpServletRequestWrapper`, security filters | `ContentCachingRequestWrapper`, `SecurityContextPersistenceFilter` |
| **Template Method** | Data access and messaging | `JdbcTemplate`, `HibernateTemplate`, `RestTemplate`, `KafkaTemplate`, `AmqpTemplate` |
| **Observer** | Application events | `ApplicationEvent`, `ApplicationListener`, `@EventListener`, `ApplicationEventPublisher` |
| **Strategy** | Sorting, MVC resolution | `HandlerMapping`, `ViewResolver`, `MessageConverter`, `AuthenticationProvider` |
| **Chain of Responsibility** | Security filter chain | `FilterChainProxy`, `UsernamePasswordAuthenticationFilter`, `SecurityFilterChain` |
| **Mediator** | `DispatcherServlet` routing | `DispatcherServlet` (mediates between `HandlerMapping`, `HandlerAdapter`, `ViewResolver`) |
| **Adapter** | MVC handler adapters | `RequestMappingHandlerAdapter`, `HttpMessageConverterExtractor` |
| **Facade** | `JdbcTemplate`, `SimpMessagingTemplate` | Hides JDBC boilerplate; hides WebSocket send complexity |
| **Composite** | Security voters | `AffirmativeBased`, `UnanimousBased` aggregate `AccessDecisionVoter` list |
| **Command** | Task execution | `Runnable` / `Callable` submitted to `ThreadPoolTaskExecutor` |
| **Visitor** | Bean definition post-processors | `BeanDefinitionRegistryPostProcessor`, `BeanFactoryPostProcessor` |

---

### Java Standard Library

| Pattern | Where in JDK | Concrete Classes |
|---|---|---|
| **Singleton** | Runtime, Desktop | `Runtime.getRuntime()`, `Desktop.getDesktop()`, `System` (static facade) |
| **Factory Method** | Collections, NIO, logging | `Collections.unmodifiableList()`, `Files.newInputStream()`, `Logger.getLogger()` |
| **Abstract Factory** | XML, crypto | `DocumentBuilderFactory`, `SAXParserFactory`, `KeyFactory`, `CipherFactory` |
| **Builder** | Strings, HTTP | `StringBuilder`, `StringBuffer`, `HttpClient.newBuilder()`, `ProcessBuilder` |
| **Prototype** | Object cloning | `Object.clone()`, `Cloneable` interface; `Arrays.copyOf()` |
| **Adapter** | I/O streams, collections | `InputStreamReader` (byte→char), `Arrays.asList()`, `Collections.enumeration()` |
| **Decorator** | I/O streams | `BufferedInputStream(new FileInputStream(...))`, `GZIPOutputStream`, `PrintWriter` |
| **Composite** | Swing, AWT | `Container` extends `Component`; `JPanel` contains `JComponent` instances |
| **Proxy** | Reflection | `java.lang.reflect.Proxy`, `InvocationHandler` |
| **Flyweight** | Wrapper caches | `Integer.valueOf(-128..127)`, `Boolean.TRUE/FALSE`, `Byte.valueOf()` |
| **Iterator** | Collections framework | `java.util.Iterator`, `Iterable`, `ListIterator`, enhanced-for loop |
| **Observer** | Beans, events | `java.util.Observable` (deprecated), `PropertyChangeListener`, `EventListener` |
| **Strategy** | Sorting, comparison | `Comparator<T>`, `java.util.function.Predicate`, `ExecutorService` (strategy for thread management) |
| **Template Method** | Abstract collections | `AbstractList.get()` (abstract), `AbstractMap.entrySet()` (abstract) |
| **Command** | Concurrency | `Runnable`, `Callable`, `FutureTask` |
| **State** | Thread lifecycle | `Thread.State` enum (`NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TERMINATED`) |
| **Visitor** | NIO file system | `FileVisitor<Path>`, `Files.walkFileTree()`, `SimpleFileVisitor` |
| **Memento** | Serialisation | `Serializable` (object state externalised), `ObjectOutputStream` / `ObjectInputStream` |
| **Interpreter** | Regex, format | `java.util.regex.Pattern`, `java.text.Format` subclasses |
| **Chain of Responsibility** | Exception handling | Exception propagation up the call stack; `ClassLoader` delegation model |

---

### Java EE / Jakarta EE

| Pattern | Where | Concrete Classes |
|---|---|---|
| **Facade** | Session beans as service layer | `@Stateless`, `@Stateful` beans hiding complex subsystems behind a single EJB interface |
| **Proxy** | CDI, EJB | CDI interceptors (`@Interceptor`), EJB container-generated proxies for transaction/security |
| **Chain of Responsibility** | Servlet filters, JAX-RS | `javax.servlet.Filter` chain, JAX-RS `ContainerRequestFilter` / `ContainerResponseFilter` |
| **Factory Method** | CDI producers | `@Produces` methods in CDI beans |
| **Observer** | CDI events | `@Observes` annotation, `Event<T>` injection |
| **Decorator** | CDI decorators | `@Decorator`, `@Delegate` — wraps a CDI bean at the interface level |
| **Composite** | JSF component tree | `UIComponent` tree (`UIViewRoot` → `UIForm` → `UIInput`) |
| **Strategy** | JPA, CDI alternatives | `@Alternative` beans; pluggable `JPA` providers (Hibernate, EclipseLink) |
| **Command** | JMS message consumers | `MessageDrivenBean`, `@MessageDriven` — request encapsulated in JMS `Message` |
| **Template Method** | CDI lifecycle | `@PostConstruct` / `@PreDestroy` callbacks in managed beans |

---

## 7. Interview Cheat Sheet

One precise two-liner for each of the 23 GoF patterns. Memorise these; say exactly this at a whiteboard.

---

### Creational

**Factory Method:** Defines an interface for creating an object, but lets subclasses decide which class to instantiate. Key signal: "I have a base class that should not depend on concrete products — the subclass knows what to make."

**Abstract Factory:** Provides an interface for creating families of related objects without specifying their concrete classes. Key signal: "I need consistent sets of products — swap the factory, swap the whole family."

**Builder:** Separates the construction of a complex object from its representation so the same construction process can create different representations. Key signal: "Object has too many constructor parameters, some optional — build it step by step."

**Prototype:** Specifies the kinds of objects to create using a prototypical instance, then creates new objects by copying that prototype. Key signal: "Creating from scratch is expensive — clone a configured template instead."

**Singleton:** Ensures a class has only one instance and provides a global access point to it. Key signal: "This resource must be shared, and two instances would be a bug — but prefer dependency injection over global access."

---

### Structural

**Adapter:** Converts the interface of a class into another interface that clients expect. Key signal: "I have an existing class but its interface doesn't match what my code expects — wrap it."

**Bridge:** Decouples an abstraction from its implementation so both can vary independently. Key signal: "I have two dimensions of variation — without Bridge I'd have N × M subclasses."

**Composite:** Composes objects into tree structures to represent part-whole hierarchies; clients treat individual objects and compositions uniformly. Key signal: "I have a tree; leaf and branch must be treated the same by the client."

**Decorator:** Attaches additional responsibilities to an object dynamically; an alternative to subclassing for extending functionality. Key signal: "I need to add optional behaviours at runtime — not bake them into subclasses."

**Facade:** Provides a simplified interface to a complex subsystem. Key signal: "My client has to orchestrate a dozen classes just to do one thing — hide that behind a single method."

**Flyweight:** Uses sharing to support a large number of fine-grained objects efficiently. Key signal: "I need millions of objects but most of their state is identical — share the intrinsic part."

**Proxy:** Provides a surrogate or placeholder for another object to control access to it. Key signal: "I need to intercept calls to an object — for lazy load, access control, caching, or remoting."

---

### Behavioral

**Chain of Responsibility:** Avoids coupling the sender of a request to its receiver by giving more than one object a chance to handle the request; chains the receiving objects. Key signal: "Multiple handlers might handle this request — pass it along until someone claims it."

**Command:** Encapsulates a request as an object, thereby letting you parameterise clients with different requests, queue or log requests, and support undoable operations. Key signal: "I need to undo, queue, or log operations — make the request itself a first-class object."

**Interpreter:** Given a language, defines a representation for its grammar along with an interpreter that uses the representation to interpret sentences. Key signal: "I'm evaluating a small, well-defined language or expression grammar repeatedly."

**Iterator:** Provides a way to access the elements of an aggregate object sequentially without exposing its underlying representation. Key signal: "Clients need to traverse a collection, but the collection's internal structure should stay hidden."

**Mediator:** Defines an object that encapsulates how a set of objects interact; promotes loose coupling by keeping objects from referring to each other explicitly. Key signal: "My objects are all talking to each other — I have N² relationships and it's chaos."

**Memento:** Without violating encapsulation, captures and externalises an object's internal state so the object can be restored to this state later. Key signal: "I need undo without exposing the object's internals — the object itself packages its own snapshot."

**Observer:** Defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically. Key signal: "When this thing changes, several unrelated things need to react — and the publisher shouldn't know who they are."

**State:** Allows an object to alter its behaviour when its internal state changes; the object will appear to change its class. Key signal: "My object behaves differently in different modes and the switch/if-chains are duplicated everywhere."

**Strategy:** Defines a family of algorithms, encapsulates each one, and makes them interchangeable; lets the algorithm vary independently of the clients that use it. Key signal: "I have the same operation implemented multiple ways — inject the right one rather than branching."

**Template Method:** Defines the skeleton of an algorithm in a base class, deferring some steps to subclasses; subclasses redefine certain steps without changing the overall structure. Key signal: "The overall steps are the same, only certain steps differ — define the skeleton in the base, let subclasses fill in the blanks."

**Visitor:** Represents an operation to be performed on the elements of an object structure; lets you define a new operation without changing the classes of the elements on which it operates. Key signal: "I have a stable class hierarchy but keep needing to add new operations — add operations as visitors, not as methods."

---

## 8. Hardest Cross-Pattern Interview Questions

These are the questions that trip up even experienced engineers. Each answer is a model response at the staff/senior engineer level.

---

### Q1. Strategy vs Template Method vs State — when would you pick each?

**Model Answer:**

All three vary *behaviour*, but they differ in mechanism and trigger:

- **Strategy** uses *composition* to vary a *whole algorithm* at *runtime*. The client explicitly chooses and injects a strategy object. There is no inheritance relationship between context and strategy. Use it when algorithms are independently reusable and the client selects one at construction or method call time.

- **Template Method** uses *inheritance* to vary *steps within a fixed algorithm* at *compile time*. The base class owns the algorithm's skeleton and calls overrideable hooks. The subclass cannot change the overall flow. Use it when the invariant part of the algorithm is the point, and variation is in the details. The tradeoff is that it requires subclassing, which is less flexible than composition.

- **State** changes behaviour based on the *object's own internal state*, and the object controls its own transitions. The context class is not aware of which state it is in at a detailed level — it delegates everything to the current state object, which may also trigger the transition to the next state. Use it when an object's behaviour changes wholesale depending on a lifecycle phase, and those phases are modelled explicitly.

**One-line rule:** *Strategy is injected from outside; Template Method is inherited from above; State transitions itself from within.*

---

### Q2. Command vs Strategy — they both encapsulate behaviour, so what's the difference?

**Model Answer:**

Both wrap behaviour in an object, but their *intent and lifecycle* differ fundamentally:

| Dimension | Command | Strategy |
|---|---|---|
| Intent | Encapsulate a *request* (an action + its parameters) | Encapsulate an *algorithm* (a computation policy) |
| Undo | Central feature — commands are designed to be reversed | Not a concern |
| Lifecycle | Executed once, stored for history, replayed or undone | Injected and reused; not stored per-invocation |
| Receiver | Has a specific receiver object it acts on | The context IS the receiver |
| Parameters | Carries all parameters it needs at construction time | Parameters come from the context at call time |

A `Command` asks "what should happen and to what?" and stores the answer. A `Strategy` asks "how should this be computed?" and delegates to the injected algorithm.

Concrete example: A `PasteCommand` is a Command — it records what was pasted where and can be undone. A `CompressionStrategy` is a Strategy — it defines how bytes are compressed; you don't undo it.

---

### Q3. Observer vs Mediator — both decouple objects, so when does Observer become Mediator?

**Model Answer:**

Both decouple components but at different scales and in different directions.

**Observer** is a *one-to-many* dependency: one subject, many observers. Observers register interest in a single event source. Observers may still be aware of which subject they are observing. As the number of subjects and observer cross-registrations grows, you get a web of references — every object observing every other object.

**Mediator** is a *many-to-many* hub: all colleagues know only the mediator, never each other. The mediator contains all the coordination logic that would otherwise be spread across the colleagues.

**The inflection point:** When you find yourself with `A.addObserver(B)`, `B.addObserver(C)`, `C.addObserver(A)`, and the system has become circular and hard to trace, you have outgrown Observer. Introduce a Mediator. All objects send events to the mediator, and the mediator decides who gets notified.

A practical heuristic: Observer when the subject is the natural owner of the notification; Mediator when no single object should own the routing logic.

---

### Q4. Decorator vs Proxy vs Adapter — they all wrap an object. How do you tell them apart?

**Model Answer:**

All three use composition to wrap an object, but each has a distinct *purpose* and *interface relationship*:

| Dimension | Adapter | Proxy | Decorator |
|---|---|---|---|
| Interface relationship | Converts *incompatible* interface to expected one — Target ≠ Adaptee | Same interface as the real subject | Same interface as the wrapped component |
| Purpose | Interface translation | Access control, lazy init, caching, remote | Adding behaviour / responsibility |
| Transparency | Client knows it is talking to an adapter (or doesn't care) | Client is usually unaware of proxy | Client is usually unaware of decorator |
| Stacking | Rarely stacked | Rarely stacked | Designed to be stacked |
| When determined | Usually compile-time integration task | Usually infrastructure-level concern | Usually runtime feature composition |

Memory aid: **Adapter** changes the *shape* of the interface. **Proxy** controls *access* through the same shape. **Decorator** *adds* to the same shape.

---

### Q5. Composite vs Decorator — both involve recursive composition. What is the structural difference?

**Model Answer:**

Both use the same component interface and hold references to component objects, but:

- **Composite** is about *tree structure*: it represents part-whole hierarchies. A Composite node holds *multiple* children, all of type `Component`. The purpose is uniform treatment of leaf and branch. Operations typically propagate down the tree.

- **Decorator** is about *linear wrapping*: a Decorator holds exactly *one* component, always of the same type. The purpose is to add behaviour transparently. Multiple decorators form a chain (not a tree).

Key structural difference: Composite has `List<Component> children`; Decorator has `Component wrappee` (singular).

Another difference: Composite is closed (you don't expect to add new node types frequently); Decorator is open (you expect new decorators to be added). Composite + Visitor handles new *operations* without changing the tree.

---

### Q6. Factory Method vs Abstract Factory — when is one enough and when do you need the other?

**Model Answer:**

**Factory Method** solves: "I need to create one product, but I want the subclass to decide which concrete product." It is a single method — one product type, one creation point. The variation is through *inheritance*.

**Abstract Factory** solves: "I need to create *a coordinated family* of products without binding to concrete classes." It is an *object* with multiple factory methods — one per product in the family. The variation is through *composition* (inject a different factory object).

**Rule of thumb:** Start with Factory Method. Upgrade to Abstract Factory when you identify multiple products that must be consistent with each other (e.g., `createButton()` + `createScrollBar()` must both come from the same platform theme).

Abstract Factory is essentially a bundle of Factory Methods grouped into a coherent interface for a product family.

---

### Q7. What are the Singleton anti-patterns and how do you avoid them?

**Model Answer:**

Singleton is the most abused pattern. The problems:

1. **Hidden global state.** Singletons are mutable global state. Any code anywhere can mutate the singleton, making bugs non-reproducible and test order-dependent.

2. **Tight coupling.** Code that calls `Foo.getInstance()` is impossible to test in isolation. You cannot substitute a test double.

3. **SRP violation.** The class is responsible for its own domain logic AND for managing its own lifecycle (instance control). Two jobs, one class.

4. **Classloader issues in multi-tenant / OSGi environments.** "One instance per JVM" is not true when multiple classloaders are in play.

5. **Thread-safety traps.** Naive implementations (`if (instance == null)`) race under concurrency.

**Mitigations:**
- Prefer *dependency injection* over `getInstance()`. Register the object as a singleton in a DI container (Spring `@Bean`, Guice `@Singleton`). The container enforces one instance; your class stays testable.
- Use the *enum Singleton* idiom for true Singletons in Java (JLS-guaranteed thread safety, serialisation-safe).
- If a Singleton must exist, make it stateless or immutable.

---

### Q8. When should you NOT use design patterns?

**Model Answer:**

Patterns are solutions to *recurring problems in specific contexts*. They impose cost — extra classes, extra indirection, extra conceptual load. Do not use them when:

1. **The problem is simpler than the pattern.** A Strategy for two algorithms is over-engineering. An if/else is clearer.

2. **You are optimising prematurely.** Flyweight and Proxy add complexity. Measure first.

3. **The team cannot maintain it.** A pattern only works if the team knows it. Visitor applied to a codebase where nobody recognises it becomes unmaintainable indirection.

4. **You are "pattern matching" rather than solving.** Starting with "I'll use an Abstract Factory here" without a concrete reason for a product family is cargo-cult engineering.

5. **The pattern fights the language.** Many GoF patterns exist to compensate for missing language features. Java lambdas largely replace `Command`, `Strategy`, and `Template Method` for simple cases. `Comparator.comparing()` is Strategy without the boilerplate class.

**Heuristic:** Apply a pattern only when you can name the specific problem it solves and verify the resulting code is easier to change than the alternative.

---

### Q9. How do the 23 GoF patterns map to SOLID principles?

**Model Answer:**

| SOLID Principle | Patterns that enforce / enable it |
|---|---|
| **S — Single Responsibility** | **Facade** (subsystem classes have focused jobs, Facade coordinates them); **Command** (request logic separated from invoker); **Mediator** (coordination logic extracted from colleagues); **Visitor** (operations extracted from element classes) |
| **O — Open/Closed** | **Strategy** (new algorithms without modifying context); **Decorator** (new behaviours without modifying component); **Observer** (new observers without modifying subject); **Visitor** (new operations without modifying elements); **Template Method** (new subclass steps without changing skeleton) |
| **L — Liskov Substitution** | **Composite** (Leaf and Composite are substitutable as Component); **Strategy** (all ConcreteStrategy objects are substitutable); **Factory Method** (ConcreteProduct substitutes Product) |
| **I — Interface Segregation** | **Facade** (clients use a narrow Facade interface, not the full subsystem); **Adapter** (client uses only the Target interface); all patterns where a narrow interface is extracted (Command, Observer, Strategy, Iterator) |
| **D — Dependency Inversion** | **Abstract Factory** (clients depend on factory and product abstractions); **Factory Method** (creator depends on Product abstraction); **Strategy** (context depends on Strategy interface); **Observer** (subject depends on Observer interface); **Bridge** (abstraction layer depends on Implementor abstraction); **Proxy** / **Decorator** (wrap Subject interface, not concrete class) |

A key insight: most structural and behavioral patterns achieve OCP and DIP simultaneously, because they extract an interface and inject concrete implementations.

---

### Q10. Given a new system design problem, how do you identify which patterns to apply?

**Model Answer:**

Follow a structured thought process, not a pattern catalogue lookup:

**Step 1 — Identify the axes of change.**
Ask: "What is likely to change over time?" Variation in object *creation* → creational patterns. Variation in object *structure* → structural patterns. Variation in object *behaviour* or *interaction* → behavioral patterns.

**Step 2 — Detect the code smells.**
Before architecture review, read the anti-pattern recognition section. Long if/else chains are the most reliable signal. Count the number of places in the code that would change if a new variant were added — if the answer is "more than one file," a pattern is likely warranted.

**Step 3 — Check the relationship type.**
- One-to-many notification: Observer.
- Object creating objects: Factory family.
- Object delegating to another object that varies: Strategy or Bridge.
- Wrapping to add behaviour: Decorator.
- Wrapping to control access: Proxy.
- Wrapping to translate: Adapter.
- Tree structures: Composite.

**Step 4 — Check combinatorial explosion.**
N types × M behaviours = N×M subclasses is always solved by Bridge or Decorator (separate concerns) or Strategy (inject the varying concern).

**Step 5 — Validate against YAGNI.**
Only add a pattern when the second variant arrives. One algorithm does not need Strategy. One observer does not need Observer infrastructure. The pain of the pattern should be less than the pain of the problem.

**Step 6 — Name the pattern explicitly in code.**
If you apply a pattern, name it: `PaymentStrategy`, `OrderObserver`, `FileVisitor`. Naming signals intent and helps future maintainers recognise the structure without re-deriving it.

---

## Quick Reference Index

| If you need to... | Pattern(s) |
|---|---|
| Create objects flexibly | Factory Method, Abstract Factory |
| Build complex objects | Builder |
| Copy objects | Prototype |
| One global instance | Singleton |
| Connect incompatible interfaces | Adapter |
| Vary abstraction AND implementation | Bridge |
| Build trees, uniform treatment | Composite |
| Add behaviour dynamically | Decorator |
| Simplify a complex subsystem | Facade |
| Handle huge numbers of similar objects | Flyweight |
| Control access to an object | Proxy |
| Handle a request through a pipeline | Chain of Responsibility |
| Undo operations / queue requests | Command |
| Parse a language or expression | Interpreter |
| Traverse a collection | Iterator |
| Centralise many-to-many communication | Mediator |
| Save and restore state | Memento |
| Notify on change | Observer |
| Vary behaviour by lifecycle phase | State |
| Plug in algorithms | Strategy |
| Define algorithm skeleton | Template Method |
| Add operations to stable hierarchy | Visitor |

---

*This guide covers the complete GoF catalogue. For deep implementation walkthroughs of the 8 core behavioral patterns, see files 01 through 08 in this directory.*
