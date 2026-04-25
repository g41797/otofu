# 📦 otofu — Architecture Specification (Final)

## 1. System Identity

otofu is:

> A single-threaded, reactor-based, frame-oriented async messaging system built on matryoshka primitives, providing inter-process and inter-computer communication via channel abstractions.

It is NOT:

* a framework
* an RPC system
* a stream system
* a transport abstraction layer

---

# 2. Layer Model

```text id="otofu_layers"
[ Application ]
      ↓
[ Channel Group (API façade) ]
      ↓
[ Reactor (Engine implementation) ]
      ↓
[ Frame + Pool (matryoshka) ]
      ↓
[ OS sockets (epoll/kqueue/wepoll) ]
```

---

# 3. Core Dependencies

## matryoshka

Used for:

* polynode (structural base of items)
* mailbox (internal async communication)
* pool (allocation + backpressure policy)

No transport logic exists here.

---

# 4. Frame Model

A frame consists of:

### Binary header (authoritative control plane)

* kind / flags (control + routing semantics)
* endian-aware (BE ↔ LE conversion by reactor)
* connection/channel identity info
* control signals encoded here:

  * HelloRequest
  * WelcomeRequest
  * ByeSignal
  * Control messages

---

### Text headers (optional)

* only used for:

  * addressing (IP, host, port, UDS path, etc.)
  * user-defined metadata

---

### Body (optional)

* opaque payload
* never interpreted by otofu

---

## Key rule:

> otofu ONLY understands binary header. Everything else is opaque.

---

# 5. Reactor (Engine)

## Definition

The reactor is the **execution engine of otofu**.

It is:

* single-threaded
* event-loop based
* built on epoll / kqueue / wepoll

---

## Responsibilities

### Input path

1. read socket
2. allocate frame from matryoshka pool
3. read binary header
4. convert endian
5. read optional text headers/body (opaque)
6. attach buffers to frame
7. send via mailbox to channel group

---

### Output path

1. receive frame from channel group mailbox
2. enqueue into per-channel queue
3. schedule socket write
4. on success → return frame to pool
5. on failure → return frame to waiter with disconnect status

---

# 6. Channel Model

## Concept

A channel is:

> a logical FIFO message queue + routing abstraction for a peer connection.

NOT a transport endpoint.

---

## Properties

* created via `HelloRequest`
* represents logical peer connection
* can exist before physical connection exists
* supports buffering before connect

---

## Channel Group

Channel group is:

* user-facing API façade
* wrapper over internal mailbox communication with reactor
* does NOT contain logic

Engine may have multiple channel groups.

Multiple engines may exist per process.

---

# 7. Channel Lifecycle

## Creation

* initiated by user via `HelloRequest`
* engine creates channel implicitly

## Listening

* `WelcomeRequest` creates server-side channel group
* includes address info (IP, host, port, UDS path, etc.)

---

## Destruction

* triggered by:

  * `ByeSignal` (OOB)
  * connection failure
  * engine decision

---

# 8. Message Flow Semantics

## Send path

```text id="send_flow"
cg.send(frame)
  → mailbox
  → reactor
  → per-channel queue
  → socket write
```

---

## Buffering model

* per-channel **unbounded logical queue**
* physically constrained by:

  * matryoshka pool policy

---

## Non-connected peer behavior

If channel is not connected:

* frame is queued in order
* flushed upon connection
* if connection fails:

  * frame returned to waiter with disconnect status
  * frame returned to pool by waiter

---

## Completion model

* ❌ no positive ACK
* ✔ only negative failure reporting

---

# 9. Ordering Guarantees

* strict FIFO per channel
* preserved across:

  * disconnect
  * reconnect
* exception:

## ByeSignal (OOB channel)

* bypasses normal queue
* may interrupt ordering at any time

---

# 10. Backpressure Model

Backpressure is **pool-driven**

## Mechanism

* frame allocation comes from matryoshka pool

## Failure modes

* allocation failure → send blocked
* receive failure → inbound blocked
* system pressure propagates via pool exhaustion

---

# 11. Control Plane

All control messages are encoded in:

> binary header (kind/flags field)

### Control types:

* HelloRequest → connect
* WelcomeRequest → listen
* ByeSignal → disconnect (OOB allowed anytime)
* Control → system signals

---

### Addressing (only for Hello/Welcome)

Text headers include:

* IP
* hostnames
* ports
* UDS paths
* user metadata

---

# 12. Engine Isolation Model

* engines are independent
* no shared state required
* can coexist in same process:

  * client engine
  * server engine
* can simulate distributed systems locally

---

# 13. Matryoshka Integration

Inside reactor:

* mailbox = internal communication bus
* pool = frame lifecycle + backpressure control

Matryoshka is:

> execution substrate, not transport layer

---

# 14. Key System Invariants

### I1 — Frame integrity

Frame is allocated once and returned to pool exactly once.

---

### I2 — FIFO guarantee

Per-channel strict ordering always preserved.

---

### I3 — Negative completion model

Only failures are reported, no success ACK.

---

### I4 — Pool-driven pressure

System stability is governed by matryoshka pool availability.

---

### I5 — Transport opacity

Text headers and body are never interpreted by otofu.

---

# 15. One-line definition

> otofu is a reactor-based, pool-driven, frame-oriented messaging system where channels are logical FIFO buffers mapped onto asynchronous mailbox routing over matryoshka primitives.

---

# If you want next step

I can now:

### 1. Turn this into a **repo layout (final otofu structure)**

### 2. Or design the **actual Odin module APIs (engine/channel/frame structs)**

### 3. Or map it directly onto your starter repo integration plan
