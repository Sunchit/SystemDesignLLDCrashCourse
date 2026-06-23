# Prototype Pattern

## Table of Contents

1. [Intent](#intent)
2. [When to Use / When NOT to Use](#when-to-use--when-not-to-use)
3. [UML Class Diagram](#uml-class-diagram)
4. [Core Concepts: Shallow vs Deep Copy](#core-concepts-shallow-vs-deep-copy)
5. [Implementation A: Shallow Copy with Cloneable](#implementation-a-shallow-copy-with-cloneable)
6. [Implementation B: Deep Copy - Manual Approach](#implementation-b-deep-copy---manual-approach)
7. [Implementation C: Copy Constructor (Joshua Bloch's Preferred Approach)](#implementation-c-copy-constructor-joshua-blochs-preferred-approach)
8. [Implementation D: Prototype Registry](#implementation-d-prototype-registry)
9. [Implementation E: Game Entity Spawning](#implementation-e-game-entity-spawning)
10. [Implementation F: Document Template System](#implementation-f-document-template-system)
11. [Implementation G: Serialization-Based Deep Copy](#implementation-g-serialization-based-deep-copy)
12. [Java Cloneable: Why It Is Considered Broken](#java-cloneable-why-it-is-considered-broken)
13. [Real-World Analogy](#real-world-analogy)
14. [Real-World Examples in Java Ecosystem](#real-world-examples-in-java-ecosystem)
15. [Trade-offs](#trade-offs)
16. [Interview Questions and Answers](#interview-questions-and-answers)
17. [Comparison with Related Patterns](#comparison-with-related-patterns)
18. [Summary](#summary)

---

## Intent

The **Prototype Pattern** is a creational design pattern that lets you produce new objects by **copying (cloning) an existing object** — called the prototype — rather than creating a brand-new instance from scratch using a constructor.

The core insight is deceptively simple: sometimes you already have an object in a valid, fully-initialized state, and you need several more objects that look just like it (possibly with small variations). Instead of re-running the entire construction process — which might involve database lookups, network calls, expensive computations, or a complex chain of dependencies — you duplicate the ready-made prototype.

### The Central Problem Prototype Solves

Consider the lifecycle of a complex object:

```
new ComplexObject()
    -> reads config from DB
    -> calls external pricing API
    -> loads ML model weights
    -> validates against business rules
    -> sets 47 fields
    -> [200ms later] object is ready
```

If you need 1,000 such objects with minor differences (e.g., different quantities), repeating this process 1,000 times is wasteful. The prototype pattern says: **do the expensive construction once, then clone**.

### Cloning vs Constructing

| Dimension             | Construction                          | Cloning                                  |
|-----------------------|---------------------------------------|------------------------------------------|
| Cost                  | Full initialization cost every time   | Cheap field copy + targeted overrides    |
| State                 | Starts fresh from defaults/args       | Starts from a known good state           |
| Dependencies          | Must know all constructor parameters  | Only need a reference to the prototype   |
| Subtype handling      | Must know the concrete class          | Polymorphic: works through interface     |
| When state is unknown | Cannot create without full info       | Clone captures all state including hidden|

### What "Prototype" Means in Code

A prototype is an object whose sole job — among its normal responsibilities — is to be a template. The pattern formalizes this with a `clone()` method on a `Prototype` interface. Any concrete class that implements this interface declares: "I know how to copy myself."

```
Client --clone()--> Prototype (interface)
                        ^
                        |
               ConcretePrototype
               (knows how to copy itself correctly)
```

---

## When to Use / When NOT to Use

### When to Use

**1. Object construction is expensive.**
If constructing an object requires I/O, network calls, heavy computation, or loading resources (images, ML models, config), cloning a pre-built prototype is far cheaper.

**2. You need many slightly different configurations of the same object.**
A game level might spawn 500 orcs with the same base stats but different positions. Clone a master orc, then update position — do not reconstruct each orc from scratch.

**3. You want to decouple the client from concrete classes.**
The client only knows the `Prototype` interface. It does not need to know whether it is cloning an `OrcEnemy`, `DragonEnemy`, or `GoblinEnemy`. This is especially useful for plugin architectures and frameworks.

**4. The class hierarchy is large and you want to avoid a parallel class hierarchy of factories.**
Abstract Factory requires a factory class per product variant. Prototype avoids that by having each product be its own factory.

**5. Initializing the object requires data known only at runtime.**
Sometimes the state of a "template" object is built up through multiple steps and is not known until runtime. A prototype captures that fully-realized state.

**6. Undo/Redo / Snapshotting.**
Cloning an object's state before an operation gives you a cheap snapshot you can restore.

### When NOT to Use

**1. When objects are cheap to construct.**
If your constructor does nothing more than `this.x = x`, cloning offers zero benefit and adds conceptual complexity.

**2. When objects contain resources that cannot be safely duplicated.**
File handles, database connections, thread instances, and socket connections typically cannot be meaningfully cloned. Cloning such objects would either fail or produce a broken clone.

**3. When circular references make cloning complex.**
Objects with circular references (A -> B -> A) require careful clone logic to avoid infinite loops or double-copies. In these cases, serialization-based deep copy or a different pattern is safer.

**4. When construction guarantees invariants that cloning would bypass.**
If constructors validate complex invariants, cloning might produce objects with those invariants un-enforced. `clone()` does not call constructors.

**5. When all objects must be singletons or unique.**
Cloning is antithetical to singleton semantics.

---

## UML Class Diagram

```
+---------------------------+
|       <<interface>>       |
|         Prototype         |
+---------------------------+
| + clone(): Prototype      |
+---------------------------+
              ^
              |  implements
              |
+---------------------------+        uses          +---------------------------+
|     ConcretePrototype1    |<--------------------+|           Client          |
+---------------------------+                      +---------------------------+
| - field1: String          |                      | - prototype: Prototype    |
| - field2: List<String>    |                      +---------------------------+
| - nested: NestedObject    |                      | + operation(): void       |
+---------------------------+                      |   // calls prototype.clone|
| + clone(): Prototype      |                      +---------------------------+
| + getField1(): String     |
| + setField1(String): void |
+---------------------------+

+---------------------------+
|     ConcretePrototype2    |
+---------------------------+
| - name: String            |
| - config: Map<String,Obj> |
+---------------------------+
| + clone(): Prototype      |
+---------------------------+

+---------------------------+       manages       +---------------------------+
|    PrototypeRegistry      |-------------------->|         Prototype         |
+---------------------------+                      +---------------------------+
| - registry: Map<String,   |
|             Prototype>    |
+---------------------------+
| + register(key, proto)    |
| + get(key): Prototype     |   // returns clone, not original
+---------------------------+
```

**Key relationships:**
- `Prototype` interface declares `clone()` — all concrete prototypes implement it.
- `Client` holds a reference to `Prototype` (the interface), never a concrete type.
- `PrototypeRegistry` is a named store of master prototypes; calling `get()` returns a fresh clone.
- Each `ConcretePrototype` is responsible for copying itself correctly, including nested objects.

---

## Core Concepts: Shallow vs Deep Copy

Before examining implementations, the distinction between shallow and deep copy is fundamental.

```
Original Object:
+-------------------+
| name = "Template" |
| tags = [ref] -----+--------> ["java", "pattern"]  (heap object)
| nested = [ref] ---+--------> NestedObject { x=10 } (heap object)
+-------------------+

Shallow Copy:
+-------------------+
| name = "Template" |   <-- primitive/String: new copy (Strings are immutable)
| tags = [ref] -----+--------> ["java", "pattern"]  SAME heap object!
| nested = [ref] ---+--------> NestedObject { x=10 } SAME heap object!
+-------------------+

Deep Copy:
+-------------------+
| name = "Template" |
| tags = [ref] -----+--------> ["java", "pattern"]  NEW ArrayList (copied)
| nested = [ref] ---+--------> NestedObject { x=10 } NEW object (copied)
+-------------------+
```

With a shallow copy, mutating a `List` field in the clone also mutates it in the original. This is the single most common bug in prototype implementations.

---

## Implementation A: Shallow Copy with Cloneable

### The Code

```java
package designpatterns.prototype.shallow;

import java.util.ArrayList;
import java.util.List;

/**
 * Demonstrates shallow copy using Java's Cloneable interface.
 *
 * WARNING: This implementation has a known pitfall with mutable fields.
 * The shallow copy bug is demonstrated explicitly in ShallowCopyDemo.
 */
public class UserProfile implements Cloneable {

    private String username;
    private String email;
    private int age;
    // Mutable reference type — this is where the bug lurks
    private List<String> roles;
    // Another mutable nested object
    private Address address;

    public UserProfile(String username, String email, int age, List<String> roles, Address address) {
        this.username = username;
        this.email = email;
        this.age = age;
        this.roles = roles;
        this.address = address;
    }

    /**
     * Shallow clone: primitives and Strings are copied by value,
     * but List<String> roles and Address address are copied by reference.
     * The clone shares the same List and Address object as the original.
     */
    @Override
    public UserProfile clone() {
        try {
            // Object.clone() copies all fields bit-for-bit.
            // For primitives: a true copy. For references: copies the pointer, not the object.
            return (UserProfile) super.clone();
        } catch (CloneNotSupportedException e) {
            // This cannot happen since we implement Cloneable,
            // but the checked exception forces this catch block.
            throw new AssertionError("Should never happen", e);
        }
    }

    // Getters and setters
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }

    public List<String> getRoles() { return roles; }
    public void setRoles(List<String> roles) { this.roles = roles; }

    public Address getAddress() { return address; }
    public void setAddress(Address address) { this.address = address; }

    @Override
    public String toString() {
        return String.format(
            "UserProfile{username='%s', email='%s', age=%d, roles=%s, address=%s}",
            username, email, age, roles, address
        );
    }
}
```

```java
package designpatterns.prototype.shallow;

public class Address {
    private String street;
    private String city;
    private String country;

    public Address(String street, String city, String country) {
        this.street = street;
        this.city = city;
        this.country = country;
    }

    public String getStreet() { return street; }
    public void setStreet(String street) { this.street = street; }

    public String getCity() { return city; }
    public void setCity(String city) { this.city = city; }

    public String getCountry() { return country; }
    public void setCountry(String country) { this.country = country; }

    @Override
    public String toString() {
        return String.format("Address{street='%s', city='%s', country='%s'}", street, city, country);
    }
}
```

### The Shallow Copy Bug Demonstrated

```java
package designpatterns.prototype.shallow;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

/**
 * Explicitly demonstrates the shallow copy bug.
 * Run this to see how mutating the clone's List also mutates the original.
 */
public class ShallowCopyBugDemo {

    public static void main(String[] args) {
        // --- Setup ---
        List<String> roles = new ArrayList<>(Arrays.asList("VIEWER", "EDITOR"));
        Address address = new Address("123 Main St", "New York", "USA");
        UserProfile original = new UserProfile("alice", "alice@example.com", 30, roles, address);

        // --- Clone ---
        UserProfile clone = original.clone();

        System.out.println("=== Before mutation ===");
        System.out.println("Original: " + original);
        System.out.println("Clone:    " + clone);
        System.out.println();

        // --- Mutate a primitive/String field on the clone (SAFE) ---
        clone.setUsername("bob");
        clone.setAge(25);
        System.out.println("=== After changing username and age on clone ===");
        System.out.println("Original username: " + original.getUsername()); // alice — unaffected
        System.out.println("Clone username:    " + clone.getUsername());    // bob
        System.out.println();

        // --- BUG: Mutate the shared List through the clone ---
        System.out.println("=== BUG: Adding ADMIN role through clone ===");
        clone.getRoles().add("ADMIN"); // modifies the SHARED list!
        System.out.println("Original roles: " + original.getRoles()); // [VIEWER, EDITOR, ADMIN] — CORRUPTED!
        System.out.println("Clone roles:    " + clone.getRoles());    // [VIEWER, EDITOR, ADMIN]
        System.out.println();

        // --- BUG: Mutate the shared Address through the clone ---
        System.out.println("=== BUG: Changing city through clone's address ===");
        clone.getAddress().setCity("Los Angeles"); // modifies the SHARED address!
        System.out.println("Original address: " + original.getAddress()); // Los Angeles — CORRUPTED!
        System.out.println("Clone address:    " + clone.getAddress());
        System.out.println();

        // --- Verify with reference equality ---
        System.out.println("=== Reference equality check ===");
        System.out.println("Same roles list?   " + (original.getRoles() == clone.getRoles())); // true — same object!
        System.out.println("Same address?      " + (original.getAddress() == clone.getAddress())); // true — same object!
        System.out.println("Same username ref? " + (original.getUsername() == clone.getUsername())); // true — but safe because String is immutable

        System.out.println();
        System.out.println("CONCLUSION: Shallow copy is only safe when all mutable fields are immutable objects.");
        System.out.println("String is immutable, so sharing its reference is fine.");
        System.out.println("ArrayList and Address are mutable, so sharing their references is a BUG.");
    }
}
```

### Explanation of the Pitfall

When `super.clone()` copies the `UserProfile`, it copies the `roles` field as a reference. Both original and clone now point to the **exact same `ArrayList` object** in memory. Adding `"ADMIN"` through the clone modifies this shared list, which is visible from the original.

The same applies to `address`. The clone's `address` field points to the same `Address` object. Any mutation through the clone is reflected in the original.

**The rule**: Shallow copy is only safe if all mutable fields hold immutable objects (e.g., `String`, `Integer`, unmodifiable collections). If any mutable field holds a mutable object, you need a deep copy.

---

## Implementation B: Deep Copy - Manual Approach

```java
package designpatterns.prototype.deep;

import java.util.ArrayList;
import java.util.List;

/**
 * Deep copy implementation: manually copy all nested mutable objects.
 * This is the correct approach when using Cloneable.
 */
public class UserProfile implements Cloneable {

    private String username;
    private String email;
    private int age;
    private List<String> roles;       // mutable — must be deep-copied
    private Address address;          // mutable — must be deep-copied

    public UserProfile(String username, String email, int age, List<String> roles, Address address) {
        this.username = username;
        this.email = email;
        this.age = age;
        // Defensive copy on construction too — good practice
        this.roles = new ArrayList<>(roles);
        this.address = new Address(address.getStreet(), address.getCity(), address.getCountry());
    }

    /**
     * Deep clone: we call super.clone() for primitives and Strings,
     * then manually deep-copy every mutable reference field.
     */
    @Override
    public UserProfile clone() {
        try {
            UserProfile cloned = (UserProfile) super.clone();

            // Deep copy the mutable List — create a new ArrayList with the same elements
            // String elements inside are immutable, so a shallow copy of them is fine
            cloned.roles = new ArrayList<>(this.roles);

            // Deep copy the mutable Address
            cloned.address = this.address.clone(); // Address also implements Cloneable

            return cloned;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError("Should never happen", e);
        }
    }

    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
    public List<String> getRoles() { return roles; }
    public Address getAddress() { return address; }

    @Override
    public String toString() {
        return String.format(
            "UserProfile{username='%s', email='%s', age=%d, roles=%s, address=%s}",
            username, email, age, roles, address
        );
    }
}
```

```java
package designpatterns.prototype.deep;

/**
 * Address also implements Cloneable so UserProfile.clone() can deep-copy it.
 */
public class Address implements Cloneable {
    private String street;
    private String city;
    private String country;

    public Address(String street, String city, String country) {
        this.street = street;
        this.city = city;
        this.country = country;
    }

    @Override
    public Address clone() {
        try {
            return (Address) super.clone(); // All fields are Strings (immutable) — shallow is deep here
        } catch (CloneNotSupportedException e) {
            throw new AssertionError("Should never happen", e);
        }
    }

    public String getStreet() { return street; }
    public void setStreet(String street) { this.street = street; }
    public String getCity() { return city; }
    public void setCity(String city) { this.city = city; }
    public String getCountry() { return country; }
    public void setCountry(String country) { this.country = country; }

    @Override
    public String toString() {
        return String.format("Address{street='%s', city='%s', country='%s'}", street, city, country);
    }
}
```

```java
package designpatterns.prototype.deep;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

/**
 * Demonstrates that deep copy correctly isolates the clone from the original.
 */
public class DeepCopyDemo {

    public static void main(String[] args) {
        List<String> roles = new ArrayList<>(Arrays.asList("VIEWER", "EDITOR"));
        Address address = new Address("123 Main St", "New York", "USA");
        UserProfile original = new UserProfile("alice", "alice@example.com", 30, roles, address);

        UserProfile clone = original.clone();

        System.out.println("=== Before mutation ===");
        System.out.println("Original: " + original);
        System.out.println("Clone:    " + clone);

        // Mutate clone's roles
        clone.getRoles().add("ADMIN");
        clone.getAddress().setCity("Los Angeles");

        System.out.println("\n=== After mutating clone ===");
        System.out.println("Original roles:   " + original.getRoles());   // [VIEWER, EDITOR] — unchanged!
        System.out.println("Clone roles:      " + clone.getRoles());      // [VIEWER, EDITOR, ADMIN]
        System.out.println("Original address: " + original.getAddress()); // New York — unchanged!
        System.out.println("Clone address:    " + clone.getAddress());    // Los Angeles

        System.out.println("\n=== Reference equality check ===");
        System.out.println("Same roles list? " + (original.getRoles() == clone.getRoles())); // false
        System.out.println("Same address?    " + (original.getAddress() == clone.getAddress())); // false

        System.out.println("\nSUCCESS: Deep copy correctly isolates clone from original.");
    }
}
```

---

## Implementation C: Copy Constructor (Joshua Bloch's Preferred Approach)

Joshua Bloch, in *Effective Java* (Item 13), argues strongly against `Cloneable` and `clone()` and recommends **copy constructors** or **copy factory methods** instead.

### Why Copy Constructors Are Superior

A copy constructor is simply a constructor that takes an instance of the same class as its argument and copies all fields:

```java
public MyClass(MyClass other) {
    this.field1 = other.field1;
    this.mutableList = new ArrayList<>(other.mutableList);
    // ... etc
}
```

Benefits over `Cloneable`:
- It is a normal constructor — no `CloneNotSupportedException`, no casting, no `super.clone()` dance.
- It is explicit and readable: you can see exactly what is being copied.
- It respects constructors — invariants enforced in the constructor are always applied.
- Final fields work correctly (clone() cannot assign to final fields after `super.clone()`).
- It can accept an interface type, enabling cross-type conversion (e.g., constructing a `TreeSet` from a `HashSet`).

```java
package designpatterns.prototype.copyconstructor;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

/**
 * Production-quality UserProfile using copy constructor — Joshua Bloch's recommended approach.
 * No Cloneable, no CloneNotSupportedException, no casting.
 */
public class UserProfile {

    private final String username;      // immutable field — can be final
    private String email;
    private int age;
    private List<String> roles;         // mutable — must be deep-copied
    private Address address;            // mutable — must be deep-copied

    /**
     * Primary constructor — the only place where invariants are checked.
     */
    public UserProfile(String username, String email, int age, List<String> roles, Address address) {
        if (username == null || username.isBlank()) {
            throw new IllegalArgumentException("Username must not be blank");
        }
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Age must be between 0 and 150");
        }
        this.username = username;
        this.email = email;
        this.age = age;
        // Defensive copy: we own our list, caller cannot mutate it externally
        this.roles = new ArrayList<>(roles != null ? roles : Collections.emptyList());
        this.address = address != null ? new Address(address) : null;
    }

    /**
     * Copy constructor — the Prototype Pattern without Cloneable.
     * Explicitly deep-copies all mutable fields.
     * Invariants are enforced because this IS a constructor.
     */
    public UserProfile(UserProfile other) {
        // Delegates to the primary constructor — invariants always checked
        this(
            other.username,
            other.email,
            other.age,
            other.roles,         // primary constructor will defensively copy this
            other.address        // primary constructor will defensively copy this
        );
    }

    /**
     * Static copy factory method — an alternative to copy constructor.
     * Bloch often prefers this as it has a descriptive name.
     */
    public static UserProfile copyOf(UserProfile other) {
        return new UserProfile(other);
    }

    /**
     * Returns a copy with a different username — "wither" pattern.
     * Useful for creating variants without modifying the original.
     */
    public UserProfile withUsername(String newUsername) {
        UserProfile copy = new UserProfile(this);
        // We'd need a builder here for a truly immutable approach,
        // but this illustrates the concept.
        return new UserProfile(newUsername, this.email, this.age, this.roles, this.address);
    }

    public String getUsername() { return username; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }

    // Return defensive copy — callers cannot mutate our internal list
    public List<String> getRoles() { return Collections.unmodifiableList(roles); }

    public void addRole(String role) {
        this.roles.add(role);
    }

    public Address getAddress() { return address != null ? new Address(address) : null; }

    @Override
    public String toString() {
        return String.format(
            "UserProfile{username='%s', email='%s', age=%d, roles=%s, address=%s}",
            username, email, age, roles, address
        );
    }
}
```

```java
package designpatterns.prototype.copyconstructor;

/**
 * Address with a copy constructor — no Cloneable needed.
 */
public class Address {
    private final String street;
    private final String city;
    private final String country;

    public Address(String street, String city, String country) {
        this.street = street;
        this.city = city;
        this.country = country;
    }

    // Copy constructor
    public Address(Address other) {
        this(other.street, other.city, other.country);
    }

    public static Address copyOf(Address other) {
        return new Address(other);
    }

    public String getStreet() { return street; }
    public String getCity() { return city; }
    public String getCountry() { return country; }

    @Override
    public String toString() {
        return String.format("Address{street='%s', city='%s', country='%s'}", street, city, country);
    }
}
```

```java
package designpatterns.prototype.copyconstructor;

import java.util.Arrays;

public class CopyConstructorDemo {

    public static void main(String[] args) {
        UserProfile original = new UserProfile(
            "alice",
            "alice@example.com",
            30,
            Arrays.asList("VIEWER", "EDITOR"),
            new Address("123 Main St", "New York", "USA")
        );

        // Copy using copy constructor
        UserProfile copy1 = new UserProfile(original);

        // Copy using static factory method
        UserProfile copy2 = UserProfile.copyOf(original);

        // Variant using wither
        UserProfile bobVersion = original.withUsername("bob");

        System.out.println("Original: " + original);
        System.out.println("Copy1:    " + copy1);
        System.out.println("Copy2:    " + copy2);
        System.out.println("Bob ver:  " + bobVersion);

        // Mutate copy1 — original is unaffected
        copy1.addRole("ADMIN");
        copy1.setEmail("alice-copy@example.com");

        System.out.println("\nAfter mutating copy1:");
        System.out.println("Original roles: " + original.getRoles()); // [VIEWER, EDITOR]
        System.out.println("Copy1 roles:    " + copy1.getRoles());    // [VIEWER, EDITOR, ADMIN]

        // Invalid argument test — invariants enforced by constructor
        try {
            UserProfile invalid = new UserProfile("", "e@e.com", 25, null, null);
        } catch (IllegalArgumentException e) {
            System.out.println("\nInvariant enforced: " + e.getMessage());
        }
    }
}
```

---

## Implementation D: Prototype Registry

A `PrototypeRegistry` (also called a Prototype Manager) is a central store that holds pre-configured prototype objects keyed by name. Clients request a named prototype and receive a fresh clone — they never get the master prototype itself.

```java
package designpatterns.prototype.registry;

/**
 * The Prototype interface. Every cloneable entity implements this.
 */
public interface Prototype {
    /**
     * Returns a deep copy of this object.
     * The return type is covariant — implementations may return their own type.
     */
    Prototype clone();

    /**
     * Returns a human-readable type identifier for logging/debugging.
     */
    String getPrototypeName();
}
```

```java
package designpatterns.prototype.registry;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

/**
 * Registry that stores named Prototype instances and returns clones on demand.
 *
 * Thread-safe variant: use ConcurrentHashMap and volatile references if needed.
 * This implementation is not thread-safe by design (simpler, suitable for single-threaded use).
 */
public class PrototypeRegistry {

    private final Map<String, Prototype> registry = new HashMap<>();

    /**
     * Register a prototype under a given key.
     * The registry stores the prototype itself — clients receive clones.
     */
    public void register(String key, Prototype prototype) {
        if (key == null || key.isBlank()) {
            throw new IllegalArgumentException("Key must not be blank");
        }
        if (prototype == null) {
            throw new IllegalArgumentException("Prototype must not be null");
        }
        registry.put(key, prototype);
    }

    /**
     * Remove a prototype from the registry.
     */
    public void unregister(String key) {
        registry.remove(key);
    }

    /**
     * Retrieve a clone of the registered prototype.
     * Always returns a new clone — the master prototype is never exposed.
     *
     * @param key the registered name
     * @return a fresh clone of the prototype
     * @throws IllegalArgumentException if no prototype is registered under that key
     */
    public Prototype get(String key) {
        return Optional.ofNullable(registry.get(key))
            .map(Prototype::clone)
            .orElseThrow(() -> new IllegalArgumentException(
                "No prototype registered for key: '" + key + "'"
            ));
    }

    /**
     * Convenience method that casts the clone to the expected type.
     * Safe when the caller knows the type registered under the key.
     */
    @SuppressWarnings("unchecked")
    public <T extends Prototype> T get(String key, Class<T> type) {
        Prototype clone = get(key);
        if (!type.isInstance(clone)) {
            throw new ClassCastException(
                "Prototype '" + key + "' is " + clone.getClass().getName() +
                ", not " + type.getName()
            );
        }
        return (T) clone;
    }

    public boolean isRegistered(String key) {
        return registry.containsKey(key);
    }

    public int size() {
        return registry.size();
    }

    @Override
    public String toString() {
        return "PrototypeRegistry{keys=" + registry.keySet() + "}";
    }
}
```

```java
package designpatterns.prototype.registry;

import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

/**
 * A concrete prototype: a configurable notification template.
 */
public class NotificationTemplate implements Prototype {

    private String templateName;
    private String subject;
    private String bodyHtml;
    private List<String> recipients;
    private boolean sendImmediately;

    public NotificationTemplate(String templateName, String subject, String bodyHtml,
                                 List<String> recipients, boolean sendImmediately) {
        this.templateName = templateName;
        this.subject = subject;
        this.bodyHtml = bodyHtml;
        this.recipients = new ArrayList<>(recipients);
        this.sendImmediately = sendImmediately;
    }

    // Copy constructor — used by clone()
    private NotificationTemplate(NotificationTemplate other) {
        this.templateName = other.templateName;
        this.subject = other.subject;
        this.bodyHtml = other.bodyHtml;
        this.recipients = new ArrayList<>(other.recipients);
        this.sendImmediately = other.sendImmediately;
    }

    @Override
    public NotificationTemplate clone() {
        return new NotificationTemplate(this); // delegate to copy constructor
    }

    @Override
    public String getPrototypeName() {
        return "NotificationTemplate:" + templateName;
    }

    // Fluent setters for customizing the clone after creation
    public NotificationTemplate withSubject(String subject) {
        this.subject = subject;
        return this;
    }

    public NotificationTemplate withRecipients(List<String> recipients) {
        this.recipients = new ArrayList<>(recipients);
        return this;
    }

    public NotificationTemplate addRecipient(String recipient) {
        this.recipients.add(recipient);
        return this;
    }

    public String getTemplateName() { return templateName; }
    public String getSubject() { return subject; }
    public String getBodyHtml() { return bodyHtml; }
    public List<String> getRecipients() { return new ArrayList<>(recipients); }
    public boolean isSendImmediately() { return sendImmediately; }

    @Override
    public String toString() {
        return String.format(
            "NotificationTemplate{name='%s', subject='%s', recipients=%s, immediate=%b}",
            templateName, subject, recipients, sendImmediately
        );
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof NotificationTemplate)) return false;
        NotificationTemplate that = (NotificationTemplate) o;
        return sendImmediately == that.sendImmediately
            && Objects.equals(templateName, that.templateName)
            && Objects.equals(subject, that.subject)
            && Objects.equals(bodyHtml, that.bodyHtml)
            && Objects.equals(recipients, that.recipients);
    }

    @Override
    public int hashCode() {
        return Objects.hash(templateName, subject, bodyHtml, recipients, sendImmediately);
    }
}
```

```java
package designpatterns.prototype.registry;

import java.util.Arrays;
import java.util.Collections;

/**
 * Demonstrates the PrototypeRegistry in a notification system context.
 */
public class RegistryDemo {

    public static void main(String[] args) {
        // --- Build the registry with master templates ---
        PrototypeRegistry registry = new PrototypeRegistry();

        NotificationTemplate welcomeTemplate = new NotificationTemplate(
            "WELCOME",
            "Welcome to Our Platform!",
            "<h1>Welcome, {{username}}!</h1><p>Click <a href='{{link}}'>here</a> to get started.</p>",
            Collections.emptyList(),
            true
        );

        NotificationTemplate passwordResetTemplate = new NotificationTemplate(
            "PASSWORD_RESET",
            "Password Reset Request",
            "<p>Click <a href='{{resetLink}}'>here</a> to reset your password. Link expires in 1 hour.</p>",
            Collections.emptyList(),
            true
        );

        NotificationTemplate weeklyDigestTemplate = new NotificationTemplate(
            "WEEKLY_DIGEST",
            "Your Weekly Summary",
            "<h2>Here's what happened this week...</h2>{{content}}",
            Arrays.asList("digest-team@example.com"),
            false
        );

        registry.register("welcome", welcomeTemplate);
        registry.register("password-reset", passwordResetTemplate);
        registry.register("weekly-digest", weeklyDigestTemplate);

        System.out.println("Registry: " + registry);
        System.out.println();

        // --- Client code: get clones and customize ---
        // New user Alice signs up
        NotificationTemplate aliceWelcome = registry.get("welcome", NotificationTemplate.class)
            .addRecipient("alice@example.com")
            .withSubject("Welcome, Alice!");

        // New user Bob signs up
        NotificationTemplate bobWelcome = registry.get("welcome", NotificationTemplate.class)
            .addRecipient("bob@example.com")
            .withSubject("Welcome, Bob!");

        // Password reset for Charlie
        NotificationTemplate charlieReset = registry.get("password-reset", NotificationTemplate.class)
            .addRecipient("charlie@example.com");

        System.out.println("Alice's welcome: " + aliceWelcome);
        System.out.println("Bob's welcome:   " + bobWelcome);
        System.out.println("Charlie's reset: " + charlieReset);

        // Verify master templates are unchanged
        System.out.println("\n--- Master template recipients (should be empty) ---");
        System.out.println("Welcome template recipients: " + welcomeTemplate.getRecipients()); // []

        // Verify Alice and Bob got independent clones
        System.out.println("\n--- Alice and Bob have separate recipient lists ---");
        System.out.println("Alice recipients: " + aliceWelcome.getRecipients()); // [alice@...]
        System.out.println("Bob recipients:   " + bobWelcome.getRecipients());   // [bob@...]

        // Verify the clones are not the same object
        NotificationTemplate clone1 = registry.get("welcome", NotificationTemplate.class);
        NotificationTemplate clone2 = registry.get("welcome", NotificationTemplate.class);
        System.out.println("\nclone1 == clone2? " + (clone1 == clone2));                 // false
        System.out.println("clone1.equals(clone2)? " + clone1.equals(clone2));           // true (same content)
    }
}
```

---

## Implementation E: Game Entity Spawning

This example demonstrates the performance argument for Prototype: spawning many enemies from a prototype is dramatically faster than constructing each from scratch when construction involves expensive operations.

```java
package designpatterns.prototype.game;

/**
 * Represents a 2D position in the game world.
 */
public class Position {
    private float x;
    private float y;

    public Position(float x, float y) {
        this.x = x;
        this.y = y;
    }

    public Position(Position other) {
        this.x = other.x;
        this.y = other.y;
    }

    public float getX() { return x; }
    public float getY() { return y; }
    public void setX(float x) { this.x = x; }
    public void setY(float y) { this.y = y; }

    @Override
    public String toString() {
        return String.format("Position(%.1f, %.1f)", x, y);
    }
}
```

```java
package designpatterns.prototype.game;

import java.util.List;

/**
 * Abstract base enemy class. All enemies are prototypes.
 * Subclasses define the clone() return type covariantly.
 *
 * In a real game engine, construction would load sprites, audio clips,
 * animation states, and AI behavior trees from disk — all expensive operations.
 */
public abstract class Enemy {

    protected String name;
    protected int health;
    protected int maxHealth;
    protected int damage;
    protected float speed;
    protected Position position;
    protected List<String> abilities;   // e.g., ["POISON_STRIKE", "SHIELD_BASH"]
    protected int experienceReward;

    // Simulated expensive construction — in reality this would load assets
    protected Enemy(String name, int health, int damage, float speed,
                    List<String> abilities, int experienceReward) {
        this.name = name;
        this.health = health;
        this.maxHealth = health;
        this.damage = damage;
        this.speed = speed;
        this.abilities = abilities;
        this.experienceReward = experienceReward;
        this.position = new Position(0, 0);

        simulateExpensiveAssetLoading();
    }

    // Copy constructor for subclass use
    protected Enemy(Enemy other) {
        this.name = other.name;
        this.health = other.health;
        this.maxHealth = other.maxHealth;
        this.damage = other.damage;
        this.speed = other.speed;
        this.abilities = new java.util.ArrayList<>(other.abilities);
        this.experienceReward = other.experienceReward;
        this.position = new Position(other.position);
        // Note: NO simulateExpensiveAssetLoading() called here — that's the whole point
    }

    private void simulateExpensiveAssetLoading() {
        try {
            // In a real engine: load sprite sheets, audio, compile shader programs, etc.
            Thread.sleep(50); // Simulate 50ms of asset loading per enemy
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    /**
     * Each concrete enemy knows how to clone itself correctly.
     * Return type is covariant — callers get the specific type, not just Enemy.
     */
    public abstract Enemy clone();

    public Enemy spawnAt(float x, float y) {
        Enemy spawned = this.clone();
        spawned.position = new Position(x, y);
        spawned.health = spawned.maxHealth; // Reset HP for new spawn
        return spawned;
    }

    public void takeDamage(int amount) {
        this.health = Math.max(0, this.health - amount);
    }

    public boolean isAlive() { return health > 0; }

    public String getName() { return name; }
    public int getHealth() { return health; }
    public int getMaxHealth() { return maxHealth; }
    public int getDamage() { return damage; }
    public float getSpeed() { return speed; }
    public Position getPosition() { return position; }
    public List<String> getAbilities() { return java.util.Collections.unmodifiableList(abilities); }
    public int getExperienceReward() { return experienceReward; }

    @Override
    public String toString() {
        return String.format(
            "%s{hp=%d/%d, dmg=%d, speed=%.1f, pos=%s, abilities=%s, xp=%d}",
            name, health, maxHealth, damage, speed, position, abilities, experienceReward
        );
    }
}
```

```java
package designpatterns.prototype.game;

import java.util.Arrays;

/**
 * OrcEnemy — a melee fighter with moderate stats.
 */
public class OrcEnemy extends Enemy {

    private String weaponType;
    private int armor;

    public OrcEnemy(String weaponType, int armor) {
        super(
            "Orc Warrior",
            120,                                   // health
            35,                                    // damage
            2.5f,                                  // speed
            Arrays.asList("SHIELD_BASH", "BATTLE_CRY"), // abilities
            80                                     // xp reward
        );
        this.weaponType = weaponType;
        this.armor = armor;
    }

    // Copy constructor — called by clone()
    private OrcEnemy(OrcEnemy other) {
        super(other);                  // copies all Enemy fields efficiently
        this.weaponType = other.weaponType;
        this.armor = other.armor;
        // No asset loading — that's the performance win
    }

    @Override
    public OrcEnemy clone() {
        return new OrcEnemy(this);
    }

    public String getWeaponType() { return weaponType; }
    public int getArmor() { return armor; }

    @Override
    public String toString() {
        return super.toString() + String.format("[weapon=%s, armor=%d]", weaponType, armor);
    }
}
```

```java
package designpatterns.prototype.game;

import java.util.Arrays;

/**
 * DragonEnemy — a powerful boss-type enemy with flying and fire breath.
 */
public class DragonEnemy extends Enemy {

    private int fireBreathDamage;
    private int flightAltitude;
    private boolean isAncient;

    public DragonEnemy(int fireBreathDamage, boolean isAncient) {
        super(
            isAncient ? "Ancient Dragon" : "Young Dragon",
            isAncient ? 5000 : 2000,               // health
            isAncient ? 200 : 80,                  // damage
            isAncient ? 4.0f : 6.0f,               // speed
            Arrays.asList("FIRE_BREATH", "WING_BUFFET", "TAIL_SWEEP"), // abilities
            isAncient ? 5000 : 1500                // xp reward
        );
        this.fireBreathDamage = fireBreathDamage;
        this.flightAltitude = 50;
        this.isAncient = isAncient;
    }

    private DragonEnemy(DragonEnemy other) {
        super(other);
        this.fireBreathDamage = other.fireBreathDamage;
        this.flightAltitude = other.flightAltitude;
        this.isAncient = other.isAncient;
    }

    @Override
    public DragonEnemy clone() {
        return new DragonEnemy(this);
    }

    public int getFireBreathDamage() { return fireBreathDamage; }
    public int getFlightAltitude() { return flightAltitude; }
    public boolean isAncient() { return isAncient; }

    @Override
    public String toString() {
        return super.toString() +
            String.format("[fireDmg=%d, alt=%d, ancient=%b]", fireBreathDamage, flightAltitude, isAncient);
    }
}
```

```java
package designpatterns.prototype.game;

import java.util.ArrayList;
import java.util.List;

/**
 * Demonstrates the performance difference between constructing enemies from scratch
 * vs spawning them from a prototype.
 *
 * A game level might need 500+ enemies. If each costs 50ms to construct, that's 25 seconds
 * of loading time. Using prototypes, the 50ms happens once and spawning is near-instant.
 */
public class GameEntitySpawningDemo {

    private static final int SPAWN_COUNT = 20;

    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Game Entity Spawning Performance Demo ===");
        System.out.println("Spawning " + SPAWN_COUNT + " enemies each way...\n");

        // --- Method 1: Construct each enemy from scratch ---
        System.out.println("--- Method 1: Construction from scratch ---");
        long startConstruct = System.currentTimeMillis();

        List<Enemy> constructedEnemies = new ArrayList<>();
        for (int i = 0; i < SPAWN_COUNT; i++) {
            OrcEnemy orc = new OrcEnemy("Axe", 20); // Each call loads assets (50ms simulated)
            orc.spawnAt(i * 10f, 0f);
            constructedEnemies.add(orc);
        }

        long constructTime = System.currentTimeMillis() - startConstruct;
        System.out.println("Time to construct " + SPAWN_COUNT + " orcs: " + constructTime + "ms");

        // --- Method 2: Spawn from prototype ---
        System.out.println("\n--- Method 2: Spawn from prototype ---");

        // One-time asset loading: create the master prototype
        System.out.println("Creating master prototype (one-time cost)...");
        long protoCreateStart = System.currentTimeMillis();
        OrcEnemy masterOrc = new OrcEnemy("Axe", 20); // Asset loading happens ONCE
        long protoCreateTime = System.currentTimeMillis() - protoCreateStart;
        System.out.println("Master prototype created in: " + protoCreateTime + "ms");

        // Spawning from prototype — no asset loading
        long startSpawn = System.currentTimeMillis();
        List<Enemy> spawnedEnemies = new ArrayList<>();
        for (int i = 0; i < SPAWN_COUNT; i++) {
            Enemy spawned = masterOrc.spawnAt(i * 10f, 0f); // clone() — no asset loading
            spawnedEnemies.add(spawned);
        }
        long spawnTime = System.currentTimeMillis() - startSpawn;
        System.out.println("Time to spawn " + SPAWN_COUNT + " orcs from prototype: " + spawnTime + "ms");

        System.out.println("\n--- Summary ---");
        System.out.println("Construction: " + constructTime + "ms");
        System.out.println("Prototype spawning: " + protoCreateTime + "ms (setup) + " + spawnTime + "ms (spawning)");
        System.out.printf("Speed improvement for spawning phase: %.1fx%n",
            (double) constructTime / spawnTime);

        // --- Verify independence: spawned enemies are independent ---
        System.out.println("\n--- Verification: each spawned enemy is independent ---");
        Enemy firstSpawn = spawnedEnemies.get(0);
        Enemy secondSpawn = spawnedEnemies.get(1);

        firstSpawn.takeDamage(50);
        System.out.println("First orc HP after 50 damage: " + firstSpawn.getHealth());  // 70
        System.out.println("Second orc HP (untouched):    " + secondSpawn.getHealth()); // 120
        System.out.println("Master prototype HP:          " + masterOrc.getHealth());   // 120

        // --- Mixed enemy types from a registry ---
        System.out.println("\n--- Mixed enemy level with registry ---");
        EnemyRegistry enemyRegistry = new EnemyRegistry();
        enemyRegistry.registerMaster("orc", new OrcEnemy("Sword", 15));
        enemyRegistry.registerMaster("young-dragon", new DragonEnemy(120, false));
        enemyRegistry.registerMaster("ancient-dragon", new DragonEnemy(300, true));

        float[][] spawnPoints = {{0,0},{100,50},{200,0},{50,100},{150,100}};
        String[] enemyTypes = {"orc","orc","young-dragon","orc","ancient-dragon"};

        System.out.println("Spawning level enemies:");
        for (int i = 0; i < spawnPoints.length; i++) {
            Enemy e = enemyRegistry.spawn(enemyTypes[i], spawnPoints[i][0], spawnPoints[i][1]);
            System.out.println("  Spawned: " + e.getName() + " at " + e.getPosition());
        }
    }
}
```

```java
package designpatterns.prototype.game;

import java.util.HashMap;
import java.util.Map;

/**
 * Registry for master enemy prototypes.
 */
public class EnemyRegistry {

    private final Map<String, Enemy> masterPrototypes = new HashMap<>();

    public void registerMaster(String key, Enemy master) {
        masterPrototypes.put(key, master);
    }

    public Enemy spawn(String key, float x, float y) {
        Enemy master = masterPrototypes.get(key);
        if (master == null) {
            throw new IllegalArgumentException("No enemy prototype registered for: " + key);
        }
        return master.spawnAt(x, y);
    }
}
```

---

## Implementation F: Document Template System

This example shows a document management system where users create new documents by cloning templates, then customizing the clone.

```java
package designpatterns.prototype.document;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;
import java.util.UUID;

/**
 * A Section within a document — mutable, so it needs to be deep-copied.
 */
public class DocumentSection {
    private String title;
    private String content;
    private String style; // e.g., "HEADING_1", "BODY", "BULLET_LIST"

    public DocumentSection(String title, String content, String style) {
        this.title = title;
        this.content = content;
        this.style = style;
    }

    // Copy constructor
    public DocumentSection(DocumentSection other) {
        this.title = other.title;
        this.content = other.content;
        this.style = other.style;
    }

    public static DocumentSection copyOf(DocumentSection other) {
        return new DocumentSection(other);
    }

    public String getTitle() { return title; }
    public String getContent() { return content; }
    public String getStyle() { return style; }
    public void setTitle(String title) { this.title = title; }
    public void setContent(String content) { this.content = content; }

    @Override
    public String toString() {
        return String.format("[%s] %s: %.40s...", style, title, content);
    }
}
```

```java
package designpatterns.prototype.document;

/**
 * Metadata associated with a document.
 */
public class DocumentMetadata {
    private String author;
    private String department;
    private String version;
    private String classification; // e.g., "PUBLIC", "INTERNAL", "CONFIDENTIAL"

    public DocumentMetadata(String author, String department, String version, String classification) {
        this.author = author;
        this.department = department;
        this.version = version;
        this.classification = classification;
    }

    // Copy constructor
    public DocumentMetadata(DocumentMetadata other) {
        this.author = other.author;
        this.department = other.department;
        this.version = other.version;
        this.classification = other.classification;
    }

    public String getAuthor() { return author; }
    public void setAuthor(String author) { this.author = author; }
    public String getDepartment() { return department; }
    public void setDepartment(String department) { this.department = department; }
    public String getVersion() { return version; }
    public void setVersion(String version) { this.version = version; }
    public String getClassification() { return classification; }

    @Override
    public String toString() {
        return String.format("Metadata{author='%s', dept='%s', v='%s', class='%s'}",
            author, department, version, classification);
    }
}
```

```java
package designpatterns.prototype.document;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

/**
 * DocumentTemplate: a Prototype that can be cloned into new document instances.
 *
 * Design decisions:
 * - clone() produces a clean copy with a new UUID, reset timestamps, and a "Draft" status.
 * - The template itself retains its original metadata.
 * - Deep copy via copy constructors — no Cloneable, no casting.
 */
public class DocumentTemplate {

    private final String templateId;     // immutable — each instance has a unique ID
    private String title;
    private String documentType;         // e.g., "REPORT", "PROPOSAL", "MEETING_MINUTES"
    private List<DocumentSection> sections;
    private DocumentMetadata metadata;
    private String status;               // "TEMPLATE", "DRAFT", "REVIEW", "PUBLISHED"
    private LocalDateTime createdAt;
    private LocalDateTime lastModifiedAt;
    private String pageSize;             // e.g., "A4", "LETTER"
    private String fontFamily;

    /**
     * Primary constructor for creating a new template from scratch.
     */
    public DocumentTemplate(String title, String documentType, DocumentMetadata metadata,
                             String pageSize, String fontFamily) {
        this.templateId = UUID.randomUUID().toString();
        this.title = title;
        this.documentType = documentType;
        this.metadata = new DocumentMetadata(metadata);
        this.sections = new ArrayList<>();
        this.status = "TEMPLATE";
        this.createdAt = LocalDateTime.now();
        this.lastModifiedAt = LocalDateTime.now();
        this.pageSize = pageSize;
        this.fontFamily = fontFamily;
    }

    /**
     * Copy constructor — creates a deep copy, resetting instance-specific state.
     * The clone gets: a new UUID, reset timestamps, "DRAFT" status, empty author.
     * The clone inherits: structure, sections, formatting, document type.
     */
    public DocumentTemplate(DocumentTemplate other) {
        this.templateId = UUID.randomUUID().toString(); // New document = new ID
        this.title = "Copy of " + other.title;
        this.documentType = other.documentType;
        this.status = "DRAFT";                          // Clone starts as a draft
        this.createdAt = LocalDateTime.now();
        this.lastModifiedAt = LocalDateTime.now();
        this.pageSize = other.pageSize;
        this.fontFamily = other.fontFamily;

        // Deep copy sections
        this.sections = new ArrayList<>();
        for (DocumentSection section : other.sections) {
            this.sections.add(new DocumentSection(section));
        }

        // Deep copy metadata, but reset author (the new document has a new author)
        this.metadata = new DocumentMetadata(other.metadata);
        this.metadata.setAuthor(""); // New document needs a new author
        this.metadata.setVersion("1.0");
    }

    /**
     * Clone factory method — produces a fully customizable new document from this template.
     */
    public DocumentTemplate clone() {
        return new DocumentTemplate(this);
    }

    /**
     * Convenience: clone and immediately set the new author.
     */
    public DocumentTemplate cloneForAuthor(String author) {
        DocumentTemplate doc = new DocumentTemplate(this);
        doc.metadata.setAuthor(author);
        doc.title = this.title + " - " + author;
        return doc;
    }

    public DocumentTemplate addSection(DocumentSection section) {
        this.sections.add(new DocumentSection(section)); // defensive copy
        this.lastModifiedAt = LocalDateTime.now();
        return this;
    }

    public DocumentTemplate addSection(String title, String content, String style) {
        return addSection(new DocumentSection(title, content, style));
    }

    public void publish() {
        if ("DRAFT".equals(status) || "REVIEW".equals(status)) {
            this.status = "PUBLISHED";
            this.lastModifiedAt = LocalDateTime.now();
        } else {
            throw new IllegalStateException("Can only publish DRAFT or REVIEW documents, current status: " + status);
        }
    }

    public String getTemplateId() { return templateId; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getDocumentType() { return documentType; }
    public List<DocumentSection> getSections() { return Collections.unmodifiableList(sections); }
    public DocumentMetadata getMetadata() { return metadata; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getLastModifiedAt() { return lastModifiedAt; }
    public String getPageSize() { return pageSize; }
    public String getFontFamily() { return fontFamily; }

    @Override
    public String toString() {
        DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        return String.format(
            "Document{id='%s...', title='%s', type=%s, status=%s, sections=%d, %s, created=%s}",
            templateId.substring(0, 8), title, documentType, status,
            sections.size(), metadata, createdAt.format(fmt)
        );
    }
}
```

```java
package designpatterns.prototype.document;

import java.util.HashMap;
import java.util.Map;

/**
 * Manages a library of document templates. Clients clone templates to create new documents.
 */
public class DocumentTemplateLibrary {

    private final Map<String, DocumentTemplate> templates = new HashMap<>();

    public void addTemplate(String key, DocumentTemplate template) {
        templates.put(key, template);
    }

    public DocumentTemplate createDocument(String templateKey) {
        DocumentTemplate template = templates.get(templateKey);
        if (template == null) {
            throw new IllegalArgumentException("Template not found: " + templateKey);
        }
        return template.clone();
    }

    public DocumentTemplate createDocumentFor(String templateKey, String author) {
        DocumentTemplate template = templates.get(templateKey);
        if (template == null) {
            throw new IllegalArgumentException("Template not found: " + templateKey);
        }
        return template.cloneForAuthor(author);
    }
}
```

```java
package designpatterns.prototype.document;

public class DocumentTemplateDemo {

    public static void main(String[] args) {
        // --- Build the master quarterly report template ---
        DocumentMetadata templateMeta = new DocumentMetadata(
            "Template Admin", "Operations", "1.0", "INTERNAL"
        );

        DocumentTemplate quarterlyReportTemplate = new DocumentTemplate(
            "Quarterly Business Review",
            "REPORT",
            templateMeta,
            "A4",
            "Arial"
        );

        quarterlyReportTemplate
            .addSection("Executive Summary",
                "Provide a 1-paragraph overview of the quarter's performance...", "HEADING_1")
            .addSection("Financial Highlights",
                "Revenue: [Insert], Expenses: [Insert], Net: [Insert]...", "BODY")
            .addSection("Key Achievements",
                "List top 3-5 achievements this quarter...", "BULLET_LIST")
            .addSection("Challenges and Risks",
                "Describe key risks and mitigation strategies...", "BODY")
            .addSection("Next Quarter Objectives",
                "OKRs and targets for Q[N+1]...", "BODY");

        System.out.println("=== Master Template ===");
        System.out.println(quarterlyReportTemplate);
        System.out.println("Sections: " + quarterlyReportTemplate.getSections().size());
        System.out.println();

        // --- Register template in the library ---
        DocumentTemplateLibrary library = new DocumentTemplateLibrary();
        library.addTemplate("quarterly-review", quarterlyReportTemplate);

        // --- Three teams each create their own document from the template ---
        DocumentTemplate engineeringDoc = library.createDocumentFor("quarterly-review", "Alice Chen");
        DocumentTemplate marketingDoc = library.createDocumentFor("quarterly-review", "Bob Smith");
        DocumentTemplate salesDoc = library.createDocumentFor("quarterly-review", "Carol White");

        System.out.println("=== Cloned Documents ===");
        System.out.println("Engineering: " + engineeringDoc);
        System.out.println("Marketing:   " + marketingDoc);
        System.out.println("Sales:       " + salesDoc);

        // --- Each team customizes their own copy ---
        engineeringDoc.getSections(); // immutable view
        engineeringDoc.addSection("Technical Debt", "Outstanding tech debt items...", "BODY");

        System.out.println("\nEngineering doc sections: " + engineeringDoc.getSections().size()); // 6
        System.out.println("Marketing doc sections:   " + marketingDoc.getSections().size());    // 5 — unchanged
        System.out.println("Master template sections: " + quarterlyReportTemplate.getSections().size()); // 5 — unchanged

        // --- Verify documents are independent ---
        System.out.println("\nEngineering ID: " + engineeringDoc.getTemplateId().substring(0, 8));
        System.out.println("Marketing ID:   " + marketingDoc.getTemplateId().substring(0, 8));
        System.out.println("IDs are different: " + !engineeringDoc.getTemplateId().equals(marketingDoc.getTemplateId()));

        // --- Publish a document ---
        engineeringDoc.setStatus("REVIEW"); // pretend it went through review
        engineeringDoc.publish();
        System.out.println("\nEngineering doc status: " + engineeringDoc.getStatus()); // PUBLISHED
        System.out.println("Marketing doc status:   " + marketingDoc.getStatus());     // DRAFT
    }
}
```

---

## Implementation G: Serialization-Based Deep Copy

Serialization offers a brute-force deep copy mechanism: serialize the object to a byte stream, then deserialize it. The deserialized object is a complete deep copy with zero shared references. This technique is useful when the object graph is complex, has circular references, or when you do not control the source code.

```java
package designpatterns.prototype.serialization;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

/**
 * Serialization-based deep copy utility.
 *
 * Requirements:
 * - The object and all objects it references must implement Serializable.
 * - Non-serializable fields must be marked transient (they will be null in the copy).
 *
 * Performance note: Serialization-based copy is significantly slower than manual deep copy.
 * Benchmark before using in a hot path.
 *
 * When to prefer this over manual deep copy:
 * - Complex object graphs with many levels of nesting
 * - Circular references (Java serialization handles them correctly)
 * - Third-party objects you cannot modify
 * - Rapid prototyping where performance is not critical
 */
public final class SerializationCopier {

    private SerializationCopier() {
        throw new UnsupportedOperationException("Utility class");
    }

    /**
     * Creates a deep copy of the given object via serialization/deserialization.
     *
     * @param original the object to copy
     * @param <T>      the type of the object
     * @return a completely independent deep copy
     * @throws IllegalArgumentException if the object is not serializable or copy fails
     */
    @SuppressWarnings("unchecked")
    public static <T extends Serializable> T deepCopy(T original) {
        try {
            // Serialize to byte array
            ByteArrayOutputStream baos = new ByteArrayOutputStream(512);
            try (ObjectOutputStream oos = new ObjectOutputStream(baos)) {
                oos.writeObject(original);
            }

            // Deserialize from byte array — produces a completely independent copy
            ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
            try (ObjectInputStream ois = new ObjectInputStream(bais)) {
                return (T) ois.readObject();
            }
        } catch (IOException | ClassNotFoundException e) {
            throw new IllegalArgumentException(
                "Cannot deep-copy " + original.getClass().getName() + ": " + e.getMessage(), e
            );
        }
    }
}
```

```java
package designpatterns.prototype.serialization;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;

/**
 * A complex, deeply-nested order object demonstrating serialization-based deep copy.
 */
public class Order implements Serializable {

    private static final long serialVersionUID = 1L;

    private String orderId;
    private Customer customer;
    private List<OrderItem> items;
    private ShippingAddress shippingAddress;
    private double totalPrice;

    public Order(String orderId, Customer customer, List<OrderItem> items,
                 ShippingAddress shippingAddress) {
        this.orderId = orderId;
        this.customer = customer;
        this.items = new ArrayList<>(items);
        this.shippingAddress = shippingAddress;
        this.totalPrice = items.stream().mapToDouble(i -> i.getPrice() * i.getQuantity()).sum();
    }

    /**
     * Creates a deep copy via serialization.
     * Simpler than writing copy constructors for every nested class.
     */
    public Order deepCopy() {
        return SerializationCopier.deepCopy(this);
    }

    public String getOrderId() { return orderId; }
    public void setOrderId(String orderId) { this.orderId = orderId; }
    public Customer getCustomer() { return customer; }
    public List<OrderItem> getItems() { return items; }
    public ShippingAddress getShippingAddress() { return shippingAddress; }
    public double getTotalPrice() { return totalPrice; }

    @Override
    public String toString() {
        return String.format("Order{id='%s', customer=%s, items=%d, total=%.2f, ship=%s}",
            orderId, customer.getName(), items.size(), totalPrice, shippingAddress.getCity());
    }
}
```

```java
package designpatterns.prototype.serialization;

import java.io.Serializable;

public class Customer implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private String email;

    public Customer(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public String getName() { return name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

```java
package designpatterns.prototype.serialization;

import java.io.Serializable;

public class OrderItem implements Serializable {
    private static final long serialVersionUID = 1L;
    private String productSku;
    private String productName;
    private int quantity;
    private double price;

    public OrderItem(String productSku, String productName, int quantity, double price) {
        this.productSku = productSku;
        this.productName = productName;
        this.quantity = quantity;
        this.price = price;
    }

    public String getProductSku() { return productSku; }
    public String getProductName() { return productName; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
    public double getPrice() { return price; }
}
```

```java
package designpatterns.prototype.serialization;

import java.io.Serializable;

public class ShippingAddress implements Serializable {
    private static final long serialVersionUID = 1L;
    private String street;
    private String city;
    private String postalCode;

    public ShippingAddress(String street, String city, String postalCode) {
        this.street = street;
        this.city = city;
        this.postalCode = postalCode;
    }

    public String getStreet() { return street; }
    public String getCity() { return city; }
    public String getPostalCode() { return postalCode; }
    public void setCity(String city) { this.city = city; }
}
```

```java
package designpatterns.prototype.serialization;

import java.util.Arrays;

public class SerializationCopyDemo {

    public static void main(String[] args) {
        Order original = new Order(
            "ORD-001",
            new Customer("Alice", "alice@example.com"),
            Arrays.asList(
                new OrderItem("SKU-A", "Java Book", 2, 49.99),
                new OrderItem("SKU-B", "Design Patterns Poster", 1, 19.99)
            ),
            new ShippingAddress("123 Main St", "New York", "10001")
        );

        System.out.println("Original: " + original);

        // Deep copy via serialization
        Order copy = original.deepCopy();
        copy.setOrderId("ORD-002");
        copy.getCustomer().setEmail("alice-new@example.com");
        copy.getShippingAddress().setCity("Los Angeles");
        copy.getItems().get(0).setQuantity(5);

        System.out.println("\nAfter modifying copy:");
        System.out.println("Original: " + original); // unchanged
        System.out.println("Copy:     " + copy);

        System.out.println("\nOriginal customer email: " + original.getCustomer().getEmail()); // alice@example.com
        System.out.println("Copy customer email:     " + copy.getCustomer().getEmail());      // alice-new@example.com
        System.out.println("Original item1 qty: " + original.getItems().get(0).getQuantity()); // 2
        System.out.println("Copy item1 qty:     " + copy.getItems().get(0).getQuantity());     // 5

        System.out.println("\nOriginal ship city: " + original.getShippingAddress().getCity()); // New York
        System.out.println("Copy ship city:     " + copy.getShippingAddress().getCity());       // Los Angeles
    }
}
```

---

## Java Cloneable: Why It Is Considered Broken

### Problem 1: Cloneable Does Not Declare `clone()`

`Cloneable` is a marker interface with no methods:

```java
public interface Cloneable {
    // Intentionally empty
}
```

The `clone()` method is declared on `Object`, not on `Cloneable`. You cannot write:

```java
Cloneable c = getCloneable();
c.clone(); // COMPILE ERROR — Cloneable doesn't declare clone()
```

This defeats the purpose of a polymorphic cloning contract. You must cast to the concrete type or define your own `Prototype` interface that declares `clone()`.

### Problem 2: `Object.clone()` Does Shallow Copy by Default

`Object.clone()` copies all fields bit-for-bit. For mutable reference types, this means sharing — the bug demonstrated in Implementation A above.

### Problem 3: `clone()` Does Not Call Constructors

This is subtle but dangerous. When `super.clone()` creates a new instance, it bypasses the constructor chain entirely. Any invariant enforcement in constructors (null checks, range validation, logging, security checks) is skipped.

```java
public class SecureCounter {
    private static final AtomicInteger instanceCount = new AtomicInteger(0);
    private int value;

    public SecureCounter(int value) {
        instanceCount.incrementAndGet(); // tracks how many instances exist
        SecurityAudit.log("Counter created");
        this.value = value;
    }

    @Override
    public SecureCounter clone() {
        try {
            return (SecureCounter) super.clone();
            // instanceCount is NOT incremented!
            // SecurityAudit.log() is NOT called!
            // You now have an "invisible" instance.
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);
        }
    }
}
```

### Problem 4: `CloneNotSupportedException` Is a Checked Exception

`Object.clone()` is declared as:

```java
protected native Object clone() throws CloneNotSupportedException;
```

Even when you implement `Cloneable` (which guarantees this exception won't be thrown), you still must catch or declare it. This forces ugly boilerplate:

```java
@Override
public MyClass clone() {
    try {
        return (MyClass) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new AssertionError("Should never happen", e); // noise
    }
}
```

### Problem 5: Final Fields Cannot Be Assigned After `super.clone()`

If a class has final fields that need to be different in the clone, you cannot set them:

```java
public class Config implements Cloneable {
    private final String id; // must be unique per instance

    @Override
    public Config clone() {
        try {
            Config c = (Config) super.clone();
            c.id = UUID.randomUUID().toString(); // COMPILE ERROR — cannot assign to final field
            return c;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);
        }
    }
}
```

Copy constructors handle this naturally since they go through the constructor where final fields can be set.

### Problem 6: The Broken Inheritance Contract

If a superclass implements `Cloneable` correctly but a subclass adds a mutable field and forgets to deep-copy it in its overridden `clone()`, the superclass has no way to enforce correct behavior. The contract is implicit and fragile across inheritance chains.

### Joshua Bloch's View (Effective Java, Item 13)

> "The Cloneable interface was intended as a mixin interface for objects to advertise that they permit cloning. Unfortunately, it fails to serve this purpose... Object's clone method is protected; so without reflection we can't invoke clone() on an object merely because it implements Cloneable. Even a reflective invocation may fail, because there is no guarantee that the object has an accessible clone method."
>
> "Given all the problems associated with Cloneable, new interfaces should not extend it, and new extendable classes should not implement it."
>
> "A better approach to object copying is to provide a copy constructor or copy factory."

---

## Real-World Analogy

### Cell Division (Biology)

When a cell divides (mitosis), it does not consult a "cell blueprint" and construct two new cells from raw materials. Instead, it **copies itself** — duplicating its DNA, organelles, and membrane — and produces two daughter cells that start as identical copies of the parent. Each daughter then differentiates and adapts independently.

This is the Prototype Pattern in nature: one expensive "construction" (the parent cell achieving its fully-functional state) enables many cheap copies.

### Photocopying a Document (Office Analogy)

A secretary creates one master document — carefully formatted, spell-checked, with the correct letterhead — and then photocopies it for each recipient. Each recipient can write notes on their copy without affecting others' copies or the master.

The "master" is the prototype. Each photocopy is a clone. The note-taking is post-clone customization.

### Stamping with a Mold (Manufacturing)

A mold (die) for a car part is expensive to design and build. Once built, stamping thousands of identical parts from it is cheap. The mold is the prototype; each stamped part is a clone. If you want a slightly different part, you start from the same mold and modify the resulting stamp, not the mold itself.

---

## Real-World Examples in Java Ecosystem

### 1. `Object.clone()` in Java Standard Library

Many JDK classes use `clone()` internally:
- `java.util.ArrayList.clone()` — creates a shallow copy of the list (new array, same element references)
- `java.util.HashMap.clone()` — creates a shallow copy
- `java.util.LinkedList.clone()` — shallow copy

```java
ArrayList<String> original = new ArrayList<>(Arrays.asList("a", "b", "c"));
ArrayList<String> copy = (ArrayList<String>) original.clone(); // shallow — ok for immutable String elements
copy.add("d");
System.out.println(original); // [a, b, c] — unaffected
System.out.println(copy);     // [a, b, c, d]
```

Note that `ArrayList.clone()` is a shallow copy — if the list contains mutable objects, those are shared.

### 2. Spring Framework Prototype Scope

Spring's prototype bean scope is a direct application of the Prototype Pattern. When you annotate a bean with `@Scope("prototype")`, each call to `applicationContext.getBean()` returns a **new instance** rather than a cached singleton.

```java
@Component
@Scope("prototype") // Each injection/getBean creates a new instance
public class ShoppingCart {
    private List<CartItem> items = new ArrayList<>();
    private String sessionId;
    // ...
}

// In a service:
@Autowired
private ApplicationContext context;

public ShoppingCart createNewCart(String sessionId) {
    // Spring creates a fresh ShoppingCart (no shared state)
    ShoppingCart cart = context.getBean(ShoppingCart.class);
    cart.setSessionId(sessionId);
    return cart;
}
```

Spring uses the bean definition as the "prototype" and creates new instances from it. Each user gets their own cart — no shared state.

### 3. `Collections.copy()` and `Collections.unmodifiableList()`

```java
List<String> source = Arrays.asList("alpha", "beta", "gamma");
List<String> dest = new ArrayList<>(source.size()); // must pre-size
Collections.addAll(dest, new String[source.size()]); // fill with nulls
Collections.copy(dest, source); // copies elements from source to dest
```

### 4. `BufferedImage.copyData()` and Graphics Contexts

Java's `BufferedImage` supports cloning pixel data for image processing pipelines — apply filters to a copy without destroying the original.

### 5. Hibernate / JPA Entity Detach and Copy

When you detach a JPA entity and re-attach it (or create an "off-graph" copy for a DTO transformation), you are effectively cloning the entity. Libraries like MapStruct generate copy code that follows the copy-constructor pattern.

### 6. JavaScript / Prototypal Inheritance

JavaScript's entire object model is built on prototypes. Every object has a `[[Prototype]]` chain. `Object.create(proto)` creates a new object whose prototype is `proto` — a direct implementation of the Prototype Pattern at the language level.

---

## Trade-offs

### Advantages

| Advantage                              | Explanation                                                                                            |
|----------------------------------------|--------------------------------------------------------------------------------------------------------|
| **Performance**                        | Cloning is O(n) in the object graph size. Expensive construction (I/O, network, computation) is paid only once for the master prototype. |
| **Decoupling from concrete classes**   | The client codes against the `Prototype` interface. It does not need to know or import concrete types. |
| **Avoids parallel factory hierarchy**  | Each product is its own factory. Contrast with Abstract Factory, where each product variant requires a dedicated factory class. |
| **Captures runtime state**             | A clone captures all runtime-accumulated state — including private state set during initialization — which a constructor cannot replicate without knowing that state. |
| **Simplifies complex initialization**  | Objects built through multi-step initialization (builder, fluent API) can be cloned after the build phase without repeating all steps. |
| **Natural fit for undo/redo**          | Taking a snapshot before an operation is trivial: `previousState = currentState.clone()`. |

### Disadvantages

| Disadvantage                                     | Explanation                                                                                                     |
|--------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| **Cloning complex graphs is hard**               | Objects with deep nesting, circular references, or polymorphic fields require careful clone logic. Missing one mutable field silently creates a bug. |
| **Java's `Cloneable` is poorly designed**        | As described above — no method declaration, bypasses constructors, checked exception, final field problem.       |
| **Shallow copy bugs are silent**                 | A shallow copy compiles and runs but produces subtle corruption bugs that only manifest when the shared object is mutated. |
| **Cannot clone non-serializable/non-clonable types** | Resources like `Thread`, `Socket`, `InputStream` cannot be cloned. |
| **Violates constructor invariants**              | `clone()` does not call constructors, so any invariant checking, logging, or side-effect in constructors is skipped. |
| **Maintenance burden**                           | Every time a new mutable field is added to a class, the `clone()` / copy constructor must be updated. Forgetting causes bugs. |
| **Performance cost of serialization-based copy** | Serialization is significantly slower (10-100x) than manual copy constructors. Do not use in hot paths. |

---

## Interview Questions and Answers

### Q1: What is the Prototype Pattern, and when should you use it over `new`?

**Answer:**

The Prototype Pattern is a creational design pattern where new objects are created by copying an existing "prototype" object rather than instantiating from scratch. The central interface provides a `clone()` method, and each concrete class implements it to return a deep (or shallow, where safe) copy of itself.

You should prefer cloning over `new` in three primary scenarios. First, when construction is expensive — if creating an object requires I/O, network calls, loading assets, or heavy computation, you pay that cost once for the master prototype, then clone cheaply for each subsequent instance. Second, when you need many instances that differ only in minor aspects from a template — it is cheaper to clone and patch than to pass all template parameters through the constructor repeatedly. Third, when the client must decouple from the concrete class — by holding a reference to the `Prototype` interface, the client can clone without knowing or importing the concrete type.

A counter-example: if the constructor is just `this.x = x`, cloning adds no value and only introduces complexity.

---

### Q2: What is the difference between shallow copy and deep copy? How does each affect the Prototype Pattern?

**Answer:**

A shallow copy creates a new object but copies reference fields by value — meaning the new object's mutable reference fields point to the same heap objects as the original. A deep copy creates new objects for every mutable reference in the graph, so the copy and the original share no mutable state.

In the context of Prototype, shallow copy is safe only when all reference fields are immutable (e.g., `String`, `Integer`, `LocalDate`). If any reference field is mutable (e.g., `ArrayList`, custom objects), a shallow copy creates a shared-state bug: mutating the clone's list actually mutates the original's list, because they are the same object.

Java's `Object.clone()` does a shallow copy by default. To implement deep copy with `Cloneable`, you must override `clone()`, call `super.clone()`, and then manually create new copies of every mutable reference field. This is error-prone because adding a new mutable field later can silently break the clone.

The preferred production approach is the copy constructor or copy factory: explicitly copy every field in a constructor. This is readable, maintainable, and does not have the surprise behaviors of `Object.clone()`. Alternatively, serialization-based copy achieves deep copy automatically but at significantly higher performance cost.

---

### Q3: Why does Joshua Bloch consider `Cloneable` broken, and what does he recommend instead?

**Answer:**

Bloch identifies several fundamental design flaws in Java's `Cloneable` mechanism. First, `Cloneable` is a marker interface with no methods — it does not declare `clone()`. The method lives on `Object`, so you cannot invoke `clone()` polymorphically on a `Cloneable` reference without casting. This defeats the purpose of a clone interface.

Second, `Object.clone()` bypasses constructors entirely. The new instance is created through native code rather than through any constructor in the hierarchy. This means all invariant checking, logging, security audits, and side effects coded into constructors are silently skipped. For classes where constructor logic matters — and it usually does — this is a correctness hazard.

Third, `clone()` throws a checked `CloneNotSupportedException`, which is unnecessary boilerplate when you implement `Cloneable` (the exception will never actually be thrown). Fourth, final fields cannot be assigned in `clone()` after `super.clone()` returns, so classes with final fields that need to differ across instances cannot use `Cloneable` correctly.

Bloch recommends copy constructors or copy factory methods. They are normal Java constructors, so invariants are enforced, logging happens, final fields work, and there is no checked exception. They are also more readable — `new UserProfile(other)` or `UserProfile.copyOf(other)` makes the intent explicit. Additionally, copy constructors can accept interface types (e.g., `new TreeSet<>(hashSet)`), enabling cross-type construction.

---

### Q4: How does the Prototype Registry work, and when would you use it?

**Answer:**

A Prototype Registry (also called a Prototype Manager) is a centralized store that maps names or keys to pre-configured prototype objects. When a client requests an object by key, the registry returns a clone of the registered prototype — never the prototype itself. The master prototype stays in the registry, pristine, and every consumer receives an independent copy.

The registry is valuable in two scenarios. First, when you have a fixed set of "template" objects that are expensive to construct but frequently needed in slightly varied forms — a game's enemy types, email notification templates, document formats, or product configurations. You pay the construction cost once at startup, store prototypes in the registry, and clone cheaply at runtime.

Second, the registry decouples clients from concrete types. The client asks for `"orc-warrior"` or `"quarterly-report"` by string key. It never imports `OrcWarrior` or `QuarterlyReport`. New prototype types can be registered without changing client code — this is the Open/Closed Principle applied to object creation.

A production implementation should: return clones (never the master), validate keys on registration (null/blank), throw meaningful exceptions for missing keys, optionally support typed retrieval to avoid unchecked casts, and consider thread safety if the registry is shared across threads (using `ConcurrentHashMap` and proper synchronization for registration).

---

### Q5: How would you implement serialization-based deep copy? What are its trade-offs?

**Answer:**

Serialization-based deep copy uses Java's object serialization mechanism: serialize the object to an in-memory byte stream using `ObjectOutputStream`, then deserialize it back using `ObjectInputStream`. The deserialized object has no references shared with the original — it is a complete deep copy.

The implementation is straightforward:

```java
public static <T extends Serializable> T deepCopy(T original) {
    try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
         ObjectOutputStream oos = new ObjectOutputStream(baos)) {
        oos.writeObject(original);
        try (ObjectInputStream ois = new ObjectInputStream(
                new ByteArrayInputStream(baos.toByteArray()))) {
            return (T) ois.readObject();
        }
    } catch (IOException | ClassNotFoundException e) {
        throw new IllegalArgumentException("Cannot copy: " + e.getMessage(), e);
    }
}
```

The primary advantages are that it handles arbitrary object graph depth automatically, correctly handles circular references (Java serialization tracks already-serialized objects), and requires no per-class copy logic — you write the utility once.

The trade-offs are significant. Performance: serialization involves reflection, byte encoding, and I/O buffering — typically 10 to 100 times slower than a hand-written copy constructor. All classes in the graph must implement `Serializable`, which is an invasive constraint. Transient fields are null in the copy, which may not be desired. The `serialVersionUID` must be maintained for binary compatibility. And finally, serialization has security implications — `ObjectInputStream.readObject()` can trigger arbitrary code execution if the stream is untrusted (which is fine here since we generate the stream ourselves).

Use serialization-based deep copy for: complex object graphs you do not control, rapid prototyping, or situations where correctness matters more than performance. Use copy constructors in production-grade, performance-sensitive paths.

---

### Q6: Explain how the Prototype Pattern enables decoupling from concrete types. Give a concrete example.

**Answer:**

The Prototype Pattern enables type decoupling through its core interface: `Prototype` declares `clone()`, and every concrete type implements it. The client only holds a `Prototype` reference and calls `clone()` — it never needs to know or import the actual concrete class.

Consider a game engine that loads level configurations from JSON:

```java
// Client code
public class LevelLoader {
    private PrototypeRegistry registry; // populated at startup

    public List<Enemy> loadEnemies(LevelConfig config) {
        List<Enemy> enemies = new ArrayList<>();
        for (SpawnEntry entry : config.getSpawnEntries()) {
            // Client only knows the string key — not OrcEnemy, DragonEnemy, etc.
            Enemy enemy = (Enemy) registry.get(entry.getEnemyType());
            enemy.spawnAt(entry.getX(), entry.getY());
            enemies.add(enemy);
        }
        return enemies;
    }
}
```

When the game design team adds a new enemy type `TrollEnemy`, the developer: creates the `TrollEnemy` class implementing `Enemy` (which has `clone()`), registers a master instance in the registry under `"troll"`, and updates the JSON level files with `"enemy_type": "troll"`. The `LevelLoader` class requires zero changes. This is the Open/Closed Principle: `LevelLoader` is closed to modification, open to extension via new prototype registrations.

Contrast this with Abstract Factory, where adding a new enemy would require a new `TrollFactory` class and modifications to the factory interface.

---

### Q7: How does the Prototype Pattern compare to the Builder Pattern? When would you choose one over the other?

**Answer:**

Builder and Prototype are both creational patterns, but they solve different problems and operate on different phases of object creation.

Builder addresses the problem of **constructing a complex object step-by-step** when there are many optional parameters or when construction order matters. It assembles an object from scratch, calling setter-like methods to configure each part before calling `build()`. Builder is the right choice when: different configurations are needed from the same raw parts, when the number of constructor parameters is large, or when you want a fluent API that reads like English (`new PizzaBuilder().size(LARGE).cheese(MOZZARELLA).topping(PEPPERONI).build()`).

Prototype addresses the problem of **duplicating an existing, fully-initialized object** cheaply. It assumes the object is already in a valid, complete state. It is the right choice when: construction is expensive (I/O, computation), you need many instances that are slight variations of an existing object, or when you want to decouple the client from the concrete type.

They can be combined: use Builder to construct the master prototype (because the builder's fluent API makes complex initialization readable), then use Prototype (clone) to produce the many instances needed at runtime. The Builder pays the initialization cost once; Prototype amortizes it across all clones.

Rule of thumb: **Builder** when you are constructing from parts and configuration. **Prototype** when you are duplicating an existing good state.

---

### Q8: Describe a scenario where implementing Prototype is more complex than it appears at first. How would you handle it?

**Answer:**

The complexity of Prototype surfaces most dramatically with **polymorphic object graphs containing circular references**. Suppose you have a `Department` that has a list of `Employee` objects, and each `Employee` has a reference back to their `Department`. A naive clone would loop: cloning the department clones the list of employees, cloning each employee triggers a clone of their department, which triggers cloning their employees again — infinite recursion.

The correct solution for manual deep copy uses a **clone context** (a visited map): before cloning any object, check if it has already been cloned in the current clone operation and return the existing clone if so. This mirrors how Java's serialization handles circular references internally.

```java
public Department clone() {
    Map<Object, Object> cloneContext = new IdentityHashMap<>();
    return cloneWithContext(cloneContext);
}

Department cloneWithContext(Map<Object, Object> ctx) {
    if (ctx.containsKey(this)) return (Department) ctx.get(this);
    Department clone = new Department(this.name);
    ctx.put(this, clone); // register BEFORE cloning children
    for (Employee e : this.employees) {
        clone.addEmployee(e.cloneWithContext(ctx)); // employees reuse the same context
    }
    return clone;
}
```

A second complexity: **non-serializable third-party objects**. If your prototype contains a field of type `javax.sql.DataSource` or `java.io.InputStream`, you cannot clone it. Options: mark the field `transient` and re-initialize it in the clone, or use a shared reference intentionally (if it is truly shared-by-design, like a connection pool).

A third complexity: **class hierarchies where subclasses add mutable fields**. If a superclass has a correct deep-copy `clone()`, but a subclass adds a new mutable field and forgets to override `clone()`, the superclass clone is silently shallow for the new field. The solution is to always mark `clone()` as `abstract` in the abstract base class, forcing every concrete subclass to implement it.

The takeaway: Prototype is conceptually simple, but production-quality implementation requires discipline about every mutable field in the entire inheritance and composition hierarchy.

---

## Comparison with Related Patterns

### Prototype vs Builder

```
Problem:  CONSTRUCTION                 vs  DUPLICATION
Builder:  many optional params,            [not applicable]
          step-by-step assembly
Prototype: [not applicable]               copy existing state cheaply

Key question: Are you building from SCRATCH or COPYING an existing object?

Combination: Use Builder to BUILD the prototype, then Prototype to CLONE it many times.

Builder:         Director -> Builder -> Product
Prototype:       MasterObject.clone() -> Clone
Combined:        Director -> Builder -> MasterObject -> (clone x N) -> [N instances]
```

**When Builder wins:** You need different configurations from the same raw materials. You want a fluent construction API. Construction logic is complex but not expensive to repeat.

**When Prototype wins:** You need many instances of the same fully-initialized object. Construction is expensive. You want to decouple from the concrete class.

---

### Prototype vs Abstract Factory

```
Abstract Factory:
  AbstractFactory
    +-- ConcreteFactory1  --> Product1A, Product2A
    +-- ConcreteFactory2  --> Product1B, Product2B
  [Parallel class hierarchy: one factory per product family]

Prototype:
  PrototypeRegistry {
    "productA-variant1" -> ConcreteProductA(variant1 config)
    "productA-variant2" -> ConcreteProductA(variant2 config)
    "productB-variant1" -> ConcreteProductB(variant1 config)
  }
  [No parallel factory hierarchy — each product is its own factory]
```

**Abstract Factory wins when:** Product families are fixed and known at compile time. Type safety is paramount — factories return concrete types. Construction logic is in factory methods, not in the products themselves.

**Prototype wins when:** Product variants are dynamic (configured at runtime, loaded from config). You want to avoid a proliferating factory class hierarchy. New variants need to be added without recompiling the client.

---

### Prototype vs Singleton

These patterns have **opposing intents**:

```
Singleton: ensure only ONE instance exists — never clone it.
Prototype: ensure you CAN produce multiple independent copies.
```

A Singleton can hold a registry of Prototypes — that combination is valid. The singleton manages the registry (one registry), while the prototypes enable cloning (many independent copies).

```java
// Anti-pattern: do NOT do this
public class Config implements Cloneable {
    private static final Config INSTANCE = new Config();
    private Config() {}
    public static Config getInstance() { return INSTANCE; }

    // Returning a clone from singleton breaks the singleton contract
    // Either the class is a Singleton or it's a Prototype — not both
    @Override
    public Config clone() { ... }
}
```

---

### Prototype vs Factory Method

Factory Method uses inheritance and a virtual `create()` method to defer construction to subclasses. Prototype uses `clone()` on an existing object. The difference is **inheritance + construction** vs **delegation + copying**.

```
Factory Method:
  Creator (abstract)
    + createProduct(): Product  // override in subclass
  ConcreteCreator extends Creator
    + createProduct(): return new ConcreteProduct()

Prototype:
  Product implements Prototype
    + clone(): return new ConcreteProduct(this)
  Client just calls prototype.clone() — no Creator hierarchy needed
```

Factory Method requires you to subclass the creator to produce new types. Prototype requires you to have an existing instance to clone. Factory Method is better when creation logic is complex and belongs in a creator class. Prototype is better when the object's construction is encapsulated within itself and cloning is the primary need.

---

## Summary

The Prototype Pattern is the right tool when:

1. **Construction is expensive** — pay once, clone many times.
2. **Many similar instances** are needed with small variations.
3. **Decoupling from concrete types** is desirable.
4. **Runtime-determined state** must be captured and reproduced.

### Decision Tree

```
Need a new object?
  |
  +-- Do I already have an instance in the right state?
  |     YES -> Use Prototype (clone it)
  |     NO  -> Use Constructor, Factory, or Builder
  |
  +-- Is construction expensive?
  |     YES -> Build prototype once, clone for each need
  |     NO  -> Just use new — Prototype adds no value
  |
  +-- Do I need many slightly-different instances?
        YES -> Prototype + post-clone customization
        NO  -> Direct construction is simpler
```

### Preferred Java Implementation Ranking

| Approach                        | Recommended | Reason                                                     |
|---------------------------------|-------------|------------------------------------------------------------|
| Copy constructor                | Best        | Explicit, safe, supports final fields, no checked exception|
| Copy factory method             | Best        | Same as copy constructor, can accept interface types        |
| Manual deep `clone()` override  | Acceptable  | Works but requires discipline; boilerplate-heavy            |
| Serialization-based copy        | Niche       | Automatic but slow; use for complex graphs you don't control|
| `super.clone()` shallow only    | Dangerous   | Only safe if ALL mutable fields are immutable types         |

### Key Takeaways

- **Shallow copy is not deep copy.** Every mutable reference field must be explicitly deep-copied, or the clone and the original silently share state.
- **Cloneable is broken.** Use copy constructors (Bloch's recommendation) in production code.
- **Prototype Registry** decouples clients from concrete types and enables dynamic addition of new variants without client changes.
- **Performance gains** are real: game engines, document systems, and notification platforms all benefit from the pattern when object construction is non-trivial.
- **Combine with other patterns**: Builder to construct the prototype, Singleton to manage the registry, Prototype to clone.

---

*Part of the Industry-Grade LLD & OOP Design Patterns Course — Creational Patterns Module*
