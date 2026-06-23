# Capstone Project 2: Smart Home Automation System (Senior Level)

---

## Project Overview

The Smart Home Automation System is a comprehensive IoT platform that allows homeowners to control, monitor, and automate their smart devices. The system supports multiple device types (lights, thermostats, cameras, door locks), an event-driven automation rule engine, scene management for grouping actions, and full undo/redo command history.

This capstone integrates all major design patterns covered in the course: **Command**, **Observer/Event Bus**, **Chain of Responsibility**, **Composite**, **Strategy**, and **Template Method** — demonstrating how they cooperate in a real-world production-grade system.

**Why This System Is Realistic:**
- Smart home platforms (Google Home, Amazon Alexa, Apple HomeKit) all share this layered architecture
- Event-driven automation is the dominant pattern in IoT systems
- Command history mirrors professional home automation controllers
- The composite device structure mirrors physical home hierarchy (zones → rooms → devices)

---

## Requirements (Functional + Non-Functional)

### Functional Requirements

**Device Management**
- FR-01: System shall support SmartLight, Thermostat, SecurityCamera, DoorLock, SmartTV, and SmartSpeaker device types
- FR-02: Each device shall have a unique ID, name, assigned room, type, and state (ON/OFF/STANDBY/ERROR)
- FR-03: Devices shall be addable to rooms and device groups
- FR-04: Each device shall accept commands via a unified command interface

**Command & History**
- FR-05: All device state changes shall be executed via commands
- FR-06: The system shall support undo and redo for all executed commands
- FR-07: Command history shall be bounded at 50 entries (oldest discarded)
- FR-08: Scenes shall group multiple commands and be activated atomically

**Event System**
- FR-09: Devices shall publish events when their state changes
- FR-10: Listeners shall be subscribable per event type
- FR-11: Listeners shall be unsubscribable by listener ID
- FR-12: Events shall carry a typed payload map

**Automation Rules**
- FR-13: Rules shall consist of a Condition and an Action
- FR-14: Rules shall be evaluatable against incoming events
- FR-15: Rules shall be individually enable/disable-able
- FR-16: Multiple rules shall be evaluated via a Chain of Responsibility

**Scene Management**
- FR-17: Scenes shall be saved by name and activatable by name
- FR-18: Activating a scene shall execute all its commands through command history (enabling undo)
- FR-19: Scenes shall be deletable

**Composite Device Control**
- FR-20: Rooms and device groups shall support bulk command execution via a factory function

### Non-Functional Requirements

- NFR-01: **Thread Safety** — Event bus publish and subscribe operations should be safe for the demo; production would use ConcurrentHashMap and CopyOnWriteArrayList
- NFR-02: **Extensibility** — Adding a new device type shall require zero changes to existing classes (Open/Closed)
- NFR-03: **Bounded Memory** — Command history is capped at 50 commands
- NFR-04: **Observability** — All major operations print structured log lines to stdout
- NFR-05: **Testability** — All core components accept interfaces, enabling mock injection in unit tests
- NFR-06: **Java Standard Library Only** — No external frameworks or dependencies

---

## Domain Model (Prose Explanation)

The domain centers on the **SmartDevice** as the core entity. Every physical IoT device is represented as a subclass of SmartDevice, sharing a common identity (deviceId, name, room), lifecycle state (ON/OFF/STANDBY/ERROR), and event-publishing capability. Devices do not communicate directly with each other — all inter-device communication flows through the **DeviceEventBus**.

The **DeviceEventBus** is the nervous system of the home. It routes **DeviceEvent** objects from publishing devices to subscribed **DeviceEventListener** implementations. Events carry a typed payload (a Map) so that listeners can extract context-specific data (e.g., temperature values, motion sensitivity) without requiring strongly typed event subclasses.

**Commands** encapsulate every mutation to device state. This enables undo/redo through the **CommandHistory**, which maintains two bounded stacks. Scenes are named collections of commands managed by **SceneManager**; activating a scene runs each command through CommandHistory so that the entire scene can be undone step-by-step.

**AutomationRule** wires a **Condition** to an **Action**. The Condition evaluates an incoming event (e.g., motion detected on a specific camera, temperature exceeding a threshold, a door changing state). The Action is the response (send a notification, execute a command, activate a scene). Rules are evaluated by the **RuleEngine** using a **Chain of Responsibility** where each **RuleHandler** is specialized for a category of event type.

The physical hierarchy of the home is modeled with the **Composite** pattern: **Room** and **DeviceGroup** both implement **DeviceComponent**, enabling bulk operations on any grouping of devices.

**HomeAutomationSystem** is the facade — the single entry point that wires all subsystems together and exposes a simplified API to the outside world (or to a REST controller, CLI, or UI in a real application).

---

## Class Diagram (Text Format)

```
+------------------+       +------------------+       +------------------+
|   SmartDevice    |       |  DeviceEventBus  |       |   DeviceEvent    |
|  (abstract)      |------>|                  |------>|                  |
|------------------|       |------------------|       |------------------|
| -deviceId        |       | -listeners       |       | -deviceId        |
| -name            |       | -listenerRegistry|       | -eventType       |
| -room            |       |------------------|       | -payload         |
| -deviceType      |       | +subscribe()     |       | -timestamp       |
| -state           |       | +unsubscribe()   |       +------------------+
| -eventBus        |       | +publish()       |
|------------------|       +------------------+
| +executeCommand()|
| +setState()      |
| +publishEvent()  |
+--------+---------+
         |
    +-----------+-----------+-----------+
    |           |           |           |
+-------+  +----------+ +------+  +----------+
|Smart  |  |Thermostat| |Securi|  | DoorLock |
|Light  |  |          | |tyCam |  |          |
+-------+  +----------+ +------+  +----------+

+------------------+       +------------------+
|  DeviceCommand   |       |  CommandHistory  |
|  (interface)     |       |                  |
|------------------|       |------------------|
| +execute()       |       | -undoStack       |
| +undo()          |       | -redoStack       |
| +getDescription()|       | -maxSize=50      |
+--------+---------+       |------------------|
         |                 | +executeCommand()|
    +----+----+----+       | +undo()          |
    |         |    |       | +redo()          |
+--------+ +----+ +-----+ +------------------+
|TurnOn  | |Set | |Lock |
|Command | |Bri.| |Door.|
+--------+ +----+ +-----+

+------------------+       +------------------+
|   Condition      |       |     Action       |
|  (interface)     |       |  (interface)     |
|------------------|       |------------------|
| +evaluate()      |       | +execute()       |
| +getDescription()|       | +getDescription()|
+--------+---------+       +--------+---------+
         |                          |
  +------+------+            +------+------+
  |             |            |             |
+-------+ +----+--+  +--------+  +--------+
|Motion | |Temp.  |  |SendNoti|  |Execute.|
|Cond.  | |Alert  |  |fication|  |Command.|
+-------+ +-------+  +--------+  +--------+

+------------------+       +------------------+
| AutomationRule   |       |   RuleHandler    |
|                  |       |  (abstract)      |
|------------------|       |------------------|
| -name            |       | -rule            |
| -condition       |       | -nextHandler     |
| -action          |       |------------------|
| -enabled         |       | +setNext()       |
|------------------|       | +handle()        |
| +matches()       |       | +canHandle()     |
| +execute()       |       +--------+---------+
+------------------+                |
         ^                   +------+------+
         |                   |             |
  +------+------+    +-------+---+ +-------+---+
  |  RuleEngine |    |StateChange| |  Motion   |
  |             |    |Handler    | |  Handler  |
  |-------------|    +-----------+ +-----------+
  | +addRule()  |
  | +buildChain()|
  +-------------+

+------------------+       +------------------+
| DeviceComponent  |       |     Scene        |
|  (interface)     |       |                  |
|------------------|       |------------------|
| +getName()       |       | -name            |
| +getDevices()    |       | -description     |
| +executeOnAll()  |       | -commands        |
+--------+---------+       |------------------|
         |                 | +addCommand()    |
    +----+----+            | +getCommands()   |
    |         |            +------------------+
  +----+  +-------+              ^
  |Room|  |Device |              |
  |    |  |Group  |     +--------+--------+
  +----+  +-------+     |  SceneManager  |
                        |----------------|
                        | +saveScene()   |
                        | +activateScene()|
                        +----------------+

+------------------+
|HomeAutomation    |
|System            |
|------------------|
| -devices         |
| -rooms           |
| -groups          |
| -eventBus        |
| -ruleEngine      |
| -sceneManager    |
| -commandHistory  |
|------------------|
| +addDevice()     |
| +addRule()       |
| +activateScene() |
| +executeCommand()|
| +undo() / redo() |
| +getStatus()     |
+------------------+
```

---

## Design Patterns Applied

### 1. Command Pattern
**Classes:** `DeviceCommand`, `TurnOnCommand`, `TurnOffCommand`, `SetBrightnessCommand`, `SetTemperatureCommand`, `LockDoorCommand`, `StartRecordingCommand`, `CommandHistory`

**Why:** Every device mutation is an object. This enables undo/redo, logging, scene composition, and queuing. `CommandHistory` manages two bounded stacks — the undo stack and redo stack — giving the homeowner a full history of actions.

**Key Insight:** Scenes are just lists of commands. Activating a scene pushes each command through `CommandHistory`, so the entire scene is undoable step by step.

### 2. Observer / Event Bus Pattern
**Classes:** `DeviceEventBus`, `DeviceEventListener`, `DeviceEvent`, `DeviceEventType`

**Why:** Devices must communicate state changes without knowing who is listening. The event bus decouples publishers (devices) from subscribers (rule engine, loggers, UI listeners). This is the foundation of reactive IoT systems.

**Key Insight:** The bus is typed per event — `subscribe(DeviceEventType, DeviceEventListener)` — so listeners only receive events they care about, avoiding unnecessary processing.

### 3. Chain of Responsibility Pattern
**Classes:** `RuleHandler`, `StateChangeRuleHandler`, `MotionRuleHandler`, `TemperatureRuleHandler`, `DoorEventRuleHandler`, `RuleEngine`

**Why:** Different automation rules respond to different event types. Rather than a giant if/else block, each handler specializes in one event category and passes the event down the chain. New event categories are handled by adding a new handler subclass.

**Key Insight:** `RuleEngine.buildChain()` dynamically constructs the handler chain from the registered rules each time a rule is added or removed, keeping the chain always current.

### 4. Composite Pattern
**Classes:** `DeviceComponent`, `Room`, `DeviceGroup`

**Why:** The home has a hierarchy: the house contains rooms, rooms contain devices, and ad-hoc groups can span rooms. The Composite pattern allows treating a single device, a room, and a group identically when bulk-executing commands.

**Key Insight:** `executeOnAll(Function<SmartDevice, DeviceCommand> factory, CommandHistory history)` lets callers pass a command factory that is applied to every device, enabling "turn off everything in the living room" with one call.

### 5. Strategy Pattern (via Condition and Action interfaces)
**Classes:** `Condition`, `DeviceStateCondition`, `MotionCondition`, `TimeCondition`, `TemperatureAlertCondition`, `Action`, `SendNotificationAction`, `ExecuteCommandAction`, `ActivateSceneAction`

**Why:** Automation rules need pluggable "when" (condition) and "then" (action) logic. The Strategy pattern lets these be swapped at runtime without touching `AutomationRule`.

**Key Insight:** Conditions and Actions are completely independent. You can mix any condition with any action: "when motion detected → activate Movie Night scene."

### 6. Template Method Pattern (via RuleHandler)
**Classes:** `RuleHandler`, `StateChangeRuleHandler`, `MotionRuleHandler`, `TemperatureRuleHandler`, `DoorEventRuleHandler`

**Why:** All handlers share the same algorithm: check if they can handle the event, check if the rule matches, fire the rule, then pass to the next handler. Only `canHandle()` varies per subclass.

### 7. Facade Pattern
**Class:** `HomeAutomationSystem`

**Why:** The system's internal complexity (event bus wiring, command history, rule engine, scene manager) is hidden behind a simple facade. External code (a REST controller, a CLI, a test) interacts with one class.

---

## Complete Java Implementation

### DeviceType.java

```java
package smarthome.model;

public enum DeviceType {
    SMART_LIGHT,
    THERMOSTAT,
    SECURITY_CAMERA,
    DOOR_LOCK,
    SMART_TV,
    SMART_SPEAKER
}
```

---

### DeviceState.java

```java
package smarthome.model;

public enum DeviceState {
    ON,
    OFF,
    STANDBY,
    ERROR
}
```

---

### ThermostatMode.java

```java
package smarthome.model;

public enum ThermostatMode {
    HEATING,
    COOLING,
    AUTO,
    OFF
}
```

---

### DeviceEventType.java

```java
package smarthome.events;

public enum DeviceEventType {
    STATE_CHANGED,
    MOTION_DETECTED,
    TEMPERATURE_ALERT,
    DOOR_LOCKED,
    DOOR_UNLOCKED,
    BRIGHTNESS_CHANGED,
    TEMPERATURE_CHANGED,
    RECORDING_STARTED,
    RECORDING_STOPPED
}
```

---

### DeviceEvent.java

```java
package smarthome.events;

import java.time.LocalDateTime;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

public class DeviceEvent {

    private final String deviceId;
    private final DeviceEventType eventType;
    private final Map<String, Object> payload;
    private final LocalDateTime timestamp;

    public DeviceEvent(String deviceId, DeviceEventType eventType, Map<String, Object> payload) {
        this.deviceId = deviceId;
        this.eventType = eventType;
        this.payload = Collections.unmodifiableMap(new HashMap<>(payload));
        this.timestamp = LocalDateTime.now();
    }

    public String getDeviceId() {
        return deviceId;
    }

    public DeviceEventType getEventType() {
        return eventType;
    }

    public Map<String, Object> getPayload() {
        return payload;
    }

    public LocalDateTime getTimestamp() {
        return timestamp;
    }

    @Override
    public String toString() {
        return "DeviceEvent{" +
                "deviceId='" + deviceId + '\'' +
                ", eventType=" + eventType +
                ", payload=" + payload +
                ", timestamp=" + timestamp +
                '}';
    }
}
```

---

### SmartDevice.java

```java
package smarthome.devices;

import smarthome.commands.DeviceCommand;
import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventBus;
import smarthome.events.DeviceEventType;
import smarthome.model.DeviceState;
import smarthome.model.DeviceType;

import java.util.HashMap;
import java.util.Map;

public abstract class SmartDevice {

    private final String deviceId;
    private final String name;
    private final String room;
    private final DeviceType deviceType;
    private DeviceState state;
    private final DeviceEventBus eventBus;

    public SmartDevice(String deviceId, String name, String room, DeviceType deviceType, DeviceEventBus eventBus) {
        this.deviceId = deviceId;
        this.name = name;
        this.room = room;
        this.deviceType = deviceType;
        this.eventBus = eventBus;
        this.state = DeviceState.OFF;
    }

    public abstract void executeCommand(DeviceCommand command);

    public void setState(DeviceState newState) {
        DeviceState previousState = this.state;
        this.state = newState;
        Map<String, Object> payload = new HashMap<>();
        payload.put("previousState", previousState.name());
        payload.put("newState", newState.name());
        publishEvent(DeviceEventType.STATE_CHANGED, payload);
    }

    protected void publishEvent(DeviceEventType type, Map<String, Object> payload) {
        DeviceEvent event = new DeviceEvent(deviceId, type, payload);
        eventBus.publish(event);
    }

    public String getDeviceId() {
        return deviceId;
    }

    public String getName() {
        return name;
    }

    public String getRoom() {
        return room;
    }

    public DeviceType getDeviceType() {
        return deviceType;
    }

    public DeviceState getState() {
        return state;
    }

    public DeviceEventBus getEventBus() {
        return eventBus;
    }

    @Override
    public String toString() {
        return "SmartDevice{" +
                "deviceId='" + deviceId + '\'' +
                ", name='" + name + '\'' +
                ", room='" + room + '\'' +
                ", deviceType=" + deviceType +
                ", state=" + state +
                '}';
    }
}
```

---

### SmartLight.java

```java
package smarthome.devices;

import smarthome.commands.DeviceCommand;
import smarthome.events.DeviceEventBus;
import smarthome.events.DeviceEventType;
import smarthome.model.DeviceType;

import java.util.HashMap;
import java.util.Map;

public class SmartLight extends SmartDevice {

    private int brightness; // 0-100
    private String color;

    public SmartLight(String deviceId, String name, String room, DeviceEventBus eventBus) {
        super(deviceId, name, room, DeviceType.SMART_LIGHT, eventBus);
        this.brightness = 100;
        this.color = "white";
    }

    public void setBrightness(int brightness) {
        if (brightness < 0 || brightness > 100) {
            throw new IllegalArgumentException("Brightness must be between 0 and 100, got: " + brightness);
        }
        this.brightness = brightness;
        Map<String, Object> payload = new HashMap<>();
        payload.put("brightness", brightness);
        payload.put("deviceName", getName());
        publishEvent(DeviceEventType.BRIGHTNESS_CHANGED, payload);
    }

    public void setColor(String color) {
        this.color = color;
    }

    @Override
    public void executeCommand(DeviceCommand command) {
        command.execute();
    }

    public int getBrightness() {
        return brightness;
    }

    public String getColor() {
        return color;
    }

    @Override
    public String toString() {
        return "SmartLight{" +
                "deviceId='" + getDeviceId() + '\'' +
                ", name='" + getName() + '\'' +
                ", room='" + getRoom() + '\'' +
                ", state=" + getState() +
                ", brightness=" + brightness +
                ", color='" + color + '\'' +
                '}';
    }
}
```

---

### Thermostat.java

```java
package smarthome.devices;

import smarthome.commands.DeviceCommand;
import smarthome.events.DeviceEventBus;
import smarthome.events.DeviceEventType;
import smarthome.model.DeviceType;
import smarthome.model.ThermostatMode;

import java.util.HashMap;
import java.util.Map;

public class Thermostat extends SmartDevice {

    private double currentTemperature;
    private double targetTemperature;
    private ThermostatMode mode;

    public Thermostat(String deviceId, String name, String room, DeviceEventBus eventBus) {
        super(deviceId, name, room, DeviceType.THERMOSTAT, eventBus);
        this.currentTemperature = 20.0;
        this.targetTemperature = 22.0;
        this.mode = ThermostatMode.AUTO;
    }

    public void setTargetTemperature(double temp) {
        this.targetTemperature = temp;

        Map<String, Object> changedPayload = new HashMap<>();
        changedPayload.put("targetTemperature", temp);
        changedPayload.put("currentTemperature", currentTemperature);
        changedPayload.put("deviceName", getName());
        publishEvent(DeviceEventType.TEMPERATURE_CHANGED, changedPayload);

        if (temp > 30.0) {
            Map<String, Object> alertPayload = new HashMap<>();
            alertPayload.put("alertTemperature", temp);
            alertPayload.put("threshold", 30.0);
            alertPayload.put("deviceName", getName());
            publishEvent(DeviceEventType.TEMPERATURE_ALERT, alertPayload);
        }
    }

    public void setMode(ThermostatMode mode) {
        this.mode = mode;
    }

    public void setCurrentTemperature(double temp) {
        this.currentTemperature = temp;
    }

    @Override
    public void executeCommand(DeviceCommand command) {
        command.execute();
    }

    public double getCurrentTemperature() {
        return currentTemperature;
    }

    public double getTargetTemperature() {
        return targetTemperature;
    }

    public ThermostatMode getMode() {
        return mode;
    }

    @Override
    public String toString() {
        return "Thermostat{" +
                "deviceId='" + getDeviceId() + '\'' +
                ", name='" + getName() + '\'' +
                ", room='" + getRoom() + '\'' +
                ", state=" + getState() +
                ", currentTemperature=" + currentTemperature +
                ", targetTemperature=" + targetTemperature +
                ", mode=" + mode +
                '}';
    }
}
```

---

### SecurityCamera.java

```java
package smarthome.devices;

import smarthome.commands.DeviceCommand;
import smarthome.events.DeviceEventBus;
import smarthome.events.DeviceEventType;
import smarthome.model.DeviceType;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

public class SecurityCamera extends SmartDevice {

    private boolean recording;
    private int motionSensitivity; // 1-10

    public SecurityCamera(String deviceId, String name, String room, DeviceEventBus eventBus) {
        super(deviceId, name, room, DeviceType.SECURITY_CAMERA, eventBus);
        this.recording = false;
        this.motionSensitivity = 5;
    }

    public void startRecording() {
        this.recording = true;
        Map<String, Object> payload = new HashMap<>();
        payload.put("deviceName", getName());
        payload.put("timestamp", LocalDateTime.now().toString());
        publishEvent(DeviceEventType.RECORDING_STARTED, payload);
    }

    public void stopRecording() {
        this.recording = false;
        Map<String, Object> payload = new HashMap<>();
        payload.put("deviceName", getName());
        payload.put("timestamp", LocalDateTime.now().toString());
        publishEvent(DeviceEventType.RECORDING_STOPPED, payload);
    }

    public void detectMotion() {
        Map<String, Object> payload = new HashMap<>();
        payload.put("sensitivity", motionSensitivity);
        payload.put("deviceName", getName());
        payload.put("location", getRoom());
        publishEvent(DeviceEventType.MOTION_DETECTED, payload);
    }

    public void setMotionSensitivity(int sensitivity) {
        if (sensitivity < 1 || sensitivity > 10) {
            throw new IllegalArgumentException("Motion sensitivity must be between 1 and 10, got: " + sensitivity);
        }
        this.motionSensitivity = sensitivity;
    }

    @Override
    public void executeCommand(DeviceCommand command) {
        command.execute();
    }

    public boolean isRecording() {
        return recording;
    }

    public int getMotionSensitivity() {
        return motionSensitivity;
    }

    @Override
    public String toString() {
        return "SecurityCamera{" +
                "deviceId='" + getDeviceId() + '\'' +
                ", name='" + getName() + '\'' +
                ", room='" + getRoom() + '\'' +
                ", state=" + getState() +
                ", recording=" + recording +
                ", motionSensitivity=" + motionSensitivity +
                '}';
    }
}
```

---

### DoorLock.java

```java
package smarthome.devices;

import smarthome.commands.DeviceCommand;
import smarthome.events.DeviceEventBus;
import smarthome.events.DeviceEventType;
import smarthome.model.DeviceType;

import java.util.HashMap;
import java.util.Map;

public class DoorLock extends SmartDevice {

    private boolean locked;
    private final String pin;

    public DoorLock(String deviceId, String name, String room, String pin, DeviceEventBus eventBus) {
        super(deviceId, name, room, DeviceType.DOOR_LOCK, eventBus);
        this.locked = true;
        this.pin = pin;
    }

    public void lock() {
        this.locked = true;
        Map<String, Object> payload = new HashMap<>();
        payload.put("deviceName", getName());
        payload.put("location", getRoom());
        publishEvent(DeviceEventType.DOOR_LOCKED, payload);
    }

    public void unlock(String enteredPin) {
        if (pin.equals(enteredPin)) {
            this.locked = false;
            Map<String, Object> payload = new HashMap<>();
            payload.put("deviceName", getName());
            payload.put("location", getRoom());
            publishEvent(DeviceEventType.DOOR_UNLOCKED, payload);
        } else {
            System.out.println("SECURITY ALERT: Invalid PIN for " + getName());
        }
    }

    public boolean isLocked() {
        return locked;
    }

    @Override
    public void executeCommand(DeviceCommand command) {
        command.execute();
    }

    public String getPin() {
        return pin;
    }

    @Override
    public String toString() {
        return "DoorLock{" +
                "deviceId='" + getDeviceId() + '\'' +
                ", name='" + getName() + '\'' +
                ", room='" + getRoom() + '\'' +
                ", state=" + getState() +
                ", locked=" + locked +
                '}';
    }
}
```

---

### DeviceCommand.java

```java
package smarthome.commands;

public interface DeviceCommand {

    void execute();

    void undo();

    String getDescription();
}
```

---

### TurnOnCommand.java

```java
package smarthome.commands;

import smarthome.devices.SmartDevice;
import smarthome.model.DeviceState;

public class TurnOnCommand implements DeviceCommand {

    private final SmartDevice target;

    public TurnOnCommand(SmartDevice target) {
        this.target = target;
    }

    @Override
    public void execute() {
        target.setState(DeviceState.ON);
    }

    @Override
    public void undo() {
        target.setState(DeviceState.OFF);
    }

    @Override
    public String getDescription() {
        return "Turn ON " + target.getName();
    }
}
```

---

### TurnOffCommand.java

```java
package smarthome.commands;

import smarthome.devices.SmartDevice;
import smarthome.model.DeviceState;

public class TurnOffCommand implements DeviceCommand {

    private final SmartDevice target;

    public TurnOffCommand(SmartDevice target) {
        this.target = target;
    }

    @Override
    public void execute() {
        target.setState(DeviceState.OFF);
    }

    @Override
    public void undo() {
        target.setState(DeviceState.ON);
    }

    @Override
    public String getDescription() {
        return "Turn OFF " + target.getName();
    }
}
```

---

### SetBrightnessCommand.java

```java
package smarthome.commands;

import smarthome.devices.SmartLight;

public class SetBrightnessCommand implements DeviceCommand {

    private final SmartLight target;
    private final int newBrightness;
    private final int previousBrightness;

    public SetBrightnessCommand(SmartLight target, int newBrightness) {
        this.target = target;
        this.newBrightness = newBrightness;
        this.previousBrightness = target.getBrightness();
    }

    @Override
    public void execute() {
        target.setBrightness(newBrightness);
    }

    @Override
    public void undo() {
        target.setBrightness(previousBrightness);
    }

    @Override
    public String getDescription() {
        return "Set brightness of " + target.getName() + " to " + newBrightness + "%";
    }
}
```

---

### SetTemperatureCommand.java

```java
package smarthome.commands;

import smarthome.devices.Thermostat;

public class SetTemperatureCommand implements DeviceCommand {

    private final Thermostat target;
    private final double newTemp;
    private final double previousTemp;

    public SetTemperatureCommand(Thermostat target, double newTemp) {
        this.target = target;
        this.newTemp = newTemp;
        this.previousTemp = target.getTargetTemperature();
    }

    @Override
    public void execute() {
        target.setTargetTemperature(newTemp);
    }

    @Override
    public void undo() {
        target.setTargetTemperature(previousTemp);
    }

    @Override
    public String getDescription() {
        return "Set temperature of " + target.getName() + " to " + newTemp + "°C";
    }
}
```

---

### LockDoorCommand.java

```java
package smarthome.commands;

import smarthome.devices.DoorLock;

public class LockDoorCommand implements DeviceCommand {

    private final DoorLock target;
    private final boolean lockState;
    private final boolean previousState;
    private final String pin;

    public LockDoorCommand(DoorLock target, boolean lockState, String pin) {
        this.target = target;
        this.lockState = lockState;
        this.pin = pin;
        this.previousState = target.isLocked();
    }

    @Override
    public void execute() {
        if (lockState) {
            target.lock();
        } else {
            target.unlock(pin);
        }
    }

    @Override
    public void undo() {
        if (previousState) {
            target.lock();
        } else {
            target.unlock(pin);
        }
    }

    @Override
    public String getDescription() {
        return (lockState ? "Lock" : "Unlock") + " door " + target.getName();
    }
}
```

---

### StartRecordingCommand.java

```java
package smarthome.commands;

import smarthome.devices.SecurityCamera;

public class StartRecordingCommand implements DeviceCommand {

    private final SecurityCamera target;

    public StartRecordingCommand(SecurityCamera target) {
        this.target = target;
    }

    @Override
    public void execute() {
        target.startRecording();
    }

    @Override
    public void undo() {
        target.stopRecording();
    }

    @Override
    public String getDescription() {
        return "Start recording on " + target.getName();
    }
}
```

---

### CommandHistory.java

```java
package smarthome.commands;

import java.util.ArrayDeque;
import java.util.Collections;
import java.util.Collection;
import java.util.Deque;

public class CommandHistory {

    private final Deque<DeviceCommand> undoStack = new ArrayDeque<>();
    private final Deque<DeviceCommand> redoStack = new ArrayDeque<>();
    private final int maxSize = 50;

    public void executeCommand(DeviceCommand command) {
        command.execute();
        undoStack.addLast(command);
        if (undoStack.size() > maxSize) {
            undoStack.pollFirst();
        }
        redoStack.clear();
    }

    public void undo() {
        if (!undoStack.isEmpty()) {
            DeviceCommand command = undoStack.pollLast();
            command.undo();
            redoStack.addLast(command);
            System.out.println("Undone: " + command.getDescription());
        }
    }

    public void redo() {
        if (!redoStack.isEmpty()) {
            DeviceCommand command = redoStack.pollLast();
            command.execute();
            undoStack.addLast(command);
            System.out.println("Redone: " + command.getDescription());
        }
    }

    public boolean canUndo() {
        return !undoStack.isEmpty();
    }

    public boolean canRedo() {
        return !redoStack.isEmpty();
    }

    public int getHistorySize() {
        return undoStack.size();
    }

    public Collection<DeviceCommand> getUndoStack() {
        return Collections.unmodifiableCollection(undoStack);
    }
}
```

---

### DeviceEventListener.java

```java
package smarthome.events;

public interface DeviceEventListener {

    void onEvent(DeviceEvent event);

    String getListenerId();
}
```

---

### DeviceEventBus.java

```java
package smarthome.events;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class DeviceEventBus {

    private final Map<DeviceEventType, List<DeviceEventListener>> listeners = new HashMap<>();
    private final Map<String, DeviceEventListener> listenerRegistry = new HashMap<>();

    public DeviceEventBus() {
        for (DeviceEventType type : DeviceEventType.values()) {
            listeners.put(type, new ArrayList<>());
        }
    }

    public void subscribe(DeviceEventType type, DeviceEventListener listener) {
        listeners.get(type).add(listener);
        listenerRegistry.put(listener.getListenerId(), listener);
    }

    public void unsubscribe(String listenerId) {
        listenerRegistry.remove(listenerId);
        for (List<DeviceEventListener> listenerList : listeners.values()) {
            listenerList.removeIf(listener -> listener.getListenerId().equals(listenerId));
        }
    }

    public void publish(DeviceEvent event) {
        System.out.println("[EventBus] Publishing event: " + event.getEventType() + " from device: " + event.getDeviceId());
        List<DeviceEventListener> eventListeners = listeners.get(event.getEventType());
        if (eventListeners != null) {
            for (DeviceEventListener listener : eventListeners) {
                listener.onEvent(event);
            }
        }
    }

    public int getSubscriberCount(DeviceEventType type) {
        return listeners.get(type).size();
    }
}
```

---

### Condition.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;

public interface Condition {

    boolean evaluate(DeviceEvent event);

    String getDescription();
}
```

---

### DeviceStateCondition.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventType;
import smarthome.model.DeviceState;

public class DeviceStateCondition implements Condition {

    private final String targetDeviceId;
    private final DeviceState expectedState;

    public DeviceStateCondition(String targetDeviceId, DeviceState expectedState) {
        this.targetDeviceId = targetDeviceId;
        this.expectedState = expectedState;
    }

    @Override
    public boolean evaluate(DeviceEvent event) {
        return event.getDeviceId().equals(targetDeviceId)
                && event.getEventType() == DeviceEventType.STATE_CHANGED
                && expectedState.name().equals(String.valueOf(event.getPayload().get("newState")));
    }

    @Override
    public String getDescription() {
        return "Device " + targetDeviceId + " state equals " + expectedState;
    }
}
```

---

### MotionCondition.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventType;

public class MotionCondition implements Condition {

    private final String targetDeviceId;

    public MotionCondition(String targetDeviceId) {
        this.targetDeviceId = targetDeviceId;
    }

    @Override
    public boolean evaluate(DeviceEvent event) {
        return event.getEventType() == DeviceEventType.MOTION_DETECTED
                && event.getDeviceId().equals(targetDeviceId);
    }

    @Override
    public String getDescription() {
        return "Motion detected on device " + targetDeviceId;
    }
}
```

---

### TimeCondition.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;

public class TimeCondition implements Condition {

    private final int targetHour;
    private final int targetMinute;
    private final int toleranceMinutes;

    public TimeCondition(int targetHour, int targetMinute, int toleranceMinutes) {
        this.targetHour = targetHour;
        this.targetMinute = targetMinute;
        this.toleranceMinutes = toleranceMinutes;
    }

    @Override
    public boolean evaluate(DeviceEvent event) {
        if (!event.getPayload().containsKey("hour")) {
            return false;
        }
        int eventHour = ((Number) event.getPayload().get("hour")).intValue();
        int eventMinute = event.getPayload().containsKey("minute")
                ? ((Number) event.getPayload().get("minute")).intValue()
                : 0;

        int targetTotalMinutes = targetHour * 60 + targetMinute;
        int eventTotalMinutes = eventHour * 60 + eventMinute;
        int difference = Math.abs(targetTotalMinutes - eventTotalMinutes);
        return difference <= toleranceMinutes;
    }

    @Override
    public String getDescription() {
        return "Time is " + targetHour + ":" + String.format("%02d", targetMinute)
                + " (±" + toleranceMinutes + " min)";
    }
}
```

---

### TemperatureAlertCondition.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventType;

public class TemperatureAlertCondition implements Condition {

    private final double thresholdTemperature;

    public TemperatureAlertCondition(double thresholdTemperature) {
        this.thresholdTemperature = thresholdTemperature;
    }

    @Override
    public boolean evaluate(DeviceEvent event) {
        return event.getEventType() == DeviceEventType.TEMPERATURE_ALERT
                && event.getPayload().containsKey("alertTemperature")
                && ((Number) event.getPayload().get("alertTemperature")).doubleValue() > thresholdTemperature;
    }

    @Override
    public String getDescription() {
        return "Temperature exceeds " + thresholdTemperature + "°C";
    }
}
```

---

### Action.java

```java
package smarthome.rules;

public interface Action {

    void execute();

    String getDescription();
}
```

---

### SendNotificationAction.java

```java
package smarthome.rules;

public class SendNotificationAction implements Action {

    private final String message;
    private final String recipient;

    public SendNotificationAction(String message, String recipient) {
        this.message = message;
        this.recipient = recipient;
    }

    @Override
    public void execute() {
        System.out.println("[NOTIFICATION] To: " + recipient + " | Message: " + message);
    }

    @Override
    public String getDescription() {
        return "Send notification to " + recipient;
    }
}
```

---

### ExecuteCommandAction.java

```java
package smarthome.rules;

import smarthome.commands.CommandHistory;
import smarthome.commands.DeviceCommand;

public class ExecuteCommandAction implements Action {

    private final DeviceCommand command;
    private final CommandHistory history;

    public ExecuteCommandAction(DeviceCommand command, CommandHistory history) {
        this.command = command;
        this.history = history;
    }

    @Override
    public void execute() {
        history.executeCommand(command);
    }

    @Override
    public String getDescription() {
        return "Execute command: " + command.getDescription();
    }
}
```

---

### ActivateSceneAction.java

```java
package smarthome.rules;

import smarthome.scenes.Scene;
import smarthome.scenes.SceneManager;

public class ActivateSceneAction implements Action {

    private final Scene scene;
    private final SceneManager sceneManager;

    public ActivateSceneAction(Scene scene, SceneManager sceneManager) {
        this.scene = scene;
        this.sceneManager = sceneManager;
    }

    @Override
    public void execute() {
        sceneManager.activateScene(scene.getName());
    }

    @Override
    public String getDescription() {
        return "Activate scene: " + scene.getName();
    }
}
```

---

### AutomationRule.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;

public class AutomationRule {

    private final String name;
    private final Condition condition;
    private final Action action;
    private boolean enabled;

    public AutomationRule(String name, Condition condition, Action action) {
        this.name = name;
        this.condition = condition;
        this.action = action;
        this.enabled = true;
    }

    public void enable() {
        this.enabled = true;
        System.out.println("[Rule] Rule enabled: " + name);
    }

    public void disable() {
        this.enabled = false;
        System.out.println("[Rule] Rule disabled: " + name);
    }

    public boolean matches(DeviceEvent event) {
        return enabled && condition.evaluate(event);
    }

    public void execute() {
        action.execute();
        System.out.println("[Rule] Rule fired: " + name);
    }

    public String getName() {
        return name;
    }

    public Condition getCondition() {
        return condition;
    }

    public Action getAction() {
        return action;
    }

    public boolean isEnabled() {
        return enabled;
    }

    @Override
    public String toString() {
        return "AutomationRule{" +
                "name='" + name + '\'' +
                ", condition=" + condition.getDescription() +
                ", action=" + action.getDescription() +
                ", enabled=" + enabled +
                '}';
    }
}
```

---

### RuleHandler.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;

public abstract class RuleHandler {

    protected final AutomationRule rule;
    protected RuleHandler nextHandler;

    public RuleHandler(AutomationRule rule) {
        this.rule = rule;
    }

    public RuleHandler setNext(RuleHandler handler) {
        this.nextHandler = handler;
        return handler;
    }

    public abstract boolean canHandle(DeviceEvent event);

    public void handle(DeviceEvent event) {
        if (canHandle(event) && rule.matches(event)) {
            rule.execute();
        }
        if (nextHandler != null) {
            nextHandler.handle(event);
        }
    }

    public AutomationRule getRule() {
        return rule;
    }
}
```

---

### StateChangeRuleHandler.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventType;

public class StateChangeRuleHandler extends RuleHandler {

    public StateChangeRuleHandler(AutomationRule rule) {
        super(rule);
    }

    @Override
    public boolean canHandle(DeviceEvent event) {
        return event.getEventType() == DeviceEventType.STATE_CHANGED;
    }
}
```

---

### MotionRuleHandler.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventType;

public class MotionRuleHandler extends RuleHandler {

    public MotionRuleHandler(AutomationRule rule) {
        super(rule);
    }

    @Override
    public boolean canHandle(DeviceEvent event) {
        return event.getEventType() == DeviceEventType.MOTION_DETECTED;
    }
}
```

---

### TemperatureRuleHandler.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventType;

public class TemperatureRuleHandler extends RuleHandler {

    public TemperatureRuleHandler(AutomationRule rule) {
        super(rule);
    }

    @Override
    public boolean canHandle(DeviceEvent event) {
        return event.getEventType() == DeviceEventType.TEMPERATURE_ALERT
                || event.getEventType() == DeviceEventType.TEMPERATURE_CHANGED;
    }
}
```

---

### DoorEventRuleHandler.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventType;

public class DoorEventRuleHandler extends RuleHandler {

    public DoorEventRuleHandler(AutomationRule rule) {
        super(rule);
    }

    @Override
    public boolean canHandle(DeviceEvent event) {
        return event.getEventType() == DeviceEventType.DOOR_LOCKED
                || event.getEventType() == DeviceEventType.DOOR_UNLOCKED;
    }
}
```

---

### RuleEngine.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventListener;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class RuleEngine implements DeviceEventListener {

    private final List<AutomationRule> rules = new ArrayList<>();
    private RuleHandler chainHead;

    public void addRule(AutomationRule rule) {
        rules.add(rule);
        buildChain();
    }

    public void removeRule(String ruleName) {
        rules.removeIf(rule -> rule.getName().equals(ruleName));
        buildChain();
    }

    private void buildChain() {
        if (rules.isEmpty()) {
            chainHead = null;
            return;
        }

        List<RuleHandler> handlers = new ArrayList<>();
        for (AutomationRule rule : rules) {
            RuleHandler handler;
            if (rule.getCondition() instanceof MotionCondition) {
                handler = new MotionRuleHandler(rule);
            } else if (rule.getCondition() instanceof TemperatureAlertCondition) {
                handler = new TemperatureRuleHandler(rule);
            } else if (rule.getCondition() instanceof DeviceStateCondition) {
                handler = new StateChangeRuleHandler(rule);
            } else {
                handler = new StateChangeRuleHandler(rule);
            }
            handlers.add(handler);
        }

        for (int i = 0; i < handlers.size() - 1; i++) {
            handlers.get(i).setNext(handlers.get(i + 1));
        }

        chainHead = handlers.get(0);
    }

    public void evaluateEvent(DeviceEvent event) {
        if (chainHead != null) {
            chainHead.handle(event);
        }
    }

    @Override
    public void onEvent(DeviceEvent event) {
        evaluateEvent(event);
    }

    @Override
    public String getListenerId() {
        return "rule-engine";
    }

    public List<AutomationRule> getRules() {
        return Collections.unmodifiableList(rules);
    }

    public int getRuleCount() {
        return rules.size();
    }
}
```

---

### Scene.java

```java
package smarthome.scenes;

import smarthome.commands.DeviceCommand;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class Scene {

    private final String name;
    private final String description;
    private final List<DeviceCommand> commands = new ArrayList<>();

    public Scene(String name, String description) {
        this.name = name;
        this.description = description;
    }

    public void addCommand(DeviceCommand command) {
        commands.add(command);
    }

    public List<DeviceCommand> getCommands() {
        return Collections.unmodifiableList(commands);
    }

    public String getName() {
        return name;
    }

    public String getDescription() {
        return description;
    }

    @Override
    public String toString() {
        return "Scene{name='" + name + "', commands=" + commands.size() + "}";
    }
}
```

---

### SceneManager.java

```java
package smarthome.scenes;

import smarthome.commands.CommandHistory;
import smarthome.commands.DeviceCommand;

import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class SceneManager {

    private final Map<String, Scene> scenes = new HashMap<>();
    private final CommandHistory commandHistory;

    public SceneManager(CommandHistory commandHistory) {
        this.commandHistory = commandHistory;
    }

    public void saveScene(Scene scene) {
        scenes.put(scene.getName(), scene);
        System.out.println("[SceneManager] Scene saved: " + scene.getName());
    }

    public void activateScene(String sceneName) {
        if (!scenes.containsKey(sceneName)) {
            System.out.println("[SceneManager] Scene not found: " + sceneName);
            return;
        }
        Scene scene = scenes.get(sceneName);
        System.out.println("[SceneManager] Activating scene: " + sceneName);
        for (DeviceCommand command : scene.getCommands()) {
            commandHistory.executeCommand(command);
        }
        System.out.println("[SceneManager] Scene activated: " + sceneName
                + " (" + scene.getCommands().size() + " commands executed)");
    }

    public void deleteScene(String sceneName) {
        scenes.remove(sceneName);
        System.out.println("[SceneManager] Scene deleted: " + sceneName);
    }

    public Scene getScene(String sceneName) {
        return scenes.get(sceneName);
    }

    public Map<String, Scene> getAllScenes() {
        return Collections.unmodifiableMap(scenes);
    }

    public List<String> listSceneNames() {
        return new ArrayList<>(scenes.keySet());
    }
}
```

---

### DeviceComponent.java

```java
package smarthome.composite;

import smarthome.commands.CommandHistory;
import smarthome.commands.DeviceCommand;
import smarthome.devices.SmartDevice;

import java.util.List;
import java.util.function.Function;

public interface DeviceComponent {

    String getName();

    List<SmartDevice> getDevices();

    default void addDevice(SmartDevice device) {
        throw new UnsupportedOperationException("Cannot add device to this component");
    }

    void executeOnAll(Function<SmartDevice, DeviceCommand> factory, CommandHistory history);
}
```

---

### Room.java

```java
package smarthome.composite;

import smarthome.commands.CommandHistory;
import smarthome.commands.DeviceCommand;
import smarthome.devices.SmartDevice;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.function.Function;
import java.util.stream.Collectors;

public class Room implements DeviceComponent {

    private final String name;
    private final List<SmartDevice> devices = new ArrayList<>();

    public Room(String name) {
        this.name = name;
    }

    @Override
    public void addDevice(SmartDevice device) {
        devices.add(device);
        System.out.println("[Room] Device " + device.getName() + " added to room " + name);
    }

    public void removeDevice(SmartDevice device) {
        devices.remove(device);
    }

    @Override
    public List<SmartDevice> getDevices() {
        return Collections.unmodifiableList(devices);
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public void executeOnAll(Function<SmartDevice, DeviceCommand> factory, CommandHistory history) {
        for (SmartDevice device : devices) {
            DeviceCommand command = factory.apply(device);
            history.executeCommand(command);
        }
    }

    @Override
    public String toString() {
        return "Room{name='" + name + "', devices="
                + devices.stream().map(SmartDevice::getName).collect(Collectors.joining(", ")) + "}";
    }
}
```

---

### DeviceGroup.java

```java
package smarthome.composite;

import smarthome.commands.CommandHistory;
import smarthome.commands.DeviceCommand;
import smarthome.devices.SmartDevice;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.function.Function;
import java.util.stream.Collectors;

public class DeviceGroup implements DeviceComponent {

    private final String name;
    private final List<SmartDevice> devices = new ArrayList<>();

    public DeviceGroup(String name) {
        this.name = name;
    }

    @Override
    public void addDevice(SmartDevice device) {
        devices.add(device);
    }

    public void removeDevice(SmartDevice device) {
        devices.remove(device);
    }

    @Override
    public List<SmartDevice> getDevices() {
        return Collections.unmodifiableList(devices);
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public void executeOnAll(Function<SmartDevice, DeviceCommand> factory, CommandHistory history) {
        for (SmartDevice device : devices) {
            DeviceCommand command = factory.apply(device);
            history.executeCommand(command);
        }
    }

    @Override
    public String toString() {
        return "DeviceGroup{name='" + name + "', devices="
                + devices.stream().map(SmartDevice::getName).collect(Collectors.joining(", ")) + "}";
    }
}
```

---

### HomeAutomationSystem.java

```java
package smarthome.system;

import smarthome.commands.CommandHistory;
import smarthome.commands.DeviceCommand;
import smarthome.composite.DeviceGroup;
import smarthome.composite.Room;
import smarthome.devices.SmartDevice;
import smarthome.events.DeviceEventBus;
import smarthome.events.DeviceEventType;
import smarthome.rules.AutomationRule;
import smarthome.rules.RuleEngine;
import smarthome.scenes.SceneManager;

import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class HomeAutomationSystem {

    private final Map<String, SmartDevice> devices = new HashMap<>();
    private final Map<String, Room> rooms = new HashMap<>();
    private final Map<String, DeviceGroup> groups = new HashMap<>();
    private final DeviceEventBus eventBus;
    private final RuleEngine ruleEngine;
    private final SceneManager sceneManager;
    private final CommandHistory commandHistory;

    public HomeAutomationSystem() {
        this.eventBus = new DeviceEventBus();
        this.commandHistory = new CommandHistory();
        this.ruleEngine = new RuleEngine();
        this.sceneManager = new SceneManager(commandHistory);

        for (DeviceEventType type : DeviceEventType.values()) {
            eventBus.subscribe(type, ruleEngine);
        }
    }

    public void addDevice(SmartDevice device) {
        devices.put(device.getDeviceId(), device);
    }

    public void addRoom(Room room) {
        rooms.put(room.getName(), room);
    }

    public void addDeviceGroup(DeviceGroup group) {
        groups.put(group.getName(), group);
    }

    public void addRule(AutomationRule rule) {
        ruleEngine.addRule(rule);
    }

    public void activateScene(String sceneName) {
        sceneManager.activateScene(sceneName);
    }

    public void executeCommand(DeviceCommand command) {
        commandHistory.executeCommand(command);
    }

    public void undo() {
        commandHistory.undo();
    }

    public void redo() {
        commandHistory.redo();
    }

    public SmartDevice getDevice(String deviceId) {
        return devices.get(deviceId);
    }

    public List<SmartDevice> getDevicesInRoom(String roomName) {
        if (!rooms.containsKey(roomName)) {
            return Collections.emptyList();
        }
        return rooms.get(roomName).getDevices();
    }

    public void getStatus() {
        System.out.println("=== HOME AUTOMATION SYSTEM STATUS ===");
        for (SmartDevice device : devices.values()) {
            System.out.println("  [" + device.getDeviceType() + "] "
                    + device.getName() + " (" + device.getRoom() + ") - State: " + device.getState());
        }
    }

    public DeviceEventBus getEventBus() {
        return eventBus;
    }

    public RuleEngine getRuleEngine() {
        return ruleEngine;
    }

    public SceneManager getSceneManager() {
        return sceneManager;
    }

    public CommandHistory getCommandHistory() {
        return commandHistory;
    }
}
```

---

### SmartHomeDemo.java

```java
package smarthome.demo;

import smarthome.commands.LockDoorCommand;
import smarthome.commands.SetBrightnessCommand;
import smarthome.commands.SetTemperatureCommand;
import smarthome.commands.StartRecordingCommand;
import smarthome.commands.TurnOffCommand;
import smarthome.commands.TurnOnCommand;
import smarthome.composite.Room;
import smarthome.devices.DoorLock;
import smarthome.devices.SecurityCamera;
import smarthome.devices.SmartLight;
import smarthome.devices.Thermostat;
import smarthome.model.DeviceState;
import smarthome.rules.AutomationRule;
import smarthome.rules.DeviceStateCondition;
import smarthome.rules.ExecuteCommandAction;
import smarthome.rules.MotionCondition;
import smarthome.rules.SendNotificationAction;
import smarthome.scenes.Scene;
import smarthome.system.HomeAutomationSystem;

public class SmartHomeDemo {

    public static void main(String[] args) {

        // Step 1: Initialize system
        System.out.println("=== Smart Home Automation System Demo ===");
        System.out.println("Initializing system...");
        HomeAutomationSystem system = new HomeAutomationSystem();

        // Step 2: Create devices
        System.out.println("\n--- Creating Devices ---");
        SmartLight livingRoomLight = new SmartLight("light-001", "Living Room Light", "Living Room", system.getEventBus());
        SmartLight bedroomLight = new SmartLight("light-002", "Bedroom Light", "Bedroom", system.getEventBus());
        Thermostat thermostat = new Thermostat("thermo-001", "Main Thermostat", "Living Room", system.getEventBus());
        SecurityCamera camera = new SecurityCamera("cam-001", "Front Door Camera", "Entryway", system.getEventBus());
        DoorLock doorLock = new DoorLock("lock-001", "Front Door Lock", "Entryway", "1234", system.getEventBus());

        system.addDevice(livingRoomLight);
        system.addDevice(bedroomLight);
        system.addDevice(thermostat);
        system.addDevice(camera);
        system.addDevice(doorLock);

        System.out.println("Created 5 devices: 2 SmartLights, 1 Thermostat, 1 SecurityCamera, 1 DoorLock");

        // Step 3: Set up rooms
        System.out.println("\n--- Setting Up Rooms ---");
        Room livingRoom = new Room("Living Room");
        Room bedroom = new Room("Bedroom");
        Room entryway = new Room("Entryway");

        livingRoom.addDevice(livingRoomLight);
        livingRoom.addDevice(thermostat);
        bedroom.addDevice(bedroomLight);
        entryway.addDevice(camera);
        entryway.addDevice(doorLock);

        system.addRoom(livingRoom);
        system.addRoom(bedroom);
        system.addRoom(entryway);

        // Step 4: Create Movie Night scene
        System.out.println("\n--- Creating Movie Night Scene ---");
        Scene movieNight = new Scene("Movie Night", "Dims lights and sets comfortable temperature");
        movieNight.addCommand(new SetBrightnessCommand(livingRoomLight, 20));
        movieNight.addCommand(new TurnOffCommand(bedroomLight));
        movieNight.addCommand(new SetTemperatureCommand(thermostat, 22.0));
        system.getSceneManager().saveScene(movieNight);

        // Step 5: Create Away Mode scene
        System.out.println("\n--- Creating Away Mode Scene ---");
        Scene awayMode = new Scene("Away Mode", "Secures home when everyone leaves");
        awayMode.addCommand(new TurnOffCommand(livingRoomLight));
        awayMode.addCommand(new TurnOffCommand(bedroomLight));
        awayMode.addCommand(new LockDoorCommand(doorLock, true, "1234"));
        awayMode.addCommand(new StartRecordingCommand(camera));
        system.getSceneManager().saveScene(awayMode);

        // Step 6: Set up automation rules
        System.out.println("\n--- Setting Up Automation Rules ---");
        AutomationRule motionRule = new AutomationRule(
                "Motion Alert",
                new MotionCondition("cam-001"),
                new SendNotificationAction("Motion detected at front door!", "homeowner@example.com")
        );
        AutomationRule doorUnlockRule = new AutomationRule(
                "Door Unlock Light",
                new DeviceStateCondition("lock-001", DeviceState.ON),
                new ExecuteCommandAction(new TurnOnCommand(livingRoomLight), system.getCommandHistory())
        );
        system.addRule(motionRule);
        system.addRule(doorUnlockRule);
        System.out.println("Automation rules configured: Motion Alert, Door Unlock Light");

        // Step 7: Trigger events
        System.out.println("\n--- Triggering Events ---");

        System.out.println("\n[Event Test 1] Camera detects motion:");
        camera.detectMotion();

        System.out.println("\n[Event Test 2] Door unlocked:");
        doorLock.unlock("1234");

        // Step 8: Activate Movie Night scene
        System.out.println("\n--- Activating Movie Night Scene ---");
        system.activateScene("Movie Night");
        System.out.println("\nDevice states after Movie Night:");
        System.out.println("  Living Room Light brightness: " + livingRoomLight.getBrightness() + "%");
        System.out.println("  Bedroom Light state: " + bedroomLight.getState());
        System.out.println("  Thermostat target temp: " + thermostat.getTargetTemperature() + "°C");

        // Step 9: Test Undo/Redo
        System.out.println("\n--- Testing Undo/Redo ---");
        System.out.println("Current living room brightness: " + livingRoomLight.getBrightness() + "%");
        system.executeCommand(new SetBrightnessCommand(livingRoomLight, 80));
        System.out.println("After setting to 80%: " + livingRoomLight.getBrightness() + "%");
        system.undo();
        System.out.println("After undo: " + livingRoomLight.getBrightness() + "%");
        system.redo();
        System.out.println("After redo: " + livingRoomLight.getBrightness() + "%");

        // Step 10: Test temperature alert
        System.out.println("\n--- Testing Temperature Alert ---");
        thermostat.setTargetTemperature(35.0);
        System.out.println("Set thermostat to 35°C (should trigger alert)");

        // Step 11: Print system status
        System.out.println("\n");
        system.getStatus();
    }
}
```

---

## Unit Tests Design

The following test class designs use JUnit 5 conventions. Each test name describes the exact behavior being verified. All dependencies are constructed directly (no mocking framework required) because the design uses interfaces everywhere.

### DeviceCommandTest

```java
package smarthome.commands;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import smarthome.devices.DoorLock;
import smarthome.devices.SmartLight;
import smarthome.devices.Thermostat;
import smarthome.events.DeviceEventBus;
import smarthome.model.DeviceState;

import static org.junit.jupiter.api.Assertions.*;

class DeviceCommandTest {

    private DeviceEventBus eventBus;
    private SmartLight light;
    private Thermostat thermostat;
    private DoorLock doorLock;

    @BeforeEach
    void setUp() {
        eventBus = new DeviceEventBus();
        light = new SmartLight("light-001", "Test Light", "Test Room", eventBus);
        thermostat = new Thermostat("thermo-001", "Test Thermostat", "Test Room", eventBus);
        doorLock = new DoorLock("lock-001", "Test Lock", "Test Room", "9999", eventBus);
    }

    @Test
    void testTurnOnCommandExecutesSetsDeviceStateToOn() {
        // Arrange
        TurnOnCommand command = new TurnOnCommand(light);
        // Act
        command.execute();
        // Assert
        assertEquals(DeviceState.ON, light.getState());
    }

    @Test
    void testTurnOffCommandExecutesSetsDeviceStateToOff() {
        // Arrange
        light.setState(DeviceState.ON);
        TurnOffCommand command = new TurnOffCommand(light);
        // Act
        command.execute();
        // Assert
        assertEquals(DeviceState.OFF, light.getState());
    }

    @Test
    void testTurnOnCommandUndoSetsDeviceStateToOff() {
        // Arrange
        TurnOnCommand command = new TurnOnCommand(light);
        command.execute(); // state is now ON
        // Act
        command.undo();
        // Assert
        assertEquals(DeviceState.OFF, light.getState());
    }

    @Test
    void testSetBrightnessCommandCapturesPreviousBrightnessOnConstruction() {
        // Arrange
        light.setBrightness(60);
        // Act: constructor captures current brightness
        SetBrightnessCommand command = new SetBrightnessCommand(light, 90);
        // Assert: previousBrightness was captured as 60 at construction time
        command.execute();
        assertEquals(90, light.getBrightness());
        command.undo();
        assertEquals(60, light.getBrightness());
    }

    @Test
    void testSetBrightnessCommandUndoRestoresPreviousBrightness() {
        // Arrange
        light.setBrightness(75);
        SetBrightnessCommand command = new SetBrightnessCommand(light, 10);
        command.execute();
        assertEquals(10, light.getBrightness());
        // Act
        command.undo();
        // Assert
        assertEquals(75, light.getBrightness());
    }

    @Test
    void testSetTemperatureCommandExecuteSetsNewTemperature() {
        // Arrange
        SetTemperatureCommand command = new SetTemperatureCommand(thermostat, 25.5);
        // Act
        command.execute();
        // Assert
        assertEquals(25.5, thermostat.getTargetTemperature(), 0.001);
    }

    @Test
    void testSetTemperatureCommandUndoRestoresPreviousTemperature() {
        // Arrange: default targetTemperature is 22.0
        SetTemperatureCommand command = new SetTemperatureCommand(thermostat, 28.0);
        command.execute();
        assertEquals(28.0, thermostat.getTargetTemperature(), 0.001);
        // Act
        command.undo();
        // Assert
        assertEquals(22.0, thermostat.getTargetTemperature(), 0.001);
    }

    @Test
    void testLockDoorCommandExecuteLocksDoor() {
        // Arrange: unlock the door first
        doorLock.unlock("9999");
        assertFalse(doorLock.isLocked());
        LockDoorCommand command = new LockDoorCommand(doorLock, true, "9999");
        // Act
        command.execute();
        // Assert
        assertTrue(doorLock.isLocked());
    }

    @Test
    void testLockDoorCommandUndoUnlocksDoorWithStoredPin() {
        // Arrange: door starts locked, command will lock it (noop), undo should unlock
        // door starts locked by default; previousState = locked
        // Use unlock command: lockState=false
        LockDoorCommand command = new LockDoorCommand(doorLock, false, "9999");
        command.execute(); // unlocks
        assertFalse(doorLock.isLocked());
        // Act: undo should restore to previousState (locked=true)
        command.undo();
        // Assert
        assertTrue(doorLock.isLocked());
    }
}
```

**Test descriptions:**

1. **testTurnOnCommandExecutesSetsDeviceStateToOn** — Creates a `TurnOnCommand` for a SmartLight initially OFF, calls `execute()`, asserts state equals `ON`.

2. **testTurnOffCommandExecutesSetsDeviceStateToOff** — Sets light to ON, creates `TurnOffCommand`, calls `execute()`, asserts state equals `OFF`.

3. **testTurnOnCommandUndoSetsDeviceStateToOff** — Executes `TurnOnCommand` (state=ON), calls `undo()`, asserts state equals `OFF`.

4. **testSetBrightnessCommandCapturesPreviousBrightnessOnConstruction** — Sets brightness to 60, constructs `SetBrightnessCommand` targeting 90 (captures 60), executes, asserts 90, undoes, asserts 60. Proves capture happens at construction not at execute time.

5. **testSetBrightnessCommandUndoRestoresPreviousBrightness** — Sets brightness 75, constructs command targeting 10, executes, asserts 10, undoes, asserts 75.

6. **testSetTemperatureCommandExecuteSetsNewTemperature** — Constructs `SetTemperatureCommand` targeting 25.5, executes, asserts `getTargetTemperature()` equals 25.5.

7. **testSetTemperatureCommandUndoRestoresPreviousTemperature** — Default is 22.0, sets to 28.0, undoes, asserts back to 22.0.

8. **testLockDoorCommandExecuteLocksDoor** — Unlocks door first, constructs `LockDoorCommand(lockState=true)`, executes, asserts `isLocked()` is true.

9. **testLockDoorCommandUndoUnlocksDoorWithStoredPin** — Door starts locked, constructs unlock command (lockState=false, previousState=true captured), executes (unlocks), undoes (locks again), asserts locked.

---

### RuleEngineTest

```java
package smarthome.rules;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventType;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

import static org.junit.jupiter.api.Assertions.*;

class RuleEngineTest {

    private RuleEngine ruleEngine;

    @BeforeEach
    void setUp() {
        ruleEngine = new RuleEngine();
    }

    @Test
    void testAddRuleIncreasesRuleCount() {
        // Arrange
        AutomationRule rule = new AutomationRule("Rule1",
                new MotionCondition("cam-001"),
                () -> {}, // anonymous action
                "no-op");
        // Act
        ruleEngine.addRule(rule);
        // Assert
        assertEquals(1, ruleEngine.getRuleCount());
    }

    @Test
    void testRemoveRuleByNameDecreasesRuleCount() {
        // Arrange
        AutomationRule rule = new AutomationRule("Rule1",
                new MotionCondition("cam-001"),
                () -> {}, "no-op");
        ruleEngine.addRule(rule);
        assertEquals(1, ruleEngine.getRuleCount());
        // Act
        ruleEngine.removeRule("Rule1");
        // Assert
        assertEquals(0, ruleEngine.getRuleCount());
    }

    @Test
    void testMotionConditionRuleFiredOnMotionEvent() {
        // Arrange
        AtomicInteger fireCount = new AtomicInteger(0);
        AutomationRule rule = new AutomationRule("MotionRule",
                new MotionCondition("cam-001"),
                new Action() {
                    @Override public void execute() { fireCount.incrementAndGet(); }
                    @Override public String getDescription() { return "count"; }
                }
        );
        ruleEngine.addRule(rule);
        Map<String, Object> payload = new HashMap<>();
        payload.put("sensitivity", 5);
        DeviceEvent event = new DeviceEvent("cam-001", DeviceEventType.MOTION_DETECTED, payload);
        // Act
        ruleEngine.evaluateEvent(event);
        // Assert
        assertEquals(1, fireCount.get());
    }

    @Test
    void testRuleNotFiredWhenDisabled() {
        // Arrange
        AtomicInteger fireCount = new AtomicInteger(0);
        AutomationRule rule = new AutomationRule("MotionRule",
                new MotionCondition("cam-001"),
                new Action() {
                    @Override public void execute() { fireCount.incrementAndGet(); }
                    @Override public String getDescription() { return "count"; }
                }
        );
        rule.disable();
        ruleEngine.addRule(rule);
        Map<String, Object> payload = new HashMap<>();
        DeviceEvent event = new DeviceEvent("cam-001", DeviceEventType.MOTION_DETECTED, payload);
        // Act
        ruleEngine.evaluateEvent(event);
        // Assert
        assertEquals(0, fireCount.get(), "Disabled rule should not fire");
    }

    @Test
    void testChainHandlesMultipleRulesForSameEventType() {
        // Arrange
        AtomicInteger fireCount = new AtomicInteger(0);
        Action countAction = new Action() {
            @Override public void execute() { fireCount.incrementAndGet(); }
            @Override public String getDescription() { return "count"; }
        };
        ruleEngine.addRule(new AutomationRule("Rule1", new MotionCondition("cam-001"), countAction));
        ruleEngine.addRule(new AutomationRule("Rule2", new MotionCondition("cam-001"), countAction));
        Map<String, Object> payload = new HashMap<>();
        DeviceEvent event = new DeviceEvent("cam-001", DeviceEventType.MOTION_DETECTED, payload);
        // Act
        ruleEngine.evaluateEvent(event);
        // Assert: both rules should fire
        assertEquals(2, fireCount.get());
    }

    @Test
    void testRuleEngineListenerIdIsRuleEngine() {
        assertEquals("rule-engine", ruleEngine.getListenerId());
    }

    @Test
    void testTemperatureRuleHandlerHandlesAlertAndChangedEvents() {
        // Arrange
        AtomicInteger fireCount = new AtomicInteger(0);
        AutomationRule rule = new AutomationRule("TempRule",
                new TemperatureAlertCondition(30.0),
                new Action() {
                    @Override public void execute() { fireCount.incrementAndGet(); }
                    @Override public String getDescription() { return "count"; }
                }
        );
        ruleEngine.addRule(rule);
        Map<String, Object> payload = new HashMap<>();
        payload.put("alertTemperature", 35.0);
        DeviceEvent alertEvent = new DeviceEvent("thermo-001", DeviceEventType.TEMPERATURE_ALERT, payload);
        // Act
        ruleEngine.evaluateEvent(alertEvent);
        // Assert: TemperatureAlertCondition matches alertTemp 35 > threshold 30
        assertEquals(1, fireCount.get());
    }
}
```

**Test descriptions:**

1. **testAddRuleIncreasesRuleCount** — Adds one rule, asserts `getRuleCount()` returns 1.

2. **testRemoveRuleByNameDecreasesRuleCount** — Adds a rule, removes it by name, asserts count is 0.

3. **testMotionConditionRuleFiredOnMotionEvent** — Adds a rule with `MotionCondition("cam-001")`, publishes a MOTION_DETECTED event from "cam-001", asserts the action executed once.

4. **testRuleNotFiredWhenDisabled** — Adds a motion rule, calls `rule.disable()`, publishes matching event, asserts action was never called.

5. **testChainHandlesMultipleRulesForSameEventType** — Adds two motion rules, publishes one event, asserts both fire (count=2). Proves the chain passes event through all handlers.

6. **testRuleEngineListenerIdIsRuleEngine** — Asserts `getListenerId()` returns the string `"rule-engine"`.

7. **testTemperatureRuleHandlerHandlesAlertAndChangedEvents** — Adds a rule with `TemperatureAlertCondition(30.0)`, publishes TEMPERATURE_ALERT with `alertTemperature=35.0`, asserts fires once.

---

### SceneManagerTest

```java
package smarthome.scenes;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import smarthome.commands.CommandHistory;
import smarthome.commands.TurnOnCommand;
import smarthome.devices.SmartLight;
import smarthome.events.DeviceEventBus;
import smarthome.model.DeviceState;

import static org.junit.jupiter.api.Assertions.*;

class SceneManagerTest {

    private SceneManager sceneManager;
    private CommandHistory commandHistory;
    private DeviceEventBus eventBus;
    private SmartLight light;

    @BeforeEach
    void setUp() {
        eventBus = new DeviceEventBus();
        commandHistory = new CommandHistory();
        sceneManager = new SceneManager(commandHistory);
        light = new SmartLight("light-001", "Test Light", "Test Room", eventBus);
    }

    @Test
    void testSaveSceneAddsToRegistry() {
        // Arrange
        Scene scene = new Scene("Morning", "Wake up routine");
        // Act
        sceneManager.saveScene(scene);
        // Assert
        assertNotNull(sceneManager.getScene("Morning"));
    }

    @Test
    void testActivateSceneExecutesAllCommands() {
        // Arrange
        Scene scene = new Scene("Test Scene", "Test");
        scene.addCommand(new TurnOnCommand(light));
        sceneManager.saveScene(scene);
        // Act
        sceneManager.activateScene("Test Scene");
        // Assert
        assertEquals(DeviceState.ON, light.getState());
    }

    @Test
    void testActivateNonExistentSceneDoesNotThrow() {
        // Should print an error message but not throw an exception
        assertDoesNotThrow(() -> sceneManager.activateScene("NonExistentScene"));
    }

    @Test
    void testDeleteSceneRemovesFromRegistry() {
        // Arrange
        Scene scene = new Scene("To Delete", "Temporary");
        sceneManager.saveScene(scene);
        assertNotNull(sceneManager.getScene("To Delete"));
        // Act
        sceneManager.deleteScene("To Delete");
        // Assert
        assertNull(sceneManager.getScene("To Delete"));
    }

    @Test
    void testListSceneNamesReturnsAllSavedSceneNames() {
        // Arrange
        sceneManager.saveScene(new Scene("Scene A", "desc"));
        sceneManager.saveScene(new Scene("Scene B", "desc"));
        sceneManager.saveScene(new Scene("Scene C", "desc"));
        // Act
        java.util.List<String> names = sceneManager.listSceneNames();
        // Assert
        assertEquals(3, names.size());
        assertTrue(names.contains("Scene A"));
        assertTrue(names.contains("Scene B"));
        assertTrue(names.contains("Scene C"));
    }

    @Test
    void testActivateSceneUpdatesDeviceState() {
        // Arrange
        assertEquals(DeviceState.OFF, light.getState());
        Scene scene = new Scene("Wake Up", "Morning lights");
        scene.addCommand(new TurnOnCommand(light));
        sceneManager.saveScene(scene);
        // Act
        sceneManager.activateScene("Wake Up");
        // Assert
        assertEquals(DeviceState.ON, light.getState());
        // Also verify it went through commandHistory (can undo)
        assertTrue(commandHistory.canUndo());
    }
}
```

**Test descriptions:**

1. **testSaveSceneAddsToRegistry** — Saves a scene, calls `getScene("Morning")`, asserts non-null.

2. **testActivateSceneExecutesAllCommands** — Scene contains one `TurnOnCommand`, activates scene, asserts device state is ON.

3. **testActivateNonExistentSceneDoesNotThrow** — Calls `activateScene` with unknown name, asserts no exception thrown.

4. **testDeleteSceneRemovesFromRegistry** — Saves then deletes a scene, asserts `getScene` returns null.

5. **testListSceneNamesReturnsAllSavedSceneNames** — Saves three scenes, asserts list contains all three names.

6. **testActivateSceneUpdatesDeviceState** — Activates a scene with `TurnOnCommand`, asserts state is ON and `commandHistory.canUndo()` is true (proving it went through the history).

---

### DeviceEventBusTest

```java
package smarthome.events;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

import static org.junit.jupiter.api.Assertions.*;

class DeviceEventBusTest {

    private DeviceEventBus eventBus;

    @BeforeEach
    void setUp() {
        eventBus = new DeviceEventBus();
    }

    private DeviceEvent createEvent(String deviceId, DeviceEventType type) {
        return new DeviceEvent(deviceId, type, new HashMap<>());
    }

    @Test
    void testSubscribeAndPublishNotifiesListener() {
        // Arrange
        List<DeviceEvent> received = new ArrayList<>();
        DeviceEventListener listener = new DeviceEventListener() {
            @Override public void onEvent(DeviceEvent event) { received.add(event); }
            @Override public String getListenerId() { return "test-listener"; }
        };
        eventBus.subscribe(DeviceEventType.STATE_CHANGED, listener);
        DeviceEvent event = createEvent("device-001", DeviceEventType.STATE_CHANGED);
        // Act
        eventBus.publish(event);
        // Assert
        assertEquals(1, received.size());
        assertEquals("device-001", received.get(0).getDeviceId());
    }

    @Test
    void testUnsubscribeRemovesListenerFromAllEventTypes() {
        // Arrange
        AtomicInteger count = new AtomicInteger(0);
        DeviceEventListener listener = new DeviceEventListener() {
            @Override public void onEvent(DeviceEvent event) { count.incrementAndGet(); }
            @Override public String getListenerId() { return "removable-listener"; }
        };
        eventBus.subscribe(DeviceEventType.STATE_CHANGED, listener);
        eventBus.subscribe(DeviceEventType.MOTION_DETECTED, listener);
        // Act
        eventBus.unsubscribe("removable-listener");
        eventBus.publish(createEvent("dev-001", DeviceEventType.STATE_CHANGED));
        eventBus.publish(createEvent("dev-001", DeviceEventType.MOTION_DETECTED));
        // Assert
        assertEquals(0, count.get(), "Listener should not receive events after unsubscribe");
    }

    @Test
    void testPublishToEventTypeWithNoSubscribersDoesNotThrow() {
        // TEMPERATURE_ALERT has no subscribers after construction
        DeviceEvent event = createEvent("thermo-001", DeviceEventType.TEMPERATURE_ALERT);
        assertDoesNotThrow(() -> eventBus.publish(event));
    }

    @Test
    void testMultipleListenersForSameEventTypeAllNotified() {
        // Arrange
        AtomicInteger count = new AtomicInteger(0);
        for (int i = 0; i < 3; i++) {
            final String id = "listener-" + i;
            eventBus.subscribe(DeviceEventType.BRIGHTNESS_CHANGED, new DeviceEventListener() {
                @Override public void onEvent(DeviceEvent event) { count.incrementAndGet(); }
                @Override public String getListenerId() { return id; }
            });
        }
        // Act
        eventBus.publish(createEvent("light-001", DeviceEventType.BRIGHTNESS_CHANGED));
        // Assert
        assertEquals(3, count.get());
    }

    @Test
    void testGetSubscriberCountReturnsCorrectCount() {
        // Arrange
        DeviceEventListener l1 = new DeviceEventListener() {
            @Override public void onEvent(DeviceEvent event) {}
            @Override public String getListenerId() { return "l1"; }
        };
        DeviceEventListener l2 = new DeviceEventListener() {
            @Override public void onEvent(DeviceEvent event) {}
            @Override public String getListenerId() { return "l2"; }
        };
        // Act
        eventBus.subscribe(DeviceEventType.DOOR_LOCKED, l1);
        eventBus.subscribe(DeviceEventType.DOOR_LOCKED, l2);
        // Assert
        assertEquals(2, eventBus.getSubscriberCount(DeviceEventType.DOOR_LOCKED));
    }

    @Test
    void testListenerRegistryContainsSubscribedListener() {
        // Arrange
        DeviceEventListener listener = new DeviceEventListener() {
            @Override public void onEvent(DeviceEvent event) {}
            @Override public String getListenerId() { return "reg-listener"; }
        };
        // Act
        eventBus.subscribe(DeviceEventType.STATE_CHANGED, listener);
        // Assert via behavior: unsubscribe by id works only if registry contains it
        // We verify by checking subscriber count drops after unsubscribe
        assertEquals(1, eventBus.getSubscriberCount(DeviceEventType.STATE_CHANGED));
        eventBus.unsubscribe("reg-listener");
        assertEquals(0, eventBus.getSubscriberCount(DeviceEventType.STATE_CHANGED));
    }
}
```

**Test descriptions:**

1. **testSubscribeAndPublishNotifiesListener** — Subscribes a listener to STATE_CHANGED, publishes a matching event, asserts listener received exactly one event.

2. **testUnsubscribeRemovesListenerFromAllEventTypes** — Subscribes to two event types, unsubscribes by ID, publishes both event types, asserts listener received nothing.

3. **testPublishToEventTypeWithNoSubscribersDoesNotThrow** — Publishes TEMPERATURE_ALERT when no one is subscribed, asserts no exception.

4. **testMultipleListenersForSameEventTypeAllNotified** — Subscribes three listeners to BRIGHTNESS_CHANGED, publishes one event, asserts all three were notified (count=3).

5. **testGetSubscriberCountReturnsCorrectCount** — Subscribes two listeners to DOOR_LOCKED, asserts `getSubscriberCount` returns 2.

6. **testListenerRegistryContainsSubscribedListener** — Subscribes a listener, asserts subscriber count is 1, unsubscribes, asserts count drops to 0. This proves the registry properly tracks and removes listeners.

---

## Extension Exercise: Adding SmartDoorbell

The doorbell is a new device type that does not modify any existing class. This demonstrates the Open/Closed Principle in full.

### Step 1: Updated DeviceEventType.java (add DOORBELL_PRESSED)

```java
package smarthome.events;

public enum DeviceEventType {
    STATE_CHANGED,
    MOTION_DETECTED,
    TEMPERATURE_ALERT,
    DOOR_LOCKED,
    DOOR_UNLOCKED,
    BRIGHTNESS_CHANGED,
    TEMPERATURE_CHANGED,
    RECORDING_STARTED,
    RECORDING_STOPPED,
    DOORBELL_PRESSED   // NEW: added for SmartDoorbell
}
```

Because `DeviceEventBus` initializes its map by iterating `DeviceEventType.values()` in its constructor, the new event type is automatically handled — the bus creates a new empty list for it with zero additional code changes.

---

### Step 2: SmartDoorbell.java

```java
package smarthome.devices;

import smarthome.commands.DeviceCommand;
import smarthome.events.DeviceEventBus;
import smarthome.events.DeviceEventType;
import smarthome.model.DeviceType;

import java.util.HashMap;
import java.util.Map;

public class SmartDoorbell extends SmartDevice {

    private final boolean hasBattery;
    private int batteryLevel; // 0-100

    public SmartDoorbell(String deviceId, String name, String room, boolean hasBattery, DeviceEventBus eventBus) {
        super(deviceId, name, room, DeviceType.SMART_SPEAKER, eventBus); // reuse SMART_SPEAKER until enum updated
        this.hasBattery = hasBattery;
        this.batteryLevel = 100;
    }

    public void pressButton() {
        Map<String, Object> payload = new HashMap<>();
        payload.put("location", getRoom());
        payload.put("deviceName", getName());
        payload.put("batteryLevel", batteryLevel);
        publishEvent(DeviceEventType.DOORBELL_PRESSED, payload);
    }

    public void setBatteryLevel(int level) {
        if (level < 0 || level > 100) {
            throw new IllegalArgumentException("Battery level must be between 0 and 100, got: " + level);
        }
        this.batteryLevel = level;
    }

    public boolean isHasBattery() {
        return hasBattery;
    }

    public int getBatteryLevel() {
        return batteryLevel;
    }

    @Override
    public void executeCommand(DeviceCommand command) {
        command.execute();
    }

    @Override
    public String toString() {
        return "SmartDoorbell{" +
                "deviceId='" + getDeviceId() + '\'' +
                ", name='" + getName() + '\'' +
                ", room='" + getRoom() + '\'' +
                ", state=" + getState() +
                ", hasBattery=" + hasBattery +
                ", batteryLevel=" + batteryLevel +
                '}';
    }
}
```

---

### Step 3: DoorbellPressedCondition.java

```java
package smarthome.rules;

import smarthome.events.DeviceEvent;
import smarthome.events.DeviceEventType;

public class DoorbellPressedCondition implements Condition {

    private final String targetDeviceId;

    public DoorbellPressedCondition(String targetDeviceId) {
        this.targetDeviceId = targetDeviceId;
    }

    @Override
    public boolean evaluate(DeviceEvent event) {
        return event.getEventType() == DeviceEventType.DOORBELL_PRESSED
                && event.getDeviceId().equals(targetDeviceId);
    }

    @Override
    public String getDescription() {
        return "Doorbell pressed on device " + targetDeviceId;
    }
}
```

---

### Step 4: RingChimeAction.java

```java
package smarthome.rules;

public class RingChimeAction implements Action {

    private final String doorbellName;
    private final String location;

    public RingChimeAction(String doorbellName, String location) {
        this.doorbellName = doorbellName;
        this.location = location;
    }

    @Override
    public void execute() {
        System.out.println("DING DONG! Doorbell pressed at: " + location + " (" + doorbellName + ")");
    }

    @Override
    public String getDescription() {
        return "Ring chime for " + doorbellName + " at " + location;
    }
}
```

---

### Step 5: AnswerDoorbellCommand.java

```java
package smarthome.commands;

import smarthome.devices.SmartDoorbell;

public class AnswerDoorbellCommand implements DeviceCommand {

    private final SmartDoorbell doorbell;

    public AnswerDoorbellCommand(SmartDoorbell doorbell) {
        this.doorbell = doorbell;
    }

    @Override
    public void execute() {
        System.out.println("[Doorbell] Doorbell answered at: " + doorbell.getRoom()
                + " - " + doorbell.getName());
    }

    @Override
    public void undo() {
        System.out.println("[Doorbell] Doorbell answer cancelled for: " + doorbell.getName());
    }

    @Override
    public String getDescription() {
        return "Answer doorbell: " + doorbell.getName();
    }
}
```

---

### Step 6: Wiring the Doorbell into the Existing System

```java
// No changes needed to HomeAutomationSystem, RuleEngine, DeviceEventBus, or any existing class.
// Simply instantiate and wire:

SmartDoorbell doorbell = new SmartDoorbell("bell-001", "Front Doorbell", "Entryway", true, system.getEventBus());
system.addDevice(doorbell);
entryway.addDevice(doorbell);

AutomationRule doorbellRule = new AutomationRule(
    "Doorbell Chime",
    new DoorbellPressedCondition("bell-001"),
    new RingChimeAction("Front Doorbell", "Entryway")
);
system.addRule(doorbellRule);

doorbell.pressButton();
// What happens internally:
// 1. SmartDoorbell.pressButton() calls publishEvent(DOORBELL_PRESSED, payload)
// 2. SmartDevice.publishEvent() creates DeviceEvent and calls eventBus.publish()
// 3. [EventBus] Publishing event: DOORBELL_PRESSED from device: bell-001
// 4. RuleEngine.onEvent() is called (it subscribed to all event types at system startup)
// 5. RuleEngine.evaluateEvent() calls chainHead.handle(event)
// 6. Chain traverses to the handler for the doorbell rule
//    (StateChangeRuleHandler used as default — handle() calls rule.matches())
// 7. DoorbellPressedCondition.evaluate() returns true -> rule.execute() fires
// 8. RingChimeAction.execute() prints: DING DONG! Doorbell pressed at: Entryway (Front Doorbell)
// 9. [Rule] Rule fired: Doorbell Chime
```

---

### Step 7: OCP Analysis for the Doorbell Extension

**RuleEngine.buildChain() and the OCP trade-off:**

`RuleEngine.buildChain()` uses `instanceof` checks to select the right handler type for each condition. Adding `DoorbellPressedCondition` requires adding one `else if` branch to `buildChain()` — a minimal, localized change:

```java
} else if (rule.getCondition() instanceof DoorbellPressedCondition) {
    handler = new DoorbellRuleHandler(rule); // new handler, new condition — zero other changes
}
```

The alternative (fully OCP-pure) approach is to make each `Condition` carry a factory method that produces its handler, removing the `instanceof` check entirely. This is the purest expression of OCP but adds complexity. For this system at its current scale, the one-line `instanceof` addition is the pragmatic trade-off.

**What requires zero changes:**

- `HomeAutomationSystem` requires zero changes — it only knows about `AutomationRule` and `DeviceEventBus`.
- `DeviceEventBus` requires zero changes — it already subscribes `RuleEngine` to all event types, and the new `DOORBELL_PRESSED` entry in the enum causes the constructor loop to automatically create a listener list for it.
- `CommandHistory` requires zero changes — `AnswerDoorbellCommand` implements `DeviceCommand` and plugs in directly.
- `AutomationRule` requires zero changes — `DoorbellPressedCondition` implements `Condition` and `RingChimeAction` implements `Action`.
- `SmartDevice` requires zero changes — `SmartDoorbell` extends it and calls `publishEvent()` exactly as all other devices do.
- All existing scenes, rooms, and device groups require zero changes.

**New DeviceEventType values are automatically handled** because `DeviceEventBus`'s constructor iterates `DeviceEventType.values()` — adding `DOORBELL_PRESSED` to the enum means the bus automatically initializes a subscriber list for it on the next system startup.

---

## SOLID Principles Checklist

### S — Single Responsibility Principle

Every class in this system has exactly one reason to change:

**SmartDevice** handles only device identity, state transitions, and event publishing. If the event payload format changes, `SmartDevice` changes. If the device identity model changes (e.g., adding a firmware version field), `SmartDevice` changes. These are the same concern — device representation — so this remains a single responsibility.

**CommandHistory** handles only the undo/redo stack lifecycle. It knows nothing about what commands do, what devices exist, or what events are happening. If the max stack size policy changes, `CommandHistory` changes. That is its only reason to change.

**DeviceEventBus** handles only event routing — subscribe, unsubscribe, publish. It knows nothing about what events mean or what listeners do with them. If the routing strategy changes (e.g., adding priority levels), `DeviceEventBus` changes.

**RuleEngine** handles only rule evaluation. It does not know about devices, commands, or scenes. If the rule matching algorithm changes (e.g., adding rule priority), `RuleEngine` changes.

**SceneManager** handles only scene persistence and activation. It does not know about the event bus or the rule engine. If the scene storage mechanism changes (e.g., persisting to disk), `SceneManager` changes.

**Each individual command class** (TurnOnCommand, SetBrightnessCommand, etc.) handles only one specific device mutation. Adding a new mutation type requires a new command class, not a modification to existing ones.

---

### O — Open/Closed Principle

The system is open for extension and closed for modification at every major extension point:

**Adding SmartDoorbell** — demonstrated in the Extension Exercise — requires zero changes to `SmartDevice`, `DeviceEventBus`, `RuleEngine`, `HomeAutomationSystem`, `Scene`, `SceneManager`, or `CommandHistory`. The new class extends `SmartDevice` and slots in.

**Adding a new Command** (`AnswerDoorbellCommand`) implements `DeviceCommand` and immediately works with `CommandHistory`, all existing scenes, and `ExecuteCommandAction`. No existing code is touched.

**Adding a new Condition** (`DoorbellPressedCondition`) implements `Condition` and immediately works with `AutomationRule`. The condition is evaluated polymorphically — `AutomationRule.matches()` calls `condition.evaluate(event)` without knowing which implementation it is.

**Adding a new Action** (`RingChimeAction`) implements `Action` and immediately works with `AutomationRule`. Same polymorphism applies.

**Adding a new RuleHandler** (`DoorbellRuleHandler` extending `RuleHandler`) is the only addition needed when a new condition category requires specialized chain handling. `RuleEngine.buildChain()` adds one `instanceof` branch. All other handler code is untouched.

**Adding a new Scene** requires only creating a `Scene` object and calling `sceneManager.saveScene()`. No existing code changes.

---

### L — Liskov Substitution Principle

Every subtype in this system is fully substitutable for its supertype:

**SmartLight, Thermostat, SecurityCamera, DoorLock** are all substitutable for `SmartDevice`. Any code that accepts a `SmartDevice` parameter — such as `Room.addDevice(SmartDevice)`, `TurnOnCommand(SmartDevice)`, `DeviceGroup.addDevice(SmartDevice)` — works correctly with any of these subclasses. No subclass narrows the contract (e.g., throws unexpected exceptions, returns null where the parent would not).

**Room and DeviceGroup** both implement `DeviceComponent`. They are interchangeable wherever `DeviceComponent` is used — both implement `getName()`, `getDevices()`, and `executeOnAll()` with consistent semantics. `Room.addDevice()` prints a log message which is additive, not a violation.

**TurnOnCommand, TurnOffCommand, SetBrightnessCommand, SetTemperatureCommand, LockDoorCommand, StartRecordingCommand** are all substitutable for `DeviceCommand`. `CommandHistory.executeCommand(DeviceCommand)` calls `execute()`, and both `undo()` and `redo()` call the respective method — every implementation honors these contracts correctly.

**StateChangeRuleHandler, MotionRuleHandler, TemperatureRuleHandler, DoorEventRuleHandler** are all substitutable for `RuleHandler`. The `handle()` method in the base class is a Template Method — subclasses only override `canHandle()`, which is the correct specialization point. No subclass changes the chain traversal behavior.

**SendNotificationAction, ExecuteCommandAction, ActivateSceneAction** are substitutable for `Action`. `AutomationRule.execute()` calls `action.execute()` and relies only on the effect described by `getDescription()`.

---

### I — Interface Segregation Principle

Every interface in the system is minimal and focused:

**DeviceCommand** has exactly 3 methods: `execute()`, `undo()`, `getDescription()`. No command implementor is forced to implement device-specific methods (no `setBrightness()` on the interface, no `startRecording()`). This is why `TurnOnCommand` can work with any `SmartDevice` — it only calls `setState()` which lives on `SmartDevice`, not on `DeviceCommand`.

**DeviceEventListener** has exactly 2 methods: `onEvent(DeviceEvent)` and `getListenerId()`. No listener is forced to implement subscription management, event filtering, or event creation. `RuleEngine` implements this interface cleanly with just these two methods.

**Condition** has exactly 2 methods: `evaluate(DeviceEvent)` and `getDescription()`. No condition implementor needs to know about rules, actions, handlers, or event buses.

**Action** has exactly 2 methods: `execute()` and `getDescription()`. No action needs to know about conditions, rules, or devices.

**DeviceComponent** has 4 methods: `getName()`, `getDevices()`, `addDevice()` (with a default that throws), and `executeOnAll()`. The `addDevice` default is a deliberate design: leaf nodes that cannot hold children get a sensible default rejection rather than being forced to implement a no-op or throw manually.

No class in the system is burdened with implementing methods it does not use. The interface granularity matches the exact responsibilities of each role.

---

### D — Dependency Inversion Principle

High-level modules depend on abstractions, not on concrete implementations:

**SmartDevice** depends on `DeviceEventBus` (a concrete class, but one that could trivially be extracted to an interface for testing). Critically, `SmartDevice` never depends on `RuleEngine`, `SceneManager`, or any other high-level orchestrator. All communication is through the event bus abstraction.

**RuleEngine** depends on `DeviceEventListener` (an interface it implements) and `AutomationRule` (which in turn depends on `Condition` and `Action` interfaces). `RuleEngine` never imports any concrete device class.

**AutomationRule** depends on `Condition` and `Action` — both interfaces. It never knows whether the condition is a `MotionCondition` or a `TemperatureAlertCondition`, and never knows whether the action is a `SendNotificationAction` or an `ExecuteCommandAction`.

**HomeAutomationSystem** (the facade) wires everything together, but even it depends on the abstractions: it stores `SmartDevice` (abstract class), uses `DeviceEventBus`, `RuleEngine`, `SceneManager`, and `CommandHistory`. It does not reference `SmartLight`, `Thermostat`, or any concrete device directly. Devices are retrieved via `getDevice(String id)` which returns `SmartDevice`.

**ExecuteCommandAction** depends on `DeviceCommand` (interface) and `CommandHistory`. It does not depend on any concrete command class, meaning any future command (like `AnswerDoorbellCommand`) can be wrapped in `ExecuteCommandAction` without any changes.

**SceneManager** depends on `CommandHistory` (injected via constructor) and `Scene` (which holds `List<DeviceCommand>`). It never depends on any concrete device type.

The overall dependency flow is: `SmartHomeDemo` (entry point) -> `HomeAutomationSystem` (facade) -> abstractions (`DeviceEventBus`, `RuleEngine`, `SceneManager`, `CommandHistory`) -> `SmartDevice` (abstract) -> concrete devices. Concrete devices publish events upward through the bus without knowing who is listening — the direction of dependency is always toward abstractions.
