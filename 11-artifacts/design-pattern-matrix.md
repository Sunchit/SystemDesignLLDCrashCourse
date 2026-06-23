# Design Pattern Matrix — Comprehensive Selection and Reference Guide

> Reference artifact for pattern selection decisions. Use the tables to identify the right pattern quickly, understand trade-offs, and communicate choices clearly in design discussions and interviews.

---

## Part 1: Pattern Overview Matrix

Patterns marked **[CRITICAL]** appear in the majority of senior engineering interviews and are essential to master first.

| Pattern | Category | Intent (1 line) | Key Abstraction | Scope | Complexity | Interview Frequency |
|---|---|---|---|---|---|---|
| **Abstract Factory** **[CRITICAL]** | Creational | Create families of related objects without specifying concrete classes | Product family interface | Object | Med | High |
| **Builder** **[CRITICAL]** | Creational | Separate complex object construction from its representation | Director + Builder interface | Object | Med | High |
| **Factory Method** **[CRITICAL]** | Creational | Let subclasses decide which class to instantiate | Creator with abstract factory method | Class | Low | High |
| **Prototype** | Creational | Create new objects by cloning an existing instance | Cloneable interface | Object | Low | Med |
| **Singleton** **[CRITICAL]** | Creational | Ensure a class has only one instance with a global access point | Single static instance | Object | Low | High |
| **Adapter** **[CRITICAL]** | Structural | Convert an interface into another interface that clients expect | Wrapper around adaptee | Class / Object | Low | High |
| **Bridge** | Structural | Decouple abstraction from implementation so both can vary independently | Abstraction holds impl reference | Object | Med | Med |
| **Composite** **[CRITICAL]** | Structural | Compose objects into tree structures to represent part-whole hierarchies | Component interface for leaf and composite | Object | Med | High |
| **Decorator** **[CRITICAL]** | Structural | Attach additional responsibilities to an object dynamically | Wrapping via same interface | Object | Med | High |
| **Facade** **[CRITICAL]** | Structural | Provide a simplified interface to a complex subsystem | Unified front-facing class | Object | Low | High |
| **Flyweight** | Structural | Use sharing to efficiently support a large number of fine-grained objects | Intrinsic vs extrinsic state split | Object | High | Low |
| **Proxy** **[CRITICAL]** | Structural | Provide a surrogate to control access to another object | Same interface as real subject | Object | Med | High |
| **Chain of Responsibility** **[CRITICAL]** | Behavioral | Pass a request along a chain of handlers until one handles it | Handler interface with successor | Object | Med | High |
| **Command** **[CRITICAL]** | Behavioral | Encapsulate a request as an object to support undo, queuing, and logging | Command interface with execute() | Object | Med | High |
| **Interpreter** | Behavioral | Define a grammar and an interpreter for sentences in that language | Abstract expression tree | Class | High | Low |
| **Iterator** **[CRITICAL]** | Behavioral | Provide a way to sequentially access elements without exposing internals | Iterator interface | Object | Low | High |
| **Mediator** | Behavioral | Define an object that encapsulates how a set of objects interact | Central mediator reference | Object | Med | Med |
| **Memento** | Behavioral | Capture and restore an object's internal state without violating encapsulation | Opaque memento object | Object | Med | Med |
| **Observer** **[CRITICAL]** | Behavioral | Define a one-to-many dependency so dependents are notified automatically | Subject + Observer interface | Object | Low | High |
| **State** **[CRITICAL]** | Behavioral | Allow an object to alter its behavior when its internal state changes | State interface per state | Object | Med | High |
| **Strategy** **[CRITICAL]** | Behavioral | Define a family of algorithms, encapsulate each, and make them interchangeable | Strategy interface | Object | Low | High |
| **Template Method** **[CRITICAL]** | Behavioral | Define the skeleton of an algorithm in a base class, defer steps to subclasses | Abstract class with hook methods | Class | Low | High |
| **Visitor** | Behavioral | Add operations to object structures without modifying the elements | Visitor interface per operation | Object | High | Med |

### Category Summary

| Category | Count | Primary Purpose |
|---|---|---|
| Creational (5) | Abstract Factory, Builder, Factory Method, Prototype, Singleton | Object creation and initialization |
| Structural (7) | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy | Object composition and interface shaping |
| Behavioral (11) | Chain of Responsibility, Command, Interpreter, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor | Object communication and algorithm encapsulation |

---

## Part 2: Pattern Selection Matrix

Use this table when you have a design problem and need to identify the right pattern. Match the smell or requirement to the pattern, then verify with the "Why it fits" column.

| Problem / Smell | Pattern to Use | Why It Fits |
|---|---|---|
| Need to create objects without specifying the exact concrete class at compile time | **Factory Method** | Creator defines factory method; subclasses override to return concrete type. Client depends only on abstract type. |
| Need to create entire families of related objects that must be used together | **Abstract Factory** | Factory interface creates a suite of related products. Switching the factory switches the whole product family atomically. |
| Need to construct a complex object step by step with many optional parts | **Builder** | Director calls step methods on builder; client chooses which builder. Keeps constructor clean and readable. |
| Need a single shared instance across the entire application | **Singleton** | Private constructor + static accessor guarantees one instance; use double-checked locking or enum for thread safety. |
| Need to duplicate objects at runtime, cloning from a prototype instead of constructing | **Prototype** | `clone()` copies the object graph; avoids costly instantiation and subclass explosion for product hierarchies. |
| Need to wrap a legacy or third-party API so it matches the interface your code expects | **Adapter** | Adapter implements the target interface and delegates to the adaptee. No change to existing code. |
| Need to add behavior to individual objects dynamically without modifying the class or using inheritance | **Decorator** | Decorator wraps the same interface, adding behavior before/after delegation. Composable at runtime. |
| Need to simplify a complex subsystem behind a single clean entry point | **Facade** | Facade class orchestrates subsystem classes; clients call one simple API. Reduces coupling to internals. |
| Need to represent hierarchical trees (part-whole) and treat leaves and composites uniformly | **Composite** | Component interface shared by Leaf and Composite. Client code is identical for single items and entire subtrees. |
| Need to decouple an abstraction (what) from its implementation (how) so both can vary independently | **Bridge** | Abstraction holds a reference to the Implementor interface. Orthogonal hierarchies can grow independently. |
| Need to control access to an object — for lazy init, security checks, or remote access | **Proxy** | Proxy implements the same interface as RealSubject, intercepting calls before or after delegation. |
| Need to notify multiple objects when another object's state changes | **Observer** | Subject maintains a list of Observers; calls `update()` on all when state changes. Loose coupling via interface. |
| Need to encapsulate a request as an object so it can be queued, logged, or undone | **Command** | Command object holds action + receiver. Invoker stores commands; undo stack is a list of executed commands. |
| Need to pass a request through a pipeline of handlers, each deciding to handle or pass on | **Chain of Responsibility** | Handler interface has `handle()` + reference to next handler. Handlers can be assembled at runtime. |
| Need to change an object's behavior based on its internal state without large conditionals | **State** | Each state is a separate class implementing a State interface. Context delegates to current state object. |
| Need to select and swap algorithms at runtime without client code knowing the difference | **Strategy** | Strategy interface defines the algorithm contract. Context holds a reference; client injects the desired strategy. |
| Need to define the skeleton of an algorithm but let subclasses fill in specific steps | **Template Method** | Abstract class defines the algorithm; hook methods are overridden by subclasses. Inversion of control within the hierarchy. |
| Need to traverse a collection without exposing its internal structure | **Iterator** | Iterator interface (`hasNext`, `next`) decouples traversal logic from the collection. Multiple iterators can coexist. |
| Need to undo operations and restore previous state | **Memento** | Originator creates a Memento snapshot; Caretaker holds it. Originator restores from Memento without exposing internals. |
| Need to reduce many-to-many communication dependencies between objects | **Mediator** | Mediator centralizes interaction logic. Objects only talk to the mediator, not to each other. |
| Need to perform operations across a heterogeneous object structure without polluting element classes | **Visitor** | Visitor interface has a visit method per element type. Elements call `accept(visitor)`. New operations = new Visitor. |
| Need to efficiently support millions of fine-grained objects by sharing common state | **Flyweight** | Intrinsic (shared, immutable) state lives in the flyweight; extrinsic state is passed in at runtime. |
| Need to support multiple languages or expression grammars | **Interpreter** | Abstract expression tree; each node implements `interpret(context)`. Good for small, well-defined grammars only. |
| Need to decouple the sender of a request from its receiver without knowing the receiver at compile time | **Command** or **Mediator** | Command decouples single sender/receiver pairs; Mediator decouples many-to-many within a component group. |
| Class hierarchy is exploding due to every combination of features needing its own subclass | **Decorator** or **Bridge** | Decorator adds features at object level; Bridge separates two independent dimensions. Pick Bridge if both dimensions are deep hierarchies. |

---

## Part 3: Pattern Relationships

### 3a. Commonly Used Together

| Pattern Pair / Group | Relationship Type | Why They Complement Each Other |
|---|---|---|
| **Factory Method + Template Method** | Structural complement | Factory Method often appears inside a Template Method. The template defines the algorithm skeleton; the factory method step instantiates the right product variant. |
| **Abstract Factory + Singleton** | Lifecycle complement | The concrete factory is usually a Singleton — you need exactly one factory per product family at runtime. |
| **Builder + Composite** | Construction complement | Builder is ideal for constructing complex Composite trees step by step. The builder accumulates nodes; the finished product is a Composite root. |
| **Command + Memento** | Undo/redo pairing | Command encapsulates the action; Memento captures the state before execution. Together they give full undo/redo: execute command, store memento, restore memento on undo. |
| **Observer + Mediator** | Event routing complement | Observer is used for one-to-many broadcast; Mediator routes events among a closed group. Use Observer for domain events, Mediator for UI or protocol coordination inside a component. |
| **Decorator + Strategy** | Behavior layering | Decorator adds fixed wrapping behavior; Strategy injects a swappable algorithm. Combine when you need both static wrapping layers and runtime-selected core logic. |
| **Proxy + Decorator** | Interface wrapping pair | Both wrap the same interface. Proxy controls access (security, lazy init, remote); Decorator adds behavior. They are often layered: a Security Proxy wraps a Logging Decorator wrapping the real object. |
| **Composite + Visitor** | Traversal pairing | Visitor traverses the Composite tree performing operations on each node type. This keeps the tree nodes clean of operation logic while enabling arbitrary new operations through new Visitors. |
| **State + Strategy** | Delegation pair | State transitions manage which Strategy-like object is active. State adds the self-transition concern; Strategy is purely about algorithm selection. Often confused — use State when the object owns transition logic. |
| **Chain of Responsibility + Command** | Pipeline pair | Command objects travel down a Chain of Responsibility. Each handler in the chain decides whether to execute the command or pass it on. Common in middleware and filter pipelines. |
| **Abstract Factory + Prototype** | Object creation pair | When products are complex, the factory methods inside an Abstract Factory can use Prototype (clone) rather than constructing new instances from scratch. |
| **Facade + Singleton** | Access control pair | Facades are frequently implemented as Singletons because only one simplified entry point into the subsystem is needed application-wide. |

### 3b. Patterns That Solve Similar Problems Differently

| Problem | Option A | Option B | When to Pick A | When to Pick B |
|---|---|---|---|---|
| **Adding behavior to objects** | **Decorator** | **Subclassing (Inheritance)** | Behavior needs to be added/removed at runtime; multiple independent behaviors may be combined | Behavior is fixed at compile time and the class hierarchy is shallow |
| **Object creation without specifying class** | **Factory Method** | **Abstract Factory** | One product family; subclasses control which product is created | Multiple related products that must be used together as a consistent family |
| **Encapsulating varying algorithms** | **Strategy** | **Template Method** | Algorithm selection at runtime; composition preferred over inheritance | Algorithm skeleton is fixed; only specific steps vary; inheritance is acceptable |
| **One-to-many dependency notification** | **Observer** | **Mediator** | Open-ended broadcast to an unknown or variable set of listeners | Closed set of objects that communicate in complex ways; centralizing logic reduces coupling |
| **Decoupling a request from its execution** | **Command** | **Chain of Responsibility** | One specific receiver per command; undo/queue/log required | Multiple potential handlers; first able handler processes the request |
| **Controlling object access** | **Proxy** | **Decorator** | Goal is access control, lazy init, or remoting — same interface, different lifecycle | Goal is adding behavior — same interface, feature enrichment |
| **Wrapping a foreign interface** | **Adapter** | **Facade** | Adapting one incompatible interface to another; structural translation | Simplifying many interfaces in a subsystem into one; not translating types |
| **Managing complex state transitions** | **State** | **Strategy** | State object drives transitions; the context transitions itself | Algorithm selection is external; context does not self-transition |

---

## Part 4: Patterns in Real Systems

### 4a. Spring Framework Patterns

| Pattern | Where Spring Uses It | Specific Component |
|---|---|---|
| **Singleton** | Default bean scope in the IoC container | `ApplicationContext` — every bean is a singleton by default unless scoped otherwise |
| **Factory Method** | Bean instantiation | `BeanFactory.getBean()`, `@Bean` methods in `@Configuration` classes |
| **Abstract Factory** | Context bootstrapping | `ApplicationContextFactory`, `WebApplicationContext` variants created by different factory implementations |
| **Proxy** | AOP, `@Transactional`, `@Cacheable`, `@Async` | JDK dynamic proxies and CGLIB subclass proxies wrap target beans transparently |
| **Template Method** | Boilerplate elimination | `JdbcTemplate`, `RestTemplate`, `TransactionTemplate` — algorithm skeleton with hook callbacks |
| **Observer / Event** | Application event system | `ApplicationEventPublisher`, `@EventListener`, `ApplicationListener<E>` |
| **Decorator** | Request/response processing | `HttpMessageConverter` chain, `HandlerInterceptor` stack wrapping request handling |
| **Chain of Responsibility** | Servlet filter pipeline | `FilterChain` in Spring Security (`SecurityFilterChain`); `HandlerInterceptor` pre/post processing |
| **Strategy** | Pluggable algorithm injection | `ResourceLoader`, `MessageSource`, `ViewResolver` — each is a strategy injected into `DispatcherServlet` |
| **Adapter** | MVC handler mapping | `HandlerAdapter` converts diverse controller types (annotated, `HttpRequestHandler`, `Servlet`) to a uniform interface |
| **Composite** | Security filter chains | `CompositeAuthenticationProvider`, `AuthorityAuthorizationManager` composing multiple managers |
| **Command** | Transaction management | `TransactionCallback` passed to `TransactionTemplate.execute()` |
| **Mediator** | `DispatcherServlet` | Central front controller routes requests to the right handler, view resolver, and message converter |
| **Flyweight** | Scope caching | Prototype-scoped beans with shared configuration metadata; `ResolvableType` caches type resolution results |

### 4b. Java Standard Library Patterns

| Pattern | Java Class / Interface | Notes |
|---|---|---|
| **Iterator** | `java.util.Iterator`, `java.lang.Iterable` | Universal traversal contract for all collection types; for-each syntax desugars to `Iterator` |
| **Observer** | `java.util.Observable` (deprecated), `PropertyChangeListener`, `java.util.EventListener` | AWT/Swing event model; `PropertyChangeSupport` in JavaBeans |
| **Factory Method** | `java.util.Calendar.getInstance()`, `java.nio.file.Files.newInputStream()`, `java.sql.DriverManager.getConnection()` | Returns a concrete subtype decided at runtime; caller sees only the abstract type |
| **Abstract Factory** | `javax.xml.parsers.DocumentBuilderFactory`, `javax.xml.transform.TransformerFactory` | Creates families of XML processing objects (parser, transformer, validator) |
| **Singleton** | `java.lang.Runtime.getRuntime()`, `java.awt.Desktop.getDesktop()` | Single JVM-wide resource instance accessed via static accessor |
| **Prototype** | `java.lang.Cloneable`, `Object.clone()`, `java.util.Arrays.copyOf()` | Shallow and deep copy mechanisms; `Cloneable` marks classes supporting clone |
| **Builder** | `java.lang.StringBuilder`, `java.util.stream.Stream.Builder`, `java.net.http.HttpRequest.newBuilder()` | Step-by-step construction; `StringBuilder` is the canonical Java Builder example |
| **Decorator** | `java.io.BufferedInputStream(new FileInputStream(...))`, `java.util.Collections.unmodifiableList()`, `java.util.Collections.synchronizedList()` | IO stream wrappers compose decorators; `Collections` utility wrappers add behavior |
| **Adapter** | `java.util.Arrays.asList()`, `java.io.InputStreamReader` (adapts `InputStream` to `Reader`) | Structural translation between incompatible interfaces within the standard library |
| **Composite** | `java.awt.Container` (contains `Component`), `javax.swing.JPanel` | AWT/Swing widget tree; containers hold components uniformly |
| **Strategy** | `java.util.Comparator`, `java.util.concurrent.ThreadFactory`, `java.util.function.Predicate` | Algorithms passed as strategy objects; lambda-friendly with functional interfaces |
| **Template Method** | `java.io.InputStream` (defines read contract; concrete classes implement `read(byte[], int, int)`) | Abstract class defines algorithm skeleton; subclasses implement primitive operations |
| **Command** | `java.lang.Runnable`, `java.util.concurrent.Callable`, `java.awt.event.ActionListener` | Encapsulates an action as an object; passed to executors, event queues, or thread pools |
| **Proxy** | `java.lang.reflect.Proxy`, `java.rmi.Remote` stubs | Dynamic proxy creates runtime implementation of interfaces; RMI stubs are remote proxies |

### 4c. Famous Real-World Uses

| System / Product | Pattern(s) Used | How and Why |
|---|---|---|
| **Netflix Hystrix (Circuit Breaker / Resilience)** | **Proxy + State + Command** | Every service call is wrapped in a `HystrixCommand` (Command pattern). The command proxy intercepts the call and transitions through states (CLOSED → OPEN → HALF-OPEN) based on failure thresholds. The State pattern drives the circuit's current behavior. |
| **JUnit 5 (Test Framework)** | **Template Method + Composite + Decorator + Observer** | `TestCase` uses Template Method (setUp → test → tearDown). Test suites are Composites of tests and sub-suites. Test runners attach as Observers (via `TestListener`) to receive pass/fail events. Extensions decorate test execution. |
| **Android Framework (UI)** | **Composite + Observer + Template Method + Factory Method** | `View`/`ViewGroup` form a Composite tree. `setOnClickListener` is Observer. `Activity` lifecycle (`onCreate`, `onResume`) is Template Method. `LayoutInflater` is Factory Method that instantiates views by XML tag name. |
| **Apache Kafka (Messaging)** | **Observer + Iterator + Strategy** | Producers publish events; consumers observe topics (Observer at the distribution level). `ConsumerRecord` iteration follows Iterator. `Partitioner`, `Serializer`, and `Deserializer` are swappable Strategies injected via configuration. |
| **Hibernate / JPA** | **Proxy + Unit of Work (Command variant) + Template Method** | Hibernate generates CGLIB proxies for lazy-loaded entities. The `Session` accumulates changes as a unit of work (Command batch). `SessionFactory` builds sessions using Template Method hooks for dialects. |
| **IntelliJ IDEA Plugin System** | **Factory + Strategy + Composite + Visitor** | `PsiElementVisitor` traverses the AST (Visitor over Composite PSI tree). Inspections are Strategies registered with the platform. `PsiElementFactory` creates AST nodes. Plugin actions are Commands registered in the IDE action system. |
| **AWS SDK (Java v2)** | **Builder + Strategy + Decorator + Proxy** | Every request is built with a fluent Builder. `ExecutionInterceptor` chain is a Decorator stack around the HTTP call. `AwsCredentialsProvider` is a Strategy. The SDK client itself is a Proxy to the remote service. |
| **Google Guava** | **Builder + Decorator + Strategy + Flyweight** | `ImmutableList.builder()` (Builder). `Iterables`, `Collections2` wrap collections (Decorator). `Function`, `Predicate` (Strategy). `Interners` (Flyweight for string/object interning). |
| **Log4j / SLF4J** | **Chain of Responsibility + Decorator + Strategy + Facade** | `Logger` → `Appender` chain (Chain of Responsibility). `Appender` wrappers like `AsyncAppender` decorate synchronous appenders. `Layout` (Strategy) controls formatting. SLF4J itself is a Facade over Log4j, Logback, or java.util.logging. |
| **React (UI Library)** | **Composite + Observer + Strategy + Decorator** | The component tree is a Composite (every component can hold child components). State management (hooks) is Observer-inspired. Higher-Order Components are Decorators. Renderers (DOM, Native, Test) are Strategies injected at the platform layer. |

---

## Quick-Reference: Pattern Decision Flowchart (Text)

```
START: What is your primary concern?

├─ OBJECT CREATION
│   ├─ One object, one instance globally?          → Singleton
│   ├─ Clone an existing instance?                 → Prototype
│   ├─ Complex multi-step construction?            → Builder
│   ├─ Subclass decides which class to make?       → Factory Method
│   └─ Families of related objects?                → Abstract Factory
│
├─ OBJECT STRUCTURE / COMPOSITION
│   ├─ Wrap incompatible interface?                → Adapter
│   ├─ Add behavior dynamically, composably?       → Decorator
│   ├─ Simplify a subsystem API?                   → Facade
│   ├─ Part-whole tree structure?                  → Composite
│   ├─ Control access / lazy init / remoting?      → Proxy
│   ├─ Vary abstraction AND implementation?        → Bridge
│   └─ Many identical fine-grained objects?        → Flyweight
│
└─ OBJECT BEHAVIOR / COMMUNICATION
    ├─ Broadcast state change to many listeners?   → Observer
    ├─ Swap algorithm at runtime?                  → Strategy
    ├─ Vary steps within a fixed algorithm?        → Template Method
    ├─ Behavior changes with internal state?       → State
    ├─ Traverse collection without exposing it?    → Iterator
    ├─ Encapsulate request for undo/queue/log?     → Command
    ├─ Pass request through pipeline of handlers?  → Chain of Responsibility
    ├─ Decouple many-to-many object interaction?   → Mediator
    ├─ Capture/restore object state?               → Memento
    ├─ Add operations to object structure?         → Visitor
    └─ Interpret a grammar/expression tree?        → Interpreter
```

---

## Interview Cheat Sheet: Most Frequently Asked Patterns

| Rank | Pattern | What Interviewers Test |
|---|---|---|
| 1 | **Observer** | Event systems, MVC, reactive streams, pub-sub architecture |
| 2 | **Strategy** | Replacing conditionals, dependency injection, algorithm selection |
| 3 | **Factory Method / Abstract Factory** | Object creation, dependency inversion, plug-in architectures |
| 4 | **Decorator** | Open/Closed Principle, Java IO streams, feature toggles |
| 5 | **Singleton** | Thread safety (double-checked locking, enum), anti-pattern awareness |
| 6 | **Command** | Undo/redo, queues, macro recording, transaction logs |
| 7 | **Builder** | Fluent APIs, complex object construction, immutable objects |
| 8 | **Proxy** | AOP, lazy loading, security, caching |
| 9 | **Chain of Responsibility** | Middleware, filter pipelines, Spring Security |
| 10 | **State** | FSM design, workflow engines, replacing nested conditionals |
| 11 | **Composite** | File systems, UI trees, organizational hierarchies |
| 12 | **Template Method** | Framework design, hook points, IoC inside a class hierarchy |
| 13 | **Adapter** | Legacy integration, third-party API wrapping |
| 14 | **Facade** | Service layers, SDK design, API gateway simplification |
| 15 | **Iterator** | Custom collections, lazy sequences, cursor-based pagination |
