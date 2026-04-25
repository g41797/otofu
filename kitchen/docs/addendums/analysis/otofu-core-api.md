# 📦 otofu — Core Odin APIs (Engine / Channel / Frame)

---

# 1. 📦 FRAME API (public)

## Key invariant

> Frame = polynode + binary header + opaque payload

```odin
package frame

import m "vendor/matryoshka/polynode"
```

---

## 🔷 Binary Header (16 bytes conceptually)

```odin
FrameKind :: enum u8 {
    HELLO_REQUEST,
    WELCOME_REQUEST,
    DATA,
    BYE,
    CONTROL,
}
```

---

```odin
FrameFlags :: bit_set[u8]
```

---

## 🔷 Header

```odin
FrameHeader :: struct #packed {
    kind: FrameKind,
    flags: FrameFlags,

    channel_id: u64,
    seq: u64,
}
```

---

## 🔷 Frame (core object)

```odin
Frame :: struct {
    node: m.PolyNode, // MUST be first conceptually (your invariant)

    header: FrameHeader,

    text_headers: []u8, // opaque
    body: []u8,         // opaque

    status: FrameStatus,
}
```

---

## 🔷 Status (negative-only model)

```odin
FrameStatus :: enum {
    OK,              // internal only (no user ACK semantics)
    DISCONNECTED,
    POOL_EXHAUSTED,
    DROPPED,
}
```

---

## Notes

* NO parsing of text/body
* NO ownership transfer logic here
* Frame is **pure data container**

---

# 2. 📦 CHANNEL API (public abstraction)

```odin
package engine
```

---

## 🔷 Channel

```odin
Channel :: struct {
    id: u64,

    // opaque pointer to engine state
    _engine: ^Engine,
}
```

---

## 🔷 Channel Group

> This is the user-visible façade

```odin
ChannelGroup :: struct {
    _engine: ^Engine,
}
```

---

## 🔷 API

### Create

```odin
create_channel_group :: proc(e: ^Engine) -> ChannelGroup
```

---

### Send

```odin
channel_send :: proc(c: Channel, f: ^Frame) -> FrameStatus
```

Semantics:

* enqueue into per-channel queue
* returns ONLY failure states (if any)

---

### Receive

```odin
channel_receive :: proc(c: Channel) -> (f: ^Frame, ok: bool)
```

Blocking or polling depends on engine mode (configurable)

---

### Connect (HelloRequest)

```odin
channel_connect :: proc(
    c: Channel,
    text_headers: []u8,
) -> FrameStatus
```

* triggers HELLO_REQUEST in binary header
* engine creates/activates channel

---

### Listen (WelcomeRequest)

```odin
channel_listen :: proc(
    g: ChannelGroup,
    text_headers: []u8,
) -> FrameStatus
```

* server-side entry point
* creates listening channel group context

---

### Close

```odin
channel_close :: proc(c: Channel) -> FrameStatus
```

* sends BYE (OOB behavior inside engine)

---

# 3. 📦 ENGINE API (reactor = engine implementation)

```odin
package engine
```

---

## 🔷 Engine

```odin
Engine :: struct {
    _state: rawptr, // internal reactor state

    config: EngineConfig,
}
```

---

## 🔷 Config

```odin
EngineConfig :: struct {
    max_channels: int,
    max_queue_per_channel: int,

    poll_timeout_ms: int,

    // matryoshka integration
    pool: ^any,
    mailbox: ^any,
}
```

---

## 🔷 Lifecycle

### Create

```odin
engine_create :: proc(cfg: EngineConfig) -> Engine
```

---

### Run

```odin
engine_run :: proc(e: ^Engine)
```

* starts reactor loop
* single-threaded event loop
* blocks

---

### Stop

```odin
engine_stop :: proc(e: ^Engine)
```

---

## 🔷 Internal responsibilities (not exposed)

Engine handles:

* epoll/kqueue/wepoll loop
* socket IO
* frame assembly
* per-channel queues
* matryoshka mailbox bridge
* pool-based backpressure

---

# 4. 🔷 Internal contracts (IMPORTANT)

These are NOT API, but define correctness:

---

## Frame lifecycle rule

```text
ALLOC → FILL → DISPATCH → CONSUME → RETURN_TO_POOL
```

* always exactly one return to pool
* never user-managed free

---

## Channel rule

> Channel is only an ID + queue context, never transport

---

## Delivery rule

* FIFO guaranteed per channel
* ordering preserved across reconnect

---

## Failure model

Only:

```odin
FrameStatus
```

is meaningful:

* NO ACK system
* only failure reporting

---

# 5. 🧠 Mental model (very important)

## User sees:

```text
ChannelGroup → Channel → Frame
```

## System actually is:

```text
ChannelGroup → Engine(Reactor) → Matryoshka → OS IO
```

---

# 6. 🔥 Critical design invariants

### I1 — Frame is immutable in semantics after enqueue

(no mutation after send path)

---

### I2 — Engine owns execution, not channels

channels are passive routing keys

---

### I3 — Pool controls system pressure

not queues, not sockets

---

### I4 — No success ACK model exists

only failure propagation

---

# 7. One-line API definition

> otofu exposes a minimal Channel/Frame API backed by a reactor engine that routes opaque frames through FIFO per-channel queues using matryoshka-backed mailbox and pool-controlled memory lifecycle.

---
