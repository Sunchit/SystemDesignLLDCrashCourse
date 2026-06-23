# Abstract Factory Pattern

## Table of Contents

1. [Intent](#intent)
2. [When to Use / When NOT to Use](#when-to-use--when-not-to-use)
3. [UML Class Diagram](#uml-class-diagram)
4. [Core Structure Explained](#core-structure-explained)
5. [Implementation: UI Component Families](#implementation-ui-component-families)
6. [Implementation: Database Provider Family](#implementation-database-provider-family)
7. [Real-World Analogy](#real-world-analogy)
8. [Real-World Examples in Java Ecosystem](#real-world-examples-in-java-ecosystem)
9. [Abstract Factory vs Factory Method](#abstract-factory-vs-factory-method)
10. [The "Adding a New Product Type" Problem](#the-adding-a-new-product-type-problem)
11. [Trade-offs](#trade-offs)
12. [When This Becomes Over-Engineering](#when-this-becomes-over-engineering)
13. [Interview Questions and Model Answers](#interview-questions-and-model-answers)

---

## Intent

The **Abstract Factory** is a creational design pattern that provides an interface for creating **families of related or dependent objects** without specifying their concrete classes.

Think of it as a "factory of factories." Where the Factory Method pattern creates one product, Abstract Factory creates an entire *suite* of products that are designed to work together. The pattern enforces that only compatible objects from the same family are ever assembled together, which is its defining guarantee.

The formal GoF definition: *"Provide an interface for creating families of related or dependent objects without specifying their concrete classes."*

### Core Problem It Solves

Suppose you are building a cross-platform UI toolkit. A button on Windows looks different from a button on macOS. More importantly, Windows-style buttons are designed to work with Windows-style text fields and Windows-style checkboxes — mixing Windows buttons with macOS text fields creates an incoherent, broken user experience.

Without Abstract Factory, client code is littered with `if (platform == WINDOWS)` checks every time it needs to create any UI component. Abstract Factory eliminates all of those checks by making the entire family selection happen in exactly one place, at startup.

### Key Vocabulary

| Term | Meaning |
|------|---------|
| **Abstract Factory** | The interface declaring creation methods for each abstract product type |
| **Concrete Factory** | Implements the abstract factory; creates a complete family of products |
| **Abstract Product** | Interface for a type of product object (e.g., `Button`) |
| **Concrete Product** | Platform-specific implementation of a product (e.g., `WindowsButton`) |
| **Client** | Uses only the abstract factory and abstract product interfaces |

---

## When to Use / When NOT to Use

### Use Abstract Factory When

- **You need families of related objects** that must be used together. The word "family" is key — if your products have no inter-dependency constraint, you probably do not need this pattern.
- **You want to enforce consistency** among products. Abstract Factory makes it structurally impossible to mix a `WindowsButton` with a `MacTextField` because both are created by the same factory.
- **You need to switch entire product families at runtime** without modifying client code. Swapping `new WindowsFactory()` for `new MacFactory()` in one place changes everything.
- **You are building a platform-independent system** (database drivers, UI toolkits, cloud provider SDKs) where each platform has a coherent set of implementations.
- **You are configuring a system from external input** (e.g., a config file says `db=mysql`) and need to wire up the correct family of objects before any business logic runs.

### Do NOT Use Abstract Factory When

- **You only have one product type.** Use Factory Method instead. Abstract Factory is designed for multiple product types that belong together.
- **Your products are independent of each other.** If a `Button` and a `Logger` have no reason to belong to the same family, Abstract Factory adds indirection with no benefit.
- **You have only one platform or one implementation.** Introducing the pattern when there is only one concrete factory is premature abstraction that makes code harder to navigate.
- **The product family rarely or never changes.** If you are certain that only one platform will ever be supported, the pattern adds complexity with no payoff.
- **You need to add new product types frequently.** This is the pattern's Achilles heel (see the "Adding a New Product Type" problem below). If your product taxonomy is unstable, Abstract Factory locks you into a painful interface-change cycle.
- **Simple `if/else` or a configuration map would suffice.** Not every platform switch needs a full creational pattern. Apply it when the family concept appears organically in the domain.

---

## UML Class Diagram

```
+---------------------------+         +---------------------------+
|    <<interface>>          |         |    <<interface>>          |
|    UIComponentFactory     |         |    Button                 |
+---------------------------+         +---------------------------+
| + createButton(): Button  |         | + render(): void          |
| + createTextField():      |         | + onClick(): void         |
|     TextField             |         +---------------------------+
| + createCheckbox():       |                    ^
|     Checkbox              |                    |
+---------------------------+         +----------+-----------+
          ^                           |                      |
          |                           |                      |
+---------+----------+   +-----------+--------+ +-----------+--------+
| WindowsFactory     |   | WindowsButton      | | MacButton          |
+--------------------+   +--------------------+ +--------------------+
|+createButton()     |   |+render(): void     | |+render(): void     |
|+createTextField()  |   |+onClick(): void    | |+onClick(): void    |
|+createCheckbox()   |   +--------------------+ +--------------------+
+--------------------+
          |                +---------------------------+
          |                |    <<interface>>          |
+---------+----------+     |    TextField              |
| MacFactory         |     +---------------------------+
+--------------------+     | + setValue(s: String)     |
|+createButton()     |     | + getValue(): String      |
|+createTextField()  |     | + render(): void          |
|+createCheckbox()   |     +---------------------------+
+--------------------+                  ^
                                        |
                           +------------+-----------+
                           |                        |
                +----------+-------+    +-----------+------+
                | WindowsTextField |    | MacTextField     |
                +------------------+    +------------------+
                |+setValue(s)      |    |+setValue(s)      |
                |+getValue()       |    |+getValue()       |
                |+render()         |    |+render()         |
                +------------------+    +------------------+

+---------------------------+
|    <<interface>>          |
|    Checkbox               |
+---------------------------+
| + check(): void           |
| + uncheck(): void         |
| + isChecked(): boolean    |
| + render(): void          |
+---------------------------+
            ^
            |
+-----------+-----------+
|                       |
+--------+--------+  +--+----------+
| WindowsCheckbox |  | MacCheckbox |
+-----------------+  +-------------+
|+check()         |  |+check()     |
|+uncheck()       |  |+uncheck()   |
|+isChecked()     |  |+isChecked() |
|+render()        |  |+render()    |
+-----------------+  +-------------+

          Client
+----------------------+
|  Application         |
+----------------------+           uses
| -factory:            +---------> UIComponentFactory (interface)
|   UIComponentFactory |
| -button: Button      +---------> Button (interface)
| -textField: TextField+---------> TextField (interface)
| -checkbox: Checkbox  +---------> Checkbox (interface)
+----------------------+
```

**Key structural observations:**

1. `Application` (Client) holds a reference only to `UIComponentFactory`, `Button`, `TextField`, and `Checkbox` — all interfaces. It never mentions `WindowsButton` or `MacFactory`.
2. The choice of which concrete factory to instantiate happens **outside** the client — in a factory selector, a config reader, or `main()`.
3. Because `WindowsFactory` creates `WindowsButton`, `WindowsTextField`, and `WindowsCheckbox` together, mixing is prevented at the architectural level.

---

## Core Structure Explained

The Abstract Factory pattern has exactly five participants:

1. **AbstractFactory** (`UIComponentFactory`) — declares a creation method for each abstract product type. There is one method per product in the family.
2. **ConcreteFactory** (`WindowsFactory`, `MacFactory`) — implements every creation method from the abstract factory. Each concrete factory is responsible for exactly one product family variant.
3. **AbstractProduct** (`Button`, `TextField`, `Checkbox`) — declares the interface for a type of product. Client code only interacts through these interfaces.
4. **ConcreteProduct** (`WindowsButton`, `MacButton`, etc.) — implements an abstract product interface for a specific family variant.
5. **Client** (`Application`) — uses only the AbstractFactory and AbstractProduct interfaces. It is completely blind to concrete classes.

The invariant the pattern enforces: **a concrete factory only ever produces products from its own family.**

---

## Implementation: UI Component Families

### Abstract Product Interfaces

```java
// Button.java
package patterns.abstractfactory.ui;

/**
 * AbstractProduct A — the Button abstraction.
 * Client code interacts only through this interface.
 */
public interface Button {
    void render();
    void onClick(Runnable handler);
    String getLabel();
}
```

```java
// TextField.java
package patterns.abstractfactory.ui;

/**
 * AbstractProduct B — the TextField abstraction.
 */
public interface TextField {
    void render();
    void setValue(String value);
    String getValue();
    void setPlaceholder(String placeholder);
}
```

```java
// Checkbox.java
package patterns.abstractfactory.ui;

/**
 * AbstractProduct C — the Checkbox abstraction.
 */
public interface Checkbox {
    void render();
    void check();
    void uncheck();
    boolean isChecked();
    String getLabel();
}
```

### Abstract Factory Interface

```java
// UIComponentFactory.java
package patterns.abstractfactory.ui;

/**
 * AbstractFactory — declares creation methods for the full family of UI products.
 *
 * Every method returns an abstract type. The concrete factory decides
 * which platform-specific class is actually instantiated.
 *
 * Adding a new product type (e.g., Dropdown) requires adding a new
 * method here — which is the main extensibility cost of this pattern.
 */
public interface UIComponentFactory {

    /**
     * Creates a platform-appropriate Button.
     *
     * @param label the button text
     * @return a Button implementation consistent with this factory's platform
     */
    Button createButton(String label);

    /**
     * Creates a platform-appropriate TextField.
     *
     * @param placeholder hint text shown when the field is empty
     * @return a TextField implementation consistent with this factory's platform
     */
    TextField createTextField(String placeholder);

    /**
     * Creates a platform-appropriate Checkbox.
     *
     * @param label the checkbox label text
     * @param initiallyChecked starting checked state
     * @return a Checkbox implementation consistent with this factory's platform
     */
    Checkbox createCheckbox(String label, boolean initiallyChecked);

    /**
     * Returns a human-readable platform name for diagnostics and logging.
     */
    String getPlatformName();
}
```

### Windows Concrete Products

```java
// WindowsButton.java
package patterns.abstractfactory.ui.windows;

import patterns.abstractfactory.ui.Button;

/**
 * ConcreteProduct — Windows-style Button.
 *
 * Renders with Windows Fluent Design conventions: flat borders,
 * accent-color hover, system font.
 */
public class WindowsButton implements Button {

    private final String label;
    private Runnable clickHandler;

    public WindowsButton(String label) {
        if (label == null || label.isBlank()) {
            throw new IllegalArgumentException("Button label must not be blank");
        }
        this.label = label;
    }

    @Override
    public void render() {
        System.out.printf("[Windows Button | Fluent Design] [ %s ]%n", label);
    }

    @Override
    public void onClick(Runnable handler) {
        this.clickHandler = handler;
        System.out.printf("Windows: Click handler registered for button '%s'%n", label);
    }

    @Override
    public String getLabel() {
        return label;
    }

    /**
     * Simulates a click event — used in tests and demos.
     */
    public void simulateClick() {
        System.out.printf("Windows: Button '%s' clicked%n", label);
        if (clickHandler != null) {
            clickHandler.run();
        }
    }

    @Override
    public String toString() {
        return String.format("WindowsButton{label='%s'}", label);
    }
}
```

```java
// WindowsTextField.java
package patterns.abstractfactory.ui.windows;

import patterns.abstractfactory.ui.TextField;

/**
 * ConcreteProduct — Windows-style TextField.
 */
public class WindowsTextField implements TextField {

    private String value = "";
    private String placeholder;

    public WindowsTextField(String placeholder) {
        this.placeholder = placeholder != null ? placeholder : "";
    }

    @Override
    public void render() {
        String display = value.isEmpty() ? "[" + placeholder + "]" : value;
        System.out.printf("[Windows TextField | Segoe UI] |  %-30s  |%n", display);
    }

    @Override
    public void setValue(String value) {
        this.value = value != null ? value : "";
    }

    @Override
    public String getValue() {
        return value;
    }

    @Override
    public void setPlaceholder(String placeholder) {
        this.placeholder = placeholder != null ? placeholder : "";
    }

    @Override
    public String toString() {
        return String.format("WindowsTextField{value='%s', placeholder='%s'}", value, placeholder);
    }
}
```

```java
// WindowsCheckbox.java
package patterns.abstractfactory.ui.windows;

import patterns.abstractfactory.ui.Checkbox;

/**
 * ConcreteProduct — Windows-style Checkbox.
 * Uses Windows toggle/checkbox visual conventions.
 */
public class WindowsCheckbox implements Checkbox {

    private final String label;
    private boolean checked;

    public WindowsCheckbox(String label, boolean initiallyChecked) {
        this.label = label;
        this.checked = initiallyChecked;
    }

    @Override
    public void render() {
        String box = checked ? "[X]" : "[ ]";
        System.out.printf("[Windows Checkbox | Fluent] %s %s%n", box, label);
    }

    @Override
    public void check() {
        this.checked = true;
        System.out.printf("Windows: Checkbox '%s' checked%n", label);
    }

    @Override
    public void uncheck() {
        this.checked = false;
        System.out.printf("Windows: Checkbox '%s' unchecked%n", label);
    }

    @Override
    public boolean isChecked() {
        return checked;
    }

    @Override
    public String getLabel() {
        return label;
    }

    @Override
    public String toString() {
        return String.format("WindowsCheckbox{label='%s', checked=%s}", label, checked);
    }
}
```

### Windows Concrete Factory

```java
// WindowsFactory.java
package patterns.abstractfactory.ui.windows;

import patterns.abstractfactory.ui.Button;
import patterns.abstractfactory.ui.Checkbox;
import patterns.abstractfactory.ui.TextField;
import patterns.abstractfactory.ui.UIComponentFactory;

/**
 * ConcreteFactory 1 — produces the Windows family of UI components.
 *
 * Every object created here belongs to the Windows product family.
 * This factory will never create a MacButton or MacTextField.
 */
public class WindowsFactory implements UIComponentFactory {

    @Override
    public Button createButton(String label) {
        System.out.println("WindowsFactory: Creating WindowsButton");
        return new WindowsButton(label);
    }

    @Override
    public TextField createTextField(String placeholder) {
        System.out.println("WindowsFactory: Creating WindowsTextField");
        return new WindowsTextField(placeholder);
    }

    @Override
    public Checkbox createCheckbox(String label, boolean initiallyChecked) {
        System.out.println("WindowsFactory: Creating WindowsCheckbox");
        return new WindowsCheckbox(label, initiallyChecked);
    }

    @Override
    public String getPlatformName() {
        return "Windows (Fluent Design)";
    }
}
```

### macOS Concrete Products

```java
// MacButton.java
package patterns.abstractfactory.ui.mac;

import patterns.abstractfactory.ui.Button;

/**
 * ConcreteProduct — macOS-style Button.
 * Uses macOS Human Interface Guidelines: rounded corners, vibrancy.
 */
public class MacButton implements Button {

    private final String label;
    private Runnable clickHandler;

    public MacButton(String label) {
        if (label == null || label.isBlank()) {
            throw new IllegalArgumentException("Button label must not be blank");
        }
        this.label = label;
    }

    @Override
    public void render() {
        System.out.printf("[macOS Button | HIG Rounded] ( %s )%n", label);
    }

    @Override
    public void onClick(Runnable handler) {
        this.clickHandler = handler;
        System.out.printf("macOS: Click handler registered for button '%s'%n", label);
    }

    @Override
    public String getLabel() {
        return label;
    }

    public void simulateClick() {
        System.out.printf("macOS: Button '%s' clicked%n", label);
        if (clickHandler != null) {
            clickHandler.run();
        }
    }

    @Override
    public String toString() {
        return String.format("MacButton{label='%s'}", label);
    }
}
```

```java
// MacTextField.java
package patterns.abstractfactory.ui.mac;

import patterns.abstractfactory.ui.TextField;

/**
 * ConcreteProduct — macOS-style TextField.
 * Uses SF Pro font conventions and rounded border style.
 */
public class MacTextField implements TextField {

    private String value = "";
    private String placeholder;

    public MacTextField(String placeholder) {
        this.placeholder = placeholder != null ? placeholder : "";
    }

    @Override
    public void render() {
        String display = value.isEmpty() ? placeholder : value;
        System.out.printf("[macOS TextField | SF Pro] ( %-30s )%n", display);
    }

    @Override
    public void setValue(String value) {
        this.value = value != null ? value : "";
    }

    @Override
    public String getValue() {
        return value;
    }

    @Override
    public void setPlaceholder(String placeholder) {
        this.placeholder = placeholder != null ? placeholder : "";
    }

    @Override
    public String toString() {
        return String.format("MacTextField{value='%s', placeholder='%s'}", value, placeholder);
    }
}
```

```java
// MacCheckbox.java
package patterns.abstractfactory.ui.mac;

import patterns.abstractfactory.ui.Checkbox;

/**
 * ConcreteProduct — macOS-style Checkbox.
 */
public class MacCheckbox implements Checkbox {

    private final String label;
    private boolean checked;

    public MacCheckbox(String label, boolean initiallyChecked) {
        this.label = label;
        this.checked = initiallyChecked;
    }

    @Override
    public void render() {
        String box = checked ? "(●)" : "(○)";
        System.out.printf("[macOS Checkbox | HIG] %s %s%n", box, label);
    }

    @Override
    public void check() {
        this.checked = true;
        System.out.printf("macOS: Checkbox '%s' checked%n", label);
    }

    @Override
    public void uncheck() {
        this.checked = false;
        System.out.printf("macOS: Checkbox '%s' unchecked%n", label);
    }

    @Override
    public boolean isChecked() {
        return checked;
    }

    @Override
    public String getLabel() {
        return label;
    }

    @Override
    public String toString() {
        return String.format("MacCheckbox{label='%s', checked=%s}", label, checked);
    }
}
```

### macOS Concrete Factory

```java
// MacFactory.java
package patterns.abstractfactory.ui.mac;

import patterns.abstractfactory.ui.Button;
import patterns.abstractfactory.ui.Checkbox;
import patterns.abstractfactory.ui.TextField;
import patterns.abstractfactory.ui.UIComponentFactory;

/**
 * ConcreteFactory 2 — produces the macOS family of UI components.
 *
 * Only ever creates Mac-flavored products, ensuring the entire UI
 * is consistent with Apple's Human Interface Guidelines.
 */
public class MacFactory implements UIComponentFactory {

    @Override
    public Button createButton(String label) {
        System.out.println("MacFactory: Creating MacButton");
        return new MacButton(label);
    }

    @Override
    public TextField createTextField(String placeholder) {
        System.out.println("MacFactory: Creating MacTextField");
        return new MacTextField(placeholder);
    }

    @Override
    public Checkbox createCheckbox(String label, boolean initiallyChecked) {
        System.out.println("MacFactory: Creating MacCheckbox");
        return new MacCheckbox(label, initiallyChecked);
    }

    @Override
    public String getPlatformName() {
        return "macOS (Human Interface Guidelines)";
    }
}
```

### Linux Concrete Products and Factory

```java
// LinuxButton.java
package patterns.abstractfactory.ui.linux;

import patterns.abstractfactory.ui.Button;

/**
 * ConcreteProduct — Linux GTK-style Button.
 */
public class LinuxButton implements Button {

    private final String label;
    private Runnable clickHandler;

    public LinuxButton(String label) {
        this.label = label;
    }

    @Override
    public void render() {
        System.out.printf("[Linux Button | GTK3] <  %s  >%n", label);
    }

    @Override
    public void onClick(Runnable handler) {
        this.clickHandler = handler;
    }

    @Override
    public String getLabel() {
        return label;
    }
}
```

```java
// LinuxTextField.java
package patterns.abstractfactory.ui.linux;

import patterns.abstractfactory.ui.TextField;

/**
 * ConcreteProduct — Linux GTK-style TextField.
 */
public class LinuxTextField implements TextField {

    private String value = "";
    private String placeholder;

    public LinuxTextField(String placeholder) {
        this.placeholder = placeholder != null ? placeholder : "";
    }

    @Override
    public void render() {
        String display = value.isEmpty() ? placeholder : value;
        System.out.printf("[Linux TextField | GTK3] { %-30s }%n", display);
    }

    @Override
    public void setValue(String value) { this.value = value != null ? value : ""; }

    @Override
    public String getValue() { return value; }

    @Override
    public void setPlaceholder(String placeholder) { this.placeholder = placeholder; }
}
```

```java
// LinuxCheckbox.java
package patterns.abstractfactory.ui.linux;

import patterns.abstractfactory.ui.Checkbox;

/**
 * ConcreteProduct — Linux GTK-style Checkbox.
 */
public class LinuxCheckbox implements Checkbox {

    private final String label;
    private boolean checked;

    public LinuxCheckbox(String label, boolean initiallyChecked) {
        this.label = label;
        this.checked = initiallyChecked;
    }

    @Override
    public void render() {
        System.out.printf("[Linux Checkbox | GTK3] %s %s%n", checked ? "[v]" : "[ ]", label);
    }

    @Override
    public void check() { this.checked = true; }

    @Override
    public void uncheck() { this.checked = false; }

    @Override
    public boolean isChecked() { return checked; }

    @Override
    public String getLabel() { return label; }
}
```

```java
// LinuxFactory.java
package patterns.abstractfactory.ui.linux;

import patterns.abstractfactory.ui.Button;
import patterns.abstractfactory.ui.Checkbox;
import patterns.abstractfactory.ui.TextField;
import patterns.abstractfactory.ui.UIComponentFactory;

/**
 * ConcreteFactory 3 — produces the Linux GTK family of UI components.
 * Adding this third factory required zero changes to client code or
 * to the other factories — this is the Open/Closed Principle in action.
 */
public class LinuxFactory implements UIComponentFactory {

    @Override
    public Button createButton(String label) {
        return new LinuxButton(label);
    }

    @Override
    public TextField createTextField(String placeholder) {
        return new LinuxTextField(placeholder);
    }

    @Override
    public Checkbox createCheckbox(String label, boolean initiallyChecked) {
        return new LinuxCheckbox(label, initiallyChecked);
    }

    @Override
    public String getPlatformName() {
        return "Linux (GTK3)";
    }
}
```

### Factory Selector (The One Decision Point)

```java
// UIComponentFactoryProvider.java
package patterns.abstractfactory.ui;

import patterns.abstractfactory.ui.linux.LinuxFactory;
import patterns.abstractfactory.ui.mac.MacFactory;
import patterns.abstractfactory.ui.windows.WindowsFactory;

/**
 * Utility that reads the operating system and returns the correct factory.
 *
 * This is the ONLY place in the codebase where concrete factory class names
 * appear. Everything else works through the UIComponentFactory interface.
 *
 * In production this could read from:
 *   - System.getProperty("os.name")
 *   - An environment variable
 *   - A Spring @Configuration bean
 *   - A config file entry
 */
public final class UIComponentFactoryProvider {

    private UIComponentFactoryProvider() {}

    public static UIComponentFactory getFactory() {
        String os = System.getProperty("os.name", "").toLowerCase();

        if (os.contains("win")) {
            return new WindowsFactory();
        } else if (os.contains("mac")) {
            return new MacFactory();
        } else {
            return new LinuxFactory();
        }
    }

    /**
     * Override for testing — allows injecting any factory by name.
     */
    public static UIComponentFactory getFactory(String platformOverride) {
        return switch (platformOverride.toLowerCase()) {
            case "windows" -> new WindowsFactory();
            case "mac", "macos" -> new MacFactory();
            case "linux" -> new LinuxFactory();
            default -> throw new IllegalArgumentException(
                "Unknown platform: " + platformOverride +
                ". Supported: windows, mac, linux"
            );
        };
    }
}
```

### Client Application (Completely Decoupled)

```java
// LoginForm.java
package patterns.abstractfactory.ui;

/**
 * Client — builds and manages a login form entirely through abstract interfaces.
 *
 * This class has ZERO imports from any concrete product package.
 * It does not know whether it is running on Windows, macOS, or Linux.
 * Swap the factory and the entire visual family changes automatically.
 */
public class LoginForm {

    private final Button loginButton;
    private final Button cancelButton;
    private final TextField usernameField;
    private final TextField passwordField;
    private final Checkbox rememberMeCheckbox;
    private final String platformName;

    /**
     * Constructor injection — the factory is provided externally.
     * This makes LoginForm trivially testable with a mock factory.
     */
    public LoginForm(UIComponentFactory factory) {
        this.platformName = factory.getPlatformName();
        this.loginButton = factory.createButton("Log In");
        this.cancelButton = factory.createButton("Cancel");
        this.usernameField = factory.createTextField("Enter username");
        this.passwordField = factory.createTextField("Enter password");
        this.rememberMeCheckbox = factory.createCheckbox("Remember me", false);

        // Wire up behaviour — also platform-agnostic
        this.loginButton.onClick(this::handleLogin);
        this.cancelButton.onClick(this::handleCancel);
    }

    public void render() {
        System.out.println("=".repeat(50));
        System.out.printf("Login Form — Platform: %s%n", platformName);
        System.out.println("=".repeat(50));
        usernameField.render();
        passwordField.render();
        rememberMeCheckbox.render();
        loginButton.render();
        cancelButton.render();
        System.out.println("=".repeat(50));
    }

    public void fillCredentials(String username, String password) {
        usernameField.setValue(username);
        passwordField.setValue(password);
    }

    public void setRememberMe(boolean remember) {
        if (remember) {
            rememberMeCheckbox.check();
        } else {
            rememberMeCheckbox.uncheck();
        }
    }

    private void handleLogin() {
        String username = usernameField.getValue();
        String password = passwordField.getValue();
        boolean remember = rememberMeCheckbox.isChecked();

        System.out.printf(
            "Login attempt: user='%s', rememberMe=%s, platform='%s'%n",
            username, remember, platformName
        );
        // Delegate to authentication service...
    }

    private void handleCancel() {
        System.out.println("Login cancelled by user.");
        usernameField.setValue("");
        passwordField.setValue("");
        rememberMeCheckbox.uncheck();
    }
}
```

### Main — The Composition Root

```java
// Main.java
package patterns.abstractfactory.ui;

/**
 * Composition Root — the only place that knows about concrete factories.
 *
 * Notice: swapping the factory (line marked **) changes the entire
 * visual family of the application. LoginForm does not change at all.
 */
public class Main {

    public static void main(String[] args) {

        // In production: UIComponentFactoryProvider.getFactory() reads os.name
        // For demo: explicitly choose a platform
        System.out.println(">>> Running on Windows platform <<<");
        UIComponentFactory windowsFactory = UIComponentFactoryProvider.getFactory("windows"); // **
        LoginForm windowsForm = new LoginForm(windowsFactory);
        windowsForm.fillCredentials("alice", "secret123");
        windowsForm.setRememberMe(true);
        windowsForm.render();

        System.out.println();

        System.out.println(">>> Running on macOS platform <<<");
        UIComponentFactory macFactory = UIComponentFactoryProvider.getFactory("mac");         // **
        LoginForm macForm = new LoginForm(macFactory);
        macForm.fillCredentials("bob", "p@ssw0rd");
        macForm.setRememberMe(false);
        macForm.render();

        System.out.println();

        System.out.println(">>> Running on Linux platform <<<");
        UIComponentFactory linuxFactory = UIComponentFactoryProvider.getFactory("linux");     // **
        LoginForm linuxForm = new LoginForm(linuxFactory);
        linuxForm.fillCredentials("carol", "hunter2");
        linuxForm.setRememberMe(true);
        linuxForm.render();
    }
}
```

**Expected Output:**

```
>>> Running on Windows platform <<<
WindowsFactory: Creating WindowsButton
WindowsFactory: Creating WindowsButton
WindowsFactory: Creating WindowsTextField
WindowsFactory: Creating WindowsTextField
WindowsFactory: Creating WindowsCheckbox
==================================================
Login Form — Platform: Windows (Fluent Design)
==================================================
[Windows TextField | Segoe UI] |  alice                           |
[Windows TextField | Segoe UI] |  secret123                       |
[Windows Checkbox | Fluent] [X] Remember me
[Windows Button | Fluent Design] [ Log In ]
[Windows Button | Fluent Design] [ Cancel ]
==================================================

>>> Running on macOS platform <<<
...
[macOS TextField | SF Pro] ( bob                            )
[macOS Checkbox | HIG] (○) Remember me
[macOS Button | HIG Rounded] ( Log In )
...
```

---

## Implementation: Database Provider Family

This second example demonstrates Abstract Factory in the infrastructure layer, where products model different aspects of a database connection.

### Abstract Products

```java
// DatabaseConnection.java
package patterns.abstractfactory.db;

import java.sql.SQLException;

/**
 * AbstractProduct A — represents an active connection to a database.
 */
public interface DatabaseConnection {
    void open() throws SQLException;
    void close() throws SQLException;
    boolean isOpen();
    String getConnectionUrl();
    void beginTransaction() throws SQLException;
    void commit() throws SQLException;
    void rollback() throws SQLException;
}
```

```java
// PreparedCommand.java
package patterns.abstractfactory.db;

import java.sql.SQLException;
import java.util.List;

/**
 * AbstractProduct B — represents a parameterized query.
 * Named "Command" to avoid confusion with java.sql.PreparedStatement.
 */
public interface PreparedCommand {
    void setParameter(int index, Object value) throws SQLException;
    DataReader executeQuery() throws SQLException;
    int executeUpdate() throws SQLException;
    void close() throws SQLException;
    String getSql();
}
```

```java
// DataReader.java
package patterns.abstractfactory.db;

import java.sql.SQLException;
import java.util.Optional;

/**
 * AbstractProduct C — represents a forward-only cursor over a result set.
 */
public interface DataReader {
    boolean next() throws SQLException;
    String getString(String columnName) throws SQLException;
    int getInt(String columnName) throws SQLException;
    long getLong(String columnName) throws SQLException;
    boolean getBoolean(String columnName) throws SQLException;
    Optional<Object> getObject(String columnName) throws SQLException;
    void close() throws SQLException;
    boolean isClosed();
}
```

### Abstract Factory Interface

```java
// DatabaseFactory.java
package patterns.abstractfactory.db;

/**
 * AbstractFactory — creates a consistent family of database objects.
 *
 * All three products (Connection, PreparedCommand, DataReader) produced
 * by a single concrete factory are guaranteed to be mutually compatible.
 */
public interface DatabaseFactory {

    /**
     * Creates a new database connection using this provider's driver and URL format.
     */
    DatabaseConnection createConnection(String host, int port, String database,
                                        String username, String password);

    /**
     * Creates a prepared command bound to an existing connection.
     * The connection must already be open.
     */
    PreparedCommand createPreparedCommand(DatabaseConnection connection, String sql);

    /**
     * Wraps a provider-specific result set in a DataReader abstraction.
     * Normally called internally by PreparedCommand.executeQuery(),
     * but exposed here for testing and custom integration scenarios.
     */
    DataReader createDataReader(Object nativeResultSet);

    /**
     * Returns a human-readable description of this provider.
     */
    String getProviderName();

    /**
     * Returns the JDBC URL pattern for this provider.
     * Useful for logging and diagnostics.
     */
    String getJdbcUrlPattern();
}
```

### MySQL Family

```java
// MySQLConnection.java
package patterns.abstractfactory.db.mysql;

import patterns.abstractfactory.db.DatabaseConnection;
import java.sql.SQLException;

/**
 * ConcreteProduct — MySQL connection.
 *
 * In production this would wrap a real java.sql.Connection obtained
 * via DriverManager.getConnection() or a HikariCP pool.
 * For illustration we simulate the behavior.
 */
public class MySQLConnection implements DatabaseConnection {

    private final String host;
    private final int port;
    private final String database;
    private final String username;
    private final String password;
    private boolean open = false;
    private boolean inTransaction = false;

    public MySQLConnection(String host, int port, String database,
                           String username, String password) {
        this.host = host;
        this.port = port;
        this.database = database;
        this.username = username;
        this.password = password;
    }

    @Override
    public void open() throws SQLException {
        if (open) {
            throw new SQLException("MySQL connection already open");
        }
        System.out.printf("MySQL: Connecting to %s:%d/%s as %s%n",
            host, port, database, username);
        // In production: conn = DriverManager.getConnection(getConnectionUrl(), username, password);
        this.open = true;
        System.out.println("MySQL: Connection established");
    }

    @Override
    public void close() throws SQLException {
        if (!open) return;
        if (inTransaction) {
            rollback();
        }
        System.out.println("MySQL: Closing connection");
        this.open = false;
    }

    @Override
    public boolean isOpen() { return open; }

    @Override
    public String getConnectionUrl() {
        return String.format("jdbc:mysql://%s:%d/%s?useSSL=true&serverTimezone=UTC",
            host, port, database);
    }

    @Override
    public void beginTransaction() throws SQLException {
        ensureOpen();
        System.out.println("MySQL: BEGIN TRANSACTION");
        this.inTransaction = true;
    }

    @Override
    public void commit() throws SQLException {
        ensureOpen();
        System.out.println("MySQL: COMMIT");
        this.inTransaction = false;
    }

    @Override
    public void rollback() throws SQLException {
        ensureOpen();
        System.out.println("MySQL: ROLLBACK");
        this.inTransaction = false;
    }

    private void ensureOpen() throws SQLException {
        if (!open) throw new SQLException("MySQL connection is not open");
    }
}
```

```java
// MySQLPreparedCommand.java
package patterns.abstractfactory.db.mysql;

import patterns.abstractfactory.db.DatabaseConnection;
import patterns.abstractfactory.db.DataReader;
import patterns.abstractfactory.db.PreparedCommand;

import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;

/**
 * ConcreteProduct — MySQL parameterized query.
 *
 * Wraps MySQL-specific prepared statement behavior, including
 * MySQL's 1-based parameter indexing and its batch update semantics.
 */
public class MySQLPreparedCommand implements PreparedCommand {

    private final String sql;
    private final List<Object> parameters = new ArrayList<>();
    private boolean closed = false;

    public MySQLPreparedCommand(DatabaseConnection connection, String sql) throws SQLException {
        if (!connection.isOpen()) {
            throw new SQLException("Cannot create PreparedCommand on a closed connection");
        }
        this.sql = sql;
        System.out.printf("MySQL: Preparing statement: %s%n", sql);
    }

    @Override
    public void setParameter(int index, Object value) throws SQLException {
        ensureNotClosed();
        // Grow list to accommodate 1-based index
        while (parameters.size() < index) {
            parameters.add(null);
        }
        parameters.set(index - 1, value);
        System.out.printf("MySQL: Set parameter[%d] = %s%n", index, value);
    }

    @Override
    public DataReader executeQuery() throws SQLException {
        ensureNotClosed();
        System.out.printf("MySQL: Executing query: %s with params %s%n", sql, parameters);
        // In production: return new MySQLDataReader(preparedStatement.executeQuery());
        return new MySQLDataReader();
    }

    @Override
    public int executeUpdate() throws SQLException {
        ensureNotClosed();
        System.out.printf("MySQL: Executing update: %s with params %s%n", sql, parameters);
        // Simulated: return preparedStatement.executeUpdate();
        return 1;
    }

    @Override
    public void close() {
        this.closed = true;
        System.out.println("MySQL: PreparedCommand closed");
    }

    @Override
    public String getSql() { return sql; }

    private void ensureNotClosed() throws SQLException {
        if (closed) throw new SQLException("PreparedCommand is already closed");
    }
}
```

```java
// MySQLDataReader.java
package patterns.abstractfactory.db.mysql;

import patterns.abstractfactory.db.DataReader;

import java.sql.SQLException;
import java.util.List;
import java.util.Map;
import java.util.Optional;

/**
 * ConcreteProduct — MySQL result set reader.
 *
 * In production this wraps java.sql.ResultSet with MySQL-specific
 * type coercions and null handling.
 */
public class MySQLDataReader implements DataReader {

    // Simulated result rows for demonstration
    private final List<Map<String, Object>> rows = List.of(
        Map.of("id", 1L, "username", "alice", "active", true),
        Map.of("id", 2L, "username", "bob",   "active", false)
    );

    private int cursor = -1;
    private boolean closed = false;

    @Override
    public boolean next() throws SQLException {
        ensureNotClosed();
        cursor++;
        return cursor < rows.size();
    }

    @Override
    public String getString(String columnName) throws SQLException {
        ensureNotClosed();
        Object val = getCurrentRow().get(columnName);
        return val != null ? val.toString() : null;
    }

    @Override
    public int getInt(String columnName) throws SQLException {
        ensureNotClosed();
        Object val = getCurrentRow().get(columnName);
        if (val instanceof Number n) return n.intValue();
        throw new SQLException("Cannot convert " + val + " to int");
    }

    @Override
    public long getLong(String columnName) throws SQLException {
        ensureNotClosed();
        Object val = getCurrentRow().get(columnName);
        if (val instanceof Number n) return n.longValue();
        throw new SQLException("Cannot convert " + val + " to long");
    }

    @Override
    public boolean getBoolean(String columnName) throws SQLException {
        ensureNotClosed();
        Object val = getCurrentRow().get(columnName);
        if (val instanceof Boolean b) return b;
        if (val instanceof Number n) return n.intValue() != 0;
        throw new SQLException("Cannot convert " + val + " to boolean");
    }

    @Override
    public Optional<Object> getObject(String columnName) throws SQLException {
        ensureNotClosed();
        return Optional.ofNullable(getCurrentRow().get(columnName));
    }

    @Override
    public void close() throws SQLException {
        this.closed = true;
        System.out.println("MySQL: DataReader closed");
    }

    @Override
    public boolean isClosed() { return closed; }

    private Map<String, Object> getCurrentRow() throws SQLException {
        if (cursor < 0 || cursor >= rows.size()) {
            throw new SQLException("No current row — call next() first");
        }
        return rows.get(cursor);
    }

    private void ensureNotClosed() throws SQLException {
        if (closed) throw new SQLException("DataReader is already closed");
    }
}
```

```java
// MySQLFactory.java
package patterns.abstractfactory.db.mysql;

import patterns.abstractfactory.db.DatabaseConnection;
import patterns.abstractfactory.db.DatabaseFactory;
import patterns.abstractfactory.db.DataReader;
import patterns.abstractfactory.db.PreparedCommand;

import java.sql.SQLException;

/**
 * ConcreteFactory — produces the MySQL family of database objects.
 *
 * Key MySQL-specific behaviors encoded here:
 *   - URL format: jdbc:mysql://host:port/db?useSSL=true&serverTimezone=UTC
 *   - Default port: 3306
 *   - Supports mysql_native_password and caching_sha2_password auth plugins
 */
public class MySQLFactory implements DatabaseFactory {

    @Override
    public DatabaseConnection createConnection(String host, int port, String database,
                                               String username, String password) {
        System.out.println("MySQLFactory: Creating MySQLConnection");
        return new MySQLConnection(host, port, database, username, password);
    }

    @Override
    public PreparedCommand createPreparedCommand(DatabaseConnection connection, String sql) {
        System.out.println("MySQLFactory: Creating MySQLPreparedCommand");
        try {
            return new MySQLPreparedCommand(connection, sql);
        } catch (SQLException e) {
            throw new RuntimeException("Failed to create MySQL PreparedCommand", e);
        }
    }

    @Override
    public DataReader createDataReader(Object nativeResultSet) {
        System.out.println("MySQLFactory: Wrapping ResultSet in MySQLDataReader");
        return new MySQLDataReader();
    }

    @Override
    public String getProviderName() {
        return "MySQL 8.x";
    }

    @Override
    public String getJdbcUrlPattern() {
        return "jdbc:mysql://{host}:{port}/{database}?useSSL=true&serverTimezone=UTC";
    }
}
```

### PostgreSQL Family

```java
// PostgreSQLConnection.java
package patterns.abstractfactory.db.postgresql;

import patterns.abstractfactory.db.DatabaseConnection;
import java.sql.SQLException;

/**
 * ConcreteProduct — PostgreSQL connection.
 *
 * PostgreSQL-specific behaviors:
 *   - URL format: jdbc:postgresql://host:port/db
 *   - Supports schema search_path
 *   - SAVEPOINT support within transactions
 */
public class PostgreSQLConnection implements DatabaseConnection {

    private final String host;
    private final int port;
    private final String database;
    private final String username;
    private final String password;
    private boolean open = false;
    private boolean inTransaction = false;

    public PostgreSQLConnection(String host, int port, String database,
                                String username, String password) {
        this.host = host;
        this.port = port;
        this.database = database;
        this.username = username;
        this.password = password;
    }

    @Override
    public void open() throws SQLException {
        if (open) throw new SQLException("PostgreSQL connection already open");
        System.out.printf("PostgreSQL: Connecting to %s:%d/%s as %s%n",
            host, port, database, username);
        this.open = true;
        System.out.println("PostgreSQL: Connection established (SSL negotiated)");
    }

    @Override
    public void close() throws SQLException {
        if (!open) return;
        if (inTransaction) rollback();
        System.out.println("PostgreSQL: Closing connection");
        this.open = false;
    }

    @Override
    public boolean isOpen() { return open; }

    @Override
    public String getConnectionUrl() {
        return String.format("jdbc:postgresql://%s:%d/%s", host, port, database);
    }

    @Override
    public void beginTransaction() throws SQLException {
        ensureOpen();
        System.out.println("PostgreSQL: BEGIN");
        this.inTransaction = true;
    }

    @Override
    public void commit() throws SQLException {
        ensureOpen();
        System.out.println("PostgreSQL: COMMIT");
        this.inTransaction = false;
    }

    @Override
    public void rollback() throws SQLException {
        ensureOpen();
        System.out.println("PostgreSQL: ROLLBACK");
        this.inTransaction = false;
    }

    private void ensureOpen() throws SQLException {
        if (!open) throw new SQLException("PostgreSQL connection is not open");
    }
}
```

```java
// PostgreSQLPreparedCommand.java
package patterns.abstractfactory.db.postgresql;

import patterns.abstractfactory.db.DatabaseConnection;
import patterns.abstractfactory.db.DataReader;
import patterns.abstractfactory.db.PreparedCommand;

import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;

/**
 * ConcreteProduct — PostgreSQL parameterized query.
 *
 * PostgreSQL uses $1, $2, ... positional parameters in its wire protocol,
 * but JDBC normalizes them to ? placeholders. This implementation reflects
 * PostgreSQL-specific extended query protocol and server-side prepared statements.
 */
public class PostgreSQLPreparedCommand implements PreparedCommand {

    private final String sql;
    private final List<Object> parameters = new ArrayList<>();
    private boolean closed = false;

    public PostgreSQLPreparedCommand(DatabaseConnection connection, String sql) throws SQLException {
        if (!connection.isOpen()) {
            throw new SQLException("Cannot create PreparedCommand on a closed PostgreSQL connection");
        }
        this.sql = sql;
        System.out.printf("PostgreSQL: Server-side preparing statement: %s%n", sql);
    }

    @Override
    public void setParameter(int index, Object value) throws SQLException {
        ensureNotClosed();
        while (parameters.size() < index) parameters.add(null);
        parameters.set(index - 1, value);
        System.out.printf("PostgreSQL: Bind parameter[%d] = %s%n", index, value);
    }

    @Override
    public DataReader executeQuery() throws SQLException {
        ensureNotClosed();
        System.out.printf("PostgreSQL: Execute query: %s%n", sql);
        return new PostgreSQLDataReader();
    }

    @Override
    public int executeUpdate() throws SQLException {
        ensureNotClosed();
        System.out.printf("PostgreSQL: Execute update: %s%n", sql);
        return 1;
    }

    @Override
    public void close() {
        this.closed = true;
        System.out.println("PostgreSQL: PreparedCommand (server-side statement) closed");
    }

    @Override
    public String getSql() { return sql; }

    private void ensureNotClosed() throws SQLException {
        if (closed) throw new SQLException("PostgreSQL PreparedCommand is already closed");
    }
}
```

```java
// PostgreSQLDataReader.java
package patterns.abstractfactory.db.postgresql;

import patterns.abstractfactory.db.DataReader;

import java.sql.SQLException;
import java.util.List;
import java.util.Map;
import java.util.Optional;

/**
 * ConcreteProduct — PostgreSQL result reader.
 *
 * PostgreSQL-specific: handles numeric types (NUMERIC/DECIMAL mapped to
 * BigDecimal), UUID columns, and JSONB columns as Strings.
 */
public class PostgreSQLDataReader implements DataReader {

    private final List<Map<String, Object>> rows = List.of(
        Map.of("id", 1L, "username", "alice", "active", true),
        Map.of("id", 2L, "username", "bob",   "active", false)
    );

    private int cursor = -1;
    private boolean closed = false;

    @Override
    public boolean next() throws SQLException {
        ensureNotClosed();
        cursor++;
        return cursor < rows.size();
    }

    @Override
    public String getString(String columnName) throws SQLException {
        Object val = getCurrentRow().get(columnName);
        return val != null ? val.toString() : null;
    }

    @Override
    public int getInt(String columnName) throws SQLException {
        Object val = getCurrentRow().get(columnName);
        if (val instanceof Number n) return n.intValue();
        throw new SQLException("Cannot coerce PostgreSQL value to int: " + val);
    }

    @Override
    public long getLong(String columnName) throws SQLException {
        Object val = getCurrentRow().get(columnName);
        if (val instanceof Number n) return n.longValue();
        throw new SQLException("Cannot coerce PostgreSQL value to long: " + val);
    }

    @Override
    public boolean getBoolean(String columnName) throws SQLException {
        Object val = getCurrentRow().get(columnName);
        if (val instanceof Boolean b) return b;
        throw new SQLException("Cannot coerce PostgreSQL value to boolean: " + val);
    }

    @Override
    public Optional<Object> getObject(String columnName) throws SQLException {
        return Optional.ofNullable(getCurrentRow().get(columnName));
    }

    @Override
    public void close() {
        this.closed = true;
        System.out.println("PostgreSQL: DataReader (ResultSet) closed");
    }

    @Override
    public boolean isClosed() { return closed; }

    private Map<String, Object> getCurrentRow() throws SQLException {
        if (cursor < 0 || cursor >= rows.size()) {
            throw new SQLException("No current PostgreSQL row — call next() first");
        }
        return rows.get(cursor);
    }

    private void ensureNotClosed() throws SQLException {
        if (closed) throw new SQLException("PostgreSQL DataReader is already closed");
    }
}
```

```java
// PostgreSQLFactory.java
package patterns.abstractfactory.db.postgresql;

import patterns.abstractfactory.db.DatabaseConnection;
import patterns.abstractfactory.db.DatabaseFactory;
import patterns.abstractfactory.db.DataReader;
import patterns.abstractfactory.db.PreparedCommand;

import java.sql.SQLException;

/**
 * ConcreteFactory — produces the PostgreSQL family of database objects.
 *
 * PostgreSQL-specific behaviors:
 *   - URL format: jdbc:postgresql://host:port/db
 *   - Default port: 5432
 *   - Supports server-side named prepared statements
 *   - RETURNING clause support in executeUpdate()
 */
public class PostgreSQLFactory implements DatabaseFactory {

    @Override
    public DatabaseConnection createConnection(String host, int port, String database,
                                               String username, String password) {
        System.out.println("PostgreSQLFactory: Creating PostgreSQLConnection");
        return new PostgreSQLConnection(host, port, database, username, password);
    }

    @Override
    public PreparedCommand createPreparedCommand(DatabaseConnection connection, String sql) {
        System.out.println("PostgreSQLFactory: Creating PostgreSQLPreparedCommand");
        try {
            return new PostgreSQLPreparedCommand(connection, sql);
        } catch (SQLException e) {
            throw new RuntimeException("Failed to create PostgreSQL PreparedCommand", e);
        }
    }

    @Override
    public DataReader createDataReader(Object nativeResultSet) {
        System.out.println("PostgreSQLFactory: Wrapping ResultSet in PostgreSQLDataReader");
        return new PostgreSQLDataReader();
    }

    @Override
    public String getProviderName() {
        return "PostgreSQL 15.x";
    }

    @Override
    public String getJdbcUrlPattern() {
        return "jdbc:postgresql://{host}:{port}/{database}";
    }
}
```

### Database Client (Repository)

```java
// UserRepository.java
package patterns.abstractfactory.db;

import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;

/**
 * Client — a repository that uses only abstract database interfaces.
 *
 * Zero imports from mysql or postgresql packages.
 * Swap the factory in the constructor and it works with a different DB.
 */
public class UserRepository implements AutoCloseable {

    private final DatabaseFactory factory;
    private final DatabaseConnection connection;

    public UserRepository(DatabaseFactory factory, String host, int port,
                          String database, String username, String password)
            throws SQLException {

        this.factory = factory;
        this.connection = factory.createConnection(host, port, database, username, password);
        this.connection.open();

        System.out.printf("UserRepository initialized with provider: %s%n",
            factory.getProviderName());
    }

    public List<String> findActiveUsernames() throws SQLException {
        String sql = "SELECT username FROM users WHERE active = ?";
        List<String> usernames = new ArrayList<>();

        try (PreparedCommand cmd = factory.createPreparedCommand(connection, sql)) {
            cmd.setParameter(1, true);

            try (DataReader reader = cmd.executeQuery()) {
                while (reader.next()) {
                    usernames.add(reader.getString("username"));
                }
            }
        }

        return usernames;
    }

    public boolean createUser(String username, String hashedPassword) throws SQLException {
        String sql = "INSERT INTO users (username, password_hash, active) VALUES (?, ?, ?)";

        connection.beginTransaction();
        try {
            try (PreparedCommand cmd = factory.createPreparedCommand(connection, sql)) {
                cmd.setParameter(1, username);
                cmd.setParameter(2, hashedPassword);
                cmd.setParameter(3, true);
                int rowsAffected = cmd.executeUpdate();

                if (rowsAffected == 1) {
                    connection.commit();
                    return true;
                } else {
                    connection.rollback();
                    return false;
                }
            }
        } catch (SQLException e) {
            connection.rollback();
            throw e;
        }
    }

    @Override
    public void close() throws Exception {
        connection.close();
        System.out.println("UserRepository closed");
    }
}
```

```java
// DatabaseMain.java
package patterns.abstractfactory.db;

import patterns.abstractfactory.db.mysql.MySQLFactory;
import patterns.abstractfactory.db.postgresql.PostgreSQLFactory;

import java.sql.SQLException;
import java.util.List;

/**
 * Demonstrates switching database providers by changing only the factory.
 * UserRepository code does not change at all.
 */
public class DatabaseMain {

    public static void main(String[] args) throws Exception {

        System.out.println("=== Using MySQL ===");
        DatabaseFactory mysqlFactory = new MySQLFactory();
        runDemo(mysqlFactory);

        System.out.println();

        System.out.println("=== Using PostgreSQL ===");
        DatabaseFactory postgresFactory = new PostgreSQLFactory();
        runDemo(postgresFactory);
    }

    private static void runDemo(DatabaseFactory factory) throws Exception {
        try (UserRepository repo = new UserRepository(
                factory, "localhost", 5432, "appdb", "appuser", "secret")) {

            List<String> users = repo.findActiveUsernames();
            System.out.println("Active users: " + users);

            boolean created = repo.createUser("dave", "$2b$12$hashedPasswordHere");
            System.out.println("User created: " + created);
        }
    }
}
```

---

## Real-World Analogy

### The Furniture Store

Imagine a high-end furniture store that sells furniture in two distinct styles: **Victorian** and **Modern**.

- **Victorian** style includes: a Victorian chair (ornate carved wood, velvet upholstery), a Victorian sofa (curved legs, tufted back), and a Victorian coffee table (mahogany with carved edges).
- **Modern** style includes: a Modern chair (chrome frame, leather), a Modern sofa (clean lines, wool fabric), and a Modern coffee table (tempered glass, steel frame).

The store has two interior designers: one who is a specialist in Victorian decor and one who is a specialist in Modern decor.

When you hire the Victorian designer, you are guaranteed that every piece they select — chair, sofa, table — will be aesthetically consistent. They will never mix a Modern chrome chair with a Victorian mahogany table. This is exactly what `WindowsFactory` does: every component it produces belongs to the Windows visual family.

When you hire the Modern designer, everything is clean, contemporary, and consistent. Swapping designers (factories) changes the entire room's aesthetic without changing the layout of the room (the client code).

The furniture pieces map directly:

| Furniture Analogy | Pattern Participant |
|-------------------|---------------------|
| Interior designer | `UIComponentFactory` (Abstract Factory) |
| Victorian designer | `WindowsFactory` (Concrete Factory) |
| Modern designer | `MacFactory` (Concrete Factory) |
| "Chair" concept | `Button` (Abstract Product) |
| Victorian chair | `WindowsButton` (Concrete Product) |
| Modern chair | `MacButton` (Concrete Product) |
| Homeowner | `Application` / `LoginForm` (Client) |

---

## Real-World Examples in Java Ecosystem

### 1. `javax.xml.parsers.DocumentBuilderFactory`

The canonical textbook example. `DocumentBuilderFactory` is the Abstract Factory. It creates `DocumentBuilder` objects (Concrete Products). The concrete factories are the Xerces, JDK-bundled, or other XML parser implementations selected at runtime.

```java
// The client uses only the abstract factory and abstract product
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance(); // selects impl
factory.setNamespaceAware(true);
factory.setValidating(false);

DocumentBuilder builder = factory.newDocumentBuilder(); // creates product
Document doc = builder.parse(new InputSource(new StringReader(xmlString)));
```

`DocumentBuilderFactory.newInstance()` uses `ServiceLoader` internally to discover which parser implementation is on the classpath, then returns the appropriate concrete factory. The caller never imports `org.apache.xerces.jaxp.DocumentBuilderFactoryImpl`.

### 2. `javax.xml.transform.TransformerFactory`

Same pattern for XSLT transformation:

```java
TransformerFactory transformerFactory = TransformerFactory.newInstance();
Transformer transformer = transformerFactory.newTransformer(xsltSource);
transformer.transform(xmlSource, result);
```

The factory creates `Transformer` and `Templates` objects. Saxon, Xalan, and the JDK's built-in XSLT engine all provide concrete factories that produce their respective product families.

### 3. `java.sql` — The Complete Family

`java.sql` is arguably the most impactful Abstract Factory in the Java ecosystem. The family has three main members:

| Abstract Product | MySQL Concrete | PostgreSQL Concrete |
|-----------------|----------------|---------------------|
| `java.sql.Connection` | `com.mysql.cj.jdbc.ConnectionImpl` | `org.postgresql.jdbc.PgConnection` |
| `java.sql.PreparedStatement` | `com.mysql.cj.jdbc.ClientPreparedStatement` | `org.postgresql.jdbc.PgPreparedStatement` |
| `java.sql.ResultSet` | `com.mysql.cj.jdbc.result.ResultSetImpl` | `org.postgresql.jdbc.PgResultSet` |

The Abstract Factory here is `java.sql.Driver` and `javax.sql.DataSource`. Calling `dataSource.getConnection()` returns a `Connection` that internally creates `PreparedStatement` and `ResultSet` objects from the same provider family.

```java
// Abstract factory: DataSource (obtained from Spring context or JNDI)
DataSource ds = ...; // MySQLDataSource or PGSimpleDataSource underneath

// All three products come from the same family — guaranteed compatible
Connection conn = ds.getConnection();
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
ps.setLong(1, userId);
ResultSet rs = ps.executeQuery();
```

The application code is identical whether the underlying database is MySQL or PostgreSQL. Only the `DataSource` configuration differs.

### 4. Spring `ApplicationContext`

Spring's `ApplicationContext` is an Abstract Factory for beans. Different context implementations (`ClassPathXmlApplicationContext`, `AnnotationConfigApplicationContext`, `GenericWebApplicationContext`) create beans that are wired together in a consistent, environment-specific way.

```java
// Switch context type = switch entire configuration family
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

// Products are beans — always consistent within their context
UserService userService = context.getBean(UserService.class);
EmailService emailService = context.getBean(EmailService.class);
```

In test environments you swap the context for a `GenericApplicationContext` with mock beans — zero changes to the services themselves.

### 5. `java.awt.Toolkit`

`java.awt.Toolkit.getDefaultToolkit()` returns a platform-specific AWT toolkit that creates platform-native window, image, clipboard, and drag-and-drop objects. This is the direct predecessor to the UI factory pattern shown in this guide.

---

## Abstract Factory vs Factory Method

These two patterns are closely related and often confused. The key distinction is **scope**: Factory Method creates one product; Abstract Factory creates a family.

### Comparison Table

| Dimension | Factory Method | Abstract Factory |
|-----------|---------------|-----------------|
| **Scope** | Creates one type of product | Creates a family of related products |
| **Intent** | Let subclasses decide which class to instantiate | Ensure families of products are used together |
| **Structure** | One creator class with one factory method | One factory interface with multiple factory methods |
| **Polymorphism axis** | Creator subclass | Factory implementation |
| **Number of products** | One abstract product | Multiple abstract products |
| **Adding a new variant** | Add a new creator subclass | Add a new concrete factory class |
| **Adding a new product type** | Add a new factory method to that one creator | Must add method to the abstract factory interface (breaking change) |
| **Client coupling** | Client is tied to the creator (often via inheritance) | Client is tied only to the factory interface (via composition) |
| **When to prefer** | When you have one type of product and want variation | When you have multiple types that must be consistent |
| **Relationship** | Simpler, often used inside Abstract Factory products | Often uses Factory Methods internally |

### Structural Illustration

```
Factory Method:                  Abstract Factory:

  Creator (abstract)               AbstractFactory (interface)
   + createProduct()                + createProductA()
        |                           + createProductB()
        v                                |
  ConcreteCreator                  ConcreteFactory
   + createProduct()                + createProductA() -> ConcreteProductA1
        |                           + createProductB() -> ConcreteProductB1
        v
  ConcreteProduct              ConcreteFactory2
                                + createProductA() -> ConcreteProductA2
                                + createProductB() -> ConcreteProductB2
```

### Code-Level Difference

```java
// Factory Method: the "factory" is an overridable method in a class
public abstract class DataExporter {
    // Factory Method — subclasses override to pick the format
    protected abstract Formatter createFormatter();

    public void export(List<Record> records) {
        Formatter formatter = createFormatter(); // polymorphism via subclass
        records.forEach(formatter::format);
    }
}

public class CsvExporter extends DataExporter {
    @Override
    protected Formatter createFormatter() { return new CsvFormatter(); }
}
```

```java
// Abstract Factory: the "factory" is an interface with multiple creation methods
public interface ReportFactory {
    Formatter createFormatter();
    Writer createWriter();     // second product type — Abstract Factory territory
    Validator createValidator();
}

public class CsvReportFactory implements ReportFactory {
    @Override public Formatter createFormatter() { return new CsvFormatter(); }
    @Override public Writer createWriter() { return new CsvWriter(); }
    @Override public Validator createValidator() { return new CsvValidator(); }
}
```

**Rule of thumb:** If you find yourself adding multiple factory methods to a Factory Method class because you have multiple related products, you have outgrown Factory Method and should refactor to Abstract Factory.

---

## The "Adding a New Product Type" Problem

This is Abstract Factory's most significant structural weakness, and interviewers frequently ask about it.

### The Problem

Suppose after shipping the UI toolkit with `Button`, `TextField`, and `Checkbox`, the product team wants to add `Dropdown`. The natural instinct is to add `createDropdown()` to `UIComponentFactory`.

```java
// Before — stable interface
public interface UIComponentFactory {
    Button createButton(String label);
    TextField createTextField(String placeholder);
    Checkbox createCheckbox(String label, boolean initiallyChecked);
}

// After — adding Dropdown BREAKS every existing concrete factory
public interface UIComponentFactory {
    Button createButton(String label);
    TextField createTextField(String placeholder);
    Checkbox createCheckbox(String label, boolean initiallyChecked);
    Dropdown createDropdown(String label, List<String> options); // NEW — BREAKING CHANGE
}
```

Now `WindowsFactory`, `MacFactory`, and `LinuxFactory` all fail to compile until each implements `createDropdown()`. If these factories live in different modules or are provided by third parties, this is a serious coordination problem.

### Mitigation Options

**Option 1: Default method in the interface (Java 8+)**

```java
public interface UIComponentFactory {
    // ... existing methods ...

    /**
     * Creates a Dropdown. Concrete factories may override for platform styling.
     * Default returns a generic cross-platform Dropdown.
     */
    default Dropdown createDropdown(String label, List<String> options) {
        return new GenericDropdown(label, options);
    }
}
```

This lets existing factories compile without modification. They get a functional (if unstyled) default. Platform-specific factories can override when ready.

**Option 2: Capability interface / optional factory**

```java
/**
 * Extended factory for richer component types.
 * Only platforms that support advanced components implement this.
 */
public interface AdvancedUIComponentFactory extends UIComponentFactory {
    Dropdown createDropdown(String label, List<String> options);
    DatePicker createDatePicker(String label);
}
```

Client code checks capability:

```java
if (factory instanceof AdvancedUIComponentFactory advancedFactory) {
    Dropdown dropdown = advancedFactory.createDropdown("Country", List.of("US", "UK", "IN"));
    dropdown.render();
} else {
    // fallback — render a TextField with autocomplete hint
    TextField fallback = factory.createTextField("Country");
    fallback.render();
}
```

**Option 3: Accept instability and version the factory**

Treat the interface change as a planned API version bump. Introduce `UIComponentFactoryV2` extending `UIComponentFactoryV1` and migrate incrementally.

### Takeaway

If your product family is stable and you know what the full set of product types is, Abstract Factory is an excellent fit. If new product types are added frequently, the interface-change cost accumulates and you should consider a more flexible mechanism (a registry pattern, a plugin system, or decorator-based composition).

---

## Trade-offs

### Pros

**1. Family Consistency Enforced Structurally**
Mixing a `WindowsButton` with a `MacTextField` is architecturally impossible. The compiler prevents it. This eliminates an entire class of bugs — mismatched UI states, incompatible transaction scopes — without requiring documentation or discipline.

**2. Open/Closed Principle for New Families**
Adding `LinuxFactory` required zero changes to `WindowsFactory`, `MacFactory`, `LoginForm`, or `UIComponentFactory`. Extending with a new variant (a new column, not a new row in the product matrix) is completely non-invasive.

**3. Single Point of Configuration**
The concrete factory is chosen in exactly one place. Changing that choice affects the entire system. For multi-layered applications, this is enormously valuable: `application.properties: ui.platform=mac` is sufficient to rewire everything.

**4. Testability**
Injecting a `MockUIComponentFactory` in unit tests makes it trivial to assert that the client creates and uses products correctly without any rendering or I/O.

**5. Inversion of Control Ready**
Abstract Factory composes naturally with dependency injection. The factory itself is injected as a bean, and all client code uses constructor injection to receive abstract types.

### Cons

**1. Interface Modification Cost for New Product Types**
As described above: adding a new product type to the abstract factory is a breaking interface change that ripples to every concrete factory.

**2. Proliferation of Classes**
A system with 3 platforms and 5 product types requires 1 interface + 3 factories + 5 abstract products + 15 concrete products = 24 classes. This is manageable but can feel disproportionate for simple cases.

**3. Difficulty Partially Overriding**
If `MacFactory` needs to use `WindowsButton` for some reason (an unusual but real scenario), the pattern offers no clean way to do it. The factory is all-or-nothing per family.

**4. Hard to Discover**
Developers tracing a bug to a specific concrete class must follow: client -> factory interface -> concrete factory -> concrete product. The indirection is powerful but increases the mental stack required to understand instantiation.

**5. Factories Must Be Stateless or Carefully Managed**
If a concrete factory accumulates state (connection pools, caches), lifecycle management becomes non-trivial. Factories are often designed as singletons or Spring beans for this reason.

---

## When This Becomes Over-Engineering

Apply the pattern when it earns its complexity. Avoid it when simpler alternatives suffice.

**Smell 1: You have only one concrete factory.**
If `WindowsFactory` is the only factory you will ever need, you have added a factory interface and a layer of indirection for no benefit. Use direct instantiation.

**Smell 2: Your products are not actually a family.**
If `Button` and `AuditLogger` happen to both be needed by a class, but they have no consistency requirement with each other, there is no family. An Abstract Factory adds coupling (both products to the same factory) where none is needed.

**Smell 3: A configuration map or `ServiceLoader` suffices.**
If you just need to choose between two database drivers and the JDBC API already abstracts them, you do not need a custom Abstract Factory on top. The JDK provides one.

**Smell 4: You're wrapping a singleton.**
If the "factory" really just returns a single shared instance, a simple `@Bean` method or a registry map is the right tool.

**Smell 5: The concrete classes are selected at compile time, not runtime.**
If the platform is determined by which JAR is included in the build (not by runtime configuration), a build system profile or module system is the right mechanism, not a runtime factory.

**Practical guideline:** The minimum threshold for Abstract Factory is:
- At least **two** concrete families that are likely to be used simultaneously (e.g., in tests vs. production), AND
- At least **two** product types that must be consistent within a family, AND
- A genuine need to choose the family at **runtime** (or at test time via injection).

If any of these conditions is absent, consider Factory Method, a simple if/else, or direct instantiation.

---

## Interview Questions and Model Answers

### Q1: What is the Abstract Factory Pattern and how does it differ from the Factory Method Pattern?

**Model Answer:**

The Abstract Factory pattern provides an interface for creating *families* of related or dependent objects without specifying their concrete classes. The critical word is "families" — the pattern is designed for scenarios where multiple types of objects must be created together and must be mutually compatible.

Factory Method, by contrast, is a narrower pattern that defines a single overridable method for creating one type of object. It solves the problem of letting subclasses decide which class to instantiate, but it only deals with a single product type. Abstract Factory is essentially a generalization of Factory Method applied to a suite of products simultaneously.

The structural difference is equally telling. Factory Method uses inheritance — a creator class has a factory method that concrete subclasses override. Abstract Factory uses composition — client code holds a reference to a factory interface and calls multiple creation methods on it. This makes Abstract Factory more flexible: the factory can be swapped at runtime by passing a different implementation, whereas Factory Method requires creating a different subclass of the creator.

In practice, Abstract Factory often *contains* factory methods. Each method on `UIComponentFactory` (`createButton()`, `createTextField()`) is technically a Factory Method. But the pattern as a whole is Abstract Factory because those methods collectively produce a consistent family of objects.

A simple heuristic: if you find yourself adding a second factory method to a Factory Method class because you have a second related product, you are sliding toward Abstract Factory territory and should explicitly adopt the pattern.

---

### Q2: Explain the "adding a new product type" problem in Abstract Factory. How do you mitigate it?

**Model Answer:**

The problem arises from a fundamental tension in Abstract Factory: the pattern excels at adding new product families (new concrete factories) but is rigid when adding new product types (new abstract products).

Concretely: if `UIComponentFactory` defines `createButton()`, `createTextField()`, and `createCheckbox()`, and a new `Dropdown` product type is needed, the `createDropdown()` method must be added to the `UIComponentFactory` interface. This is a breaking change — every existing concrete factory (`WindowsFactory`, `MacFactory`, `LinuxFactory`) must implement the new method before the code compiles. In large systems with many factories — including third-party factories — this creates significant coordination overhead.

The pattern's product matrix is inherently asymmetric: it is open on the "families" axis (adding `LinuxFactory` is non-breaking) and closed on the "product types" axis (adding `Dropdown` is breaking).

There are three principal mitigations. First, default methods (Java 8+) can provide a baseline implementation for new product types, allowing existing factories to compile unchanged while platform-specific factories override at their own pace. This works well when a reasonable cross-platform default exists. Second, the product family can be extended via a sub-interface: `AdvancedUIComponentFactory extends UIComponentFactory` adds `createDropdown()`, and only factories that implement it support the advanced component. Client code uses `instanceof` to check capability. Third, for systems where product types change frequently, the Abstract Factory pattern itself may be the wrong choice — a plugin registry or a function-map-based factory may offer more flexibility at the cost of compile-time safety.

The best defense is discipline during interface design: invest in identifying the complete set of product types up front before committing to the Abstract Factory contract.

---

### Q3: How would you implement Abstract Factory in a Spring application?

**Model Answer:**

In a Spring application, Abstract Factory maps naturally to the dependency injection container. The abstract factory interface becomes a Spring bean type, and concrete factories become `@Configuration` classes or `@Bean` methods that are conditionally activated.

The most idiomatic approach uses `@Profile` or `@Conditional` to select the concrete factory bean. For example:

```java
@Configuration
@Profile("windows")
public class WindowsUIConfig {
    @Bean
    public UIComponentFactory uiComponentFactory() {
        return new WindowsFactory();
    }
}

@Configuration
@Profile("mac")
public class MacUIConfig {
    @Bean
    public UIComponentFactory uiComponentFactory() {
        return new MacFactory();
    }
}
```

All client beans receive `UIComponentFactory` by constructor injection:

```java
@Service
public class LoginForm {
    private final UIComponentFactory factory;

    public LoginForm(UIComponentFactory factory) {
        this.factory = factory;
    }
}
```

Activating `spring.profiles.active=mac` in `application.properties` switches the entire product family. No client bean changes.

For database providers, Spring Boot's `DataSourceAutoConfiguration` follows this exact pattern: it reads `spring.datasource.url`, detects the driver, and configures the appropriate DataSource implementation (HikariCP with MySQL driver vs. HikariCP with PostgreSQL driver). The `Connection`, `PreparedStatement`, and `ResultSet` objects produced downstream are all from the same JDBC driver family.

A more dynamic approach uses `@Qualifier` or `@Conditional` with externalized properties, allowing the factory to be selected from a configuration file without code changes. This is the right approach when the factory choice is an operational decision rather than a development-time decision.

---

### Q4: Can you walk through a scenario where you refactored code to use Abstract Factory? What was the trigger?

**Model Answer:**

A common real-world trigger is discovering that `if (platform == X)` checks have proliferated across the codebase. What starts as a single conditional in one place grows: someone needs to create a different type of widget on Windows, so they add a check. A month later a third widget type requires a check. Now three separate classes each contain the same platform check independently. When a fourth platform is added, all three classes must be updated — and inevitably one is missed.

The refactoring proceeds in three phases. First, identify the product family: what are all the types of objects being created conditionally? That becomes the set of abstract products. Second, create the abstract factory interface with one creation method per product type. Third, extract each branch of each `if` statement into a concrete factory class. The `if` block now lives in exactly one place — the factory selector at startup.

The trigger that makes the refactoring feel urgent is usually the second or third `if (platform)` block appearing in unrelated classes. At that point it becomes clear that the decision is being made repeatedly rather than once, and that missing a location when adding a new platform is a genuine regression risk.

In testing, the payoff is immediate. Instead of complex mocking of static factory calls, tests inject a `TestFactory` that returns predictable stub objects with zero platform dependencies. The ability to test client code against a controlled family of objects without any OS-level setup is a strong signal that the abstraction boundary is in the right place.

---

### Q5: What design principles does Abstract Factory embody? Explain each.

**Model Answer:**

Abstract Factory is one of the richest patterns in terms of the design principles it demonstrates.

**Open/Closed Principle (OCP):** The system is open for extension (add `LinuxFactory` without modifying any existing code) and closed for modification (adding a new family does not touch `WindowsFactory`, `MacFactory`, or any client). This is the pattern's headline benefit. The caveat is that it is closed only on the "families" axis; it is open (in the wrong sense) on the "product types" axis.

**Dependency Inversion Principle (DIP):** High-level client modules (`LoginForm`, `UserRepository`) depend on abstractions (`UIComponentFactory`, `Button`, `TextField`), not on concrete classes (`WindowsButton`, `MacTextField`). Concrete classes depend on the same abstractions. Both the client and the concrete factories converge on the abstract types without either depending on the other.

**Single Responsibility Principle (SRP):** Each class has exactly one reason to change. `WindowsFactory` changes only if the Windows product family changes. `LoginForm` changes only if the login form's behavior changes. Neither is burdened with the decision of which platform it runs on.

**Interface Segregation Principle (ISP):** Each abstract product interface (`Button`, `TextField`) is narrow and focused on that product's behavior. `LoginForm` depends on `UIComponentFactory` for creation and on `Button` / `TextField` / `Checkbox` for use — three clean, narrow interfaces rather than one fat class.

**Liskov Substitution Principle (LSP):** Every concrete product must be substitutable for its abstract product. `MacButton` must honor the full contract of `Button`. `MySQLConnection` must honor the full contract of `DatabaseConnection`. LSP violations (e.g., `MacButton.onClick()` throwing `UnsupportedOperationException`) would break the pattern's guarantee.

Understanding these connections demonstrates not just knowledge of the pattern but the ability to reason about design quality, which is what senior engineers are evaluated on.

---

### Q6: How does Abstract Factory improve testability?

**Model Answer:**

Abstract Factory dramatically improves testability because it separates the *decision* of which objects to create from the *use* of those objects. In a system without this pattern, client code directly instantiates concrete classes, making unit testing difficult — you cannot swap in a faster, simpler, or controllable alternative without modifying the source code.

With Abstract Factory, tests inject a mock or stub factory. The mock factory returns test doubles — in-memory implementations, objects with predictable behavior, or Mockito mocks — instead of real Windows buttons or real MySQL connections. This means tests run without a database, without a specific OS, without network access, and without any rendering.

```java
// In a test — no Windows, no macOS, no rendering
@Test
void loginForm_shouldCallLoginHandler_whenLoginButtonClicked() {
    UIComponentFactory mockFactory = new TestUIComponentFactory(); // returns stubs
    LoginForm form = new LoginForm(mockFactory);
    form.fillCredentials("testuser", "testpass");

    // Invoke login directly on the button stub
    form.getLoginButton().simulateClick();

    // Assert behavior, not rendering
    verify(authService).authenticate("testuser", "testpass");
}
```

A `TestUIComponentFactory` is trivial to write: it returns simple in-memory objects that record interactions. This eliminates the need for `@RunWith(PowerMockRunner.class)` or static method mocking, which are symptoms of poor testability.

The deeper principle is that testability and good design are correlated. A class that is hard to test without mocking statics or final classes is usually a class that violates the Dependency Inversion Principle. Abstract Factory is a direct application of DIP, and its testability benefit is a consequence of that sound design.

For integration tests, a `DockerDatabaseFactory` or `EmbeddedDatabaseFactory` can be injected to provide a real but isolated database without changing any application code. The boundary between unit and integration testing becomes a configuration decision rather than a code change.

---

### Q7: What is the relationship between Abstract Factory, the JDBC API, and the ServiceLoader mechanism?

**Model Answer:**

The JDBC API is arguably the most consequential application of Abstract Factory in Java's history. Understanding this connection illuminates both the pattern and the design of a fundamental Java subsystem.

The JDBC Abstract Factory hierarchy works as follows. `javax.sql.DataSource` (or `java.sql.Driver` in older usage) is the Abstract Factory. It creates `java.sql.Connection` objects (Abstract Product A). A `Connection` creates `java.sql.PreparedStatement` objects (Abstract Product B). A `PreparedStatement` creates `java.sql.ResultSet` objects (Abstract Product C). The MySQL Connector/J JAR provides `MySQLDataSource`, which creates `MySQLConnection`, which creates `MySQLPreparedStatement`, which creates `MySQLResultSetImpl`. The PostgreSQL JDBC driver provides its own parallel family. Application code interacts only through the `javax.sql` and `java.sql` abstract types.

The concrete factory selection is handled by the `ServiceLoader` mechanism (since JDBC 4.0 / Java 6). When you call `DriverManager.getConnection(url, ...)`, it uses `ServiceLoader` to discover all `Driver` implementations on the classpath (via `META-INF/services/java.sql.Driver` entries in each driver JAR). The driver whose URL prefix matches is selected and its `connect()` method returns a `Connection`. The application does not import any driver class — the driver JAR's presence on the classpath is the entire configuration.

This design is powerful because JDBC achieved genuine provider independence in 1997, before Spring existed. It is also the reason frameworks like Hibernate, jOOQ, and MyBatis can claim to be database-agnostic: they all sit atop the JDBC abstract layer and never reach through to concrete classes.

The limitation mirrors the Abstract Factory pattern's limitation: the `Connection` and `PreparedStatement` interfaces are extremely stable (they haven't changed structurally in decades) precisely *because* changing them would break every JDBC driver on the planet. New features (like `java.sql.Blob` in JDBC 2.0, or batching) had to be added carefully with backward compatibility in mind.

---

### Q8: Compare Abstract Factory to Dependency Injection. Are they in tension or complementary?

**Model Answer:**

Abstract Factory and Dependency Injection are complementary, not in tension. They operate at different levels of the object creation problem and work best when combined.

Abstract Factory answers the question: *when creating a family of related objects, how do we ensure consistency?* It provides an interface through which compatible objects are created together. The factory itself is an object with creation logic.

Dependency Injection answers: *how do objects acquire their dependencies?* It externalizes the wiring of objects, so that a class does not call `new` on its dependencies but receives them through its constructor or setter.

The combination is natural and powerful. The abstract factory becomes a dependency that is injected:

```java
// Without DI — the class reaches out for its factory
public class LoginForm {
    private final UIComponentFactory factory =
        UIComponentFactoryProvider.getFactory(); // static call, hard to test
}

// With DI — the factory is injected
public class LoginForm {
    public LoginForm(UIComponentFactory factory) { ... } // testable, flexible
}
```

In Spring, the concrete factory is a `@Bean` (selected via `@Profile` or `@Conditional`), and every class that needs to create UI components receives `UIComponentFactory` through constructor injection. The DI container manages the factory's lifecycle (singleton scope is typical). The Abstract Factory manages the consistency of the objects the factory creates.

Where they can feel redundant is in simple cases: if a DI container already manages all object creation via beans, adding an explicit Abstract Factory layer can be unnecessary indirection. Spring beans are themselves managed objects produced by a container that acts as a kind of factory. But when you need to create *multiple instances* of platform-specific objects at runtime (not just once at startup), an Abstract Factory injected as a singleton is the clean solution — it creates many short-lived product objects on demand, which is outside the scope of typical bean management.

The principle: use DI to wire the factory; use Abstract Factory to create the products. The two patterns solve different problems and compose without conflict.

---

*End of Abstract Factory Pattern Guide*
