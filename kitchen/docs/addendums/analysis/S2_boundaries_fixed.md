# S2 — Module Boundaries (Fixed)

Sources: S1_modules_fixed.md, P3_ownership.md
Pipeline position: S2 (boundary enforcement — derived from S1 module catalog)

## Corrections Applied

| Rule | Violation in original | Fix |
|------|-----------------------|-----|
| RULE-1 | Shared Types table missing ProtoFields, OriginFlag, MoreMessagesFlag, BinaryHeader | Added |
| RULE-1 | Shared Types table missing TextHeaders, TextHeader, TextHeaderIterator | Added |
| RULE-1/4/6/7 | Address listed as TCP-only, passive, client/server collapsed | Four active variants (TCP_Client, TCP_Server, UDS_Client, UDS_Server); format/parse behavior; header integration in message/ |
| RULE-8 | IntrusiveQueue/MessageQueue absent from design | Message_Queue in message/queue.odin — single-threaded FIFO; part of public message API |
| RULE-2 | Thread Access Map missing `updateReceiver` for channel_group | Added |
| RULE-2 | channel_group boundary missing updateReceiver operation | Added |
| RULE-2 | engine boundary missing getAllocator | Added |
| RULE-2 | api_message boundary references "meta" buffers | Corrected to "thdrs" (TextHeaders) |
| RULE-1 | framer wire format constraint references wrong "Header" type | Corrected to BinaryHeader |
| Structural | api_message listed as in otofu/ package | Note: api_message is in top-level message/ package |

---

This document defines strict access rules for every module. Each entry states:
what the module owns, what it may access without owning, what is explicitly forbidden,
and which dependencies are allowed vs prohibited.

No module may access another module's owned state directly.
No module may call a module in a higher layer.
No module may hold shared ownership with another module.

---

## Shared Types (No Layer — Accessible to All)

The following types are data definitions with no layer assignment. Any module at any layer may use them as parameter types, return types, or field types. Using them does not create a layer dependency.

| Type | Definition location | Purpose |
|------|-------------------|---------|
| `Engine_Error` | Shared types package | Closed error enum; 13 values |
| `MayItem` | Matryoshka (L0) | Ownership token: `Maybe(^PolyNode)` |
| `PolyNode` | Matryoshka (L0) | Intrusive list node |
| `TriggerFlags` | Shared types package | Packed u8 I/O event flags |
| `Address` | `message/address.odin` | Network address — four active variants: TCP_Client, TCP_Server, UDS_Client, UDS_Server; format/parse to TextHeader value string; header integration in same file (consolidated) |
| `Message_Queue` | `message/queue.odin` | Single-threaded intrusive FIFO for `*Message`; used by chanmgr (outbound queue) and reactor (scratch collections) |
| `ChannelNumber` | Shared types package | u16 Channel correlation token |
| `SequenceNumber` | Shared types package | u64 ABA token |
| `OpCode` | Shared types package | Closed enum; 10 wire protocol values (Request, Response, Signal, HelloRequest, HelloResponse, WelcomeRequest, WelcomeResponse, ByeRequest, ByeResponse, ByeSignal) |
| `ProtoFields` | Shared types package | Packed u8: opCode(u4), origin(u1), more(u1), _internalA(u1), _internalB(u1) |
| `OriginFlag` | Shared types package | u1 enum: application(0) or engine(1) |
| `MoreMessagesFlag` | Shared types package | u1 enum: last(0) or more(1) |
| `BinaryHeader` | Shared types / message package | 16-byte packed wire header: channel_number(u16), proto(ProtoFields), status(u8), message_id(u64), `<thl>`(u16), `<bl>`(u16) |
| `TextHeader` | Shared types / message package | Single key-value header: name []u8, value []u8 |
| `TextHeaderIterator` | Shared types / message package | Stateful iterator over TextHeaders bytes |
| `TextHeaders` | Shared types / message package | Key-value pairs buffer in "name: value\r\n" format |
| `ChannelState` | Shared types package | SM2 state enum |
| `TC_State` | Shared types package | SM5 state enum |
| `AllocationStrategy` | Shared types package | poolOnly / always |
| `RecvResult` | Matryoshka (L0) | Mailbox receive outcome |
| `SendResult` | Matryoshka (L0) | Mailbox send outcome |
| `DecodeResult` | Shared types package | Framer decode outcome |

Using `Engine_Error` in L1 is NOT a dependency on the L5 `engine` module.
Using `MayItem` in L3 is NOT a dependency on L0 modules beyond the type definition.
Using `BinaryHeader`, `TextHeaders` in L3/L4 is NOT a dependency on the L5 `message` module.

---

## Thread Access Map

Modules are partitioned by which thread(s) may call them.

| Module | Reactor thread | Application thread | Notes |
|--------|---------------|-------------------|-------|
| `matryoshka.poly` | Yes | Yes | Value type operations |
| `matryoshka.mailbox` | Yes (try_receive_batch, mbox_send to outboxes) | Yes (mbox_send to inbox, mbox_wait_receive from outboxes) | Thread-safe internally |
| `matryoshka.pool` | Yes | Yes (via L5 wrappers only) | Thread-safe internally |
| `poller` | **Reactor only** | **FORBIDDEN** | OS handle is Reactor-private |
| `notifier` | Yes (drain) | Yes (notify) | Different operations per thread |
| `socket` | **Reactor only** | **FORBIDDEN** | All FDs are Reactor-exclusive |
| `reactor` | **Reactor only** | **FORBIDDEN** | Entry point IS the Reactor thread |
| `dual_map` | **Reactor only** | **FORBIDDEN** | No lock; Reactor-exclusive |
| `tc_pool` | **Reactor only** | **FORBIDDEN** | Reactor-internal pool |
| `channel_manager` | **Reactor only** | **FORBIDDEN** | All Channel state is Reactor-private |
| `io_dispatch` | **Reactor only** | **FORBIDDEN** | I/O event handler |
| `timeout_manager` | **Reactor only** | **FORBIDDEN** | Reactor-internal tracking |
| `message_pool` | Reactor (put only, via reserved return path) | Yes (get/put via L5) | Thread-safe via Matryoshka |
| `reserved_pool` | **Reactor only** | **FORBIDDEN** | Internal notification pool |
| `mailbox_router` | Yes (send_app, drain_inbox) | Yes (send_reactor, wait_app) | Different operations per thread |
| `framer` | Yes (encode for send, decode from recv) | No (application does not touch wire bytes) | |
| `protocol` | **Reactor only** | **FORBIDDEN** | Executes inside Reactor event loop |
| `handshake` | **Reactor only** | **FORBIDDEN** | Executes inside Reactor event loop |
| `engine` | No direct calls | Yes (create/destroy/get_allocator) | Reactor is spawned by engine |
| `channel_group` | No direct calls | Yes (post/waitReceive/updateReceiver/connect/listen) | Application-facing API |
| `api_message` | No direct calls | Yes (get/put/read/write) | Application-facing API |

---

## Layer Access Matrix

```
Caller →  L0   L1   L2   L3   L4   L5
L0        —    ✗    ✗    ✗    ✗    ✗    (external; calls nothing)
L1        ✓    —    ✗    ✗    ✗    ✗    (OS only)
L2        ✓    ✓    ↕*   ✓    ✓    ✗    (Reactor coordinates)
L3        ✓    ✗    ✗    ↕*   ✗    ✗    (Pool and Mailbox only)
L4        ✗    ✗    ✓**  ✓    ↕*   ✗    (issues directives to L2 modules; reads from L3)
L5        ✗    ✗    ✗    ✓    ✗    ↕*   (wraps L3 only)

✓  = allowed
✗  = FORBIDDEN
↕* = lateral within same layer: allowed only between explicitly listed module pairs
✓**= L4 may call specific L2 modules (channel_manager) for state transitions only;
     L4 may NOT call reactor, dual_map, tc_pool, io_dispatch, timeout_manager
```

**Critical rule:** No module may call any module in a higher layer than itself. The only exception is shared types (no layer).

---

## Per-Module Boundary Definitions

Format: Owns / Can Access (non-owning) / Allowed Dependencies / Forbidden Dependencies / Forbidden Actions

---

### `matryoshka.poly` — L0 External

**Owns:** Nothing. PolyNode is embedded in caller structs. MayItem is a value type held by callers.

**Can Access:** Nothing.

**Allowed Dependencies:** None.

**Forbidden Dependencies:** All otofu modules.

**Forbidden Actions:**
- otofu must not modify, wrap, or re-export PolyNode or MayItem
- otofu must not bypass the offset-0 rule for any traveling struct

---

### `matryoshka.mailbox` — L0 External

**Owns:** Mailbox struct internals (lock, linked list of queued nodes). Items in the queue are owned by the Mailbox — neither sender nor receiver owns them while queued.

**Can Access:** PolyNode internals.

**Allowed Dependencies:** `matryoshka.poly`.

**Forbidden Dependencies:** All otofu modules.

**Forbidden Actions:**
- otofu callers must not call `mbox_interrupt` as a substitute for `notifier_notify` (DR-5)
- otofu callers must not call `mbox_wait_receive` from the Reactor thread (AI-3, V-R1)
- otofu callers must not call `mbox_send` to a closed Mailbox and ignore the error (INV-22)

---

### `matryoshka.pool` — L0 External

**Owns:** Pool struct internals (lock, free-list). Items in the free-list are owned by the Pool.

**Can Access:** PolyNode internals.

**Allowed Dependencies:** `matryoshka.poly`.

**Forbidden Dependencies:** All otofu modules.

**Forbidden Actions:**
- otofu hook implementations (`on_get`, `on_put`) must not call `pool_get` or `pool_put` on the same pool (C4 — reentrancy corruption)
- otofu must not call `pool_get_wait` from the Reactor thread (V-R2)

---

### `poller` — L1

**Owns:** OS multiplexer handle (`epoll_fd` / `kqueue_fd` / wepoll handle). Registration metadata internal to the OS (not accessible as a data structure).

**Can Access:** OS syscall layer only. Receives `fd` values from caller. Receives `seqn` (u64) from caller; treats as opaque data.

**Allowed Dependencies:** OS syscall layer. No otofu module.

**Forbidden Dependencies:** `notifier`, `socket`, any L2–L5 module.

**Forbidden Actions:**
- May not know what Channel, Message, TriggeredChannel, or SequenceNumber mean semantically
- May not interpret event data beyond returning the (seqn, flags) pair it was given
- May not block except inside `poller_wait`
- Application thread must not call any `poller_*` procedure

---

### `notifier` — L1

**Owns:** `read_fd`, `write_fd` (both socket FDs created by `notifier_create`).

**Can Access:** OS socket API (write/recv on owned FDs).

**Allowed Dependencies:** OS socket API. No otofu module.

**Forbidden Dependencies:** `poller`, `socket`, any L2–L5 module, `matryoshka.mailbox` (mbox_interrupt must not substitute for notifier_notify — DR-5).

**Forbidden Actions:**
- `notifier_notify` must not be called BEFORE `mailbox_router.send_to_reactor` (ordering invariant AI-7) — this is a caller obligation enforced by `channel_group.cg_post`
- Reactor thread must not call `notifier_notify` (write side is application-only)
- Application thread must not call `notifier_drain` (read side is Reactor-only)

---

### `socket` — L1

**Owns:** `Socket` struct per instance. The OS file descriptor (`fd`) embedded in each Socket struct.

**Can Access:** OS socket API. Nothing else.

**Allowed Dependencies:** OS socket API. `Engine_Error` shared type (for return values only).

**Forbidden Dependencies:** `poller`, `notifier`, any L2–L5 module.

**Forbidden Actions:**
- May not know what Channel, Message, or ChannelNumber mean
- May not block (all socket operations are non-blocking; SO_LINGER applied before close, not during)
- Must not close a fd that is still registered with Poller — enforced by caller (reactor Phase 8 ordering)
- Application thread must not call any `socket_*` procedure
- `socket_close` must be called exactly once per Socket; caller ensures no double-close
- UDS: `socket` is responsible for unlinking the socket path file on close (or signaling caller to do so)

---

### `reactor` — L2

**Owns:**
- `Poller` instance (one per Engine)
- `Notifier` instance (read_fd registered with Poller)
- `Dual_Map` instance
- `TC_Pool` instance
- `Channel_Manager` instance
- `Timeout_Manager` instance
- Per-iteration scratch buffers: `io_events[]`, `resolved[]`, `pending_close[]`, `inbox_pending: bool`

**Can Access (non-owning):**
- `mailbox_router` (calls drain_inbox in Phase 5; does not own the Mailbox instances)
- `reserved_pool` (calls reserved_get for notification Messages; does not own the Messages while the Mailbox holds them)
- `message_pool` (calls put for application Messages silently returned on dispatch failure — TP-M5)
- Engine state reference (to signal `running` / `destroyed`)

**Allowed Dependencies:**
`poller`, `notifier`, `dual_map`, `tc_pool`, `channel_manager`, `io_dispatch`, `timeout_manager`, `mailbox_router`, `reserved_pool`, `message_pool`, `protocol` (for inbox command dispatch in Phase 5).

**Forbidden Dependencies:**
`socket` (direct — socket operations go through `io_dispatch`), `framer`, `handshake`, `engine`, `channel_group`, `api_message`.

**Forbidden Actions:**
- Must not call `mbox_wait_receive` on any Mailbox (V-R1, AI-3)
- Must not call `pool_get_wait` on any Pool (V-R2)
- Must not mutate `dual_map` in Phases 6 or 7 (AI-1: INSERTs in Phase 5, REMOVEs in Phase 8 only)
- Must not call `io_dispatch` before Phase 6 completes (all events resolved before any dispatch — AI-2)
- Must not be called from the application thread
- Phase 8 close sequence must follow: deregister → remove from dual_map → close(fd) → pool_put(tc) → send_to_app(notification) → release_number (LI-5, AI-5)

---

### `dual_map` — L2

**Owns:**
- `chan_to_seqn: HashMap(ChannelNumber, SequenceNumber)`
- `seqn_to_tc: HashMap(SequenceNumber, *TriggeredChannel)`

**Can Access:** Externally-provided `*TriggeredChannel` pointers and scalar types. Does not access the TriggeredChannel struct contents.

**Allowed Dependencies:** None. Pure data structure.

**Forbidden Dependencies:** All other modules.

**Forbidden Actions:**
- Must not be accessed from the application thread (no lock; Reactor-exclusive)
- Must not dereference `*TriggeredChannel` pointers (stores and returns them opaquely)
- Insertion must not occur during Phase 6 or Phase 7 (caller responsibility; AI-1)
- Removal must not occur during Phase 5 or Phase 6 (caller responsibility; AI-1)

---

### `tc_pool` — L2

**Owns:**
- One Matryoshka `Pool` instance
- All `TriggeredChannel` instances while pooled (linked via PolyNode)

**Can Access:**
- Explicit allocator (received via hook context; used to allocate/free TriggeredChannel structs)

**Allowed Dependencies:** `matryoshka.pool`, `matryoshka.poly`. Explicit allocator parameter.

**Forbidden Dependencies:** `dual_map`, `channel_manager`, `io_dispatch`, `timeout_manager`, `reactor`, any L3–L5 module.

**Forbidden Actions:**
- `pool_put` (tc_pool.put) must not be called while the TriggeredChannel is still registered in `dual_map` or Poller — caller responsibility (V-TC1)
- `pool_put` must not be called while a kernel AFD_POLL operation is pending on the TriggeredChannel (Windows; V-TC2)
- Must not call `pool_get_wait` (blocks Reactor; V-R2)
- Must not be called from the application thread
- `on_get` and `on_put` hooks must not call `pool_get` or `pool_put` on this pool (C4)

---

### `channel_manager` — L2

**Owns:**
- All `Channel` structs across all ChannelGroups
- Per-Channel outbound send queues (`Queue(*Message)` per Channel)
- Per-Channel receive buffers (`recv_buf: []u8` per Channel, for framing cursor)
- ChannelNumber bitmap (allocates and releases u16 values in range 1–65534)

**Can Access (non-owning references):**
- `*Socket` (referenced by each Channel; Socket is owned by `reactor`/`socket` module)
- `*TriggeredChannel` (referenced by each Channel; owned by `tc_pool`)
- `*ChannelGroup` (referenced by each Channel; ChannelGroup struct owned by Engine via `engine` module)

**Allowed Dependencies:** None. Pure data and state machine logic.

**Forbidden Dependencies:** `socket`, `poller`, `dual_map`, `tc_pool`, `io_dispatch`, `timeout_manager`, `reactor`, any L3–L5 module.

**Forbidden Actions:**
- Must not perform socket I/O (send/recv/accept) — that is `socket` and `io_dispatch`
- Must not call Poller registration or deregistration — that is `reactor` via `poller`
- Must not call Mailbox operations — that is `mailbox_router`
- Must not be accessed from the application thread (all Channel state is Reactor-exclusive, INV-13)
- `ch_mgr_transition` must validate legality: panic in debug on illegal SM2 transition; return `Engine_Error.InternalError` in release
- `ch_mgr_free` must only be called when Channel is in `closed` state
- `ch_mgr_release_number` must only be called AFTER the channel notification is sent (AI-5)

**Shared ownership hazard — none:** Socket, TriggeredChannel, and ChannelGroup are referenced (non-owning). channel_manager does not free them. Their lifecycle is managed by their respective owners.

---

### `io_dispatch` — L2

**Owns:** Nothing. Stateless per invocation.

**Can Access (via Dispatch_Context):**
- `*Channel` (non-owning; owned by `channel_manager`)
- `*TriggeredChannel` (non-owning; owned by `tc_pool`)
- All modules in Dispatch_Context: channel_manager, dual_map, mailbox_router, reserved_pool, message_pool, timeout_manager

**Allowed Dependencies:**
`socket`, `framer`, `protocol`, `channel_manager`, `dual_map`, `mailbox_router`, `reserved_pool`, `message_pool`, `timeout_manager`.

**Forbidden Dependencies:**
`reactor` (no upward call within L2), `tc_pool` (pool_put is called by `reactor` Phase 8, not by io_dispatch), `poller` (re-arm via reactor, not direct), `engine`, `channel_group`, `api_message`, `handshake` (called via `protocol`, not directly).

**Forbidden Actions:**
- Must not call `reactor` — io_dispatch is called BY reactor, not the other way
- Must not call `tc_pool.put` — TriggeredChannel return to pool is Phase 8 (reactor), not Phase 7 (io_dispatch)
- Must not call `poller_deregister` — deregistration is Phase 8 (reactor)
- Must not insert new TriggeredChannels into dual_map after Phase 5 ends — accepted Channels' new TCs ARE inserted here (Phase 7 accept path is the one exception, explicitly permitted by AI-2)
- Must not block on any Mailbox or Pool operation
- Must not be called from the application thread

**Timeout management rule:** After calling `channel_manager.transition(ch, new_state)`, `io_dispatch` is responsible for calling `timeout_manager.set` or `timeout_manager.clear` on the same Channel based on the resulting state.

---

### `timeout_manager` — L2

**Owns:** Deadline table (per-Channel, per-deadline-kind entries; monotonic timestamps).

**Can Access (non-owning):** `*Channel` pointers (for deadline table keys; does not access Channel fields).

**Allowed Dependencies:** None. Pure data structure.

**Forbidden Dependencies:** All other modules.

**Forbidden Actions:**
- Must not dereference `*Channel` pointers (stores and returns them opaquely)
- Must not be accessed from the application thread
- Deadline entries must always be cleared before `channel_manager.free(ch)` is called — caller responsibility

---

### `message_pool` — L3

**Owns:**
- One Matryoshka `Pool` instance
- All `Message` instances while pooled
- All `Appendable` backing buffers inside pooled Messages (capacity retained, content cleared)

**Can Access:**
- Explicit allocator (via hook context; for Message struct allocation/free)
- Message struct fields (via `on_get` and `on_put` hooks): `bh` (BinaryHeader), `thdrs` (TextHeaders), `body` (Appendable)

**Allowed Dependencies:** `matryoshka.pool`, `matryoshka.poly`. Explicit allocator.

**Forbidden Dependencies:** `reserved_pool`, any L2 module, any L4–L5 module.

**Forbidden Actions:**
- `on_get` and `on_put` hooks must not call `pool_get` or `pool_put` on this pool (C4)
- Must not be called from the Reactor thread for new Message allocation — Reactor uses `reserved_pool` for internal Messages
- Reactor may call `message_pool.put` for TP-M5 (silent pool return on dispatch failure) — this is the only Reactor access to this pool
- Must not call `pool_get_wait` (would block caller thread; V-R2 for Reactor, bad practice for application)
- Must not share Message instances with `reserved_pool` — the two pools are strictly separate

**Ownership boundary:** After `pool_put(&m)`, `m^` is nil. The pool owns the Message. The caller must not access the Message or its buffer contents after put (INV-08, INV-09).

---

### `reserved_pool` — L3

**Owns:**
- One Matryoshka `Pool` instance (separate from `message_pool`)
- Pre-allocated `options.reserved_messages` Message instances

**Can Access:** Same as `message_pool` — explicit allocator, Message struct fields (`bh`, `thdrs`, `body`), Appendable buffers.

**Allowed Dependencies:** `matryoshka.pool`, `matryoshka.poly`. Explicit allocator.

**Forbidden Dependencies:** `message_pool`, any L2 module, any L4–L5 module.

**Forbidden Actions:**
- Must not expose `reserved_get` to application threads — engine-internal use only
- Must not call `pool_get_wait` — `.Available_Only` mode only; exhaustion is a capacity failure, not a wait condition
- Must not allocate new Messages on exhaustion — pre-allocated count is fixed
- Application calls `engine_put(&m)` which routes to `message_pool`, not back to `reserved_pool` — this is correct and intentional; do not attempt to detect and re-route reserved Messages

**Size invariant:** `options.reserved_messages ≥ options.max_channels × 2`. Violating this is a capacity planning error that causes `Engine_Error.ReservedPoolExhausted`.

---

### `mailbox_router` — L3

**Owns:**
- `reactor_inbox: Mailbox` — one instance (created on Engine startup; closed during Engine drain)

**Can Access (non-owning references):**
- Per-ChannelGroup outbox Mailbox instances — owned by ChannelGroup structs (Engine scope); `mailbox_router` holds pointers for routing but does not own them

**Allowed Dependencies:** `matryoshka.mailbox`, `notifier` (for `notifier_notify` after `send_to_reactor`).

**Forbidden Dependencies:** `message_pool`, `reserved_pool`, `framer`, any L2 module, any L4–L5 module.

**Forbidden Actions:**
- Must not interpret Message content (no OpCode, no Channel semantics)
- Must not call `mbox_wait_receive` from the Reactor thread — Reactor uses `router_drain_inbox` (try_receive_batch only)
- `notifier_notify` must be called ONLY inside `router_send_reactor`, AFTER `mbox_send` completes — callers must use `router_send_reactor`, not call both manually (AI-7)
- Must not expose raw Mailbox handles to callers — routing is opaque to caller
- The `reactor_inbox` Mailbox must not be shared with `reserved_pool` or `tc_pool` — it is exclusively the App→Reactor command channel

**Thread access rule:**
- `router_send_reactor(&m)` — application thread only
- `router_drain_inbox(batch)` — Reactor thread only
- `router_send_app(cg_id, &m)` — Reactor thread only
- `router_wait_app(cg_id, &m, timeout)` — application thread only (one thread per ChannelGroup)

---

### `framer` — L3

**Owns:** Nothing. Stateless.

**Can Access:** Caller-provided `*Message` (reads `bh`/BinaryHeader, `thdrs`/TextHeaders, `body`) and caller-provided byte buffers. `recv_buf` is owned by `channel_manager` (via Channel struct) and passed to `framer_try_decode` by `io_dispatch`.

**Allowed Dependencies:** None (operates purely on caller-provided types).

**Forbidden Dependencies:** All modules. Framer has zero module dependencies.

**Forbidden Actions:**
- Must not interpret OpCode or make protocol decisions — content-blind encoding/decoding only
- Must not allocate memory internally — uses caller-provided output buffers
- Must not retain pointers to Message fields between calls — stateless per invocation
- Must not access `channel_manager`, `mailbox_router`, or any routing logic
- Wire format BinaryHeader must be exactly 16 bytes: `#assert(size_of(BinaryHeader) == 16)` must pass at compile time (CC-4)

---

### `protocol` — L4

**Owns:** Nothing. OpCode is a shared type definition. Dispatch is a stateless function.

**Can Access:**
- Incoming `*Message` fields (reads `bh.proto.opCode`, `bh.channel_number`, `bh.message_id`, `bh.status` from BinaryHeader; reads TextHeaders/Body as needed)
- `*Channel` state (reads state for dispatch decisions)
- `*ChannelGroup` reference (for notification routing)

**Allowed Dependencies (L4 lateral):** `handshake`.
**Allowed Dependencies (downward L3):** `reserved_pool`, `mailbox_router`.
**Allowed Dependencies (downward L2):** `channel_manager` (state transitions only — the one permitted L4→L2 call).

**Forbidden Dependencies:** `reactor`, `dual_map`, `tc_pool`, `io_dispatch`, `timeout_manager`, `poller`, `socket`, `notifier`, `message_pool` (application pool), `engine`, `channel_group`, `api_message`, `framer`.

**Forbidden Actions:**
- Must not perform socket I/O
- Must not access Poller or dual_map
- Must not call `message_pool.get` — response Messages come from `reserved_pool` only
- Must not call `mailbox_router.send_to_reactor` — protocol is inbound-only; no sending application commands
- Must not use `#partial switch` on OpCode — exhaustive switch with panic-default required (PM-3, DR-4)
- Must not access `timeout_manager` — timeout side effects are handled by `io_dispatch` after protocol returns
- Must not be called from the application thread

**Ownership rule:** Incoming Message is owned by the Reactor when protocol is called. Protocol reads it but does not take ownership. Protocol allocates response Messages from `reserved_pool` and passes them to `channel_manager` outbound queue or `mailbox_router` — ownership transfers at those call sites.

---

### `handshake` — L4

**Owns:** Nothing. Stateless. Sequence state lives in `Channel` struct (owned by `channel_manager`).

**Can Access:**
- `*Channel` state (reads and drives via `channel_manager` transition calls)
- `*Message` fields (reads incoming handshake content: `bh.proto.opCode`, TextHeaders for address extraction)

**Allowed Dependencies (downward L3):** `reserved_pool`, `mailbox_router`.
**Allowed Dependencies (downward L2):** `channel_manager`.

**Forbidden Dependencies:** `reactor`, `dual_map`, `tc_pool`, `io_dispatch`, `timeout_manager`, `poller`, `socket`, `notifier`, `message_pool`, `framer`, `protocol` (no lateral upward call to dispatcher), `engine`, `channel_group`, `api_message`.

**Forbidden Actions:**
- Must not call `timeout_manager` — timeouts are set by `io_dispatch` after handshake returns
- Must not call `message_pool.get` — response Messages from `reserved_pool` only
- Must not call `router_send_to_reactor` — handshake only produces outbound Messages to peer or notifications to application
- Must not be called from the application thread
- Must not access any L1 module

**Simultaneous-Bye tiebreaker rule:** When both peers have sent ByeRequest, the peer with the **lower ChannelNumber** sends ByeResponse and closes. `handshake` reads `ch.remote_channel_num` (set during Hello exchange) and `ch.number` (local) to determine role.

---

### `engine` — L5

**Owns:**
- `Engine` struct (root aggregate): explicit allocator reference (non-owning), all submodule instances, Reactor thread handle, Engine state
- The Engine struct is allocated from the provided allocator; freed by `engine_destroy` before returning

**Can Access (non-owning):**
- Caller-provided `mem.Allocator` — held as reference; caller owns; must outlive Engine

**Allowed Dependencies (downward L3):** `message_pool`, `reserved_pool`, `mailbox_router`.
**Allowed Dependencies (downward L2):** `reactor` (spawns thread), `dual_map`, `tc_pool`, `channel_manager`, `timeout_manager`.

**Forbidden Dependencies:** `poller`, `socket`, `notifier` (all L1 — accessed via Reactor thread only), `protocol`, `handshake`, `framer`, `io_dispatch`, `channel_group`, `api_message`.

**Forbidden Actions:**
- Must not perform any I/O directly — all I/O is Reactor-thread-only
- Must not access L1 modules — the Reactor owns and uses L1
- Must not share the Engine allocator with any external code while Engine is alive (INV-27)
- Must not return from `engine_destroy` before all deallocations complete (Allocator ownership rule)
- `engine_destroy` must join the Reactor thread before freeing any owned state

**Public procedures (RULE-2):**
- `Engine_Create(opts, allocator)` — allocate and start
- `Engine_Destroy(engine)` — drain, join, free
- `Engine_Get_Allocator(engine)` — return the caller-provided allocator

**Destruction order enforced by `engine_destroy`:**
```
1. Inject EngineShutdown command into reactor_inbox (via mailbox_router)
2. notifier_notify (wake Reactor)
3. Join Reactor thread (Reactor completes drain and exits)
4. mbox_close all ChannelGroup outbox Mailboxes; dispose remaining Messages
5. mbox_close reactor_inbox; dispose remaining Messages
6. reserved_pool.close_and_dispose
7. tc_pool.close_and_dispose
8. message_pool.close_and_dispose
9. dual_map.free
10. timeout_manager.free
11. channel_manager.free (all Channels must be closed by this point)
12. Free Engine struct
```

Caller may free Allocator after step 12 returns.

---

### `channel_group` — L5

**Owns:** Procedures only. The `ChannelGroup` struct (containing the outbox Mailbox) is owned by Engine; `engine_create_cg` allocates it; `engine_destroy_cg` frees it. The L5 `channel_group` module provides the API procedures that operate on ChannelGroup handles.

**Can Access:**
- `ChannelGroup` handle (non-owning reference; passed by caller)
- `mailbox_router` (for post and waitReceive operations)
- `reserved_pool` (for connect/listen control Message allocation — encoding HelloRequest/WelcomeRequest with address in TextHeaders)

**Allowed Dependencies (downward L3):** `mailbox_router`, `reserved_pool`.

**Forbidden Dependencies:** `message_pool` (control Messages use reserved pool, not application pool), any L2 module, any L4 module, `engine` (no circular L5 dependency), `api_message`.

**Forbidden Actions:**
- Must not access `dual_map`, `channel_manager`, `tc_pool`, `io_dispatch` — all Reactor-internal
- Must not call any L1 module
- Must not call any L4 module — protocol interpretation is Reactor-side
- `cg_post(&m)` must call `router_send_reactor(&m)` first, then `notifier_notify` — caller does not call these separately (AI-7)
- On `cg_post` error (`mbox_send` fails — e.g., Mailbox draining/closed), `m^` must remain non-nil — ownership stays with caller (TP-M2-fail, AI-9)
- `cg_wait_receive` must not be called by more than one thread per ChannelGroup — enforced by convention; L5 does not prevent it (AI-10)
- All ChannelGroup handles must be abandoned by the application before `engine_destroy` is called — L5 does not prevent use-after-free; this is a caller obligation (V-CG1)

**Public procedures (RULE-2 — full tofu ChannelGroup API):**
- `CG_Post(cg, &m)` — submit message for send or engine command
- `CG_Wait_Receive(cg, &m, timeout_ns)` — blocking receive from outbox
- `CG_Update_Receiver(cg, &m)` — inject message into outbox from any thread; wake the receiver
- `CG_Connect(cg, addr, &m)` — encode HelloRequest with connect_to header; call CG_Post
- `CG_Listen(cg, addr, &m)` — encode WelcomeRequest with listen_on header; call CG_Post

**`cg_post` ownership contract:**
```
Before: m^ != nil (caller owns Message)
Call:   cg_post(cg, &m)
After on ok:    m^ == nil (Reactor owns)
After on error: m^ != nil (caller still owns — MUST NOT be nil on any error path)
```

**`cg_update_receiver` ownership contract:**
```
Before: m^ != nil OR m^ == nil (both valid)
Call:   cg_update_receiver(cg, &m)
After on ok (m^ was not nil): m^ == nil (outbox Mailbox owns Message)
After on ok (m^ was nil):     Engine injects synthetic Signal; no Message consumed
After on error:                m^ unchanged; caller still owns if was not nil
```

---

### `api_message` — L5

Note: This module maps to the top-level `message/` package. See S3_fixed for details.

**Owns:** Nothing. Stateless accessor layer.

**Can Access:**
- Caller-provided `*Message` in `owned-app` state
- `bh` (BinaryHeader) field: all sub-fields (channel_number, proto, status, message_id, thl, bl)
- `thdrs` (TextHeaders) Appendable buffer operations (reads and writes)
- `body` (Appendable) buffer operations (reads and writes)

**Allowed Dependencies (downward L3):** `message_pool` (for `engine_get` and `engine_put` wrappers).

**Forbidden Dependencies:** `reserved_pool` (application must not access internal notification pool), any L2 module, any L4 module, `channel_group`, `engine` (no cross-L5 dependency).

**Forbidden Actions:**
- Must not access any Message in `owned-engine` state — application thread may only access `owned-app` Messages (INV-07)
- Must not cache raw pointers to Appendable buffer contents across `put()` or `post()` — buffer may be reset or freed (INV-08, INV-09, L-B1)
- `msg_body_slice` returns a read-only view valid only while Message is `owned-app`; must not retain slice past `put` or `post`
- `engine_put(&m)` sets `m^ = nil` unconditionally — caller must not use the Message after this call (INV-01 for put case)
- `engine_get(strategy=poolOnly)` may return `m^ == nil` (pool empty) — caller must check before use (INV-04)
- Must not call `reserved_pool.reserved_get` — engine_get exposes the application pool only
- Must not set `bh.proto.origin = .engine` — engine-origin Messages are for internal engine use only
- Must not write to `bh.<thl>` or `bh.<bl>` directly — framer sets these fields during encoding

**Appendable buffer address stability:**
`msg_write_thdr` and `msg_write_body` may cause `Appendable` reallocation. After any write, the internal buffer address may have changed. Callers who held a slice from a prior `msg_body_slice` call must re-call `msg_body_slice` to get the current slice. Never cache `[]u8` pointers across write calls.

---

## Forbidden Dependency Quick Reference

Key violations listed explicitly. These are the most likely implementation mistakes.

| If module... | ...tries to call... | Violation |
|-------------|--------------------|-----------|
| `protocol` | `message_pool.get` | L4 must use `reserved_pool` only for response Messages |
| `handshake` | `timeout_manager.set` | L4 cannot call L2; `io_dispatch` sets timeouts after handshake returns |
| `channel_group` | `channel_manager.*` | L5 cannot call L2; all Channel operations go through Reactor inbox |
| `channel_group` | `dual_map.*` | L5 cannot call L2 |
| `reactor` | `socket.*` directly in event loop | Socket I/O goes through `io_dispatch` |
| `io_dispatch` | `tc_pool.put` | TC pool return is Phase 8 (`reactor`), not Phase 7 |
| `io_dispatch` | `poller.*` | Poller re-arm is `reactor`'s responsibility after Phase 7 |
| `channel_manager` | `timeout_manager.*` | channel_manager has zero dependencies; `io_dispatch` manages timeouts |
| `mailbox_router` | any L2 module | L3 cannot call L2 |
| `framer` | `protocol.*` | framer is content-blind; protocol decisions are forbidden |
| `api_message` | `reserved_pool.*` | Application pool only; reserved pool is engine-internal |
| any module | `context.allocator` (implicit) | All allocations must name allocator explicitly (DR-3, MR-1) |
| application thread | any L2 module | All L2 is Reactor-thread-only |
| Reactor thread | `mbox_wait_receive` | Reactor never blocks on Mailbox (V-R1, AI-3) |
| Reactor thread | `pool_get_wait` | Reactor never blocks on Pool (V-R2) |

---

## Shared Ownership Violations

No two modules may own the same runtime instance. Confirmed violations would be architectural defects.

| What | Owner | Non-owning accessors |
|------|-------|---------------------|
| `Socket` struct + fd | `socket` module / `reactor` (via Socket instances) | `channel_manager` (reference only), `io_dispatch` (reference only) |
| `TriggeredChannel` struct | `tc_pool` (while pooled), `reactor` (while active) | `dual_map` (pointer only), `channel_manager` (pointer only), `io_dispatch` (pointer only) |
| `Channel` struct | `channel_manager` | `io_dispatch` (reference only), `protocol` (reference only), `handshake` (reference only) |
| `Message` struct | `message_pool` (while pooled), application (while owned-app), Reactor (while owned-engine), Mailbox (while queued) | No shared ownership — exactly one owner at all times (AI-14) |
| Reactor Inbox `Mailbox` | `mailbox_router` | Application threads (call `mbox_send` — thread-safe API; not ownership sharing) |
| ChannelGroup outbox `Mailbox` | Engine (via ChannelGroup struct) | `mailbox_router` (routing reference only), Reactor (`mbox_send` only), Application (`mbox_wait_receive` only) |
| `Poller` handle | `reactor` | No other module |
| `Notifier` FDs | `notifier` | `reactor` (holds reference to notifier; calls drain) |
| `dual_map` HashMaps | `dual_map` | `reactor` (owns dual_map instance) |
| ChannelGroup struct | Engine (`engine` module) | Application (non-owning handle — V-CG1) |

**Ownership is exclusive at all times.** The Matryoshka `MayItem` convention enforces this for Messages at every transfer point. For all other objects, the exclusivity is enforced by the thread access map and the module boundary rules above.

---

## No Implicit Access

The following accesses are explicitly forbidden because they would bypass module boundaries:

1. **No global state.** No module stores state in package-level variables accessible to other modules. All shared state flows through explicit procedure parameters.

2. **No context.allocator.** Inside any otofu module, no allocation uses `context.allocator` implicitly. Every allocation names its allocator explicitly (DR-3, MR-1). Modules receive the allocator as a parameter or as part of a context struct.

3. **No raw pointer casts to access another module's owned struct.** For example, a `*PolyNode` cast to `*TriggeredChannel` is only legal inside `tc_pool` (where TriggeredChannels are owned) and inside `io_dispatch` (which receives the pointer from `dual_map` after `tc_pool` allocated it). A cast to `*Message` is only legal inside `message_pool` and `reserved_pool` hooks, and inside `api_message` / `message` package (with owned-app Message).

4. **No passing module-internal structs by pointer across module boundaries without ownership transfer.** A `*Channel` passed to `io_dispatch` is a non-owning reference — `io_dispatch` must not free it, store it beyond the call, or share the pointer with another module not listed in its allowed access set.

5. **No Reactor thread entering application-thread-only procedures.** `cg_post`, `cg_wait_receive`, `cg_update_receiver`, `engine_get`, `engine_put`, `engine_create`, `engine_destroy` are application-thread procedures. The Reactor must never call them.

6. **No application thread entering Reactor-only procedures.** All L2 procedures, `poller_*`, `socket_*`, `reserved_get`, `protocol_*`, `handshake_*`, `framer_*` are Reactor-thread procedures. Application threads must never call them.
