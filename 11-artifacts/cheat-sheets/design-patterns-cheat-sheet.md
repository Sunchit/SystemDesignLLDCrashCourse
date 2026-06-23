# Design Patterns Cheat Sheet — All 23 GoF Patterns

> **Legend:** ⭐ = Top 18 interview-relevant patterns (most frequently tested in FAANG/product company LLD rounds)

---

## Master Reference Table

| Pattern | Category | Intent (1 line) | Key Abstraction | Java Example (call-site / class) | When to Use | Common Pitfall |
|---|---|---|---|---|---|---|
| **Singleton** ⭐ | Creational | Ensure exactly one instance exists and provide a global access point | Single instance + private constructor | `Runtime.getRuntime()` | • Shared resource (config, thread pool, logger)<br>• Exactly-one constraint<br>• Expensive init done once | Double-checked locking without `volatile`; makes testing hard (global state) |
| **Factory Method** ⭐ | Creational | Defer object creation to subclasses; define the interface, let subclass decide the type | Creator + ConcreteCreator | `Collection.iterator()` | • Don't know exact type at compile time<br>• Subclasses should control creation<br>• Avoid tight coupling to concrete classes | Overusing — simple `new` is fine when type is known and stable |
| **Abstract Factory** ⭐ | Creational | Produce families of related objects without specifying concrete classes | Factory interface + product families | `DocumentBuilderFactory.newInstance()` | • Cross-platform UI kits<br>• Families of related objects must be consistent<br>• Swap entire product families at runtime | Explosion of classes when adding new product variants |
| **Builder** ⭐ | Creational | Separate complex object construction from its representation | Builder + Director | `StringBuilder`, `Locale.Builder` | • Object has many optional parameters<br>• Construction is multi-step / order matters<br>• Immutable objects with many fields | Mutable builder leaking; forgetting required fields (no compile-time check) |
| **Prototype** | Creational | Create new objects by copying an existing instance | `clone()` / copy constructor | `Object.clone()` | • Object init is expensive<br>• Runtime configuration of new objects<br>• Registry of prototype templates | Shallow vs. deep copy confusion; breaking `equals`/`hashCode` after clone |
| **Adapter** ⭐ | Structural | Convert an interface into another interface clients expect | Adapter wraps Adaptee | `Arrays.asList()`, `InputStreamReader` | • Integrate legacy or third-party code<br>• Reuse existing class with incompatible interface<br>• Bridge two independently-evolved APIs | Two-layer adapters (adapter wrapping adapter); prefer composition over inheritance |
| **Bridge** | Structural | Decouple abstraction from implementation so both can vary independently | Abstraction + Implementor | `java.util.logging` (Logger + Handler) | • Avoid permanent binding between abstraction and implementation<br>• Both abstraction and implementation should extend independently<br>• Hide implementation from client entirely | Confusing with Adapter (Bridge is proactive; Adapter is retrofit) |
| **Composite** ⭐ | Structural | Treat individual objects and compositions uniformly via a common interface | Component + Leaf + Composite | `javax.swing.JComponent`, file system | • Tree structures (UI hierarchy, org charts, file system)<br>• Client should treat leaves and composites identically<br>• Recursive part-whole hierarchy | Overly general component interface forces irrelevant methods on Leaf |
| **Decorator** ⭐ | Structural | Attach additional responsibilities to an object dynamically without subclassing | Decorator wraps Component | `BufferedInputStream(new FileInputStream(...))` | • Add behavior at runtime<br>• Subclassing leads to class explosion<br>• Responsibilities can be combined in arbitrary order | Deep nesting hard to debug; ordering of decorators matters and is implicit |
| **Facade** ⭐ | Structural | Provide a simplified interface to a complex subsystem | Facade + subsystem classes | `javax.faces.context.FacesContext` | • Simplify complex API for common use cases<br>• Layer subsystem for dependency management<br>• Hide third-party complexity | Turning Facade into a god-object that knows too much |
| **Flyweight** | Structural | Share fine-grained objects efficiently to reduce memory when many instances are needed | Flyweight (intrinsic state) + context (extrinsic state) | `Integer.valueOf()` (cache −128..127), `String` intern | • Large number of similar small objects<br>• Memory is critical<br>• Object state can be split into intrinsic/extrinsic | Passing extrinsic state everywhere becomes messy; not useful when objects are few |
| **Proxy** ⭐ | Structural | Provide a surrogate that controls access to another object | Subject interface + Proxy + RealSubject | `java.lang.reflect.Proxy`, Spring AOP | • Lazy initialization (virtual proxy)<br>• Access control (protection proxy)<br>• Remote object, caching, logging | Proxy vs. Decorator distinction blurred — Proxy controls access, Decorator adds behavior |
| **Chain of Responsibility** ⭐ | Behavioral | Pass request along a chain of handlers; each handler decides to handle or forward | Handler interface + chain links | `javax.servlet.Filter`, `java.util.logging.Logger` | • Multiple objects may handle a request<br>• Handler set should be dynamic<br>• Decouple sender from receiver | Request can fall off end of chain silently; hard to debug when no handler fires |
| **Command** ⭐ | Behavioral | Encapsulate a request as an object to support undo, queuing, logging | Command interface with `execute()` | `java.lang.Runnable`, `javax.swing.Action` | • Undo/redo support<br>• Queue or log operations<br>• Parameterize objects with operations | Command object proliferation; state management for complex undo gets complicated |
| **Interpreter** | Behavioral | Define a grammar and an interpreter that uses the grammar to interpret sentences | AbstractExpression + Terminal/Nonterminal | `java.util.regex.Pattern` | • Recurring grammar to interpret<br>• Simple grammar (complex → use parser generators)<br>• Grammar changes rarely | Poor performance on complex grammars; rarely used directly |
| **Iterator** ⭐ | Behavioral | Provide a way to sequentially access elements of a collection without exposing its representation | `Iterator<E>` interface | `java.util.Iterator`, enhanced for-loop | • Traverse without exposing internals<br>• Support multiple traversal strategies<br>• Unified iteration over different collections | Modifying collection during iteration → `ConcurrentModificationException` |
| **Mediator** | Behavioral | Define an object that encapsulates how a set of objects interact, reducing direct references | Mediator interface + Colleagues | `java.util.concurrent.Executor`, ATC analogy | • Many-to-many colleague interactions<br>• Tight coupling between collaborators<br>• Centralize complex coordination logic | Mediator can become a monolithic god-object |
| **Memento** | Behavioral | Capture and externalize an object's internal state for later restoration without violating encapsulation | Originator + Memento + Caretaker | `java.io.Serializable` (conceptually) | • Undo/snapshot functionality<br>• State rollback without exposing internals<br>• Transaction-like save points | Storing too many snapshots is memory-expensive; deep-copy required |
| **Observer** ⭐ | Behavioral | Define a one-to-many dependency so all dependents are notified automatically when state changes | Subject + Observer interface | `java.util.EventListener`, PropertyChangeListener | • Event-driven systems<br>• Decoupled notification<br>• Multiple views of the same model | Memory leaks from unregistered observers; notification storms with many observers |
| **State** ⭐ | Behavioral | Allow an object to alter its behavior when its internal state changes — appears to change its class | Context + State interface + ConcreteStates | `javax.faces.lifecycle.Lifecycle` | • Behavior changes based on object state<br>• Large conditionals on state variables<br>• State transitions need to be explicit | State explosion; state objects sharing too much context (tight coupling) |
| **Strategy** ⭐ | Behavioral | Define a family of algorithms, encapsulate each, and make them interchangeable | Strategy interface + Context | `java.util.Comparator`, `Collections.sort()` | • Multiple algorithms for same task<br>• Eliminate conditionals on algorithm selection<br>• Runtime algorithm swap needed | Clients must know about strategies to choose; overhead for trivial variations |
| **Template Method** ⭐ | Behavioral | Define skeleton of algorithm in base class; defer some steps to subclasses | Abstract class with `templateMethod()` + hook methods | `java.io.InputStream.read()`, `AbstractList` | • Invariant algorithm structure with variant steps<br>• Avoid code duplication across subclasses<br>• Framework hook points | Inheritance lock-in; "Hollywood Principle" violations when hooks call back to client |
| **Visitor** | Behavioral | Add operations to object structures without modifying the classes | Visitor interface + `accept(Visitor)` on Element | `javax.lang.model.element.ElementVisitor` | • Many distinct unrelated operations on an object structure<br>• Object structure rarely changes but operations do<br>• Double dispatch needed | Breaking encapsulation (Visitor needs to access internals); adding new element types breaks all visitors |

---

## Pattern Quick-Reference by Category

### Creational — "How objects are created"

| Pattern | 3-Word Description | ⭐ |
|---|---|---|
| Singleton | One global instance | ⭐ |
| Factory Method | Subclass decides type | ⭐ |
| Abstract Factory | Families of objects | ⭐ |
| Builder | Step-by-step construction | ⭐ |
| Prototype | Clone existing object | |

### Structural — "How objects are composed"

| Pattern | 3-Word Description | ⭐ |
|---|---|---|
| Adapter | Wrap incompatible interface | ⭐ |
| Bridge | Decouple abstraction-implementation | |
| Composite | Uniform tree treatment | ⭐ |
| Decorator | Wrap adding behavior | ⭐ |
| Facade | Simplify subsystem interface | ⭐ |
| Flyweight | Share fine-grained objects | |
| Proxy | Controlled surrogate access | ⭐ |

### Behavioral — "How objects communicate"

| Pattern | 3-Word Description | ⭐ |
|---|---|---|
| Chain of Responsibility | Pass-along chain | ⭐ |
| Command | Encapsulate as object | ⭐ |
| Interpreter | Grammar-based evaluation | |
| Iterator | Sequential collection traversal | ⭐ |
| Mediator | Centralize colleague interaction | |
| Memento | Save/restore state | |
| Observer | Notify all dependents | ⭐ |
| State | Behavior-per-state object | ⭐ |
| Strategy | Swappable algorithm family | ⭐ |
| Template Method | Skeleton with hooks | ⭐ |
| Visitor | Operation without modification | |

---

## Pattern Recognition Signals

> If you see **X** in a problem description, think **Pattern Y**.

| Signal in Problem | Pattern | Reasoning |
|---|---|---|
| "Only one instance", "shared config", "global registry" | **Singleton** | Single access point constraint |
| "Create objects without specifying exact class", "plugin system", "product catalog" | **Factory Method / Abstract Factory** | Decouple creation from usage |
| "Undo/redo", "operation history", "transaction rollback" | **Command + Memento** | Command encapsulates op; Memento stores state |
| "Wrap third-party library", "legacy API integration", "impedance mismatch" | **Adapter** | Convert interface to expected one |
| "Add logging/caching/auth transparently", "middleware pipeline" | **Decorator / Proxy** | Decorator = behavior; Proxy = access control |
| "Tree structure", "file system", "org chart", "nested components" | **Composite** | Leaf and branch treated uniformly |
| "Multiple objects might handle a request", "escalation chain", "filter pipeline" | **Chain of Responsibility** | Dynamic handler chain |
| "Notify multiple subscribers on state change", "event bus", "pub-sub" | **Observer** | One-to-many notification |
| "Behavior changes based on current mode/status", "state machine", "workflow" | **State** | Encapsulate state-specific behavior |
| "Choose algorithm at runtime", "pluggable sort/validation/pricing" | **Strategy** | Interchangeable algorithm family |
| "Simplify complex subsystem", "API gateway facade", "service layer" | **Facade** | Unified simplified interface |
| "Reduce memory for many similar objects", "glyph rendering", "particle system" | **Flyweight** | Share intrinsic state |
| "Lazy load", "access control to resource", "remote object" | **Proxy** | Surrogate with controlled access |
| "Invariant algorithm with variant steps", "hook methods", "framework skeleton" | **Template Method** | Defer steps to subclass |

---

## Java Standard Library Pattern Examples

| Pattern | JDK Occurrences |
|---|---|
| **Singleton** | `java.lang.Runtime.getRuntime()`, `java.awt.Desktop.getDesktop()` |
| **Factory Method** | `java.util.Collection.iterator()`, `java.nio.charset.Charset.forName()`, `LogManager.getLogger()` |
| **Abstract Factory** | `javax.xml.parsers.DocumentBuilderFactory`, `javax.xml.transform.TransformerFactory` |
| **Builder** | `java.lang.StringBuilder`, `java.util.Locale.Builder`, `java.util.stream.Stream.Builder` |
| **Prototype** | `java.lang.Object.clone()`, `java.util.ArrayList` (copy constructor) |
| **Adapter** | `java.util.Arrays.asList()`, `java.io.InputStreamReader(InputStream)`, `Collections.list(Enumeration)` |
| **Bridge** | `java.util.logging` (Logger ↔ Handler), JDBC (`Connection` abstraction over driver implementations) |
| **Composite** | `javax.swing.JComponent`, `java.awt.Container`, `java.nio.file.Path` |
| **Decorator** | `java.io.BufferedInputStream`, `java.io.DataOutputStream`, `Collections.unmodifiableList()` |
| **Facade** | `javax.faces.context.FacesContext`, `java.net.URL` (hides URLConnection complexity) |
| **Flyweight** | `Integer.valueOf()` (cache −128..127), `String.intern()`, `java.awt.Font` |
| **Proxy** | `java.lang.reflect.Proxy`, `java.rmi.stub`, Spring AOP `@Transactional` |
| **Chain of Responsibility** | `javax.servlet.Filter`, `java.util.logging.Logger` (parent delegation), `java.awt.AWTEvent` |
| **Command** | `java.lang.Runnable`, `java.util.concurrent.Callable`, `javax.swing.Action` |
| **Interpreter** | `java.util.regex.Pattern`, `java.text.Format.parseObject()` |
| **Iterator** | `java.util.Iterator`, `java.util.Scanner`, `java.nio.file.DirectoryStream` |
| **Mediator** | `java.util.concurrent.ExecutorService` (mediates task submission to workers) |
| **Memento** | `java.io.Serializable`, `javax.swing.undo.UndoManager` |
| **Observer** | `java.util.EventListener`, `java.beans.PropertyChangeListener`, `java.util.Observer` (deprecated) |
| **State** | `javax.faces.lifecycle.Lifecycle`, `java.net.Socket` (state-based behavior) |
| **Strategy** | `java.util.Comparator`, `java.util.concurrent.ThreadPoolExecutor` (rejection policies) |
| **Template Method** | `java.io.InputStream.read(byte[], int, int)`, `java.util.AbstractList`, `javax.servlet.HttpServlet` |
| **Visitor** | `javax.lang.model.element.ElementVisitor`, `java.nio.file.FileVisitor`, ASM bytecode visitors |

---

*Total: 23 GoF patterns. ⭐ marks the 18 most frequently tested in LLD interviews.*
