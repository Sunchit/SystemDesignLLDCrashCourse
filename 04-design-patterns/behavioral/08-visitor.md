# Visitor Pattern

## Table of Contents
1. [Intent and Problem Statement](#1-intent-and-problem-statement)
2. [Double Dispatch — Deep Explanation](#2-double-dispatch--deep-explanation)
3. [When to Use / When NOT to Use](#3-when-to-use--when-not-to-use)
4. [UML Class Diagram](#4-uml-class-diagram)
5. [Implementation 1: Document Element Visitor](#5-implementation-1-document-element-visitor)
6. [Implementation 2: AST Visitor](#6-implementation-2-ast-visitor)
7. [Tax Calculation Example](#7-tax-calculation-example)
8. [Acyclic Visitor Pattern](#8-acyclic-visitor-pattern)
9. [When to AVOID Visitor](#9-when-to-avoid-visitor)
10. [Real-World Framework Examples](#10-real-world-framework-examples)
11. [Trade-offs](#11-trade-offs)
12. [Comparison with Related Patterns](#12-comparison-with-related-patterns)
13. [Interview Questions and Model Answers](#13-interview-questions-and-model-answers)

---

## 1. Intent and Problem Statement

### The Core Problem

Imagine you have a stable object hierarchy — say, a document model with `Paragraph`, `Table`, `Image`, `Header`, and `List` elements. Your system works correctly. Then business requirements arrive: "We need to export to HTML. And PDF. And count words. And calculate file size."

The naive solution is to add `toHtml()`, `toPdf()`, `countWords()`, and `calculateSize()` methods to every class. This violates the **Open/Closed Principle**: you are modifying existing, tested classes every time a new operation is needed. The class hierarchy becomes a dumping ground for every feature anyone ever thought of.

The alternative — writing a single class that does `instanceof` checks and casts — violates the principle of polymorphism and is fragile: add a new element type and the switch statement silently skips it.

### What Visitor Solves

The Visitor pattern separates **algorithms** from the **object structure** they operate on.

- The object hierarchy (`Element` and its subclasses) stays frozen. Each class gets one method: `accept(Visitor v)`.
- Each new operation is encapsulated in a concrete `Visitor` class that has one `visit(ConcreteElement e)` method per element type.
- Adding a new operation = adding a new `Visitor` implementation. Zero changes to existing classes.
- Type safety is enforced at compile time: if a new element type is added to the hierarchy, the compiler forces you to implement `visit` for it in every existing visitor.

### The GoF Definition

> "Represent an operation to be performed on elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates."
> — Design Patterns: Elements of Reusable Object-Oriented Software

### When This Matters at Scale

In production systems — compilers, document processors, game engines, financial calculators — the object structure is defined once and owned by a core team. Operations are contributed by many teams over time. Visitor enforces a clean boundary: the structure team owns `Element`; the feature teams own `Visitor` implementations. This is the exact model used by the Java compiler API (`javax.lang.model`), ANTLR4, and Apache Calcite.

---

## 2. Double Dispatch — Deep Explanation

This is the conceptual heart of the pattern and the most common interview deep-dive topic.

### Single Dispatch: Java's Default

Java is a **single-dispatch** language. When you call a virtual method, the runtime selects the implementation based on **one** type: the runtime type of the receiver object (`this`).

```java
Animal a = new Dog();
a.speak(); // always calls Dog.speak() — dispatch on runtime type of 'a'
```

The JVM implements this via **vtables** (virtual method tables). Every class has a vtable — a fixed-size array of method pointers. When `Dog` extends `Animal` and overrides `speak()`, `Dog`'s vtable entry for `speak` points to `Dog::speak`. The call `a.speak()` looks up the vtable of the object referenced by `a` at runtime and jumps to the correct method.

This is **runtime polymorphism** — the dispatch decision happens at runtime.

### Overloading Is Compile-Time (Static Dispatch)

Method **overloading** is resolved entirely at compile time based on the **static (declared) type** of arguments — not their runtime type. This is critical to understanding why Visitor needs the pattern it uses.

```java
class Printer {
    void print(Animal a)  { System.out.println("Animal"); }
    void print(Dog d)     { System.out.println("Dog");    }
    void print(Cat c)     { System.out.println("Cat");    }
}

Animal a = new Dog();
Printer p = new Printer();
p.print(a); // prints "Animal" — resolved at compile time based on declared type of 'a'
```

Even though `a` holds a `Dog` at runtime, the compiler sees `Animal` and binds to `print(Animal)`. There is no vtable lookup for the argument type. This is **static dispatch**.

### The Double Dispatch Problem

Suppose we want behavior that depends on **two** types simultaneously: the type of the object being operated on AND the type of the operation (visitor). We need:

```
result = f(element_runtime_type, visitor_runtime_type)
```

Neither pure overloading nor pure overriding alone achieves this. We need **two** dispatches.

### How Visitor Achieves Double Dispatch

The trick is to perform two successive single dispatches:

**Dispatch 1** — on the element's runtime type (via `accept`):

```java
Element e = new Paragraph(); // runtime type: Paragraph
e.accept(visitor);           // vtable lookup → Paragraph.accept(visitor)
```

Inside `Paragraph.accept`:

```java
// Paragraph.java
public void accept(DocumentVisitor v) {
    v.visit(this); // 'this' is statically typed as Paragraph
}
```

**Dispatch 2** — on the visitor's runtime type (via `visit`):

```java
v.visit(this); // v's vtable lookup → HtmlRendererVisitor.visit(Paragraph)
               // 'this' is Paragraph — compiler knows the exact overload
```

Because `this` inside `Paragraph.accept` is **statically typed** as `Paragraph` (not `Element`), the compiler selects the correct overload `visit(Paragraph p)`. Then at runtime, the vtable of `v` selects the correct visitor implementation.

### Step-by-Step Vtable Trace

```
Call: element.accept(htmlVisitor)
  → JVM looks up vtable of element's actual class (Paragraph)
  → Finds Paragraph::accept
  → Executes: htmlVisitor.visit(this)  [this = Paragraph instance, static type = Paragraph]
    → JVM looks up vtable of htmlVisitor's actual class (HtmlRendererVisitor)
    → Finds HtmlRendererVisitor::visit(Paragraph)
    → Executes the HTML rendering logic for Paragraph
```

Two vtable lookups. Two runtime type resolutions. That is double dispatch.

### Why You Cannot Fake This With Overloading Alone

```java
// BROKEN — does NOT work
visitor.visit(element); // element is declared as Element
                        // compiler picks visit(Element) regardless of runtime type
                        // overloading is static — no vtable involved for argument
```

### Languages With Native Multiple Dispatch

Some languages (Common Lisp via CLOS, Julia, Clojure via multimethods, Groovy) support **multiple dispatch** natively — the runtime selects a function based on the runtime types of all arguments. In those languages, you do not need the Visitor pattern at all; you just define multimethods. Java, C++, and C# lack this, so Visitor is the idiomatic workaround.

### The accept() Contract

Every concrete element must implement `accept` like this — and critically, must **not** call `super.accept()` or do anything other than `v.visit(this)`:

```java
@Override
public void accept(Visitor v) {
    v.visit(this); // 'this' must be the concrete type, not upcasted
}
```

If a base class implements `accept` as `v.visit(this)` where `this` is typed as `BaseElement`, and subclasses inherit it without overriding, double dispatch breaks — you lose the first dispatch. Every leaf class must override `accept`.

---

## 3. When to Use / When NOT to Use

### Use Visitor When

1. **The object hierarchy is stable** but you need to add operations frequently. The hierarchy is the "closed" axis; operations are the "open" axis.

2. **Operations are unrelated** and would clutter the element classes if added as methods. A `Paragraph` class should not know about PDF rendering, HTML generation, word counting, and spell checking all at once.

3. **You need to accumulate state across elements** in a single traversal. A `WordCountVisitor` naturally accumulates a running total as it visits each element.

4. **You need to apply different algorithms to a composite structure** (see Composite pattern). Visitor and Composite are natural partners.

5. **You are building a compiler, interpreter, or data format transformer** where operations on AST/parse-tree nodes are numerous and evolve independently.

6. **You need compile-time enforcement** that every new element type is handled by every existing visitor. The compiler will reject a concrete visitor that does not implement all `visit` overloads.

### Do NOT Use Visitor When

1. **The object hierarchy changes frequently.** Every new element type forces you to update every existing visitor. If you have 20 visitors and add a new element type, you touch 20 files. This is the exact inverse of the pattern's value proposition.

2. **The operations are few and stable.** If you have 2 operations and 10 element types, a simple abstract method on the base class is cleaner.

3. **Elements need to control traversal.** Visitor usually delegates traversal to the elements (they call `accept` on children) or to an external iterator. If traversal logic is complex and interleaved with operation logic, a different pattern may be better.

4. **Encapsulation is critical.** Visitor often requires elements to expose internal state (through getters) so visitors can do their work. This leaks implementation details.

5. **The team is small and the codebase is young.** Visitor adds indirection. For a small, evolving domain model, the overhead outweighs the benefit.

---

## 4. UML Class Diagram

```
«interface»                          «interface»
DocumentVisitor                      DocumentElement
─────────────────────                ─────────────────
+ visit(Paragraph p)                 + accept(DocumentVisitor v)
+ visit(Table t)                          ▲
+ visit(Image i)                          │ implements
+ visit(Header h)              ┌──────────┼──────────┬────────────┐
+ visit(ListElement l)         │          │          │            │
        ▲                  Paragraph   Table      Image        Header
        │ implements        ──────     ─────      ─────        ──────
        │                  + accept   + accept   + accept     + accept
┌───────┼────────────┐
│       │            │
WordCount  HtmlRenderer  PdfRenderer
Visitor    Visitor       Visitor
─────────  ───────────   ───────────
+ visit(P) + visit(P)   + visit(P)
+ visit(T) + visit(T)   + visit(T)
+ visit(I) + visit(I)   + visit(I)
+ visit(H) + visit(H)   + visit(H)
+ visit(L) + visit(L)   + visit(L)


Client
  │
  ├─ builds Document (contains DocumentElements)
  └─ creates Visitor, calls document.accept(visitor)
       │
       └─ Document.accept iterates children, calls child.accept(visitor)
            │
            └─ child.accept calls visitor.visit(this)  ← double dispatch
```

---

## 5. Implementation 1: Document Element Visitor

This is a complete, compilable implementation. All classes can be placed in a single file for simplicity, or split by class.

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

// ─────────────────────────────────────────────────────────────────
// VISITOR INTERFACE
// ─────────────────────────────────────────────────────────────────

interface DocumentVisitor {
    void visit(Paragraph paragraph);
    void visit(TableElement table);
    void visit(ImageElement image);
    void visit(Header header);
    void visit(ListElement list);
    void visit(Document document);
}

// ─────────────────────────────────────────────────────────────────
// ELEMENT INTERFACE
// ─────────────────────────────────────────────────────────────────

interface DocumentElement {
    void accept(DocumentVisitor visitor);
}

// ─────────────────────────────────────────────────────────────────
// CONCRETE ELEMENTS
// ─────────────────────────────────────────────────────────────────

class Paragraph implements DocumentElement {
    private final String text;
    private final String style; // "normal", "bold", "italic"

    public Paragraph(String text, String style) {
        this.text = text;
        this.style = style;
    }

    public String getText()  { return text;  }
    public String getStyle() { return style; }

    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visit(this);
    }
}

class Header implements DocumentElement {
    private final String text;
    private final int level; // 1–6, like H1–H6

    public Header(String text, int level) {
        this.text  = text;
        this.level = level;
    }

    public String getText()  { return text;  }
    public int    getLevel() { return level; }

    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visit(this);
    }
}

class ImageElement implements DocumentElement {
    private final String filePath;
    private final String altText;
    private final long   fileSizeBytes;
    private final int    widthPx;
    private final int    heightPx;

    public ImageElement(String filePath, String altText, long fileSizeBytes,
                        int widthPx, int heightPx) {
        this.filePath      = filePath;
        this.altText       = altText;
        this.fileSizeBytes = fileSizeBytes;
        this.widthPx       = widthPx;
        this.heightPx      = heightPx;
    }

    public String getFilePath()       { return filePath;      }
    public String getAltText()        { return altText;       }
    public long   getFileSizeBytes()  { return fileSizeBytes; }
    public int    getWidthPx()        { return widthPx;       }
    public int    getHeightPx()       { return heightPx;      }

    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visit(this);
    }
}

class TableCell {
    private final String content;
    private final boolean isHeader;

    public TableCell(String content, boolean isHeader) {
        this.content  = content;
        this.isHeader = isHeader;
    }

    public String  getContent()  { return content;  }
    public boolean isHeader()    { return isHeader; }
}

class TableElement implements DocumentElement {
    private final List<List<TableCell>> rows;
    private final String caption;

    public TableElement(String caption) {
        this.caption = caption;
        this.rows    = new ArrayList<>();
    }

    public void addRow(List<TableCell> row) {
        rows.add(Collections.unmodifiableList(new ArrayList<>(row)));
    }

    public List<List<TableCell>> getRows()    { return Collections.unmodifiableList(rows); }
    public String                getCaption() { return caption; }

    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visit(this);
    }
}

class ListElement implements DocumentElement {
    private final List<String> items;
    private final boolean ordered;

    public ListElement(boolean ordered) {
        this.ordered = ordered;
        this.items   = new ArrayList<>();
    }

    public void addItem(String item)          { items.add(item);           }
    public List<String> getItems()            { return Collections.unmodifiableList(items); }
    public boolean      isOrdered()           { return ordered;            }

    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visit(this);
    }
}

class Document implements DocumentElement {
    private final String title;
    private final List<DocumentElement> elements;

    public Document(String title) {
        this.title    = title;
        this.elements = new ArrayList<>();
    }

    public void add(DocumentElement element) {
        elements.add(element);
    }

    public String               getTitle()    { return title;    }
    public List<DocumentElement> getElements() { return Collections.unmodifiableList(elements); }

    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visit(this);
        for (DocumentElement element : elements) {
            element.accept(visitor);
        }
    }
}

// ─────────────────────────────────────────────────────────────────
// VISITOR 1: WordCountVisitor
// ─────────────────────────────────────────────────────────────────

class WordCountVisitor implements DocumentVisitor {
    private int totalWords       = 0;
    private int paragraphWords   = 0;
    private int headerWords      = 0;
    private int tableWords       = 0;
    private int listWords        = 0;
    private int imageAltWords    = 0;

    private int countWords(String text) {
        if (text == null || text.isBlank()) return 0;
        return text.trim().split("\\s+").length;
    }

    @Override
    public void visit(Document document) {
        // reset on each document visit
        totalWords = paragraphWords = headerWords = tableWords = listWords = imageAltWords = 0;
    }

    @Override
    public void visit(Paragraph paragraph) {
        int count = countWords(paragraph.getText());
        paragraphWords += count;
        totalWords     += count;
    }

    @Override
    public void visit(Header header) {
        int count = countWords(header.getText());
        headerWords += count;
        totalWords  += count;
    }

    @Override
    public void visit(ImageElement image) {
        int count = countWords(image.getAltText());
        imageAltWords += count;
        totalWords    += count;
    }

    @Override
    public void visit(TableElement table) {
        for (List<TableCell> row : table.getRows()) {
            for (TableCell cell : row) {
                int count = countWords(cell.getContent());
                tableWords += count;
                totalWords += count;
            }
        }
    }

    @Override
    public void visit(ListElement list) {
        for (String item : list.getItems()) {
            int count = countWords(item);
            listWords  += count;
            totalWords += count;
        }
    }

    public int getTotalWords()     { return totalWords;     }
    public int getParagraphWords() { return paragraphWords; }
    public int getHeaderWords()    { return headerWords;    }
    public int getTableWords()     { return tableWords;     }
    public int getListWords()      { return listWords;      }
    public int getImageAltWords()  { return imageAltWords;  }

    public String getSummary() {
        return String.format(
            "Word Count Summary:\n" +
            "  Total:      %d\n"    +
            "  Paragraphs: %d\n"    +
            "  Headers:    %d\n"    +
            "  Tables:     %d\n"    +
            "  Lists:      %d\n"    +
            "  Image alt:  %d",
            totalWords, paragraphWords, headerWords, tableWords, listWords, imageAltWords
        );
    }
}

// ─────────────────────────────────────────────────────────────────
// VISITOR 2: HtmlRendererVisitor
// ─────────────────────────────────────────────────────────────────

class HtmlRendererVisitor implements DocumentVisitor {
    private final StringBuilder html = new StringBuilder();
    private int indentLevel = 0;

    private String indent() {
        return "  ".repeat(indentLevel);
    }

    private String escapeHtml(String text) {
        return text
            .replace("&",  "&amp;")
            .replace("<",  "&lt;")
            .replace(">",  "&gt;")
            .replace("\"", "&quot;")
            .replace("'",  "&#39;");
    }

    @Override
    public void visit(Document document) {
        html.append("<!DOCTYPE html>\n<html>\n<head>\n");
        html.append("  <meta charset=\"UTF-8\">\n");
        html.append("  <title>").append(escapeHtml(document.getTitle())).append("</title>\n");
        html.append("</head>\n<body>\n");
        indentLevel = 1;
    }

    @Override
    public void visit(Paragraph paragraph) {
        String styleAttr = "";
        if ("bold".equalsIgnoreCase(paragraph.getStyle())) {
            styleAttr = " style=\"font-weight:bold\"";
        } else if ("italic".equalsIgnoreCase(paragraph.getStyle())) {
            styleAttr = " style=\"font-style:italic\"";
        }
        html.append(indent())
            .append("<p").append(styleAttr).append(">")
            .append(escapeHtml(paragraph.getText()))
            .append("</p>\n");
    }

    @Override
    public void visit(Header header) {
        int level = Math.max(1, Math.min(6, header.getLevel()));
        html.append(indent())
            .append("<h").append(level).append(">")
            .append(escapeHtml(header.getText()))
            .append("</h").append(level).append(">\n");
    }

    @Override
    public void visit(ImageElement image) {
        html.append(indent())
            .append("<img src=\"").append(escapeHtml(image.getFilePath())).append("\"")
            .append(" alt=\"").append(escapeHtml(image.getAltText())).append("\"")
            .append(" width=\"").append(image.getWidthPx()).append("\"")
            .append(" height=\"").append(image.getHeightPx()).append("\"")
            .append(">\n");
    }

    @Override
    public void visit(TableElement table) {
        html.append(indent()).append("<table>\n");
        if (!table.getCaption().isEmpty()) {
            html.append(indent()).append("  <caption>")
                .append(escapeHtml(table.getCaption())).append("</caption>\n");
        }
        boolean firstRow = true;
        for (List<TableCell> row : table.getRows()) {
            html.append(indent()).append("  <tr>\n");
            for (TableCell cell : row) {
                String tag = (firstRow || cell.isHeader()) ? "th" : "td";
                html.append(indent()).append("    <").append(tag).append(">")
                    .append(escapeHtml(cell.getContent()))
                    .append("</").append(tag).append(">\n");
            }
            html.append(indent()).append("  </tr>\n");
            firstRow = false;
        }
        html.append(indent()).append("</table>\n");
    }

    @Override
    public void visit(ListElement list) {
        String tag = list.isOrdered() ? "ol" : "ul";
        html.append(indent()).append("<").append(tag).append(">\n");
        for (String item : list.getItems()) {
            html.append(indent()).append("  <li>")
                .append(escapeHtml(item)).append("</li>\n");
        }
        html.append(indent()).append("</").append(tag).append(">\n");
    }

    public String getHtml() {
        return html.toString() + "</body>\n</html>";
    }
}

// ─────────────────────────────────────────────────────────────────
// VISITOR 3: PdfRendererVisitor
// (Simulates PDF rendering — real PDF would use Apache PDFBox/iText)
// ─────────────────────────────────────────────────────────────────

class PdfRenderCommand {
    public enum Type { SET_FONT, DRAW_TEXT, DRAW_IMAGE, DRAW_LINE, NEW_PAGE, NEW_SECTION }
    private final Type   type;
    private final String data;

    public PdfRenderCommand(Type type, String data) {
        this.type = type;
        this.data = data;
    }

    @Override
    public String toString() {
        return String.format("PDF_CMD[%s]: %s", type, data);
    }
}

class PdfRendererVisitor implements DocumentVisitor {
    private final List<PdfRenderCommand> commands = new ArrayList<>();
    private float currentY = 792f; // US Letter height in points
    private static final float MARGIN        = 72f;
    private static final float LINE_HEIGHT   = 14f;
    private static final float PAGE_HEIGHT   = 792f;

    private void ensureSpace(float needed) {
        if (currentY - needed < MARGIN) {
            commands.add(new PdfRenderCommand(PdfRenderCommand.Type.NEW_PAGE, ""));
            currentY = PAGE_HEIGHT - MARGIN;
        }
    }

    @Override
    public void visit(Document document) {
        commands.add(new PdfRenderCommand(PdfRenderCommand.Type.NEW_PAGE,
            "title=" + document.getTitle()));
        currentY = PAGE_HEIGHT - MARGIN;
        commands.add(new PdfRenderCommand(PdfRenderCommand.Type.SET_FONT,
            "Helvetica-Bold size=24"));
        commands.add(new PdfRenderCommand(PdfRenderCommand.Type.DRAW_TEXT,
            "y=" + currentY + " text=" + document.getTitle()));
        currentY -= 36f;
    }

    @Override
    public void visit(Paragraph paragraph) {
        ensureSpace(LINE_HEIGHT * 3);
        String fontSpec = "normal".equalsIgnoreCase(paragraph.getStyle())
            ? "Times-Roman size=12"
            : "bold".equalsIgnoreCase(paragraph.getStyle())
                ? "Times-Bold size=12"
                : "Times-Italic size=12";
        commands.add(new PdfRenderCommand(PdfRenderCommand.Type.SET_FONT, fontSpec));

        // Word-wrap simulation: split at 80-char lines
        String text = paragraph.getText();
        while (text.length() > 80) {
            int split = text.lastIndexOf(' ', 80);
            if (split == -1) split = 80;
            commands.add(new PdfRenderCommand(PdfRenderCommand.Type.DRAW_TEXT,
                "y=" + currentY + " x=" + MARGIN + " text=" + text.substring(0, split)));
            currentY -= LINE_HEIGHT;
            text = text.substring(split).trim();
            ensureSpace(LINE_HEIGHT);
        }
        commands.add(new PdfRenderCommand(PdfRenderCommand.Type.DRAW_TEXT,
            "y=" + currentY + " x=" + MARGIN + " text=" + text));
        currentY -= LINE_HEIGHT * 1.5f;
    }

    @Override
    public void visit(Header header) {
        ensureSpace(LINE_HEIGHT * 2);
        float[] sizes = {24f, 20f, 16f, 14f, 12f, 10f};
        float size = sizes[Math.min(header.getLevel() - 1, 5)];
        commands.add(new PdfRenderCommand(PdfRenderCommand.Type.SET_FONT,
            "Helvetica-Bold size=" + size));
        commands.add(new PdfRenderCommand(PdfRenderCommand.Type.DRAW_TEXT,
            "y=" + currentY + " x=" + MARGIN + " text=" + header.getText()));
        commands.add(new PdfRenderCommand(PdfRenderCommand.Type.DRAW_LINE,
            "x1=" + MARGIN + " y1=" + (currentY - 2) + " x2=540 y2=" + (currentY - 2)));
        currentY -= size + 8f;
    }

    @Override
    public void visit(ImageElement image) {
        float imgHeight = Math.min(image.getHeightPx() * 0.75f, 300f);
        ensureSpace(imgHeight + LINE_HEIGHT);
        commands.add(new PdfRenderCommand(PdfRenderCommand.Type.DRAW_IMAGE,
            "src=" + image.getFilePath()
            + " y=" + currentY + " x=" + MARGIN
            + " w=" + (image.getWidthPx() * 0.75f)
            + " h=" + imgHeight));
        currentY -= imgHeight + LINE_HEIGHT;
    }

    @Override
    public void visit(TableElement table) {
        List<List<TableCell>> rows = table.getRows();
        if (rows.isEmpty()) return;

        int colCount = rows.get(0).size();
        float colWidth  = (468f) / colCount; // 468 = 612 - 2*72 margins

        if (!table.getCaption().isEmpty()) {
            ensureSpace(LINE_HEIGHT);
            commands.add(new PdfRenderCommand(PdfRenderCommand.Type.SET_FONT,
                "Helvetica-Oblique size=10"));
            commands.add(new PdfRenderCommand(PdfRenderCommand.Type.DRAW_TEXT,
                "y=" + currentY + " x=" + MARGIN + " text=Table: " + table.getCaption()));
            currentY -= LINE_HEIGHT;
        }

        for (List<TableCell> row : rows) {
            ensureSpace(LINE_HEIGHT + 4);
            float x = MARGIN;
            for (TableCell cell : row) {
                String font = cell.isHeader() ? "Helvetica-Bold size=10" : "Helvetica size=10";
                commands.add(new PdfRenderCommand(PdfRenderCommand.Type.SET_FONT, font));
                commands.add(new PdfRenderCommand(PdfRenderCommand.Type.DRAW_TEXT,
                    "y=" + currentY + " x=" + x + " text=" + cell.getContent()));
                x += colWidth;
            }
            currentY -= LINE_HEIGHT + 4;
        }
        currentY -= 8f;
    }

    @Override
    public void visit(ListElement list) {
        List<String> items = list.getItems();
        for (int i = 0; i < items.size(); i++) {
            ensureSpace(LINE_HEIGHT);
            String bullet = list.isOrdered() ? (i + 1) + ". " : "• ";
            commands.add(new PdfRenderCommand(PdfRenderCommand.Type.SET_FONT,
                "Times-Roman size=12"));
            commands.add(new PdfRenderCommand(PdfRenderCommand.Type.DRAW_TEXT,
                "y=" + currentY + " x=" + (MARGIN + 10) + " text=" + bullet + items.get(i)));
            currentY -= LINE_HEIGHT;
        }
        currentY -= 6f;
    }

    public List<PdfRenderCommand> getCommands() {
        return Collections.unmodifiableList(commands);
    }

    public void printCommands() {
        commands.forEach(System.out::println);
    }
}

// ─────────────────────────────────────────────────────────────────
// VISITOR 4: FileSizeCalculatorVisitor
// ─────────────────────────────────────────────────────────────────

class FileSizeCalculatorVisitor implements DocumentVisitor {
    // Approximate bytes per character for UTF-8 plain text
    private static final int BYTES_PER_CHAR = 2;
    // Overhead per HTML tag, table cell markup, etc.
    private static final int TABLE_OVERHEAD_PER_CELL = 10;

    private long textBytes  = 0;
    private long imageBytes = 0;
    private long tableBytes = 0;
    private long listBytes  = 0;
    private long totalBytes = 0;

    private long estimateTextBytes(String text) {
        if (text == null) return 0;
        return (long) text.length() * BYTES_PER_CHAR;
    }

    @Override
    public void visit(Document document) {
        textBytes = imageBytes = tableBytes = listBytes = totalBytes = 0;
        long titleBytes = estimateTextBytes(document.getTitle());
        textBytes  += titleBytes;
        totalBytes += titleBytes;
    }

    @Override
    public void visit(Paragraph paragraph) {
        long b = estimateTextBytes(paragraph.getText());
        textBytes  += b;
        totalBytes += b;
    }

    @Override
    public void visit(Header header) {
        long b = estimateTextBytes(header.getText());
        textBytes  += b;
        totalBytes += b;
    }

    @Override
    public void visit(ImageElement image) {
        long b = image.getFileSizeBytes();
        imageBytes += b;
        totalBytes += b;
        // Also count alt text
        long altBytes = estimateTextBytes(image.getAltText());
        textBytes  += altBytes;
        totalBytes += altBytes;
    }

    @Override
    public void visit(TableElement table) {
        long b = estimateTextBytes(table.getCaption());
        for (List<TableCell> row : table.getRows()) {
            for (TableCell cell : row) {
                b += estimateTextBytes(cell.getContent()) + TABLE_OVERHEAD_PER_CELL;
            }
        }
        tableBytes += b;
        totalBytes += b;
    }

    @Override
    public void visit(ListElement list) {
        long b = 0;
        for (String item : list.getItems()) {
            b += estimateTextBytes(item);
        }
        listBytes  += b;
        totalBytes += b;
    }

    private String formatSize(long bytes) {
        if (bytes < 1024)        return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
        return String.format("%.2f MB", bytes / (1024.0 * 1024));
    }

    public long getTotalBytes()  { return totalBytes;  }
    public long getImageBytes()  { return imageBytes;  }
    public long getTextBytes()   { return textBytes;   }
    public long getTableBytes()  { return tableBytes;  }
    public long getListBytes()   { return listBytes;   }

    public String getSummary() {
        return String.format(
            "File Size Estimate:\n"     +
            "  Total:  %s (%d bytes)\n" +
            "  Text:   %s\n"            +
            "  Images: %s\n"            +
            "  Tables: %s\n"            +
            "  Lists:  %s",
            formatSize(totalBytes), totalBytes,
            formatSize(textBytes),
            formatSize(imageBytes),
            formatSize(tableBytes),
            formatSize(listBytes)
        );
    }
}

// ─────────────────────────────────────────────────────────────────
// DEMO
// ─────────────────────────────────────────────────────────────────

class DocumentVisitorDemo {
    public static void main(String[] args) {
        // Build a sample document
        Document doc = new Document("Visitor Pattern Deep Dive");

        doc.add(new Header("Introduction", 1));
        doc.add(new Paragraph(
            "The Visitor pattern separates algorithms from the object structure " +
            "they operate on, enabling new operations without modifying classes.", "normal"));

        Header h2 = new Header("Benefits", 2);
        doc.add(h2);

        ListElement benefits = new ListElement(false);
        benefits.addItem("Open/Closed Principle compliance");
        benefits.addItem("Single Responsibility per visitor class");
        benefits.addItem("Compile-time type safety across all element types");
        doc.add(benefits);

        TableElement table = new TableElement("Pattern Comparison");
        table.addRow(List.of(
            new TableCell("Pattern",   true),
            new TableCell("Adds Ops?", true),
            new TableCell("Adds Types?", true)
        ));
        table.addRow(List.of(
            new TableCell("Visitor",   false),
            new TableCell("Easy",      false),
            new TableCell("Hard",      false)
        ));
        table.addRow(List.of(
            new TableCell("Strategy",  false),
            new TableCell("Medium",    false),
            new TableCell("Easy",      false)
        ));
        doc.add(table);

        doc.add(new ImageElement(
            "images/uml-diagram.png", "UML class diagram for Visitor",
            45_000L, 800, 600
        ));

        doc.add(new Paragraph(
            "In conclusion the Visitor pattern is ideal when the type hierarchy " +
            "is stable and the set of operations is growing rapidly.", "italic"));

        // Apply visitors
        WordCountVisitor wordCount = new WordCountVisitor();
        doc.accept(wordCount);
        System.out.println(wordCount.getSummary());
        System.out.println();

        HtmlRendererVisitor htmlRenderer = new HtmlRendererVisitor();
        doc.accept(htmlRenderer);
        System.out.println("=== HTML Output (first 500 chars) ===");
        String html = htmlRenderer.getHtml();
        System.out.println(html.substring(0, Math.min(500, html.length())));
        System.out.println("...\n");

        FileSizeCalculatorVisitor sizeCalc = new FileSizeCalculatorVisitor();
        doc.accept(sizeCalc);
        System.out.println(sizeCalc.getSummary());
        System.out.println();

        PdfRendererVisitor pdfRenderer = new PdfRendererVisitor();
        doc.accept(pdfRenderer);
        System.out.println("=== PDF Render Commands (first 10) ===");
        pdfRenderer.getCommands().stream().limit(10).forEach(System.out::println);
    }
}
```

---

## 6. Implementation 2: AST Visitor

A mini expression language with nodes for numbers, variables, addition, and multiplication. Three visitors: evaluator, pretty printer, and type checker.

```java
import java.util.HashMap;
import java.util.Map;
import java.util.Set;
import java.util.HashSet;
import java.util.List;
import java.util.ArrayList;

// ─────────────────────────────────────────────────────────────────
// AST NODE INTERFACES AND TYPES
// ─────────────────────────────────────────────────────────────────

/**
 * Generic visitor over AST nodes.
 * T is the return type of each visit — enables typed results without
 * mutable state accumulation.
 */
interface AstVisitor<T> {
    T visitNumber(NumberNode node);
    T visitVariable(VariableNode node);
    T visitAdd(AddNode node);
    T visitMultiply(MultiplyNode node);
    T visitNegate(NegateNode node);
    T visitDivide(DivideNode node);
}

interface AstNode {
    <T> T accept(AstVisitor<T> visitor);
}

// ─────────────────────────────────────────────────────────────────
// CONCRETE AST NODES
// ─────────────────────────────────────────────────────────────────

class NumberNode implements AstNode {
    private final double value;

    public NumberNode(double value) {
        this.value = value;
    }

    public double getValue() { return value; }

    @Override
    public <T> T accept(AstVisitor<T> visitor) {
        return visitor.visitNumber(this);
    }
}

class VariableNode implements AstNode {
    private final String name;

    public VariableNode(String name) {
        this.name = name;
    }

    public String getName() { return name; }

    @Override
    public <T> T accept(AstVisitor<T> visitor) {
        return visitor.visitVariable(this);
    }
}

class AddNode implements AstNode {
    private final AstNode left;
    private final AstNode right;

    public AddNode(AstNode left, AstNode right) {
        this.left  = left;
        this.right = right;
    }

    public AstNode getLeft()  { return left;  }
    public AstNode getRight() { return right; }

    @Override
    public <T> T accept(AstVisitor<T> visitor) {
        return visitor.visitAdd(this);
    }
}

class MultiplyNode implements AstNode {
    private final AstNode left;
    private final AstNode right;

    public MultiplyNode(AstNode left, AstNode right) {
        this.left  = left;
        this.right = right;
    }

    public AstNode getLeft()  { return left;  }
    public AstNode getRight() { return right; }

    @Override
    public <T> T accept(AstVisitor<T> visitor) {
        return visitor.visitMultiply(this);
    }
}

class NegateNode implements AstNode {
    private final AstNode operand;

    public NegateNode(AstNode operand) {
        this.operand = operand;
    }

    public AstNode getOperand() { return operand; }

    @Override
    public <T> T accept(AstVisitor<T> visitor) {
        return visitor.visitNegate(this);
    }
}

class DivideNode implements AstNode {
    private final AstNode left;
    private final AstNode right;

    public DivideNode(AstNode left, AstNode right) {
        this.left  = left;
        this.right = right;
    }

    public AstNode getLeft()  { return left;  }
    public AstNode getRight() { return right; }

    @Override
    public <T> T accept(AstVisitor<T> visitor) {
        return visitor.visitDivide(this);
    }
}

// ─────────────────────────────────────────────────────────────────
// VISITOR 1: EvaluatorVisitor
// Evaluates the expression given a variable binding environment.
// Returns Double.
// ─────────────────────────────────────────────────────────────────

class EvaluatorVisitor implements AstVisitor<Double> {
    private final Map<String, Double> environment;

    public EvaluatorVisitor(Map<String, Double> environment) {
        this.environment = new HashMap<>(environment);
    }

    @Override
    public Double visitNumber(NumberNode node) {
        return node.getValue();
    }

    @Override
    public Double visitVariable(VariableNode node) {
        Double value = environment.get(node.getName());
        if (value == null) {
            throw new IllegalStateException(
                "Undefined variable: '" + node.getName() + "'");
        }
        return value;
    }

    @Override
    public Double visitAdd(AddNode node) {
        return node.getLeft().accept(this) + node.getRight().accept(this);
    }

    @Override
    public Double visitMultiply(MultiplyNode node) {
        return node.getLeft().accept(this) * node.getRight().accept(this);
    }

    @Override
    public Double visitNegate(NegateNode node) {
        return -node.getOperand().accept(this);
    }

    @Override
    public Double visitDivide(DivideNode node) {
        double divisor = node.getRight().accept(this);
        if (divisor == 0.0) {
            throw new ArithmeticException("Division by zero in expression");
        }
        return node.getLeft().accept(this) / divisor;
    }
}

// ─────────────────────────────────────────────────────────────────
// VISITOR 2: PrettyPrinterVisitor
// Returns a fully parenthesized, human-readable string.
// ─────────────────────────────────────────────────────────────────

class PrettyPrinterVisitor implements AstVisitor<String> {

    @Override
    public String visitNumber(NumberNode node) {
        // Print integers without decimal point for readability
        double v = node.getValue();
        if (v == Math.floor(v) && !Double.isInfinite(v)) {
            return String.valueOf((long) v);
        }
        return String.valueOf(v);
    }

    @Override
    public String visitVariable(VariableNode node) {
        return node.getName();
    }

    @Override
    public String visitAdd(AddNode node) {
        return "(" + node.getLeft().accept(this)
            + " + " + node.getRight().accept(this) + ")";
    }

    @Override
    public String visitMultiply(MultiplyNode node) {
        return "(" + node.getLeft().accept(this)
            + " * " + node.getRight().accept(this) + ")";
    }

    @Override
    public String visitNegate(NegateNode node) {
        return "(-" + node.getOperand().accept(this) + ")";
    }

    @Override
    public String visitDivide(DivideNode node) {
        return "(" + node.getLeft().accept(this)
            + " / " + node.getRight().accept(this) + ")";
    }
}

// ─────────────────────────────────────────────────────────────────
// VISITOR 3: TypeCheckerVisitor
// Determines the "type" of each sub-expression.
// In this mini language: CONSTANT (no free variables) or EXPRESSION (has variables).
// Also collects all referenced variable names.
// ─────────────────────────────────────────────────────────────────

enum ExprType { CONSTANT, EXPRESSION }

class TypeInfo {
    private final ExprType type;
    private final Set<String> referencedVariables;

    public TypeInfo(ExprType type, Set<String> referencedVariables) {
        this.type                = type;
        this.referencedVariables = new HashSet<>(referencedVariables);
    }

    public ExprType      getType()                { return type;                }
    public Set<String>   getReferencedVariables() { return Set.copyOf(referencedVariables); }
    public boolean       isConstant()             { return type == ExprType.CONSTANT; }

    public TypeInfo merge(TypeInfo other) {
        Set<String> combined = new HashSet<>(this.referencedVariables);
        combined.addAll(other.referencedVariables);
        ExprType merged = (this.type == ExprType.EXPRESSION || other.type == ExprType.EXPRESSION)
            ? ExprType.EXPRESSION : ExprType.CONSTANT;
        return new TypeInfo(merged, combined);
    }

    @Override
    public String toString() {
        if (isConstant()) return "CONSTANT";
        return "EXPRESSION(vars=" + referencedVariables + ")";
    }
}

class TypeCheckerVisitor implements AstVisitor<TypeInfo> {
    private final Set<String> declaredVariables;
    private final List<String> errors = new ArrayList<>();

    public TypeCheckerVisitor(Set<String> declaredVariables) {
        this.declaredVariables = new HashSet<>(declaredVariables);
    }

    @Override
    public TypeInfo visitNumber(NumberNode node) {
        return new TypeInfo(ExprType.CONSTANT, Set.of());
    }

    @Override
    public TypeInfo visitVariable(VariableNode node) {
        String name = node.getName();
        if (!declaredVariables.contains(name)) {
            errors.add("Undeclared variable: '" + name + "'");
        }
        return new TypeInfo(ExprType.EXPRESSION, Set.of(name));
    }

    @Override
    public TypeInfo visitAdd(AddNode node) {
        TypeInfo left  = node.getLeft().accept(this);
        TypeInfo right = node.getRight().accept(this);
        return left.merge(right);
    }

    @Override
    public TypeInfo visitMultiply(MultiplyNode node) {
        TypeInfo left  = node.getLeft().accept(this);
        TypeInfo right = node.getRight().accept(this);
        return left.merge(right);
    }

    @Override
    public TypeInfo visitNegate(NegateNode node) {
        return node.getOperand().accept(this);
    }

    @Override
    public TypeInfo visitDivide(DivideNode node) {
        TypeInfo left  = node.getLeft().accept(this);
        TypeInfo right = node.getRight().accept(this);

        // Warn if dividing by a constant zero
        if (right.isConstant()) {
            // We can't evaluate statically here without an evaluator,
            // but we flag the pattern for analysis
        }
        return left.merge(right);
    }

    public boolean hasErrors()          { return !errors.isEmpty(); }
    public List<String> getErrors()     { return List.copyOf(errors); }
}

// ─────────────────────────────────────────────────────────────────
// AST DEMO
// ─────────────────────────────────────────────────────────────────

class AstVisitorDemo {
    public static void main(String[] args) {
        // Build AST for: (x + 2) * (y - 3) / x
        // Represented as: ((x + 2) * (y + (-3))) / x

        AstNode expr = new DivideNode(
            new MultiplyNode(
                new AddNode(
                    new VariableNode("x"),
                    new NumberNode(2)
                ),
                new AddNode(
                    new VariableNode("y"),
                    new NegateNode(new NumberNode(3))
                )
            ),
            new VariableNode("x")
        );

        // Pretty print
        PrettyPrinterVisitor printer = new PrettyPrinterVisitor();
        System.out.println("Expression: " + expr.accept(printer));

        // Type check
        TypeCheckerVisitor typeChecker = new TypeCheckerVisitor(Set.of("x", "y"));
        TypeInfo typeInfo = expr.accept(typeChecker);
        System.out.println("Type: " + typeInfo);
        if (typeChecker.hasErrors()) {
            typeChecker.getErrors().forEach(e -> System.out.println("ERROR: " + e));
        }

        // Evaluate with x=4, y=7  → (4+2)*(7-3)/4 = 6*4/4 = 6
        Map<String, Double> env = Map.of("x", 4.0, "y", 7.0);
        EvaluatorVisitor evaluator = new EvaluatorVisitor(env);
        double result = expr.accept(evaluator);
        System.out.printf("Result (x=4, y=7): %.4f%n", result); // Expected: 6.0

        // Test with undeclared variable
        AstNode badExpr = new AddNode(new VariableNode("z"), new NumberNode(1));
        TypeCheckerVisitor badChecker = new TypeCheckerVisitor(Set.of("x", "y"));
        badChecker.visitVariable(new VariableNode("z"));
        badExpr.accept(badChecker);
        System.out.println("Errors: " + badChecker.getErrors());
    }
}
```

---

## 7. Tax Calculation Example

A self-contained example showing tax rules applied to heterogeneous invoice line items.

```java
import java.util.ArrayList;
import java.util.List;

// ─────────────────────────────────────────────────────────────────
// TAX VISITOR INTERFACE
// ─────────────────────────────────────────────────────────────────

interface TaxVisitor {
    double visit(PhysicalGoodsLineItem item);
    double visit(DigitalGoodsLineItem item);
    double visit(ServiceLineItem item);
    double visit(ExemptLineItem item);
    double visit(BundleLineItem item);
}

// ─────────────────────────────────────────────────────────────────
// LINE ITEM INTERFACE AND CONCRETE ITEMS
// ─────────────────────────────────────────────────────────────────

interface InvoiceLineItem {
    double accept(TaxVisitor visitor);
    String getDescription();
    double getBaseAmount();
}

class PhysicalGoodsLineItem implements InvoiceLineItem {
    private final String description;
    private final double amount;
    private final String hsnCode;       // Harmonized System nomenclature code
    private final String originCountry;

    public PhysicalGoodsLineItem(String description, double amount,
                                  String hsnCode, String originCountry) {
        this.description   = description;
        this.amount        = amount;
        this.hsnCode       = hsnCode;
        this.originCountry = originCountry;
    }

    public String getHsnCode()        { return hsnCode;        }
    public String getOriginCountry()  { return originCountry;  }

    @Override public String getDescription() { return description; }
    @Override public double getBaseAmount()  { return amount;      }

    @Override
    public double accept(TaxVisitor visitor) {
        return visitor.visit(this);
    }
}

class DigitalGoodsLineItem implements InvoiceLineItem {
    private final String  description;
    private final double  amount;
    private final boolean isSoftwareLicense;
    private final String  deliveryCountry;

    public DigitalGoodsLineItem(String description, double amount,
                                 boolean isSoftwareLicense, String deliveryCountry) {
        this.description       = description;
        this.amount            = amount;
        this.isSoftwareLicense = isSoftwareLicense;
        this.deliveryCountry   = deliveryCountry;
    }

    public boolean isSoftwareLicense() { return isSoftwareLicense; }
    public String  getDeliveryCountry(){ return deliveryCountry;   }

    @Override public String getDescription() { return description; }
    @Override public double getBaseAmount()  { return amount;      }

    @Override
    public double accept(TaxVisitor visitor) {
        return visitor.visit(this);
    }
}

class ServiceLineItem implements InvoiceLineItem {
    private final String  description;
    private final double  amount;
    private final String  serviceType;   // "consulting", "support", "training"
    private final boolean isExported;    // services exported outside tax jurisdiction

    public ServiceLineItem(String description, double amount,
                            String serviceType, boolean isExported) {
        this.description = description;
        this.amount      = amount;
        this.serviceType = serviceType;
        this.isExported  = isExported;
    }

    public String  getServiceType() { return serviceType; }
    public boolean isExported()     { return isExported;  }

    @Override public String getDescription() { return description; }
    @Override public double getBaseAmount()  { return amount;      }

    @Override
    public double accept(TaxVisitor visitor) {
        return visitor.visit(this);
    }
}

class ExemptLineItem implements InvoiceLineItem {
    private final String description;
    private final double amount;
    private final String exemptionCode;

    public ExemptLineItem(String description, double amount, String exemptionCode) {
        this.description  = description;
        this.amount       = amount;
        this.exemptionCode = exemptionCode;
    }

    public String getExemptionCode() { return exemptionCode; }

    @Override public String getDescription() { return description; }
    @Override public double getBaseAmount()  { return amount;      }

    @Override
    public double accept(TaxVisitor visitor) {
        return visitor.visit(this);
    }
}

class BundleLineItem implements InvoiceLineItem {
    private final String                description;
    private final List<InvoiceLineItem> components;

    public BundleLineItem(String description) {
        this.description = description;
        this.components  = new ArrayList<>();
    }

    public void addComponent(InvoiceLineItem item) { components.add(item); }
    public List<InvoiceLineItem> getComponents()   { return List.copyOf(components); }

    @Override
    public double getBaseAmount() {
        return components.stream().mapToDouble(InvoiceLineItem::getBaseAmount).sum();
    }

    @Override public String getDescription() { return description; }

    @Override
    public double accept(TaxVisitor visitor) {
        return visitor.visit(this);
    }
}

// ─────────────────────────────────────────────────────────────────
// CONCRETE TAX VISITORS
// ─────────────────────────────────────────────────────────────────

/**
 * US Sales Tax visitor.
 * Physical goods: 8.5% (standard state+local)
 * Digital goods: 6.5% if software license, 0% otherwise (many states exempt digital)
 * Services: 0% unless consulting (4%)
 * Exempt: 0%
 * Bundle: tax each component separately
 */
class UsSalesTaxVisitor implements TaxVisitor {
    private static final double PHYSICAL_RATE         = 0.085;
    private static final double SOFTWARE_LICENSE_RATE = 0.065;
    private static final double CONSULTING_RATE       = 0.040;

    @Override
    public double visit(PhysicalGoodsLineItem item) {
        return item.getBaseAmount() * PHYSICAL_RATE;
    }

    @Override
    public double visit(DigitalGoodsLineItem item) {
        if (item.isSoftwareLicense()) {
            return item.getBaseAmount() * SOFTWARE_LICENSE_RATE;
        }
        return 0.0; // streaming, downloads exempt in most US states
    }

    @Override
    public double visit(ServiceLineItem item) {
        if ("consulting".equalsIgnoreCase(item.getServiceType())) {
            return item.getBaseAmount() * CONSULTING_RATE;
        }
        return 0.0;
    }

    @Override
    public double visit(ExemptLineItem item) {
        return 0.0;
    }

    @Override
    public double visit(BundleLineItem bundle) {
        return bundle.getComponents().stream()
            .mapToDouble(component -> component.accept(this))
            .sum();
    }
}

/**
 * EU VAT visitor (standard rates).
 * Physical goods: 20%
 * Digital goods: 20% (EU VAT applies to all digital services since 2015)
 * Services: 20% unless exported outside EU (0% — zero-rated)
 * Exempt: 0%
 * Bundle: typically taxed at the rate of the predominant component
 */
class EuVatVisitor implements TaxVisitor {
    private static final double STANDARD_RATE = 0.20;

    @Override
    public double visit(PhysicalGoodsLineItem item) {
        return item.getBaseAmount() * STANDARD_RATE;
    }

    @Override
    public double visit(DigitalGoodsLineItem item) {
        // EU VAT applies to all digital goods/services regardless of type
        return item.getBaseAmount() * STANDARD_RATE;
    }

    @Override
    public double visit(ServiceLineItem item) {
        if (item.isExported()) {
            return 0.0; // zero-rated export
        }
        return item.getBaseAmount() * STANDARD_RATE;
    }

    @Override
    public double visit(ExemptLineItem item) {
        return 0.0;
    }

    @Override
    public double visit(BundleLineItem bundle) {
        // EU approach: find the predominant component and apply its rate to the whole
        // Simplified: apply standard rate to entire bundle amount
        return bundle.getBaseAmount() * STANDARD_RATE;
    }
}

/**
 * India GST visitor.
 * Physical goods: 18% GST (standard slab)
 * Digital goods: 18% (OIDAR services)
 * Services: 18% unless exported (0% — LUT-based export)
 * Exempt: 0%
 * Bundles: component-wise
 */
class IndiaGstVisitor implements TaxVisitor {
    private static final double GST_RATE = 0.18;

    @Override
    public double visit(PhysicalGoodsLineItem item) {
        return item.getBaseAmount() * GST_RATE;
    }

    @Override
    public double visit(DigitalGoodsLineItem item) {
        return item.getBaseAmount() * GST_RATE;
    }

    @Override
    public double visit(ServiceLineItem item) {
        if (item.isExported()) return 0.0;
        return item.getBaseAmount() * GST_RATE;
    }

    @Override
    public double visit(ExemptLineItem item) {
        return 0.0;
    }

    @Override
    public double visit(BundleLineItem bundle) {
        return bundle.getComponents().stream()
            .mapToDouble(component -> component.accept(this))
            .sum();
    }
}

// ─────────────────────────────────────────────────────────────────
// INVOICE AND DEMO
// ─────────────────────────────────────────────────────────────────

class Invoice {
    private final String             invoiceNumber;
    private final List<InvoiceLineItem> lineItems = new ArrayList<>();

    public Invoice(String invoiceNumber) {
        this.invoiceNumber = invoiceNumber;
    }

    public void addLineItem(InvoiceLineItem item) { lineItems.add(item); }

    public double calculateTax(TaxVisitor visitor) {
        return lineItems.stream()
            .mapToDouble(item -> item.accept(visitor))
            .sum();
    }

    public double getSubtotal() {
        return lineItems.stream().mapToDouble(InvoiceLineItem::getBaseAmount).sum();
    }

    public void printTaxBreakdown(TaxVisitor visitor, String jurisdictionName) {
        System.out.println("=== Tax Breakdown: " + jurisdictionName + " ===");
        System.out.printf("Invoice: %s%n", invoiceNumber);
        double totalTax = 0;
        for (InvoiceLineItem item : lineItems) {
            double tax = item.accept(visitor);
            totalTax += tax;
            System.out.printf("  %-35s  Base: %8.2f  Tax: %7.2f%n",
                item.getDescription(), item.getBaseAmount(), tax);
        }
        System.out.printf("  %-35s  Base: %8.2f  Tax: %7.2f%n",
            "TOTAL", getSubtotal(), totalTax);
        System.out.println();
    }
}

class TaxVisitorDemo {
    public static void main(String[] args) {
        Invoice invoice = new Invoice("INV-2024-001");

        invoice.addLineItem(new PhysicalGoodsLineItem(
            "Laptop Stand (ergonomic)", 89.99, "8473.30", "US"));

        invoice.addLineItem(new DigitalGoodsLineItem(
            "JetBrains IntelliJ License", 249.00, true, "US"));

        invoice.addLineItem(new DigitalGoodsLineItem(
            "4K Video Stream Subscription", 15.00, false, "US"));

        invoice.addLineItem(new ServiceLineItem(
            "Architecture Consulting (8h)", 1200.00, "consulting", false));

        invoice.addLineItem(new ServiceLineItem(
            "Technical Training Session", 500.00, "training", false));

        invoice.addLineItem(new ServiceLineItem(
            "API Integration Support (exported)", 800.00, "support", true));

        invoice.addLineItem(new ExemptLineItem(
            "Charitable Donation Processing", 100.00, "CHARITY-001"));

        BundleLineItem bundle = new BundleLineItem("Starter Kit Bundle");
        bundle.addComponent(new PhysicalGoodsLineItem(
            "  Bundle: USB Hub", 29.99, "8473.30", "CN"));
        bundle.addComponent(new DigitalGoodsLineItem(
            "  Bundle: Software Key", 19.99, true, "US"));
        invoice.addLineItem(bundle);

        // Apply different tax jurisdictions
        invoice.printTaxBreakdown(new UsSalesTaxVisitor(), "US Sales Tax");
        invoice.printTaxBreakdown(new EuVatVisitor(),      "EU VAT (20%)");
        invoice.printTaxBreakdown(new IndiaGstVisitor(),   "India GST (18%)");
    }
}
```

---

## 8. Acyclic Visitor Pattern

### The Circular Dependency Problem

The classic Visitor pattern introduces a **mutual dependency** between the visitor interface and the element hierarchy:

```
DocumentVisitor  ←→  Paragraph, Table, Image, Header, ListElement
```

- `DocumentVisitor` must import/reference all element types to declare its `visit` overloads.
- All element types must import `DocumentVisitor` for their `accept` method.

This is a direct circular dependency between the visitor interface and every element type. In a large system with multiple packages, this means all packages must be compiled together and cannot be deployed independently. Adding a new element type requires recompiling (and redeploying) every visitor implementation, even ones that have no interest in the new type.

### The Acyclic Visitor Solution

Robert C. Martin described the Acyclic Visitor to break this dependency cycle. The key ideas:

1. Define a **degenerate base visitor interface** with no methods (a marker interface).
2. Define a **separate, specialized visitor interface** for each element type.
3. Each concrete visitor implements only the specialized interfaces it cares about.
4. The `accept` method casts the visitor and calls the appropriate interface, or skips if the visitor does not implement that interface.

```java
// ─────────────────────────────────────────────────────────────────
// ACYCLIC VISITOR
// ─────────────────────────────────────────────────────────────────

/**
 * Degenerate base — no methods. Acts as a type anchor.
 * No element type imports are needed here.
 */
interface AcyclicVisitor {
    // intentionally empty — marker interface
}

/**
 * Each element defines its own visitor interface.
 * These are in the same package as the element, so no circular dependency.
 */
interface ParagraphVisitor extends AcyclicVisitor {
    void visit(AcyclicParagraph paragraph);
}

interface HeaderVisitor extends AcyclicVisitor {
    void visit(AcyclicHeader header);
}

interface ImageVisitor extends AcyclicVisitor {
    void visit(AcyclicImage image);
}

/**
 * Element base — depends only on AcyclicVisitor (the marker).
 */
interface AcyclicElement {
    void accept(AcyclicVisitor visitor);
}

class AcyclicParagraph implements AcyclicElement {
    private final String text;
    public AcyclicParagraph(String text) { this.text = text; }
    public String getText() { return text; }

    @Override
    public void accept(AcyclicVisitor visitor) {
        // Only dispatch if visitor implements ParagraphVisitor
        if (visitor instanceof ParagraphVisitor) {
            ((ParagraphVisitor) visitor).visit(this);
        }
        // Otherwise silently ignore — graceful degradation
    }
}

class AcyclicHeader implements AcyclicElement {
    private final String text;
    private final int level;
    public AcyclicHeader(String text, int level) { this.text = text; this.level = level; }
    public String getText()  { return text;  }
    public int    getLevel() { return level; }

    @Override
    public void accept(AcyclicVisitor visitor) {
        if (visitor instanceof HeaderVisitor) {
            ((HeaderVisitor) visitor).visit(this);
        }
    }
}

class AcyclicImage implements AcyclicElement {
    private final String src;
    private final long   sizeBytes;
    public AcyclicImage(String src, long sizeBytes) { this.src = src; this.sizeBytes = sizeBytes; }
    public String getSrc()       { return src;       }
    public long   getSizeBytes() { return sizeBytes; }

    @Override
    public void accept(AcyclicVisitor visitor) {
        if (visitor instanceof ImageVisitor) {
            ((ImageVisitor) visitor).visit(this);
        }
    }
}

/**
 * A visitor that only cares about paragraphs and headers — ignores images.
 * Note: it does NOT need to know about ImageElement at all.
 * There is no compile-time coupling to AcyclicImage.
 */
class AcyclicWordCounter implements AcyclicVisitor, ParagraphVisitor, HeaderVisitor {
    private int count = 0;

    @Override
    public void visit(AcyclicParagraph paragraph) {
        count += paragraph.getText().trim().split("\\s+").length;
    }

    @Override
    public void visit(AcyclicHeader header) {
        count += header.getText().trim().split("\\s+").length;
    }

    public int getCount() { return count; }
}

/**
 * A visitor that only cares about images — does not reference paragraphs or headers.
 */
class AcyclicImageSizer implements AcyclicVisitor, ImageVisitor {
    private long totalBytes = 0;

    @Override
    public void visit(AcyclicImage image) {
        totalBytes += image.getSizeBytes();
    }

    public long getTotalBytes() { return totalBytes; }
}

class AcyclicVisitorDemo {
    public static void main(String[] args) {
        List<AcyclicElement> elements = List.of(
            new AcyclicParagraph("Hello world, this is a test."),
            new AcyclicHeader("Introduction", 1),
            new AcyclicImage("photo.jpg", 102_400L),
            new AcyclicParagraph("Second paragraph with more words."),
            new AcyclicImage("diagram.png", 204_800L)
        );

        AcyclicWordCounter wordCounter = new AcyclicWordCounter();
        AcyclicImageSizer  imageSizer  = new AcyclicImageSizer();

        for (AcyclicElement el : elements) {
            el.accept(wordCounter);
            el.accept(imageSizer);
        }

        System.out.println("Word count: " + wordCounter.getCount());
        System.out.printf("Image total: %.1f KB%n", imageSizer.getTotalBytes() / 1024.0);
    }
}
```

### Trade-offs of Acyclic Visitor

| Aspect | Classic Visitor | Acyclic Visitor |
|---|---|---|
| Circular dependencies | Yes | No |
| Compile-time completeness | Yes (compiler enforces all visit methods) | No (silent skip at runtime) |
| Adding new element type | Breaks all existing visitors (compile error) | No impact on existing visitors |
| Adding new visitor | Easy, no element changes | Easy, no element changes |
| Runtime safety | High | Lower (instanceof chains) |
| Performance | vtable dispatch only | Additional instanceof checks |

The classic pattern is preferred when the hierarchy is truly stable and compile-time enforcement is valuable. The acyclic variant is preferred in plugin-based architectures where element types and visitor types are contributed by independent teams or plugins.

---

## 9. When to AVOID Visitor

### The Hierarchy Stability Requirement

Visitor is built on a fundamental assumption: **the element type hierarchy is more stable than the set of operations**. When this assumption is violated, Visitor becomes a maintenance liability rather than an asset.

### The N×M Problem

With the classic Visitor pattern:
- Adding a new **operation** = write 1 new class. Cost: O(1).
- Adding a new **element type** = modify all M existing visitors. Cost: O(M).

If your hierarchy is growing rapidly (adding new types frequently), every addition ripples across all existing visitors. This is the exact opposite of the benefit Visitor promises.

### When to Choose Alternatives

**Use abstract methods / polymorphism instead when:**
- The operation set is small and stable (2–4 operations that rarely change).
- Each element type fully encapsulates the logic needed for each operation.
- No cross-cutting state accumulation is needed.

**Use Strategy pattern instead when:**
- The algorithm varies per operation, not per type.
- You need to swap algorithms at runtime (Visitor is always compile-time).

**Use instanceof chains as a last resort when:**
- The hierarchy changes so frequently that any structural pattern adds overhead.
- The type set is truly open-ended (you cannot enumerate all types in advance).

**Use a functional approach (pattern matching) in Java 17+ when:**

```java
// Java 17+ sealed classes + switch expressions replace Visitor cleanly
sealed interface Shape permits Circle, Rectangle, Triangle {}
record Circle(double radius) implements Shape {}
record Rectangle(double w, double h) implements Shape {}
record Triangle(double base, double height) implements Shape {}

double area(Shape s) {
    return switch (s) {
        case Circle    c -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.w() * r.h();
        case Triangle  t -> 0.5 * t.base() * t.height();
    };
}
```

Java's `sealed` + `switch` exhaustiveness checking provides compile-time safety equivalent to Visitor, without the ceremony. For new code on Java 17+, this is often preferable.

---

## 10. Real-World Framework Examples

### Java Compiler API (javax.lang.model)

The Java compiler uses the Visitor pattern extensively for processing annotated programs. The key interface is `javax.lang.model.element.ElementVisitor<R, P>`:

```java
// From the JDK — this is the actual interface signature
public interface ElementVisitor<R, P> {
    R visit(Element e, P p);
    R visitPackage(PackageElement e, P p);
    R visitType(TypeElement e, P p);
    R visitVariable(VariableElement e, P p);
    R visitExecutable(ExecutableElement e, P p);
    R visitTypeParameter(TypeParameterElement e, P p);
    R visitUnknown(Element e, P p);
}
```

The JDK ships `SimpleElementVisitor` as a base class with no-op defaults, so annotation processors can override only the methods they care about.

Similarly, `javax.lang.model.type.TypeVisitor<R, P>` processes type mirrors, and `javax.lang.model.util.Types` provides utility visitors.

### ANTLR4

ANTLR4 generates a Visitor interface and a `BaseVisitor` class from every grammar file. Given a grammar `Expr.g4`, ANTLR generates:

```java
// Auto-generated by ANTLR4
public interface ExprVisitor<T> extends ParseTreeVisitor<T> {
    T visitProg(ExprParser.ProgContext ctx);
    T visitPrintExpr(ExprParser.PrintExprContext ctx);
    T visitAssign(ExprParser.AssignContext ctx);
    T visitBlank(ExprParser.BlankContext ctx);
    T visitParens(ExprParser.ParensContext ctx);
    T visitMulDiv(ExprParser.MulDivContext ctx);
    T visitAddSub(ExprParser.AddSubContext ctx);
    T visitId(ExprParser.IdContext ctx);
    T visitInt(ExprParser.IntContext ctx);
}
```

This is the double-dispatch Visitor in production at massive scale — every language tooling project built on ANTLR4 uses it.

### Apache Calcite (SQL Query Optimizer)

Apache Calcite, used by Hive, Flink, and Drill, uses `RelShuttle` and `RexVisitor` to traverse relational algebra trees and expression trees respectively:

```java
// Simplified from Apache Calcite
public interface RexVisitor<R> {
    R visitInputRef(RexInputRef inputRef);
    R visitLocalRef(RexLocalRef localRef);
    R visitLiteral(RexLiteral literal);
    R visitCall(RexCall call);
    R visitOver(RexOver over);
    R visitCorrelVariable(RexCorrelVariable correlVariable);
    R visitFieldAccess(RexFieldAccess fieldAccess);
    R visitSubQuery(RexSubQuery subQuery);
}
```

Query optimization passes (constant folding, type inference, null propagation) are implemented as `RexVisitor` subclasses.

### Eclipse JDT (Java Development Tools)

Eclipse's Java parser uses `ASTVisitor` with `visit`/`endVisit` pairs for each of ~80 AST node types:

```java
// Simplified from Eclipse JDT
public abstract class ASTVisitor {
    public boolean visit(MethodDeclaration node)  { return true; }
    public boolean visit(ClassDeclaration node)   { return true; }
    public boolean visit(MethodInvocation node)   { return true; }
    public void endVisit(MethodDeclaration node)  { }
    // ... ~160 more methods
}
```

The boolean return from `visit` controls whether children are traversed — an elegant traversal control mechanism built into the pattern.

---

## 11. Trade-offs

### Advantages

**Open/Closed Principle compliance.** New operations are added by writing new visitor classes. Existing element classes are never modified.

**Single Responsibility.** Each visitor class has one job. `HtmlRendererVisitor` only renders HTML. `WordCountVisitor` only counts words. Operations do not pollute the element classes.

**Compile-time type safety.** If a new element type is added to the hierarchy and the visitor interface is updated, the compiler immediately rejects any visitor that does not implement the new `visit` method. This prevents silent failures.

**State accumulation across elements.** A visitor naturally accumulates state (counts, buffers, maps) as it traverses the entire structure. This is awkward with polymorphic methods on individual elements.

**Separation across team boundaries.** The element hierarchy and each visitor can be owned and versioned independently — provided the hierarchy does not change.

### Disadvantages

**Breaking changes when the hierarchy grows.** Adding a new concrete element type to a stable hierarchy requires updating every existing visitor. If there are 15 visitors and you add one new element type, you must modify all 15.

**Encapsulation violation.** Visitors typically need access to the internal state of elements to do meaningful work. This forces elements to expose state through getters that exist solely for visitors. The elements' internal representation becomes an implicit API contract.

**Complexity overhead for simple cases.** If you have 3 element types and 2 operations, plain polymorphism is cleaner. Visitor adds indirection, multiple files, and the cognitive overhead of double dispatch.

**Circular compilation dependency.** The classic visitor interface references all element types, and all element types reference the visitor interface. This can cause circular package dependencies and compilation ordering issues in large module systems (mitigated by the Acyclic Visitor variant).

**Harder to understand for newcomers.** Double dispatch is non-obvious. Most developers who haven't seen the pattern before are confused by why `accept` needs to exist at all.

---

## 12. Comparison with Related Patterns

### Visitor vs. Strategy

| Dimension | Visitor | Strategy |
|---|---|---|
| Varies by | Element type and operation both | Algorithm choice only |
| Object structure | Operates on a fixed type hierarchy | Operates on a single context object |
| Dispatch mechanism | Double dispatch | Single dispatch (via composition/injection) |
| Runtime swappable? | No — structure is fixed at compile time | Yes — strategy can be set at runtime |
| Accumulates state? | Yes — across multiple elements | Usually no — operates on one object |
| OCP axis | Operations vary, types fixed | Algorithms vary, context fixed |

**Rule of thumb:** Use Strategy when one class needs pluggable algorithms. Use Visitor when a hierarchy of classes needs a new operation applied across all of them.

### Visitor vs. Iterator

| Dimension | Visitor | Iterator |
|---|---|---|
| Purpose | Apply operation to each element | Sequentially access elements |
| Type-awareness | Fully type-aware per element type | Type-agnostic (works on common interface) |
| Operation logic | In visitor class | In calling code |
| Structure traversal | Elements control (via accept chain) | Iterator controls |
| Works on heterogeneous trees? | Yes | Only if all types share an interface |

Visitor and Iterator are **complementary**, not competing. Iterator gives you sequential access to elements; Visitor gives you type-specific processing. You often use both: an `Iterator` traverses the structure, and a `Visitor` processes each element.

### Visitor vs. Composite

Composite is a structural pattern for building tree structures. Visitor is a behavioral pattern for processing those trees. They are natural partners:

- Composite defines the tree: `Document` → `Section` → `Paragraph`
- Visitor defines the operations on that tree: `WordCountVisitor`, `HtmlRendererVisitor`

The `accept` method in leaf and composite nodes enables the visitor to traverse the entire tree through recursive double dispatch.

---

## 13. Interview Questions and Model Answers

### Q1. What is the Visitor pattern and what problem does it solve?

**Model Answer:**

Visitor is a behavioral design pattern that separates operations from the object structure they operate on. It solves the problem of adding new operations to an existing class hierarchy without modifying the classes themselves.

The core problem is the tension between two axes of variation: (1) the types in the object hierarchy, and (2) the operations applied to those types. If you put operations as methods inside each class, adding a new operation means modifying every class. Visitor inverts this: each class gets one permanent `accept(Visitor v)` method, and each new operation is encapsulated in a new `Visitor` class that has one method per element type.

This honors the Open/Closed Principle on the operations axis: the element hierarchy is closed for modification but open for new visitors. The trade-off is that the hierarchy becomes closed for modification in the other direction too — adding a new element type requires updating all existing visitors.

---

### Q2. Explain double dispatch. Why does Java need the Visitor pattern to achieve it?

**Model Answer:**

Java is a single-dispatch language. When you invoke a virtual method, the JVM selects the implementation using exactly one runtime type: the type of the receiver object (`this`). This is done via vtable lookup — each class has a table of method pointers, and calling `obj.method()` looks up `obj`'s vtable at runtime.

Method overloading is resolved at compile time based on the static (declared) types of the arguments — not their runtime types. This is static dispatch and involves no vtable.

Double dispatch requires selecting behavior based on the runtime types of two objects simultaneously — typically the element being operated on and the operation (visitor) being applied.

Visitor achieves this with two successive single dispatches:

First dispatch: `element.accept(visitor)` — resolved at runtime by the vtable of `element`. This lands you inside the correct element's `accept` method (e.g., `Paragraph.accept`).

Second dispatch: Inside `Paragraph.accept`, the call is `visitor.visit(this)`. Because `this` is statically typed as `Paragraph` (not `Element`), the compiler selects the correct overload `visit(Paragraph p)`. At runtime, the vtable of `visitor` selects the correct visitor implementation (e.g., `HtmlRendererVisitor.visit(Paragraph)`).

The key insight is that `this` inside `Paragraph.accept` always has static type `Paragraph`, regardless of what reference type was used to point to the object. This allows the second argument to be bound correctly at compile time (selecting the right `visit` overload), while the runtime vtable of `visitor` handles the second dynamic dispatch.

Without the Visitor pattern, if you tried `visitor.visit(element)` where `element` is declared as type `Element`, the compiler would bind to `visit(Element)` regardless of the runtime type — you would lose the first type dispatch. The `accept` indirection is the mechanism that converts the first runtime type (element type) into a compile-time type for the second dispatch.

---

### Q3. What happens if you add a new element type to the hierarchy in the classic Visitor pattern? How does this affect existing code?

**Model Answer:**

Adding a new element type (say, `FootnoteElement`) to the classic Visitor pattern requires:

1. Adding `void visit(FootnoteElement footnote)` to the `DocumentVisitor` interface.
2. Implementing `accept(DocumentVisitor v) { v.visit(this); }` in `FootnoteElement`.
3. Implementing `visit(FootnoteElement footnote)` in every concrete visitor class.

Step 3 is the painful one. If you have 20 visitor classes, you must modify all 20. If any visitor is in a separately compiled module or a third-party library, that module will fail to compile because the `DocumentVisitor` interface has grown.

The compiler enforces this: any concrete class that implements `DocumentVisitor` but does not implement the new `visit(FootnoteElement)` method will produce a compilation error. This is actually a feature from a correctness standpoint — you cannot accidentally ship an `HtmlRendererVisitor` that silently ignores footnotes. But it is a deployment burden in large distributed systems.

This is why Visitor is described as having an asymmetric extensibility property:
- Adding new operations (new visitors): O(1) work, no changes to existing code.
- Adding new element types: O(number of existing visitors) work.

The Acyclic Visitor variant mitigates this at the cost of losing compile-time completeness checks.

---

### Q4. Why must every concrete element class override `accept`? What breaks if a subclass inherits `accept` from a parent?

**Model Answer:**

Every concrete element class must override `accept` because the double dispatch mechanism relies on `this` being statically typed as the **concrete** class, not the base class.

If `Element` had a base implementation:

```java
class Element {
    public void accept(Visitor v) {
        v.visit(this); // 'this' is statically Element
    }
}

class Paragraph extends Element {
    // does NOT override accept — inherits from Element
}
```

When you call `paragraph.accept(visitor)`, the JVM correctly finds `Element.accept` (since `Paragraph` inherits it). But inside `Element.accept`, `this` is typed as `Element`, not `Paragraph`. The call `v.visit(this)` will bind to `visit(Element)`, not `visit(Paragraph)`.

You lose the first dispatch. The visitor receives an `Element` reference and cannot access the `Paragraph`-specific fields without casting — defeating the entire purpose of the pattern.

The fix is mechanical but required: every concrete (leaf) class must implement:

```java
@Override
public void accept(Visitor v) {
    v.visit(this); // 'this' is statically Paragraph — compiler selects visit(Paragraph)
}
```

This is a "copy-paste" implementation that looks redundant but is semantically critical. The static type of `this` changes per class, making each override functionally distinct even though they look identical syntactically.

---

### Q5. How would you implement Visitor with generic return types to avoid mutable state in visitors?

**Model Answer:**

The classic Visitor pattern uses void `visit` methods and accumulates results via mutable instance fields. This has several drawbacks: it is not thread-safe, visitors are stateful and cannot be shared, and the API requires a separate `getResult()` call.

A cleaner approach is to make `visit` methods return a value using generics:

```java
interface AstVisitor<T> {
    T visitNumber(NumberNode node);
    T visitAdd(AddNode node);
    T visitMultiply(MultiplyNode node);
}

interface AstNode {
    <T> T accept(AstVisitor<T> visitor);
}
```

With this design, `EvaluatorVisitor implements AstVisitor<Double>` and each `visit` method returns the computed value directly. The result of `expr.accept(evaluator)` is the final value without any side effects.

This is the approach used in the AST implementation in section 6 of this document. It is purely functional — visitors are stateless objects that can be safely shared across threads. The recursive structure of `visitAdd` calls `node.getLeft().accept(this)` and `node.getRight().accept(this)` and returns their sum, threading the computation through the tree without any mutable accumulator.

The limitation is that some visitors genuinely need to accumulate state across all elements (like word count across a flat document) — in those cases, a mutable field remains necessary, but you can scope it to a single traversal by resetting in the `visit(Document)` method.

---

### Q6. Describe the Acyclic Visitor pattern and when you would use it over the classic Visitor.

**Model Answer:**

The Acyclic Visitor pattern breaks the mutual circular dependency between the visitor interface and the element hierarchy in classic Visitor.

In classic Visitor, the visitor interface references all element types, and all element types reference the visitor interface. In a multi-module system, this forces all modules into a single compilation unit or requires careful layering.

The Acyclic Visitor solution:

1. Define an empty marker interface `AcyclicVisitor` with no methods.
2. For each element type, define a separate `XxxVisitor` interface that extends `AcyclicVisitor` and declares one `visit(XxxElement)` method.
3. The `accept` method casts the visitor to the specific interface and calls it, or silently skips if the visitor does not implement that interface.
4. Concrete visitors implement only the `XxxVisitor` interfaces for element types they care about.

This eliminates circular dependencies: `AcyclicVisitor` has no element imports; each element's `XxxVisitor` is co-located with the element class; concrete visitors only import the types they process.

The trade-off is loss of compile-time completeness guarantees. With classic Visitor, forgetting to implement `visit(NewElement)` is a compile error. With Acyclic Visitor, the element is silently ignored. This moves a class of errors from compile time to runtime — which is appropriate when extensibility and module isolation outweigh the risk.

Use Acyclic Visitor when:
- The system has a plugin architecture where third parties contribute both elements and visitors.
- Elements and visitors are in separate Maven/Gradle modules with independent release cycles.
- "Partial visitors" that only care about a subset of element types are a first-class use case.

Use classic Visitor when:
- The hierarchy is small and owned by one team.
- Compile-time completeness enforcement is critical.
- All types will always be compiled together.

---

### Q7. How does Visitor interact with the Composite pattern in a tree structure?

**Model Answer:**

Visitor and Composite are the two most natural pattern partners in object-oriented design. Composite builds the tree structure; Visitor processes it.

In a Composite tree, you have leaf nodes (like `Paragraph`, `Image`) and composite nodes (like `Document`, `Section`) that contain other nodes. The traversal of the tree is typically encoded in the composite node's `accept` method:

```java
class Document implements DocumentElement {
    private final List<DocumentElement> children;

    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visit(this);            // process this node
        for (DocumentElement child : children) {
            child.accept(visitor);      // recursively visit children
        }
    }
}
```

This is depth-first pre-order traversal. The composite node processes itself first, then its children.

Key design decisions at this intersection:

**Who controls traversal?** Three options: (a) the composite element traverses children in `accept` (most common, shown above); (b) the visitor traverses by calling `element.getChildren()` and recursing; (c) an external iterator traverses. Option (a) encapsulates structure but prevents the visitor from short-circuiting. Option (b) exposes structure to the visitor but allows it to skip subtrees.

**Pre-order vs. post-order:** The default is pre-order (process node before children). Some operations need post-order (process children first, then the parent). Eclipse JDT's `ASTVisitor` uses `visit` (pre) + `endVisit` (post) pairs for exactly this reason.

**State inheritance down the tree:** A visitor might need context from parent nodes when processing children. The standard solution is a stack in the visitor: `visit(Section s)` pushes context onto a stack; `endVisit(Section s)` pops it.

---

### Q8. Java 17+ introduced sealed classes with exhaustive switch expressions. How does this change the case for using Visitor? When would you still choose Visitor over sealed + switch?

**Model Answer:**

Java 17's sealed classes combined with pattern-matching switch expressions provide a native language mechanism that achieves the same compile-time safety as Visitor, with less ceremony:

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
record Circle(double r) implements Shape {}
record Rectangle(double w, double h) implements Shape {}
record Triangle(double b, double h) implements Shape {}

// Exhaustive — compiler rejects if any permitted type is missing
double area(Shape s) {
    return switch (s) {
        case Circle    c -> Math.PI * c.r() * c.r();
        case Rectangle r -> r.w() * r.h();
        case Triangle  t -> 0.5 * t.b() * t.h();
    };
}
```

If `Diamond` is added to the `sealed` set, every switch over `Shape` becomes a compile error until `Diamond` is handled. This is the same guarantee Visitor provides.

**When to still use Visitor over sealed + switch:**

1. **The visitor needs state.** `WordCountVisitor` accumulates a running total across many elements. A `switch` expression is stateless by nature.

2. **The visitor is an object that gets passed around.** In the document example, you create a `HtmlRendererVisitor` and pass it to `document.accept(visitor)`. Visitors are first-class objects that can be stored in fields, passed to constructors, registered in lists. A `switch` expression is not an object.

3. **Visitor decouples operation from traversal.** With sealed + switch, traversal is explicit inside each function. With Visitor, traversal is encoded once in `accept` methods and reused.

4. **You need multiple distinct operations shipped as separate classes/modules.** Different teams can each write a `Visitor` implementation independently. A `switch` expression cannot be extended by third parties.

5. **Pre-Java 17 codebases.** Sealed classes require Java 17+. Many enterprise codebases target Java 11 or 8.

6. **The hierarchy is not sealed.** Sealed is a closed-world assumption. If third-party code can extend your types (not `sealed`), Visitor remains the only type-safe option.

The honest conclusion is that for new code on Java 17+ with a closed hierarchy and stateless operations, `sealed` + `switch` is often cleaner than Visitor. For stateful, object-oriented, multi-module, or multi-team scenarios, Visitor remains the right tool.

---

*End of 08-visitor.md*
