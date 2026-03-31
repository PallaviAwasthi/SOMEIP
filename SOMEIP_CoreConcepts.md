# SOME/IP — Day 1: Core Concepts and Mental Model

---

## 1. What Is SOME/IP?

**SOME/IP** = **S**calable **s**ervice-**O**riented **M**iddle**w**are over **IP**

It is an automotive middleware communication protocol designed by AUTOSAR for Ethernet-based in-vehicle networks. It defines:

- **How** ECUs expose and consume services (the wire format)
- **How** services are discovered on the network (via SOME/IP-SD — Service Discovery)

> Think of it as a lightweight RPC + pub/sub framework, purpose-built for automotive constraints (determinism, low latency, real-time).

---

## 2. Why Automotive Ethernet?

### The old world: CAN bus
- Fixed-size frames (8 bytes max)
- Low bandwidth (~500 kbps to 1 Mbps)
- Broadcast-only — every ECU hears everything
- Simple and reliable, but doesn't scale for modern vehicles

### The new world: Automotive Ethernet
| Feature | CAN | Automotive Ethernet |
|---|---|---|
| Bandwidth | 1 Mbps | 100 Mbps – 10 Gbps |
| Frame size | 8 bytes | Up to 1500 bytes |
| Addressing | Broadcast only | Unicast, Multicast, Broadcast |
| Use cases | Simple signals | Video, ADAS, OTA, diagnostics |
| Protocol overhead | Very low | Higher, but manageable |

Modern vehicles have cameras, LiDAR, OTA updates, ADAS — CAN simply cannot carry this data. Ethernet is the backbone, and SOME/IP is the application-layer protocol running on top of TCP/UDP + IP.

---

## 3. Why Service-Oriented Communication?

### Signal-based (classic CAN approach)
- ECU A sends a CAN frame with a fixed signal ID at a fixed rate (e.g., every 10 ms)
- ECU B always listens, even when it doesn't need the data
- **Problem**: Wastes bandwidth; tight coupling; every consumer must know every producer

### Service-oriented (SOME/IP approach)
- ECU A **offers a service** (e.g., "IndicatorService")
- ECU B **subscribes** to events or **calls methods** only when needed
- **Benefits**:
  - Decoupled — producer doesn't need to know who's listening
  - On-demand — data flows only when needed
  - Scalable — add new consumers without changing the producer

> Analogy: Signal-based is like a radio broadcast. Service-oriented is like a streaming service — you subscribe to what you want.

---

## 4. What SOME/IP Supports

### 4.1 Remote Procedure Calls (RPC)
- A client ECU calls a **method** on a server ECU — like a function call over the network
- Can be **Request/Response** (client waits for result) or **Fire & Forget** (client doesn't wait)

```
Cluster  ──── REQUEST: GetDoorStatus() ────►  BCM
Cluster  ◄─── RESPONSE: CLOSED ─────────────  BCM
```

### 4.2 Event Notifications (Pub/Sub)
- A server ECU publishes **events** when state changes
- Clients subscribe once; they receive updates automatically
- No polling needed

```
BCM  ──── EVENT: IndicatorState = LEFT ────►  Cluster
BCM  ──── EVENT: IndicatorState = OFF  ────►  Cluster
```

### 4.3 Serialization / Wire Format
- SOME/IP defines exactly how data is packed into bytes on the wire
- Header: fixed 16 bytes
- Payload: serialized application data (integers, strings, structs, arrays)
- Both big-endian and little-endian supported (configured per deployment)

---

## 5. Core Terms

| Term | What It Is | Example |
|---|---|---|
| **Service ID** | Identifies the service type | `0x0420` = IndicatorService |
| **Instance ID** | Identifies a specific instance of a service (multiple instances possible) | `0x0001` = first BCM instance |
| **Method ID** | Identifies a method (RPC call) within a service | `0x0421` = SetIndicator |
| **Event ID** | Identifies an event (notification) within a service | `0x8001` = IndicatorStateChanged |
| **Client ID** | Identifies the calling ECU/client | Assigned by SOME/IP-SD or configured statically |
| **Session ID** | Per-client counter to match request with response | Incremented with each request |
| **Interface Version** | Version of the service interface | `0x01` |
| **Message Type** | What kind of message this is | REQUEST, RESPONSE, NOTIFICATION, ERROR |

> **Event IDs** always start at `0x8000` or higher by AUTOSAR convention — this distinguishes them from Method IDs.

### SOME/IP Header Layout (16 bytes)
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Service ID           |           Method ID           |  ← 4 bytes
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Length                                |  ← 4 bytes (payload length + 8)
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Client ID           |          Session ID           |  ← 4 bytes
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Proto Ver  |  Iface Ver  |  Msg Type   |   Return Code   |   ← 4 bytes
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Payload ...                           |
```

---

## 6. Method vs Event — Detailed Explanation

### The Core Mental Model

Think of it from the perspective of **who initiates the communication**:

- **Method** → the **client** decides when something happens
- **Event** → the **server** decides when something happens

---

### Method (RPC — Remote Procedure Call)

A method is like calling a function on another ECU across the network.

**Two sub-types:**

#### 1. Request / Response
```
Cluster                          BCM
   |                              |
   |── REQUEST: GetDoorStatus() ─►|   ← Cluster asks
   |                              |   (BCM processes)
   |◄─ RESPONSE: {FL=closed,      |   ← BCM answers
   |    FR=open, RL=closed...}    |
   |                              |
```
- Cluster **blocks** (or registers a callback) waiting for the reply
- Session ID links the request to its response
- Used when the client **needs a value** or wants to **trigger an action and confirm it**

#### 2. Fire & Forget (Request No Return)
```
Cluster                          BCM
   |                              |
   |── REQUEST: SetIndicator(LEFT)►|   ← Cluster commands
   |                              |   (BCM does it, no reply)
   |   (no response expected)     |
```
- No waiting, no reply
- Used when the client **just wants to trigger something** and doesn't care about confirmation

**When do you use a method?**
- "Give me the current door status" → Request/Response
- "Activate the horn" → Fire & Forget
- "Set fan speed to level 3" → either, depending on whether you need confirmation

---

### Event (Pub/Sub — Publish / Subscribe)

An event is a **notification** the server sends **on its own initiative** when something changes. The client doesn't ask — it just subscribes once and waits.

**Flow:**
```
         [One-time setup via SOME/IP-SD]

Cluster                          BCM
   |── SubscribeEventgroup(0x0001)►|   ← Cluster signs up once
   |◄─ SubscribeAck ──────────────|   ← BCM confirms

         [Later, when state changes]

   |                              |   [User presses left stalk]
   |◄─ NOTIFICATION: LEFT ────────|   ← BCM fires immediately
   |                              |
   |                              |   [Turn completes, auto-cancel]
   |◄─ NOTIFICATION: OFF ─────────|   ← BCM fires again
   |                              |
   |                              |   [User presses hazard]
   |◄─ NOTIFICATION: HAZARD ──────|   ← BCM fires again
```

The Cluster never asked any of those times. BCM decided when to send.

**When do you use an event?**
- "Notify me whenever the indicator state changes"
- "Tell me whenever a door opens or closes"
- "Send me engine RPM updates at 10 Hz"

---

### Side-by-Side Comparison

| Aspect | Method | Event |
|---|---|---|
| **Who triggers it** | Client | Server |
| **Pattern** | RPC | Pub/Sub |
| **Setup** | Just send a request | Subscribe first, then receive |
| **Frequency** | Once per client request | Whenever server state changes |
| **Response** | Yes (or Fire & Forget) | No — one-way notification |
| **ID range** | `0x0001` – `0x7FFF` | `0x8000` – `0xFFFF` |
| **Message type byte** | `0x00` REQUEST / `0x80` RESPONSE | `0x02` NOTIFICATION |
| **Latency** | Round-trip (request + response) | Immediate on state change |

---

### Why the ID ranges are split that way

AUTOSAR reserved the top half of the 16-bit ID space for events:

```
0x0000          0x7FFF 0x8000         0xFFFF
|─── Methods ───────────|─── Events ──────────|
```

This lets any parser look at a single field and immediately know: **"Is this a callable method or a subscribable event?"** — without needing extra metadata. When you see `0x8001` in the SOME/IP header, you know before parsing anything else that it's an event notification.

---

### Concrete BCM Example

```cpp
// This is a METHOD — Cluster calls it to get current door state
static constexpr vsomeip::method_t METHOD_GET_DOOR  = 0x0421;  // 0x0421 < 0x8000 → method

// This is an EVENT — BCM fires it when indicator state changes
static constexpr vsomeip::event_t  EVENT_INDICATOR  = 0x8001;  // 0x8001 ≥ 0x8000 → event
```

- Cluster calls `0x0421` → BCM receives the request, queries door sensors, sends response back
- BCM fires `0x8001` → all subscribed Clusters receive it instantly, no request needed

---

### The Key Intuition

> A **method** is a question or a command — the client is in control.
> An **event** is an announcement — the server is in control.

In a car, most real-time UI updates (tell-tales, warnings, speed) are **events** because the BCM/engine ECU knows when state changes and must push it immediately. Methods are for **one-off queries** ("what's the VIN?") or **explicit commands** ("unlock the doors").

---

## 7. Why BCM Publishes Events Instead of Waiting for Cluster to Poll

**Polling** = Cluster asks BCM every N ms: "What's the indicator state?"
- Wastes bandwidth even when state hasn't changed
- Adds latency (worst case = polling interval)
- BCM does unnecessary processing answering identical queries

**Event publishing** = BCM sends a notification only when state changes
- Zero bandwidth when nothing changes
- Near-zero latency (event sent immediately on change)
- BCM controls the notification — it knows when state changes

> In safety-critical UI (instrument cluster), the tell-tale must update within ~100ms of the physical button press. A 100 ms polling loop is the worst case; an event fires immediately.

---

## 8. Practical Model: BCM → Cluster

### BCM (Body Control Module) — Service Provider

```cpp
// Service: IndicatorService
static constexpr vsomeip::service_t  SERVICE_ID   = 0x0420;
static constexpr vsomeip::instance_t INSTANCE_ID  = 0x0001;
static constexpr vsomeip::method_t   METHOD_ID    = 0x0421;
static constexpr vsomeip::event_t    EVENT_ID     = 0x8001;
static constexpr vsomeip::eventgroup_t EVENTGROUP = 0x0001;

// BCM owns these states:
enum class IndicatorSide : uint8_t {
    LEFT   = 0,
    RIGHT  = 1,
    HAZARD = 2
};

struct DoorStatus {
    bool front_left;
    bool front_right;
    bool rear_left;
    bool rear_right;
};
```

**BCM's responsibilities:**
- Offer `IndicatorService` on the network
- Publish `EVENT_ID_INDICATOR` when indicator state changes
- Respond to method calls (e.g., `SetIndicator`, `GetDoorStatus`)

### Cluster (Instrument Cluster) — Service Consumer

**Cluster's responsibilities:**
- Subscribe to `EVENT_ID_INDICATOR` from BCM
- On event received → update tell-tale lamp (left/right/hazard)
- Call `GetDoorStatus` method when needed → display door-ajar warning

### Communication Flow

```
BCM                                         Cluster
 |                                              |
 |  [SOME/IP-SD] OfferService(0x0420)          |
 |─────────────────────────────────────────►   |
 |                                              |
 |  [SOME/IP-SD] SubscribeEventgroup(0x0001)   |
 |◄─────────────────────────────────────────   |
 |                                              |
 |  [User presses left indicator stalk]         |
 |                                              |
 |  NOTIFICATION: EVENT 0x8001                  |
 |  payload: IndicatorSide::LEFT               |
 |─────────────────────────────────────────►   |
 |                                              |
 |                              [Cluster updates left tell-tale]
 |                                              |
 |  [Indicator auto-cancels after turn]         |
 |  NOTIFICATION: EVENT 0x8001                  |
 |  payload: IndicatorSide::OFF                |
 |─────────────────────────────────────────►   |
```

---

## 9. SOME/IP vs SOME/IP-SD

| | SOME/IP | SOME/IP-SD |
|---|---|---|
| **Role** | Communication protocol (wire format, RPC, events) | Service Discovery |
| **What it does** | Carries the actual data | Announces and finds services |
| **Analogy** | HTTP (how to exchange data) | DNS (how to find servers) |
| **Port** | Application-defined | UDP 30490 (standard) |

**SOME/IP-SD does:**
- `OfferService` — BCM announces: "I have IndicatorService available"
- `FindService` — Cluster searches: "Is IndicatorService available?"
- `SubscribeEventgroup` — Cluster subscribes to event groups
- `StopOfferService` — BCM going offline

---

## 10. End-of-Day Checkpoint

> **SOME/IP is a service-oriented automotive protocol over IP that supports request/response and event-based communication, while SOME/IP-SD handles service availability and pub/sub discovery.**

You should now be able to answer:

**Q: What is a service?**
A service is a logical unit of functionality offered by an ECU. It has a unique Service ID + Instance ID and exposes methods (callable by clients) and events (published to subscribers). Example: `IndicatorService` offered by BCM.

**Q: What is the difference between a method and an event?**
A method is invoked by a client (RPC pattern — client controls when it happens). An event is published by the server when something changes (pub/sub pattern — server controls when it happens). Methods use IDs `0x0001–0x7FFF`; events use `0x8000–0xFFFF`.

**Q: Why would BCM publish an event instead of waiting for Cluster to poll?**
Events fire immediately on state change — lower latency, zero bandwidth when idle. Polling wastes bandwidth and adds worst-case latency equal to the polling interval. For a tell-tale that must respond within 100 ms of a physical input, event-driven is the correct model.

---

## 11. Quick Reference Card

```
SOME/IP Message Types:
  0x00  REQUEST           (Client → Server, expects response)
  0x01  REQUEST_NO_RETURN (Fire & Forget)
  0x02  NOTIFICATION      (Server → Subscriber, event)
  0x80  RESPONSE          (Server → Client, reply to request)
  0x81  ERROR             (Server → Client, error reply)

ID Ranges (AUTOSAR convention):
  Service ID:     0x0001 – 0xFFFE
  Instance ID:    0x0001 – 0xFFFE  (0xFFFF = any)
  Method ID:      0x0001 – 0x7FFF
  Event ID:       0x8000 – 0xFFFF
  Eventgroup ID:  0x0001 – 0xFFFF

Transport:
  UDP — preferred for events/notifications (low overhead)
  TCP — used for reliable method calls with large payloads
```

---

---

## Day 2 — SOME/IP packet/header and payload design

### Goal

Understand the wire-level message format.

AUTOSAR defines the SOME/IP protocol structure, including the message header fields and semantics.

Every SOME/IP message on the wire starts with a **fixed 16-byte header**, followed by application payload. The header is always big-endian (network byte order).

---

### Header fields — detailed explanation

#### 1. Service ID (2 bytes)

Identifies **which service** the message belongs to.
Think of it as the "address" of the service provider ECU's functionality.

- Example: `0x1234` = VehicleBodyStatusService (BCM)
- Assigned statically at design time by the team.
- Both provider (BCM) and consumer (Cluster) must agree on this value.

---

#### 2. Method ID / Event ID (2 bytes)

Identifies **which operation or event** within that service.

- **Method IDs**: `0x0000` – `0x7FFF` → request/response calls (MSB = 0)
- **Event IDs**: `0x8000` – `0xFFFF` → notifications/events (MSB = 1)

This MSB convention lets the receiver immediately know: "Is this a method call or an event notification?"

- Example method: `0x0421` = `GetCurrentIndicatorStatus`
- Example event:  `0x8001` = `IndicatorStatusEvent`

---

#### 3. Message ID = Service ID + Method/Event ID (combined = 4 bytes)

The first two fields together form the **Message ID** — a 32-bit combined identifier.

```text
[Service ID: 16 bits][Method/Event ID: 16 bits]
```

This uniquely identifies the "type" of communication.

---

#### 4. Length (4 bytes)

The number of bytes that follow this `Length` field itself.

```text
Length = 8 (remaining fixed header) + len(payload)
```

So a message with 10 bytes of payload has `Length = 18`.

- The receiver uses this to know how many bytes to read.
- Minimum value: `8` (empty payload, only header remaining).

---

#### 5. Client ID (2 bytes)

Identifies **which client** (consumer ECU) sent the request.

- Assigned statically to each ECU in the system.
- Example: Cluster ECU = `0x0021`
- For notifications (server → clients), Client ID is `0x0000` because no specific client initiated it.

Why it matters: In a system with multiple ECUs subscribing to the same service, the provider needs to know who sent which request to route the response correctly.

---

#### 6. Session ID (2 bytes)

A **sequence counter** that uniquely identifies each individual request-response pair within a client session.

- The client increments this for every new request sent.
- The server echoes the same Session ID back in the response.
- This is how the client matches a response to the original request, especially when multiple requests are in-flight.
- For notifications (unsolicited events), Session ID is typically `0x0000` or increments monotonically.

---

#### 7. Request ID = Client ID + Session ID (combined = 4 bytes)

```text
[Client ID: 16 bits][Session ID: 16 bits]
```

Together they form a **Request ID** — a 32-bit token used to correlate requests and responses end-to-end.

**Why both are needed:**
- Client ID tells you *who* sent the request.
- Session ID tells you *which* request from that client this corresponds to.
- Without Session ID, if Cluster sends two `GetCurrentIndicatorStatus` requests quickly, BCM's two responses would be ambiguous.
- Without Client ID, if two different ECUs send the same Session ID (both started at 1), the response correlation breaks.

---

#### 8. Protocol Version (1 byte)

Version of the SOME/IP wire protocol itself.

- Currently always `0x01`.
- If a message arrives with an unexpected protocol version, the receiver should discard it.
- This is separate from the service interface version — it describes the framing format, not the service API.

---

#### 9. Interface Version (1 byte)

Version of the **service interface** (the API contract).

- Defined per service by the design team.
- Example: `0x01` = version 1 of VehicleBodyStatusService.
- If BCM upgrades its interface (adds fields, changes semantics) it bumps this.
- Cluster checks this at startup — a mismatch means the two ECUs are out of sync and communication is unsafe.

---

#### 10. Message Type (1 byte)

Tells the receiver what kind of message this is:

| Value  | Name                | Description                                  |
|--------|---------------------|----------------------------------------------|
| `0x00` | REQUEST             | Client calls a method, expects a response    |
| `0x01` | REQUEST_NO_RETURN   | Client calls a method, no response expected  |
| `0x02` | NOTIFICATION        | Server sends an unsolicited event            |
| `0x80` | RESPONSE            | Server replies to a REQUEST                  |
| `0x81` | ERROR               | Server replies with an error                 |

In the BCM → Cluster indicator flow:
- BCM sends `NOTIFICATION` (0x02) for event updates.
- Cluster sends `REQUEST` (0x00) for `GetCurrentIndicatorStatus`.
- BCM replies with `RESPONSE` (0x80).

---

#### 11. Return Code (1 byte)

Indicates the result of a method call:

| Value  | Name      | Description           |
|--------|-----------|-----------------------|
| `0x00` | E_OK      | Success               |
| `0x01` | E_NOT_OK  | Generic failure       |
| `0x02` | E_UNKNOWN_SERVICE | Service not found |
| ...    | ...       | Other AUTOSAR codes   |

- Always `0x00` in REQUESTs and NOTIFICATIONs (not yet known).
- Set by the server in RESPONSE or ERROR messages.

---

#### 12. Payload (variable length)

The application data — whatever the service and method/event define.

- Serialized according to SOME/IP serialization rules (big-endian, no padding by default).
- The receiver knows the payload length from: `payload_length = Length - 8`

---

### Conceptual header layout (bit-level)

```text
Byte offset:  0       1       2       3
              +-------+-------+-------+-------+
              |     Service ID        |    Method / Event ID      |
              +-------+-------+-------+-------+
              |               Length                              |
              +-------+-------+-------+-------+
              |     Client ID         |     Session ID            |
              +-------+-------+-------+-------+
              |ProtVer|IfVer  |MsgType|RetCode|
              +-------+-------+-------+-------+
              |              Payload ...                          |
              +-------+-------+-------+-------+
```

Total fixed header = **16 bytes**.

---

### Byte-by-byte walkthrough — BCM notification example

For a BCM sending an `IndicatorStatusEvent` with LEFT + BLINKING:

```text
Byte 00–01   Service ID        : 0x12 0x34   → VehicleBodyStatusService
Byte 02–03   Event ID          : 0x80 0x01   → IndicatorStatusEvent (MSB=1 = event)
Byte 04–07   Length            : 0x00 0x00 0x00 0x12  → 18 = 8 header + 10 payload
Byte 08–09   Client ID         : 0x00 0x00   → 0 = unsolicited notification
Byte 10–11   Session ID        : 0x00 0x01   → session counter
Byte 12      Protocol Version  : 0x01
Byte 13      Interface Version : 0x01
Byte 14      Message Type      : 0x02        → NOTIFICATION
Byte 15      Return Code       : 0x00        → E_OK
Byte 16      Payload[0]        : 0x00        → side = LEFT
Byte 17      Payload[1]        : 0x02        → state = BLINKING
Byte 18–25   Payload[2..9]     : timestamp (8 bytes big-endian uint64)
```

---

### BCM → Cluster payload example

We define an application payload for indicator state:

```cpp
struct IndicatorStatus {
    uint8_t side;       // 0 = LEFT, 1 = RIGHT, 2 = HAZARD
    uint8_t state;      // 0 = OFF, 1 = ON, 2 = BLINKING
    uint64_t timestamp; // milliseconds since epoch, big-endian
};
// Wire size: 1 + 1 + 8 = 10 bytes
```

**Field details:**

| Field       | Size   | Values                    | Purpose                                |
|-------------|--------|---------------------------|----------------------------------------|
| `side`      | 1 byte | 0=LEFT, 1=RIGHT, 2=HAZARD | Which indicator the BCM is reporting   |
| `state`     | 1 byte | 0=OFF, 1=ON, 2=BLINKING   | Current state of that indicator        |
| `timestamp` | 8 bytes| uint64 milliseconds       | When the state was captured by BCM HW  |

The timestamp lets the Cluster detect stale data (e.g., if a notification arrives late after buffering).

---

### Serialization rules (SOME/IP default)

- **Big-endian** byte order (network byte order) for multi-byte integers.
- **No implicit padding** — fields are packed tightly.
- **No type tags or field names** on the wire — both sides must agree on the exact layout via the service definition.

**Serializer (BCM side):**
```cpp
std::vector<uint8_t> serialize_indicator(const IndicatorStatus& s) {
    std::vector<uint8_t> buf;
    buf.push_back(s.side);    // byte 0
    buf.push_back(s.state);   // byte 1
    // timestamp: big-endian uint64, bytes 2–9
    for (int i = 7; i >= 0; --i) {
        buf.push_back((s.timestamp >> (i * 8)) & 0xFF);
    }
    return buf;
}
```

**Deserializer (Cluster side):**
```cpp
IndicatorStatus deserialize_indicator(const uint8_t* data, size_t len) {
    if (len < 10) throw std::runtime_error("payload too short");
    IndicatorStatus s;
    s.side      = data[0];
    s.state     = data[1];
    s.timestamp = 0;
    for (int i = 0; i < 8; ++i) {
        s.timestamp = (s.timestamp << 8) | data[2 + i];
    }
    return s;
}
```

**Hex dump printer:**
```cpp
void print_hex(const std::vector<uint8_t>& buf) {
    for (size_t i = 0; i < buf.size(); ++i) {
        printf("%02X ", buf[i]);
        if ((i + 1) % 8 == 0) printf("\n");
    }
    printf("\n");
}
// Example output for LEFT + BLINKING + ts=12345678:
// 00 02 00 00 00 00 00 00  BC 61 4E
```

---

### TX ECU vs RX ECU view

#### TX ECU: BCM

1. **Fills SOME/IP header**: sets Service ID, Event ID, Length, Client ID=0, Session ID (increments), ProtVer=1, IfVer=1, MsgType=NOTIFICATION, RetCode=E_OK.
2. **Serializes payload**: converts `IndicatorStatus` struct → raw bytes (big-endian, packed).
3. **Sends notification**: vSomeIP calls `app_->notify(...)` which multicasts to all current subscribers over UDP.

#### RX ECU: Cluster

1. **Validates header**: checks Service ID matches expected, Event ID matches subscribed event, Interface Version matches what was agreed at design time. Discards mismatched messages.
2. **Deserializes payload**: reads raw bytes → reconstructs `IndicatorStatus` struct.
3. **Updates tell-tale icon**: calls display/rendering layer based on `side` and `state` values (e.g., flash left indicator icon if side=LEFT and state=BLINKING).

---

### Practice

Write these three functions standalone (no vSomeIP needed):

1. **Serializer**: `std::vector<uint8_t> serialize(const IndicatorStatus&)`
2. **Deserializer**: `IndicatorStatus deserialize(const uint8_t* data, size_t len)`
3. **Hex printer**: `void print_hex(const std::vector<uint8_t>&)`

Then test with a round-trip:

```cpp
IndicatorStatus original { 0, 2, 12345678ULL }; // LEFT, BLINKING
auto bytes = serialize(original);
print_hex(bytes);                                // visual check
auto recovered = deserialize(bytes.data(), bytes.size());
assert(recovered.side == original.side);
assert(recovered.state == original.state);
assert(recovered.timestamp == original.timestamp);
```

---

### End-of-day checkpoint

You should be able to explain:

- What each of the 16 header bytes represents.
- Why **Client ID + Session ID** together are required for matching requests/responses (Client ID identifies who, Session ID identifies which request from that client).
- Why Event IDs start at `0x8000` (MSB=1 distinguishes events from methods).
- Why **Interface Version** matters at startup and what breaks when it mismatches.
- How to manually compute `Length` given a payload size.
- Why SOME/IP serialization is big-endian and what happens if one side uses little-endian.

---

## Day 3 — Concurrent ECU Communication: How BCM handles multiple senders

### The Problem

In a real vehicle network, BCM does not communicate with just one ECU at a time. Cluster, ADAS, Gateway, and Sensor ECUs may all send messages to BCM simultaneously. This section explains exactly how BCM handles that.

---

### 1. Network layer — UDP/TCP handles simultaneous packets

SOME/IP runs over standard IP networking. The OS network stack on each ECU receives all incoming packets into a **receive buffer** (socket buffer). Multiple ECUs sending at the same time just means multiple UDP datagrams arrive in the buffer — the OS queues them. No data is lost unless the buffer overflows.

```text
Sensor ECU ──┐
ADAS ECU   ──┼──► [Ethernet Switch] ──► BCM's NIC ──► OS socket buffer ──► vSomeIP
Cluster    ──┘
```

The Ethernet switch handles concurrent traffic using its own internal queues and scheduling (typically priority queues at OSI Layer 2).

---

### 2. SOME/IP / vSomeIP layer — message demultiplexing

When packets arrive at BCM's vSomeIP instance, they are demultiplexed using the header fields:

```text
Incoming message
    │
    ├─ Service ID + Method/Event ID → "which handler to call?"
    ├─ Client ID                    → "which ECU sent this?"
    ├─ Session ID                   → "which request is this?"
    └─ Message Type                 → REQUEST / NOTIFICATION / RESPONSE?
```

vSomeIP dispatches each message to the registered handler for that `(Service ID, Method ID)` combination. This is why `Client ID + Session ID` exist — BCM can receive requests from Cluster AND ADAS at the same time and respond to each correctly.

**Example:**

```text
Cluster → BCM: REQUEST  GetDoorStatus  ClientID=0x0021 SessionID=0x0001
ADAS    → BCM: REQUEST  GetDoorStatus  ClientID=0x0042 SessionID=0x0001
```

Both have `SessionID=0x0001` but different `ClientID` — BCM sends two separate responses, each echoing back the original `Client ID + Session ID`, so Cluster gets its response and ADAS gets its own.

---

### 3. Threading model — how BCM processes concurrently

vSomeIP runs an **I/O thread** (boost::asio event loop internally) that dispatches received messages. BCM registers handlers as callbacks:

```cpp
app_->register_message_handler(SERVICE_ID, INSTANCE_ID, METHOD_GET_DOOR,
    [](const std::shared_ptr<vsomeip::message>& req) {
        // This runs on the vSomeIP dispatch thread
        handle_get_door(req);
    });
```

For a production BCM, the typical threading model is:

```text
┌─────────────────────────────────────────────┐
│  BCM Process                                 │
│                                              │
│  vSomeIP I/O thread                         │
│  ├── receives packets from all ECUs          │
│  ├── demultiplexes by Service/Method ID      │
│  └── dispatches to registered handlers       │
│                                              │
│  Application thread(s)                       │
│  ├── hardware polling (indicator stalk, etc.)│
│  ├── state machine (indicator logic)         │
│  └── calls app_->notify() on state change    │
│                                              │
│  Notifier thread                             │
│  └── periodic/triggered notify to Cluster   │
└─────────────────────────────────────────────┘
```

The vSomeIP dispatch and the application logic run on separate threads — they share state via mutexes or lock-free queues.

---

### 4. BCM as server AND client simultaneously

BCM is not just a server. In a real vehicle network, BCM is often **both**:

| Role | Who BCM talks to | What happens |
|---|---|---|
| **Server** | Cluster, ADAS subscribe to BCM events | BCM pushes `IndicatorStatusEvent`, `DoorStatusEvent` |
| **Server** | Cluster calls `GetDoorStatus()` | BCM responds |
| **Client** | BCM subscribes to `SensorFusionService` from ADAS | BCM receives proximity events to auto-fold mirrors |
| **Client** | BCM calls `SetExteriorLighting()` on LightingECU | BCM sends requests |

SOME/IP handles this cleanly because each service is independently identified by `Service ID + Instance ID`. BCM can offer its own service and simultaneously be a subscriber/requester on other services — all within the same vSomeIP application instance.

---

### 5. Backpressure and overload

If too many requests arrive simultaneously:

- **UDP**: packets beyond the socket buffer are **silently dropped** — no error, no retransmit. This is why events (notifications) use UDP (loss-tolerant) and critical method calls may use **TCP** (reliable delivery, retransmit built-in).
- **TCP**: flow control built into the stack — sender slows down if receiver is overwhelmed.
- **Application level**: handlers that take too long block the dispatch thread. Production BCMs offload slow work to a thread pool and respond asynchronously.

---

### 6. Real-time OS considerations (AUTOSAR Classic)

On AUTOSAR Classic (bare-metal RTOS, not Linux), this is handled differently:

- No POSIX threads — tasks are scheduled by the RTOS (e.g., OSEK/AUTOSAR OS)
- Communication uses **AUTOSAR COM** and **PDU Router** — not vSomeIP
- Priority-based preemption ensures safety-critical tasks (braking, airbag) always preempt lower-priority tasks (cluster display update)
- SOME/IP in AUTOSAR Adaptive (Linux-based) ECUs follows the threading model described above

---

### Summary

| Concern | How it's handled |
|---|---|
| Multiple ECUs sending at once | OS network stack queues packets; Ethernet switch arbitrates |
| Identifying which ECU sent what | `Client ID` field in every SOME/IP header |
| Matching responses to requests | `Client ID + Session ID` echo in response |
| BCM serving multiple clients simultaneously | vSomeIP dispatches each message to the correct registered handler |
| BCM being server and client at the same time | Separate service registrations; SOME/IP supports both roles in one process |
| Overload / packet loss | UDP = best-effort (events tolerate loss); TCP = reliable (method calls) |
| Thread safety | Application state protected by mutexes; vSomeIP I/O thread separate from app threads |

> **Key insight**: `Client ID + Session ID` are the correlation mechanism that let a single BCM process handle simultaneous requests from many ECUs without confusing responses.

---

## Day 4 — vSomeIP Internals: How It Works Under the Hood

### What is vSomeIP?

vSomeIP is the **COVESA open-source C++ implementation** of the SOME/IP and SOME/IP-SD protocols. It is the most widely used SOME/IP stack for AUTOSAR Adaptive (Linux-based) ECU development and prototyping.

- Built on **boost::asio** for async I/O
- Uses **Unix domain sockets** for inter-process communication on the same host
- Uses **UDP / TCP sockets** for communication between ECUs over Ethernet
- Configured via a **JSON config file**

---

### Architecture overview

```text
┌──────────────────────────────────────────────────────────┐
│  Your Application (BCM / Cluster)                        │
│  vsomeip::application API                               │
│  offer_service / notify / send / register_*_handler      │
└───────────────────┬──────────────────────────────────────┘
                    │ internal
┌───────────────────▼──────────────────────────────────────┐
│  Routing Manager                                         │
│  ├── Service Registry (who offers what)                  │
│  ├── Subscription table (who subscribed to what)         │
│  └── Message routing (local IPC or remote network)       │
└───────┬──────────────────────────┬───────────────────────┘
        │                          │
┌───────▼──────────┐    ┌──────────▼──────────────────────┐
│  Local IPC       │    │  Endpoint Manager                │
│  Unix domain     │    │  ├── UDP endpoint(s)             │
│  sockets         │    │  ├── TCP endpoint(s)             │
│  (same host)     │    │  └── SD multicast endpoint       │
└──────────────────┘    └──────────────────────────────────┘
                                   │
                        ┌──────────▼──────────────────────┐
                        │  boost::asio I/O service         │
                        │  async_send / async_receive      │
                        │  I/O threads (thread pool)       │
                        └─────────────────────────────────┘
```

---

### Core components

#### 1. `vsomeip::runtime` — the factory singleton

The entry point. You never construct vSomeIP objects with `new` — you always go through `runtime`:

```cpp
auto app = vsomeip::runtime::get()->create_application("bcm_service");
auto msg = vsomeip::runtime::get()->create_request();
auto payload = vsomeip::runtime::get()->create_payload();
```

`runtime::get()` loads the JSON config file (from `VSOMEIP_CONFIGURATION` env var or default path) and returns the singleton. All applications in the same process share one runtime.

---

#### 2. `vsomeip::application` — your main handle

This is the object your code interacts with for everything:

| Method | What it does |
|---|---|
| `init()` | Loads config, connects to routing manager, sets up endpoints |
| `start()` | Starts the boost::asio event loop — **blocks** the calling thread |
| `offer_service()` | Registers this app as the provider of a service |
| `request_service()` | Registers this app as a consumer (triggers SD FindService) |
| `offer_event()` | Declares an event this service will publish |
| `request_event()` | Declares an event this consumer wants to receive |
| `subscribe()` | Sends SubscribeEventgroup to the provider |
| `notify()` | Publishes an event to all subscribers |
| `send()` | Sends a request or response message |
| `register_message_handler()` | Registers callback for incoming messages |
| `register_state_handler()` | Callback for when the app connects to routing manager |
| `register_availability_handler()` | Callback for when a service becomes available/unavailable |

---

#### 3. Routing Manager — the central switchboard

The Routing Manager (RM) is the most important internal component. It is responsible for:

- Maintaining a **service registry**: maps `(Service ID, Instance ID)` → which application/endpoint provides it
- Maintaining a **subscription table**: maps event groups → list of subscribers
- **Routing messages**: decides whether to deliver a message via local IPC (same host) or a network endpoint (remote ECU)

**Two deployment modes:**

```text
Mode 1: Routing Manager Host (one application hosts it)
─────────────────────────────────────────────────────
  BCM App (routing_manager_host = true in config)
  └── hosts Routing Manager internally
        ├── Cluster App connects via Unix socket
        └── ADAS App connects via Unix socket

Mode 2: Standalone daemon (vsomeipd / routingmanagerd)
─────────────────────────────────────────────────────
  vsomeipd process (dedicated routing daemon)
  ├── BCM App connects via Unix socket
  ├── Cluster App connects via Unix socket
  └── ADAS App connects via Unix socket
```

All applications on the same ECU connect to the RM via Unix domain sockets. The RM then decides if the message goes to another local app or out over the network to a remote ECU.

---

#### 4. Endpoints — the network sockets

The endpoint manager creates and manages the actual OS sockets:

| Endpoint type | Protocol | Used for |
|---|---|---|
| UDP client endpoint | UDP unicast | Sending requests/responses to remote ECUs |
| UDP server endpoint | UDP unicast + multicast | Receiving messages from remote ECUs |
| TCP client endpoint | TCP | Reliable method calls to remote ECUs |
| TCP server endpoint | TCP acceptor | Accepting reliable connections from remote ECUs |
| SD endpoint | UDP multicast 224.0.0.1:30490 | SOME/IP-SD discovery messages |

Each endpoint uses **boost::asio async I/O**:
- `async_receive` → puts received data into a ring buffer → parsed into SOME/IP messages → dispatched to RM
- `async_send` → queues outgoing messages → sent in order without blocking the application

---

#### 5. boost::asio I/O service — the event loop engine

Everything async in vSomeIP runs through boost::asio:

```text
boost::asio::io_service
    │
    ├── async_receive (UDP/TCP socket read)
    ├── async_send    (UDP/TCP socket write)
    ├── async_wait    (SD timers — offer interval, TTL expiry)
    └── post()        (dispatch message to application handler)
```

vSomeIP starts a configurable number of I/O threads that all call `io_service.run()`. All callbacks (receive events, timer expiry, message dispatch) execute on these threads.

**Why this matters for your code:**
Your `register_message_handler` callback runs on an I/O thread. If your handler blocks (e.g., waiting for a hardware sensor), it starves the entire I/O loop. The pattern is:

```cpp
app_->register_message_handler(SERVICE_ID, INSTANCE_ID, METHOD_ID,
    [this](const std::shared_ptr<vsomeip::message>& req) {
        // DO: quick copy of data, then hand off to application thread
        auto data = req->get_payload()->get_data();
        work_queue_.push({req, data});   // non-blocking
        // DON'T: do heavy work or blocking calls here
    });
```

---

### Internal message flow

#### Send path (BCM notifying Cluster)

```text
1. BCM app calls app_->notify(SERVICE_ID, INSTANCE_ID, EVENT_ID, payload)
2. vSomeIP builds SOME/IP message: header (16 bytes) + serialized payload
3. Routing Manager looks up subscription table:
   → Cluster subscribed to EVENTGROUP_ID_BODY on this service
4. RM checks: is Cluster local (same ECU) or remote (different ECU)?
   → Remote: forward to UDP endpoint for Cluster's IP:port
   → Local:  forward via Unix domain socket to Cluster's app process
5. UDP endpoint calls boost::asio async_send → OS sends UDP datagram
```

#### Receive path (BCM receiving a request from Cluster)

```text
1. OS receives UDP datagram → boost::asio async_receive callback fires
2. Endpoint reads raw bytes into buffer
3. SOME/IP header parsed: Service ID=0x1234, Method ID=0x0421, MsgType=REQUEST
4. Endpoint passes parsed message to Routing Manager
5. RM looks up service registry:
   → Service 0x1234 instance 0x0001 is offered by BCM app itself (local)
6. RM looks up message handler registered for (0x1234, 0x0001, 0x0421)
7. boost::asio post() → handler callback fires on I/O thread
8. BCM handler reads payload, builds response, calls app_->send(response)
9. Send path executes (as above) back to Cluster
```

---

### Service Discovery (SD) internals

The SD module runs its own state machine per service. For a service provider (BCM):

```text
┌──────────────────────────────────────────────────────┐
│  OfferService state machine                          │
│                                                      │
│  [NOT_READY]                                        │
│      │ app_->offer_service() called                 │
│      ▼                                               │
│  [INITIAL_WAIT]  ← random delay (InitialDelayMin    │
│      │             to InitialDelayMax from config)  │
│      ▼                                               │
│  [REPETITION]    ← send OfferService N times        │
│      │             doubling interval each time      │
│      ▼                                               │
│  [MAIN]          ← send OfferService periodically   │
│                    (CyclicOfferDelay from config)   │
│                    with TTL field set                │
└──────────────────────────────────────────────────────┘
```

For a service consumer (Cluster):

```text
┌──────────────────────────────────────────────────────┐
│  FindService → Subscribe state machine               │
│                                                      │
│  [INITIAL_WAIT]                                     │
│      │                                               │
│      ▼                                               │
│  [REPETITION]  ← send FindService N times           │
│      │                                               │
│      │  OfferService received from BCM              │
│      ▼                                               │
│  [SERVICE_AVAILABLE]                                │
│      │  send SubscribeEventgroup                    │
│      ▼                                               │
│  [SUBSCRIBED]  ← SubscribeEventgroupAck received    │
│      │           availability_handler fires         │
│      ▼                                               │
│  receive NOTIFICATION events from BCM               │
└──────────────────────────────────────────────────────┘
```

TTL (Time To Live) in OfferService: if Cluster does not receive a refreshed OfferService within the TTL window, it marks the service as unavailable and the `availability_handler` fires with `is_available = false`.

---

### Configuration file (vsomeip.json)

vSomeIP is almost entirely driven by its JSON config. Nothing is hardcoded.

```json
{
    "unicast": "192.168.1.10",
    "logging": {
        "level": "debug",
        "console": "true"
    },
    "applications": [
        { "name": "bcm_service", "id": "0x1277" }
    ],
    "services": [
        {
            "service": "0x1234",
            "instance": "0x0001",
            "unreliable": "30509"
        }
    ],
    "routing": "bcm_service",
    "service-discovery": {
        "enable": "true",
        "multicast": "224.0.0.1",
        "port": "30490",
        "protocol": "udp",
        "initial_delay_min": "10",
        "initial_delay_max": "100",
        "repetitions_base_delay": "200",
        "repetitions_max": "3",
        "ttl": "3",
        "cyclic_offer_delay": "1000"
    }
}
```

| Config field | What it controls |
|---|---|
| `unicast` | This ECU's IP address |
| `applications[].id` | Client ID assigned to this application |
| `services[].unreliable` | UDP port for this service |
| `services[].reliable` | TCP port for this service (if used) |
| `routing` | Which application hosts the Routing Manager |
| `initial_delay_min/max` | Random wait before first OfferService/FindService |
| `repetitions_base_delay` | Base interval for SD repetition phase |
| `ttl` | How long (seconds) an OfferService is valid without refresh |
| `cyclic_offer_delay` | How often OfferService is re-sent in main phase |

---

### vSomeIP startup sequence (BCM)

```text
1. app_->init()
   ├── reads vsomeip.json
   ├── determines if this app is the routing manager host
   ├── connects to routing manager (Unix socket) if not the host
   ├── creates UDP/TCP endpoints based on config
   └── starts boost::asio I/O threads

2. app_->start()   ← blocks, runs boost::asio event loop

3. state_handler fires with ST_REGISTERED
   └── app is now connected to routing manager

4. app_->offer_service()
   └── routing manager registers this service
   └── SD module starts OfferService state machine

5. app_->offer_event()
   └── routing manager registers the event in the subscription table

6. Cluster subscribes → SubscribeEventgroup arrives
   └── routing manager adds Cluster to subscription table for EVENTGROUP_ID_BODY

7. app_->notify()
   └── routing manager fans out to all entries in subscription table
```

---

### Summary: what happens inside when you call notify()

```text
app_->notify(0x1234, 0x0001, 0x8001, payload)
         │
         ▼
  vsomeip::application_impl::notify()
         │
         ▼
  routing_manager_->notify()
         │  looks up subscription table for event 0x8001
         │  finds: [Cluster @ 192.168.1.20:30509]
         ▼
  udp_server_endpoint_impl::send()
         │
         ▼
  boost::asio async_send()
         │
         ▼
  OS UDP socket → Ethernet → Cluster ECU
```

---

## TCP vs UDP Selection in SOME/IP

In SOME/IP, the transport protocol is **statically configured** — it is determined at design time via the service definition, not negotiated at runtime.

---

### How the Decision is Made

#### 1. Service Definition / ARXML Configuration
Each SOME/IP method, event, or field is assigned a transport protocol in the service interface design (e.g., in AUTOSAR ARXML or Franca IDL):

```xml
<SOMEIP-METHOD-PROPS>
  <TRANSPORT-PROTOCOL>TCP</TRANSPORT-PROTOCOL>
</SOMEIP-METHOD-PROPS>
```

The SD (Service Discovery) messages advertise which protocol an endpoint supports.

---

#### 2. Rules of Thumb — When Each is Used

| Scenario | Protocol | Reason |
|---|---|---|
| **Request/Response methods** | TCP | Reliability needed; response must be guaranteed |
| **Fire & Forget methods** | UDP | No response expected; low overhead preferred |
| **Events / Notifications** | UDP (usually) | Low latency, multicast support |
| **Large payloads (> 1400 bytes)** | TCP | Avoids IP fragmentation; SOME/IP-TP only segments over UDP up to a point |
| **Multicast events** | UDP only | TCP does not support multicast |
| **Reliable event groups** | TCP | When event reliability is required |

---

#### 3. SOME/IP Service Discovery (SD) Role

During SD, the **OfferService** entry contains endpoint options specifying the protocol:

- `IPv4 Endpoint Option` — includes IP, port, and **protocol field** (`0x06` = TCP, `0x11` = UDP)
- A service can advertise **both** TCP and UDP endpoints simultaneously

The client selects the endpoint matching the protocol required by the specific method/event being called.

---

#### 4. SOME/IP-TP (Transport Protocol Segmentation)

For **large UDP payloads**, SOME/IP-TP allows segmentation over UDP instead of switching to TCP:
- Controlled by the `TP` flag in the SOME/IP header
- Configured per method in the toolchain
- Common for large diagnostic or calibration data

---

### Decision Flow Summary

```
Is multicast needed?
  └─ YES → UDP (mandatory)
  └─ NO →
      Is reliability critical (request/response)?
        └─ YES → TCP
        └─ NO →
            Is payload > MTU and TP not desired?
              └─ YES → TCP
              └─ NO → UDP
```

---

### Where to Configure This (AUTOSAR)

- **ARXML**: `SomeipMethodProps`, `SomeipEventProps` under the service interface
- **Vector DaVinci / EB tresos**: Transport protocol field per method/event
- **VSOMEIP** (open source): `json` config file — `reliable` = TCP, `unreliable` = UDP

```json
"services": [{
    "service": 4660,
    "instance": 22136,
    "reliable": { "port": 30509 },
    "unreliable": 30509
}]
```

> The protocol is chosen by the **service designer**, enforced through toolchain configuration, and communicated via SD endpoint options — not negotiated dynamically between client and server.

> **Key insight**: vSomeIP is essentially a combination of a **message router** (Routing Manager), a **network abstraction layer** (endpoints over boost::asio), and a **service discovery state machine** (SD module) — all wired together through the `application` API that your code calls.

---

## Day 5 — SOME/IP Service Discovery and Publish/Subscribe Flow

### Goal

Understand **SOME/IP-SD** in realistic ECU startup scenarios.

AUTOSAR's SOME/IP-SD spec describes service availability handling and publish/subscribe control for event messages. SOME/IP-SD uses SOME/IP over **UDP** (multicast).

---

### Why SOME/IP alone is not enough

SOME/IP defines *how* to format and send messages (method calls, events, notifications). But it has **no mechanism for**:

- Knowing whether a service is currently running
- Announcing that a service has started
- Controlling who receives event messages

Without SD, every client would need to be hardcoded with server addresses and assume services are always up. In a real vehicle, ECUs boot at different times, crash and restart, and go into low-power modes. SOME/IP-SD is the dynamic layer that handles all of this.

---

### The Transport: UDP Multicast

SOME/IP-SD uses **UDP multicast** (typically `224.244.224.245:30490`). This is intentional:

- **Multicast** → one SD message reaches all interested ECUs simultaneously
- **UDP** → low overhead; SD messages are periodic/retried anyway, so reliability is built into the SD state machine, not the transport
- No point-to-point connection setup needed just to announce a service

---

### Core SD Messages

#### OfferService

Sent by the **server ECU** (provider) to announce: *"I am running Service X, Instance Y, at this IP:port."*

```
Entry type: OfferService
Service ID:  0x0021  (e.g., BCM Body State service)
Instance ID: 0x0001
Major Ver:   1
Minor Ver:   3
TTL:         5       ← seconds this offer is valid
Endpoint:    192.168.1.10:30501 (UDP)
```

The server sends OfferService:
1. **On startup** (Initial Wait Phase → Repetition Phase → Main Phase)
2. **Periodically** in the Main Phase to refresh the TTL
3. **In response to a FindService** from a client

#### FindService

Sent by the **client ECU** when it needs a service and hasn't seen an OfferService yet (or its TTL expired).

```
Entry type: FindService
Service ID:  0x0021
Instance ID: 0xFFFF  ← "any instance"
TTL:         3
```

The server responds with an OfferService unicast back to the requester.

**Key point:** FindService is a fallback. In practice, if the server is already up, the client catches the periodic OfferService without needing to ask. FindService matters when the client boots *after* the server's last multicast or during the Repetition Phase gap.

#### TTL (Time To Live)

Every SD entry carries a TTL in seconds.

| TTL value | Meaning |
|---|---|
| `> 0` | Service/subscription is valid for this many seconds |
| `= 0` | **StopOffer** or **StopSubscribe** — explicit withdrawal |
| `= 0xFFFFFF` | Infinite — session lasts until explicit stop |

If the server crashes without sending StopOfferService, clients detect the outage when their TTL expires — they stop expecting events and re-enter the Find phase. This is why servers must send periodic OfferService to refresh TTLs.

---

### Event Groups

#### Why they exist

A service can have dozens of events. Subscribing to each one individually would create O(events × clients) SD traffic. Instead, events are grouped into **EventGroups** — a logical bundle.

```
Service: BCM Body State (0x0021)
├── EventGroup 0x01: DoorEvents
│   ├── Event 0x8001: FrontLeftDoor
│   ├── Event 0x8002: FrontRightDoor
│   └── Event 0x8003: RearDoors
└── EventGroup 0x02: LightEvents
    ├── Event 0x8010: HeadlightState
    └── Event 0x8011: IndicatorState
```

The Cluster subscribes to EventGroup `0x01` and receives all door events. It doesn't need to subscribe to lighting if it doesn't display it. This is both **bandwidth control** and **access control**.

#### SubscribeEventGroup

Sent by the **client** after receiving an OfferService:

```
Entry type:   SubscribeEventGroup
Service ID:   0x0021
Instance ID:  0x0001
EventGroup:   0x0001
TTL:          5
Endpoint:     192.168.1.20:30502  ← where client wants to receive events
```

The client tells the server: *"Send events from group 0x0001 to my IP:port."*

The client must **renew** this subscription before TTL expires, or the server stops sending.

#### SubscribeEventGroupAck

Sent by the **server** to confirm the subscription. If the server sends **SubscribeEventGroupNack** (TTL=0), the subscription was rejected — e.g., server at capacity, group doesn't exist, or version mismatch.

---

### BCM → Cluster Full Startup Sequence

```text
Cluster ECU                              BCM ECU
    |                                       |
    |  [Cluster boots, enters Find phase]   |
    |                                       |  [BCM boots, enters Offer phase]
    |                                       |
    |------- FindService(0x0021) ---------->|  (multicast, if BCM not seen yet)
    |                                       |
    |<------ OfferService(0x0021) ----------|  (BCM unicast reply, or periodic)
    |                                       |
    |--- SubscribeEventGroup(0x0001) ------>|  (Cluster wants DoorEvents)
    |                                       |
    |<-- SubscribeEventGroupAck(0x0001) ----|  (BCM confirms, starts buffering)
    |                                       |
    |<------ InitialEvent(DoorState) -------|  ← server sends current state
    |                                       |     immediately on subscribe
    |<------ Notification(DoorOpened) ------|  (incremental updates)
    |<------ Notification(IndicatorOn) -----|
    |                                       |
```

**Scenario: BCM boots first**
BCM sends periodic OfferService every N seconds. Cluster catches it when it boots — no FindService needed.

**Scenario: Cluster boots first**
Cluster sends FindService (multicast). BCM (when ready) replies with OfferService.

**Scenario: BCM crashes mid-session**
Cluster's subscription TTL eventually expires → Cluster re-enters Find phase → when BCM restarts it sends OfferService → Cluster re-subscribes.

---

### Important Architectural Insight: The Production Pattern

```
1. OfferService received
2. SubscribeEventGroup → SubscribeEventGroupAck
3. Call GetCurrentBodyState()   ← synchronous SOME/IP request/response
4. Render UI from method response
5. Apply incremental Notification events from here on
```

**The problem without step 3:**

- The SOME/IP-SD spec says the server *should* send an initial event on subscribe
- But "should" is not "shall" — some implementations don't, or the initial event is lost (UDP, no retransmit)
- If Cluster boots 10 seconds after BCM, it missed all state-change events that occurred during that window
- Without a sync call, the UI shows stale or blank state until the next change event arrives

**The sync call solves this:** you get a guaranteed snapshot of current state, then notifications keep you up to date. This is a standard pattern in SOME/IP system design.

---

### vSomeIP Key Concepts for Practice

| vSomeIP API | What it does |
|---|---|
| `application` | A named SOME/IP participant; owns a socket, has a service registry |
| `offer_service(svc, inst)` | Server side: starts sending OfferService SD entries |
| `request_service(svc, inst)` | Client side: triggers FindService, waits for Offer |
| `register_state_handler` | Callback when SD state changes (available/unavailable) |
| `register_message_handler` | Callback for incoming SOME/IP messages (methods or events) |
| `notify(svc, inst, event, payload)` | Server sends an event notification to all subscribers |
| `subscribe(svc, inst, eventgroup)` | Client sends SubscribeEventGroup |

The SD state machine inside vSomeIP handles the Initial Wait → Repetition → Main phase transitions automatically when you call `offer_service` or `request_service`.

---

### End-of-Day Checkpoint

**Why is SOME/IP alone not enough without SD?**
SOME/IP defines message format and transport. It has no way to discover service addresses dynamically, no TTL-based availability signaling, and no mechanism to control which ECUs receive which events. Without SD, you need static configuration — which breaks when ECUs restart asynchronously.

**Why does SD matter when ECUs boot at different times?**
Vehicle networks have no guaranteed boot order. BCM might be ready 3 seconds before Cluster, or vice versa. SD's Find/Offer cycle lets each ECU independently discover services whenever it's ready, and TTL-based expiry lets clients detect server outages without a dedicated heartbeat protocol. Without SD, a late-booting client would silently miss service availability and either crash or show stale data indefinitely.

---

## Day 6 — Implement a Minimal BCM Service and Cluster Client

### Goal

Build a small prototype in C++ using vSomeIP that demonstrates the full SOME/IP communication cycle: SD, method call, and event notification — using BCM→Cluster indicator status as the realistic use case.

---

### Service Design

**Use case:** BCM sends left indicator status to Cluster.

| ID | Value | Role |
|---|---|---|
| Service ID | `0x1234` | Identifies `VehicleBodyStatusService` |
| Instance ID | `0x0001` | First (and only) instance of this service |
| Method ID | `0x0421` | `GetCurrentIndicatorStatus` — request/response |
| Event ID | `0x8001` | `IndicatorStatusEvent` — notification |
| EventGroup ID | `0x0001` | Body events group |

**Why Event IDs start at `0x8000`:** The SOME/IP spec reserves `0x0001–0x7FFF` for methods and `0x8000–0xFFFE` for events. This lets parsers immediately classify an incoming message as a method call or event just from the ID range.

---

### Message Format at the Wire Level

#### TX side — BCM serializes

```
Logical data:
  side      = LEFT       (IndicatorSide::LEFT  = 0x00)
  state     = BLINKING   (IndicatorState::BLINKING = 0x02)
  timestamp = 12345678ms = 0x00BC614E

Serialized payload (10 bytes, big-endian):
  [0x00] [0x02] [0x00 0x00 0x00 0x00 0x00 0xBC 0x61 0x4E]
    side   state  ←────────── timestamp_ms (8 bytes) ──────────→

Full SOME/IP frame (16-byte header + 10-byte payload):
  Service ID   : 0x1234
  Event ID     : 0x8001
  Length       : 0x0000000E  (14 = 8 header tail + 10 payload bytes — excludes first 8 bytes)
  Client ID    : 0x0021
  Session ID   : 0x0102
  Proto Ver    : 0x01
  Iface Ver    : 0x01
  Message Type : 0x02  ← NOTIFICATION (no response expected)
  Return Code  : 0x00
  Payload      : [00 02 00 00 00 00 00 BC 61 4E]
```

Message Type values:
- `0x00` = REQUEST (method call, response expected)
- `0x02` = NOTIFICATION (event push, no response)
- `0x80` = RESPONSE (method reply)
- `0x01` = REQUEST_NO_RETURN (fire-and-forget method)

#### RX side — Cluster decodes

```
data[0]     → IndicatorSide::LEFT
data[1]     → IndicatorState::BLINKING
data[2..9]  → timestamp_ms = 12345678

UI action: flash LEFT turn tell-tale icon
```

---

### BCM Service — Code Walkthrough

#### Constructor and `init()`

```cpp
app_ = vsomeip::runtime::get()->create_application("bcm_service");
```

`runtime::get()` is a singleton. It reads the JSON config file from `VSOMEIP_CONFIGURATION` env var. The name `"bcm_service"` must match an entry in `applications[]` in the config — this is how the Routing Manager assigns a Client ID to this process.

#### `register_state_handler` — gate all setup behind ST_REGISTERED

```cpp
if (state == vsomeip::state_type_e::ST_REGISTERED) { offer(); }
```

`ST_REGISTERED` fires when this app has successfully connected to the Routing Manager. **You must not call `offer_service()` before this.** The RM isn't ready to register services until this point. Calling `offer_service` too early silently fails.

#### `offer()` — the two-step offer, order matters

```cpp
app_->offer_event(SERVICE_ID, INSTANCE_ID, EVENT_ID_INDICATOR, eventgroups, ...);
app_->offer_service(SERVICE_ID, INSTANCE_ID);
```

`offer_event` **must come before** `offer_service`. The moment `offer_service` is called, SD starts advertising the service and subscribers can arrive. If they arrive before `offer_event` has run, there is no event registered in the subscription table to bind them to.

Key `offer_event` parameters:
| Parameter | Value used | Meaning |
|---|---|---|
| `cycle` | `milliseconds::zero()` | No auto-periodic notify; we notify manually |
| `change_resets_cycle` | `false` | Not relevant when cycle is zero |
| `update_on_change` | `true` | Cache last value; send it immediately to new subscribers |
| `reliability` | `RT_UNRELIABLE` | Use UDP for this event |

Setting `update_on_change = true` makes vSomeIP store the last notified payload and send it immediately when a new subscriber joins — this partially replaces the need for a sync method call, though the sync call is still the safer production pattern.

#### `on_get_current()` — responding to a method call

```cpp
auto response = vsomeip::runtime::get()->create_response(_request);
```

`create_response` automatically copies `Client ID` and `Session ID` from the incoming request into the response header. This is the correlation mechanism — the Cluster matches the response to its pending call using these two fields. Never build a response header manually; always use `create_response`.

#### `serialize_indicator()` — big-endian byte packing

```cpp
for (int i = 7; i >= 0; --i) {
    data.push_back((ts >> (i * 8)) & 0xFF);
}
```

SOME/IP uses **big-endian (network byte order)** by default. The loop walks from byte 7 (most significant) down to byte 0 (least significant), pushing each byte into the vector. The receiver runs the inverse: accumulating bytes from MSB to LSB.

#### `publish_loop()` — notifier thread pattern

```cpp
notifier_thread_ = std::thread([this]() { publish_loop(); });
app_->start();  // blocks on boost::asio I/O loop
```

`app_->start()` blocks the calling thread permanently (it runs the boost::asio event loop). All application logic that must run after startup must live on a separate thread. `app_->notify()` is thread-safe — vSomeIP queues the message internally and dispatches it through the I/O loop.

The `publish_loop` toggles `blink` every second, simulating BCM hardware that detects the indicator stalk switching on/off.

---

### Cluster Client — Code Walkthrough

#### `register_state_handler` — three setup calls in sequence

```cpp
app_->request_service(SERVICE_ID, INSTANCE_ID);
app_->request_event(SERVICE_ID, INSTANCE_ID, EVENT_ID_INDICATOR, {EVENTGROUP_ID_BODY}, ...);
app_->subscribe(SERVICE_ID, INSTANCE_ID, EVENTGROUP_ID_BODY);
```

All three are called inside `ST_REGISTERED`. Their distinct roles:

| Call | What it does |
|---|---|
| `request_service` | Tells the RM "I want this service"; triggers SD FindService if not yet found |
| `request_event` | Tells the RM the event-to-eventgroup mapping before subscription arrives |
| `subscribe` | Records subscription intent; vSomeIP queues it and sends `SubscribeEventGroup` SD message only once the service becomes available |

`request_event` **must precede `subscribe`**. Without it, the RM doesn't know how to bind incoming event `0x8001` to eventgroup `0x0001`, and subscription setup can fail silently.

##### How `subscribe` queuing works in detail

When `subscribe` is called inside `ST_REGISTERED`, BCM's OfferService may not have arrived yet — vSomeIP doesn't know BCM's IP or port. So it **cannot send a UDP packet** immediately. Instead, the Routing Manager stores a pending subscription entry:

```
PendingSubscription {
    service_id:   0x1234
    instance_id:  0x0001
    eventgroup:   0x0001
    endpoint:     (not yet known)
    ttl:          from config
}
```

No network traffic is generated yet.

When the SD module later receives an **OfferService from BCM**, it:

```text
[OfferService received from BCM @ 192.168.1.10:30501]
        │
        ├─► service registry updated: 0x1234/0x0001 → 192.168.1.10:30501
        ├─► availability_handler(true) fired in your app
        └─► pending subscription list flushed:
                → send SubscribeEventGroup(0x0001) to 192.168.1.10:30501
```

This is why the recommended pattern is to express all intent upfront in `ST_REGISTERED` and react to availability in the `availability_handler` — vSomeIP handles the timing automatically.

##### TTL renewal — subscription is not one-shot

After `SubscribeEventGroup` is sent and ACKed, vSomeIP **automatically re-sends** `SubscribeEventGroup` before the TTL expires to keep the subscription alive. You never need to call `subscribe` again manually.

##### What happens when BCM crashes

If BCM crashes and the subscription TTL expires on the Cluster side:

```text
TTL expires
        │
        ├─► service marked unavailable in RM
        ├─► availability_handler(false) fired in your app
        └─► subscription moved back to pending list
                (waiting for the next OfferService from BCM)
```

When BCM restarts and sends OfferService again, the same flush mechanism fires and `SubscribeEventGroup` is re-sent automatically — no application code needed.

#### `register_availability_handler` — drives the production pattern

```cpp
if (is_available) { request_current_state(); }
```

Fires when the service transitions available/unavailable. The sync call `GetCurrentIndicatorStatus` here implements the production pattern from Day 5: get a guaranteed state snapshot before relying on incremental events. Without this, if BCM was already blinking when Cluster connected, the UI would show blank until the next notification arrived.

`is_available = false` fires when BCM's OfferService TTL expires or BCM sends StopOfferService. This is where you would grey out the tell-tale or show a communication fault on the cluster display.

#### Two separate message handlers

```cpp
// Handles pushed notifications from BCM
app_->register_message_handler(..., EVENT_ID_INDICATOR, on_indicator_event);

// Handles reply to our GetCurrentIndicatorStatus request
app_->register_message_handler(..., METHOD_ID_GET_CURRENT, on_method_response);
```

Both call the same `deserialize()` → `print_status()` path because the payload format is identical. The `kind` string (`"EVENT"` vs `"RESPONSE"`) only distinguishes the log output — the data model is the same.

#### `deserialize()` — inverse of BCM's serialize

```cpp
for (int i = 0; i < 8; ++i) {
    ts = (ts << 8) | data[2 + i];
}
```

Reassembles big-endian bytes MSB-first. `data[2]` is the most significant byte. Each iteration shifts the accumulator left 8 bits and ORs in the next byte. After 8 iterations, `ts` holds the full 64-bit timestamp.

---

### Full Interaction Timeline

```text
BCM                                         Cluster
 |                                              |
 | init() + start()                            | init() + start()
 | ST_REGISTERED fires                         | ST_REGISTERED fires
 | offer_event(0x8001)                         | request_service(0x1234)
 | offer_service(0x1234)                       | request_event(0x8001)
 |                                             | subscribe(0x0001)  ← queued
 |                                             |
 |──── SD: OfferService(0x1234) ─────────────>|
 |                                             | availability_handler(true) fires
 |<─── SD: SubscribeEventGroup(0x0001) ───────|
 |──── SD: SubscribeEventGroupAck ───────────>|
 |                                             | request_current_state() called
 |<─── SOME/IP REQUEST(0x0421) ───────────────|
 |──── SOME/IP RESPONSE(0x0421) ─────────────>| on_method_response → render UI
 |                                             |
 | [every 1s, notifier thread]                 |
 |──── SOME/IP NOTIFICATION(0x8001, BLINKING)>| on_indicator_event → flash icon
 |──── SOME/IP NOTIFICATION(0x8001, OFF) ────>| on_indicator_event → clear icon
 |──── SOME/IP NOTIFICATION(0x8001, BLINKING)>|
```

---

### Key Architectural Points Summary

| Point | Why it matters |
|---|---|
| `offer_event` before `offer_service` | Subscribers can arrive the instant SD advertises; event must be registered first |
| Always use `create_response` | Auto-copies Client ID + Session ID; ensures correct request/response correlation |
| Notifier runs in a separate thread | `app_->start()` blocks forever; application logic must live on other threads |
| `availability_handler` triggers the sync call | Correct and guaranteed moment to fetch current state snapshot |
| `request_event` before `subscribe` | RM needs the event→eventgroup mapping before processing the subscription |
| Big-endian serialization throughout | SOME/IP default; TX and RX must use identical byte order |
| `update_on_change = true` in `offer_event` | vSomeIP caches the last payload and sends it to new subscribers automatically |

---

### Minimal CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)
project(someip_bcm_cluster)

find_package(vsomeip3 REQUIRED)
find_package(Boost REQUIRED COMPONENTS system thread)

add_executable(bcm_service bcm_service.cpp)
target_link_libraries(bcm_service vsomeip3 Boost::system Boost::thread pthread)

add_executable(cluster_client cluster_client.cpp)
target_link_libraries(cluster_client vsomeip3 Boost::system Boost::thread pthread)
```

---

### End-of-Day Checkpoint

**What happens if you call `offer_service` before `offer_event`?**
A subscriber may arrive via SD before the event is registered in vSomeIP's internal subscription table. The subscription will be silently ignored or the event fan-out will fail. Always register events first.

**Why is `create_response` mandatory instead of `create_message`?**
`create_response` copies `Client ID` and `Session ID` from the request header into the response. Without these fields, the Cluster cannot match the response to its pending call — it will be discarded as an unexpected message.

**Why does the notifier run in a separate thread?**
`app_->start()` never returns — it runs the boost::asio I/O event loop indefinitely. Any periodic or timer-driven logic must execute on a thread that isn't blocked by the I/O loop. `app_->notify()` is thread-safe and queues the message for dispatch from the I/O thread.
