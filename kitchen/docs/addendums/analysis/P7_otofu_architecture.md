# P7 otofu Architecture

Sources: P3_ownership.md, P4_matryoshka_mapping.md, P5_zig_vs_odin.md, P6_reactor_model.md
Prior phases: P1–P6

This document is the definitive architecture specification for otofu.
Every statement is a constraint, not a suggestion.

---

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────┐
│  L5 — Public API          Engine · ChannelGroup · Msg   │
├─────────────────────────────────────────────────────────┤
│  L4 — Protocol Layer      OpCode · Handshake · Framer   │
├─────────────────────────────────────────────────────────┤
│  L3 — Messaging Runtime   Pool · Reserved · Router      │
├─────────────────────────────────────────────────────────┤
│  L2 — Reactor Core        Loop · DualMap · Dispatch     │
├─────────────────────────────────────────────────────────┤
│  L1 — OS / Poller         Poller · Notifier · Socket    │
├─────────────────────────────────────────────────────────┤
│  L0 — Matryoshka          PolyNode · Mailbox · Pool     │
└─────────────────────────────────────────────────────────┘
```

### Foundational Constraints

| Constraint | Source | Consequence if violated |
|-----------|--------|------------------------|
| Single I/O thread | Architecture | Locks required in I/O path; race on Socket state |
| No shared mutable state across threads | Architecture | Data races on any shared structure |
| All cross-thread transfer via Mailbox | P3 L-MB1 | Ownership lost; no signal; silent corruption |
| No lock on hot I/O path | Architecture | Throughput collapse under load |
| Explicit allocator at every allocation site | P5 MR-1 | Hidden allocation from wrong heap |
| No blocking in Reactor except Poller.wait | P6 LI-1 | I/O thread stalls; all socket I/O halted |
| Deregister before close(fd) | P6 LI-5 | ABA on FD reuse; stale events on new socket |
| Reserved pool separate from application pool | P4 V-MP2 | R6.3 deadlock: Reactor cannot deliver notifications |
| Exhaustive switch on all closed dispatch sets | P5 PM-3 | Silent miss of new OpCode or error code |
| context.allocator never used inside otofu | P5 DR-3 | Hidden allocation from wrong thread's heap |

---

## 1. Layers

Six layers. Dependencies are strictly downward. No upward calls. No lateral calls across layer siblings.

### Layer Definitions

#### L0 — Matryoshka

External library. Not otofu code. Not modified. Provides:

- `PolyNode` (intrusive link at offset 0) + `MayItem` (`Maybe(^PolyNode)` — ownership token)
- `Mailbox` (MPMC blocking queue; `mbox_send` transfers ownership; `mbox_wait_receive` receives)
- `Pool` (reusable item storage; `on_get` / `on_put` hooks; `pool_get` / `pool_put`)

otofu uses Doll 1 (PolyNode/MayItem), Doll 2 (Mailbox), and Doll 3 (Pool). Doll 4 (Infrastructure as Items) is not required for baseline otofu.

**L0 rule:** otofu does not modify, wrap, or re-export Matryoshka types. It uses them directly.

#### L1 — OS / Poller

Platform-specific I/O abstraction. Provides exactly three interfaces: Poller, Notifier, Socket.
Selected by `when ODIN_OS` conditional compilation. No runtime dispatch. One concrete type per platform.

| Platform | Poller backend | Notifier backend |
|----------|---------------|-----------------|
| Linux | epoll | Unix socket pair |
| macOS / BSD | kqueue | Unix socket pair |
| Windows | wepoll (AFD_POLL) | AF_UNIX or loopback TCP |

**L1 rule:** L1 does not know about Channels, Messages, SequenceNumbers, or otofu structures. It operates on file descriptors, byte buffers, and addresses only.

**L1 rule:** L1 maps OS error codes to the `Engine_Error` enum (defined in L5, used throughout). No raw OS error code escapes L1.

#### L2 — Reactor Core

The event loop and all structures it owns exclusively. Single-thread. All L2 code executes on the Reactor thread only.

L2 calls L1 (Poller, Notifier, Socket) and L0 (TriggeredChannel Pool).
L2 calls L3 (reserved pool, mailbox router) for Message delivery.
L2 calls L4 (protocol, handshake) for event interpretation.

**L2 rule:** No L2 function may be called from an application thread.
**L2 rule:** No L2 function may block on a Mailbox or Pool.

#### L3 — Messaging Runtime

Message lifecycle, pool management, and cross-thread transfer wiring. Both threads interact with L3, but through different entry points.

- Reactor thread: uses reserved pool, mailbox router (send to outboxes), framer (encode/decode)
- Application thread: uses message pool (get/put), mailbox router (send to reactor inbox)

L3 calls L0 (Pool, Mailbox).
L3 does not call L1 (no OS access) or L2 (no Reactor access).
L3 does not know about protocol OpCodes (that is L4).

**L3 rule:** L3 does not interpret Message content. It manages Message lifecycle and movement.

#### L4 — Protocol Layer

Conversation-layer protocol. Interprets OpCodes. Drives Channel state machine from `opened` to `ready` to `closed` via the Hello/Bye/Welcome handshake sequences.

L4 calls L3 (to obtain Messages from Reserved Pool for responses, to frame outbound messages).
L4 calls L2 (via state transition directives — L4 tells L2 what state transition to execute).
L4 does not call L1.

**L4 rule:** L4 does not perform I/O. It reads decoded messages and produces state directives and response messages.
**L4 rule:** All OpCode dispatch uses exhaustive switch with a panic default (PM-3, DR-4).

#### L5 — Public API

Application-facing surface. Thread-safe. Minimal. L5 is the only layer the application imports.

L5 calls L3 (Engine.get, Engine.put, ChannelGroup.post, ChannelGroup.waitReceive).
L5 does not call L1 or L2 directly.

**L5 rule:** No L5 function allocates or deallocates directly. All allocation goes through Engine's explicit allocator via L3.
**L5 rule:** All L5 functions return explicit error results. No silent ignore. No panic on usage errors.

### Layer Boundary Rules

| Caller | May call | Must NOT call |
|--------|---------|---------------|
| L5 | L3 | L1, L2 |
| L4 | L3, L2 (directives only) | L1 |
| L3 | L0 | L1, L2, L4 |
| L2 | L1, L0, L3 (delivery), L4 (interpretation) | — |
| L1 | OS | L0, L2, L3, L4, L5 |
| L0 | — (external library) | — |

**Cross-layer rule:** Application threads may only call L5 and the L3 receive path (`mbox_wait_receive` via L5 wrapper). Application threads must never call L1 or L2.

---

## 2. Core Modules

One row per module: purpose, inputs, outputs, ownership.

---

### L0 — Matryoshka (external)

| Module | Purpose | Inputs | Outputs | Ownership |
|--------|---------|--------|---------|-----------|
| `matryoshka.poly` | Intrusive node + ownership token | PolyNode-embedded structs | MayItem (ownership signal) | Caller owns; token tracks it |
| `matryoshka.mailbox` | MPMC blocking queue | MayItem (send), timeout (recv) | MayItem (recv), SendResult, RecvResult | Mailbox owns while queued; transfers on recv |
| `matryoshka.pool` | Item recycling with hooks | MayItem (put), strategy (get) | MayItem (get) | Pool owns while pooled; caller owns after get |

---

### L1 — OS / Poller

#### Module: `poller`

**Purpose:** Wrap the OS I/O multiplexer. Register FDs with interest flags. Deregister FDs. Wait for readiness events with a timeout.

**Inputs:**
- `register(fd, seqn: u64, flags: TriggerFlags)` — arm FD in kernel interest set; embed seqn in event data
- `deregister(seqn: u64)` — remove FD from interest set; invalidate seqn
- `wait(timeout_ms: i64)` → `[]Event{seqn: u64, flags: TriggerFlags}` — block until event, timeout, or interrupt
- `close()` — release OS handle

**Outputs:** Event batch (seqn + TriggerFlags per fired FD). Batch is empty on timeout.

**Ownership:** Poller struct owned by Reactor. FDs are referenced, not owned — Socket module owns FDs.

**Invariant:** seqn is embedded in `epoll_data.u64` / `kevent.udata` / AFD_POLL context at registration time. OS delivers it back in the event. The Poller does not maintain a seqn→FD map; that is the dual-map (L2).

---

#### Module: `notifier`

**Purpose:** Socket-pair cross-thread wake mechanism. Application side writes; Reactor side reads and drains. Not `mbox_interrupt` — incompatible with Poller-based blocking (P4, DR-5).

**Inputs:**
- `create(allocator)` → Notifier handle (read_fd + write_fd)
- `notify(notifier)` — write 1 byte to write_fd; may silently fail if pipe buffer full (idempotent: existing byte wakes Reactor)
- `drain(notifier)` — read all bytes from read_fd until EAGAIN; called by Reactor after Poller fires on read_fd
- `read_fd(notifier)` → fd — used by Reactor to register with Poller
- `close(notifier, allocator)` — close both FDs; free struct

**Outputs:** None. Side effect: Reactor wakes from Poller.wait.

**Ownership:** Notifier struct owned by Reactor. Both FDs owned by Notifier (created and closed by Notifier).

---

#### Module: `socket`

**Purpose:** Non-blocking socket lifecycle. Create, connect, listen, accept, send, recv, close. Map OS errors to `Engine_Error`.

**Inputs:**
- `create(allocator)` → Socket handle (fd, state)
- `set_nonblocking(socket)`
- `connect(socket, addr: Address)` → ok / Engine_Error.WouldBlock / Engine_Error.ConnectionRefused / ...
- `connect_complete(socket)` → ok / error (via `getsockopt(SO_ERROR)`) — called after WRITE event on connecting socket
- `listen(socket, backlog: int)` → ok / Engine_Error
- `accept(socket)` → new Socket / Engine_Error.WouldBlock / Engine_Error
- `send(socket, buf: []u8)` → n: int / Engine_Error.WouldBlock / Engine_Error
- `recv(socket, buf: []u8)` → n: int / Engine_Error.WouldBlock / Engine_Error.ConnectionReset / ...
- `set_linger(socket, on: bool, timeout_s: int)` — SO_LINGER; always called before close on IO sockets
- `close(socket, allocator)` — close fd; free struct

**Outputs:** New Socket (from create/accept), byte counts (send/recv), `Engine_Error`.

**Ownership:** Socket struct owned by Reactor. FD owned by Socket struct. Application has no handle to any Socket or FD.

**Error mapping (partial):**

| OS error | Engine_Error |
|----------|-------------|
| EAGAIN / EWOULDBLOCK | .WouldBlock |
| ECONNREFUSED | .ConnectionRefused |
| ETIMEDOUT | .TimedOut |
| ECONNRESET | .ConnectionReset |
| EADDRINUSE | .AddressInUse |
| EBADF / EINVAL | .InternalError (programming error; panic in debug) |

---

### L2 — Reactor Core

#### Module: `reactor`

**Purpose:** Execute the 9-phase event loop (P6). Coordinate all L2 submodules. Serve as the single I/O thread. Own and supervise the Engine lifecycle from `running` to `destroyed`.

**Inputs:**
- `start(engine_ref, allocator, options)` — spawned as a new OS thread; sets `context.allocator = engine.explicit_allocator` before any other operation (TH-1, MR-2)
- Reactor inbox Mailbox (via mailbox_router) — commands from application threads
- Poller event batches (from `poller.wait`)

**Outputs:**
- Messages delivered to per-ChannelGroup outbox Mailboxes (via mailbox_router)
- Engine state transitions (via engine_ref)

**Ownership:** Reactor owns all L2 submodules (dual_map, tc_pool, channel_manager, io_dispatch, timeout_manager), all Sockets, the Notifier (registered with Poller), and the Poller itself.

**Phase sequence:** See P6 for full detail. Summary:
1. Compute timeout
2. Poller.wait
3. Classify events
4. Check timeouts → pending_close
5. Drain inbox → process commands
6. Resolve I/O events via SequenceNumber
7. Dispatch I/O per TriggeredChannel
8. Process pending closes (deregister → close(fd) → pool_put → notify → release ChannelNumber)
9. Drain check (if draining and clean: exit loop)

---

#### Module: `dual_map`

**Purpose:** Two-level indirection between Channel identity and OS events. Enables O(1) event-to-TriggeredChannel resolution. Provides ABA protection via SequenceNumber validation.

**Data:**
- `chan_to_seqn: HashMap(ChannelNumber, SequenceNumber)` — used to find seqn when closing a channel by number
- `seqn_to_tc: HashMap(SequenceNumber, *TriggeredChannel)` — used in Phase 6 to resolve raw OS events

**Inputs:**
- `insert(channel_num, seqn, tc_ptr)` — called in Phase 5 (new channel open) and Phase 7 (accepted channel)
- `remove_seqn(seqn)` — called in Phase 8 (channel close, Step 3)
- `remove_chan(channel_num)` — called in Phase 8 (channel close, Step 3)
- `lookup_seqn(seqn)` → `*TriggeredChannel` or nil — called in Phase 6 (event resolution)
- `lookup_chan(channel_num)` → SequenceNumber or 0 — called in Phase 8 (find seqn from channel)

**Outputs:** `*TriggeredChannel` pointer (stable heap address) or nil on ABA-discarded stale event.

**Ownership:** Owned exclusively by Reactor. No other thread reads or writes. Not guarded by a lock.

**Invariant:** All inserts happen in Phase 5. All removes happen in Phase 8. Phase 6 and Phase 7 only read. This ordering is mandatory (P6 LI-2).

---

#### Module: `tc_pool` (TriggeredChannel Pool)

**Purpose:** Allocate and recycle TriggeredChannel structs using a Matryoshka Pool. Maintain pointer stability — heap-allocated items never move.

**Inputs:**
- `get(allocator)` → MayItem — `pool_get(.Available_Or_New)` with on_get hook
- `put(&m, allocator)` — `pool_put` with on_put hook; only called after TriggeredChannel is `deregistered`
- `close_and_dispose(allocator)` — during Engine drain; `pool_close` → free all stored items → `matryoshka_dispose`

**Outputs:** MayItem (`m^ != nil` = valid heap-allocated TriggeredChannel, zeroed fields).

**Ownership:** Pool owns TriggeredChannels while pooled. Reactor owns while in active use. No application access.

**Hook: on_get:**
```
if m^ == nil:
    allocate TriggeredChannel via explicit allocator
    set poly.id = TriggeredChannelId
    set m^ = &poly
else:
    zero seq, channel_num, trigger_flags, platform_io_state
```

**Hook: on_put:**
```
leave m^ non-nil — pool always stores
// No capacity limit — number of TCs is bounded by max_channels configuration
```

**Invariant (V-TC1):** `pool_put` must not be called while the TriggeredChannel is registered in the Poller or referenced by the dual_map. Violation causes stale pointer dispatch. Not enforced by Matryoshka — enforced by Phase 8 ordering.

---

#### Module: `channel_manager`

**Purpose:** Maintain the per-ChannelGroup list of Channels. Execute all Channel state machine transitions (SM2). Assign and release ChannelNumbers. Track per-Channel outbound send queues and timeout deadlines.

**Data per Channel:**
- `state: ChannelState` — current SM2 state
- `number: ChannelNumber` — assigned u16 (1–65534); 0 = unassigned
- `type: ChannelType` — Listener | IO_Client | IO_Server
- `socket: *Socket` — non-owning reference (Reactor owns Socket)
- `triggered_channel: *TriggeredChannel` — non-owning reference (tc_pool owns)
- `outbound_queue: Queue(*Message)` — messages waiting to be sent; bounded by options.outbound_queue_depth
- `recv_buf: []u8` — partial receive state (framing cursor)
- `connect_deadline: i64` — monotonic ns; 0 = no deadline
- `handshake_deadline: i64`
- `bye_deadline: i64`
- `close_reason: CloseReason`
- `channel_group_ref: *ChannelGroup` — which ChannelGroup owns this Channel

**Inputs:**
- `allocate(cg_ref, type, allocator)` → *Channel — allocates struct; state = `unassigned`
- `assign_number(ch)` — state → `opened`; assigns ChannelNumber from pool
- `release_number(ch)` — releases ChannelNumber; called in Phase 8 Step 7
- `transition(ch, new_state)` — drives SM2; validates legality of transition; panics on illegal transition in debug
- `enqueue_outbound(ch, &m)` → ok / Engine_Error.BackpressureExceeded — appends Message to outbound_queue if within depth limit
- `dequeue_outbound(ch)` → *Message or nil — pops front of outbound_queue for send dispatch
- `free(ch, allocator)` — state must be `closed`; free struct

**Outputs:** Channel state, outbound messages for send dispatch, close candidates for pending_close.

**Ownership:** ChannelGroup owns all its Channels. channel_manager holds Channel list. Reactor thread exclusively accesses all Channel state.

---

#### Module: `io_dispatch`

**Purpose:** Execute the I/O handler for one resolved TriggeredChannel per Phase 7 iteration. Delegate to socket (recv/send/accept), to L4 protocol for message interpretation, and to L3 mailbox_router for delivery.

**Inputs:**
- `dispatch(tc: *TriggeredChannel, ch: *Channel, flags: TriggerFlags, ctx: *DispatchContext)`
  - DispatchContext carries: allocator, reserved_pool, app_pool, mailbox_router, channel_manager, dual_map, pending_close list

**Outputs:**
- Messages delivered to app outbox Mailboxes (via mailbox_router)
- Channels appended to pending_close (via ctx)
- Channel state transitions (via channel_manager.transition)
- New Channels allocated (for accepted connections; via channel_manager + dual_map insert)

**Ownership:**
- io_dispatch receives `owned-engine` Messages (from reserved pool or decoded from wire) and delivers them to Mailboxes via mailbox_router. After delivery, Mailbox owns.
- io_dispatch does NOT own Sockets, Channels, or TriggeredChannels — it acts on them under Reactor supervision.

**READ path (abbreviated):**
```
recv into recv_buf; call framer.try_decode(recv_buf)
on complete frame:
  m = message from reserved_pool or app_pool (depending on direction)
  framer.fill(m, frame)
  protocol.handle_inbound(m, ch, ctx)  → state directive + optional response
```

**WRITE path (abbreviated):**
```
while ch.outbound_queue not empty:
  m = channel_manager.dequeue_outbound(ch)
  encoded = framer.encode(m)
  n = socket.send(ch.socket, encoded.remaining)
  on WouldBlock: requeue, rearm WRITE, break
  on error: pending_close
  on complete: pool_put(m's pool, &m)
if outbound_queue empty: poller.rearm(seqn, READ only)
```

---

#### Module: `timeout_manager`

**Purpose:** Track per-Channel deadlines. Compute the minimum timeout for `poller.wait`. Identify expired Channels each iteration.

**Inputs:**
- `set(ch, kind: TimeoutKind, deadline_ms: i64)` — register or update a deadline
- `clear(ch, kind)` — remove deadline (channel moved to next state)
- `next_timeout_ms(now_ms: i64)` → i64 — minimum milliseconds until earliest deadline; 0 if any expired
- `collect_expired(now_ms: i64)` → [](*Channel) — Channels whose deadline has passed

**Outputs:** Expired Channel list for Phase 4; timeout value for Phase 1.

**Ownership:** Owned by Reactor. Timeout records reference Channels via pointer — Channel lifetimes exceed timeout records (timeout is cleared before Channel is freed).

---

### L3 — Messaging Runtime

#### Module: `message_pool`

**Purpose:** Manage the application Message lifecycle. Implements Matryoshka Pool for Messages. Provides `get(strategy)` and `put(&m)`. Executes `on_get` (initialize) and `on_put` (reset/free) hooks.

**Inputs:**
- `get(strategy: AllocationStrategy, &m: ^MayItem)` → Engine_Error or ok
  - `poolOnly` → `pool_get(.Available_Only)` — returns nil if pool empty; not an error
  - `always` → `pool_get(.Available_Or_New)` — on_get allocates if pool empty
- `put(&m: ^MayItem)` → `pool_put` — on_put resets/caps; m^ = nil after
- `create_and_dispose(allocator, initial_count: u32)` — startup: pre-populate pool; `pool_close` → dispose on Engine drain

**Outputs:** MayItem (owned Message with Header/MetaHeaders/Body zeroed and ready).

**Ownership:** Pool owns when pooled. Caller owns after `get`. Pool takes back on `put`.

**Hook: on_get (allocator from context, which is Engine's explicit allocator):**
```
if m^ == nil:
    allocate Message struct via explicit_allocator
    allocate initial Header (embedded — no heap)
    set poly.id = MessageId; set m^
else:
    m.header = {}           ← zero all header fields
    m.meta.reset()          ← clear content; retain capacity
    m.body.reset()          ← clear content; retain capacity
```

**Hook: on_put:**
```
if m.meta.capacity > options.max_appendable_capacity:
    free m.meta backing buffer via explicit_allocator
if m.body.capacity > options.max_appendable_capacity:
    free m.body backing buffer via explicit_allocator
// Leave m^ non-nil — pool stores the Message
// If pool count > options.max_messages:
//   free Message struct; set m^ = nil (pool discards)
```

**Invariant (C4):** on_get and on_put must not call pool_get or pool_put on this pool (reentrancy corruption).

---

#### Module: `reserved_pool`

**Purpose:** Provide a fixed count of Messages exclusively for engine-internal notifications (channel_closed, HelloRequest, HelloResponse, ByeResponse). These Messages never appear in `Engine.get()`.

**Inputs:**
- `get_reserved(&m: ^MayItem)` → ok / Engine_Error.ReservedPoolExhausted
  - Always `.Available_Only` — never allocates new; exhaustion is a capacity planning failure
- `put_reserved(&m: ^MayItem)` — returns Message to reserved pool; same on_get/on_put hooks as message_pool
- `create_and_dispose(allocator, count: u32)` — startup: pre-allocate `count` Messages; `pool_close` → dispose on drain

**Outputs:** MayItem (owned engine-internal Message).

**Ownership:** Reserved pool owns when pooled. Reactor owns during construction/dispatch. Application owns after delivery to outbox Mailbox. Application returns via `Engine.put(&m)` — which routes to the application pool (the application cannot distinguish reserved from application Messages; this is correct — both pools share the same Message struct and hooks).

**Invariant:** `count` must be ≥ `options.max_channels × 2` to guarantee that simultaneous close notifications for all channels can be sent without exhaustion (each close needs at most 2 notification Messages: ByeSignal + channel_closed).

---

#### Module: `mailbox_router`

**Purpose:** Route Messages to the correct Mailbox. Wire inbox (App→Reactor) and outboxes (Reactor→App). Provide the single abstraction over Matryoshka Mailbox for otofu's two-direction pattern.

**Data:**
- `reactor_inbox: Mailbox` — one global; all ChannelGroup.post() calls land here
- `outboxes: HashMap(ChannelGroupId, Mailbox)` — one per ChannelGroup; only Reactor sends

**Inputs:**
- `send_to_reactor(&m: ^MayItem)` → SendResult — application calls; `mbox_send(reactor_inbox, &m)` + Notifier.notify() ordering must be upheld by caller (L5 enforces)
- `drain_inbox(&batch)` → RecvResult — Reactor calls; `try_receive_batch(reactor_inbox)`
- `send_to_app(cg_id, &m: ^MayItem)` → SendResult — Reactor calls; `mbox_send(outboxes[cg_id], &m)`
- `wait_from_reactor(cg_id, &m: ^MayItem, timeout_ms)` → RecvResult — application calls; `mbox_wait_receive(outboxes[cg_id], &m, timeout_ms)`
- `drain_outbox(cg_id)` → list.List — during Engine drain; `mbox_close(outboxes[cg_id])`

**Outputs:** SendResult / RecvResult (Matryoshka enums). Ownership follows MayItem convention.

**Ownership:** mailbox_router owns the Mailbox structs (via Matryoshka). Messages are owned by the Mailbox queue while in transit. Transfer on send (m^ → nil) and receive (m^ → non-nil).

**Required ordering (P6 LI-8, P4):**
```
L5 post() path:
  1. mailbox_router.send_to_reactor(&m)   // mbox_send first
  2. notifier.notify()                     // wake Reactor second
```
Reversing this order is a race: Reactor may drain an empty inbox after waking.

---

#### Module: `framer`

**Purpose:** Wire-level encode and decode. Serialize Messages to byte streams for TCP send. Decode byte streams from TCP recv into Message fields. Handle partial reads (TCP stream framing).

**Wire format (from P1, P4):**

| Field | Size | Notes |
|-------|------|-------|
| Header | 16 bytes fixed | BinaryHeader: opcode, channel_num, message_id, flags, meta_len, body_len |
| MetaHeaders | meta_len bytes | CRLF-delimited key-value pairs |
| Body | body_len bytes | Opaque bytes |

`#assert(size_of(Header) == 16)` — compile-time wire format check (CC-4).

**Inputs:**
- `encode(m: *Message, out_buf: *[]u8)` — serialize Header + MetaHeaders + Body into byte buffer; may grow buffer
- `try_decode(in_buf: []u8, m: *Message)` → DecodeResult{.Incomplete, .Complete(consumed: int), .Error}
  - `.Incomplete`: not enough bytes yet; Reactor re-arms READ and waits
  - `.Complete`: fills m fields; returns bytes consumed from in_buf
  - `.Error`: protocol error; caller drives Channel to `closed`

**Outputs:** Encoded bytes (for send), decoded Message fields (for recv), `DecodeResult`.

**Ownership:** framer reads from and writes to Message fields. It does not own Messages. The Message owner (Reactor) provides the Message and retains ownership through the encode/decode operation.

---

### L4 — Protocol Layer

#### Module: `protocol`

**Purpose:** Define `OpCode` enum and dispatch incoming Messages by OpCode. Execute the conversation-layer state machine for Channels (Hello/Bye/Welcome sequences). Produce state transition directives and response Messages.

**OpCode enum (closed):**

| OpCode | Direction | Handler |
|--------|-----------|---------|
| HelloRequest | peer→us | Server: send HelloResponse; Client: should not receive (error) |
| HelloResponse | peer→us | Client: Channel → `ready`; notify app |
| WelcomeRequest | app→reactor | Open Listener; Reactor: bind+listen |
| WelcomeResponse | reactor→app | Listener active; application notified |
| Request | app↔peer | Data message; deliver to receiver |
| Response | app↔peer | Data message; deliver to receiver |
| Signal | app↔peer | Data message; deliver to receiver |
| ByeRequest | initiator→peer | Graceful close; peer sends ByeResponse |
| ByeResponse | responder→initiator | Close acknowledged; Channel → `closed` |
| ByeSignal | reactor→app | Channel closed notification; informs app |

**Dispatch invariant (PM-3, DR-4):**
```
switch m.header.opcode {
  case .HelloRequest:   handle_hello_request(...)
  case .HelloResponse:  handle_hello_response(...)
  ...
  case:                 panic("unknown OpCode — programming error")
}
```
No `#partial switch`. No silent ignore.

**Inputs:**
- `handle_inbound(m: *Message, ch: *Channel, ctx: *DispatchContext)` — called from io_dispatch after decode
- `handle_command(m: *Message, ch: *Channel, ctx: *DispatchContext)` — called from Phase 5 for inbox commands

**Outputs:** State transition directive to channel_manager; optional response Messages (from reserved_pool) enqueued to ch.outbound_queue.

**Ownership:** protocol reads incoming Messages (owned by Reactor at this point). Protocol constructs response Messages from the reserved pool (owns them until enqueued to outbound_queue; enqueue transfers ownership to queue). Protocol does not free incoming Messages — io_dispatch handles that after protocol returns.

---

#### Module: `handshake`

**Purpose:** Implement the Hello/Bye/Welcome handshake sequences in detail. Handle edge cases: simultaneous Bye (R2.1 tiebreaker), handshake timeout (driven by timeout_manager), Listener-specific WelcomeRequest path.

**Simultaneous Bye tiebreaker (R2.1):**
When both peers send ByeRequest simultaneously:
- Both enter `closing`. Both will receive a ByeRequest and a ByeResponse.
- Tiebreaker rule: the peer with the lower ChannelNumber (numerically) is the "responder" and sends ByeResponse without waiting.
- The peer with the higher ChannelNumber treats the incoming ByeRequest as the response to its own ByeRequest and closes.
- This requires ChannelNumber to be exchanged in HelloRequest/HelloResponse (included in Header.channel_num field).

**Inputs:** Incoming handshake Message, Channel state, timeout deadlines, reserved pool access.

**Outputs:** Channel state directive, response Message (from reserved pool).

**Ownership:** Handshake constructs response Messages from reserved pool (owned after pool_get). Ownership transfers to ch.outbound_queue on enqueue.

---

### L5 — Public API

#### Module: `engine`

**Purpose:** Engine lifecycle. Single entry point for the system. Creates and owns all L2/L3 structures. Provides `Engine_Error` enum. Defines `Engine_Options`.

**Engine_Options:**

| Field | Type | Default | Meaning |
|-------|------|---------|---------|
| max_messages | u32 | 64 | Application pool capacity |
| reserved_messages | u32 | 16 | Internal notification pool (must be ≥ max_channels × 2) |
| max_channels | u32 | 256 | Max concurrent Channels per ChannelGroup |
| outbound_queue_depth | u32 | 32 | Max queued outbound Messages per Channel |
| max_appendable_capacity | uint | 4096 | Max retained MetaHeaders/Body buffer per pooled Message |
| connect_timeout_ms | u32 | 5000 | TCP connect deadline |
| handshake_timeout_ms | u32 | 5000 | Hello/Welcome handshake deadline |
| bye_timeout_ms | u32 | 2000 | Bye sequence deadline before forced close |

**Engine_Error enum (closed):**

| Value | Meaning |
|-------|---------|
| .WouldBlock | Non-blocking I/O has no data (transient) |
| .ConnectionRefused | TCP ECONNREFUSED |
| .ConnectionReset | TCP ECONNRESET |
| .TimedOut | Connect/handshake/bye timeout |
| .AddressInUse | EADDRINUSE on bind |
| .ProtocolError | Malformed wire frame or invalid OpCode |
| .BackpressureExceeded | Channel outbound queue full; post() refused |
| .PoolEmpty | get(poolOnly) with empty pool; not an error — caller must handle |
| .EngineDraining | post() rejected; Engine is draining |
| .EngineDestroyed | API call on destroyed Engine |
| .ReservedPoolExhausted | Capacity planning failure; increase reserved_messages |
| .InternalError | Programming error (invalid state, bad cast, etc.) |

**Inputs:**
- `Engine.create(allocator: mem.Allocator, options: Engine_Options)` → (Engine, Engine_Error)
- `Engine.destroy(engine: Engine)` — blocking; drains and destroys all owned resources; after return caller may free allocator

**Outputs:** Engine handle (opaque; holds all system state).

**Allocator ownership (resolved from P3 Open Issue #1):**
- Caller owns the Allocator.
- Engine holds a non-owning reference.
- Engine.destroy() completes all deallocations before returning.
- After Engine.destroy() returns, the caller may safely free the Allocator.
- The Allocator must not be freed while Engine is in any state other than `destroyed`.

---

#### Module: `channel_group`

**Purpose:** The application's cross-thread messaging handle. Provides post() and waitReceive(). Carries a non-owning reference to the Engine-owned ChannelGroup struct (V-CG1).

**Inputs:**
- `ChannelGroup.create(engine: Engine)` → (ChannelGroup, Engine_Error) — Engine allocates ChannelGroup + outbox Mailbox
- `ChannelGroup.destroy(engine: Engine, cg: ChannelGroup)` — Engine destroys; application must have exited receive loop first
- `ChannelGroup.post(cg: ChannelGroup, &m: ^MayItem)` → Engine_Error — sends Message; m^ = nil on ok; m^ stays non-nil on error (TP-M2-fail)
- `ChannelGroup.waitReceive(cg: ChannelGroup, &m: ^MayItem, timeout_ms: i64)` → RecvResult — blocks; m^ = non-nil on .Ok; nil on .Timeout or .Closed
- `Channel.connect(cg: ChannelGroup, addr: Address, &m: ^MayItem)` — encode HelloRequest in m; then post(&m)
- `Channel.listen(cg: ChannelGroup, addr: Address, &m: ^MayItem)` — encode WelcomeRequest in m; then post(&m)

**Outputs:** RecvResult (.Ok / .Timeout / .Closed / .Interrupted), Engine_Error for post().

**Ownership:**
- ChannelGroup handle: non-owning reference. Application must not use after Engine.destroy().
- Message: transferred in on post() (m^ → nil); transferred out on waitReceive (m^ → non-nil).

**Single-receiver rule (V-CG2, INV-21):** Exactly one application thread calls waitReceive on any given ChannelGroup. Enforced by convention — the ChannelGroup handle must be passed to only one receiver thread.

---

#### Module: `api_message`

**Purpose:** Message construction and inspection for application code. Field accessors for Header, MetaHeaders, Body.

**Inputs:**
- `Engine.get(engine: Engine, strategy: AllocationStrategy, &m: ^MayItem)` → Engine_Error — message_pool.get wrapper
- `Engine.put(engine: Engine, &m: ^MayItem)` — message_pool.put wrapper; m^ = nil after
- `Message.set_opcode(m: *Message, op: OpCode)` — Header field
- `Message.set_channel(m: *Message, num: ChannelNumber)` — Header field
- `Message.set_message_id(m: *Message, id: u32)` — Header field; for correlation across TP-M5 silent drops
- `Message.write_meta(m: *Message, key: string, value: string)` — append to MetaHeaders Appendable
- `Message.write_body(m: *Message, data: []u8)` — append to Body Appendable
- `Message.read_opcode(m: *Message)` → OpCode
- `Message.read_channel(m: *Message)` → ChannelNumber
- `Message.read_meta(m: *Message, key: string)` → string or nil
- `Message.body_slice(m: *Message)` → []u8 — read-only view; must not hold past put()/post()

**Outputs:** Field values (read), modified Message fields (write).

**Ownership:** All operations require `owned-app` state. No access after post() or put(). Appendable buffer addresses may change after write (realloc) — caller must not cache raw buffer pointers (L-B1).

---

## 3. Thread Model

### Two Threads

```
┌──────────────────────────┐      Mailbox (mbox_send)     ┌──────────────────────────┐
│   Application Thread(s)  │ ──────────────────────────► │      Reactor Thread       │
│                          │ ◄────────────────────────── │                          │
│  L5, L3(app pool),       │      Mailbox (mbox_send)     │  L2, L1, L3(reserved,   │
│  L4(msg construction)    │      + Notifier wake         │  router, framer), L4     │
└──────────────────────────┘                              └──────────────────────────┘
```

### Reactor Thread — What It Owns Exclusively

| Object | Note |
|--------|------|
| Poller | Created at startup; closed at drain |
| Notifier | Read FD registered with Poller |
| All Socket structs and FDs | Created and closed exclusively here |
| All TriggeredChannel structs | Heap-allocated; stable addresses |
| Dual-map (both hash maps) | Not guarded; no other thread accesses |
| TriggeredChannel Pool | Reactor-internal Pool |
| Reserved Message Pool | Engine-internal notification Pool |
| All Channel structs | State machine driven here only |
| Reactor Inbox Mailbox | Drained non-blocking only |
| Pending close list | Per-iteration scratch; Reactor-private |

### Application Thread — What It Owns

| Object | Condition |
|--------|-----------|
| Messages (`owned-app`) | After `Engine.get()` or `ChannelGroup.waitReceive()`, until `post()` or `put()` |
| ChannelGroup handle | Non-owning reference; must not outlive Engine |
| Engine handle | Owning reference; caller must call Engine.destroy() |

### Cross-Thread Rules

| Rule | Mechanism |
|------|-----------|
| Only one cross-thread transfer mechanism | Matryoshka Mailbox (mbox_send / mbox_wait_receive) |
| Only one cross-thread wake mechanism | Notifier socket pair (not mbox_interrupt — DR-5) |
| App→Reactor ordering | mbox_send BEFORE Notifier.notify (P6 LI-8) |
| Reactor→App ordering | mbox_send only; app calls mbox_wait_receive independently |
| Reactor never blocks on Mailbox | try_receive_batch only (V-R1) |
| Reactor never blocks on Pool | .Available_Only only (V-R2) |
| No shared mutable state | All mutable state is either thread-confined or transferred via Mailbox |
| No mutex on hot I/O path | All I/O state mutations are Reactor-exclusive |

### Context Discipline (TH-1, MR-2)

```
Reactor thread entry point (first line, before any other operation):
  context.allocator = engine.explicit_allocator

No otofu code uses context.allocator implicitly.
Every allocation names the allocator explicitly. (MR-1, DR-3)
```

---

## 4. Message Lifecycle

### States and Ownership

| State | Owner | MayItem | Access |
|-------|-------|---------|--------|
| `nonexistent` | — | n/a | None |
| `pooled` | MessagePool | not exposed | Forbidden. Any access is a violation. |
| `owned-app` | Application thread | `m^ != nil` | Application thread only |
| `queued` | Mailbox (in transit) | n/a (inside Mailbox) | Forbidden. Mailbox internal. |
| `owned-engine` | Reactor thread | `m^ == nil` (caller's) | Reactor thread only |

### Transfer Points (all 10 — no others are legal)

| ID | From | To | Mechanism | Caller's m^ after |
|----|------|----|-----------|--------------------|
| TP-M1 | pooled | owned-app | Engine.get(always or poolOnly with non-empty pool) | non-nil |
| TP-M1-nil | pooled | owned-app (fail) | Engine.get(poolOnly, pool empty) | nil — caller must check before use |
| TP-M2 | owned-app | owned-engine | ChannelGroup.post(&m) ok | nil |
| TP-M2-fail | owned-app | owned-app (no transfer) | ChannelGroup.post(&m) error | non-nil — MUST stay; ownership did not transfer |
| TP-M3 | queued | owned-app | ChannelGroup.waitReceive(.Ok) | non-nil |
| TP-M4 | owned-app | pooled | Engine.put(&m) | nil |
| TP-M5 | owned-engine | pooled | Reactor internal on dispatch failure | n/a (no caller) |
| TP-M6 | nonexistent | owned-app | Message.create(allocator) | non-nil |
| TP-M7 | owned-app | nonexistent | Message.destroy(m, allocator) | nil |
| TP-MB | owned-app/engine | queued | mbox_send (internal) | nil |

### Buffer Rules

| Rule | Statement |
|------|-----------|
| L-M3 | Never store a raw `*Message` pointer. The `^MayItem` is the ownership token. |
| L-M5 | MetaHeaders and Body buffer references are invalid after post() or put(). |
| L-B1 | MetaHeaders / Body buffer address may change on Appendable growth. Never cache the raw pointer. |
| L-Body-P1 | Pointers embedded in Body via ptrToBody() are NOT owned by Message. Target must outlive Message. |

### Pattern: defer-put-early (Matryoshka `[itc: defer-put-early]`)

```
m: MayItem   // declare before get
defer Engine.put(engine, &m)   // defer before get; no-op if m^ == nil
Engine.get(engine, .poolOnly, &m) or return err
// ... use m
// on any early return: defer runs; m^ may be nil (post succeeded) or non-nil (get failed / post failed)
// double-put is safe (nil ^ is no-op)
```

### Pattern: errdefer → success-flag (MR-3)

```
ok := false
defer if !ok {
    Engine.put(engine, &m)   // cleanup only on error exit
}
// ... multi-step initialization
ok = true  // set before all success returns
```

---

## 5. Channel Lifecycle

### States — From the System's Perspective

```
Application thread view (via notification Messages only):
  unassigned → [opened notification: channel_num assigned]
             → [connected/listening notification: ready to use]
             → [channel_closed notification: clear state, release channel_num]

Reactor thread view (SM2, all transitions):
  unassigned → opened → connecting/listening → handshaking → ready → closing → [pending_close] → closed
```

### State Driver Table

| State transition | Driver | Phase | Via |
|-----------------|--------|-------|-----|
| unassigned → opened | Reactor | P5 | HelloRequest or WelcomeRequest command in inbox |
| opened → connecting | Reactor | P5 | Socket.connect() after ChannelNumber assigned |
| opened → listening | Reactor | P5 | Socket.bind()+listen() |
| connecting → handshaking | Reactor | P7 | WRITE event + getsockopt OK |
| connecting → pending_close | Reactor | P7/P4 | getsockopt error or connect timeout |
| listening → ready | Reactor | P7 | First accept (WelcomeResponse path) |
| handshaking → ready | Reactor | P7 | HelloResponse received (client) or sent (server) |
| handshaking → pending_close | Reactor | P4/P7 | Handshake timeout or ProtocolError |
| ready → closing | Reactor | P5/P7 | ByeRequest sent (app command) or received (peer) |
| closing → pending_close | Reactor | P7/P4 | ByeResponse received or bye timeout |
| any → pending_close | Reactor | P7 | ERROR or HUP event |
| any → pending_close | Reactor | P9 | Engine draining |
| pending_close → closed | Reactor | P8 | deregister → close(fd) → pool_put → notify → release |

### Application Notification Messages (delivered via outbox Mailbox)

| Event | OpCode | When sent |
|-------|--------|-----------|
| Connection established (client) | HelloResponse | Channel enters `ready` (client path) |
| Listener accepted connection | WelcomeResponse | New IO Server Channel enters `ready` |
| Peer initiated close | ByeSignal | Reactor receives ByeRequest from peer |
| Channel closed (any reason) | ByeSignal | Phase 8 Step 6 |

### ChannelNumber Rules

| Rule | Statement |
|------|-----------|
| Assigned | On Channel entering `opened` state |
| Released | After channel_closed notification is sent (Phase 8 Step 7) |
| Not an identity | ChannelNumber recycles. Never use as persistent session key. |
| Application duty | Clear all state keyed on ChannelNumber when ByeSignal (channel_closed) received. |

### Critical Ordering Constraint (P6 LI-5, R4.1)

```
For every Channel close:
  1. Poller.deregister(seqn)         // epoll_ctl DEL / kevent delete
  2. dual_map.remove_seqn(seqn)      // seqn no longer in Object Map
  3. dual_map.remove_chan(chan_num)
  4. Socket.set_linger(fd, true, 0)
  5. Socket.close(fd)                // FD released to OS — safe now
  6. tc_pool.put(&tc_item)           // TriggeredChannel returned to pool
  7. mailbox_router.send_to_app(channel_closed notification)
  8. channel_manager.release_number(chan_num)
```

Steps 1–6 must complete before step 5. Reversing steps 1 and 5 exposes FD ABA. Reversing step 6 and step 1 causes V-TC1 (stale pointer in Poller dispatch).

---

## 6. Backpressure Strategy

Three independent directions. Each has a defined policy.

### Outbound Backpressure (Application → Network)

**Problem:** Application posts Messages faster than the Reactor can drain them to the network. Per-Channel outbound queue grows without bound.

**Policy:** Per-Channel outbound queue limit = `options.outbound_queue_depth`.

| Condition | Action |
|-----------|--------|
| Queue depth < limit | post() enqueues Message; m^ = nil |
| Queue depth == limit | post() returns Engine_Error.BackpressureExceeded; m^ stays non-nil; application must handle |

**Application obligation:** On BackpressureExceeded, the application must not discard the Message. It holds ownership (m^ non-nil). The application may retry (after yielding to allow the Reactor to drain), or may implement application-level drop policy (put the Message back to pool).

**No automatic blocking:** post() never blocks the application thread waiting for queue space. Explicit error return forces the application to be deliberate about backpressure response.

---

### Inbound Backpressure (Network → Application)

**Problem:** Peer sends data faster than the application consumes from the outbox Mailbox.

**Policy:** Rely on TCP flow control.

| Level | Mechanism |
|-------|-----------|
| Application | Application must consume from outbox Mailbox promptly. waitReceive() is the only backpressure lever. |
| Mailbox | Unbounded by default. Outbox Mailbox grows if application is slow. |
| OS | TCP receive buffer fills. TCP window shrinks. Peer's send stalls. Natural flow control. |

**Implication:** The Reactor will continue delivering to the outbox Mailbox as fast as it decodes. If the application falls behind, memory grows. This is acceptable because TCP flow control puts a physical limit: the OS stops ACKing, the peer stops sending, recv() in the Reactor returns EAGAIN, the Reactor re-arms READ and does nothing until the application drains the outbox.

**Optional future mitigation:** Bound outbox Mailbox depth. When full, Reactor stops arming READ for that Channel. Not required for baseline otofu.

---

### Pool-Level Backpressure (Allocation)

**Problem:** Application holds all Messages in `owned-app` state. Pool is empty. Reactor needs a Message for a notification.

**Policy:** Reserved pool (V-MP2) + strategy discipline.

| Scenario | Policy |
|----------|--------|
| App pool empty + poolOnly | Engine.get() returns nil (Engine_Error.PoolEmpty). Not an error — application must handle: retry or put() a held Message first. |
| App pool empty + always | Engine.get() allocates a new Message. Pool grows. Unbounded if used persistently. |
| Reserved pool needed for notification | Reserved pool is pre-allocated; never exposed to application get(); cannot be depleted by application. |
| Reserved pool exhausted | Engine_Error.ReservedPoolExhausted logged (capacity planning failure). Notification lost. Channel still closes correctly — the close is not notification-gated. |

**Recommendation:** Application code must use `poolOnly` strategy. The `always` strategy is only for initialization or testing. A correctly configured system should never exhaust the pool under load — if it does, the pool size (`options.max_messages`) is too small.

**Reserved pool sizing rule:**
```
options.reserved_messages >= options.max_channels * 2
```
Each Channel close may require up to 2 reserved Messages (ByeResponse to peer + ByeSignal to application). All Channels may close simultaneously on Engine drain.

---

## 7. Memory Model

### One Allocator

All otofu allocations flow through a single explicit `mem.Allocator` provided to `Engine.create`. No other allocator is used internally. `context.allocator` is never used inside otofu (DR-3, MR-1).

**Allocator ownership:** Caller owns. Engine holds a non-owning reference. Engine.destroy() completes all deallocations before returning. After Engine.destroy() returns, the caller may free the Allocator.

### Allocation Map

| What | Allocated by | Freed by | Notes |
|------|-------------|----------|-------|
| Engine struct | Engine.create (from explicit allocator) | Engine.destroy | Last thing freed |
| MessagePool struct | Engine.create | Engine.destroy → pool_close → matryoshka_dispose | |
| Reserved pool struct | Engine.create | Engine.destroy → pool_close → matryoshka_dispose | |
| TriggeredChannel pool struct | Engine.create | Engine.destroy (after Reactor joins) | |
| Reactor Inbox Mailbox struct | Engine.create | Engine.destroy → mbox_close → matryoshka_dispose | |
| ChannelGroup struct + Outbox Mailbox | Engine (per ChannelGroup.create) | Engine (per ChannelGroup.destroy) | |
| Message structs | on_get hook (when pool empty + Available_Or_New) | on_put hook (when over pool cap) or Engine.destroy | Pooled; reused |
| Appendable buffers (MetaHeaders, Body) | First write to MetaHeaders/Body (via Appendable.write) | on_put hook (if over max_appendable_capacity) or Message.destroy | Retained across pool cycles |
| TriggeredChannel structs | on_get hook in tc_pool | on_put hook in tc_pool (frees if over limit — but no limit set; bounded by max_channels) | Pooled; reused |
| Socket structs | Socket.create | Socket.close | Not pooled |
| Channel structs | channel_manager.allocate | channel_manager.free (after state=closed) | Not pooled |
| Dual-map hash maps | Reactor startup | Reactor shutdown (after all channels closed) | |

### Destruction Order (Reactor thread drain sequence)

```
1. Force-close all remaining Channels (Phase 9) → all Socket.close, all tc_pool.put
2. mbox_close all outbox Mailboxes → drain and free remaining queued Messages
3. mbox_close reactor inbox → drain and free remaining inbox Messages
4. reserved_pool.close_and_dispose → free all reserved Messages
5. tc_pool.close_and_dispose → free all pooled TriggeredChannels
6. message_pool.close_and_dispose → free all pooled Messages
7. Poller.close → release OS handle
8. Notifier.close → close both FDs
9. dual_map.free → free hash map storage
10. Reactor thread exits
(Engine.create joins Reactor thread, then frees Engine struct)
```

**Rule:** Pool must be closed before Mailboxes (Matryoshka Doll 3 freeMaster ordering). If a Mailbox is closed while the Pool's on_get/on_put hooks reference Engine state, the hooks may execute after Engine state is gone.

### Init-Path Cleanup (MR-3 — errdefer replacement)

Every multi-step initialization (Engine.create, ChannelGroup.create) uses the success-flag pattern:

```
ok := false
defer if !ok {
    // reverse-order cleanup of all successfully initialized components
    // (same order as destruction sequence above, up to the failed step)
}
// step 1 ...
// step 2 ...
ok = true  // set immediately before all success returns
```

Every flag placement must be audited. A missing `ok = true` causes cleanup on success (double-free). A missing `defer if !ok` causes leak on failure.

---

## 8. OS Abstraction

L1 is the OS abstraction boundary. No L2/L3/L4/L5 code references OS types, constants, or error codes directly. All OS specifics are contained in L1 modules.

### Poller Backend Selection

```
when ODIN_OS == .linux:
    // epoll backend
    poller_create :: proc(allocator) -> (Poller, Engine_Error) { ... epoll_create1 ... }
    poller_register :: proc(p: ^Poller, fd, seqn: u64, flags: TriggerFlags) -> Engine_Error { ... epoll_ctl ADD ... }
    // EPOLLET always set; EPOLLONESHOT not used
    // seqn stored in epoll_event.data.u64

when ODIN_OS == .darwin || ODIN_OS == .freebsd || ...:
    // kqueue backend
    // EV_CLEAR for edge-triggered semantics
    // seqn stored in kevent.udata

when ODIN_OS == .windows:
    // wepoll backend (C FFI to wepoll.h — AFD_POLL driver)
    // seqn stored in OVERLAPPED-equivalent field
    // IO_STATUS_BLOCK lifetime rules apply (V-TC2 from P4)
```

### Poller Contract (platform-independent)

```
Poller.register(fd, seqn: u64, flags: TriggerFlags) → Engine_Error
  - Adds fd to kernel interest set
  - Embeds seqn in event data (epoll_data.u64 / kevent.udata)
  - TriggerFlags{READ} → EPOLLIN / EVFILT_READ
  - TriggerFlags{WRITE} → EPOLLOUT / EVFILT_WRITE
  - Edge-triggered mode always active

Poller.deregister(seqn: u64) → Engine_Error
  - Removes fd from kernel interest set
  - Must be called before Socket.close(fd)
  - After return: no further OS events for this FD will be delivered (Linux guarantee for EPOLL_CTL_DEL)

Poller.wait(timeout_ms: i64) → []Event{seqn: u64, flags: TriggerFlags}
  - Blocks until: event fires, timeout expires, or OS interrupt
  - Returns batch of (seqn, flags) pairs
  - Empty batch on timeout — not an error
  - Reactor does not call this from a non-Reactor thread

Poller.close() → void
  - Release OS handle; must be called after all FDs are deregistered
```

### Notifier Backend (platform-selected, same interface)

```
Linux / macOS:
  create: socketpair(AF_UNIX, SOCK_STREAM) → (read_fd, write_fd)
  notify: write(write_fd, [1]u8{1}) — ignores EAGAIN (pipe-full is fine)
  drain: loop recv(read_fd) until EAGAIN

Windows (AF_UNIX available Windows 10 1803+):
  create: socketpair(AF_UNIX, SOCK_STREAM) or loopback TCP localhost pair
  notify: send(write_fd, [1]u8{1})
  drain: loop recv(read_fd) until WSAEWOULDBLOCK
```

### Socket Lifecycle Rules (L1 contract for L2)

| Rule | Statement |
|------|-----------|
| All sockets non-blocking | Set immediately after creation and after accept() |
| Connect completion detection | WRITE readiness event + getsockopt(SO_ERROR) (EH-4) |
| Accept on closing Listener | FD returned by accept() when Listener is closing must be immediately closed; never stored |
| SO_LINGER on close | Applied before close(fd) on all IO sockets (all platforms) |
| Deregister before close | Mandatory; see Phase 8 ordering |

---

## 9. Failure Handling

### Failure Categories

#### Category 1: Recoverable Network Errors

Network-level I/O failures. Reactor handles. Application notified via channel_closed Message.

| Failure | Detection | Reactor action | App notification |
|---------|-----------|----------------|-----------------|
| ECONNRESET (peer RST) | recv returns error | Channel → pending_close | ByeSignal (close_reason=ConnectionReset) |
| ETIMEDOUT (connect timeout) | timeout_manager | Channel → pending_close | ByeSignal (close_reason=TimedOut) |
| EHOSTUNREACH | connect() error | Channel → pending_close | ByeSignal (close_reason=ConnectionRefused) |
| HUP (peer closed) | Poller HUP event | Channel → pending_close | ByeSignal (close_reason=Normal) |
| ERROR flag | Poller ERROR event | Channel → pending_close | ByeSignal (close_reason=NetworkError) |

These errors do not surface as Engine_Error to the application. The application receives a ByeSignal and reacts accordingly.

---

#### Category 2: Protocol Errors

Conversation-layer failures. L4 detects. Reactor executes close.

| Failure | Detection | Reactor action | App notification |
|---------|-----------|----------------|-----------------|
| Malformed wire frame | framer.try_decode → .Error | Channel → pending_close | ByeSignal (close_reason=ProtocolError) |
| Unknown OpCode from peer | protocol: exhaustive switch → panic (debug) or ProtocolError (release) | Channel → pending_close | ByeSignal |
| Handshake timeout | timeout_manager | Channel → pending_close | ByeSignal (close_reason=TimedOut) |
| Simultaneous Bye (R2.1) | handshake tiebreaker | Both sides resolve to single responder | ByeSignal on close |

**Debug vs release for unknown OpCode:**
- Debug build: panic immediately. Programming error must be caught early.
- Release build: log Engine_Error.ProtocolError; close Channel; continue.

---

#### Category 3: Resource Exhaustion

Pool and queue limits.

| Failure | Detector | Returned to | Action required |
|---------|----------|-------------|----------------|
| App pool empty + poolOnly | message_pool.get | Application (Engine_Error.PoolEmpty) | Application must put() a held Message, or retry |
| Channel outbound queue full | channel_manager.enqueue_outbound | Application (Engine_Error.BackpressureExceeded) | Application must not discard m; must retry or put() |
| Reserved pool empty | reserved_pool.get_reserved | Reactor (Engine_Error.ReservedPoolExhausted) | Notification lost; Channel still closes; increase reserved_messages |

---

#### Category 4: Engine Lifecycle Violations

API misuse at the Engine level.

| Violation | Behavior |
|-----------|----------|
| post() during Engine.draining | Engine_Error.EngineDraining returned; m^ stays non-nil |
| Any API call after Engine.destroy() | Engine_Error.EngineDestroyed; panic in debug (INV-24) |
| Engine.destroy() called twice | No-op or Engine_Error.EngineDestroyed; never double-free (INV-26) |
| Channel.connect() during Engine.draining | Engine_Error.EngineDraining; no Channel created |

---

#### Category 5: Thread Lifecycle Errors

| Failure | Behavior |
|---------|----------|
| Reactor thread fails to start | Engine stays in `starting`; Engine.create() returns Engine_Error.InternalError; all startup allocations freed in reverse order (MR-3 success-flag pattern) |
| Reactor thread panics after running | No recovery path. Engine is in undefined state. OS process should terminate. This is a programming error, not a runtime failure — the `faulted` state (P2 SM1 missing state) is not implemented in baseline otofu. |

---

#### Category 6: Deadlock Prevention

| Risk | ID | Mitigation | Where enforced |
|------|-----|-----------|----------------|
| Pool exhaustion with waitReceive block | R6.3 | Reserved pool; never exposed to application get() | reserved_pool module; Engine_Options sizing rule |
| Infinite waitReceive after drain | R6.2 | Mailbox.Closed return from mbox_wait_receive; application MUST handle .Closed | L5 waitReceive contract; Mailbox draining state |
| I/O thread blocked on pool | — | Reactor uses .Available_Only only (V-R2); never pool_get_wait | reactor module; V-R2 constraint |
| I/O thread blocked on Mailbox | — | Reactor uses try_receive_batch only (V-R1); never mbox_wait_receive | reactor module; V-R1 constraint |

---

### Failure Propagation Paths

```
Network error:
  OS errno → L1 socket.send/recv → Engine_Error mapping
           → L2 io_dispatch → pending_close
           → L2 reactor Phase 8 → channel_closed notification
           → L3 mailbox_router.send_to_app
           → Application.waitReceive → ByeSignal Message

Protocol error:
  L4 framer.try_decode(.Error) or protocol.handle_inbound(.Error)
           → L2 io_dispatch → pending_close
           → same path as network error above

Resource error:
  L3 message_pool.get → Engine_Error.PoolEmpty
           → L5 Engine.get → returned to application thread

API misuse:
  L5 Engine.post() during drain → Engine_Error.EngineDraining → returned to application
  Application's m^ stays non-nil (TP-M2-fail invariant upheld)

Fatal error (programming error in debug):
  L4 protocol: unknown OpCode → panic
  L2 channel_manager.transition: illegal transition → panic
  L1 socket: EBADF / EINVAL → Engine_Error.InternalError → panic in debug
```

### Recovery Policy

| Level | Recovery |
|-------|---------|
| Channel | Recoverable. Close the Channel. Application may open a new one. |
| Engine | Not recoverable during operation. Engine.destroy() is the only valid response to Engine-level failure. |
| Message | Ownership-explicit. post() failure → m^ non-nil → application decides: retry, put(), or destroy(). |
| Process | Reactor thread panic → no Engine recovery. Process must restart. |

---

## Invariant Index

The following invariants are enforced by the architecture. Violation of any one is a defect, not a configuration issue.

| ID | Layer | Statement |
|----|-------|-----------|
| AI-1 | L2 | All dual-map INSERTs occur in Phase 5. All REMOVEs in Phase 8. No mutation in Phase 6 or 7. |
| AI-2 | L2 | Phase 6 resolves all TriggeredChannel pointers before Phase 7 dispatches any. |
| AI-3 | L2 | Reactor never calls mbox_wait_receive or pool_get_wait. |
| AI-4 | L2 | Deregister from Poller before close(fd). close(fd) before pool_put(tc). |
| AI-5 | L2 | channel_closed notification is sent before ChannelNumber is released. |
| AI-6 | L3 | Application pool and reserved pool are never mixed. Application never draws from reserved pool. |
| AI-7 | L3 | mbox_send(reactor_inbox) is called before Notifier.notify() — always. |
| AI-8 | L4 | All OpCode dispatch uses exhaustive switch with panic-on-default. |
| AI-9 | L5 | post() failure leaves m^ non-nil. Ownership stays with caller. |
| AI-10 | L5 | Exactly one thread calls waitReceive on any given ChannelGroup. |
| AI-11 | L0 | PolyNode is the first field (offset 0) in every struct that travels through Matryoshka. |
| AI-12 | L0 | on_get and on_put hooks do not call pool_get or pool_put on the same pool. |
| AI-13 | All | No otofu code uses context.allocator. All allocations name the allocator explicitly. |
| AI-14 | All | Message state transitions: owned-app only from application thread; owned-engine only from Reactor thread. |
| AI-15 | L5 | Allocator outlives Engine. Engine.destroy() completes all deallocs before returning. |
