# Composite Pattern

> **Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.**
> — Gang of Four, *Design Patterns: Elements of Reusable Object-Oriented Software*

---

## Table of Contents

1. [Intent](#1-intent)
2. [Real-World Analogy](#2-real-world-analogy)
3. [When to Use / When NOT to Use](#3-when-to-use--when-not-to-use)
4. [UML Class Diagram](#4-uml-class-diagram)
5. [Core Participants](#5-core-participants)
6. [Implementation 1: File System](#6-implementation-1-file-system)
7. [Implementation 2: Menu System](#7-implementation-2-menu-system)
8. [Implementation 3: Organization Hierarchy](#8-implementation-3-organization-hierarchy)
9. [Transparent vs Safe Composite Design](#9-transparent-vs-safe-composite-design)
10. [Handling Composite-Only Operations](#10-handling-composite-only-operations)
11. [Composite + Iterator Pattern](#11-composite--iterator-pattern)
12. [Real-World Examples in Java Ecosystem](#12-real-world-examples-in-java-ecosystem)
13. [Trade-offs](#13-trade-offs)
14. [Common Pitfalls and Gotchas](#14-common-pitfalls-and-gotchas)
15. [Pattern Comparisons](#15-pattern-comparisons)
16. [Interview Questions and Model Answers](#16-interview-questions-and-model-answers)

---

## 1. Intent

The Composite pattern solves a fundamental problem in software design: **how do you model tree-structured data so that client code does not need to distinguish between a leaf node and a branch node?**

### The Core Problem

Consider a file system. You have files and directories. A directory can contain files and other directories. If you want to compute the total size of a directory, you must recursively compute sizes of all contents. Without the Composite pattern, client code must constantly check: "Is this a file or a directory?" — and branch accordingly:

```java
// Without Composite — ugly, fragile, violates OCP
public long computeSize(Object node) {
    if (node instanceof File) {
        return ((File) node).getSize();
    } else if (node instanceof Directory) {
        long total = 0;
        for (Object child : ((Directory) node).getChildren()) {
            total += computeSize(child);  // recursive, but requires instanceof checks
        }
        return total;
    }
    throw new IllegalArgumentException("Unknown node type");
}
```

This code breaks every time you add a new node type. It violates the Open/Closed Principle — you must modify `computeSize` whenever the hierarchy evolves.

### The Solution

The Composite pattern defines a **Component** interface that both leaf objects and composite objects implement. The composite object holds a collection of components and delegates operations to them recursively. Client code operates only against the Component interface — it never knows whether it is talking to a leaf or a composite.

```
Client --> Component (interface: operation())
               |
        -------+--------
        |               |
      Leaf          Composite
   (no children)  (children: List<Component>)
                  delegates operation() to all children
```

### Key Properties

- **Uniformity**: Clients treat leaves and composites identically through a common interface.
- **Recursion**: A composite's operation is defined in terms of the same operation on its children, forming natural recursion.
- **Open for extension**: Adding a new node type (e.g., a `SymbolicLink`) requires only a new class that implements `Component`. No client code changes.
- **Part-whole hierarchy**: The pattern explicitly models the relationship between whole structures and their constituent parts.

---

## 2. Real-World Analogy

Think of a corporate **organizational chart**.

- An individual **employee** does work: they write code, design systems, answer emails.
- A **department** (e.g., Engineering) is composed of employees and sub-departments (Backend Team, Frontend Team, DevOps Team).
- The **company** is a department composed of other departments.

Now ask: "What is the total salary expense for Engineering?" You do not care whether a node is a person or a sub-team — you recursively sum salaries down the entire subtree.

When the CEO says "all of Engineering must submit performance reviews", they do not enumerate individual employees — they tell Engineering, and Engineering propagates the instruction downward. The CEO's interface with Engineering is identical to Engineering's interface with its sub-teams and individual members. That uniformity is the Composite pattern.

Other everyday analogies:
- **HTML/XML DOM**: A `<div>` is a composite node; a text node is a leaf. Both are `Node` objects.
- **A bill of materials**: A "car" is composed of "engine", "chassis", "body". "Engine" is composed of "cylinder block", "pistons". Each component, at any level, has a cost and a weight.
- **A menu at a restaurant**: The "Main Course" section is a menu containing items. The full menu contains sections. Rendering the full menu means recursively rendering every section and every item within it.

---

## 3. When to Use / When NOT to Use

### When to Use

**Use the Composite pattern when:**

1. **You need to represent part-whole hierarchies of objects.** Any time you have a tree structure where nodes can be either leaves or containers of other nodes, Composite is the natural fit.

2. **You want clients to ignore the difference between compositions of objects and individual objects.** If your client code constantly writes `if (x instanceof Leaf)` / `else if (x instanceof Composite)`, you need Composite.

3. **The recursive nature of the structure is stable.** Once you model a file system with Composite, new file types just implement the interface — they do not change the recursive logic.

4. **Operations are naturally recursive.** `getSize()`, `render()`, `delete()`, `print()`, `validate()` — anything that aggregates or propagates across a tree.

5. **You are modeling GUI widget hierarchies.** A `Panel` contains `Button`, `Label`, and other `Panel` objects. All are `Widget`. This is the original use case from the GoF book.

### When NOT to Use

**Avoid the Composite pattern when:**

1. **The hierarchy is shallow or fixed.** If you always have exactly two levels (e.g., a group of items), Composite adds unnecessary abstraction. A simple `List<Item>` suffices.

2. **Leaf and Composite operations diverge significantly.** If leaves and composites have fundamentally different behaviors with few shared operations, forcing them into a common interface produces a bloated, incoherent contract.

3. **You need strong typing to distinguish leaves from composites.** If calling `add(child)` on a leaf must be a compile-time error (not a runtime error), the Transparent Composite design (see Section 9) is inappropriate. You may want a different structure.

4. **The tree is not recursive but just a flat collection.** A shopping cart holding `CartItem` objects does not need Composite — it needs a `List<CartItem>`.

5. **Performance is critical and tree depth is large.** Composite introduces per-node object overhead and recursive call stacks. For very deep trees with millions of nodes, evaluate whether a more cache-friendly structure is warranted.

6. **You need to share objects between multiple parents.** Composite implies tree ownership; if a node can have multiple parents, you are dealing with a DAG (directed acyclic graph), not a tree. Standard Composite does not handle this without careful reference management.

---

## 4. UML Class Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            <<interface>>                                │
│                              Component                                  │
├─────────────────────────────────────────────────────────────────────────┤
│  + operation(): void                                                    │
│  + getName(): String                                                    │
│  + add(component: Component): void          // optional (see Section 9) │
│  + remove(component: Component): void       // optional (see Section 9) │
│  + getChildren(): List<Component>           // optional (see Section 9) │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │ implements
           ─────────────┴──────────────
           │                          │
┌──────────┴──────────┐    ┌──────────┴────────────────────────────────┐
│        Leaf         │    │                  Composite                 │
├─────────────────────┤    ├────────────────────────────────────────────┤
│  - name: String     │    │  - name: String                            │
│  - ...              │    │  - children: List<Component>               │
├─────────────────────┤    ├────────────────────────────────────────────┤
│  + operation()      │    │  + operation()                             │
│    (performs work)  │    │    (delegates to each child)               │
│                     │    │  + add(c: Component): void                 │
│  + add() throws     │    │  + remove(c: Component): void              │
│    UnsupportedOp    │    │  + getChildren(): List<Component>          │
└─────────────────────┘    └──────────────┬─────────────────────────────┘
                                          │ has-many (composition)
                                          │ 0..*
                                    ┌─────┴──────┐
                                    │  Component │  (recursive association)
                                    └────────────┘

Legend:
  ──── Implementation/Inheritance arrow
  ──── Aggregation arrow (has-many)
  <<interface>> marks the Component abstraction
```

### Collaboration Diagram

```
Client                 Composite (root)         Composite (child)        Leaf
   │                        │                         │                   │
   │  operation()           │                         │                   │
   │───────────────────────>│                         │                   │
   │                        │  for each child:        │                   │
   │                        │  child.operation()      │                   │
   │                        │────────────────────────>│                   │
   │                        │                         │  for each child:  │
   │                        │                         │  child.operation()│
   │                        │                         │──────────────────>│
   │                        │                         │                   │ (performs work)
   │                        │                         │<──────────────────│
   │                        │<────────────────────────│                   │
   │<───────────────────────│                         │                   │
```

---

## 5. Core Participants

| Participant | Role |
|---|---|
| **Component** | Declares the interface for objects in the composition, including default behavior for leaf-appropriate operations and an interface for accessing/managing child components. |
| **Leaf** | Represents leaf nodes that have no children. Defines behavior for primitive objects. Does the actual work. |
| **Composite** | Defines behavior for components that have children. Stores child components. Implements child-related operations in the Component interface. Delegates operations to children and may aggregate results. |
| **Client** | Manipulates objects in the composition through the Component interface. Does not know whether it is working with a Leaf or a Composite. |

---

## 6. Implementation 1: File System

This is the canonical example. Both `File` (leaf) and `Directory` (composite) implement `FileSystemComponent`. Key operations: `getSize()`, `list()`, and `delete()`.

```java
package com.coursedesign.patterns.structural.composite.filesystem;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Component interface for the File System Composite pattern.
 *
 * <p>Declares operations that make sense for both files and directories.
 * The {@code add} and {@code remove} methods are declared here for
 * transparency (see Section 9 of the course notes for the trade-off
 * discussion). A default implementation that throws
 * {@link UnsupportedOperationException} is provided so that Leaf
 * implementations do not need to implement child-management methods.
 *
 * @version 1.0
 */
public interface FileSystemComponent {

    /**
     * Returns the name of this file system node.
     *
     * @return the node name (file or directory name, without full path)
     */
    String getName();

    /**
     * Returns the size of this node in bytes.
     * For a file, this is the file's own size.
     * For a directory, this is the recursive sum of all children's sizes.
     *
     * @return total size in bytes
     */
    long getSize();

    /**
     * Lists the contents of this node. Files print their own name and size.
     * Directories recursively list their contents with indentation.
     *
     * @param indent the indentation prefix for pretty-printing
     */
    void list(String indent);

    /**
     * Deletes this node. For a directory, recursively deletes all children first.
     */
    void delete();

    /**
     * Adds a child component. Only meaningful for directories.
     * Default implementation throws {@link UnsupportedOperationException}.
     *
     * @param component the child to add
     * @throws UnsupportedOperationException if this node is a leaf
     */
    default void add(FileSystemComponent component) {
        throw new UnsupportedOperationException(
            getName() + " is a leaf node and cannot have children."
        );
    }

    /**
     * Removes a child component. Only meaningful for directories.
     * Default implementation throws {@link UnsupportedOperationException}.
     *
     * @param component the child to remove
     * @throws UnsupportedOperationException if this node is a leaf
     */
    default void remove(FileSystemComponent component) {
        throw new UnsupportedOperationException(
            getName() + " is a leaf node and cannot have children."
        );
    }

    /**
     * Returns the children of this node. Only meaningful for directories.
     * Default implementation returns an empty unmodifiable list.
     *
     * @return list of child components; empty list for leaves
     */
    default List<FileSystemComponent> getChildren() {
        return Collections.emptyList();
    }
}
```

```java
package com.coursedesign.patterns.structural.composite.filesystem;

/**
 * Leaf node in the file system Composite hierarchy.
 *
 * <p>Represents an individual file with a fixed size. A {@code File} cannot
 * have children. It performs all operations directly without delegation.
 *
 * <p>The {@link #add}, {@link #remove}, and {@link #getChildren} methods
 * are inherited from the interface as default implementations that throw
 * {@link UnsupportedOperationException} or return empty lists — the leaf
 * does not override them.
 */
public class File implements FileSystemComponent {

    private final String name;
    private final long   size;      // size in bytes
    private       boolean deleted;

    /**
     * Creates a new File.
     *
     * @param name the file name (e.g., "report.pdf")
     * @param size the file size in bytes; must be non-negative
     * @throws IllegalArgumentException if {@code size} is negative
     */
    public File(String name, long size) {
        if (size < 0) {
            throw new IllegalArgumentException("File size cannot be negative: " + size);
        }
        this.name    = name;
        this.size    = size;
        this.deleted = false;
    }

    /** {@inheritDoc} */
    @Override
    public String getName() {
        return name;
    }

    /**
     * {@inheritDoc}
     * Returns the raw size of this file. Returns 0 if the file has been deleted.
     */
    @Override
    public long getSize() {
        return deleted ? 0L : size;
    }

    /**
     * {@inheritDoc}
     * Prints this file's name and size to stdout.
     */
    @Override
    public void list(String indent) {
        System.out.printf("%s- %s (%,d bytes)%s%n",
            indent,
            name,
            size,
            deleted ? " [DELETED]" : ""
        );
    }

    /**
     * {@inheritDoc}
     * Marks this file as deleted. In a real implementation, this would
     * call the OS to unlink the file from the file system.
     */
    @Override
    public void delete() {
        System.out.println("Deleting file: " + name);
        this.deleted = true;
    }

    @Override
    public String toString() {
        return String.format("File{name='%s', size=%d, deleted=%s}", name, size, deleted);
    }
}
```

```java
package com.coursedesign.patterns.structural.composite.filesystem;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Composite node in the file system Composite hierarchy.
 *
 * <p>Represents a directory that can contain both {@link File} instances
 * (leaves) and other {@code Directory} instances (composites). All operations
 * are implemented by delegating to children recursively, which is the
 * hallmark of the Composite pattern.
 *
 * <p>The {@link #getSize()} method demonstrates recursive aggregation:
 * the total size of a directory is the sum of sizes of all its descendants.
 */
public class Directory implements FileSystemComponent {

    private final String                    name;
    private final List<FileSystemComponent> children;

    /**
     * Creates an empty directory with the given name.
     *
     * @param name the directory name (e.g., "src")
     */
    public Directory(String name) {
        this.name     = name;
        this.children = new ArrayList<>();
    }

    /** {@inheritDoc} */
    @Override
    public String getName() {
        return name;
    }

    /**
     * {@inheritDoc}
     *
     * <p>Computes the size of this directory by summing the sizes of all
     * children. This is a recursive operation — child directories will in
     * turn sum their own children, and so on down the tree.
     *
     * <p>Time complexity: O(N) where N is the total number of nodes in the subtree.
     */
    @Override
    public long getSize() {
        return children.stream()
                       .mapToLong(FileSystemComponent::getSize)
                       .sum();
    }

    /**
     * {@inheritDoc}
     *
     * <p>Lists this directory and recursively lists all children with
     * increasing indentation levels to visualize the tree structure.
     */
    @Override
    public void list(String indent) {
        System.out.printf("%s+ %s/ (%,d bytes total)%n", indent, name, getSize());
        for (FileSystemComponent child : children) {
            child.list(indent + "    ");
        }
    }

    /**
     * {@inheritDoc}
     *
     * <p>Deletes all children before deleting the directory itself.
     * This mirrors the behavior of {@code rm -rf} — children are removed
     * first in a depth-first manner.
     */
    @Override
    public void delete() {
        System.out.println("Deleting directory: " + name + "/");
        // Must delete children before the parent — depth-first traversal
        new ArrayList<>(children).forEach(FileSystemComponent::delete);
        children.clear();
        System.out.println("Directory removed: " + name + "/");
    }

    /**
     * Adds a child component (file or directory) to this directory.
     *
     * @param component the child to add; must not be {@code null}
     * @throws IllegalArgumentException if {@code component} is {@code null}
     */
    @Override
    public void add(FileSystemComponent component) {
        if (component == null) {
            throw new IllegalArgumentException("Cannot add null component to directory.");
        }
        children.add(component);
    }

    /**
     * Removes a child component from this directory.
     *
     * @param component the child to remove
     */
    @Override
    public void remove(FileSystemComponent component) {
        children.remove(component);
    }

    /**
     * Returns an unmodifiable view of this directory's children.
     *
     * @return unmodifiable list of children
     */
    @Override
    public List<FileSystemComponent> getChildren() {
        return Collections.unmodifiableList(children);
    }

    @Override
    public String toString() {
        return String.format("Directory{name='%s', children=%d}", name, children.size());
    }
}
```

```java
package com.coursedesign.patterns.structural.composite.filesystem;

/**
 * Demonstrates the File System Composite pattern.
 *
 * <p>Constructs the following tree:
 * <pre>
 * project/
 *   src/
 *     main/
 *       Main.java        (2048 bytes)
 *       Config.java       (512 bytes)
 *     test/
 *       MainTest.java    (1024 bytes)
 *   docs/
 *     README.md           (768 bytes)
 *     CHANGELOG.md        (256 bytes)
 *   build.gradle          (384 bytes)
 * </pre>
 */
public class FileSystemDemo {

    public static void main(String[] args) {

        // ── Build the leaf nodes (individual files) ─────────────────────────
        File mainJava      = new File("Main.java",      2048);
        File configJava    = new File("Config.java",     512);
        File mainTestJava  = new File("MainTest.java",  1024);
        File readmeMd      = new File("README.md",       768);
        File changelogMd   = new File("CHANGELOG.md",    256);
        File buildGradle   = new File("build.gradle",    384);

        // ── Build intermediate composite nodes (subdirectories) ─────────────
        Directory mainDir  = new Directory("main");
        mainDir.add(mainJava);
        mainDir.add(configJava);

        Directory testDir  = new Directory("test");
        testDir.add(mainTestJava);

        Directory srcDir   = new Directory("src");
        srcDir.add(mainDir);
        srcDir.add(testDir);

        Directory docsDir  = new Directory("docs");
        docsDir.add(readmeMd);
        docsDir.add(changelogMd);

        // ── Build the root composite ─────────────────────────────────────────
        Directory project  = new Directory("project");
        project.add(srcDir);
        project.add(docsDir);
        project.add(buildGradle);

        // ── Client code — treats everything uniformly ────────────────────────
        System.out.println("=== File System Listing ===");
        project.list("");

        System.out.println("\n=== Size Queries (uniform interface for leaf and composite) ===");
        System.out.printf("Total project size : %,d bytes%n", project.getSize());
        System.out.printf("src/ size          : %,d bytes%n", srcDir.getSize());
        System.out.printf("Main.java size     : %,d bytes%n", mainJava.getSize());

        System.out.println("\n=== Deleting docs/ directory ===");
        project.remove(docsDir);
        docsDir.delete();

        System.out.println("\n=== File System After Deletion ===");
        project.list("");
        System.out.printf("Project size after docs deletion: %,d bytes%n", project.getSize());
    }
}
```

**Expected output:**

```
=== File System Listing ===
+ project/ (4,992 bytes total)
    + src/ (3,584 bytes total)
        + main/ (2,560 bytes total)
            - Main.java (2,048 bytes)
            - Config.java (512 bytes)
        + test/ (1,024 bytes total)
            - MainTest.java (1,024 bytes)
    + docs/ (1,024 bytes total)
        - README.md (768 bytes)
        - CHANGELOG.md (256 bytes)
    - build.gradle (384 bytes)
```

---

## 7. Implementation 2: Menu System

A restaurant menu system demonstrates Composite in a UI/domain context. A `Menu` can contain `MenuItem` objects (leaves) and other `Menu` objects (sub-menus). Operations include `render()` and `getPrice()`.

```java
package com.coursedesign.patterns.structural.composite.menu;

import java.util.List;

/**
 * Component interface for the Menu Composite pattern.
 *
 * <p>Both {@link MenuItem} (leaf) and {@link Menu} (composite) implement this
 * interface, allowing the waiter, kiosk, or POS system to operate on any
 * menu element without caring about its concrete type.
 */
public interface MenuComponent {

    /**
     * Returns the name of this menu component.
     *
     * @return display name
     */
    String getName();

    /**
     * Returns the description of this menu component.
     *
     * @return human-readable description
     */
    String getDescription();

    /**
     * Returns the price of this menu component.
     * For a {@link MenuItem}, this is the item's price.
     * For a {@link Menu}, this is the sum of all children's prices
     * (useful for fixed-price set menus).
     *
     * @return price in the local currency unit
     */
    double getPrice();

    /**
     * Returns whether this component is vegetarian.
     * For a {@link Menu}, returns {@code true} only if ALL items are vegetarian.
     *
     * @return {@code true} if vegetarian-safe
     */
    boolean isVegetarian();

    /**
     * Renders this menu component. Items render their own details.
     * Menus render a header and recursively render all children.
     *
     * @param depth the nesting depth, used for indentation
     */
    void render(int depth);

    /**
     * Adds a child component. Only meaningful for {@link Menu}.
     * Default throws {@link UnsupportedOperationException}.
     *
     * @param component the child to add
     */
    default void add(MenuComponent component) {
        throw new UnsupportedOperationException(
            getName() + " is a menu item and cannot contain children."
        );
    }

    /**
     * Removes a child component. Only meaningful for {@link Menu}.
     * Default throws {@link UnsupportedOperationException}.
     *
     * @param component the child to remove
     */
    default void remove(MenuComponent component) {
        throw new UnsupportedOperationException(
            getName() + " is a menu item and cannot contain children."
        );
    }

    /**
     * Returns the list of children. Only meaningful for {@link Menu}.
     * Default returns an empty list.
     *
     * @return list of children
     */
    default List<MenuComponent> getChildren() {
        return List.of();
    }
}
```

```java
package com.coursedesign.patterns.structural.composite.menu;

/**
 * Leaf node in the Menu Composite hierarchy.
 *
 * <p>Represents an individual menu item — something a customer can actually
 * order (e.g., "Margherita Pizza", "Caesar Salad"). A {@code MenuItem} has a
 * fixed price and performs all operations directly.
 */
public class MenuItem implements MenuComponent {

    private final String  name;
    private final String  description;
    private final boolean vegetarian;
    private final double  price;

    /**
     * Creates a new menu item.
     *
     * @param name        the item name
     * @param description a brief description
     * @param vegetarian  {@code true} if this item is vegetarian
     * @param price       the price; must be non-negative
     */
    public MenuItem(String name, String description, boolean vegetarian, double price) {
        if (price < 0) {
            throw new IllegalArgumentException("Price cannot be negative: " + price);
        }
        this.name        = name;
        this.description = description;
        this.vegetarian  = vegetarian;
        this.price       = price;
    }

    @Override public String  getName()        { return name; }
    @Override public String  getDescription() { return description; }
    @Override public boolean isVegetarian()   { return vegetarian; }
    @Override public double  getPrice()       { return price; }

    /**
     * Renders this item with proper indentation. Prints name, price,
     * vegetarian flag, and description.
     */
    @Override
    public void render(int depth) {
        String indent = "  ".repeat(depth);
        System.out.printf("%s%-30s  $%6.2f  %s%n",
            indent,
            name + (vegetarian ? " (V)" : ""),
            price,
            description
        );
    }
}
```

```java
package com.coursedesign.patterns.structural.composite.menu;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Composite node in the Menu Composite hierarchy.
 *
 * <p>Represents a menu section or a full menu (e.g., "Starters", "Main Course",
 * "Desserts", or the full "Dinner Menu"). A {@code Menu} can contain both
 * {@link MenuItem} objects and other {@code Menu} objects (sub-menus).
 *
 * <p>Operations like {@link #getPrice()} and {@link #isVegetarian()} are
 * computed by aggregating results from all children — the defining characteristic
 * of the Composite pattern's recursive delegation.
 */
public class Menu implements MenuComponent {

    private final String              name;
    private final String              description;
    private final List<MenuComponent> children;

    /**
     * Creates a new menu section.
     *
     * @param name        the section name (e.g., "Main Course")
     * @param description a brief description of this section
     */
    public Menu(String name, String description) {
        this.name        = name;
        this.description = description;
        this.children    = new ArrayList<>();
    }

    @Override public String getName()        { return name; }
    @Override public String getDescription() { return description; }

    /**
     * Returns the sum of all children's prices.
     * Useful for computing the cost of a fixed-price set menu.
     */
    @Override
    public double getPrice() {
        return children.stream()
                       .mapToDouble(MenuComponent::getPrice)
                       .sum();
    }

    /**
     * Returns {@code true} only if every item in this menu (recursively) is vegetarian.
     */
    @Override
    public boolean isVegetarian() {
        return children.stream().allMatch(MenuComponent::isVegetarian);
    }

    /**
     * Renders the menu section header followed by all children with
     * increasing indentation.
     */
    @Override
    public void render(int depth) {
        String indent = "  ".repeat(depth);
        System.out.println();
        System.out.println(indent + "┌─── " + name.toUpperCase() + " ───────────────────────");
        System.out.println(indent + "│    " + description);
        System.out.println(indent + "│");
        for (MenuComponent child : children) {
            System.out.print(indent + "│  ");
            child.render(depth + 1);
        }
        System.out.println(indent + "└────────────────────────────────────────");
    }

    /**
     * Adds a child component to this menu section.
     *
     * @param component the child menu item or sub-menu to add
     * @throws IllegalArgumentException if {@code component} is {@code null}
     */
    @Override
    public void add(MenuComponent component) {
        if (component == null) {
            throw new IllegalArgumentException("Cannot add null to menu.");
        }
        children.add(component);
    }

    /**
     * Removes a child component from this menu section.
     *
     * @param component the child to remove
     */
    @Override
    public void remove(MenuComponent component) {
        children.remove(component);
    }

    /**
     * Returns an unmodifiable view of this menu's children.
     *
     * @return unmodifiable list of children
     */
    @Override
    public List<MenuComponent> getChildren() {
        return Collections.unmodifiableList(children);
    }
}
```

```java
package com.coursedesign.patterns.structural.composite.menu;

/**
 * Demonstrates the Menu System Composite pattern.
 */
public class MenuDemo {

    public static void main(String[] args) {

        // ── Leaf items ───────────────────────────────────────────────────────
        MenuItem garlic   = new MenuItem("Garlic Bread",       "Toasted with herb butter",    true,   5.99);
        MenuItem soup     = new MenuItem("Tomato Basil Soup",  "Creamy seasonal soup",        true,   7.50);
        MenuItem calamari = new MenuItem("Calamari",           "Crispy fried squid rings",    false,  9.99);

        MenuItem margherita = new MenuItem("Margherita Pizza",  "Classic tomato and mozzarella", true, 14.99);
        MenuItem pasta      = new MenuItem("Penne Arrabbiata", "Spicy tomato penne",            true, 13.50);
        MenuItem salmon     = new MenuItem("Grilled Salmon",   "Atlantic salmon, lemon butter", false, 22.00);
        MenuItem steak      = new MenuItem("Ribeye Steak",     "200g, served with fries",       false, 35.00);

        MenuItem tiramisu = new MenuItem("Tiramisu",    "Classic Italian dessert",      true, 8.50);
        MenuItem gelato   = new MenuItem("Gelato",      "Two scoops, seasonal flavours", true, 6.50);

        // ── Composite sections ───────────────────────────────────────────────
        Menu starters = new Menu("Starters", "To begin your meal");
        starters.add(garlic);
        starters.add(soup);
        starters.add(calamari);

        Menu mains = new Menu("Main Course", "Our signature dishes");
        mains.add(margherita);
        mains.add(pasta);
        mains.add(salmon);
        mains.add(steak);

        Menu desserts = new Menu("Desserts", "Sweet endings");
        desserts.add(tiramisu);
        desserts.add(gelato);

        // ── Root composite: the full dinner menu ────────────────────────────
        Menu dinnerMenu = new Menu("Dinner Menu", "Available from 6pm onwards");
        dinnerMenu.add(starters);
        dinnerMenu.add(mains);
        dinnerMenu.add(desserts);

        // ── Client code operates on the root — uniform interface ─────────────
        System.out.println("======================================");
        System.out.println("      BELLA VISTA RISTORANTE");
        System.out.println("======================================");
        dinnerMenu.render(0);

        System.out.printf("%nTotal value of all menu items: $%.2f%n", dinnerMenu.getPrice());
        System.out.println("Starters section vegetarian: " + starters.isVegetarian());
        System.out.println("Full dinner menu vegetarian: " + dinnerMenu.isVegetarian());
        System.out.println("Desserts section vegetarian: " + desserts.isVegetarian());
    }
}
```

---

## 8. Implementation 3: Organization Hierarchy

This implementation models an organization where `Employee` (leaf) and `Department` (composite) both implement `OrgComponent`. Useful for payroll calculation, headcount reporting, and org-chart rendering.

```java
package com.coursedesign.patterns.structural.composite.org;

import java.util.List;

/**
 * Component interface for the Organization Hierarchy Composite pattern.
 *
 * <p>Both individual {@link Employee} objects and {@link Department} groups
 * implement this interface, allowing HR systems, payroll engines, and
 * org-chart renderers to operate uniformly on any level of the hierarchy.
 */
public interface OrgComponent {

    /**
     * Returns the name of this organizational unit.
     *
     * @return name (employee name or department name)
     */
    String getName();

    /**
     * Returns the total salary budget for this unit.
     * For an {@link Employee}, this is their individual salary.
     * For a {@link Department}, this is the recursive sum of all member salaries.
     *
     * @return total salary in the company's base currency (USD)
     */
    double getSalary();

    /**
     * Returns the total headcount of this unit.
     * For an {@link Employee}, returns 1.
     * For a {@link Department}, returns the recursive count of all employees.
     *
     * @return number of individual contributors (not departments)
     */
    int getHeadcount();

    /**
     * Displays this unit's information. Departments display headers;
     * employees display their details. Indented for tree visualization.
     *
     * @param indent the indentation prefix
     */
    void display(String indent);

    /**
     * Adds a member to this unit. Only meaningful for {@link Department}.
     *
     * @param component the member to add (employee or sub-department)
     * @throws UnsupportedOperationException if this is a leaf (Employee)
     */
    default void add(OrgComponent component) {
        throw new UnsupportedOperationException(getName() + " is an individual contributor.");
    }

    /**
     * Removes a member from this unit. Only meaningful for {@link Department}.
     *
     * @param component the member to remove
     * @throws UnsupportedOperationException if this is a leaf (Employee)
     */
    default void remove(OrgComponent component) {
        throw new UnsupportedOperationException(getName() + " is an individual contributor.");
    }

    /**
     * Returns the direct reports or sub-units of this unit.
     *
     * @return list of direct reports; empty list for leaves
     */
    default List<OrgComponent> getMembers() {
        return List.of();
    }
}
```

```java
package com.coursedesign.patterns.structural.composite.org;

/**
 * Leaf node representing an individual employee.
 *
 * <p>An employee is at the bottom of the organizational hierarchy —
 * they have no direct reports and represent a single unit of headcount
 * and salary budget.
 */
public class Employee implements OrgComponent {

    private final String title;
    private final String name;
    private final double salary;
    private final String department;

    /**
     * Creates an Employee.
     *
     * @param name       employee's full name
     * @param title      job title (e.g., "Senior Software Engineer")
     * @param department home department name (for display)
     * @param salary     annual salary in USD
     */
    public Employee(String name, String title, String department, double salary) {
        this.name       = name;
        this.title      = title;
        this.department = department;
        this.salary     = salary;
    }

    @Override public String getName()    { return name; }
    @Override public double getSalary()  { return salary; }
    @Override public int    getHeadcount() { return 1; }

    /**
     * Displays this employee's details on a single line.
     */
    @Override
    public void display(String indent) {
        System.out.printf("%s[%s] %s — $%,.0f/yr%n", indent, title, name, salary);
    }

    @Override
    public String toString() {
        return String.format("Employee{name='%s', title='%s', salary=$%.0f}", name, title, salary);
    }
}
```

```java
package com.coursedesign.patterns.structural.composite.org;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * Composite node representing a department or organizational unit.
 *
 * <p>A {@code Department} can contain {@link Employee} objects (leaves)
 * and other {@code Department} objects (composites). All aggregation
 * operations — total salary, headcount — work recursively across the entire
 * subtree, regardless of how deep the hierarchy goes.
 *
 * <p>Example hierarchy:
 * <pre>
 * Engineering (Department)
 *   ├── Backend (Department)
 *   │     ├── Alice (Employee)
 *   │     └── Bob   (Employee)
 *   └── Frontend (Department)
 *         └── Charlie (Employee)
 * </pre>
 */
public class Department implements OrgComponent {

    private final String              name;
    private final String              managerName;
    private final List<OrgComponent>  members;

    /**
     * Creates a Department.
     *
     * @param name        the department name (e.g., "Engineering")
     * @param managerName the name of the department head
     */
    public Department(String name, String managerName) {
        this.name        = name;
        this.managerName = managerName;
        this.members     = new ArrayList<>();
    }

    @Override public String getName() { return name; }

    /**
     * Returns the total salary budget for this department by recursively
     * summing the salaries of all members (employees and sub-departments).
     */
    @Override
    public double getSalary() {
        return members.stream()
                      .mapToDouble(OrgComponent::getSalary)
                      .sum();
    }

    /**
     * Returns the total number of individual employees in this department
     * and all its sub-departments.
     */
    @Override
    public int getHeadcount() {
        return members.stream()
                      .mapToInt(OrgComponent::getHeadcount)
                      .sum();
    }

    /**
     * Displays this department's header (name, manager, aggregated stats)
     * followed by all members with increased indentation.
     */
    @Override
    public void display(String indent) {
        System.out.printf("%s%n", indent + "┌─────────────────────────────────");
        System.out.printf("%sDEPARTMENT : %s%n",     indent, name);
        System.out.printf("%sManager    : %s%n",     indent, managerName);
        System.out.printf("%sHeadcount  : %d%n",     indent, getHeadcount());
        System.out.printf("%sBudget     : $%,.0f/yr%n", indent, getSalary());
        System.out.printf("%s%n", indent + "└─────────────────────────────────");
        for (OrgComponent member : members) {
            member.display(indent + "    ");
        }
    }

    /**
     * Adds an employee or sub-department to this department.
     *
     * @param component the member to add
     * @throws IllegalArgumentException if {@code component} is {@code null}
     */
    @Override
    public void add(OrgComponent component) {
        if (component == null) {
            throw new IllegalArgumentException("Cannot add null member to department.");
        }
        members.add(component);
    }

    /**
     * Removes a member from this department.
     *
     * @param component the member to remove
     */
    @Override
    public void remove(OrgComponent component) {
        members.remove(component);
    }

    /**
     * Returns an unmodifiable view of this department's direct members.
     *
     * @return unmodifiable list of direct members
     */
    @Override
    public List<OrgComponent> getMembers() {
        return Collections.unmodifiableList(members);
    }
}
```

```java
package com.coursedesign.patterns.structural.composite.org;

/**
 * Demonstrates the Organization Hierarchy Composite pattern.
 */
public class OrgHierarchyDemo {

    public static void main(String[] args) {

        // ── Individual contributors (leaves) ────────────────────────────────
        Employee alice   = new Employee("Alice Chen",    "Staff Engineer",        "Backend",  185_000);
        Employee bob     = new Employee("Bob Gupta",     "Senior Engineer",       "Backend",  155_000);
        Employee carol   = new Employee("Carol Okafor",  "Senior Engineer",       "Backend",  150_000);
        Employee david   = new Employee("David Kim",     "Senior Engineer",       "Frontend", 148_000);
        Employee eve     = new Employee("Eve Martinez",  "Engineer II",           "Frontend", 125_000);
        Employee frank   = new Employee("Frank Liu",     "Principal Engineer",    "Platform", 195_000);
        Employee grace   = new Employee("Grace Patel",   "DevOps Engineer",       "Platform", 140_000);
        Employee henry   = new Employee("Henry Torres",  "VP Engineering",        "Eng Exec", 280_000);

        // ── Sub-departments (intermediary composites) ────────────────────────
        Department backend = new Department("Backend Engineering", "Alice Chen");
        backend.add(alice);
        backend.add(bob);
        backend.add(carol);

        Department frontend = new Department("Frontend Engineering", "David Kim");
        frontend.add(david);
        frontend.add(eve);

        Department platform = new Department("Platform & Infra", "Frank Liu");
        platform.add(frank);
        platform.add(grace);

        // ── Root composite: full engineering org ─────────────────────────────
        Department engineering = new Department("Engineering", "Henry Torres");
        engineering.add(henry);       // VP sits in the engineering dept
        engineering.add(backend);
        engineering.add(frontend);
        engineering.add(platform);

        // ── Client code — uniform interface regardless of depth ──────────────
        System.out.println("=== ORGANIZATION CHART ===");
        engineering.display("");

        System.out.println("\n=== SUMMARY QUERIES ===");
        System.out.printf("Engineering headcount : %d people%n",  engineering.getHeadcount());
        System.out.printf("Engineering budget    : $%,.0f/yr%n", engineering.getSalary());
        System.out.printf("Backend headcount     : %d people%n",  backend.getHeadcount());
        System.out.printf("Backend budget        : $%,.0f/yr%n", backend.getSalary());
        System.out.printf("Alice's salary        : $%,.0f/yr%n", alice.getSalary());  // Leaf query
        System.out.printf("Alice's headcount     : %d%n",         alice.getHeadcount());

        System.out.println("\n=== RESTRUCTURING: Moving Carol to Platform ===");
        backend.remove(carol);
        platform.add(carol);
        System.out.printf("Backend headcount after restructure : %d%n", backend.getHeadcount());
        System.out.printf("Platform headcount after restructure: %d%n", platform.getHeadcount());
    }
}
```

---

## 9. Transparent vs Safe Composite Design

This is one of the most important and debated design decisions in the Composite pattern, directly relating to the **Liskov Substitution Principle (LSP)**.

### The Dilemma

The `add()`, `remove()`, and `getChildren()` methods are only meaningful for Composite nodes. Leaf nodes have no children. Yet the Composite pattern tempts you to put these methods on the `Component` interface so that clients can call them without casting. This creates a fundamental tension.

### Option A: Transparent Composite (Interface declares child-management methods)

```java
// Component declares add/remove — client never needs to cast
public interface Component {
    void operation();
    void add(Component c);       // declared on Component
    void remove(Component c);    // declared on Component
    List<Component> getChildren();
}

// Leaf: forced to implement add/remove — what should it do?
public class Leaf implements Component {
    @Override
    public void add(Component c) {
        // Option 1: throw at runtime — violates LSP
        throw new UnsupportedOperationException("Leaf has no children.");
        // Option 2: silently ignore — surprising behavior
        // Option 3: return Optional or boolean — changes signature contract
    }
}
```

**Pros:**
- Client code is uniform — no casting needed to manage children.
- Simpler iteration and tree-building code.
- Easier to swap a leaf for a composite without client code changes.

**Cons:**
- Violates LSP: `Leaf` cannot honor the contract of `add()`. The interface promises something the leaf cannot deliver.
- Runtime exceptions are a design smell — the compiler cannot catch calling `leaf.add(child)`.
- Leaf is polluted with methods that are semantically meaningless for it.
- Unit tests must handle the exception case, making the contract harder to specify.

### Option B: Safe Composite (Only Composite declares child-management methods)

```java
// Component only declares what makes sense for BOTH leaf and composite
public interface Component {
    void operation();
    String getName();
}

// Composite: declares add/remove only here
public class Composite implements Component {
    private List<Component> children = new ArrayList<>();

    @Override
    public void operation() {
        children.forEach(Component::operation);
    }

    // Only accessible by explicitly typing as Composite
    public void add(Component c)    { children.add(c); }
    public void remove(Component c) { children.remove(c); }
    public List<Component> getChildren() { return Collections.unmodifiableList(children); }
}

// Leaf: clean — only implements what it can actually do
public class Leaf implements Component {
    @Override
    public void operation() { /* does the work */ }
}
```

**Usage — requires a cast:**
```java
// Building the tree requires knowing the type
Component root = new Composite("root");
((Composite) root).add(new Leaf("a"));     // cast required — less uniform
```

**Pros:**
- Type-safe at compile time — calling `add()` on a `Leaf` is a compile error.
- Clean, honest interfaces — LSP is not violated.
- Leaf has no meaningless methods.
- Better for strictly-typed APIs.

**Cons:**
- Client code must cast to `Composite` to build the tree — loses uniformity.
- Type-checking (`instanceof`) may be needed when traversing.
- More verbose tree-construction code.

### The Verdict

**For tree-building code (construction phase):** Safe Composite is better. You know whether you're building a composite — use the type.

**For tree-traversal/operation code (usage phase):** Transparent Composite is better. Once the tree is built, clients should never need to know whether they hold a leaf or composite.

**Best Practice in Production Code:** Use the **Transparent** design for the operation methods (`operation()`, `getSize()`, `render()`) but use **Safe** design for child management — expose child-management on `Composite` only, not on the `Component` interface. Java's `default` interface methods help here: declare `add()` on the interface with a default implementation that throws, so leaves inherit the throw automatically without explicit implementation.

```java
// Pragmatic hybrid — used in the implementations above
public interface Component {
    void operation();
    // Child management: declared here for transparency, but with sensible defaults
    default void add(Component c)    { throw new UnsupportedOperationException(); }
    default void remove(Component c) { throw new UnsupportedOperationException(); }
    default List<Component> getChildren() { return List.of(); }
}
```

This is the approach taken in all three implementations above — and it is the approach used by the Java AWT/Swing hierarchy (`java.awt.Container` extends `java.awt.Component`).

---

## 10. Handling Operations That Only Make Sense on Composites

Beyond `add/remove`, there are other operations that may only apply to composites: path navigation, child search, parent tracking, and depth queries. Here are idiomatic Java approaches.

### Pattern: findByName — searching the tree

```java
/**
 * Searches for a component by name in the tree rooted at this node.
 * Works on both leaves and composites through the Component interface.
 *
 * @param root the root of the subtree to search
 * @param name the name to search for
 * @return an Optional containing the found component, or empty
 */
public static Optional<FileSystemComponent> findByName(FileSystemComponent root, String name) {
    if (root.getName().equals(name)) {
        return Optional.of(root);
    }
    // Recursively search children — leaves return empty list, so this is safe
    for (FileSystemComponent child : root.getChildren()) {
        Optional<FileSystemComponent> found = findByName(child, name);
        if (found.isPresent()) {
            return found;
        }
    }
    return Optional.empty();
}

// Usage — client does not need to know if root is a file or directory
Optional<FileSystemComponent> result = findByName(projectRoot, "Main.java");
result.ifPresent(node -> System.out.println("Found: " + node.getName()));
```

### Pattern: Parent tracking with a back-reference

If you need to navigate upward (e.g., `getParent()`), add an optional parent reference:

```java
public interface FileSystemComponent {
    // ... existing methods ...

    /**
     * Returns the parent directory of this component, or {@code null}
     * if this is the root.
     */
    FileSystemComponent getParent();

    /**
     * Sets the parent. Called by {@link Directory#add} when a child is added.
     */
    void setParent(FileSystemComponent parent);
}

// In Directory.add():
@Override
public void add(FileSystemComponent component) {
    children.add(component);
    component.setParent(this);   // maintain the back-reference
}
```

**Caution:** Parent tracking introduces a bidirectional reference, which complicates serialization, garbage collection, and deep-copy logic. Only add it when navigation upward is genuinely required.

### Pattern: Depth and path queries

```java
/**
 * Returns the depth of the given node in the tree (root has depth 0).
 * Requires parent tracking (see above).
 */
public static int getDepth(FileSystemComponent node) {
    int depth = 0;
    FileSystemComponent current = node;
    while (current.getParent() != null) {
        depth++;
        current = current.getParent();
    }
    return depth;
}

/**
 * Builds the full path from root to this node.
 */
public static String getFullPath(FileSystemComponent node) {
    Deque<String> parts = new ArrayDeque<>();
    FileSystemComponent current = node;
    while (current != null) {
        parts.addFirst(current.getName());
        current = current.getParent();
    }
    return String.join("/", parts);
}
```

### Pattern: Visitor for type-specific operations without casting

When you need different behavior on leaves vs. composites without instanceof:

```java
/**
 * Visitor interface — dispatches on the concrete type without casting.
 */
public interface FileSystemVisitor {
    void visitFile(File file);
    void visitDirectory(Directory directory);
}

// In the Component interface:
void accept(FileSystemVisitor visitor);

// In File:
@Override
public void accept(FileSystemVisitor visitor) { visitor.visitFile(this); }

// In Directory:
@Override
public void accept(FileSystemVisitor visitor) {
    visitor.visitDirectory(this);
    children.forEach(child -> child.accept(visitor));
}

// Usage: count files only
FileSystemVisitor counter = new FileSystemVisitor() {
    int count = 0;
    @Override public void visitFile(File file) { count++; }
    @Override public void visitDirectory(Directory dir) { /* no-op */ }
};
project.accept(counter);
```

This cleanly separates type-specific logic from the component hierarchy — a natural complement to Composite.

---

## 11. Composite + Iterator Pattern

The Iterator pattern and Composite pattern are natural partners. Composite defines tree structure; Iterator defines how to traverse it. Java's `Iterable<T>` interface makes this elegant.

```java
package com.coursedesign.patterns.structural.composite.iterator;

import com.coursedesign.patterns.structural.composite.filesystem.FileSystemComponent;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.Iterator;
import java.util.NoSuchElementException;

/**
 * An external iterator that performs a depth-first (pre-order) traversal
 * of a file system tree, visiting every node exactly once.
 *
 * <p>Implementing external iteration over a Composite tree is more complex
 * than internal iteration (using recursion) but gives the caller full control
 * over traversal — they can pause, filter, or stop at any point without
 * evaluating the entire tree.
 *
 * <p>This iterator uses an explicit stack (a {@link Deque}) to simulate
 * the recursion that the Composite pattern normally expresses naturally.
 */
public class DepthFirstIterator implements Iterator<FileSystemComponent> {

    private final Deque<FileSystemComponent> stack;

    /**
     * Creates an iterator starting at the given root node.
     *
     * @param root the root of the tree to traverse
     */
    public DepthFirstIterator(FileSystemComponent root) {
        this.stack = new ArrayDeque<>();
        this.stack.push(root);
    }

    /** Returns {@code true} if there are more nodes to visit. */
    @Override
    public boolean hasNext() {
        return !stack.isEmpty();
    }

    /**
     * Returns the next node in depth-first, pre-order traversal order.
     * Children are pushed onto the stack in reverse order so that the
     * first child is visited first (LIFO stack discipline).
     *
     * @throws NoSuchElementException if no more nodes remain
     */
    @Override
    public FileSystemComponent next() {
        if (!hasNext()) {
            throw new NoSuchElementException("No more nodes in the tree.");
        }

        FileSystemComponent current = stack.pop();

        // Push children in reverse order so first child is processed first
        // getChildren() returns empty list for leaves — no special casing needed
        java.util.List<FileSystemComponent> children = current.getChildren();
        for (int i = children.size() - 1; i >= 0; i--) {
            stack.push(children.get(i));
        }

        return current;
    }
}
```

```java
/**
 * Makes the Composite tree itself Iterable using the external iterator.
 * Demonstrates composing Iterator and Composite patterns cleanly.
 */
public class IterableDirectory extends Directory implements Iterable<FileSystemComponent> {

    public IterableDirectory(String name) {
        super(name);
    }

    /**
     * Returns a depth-first iterator over this directory and all descendants.
     *
     * @return iterator starting from this directory
     */
    @Override
    public Iterator<FileSystemComponent> iterator() {
        return new DepthFirstIterator(this);
    }
}
```

```java
// Usage — the for-each loop works because Directory implements Iterable
IterableDirectory project = new IterableDirectory("project");
project.add(srcDir);
project.add(docsDir);

// Iterate over every node — works transparently for leaves and composites
System.out.println("All nodes (depth-first):");
for (FileSystemComponent node : project) {
    System.out.printf("  %s (%s)%n",
        node.getName(),
        node.getChildren().isEmpty() ? "file" : "directory"
    );
}

// Streaming — compose with Java Streams
long totalFiles = StreamSupport.stream(project.spliterator(), false)
    .filter(n -> n.getChildren().isEmpty())   // only leaves
    .count();
System.out.println("Total files: " + totalFiles);

// Find the largest file
StreamSupport.stream(project.spliterator(), false)
    .filter(n -> n.getChildren().isEmpty())
    .max(Comparator.comparingLong(FileSystemComponent::getSize))
    .ifPresent(f -> System.out.println("Largest file: " + f.getName()));
```

---

## 12. Real-World Examples in Java Ecosystem

### 12.1 java.awt.Container — The Original Java Composite

The Java AWT/Swing library is a textbook implementation of the Composite pattern:

```
java.awt.Component  ←─ The Component interface (it's an abstract class in Java)
        │
        ├── java.awt.Button     (Leaf)
        ├── java.awt.Label      (Leaf)
        ├── java.awt.TextField  (Leaf)
        └── java.awt.Container  (Composite — extends Component, holds List<Component>)
                │
                ├── java.awt.Panel     (Concrete Composite)
                ├── java.awt.Frame     (Concrete Composite — top-level window)
                └── javax.swing.JPanel (Concrete Composite in Swing)
```

`Container.add(Component comp)` adds any component — whether a leaf widget or another container. `Container.paint(Graphics g)` paints the container by calling `paint()` on every child. The client code (your application layout) operates on `Component` references uniformly:

```java
// This code does not know or care whether header is a JLabel or a JPanel
Component header = createHeader();
frame.add(header, BorderLayout.NORTH);
```

**Why AWT used a class hierarchy instead of interface:** Java did not have default interface methods until Java 8. In 1996, the only way to provide default behavior in the Component contract was through an abstract class. The principle is identical to the Composite pattern.

### 12.2 XML / HTML DOM

The W3C Document Object Model is a direct implementation of Composite:

```
org.w3c.dom.Node  ←─ Component interface
        │
        ├── org.w3c.dom.Text          (Leaf — text content)
        ├── org.w3c.dom.Attr          (Leaf — attribute)
        ├── org.w3c.dom.Comment       (Leaf — comment node)
        └── org.w3c.dom.Element       (Composite — has child nodes)
                └── org.w3c.dom.Document (Root composite)
```

```java
// Using DOM — uniform interface across all node types
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
DocumentBuilder builder = factory.newDocumentBuilder();
Document doc = builder.parse(new File("config.xml"));

// Client traverses nodes uniformly — does not know if it's Element or Text
NodeList children = doc.getDocumentElement().getChildNodes();
for (int i = 0; i < children.getLength(); i++) {
    Node child = children.item(i);
    // child could be Element, Text, Comment — processed uniformly
    System.out.println(child.getNodeName() + ": " + child.getTextContent());
}
```

### 12.3 JSON Structure — Jackson's JsonNode

Jackson's `JsonNode` hierarchy is a Composite:

```
com.fasterxml.jackson.databind.JsonNode  ←─ Component (abstract class)
        │
        ├── TextNode      (Leaf)
        ├── IntNode       (Leaf)
        ├── BooleanNode   (Leaf)
        ├── NullNode      (Leaf)
        └── ContainerNode (Composite base)
                ├── ObjectNode  (Composite — key-value children)
                └── ArrayNode   (Composite — indexed children)
```

```java
ObjectMapper mapper = new ObjectMapper();
JsonNode root = mapper.readTree(jsonString);

// Client operates uniformly — isObject/isArray are helpers, not requirements
root.fields().forEachRemaining(entry -> {
    String key     = entry.getKey();
    JsonNode value = entry.getValue();
    // value could be a leaf (TextNode) or composite (ObjectNode, ArrayNode)
    System.out.println(key + " -> " + value);
});

// Recursive sum of all numeric values in a JSON tree
static double sumNumbers(JsonNode node) {
    if (node.isNumber()) return node.doubleValue();
    double sum = 0;
    for (JsonNode child : node) {    // Iterator works on both ObjectNode and ArrayNode
        sum += sumNumbers(child);
    }
    return sum;
}
```

### 12.4 Spring Boot's ApplicationContext Hierarchy

Spring's `ApplicationContext` supports parent-child context hierarchies, a composite structure:

```java
// Parent context: shared infrastructure beans
AnnotationConfigApplicationContext parentContext = new AnnotationConfigApplicationContext();
parentContext.register(InfrastructureConfig.class);
parentContext.refresh();

// Child context: web-layer beans — can access parent's beans transparently
AnnotationConfigWebApplicationContext childContext = new AnnotationConfigWebApplicationContext();
childContext.setParent(parentContext);  // establishes the Composite relationship
childContext.register(WebConfig.class);
childContext.refresh();

// Child looks up a bean — transparently delegates to parent if not found locally
DataSource ds = childContext.getBean(DataSource.class);
// ds is actually defined in parentContext — the composite tree is transparent
```

This is also why `@SpringBootApplication` creates a root context that is the parent of the `WebApplicationContext` — requests for beans propagate up the parent chain (Composite delegation).

### 12.5 Gradle Build Scripts

Gradle's project hierarchy is a Composite:

```groovy
// Root project (Composite)
rootProject.name = 'my-app'

// Sub-projects (children — themselves Composites or leaves)
include ':core', ':api', ':web', ':data'
```

Gradle operations like `./gradlew build` traverse the project tree and invoke `build` on every node — leaves (subprojects with no further children) compile their own code; composites (multi-module parents) delegate to children.

---

## 13. Trade-offs

### Pros

**1. Uniform client interface**
Client code is dramatically simpler. One method call on the root propagates through the entire tree. No `instanceof` checks, no casting, no branching on node type.

**2. Open for extension**
Adding a new node type (`SymbolicLink`, `ZipArchiveEntry`) requires only a new class that implements `Component`. No existing code changes. This is the Open/Closed Principle in action.

**3. Natural recursion**
Many algorithms over hierarchical data are naturally recursive. Composite externalizes the recursion from client code into the tree structure itself. `getSize()` on a directory is a one-liner; the recursion is handled automatically.

**4. Ease of tree manipulation**
Adding and removing subtrees is trivial. Moving a sub-tree from one parent to another requires one `remove()` and one `add()` call, regardless of the subtree's depth or complexity.

**5. Polymorphic behavior**
Operations can be overridden at any level. A `CompressedDirectory` might override `getSize()` to return the compressed size. The tree naturally accommodates specialization.

### Cons

**1. Overly general interface**
The Component interface must be broad enough to cover both leaves and composites, but this often means declaring methods that make no sense for one or the other. The `add()` method on a `File` is meaningless. Every interface compromise degrades type safety.

**2. LSP violations in Transparent design**
When child-management methods are declared on the Component interface but only implemented meaningfully by Composite, the Liskov Substitution Principle is violated at runtime. Libraries like Java AWT accept this trade-off; production code should be explicit about which design is chosen and why.

**3. Difficulty restricting the tree**
Standard Composite does not let you constrain which types can be children of which composites. A `MenuItem` can accidentally be added to a `Directory`. Type constraints require generics or explicit validation logic, complicating the design.

**4. Performance overhead**
Each node is a separate object. Deep trees with millions of nodes create GC pressure. Recursive traversal generates deep call stacks (risk of `StackOverflowError` for extremely deep trees). For performance-critical code, consider converting the tree to a flat array representation or using tail-call optimization techniques.

**5. Difficulty implementing ordered operations**
If children must be processed in a specific order (e.g., by priority), you must either sort during insertion, sort during traversal, or maintain a sorted collection — none of which the basic Composite pattern specifies. This is an accidental complexity that clients must manage.

**6. Shared nodes break tree semantics**
Composite assumes strict tree ownership (one parent per node). If a node is shared between two parents (turning the tree into a DAG), size computations double-count shared nodes, delete operations can corrupt both parents, and cycles cause infinite recursion. None of this is protected against by the pattern itself.

**7. Composite + Iterator complexity**
External iteration over a Composite tree is significantly more complex than internal (recursive) iteration. You must manually maintain a traversal stack. For bidirectional or level-order traversal, the complexity increases further.

---

## 14. Common Pitfalls and Gotchas

### Pitfall 1: Forgetting to copy the children list before iterating during deletion

```java
// WRONG — ConcurrentModificationException if delete() modifies children
public void delete() {
    for (FileSystemComponent child : children) {
        child.delete();             // this may call remove() on children
    }
    children.clear();
}

// CORRECT — iterate over a copy
public void delete() {
    new ArrayList<>(children).forEach(FileSystemComponent::delete);
    children.clear();
}
```

### Pitfall 2: Cycles in the tree

Standard Composite has no cycle detection. If A is a child of B and B is accidentally added as a child of A, recursive operations loop forever:

```java
// WRONG — this creates an infinite loop
Directory a = new Directory("a");
Directory b = new Directory("b");
a.add(b);
b.add(a);                   // cycle!
a.getSize();                // StackOverflowError
```

**Fix:** Add cycle detection in `add()`:

```java
@Override
public void add(FileSystemComponent component) {
    if (isAncestor(component, this)) {
        throw new IllegalArgumentException(
            "Adding " + component.getName() + " would create a cycle."
        );
    }
    children.add(component);
    component.setParent(this);
}

private boolean isAncestor(FileSystemComponent candidate, FileSystemComponent node) {
    FileSystemComponent current = node.getParent();
    while (current != null) {
        if (current == candidate) return true;
        current = current.getParent();
    }
    return false;
}
```

### Pitfall 3: Mutable shared children

If the same `Leaf` object is added to two different `Directory` objects, a `delete()` on one directory's leaf affects the other's view. Always clarify ownership semantics. For shared objects, consider the Flyweight pattern instead.

### Pitfall 4: Comparing patterns by structure instead of intent

Composite and Decorator have superficially similar structures — both involve a component interface with leaf and wrapper implementations. The difference is **intent**:

- **Composite**: Aggregates a collection of components. Models a part-whole hierarchy. One call to the composite triggers one call on each child. Result is aggregated.
- **Decorator**: Wraps a single component to add behavior. One call to the decorator triggers one call on the wrapped component. Result is transformed, not aggregated.

Do not confuse them. In an interview, explicitly state the intent before describing the structure.

### Pitfall 5: Deep stacks and StackOverflowError

Java's default thread stack is ~512KB. For balanced trees, `getSize()` recursion is safe up to roughly 1000-1500 levels. For extremely deep chains (degenerate trees where each node has one child), you may need iterative implementations:

```java
// Iterative depth-first sum using an explicit stack
public long getSizeIterative() {
    long total = 0;
    Deque<FileSystemComponent> stack = new ArrayDeque<>();
    stack.push(this);
    while (!stack.isEmpty()) {
        FileSystemComponent current = stack.pop();
        if (current.getChildren().isEmpty()) {
            total += current.getSize();
        } else {
            stack.addAll(current.getChildren());
        }
    }
    return total;
}
```

### Pitfall 6: Thread safety

The Composite tree is not thread-safe by default. Concurrent `add()` and iteration will cause `ConcurrentModificationException`. Options:
- Use `CopyOnWriteArrayList` for children (good for read-heavy, rarely-modified trees).
- Use `ReentrantReadWriteLock` for read/write separation.
- Make the tree immutable after construction (safest for concurrent access).

---

## 15. Pattern Comparisons

### Composite vs. Decorator

| Aspect | Composite | Decorator |
|---|---|---|
| **Intent** | Represent part-whole hierarchies; uniform treatment of leaves and groups | Add responsibilities to objects dynamically |
| **Children** | Many children per node | Exactly one wrapped component |
| **Result aggregation** | Aggregates results from all children | Transforms result of the single wrapped component |
| **Structural similarity** | Both use a common Component interface | Both use a common Component interface |
| **Adding behavior?** | No — passes the call through unchanged | Yes — wraps and augments |
| **Classic example** | File system, org chart, GUI widgets | Java I/O streams, logging wrappers |

### Composite vs. Chain of Responsibility

| Aspect | Composite | Chain of Responsibility |
|---|---|---|
| **Intent** | Aggregate a tree of nodes | Pass a request along a chain until handled |
| **Structure** | Tree (one-to-many children) | Linear chain (one successor per handler) |
| **Processing** | Every node processes the request | One node handles; the rest may not |
| **Classic example** | File system sizes, GUI rendering | Request interceptors, middleware pipelines |

### Composite vs. Iterator

| Aspect | Composite | Iterator |
|---|---|---|
| **Intent** | Define tree structure and recursive operations | Provide sequential access to elements |
| **Relation** | Complementary — Iterator traverses a Composite | Complementary — Composite provides structure for Iterator |
| **Responsibility** | Defines how nodes compose | Defines how to visit nodes |

---

## 16. Interview Questions and Model Answers

---

### Q1. What is the Composite pattern and what problem does it solve?

**Model Answer:**

The Composite pattern is a structural design pattern that lets you compose objects into tree structures and then work with these structures as if they were individual objects. The problem it solves is the constant need in client code to distinguish between leaf nodes and container nodes in a hierarchical structure.

Without Composite, any algorithm over a tree structure must constantly check "is this node a leaf or a container?" and branch accordingly. This means every new node type breaks every algorithm that operates on the tree — a violation of the Open/Closed Principle. The code becomes littered with `instanceof` checks and casts.

The Composite pattern resolves this by defining a common `Component` interface that both leaves and composites implement. Composites hold a collection of `Component` objects and implement operations by delegating to children. Leaves implement operations directly. The client code operates only on `Component` — it never cares whether it has a leaf or a composite.

The defining characteristic is that the same call on a leaf and on a composite have the same signature but different behavior: a leaf does work; a composite delegates to children and aggregates results. This recursive delegation is what makes algorithms over trees so clean under Composite.

Classic examples include: file systems (File and Directory both implement `FileSystemComponent`), GUI widget hierarchies (Button and Panel both implement `Component`), and organizational hierarchies (Employee and Department both implement `OrgComponent`).

---

### Q2. Explain the difference between the Transparent Composite and the Safe Composite design. Which is better?

**Model Answer:**

This is one of the central design debates in the Composite pattern, and it revolves around a LSP vs. usability trade-off.

In the **Transparent Composite** design, child-management methods (`add()`, `remove()`, `getChildren()`) are declared on the `Component` interface itself. This means every implementation — both leaves and composites — must deal with these methods. Leaves typically throw `UnsupportedOperationException`. The benefit is that client code can add children to any component reference without casting. The cost is that the interface promises a contract that leaves cannot fulfill, which is a Liskov Substitution Principle violation.

In the **Safe Composite** design, child-management methods are declared only on the `Composite` class, not on the `Component` interface. The interface is clean and honest — it only declares what both leaves and composites can genuinely do. The cost is that tree-building code must cast to `Composite` when adding children, and clients who receive a `Component` reference cannot call `add()` without an `instanceof` check and a cast.

Neither is strictly better — the choice depends on the use case.

Prefer Transparent when: The primary use case is traversal and operation invocation. Tree construction is done in factory or builder code, not in general client code. The runtime exception on leaf is acceptable and well-documented. This is what Java AWT/Swing does — the `Container.add()` method is on `Container`, not on `Component`, so it is technically a Safe design, but the design intent is uniform treatment during rendering.

Prefer Safe when: Compile-time type safety is critical. The API is public and callers should not accidentally call `add()` on a leaf. LSP compliance is non-negotiable. This is more common in domain models.

A practical hybrid approach that works well in modern Java: declare child-management methods on the interface with a default implementation that throws `UnsupportedOperationException`. Leaves inherit the throw without extra code. Composites override to provide real behavior. This gives you the uniform interface (Transparent benefit) while making the exception behavior explicit in the interface contract.

---

### Q3. How does the Composite pattern relate to the Liskov Substitution Principle?

**Model Answer:**

The relationship between Composite and LSP is complex and is one of the most nuanced design trade-offs in the pattern.

LSP states that objects of a subtype must be substitutable for objects of their supertype without altering the correctness of the program. Concretely: if `Component` declares `add(Component c)`, then any implementation of `Component` must support `add()` without breaking client expectations.

In the Transparent Composite design, `Leaf.add()` throws `UnsupportedOperationException`. This is an LSP violation because a client that holds a `Component` reference and calls `add()` on it cannot know whether it will succeed or throw. The contract implied by the interface is broken.

However, "violation" here needs context. LSP permits subtypes to restrict preconditions only if the restriction is documented in the contract. Some practitioners argue that declaring `add()` with a documented default behavior of throwing (rather than successfully adding) is acceptable contract narrowing rather than an LSP violation. This is debatable.

The practical question is: does your system have clients that call `add()` on a generic `Component` reference without knowing if it's a leaf? If yes, those clients cannot handle the exception uniformly, and you have a real LSP violation causing a runtime defect. If tree construction code always knows it's talking to a Composite (because it built the tree), the Transparent design is safe in practice even if it's technically an LSP violation in theory.

The Safe Composite design avoids the LSP issue entirely by removing `add()` from the interface, but at the cost of requiring casts in tree-construction code.

In senior design discussions, you should acknowledge the trade-off explicitly, state which design you chose and why, and ensure that if you use the Transparent design, the interface Javadoc clearly documents the exception behavior.

---

### Q4. How would you implement Composite for a use case where different node types should only accept specific child types?

**Model Answer:**

Standard Composite does not provide type constraints on children — any `Component` can be added to any `Composite`. To enforce constraints, you have several options:

**Option 1: Generics on the Component interface**

```java
// T is the specific component type this composite manages
public interface Component<T extends Component<T>> {
    void add(T child);
    List<T> getChildren();
}

// Strongly typed — a Directory<File> can only contain Files
public class Directory implements Component<FileSystemComponent> {
    // ...
}
```

This works but can become complex with recursive generic bounds (`T extends Component<T>`). It also prevents mixing different concrete types in the same composite.

**Option 2: Runtime validation in `add()`**

```java
@Override
public void add(MenuComponent component) {
    if (component instanceof Menu && !(this instanceof RootMenu)) {
        throw new IllegalArgumentException(
            "Sub-menus can only be added to the RootMenu."
        );
    }
    children.add(component);
}
```

Simple, but relies on `instanceof` and is fragile — adding new types requires updating validation logic everywhere.

**Option 3: Type tokens in the factory method**

Use a factory/builder to construct the tree, embedding type constraints in the builder rather than the component:

```java
public class MenuBuilder {
    public MenuBuilder addItem(MenuItem item) { /* only allows MenuItem leaves */ }
    public MenuBuilder addSection(Menu section) { /* only allows Menu composites */ }
    public Menu build() { /* ... */ }
}
```

This is the cleanest approach for public APIs — constraints live in construction logic rather than the component model.

For most production cases, Option 3 (constrained factory/builder) is the right choice. It separates construction concerns from the component's runtime behavior and keeps the Composite hierarchy clean.

---

### Q5. What is the time and space complexity of typical Composite operations?

**Model Answer:**

Composite operations are inherently recursive, so their complexity is a function of the tree shape:

**Time complexity:**

- `getSize()` / `operation()` on a tree with N total nodes: **O(N)**. Every node is visited exactly once. The recursion unwinds depth-first.
- `add(child)` to a composite: **O(1)** for the `ArrayList` add operation (amortized). O(N) if the child list is a sorted structure.
- `remove(child)` from a composite: **O(C)** where C is the number of direct children (linear scan of the children list). Can be O(log C) with a sorted set.
- `findByName()` (search): **O(N)** in the worst case — must visit every node.
- `delete()`: **O(N)** — must visit and delete every node in the subtree.

**Space complexity:**

- The tree structure itself: **O(N)** for N nodes. Each node holds a reference to its children list.
- Recursive call stack during traversal: **O(D)** where D is the tree depth. For a balanced tree with N nodes, D = O(log N). For a degenerate chain (each node has one child), D = O(N), which risks `StackOverflowError` for large N.
- Iterative traversal using an explicit stack: **O(W)** where W is the maximum width of the tree (max number of siblings at any level). For a balanced tree, W ≈ N/2 in the worst case.

**Practical implications:**

For most realistic trees (file systems, org charts, UI hierarchies) the depth is well under a few hundred levels, making recursive traversal safe. For pathologically deep trees (e.g., deeply nested JSON from untrusted input), iterative traversal is safer.

For performance-sensitive code (e.g., computing `getSize()` on a large file system tree on every request), caching the aggregated value and invalidating on `add()` / `remove()` is the production approach. This reduces the per-query cost from O(N) to O(1) at the cost of additional bookkeeping.

---

### Q6. How does Java's java.awt.Container implement the Composite pattern, and what design choices did the Java designers make?

**Model Answer:**

Java's AWT/Swing library is a canonical, production-scale example of the Composite pattern. Here is how it maps:

- **Component** → `java.awt.Component` (abstract class, not interface — Java had no default interface methods in 1996)
- **Leaf** → `java.awt.Button`, `java.awt.Label`, `java.awt.TextField`, `javax.swing.JButton`, etc.
- **Composite** → `java.awt.Container` (extends `Component`, adds `add()`, `remove()`, `getComponents()`)

Notable design choices:

**Safe Composite design**: The `add()` and `remove()` methods are declared on `Container`, not on `Component`. This is the Safe design — you need a `Container` reference to add children. Java chose LSP compliance over interface uniformity.

**Abstract class over interface**: In 1996, Java had no default methods, so the only way to provide shared behavior in the Component contract was an abstract class. Today, this would likely be an interface with default methods.

**The `paint(Graphics g)` method**: `Component.paint()` is abstract (overridden by leaves to draw themselves). `Container.paint()` calls `super.paint()` and then calls `paint()` on each child. This is the Composite delegation pattern for rendering.

**Layout managers**: Containers do not position children themselves. They delegate layout to a `LayoutManager` strategy object. This is Composite (for tree structure) + Strategy (for layout algorithm) combined — a common real-world pattern combination.

**Event propagation**: Mouse events bubble up from the leaf that received them to their container, and then to the container's container, etc. This is the Chain of Responsibility pattern layered on top of the Composite tree.

The AWT design shows that in production systems, patterns are rarely used in isolation. Composite provides the structural backbone, while Strategy, Chain of Responsibility, and Template Method layer behavioral concerns on top of it.

---

### Q7. Can you walk through how you would design a Composite for a UI rendering engine, and what extensions you would consider for production?

**Model Answer:**

For a UI rendering engine, the Composite maps naturally: every UI element is a `Widget`. A `Label` or `Button` is a leaf; a `Panel`, `ScrollView`, or `Window` is a composite that contains other widgets.

The base interface would look like:

```java
public interface Widget {
    void render(RenderContext ctx);
    Dimension preferredSize();
    void layout(Rectangle bounds);       // position this widget in given bounds
    void handleEvent(UIEvent event);     // propagate to children
    Rectangle getBounds();
}
```

For production, the following extensions are critical:

**Dirty-region tracking:** Re-rendering the entire tree on every frame is expensive. Add a `markDirty(Rectangle region)` method and `isDirty()` flag. Composites propagate `markDirty` to the root; the render engine only repaints dirty regions. This is the technique used by Android's View system and CSS's invalidation model.

**Z-ordering:** Children are typically rendered in order (later children painted on top). Composites must maintain an ordered list and support `bringToFront()` / `sendToBack()` operations.

**Clipping:** Each composite should clip rendering to its own bounds so that children cannot paint outside the parent's visible area. The `RenderContext` carries a clip region that composites push/pop.

**Event bubbling and capturing:** Events travel in two phases — capture (root to target, top-down) and bubble (target back to root, bottom-up). The Composite must support both traversal directions. This requires parent references (back-pointers from child to parent).

**Accessibility tree:** Modern UI engines expose a parallel Composite tree for accessibility (screen readers). The rendering Composite and the accessibility Composite mirror each other but are separate trees, each optimized for their concern.

**Virtual rendering / windowing:** For very large lists (thousands of items), render only the visible viewport. The composite structure handles this by skipping children whose bounds do not intersect the viewport's clip region — the recursive `render()` call becomes `if (bounds.intersects(clipRegion)) { child.render(ctx); }`.

These extensions show that the Composite pattern is the foundation of a production UI engine, but the actual implementation layers many performance, accessibility, and interaction concerns on top of the pure structural pattern.

---

### Q8. What are the most common ways to violate the Composite pattern, and what are the symptoms?

**Model Answer:**

The most common violations and their symptoms:

**Violation 1: Breaking the uniform interface by type-checking**

```java
// BAD — client knows too much about the tree
public long computeSize(FileSystemComponent node) {
    if (node instanceof File) {
        return ((File) node).getSize();
    } else if (node instanceof Directory) {
        long total = 0;
        for (FileSystemComponent child : ((Directory) node).getChildren()) {
            total += computeSize(child);
        }
        return total;
    }
    throw new IllegalStateException("Unknown type");
}
```

Symptom: `instanceof` checks in client code. Every new node type breaks this method. The fix is to push `getSize()` into the interface.

**Violation 2: Circular references without detection**

Adding node A to node B when A is already an ancestor of B creates an infinite loop in any recursive operation. Symptom: `StackOverflowError` during what should be simple tree traversal.

**Violation 3: Shared mutable state in leaf objects**

Adding the same leaf to multiple composites and then mutating the leaf. Symptom: changes in one composite's view of the leaf are visible in another composite — unexpected coupling that is very hard to debug.

**Violation 4: Composite with no children treated as a leaf**

An empty `Directory` and a `File` should have different semantics but might be accidentally treated identically because `getSize()` returns 0 for both. Symptom: empty directories disappear from listings or are confused with files. Fix: distinguish structural identity (leaf vs. composite) from operational values (size = 0).

**Violation 5: Not making the Component interface the primary type**

Declaring method signatures with `Directory` or `Composite` as parameter types instead of `Component`. Symptom: tree-building code must know concrete types; polymorphism is lost; unit tests cannot use mocks.

**Violation 6: Mixing Composite with business logic**

Adding domain-specific logic (e.g., "only directories named 'archive' can be deleted") directly into `Directory.delete()`. Symptom: the deletion logic is scattered across the component hierarchy instead of being in a separate policy/visitor. The Composite structure becomes bloated and impossible to test in isolation. Fix: use the Visitor pattern to separate tree operations from the tree structure.

---

*End of Composite Pattern Reference*

---

> **Next**: [05 — Proxy Pattern](./06-proxy.md) | **Previous**: [03 — Facade Pattern](./03-facade.md)
>
> **Module**: [Structural Patterns](./README.md) | **Course**: [LLD & OOP Mastery](../../README.md)
