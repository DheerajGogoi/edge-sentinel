Yes — the previous plan covered **what to study and when**, but not the actual **how to build Edge Sentinel**.

I'd approach it as a **progressive build**, where every phase leaves you with a working version. You shouldn't wait until Week 10 to assemble the whole thing.

# Edge Sentinel — Actual Build Roadmap

## What we're ultimately building

```text
                         ┌──────────────────┐
                         │      iPhone      │
                         │                  │
                         │ Dashboard        │
                         │ Arm/Disarm       │
                         │ Alerts            │
                         │ Device control   │
                         └────────┬─────────┘
                                  │ HTTPS
                                  ▼
                     ┌────────────────────────┐
                     │     Cloud/VPS          │
                     │                        │
                     │ React Dashboard        │
                     │ Node.js API            │
                     │ MQTT Broker             │
                     │ MongoDB                │
                     └───────────┬────────────┘
                                 │
                              MQTT
                                 │
              ┌──────────────────▼──────────────────┐
              │             EDGE SENTINEL           │
              │                                     │
              │ ESP32-CAM                            │
              │                                     │
              │ ┌────────┐  ┌────────┐              │
              │ │ Camera │  │  PIR   │              │
              │ └────────┘  └────────┘              │
              │                                     │
              │ ┌────────┐  ┌────────┐              │
              │ │ DHT22  │  │ OLED   │              │
              │ └────────┘  └────────┘              │
              │                                     │
              │ ┌────────┐  ┌────────┐              │
              │ │ Buzzer │  │  LED   │              │
              │ └────────┘  └────────┘              │
              │                                     │
              │          TinyML Model               │
              └─────────────────────────────────────┘
```

---

# BUILD 1 — Basic ESP32 Controller

### Week 1

Start with only:

```text
ESP32
 │
 ├── LED
 └── Button
```

### Wiring

```text
ESP32 GPIO 2 ─────── LED ──── 220Ω ─── GND

ESP32 GPIO 4 ─────── Button ───── GND
```

Configure the button using the ESP32's internal pull-up.

Your firmware should do:

```text
BOOT
 ↓
Initialize GPIO
 ↓
Read button
 ↓
Button pressed?
 ├── YES → LED ON
 └── NO  → LED OFF
```

Then add Serial logging:

```text
Button pressed
LED ON
```

### Why this matters

You're learning the complete embedded loop:

```text
physical input
     ↓
GPIO
     ↓
firmware
     ↓
GPIO
     ↓
physical output
```

---

# BUILD 2 — Add Temperature

### Week 2

Connect DHT22.

```text
DHT22
 ├── VCC → 3.3V
 ├── GND → GND
 └── DATA → GPIO
```

Your firmware becomes:

```text
BOOT
 ↓
Initialize DHT22
 ↓
Initialize OLED
 ↓
loop()
 ↓
Read temperature
 ↓
Read humidity
 ↓
Display values
 ↓
repeat
```

OLED:

```text
┌────────────────────┐
│ EDGE SENTINEL      │
│                    │
│ TEMP: 29.4 C       │
│ HUM : 67 %         │
│                    │
└────────────────────┘
```

---

# BUILD 3 — Add Motion

### Week 2

Connect:

```text
HC-SR501
       │
       ├── VCC
       ├── GND
       └── OUT → ESP32 GPIO
```

Now firmware maintains:

```cpp
bool motionDetected;
float temperature;
float humidity;
```

Your device can now understand:

```text
Temperature = 29.4°C
Humidity = 67%
Motion = TRUE
```

OLED:

```text
┌────────────────────┐
│ EDGE SENTINEL      │
│                    │
│ TEMP: 29.4 C       │
│ HUM : 67 %         │
│ MOTION: DETECTED   │
└────────────────────┘
```

---

# BUILD 4 — Local Alarm

Add the buzzer.

```text
Motion detected
       ↓
Check armed status
       ↓
     ARMED?
     /    \
   YES     NO
    ↓       ↓
 BUZZER   nothing
```

This introduces your first **state machine**.

Don't write your entire program as nested `if` statements.

Think in states:

```text
DISARMED
ARMED
ALARM
BOOTING
OFFLINE
```

For example:

```text
ARMED
 │
 ├── no motion → ARMED
 │
 └── motion → ALARM
```

---

# BUILD 5 — Give the Device Wi-Fi

### Week 3

Now connect the ESP32 to Wi-Fi.

At boot:

```text
BOOT
 ↓
Initialize sensors
 ↓
Initialize Wi-Fi
 ↓
Connect
 ↓
 ┌─────────────────┐
 │ Connected       │
 │ IP: 192.168.x.x  │
 └─────────────────┘
```

If Wi-Fi fails:

```text
Wi-Fi unavailable
 ↓
wait
 ↓
retry
 ↓
retry
 ↓
retry
```

**Do not let Wi-Fi failure freeze your sensor system.**

That's an important embedded programming lesson.

---

# BUILD 6 — Send Telemetry to Backend

### Week 3–4

Create your Node backend:

```text
backend/
├── src/
│   ├── server.js
│   ├── routes/
│   ├── controllers/
│   ├── services/
│   └── models/
└── package.json
```

Create:

```text
POST /api/telemetry
```

ESP32 sends:

```json
{
  "device_id": "ES-001",
  "temperature": 29.4,
  "humidity": 67,
  "motion": true,
  "timestamp": 1787091234
}
```

Backend receives it.

First simply print:

```text
Telemetry received:

Device: ES-001
Temperature: 29.4
Humidity: 67
Motion: TRUE
```

Then store it in MongoDB.

---

# BUILD 7 — Replace HTTP With MQTT

### Week 4

Now introduce MQTT.

Run Mosquitto locally.

Architecture:

```text
ESP32
   │
   │ publish
   ▼
MQTT Broker
   │
   │ subscribe
   ▼
Node.js
```

Create topics:

```text
edge/ES-001/telemetry
edge/ES-001/events
edge/ES-001/status
edge/ES-001/commands
edge/ES-001/config
```

### Telemetry

ESP32:

```text
publish
edge/ES-001/telemetry
```

Payload:

```json
{
  "temperature": 29.4,
  "humidity": 67,
  "motion": false
}
```

Backend subscribes:

```text
edge/+/telemetry
```

Now the backend can receive telemetry from **any number of devices**.

That's much closer to real IoT architecture.

---

# BUILD 8 — Command Channel

This is where it starts becoming cool.

The backend publishes:

```text
edge/ES-001/commands
```

Payload:

```json
{
  "command": "ARM"
}
```

ESP32 subscribes to:

```text
edge/ES-001/commands
```

and executes:

```text
ARM
 ↓
set armed = true
 ↓
OLED → SYSTEM ARMED
```

Similarly:

```json
{
  "command": "BUZZER_ON"
}
```

or:

```json
{
  "command": "DISARM"
}
```

Now you have **two-way IoT communication**.

```text
ESP32 ──────────→ Backend
 telemetry

ESP32 ←────────── Backend
 commands
```

---

# BUILD 9 — Device State

### Week 5

Create a device registry.

MongoDB:

```text
devices
```

Example:

```json
{
  "deviceId": "ES-001",
  "name": "Bedroom Sentinel",
  "firmware": "1.0.0",
  "status": "ONLINE",
  "armed": true,
  "lastSeen": "2026-08-20T..."
}
```

ESP32 sends heartbeat:

```text
edge/ES-001/status
```

every 30 seconds.

Backend updates:

```text
lastSeen
```

If:

```text
NOW - lastSeen > 90 seconds
```

mark:

```text
OFFLINE
```

---

# BUILD 10 — Dashboard

### Week 6

Build React dashboard.

The dashboard talks to your backend:

```text
React
 ↓
REST API
 ↓
Node.js
 ↓
MongoDB
```

For live updates:

```text
ESP32
 ↓
MQTT
 ↓
Node.js
 ↓
WebSocket
 ↓
React
```

Dashboard:

```text
┌──────────────────────────────────────┐
│          EDGE SENTINEL               │
├──────────────────────────────────────┤
│                                      │
│ Device       ES-001                  │
│ Status       🟢 ONLINE               │
│ Armed        🔴 YES                  │
│                                      │
│ Temperature  29.4°C                  │
│ Humidity     67%                     │
│ Motion       No                      │
│                                      │
│ Uptime       14h 32m                 │
│ WiFi RSSI    -58 dBm                 │
│                                      │
├──────────────────────────────────────┤
│ TEMPERATURE                          │
│          📈                          │
│                                      │
├──────────────────────────────────────┤
│ MOTION EVENTS                        │
│                                      │
│ 10:42 PM  Motion detected            │
│ 09:17 PM  Motion detected            │
└──────────────────────────────────────┘
```

---

# BUILD 11 — iPhone Control

### Week 7

Make the dashboard responsive.

Open:

```text
https://your-server/edge
```

on Safari.

Now your iPhone becomes your controller.

Buttons:

```text
┌────────────────────────┐
│ EDGE SENTINEL           │
│                         │
│ 🟢 ONLINE               │
│                         │
│ [ 🔴 ARM ]              │
│                         │
│ Motion       ON         │
│ AI           ON         │
│                         │
│ [BUZZER] [LED]         │
│                         │
└────────────────────────┘
```

When you tap ARM:

```text
iPhone
 ↓
React
 ↓
POST /api/devices/ES-001/arm
 ↓
Node.js
 ↓
MQTT publish
 ↓
ESP32
 ↓
armed = true
```

---

# BUILD 12 — Add Camera

### Week 8

Now bring in the ESP32-CAM.

At this point you already have:

```text
Wi-Fi
MQTT
Backend
Dashboard
Device management
```

So don't make the camera project a separate thing.

Integrate it.

First test:

```text
ESP32-CAM
 ↓
Capture JPEG
 ↓
HTTP
 ↓
Browser
```

Then:

```text
PIR detects motion
       ↓
ESP32-CAM captures image
       ↓
Image uploaded
       ↓
Backend stores image/event
       ↓
Dashboard displays event
```

For example:

```text
🚨 MOTION EVENT

Time: 22:41

[ captured image ]

Motion detected
Temperature: 29.8°C
```

---

# BUILD 13 — AI

### Week 9

Only now introduce the ML model.

Your pipeline:

```text
Camera
 ↓
Image
 ↓
Resize
 ↓
Normalize
 ↓
TinyML model
 ↓
Prediction
```

Example:

```text
PERSON     94%
EMPTY       4%
OTHER       2%
```

Then firmware:

```text
IF person confidence > 80%
       ↓
create AI event
```

MQTT:

```text
edge/ES-001/events
```

Payload:

```json
{
  "type": "AI_DETECTION",
  "class": "person",
  "confidence": 0.94
}
```

Backend stores it.

Dashboard:

```text
🚨 PERSON DETECTED

Confidence: 94%
Device: ES-001
Time: 22:41
```

---

# BUILD 14 — Sensor Fusion

This is where I'd make Edge Sentinel genuinely interesting.

Don't rely only on AI.

Combine:

```text
PIR
 │
 ├──────┐
Camera │
 │     │
DHT22  │
 │     │
 │     ▼
 └──→ Decision Engine
          │
          ▼
        EVENT
```

Example:

```text
PIR = TRUE
Camera = PERSON 94%
Time = 02:15 AM
System = ARMED
```

Decision:

```text
HIGH PRIORITY ALERT
```

But:

```text
PIR = TRUE
Camera = EMPTY
```

might be:

```text
LOW PRIORITY
```

This introduces **sensor fusion** and event-processing logic.

---

# BUILD 15 — iPhone Notifications

Now when:

```text
PERSON DETECTED
```

your phone receives an alert.

Conceptually:

```text
ESP32
 ↓
AI detection
 ↓
MQTT
 ↓
Backend
 ↓
Notification service
 ↓
iPhone
```

Notification:

> 🚨 Edge Sentinel
> Person detected at Bedroom Sentinel
> Confidence: 94%

This is where the project starts feeling like a real product.

---

# BUILD 16 — OTA

### Week 11

Backend stores:

```text
latestFirmware = 1.2.0
```

Device reports:

```text
firmware = 1.1.0
```

Backend tells it:

```text
UPDATE_AVAILABLE
```

ESP32 downloads firmware.

```text
1.1.0
 ↓
download 1.2.0
 ↓
verify
 ↓
flash
 ↓
reboot
 ↓
report 1.2.0
```

Dashboard:

```text
ES-001

Firmware
1.2.0 ✓
```

---

# BUILD 17 — Power Monitoring

Add INA219.

```text
Power supply
     │
     ▼
   INA219
     │
     ▼
   ESP32
```

Measure:

```text
Voltage
Current
Power
```

Send:

```json
{
  "voltage": 4.91,
  "current": 86,
  "power": 0.42
}
```

Dashboard:

```text
POWER

4.91 V
86 mA
0.42 W
```

Now you can experiment with:

```text
WiFi always ON
vs
WiFi sleep
vs
deep sleep
```

and actually measure the difference.

---

# BUILD 18 — Physical Device

Only **after everything works on breadboard**.

Don't rush this.

Move from:

```text
breadboard
jumper wires
random modules
```

to:

```text
ESP32
   │
   ├── DHT22
   ├── PIR
   ├── OLED
   ├── buzzer
   ├── LED
   └── INA219
```

Use perfboard or a custom mounting board.

Then put everything inside an enclosure.

Something like:

```text
┌───────────────────────────────┐
│          EDGE SENTINEL        │
│                               │
│          ┌─────────┐          │
│          │ CAMERA  │          │
│          └─────────┘          │
│                               │
│  🌡                         🔴 │
│  DHT                       LED │
│                               │
│       ┌───────────────┐       │
│       │     OLED      │       │
│       └───────────────┘       │
│                               │
│          PIR SENSOR            │
│                               │
└───────────────────────────────┘
```

---

# The key development rule

**Don't build the whole thing at once.**

Your milestones should be:

### Milestone 1

```text
ESP32
↓
LED
↓
Button
```

### Milestone 2

```text
ESP32
↓
Sensors
↓
OLED
↓
Buzzer
```

### Milestone 3

```text
ESP32
↓
Wi-Fi
↓
Backend
```

### Milestone 4

```text
ESP32
↓
MQTT
↕
Backend
```

### Milestone 5

```text
Backend
↕
ESP32
↓
Dashboard
```

### Milestone 6

```text
iPhone
↕
Dashboard
↕
Backend
↕
ESP32
```

### Milestone 7

```text
ESP32-CAM
↓
Images
↓
Backend
```

### Milestone 8

```text
ESP32-CAM
↓
TinyML
↓
Person detection
```

### Milestone 9

```text
AI
+
PIR
+
Temperature
+
Rules
↓
Intelligent events
```

### Milestone 10

```text
OTA
+
Power monitoring
+
Linux deployment
+
Physical enclosure
```

---

# What "finished" looks like

Imagine you leave home with Edge Sentinel running.

From your iPhone:

```text
EDGE SENTINEL

🟢 ONLINE

Bedroom Sentinel

Temperature       28.9°C
Humidity          65%
Motion            Clear

AI                ON
System            ARMED

Power             0.41 W
WiFi              -57 dBm

────────────────────

LAST EVENTS

11:42 PM
🚨 Person detected
94% confidence

11:41 PM
Motion detected

────────────────────

[ DISARM ]

[ VIEW CAMERA ]

[ DEVICE SETTINGS ]
```

And the physical device is independently capable of:

```text
detecting
    ↓
processing
    ↓
making a local AI decision
    ↓
communicating
    ↓
receiving commands
    ↓
acting on commands
```

**That is the part I'd focus on:** you're not merely assembling components. You're learning how to design and build an end-to-end **edge computing + IoT system** from the hardware all the way to the iPhone UI.
