# OTOFU IMPLEMENTATION PLAN (PORT FROM ZIG TO ODIN)

## 0. Goal

Port tofu (Zig) into Odin as **otofu**, preserving:

* message (frame) ABI exactly
* polynode as structural primitive
* transport abstraction
* async/message-driven model

NOT porting blindly — but rebuilding **cleanly with same invariants**.

---

# 1. Core Principles (DO NOT BREAK)

These are invariants. If you violate them, system collapses conceptually.

### 1.1 Structure-first

* All data = polynode-based items
* No “special message types” outside frame

### 1.2 Frame is ABI

* Binary header + text headers + body
* Must remain compatible with Zig version

### 1.3 Transport is invisible

* No HTTP assumptions
* No socket logic in user-facing API

### 1.4 Runtime is orchestration only

* No business logic inside runtime
* Only routing / scheduling

---

# 2. Target Architecture (Odin)

```text
otofu/
├── frame/          ← FIRST milestone (must be perfect)
├── types/
├── runtime/        ← minimal initially
├── transport/      ← later
└── internal/       ← hidden machinery
```

---

# 3. Implementation Phases

---

## PHASE 1 — FRAME (CRITICAL FOUNDATION)

> STOP everything else until this is DONE and STABLE.

### 3.1 Port message.zig → frame/

Split into:

```text
frame/
  frame.odin            ← main struct
  header_binary.odin    ← binary layout
  header_text.odin      ← key-value headers
  body.odin             ← payload
  codec.odin            ← encode/decode
```

---

### 3.2 Frame struct (strict)

```odin
Frame :: struct {
    node: Polynode,        // FIRST FIELD (invariant)

    // binary header fields
    kind: u8
    flags: u8
    stream_id: u32
    length: u32

    // text headers
    headers: []Header

    // payload
    body: []u8
}
```

---

### 3.3 Requirements

* byte-exact compatibility with Zig
* no allocations inside decode (if possible)
* clear separation:

  * parse
  * validate
  * interpret

---

### 3.4 Deliverables

* encode(frame) → []u8
* decode([]u8) → Frame
* roundtrip tests

---

### 3.5 Tests (MANDATORY)

* binary compatibility tests vs Zig
* malformed input tests
* partial read tests

---

## PHASE 2 — TYPES

Minimal shared constants:

```text
types/
  opcodes.odin
  flags.odin
  errors.odin
```

NO logic here.

---

## PHASE 3 — MINIMAL RUNTIME (NO NETWORK)

> This is where most people overengineer. Don’t.

### 3.1 Goal

Run:

```text
frame → handler → frame
```

---

### 3.2 Runtime shape

```text
runtime/
  runtime.odin
  router.odin
```

---

### 3.3 Responsibilities

* receive frame
* route by type/opcode
* call handler
* return frame

---

### 3.4 No:

* threads
* pools
* mailbox (yet)
* async engine

---

### 3.5 API

```odin
handle :: proc(f: Frame) -> Frame
```

---

## PHASE 4 — LOOPBACK TRANSPORT

> Replace HTTP with this.

### 4.1 Purpose

* testing
* demos
* validation

---

### 4.2 Implementation

```text
transport/loopback/
  client.odin
  server.odin
```

---

### 4.3 Behavior

```text
client.send(frame)
  → runtime.handle(frame)
  → response
```

---

## PHASE 5 — MATRYOSHKA INTEGRATION

Now introduce real power.

---

### 5.1 Use ONLY:

* polynode
* mailbox (optional at this stage)

---

### 5.2 Replace runtime routing with:

* polynode-based flow
* message passing

---

### 5.3 Introduce:

```text
runtime/
  dispatcher.odin
  flow.odin
```

---

### 5.4 Goal

Support:

* chaining
* async execution (via mailbox)
* multi-step flows

---

## PHASE 6 — ASYNC MODEL

Now bring real tofu behavior.

---

### 6.1 Introduce mailbox

* request/response pattern
* delayed processing

---

### 6.2 Patterns to support

* request → response
* fire-and-forget
* fan-out
* pipeline

---

### 6.3 Keep:

NO futures
NO async/await

Only:

* messages
* queues
* handlers

---

## PHASE 7 — REAL TRANSPORT (OPTIONAL)

Only after everything works.

---

### 7.1 Implement:

```text
transport/tcp/
```

---

### 7.2 Responsibilities

* read bytes
* assemble frame
* pass to runtime

---

### 7.3 Important

Transport MUST NOT:

* know business logic
* inspect headers deeply

---

# 4. Directory Evolution

## Early stage

```text
otofu/
  frame/
  types/
  runtime/
```

## Later

```text
otofu/
  frame/
  types/
  runtime/
  transport/
  internal/
```

---

# 5. What NOT to do

### ❌ Do not:

* implement full engine early
* introduce threads too soon
* mix transport with runtime
* wrap frame in “nice API”

---

# 6. First Working Milestone (VERY IMPORTANT)

You are DONE when this works:

```text
client → encode(frame)
       → send bytes
       → decode(frame)
       → runtime.handle()
       → encode(response)
       → decode(response)
       → client
```

No shortcuts.

---

# 7. Recommended First App

Implement ONLY this:

### echo / request-response

* client sends frame
* handler echoes or modifies
* returns frame

This proves:

* frame works
* runtime works
* transport abstraction works

---

# 8. Key Risks

### 8.1 Frame drift

* if Odin ≠ Zig → system breaks

### 8.2 Overengineering runtime

* biggest danger

### 8.3 Mixing layers

* frame ↔ runtime ↔ transport

---

# 9. Development Order (STRICT)

1. frame
2. frame tests
3. minimal runtime
4. loopback transport
5. simple app
6. matryoshka integration
7. async patterns
8. real transport

---

# 10. Bottom Line

* frame is the product
* runtime is glue
* transport is replaceable
* matryoshka gives power

---

# ONE-LINE SUMMARY

Build frame first, prove it with a minimal runtime and loopback transport, then layer matryoshka and async behavior — never the other way around.

---
