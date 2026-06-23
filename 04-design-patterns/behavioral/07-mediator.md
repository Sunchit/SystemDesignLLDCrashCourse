# Mediator Design Pattern

> **Category:** Behavioral | **Difficulty:** Intermediate-Advanced | **GoF Pattern #16**

---

## Table of Contents

1. [Intent and Problem Statement](#1-intent-and-problem-statement)
2. [Air Traffic Control Analogy](#2-air-traffic-control-analogy)
3. [When to Use / When NOT to Use](#3-when-to-use--when-not-to-use)
4. [UML Class Diagrams](#4-uml-class-diagrams)
5. [Implementation 1: Chat Application](#5-implementation-1-chat-application)
6. [Implementation 2: Air Traffic Control System](#6-implementation-2-air-traffic-control-system)
7. [MVC — Controller as Mediator](#7-mvc--controller-as-mediator)
8. [Mediator vs Observer: Deep Comparison](#8-mediator-vs-observer-deep-comparison)
9. [Real-World Framework Examples](#9-real-world-framework-examples)
10. [Trade-offs](#10-trade-offs)
11. [Common Interview Discussion Points](#11-common-interview-discussion-points)
12. [Interview Questions and Model Answers](#12-interview-questions-and-model-answers)
13. [Comparison with Related Patterns](#13-comparison-with-related-patterns)

---

## 1. Intent and Problem Statement

### Formal Definition

> **Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly, and it lets you vary their interaction independently.**
>
> — Gang of Four, *Design Patterns: Elements of Reusable Object-Oriented Software*

### The Core Problem: Many-to-Many Dependency Explosion

Consider a system with `n` components that need to communicate with each other. Without a mediator, every component must know about every other component it needs to interact with. This creates **O(n²) dependencies** — the classic many-to-many coupling problem.

**Without Mediator (Many-to-Many):**

```
n = 5 components
Maximum connections = n * (n-1) / 2 = 10 direct dependencies

   A ←——→ B
   ↕ ↘  ↗ ↕
   ↕   ×   ↕
   ↕ ↗  ↘ ↕
   C ←——→ D
      ↕
      E
```

Every component is tightly coupled to several others. Adding a 6th component can require up to 5 new connections. Removing a component requires hunting down every reference across the codebase. Testing a single component requires mocking all its peers.

**With Mediator (Many-to-One + One-to-Many):**

```
n = 5 components + 1 mediator
Connections = 2n = 10 (each component talks only to mediator)

   A ——→ M ←—— B
         ↕
   C ——→ M ←—— D
         ↕
         E
```

Each component depends only on the mediator interface. Adding or removing components touches exactly one place: the mediator. The interaction logic is centralized, auditable, and replaceable.

### What the Mediator Solves

| Problem | Without Mediator | With Mediator |
|---------|-----------------|---------------|
| Adding a new component | Update N existing components | Update only the mediator |
| Changing an interaction | Find and update all involved parties | Change one method in the mediator |
| Testing a component in isolation | Mock N-1 peers | Mock one mediator interface |
| Understanding the flow | Trace calls across N files | Read the mediator class |
| Reusing a component in a different context | Drag along all its dependencies | Swap the mediator implementation |

### Participants

- **Mediator (interface):** Declares the communication interface used by Colleague objects. Usually a single `notify()` or `send()` method.
- **ConcreteMediator:** Implements the Mediator interface. Knows and coordinates all Colleague objects. Contains the interaction logic.
- **Colleague (interface/abstract class):** Each Colleague knows its Mediator and communicates with peers only through it.
- **ConcreteColleague:** Sends events to the mediator; receives callbacks from it.

---

## 2. Air Traffic Control Analogy

### The Problem Without ATC

Imagine an airport with no Air Traffic Control tower. Each pilot must:

1. Continuously broadcast their position, altitude, heading, and speed on a shared radio frequency.
2. Listen to every other aircraft's broadcast.
3. Independently compute whether any other aircraft is on a collision course.
4. Negotiate runway access directly with every other aircraft.
5. Coordinate gate assignments bilaterally.

With 10 aircraft in the pattern, each aircraft must simultaneously track 9 others. With 50 aircraft — a typical busy airport — each must track 49 others. The cognitive load is impossible, the radio chatter is deafening, and the probability of a catastrophic miscommunication approaches certainty. This is the **O(n²) problem** in the physical world.

### The Solution: The Control Tower (Mediator)

Every pilot is trained in a single, simple rule: **talk to the tower, not to each other**. The ATC controller becomes the single mediator for all airspace activity.

**What ATC centralizes:**

- **State awareness:** The radar operator at ATC tracks all aircraft simultaneously — position, altitude, speed, fuel state, emergencies. Individual pilots do not need to track each other.
- **Runway sequencing:** When three aircraft want to land on runway 28R, they don't negotiate among themselves. Each simply tells ATC "ready to land." ATC sequences them: "Delta 447, number one for 28R, cleared to land. United 112, number two, maintain 3000 feet. Southwest 89, enter the holding pattern."
- **Conflict resolution:** ATC spots that two aircraft are converging and issues corrective headings to both. Neither aircraft needed to know the other existed until ATC resolved the conflict.
- **Takeoff coordination:** No aircraft begins its takeoff roll without explicit ATC clearance. ATC knows what's on the runway, what's on final approach, and what's taxiing.
- **Emergency handling:** When a pilot declares an emergency, ATC immediately clears all traffic on affected runways, coordinates with rescue services, and notifies the airline — without any pilot needing to manage the chaos.

**The ATC protocol in software terms:**

```
Pilot (Colleague) → send("READY_TO_LAND", flightData) → ControlTower (Mediator)
ControlTower → consults runway state, sequence queue → determinesAction()
ControlTower → notify(pilot1, "CLEARED_TO_LAND") + notify(pilot2, "GO_AROUND")
```

### Why This Analogy Is Perfect

The ATC analogy captures every nuance of the Mediator pattern:

- **Colleagues (Aircraft)** know the mediator (the tower frequency) but not each other.
- **The mediator (ATC)** knows all colleagues and orchestrates interactions.
- **Interaction logic** (runway sequencing rules, separation minima, approach paths) lives entirely in the mediator, not in the aircraft.
- **Extensibility:** When drones are added to the airspace, the "new colleague type" registers with ATC. Existing aircraft are unaffected.
- **The god object risk:** ATC controllers can become overloaded (the "god object" problem). Large airports solve this by splitting into sectors — Ground Control, Tower, Approach, En-Route — each a more focused mediator. This mirrors how you decompose large mediators in software.

---

## 3. When to Use / When NOT to Use

### When to Use

**1. You have a set of objects that communicate in complex but well-defined ways.**
The communication is complex (not just A calls B), but the rules are stable enough to codify in one place. Chat rooms, UI form validation (changing one field affects which others are enabled), workflow engines.

**2. Reusing an object is difficult because it holds references to many others.**
If you can't extract a component into a library or a new microservice because it carries too many cross-component dependencies, a mediator breaks those chains.

**3. Behavior distributed across multiple classes needs to be customizable without subclassing everything.**
Instead of subclassing five components, you subclass or replace the one mediator. The GoF note: "you can vary the interaction independently of the colleagues."

**4. You find yourself passing the same set of objects into many constructors.**
When five service classes all receive the same three or four dependencies, those dependencies are acting as an implicit mediator — make it explicit.

**5. Event-driven UI components.**
Classic scenario: a form where selecting a country updates a state/province dropdown, which updates a city dropdown, and the Submit button only enables when all three are valid. Each component should not know about the others; the form controller (mediator) wires them.

**6. Workflow or state machine orchestration.**
The mediator pattern is the backbone of workflow engines like Apache Airflow, AWS Step Functions, and BPMN engines. The orchestrator is the mediator; task nodes are colleagues.

### When NOT to Use

**1. You have a small, stable set of well-defined interactions.**
If A always calls B and that's it, direct coupling is clearer. Introducing a mediator to manage two classes communicating in one direction is over-engineering.

**2. The interaction complexity is inherent, not accidental.**
If the many-to-many communication represents genuine business complexity (e.g., a mature rules engine), packing it into a single mediator class just moves the complexity rather than reducing it.

**3. Performance is critical and the indirection is prohibitive.**
Game engines processing thousands of entity interactions per frame often cannot afford the mediator dispatch overhead. Direct references with careful architecture are preferable.

**4. The "god object" risk is already materializing.**
If your mediator is already 2000 lines and growing, the pattern has become an anti-pattern. Decompose into domain-specific sub-mediators before adding more. If decomposition is not clean, consider whether the problem truly fits the mediator pattern.

**5. You need guaranteed message ordering or delivery semantics.**
Mediator is typically in-process and synchronous. If you need durable messaging, fan-out with acknowledgments, or replay, use a proper message broker (Kafka, RabbitMQ) rather than a handrolled mediator.

---

## 4. UML Class Diagrams

### Before: Many-to-Many (The Problem)

```
┌─────────────┐     ←knows→    ┌─────────────┐
│    UserA    │ ─────────────→ │    UserB    │
│             │ ←───────────── │             │
└─────────────┘                └─────────────┘
      │  ↑                           │  ↑
      │  │                           │  │
      ↓  │                           ↓  │
┌─────────────┐     ←knows→    ┌─────────────┐
│    UserC    │ ─────────────→ │    UserD    │
│             │ ←───────────── │             │
└─────────────┘                └─────────────┘

Every User must hold references to every other User it communicates with.
Adding UserE requires updating ALL existing users.
n=4 → up to 6 bidirectional dependencies.
n=10 → up to 45 bidirectional dependencies.
```

### After: Through Mediator (The Solution)

```
             «interface»
            ┌────────────────────┐
            │      Mediator      │
            │ ─────────────────  │
            │ + send(msg, sender)│
            │ + addColleague(..) │
            └────────────────────┘
                      ▲
                      │ implements
                      │
            ┌─────────────────────────────┐
            │      ConcreteMediator       │
            │ ──────────────────────────  │
            │ - colleagues: List<Colleague>│
            │ + send(msg, sender)         │
            │   (routes to correct peer)  │
            └─────────────────────────────┘
              /    /       \    \
             /    /         \    \
     notifies    notifies  notifies notifies
           /    /             \    \
          ↓    ↓               ↓    ↓
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│Colleague │ │Colleague │ │Colleague │ │Colleague │
│    A     │ │    B     │ │    C     │ │    D     │
│ ─────────│ │ ─────────│ │ ─────────│ │ ─────────│
│-mediator │ │-mediator │ │-mediator │ │-mediator │
│+receive()│ │+receive()│ │+receive()│ │+receive()│
│+send()   │ │+send()   │ │+send()   │ │+send()   │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
     │             │             │             │
     └─────────────┴─────────────┴─────────────┘
          All colleagues hold ONLY a reference
          to the Mediator interface — never
          to each other.
```

### Chat Application Specific Diagram

```
         «interface»
        ┌──────────────────────────────────────────┐
        │               ChatMediator               │
        │ ──────────────────────────────────────── │
        │ + sendMessage(msg: Message, from: User)  │
        │ + sendPrivateMessage(msg, from, to: User)│
        │ + broadcast(msg: Message, from: User)    │
        │ + registerUser(user: User)               │
        │ + removeUser(user: User)                 │
        └──────────────────────────────────────────┘
                          ▲
                          │
              ┌───────────────────────┐
              │       ChatRoom        │
              │ ─────────────────────│
              │ - name: String        │
              │ - users: Map<String,  │
              │     User>             │
              │ - moderators: Set     │
              │ - messageHistory: List│
              │ + sendMessage(...)    │
              │ + banUser(...)        │
              │ + muteUser(...)       │
              └───────────────────────┘
        /            |            \
       /             |             \
«uses»/          «uses»|         «uses»\
     /               |               \
┌──────────┐  ┌────────────┐  ┌──────────────┐
│   User   │  │ AdminUser  │  │  BotUser     │
│ ─────────│  │ ───────────│  │ ────────────  │
│-mediator │  │-mediator   │  │-mediator     │
│-username │  │-permissions│  │-triggers     │
│+send()   │  │+banUser()  │  │+autoRespond()│
│+receive()│  │+muteUser() │  │+receive()    │
└──────────┘  └────────────┘  └──────────────┘
```

---

## 5. Implementation 1: Chat Application

A production-quality chat system demonstrating the Mediator pattern with multiple user types, private messaging, broadcast, channel-based messaging, and moderation.

```java
package patterns.behavioral.mediator.chat;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.stream.Collectors;

// ─────────────────────────────────────────────────────────────
// DOMAIN OBJECTS
// ─────────────────────────────────────────────────────────────

enum MessageType {
    BROADCAST,
    PRIVATE,
    CHANNEL,
    SYSTEM,
    MODERATION
}

class Message {
    private final String id;
    private final String content;
    private final MessageType type;
    private final LocalDateTime timestamp;
    private final String senderId;
    private final String recipientId; // null for broadcast/channel
    private final String channelId;   // null for private/broadcast
    private boolean flagged;

    public Message(String content, MessageType type, String senderId,
                   String recipientId, String channelId) {
        this.id = UUID.randomUUID().toString().substring(0, 8);
        this.content = content;
        this.type = type;
        this.senderId = senderId;
        this.recipientId = recipientId;
        this.channelId = channelId;
        this.timestamp = LocalDateTime.now();
        this.flagged = false;
    }

    // Static factory methods for clarity
    public static Message broadcast(String content, String senderId) {
        return new Message(content, MessageType.BROADCAST, senderId, null, null);
    }

    public static Message privateMessage(String content, String senderId, String recipientId) {
        return new Message(content, MessageType.PRIVATE, senderId, recipientId, null);
    }

    public static Message channelMessage(String content, String senderId, String channelId) {
        return new Message(content, MessageType.CHANNEL, senderId, null, channelId);
    }

    public static Message systemMessage(String content) {
        return new Message(content, MessageType.SYSTEM, "SYSTEM", null, null);
    }

    public void flag() { this.flagged = true; }

    public String getId() { return id; }
    public String getContent() { return content; }
    public MessageType getType() { return type; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public String getSenderId() { return senderId; }
    public String getRecipientId() { return recipientId; }
    public String getChannelId() { return channelId; }
    public boolean isFlagged() { return flagged; }

    @Override
    public String toString() {
        String time = timestamp.format(DateTimeFormatter.ofPattern("HH:mm:ss"));
        return String.format("[%s][%s][%s] %s", time, type, senderId, content);
    }
}

class ModerationEvent {
    public enum Action { BAN, MUTE, UNMUTE, WARN, FLAG_MESSAGE }

    private final Action action;
    private final String targetUserId;
    private final String moderatorId;
    private final String reason;
    private final LocalDateTime timestamp;

    public ModerationEvent(Action action, String targetUserId,
                           String moderatorId, String reason) {
        this.action = action;
        this.targetUserId = targetUserId;
        this.moderatorId = moderatorId;
        this.reason = reason;
        this.timestamp = LocalDateTime.now();
    }

    public Action getAction() { return action; }
    public String getTargetUserId() { return targetUserId; }
    public String getModeratorId() { return moderatorId; }
    public String getReason() { return reason; }

    @Override
    public String toString() {
        return String.format("[MODERATION] %s applied %s to %s. Reason: %s",
                moderatorId, action, targetUserId, reason);
    }
}

// ─────────────────────────────────────────────────────────────
// MEDIATOR INTERFACE
// ─────────────────────────────────────────────────────────────

interface ChatMediator {
    void registerUser(User user);
    void removeUser(String userId);
    void sendMessage(Message message);
    void joinChannel(String userId, String channelId);
    void leaveChannel(String userId, String channelId);
    void applyModeration(ModerationEvent event);
    List<Message> getHistory(String channelId, int limit);
    List<String> getActiveUsers();
}

// ─────────────────────────────────────────────────────────────
// CONCRETE MEDIATOR — ChatRoom
// ─────────────────────────────────────────────────────────────

class ChatRoom implements ChatMediator {
    private final String roomName;

    // User registry
    private final Map<String, User> users = new ConcurrentHashMap<>();

    // Channel subscriptions: channelId → set of userIds
    private final Map<String, Set<String>> channels = new ConcurrentHashMap<>();

    // Message history per channel (and "global" for broadcasts)
    private final Map<String, List<Message>> messageHistory = new ConcurrentHashMap<>();

    // Moderation state
    private final Set<String> bannedUsers = Collections.synchronizedSet(new HashSet<>());
    private final Set<String> mutedUsers = Collections.synchronizedSet(new HashSet<>());
    private final List<ModerationEvent> moderationLog = new CopyOnWriteArrayList<>();

    // Profanity filter (simplified)
    private final Set<String> bannedWords = new HashSet<>(Arrays.asList("spam", "badword"));

    public ChatRoom(String roomName) {
        this.roomName = roomName;
        messageHistory.put("global", new CopyOnWriteArrayList<>());
    }

    // ── Registration ──────────────────────────────────────────

    @Override
    public void registerUser(User user) {
        if (bannedUsers.contains(user.getUserId())) {
            user.receive(Message.systemMessage(
                    "You are banned from " + roomName + "."));
            return;
        }
        users.put(user.getUserId(), user);
        broadcast(Message.systemMessage(
                user.getDisplayName() + " has joined " + roomName + "."));
        System.out.println("[ChatRoom] " + user.getDisplayName() + " registered.");
    }

    @Override
    public void removeUser(String userId) {
        User user = users.remove(userId);
        if (user != null) {
            // Remove from all channel subscriptions
            channels.values().forEach(subs -> subs.remove(userId));
            broadcast(Message.systemMessage(
                    user.getDisplayName() + " has left " + roomName + "."));
        }
    }

    // ── Message Routing (core mediator logic) ─────────────────

    @Override
    public void sendMessage(Message message) {
        String senderId = message.getSenderId();

        // Sender must be registered (SYSTEM is exempt)
        if (!"SYSTEM".equals(senderId) && !users.containsKey(senderId)) {
            System.err.println("[ChatRoom] Unknown sender: " + senderId);
            return;
        }

        // Muted check
        if (mutedUsers.contains(senderId)) {
            User sender = users.get(senderId);
            if (sender != null) {
                sender.receive(Message.systemMessage(
                        "You are muted and cannot send messages."));
            }
            return;
        }

        // Content moderation
        if (containsBannedWords(message.getContent())) {
            User sender = users.get(senderId);
            if (sender != null) {
                sender.receive(Message.systemMessage(
                        "Your message was blocked by the content filter."));
            }
            applyModeration(new ModerationEvent(
                    ModerationEvent.Action.WARN, senderId,
                    "AUTO_MODERATOR", "Banned words in message"));
            return;
        }

        // Route by message type
        switch (message.getType()) {
            case BROADCAST:
                broadcast(message);
                break;
            case PRIVATE:
                routePrivate(message);
                break;
            case CHANNEL:
                routeToChannel(message);
                break;
            case SYSTEM:
            case MODERATION:
                broadcast(message);
                break;
        }
    }

    private void broadcast(Message message) {
        storeMessage("global", message);
        users.values().forEach(user -> user.receive(message));
    }

    private void routePrivate(Message message) {
        User recipient = users.get(message.getRecipientId());
        if (recipient == null) {
            User sender = users.get(message.getSenderId());
            if (sender != null) {
                sender.receive(Message.systemMessage(
                        "User " + message.getRecipientId() + " is not online."));
            }
            return;
        }
        // Deliver to recipient and echo back to sender
        storeMessage("private_" + normalize(message.getSenderId(), message.getRecipientId()),
                message);
        recipient.receive(message);
        User sender = users.get(message.getSenderId());
        if (sender != null && !sender.getUserId().equals(message.getRecipientId())) {
            sender.receive(message); // Echo
        }
    }

    private void routeToChannel(Message message) {
        String channelId = message.getChannelId();
        Set<String> subscribers = channels.getOrDefault(channelId, Collections.emptySet());

        if (!subscribers.contains(message.getSenderId())) {
            User sender = users.get(message.getSenderId());
            if (sender != null) {
                sender.receive(Message.systemMessage(
                        "You must join #" + channelId + " before posting."));
            }
            return;
        }

        storeMessage(channelId, message);
        subscribers.stream()
                .map(users::get)
                .filter(Objects::nonNull)
                .forEach(user -> user.receive(message));
    }

    // ── Channel Management ────────────────────────────────────

    @Override
    public void joinChannel(String userId, String channelId) {
        if (!users.containsKey(userId)) return;
        channels.computeIfAbsent(channelId,
                k -> Collections.synchronizedSet(new HashSet<>())).add(userId);
        messageHistory.computeIfAbsent(channelId, k -> new CopyOnWriteArrayList<>());
        User user = users.get(userId);
        routeToChannel(Message.channelMessage(
                user.getDisplayName() + " joined #" + channelId, "SYSTEM", channelId));
        // Also add SYSTEM to channel so the message routes
        channels.get(channelId).add("SYSTEM");
    }

    @Override
    public void leaveChannel(String userId, String channelId) {
        Set<String> subs = channels.get(channelId);
        if (subs != null) {
            subs.remove(userId);
        }
    }

    // ── Moderation ────────────────────────────────────────────

    @Override
    public void applyModeration(ModerationEvent event) {
        moderationLog.add(event);
        System.out.println(event);

        switch (event.getAction()) {
            case BAN:
                bannedUsers.add(event.getTargetUserId());
                User bannedUser = users.get(event.getTargetUserId());
                if (bannedUser != null) {
                    bannedUser.receive(Message.systemMessage(
                            "You have been banned. Reason: " + event.getReason()));
                }
                removeUser(event.getTargetUserId());
                broadcast(Message.systemMessage(
                        event.getTargetUserId() + " was banned from " + roomName + "."));
                break;

            case MUTE:
                mutedUsers.add(event.getTargetUserId());
                User mutedUser = users.get(event.getTargetUserId());
                if (mutedUser != null) {
                    mutedUser.receive(Message.systemMessage(
                            "You have been muted. Reason: " + event.getReason()));
                }
                break;

            case UNMUTE:
                mutedUsers.remove(event.getTargetUserId());
                User unmutedUser = users.get(event.getTargetUserId());
                if (unmutedUser != null) {
                    unmutedUser.receive(Message.systemMessage("You have been unmuted."));
                }
                break;

            case WARN:
                User warnedUser = users.get(event.getTargetUserId());
                if (warnedUser != null) {
                    warnedUser.receive(Message.systemMessage(
                            "Warning: " + event.getReason()));
                }
                break;

            case FLAG_MESSAGE:
                // Would normally look up message by ID and flag it
                System.out.println("[ChatRoom] Message flagged for review.");
                break;
        }
    }

    // ── Query ─────────────────────────────────────────────────

    @Override
    public List<Message> getHistory(String channelId, int limit) {
        List<Message> history = messageHistory.getOrDefault(channelId, Collections.emptyList());
        int fromIndex = Math.max(0, history.size() - limit);
        return Collections.unmodifiableList(history.subList(fromIndex, history.size()));
    }

    @Override
    public List<String> getActiveUsers() {
        return users.values().stream()
                .map(User::getDisplayName)
                .collect(Collectors.toList());
    }

    // ── Helpers ───────────────────────────────────────────────

    private void storeMessage(String key, Message message) {
        messageHistory.computeIfAbsent(key, k -> new CopyOnWriteArrayList<>()).add(message);
    }

    private boolean containsBannedWords(String content) {
        String lower = content.toLowerCase();
        return bannedWords.stream().anyMatch(lower::contains);
    }

    /** Creates a canonical key for a private conversation between two users. */
    private String normalize(String a, String b) {
        return a.compareTo(b) < 0 ? a + "_" + b : b + "_" + a;
    }

    public String getRoomName() { return roomName; }
    public List<ModerationEvent> getModerationLog() {
        return Collections.unmodifiableList(moderationLog);
    }
}

// ─────────────────────────────────────────────────────────────
// COLLEAGUE — User (abstract base)
// ─────────────────────────────────────────────────────────────

abstract class User {
    protected final String userId;
    protected final String displayName;
    protected ChatMediator mediator;
    protected final List<Message> inbox = new CopyOnWriteArrayList<>();

    public User(String userId, String displayName) {
        this.userId = userId;
        this.displayName = displayName;
    }

    public void setMediator(ChatMediator mediator) {
        this.mediator = mediator;
    }

    /** Called BY the mediator to deliver a message to this user. */
    public abstract void receive(Message message);

    /** Sends a broadcast message through the mediator. */
    public void sendBroadcast(String content) {
        requireMediator();
        mediator.sendMessage(Message.broadcast(content, userId));
    }

    /** Sends a private message through the mediator. */
    public void sendPrivate(String content, String recipientId) {
        requireMediator();
        mediator.sendMessage(Message.privateMessage(content, userId, recipientId));
    }

    /** Sends a message to a channel through the mediator. */
    public void sendToChannel(String content, String channelId) {
        requireMediator();
        mediator.sendMessage(Message.channelMessage(content, userId, channelId));
    }

    public void joinChannel(String channelId) {
        requireMediator();
        mediator.joinChannel(userId, channelId);
    }

    public void leaveChannel(String channelId) {
        requireMediator();
        mediator.leaveChannel(userId, channelId);
    }

    protected void requireMediator() {
        if (mediator == null) {
            throw new IllegalStateException(
                    displayName + " is not registered with any chat room.");
        }
    }

    public String getUserId() { return userId; }
    public String getDisplayName() { return displayName; }
    public List<Message> getInbox() { return Collections.unmodifiableList(inbox); }
}

// ─────────────────────────────────────────────────────────────
// CONCRETE COLLEAGUE — RegularUser
// ─────────────────────────────────────────────────────────────

class RegularUser extends User {
    public RegularUser(String userId, String displayName) {
        super(userId, displayName);
    }

    @Override
    public void receive(Message message) {
        inbox.add(message);
        // Simple console output — in production this would push to a WebSocket/SSE stream
        System.out.printf("  [→ %s] %s%n", displayName, message);
    }
}

// ─────────────────────────────────────────────────────────────
// CONCRETE COLLEAGUE — AdminUser
// ─────────────────────────────────────────────────────────────

class AdminUser extends User {
    private final Set<String> permissions;

    public AdminUser(String userId, String displayName, String... permissions) {
        super(userId, displayName);
        this.permissions = new HashSet<>(Arrays.asList(permissions));
    }

    @Override
    public void receive(Message message) {
        inbox.add(message);
        System.out.printf("  [→ ADMIN:%s] %s%n", displayName, message);
    }

    public void banUser(String targetUserId, String reason) {
        requirePermission("BAN");
        requireMediator();
        mediator.applyModeration(new ModerationEvent(
                ModerationEvent.Action.BAN, targetUserId, userId, reason));
    }

    public void muteUser(String targetUserId, String reason) {
        requirePermission("MUTE");
        requireMediator();
        mediator.applyModeration(new ModerationEvent(
                ModerationEvent.Action.MUTE, targetUserId, userId, reason));
    }

    public void unmuteUser(String targetUserId) {
        requirePermission("MUTE");
        requireMediator();
        mediator.applyModeration(new ModerationEvent(
                ModerationEvent.Action.UNMUTE, targetUserId, userId, "Admin decision"));
    }

    public void warnUser(String targetUserId, String reason) {
        requirePermission("WARN");
        requireMediator();
        mediator.applyModeration(new ModerationEvent(
                ModerationEvent.Action.WARN, targetUserId, userId, reason));
    }

    private void requirePermission(String permission) {
        if (!permissions.contains(permission) && !permissions.contains("ALL")) {
            throw new SecurityException(
                    displayName + " lacks permission: " + permission);
        }
    }
}

// ─────────────────────────────────────────────────────────────
// CONCRETE COLLEAGUE — BotUser
// Auto-responds to trigger phrases; demonstrates non-human colleague
// ─────────────────────────────────────────────────────────────

class BotUser extends User {
    private final Map<String, String> triggerResponses = new LinkedHashMap<>();

    public BotUser(String botId, String displayName) {
        super(botId, displayName);
        // Register default triggers
        triggerResponses.put("!help", "Available commands: !help, !time, !rules");
        triggerResponses.put("!time", "Current server time: " +
                java.time.LocalTime.now().format(DateTimeFormatter.ofPattern("HH:mm:ss")));
        triggerResponses.put("!rules", "1. Be respectful. 2. No spam. 3. Stay on topic.");
    }

    public void addTrigger(String trigger, String response) {
        triggerResponses.put(trigger.toLowerCase(), response);
    }

    @Override
    public void receive(Message message) {
        inbox.add(message);
        // Bot processes incoming messages to detect triggers
        if (message.getType() == MessageType.BROADCAST
                || message.getType() == MessageType.CHANNEL) {
            String content = message.getContent().trim().toLowerCase();
            String response = triggerResponses.get(content);
            if (response != null && mediator != null
                    && !message.getSenderId().equals(userId)) {
                // Route response back through the mediator — bot is a proper colleague
                String channelId = message.getChannelId();
                if (channelId != null) {
                    mediator.sendMessage(Message.channelMessage(response, userId, channelId));
                } else {
                    mediator.sendMessage(Message.broadcast(response, userId));
                }
            }
        }
    }

    @Override
    public void joinChannel(String channelId) {
        super.joinChannel(channelId);
        // Bots also subscribe themselves to the channel
        if (mediator != null) {
            mediator.joinChannel(userId, channelId);
        }
    }
}

// ─────────────────────────────────────────────────────────────
// DEMO / MAIN
// ─────────────────────────────────────────────────────────────

class ChatApplicationDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("═══════════════════════════════════════════");
        System.out.println("     MEDIATOR PATTERN — CHAT APPLICATION   ");
        System.out.println("═══════════════════════════════════════════\n");

        // 1. Create the mediator (ChatRoom)
        ChatRoom room = new ChatRoom("TechTalks");

        // 2. Create colleagues (Users) — they hold a reference to the mediator
        RegularUser alice   = new RegularUser("alice", "Alice");
        RegularUser bob     = new RegularUser("bob", "Bob");
        RegularUser charlie = new RegularUser("charlie", "Charlie");
        AdminUser   diana   = new AdminUser("diana", "Diana", "ALL");
        BotUser     helpBot = new BotUser("helpbot", "HelpBot");

        // 3. Wire mediator into each colleague
        alice.setMediator(room);
        bob.setMediator(room);
        charlie.setMediator(room);
        diana.setMediator(room);
        helpBot.setMediator(room);

        // 4. Register with the mediator
        System.out.println("--- Registration ---");
        room.registerUser(alice);
        room.registerUser(bob);
        room.registerUser(charlie);
        room.registerUser(diana);
        room.registerUser(helpBot);

        // 5. Broadcast messaging
        System.out.println("\n--- Broadcast Messages ---");
        alice.sendBroadcast("Hello everyone! Excited to be here.");
        bob.sendBroadcast("Great to meet you all!");

        // 6. Private messaging (routed through mediator, not directly)
        System.out.println("\n--- Private Messages ---");
        alice.sendPrivate("Hey Bob, want to collaborate on a project?", "bob");
        bob.sendPrivate("Absolutely Alice, let's discuss in #projects channel.", "alice");

        // 7. Channel-based messaging
        System.out.println("\n--- Channel: #projects ---");
        alice.joinChannel("projects");
        bob.joinChannel("projects");
        helpBot.joinChannel("projects");

        alice.sendToChannel("I'm starting the new design document.", "projects");
        bob.sendToChannel("I'll review it once it's ready.", "projects");
        alice.sendToChannel("!help", "projects");  // Triggers bot response

        // Charlie not in channel — mediator blocks and notifies
        charlie.sendToChannel("Can I join?", "projects");
        charlie.joinChannel("projects");
        charlie.sendToChannel("Now I can contribute!", "projects");

        // 8. Content moderation — auto-filter
        System.out.println("\n--- Content Moderation (Auto-filter) ---");
        bob.sendBroadcast("Please don't spam the channel.");
        // This will be blocked by profanity filter
        alice.sendBroadcast("This is spam and should be filtered.");

        // 9. Admin moderation actions
        System.out.println("\n--- Admin Moderation ---");
        diana.warnUser("charlie", "Please stay on topic.");
        diana.muteUser("bob", "Repeated off-topic messages (demo).");
        bob.sendBroadcast("Can I speak now?");   // Should be blocked
        diana.unmuteUser("bob");
        bob.sendBroadcast("Thanks Diana, I'm back!");

        // 10. History retrieval
        System.out.println("\n--- Message History (last 3 in #projects) ---");
        room.getHistory("projects", 3)
                .forEach(m -> System.out.println("  " + m));

        // 11. Active users
        System.out.println("\n--- Active Users ---");
        System.out.println("  " + room.getActiveUsers());

        // 12. User leaves
        System.out.println("\n--- User Departure ---");
        room.removeUser("charlie");

        // 13. Moderation log
        System.out.println("\n--- Moderation Log ---");
        room.getModerationLog().forEach(e -> System.out.println("  " + e));

        System.out.println("\n═══════════════════════════════════════════");
        System.out.println("          DEMO COMPLETE");
        System.out.println("═══════════════════════════════════════════");
    }
}
```

---

## 6. Implementation 2: Air Traffic Control System

A complete ATC simulation demonstrating runway management, landing/takeoff sequencing, and emergency handling.

```java
package patterns.behavioral.mediator.atc;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.PriorityBlockingQueue;

// ─────────────────────────────────────────────────────────────
// DOMAIN OBJECTS
// ─────────────────────────────────────────────────────────────

enum AircraftState {
    PARKED,
    TAXIING_OUT,
    HOLDING_SHORT,          // Waiting at runway hold line
    LINED_UP,               // On runway awaiting clearance
    TAKING_OFF,
    AIRBORNE,
    IN_HOLDING_PATTERN,     // Circling while waiting to land
    ON_APPROACH,            // Descending toward runway
    LANDING,
    TAXIING_IN,
    EMERGENCY
}

enum AircraftType {
    LIGHT,          // Cessna 172, etc.
    MEDIUM,         // Boeing 737, A320
    HEAVY,          // Boeing 777, A380
    SUPER_HEAVY     // Antonov An-225
}

class FlightPlan {
    private final String flightNumber;
    private final String origin;
    private final String destination;
    private final AircraftType aircraftType;
    private final int fuelLevelMinutes; // Fuel remaining in minutes of flight
    private final boolean emergency;

    public FlightPlan(String flightNumber, String origin, String destination,
                      AircraftType aircraftType, int fuelLevelMinutes, boolean emergency) {
        this.flightNumber = flightNumber;
        this.origin = origin;
        this.destination = destination;
        this.aircraftType = aircraftType;
        this.fuelLevelMinutes = fuelLevelMinutes;
        this.emergency = emergency;
    }

    public String getFlightNumber() { return flightNumber; }
    public String getOrigin() { return origin; }
    public String getDestination() { return destination; }
    public AircraftType getAircraftType() { return aircraftType; }
    public int getFuelLevelMinutes() { return fuelLevelMinutes; }
    public boolean isEmergency() { return emergency; }
}

class ATCMessage {
    public enum Type {
        // Requests from Aircraft to Tower
        REQUEST_TAKEOFF,
        REQUEST_LANDING,
        DECLARE_EMERGENCY,
        POSITION_REPORT,
        READY_FOR_DEPARTURE,
        // Instructions from Tower to Aircraft
        CLEARED_FOR_TAKEOFF,
        CLEARED_TO_LAND,
        GO_AROUND,
        HOLD_POSITION,
        ENTER_HOLDING_PATTERN,
        TAXI_TO_RUNWAY,
        TAXI_TO_GATE,
        SQUAWK_CODE,
        EMERGENCY_CLEARED,
        // Internal tower coordination
        RUNWAY_CLEAR,
        RUNWAY_OCCUPIED
    }

    private final Type type;
    private final String fromId;
    private final String toId;
    private final String runwayId;
    private final String additionalInfo;
    private final LocalDateTime timestamp;

    public ATCMessage(Type type, String fromId, String toId,
                      String runwayId, String additionalInfo) {
        this.type = type;
        this.fromId = fromId;
        this.toId = toId;
        this.runwayId = runwayId;
        this.additionalInfo = additionalInfo;
        this.timestamp = LocalDateTime.now();
    }

    public Type getType() { return type; }
    public String getFromId() { return fromId; }
    public String getToId() { return toId; }
    public String getRunwayId() { return runwayId; }
    public String getAdditionalInfo() { return additionalInfo; }

    @Override
    public String toString() {
        String time = timestamp.format(DateTimeFormatter.ofPattern("HH:mm:ss"));
        return String.format("[%s] %s → %s | %s | RWY:%s %s",
                time, fromId, toId, type,
                runwayId != null ? runwayId : "N/A",
                additionalInfo != null ? "(" + additionalInfo + ")" : "");
    }
}

class Runway {
    private final String runwayId;
    private final String heading;        // e.g., "28R", "10L"
    private final int lengthMeters;
    private final AircraftType maxAircraftType;

    private boolean occupied;
    private String occupyingFlightId;

    // Wake turbulence separation timer (simplified as a counter)
    private long lastDepartureTimeMs;
    private AircraftType lastDepartingType;

    public Runway(String runwayId, String heading, int lengthMeters,
                  AircraftType maxAircraftType) {
        this.runwayId = runwayId;
        this.heading = heading;
        this.lengthMeters = lengthMeters;
        this.maxAircraftType = maxAircraftType;
        this.occupied = false;
    }

    public boolean canAccept(AircraftType type) {
        return !occupied && type.ordinal() <= maxAircraftType.ordinal();
    }

    public boolean isClearAfterWakeTurbulence(AircraftType incomingType) {
        if (lastDepartingType == null) return true;
        long elapsed = System.currentTimeMillis() - lastDepartureTimeMs;
        // Simplified: heavy aircraft need 120s separation, medium 60s, light 30s
        long requiredSeparationMs = switch (lastDepartingType) {
            case SUPER_HEAVY -> 180_000;
            case HEAVY -> 120_000;
            case MEDIUM -> 60_000;
            case LIGHT -> 30_000;
        };
        return elapsed >= requiredSeparationMs;
    }

    public void occupy(String flightId) {
        this.occupied = true;
        this.occupyingFlightId = flightId;
    }

    public void clear() {
        this.occupied = false;
        this.occupyingFlightId = null;
    }

    public void recordDeparture(AircraftType type) {
        this.lastDepartingType = type;
        this.lastDepartureTimeMs = System.currentTimeMillis();
    }

    public boolean isOccupied() { return occupied; }
    public String getRunwayId() { return runwayId; }
    public String getHeading() { return heading; }
    public String getOccupyingFlightId() { return occupyingFlightId; }
}

// ─────────────────────────────────────────────────────────────
// MEDIATOR INTERFACE
// ─────────────────────────────────────────────────────────────

interface AirTrafficMediator {
    void registerAircraft(Aircraft aircraft);
    void deregisterAircraft(String flightId);
    void sendToTower(ATCMessage message);
    void coordinateEmergency(String flightId, String emergencyType);
}

// ─────────────────────────────────────────────────────────────
// CONCRETE MEDIATOR — ControlTower
// ─────────────────────────────────────────────────────────────

class ControlTower implements AirTrafficMediator {
    private final String airportCode;

    // Aircraft registry
    private final Map<String, Aircraft> aircraft = new ConcurrentHashMap<>();

    // Runway registry
    private final Map<String, Runway> runways = new ConcurrentHashMap<>();

    // Sequencing queues
    private final Queue<String> landingQueue = new LinkedList<>();  // FIFO for normal ops
    private final Queue<String> takeoffQueue = new LinkedList<>();

    // Communication log
    private final List<ATCMessage> communicationLog = new CopyOnWriteArrayList<>();

    // Active emergencies
    private final Set<String> activeEmergencies = Collections.synchronizedSet(new HashSet<>());

    public ControlTower(String airportCode) {
        this.airportCode = airportCode;
    }

    // ── Infrastructure Setup ──────────────────────────────────

    public void addRunway(Runway runway) {
        runways.put(runway.getRunwayId(), runway);
        log("Runway " + runway.getRunwayId() + " (" + runway.getHeading() + ") activated.");
    }

    // ── Aircraft Registration ─────────────────────────────────

    @Override
    public void registerAircraft(Aircraft a) {
        aircraft.put(a.getFlightId(), a);
        log("Aircraft " + a.getFlightId() + " registered with " + airportCode + " ATC.");
        // Assign squawk code (transponder code)
        int squawk = 1000 + aircraft.size();
        ATCMessage squawkMsg = new ATCMessage(
                ATCMessage.Type.SQUAWK_CODE, airportCode, a.getFlightId(),
                null, "Squawk " + squawk);
        sendToAircraft(squawkMsg);
    }

    @Override
    public void deregisterAircraft(String flightId) {
        aircraft.remove(flightId);
        landingQueue.remove(flightId);
        takeoffQueue.remove(flightId);
        log("Aircraft " + flightId + " deregistered.");
    }

    // ── Core Mediation Logic ──────────────────────────────────

    @Override
    public void sendToTower(ATCMessage message) {
        communicationLog.add(message);
        log("RECEIVED: " + message);

        switch (message.getType()) {
            case REQUEST_TAKEOFF        -> handleTakeoffRequest(message);
            case REQUEST_LANDING        -> handleLandingRequest(message);
            case DECLARE_EMERGENCY      -> handleEmergencyDeclaration(message);
            case POSITION_REPORT        -> handlePositionReport(message);
            case READY_FOR_DEPARTURE    -> handleReadyForDeparture(message);
            case RUNWAY_CLEAR           -> handleRunwayClear(message);
            default -> log("Unhandled message type: " + message.getType());
        }
    }

    private void handleTakeoffRequest(ATCMessage message) {
        String flightId = message.getFromId();
        Aircraft requestingAircraft = aircraft.get(flightId);
        if (requestingAircraft == null) return;

        // Check for an available runway suitable for this aircraft type
        Optional<Runway> availableRunway = findAvailableRunwayForDeparture(
                requestingAircraft.getFlightPlan().getAircraftType());

        if (availableRunway.isPresent()) {
            Runway runway = availableRunway.get();
            runway.occupy(flightId);
            ATCMessage clearance = new ATCMessage(
                    ATCMessage.Type.CLEARED_FOR_TAKEOFF,
                    airportCode, flightId, runway.getRunwayId(),
                    "Wind 270/10, runway " + runway.getRunwayId() + " cleared for takeoff");
            sendToAircraft(clearance);
            log("Cleared " + flightId + " for takeoff on " + runway.getRunwayId());
        } else {
            // No runway available — add to queue
            if (!takeoffQueue.contains(flightId)) {
                takeoffQueue.offer(flightId);
            }
            ATCMessage holdMsg = new ATCMessage(
                    ATCMessage.Type.HOLD_POSITION,
                    airportCode, flightId, null,
                    "Hold short. Position #" + takeoffQueue.size() + " for departure.");
            sendToAircraft(holdMsg);
            log("No runway available. " + flightId + " queued for takeoff.");
        }
    }

    private void handleLandingRequest(ATCMessage message) {
        String flightId = message.getFromId();
        Aircraft requestingAircraft = aircraft.get(flightId);
        if (requestingAircraft == null) return;

        FlightPlan fp = requestingAircraft.getFlightPlan();
        boolean isEmergency = fp.isEmergency()
                || activeEmergencies.contains(flightId)
                || fp.getFuelLevelMinutes() < 20;

        if (isEmergency) {
            // Emergency aircraft get priority — clear the runway immediately
            handleEmergencyLanding(flightId, message.getRunwayId());
            return;
        }

        Optional<Runway> availableRunway = findAvailableRunwayForLanding(
                fp.getAircraftType());

        if (availableRunway.isPresent()) {
            Runway runway = availableRunway.get();
            runway.occupy(flightId);
            ATCMessage clearance = new ATCMessage(
                    ATCMessage.Type.CLEARED_TO_LAND,
                    airportCode, flightId, runway.getRunwayId(),
                    "Wind 270/10, runway " + runway.getRunwayId()
                            + " cleared to land. Traffic: none");
            sendToAircraft(clearance);
            log("Cleared " + flightId + " to land on " + runway.getRunwayId());
        } else {
            // Direct to holding pattern
            if (!landingQueue.contains(flightId)) {
                landingQueue.offer(flightId);
            }
            ATCMessage holdMsg = new ATCMessage(
                    ATCMessage.Type.ENTER_HOLDING_PATTERN,
                    airportCode, flightId, null,
                    "Enter holding pattern at ALPHA fix, 5000ft. Expect #"
                            + landingQueue.size() + " for approach.");
            sendToAircraft(holdMsg);
            log("No runway available. " + flightId + " directed to holding pattern.");
        }
    }

    private void handleEmergencyDeclaration(ATCMessage message) {
        String flightId = message.getFromId();
        activeEmergencies.add(flightId);
        log("!!! EMERGENCY DECLARED by " + flightId + " — " + message.getAdditionalInfo());

        // Immediately clear the best available runway
        Runway emergencyRunway = findLongestRunway();
        if (emergencyRunway.isOccupied()) {
            // Issue go-around to current occupant
            String occupant = emergencyRunway.getOccupyingFlightId();
            if (occupant != null) {
                ATCMessage goAround = new ATCMessage(
                        ATCMessage.Type.GO_AROUND, airportCode, occupant,
                        emergencyRunway.getRunwayId(),
                        "Go around. Emergency traffic inbound. Climb runway heading, 3000ft.");
                sendToAircraft(goAround);
                emergencyRunway.clear();
                // Re-queue the displaced aircraft
                if (!landingQueue.contains(occupant)) {
                    landingQueue.offer(occupant);
                }
            }
        }

        // Clear all aircraft on taxiways crossing this runway
        notifyAllAircraftOfEmergency(flightId, emergencyRunway.getRunwayId());

        // Clear emergency aircraft to land
        emergencyRunway.occupy(flightId);
        ATCMessage emergencyClearance = new ATCMessage(
                ATCMessage.Type.CLEARED_TO_LAND, airportCode, flightId,
                emergencyRunway.getRunwayId(),
                "Emergency clearance. Runway " + emergencyRunway.getRunwayId()
                        + " cleared. Emergency services standing by.");
        sendToAircraft(emergencyClearance);
    }

    private void handleEmergencyLanding(String flightId, String preferredRunway) {
        // Reuse emergency declaration logic
        handleEmergencyDeclaration(new ATCMessage(
                ATCMessage.Type.DECLARE_EMERGENCY, flightId, airportCode,
                preferredRunway, "Fuel emergency / declared emergency"));
    }

    private void handlePositionReport(ATCMessage message) {
        // Update aircraft state based on position reports
        // In production this would update radar tracks
        log("Position report from " + message.getFromId() + ": "
                + message.getAdditionalInfo());
    }

    private void handleReadyForDeparture(ATCMessage message) {
        // Aircraft is at the gate, ready to push back and taxi
        String flightId = message.getFromId();
        ATCMessage taxiInstruction = new ATCMessage(
                ATCMessage.Type.TAXI_TO_RUNWAY,
                airportCode, flightId, determineActiveRunway().getRunwayId(),
                "Push back approved. Taxi via Alpha, Bravo to runway "
                        + determineActiveRunway().getRunwayId() + ". Hold short.");
        sendToAircraft(taxiInstruction);
    }

    private void handleRunwayClear(ATCMessage message) {
        String clearedRunwayId = message.getRunwayId();
        Runway runway = runways.get(clearedRunwayId);
        if (runway == null) return;

        runway.clear();
        log("Runway " + clearedRunwayId + " is now clear.");

        // Process landing queue first (safety priority)
        if (!landingQueue.isEmpty()) {
            String nextFlightId = landingQueue.poll();
            Aircraft nextAircraft = aircraft.get(nextFlightId);
            if (nextAircraft != null) {
                runway.occupy(nextFlightId);
                ATCMessage clearance = new ATCMessage(
                        ATCMessage.Type.CLEARED_TO_LAND,
                        airportCode, nextFlightId, clearedRunwayId,
                        "Runway " + clearedRunwayId + " cleared to land. "
                                + "Previous traffic has vacated.");
                sendToAircraft(clearance);
            }
        } else if (!takeoffQueue.isEmpty()) {
            // Then process departures
            String nextFlightId = takeoffQueue.poll();
            Aircraft nextAircraft = aircraft.get(nextFlightId);
            if (nextAircraft != null) {
                runway.occupy(nextFlightId);
                runway.recordDeparture(
                        nextAircraft.getFlightPlan().getAircraftType());
                ATCMessage clearance = new ATCMessage(
                        ATCMessage.Type.CLEARED_FOR_TAKEOFF,
                        airportCode, nextFlightId, clearedRunwayId,
                        "Runway " + clearedRunwayId + " cleared for takeoff.");
                sendToAircraft(clearance);
            }
        }
    }

    // ── Emergency Coordination ────────────────────────────────

    @Override
    public void coordinateEmergency(String flightId, String emergencyType) {
        log("!!! COORDINATING EMERGENCY for " + flightId + ": " + emergencyType);
        activeEmergencies.add(flightId);

        // Notify all airborne aircraft to increase separation
        aircraft.values().stream()
                .filter(a -> !a.getFlightId().equals(flightId))
                .filter(a -> a.getState() == AircraftState.AIRBORNE
                        || a.getState() == AircraftState.ON_APPROACH)
                .forEach(a -> {
                    ATCMessage alert = new ATCMessage(
                            ATCMessage.Type.ENTER_HOLDING_PATTERN,
                            airportCode, a.getFlightId(), null,
                            "Emergency traffic. Maintain position. Await further instructions.");
                    sendToAircraft(alert);
                });
    }

    // ── Helper Methods ────────────────────────────────────────

    private void sendToAircraft(ATCMessage message) {
        communicationLog.add(message);
        Aircraft recipient = aircraft.get(message.getToId());
        if (recipient != null) {
            recipient.receiveFromTower(message);
        }
    }

    private Optional<Runway> findAvailableRunwayForDeparture(AircraftType type) {
        return runways.values().stream()
                .filter(r -> r.canAccept(type))
                .filter(r -> r.isClearAfterWakeTurbulence(type))
                .findFirst();
    }

    private Optional<Runway> findAvailableRunwayForLanding(AircraftType type) {
        return runways.values().stream()
                .filter(r -> r.canAccept(type))
                .findFirst();
    }

    private Runway findLongestRunway() {
        return runways.values().stream()
                .max(Comparator.comparing(r -> 1)) // Simplified — would sort by length
                .orElseThrow(() -> new IllegalStateException("No runways configured"));
    }

    private Runway determineActiveRunway() {
        return runways.values().iterator().next(); // Simplified
    }

    private void notifyAllAircraftOfEmergency(String emergencyFlight, String runwayId) {
        aircraft.values().stream()
                .filter(a -> !a.getFlightId().equals(emergencyFlight))
                .forEach(a -> {
                    ATCMessage alert = new ATCMessage(
                            ATCMessage.Type.HOLD_POSITION,
                            airportCode, a.getFlightId(), null,
                            "Emergency aircraft on final approach to " + runwayId
                                    + ". All traffic hold position.");
                    sendToAircraft(alert);
                });
    }

    private void log(String msg) {
        String time = LocalDateTime.now()
                .format(DateTimeFormatter.ofPattern("HH:mm:ss"));
        System.out.println("[" + time + "][" + airportCode + " TOWER] " + msg);
    }

    public List<ATCMessage> getCommunicationLog() {
        return Collections.unmodifiableList(communicationLog);
    }

    public void printRunwayStatus() {
        System.out.println("\n--- Runway Status ---");
        runways.values().forEach(r ->
                System.out.printf("  Runway %-5s [%s]: %s%n",
                        r.getRunwayId(), r.getHeading(),
                        r.isOccupied() ? "OCCUPIED by " + r.getOccupyingFlightId() : "CLEAR"));
    }
}

// ─────────────────────────────────────────────────────────────
// COLLEAGUE — Aircraft (abstract base)
// ─────────────────────────────────────────────────────────────

abstract class Aircraft {
    protected final String flightId;
    protected final FlightPlan flightPlan;
    protected AirTrafficMediator mediator;
    protected AircraftState state;
    protected final List<ATCMessage> receivedMessages = new CopyOnWriteArrayList<>();

    public Aircraft(String flightId, FlightPlan flightPlan) {
        this.flightId = flightId;
        this.flightPlan = flightPlan;
        this.state = AircraftState.PARKED;
    }

    public void setMediator(AirTrafficMediator mediator) {
        this.mediator = mediator;
    }

    /** Called BY the mediator (tower) to deliver instructions to this aircraft. */
    public abstract void receiveFromTower(ATCMessage message);

    // ── Pilot actions (send TO mediator) ─────────────────────

    public void requestTakeoff() {
        state = AircraftState.HOLDING_SHORT;
        ATCMessage request = new ATCMessage(
                ATCMessage.Type.REQUEST_TAKEOFF, flightId,
                "TOWER", null, flightPlan.getDestination());
        log("Requesting takeoff clearance for " + flightPlan.getDestination());
        mediator.sendToTower(request);
    }

    public void requestLanding(String preferredRunway) {
        state = AircraftState.ON_APPROACH;
        ATCMessage request = new ATCMessage(
                ATCMessage.Type.REQUEST_LANDING, flightId,
                "TOWER", preferredRunway,
                "Inbound from " + flightPlan.getOrigin()
                        + ", fuel: " + flightPlan.getFuelLevelMinutes() + "min");
        log("Requesting landing clearance on " + preferredRunway);
        mediator.sendToTower(request);
    }

    public void declareEmergency(String emergencyType) {
        state = AircraftState.EMERGENCY;
        ATCMessage emergency = new ATCMessage(
                ATCMessage.Type.DECLARE_EMERGENCY, flightId,
                "TOWER", null, emergencyType);
        log("MAYDAY MAYDAY MAYDAY — " + emergencyType);
        mediator.sendToTower(emergency);
        mediator.coordinateEmergency(flightId, emergencyType);
    }

    public void reportRunwayClear(String runwayId) {
        ATCMessage report = new ATCMessage(
                ATCMessage.Type.RUNWAY_CLEAR, flightId,
                "TOWER", runwayId, "Runway vacated");
        mediator.sendToTower(report);
    }

    public void reportPosition(String position) {
        ATCMessage report = new ATCMessage(
                ATCMessage.Type.POSITION_REPORT, flightId,
                "TOWER", null, position);
        mediator.sendToTower(report);
    }

    protected void log(String msg) {
        System.out.printf("  [%s][%s] %s%n",
                state, flightId, msg);
    }

    public String getFlightId() { return flightId; }
    public FlightPlan getFlightPlan() { return flightPlan; }
    public AircraftState getState() { return state; }
    public List<ATCMessage> getReceivedMessages() {
        return Collections.unmodifiableList(receivedMessages);
    }
}

// ─────────────────────────────────────────────────────────────
// CONCRETE COLLEAGUE — CommercialFlight
// ─────────────────────────────────────────────────────────────

class CommercialFlight extends Aircraft {
    public CommercialFlight(String flightId, FlightPlan flightPlan) {
        super(flightId, flightPlan);
    }

    @Override
    public void receiveFromTower(ATCMessage message) {
        receivedMessages.add(message);
        log("RECEIVED from Tower: " + message.getType()
                + (message.getRunwayId() != null ? " on " + message.getRunwayId() : "")
                + (message.getAdditionalInfo() != null
                        ? " — " + message.getAdditionalInfo() : ""));

        // React to ATC instructions — state transitions
        switch (message.getType()) {
            case CLEARED_FOR_TAKEOFF -> {
                state = AircraftState.TAKING_OFF;
                log("Commencing takeoff roll on " + message.getRunwayId()
                        + ". Rotating...");
                // Simulate becoming airborne
                state = AircraftState.AIRBORNE;
                log("Airborne! Climbing to assigned altitude.");
                // Notify tower that runway is clear
                reportRunwayClear(message.getRunwayId());
            }
            case CLEARED_TO_LAND -> {
                state = AircraftState.LANDING;
                log("Established on final approach to " + message.getRunwayId()
                        + ". Gear down.");
                // Simulate landing
                state = AircraftState.TAXIING_IN;
                log("Landed. Vacating " + message.getRunwayId() + " via rapid exit.");
                reportRunwayClear(message.getRunwayId());
                state = AircraftState.PARKED;
            }
            case GO_AROUND -> {
                state = AircraftState.AIRBORNE;
                log("Going around. Climbing runway heading, 3000ft. ");
            }
            case HOLD_POSITION -> {
                log("Holding position. " + message.getAdditionalInfo());
            }
            case ENTER_HOLDING_PATTERN -> {
                state = AircraftState.IN_HOLDING_PATTERN;
                log("Entering holding pattern. " + message.getAdditionalInfo());
            }
            case TAXI_TO_RUNWAY -> {
                state = AircraftState.TAXIING_OUT;
                log("Taxiing via " + message.getAdditionalInfo());
            }
            case SQUAWK_CODE -> {
                log("Setting transponder: " + message.getAdditionalInfo());
            }
            default -> log("Acknowledged: " + message.getAdditionalInfo());
        }
    }
}

// ─────────────────────────────────────────────────────────────
// CONCRETE COLLEAGUE — PrivatePlane
// Lighter aircraft with different separation requirements
// ─────────────────────────────────────────────────────────────

class PrivatePlane extends Aircraft {
    public PrivatePlane(String flightId, FlightPlan flightPlan) {
        super(flightId, flightPlan);
    }

    @Override
    public void receiveFromTower(ATCMessage message) {
        receivedMessages.add(message);
        log("RCVD: " + message.getType()
                + (message.getAdditionalInfo() != null
                        ? " — " + message.getAdditionalInfo() : ""));

        switch (message.getType()) {
            case CLEARED_FOR_TAKEOFF -> {
                state = AircraftState.TAKING_OFF;
                log("Rolling on " + message.getRunwayId() + ".");
                state = AircraftState.AIRBORNE;
                reportRunwayClear(message.getRunwayId());
            }
            case CLEARED_TO_LAND -> {
                state = AircraftState.LANDING;
                log("Short final " + message.getRunwayId() + ".");
                state = AircraftState.PARKED;
                reportRunwayClear(message.getRunwayId());
            }
            case HOLD_POSITION, ENTER_HOLDING_PATTERN ->
                log("Holding. " + message.getAdditionalInfo());
            case GO_AROUND -> {
                state = AircraftState.AIRBORNE;
                log("Going around.");
            }
            default -> log("Roger: " + message.getAdditionalInfo());
        }
    }
}

// ─────────────────────────────────────────────────────────────
// DEMO / MAIN
// ─────────────────────────────────────────────────────────────

class ATCDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("═══════════════════════════════════════════");
        System.out.println("   MEDIATOR PATTERN — AIR TRAFFIC CONTROL  ");
        System.out.println("═══════════════════════════════════════════\n");

        // 1. Create the mediator (ControlTower)
        ControlTower tower = new ControlTower("KLAX");

        // 2. Configure runways
        tower.addRunway(new Runway("24L", "240°", 3685, AircraftType.SUPER_HEAVY));
        tower.addRunway(new Runway("24R", "240°", 3382, AircraftType.HEAVY));
        tower.addRunway(new Runway("06L", "060°", 2720, AircraftType.MEDIUM));

        // 3. Create Aircraft (colleagues) — they know only the mediator interface
        FlightPlan ua123Plan = new FlightPlan(
                "UA123", "KSFO", "KLAX", AircraftType.HEAVY, 45, false);
        FlightPlan dl456Plan = new FlightPlan(
                "DL456", "KJFK", "KLAX", AircraftType.MEDIUM, 60, false);
        FlightPlan n789Plan = new FlightPlan(
                "N789PV", "KVNY", "KLAX", AircraftType.LIGHT, 30, false);
        FlightPlan sw321Plan = new FlightPlan(
                "SW321", "KLAS", "KLAX", AircraftType.MEDIUM, 15, false); // Low fuel!
        FlightPlan aa555Plan = new FlightPlan(
                "AA555", "KLAX", "KORD", AircraftType.HEAVY, 480, false);

        CommercialFlight ua123 = new CommercialFlight("UA123", ua123Plan);
        CommercialFlight dl456 = new CommercialFlight("DL456", dl456Plan);
        PrivatePlane     n789  = new PrivatePlane("N789PV", n789Plan);
        CommercialFlight sw321 = new CommercialFlight("SW321", sw321Plan);
        CommercialFlight aa555 = new CommercialFlight("AA555", aa555Plan);

        // 4. Wire mediator
        ua123.setMediator(tower);
        dl456.setMediator(tower);
        n789.setMediator(tower);
        sw321.setMediator(tower);
        aa555.setMediator(tower);

        // 5. Register with the mediator (tower)
        System.out.println("--- Aircraft Registration ---");
        tower.registerAircraft(ua123);
        tower.registerAircraft(dl456);
        tower.registerAircraft(n789);
        tower.registerAircraft(sw321);
        tower.registerAircraft(aa555);

        // 6. Scenario: Multiple landing requests
        System.out.println("\n--- Scenario 1: Multiple Landing Requests ---");
        ua123.requestLanding("24L");   // Gets clearance (runway available)
        dl456.requestLanding("24L");   // 24L now occupied, gets 24R
        n789.requestLanding("06L");    // Gets 06L
        sw321.requestLanding("24R");   // All runways occupied → holding pattern
        // UA123 and DL456 landing clears the runway → SW321 queued gets clearance

        // 7. Scenario: Departure
        System.out.println("\n--- Scenario 2: Departure ---");
        aa555.requestTakeoff();

        tower.printRunwayStatus();

        // 8. Scenario: Emergency
        System.out.println("\n--- Scenario 3: EMERGENCY ---");
        FlightPlan qf1Plan = new FlightPlan(
                "QF001", "YSSY", "KLAX", AircraftType.SUPER_HEAVY, 8, true);
        CommercialFlight qf001 = new CommercialFlight("QF001", qf1Plan);
        qf001.setMediator(tower);
        tower.registerAircraft(qf001);

        qf001.declareEmergency("ENGINE FIRE — engine #4. Fuel dumping.");

        tower.printRunwayStatus();

        System.out.println("\n--- Communication Log Summary ---");
        System.out.printf("Total ATC communications: %d messages%n",
                tower.getCommunicationLog().size());

        System.out.println("\n═══════════════════════════════════════════");
        System.out.println("          ATC DEMO COMPLETE");
        System.out.println("═══════════════════════════════════════════");
    }
}
```

---

## 7. MVC — Controller as Mediator

### How MVC Embodies the Mediator Pattern

MVC (Model-View-Controller) is the most pervasive application of the Mediator pattern in software engineering. The Controller is the mediator between the Model and the View. Neither the Model nor the View knows about the other directly.

```
                    ┌─────────────────┐
                    │   Controller    │ ← THE MEDIATOR
                    │ (e.g., Spring   │
                    │  @Controller)   │
                    └────────┬────────┘
                   /         │         \
  User Input      /          │          \  Model Update
                 ↓           │           ↓
        ┌────────────┐   reads/writes  ┌────────────┐
        │    View    │                 │   Model    │
        │ (Thymeleaf │  ←————————————  │  (JPA      │
        │  / React)  │  (rendered data)│  Entity)   │
        └────────────┘                 └────────────┘

  View never directly modifies Model.
  Model never directly renders itself to View.
  Controller mediates all interactions.
```

### Spring MVC DispatcherServlet as Macro-Level Mediator

Spring MVC takes this a level higher. The `DispatcherServlet` is itself a mediator that coordinates multiple subsystems:

```java
// Spring's DispatcherServlet acts as a mediator orchestrating:
// - HandlerMapping      (finds which @Controller handles the request)
// - HandlerAdapter      (knows how to invoke the handler)
// - ViewResolver        (translates logical view name to a template)
// - MessageConverters   (handles @ResponseBody serialization)
// - ExceptionHandlers   (@ExceptionHandler / @ControllerAdvice)

// You, as the developer, write a controller (colleague):
@Controller
public class OrderController {
    // Controller has NO knowledge of:
    // - How HandlerMapping found it
    // - Which ViewResolver will render the response
    // - Which MessageConverter will serialize the JSON
    // The DispatcherServlet (mediator) handles all of that.

    private final OrderService orderService;
    private final InventoryService inventoryService;

    public OrderController(OrderService orderService,
                           InventoryService inventoryService) {
        this.orderService = orderService;
        this.inventoryService = inventoryService;
    }

    // This method IS ITSELF acting as a mediator between
    // OrderService and InventoryService — within the MVC tier.
    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {

        // Coordinate two services — the controller mediates
        boolean inStock = inventoryService.checkAvailability(
                request.getProductId(), request.getQuantity());

        if (!inStock) {
            return ResponseEntity.status(HttpStatus.CONFLICT)
                    .body(OrderResponse.outOfStock(request.getProductId()));
        }

        Order order = orderService.createOrder(request);
        inventoryService.reserve(request.getProductId(), request.getQuantity());

        return ResponseEntity.status(HttpStatus.CREATED)
                .body(OrderResponse.from(order));
    }
}
```

### UI Form Controller as Mediator — Classic Example

The most textbook demonstration is a complex form where UI components must react to each other without directly knowing about each other:

```java
// Without Mediator — tight coupling between form fields:
class CountryDropdown {
    private StateDropdown stateDropdown;       // Direct reference
    private CityDropdown cityDropdown;         // Direct reference
    private SubmitButton submitButton;         // Direct reference

    void onSelectionChanged(String country) {
        stateDropdown.populateFor(country);   // Directly calls peer
        cityDropdown.clear();
        submitButton.disable();
    }
}

// With Mediator — form controller decouples components:
interface FormMediator {
    void onFieldChanged(String fieldId, Object newValue);
}

class RegistrationFormController implements FormMediator {
    private CountryDropdown countryDropdown;
    private StateDropdown stateDropdown;
    private CityDropdown cityDropdown;
    private PostalCodeField postalCode;
    private SubmitButton submitButton;
    private AddressValidationService validationService;

    @Override
    public void onFieldChanged(String fieldId, Object newValue) {
        switch (fieldId) {
            case "country" -> {
                String country = (String) newValue;
                stateDropdown.populate(
                        validationService.getStatesFor(country));
                cityDropdown.clear();
                postalCode.setFormat(
                        validationService.getPostalFormat(country));
                submitButton.setEnabled(isFormComplete());
            }
            case "state" -> {
                String state = (String) newValue;
                cityDropdown.populate(
                        validationService.getCitiesFor(state));
                submitButton.setEnabled(isFormComplete());
            }
            case "city", "postalCode" ->
                submitButton.setEnabled(isFormComplete());
        }
    }

    private boolean isFormComplete() {
        return countryDropdown.hasSelection()
                && stateDropdown.hasSelection()
                && cityDropdown.hasSelection()
                && postalCode.isValid();
    }
}

// Each UI component:
class CountryDropdown {
    private FormMediator mediator;   // Only knows the mediator

    public CountryDropdown(FormMediator mediator) {
        this.mediator = mediator;
    }

    void onSelectionChanged(String country) {
        // Does NOT call stateDropdown.populate(...)
        // Delegates to mediator instead
        mediator.onFieldChanged("country", country);
    }
}
```

---

## 8. Mediator vs Observer: Deep Comparison

This is one of the most commonly confused pattern pairs. Understanding the distinction deeply is a strong signal of seniority.

### The Core Philosophical Difference

| Dimension | Mediator | Observer |
|-----------|----------|----------|
| **Topology** | Star (hub-and-spoke) | Broadcast tree |
| **Knowledge** | Mediator knows all colleagues | Publisher knows only Subject interface |
| **Direction** | Bidirectional coordination | Unidirectional notification |
| **Coupling** | Colleagues coupled to mediator | Observers coupled to Subject |
| **Intent** | Reduce many-to-many dependencies | Notify dependents of state change |
| **Logic location** | Centralized in mediator | Distributed in each observer |
| **Control** | Mediator decides what happens | Each observer decides independently |

### Communication Patterns

**Observer (Decentralized):**
```
Subject changes state
  → notifies Observer1 → Observer1 reacts independently
  → notifies Observer2 → Observer2 reacts independently
  → notifies Observer3 → Observer3 reacts independently

Subject has no idea what observers do.
Observers have no idea what other observers do.
No coordination between observers.
```

**Mediator (Centralized):**
```
ColleagueA sends event to Mediator
  → Mediator evaluates overall state
  → Mediator decides ColleagueB gets X instruction
  → Mediator decides ColleagueC gets Y instruction
  → Mediator decides ColleagueD is unaffected

Mediator has full awareness and control.
Colleagues have no awareness of each other.
Responses can be conditional, sequential, and coordinated.
```

### When Each Pattern Applies

| Use Observer when... | Use Mediator when... |
|---------------------|---------------------|
| A state change should notify any number of subscribers without needing to know what they do | The interaction between components is complex and has logic that doesn't belong in any single component |
| The publisher should remain decoupled from subscriber behavior | Different components must react to each other's changes in a coordinated way |
| Event fans out with no coordination needed between receivers | The response of one component affects what another component should do |
| You want to support dynamic subscription lists at runtime | The set of interacting components is known and relatively stable |
| The pattern scales via the event bus / message broker | You need a single point of auditing, modification, or replacement of interaction logic |

### Concrete Comparison Example

**Scenario:** A stock trading platform where price changes in AAPL should:
1. Update the price display widget
2. Check all limit orders and execute any triggered
3. Recalculate portfolio values for all holders
4. Log the price change for audit

```java
// Observer approach: appropriate here because each reaction
// is independent and the publisher doesn't coordinate them.

interface StockObserver {
    void onPriceChanged(String ticker, double oldPrice, double newPrice);
}

class StockPrice { // Subject
    private final List<StockObserver> observers = new ArrayList<>();
    private double currentPrice;

    public void setPrice(double newPrice) {
        double old = this.currentPrice;
        this.currentPrice = newPrice;
        observers.forEach(o -> o.onPriceChanged(ticker, old, newPrice));
        // Publisher does NOT coordinate what observers do — they're independent
    }
}
```

```java
// Mediator approach: appropriate when coordination IS needed.
// Example: in a trading system, when executing a limit order,
// you must coordinate inventory (reserve shares),
// settlement (lock funds), and notification (alert trader)
// in a specific order, with rollback if any step fails.

class TradingMediator {
    private InventoryService inventory;
    private SettlementService settlement;
    private NotificationService notifications;
    private AuditService audit;

    public void executeLimitOrder(Order order) {
        // Coordinated transaction — order matters, rollback needed
        try {
            inventory.reserveShares(order.getTicker(), order.getQuantity());
            settlement.lockFunds(order.getAccountId(), order.getValue());
            // Only after both succeed:
            inventory.confirmTransfer(order);
            settlement.confirmSettlement(order);
            notifications.alertTrader(order.getAccountId(), order);
            audit.logExecution(order);
        } catch (InsufficientFundsException e) {
            inventory.releaseReservation(order);
            notifications.alertFailed(order.getAccountId(), order, e.getMessage());
        }
    }
}
```

### Can They Be Used Together?

Yes — and this is a senior-level insight. The Mediator can internally use Observer to notify colleagues:

```java
class EventBusMediator implements Mediator {
    // Uses Observer (event bus) internally for delivery,
    // but applies Mediator logic for routing and coordination.

    private final Map<String, List<BiConsumer<String, Object>>> handlers =
            new HashMap<>();

    public void subscribe(String eventType, BiConsumer<String, Object> handler) {
        handlers.computeIfAbsent(eventType, k -> new ArrayList<>()).add(handler);
    }

    public void notify(Object sender, String event, Object data) {
        // Mediator logic: conditional routing, transformation, filtering
        if ("user.login".equals(event) && isUserSuspended(data)) {
            // Intercept and redirect — mediator decision
            handlers.getOrDefault("user.suspended", List.of())
                    .forEach(h -> h.accept(event, data));
            return;
        }
        handlers.getOrDefault(event, List.of())
                .forEach(h -> h.accept(event, data));
    }
}
```

---

## 9. Real-World Framework/Library Examples

### 1. Spring MVC DispatcherServlet

As discussed in Section 7, `DispatcherServlet` is the macro-level mediator in every Spring MVC application. It coordinates `HandlerMapping`, `HandlerAdapter`, `ViewResolver`, `HandlerExceptionResolver`, and `MessageConverters` — none of which know about each other.

**Key source insight:** Look at `DispatcherServlet.doDispatch()` — it is pure mediator logic: receive the request, find the handler, adapt the invocation, resolve the view, handle exceptions. Everything is centrally orchestrated.

### 2. java.util.Timer

`java.util.Timer` is a simple mediator between `TimerTask` instances and thread scheduling. Each `TimerTask` (colleague) simply defines `run()` and registers with the `Timer` (mediator). The `Timer` manages the single background thread, the scheduling queue, and fires each task at the right time. `TimerTask` objects have no knowledge of each other.

```java
Timer timer = new Timer("SchedulerMediator", true); // daemon

// Each TimerTask is a colleague — knows only the Timer interface
TimerTask healthCheck = new TimerTask() {
    public void run() { performHealthCheck(); }
};

TimerTask cacheEviction = new TimerTask() {
    public void run() { evictExpiredCacheEntries(); }
};

// Timer (mediator) coordinates scheduling — tasks don't know about each other
timer.scheduleAtFixedRate(healthCheck, 0, 30_000);
timer.scheduleAtFixedRate(cacheEviction, 5_000, 60_000);
```

### 3. Google Guava EventBus

Guava's `EventBus` is a mediator that decouples event producers from event consumers. Producers post events; consumers subscribe to event types. The `EventBus` acts as the mediator, routing events to appropriate subscribers.

```java
// Guava EventBus as Mediator
EventBus eventBus = new EventBus("ApplicationBus");

// Colleague 1: Producer (posts to mediator)
class OrderService {
    private final EventBus eventBus;

    public OrderService(EventBus eventBus) {
        this.eventBus = eventBus;
    }

    public Order placeOrder(CreateOrderRequest req) {
        Order order = /* ... create order ... */ null;
        eventBus.post(new OrderPlacedEvent(order)); // Sends TO mediator
        return order;
    }
}

// Colleague 2: Consumer (receives FROM mediator)
class InventoryListener {
    @Subscribe
    public void onOrderPlaced(OrderPlacedEvent event) {
        // Reserve inventory — knows nothing about OrderService
        reserveStock(event.getOrder().getProductId(),
                     event.getOrder().getQuantity());
    }
}

// Colleague 3: Another consumer — also knows nothing about OrderService
class EmailNotificationListener {
    @Subscribe
    public void onOrderPlaced(OrderPlacedEvent event) {
        sendConfirmationEmail(event.getOrder().getCustomerEmail());
    }
}

// Registration with the mediator:
eventBus.register(new InventoryListener());
eventBus.register(new EmailNotificationListener());
```

### 4. Java Swing — JButton and ActionListener

`JButton` is a colleague. `ActionListener` implementations are other colleagues. The AWT Event Dispatch Thread + event queue acts as the mediator — collecting events from UI components and dispatching them to registered listeners without the button needing to know what the listener does, or the listener needing to know which button triggered it (beyond what's in the event).

### 5. Apache Camel — CamelContext

Apache Camel's `CamelContext` is a heavyweight enterprise mediator. `Route` definitions declare how messages flow between `Endpoint` colleagues. The `CamelContext` mediates message routing, transformation, error handling, transaction management, and retry logic. Endpoints (Kafka, HTTP, JMS, files) are pure colleagues — they know only the Camel exchange contract.

```java
// Each from/to is a colleague (Endpoint)
// CamelContext is the mediator orchestrating the flow
from("kafka:orders-topic")
    .unmarshal().json(Order.class)
    .choice()
        .when(simple("${body.value} > 1000"))
            .to("direct:high-value-processing")
        .otherwise()
            .to("direct:standard-processing")
    .end()
    .to("jms:queue:processed-orders");
```

### 6. MediatR (C# — conceptual Java equivalent)

While MediatR is C#, the Java ecosystem has equivalents like **axon-framework** and **JMeditr**. The pattern: a `Mediator` interface accepts `Request` objects and dispatches them to the appropriate `Handler`. No controller knows which handler processes its request.

```java
// Conceptual Java MediatR equivalent
interface Request<R> {}
interface RequestHandler<Q extends Request<R>, R> {
    R handle(Q request);
}

class Mediator {
    private final Map<Class<?>, RequestHandler<?, ?>> handlers = new HashMap<>();

    public <R> void registerHandler(
            Class<? extends Request<R>> requestType,
            RequestHandler<? extends Request<R>, R> handler) {
        handlers.put(requestType, handler);
    }

    @SuppressWarnings("unchecked")
    public <R> R send(Request<R> request) {
        RequestHandler<Request<R>, R> handler =
                (RequestHandler<Request<R>, R>) handlers.get(request.getClass());
        if (handler == null) {
            throw new NoHandlerException(request.getClass());
        }
        return handler.handle(request);
    }
}

// Usage in a Spring Controller — controller sends requests, knows no handlers
@RestController
class ProductController {
    private final Mediator mediator;

    @GetMapping("/products/{id}")
    public ProductResponse getProduct(@PathVariable Long id) {
        return mediator.send(new GetProductQuery(id));
    }

    @PostMapping("/products")
    public ProductResponse createProduct(@RequestBody CreateProductCommand cmd) {
        return mediator.send(cmd);
    }
}
```

---

## 10. Trade-offs

### Pros

**1. Reduced coupling between colleagues.**
Components become reusable "plug-ins" to any context that provides a compatible mediator. A `User` colleague from the chat system could be reused in a game lobby or video conference system with a different mediator.

**2. Single place to change interaction logic.**
When the business rule "if a user sends more than 5 messages per second, apply rate limiting" is needed, you add it to the mediator. Without a mediator, this logic might need to go into every message-sending component.

**3. Simplified colleague objects.**
Each colleague only needs to know: "I have something to communicate; I'll tell the mediator." The complexity of routing, sequencing, and coordination is lifted out of the colleagues.

**4. Improved testability.**
You can test a colleague by mocking the mediator interface. You can test the mediator by mocking all colleagues. The interaction logic is concentrated in one class that can be unit tested exhaustively.

**5. Easier to extend with new colleagues.**
Adding a new `PremiumUser` to the chat system only requires implementing `User` and registering with the `ChatRoom`. No existing user types need modification.

**6. Single audit point.**
All interactions flow through the mediator. Logging, security checks, metrics, and rate limiting can be applied centrally without touching colleagues.

### Cons / Trade-offs

**1. The God Object Problem (most serious).**
The mediator concentrates all interaction logic. As the system grows, it can become an enormous, unmaintainable "god class" that violates the Single Responsibility Principle. A `ChatRoom` mediator that handles routing, moderation, content filtering, analytics, and presence management is doing too much.

*Mitigation:* Extract focused sub-mediators. Separate `ContentModerationMediator`, `PresenceMediator`, and `RoutingMediator` that collaborate, or use the Chain of Responsibility inside the mediator.

**2. Mediator complexity increases with colleague complexity.**
The more types of colleagues and the more interactions between them, the more conditional logic accumulates in the mediator. The complexity doesn't disappear — it moves.

**3. Single point of failure.**
If the mediator fails, all communication between colleagues fails. In distributed systems this is managed with redundancy, but in-process mediators have no such protection. A bug in the mediator affects the entire interaction surface.

**4. Potential performance bottleneck.**
Every inter-colleague communication passes through the mediator. In high-throughput scenarios (game engines, real-time systems), this indirection adds latency and creates a throughput ceiling.

**5. Obscures direct relationships.**
Developers reading colleague code cannot easily determine which other colleagues their actions affect. They must read the mediator to understand the full interaction graph. With direct references, IDE "Find Usages" makes dependencies explicit.

**6. Interface rigidity.**
If the mediator interface is too broad (a single `notify(Object sender, String event)` method), it devolves into stringly-typed communication. If it's too narrow, adding new interaction types requires interface changes that break all implementations.

### The God Object Mitigation Pattern

When the mediator grows too large, decompose by domain:

```
Large ChatRoomMediator
         ↓ decompose
┌────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐
│  RoutingMediator   │  │ ModerationMediator   │  │ PresenceMediator │
│  - Routes messages │  │ - Bans/mutes/warns   │  │ - Online status  │
│  - Channel mgmt    │  │ - Content filtering  │  │ - Typing indic.  │
│  - Private msgs    │  │ - Spam detection     │  │ - Last seen      │
└────────────────────┘  └──────────────────────┘  └──────────────────┘
        ↑                         ↑                        ↑
        └──────────── All implement ChatMediator ──────────┘
                    and are composed by ChatFacade
```

---

## 11. Common Interview Discussion Points

### Point 1: Why "Define an object that encapsulates HOW objects interact"?

The GoF definition's emphasis on *how* is deliberate. The mediator doesn't encapsulate what objects *are* or what they *do* — it encapsulates the *protocol* of their interaction. This is why the mediator can vary independently of the colleagues: you can change the routing logic (how messages are delivered) without changing what a User or Aircraft *is*.

### Point 2: Mediator vs a Service Class

A common interview trap: "Isn't a mediator just a service class that calls other services?" The distinction:
- A **service** coordinates *behavior* (execute a business operation).
- A **mediator** coordinates *communication* (route messages between participants).
- A service typically has a clear return value and is invoked for its result.
- A mediator typically has a `notify()` or `send()` method and works by dispatching to colleagues.
- Services are not usually registered with by their collaborators; mediators typically are.

In practice, the lines blur. Many application services *are* mediators. The distinction matters most when discussing architecture.

### Point 3: Thread Safety in Mediators

Mediators are frequently shared across threads (multiple users sending messages concurrently; multiple aircraft requesting runway access simultaneously). Key considerations:
- **Colleague registry** must use `ConcurrentHashMap`, not `HashMap`.
- **Queues** for landing/takeoff sequences must be thread-safe (`LinkedBlockingQueue` over `LinkedList`).
- **Notification** of colleagues: if a colleague's `receive()` method has side effects, notifications may need to be dispatched on a dedicated event loop or executor.
- **Mediator state** (runway occupancy, mute/ban lists) must be guarded by synchronized blocks or atomic operations.

### Point 4: Event-Driven vs Direct-Call Mediators

Two implementation flavors exist:
- **Synchronous, direct-call:** `mediator.send(message)` → mediator calls `colleague.receive(message)` in the same thread. Simple, easy to reason about, but can cause long call stacks.
- **Asynchronous, event-driven:** colleagues post events to a queue; the mediator processes them on a dedicated thread. More resilient, naturally decoupled in time, but harder to debug (stack traces don't cross the async boundary).

### Point 5: Composing Multiple Mediators

Large systems often have a hierarchy of mediators. The ATC analogy maps perfectly: Ground Control, Tower, Approach, En-Route Center are all mediators, each responsible for a sector of airspace. Colleagues (aircraft) interact with whichever mediator governs their current phase of flight, and mediators coordinate with adjacent mediators (handing off an aircraft from Tower to Departure Control).

### Point 6: CQRS + Mediator

The MediatR library (popular in C#) and its Java equivalents apply the Mediator pattern to implement CQRS (Command Query Responsibility Segregation). Each `Command` and `Query` is a request object dispatched through the mediator to a handler. This makes it trivially easy to add cross-cutting concerns (logging, validation, caching) at the mediator level without touching individual handlers. This is increasingly adopted in Java microservices as an alternative to service-to-service direct calls.

---

## 12. Interview Questions and Model Answers

---

### Q1. Explain the Mediator pattern in one sentence and give an example from production code you've seen.

**Model Answer:**

The Mediator pattern replaces a web of many-to-many component dependencies with a single hub object that encapsulates how those components interact — collapsing O(n²) connections into O(n).

A production example I'd point to is Spring MVC's `DispatcherServlet`. Every HTTP request arrives at the `DispatcherServlet`, which mediates between `HandlerMapping` (which controller handles this?), `HandlerAdapter` (how do I invoke it?), `ViewResolver` (how do I render the result?), and `HandlerExceptionResolver` (what if something throws?). None of these components know about each other; they all collaborate through the `DispatcherServlet` mediator. You can swap out the `ViewResolver` for a different template engine without changing the `HandlerMapping` or any other participant — the hallmark of the Mediator pattern working correctly.

---

### Q2. What is the "god object" problem with Mediator, and how do you mitigate it?

**Model Answer:**

The god object problem occurs when the mediator becomes the only class that knows everything and does everything. As colleagues multiply and their interactions grow more complex, all the routing logic, validation rules, state management, and business decisions pile into the mediator. It violates the Single Responsibility Principle: the mediator is now responsible for message routing, moderation, content filtering, presence management, analytics, rate limiting, and more.

The signs to watch for:
- Mediator exceeds 300-400 lines and is still growing
- Adding a new colleague type requires changing the mediator
- The mediator has multiple unrelated responsibilities (it's doing content moderation AND analytics AND routing)

**Mitigation strategies:**

1. **Decompose by domain.** Split `ChatRoomMediator` into `RoutingMediator`, `ModerationMediator`, and `PresenceMediator`. Each handles one concern and is composed into a facade.

2. **Chain of Responsibility inside the mediator.** The mediator's routing logic becomes a pipeline of handlers: `ContentFilter → SpamDetector → RateLimiter → Router`. Each is independently testable and removable.

3. **Strategy pattern for routing rules.** The mediator uses pluggable strategy objects for different routing decisions. The mediator stays thin; the routing logic is in the strategies.

4. **Sector-based decomposition (ATC model).** Multiple focused mediators, each handling a subset of colleagues. Mediators coordinate with each other at handoff boundaries.

5. **Event sourcing + projections.** Rather than the mediator tracking all state, colleagues emit events to an event log. The mediator is a thin router; state is reconstructed from the event log as needed.

---

### Q3. How does Mediator differ from Facade? Many candidates confuse these.

**Model Answer:**

This is a very common confusion because both patterns present a simplified interface to a complex system. The difference is structural and intentional:

| Dimension | Facade | Mediator |
|-----------|--------|----------|
| **Direction** | One-way: client calls facade, facade calls subsystem | Two-way: colleagues communicate through mediator |
| **Subsystem knowledge** | Facade knows subsystem; subsystem doesn't know facade | Mediator knows colleagues; colleagues know mediator |
| **Purpose** | Simplify a client's view of a complex API | Reduce coupling between objects that must interact |
| **Colleague registration** | Subsystems are not "registered" with the facade | Colleagues register/deregister with the mediator |
| **Typical direction** | Client → Facade → Subsystem (one direction) | ColleagueA → Mediator → ColleagueB → Mediator → ColleagueA |

**Concrete analogy:** A hotel concierge desk (Facade) simplifies booking restaurants, arranging transportation, and recommending sights — but the restaurants don't call the concierge. An Air Traffic Control tower (Mediator) has all aircraft actively communicating through it, both sending and receiving instructions.

**Code distinction:**
```java
// Facade: Subsystem has no reference to facade
class VideoConversionFacade {
    // Subsystems (VideoFile, Codec, BitrateReader) don't know this class exists
    public File convertVideo(String fileName, String format) {
        VideoFile file = new VideoFile(fileName);
        Codec codec = CodecFactory.extract(file);
        // ...
    }
}

// Mediator: Colleagues hold a reference to the mediator
abstract class User {
    protected ChatMediator mediator; // Explicit reference — colleagues know the mediator
    public void send(Message m) {
        mediator.sendMessage(m, this); // User calls mediator
    }
}
```

---

### Q4. Compare Mediator and Observer. When would you use one over the other in a microservices event-driven architecture?

**Model Answer:**

In a microservices event-driven architecture, this question becomes critically practical.

**Observer (event-driven, no central coordinator):**
- Service A emits a `UserRegistered` event to a Kafka topic.
- Service B (Email), Service C (Analytics), Service D (Fraud Detection) each independently consume and react.
- No central service knows what happens downstream.
- Adding Service E (Loyalty) is zero-touch to existing services — it subscribes to the same topic.
- **Best for:** fan-out events where each subscriber's reaction is independent, additive, and doesn't affect others.

**Mediator (orchestrator pattern in microservices):**
- Service A (Order) wants to execute a checkout: reserve inventory, charge payment, trigger fulfillment, send confirmation.
- An Orchestrator service (the mediator) calls each downstream service in order, handles failures with compensating transactions, retries, and escalates to a dead-letter queue on exhaustion.
- **Best for:** multi-step workflows where the order of operations matters, failure in one step must roll back previous steps, and the overall workflow logic must be auditable.

**The practical rule:**
- **Observer/Choreography:** Use when adding downstream reactions should require zero changes to the producing service. Appropriate for analytics, notifications, audit logs.
- **Mediator/Orchestration:** Use when a business transaction spans multiple services and requires ACID-like guarantees (Saga pattern), compensating transactions, or explicit sequencing. AWS Step Functions, Apache Camel, and Spring Batch Orchestrator are all mediators in this sense.

**Hybrid:** In practice, a Saga orchestrator (mediator) may publish events (Observer) to notify downstream systems of the saga's overall outcome, while internally using direct calls (mediated) for the saga steps themselves.

---

### Q5. Design a mediator for a multiplayer game's lobby system. What are the key design decisions?

**Model Answer:**

A game lobby mediator handles: player joining/leaving, ready state changes, game start conditions, team balancing, and transitioning players from lobby to game session.

**Key participants:**
- `LobbyMediator` (interface): `join(Player)`, `leave(String playerId)`, `setReady(String playerId, boolean ready)`, `sendChat(String message, String fromId)`
- `GameLobby` (concrete mediator): knows all `Player` objects in the lobby
- `Player` (colleague): knows the `LobbyMediator`, sends events, receives state updates
- `HumanPlayer` and `BotPlayer` (concrete colleagues)

**Key design decisions:**

1. **Game start trigger logic lives in the mediator.** When every player sets ready=true, the mediator triggers `GameSessionFactory.create(lobbyState)`. No individual Player triggers the game start — the mediator decides when the condition is met.

2. **Team balancing logic belongs in the mediator.** When a player joins, the mediator assigns them to a team based on ELO, team sizes, or other rules. Players don't negotiate team assignment among themselves.

3. **Synchronization of lobby state.** The mediator is the single source of truth for who is in the lobby, who is ready, and what the current configuration is. All players receive the same consistent state snapshot.

4. **Mediator broadcasts vs. targeted notifications.** Chat messages broadcast to all; team assignments go to specific players; kick notifications go to the kicked player only. The mediator handles all routing.

5. **Thread safety.** Players can join/leave concurrently. The `players` map must be `ConcurrentHashMap`. The "all ready" check must be atomic with respect to the map contents (use `synchronized` or `StampedLock`).

6. **Lobby lifecycle.** The mediator manages the state machine: WAITING → READY → STARTING → IN_GAME. State transitions are validated and enforced in the mediator.

```java
interface LobbyMediator {
    void join(Player player);
    void leave(String playerId);
    void setReady(String playerId, boolean ready);
    void sendChat(ChatMessage message);
    void kickPlayer(String requesterId, String targetId);
}
```

---

### Q6. How would you implement the Mediator pattern in a reactive/non-blocking system using Project Reactor?

**Model Answer:**

In a reactive system, the synchronous `void receive(Message msg)` callback model doesn't work well. We need non-blocking, backpressure-aware message delivery.

**Approach 1: Sinks-based Mediator**

Each colleague subscribes to a `Sinks.Many<Message>` (a reactive broadcast sink in Project Reactor). The mediator holds a sink per colleague (or per channel) and emits to the appropriate sinks when routing messages.

```java
class ReactiveChatRoom {
    // One sink per user — the mediator holds all sinks
    private final Map<String, Sinks.Many<Message>> userSinks =
            new ConcurrentHashMap<>();

    public void registerUser(String userId) {
        userSinks.put(userId,
                Sinks.many().multicast().onBackpressureBuffer());
    }

    // Returns a Flux that the user's WebSocket handler subscribes to
    public Flux<Message> messagesFor(String userId) {
        return userSinks.get(userId).asFlux();
    }

    // Mediator routing logic — still synchronous internally, but
    // emission is non-blocking
    public Mono<Void> sendMessage(Message message) {
        return Mono.fromRunnable(() -> {
            switch (message.getType()) {
                case BROADCAST -> userSinks.values()
                        .forEach(sink -> sink.tryEmitNext(message));
                case PRIVATE -> {
                    Sinks.Many<Message> recipientSink =
                            userSinks.get(message.getRecipientId());
                    if (recipientSink != null) {
                        recipientSink.tryEmitNext(message);
                    }
                }
            }
        }).subscribeOn(Schedulers.boundedElastic());
    }
}
```

**Approach 2: FluxProcessor or reactor-core event bus**

Use `reactor.core.publisher.EmitterProcessor` (deprecated in favor of `Sinks`) or a `ConnectableFlux` as the backbone, with the mediator filtering and routing events.

**Key reactive considerations:**
- Use `Sinks.Many.multicast()` for fan-out to multiple subscribers.
- Use `Schedulers.boundedElastic()` for blocking I/O within mediator callbacks.
- Handle `Sinks.EmitResult` failure cases (subscriber dropped, buffer overflow).
- Backpressure: if a slow subscriber falls behind, the mediator must decide to drop messages, buffer them, or apply rate limiting.

---

### Q7. What are the SOLID principle implications of the Mediator pattern?

**Model Answer:**

**Single Responsibility Principle (SRP):**
- **Positive:** Each colleague has one responsibility (e.g., `User` handles its own display and input). Interaction logic is moved out of colleagues into the mediator.
- **Negative:** The mediator itself often accumulates multiple responsibilities (routing, validation, moderation, analytics) — violating SRP at the mediator level. This is the god object problem.

**Open/Closed Principle (OCP):**
- **Positive:** Adding a new colleague type (e.g., `PremiumUser`) doesn't require modifying existing colleagues — they're unaware of each other.
- **Negative:** Adding a new *type of interaction* often requires modifying the mediator. If new colleagues have new interaction semantics, the mediator must change. This is a partial OCP violation. Mitigated by designing the mediator to be extensible (Strategy or Chain of Responsibility internally).

**Liskov Substitution Principle (LSP):**
- Applied cleanly. `RegularUser`, `AdminUser`, and `BotUser` all substitute for `User`. `ChatRoom` can be substituted by `EnterpriseChannelRoom` that implements `ChatMediator` differently.

**Interface Segregation Principle (ISP):**
- The `ChatMediator` interface should not force colleagues to depend on methods they don't use. `AdminUser` needs `applyModeration()`, but regular `User` doesn't call it (though the mediator does). If the mediator interface is too broad, ISP is violated. Consider splitting into `RoutingMediator` and `ModerationMediator`.

**Dependency Inversion Principle (DIP):**
- Correctly applied: colleagues depend on the `ChatMediator` *interface*, not on `ChatRoom` (concrete class). The concrete mediator is injected at construction time. High-level modules (colleagues) depend on abstractions (mediator interface).

---

### Q8. A system has 10 microservices that all need to communicate with each other. Should you use the Mediator pattern? How would you design it?

**Model Answer:**

Yes, but the Mediator pattern at the microservices level is called an **API Gateway + Orchestration Service** pattern, and the design is substantially different from in-process mediators. Let me separate two concerns:

**Case 1: All 10 services need to share data/events (event-driven)**

Use a message broker (Kafka, RabbitMQ) as the mediator. Each service publishes domain events; the broker routes them to subscribers. This is the Observer/Choreography pattern — the broker is an infrastructure-level mediator. Services remain decoupled.

**Case 2: Multi-step business workflows spanning multiple services**

This is where the Mediator pattern applies explicitly. Design an **Orchestrator Service** (saga orchestrator):

```
┌─────────────────────────────────────────┐
│          OrderOrchestrator              │ ← Mediator (microservice)
│  ┌─────────────────────────────────┐   │
│  │ executeCheckout(CheckoutRequest)│   │
│  │   1. inventoryClient.reserve()  │   │
│  │   2. paymentClient.charge()     │   │
│  │   3. fulfillmentClient.ship()   │   │
│  │   4. notificationClient.send()  │   │
│  │ with compensating transactions  │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
        ↓         ↓          ↓        ↓
  Inventory  Payment  Fulfillment  Notification
  Service    Service    Service     Service
  (colleague)(colleague)(colleague)(colleague)
```

**Key design decisions:**

1. **Each service is a colleague, unaware of other services.** `InventoryService` doesn't know `PaymentService` exists. Only the orchestrator knows the workflow.

2. **Compensating transactions.** The mediator must implement rollback: if payment fails, call `inventoryClient.release()`. This is the Saga pattern.

3. **Durable execution.** Use a workflow engine (Temporal.io, AWS Step Functions, Conductor) rather than handrolling the orchestrator. These provide durability, retry, observability, and replay out of the box.

4. **API Gateway as a separate, infrastructure-level mediator.** The gateway handles authentication, rate limiting, and routing at the perimeter. The orchestrator handles business workflow. Don't conflate them.

5. **Avoiding the god-service anti-pattern.** Don't route ALL inter-service communication through the orchestrator. Only workflows with cross-service transactions need it. Peer-to-peer calls for simple lookups are fine.

**Interview follow-up answer:** For pure read operations and loosely coupled event notifications, prefer event-driven (Observer/Choreography). Reserve explicit orchestration mediators for workflows where ordering, compensation, and auditability matter.

---

## 13. Comparison with Related Patterns

### Mediator vs Facade

| Aspect | Mediator | Facade |
|--------|----------|--------|
| **Mutual awareness** | Colleagues know mediator; mediator knows colleagues | Subsystems don't know the facade |
| **Communication flow** | Bidirectional | Unidirectional (client → facade → subsystem) |
| **Primary goal** | Eliminate coupling between peers | Simplify complex API for clients |
| **Colleagues register** | Yes — actively register/deregister | No — facade directly calls subsystem methods |
| **Example** | ChatRoom, ControlTower, DispatcherServlet | `VideoConversionFacade`, `HomeTheaterFacade` |

**Analogy:** Facade is a hotel reception desk — guests use it to access hotel services, but hotel services don't report back to the desk. Mediator is an ATC tower — aircraft actively communicate with it, and it actively instructs aircraft.

### Mediator vs Observer

| Aspect | Mediator | Observer |
|--------|----------|----------|
| **Coordination** | Centralized in mediator | Decentralized — each observer acts independently |
| **Publisher awareness** | Mediator knows all colleagues | Subject knows observers only by interface |
| **Subscriber interaction** | Mediator can make one subscriber's action depend on another's | Observers are completely independent |
| **Event routing** | Mediator decides who gets what | All observers of a type get all notifications |
| **Complexity** | In mediator (explicit) | Distributed across observers (implicit) |
| **Debugging** | Trace interaction in one place | Must trace across all observer implementations |

### Mediator vs Command

| Aspect | Mediator | Command |
|--------|----------|---------|
| **Purpose** | Route communication between objects | Encapsulate a request as an object |
| **Who invokes** | Mediator decides who acts | Command is invoked by an invoker |
| **Undo support** | Not intrinsic | Built-in (store command history) |
| **Combination** | Mediator can dispatch Command objects | Commands can call mediator |

The Command and Mediator patterns compose naturally: the mediator receives `Command` objects from colleagues and executes them, providing undo/redo capability to the interaction coordination layer.

### Mediator vs Chain of Responsibility

| Aspect | Mediator | Chain of Responsibility |
|--------|----------|------------------------|
| **Routing** | Mediator actively routes to specific recipients | Each handler decides to handle or pass |
| **Coupling** | Mediator knows all potential recipients | Handlers are linked in a chain |
| **Flexibility** | Mediator controls routing centrally | Chain can be reconfigured dynamically |
| **Use together** | Mediator internally uses COR for processing pipeline | — |

**Combination pattern:** The mediator is the entry point; internally it applies a Chain of Responsibility for processing (content filter → spam check → rate limiter → router). The colleagues see the mediator as a monolith; internally the mediator delegates through a chain.

### Summary Table: Which Pattern When?

| Scenario | Pattern |
|----------|---------|
| Many objects need to notify others of state changes without knowing who cares | Observer |
| Many objects need to be coordinated and their interactions have business logic | Mediator |
| Client needs a simplified view of a complex subsystem | Facade |
| Request must be processed by a series of handlers, each deciding whether to pass | Chain of Responsibility |
| Request needs to be encapsulated for queuing, logging, or undo | Command |
| Objects communicate through a mediator AND need undo/redo | Mediator + Command |
| Notification fans out AND recipients must be coordinated | Mediator + Observer |

---

## Summary: Key Takeaways for Interview and Production

1. **Mediator = centralized interaction coordinator.** It converts O(n²) peer dependencies into O(n) mediator dependencies.

2. **The ATC analogy is definitive.** Every aircraft talks to the tower, never to each other. The tower decides what happens based on global state awareness.

3. **The god object is the primary failure mode.** The pattern can degenerate into an unmaintainable monolith if not actively governed. Decompose by domain concern.

4. **Mediator vs Observer is the most important nuance.** Observer decentralizes independent reactions to state change. Mediator centralizes coordinated interaction logic.

5. **MVC is Mediator in disguise.** The Controller mediates Model and View. Spring's DispatcherServlet mediates the entire request processing pipeline.

6. **In microservices, Mediator = Saga Orchestrator.** Explicit workflow coordination with compensating transactions — but only where needed. Prefer choreography (Observer) for independent event reactions.

7. **Thread safety is not optional.** Any production mediator in a concurrent environment must use thread-safe collections and atomic state transitions.

8. **Test the mediator exhaustively.** Because all interaction logic lives there, the mediator must have near-100% branch coverage. Colleagues are straightforward to test because they only interact through a mockable interface.

---

*Pattern #16 of 23 in the Gang of Four behavioral patterns.*
*Related files: `06-command.md`, `08-memento.md`, `10-observer.md`, `11-state.md`*
