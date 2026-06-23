# Command Pattern

> **Behavioral Pattern — Gang of Four**
>
> Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.

---

## Table of Contents

1. [Intent and Problem It Solves](#1-intent-and-problem-it-solves)
2. [When to Use / When NOT to Use](#2-when-to-use--when-not-to-use)
3. [UML Class Diagram](#3-uml-class-diagram)
4. [Implementation 1 — Text Editor with Full Undo/Redo Stack](#4-implementation-1--text-editor-with-full-undoredo-stack)
5. [Implementation 2 — Smart Home Remote with Macro Commands](#5-implementation-2--smart-home-remote-with-macro-commands)
6. [Command Queue and Audit Logging](#6-command-queue-and-audit-logging)
7. [Macro Commands (Composite Commands)](#7-macro-commands-composite-commands)
8. [Thread Pool / Java Executor as Command Pattern](#8-thread-pool--java-executor-as-command-pattern)
9. [Real-World Framework Examples](#9-real-world-framework-examples)
10. [Trade-offs](#10-trade-offs)
11. [Common Interview Discussion Points](#11-common-interview-discussion-points)
12. [Interview Questions and Model Answers](#12-interview-questions-and-model-answers)
13. [Comparison with Related Patterns](#13-comparison-with-related-patterns)

---

## 1. Intent and Problem It Solves

### The Core Problem

Imagine you are building a GUI application. A user can trigger the same action — say, "save the document" — in three different ways: clicking a toolbar button, pressing Ctrl+S, or selecting File → Save from the menu. Each of these widgets has its own event handler. Without a unifying abstraction, you end up writing the same save logic three times, or coupling all three widgets to the `Document` class directly.

Now add undo. Now add macros. Now add operation logging. Every new capability forces you to touch every widget. This is the coupling problem the Command pattern was invented to solve.

### The Insight

The Command pattern's key insight is to **reify the request** — to turn a verb (an action) into a noun (an object). Instead of calling `document.save()` directly, you create a `SaveCommand` object that knows how to call `document.save()`. You then hand this object to whoever wants to trigger the action. The triggerer does not need to know what the command does; it only needs to know how to call `execute()`.

This decoupling creates four powerful capabilities:

**1. Parameterization**: Commands are first-class objects. You can pass them as arguments, store them in collections, and retrieve them later — the same way you pass integers or strings.

**2. Deferred / Queued Execution**: The object can be created now and executed later — at a scheduled time, in a different thread, or when preconditions are met.

**3. Undo / Redo**: A command that knows how to execute an action also knows how to reverse it. By maintaining a history stack of executed commands, you get undo/redo for free.

**4. Audit Logging / Replay**: Because every action is a serializable object, you can log every command executed, inspect the log, and replay operations — which is exactly how event sourcing and write-ahead logs work at database scale.

### Participants

| Role | Responsibility |
|---|---|
| **Command** | Interface declaring `execute()` and (optionally) `undo()` |
| **ConcreteCommand** | Implements `Command`; holds a reference to the Receiver and the data needed to perform the action |
| **Receiver** | The object that knows how to perform the actual work (e.g., `Document`, `Light`) |
| **Invoker** | Asks the command to carry out the request; holds the command reference; may maintain history |
| **Client** | Creates ConcreteCommand objects and sets their Receiver |

The critical relationship: the **Invoker** depends only on the `Command` interface — never on any ConcreteCommand or Receiver. This is Dependency Inversion applied to actions.

---

## 2. When to Use / When NOT to Use

### Use the Command Pattern When

- **You need undo/redo** — any application where operations must be reversible (text editors, drawing apps, transaction systems, games).
- **You need operation queuing** — job queues, task schedulers, work queues in thread pools where operations are collected and dispatched later.
- **You need audit logging / replay** — financial systems, compliance-sensitive applications, game replays, event sourcing architectures.
- **You need macro recording** — allowing users to record a sequence of operations and replay them as a single composite action.
- **You want to decouple the sender from the receiver** — GUI buttons/menu items that should not know which domain class they are invoking.
- **You need transactional behavior** — grouping operations so they either all succeed or all roll back.
- **You need to support operation scheduling** — running commands at a specific time or in response to a condition.

### Do NOT Use the Command Pattern When

- **The operation is simple and undo is not needed** — adding a full command hierarchy for a trivial single-responsibility action is over-engineering. If you just need polymorphic behavior without history, use Strategy.
- **You have no variation in how actions are triggered or stored** — if the sender always calls the receiver directly with no need for queuing or history, the indirection adds no value.
- **You are concerned about object proliferation** — every distinct operation creates a new class. In systems with hundreds of operations, the class count becomes a maintenance burden if the commands carry no real logic beyond delegation.
- **Memory is constrained** — maintaining a command history means holding all command objects in memory. For operations with large data payloads (e.g., image editing), the Memento pattern or a diff-based approach may be more memory-efficient.
- **The "undo" semantics are ambiguous or impossible** — some operations cannot meaningfully be reversed (sending an email, making an external API call with side effects). Forcing Command onto these creates the illusion of safety without the reality.

---

## 3. UML Class Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          COMMAND PATTERN                                │
└─────────────────────────────────────────────────────────────────────────┘

         ┌──────────────────┐
         │     «Client»     │
         │                  │
         │  creates commands│
         │  sets receivers  │
         └────────┬─────────┘
                  │ creates
                  ▼
         ┌──────────────────┐          ┌──────────────────┐
         │    «interface»   │          │    «Invoker»     │
         │     Command      │◄─────────│   (e.g. Remote,  │
         │                  │  holds   │   CommandQueue,  │
         │ + execute(): void│          │   Button)        │
         │ + undo(): void   │          │                  │
         └────────┬─────────┘          │ + setCommand()   │
                  │                    │ + executeCommand()│
        ┌─────────┼──────────┐         │ + undoLast()     │
        │         │          │         └──────────────────┘
        ▼         ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────────────┐
│TypeCommand│ │DeleteCmd │ │  MacroCommand    │
│          │ │          │ │  (Composite)     │
│-document │ │-document │ │                  │
│-text     │ │-position │ │-commands: List   │
│-position │ │-deleted  │ │                  │
│          │ │          │ │+execute(): void  │
│+execute()│ │+execute()│ │+undo(): void     │
│+undo()   │ │+undo()   │ └──────────────────┘
└────┬─────┘ └────┬─────┘
     │            │
     │ uses       │ uses
     ▼            ▼
┌─────────────────────────────┐
│         «Receiver»          │
│   (e.g. TextDocument,       │
│    SmartDevice, etc.)       │
│                             │
│ + insertText(pos, text)     │
│ + deleteText(pos, len)      │
│ + getContent(): String      │
└─────────────────────────────┘


History / Invoker internals:

  ┌─────────────────────────────────────────────┐
  │              CommandHistory                 │
  │                                             │
  │  undoStack: Deque<Command>  ──► [C3][C2][C1]│
  │  redoStack: Deque<Command>  ──► [C4]        │
  │                                             │
  │  + push(cmd): void    (after execute)       │
  │  + undo(): void       (pop undoStack,       │
  │                        push redoStack)      │
  │  + redo(): void       (pop redoStack,       │
  │                        push undoStack)      │
  └─────────────────────────────────────────────┘
```

---

## 4. Implementation 1 — Text Editor with Full Undo/Redo Stack

This implementation models a production-quality text editor command system. Every editing operation is encapsulated as a command. The `CommandHistory` provides unlimited undo/redo. Commands capture exactly the data they need to reverse themselves — no external snapshot required.

```java
package patterns.behavioral.command.editor;

import java.util.*;
import java.time.Instant;
import java.util.logging.Logger;

// ─────────────────────────────────────────────────────────
// COMMAND INTERFACE
// ─────────────────────────────────────────────────────────

/**
 * Base Command interface for all editor operations.
 *
 * The description() method enables command-history display ("Edit > History")
 * and audit logging. getTimestamp() enables time-travel debugging.
 */
public interface EditorCommand {
    void execute();
    void undo();
    String description();
    default Instant getTimestamp() { return Instant.now(); }
}

// ─────────────────────────────────────────────────────────
// RECEIVER: The actual document model
// ─────────────────────────────────────────────────────────

/**
 * The text document that holds content and performs actual mutations.
 * Commands operate on this class — they do NOT contain document logic.
 *
 * Thread-safety note: In a real editor this would be synchronized or
 * use a lock; omitted here for clarity.
 */
public class TextDocument {

    private final StringBuilder content;
    private final String name;
    private int cursorPosition;

    // Text formatting: bold ranges stored as (start, end) pairs
    private final List<int[]> boldRanges = new ArrayList<>();
    private final List<int[]> italicRanges = new ArrayList<>();

    public TextDocument(String name) {
        this.name = name;
        this.content = new StringBuilder();
        this.cursorPosition = 0;
    }

    // ── Primitive operations (called by Commands) ──

    public void insertAt(int position, String text) {
        validatePosition(position);
        content.insert(position, text);
        // Shift formatting ranges that are past the insertion point
        shiftRanges(boldRanges, position, text.length());
        shiftRanges(italicRanges, position, text.length());
        cursorPosition = position + text.length();
    }

    public String deleteAt(int position, int length) {
        validatePosition(position);
        if (position + length > content.length()) {
            throw new IllegalArgumentException(
                "Delete range [" + position + ", " + (position + length) + ") " +
                "exceeds document length " + content.length());
        }
        String deleted = content.substring(position, position + length);
        content.delete(position, position + length);
        shiftRanges(boldRanges, position, -length);
        shiftRanges(italicRanges, position, -length);
        cursorPosition = position;
        return deleted;
    }

    public void applyBold(int start, int end) {
        boldRanges.add(new int[]{start, end});
    }

    public void removeBold(int start, int end) {
        boldRanges.removeIf(r -> r[0] == start && r[1] == end);
    }

    public void applyItalic(int start, int end) {
        italicRanges.add(new int[]{start, end});
    }

    public void removeItalic(int start, int end) {
        italicRanges.removeIf(r -> r[0] == start && r[1] == end);
    }

    public void replacePasteContent(int position, String oldContent, String newContent) {
        // Used by PasteCommand.undo(): remove the pasted text and restore original
        content.delete(position, position + newContent.length());
        content.insert(position, oldContent);
        cursorPosition = position;
    }

    // ── Read operations ──

    public String getContent() { return content.toString(); }
    public int length() { return content.length(); }
    public String getName() { return name; }
    public int getCursorPosition() { return cursorPosition; }

    public String substring(int start, int end) {
        return content.substring(start, Math.min(end, content.length()));
    }

    // ── Helpers ──

    private void validatePosition(int position) {
        if (position < 0 || position > content.length()) {
            throw new IllegalArgumentException(
                "Position " + position + " out of bounds [0, " + content.length() + "]");
        }
    }

    private void shiftRanges(List<int[]> ranges, int from, int delta) {
        for (int[] range : ranges) {
            if (range[0] >= from) range[0] += delta;
            if (range[1] >= from) range[1] += delta;
        }
    }

    @Override
    public String toString() {
        return "TextDocument{name='" + name + "', length=" + content.length() +
               ", cursor=" + cursorPosition + "}";
    }
}

// ─────────────────────────────────────────────────────────
// CONCRETE COMMANDS
// ─────────────────────────────────────────────────────────

/**
 * Inserts text at a specific position in the document.
 *
 * Undo: delete the inserted text (we know exactly where it went and how long it is).
 */
public class TypeCommand implements EditorCommand {

    private final TextDocument document;
    private final int position;
    private final String text;
    private final Instant timestamp;

    public TypeCommand(TextDocument document, int position, String text) {
        Objects.requireNonNull(document, "document must not be null");
        Objects.requireNonNull(text, "text must not be null");
        if (text.isEmpty()) throw new IllegalArgumentException("text must not be empty");
        this.document = document;
        this.position = position;
        this.text = text;
        this.timestamp = Instant.now();
    }

    @Override
    public void execute() {
        document.insertAt(position, text);
    }

    @Override
    public void undo() {
        // We inserted `text.length()` characters at `position` — delete exactly those
        document.deleteAt(position, text.length());
    }

    @Override
    public String description() {
        String preview = text.length() > 20 ? text.substring(0, 20) + "..." : text;
        return "Type \"" + preview + "\" at position " + position;
    }

    @Override
    public Instant getTimestamp() { return timestamp; }
}

/**
 * Deletes a range of text from the document.
 *
 * Undo: re-insert the deleted text at the original position.
 * The deleted text is captured at execute() time, not at construction time,
 * because the document could have changed between construction and execution.
 */
public class DeleteCommand implements EditorCommand {

    private final TextDocument document;
    private final int position;
    private final int length;
    private final Instant timestamp;

    // Captured at execute time — unknown until the command runs
    private String deletedText;

    public DeleteCommand(TextDocument document, int position, int length) {
        Objects.requireNonNull(document, "document must not be null");
        if (position < 0) throw new IllegalArgumentException("position must be >= 0");
        if (length <= 0) throw new IllegalArgumentException("length must be > 0");
        this.document = document;
        this.position = position;
        this.length = length;
        this.timestamp = Instant.now();
    }

    @Override
    public void execute() {
        // Capture what we are about to delete BEFORE deleting it
        deletedText = document.deleteAt(position, length);
    }

    @Override
    public void undo() {
        if (deletedText == null) {
            throw new IllegalStateException(
                "DeleteCommand.undo() called before execute() — no captured text to restore");
        }
        document.insertAt(position, deletedText);
    }

    @Override
    public String description() {
        String preview = (deletedText != null)
            ? (deletedText.length() > 20 ? deletedText.substring(0, 20) + "..." : deletedText)
            : (length + " chars");
        return "Delete \"" + preview + "\" at position " + position;
    }

    @Override
    public Instant getTimestamp() { return timestamp; }
}

/**
 * Pastes text from a clipboard at the current cursor position,
 * replacing any currently selected text.
 *
 * Demonstrates a command that must capture pre-execution state
 * (the replaced content) to support undo.
 */
public class PasteCommand implements EditorCommand {

    private final TextDocument document;
    private final int pastePosition;
    private final String clipboardContent;
    private final int selectionLength; // 0 if no selection (pure insert)
    private final Instant timestamp;

    // Captured at execute() time
    private String replacedText;

    public PasteCommand(TextDocument document, int pastePosition,
                        String clipboardContent, int selectionLength) {
        Objects.requireNonNull(document, "document must not be null");
        Objects.requireNonNull(clipboardContent, "clipboardContent must not be null");
        if (pastePosition < 0) throw new IllegalArgumentException("pastePosition must be >= 0");
        if (selectionLength < 0) throw new IllegalArgumentException("selectionLength must be >= 0");
        this.document = document;
        this.pastePosition = pastePosition;
        this.clipboardContent = clipboardContent;
        this.selectionLength = selectionLength;
        this.timestamp = Instant.now();
    }

    @Override
    public void execute() {
        if (selectionLength > 0) {
            // Capture what we are about to replace
            replacedText = document.deleteAt(pastePosition, selectionLength);
        } else {
            replacedText = "";
        }
        document.insertAt(pastePosition, clipboardContent);
    }

    @Override
    public void undo() {
        if (replacedText == null) {
            throw new IllegalStateException("PasteCommand.undo() called before execute()");
        }
        // Remove what we pasted, restore what was there before
        document.replacePasteContent(pastePosition, replacedText, clipboardContent);
    }

    @Override
    public String description() {
        String preview = clipboardContent.length() > 20
            ? clipboardContent.substring(0, 20) + "..." : clipboardContent;
        return "Paste \"" + preview + "\" at position " + pastePosition +
               (selectionLength > 0 ? " (replacing " + selectionLength + " chars)" : "");
    }

    @Override
    public Instant getTimestamp() { return timestamp; }
}

/**
 * Applies bold or italic formatting to a text range.
 *
 * Demonstrates that commands are not limited to content mutation —
 * they work equally well for metadata/attribute changes.
 */
public class FormatCommand implements EditorCommand {

    public enum FormatType { BOLD, ITALIC }

    private final TextDocument document;
    private final int start;
    private final int end;
    private final FormatType formatType;
    private final Instant timestamp;

    public FormatCommand(TextDocument document, int start, int end, FormatType formatType) {
        Objects.requireNonNull(document, "document must not be null");
        Objects.requireNonNull(formatType, "formatType must not be null");
        if (start < 0 || end <= start) {
            throw new IllegalArgumentException("Invalid range [" + start + ", " + end + ")");
        }
        this.document = document;
        this.start = start;
        this.end = end;
        this.formatType = formatType;
        this.timestamp = Instant.now();
    }

    @Override
    public void execute() {
        switch (formatType) {
            case BOLD   -> document.applyBold(start, end);
            case ITALIC -> document.applyItalic(start, end);
        }
    }

    @Override
    public void undo() {
        switch (formatType) {
            case BOLD   -> document.removeBold(start, end);
            case ITALIC -> document.removeItalic(start, end);
        }
    }

    @Override
    public String description() {
        return "Apply " + formatType + " to range [" + start + ", " + end + ")";
    }

    @Override
    public Instant getTimestamp() { return timestamp; }
}

// ─────────────────────────────────────────────────────────
// COMMAND HISTORY (Invoker)
// ─────────────────────────────────────────────────────────

/**
 * Maintains the undo/redo stacks and executes commands.
 *
 * Design notes:
 * - Uses Deque (ArrayDeque) for O(1) push/pop on both ends.
 * - maxHistorySize prevents unbounded memory growth. When the limit is
 *   reached the oldest command is evicted from the undo stack.
 * - Executing a new command always clears the redo stack — this matches
 *   the behavior of every major text editor (Photoshop, VS Code, Word).
 * - The history log (allCommands) is append-only for auditing, separate
 *   from the undo/redo stacks.
 */
public class CommandHistory {

    private static final Logger log = Logger.getLogger(CommandHistory.class.getName());

    private final Deque<EditorCommand> undoStack = new ArrayDeque<>();
    private final Deque<EditorCommand> redoStack = new ArrayDeque<>();
    private final List<EditorCommand> auditLog   = new ArrayList<>();
    private final int maxHistorySize;

    public CommandHistory(int maxHistorySize) {
        if (maxHistorySize <= 0) throw new IllegalArgumentException("maxHistorySize must be > 0");
        this.maxHistorySize = maxHistorySize;
    }

    /**
     * Execute a command and push it onto the undo stack.
     * Clears the redo stack — consistent with standard editor behavior.
     */
    public void execute(EditorCommand command) {
        Objects.requireNonNull(command, "command must not be null");
        command.execute();
        pushToUndoStack(command);
        redoStack.clear(); // New action invalidates redo history
        auditLog.add(command);
        log.fine(() -> "EXEC: " + command.description());
    }

    /**
     * Undo the most recently executed command.
     *
     * @return true if undo was performed, false if nothing to undo
     */
    public boolean undo() {
        if (undoStack.isEmpty()) {
            log.warning("Nothing to undo");
            return false;
        }
        EditorCommand command = undoStack.pop();
        command.undo();
        redoStack.push(command);
        log.fine(() -> "UNDO: " + command.description());
        return true;
    }

    /**
     * Redo the most recently undone command.
     *
     * @return true if redo was performed, false if nothing to redo
     */
    public boolean redo() {
        if (redoStack.isEmpty()) {
            log.warning("Nothing to redo");
            return false;
        }
        EditorCommand command = redoStack.pop();
        command.execute();
        undoStack.push(command);
        log.fine(() -> "REDO: " + command.description());
        return true;
    }

    /** @return true if there is at least one command available to undo */
    public boolean canUndo() { return !undoStack.isEmpty(); }

    /** @return true if there is at least one command available to redo */
    public boolean canRedo() { return !redoStack.isEmpty(); }

    /** Returns an unmodifiable view of the full audit log in chronological order. */
    public List<EditorCommand> getAuditLog() {
        return Collections.unmodifiableList(auditLog);
    }

    /** Returns the descriptions of commands currently in the undo stack, newest first. */
    public List<String> getUndoHistory() {
        List<String> history = new ArrayList<>();
        for (EditorCommand cmd : undoStack) {
            history.add(cmd.description());
        }
        return Collections.unmodifiableList(history);
    }

    public int undoStackSize() { return undoStack.size(); }
    public int redoStackSize() { return redoStack.size(); }

    private void pushToUndoStack(EditorCommand command) {
        if (undoStack.size() >= maxHistorySize) {
            // Evict oldest command (bottom of stack = last element in ArrayDeque)
            ((ArrayDeque<EditorCommand>) undoStack).removeLast();
            log.info("Command history limit reached; oldest entry evicted");
        }
        undoStack.push(command);
    }
}

// ─────────────────────────────────────────────────────────
// THE EDITOR (Client + thin Invoker facade)
// ─────────────────────────────────────────────────────────

/**
 * The text editor ties together the document, command history, and clipboard.
 * It creates commands and delegates execution to CommandHistory.
 *
 * This is the "Client" role in the pattern: it knows about both the
 * Receiver (TextDocument) and the ConcreteCommands.
 */
public class TextEditor {

    private final TextDocument document;
    private final CommandHistory history;
    private String clipboard = "";

    public TextEditor(String documentName) {
        this.document = new TextDocument(documentName);
        this.history  = new CommandHistory(200); // Keep last 200 operations
    }

    // ── Editing operations ──

    public void type(String text) {
        history.execute(new TypeCommand(document, document.getCursorPosition(), text));
    }

    public void typeAt(int position, String text) {
        history.execute(new TypeCommand(document, position, text));
    }

    public void delete(int position, int length) {
        history.execute(new DeleteCommand(document, position, length));
    }

    public void copy(int start, int length) {
        // Copy does not mutate the document — not a command, just a read
        clipboard = document.substring(start, start + length);
    }

    public void paste(int position, int selectionLength) {
        if (clipboard.isEmpty()) return;
        history.execute(new PasteCommand(document, position, clipboard, selectionLength));
    }

    public void bold(int start, int end) {
        history.execute(new FormatCommand(document, start, end, FormatCommand.FormatType.BOLD));
    }

    public void italic(int start, int end) {
        history.execute(new FormatCommand(document, start, end, FormatCommand.FormatType.ITALIC));
    }

    // ── Undo / Redo ──

    public boolean undo() { return history.undo(); }
    public boolean redo() { return history.redo(); }
    public boolean canUndo() { return history.canUndo(); }
    public boolean canRedo() { return history.canRedo(); }

    // ── Inspection ──

    public String getContent() { return document.getContent(); }
    public String getDocumentName() { return document.getName(); }
    public List<String> getUndoHistory() { return history.getUndoHistory(); }
    public List<EditorCommand> getAuditLog() { return history.getAuditLog(); }

    public void printState() {
        System.out.println("────────────────────────────────");
        System.out.println("Document : " + document.getName());
        System.out.println("Content  : \"" + document.getContent() + "\"");
        System.out.println("Cursor   : " + document.getCursorPosition());
        System.out.println("Can Undo : " + canUndo() + " (" + history.undoStackSize() + " steps)");
        System.out.println("Can Redo : " + canRedo() + " (" + history.redoStackSize() + " steps)");
        System.out.println("────────────────────────────────");
    }
}

// ─────────────────────────────────────────────────────────
// DEMO
// ─────────────────────────────────────────────────────────

public class TextEditorDemo {
    public static void main(String[] args) {
        TextEditor editor = new TextEditor("design-doc.txt");

        // Type some content
        editor.type("Hello, World!");
        editor.printState();
        // Content: "Hello, World!"

        // Type more
        editor.type(" How are you?");
        editor.printState();
        // Content: "Hello, World! How are you?"

        // Delete the greeting
        editor.delete(0, 7);
        editor.printState();
        // Content: "World! How are you?"

        // Apply bold formatting to "World"
        editor.bold(0, 5);
        System.out.println("Applied bold to 'World'");

        // Copy and paste
        editor.copy(0, 5); // copies "World"
        editor.paste(7, 0); // paste after "! "
        editor.printState();
        // Content: "World! World How are you?"

        System.out.println("\n--- Undo sequence ---");
        editor.undo(); editor.printState(); // Undo paste
        editor.undo(); editor.printState(); // Undo bold
        editor.undo(); editor.printState(); // Undo delete
        editor.undo(); editor.printState(); // Undo second type
        editor.undo(); editor.printState(); // Undo first type

        System.out.println("\n--- Redo sequence ---");
        editor.redo(); editor.printState(); // Redo first type
        editor.redo(); editor.printState(); // Redo second type

        // New command clears redo stack
        editor.type(" NEW CONTENT");
        System.out.println("Can redo after new command: " + editor.canRedo()); // false

        System.out.println("\n--- Audit Log ---");
        editor.getAuditLog().forEach(cmd ->
            System.out.println("[" + cmd.getTimestamp() + "] " + cmd.description()));
    }
}
```

---

## 5. Implementation 2 — Smart Home Remote with Macro Commands

This implementation showcases macro commands, scheduling, and persistent logging — three capabilities unique to the Command pattern.

```java
package patterns.behavioral.command.smarthome;

import java.util.*;
import java.util.concurrent.*;
import java.time.Instant;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.logging.*;

// ─────────────────────────────────────────────────────────
// COMMAND INTERFACE
// ─────────────────────────────────────────────────────────

public interface HomeCommand {
    void execute();
    void undo();
    String getName();
    String describe();
}

// ─────────────────────────────────────────────────────────
// RECEIVERS
// ─────────────────────────────────────────────────────────

public class Light {
    private final String location;
    private boolean on = false;
    private int brightness = 100; // 0-100

    public Light(String location) { this.location = location; }

    public void turnOn()  { on = true;  System.out.println(location + " light ON"); }
    public void turnOff() { on = false; System.out.println(location + " light OFF"); }

    public void setBrightness(int level) {
        this.brightness = Math.max(0, Math.min(100, level));
        System.out.println(location + " light brightness -> " + this.brightness + "%");
    }

    public boolean isOn() { return on; }
    public int getBrightness() { return brightness; }
    public String getLocation() { return location; }
}

public class Thermostat {
    private int temperature = 22; // Celsius
    private boolean heating = false;

    public void setTemperature(int temp) {
        int old = this.temperature;
        this.temperature = temp;
        this.heating = temp > old;
        System.out.println("Thermostat set to " + temp + "°C " + (heating ? "(heating)" : "(cooling)"));
    }

    public int getTemperature() { return temperature; }
    public boolean isHeating()  { return heating; }
}

public class SecuritySystem {
    private boolean armed = false;
    private String mode = "HOME"; // HOME, AWAY, NIGHT

    public void arm(String mode) {
        this.armed = true;
        this.mode  = mode;
        System.out.println("Security system ARMED in " + mode + " mode");
    }

    public void disarm() {
        this.armed = false;
        System.out.println("Security system DISARMED");
    }

    public boolean isArmed() { return armed; }
    public String  getMode() { return mode;  }
}

public class MusicSystem {
    private boolean playing = false;
    private String  playlist = "None";
    private int     volume   = 50;

    public void play(String playlist) {
        this.playing  = true;
        this.playlist = playlist;
        System.out.println("Music: playing \"" + playlist + "\" at volume " + volume);
    }

    public void stop() {
        this.playing = false;
        System.out.println("Music: stopped");
    }

    public void setVolume(int vol) {
        this.volume = Math.max(0, Math.min(100, vol));
        System.out.println("Music: volume -> " + this.volume);
    }

    public boolean isPlaying() { return playing; }
    public String  getPlaylist() { return playlist; }
    public int     getVolume()   { return volume; }
}

// ─────────────────────────────────────────────────────────
// CONCRETE COMMANDS — Lights
// ─────────────────────────────────────────────────────────

public class LightOnCommand implements HomeCommand {
    private final Light light;

    public LightOnCommand(Light light) { this.light = light; }

    @Override public void execute() { light.turnOn(); }
    @Override public void undo()    { light.turnOff(); }
    @Override public String getName() { return "LightOn"; }
    @Override public String describe() {
        return "Turn on " + light.getLocation() + " light";
    }
}

public class LightOffCommand implements HomeCommand {
    private final Light light;

    public LightOffCommand(Light light) { this.light = light; }

    @Override public void execute() { light.turnOff(); }
    @Override public void undo()    { light.turnOn(); }
    @Override public String getName() { return "LightOff"; }
    @Override public String describe() {
        return "Turn off " + light.getLocation() + " light";
    }
}

public class DimLightCommand implements HomeCommand {
    private final Light light;
    private final int   targetBrightness;
    private int         previousBrightness;

    public DimLightCommand(Light light, int targetBrightness) {
        this.light             = light;
        this.targetBrightness  = targetBrightness;
    }

    @Override
    public void execute() {
        previousBrightness = light.getBrightness();
        light.setBrightness(targetBrightness);
    }

    @Override
    public void undo() {
        light.setBrightness(previousBrightness);
    }

    @Override public String getName() { return "DimLight"; }
    @Override public String describe() {
        return "Dim " + light.getLocation() + " light to " + targetBrightness + "%";
    }
}

// ─────────────────────────────────────────────────────────
// CONCRETE COMMANDS — Thermostat
// ─────────────────────────────────────────────────────────

public class SetTemperatureCommand implements HomeCommand {
    private final Thermostat thermostat;
    private final int        targetTemp;
    private int              previousTemp;

    public SetTemperatureCommand(Thermostat thermostat, int targetTemp) {
        this.thermostat = thermostat;
        this.targetTemp = targetTemp;
    }

    @Override
    public void execute() {
        previousTemp = thermostat.getTemperature();
        thermostat.setTemperature(targetTemp);
    }

    @Override
    public void undo() {
        thermostat.setTemperature(previousTemp);
    }

    @Override public String getName() { return "SetTemperature"; }
    @Override public String describe() {
        return "Set thermostat to " + targetTemp + "°C";
    }
}

// ─────────────────────────────────────────────────────────
// CONCRETE COMMANDS — Security
// ─────────────────────────────────────────────────────────

public class ArmSecurityCommand implements HomeCommand {
    private final SecuritySystem security;
    private final String         mode;
    private boolean              wasArmed;
    private String               previousMode;

    public ArmSecurityCommand(SecuritySystem security, String mode) {
        this.security = security;
        this.mode     = mode;
    }

    @Override
    public void execute() {
        wasArmed     = security.isArmed();
        previousMode = security.getMode();
        security.arm(mode);
    }

    @Override
    public void undo() {
        if (!wasArmed) security.disarm();
        else           security.arm(previousMode);
    }

    @Override public String getName() { return "ArmSecurity"; }
    @Override public String describe() { return "Arm security in " + mode + " mode"; }
}

// ─────────────────────────────────────────────────────────
// CONCRETE COMMANDS — Music
// ─────────────────────────────────────────────────────────

public class PlayMusicCommand implements HomeCommand {
    private final MusicSystem music;
    private final String      playlist;
    private final int         volume;
    private boolean           wasPlaying;
    private String            previousPlaylist;

    public PlayMusicCommand(MusicSystem music, String playlist, int volume) {
        this.music    = music;
        this.playlist = playlist;
        this.volume   = volume;
    }

    @Override
    public void execute() {
        wasPlaying       = music.isPlaying();
        previousPlaylist = music.getPlaylist();
        music.setVolume(volume);
        music.play(playlist);
    }

    @Override
    public void undo() {
        if (!wasPlaying) music.stop();
        else             music.play(previousPlaylist);
    }

    @Override public String getName() { return "PlayMusic"; }
    @Override public String describe() {
        return "Play \"" + playlist + "\" at volume " + volume;
    }
}

// ─────────────────────────────────────────────────────────
// MACRO COMMAND (Composite) — full implementation here too
// but placed together with the smart home demo for context
// ─────────────────────────────────────────────────────────

/**
 * MacroCommand groups multiple commands into a single atomic unit.
 * execute() runs all sub-commands in order.
 * undo()    runs all sub-commands in reverse order.
 *
 * This is the Composite pattern applied to commands — MacroCommand
 * implements HomeCommand and contains a List<HomeCommand>.
 *
 * If any sub-command's execute() throws, the macro stops there.
 * Partial undo rolls back the commands that did execute.
 */
public class MacroCommand implements HomeCommand {

    private final String           name;
    private final List<HomeCommand> commands;
    private int lastExecutedIndex = -1; // tracks partial execution for safe undo

    public MacroCommand(String name, List<HomeCommand> commands) {
        Objects.requireNonNull(name, "name must not be null");
        Objects.requireNonNull(commands, "commands must not be null");
        this.name     = name;
        this.commands = new ArrayList<>(commands); // defensive copy
    }

    /** Convenience factory for readable construction */
    public static MacroCommand of(String name, HomeCommand... commands) {
        return new MacroCommand(name, Arrays.asList(commands));
    }

    @Override
    public void execute() {
        lastExecutedIndex = -1;
        for (int i = 0; i < commands.size(); i++) {
            commands.get(i).execute();
            lastExecutedIndex = i;
        }
    }

    /**
     * Undo in reverse order up to lastExecutedIndex.
     * If execute() was partial (threw midway), only successfully
     * executed commands are undone.
     */
    @Override
    public void undo() {
        for (int i = lastExecutedIndex; i >= 0; i--) {
            commands.get(i).undo();
        }
    }

    @Override public String getName() { return name; }
    @Override public String describe() {
        StringBuilder sb = new StringBuilder("Macro \"").append(name).append("\" [");
        for (int i = 0; i < commands.size(); i++) {
            if (i > 0) sb.append(", ");
            sb.append(commands.get(i).getName());
        }
        sb.append("]");
        return sb.toString();
    }

    public int commandCount() { return commands.size(); }
}

// ─────────────────────────────────────────────────────────
// SCHEDULED COMMAND — wraps any command with a schedule
// ─────────────────────────────────────────────────────────

/**
 * Wraps a HomeCommand with scheduling metadata.
 * The SmartRemote's scheduler uses this to fire commands at the right time.
 *
 * Note: in production you would serialize ScheduledCommand to a job store
 * so scheduled commands survive application restarts.
 */
public class ScheduledCommand {
    private final HomeCommand command;
    private final LocalDateTime scheduledAt;
    private boolean executed = false;

    public ScheduledCommand(HomeCommand command, LocalDateTime scheduledAt) {
        this.command     = command;
        this.scheduledAt = scheduledAt;
    }

    public HomeCommand   getCommand()     { return command; }
    public LocalDateTime getScheduledAt() { return scheduledAt; }
    public boolean       isExecuted()     { return executed; }
    public void          markExecuted()   { executed = true; }

    @Override
    public String toString() {
        return "[" + scheduledAt.format(DateTimeFormatter.ofPattern("HH:mm")) + "] "
               + command.describe() + (executed ? " (DONE)" : "");
    }
}

// ─────────────────────────────────────────────────────────
// COMMAND LOGGER — Audit trail
// ─────────────────────────────────────────────────────────

public class CommandAuditLog {
    private static final Logger LOG = Logger.getLogger(CommandAuditLog.class.getName());

    public enum EventType { EXECUTED, UNDONE, SCHEDULED, FAILED }

    public record LogEntry(Instant timestamp, EventType event, String commandName,
                           String description, String actor) {}

    private final List<LogEntry> entries = new ArrayList<>();

    public void log(EventType event, HomeCommand command, String actor) {
        LogEntry entry = new LogEntry(
            Instant.now(), event, command.getName(), command.describe(), actor);
        entries.add(entry);
        LOG.info(() -> String.format("[%s] %s | %s | actor=%s",
            event, command.getName(), command.describe(), actor));
    }

    public List<LogEntry> getEntries() { return Collections.unmodifiableList(entries); }

    public List<LogEntry> getEntriesByType(EventType type) {
        return entries.stream().filter(e -> e.event() == type).toList();
    }

    public void printLog() {
        System.out.println("\n═══════════════ AUDIT LOG ═══════════════");
        DateTimeFormatter fmt = DateTimeFormatter.ofPattern("HH:mm:ss");
        for (LogEntry e : entries) {
            System.out.printf("[%s] %-10s | %-16s | %s%n",
                e.timestamp().toString().substring(11, 19),
                e.event(), e.commandName(), e.description());
        }
        System.out.println("═════════════════════════════════════════\n");
    }
}

// ─────────────────────────────────────────────────────────
// SMART REMOTE — the Invoker
// ─────────────────────────────────────────────────────────

/**
 * The remote control that manages commands, macros, and scheduling.
 *
 * Slots: like a physical remote, each button slot holds an on/off command pair.
 * History: supports undo of the last executed command.
 * Macros: named sequences stored by name.
 * Scheduler: basic example using ScheduledExecutorService.
 */
public class SmartRemote {

    private static final int SLOT_COUNT = 7;

    private final HomeCommand[]      onCommands;
    private final HomeCommand[]      offCommands;
    private final Deque<HomeCommand> history;
    private final Map<String, MacroCommand> macros;
    private final CommandAuditLog    auditLog;
    private final ScheduledExecutorService scheduler;
    private final String             userName;

    public SmartRemote(String userName) {
        this.userName    = userName;
        this.onCommands  = new HomeCommand[SLOT_COUNT];
        this.offCommands = new HomeCommand[SLOT_COUNT];
        this.history     = new ArrayDeque<>();
        this.macros      = new LinkedHashMap<>();
        this.auditLog    = new CommandAuditLog();
        this.scheduler   = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "smart-home-scheduler");
            t.setDaemon(true);
            return t;
        });

        // Initialize slots with no-op commands to avoid null checks
        HomeCommand noOp = new NoOpCommand();
        Arrays.fill(onCommands,  noOp);
        Arrays.fill(offCommands, noOp);
    }

    public void setCommand(int slot, HomeCommand onCmd, HomeCommand offCmd) {
        validateSlot(slot);
        onCommands[slot]  = Objects.requireNonNull(onCmd,  "onCmd must not be null");
        offCommands[slot] = Objects.requireNonNull(offCmd, "offCmd must not be null");
    }

    public void pressOn(int slot) {
        validateSlot(slot);
        HomeCommand cmd = onCommands[slot];
        cmd.execute();
        history.push(cmd);
        auditLog.log(CommandAuditLog.EventType.EXECUTED, cmd, userName);
    }

    public void pressOff(int slot) {
        validateSlot(slot);
        HomeCommand cmd = offCommands[slot];
        cmd.execute();
        history.push(cmd);
        auditLog.log(CommandAuditLog.EventType.EXECUTED, cmd, userName);
    }

    public void undoLast() {
        if (history.isEmpty()) {
            System.out.println("Nothing to undo");
            return;
        }
        HomeCommand cmd = history.pop();
        cmd.undo();
        auditLog.log(CommandAuditLog.EventType.UNDONE, cmd, userName);
    }

    public void registerMacro(String name, MacroCommand macro) {
        macros.put(name, macro);
        System.out.println("Macro registered: " + macro.describe());
    }

    public void executeMacro(String name) {
        MacroCommand macro = macros.get(name);
        if (macro == null) throw new IllegalArgumentException("Unknown macro: " + name);
        macro.execute();
        history.push(macro);
        auditLog.log(CommandAuditLog.EventType.EXECUTED, macro, userName);
    }

    public void undoMacro() {
        // The macro itself is on the history stack; undo() reverses all its sub-commands
        undoLast();
    }

    /**
     * Schedule a command to run after a delay.
     * In a real system the ScheduledCommand would be persisted to survive restarts.
     */
    public void scheduleCommand(HomeCommand command, long delaySeconds) {
        System.out.println("Scheduling: " + command.describe() + " in " + delaySeconds + "s");
        auditLog.log(CommandAuditLog.EventType.SCHEDULED, command, userName);
        scheduler.schedule(() -> {
            command.execute();
            history.push(command);
            auditLog.log(CommandAuditLog.EventType.EXECUTED, command, userName);
        }, delaySeconds, TimeUnit.SECONDS);
    }

    public void shutdown() { scheduler.shutdown(); }
    public CommandAuditLog getAuditLog() { return auditLog; }

    private void validateSlot(int slot) {
        if (slot < 0 || slot >= SLOT_COUNT) {
            throw new IllegalArgumentException("Invalid slot: " + slot + ". Valid: 0-" + (SLOT_COUNT - 1));
        }
    }

    public void printRemoteState() {
        System.out.println("\n┌─────────── SMART REMOTE ───────────┐");
        for (int i = 0; i < SLOT_COUNT; i++) {
            System.out.printf("│ Slot %d  ON: %-14s OFF: %-14s│%n",
                i, onCommands[i].getName(), offCommands[i].getName());
        }
        System.out.println("│ Macros: " + macros.keySet() + "          │");
        System.out.println("└────────────────────────────────────┘\n");
    }
}

// ─────────────────────────────────────────────────────────
// NULL OBJECT — NoOpCommand
// ─────────────────────────────────────────────────────────

/**
 * Null Object pattern applied to commands.
 * Eliminates null checks throughout the Invoker.
 */
public class NoOpCommand implements HomeCommand {
    @Override public void execute() { /* intentionally empty */ }
    @Override public void undo()    { /* intentionally empty */ }
    @Override public String getName()   { return "NoOp"; }
    @Override public String describe()  { return "No operation"; }
}

// ─────────────────────────────────────────────────────────
// DEMO
// ─────────────────────────────────────────────────────────

public class SmartHomeDemo {
    public static void main(String[] args) throws InterruptedException {

        // Receivers
        Light         livingRoomLight = new Light("Living Room");
        Light         bedroomLight    = new Light("Bedroom");
        Light         kitchenLight    = new Light("Kitchen");
        Thermostat    thermostat      = new Thermostat();
        SecuritySystem security       = new SecuritySystem();
        MusicSystem    music          = new MusicSystem();

        // Remote (Invoker)
        SmartRemote remote = new SmartRemote("Alice");

        // Wire up slots
        remote.setCommand(0, new LightOnCommand(livingRoomLight),  new LightOffCommand(livingRoomLight));
        remote.setCommand(1, new LightOnCommand(bedroomLight),     new LightOffCommand(bedroomLight));
        remote.setCommand(2, new LightOnCommand(kitchenLight),     new LightOffCommand(kitchenLight));
        remote.setCommand(3, new SetTemperatureCommand(thermostat, 24), new SetTemperatureCommand(thermostat, 20));
        remote.setCommand(4, new ArmSecurityCommand(security, "AWAY"), new ArmSecurityCommand(security, "HOME"));

        remote.printRemoteState();

        // Individual commands
        System.out.println("=== Individual Commands ===");
        remote.pressOn(0);   // Living room light on
        remote.pressOn(3);   // Thermostat to 24
        remote.pressOff(0);  // Living room light off
        remote.undoLast();   // Undo: living room light back on

        // ── Macro: "Good Night" ──
        System.out.println("\n=== Good Night Macro ===");
        MacroCommand goodNight = MacroCommand.of("Good Night",
            new LightOffCommand(livingRoomLight),
            new LightOffCommand(kitchenLight),
            new SetTemperatureCommand(thermostat, 19),
            new ArmSecurityCommand(security, "NIGHT"),
            new DimLightCommand(bedroomLight, 20),
            new PlayMusicCommand(music, "Sleep Sounds", 15)
        );
        remote.registerMacro("goodnight", goodNight);
        remote.executeMacro("goodnight");

        System.out.println("\n=== Undo Good Night Macro ===");
        remote.undoMacro(); // Reverses all 6 sub-commands in reverse order

        // ── Macro: "Good Morning" ──
        System.out.println("\n=== Good Morning Macro ===");
        MacroCommand goodMorning = MacroCommand.of("Good Morning",
            new LightOnCommand(livingRoomLight),
            new LightOnCommand(kitchenLight),
            new SetTemperatureCommand(thermostat, 22),
            new ArmSecurityCommand(security, "HOME"),
            new PlayMusicCommand(music, "Morning Playlist", 40)
        );
        remote.registerMacro("goodmorning", goodMorning);
        remote.executeMacro("goodmorning");

        // ── Schedule a future command ──
        System.out.println("\n=== Scheduling bedroom light off in 2 seconds ===");
        remote.scheduleCommand(new LightOffCommand(bedroomLight), 2);
        Thread.sleep(3000); // Wait for scheduled command to fire

        // ── Audit log ──
        remote.getAuditLog().printLog();

        remote.shutdown();
    }
}
```

---

## 6. Command Queue and Audit Logging

Command queuing and audit logging are two of the most underappreciated capabilities of the Command pattern. The queue treats commands as data: a producer creates them, a queue holds them, a consumer processes them. The audit log makes every mutation observable and replayable.

```java
package patterns.behavioral.command.queue;

import java.util.*;
import java.util.concurrent.*;
import java.time.Instant;
import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Consumer;

// ─────────────────────────────────────────────────────────
// QUEUEABLE COMMAND
// ─────────────────────────────────────────────────────────

public interface QueueableCommand {
    void execute() throws Exception;
    String getName();
    String describe();
    default int getPriority() { return 5; } // 1=highest, 10=lowest
    default boolean isRetryable() { return true; }
    default int maxRetries() { return 3; }
}

// ─────────────────────────────────────────────────────────
// COMMAND ENVELOPE — wraps a command with queue metadata
// ─────────────────────────────────────────────────────────

/**
 * Wraps a command with metadata needed for queue management,
 * retry logic, and audit logging.
 */
public class CommandEnvelope implements Comparable<CommandEnvelope> {

    private static final AtomicLong SEQ = new AtomicLong(0);

    public enum Status { PENDING, RUNNING, COMPLETED, FAILED, DEAD_LETTERED }

    private final long              sequenceId;
    private final QueueableCommand  command;
    private final Instant           enqueueTime;
    private final String            submittedBy;
    private final String            correlationId;

    private Status  status       = Status.PENDING;
    private int     attemptCount = 0;
    private Instant startTime;
    private Instant endTime;
    private String  failureReason;

    public CommandEnvelope(QueueableCommand command, String submittedBy, String correlationId) {
        this.sequenceId    = SEQ.incrementAndGet();
        this.command       = Objects.requireNonNull(command);
        this.submittedBy   = submittedBy;
        this.correlationId = correlationId;
        this.enqueueTime   = Instant.now();
    }

    // ── Lifecycle transitions ──

    public void markRunning()   {
        status    = Status.RUNNING;
        startTime = Instant.now();
        attemptCount++;
    }

    public void markCompleted() {
        status  = Status.COMPLETED;
        endTime = Instant.now();
    }

    public void markFailed(String reason) {
        this.failureReason = reason;
        if (command.isRetryable() && attemptCount < command.maxRetries()) {
            status = Status.PENDING; // Will be retried
        } else {
            status  = Status.DEAD_LETTERED;
            endTime = Instant.now();
        }
    }

    // ── Accessors ──

    public long             getSequenceId()    { return sequenceId; }
    public QueueableCommand getCommand()       { return command; }
    public Instant          getEnqueueTime()   { return enqueueTime; }
    public String           getSubmittedBy()   { return submittedBy; }
    public String           getCorrelationId() { return correlationId; }
    public Status           getStatus()        { return status; }
    public int              getAttemptCount()  { return attemptCount; }
    public String           getFailureReason() { return failureReason; }

    public long durationMillis() {
        if (startTime == null || endTime == null) return -1;
        return endTime.toEpochMilli() - startTime.toEpochMilli();
    }

    /** For PriorityQueue: lower priority number = processed first */
    @Override
    public int compareTo(CommandEnvelope other) {
        int p = Integer.compare(this.command.getPriority(), other.command.getPriority());
        return p != 0 ? p : Long.compare(this.sequenceId, other.sequenceId);
    }

    @Override
    public String toString() {
        return String.format("Envelope{seq=%d, cmd=%s, status=%s, attempts=%d, by=%s}",
            sequenceId, command.getName(), status, attemptCount, submittedBy);
    }
}

// ─────────────────────────────────────────────────────────
// AUDIT STORE
// ─────────────────────────────────────────────────────────

/**
 * Append-only store for all command lifecycle events.
 * In production this would write to a database, Kafka topic, or S3.
 *
 * Capabilities this enables:
 * 1. Compliance audit: who did what and when
 * 2. Failure analysis: which commands fail, how often, and why
 * 3. Replay: re-execute all COMPLETED commands against a blank system
 *    to reconstruct state — the foundation of event sourcing
 */
public class CommandAuditStore {

    public enum AuditEvent {
        ENQUEUED, STARTED, COMPLETED, FAILED, DEAD_LETTERED, RETRYING
    }

    public record AuditRecord(
        long            sequenceId,
        String          correlationId,
        String          commandName,
        String          description,
        AuditEvent      event,
        String          actor,
        Instant         timestamp,
        long            durationMs,
        String          failureReason
    ) {
        @Override
        public String toString() {
            return String.format("[%s] seq=%-4d %-15s %-12s %s%s",
                timestamp.toString().substring(11, 23),
                sequenceId, commandName, event, description,
                failureReason != null ? " | ERR: " + failureReason : "");
        }
    }

    private final List<AuditRecord> records = new CopyOnWriteArrayList<>();
    private final List<Consumer<AuditRecord>> listeners = new ArrayList<>();

    public void record(CommandEnvelope envelope, AuditEvent event) {
        AuditRecord rec = new AuditRecord(
            envelope.getSequenceId(),
            envelope.getCorrelationId(),
            envelope.getCommand().getName(),
            envelope.getCommand().describe(),
            event,
            envelope.getSubmittedBy(),
            Instant.now(),
            envelope.durationMillis(),
            envelope.getFailureReason()
        );
        records.add(rec);
        listeners.forEach(l -> l.accept(rec));
    }

    public void addListener(Consumer<AuditRecord> listener) {
        listeners.add(listener);
    }

    public List<AuditRecord> getAll() { return Collections.unmodifiableList(records); }

    public List<AuditRecord> getByStatus(AuditEvent event) {
        return records.stream().filter(r -> r.event() == event).toList();
    }

    /** Replay: returns all completed commands in sequence order for event sourcing */
    public List<AuditRecord> getCompletedInOrder() {
        return records.stream()
            .filter(r -> r.event() == AuditEvent.COMPLETED)
            .sorted(Comparator.comparingLong(AuditRecord::sequenceId))
            .toList();
    }

    public void printAll() {
        System.out.println("\n════════════════════ AUDIT STORE ════════════════════");
        records.forEach(System.out::println);
        System.out.println("═════════════════════════════════════════════════════\n");
    }
}

// ─────────────────────────────────────────────────────────
// PRIORITY COMMAND QUEUE + WORKER
// ─────────────────────────────────────────────────────────

/**
 * A priority-based command queue backed by a thread pool.
 *
 * Commands with lower priority numbers are processed first.
 * Equal-priority commands are processed in FIFO sequence order.
 * Failed retryable commands are re-enqueued with exponential backoff.
 *
 * Dead-lettered commands (exhausted retries) are preserved for analysis.
 */
public class PriorityCommandQueue {

    private final PriorityBlockingQueue<CommandEnvelope> queue;
    private final CommandAuditStore                      auditStore;
    private final List<CommandEnvelope>                  deadLetterQueue;
    private final ExecutorService                        workerPool;
    private volatile boolean                             running;

    public PriorityCommandQueue(int workerThreads, CommandAuditStore auditStore) {
        this.queue          = new PriorityBlockingQueue<>();
        this.auditStore     = auditStore;
        this.deadLetterQueue = new CopyOnWriteArrayList<>();
        this.workerPool     = Executors.newFixedThreadPool(workerThreads, r -> {
            Thread t = new Thread(r, "cmd-worker-" + System.nanoTime());
            t.setDaemon(true);
            return t;
        });
        this.running = true;
        startWorkers(workerThreads);
    }

    public CommandEnvelope submit(QueueableCommand command, String actor, String correlationId) {
        CommandEnvelope envelope = new CommandEnvelope(command, actor, correlationId);
        queue.offer(envelope);
        auditStore.record(envelope, CommandAuditStore.AuditEvent.ENQUEUED);
        System.out.printf("ENQUEUED [seq=%d, priority=%d]: %s%n",
            envelope.getSequenceId(), command.getPriority(), command.describe());
        return envelope;
    }

    public List<CommandEnvelope> getDeadLetterQueue() {
        return Collections.unmodifiableList(deadLetterQueue);
    }

    public void shutdown() throws InterruptedException {
        running = false;
        workerPool.shutdown();
        workerPool.awaitTermination(5, TimeUnit.SECONDS);
    }

    private void startWorkers(int count) {
        for (int i = 0; i < count; i++) {
            workerPool.submit(this::workerLoop);
        }
    }

    private void workerLoop() {
        while (running || !queue.isEmpty()) {
            try {
                CommandEnvelope envelope = queue.poll(100, TimeUnit.MILLISECONDS);
                if (envelope == null) continue;
                processEnvelope(envelope);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    private void processEnvelope(CommandEnvelope envelope) {
        envelope.markRunning();
        auditStore.record(envelope, CommandAuditStore.AuditEvent.STARTED);

        try {
            envelope.getCommand().execute();
            envelope.markCompleted();
            auditStore.record(envelope, CommandAuditStore.AuditEvent.COMPLETED);
            System.out.printf("COMPLETED [seq=%d] in %dms: %s%n",
                envelope.getSequenceId(), envelope.durationMillis(),
                envelope.getCommand().getName());

        } catch (Exception ex) {
            envelope.markFailed(ex.getMessage());

            if (envelope.getStatus() == CommandEnvelope.Status.PENDING) {
                // Retry with backoff
                long backoffMs = (long) Math.pow(2, envelope.getAttemptCount()) * 100L;
                System.out.printf("RETRY [seq=%d] attempt=%d in %dms: %s%n",
                    envelope.getSequenceId(), envelope.getAttemptCount(),
                    backoffMs, ex.getMessage());
                auditStore.record(envelope, CommandAuditStore.AuditEvent.RETRYING);
                scheduleRequeue(envelope, backoffMs);
            } else {
                // Dead letter
                deadLetterQueue.add(envelope);
                auditStore.record(envelope, CommandAuditStore.AuditEvent.DEAD_LETTERED);
                System.err.printf("DEAD_LETTER [seq=%d]: %s | %s%n",
                    envelope.getSequenceId(),
                    envelope.getCommand().getName(), ex.getMessage());
            }
        }
    }

    private void scheduleRequeue(CommandEnvelope envelope, long delayMs) {
        CompletableFuture.delayedExecutor(delayMs, TimeUnit.MILLISECONDS)
            .execute(() -> queue.offer(envelope));
    }
}

// ─────────────────────────────────────────────────────────
// SAMPLE CONCRETE QUEUEABLE COMMANDS
// ─────────────────────────────────────────────────────────

/** Simulates an email notification command */
class SendEmailCommand implements QueueableCommand {
    private final String recipient;
    private final String subject;

    public SendEmailCommand(String recipient, String subject) {
        this.recipient = recipient;
        this.subject   = subject;
    }

    @Override
    public void execute() throws Exception {
        System.out.println("  → Sending email to " + recipient + ": " + subject);
        Thread.sleep(50); // simulate I/O
    }

    @Override public String getName()    { return "SendEmail"; }
    @Override public String describe()   { return "Email to " + recipient + ": " + subject; }
    @Override public int    getPriority() { return 3; }
    @Override public boolean isRetryable() { return true; }
    @Override public int    maxRetries()  { return 2; }
}

/** Simulates a payment processing command */
class ProcessPaymentCommand implements QueueableCommand {
    private final String orderId;
    private final double amount;

    public ProcessPaymentCommand(String orderId, double amount) {
        this.orderId = orderId;
        this.amount  = amount;
    }

    @Override
    public void execute() throws Exception {
        System.out.printf("  → Processing payment for order %s: $%.2f%n", orderId, amount);
        Thread.sleep(100); // simulate payment gateway call
    }

    @Override public String getName()    { return "ProcessPayment"; }
    @Override public String describe()   { return "Payment $" + amount + " for order " + orderId; }
    @Override public int    getPriority() { return 1; } // High priority
    @Override public boolean isRetryable() { return true; }
    @Override public int    maxRetries()  { return 3; }
}

/** Simulates a report generation command */
class GenerateReportCommand implements QueueableCommand {
    private final String reportType;

    public GenerateReportCommand(String reportType) { this.reportType = reportType; }

    @Override
    public void execute() throws Exception {
        System.out.println("  → Generating " + reportType + " report...");
        Thread.sleep(200); // simulate heavy computation
    }

    @Override public String getName()    { return "GenerateReport"; }
    @Override public String describe()   { return "Generate " + reportType + " report"; }
    @Override public int    getPriority() { return 8; } // Low priority (background)
    @Override public boolean isRetryable() { return false; }
}

// ─────────────────────────────────────────────────────────
// QUEUE DEMO
// ─────────────────────────────────────────────────────────

public class CommandQueueDemo {
    public static void main(String[] args) throws InterruptedException {

        CommandAuditStore auditStore = new CommandAuditStore();

        // Print to console whenever a command completes or dead-letters
        auditStore.addListener(rec -> {
            if (rec.event() == CommandAuditStore.AuditEvent.DEAD_LETTERED) {
                System.err.println("ALERT: Dead-lettered command: " + rec);
            }
        });

        PriorityCommandQueue commandQueue = new PriorityCommandQueue(3, auditStore);

        // Submit commands with different priorities
        // Payment (priority 1) should be processed before reports (priority 8)
        commandQueue.submit(new GenerateReportCommand("Monthly"),   "system",    "corr-001");
        commandQueue.submit(new ProcessPaymentCommand("ORD-99", 249.99), "api",  "corr-002");
        commandQueue.submit(new SendEmailCommand("bob@example.com", "Your order"), "api", "corr-003");
        commandQueue.submit(new ProcessPaymentCommand("ORD-100", 19.99), "api",  "corr-004");
        commandQueue.submit(new GenerateReportCommand("Weekly"),    "scheduler", "corr-005");
        commandQueue.submit(new SendEmailCommand("alice@example.com", "Receipt"), "api", "corr-006");

        Thread.sleep(2000); // Let the queue drain

        commandQueue.shutdown();
        auditStore.printAll();

        // Replay analysis: show completed commands in sequence order
        System.out.println("Completed commands for replay:");
        auditStore.getCompletedInOrder()
            .forEach(r -> System.out.println("  " + r.sequenceId() + ": " + r.description()));
    }
}
```

---

## 7. Macro Commands (Composite Commands)

The `MacroCommand` is the Command pattern's intersection with the Composite pattern. It is one of the most practically useful patterns in production systems because it enables:

1. **User-defined macros**: record a sequence of operations, save it, replay it
2. **Batch operations**: execute many commands as a single logical unit
3. **Transactional groups**: undo an entire batch by undoing the macro

The full implementation appears in [Section 5](#5-implementation-2--smart-home-remote-with-macro-commands). Below is a standalone, generalized version demonstrating nested macros.

```java
package patterns.behavioral.command.macro;

import java.util.*;

// ─────────────────────────────────────────────────────────
// COMMAND INTERFACE
// ─────────────────────────────────────────────────────────

public interface Command {
    void execute();
    void undo();
    String describe();
}

// ─────────────────────────────────────────────────────────
// MACRO COMMAND — supports nesting (macros within macros)
// ─────────────────────────────────────────────────────────

/**
 * MacroCommand: a command that executes a sequence of other commands.
 *
 * Key design decisions:
 * 1. Defensive copy of the command list at construction — prevents
 *    external mutation after registration.
 * 2. lastExecutedIndex tracks partial execution for safe partial undo
 *    when a sub-command throws.
 * 3. MacroCommand itself implements Command, so macros can be nested
 *    arbitrarily deeply (macro within macro within macro).
 * 4. isEmpty() guard prevents executing an empty macro silently.
 */
public class MacroCommand implements Command {

    private final String        label;
    private final List<Command> commands;
    private int lastExecutedIndex = -1;

    public MacroCommand(String label, List<Command> commands) {
        if (label == null || label.isBlank()) throw new IllegalArgumentException("label required");
        this.label    = label;
        this.commands = new ArrayList<>(commands);
    }

    public static MacroCommand of(String label, Command... commands) {
        return new MacroCommand(label, Arrays.asList(commands));
    }

    @Override
    public void execute() {
        if (commands.isEmpty()) {
            System.out.println("[Macro] " + label + " is empty — nothing to execute");
            return;
        }
        lastExecutedIndex = -1;
        System.out.println("[Macro START] " + label);
        for (int i = 0; i < commands.size(); i++) {
            commands.get(i).execute();
            lastExecutedIndex = i;
        }
        System.out.println("[Macro END] " + label);
    }

    @Override
    public void undo() {
        if (lastExecutedIndex < 0) return; // Nothing was executed
        System.out.println("[Macro UNDO] " + label);
        for (int i = lastExecutedIndex; i >= 0; i--) {
            commands.get(i).undo();
        }
        lastExecutedIndex = -1;
    }

    @Override
    public String describe() {
        StringBuilder sb = new StringBuilder("Macro[" + label + "](");
        for (int i = 0; i < commands.size(); i++) {
            if (i > 0) sb.append(" → ");
            sb.append(commands.get(i).describe());
        }
        return sb.append(")").toString();
    }

    /** Add a command to the macro (before it has been executed) */
    public MacroCommand append(Command cmd) {
        commands.add(cmd);
        return this; // fluent
    }

    public int size() { return commands.size(); }
}

// ─────────────────────────────────────────────────────────
// MACRO RECORDER — record user actions and produce a MacroCommand
// ─────────────────────────────────────────────────────────

/**
 * Records commands as the user executes them, then produces a
 * replayable MacroCommand. Follows the same principle as macro
 * recording in Excel or Photoshop.
 */
public class MacroRecorder {

    private final List<Command> recorded = new ArrayList<>();
    private boolean recording = false;
    private String  macroName;

    public void startRecording(String name) {
        if (recording) throw new IllegalStateException("Already recording");
        this.macroName = name;
        this.recorded.clear();
        this.recording = true;
        System.out.println("[Recorder] Started recording macro: " + name);
    }

    /**
     * Call this instead of command.execute() while recording.
     * The command is both executed AND recorded.
     */
    public void record(Command command) {
        if (!recording) {
            command.execute(); // Not recording, just execute
            return;
        }
        command.execute();
        recorded.add(command);
        System.out.println("[Recorder] Recorded: " + command.describe());
    }

    public MacroCommand stopRecording() {
        if (!recording) throw new IllegalStateException("Not currently recording");
        recording = false;
        MacroCommand macro = new MacroCommand(macroName, recorded);
        System.out.println("[Recorder] Macro '" + macroName + "' saved with " +
                           recorded.size() + " steps");
        return macro;
    }

    public boolean isRecording() { return recording; }
}

// ─────────────────────────────────────────────────────────
// DEMO: Nested macros
// ─────────────────────────────────────────────────────────

// Trivial leaf commands for demonstration
class PrintCommand implements Command {
    private final String message;
    public PrintCommand(String message) { this.message = message; }
    @Override public void execute() { System.out.println("  DO:   " + message); }
    @Override public void undo()    { System.out.println("  UNDO: " + message); }
    @Override public String describe() { return "Print(\"" + message + "\")"; }
}

public class MacroDemo {
    public static void main(String[] args) {
        // Nested macro: "deploy" macro contains "build" macro + "restart" macro
        MacroCommand buildMacro = MacroCommand.of("Build",
            new PrintCommand("Run tests"),
            new PrintCommand("Compile code"),
            new PrintCommand("Package artifact")
        );

        MacroCommand restartMacro = MacroCommand.of("Restart",
            new PrintCommand("Stop service"),
            new PrintCommand("Copy artifact"),
            new PrintCommand("Start service"),
            new PrintCommand("Health check")
        );

        // Macro containing other macros — Composite pattern
        MacroCommand deployMacro = MacroCommand.of("Full Deploy",
            buildMacro,
            restartMacro,
            new PrintCommand("Update load balancer")
        );

        System.out.println(deployMacro.describe());
        System.out.println();

        deployMacro.execute();
        System.out.println();

        deployMacro.undo();
        System.out.println();

        // Macro recorder demo
        MacroRecorder recorder = new MacroRecorder();
        recorder.startRecording("Quick Setup");
        recorder.record(new PrintCommand("Init database"));
        recorder.record(new PrintCommand("Seed test data"));
        recorder.record(new PrintCommand("Start server"));
        MacroCommand quickSetup = recorder.stopRecording();

        System.out.println("\nReplaying recorded macro:");
        quickSetup.execute();
    }
}
```

---

## 8. Thread Pool / Java Executor as Command Pattern

The Java `Executor` framework is the most widely used instance of the Command pattern in the entire Java ecosystem, yet most engineers never notice the connection.

```java
package patterns.behavioral.command.executor;

import java.util.concurrent.*;
import java.util.*;
import java.util.concurrent.atomic.*;
import java.util.function.Consumer;

/**
 * Runnable IS the Command interface in Java.
 *
 *   Command.execute()  ≡  Runnable.run()
 *   ConcreteCommand    ≡  Your Runnable/Callable implementation
 *   Invoker            ≡  ExecutorService
 *   Client             ≡  The code that submits tasks
 *
 * The Executor framework was specifically designed on the Command pattern:
 *   - Decouples task submission from task execution
 *   - Tasks (Runnables) are objects — they can be queued, prioritized,
 *     cancelled, and logged
 *   - ExecutorService is the Invoker — it decides when, where, and
 *     how many commands run concurrently
 *
 * This section demonstrates:
 *   1. The direct Runnable-as-Command equivalence
 *   2. A custom Executor that adds command logging and metrics
 *   3. Future/CompletableFuture as deferred command result
 *   4. Priority queuing of command-tasks
 */

// ─────────────────────────────────────────────────────────
// 1. THE EQUIVALENCE — Runnable IS Command
// ─────────────────────────────────────────────────────────

/**
 * Standard GoF Command interface:
 *   interface Command { void execute(); }
 *
 * java.lang.Runnable:
 *   interface Runnable { void run(); }     ← identical semantics
 *
 * java.util.concurrent.Callable<V>:
 *   interface Callable<V> { V call(); }    ← Command with return value
 *
 * An Executor IS a Command Invoker:
 *   executor.execute(runnable)             ← invoker.invoke(command)
 *
 * The Executor DOES NOT know what Runnable does. The Runnable DOES NOT
 * know which thread runs it or when. This is textbook Command decoupling.
 */

// ─────────────────────────────────────────────────────────
// 2. INSTRUMENTED EXECUTOR — adds logging and metrics
// ─────────────────────────────────────────────────────────

/**
 * Wraps any ExecutorService to add transparent command logging and metrics.
 * Demonstrates how the Command pattern enables cross-cutting concerns
 * without modifying the command objects themselves.
 */
public class InstrumentedExecutor implements ExecutorService {

    private final ExecutorService          delegate;
    private final AtomicLong               submitted   = new AtomicLong();
    private final AtomicLong               completed   = new AtomicLong();
    private final AtomicLong               failed      = new AtomicLong();
    private final AtomicLong               totalNanos  = new AtomicLong();
    private final List<Consumer<TaskEvent>> listeners  = new CopyOnWriteArrayList<>();

    public record TaskEvent(String type, String taskName, long durationNanos, Throwable error) {}

    public InstrumentedExecutor(ExecutorService delegate) {
        this.delegate = Objects.requireNonNull(delegate);
    }

    public void addListener(Consumer<TaskEvent> listener) { listeners.add(listener); }

    @Override
    public void execute(Runnable command) {
        submitted.incrementAndGet();
        String taskName = command.getClass().getSimpleName();

        // Wrap the command to add before/after instrumentation
        delegate.execute(() -> {
            long start = System.nanoTime();
            notifyListeners(new TaskEvent("START", taskName, 0, null));
            try {
                command.run();
                long dur = System.nanoTime() - start;
                completed.incrementAndGet();
                totalNanos.addAndGet(dur);
                notifyListeners(new TaskEvent("COMPLETE", taskName, dur, null));
            } catch (Exception ex) {
                long dur = System.nanoTime() - start;
                failed.incrementAndGet();
                notifyListeners(new TaskEvent("FAIL", taskName, dur, ex));
                throw ex;
            }
        });
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        submitted.incrementAndGet();
        String taskName = task.getClass().getSimpleName();
        return delegate.submit(() -> {
            long start = System.nanoTime();
            try {
                T result = task.call();
                completed.incrementAndGet();
                totalNanos.addAndGet(System.nanoTime() - start);
                return result;
            } catch (Exception ex) {
                failed.incrementAndGet();
                throw ex;
            }
        });
    }

    public void printMetrics() {
        long sub  = submitted.get();
        long comp = completed.get();
        long fail = failed.get();
        double avgMs = comp > 0 ? (totalNanos.get() / comp) / 1_000_000.0 : 0;
        System.out.printf("ExecutorMetrics: submitted=%d, completed=%d, failed=%d, avg=%.2fms%n",
            sub, comp, fail, avgMs);
    }

    private void notifyListeners(TaskEvent event) {
        listeners.forEach(l -> l.accept(event));
    }

    // ── Delegate all other ExecutorService methods ──
    @Override public void shutdown() { delegate.shutdown(); }
    @Override public List<Runnable> shutdownNow() { return delegate.shutdownNow(); }
    @Override public boolean isShutdown() { return delegate.isShutdown(); }
    @Override public boolean isTerminated() { return delegate.isTerminated(); }
    @Override public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
        return delegate.awaitTermination(timeout, unit);
    }
    @Override public <T> Future<T> submit(Runnable task, T result) { return delegate.submit(task, result); }
    @Override public Future<?> submit(Runnable task) { return delegate.submit(task); }
    @Override public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException {
        return delegate.invokeAll(tasks);
    }
    @Override public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException {
        return delegate.invokeAll(tasks, timeout, unit);
    }
    @Override public <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException {
        return delegate.invokeAny(tasks);
    }
    @Override public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
        return delegate.invokeAny(tasks, timeout, unit);
    }
}

// ─────────────────────────────────────────────────────────
// 3. PRIORITY RUNNABLE — command with priority metadata
// ─────────────────────────────────────────────────────────

/**
 * A Runnable that carries a priority value.
 * When submitted to a PriorityBlockingQueue-backed executor,
 * higher-priority tasks are processed before lower-priority ones.
 */
public abstract class PriorityRunnable implements Runnable, Comparable<PriorityRunnable> {
    private final int priority; // 0 = highest

    protected PriorityRunnable(int priority) { this.priority = priority; }

    public int getPriority() { return priority; }

    @Override
    public int compareTo(PriorityRunnable other) {
        return Integer.compare(this.priority, other.priority);
    }
}

/**
 * Creates a thread pool that processes tasks in priority order.
 * This is the Invoker selecting which Command to run next based
 * on metadata carried on the command object itself.
 */
public class PriorityExecutorService {

    private final ThreadPoolExecutor executor;

    public PriorityExecutorService(int coreThreads, int maxThreads) {
        this.executor = new ThreadPoolExecutor(
            coreThreads, maxThreads,
            60L, TimeUnit.SECONDS,
            new PriorityBlockingQueue<>(), // Commands queued by priority
            Executors.defaultThreadFactory()
        );
    }

    public void submit(PriorityRunnable task) {
        executor.execute(task);
    }

    public void shutdown() throws InterruptedException {
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
    }
}

// ─────────────────────────────────────────────────────────
// 4. COMPLETABLEFUTURE AS DEFERRED COMMAND
// ─────────────────────────────────────────────────────────

/**
 * CompletableFuture is a deferred Command result.
 * The computation is encapsulated as a Supplier (analogous to a command),
 * the CompletableFuture is the handle to the eventual result,
 * and chained .thenApply()/.thenAccept() calls are further commands
 * that execute when the previous stage completes.
 *
 * This is the Command pattern in an async/reactive style:
 *   - Commands are composed, not called
 *   - Execution happens asynchronously in the thread pool
 *   - The pipeline is declarative
 */
public class FutureCommandPipeline {

    private final ExecutorService pool = Executors.newFixedThreadPool(4);

    public CompletableFuture<String> buildPipeline(String input) {
        // Each stage is a command (Supplier/Function) encapsulated as a lambda
        return CompletableFuture
            .supplyAsync(() -> {
                System.out.println("Stage 1: Fetch data for " + input);
                return "raw:" + input;
            }, pool)
            .thenApplyAsync(data -> {
                System.out.println("Stage 2: Transform " + data);
                return data.toUpperCase();
            }, pool)
            .thenApplyAsync(data -> {
                System.out.println("Stage 3: Enrich " + data);
                return data + "_ENRICHED";
            }, pool)
            .thenApplyAsync(data -> {
                System.out.println("Stage 4: Persist " + data);
                return "RESULT:" + data;
            }, pool)
            .exceptionally(ex -> {
                System.err.println("Pipeline failed: " + ex.getMessage());
                return "ERROR";
            });
    }

    public void shutdown() { pool.shutdown(); }
}

// ─────────────────────────────────────────────────────────
// DEMO
// ─────────────────────────────────────────────────────────

public class ExecutorCommandDemo {
    public static void main(String[] args) throws Exception {

        // 1. Instrumented executor
        System.out.println("=== Instrumented Executor ===");
        InstrumentedExecutor exec = new InstrumentedExecutor(Executors.newFixedThreadPool(4));
        exec.addListener(e -> {
            if ("COMPLETE".equals(e.type())) {
                System.out.printf("  Task '%s' completed in %.2fms%n",
                    e.taskName(), e.durationNanos() / 1_000_000.0);
            }
        });

        // These Runnables ARE Commands — they encapsulate work as objects
        exec.execute(() -> { Thread.sleep(50); return; });  // EmailNotifyTask
        exec.execute(() -> { Thread.sleep(80); return; });  // ReportGeneratorTask
        exec.execute(() -> { Thread.sleep(20); return; });  // CacheWarmTask

        Thread.sleep(500);
        exec.printMetrics();

        // 2. Priority executor
        System.out.println("\n=== Priority Executor ===");
        PriorityExecutorService priorityExec = new PriorityExecutorService(2, 4);

        // Submit in "wrong" order — priority determines actual processing order
        priorityExec.submit(new PriorityRunnable(9) { // Low priority
            @Override public void run() { System.out.println("LOW  priority task ran"); }
        });
        priorityExec.submit(new PriorityRunnable(1) { // High priority
            @Override public void run() { System.out.println("HIGH priority task ran"); }
        });
        priorityExec.submit(new PriorityRunnable(5) { // Medium priority
            @Override public void run() { System.out.println("MED  priority task ran"); }
        });

        priorityExec.shutdown();

        // 3. CompletableFuture pipeline
        System.out.println("\n=== CompletableFuture Pipeline ===");
        FutureCommandPipeline pipeline = new FutureCommandPipeline();
        String result = pipeline.buildPipeline("order-42").get(5, TimeUnit.SECONDS);
        System.out.println("Pipeline result: " + result);
        pipeline.shutdown();

        exec.shutdown();
    }
}
```

---

## 9. Real-World Framework Examples

### java.lang.Runnable and java.util.concurrent.Callable

As demonstrated above, `Runnable` is the Command pattern's canonical instance in Java. Every `Thread`, `ExecutorService`, `Timer`, and `ForkJoinPool` is an Invoker. Every `Runnable` submitted to them is a Command. Java deliberately chose this design to decouple task definition from execution mechanics.

### javax.swing — Action Interface

```java
// Swing's Action extends ActionListener but adds metadata (name, icon, enabled state)
// This is Command with richer metadata for GUI binding
Action saveAction = new AbstractAction("Save", saveIcon) {
    @Override
    public void actionPerformed(ActionEvent e) {
        document.save(); // The actual Receiver call
    }
};
// The same Action object can be assigned to a menu item, toolbar button,
// and keyboard shortcut — all three invoke the same Command
menuItem.setAction(saveAction);
toolbarButton.setAction(saveAction);
inputMap.put(KeyStroke.getKeyStroke("ctrl S"), "save");
actionMap.put("save", saveAction);
```

### Spring @Async and Spring Batch

```java
// Spring @Async: the annotated method becomes a Command.
// Spring wraps it in a Runnable and submits it to the configured Executor.
// The caller does not know which thread runs the method.
@Service
public class NotificationService {
    @Async("notificationExecutor") // Invoker is the named executor
    public CompletableFuture<Void> sendWelcomeEmail(User user) {
        // This entire method body is the ConcreteCommand's execute()
        emailClient.send(user.getEmail(), "Welcome!");
        return CompletableFuture.completedFuture(null);
    }
}

// Spring Batch: each Step is a Command. The Job is a MacroCommand.
// Steps are queued, can be retried (isRetryable), and produce an audit trail
// (JobExecution) — all Command pattern features.
@Bean
public Job importUserJob(JobBuilderFactory jobBuilderFactory, Step step1, Step step2) {
    return jobBuilderFactory.get("importUserJob")
        .incrementer(new RunIdIncrementer())
        .flow(step1)    // Command 1
        .next(step2)    // Command 2
        .end()
        .build();
}
```

### JDBC Transactions as Commands

```java
// A JDBC transaction groups multiple SQL statements (Commands) into
// an atomic unit. Connection.commit() is the MacroCommand.execute().
// Connection.rollback() is MacroCommand.undo().
// This is the Command pattern at the database protocol level.

public class JdbcTransactionExample {
    public void transferFunds(Connection conn, long fromAccount,
                              long toAccount, BigDecimal amount) throws SQLException {
        conn.setAutoCommit(false); // Start collecting commands
        try {
            // Each SQL statement is a Command encapsulated in a PreparedStatement
            PreparedStatement debit = conn.prepareStatement(
                "UPDATE accounts SET balance = balance - ? WHERE id = ?");
            debit.setBigDecimal(1, amount);
            debit.setLong(2, fromAccount);
            debit.executeUpdate(); // execute()

            PreparedStatement credit = conn.prepareStatement(
                "UPDATE accounts SET balance = balance + ? WHERE id = ?");
            credit.setBigDecimal(1, amount);
            credit.setLong(2, toAccount);
            credit.executeUpdate(); // execute()

            conn.commit(); // MacroCommand.execute() — atomically apply all
        } catch (SQLException ex) {
            conn.rollback(); // MacroCommand.undo() — reverse all
            throw ex;
        }
    }
}
```

### Git as a Command Pattern

Every Git commit is a Command: it encapsulates a set of file changes, the author, and a timestamp. `git revert <hash>` is `command.undo()`. `git rebase` is command re-execution. `git log` is the audit log. Git is the Command pattern at version-control scale.

### Event Sourcing

Event sourcing is the Command pattern applied at architectural scale. Instead of storing the current state of an entity, you store every Command (event) that led to that state. The current state is derived by replaying commands from the beginning. This enables:
- Complete audit trail (who did what and when)
- Time travel (replay to any point in history)
- Event replay (replay events against a new read model)

---

## 10. Trade-offs

### Advantages

| Advantage | Explanation |
|---|---|
| **Undo/Redo** | Commands capture the information needed to reverse themselves. This is the cleanest undo mechanism known in OOP. |
| **Decoupling** | The sender (button, API endpoint, queue consumer) knows nothing about the receiver (domain object). Changing one does not affect the other. |
| **Queueability** | Commands are first-class objects that can be queued, prioritized, scheduled, and dispatched across threads or processes. |
| **Audit / Logging** | Every mutation is an object — log it, persist it, analyze it. Enables compliance, debugging, and replay. |
| **Macro / Composition** | Commands compose naturally via MacroCommand. Users can record sequences and replay them. |
| **Open/Closed** | Adding a new operation means adding a new Command class. Existing Invokers and Receivers are unchanged. |
| **Single Responsibility** | The Receiver handles the domain logic. The Command handles the lifecycle (execute/undo). The Invoker handles dispatch. Clean separation. |
| **Testability** | Commands are pure objects. You can test a command's execute() and undo() in complete isolation, without any GUI or messaging infrastructure. |

### Disadvantages

| Disadvantage | Explanation |
|---|---|
| **Class proliferation** | Each distinct operation requires its own class. A feature-rich application may have dozens or hundreds of Command classes. This increases the class count and cognitive load for new team members. |
| **Memory consumption** | Maintaining a deep command history means holding all command objects in memory. Commands that captured large data snapshots (e.g., image pixels) can exhaust heap. Mitigate with bounded history, diff storage, or Memento for large state. |
| **Undo complexity** | Not all operations can be meaningfully undone. Operations with external side effects (sending emails, charging credit cards, calling non-idempotent APIs) create an "undo illusion" — the UI says undone but the real-world effect persists. These require compensating actions (send a cancellation email), not true reversal. |
| **Serialization complexity** | To persist commands (for durable queues, event sourcing, or crash recovery), commands must be serializable. This means all state they capture must be serializable — which can be non-trivial for commands that hold references to live objects. |
| **Execution ordering** | In concurrent systems, maintaining correct command ordering requires careful synchronization. Out-of-order command execution can corrupt state. |
| **Debugging difficulty** | When something goes wrong, the execution path goes through layers of indirection: Invoker → Command → Receiver. Stack traces are harder to read than direct method call sequences. |

---

## 11. Common Interview Discussion Points

### The Receiver vs. Command Logic Question

Interviewers frequently ask: "Where does the business logic go — in the Command or in the Receiver?" The answer is: **in the Receiver**. The Command's job is lifecycle management (execute, undo, metadata). The Receiver's job is the actual domain work. A Command that contains domain logic violates Single Responsibility — you cannot reuse that logic without going through the Command.

Exception: very thin commands that merely delegate to a single Receiver method are fine. The anti-pattern is a Command that contains significant computation alongside its receiver calls.

### The Null Object Pattern Integration

Well-designed Command systems use the Null Object pattern (`NoOpCommand`) to eliminate null checks in the Invoker. When a remote control slot has no command assigned, it holds a `NoOpCommand` that does nothing on both `execute()` and `undo()`. This is a hallmark of production-quality implementation.

### Undo Stack vs. Memento Pattern

Interviewers often ask about the relationship between Command and Memento. In Command-based undo, the command itself stores the data it needs to reverse (e.g., the deleted text). In Memento-based undo, the Originator creates a snapshot of its entire state before the operation. Command-based undo is more memory-efficient for fine-grained operations. Memento-based undo is simpler to implement for complex state changes where capturing the delta is difficult.

### Thread Safety of Command History

The `CommandHistory` undo/redo stacks must be thread-safe if multiple threads can trigger commands concurrently. Options: use `java.util.concurrent` data structures, use `synchronized` blocks, or (for UI applications) enforce single-thread access from the UI thread (as Swing does with the EDT). Failing to address this is a common production bug.

### The Connection to Event Sourcing and CQRS

Command pattern at application scale = CQRS (Command Query Responsibility Segregation). Write operations are Commands. Read operations are Queries. Commands mutate state; Queries read it. Event sourcing stores Commands (as events) rather than current state. This is the Command pattern applied to entire system architecture.

---

## 12. Interview Questions and Model Answers

### Q1: What problem does the Command pattern solve, and when would you reach for it?

**Model Answer:**

The Command pattern solves the coupling problem between the entity that triggers an action and the entity that performs it. Without this pattern, every button, menu item, or API endpoint must directly reference the domain object it controls. This creates tight coupling — changing the domain object's interface forces changes across all callers.

The pattern solves this by reifying the request: turning the action (a verb) into an object (a noun). The triggering entity holds a reference to a `Command` object and calls `execute()`. It does not know or care what `execute()` does internally.

I reach for Command specifically when I need one or more of:
1. **Undo/redo** — the command captures sufficient context to reverse itself
2. **Operation queuing** — commands can be buffered and dispatched later (different thread, different time, different machine)
3. **Audit logging** — every mutation is an inspectable object I can log and persist
4. **Macro commands** — grouping multiple commands into a single composite operation
5. **Scheduling** — fire a command at a scheduled time, not immediately upon user action

I would NOT use it if I merely want polymorphic behavior at call time with no queuing or history requirements. For that, Strategy is the right pattern — it is lighter weight and avoids the command class proliferation.

---

### Q2: Explain how undo/redo is implemented using the Command pattern. What are the edge cases?

**Model Answer:**

The undo/redo mechanism uses two stacks: an undo stack and a redo stack.

When a command is executed, it is pushed onto the undo stack. When the user presses Undo, the top command is popped from the undo stack, its `undo()` method is called, and it is pushed onto the redo stack. When the user presses Redo, the top command is popped from the redo stack, its `execute()` method is called again, and it is pushed back onto the undo stack.

The critical design question is: what data must the command store to support undo? There are two strategies:

**Strategy A — Capture state at construction:** The command stores the state it needs at the time it is created. Risk: the document might change between creation and execution, making the captured state stale.

**Strategy B — Capture state at execution:** The command captures the data it needs to reverse itself at `execute()` time, not at construction time. For `DeleteCommand`, this means capturing the actual deleted text inside `execute()` before performing the deletion. This is safer and is the approach I always use.

**Edge cases:**

1. **New command invalidates redo:** When the user executes a new command after undoing, the redo stack must be cleared. Otherwise, redoing would re-execute operations that are now inconsistent with the current document state. Every major editor (VS Code, Word) behaves this way.

2. **Bounded history:** An unbounded undo stack will eventually exhaust memory. Production implementations impose a maximum (e.g., 200 steps) and evict the oldest entries from the bottom of the stack.

3. **Irreversible operations:** Some commands cannot meaningfully be undone — sending an external API call, writing to a log, charging a card. For these, `undo()` should either throw `UnsupportedOperationException` (and the command is never pushed to the history stack) or perform a compensating action (send a cancellation request). The architecture must decide which operations are undoable before they are added to the history stack.

4. **Partial macro failure:** If a MacroCommand's third sub-command throws, only the first two executed. The undo method must track `lastExecutedIndex` and only undo the commands that actually ran.

5. **Thread safety:** If multiple threads can execute commands concurrently, the history stack access must be synchronized. In UI applications, this is typically handled by enforcing single-thread command execution on the event dispatch thread.

---

### Q3: How does the Java Executor framework relate to the Command pattern?

**Model Answer:**

The Java Executor framework is the most prominent production instance of the Command pattern in Java. The correspondence is direct:

- `java.lang.Runnable` (or `Callable<V>`) is the **Command interface**. `Runnable.run()` is `Command.execute()`. `Callable.call()` is a Command that also returns a value.
- Each concrete `Runnable` submitted to an executor is a **ConcreteCommand**. It encapsulates a unit of work as an object, independent of when or where it runs.
- `ExecutorService` is the **Invoker**. It receives command objects, decides how many to run concurrently, manages the thread pool, and handles scheduling. The Invoker knows nothing about what any specific Runnable does.
- The code that calls `executor.submit(task)` is the **Client**. It creates the command and hands it to the invoker.

This design gives the Executor framework its power:
- **Decoupling**: task definition is completely separate from execution mechanics. You can switch from a single-threaded executor to a 32-thread pool without changing any task code.
- **Queueing**: if all threads are busy, tasks are buffered in a `BlockingQueue` — the command queue pattern.
- **Priority**: a `PriorityBlockingQueue`-backed `ThreadPoolExecutor` processes tasks in priority order — priority is metadata on the command object.
- **Cancellation**: `Future.cancel()` can prevent a command from executing if it is still in the queue.
- **Transparency**: because tasks are objects, you can wrap the executor to add logging, metrics, circuit breaking, or rate limiting without modifying any task.

`CompletableFuture` extends this further: each `.thenApply()`, `.thenAccept()`, and `.thenCompose()` stage is itself a deferred command that runs when the previous stage completes. The entire pipeline is a MacroCommand built from individual function-commands.

---

### Q4: What is a MacroCommand, and how does it relate to the Composite pattern?

**Model Answer:**

A MacroCommand is a Command that contains a list of other Commands and executes them sequentially. It implements the same `Command` interface as its sub-commands, which means it can be used anywhere a single command is expected. This structural relationship — a component that contains components of the same type — is precisely the Composite pattern.

The benefits of this combination:

1. **Uniform treatment**: an Invoker that knows how to execute one Command automatically knows how to execute a macro of any length and depth. No special-casing required.

2. **Nesting**: macros can contain other macros. A "deploy" macro might contain a "build" macro and a "restart" macro. The Invoker does not care about the depth.

3. **Atomic undo**: calling `undo()` on a macro reverses all its sub-commands in reverse order. The history stack contains one entry (the macro), not N entries for N sub-commands. This is cleaner from the user's perspective.

4. **Partial failure handling**: the MacroCommand must track which sub-commands have executed. If sub-command `k` throws, only commands `0` through `k-1` need to be undone. This requires a `lastExecutedIndex` field, not just a simple for-loop.

5. **Macro recording**: a MacroRecorder wraps command execution, records each command as it runs, and produces a MacroCommand at the end. This is how Excel macro recording and Photoshop actions work.

The key design constraint: MacroCommand's `commands` list should be a defensive copy made at construction time. If the list were shared with the caller, external modifications would corrupt the macro after registration.

---

### Q5: How would you add persistence to a command system so that scheduled commands survive application restarts?

**Model Answer:**

This is a production design question. The answer requires making commands serializable and adding a durable store.

**Step 1 — Make commands serializable.** Commands must be convertible to a format (JSON, Protobuf, Java serialization) that can be written to disk or a database and reconstructed later. I prefer JSON because it is human-readable for debugging. Each command type needs a registered deserializer.

```java
// Command envelope with JSON serialization
public record PersistedCommand(
    String commandType,        // Fully-qualified class name or short name
    String payload,            // JSON payload with command-specific data
    Instant scheduledFor,      // When to run
    String correlationId,
    int    attemptCount
) {}
```

**Step 2 — Persist before execution.** Before executing a scheduled command, write it to a durable store (database, Redis, Kafka). The record must exist before the command runs.

**Step 3 — Idempotent commands.** Since a command might be persisted but the process crashes before marking it complete, the command may execute more than once. Commands must be idempotent: executing them twice must produce the same result as executing once. Use a `commandId` (UUID) as an idempotency key checked against a `processed_commands` table.

**Step 4 — Recovery on startup.** On application startup, query the persistent store for commands that are `PENDING` or `RUNNING` (running may indicate a prior crash) and re-enqueue them. Apply exponential backoff for retries.

**Step 5 — Dead letter handling.** After N failed attempts, move the command to a dead letter queue (separate DB table or topic). Alert on-call and provide tooling to replay or discard dead-lettered commands.

This architecture is essentially a job queue / task queue system — the same design used by Sidekiq, Celery, AWS SQS, and Spring Batch. All of them are Command pattern implementations with durability.

---

### Q6: How does the Command pattern compare to Strategy? When would you use one over the other?

**Model Answer:**

Both patterns encapsulate behavior in an object, which is why they are frequently confused. The distinction is one of purpose and lifecycle.

**Strategy** encapsulates **how** to do something — an algorithm or policy. The strategy object is configured on a context object and called to perform a specific computation. The context knows it is calling a strategy. Strategies are typically stateless (or nearly so). They are selected at setup time and do not accumulate. There is no history of which strategy was used.

**Command** encapsulates **what** to do — a specific action at a specific moment. Commands are created, queued, executed, and potentially stored for later undo or replay. Commands are typically stateful (they capture the data needed for their specific invocation). They accumulate in history. The invoker does not know what the command does.

**Concrete use case distinction:**

- "Sort this list using merge sort or quick sort interchangeably" → Strategy. The sorting algorithm is the behavior being varied.
- "Save this document when the user presses Ctrl+S, and allow the user to undo it" → Command. The save is an action being captured as an object.

**Key differentiators:**

| Dimension | Strategy | Command |
|---|---|---|
| Purpose | Algorithm variation | Action encapsulation |
| State | Stateless (usually) | Stateful (captures invocation data) |
| Accumulation | No history | History for undo/replay |
| Lifecycle | Configured once | Created per invocation |
| Undo/Redo | Not applicable | Core feature |
| Queuing | Not the goal | Core feature |

In practice, a Command's `execute()` method might delegate to a Strategy to determine *how* to perform the action. The two patterns compose well.

---

### Q7: How do you handle commands that have external side effects and cannot be undone (sending emails, charging cards)?

**Model Answer:**

This is a critical production concern. The solution involves three complementary techniques:

**Technique 1 — Compensating commands, not reversal.** True reversal is impossible for external effects. Instead, `undo()` executes a compensating action: send a cancellation email, issue a refund, delete the provisioned resource. The command must capture enough state at `execute()` time to construct the compensation. For example, a `SendEmailCommand.undo()` sends a follow-up "disregard previous email" message.

**Technique 2 — Separate undoable from non-undoable commands.** Introduce a marker interface or flag: `boolean isUndoable()`. Commands that are non-undoable return `false`. The `CommandHistory` checks this flag before pushing onto the undo stack. Non-undoable commands still appear in the audit log (they still ran), but the undo button is disabled when the top of the undo stack is non-undoable.

**Technique 3 — Two-phase execution for critical operations.** For high-value operations (payment processing), use a two-phase approach:
1. `validate()` — check all preconditions without committing
2. `execute()` — commit the action

If `validate()` fails, the command is rejected before any external side effect occurs. If the user wants to "undo" a payment, the system creates a new `RefundCommand` rather than calling `undo()` on the original `ChargeCommand`.

**Technique 4 — Saga pattern for distributed transactions.** When a Command spans multiple microservices (book hotel + book flight + charge card), individual service undo may not be possible if the service has already committed. The Saga pattern extends Command with a formal compensation specification: each step declares its compensating step, and an orchestrator or choreographer invokes compensations in reverse order on failure.

The key principle: never expose an `undo()` that pretends to reverse an effect that it cannot actually reverse. Honest system design acknowledges the asymmetry between do and undo for external effects.

---

### Q8: Design a command system for a financial trading platform where every order must be auditable, atomic, and replayable. What does the Command pattern buy you here?

**Model Answer:**

This is a system design question where Command is the architectural backbone.

**The requirements map directly to Command pattern capabilities:**

- **Auditable**: every order is a Command object logged to an append-only audit store with actor, timestamp, parameters, and outcome. Compliance teams can query any order ever placed.
- **Atomic**: each order is a Command. A `BatchOrderCommand` (MacroCommand) groups related orders into a single undo unit — fill all or cancel all.
- **Replayable**: the audit log stores every Command in sequence. To reconstruct the portfolio at any historical point in time, replay commands up to that timestamp against a blank ledger. This is event sourcing.

**Architecture:**

```
[Trading UI / API]  →  [OrderCommand objects]  →  [CommandQueue]
                                                       ↓
                                               [OrderProcessingWorker]
                                                       ↓
                           [RiskCheckCommand] → [Execute] → [Persist] → [AuditLog]
                                                       ↓
                                               [MarketDataService]
                                               [PositionService]
                                               [RiskService]
```

**Additional Command features leveraged:**

1. **Priority queuing**: market orders (priority 1) are processed before limit orders (priority 5) and reports (priority 9).

2. **Retry with idempotency**: failed orders due to transient connectivity issues are retried. Each `OrderCommand` carries an `orderId` (UUID) as an idempotency key — the exchange rejects duplicates.

3. **Dead-letter queue for failed orders**: orders that exhaust retries go to a dead-letter queue. An operations dashboard shows these for manual review.

4. **Compensating orders**: cancelling a filled order does not undo the fill. It creates a new `CancelOrderCommand` that instructs the exchange to cancel. This is the compensating command approach.

5. **Event sourcing for audit**: instead of storing `Account.balance = $X`, store the sequence of `DepositCommand`, `WithdrawCommand`, `FeeCommand` events. The current balance is derived by summing them. Regulatory auditors can inspect the complete mutation history.

6. **CQRS**: write operations are Commands routed to the write model. Read operations (portfolio view, P&L) query a separate read model materialized by replaying commands. This provides independent scaling of reads and writes.

The Command pattern is not just an implementation detail here — it is the architectural backbone that makes the system auditable, recoverable, and regulatorily compliant.

---

## 13. Comparison with Related Patterns

### Command vs. Chain of Responsibility

| Dimension | Command | Chain of Responsibility |
|---|---|---|
| **Core question** | Who encapsulates the request? | Who handles the request? |
| **Receiver** | Known at construction time; fixed | Unknown; determined at runtime by chain traversal |
| **Handling** | Command always executes (by design) | Handler may pass request along or stop it |
| **History** | Commands accumulate in history for undo | No history; request traverses and is consumed |
| **Use case** | "Do this specific thing, and let me undo it" | "Find someone in this chain who can handle this" |
| **Multiplicity** | One command → one receiver | One request → zero or more handlers |

**When they intersect**: a Chain of Responsibility might create Command objects as its output. For example, a request passes through an authorization filter chain; when the chain approves, it creates and enqueues an `ExecuteOrderCommand`. The chain decided IF the command runs; the command defines WHAT runs.

### Command vs. Strategy

| Dimension | Command | Strategy |
|---|---|---|
| **Purpose** | Encapsulate a specific action instance | Encapsulate an interchangeable algorithm |
| **State** | Stateful (captures invocation-specific data) | Usually stateless |
| **Lifetime** | Created per invocation, accumulated in history | Configured once, reused many times |
| **Undo** | Core feature | Not applicable |
| **Multiplicity** | One per action | One active strategy at a time |
| **Composition** | Macros compose commands | Not composable by design |

**When they intersect**: a Command may use a Strategy internally to determine *how* to execute. A `SortCommand` might accept a `SortStrategy` to determine *how* the sort is performed. This is a natural and clean composition.

### Command vs. Memento

Both patterns support undo, but they capture state differently.

| Dimension | Command | Memento |
|---|---|---|
| **What is stored** | The operation and the data to reverse it | A complete snapshot of the Originator's state |
| **Who knows reversal** | The command itself | The Originator (it creates and restores its own snapshots) |
| **Granularity** | Fine-grained; only what changed | Coarse-grained; full object state |
| **Memory** | Efficient for small deltas | Expensive for large objects |
| **Best for** | Text editing (capture deleted chars) | Complex objects where delta is hard to compute |

**When they combine**: for complex state changes where computing the reverse delta is difficult, a Command can use Memento internally. The Command's `execute()` asks the Receiver to create a Memento of its state before the operation. The Command stores the Memento. The Command's `undo()` tells the Receiver to restore from the Memento. This avoids exposing internals while still enabling undo.

### Summary Decision Matrix

```
Need undo/redo?
  ├── Yes, fine-grained delta (e.g., text edits)     → Command
  ├── Yes, whole-object snapshot                     → Memento (or Command + Memento)
  └── No

Need to select behavior at runtime?
  ├── Behavior is an algorithm applied repeatedly    → Strategy
  └── Behavior is a specific action to be queued    → Command

Need to find a handler dynamically?
  └── Yes                                            → Chain of Responsibility

Need to notify many observers of an event?
  └── Yes                                            → Observer

Need all of: queueing + logging + undo?
  └── Yes                                            → Command (it is the only pattern that provides all three)
```

---

*End of Command Pattern module. Next: [Chain of Responsibility](./04-chain-of-responsibility.md)*
