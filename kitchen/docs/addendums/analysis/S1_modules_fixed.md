# S1 — Module Catalog (Fixed)

Sources: P7_otofu_architecture.md, P3_ownership.md
Pipeline position: S1 (first structured design step — extracted from P7 architecture)

## Corrections Applied

| Rule | Violation in original | Fix |
|------|-----------------------|-----|
| RULE-1 | Framer wire format described as "opcode(1B), channel_num(2B), message_id(4B), flags(1B), meta_len(4B), body_len(4B)" — does not match actual BinaryHeader | Wire format corrected to match tofu BinaryHeader exactly |
| RULE-1 | `message_pool` module described Message fields as `header: Header` and `meta: Appendable` — wrong struct shape | Corrected to `bh: BinaryHeader` and `thdrs: TextHeaders` |
| RULE-1 | `protocol` OpCode table described WelcomeRequest/WelcomeResponse as internal control commands | Corrected: both are wire protocol messages |
| RULE-2 | `channel_group` module missing `updateReceiver` operation | Added `cg_update_receiver` to operations table |
| RULE-2 | `engine` module missing `getAllocator` | Added `engine_get_allocator` |
| RULE-2 | `api_message` module missing BinaryHeader field accessors (status, origin, more) | Added missing accessors; renamed meta→thdrs |
| Structural | `api_message` (L5) placed in `otofu/` root package | Corrected: `api_message` maps to top-level `message/` package — 6 files (see S3_fixed, S4_fixed) |
| RULE-6/7 | `address` treated as passive TCP-only, client/server collapsed | Four active variants (TCP_Client, TCP_Server, UDS_Client, UDS_Server); format/parse; header integration in `message/address_headers.odin` |
| RULE-8 | `IntrusiveQueue` / `MessageQueue` absent from module catalog | `api_message` module now includes `Message_Queue` (single-threaded FIFO, `message/queue.odin`) as part of the public message API |
| Structural | `engine_get` / `engine_put` listed in `api_message` operations table | Moved to `engine` module public procedures — these take an `Engine` handle; placing them in `message/` would create cycle `otofu → message → otofu` |

---

This catalog defines every module in otofu. Each entry states one responsibility, its owned state, and its direct dependencies. No module has mixed responsibilities. No vague names.

---

## Summary Table

| # | Module | Layer | Status | Responsibility (one line) |
|---|--------|-------|--------|--------------------------|
| 1 | `matryoshka.poly` | L0 | External | Intrusive list node and ownership token |
| 2 | `matryoshka.mailbox` | L0 | External | MPMC blocking queue with ownership transfer |
| 3 | `matryoshka.pool` | L0 | External | Item recycling with lifecycle hooks |
| 4 | `poller` | L1 | otofu | Wrap OS I/O multiplexer; register/deregister FDs; collect readiness events |
| 5 | `notifier` | L1 | otofu | Socket-pair cross-thread wake; write side wakes, read side drains |
| 6 | `socket` | L1 | otofu | Non-blocking socket lifecycle; map OS errors to Engine_Error; TCP and UDS |
| 7 | `reactor` | L2 | otofu | Execute the 9-phase event loop; coordinate all L2 submodules |
| 8 | `dual_map` | L2 | otofu | Maintain ChannelNumber ↔ SequenceNumber ↔ *TriggeredChannel indirection |
| 9 | `tc_pool` | L2 | otofu | Allocate and recycle TriggeredChannel structs with pointer stability |
| 10 | `channel_manager` | L2 | otofu | Manage Channel list, state machine, outbound queues, ChannelNumber assignment |
| 11 | `io_dispatch` | L2 | otofu | Handle I/O events per TriggeredChannel (READ/WRITE/ERROR/HUP/ACCEPT) |
| 12 | `timeout_manager` | L2 | otofu | Track per-Channel deadlines; compute Poller.wait timeout; emit expired Channels |
| 13 | `message_pool` | L3 | otofu | Application Message Pool with allocation hooks |
| 14 | `reserved_pool` | L3 | otofu | Engine-internal notification Message Pool; fixed pre-allocated count |
| 15 | `mailbox_router` | L3 | otofu | Wire Reactor inbox and ChannelGroup outboxes; route ownership transfers |
| 16 | `framer` | L3 | otofu | Wire-level encode/decode of Message byte stream (BinaryHeader + TextHeaders + Body) |
| 17 | `protocol` | L4 | otofu | OpCode enum definition; exhaustive OpCode dispatch; conversation-layer state directives |
| 18 | `handshake` | L4 | otofu | Hello/Bye/Welcome sequence execution; simultaneous-Bye tiebreaker |
| 19 | `engine` | L5 | otofu | Engine lifecycle (create/destroy); Engine_Options; Engine_Error enum; getAllocator |
| 20 | `channel_group` | L5 | otofu | ChannelGroup lifecycle; post(); waitReceive(); updateReceiver(); channel connect/listen commands |
| 21 | `api_message` | L5 | otofu | Message construction and field access; full BinaryHeader + TextHeaders + Body API; Message_Queue (single-threaded FIFO); address-header integration |

Total: 21 modules (3 external, 18 otofu).

Note: `api_message` (module 21) maps to the top-level `message/` package (6 files: `binary_header.odin`, `text_headers.odin`, `message.odin`, `queue.odin`, `address_headers.odin`, `helpers.odin`). Not in `otofu/`. See S3_fixed and S4_fixed.

---

## Layer Dependency Rules

```
L5 → L3              (no direct L1 or L2 access from application layer)
L4 → L3, L2          (protocol reads/writes Messages; issues state directives to Reactor)
L3 → L0              (Pools and Mailboxes are Matryoshka primitives)
L2 → L1, L0, L3, L4  (Reactor uses OS, Matryoshka, Messaging, Protocol)
L1 → OS              (raw syscalls only)
L0 → —               (external; no otofu dependency)
```

Violations of these rules are defects. No upward calls. No lateral calls across layer siblings.

---

## L0 — Matryoshka (External)

These modules are provided by the Matryoshka library. otofu does not own, modify, or wrap them.

---

### Module: `matryoshka.poly`

**Responsibility:** Provide the intrusive list node (`PolyNode`) and the ownership token (`MayItem`). Every struct that travels through Matryoshka infrastructure embeds `PolyNode` at offset 0. `MayItem` (`Maybe(^PolyNode)`) signals ownership at every call site: `m^ != nil` = you own it; `m^ == nil` = you do not.

**Owned Entities:** None. PolyNode is embedded in the caller's struct. MayItem is a value type held by the caller.

**Dependencies:** None.

**Constraints:**
- `PolyNode` must be the first field in every otofu traveling struct (C1 from P4)
- `PolyNode.id` must be non-zero (C2 from P4)
- A struct can be in exactly one intrusive list at a time (C3 from P4)

---

### Module: `matryoshka.mailbox`

**Responsibility:** Provide the MPMC blocking queue. `mbox_send` transfers ownership into the queue (sender's MayItem → nil). `mbox_wait_receive` transfers ownership out (receiver's MayItem ← non-nil). `try_receive_batch` non-blocking drain. `mbox_close` drains and signals all receivers. `mbox_interrupt` available but not used by otofu (DR-5: incompatible with Poller-based blocking).

**Owned Entities:** Mailbox struct (allocated by caller via explicit allocator); linked list of queued PolyNode items.

**Dependencies:** `matryoshka.poly`.

---

### Module: `matryoshka.pool`

**Responsibility:** Provide the reusable item store. `pool_get` returns an item via caller-supplied `on_get` hook (allocate if needed, or reset existing). `pool_put` returns an item via `on_put` hook (reset for reuse, or free if over capacity). `pool_close` drains all stored items as a list for caller-managed disposal.

**Owned Entities:** Pool struct; linked free-list of pooled PolyNode items.

**Dependencies:** `matryoshka.poly`.

---

## L1 — OS / Poller

Platform-selected by `when ODIN_OS`. One concrete implementation per platform. No runtime dispatch. No otofu-level types (Channels, Messages, SequenceNumbers) visible at this layer.

---

### Module: `poller`

**Responsibility:** Wrap the OS I/O multiplexer (epoll on Linux, kqueue on macOS/BSD, wepoll/AFD_POLL on Windows). Register file descriptors with interest flags and an embedded SequenceNumber. Deregister file descriptors. Block on `poller_wait` until events fire or timeout expires. Return a batch of `(seqn: u64, flags: TriggerFlags)` pairs.

**Owned Entities:**
- OS multiplexer handle: `epoll_fd` / `kqueue_fd` / `wepoll_handle`
- Per-platform registration metadata (internal; not exposed)

**Dependencies:** OS syscall layer only. No other otofu modules.

**Constraints:**
- Edge-triggered mode always active (EPOLLET / EV_CLEAR)
- SequenceNumber embedded in event data field (`epoll_data.u64` / `kevent.udata`) at registration time; returned in event batch
- `poller_wait` is the only blocking call permitted in the Reactor thread
- `poller_deregister` must be called before `socket_close(fd)` — the caller (Reactor) is responsible for this ordering

---

### Module: `notifier`

**Responsibility:** Provide the cross-thread wake mechanism for the Reactor. Application thread calls `notifier_notify` (writes 1 byte to write_fd). Reactor's Poller fires on read_fd. Reactor calls `notifier_drain` to clear pending bytes. The Notifier is registered with Poller like any other FD, but it is not a TriggeredChannel — it has no SequenceNumber and is not in the dual_map.

**Owned Entities:**
- `read_fd`: registered with Poller; Reactor reads from this
- `write_fd`: application writes to this; owned by Notifier; not exposed to application directly

**Dependencies:** OS socket API.

**Constraints:**
- `notifier_notify` tolerates EAGAIN silently (pipe buffer full = wake already pending; idempotent)
- `mbox_interrupt` must NOT substitute for `notifier_notify` (DR-5: Reactor blocks on Poller, not on mbox_wait_receive)
- `notifier_notify` must be called AFTER `mbox_send` to reactor inbox (ordering invariant AI-7)

---

### Module: `socket`

**Responsibility:** Non-blocking socket lifecycle for TCP and UDS (Unix Domain Sockets): create, connect, listen, accept, send, recv, set options, close. Map all OS error codes to `Engine_Error`. No OS error code escapes this module.

**Owned Entities:**
- `Socket` struct per instance: OS file descriptor (`fd`) + state field + socket type (TCP or UDS)

**Dependencies:** OS socket API; `Engine_Error` type.

**Key operations and their ownership impact:**
| Operation | Ownership |
|-----------|-----------|
| `socket_create` | Caller owns returned Socket struct |
| `socket_accept` | Caller owns returned new Socket struct |
| `socket_close` | Frees Socket struct and fd; caller must not use afterward |
| `socket_send / socket_recv` | No ownership change; operates on caller-owned Socket and caller-provided buffers |

**Constraints:**
- All sockets set non-blocking immediately after creation and after `accept`
- `socket_connect_complete` (getsockopt SO_ERROR) called after WRITE readiness event, not during `socket_connect`
- `SO_LINGER=0` applied before `socket_close` on all IO sockets (all platforms)
- If `accept` returns a new fd when Listener is already closing: caller must immediately close the new fd — it must never be stored
- UDS: socket path cleanup on close is caller responsibility (Reactor, Phase 8)

---

## L2 — Reactor Core

All L2 modules execute on the Reactor thread exclusively. No L2 function may be called from an application thread. No L2 function may block on a Mailbox or Pool.

---

### Module: `reactor`

**Responsibility:** Run the 9-phase event loop. Coordinate the execution order of all L2 submodules. Manage per-iteration scratch state. Signal Engine state transitions (`starting → running`, `draining → destroyed`). This module is the Reactor thread's entry point and main loop.

**Owned Entities:**
- `Poller` instance (created at startup; closed at drain)
- `Notifier` instance (read_fd registered with Poller)
- `Dual_Map` instance
- `TC_Pool` instance (TriggeredChannel pool)
- `Channel_Manager` instance
- `Timeout_Manager` instance
- Per-iteration scratch: `io_events[]`, `resolved[]`, `pending_close[]`, `inbox_pending` flag

**Dependencies:** `poller`, `notifier`, `dual_map`, `tc_pool`, `channel_manager`, `io_dispatch`, `timeout_manager`, `mailbox_router` (for inbox drain), `reserved_pool` (for notification Messages), `protocol` (for inbox command dispatch).

**Phase sequence (summary):**
1. Compute timeout — `timeout_manager`
2. Poll — `poller`
3. Classify events — internal
4. Check timeouts — `timeout_manager`
5. Drain inbox — `notifier`, `mailbox_router`, `protocol`
6. Resolve I/O events — `dual_map`
7. Dispatch I/O — `io_dispatch`
8. Process pending closes — `poller`, `dual_map`, `socket`, `tc_pool`, `mailbox_router`, `channel_manager`
9. Drain check — exit or loop

**Constraint:** All `dual_map` INSERTs occur in Phase 5. All `dual_map` REMOVEs occur in Phase 8. No mutation in Phases 6 or 7 (AI-1).

---

### Module: `dual_map`

**Responsibility:** Maintain the two-level indirection between Channel identity and Poller events. Provide O(1) event-to-TriggeredChannel resolution (Phase 6). Provide O(1) removal by SequenceNumber or ChannelNumber (Phase 8). This is the ABA guard: if an event's SequenceNumber is absent from the map, the event is stale and is discarded.

**Owned Entities:**
- `chan_to_seqn: HashMap(ChannelNumber, SequenceNumber)` — used during close (find seqn from channel)
- `seqn_to_tc: HashMap(SequenceNumber, *TriggeredChannel)` — used during event resolution (Phase 6)

**Dependencies:** None (pure data structure; operates on externally-provided types).

**Operations:**
| Operation | Caller phase | Purpose |
|-----------|-------------|---------|
| `dual_map_insert(chan_num, seqn, tc_ptr)` | Phase 5, Phase 7 (accept) | New channel opened or accepted |
| `dual_map_lookup_seqn(seqn)` → `*TC` or nil | Phase 6 | Resolve event to TriggeredChannel (nil = ABA discard) |
| `dual_map_lookup_chan(chan_num)` → seqn or 0 | Phase 8 | Find seqn to remove during close |
| `dual_map_remove_seqn(seqn)` | Phase 8 | Deregistration step 2 |
| `dual_map_remove_chan(chan_num)` | Phase 8 | Deregistration step 3 |

**Constraint:** This module has no lock. It is exclusively accessed by the Reactor thread.

---

### Module: `tc_pool`

**Responsibility:** Allocate and recycle `TriggeredChannel` structs using a Matryoshka Pool. Guarantee pointer stability: every `TriggeredChannel` lives at a fixed heap address from `pool_get` until `pool_put`. The Poller and dual_map hold `*TriggeredChannel` pointers that must remain valid.

**Owned Entities:**
- One Matryoshka `Pool` instance (manages TriggeredChannel free-list)
- All `TriggeredChannel` instances while pooled (linked via PolyNode)
- `TriggeredChannel` instances while in active use are referenced (not owned) by dual_map

**`TriggeredChannel` struct fields:**
- `poly: PolyNode` at offset 0 (Matryoshka requirement)
- `seq: u64` — SequenceNumber; ABA token
- `channel_num: u16`
- `trigger_flags: TriggerFlags`
- `state: TC_State` — SM5
- Platform I/O state (e.g., IO_STATUS_BLOCK on Windows)

**Dependencies:** `matryoshka.pool`, `matryoshka.poly`; explicit allocator (passed via on_get context).

**Hook: on_get:**
- `m^ == nil`: allocate new TriggeredChannel from explicit allocator; set `poly.id = TriggeredChannelId`
- `m^ != nil`: zero `seq`, `channel_num`, `trigger_flags`, platform I/O state

**Hook: on_put:**
- Always store (no capacity limit; bounded by `options.max_channels`); leave `m^ != nil`

**Constraint (V-TC1):** `pool_put` must not be called while the TriggeredChannel is registered in the Poller or present in `dual_map`. Enforced by Phase 8 ordering: deregister → remove from dual_map → close(fd) → pool_put.

---

### Module: `channel_manager`

**Responsibility:** Maintain all Channel structs across all ChannelGroups. Execute SM2 state machine transitions. Assign and release ChannelNumbers. Manage per-Channel outbound send queues. Provide the partial receive buffer per Channel for TCP/UDS stream framing.

**Owned Entities (per Channel):**
- `Channel` struct: state (SM2), type (Listener / IO_Client / IO_Server), number (u16), socket reference (non-owning), triggered_channel reference (non-owning), channel_group reference (non-owning)
- Per-Channel outbound queue: bounded FIFO of `*Message` pointers (depth limit = `options.outbound_queue_depth`)
- Per-Channel recv_buf: partial receive state for TCP/UDS stream framing cursor
- Per-Channel deadlines (stored here or delegated to timeout_manager — single source of truth is here)
- ChannelNumber allocation pool: bitmap of u16 values 1–65534

**Dependencies:** None directly. State machine transitions emit directives that the Reactor executes. Channel structs reference `*Socket` and `*TriggeredChannel` but do not own them.

**Operations:**
| Operation | Effect |
|-----------|--------|
| `ch_mgr_allocate(cg_ref, type, allocator)` | Allocates Channel struct; state = `unassigned` |
| `ch_mgr_assign_number(ch)` | Assigns ChannelNumber from bitmap; state → `opened` |
| `ch_mgr_transition(ch, new_state)` | Validates and executes SM2 transition; panics on illegal in debug |
| `ch_mgr_enqueue_outbound(ch, &m)` | Appends to outbound queue; returns `BackpressureExceeded` if full |
| `ch_mgr_dequeue_outbound(ch)` | Pops front of outbound queue; nil if empty |
| `ch_mgr_release_number(ch)` | Returns ChannelNumber to bitmap; called in Phase 8 Step 7 |
| `ch_mgr_free(ch, allocator)` | Frees Channel struct; must be in `closed` state |

**Constraint:** All Channel state reads and writes occur on the Reactor thread exclusively. Application thread must not access Channel state directly (INV-13).

---

### Module: `io_dispatch`

**Responsibility:** Execute the I/O event handler for one resolved TriggeredChannel per Phase 7 call. Delegate to `socket` (recv/send/accept), `framer` (decode/encode), `protocol` (message interpretation), `channel_manager` (state transitions and queue operations), `mailbox_router` (Message delivery to application), and `reserved_pool` (notification Messages). Populate `pending_close` for Channels that must close this iteration.

**Owned Entities:** None. Stateless per invocation. Operates on caller-provided objects via `Dispatch_Context`.

**`Dispatch_Context` fields (all non-owning references):**
- `allocator: mem.Allocator`
- `reserved_pool: *Reserved_Pool`
- `app_pool: *Message_Pool`
- `mailbox_router: *Mailbox_Router`
- `channel_manager: *Channel_Manager`
- `dual_map: *Dual_Map`
- `pending_close: *[dynamic]*Channel`

**Dependencies:** `socket`, `framer`, `protocol`, `channel_manager`, `mailbox_router`, `reserved_pool`, `dual_map`.

**READ path summary:**
```
recv into ch.recv_buf
framer.try_decode → Incomplete (re-arm READ) | Error (pending_close) | Complete
protocol.handle_inbound(decoded_msg, ch, ctx) → state directive + optional response
```

**WRITE path summary:**
```
while ch.outbound_queue not empty:
  send encoded bytes
  on WouldBlock: requeue front; re-arm WRITE; break
  on complete: pool_put(m)
if queue empty: re-arm READ-only (disarm WRITE)
```

**ACCEPT path (Listener READ):**
```
socket_accept → new_fd
channel_manager.allocate(new channel)
tc_pool.get → new TriggeredChannel
dual_map.insert(new_chan_num, new_seqn, new_tc)
poller.register(new_fd, new_seqn, READ|WRITE|ET)
```

**Constraint:** New Channels inserted via accept are NOT in `resolved[]` for the current iteration. They become eligible for dispatch on the next iteration (AI-2 preserved).

---

### Module: `timeout_manager`

**Responsibility:** Track per-Channel deadlines (connect, handshake, bye timeouts). Compute the minimum timeout value for `poller_wait`. Identify Channels whose deadlines have expired each iteration.

**Owned Entities:**
- Deadline table: per-Channel, per-deadline-kind entries (monotonic timestamp in ms)

**Dependencies:** None (pure data; operates on Channel references which are externally owned).

**Operations:**
| Operation | Effect |
|-----------|--------|
| `timeout_set(ch, kind, deadline_ms)` | Register or update deadline |
| `timeout_clear(ch, kind)` | Remove deadline (Channel moved to next state) |
| `timeout_next_ms(now_ms)` | Return min(all deadlines) − now; 0 if any expired |
| `timeout_collect_expired(now_ms)` → `[]*Channel` | Return all Channels past their deadline |

**Constraint:** Deadline records reference Channels by pointer. Channel lifetimes always exceed their timeout records: `timeout_clear` is always called before `channel_manager.free`.

---

## L3 — Messaging Runtime

L3 modules are the only cross-thread access points in the system. Each module has distinct access patterns per thread.

---

### Module: `message_pool`

**Responsibility:** Manage the application Message lifecycle via a Matryoshka Pool. Provide `get(strategy)` and `put(&m)`. Execute `on_get` (initialize/reset Message fields) and `on_put` (reset buffers, enforce capacity limits, discard if over pool cap).

**Owned Entities:**
- One Matryoshka `Pool` instance
- All `Message` instances while pooled (linked via PolyNode in free-list)
- Indirectly: all `Appendable` backing buffers inside pooled Messages (capacity retained; content cleared)

**`Message` struct fields (RULE-1 — preserved from tofu wire format):**
- `poly: PolyNode` at offset 0 (Matryoshka C1)
- `bh: BinaryHeader` — 16-byte fixed, big-endian wire format:
  - `channel_number: u16` — channel correlation
  - `proto: ProtoFields` — packed u8: opCode(u4), origin(u1), more(u1), _internalA(u1), _internalB(u1)
  - `status: u8` — operation status code
  - `message_id: u64` — request-response correlation
  - `<thl>: u16` — text headers length (engine-internal)
  - `<bl>: u16` — body length (engine-internal)
- `thdrs: TextHeaders` — HTTP-style key-value pairs, format: `"name: value\r\n"`
- `body: Appendable` — opaque byte payload

**Dependencies:** `matryoshka.pool`, `matryoshka.poly`; explicit allocator.

**Hook: on_get (explicit allocator from Dispatch_Context):**
- `m^ == nil`: allocate Message struct; set `poly.id = MessageId`
- `m^ != nil`: zero BinaryHeader; `thdrs.reset()` (clear content, retain capacity); `body.reset()` (same)

**Hook: on_put:**
- If `thdrs.capacity > options.max_appendable_capacity`: free thdrs buffer
- If `body.capacity > options.max_appendable_capacity`: free body buffer
- If pool count > `options.max_messages`: free Message struct; `m^ = nil` (pool discards)
- Otherwise: leave `m^ != nil` (pool stores)

**Constraints:**
- `on_get` and `on_put` must not call `pool_get` or `pool_put` on this pool (C4; reentrancy corruption)
- Application thread accesses this module via `engine_get` / `engine_put` (L5 wrappers)
- Reactor thread must NOT use this pool for engine-internal Messages — that is `reserved_pool`

---

### Module: `reserved_pool`

**Responsibility:** Provide a fixed, pre-allocated supply of Messages for engine-internal notifications (ByeResponse, ByeSignal to peers). This pool is never exposed to `engine_get`. It is the R6.3 deadlock mitigation: even if the application holds all application-pool Messages, the Reactor can still deliver protocol messages.

**Owned Entities:**
- One Matryoshka `Pool` instance
- Pre-allocated `options.reserved_messages` Message instances

**Dependencies:** `matryoshka.pool`, `matryoshka.poly`; explicit allocator.

**Operations:**
| Operation | Mode | Error if empty |
|-----------|------|----------------|
| `reserved_get(&m)` | `.Available_Only` only — never allocates | `Engine_Error.ReservedPoolExhausted` |
| `reserved_put(&m)` | Returns to this pool | — |

**Sizing rule:** `options.reserved_messages ≥ options.max_channels × 2`. Each Channel close may require up to 2 reserved Messages (ByeResponse to peer + ByeSignal notification). All Channels may close simultaneously during Engine drain.

**Ownership note:** Application receives reserved-pool Messages via outbox Mailbox and returns them via `engine_put`. `engine_put` routes them to `message_pool` (application pool), not back to `reserved_pool`. This is correct: the reserved pool is replenished only from its pre-allocated stock. The application cannot distinguish reserved from application Messages.

---

### Module: `mailbox_router`

**Responsibility:** Wire all Mailbox instances. Provide the single abstraction over cross-thread Message routing. Route application posts to the Reactor inbox. Route Reactor delivers to the correct ChannelGroup outbox. Provide the application receive path (waitReceive).

**Owned Entities:**
- `reactor_inbox: Mailbox` — one instance; all `cg_post()` calls land here
- References (non-owning) to per-ChannelGroup outbox Mailboxes — owned by ChannelGroup structs, managed by Engine

**Dependencies:** `matryoshka.mailbox`, `notifier` (for `notifier_notify` after send-to-reactor).

**Operations and thread ownership:**

| Operation | Caller thread | Mechanism | Ownership after |
|-----------|--------------|-----------|----------------|
| `router_send_reactor(&m)` | Application | `mbox_send(inbox, &m)` | Mailbox queue owns (m^ = nil) |
| `router_drain_inbox(batch)` | Reactor | `try_receive_batch(inbox)` | Reactor owns each item (m^ ≠ nil per item) |
| `router_send_app(cg_id, &m)` | Reactor | `mbox_send(outbox[cg_id], &m)` | Mailbox queue owns (m^ = nil) |
| `router_wait_app(cg_id, &m, timeout)` | Application | `mbox_wait_receive(outbox[cg_id], &m)` | Application owns (m^ ≠ nil on .Ok) |
| `router_drain_outbox(cg_id)` | Reactor (drain) | `mbox_close(outbox[cg_id])` | Caller disposes returned list |

**Ordering invariant (AI-7):** `router_send_reactor(&m)` must be called BEFORE `notifier_notify`. The L5 `cg_post` procedure enforces this sequence; `mailbox_router` does not enforce it internally.

**Constraint:** `router_wait_app` must be called by exactly one thread per ChannelGroup. Enforced by convention; `mailbox_router` does not enforce it (AI-10).

---

### Module: `framer`

**Responsibility:** Encode a Message into a byte stream for TCP/UDS send. Decode a TCP/UDS byte stream into Message fields. Handle partial reads (stream protocol; a Message may arrive across multiple recv calls). Handle partial writes (send may not flush all bytes in one call).

**Owned Entities:** None. Stateless per invocation. State for partial reads lives in the Channel's `recv_buf` (owned by `channel_manager`).

**Wire format (RULE-1 — exact tofu BinaryHeader, big-endian, packed):**

| Section | Size | Content |
|---------|------|---------|
| BinaryHeader | 16 bytes fixed | channel_number(u16), proto/ProtoFields(u8), status(u8), message_id(u64), `<thl>`(u16), `<bl>`(u16) |
| TextHeaders | `<thl>` bytes | HTTP-style key-value pairs: `"name: value\r\n"` |
| Body | `<bl>` bytes | Opaque bytes |

`#assert(size_of(BinaryHeader) == 16)` — compile-time wire format check (CC-4).

**Dependencies:** None (operates on caller-provided Message and byte buffers).

**Operations:**
| Operation | Input | Output |
|-----------|-------|--------|
| `framer_encode(m, out_buf)` | `*Message` (owned-engine), output buffer | Serialized bytes appended to out_buf |
| `framer_try_decode(in_buf, m)` | Byte slice, `*Message` to fill | `Decode_Result`: Incomplete / Complete(consumed) / Error |

**Constraint:** `framer` does not interpret OpCodes or drive state transitions. It only moves bytes to and from Message fields. Interpretation is `protocol`'s responsibility.

---

## L4 — Protocol Layer

L4 interprets Message content. It does not perform I/O. It does not own persistent state — Channel state lives in `channel_manager`.

---

### Module: `protocol`

**Responsibility:** Define the closed `OpCode` enum (10 values, matching tofu wire protocol exactly). Dispatch incoming Messages via exhaustive switch on OpCode. Issue state transition directives to `channel_manager`. Produce response Messages from `reserved_pool`.

**Owned Entities:** None. `OpCode` is a type definition. Dispatch logic is a pure function of incoming Message + Channel state.

**`OpCode` enum (closed — 10 values, RULE-3 — exact tofu wire protocol values):**

| OpCode | Wire role |
|--------|-----------|
| `Request` | Application data — bidirectional |
| `Response` | Application data — bidirectional |
| `Signal` | Application data — bidirectional |
| `HelloRequest` | Client → Server: initiate connection handshake |
| `HelloResponse` | Server → Client: handshake accepted |
| `WelcomeRequest` | Client → Server: post-hello completion (connection ready) |
| `WelcomeResponse` | Server → Client: channel open confirmation |
| `ByeRequest` | Either side: begin graceful close |
| `ByeResponse` | Responding side: close acknowledged |
| `ByeSignal` | Engine → peer: force close signal |

**Note on addressing:** Connect and listen addresses are embedded in TextHeaders of HelloRequest/WelcomeRequest messages (ConnectToHeader `~connect_to`, ListenOnHeader `~listen_on`), not in OpCodes. No additional "command" OpCodes exist.

**Dispatch rule (PM-3, DR-4):** All OpCode dispatch uses exhaustive `switch`, not `#partial switch`. Unknown OpCode → panic in debug; `Engine_Error.ProtocolError` in release.

**Dependencies:** `channel_manager` (state transitions), `reserved_pool` (response Message allocation), `mailbox_router` (deliver notifications to application), `handshake` (delegate Hello/Bye/Welcome sequences).

---

### Module: `handshake`

**Responsibility:** Execute the Hello/Bye/Welcome protocol sequences in detail. Handle the simultaneous-Bye tiebreaker (R2.1). Integrate handshake timeouts with `timeout_manager`. Manage the `handshaking` and `closing` sub-states of SM2.

**Owned Entities:** None. Sequence state lives in the Channel struct (owned by `channel_manager`). Handshake is a stateless executor invoked by `protocol`.

**Simultaneous-Bye tiebreaker (R2.1):**
Both peers send ByeRequest simultaneously. Both enter `closing`. Tiebreaker: the peer with the **lower ChannelNumber** (as exchanged in HelloRequest/HelloResponse Header) acts as responder and sends ByeResponse immediately. The peer with the higher ChannelNumber treats the incoming ByeRequest as its ByeResponse and proceeds to close.

**Dependencies:** `channel_manager` (state transitions, deadline setting), `reserved_pool` (HelloResponse / ByeResponse Messages), `timeout_manager` (set/clear handshake and bye timeouts), `mailbox_router` (deliver channel-open/close notifications to application).

---

## L5 — Public API

L5 is the only layer imported by application code. All functions are thread-safe from the caller's perspective. All functions return explicit error results. No silent ignore. No panic on usage errors.

---

### Module: `engine`

**Responsibility:** Engine lifecycle management. Allocate and initialize all subsystems in the correct order. Tear down all subsystems in reverse order on `engine_destroy`. Define `Engine_Options` (all tunable parameters). Define `Engine_Error` (closed error enum). Expose allocator via `engine_get_allocator` (RULE-2).

**Owned Entities:**
- `Engine` struct: the root aggregate of all otofu state. Contains: explicit allocator reference (non-owning), all L2/L3 module instances (or references to heap-allocated instances), Reactor thread handle, Engine state (`starting` / `running` / `draining` / `destroyed`).

**`Engine_Options` fields:**
| Field | Type | Meaning |
|-------|------|---------|
| `max_messages` | u32 | Application pool capacity |
| `reserved_messages` | u32 | Internal notification pool (≥ max_channels × 2) |
| `max_channels` | u32 | Max concurrent Channels per ChannelGroup |
| `outbound_queue_depth` | u32 | Per-Channel send queue limit |
| `max_appendable_capacity` | uint | Max retained TextHeaders/Body buffer per pooled Message |
| `connect_timeout_ms` | u32 | TCP/UDS connect deadline |
| `handshake_timeout_ms` | u32 | Hello/Welcome handshake deadline |
| `bye_timeout_ms` | u32 | Bye sequence deadline before forced close |

**`Engine_Error` enum (closed — 13 values):**
| Value | Meaning |
|-------|---------|
| `.None` | No error |
| `.WouldBlock` | Non-blocking I/O has no data (transient) |
| `.ConnectionRefused` | TCP/UDS ECONNREFUSED |
| `.ConnectionReset` | TCP/UDS ECONNRESET |
| `.TimedOut` | Connect / handshake / bye timeout |
| `.AddressInUse` | EADDRINUSE on bind |
| `.ProtocolError` | Malformed frame or invalid OpCode |
| `.BackpressureExceeded` | Channel outbound queue full |
| `.PoolEmpty` | get(poolOnly) with empty pool — caller must handle |
| `.EngineDraining` | post() rejected; Engine is draining |
| `.EngineDestroyed` | API call on destroyed Engine |
| `.ReservedPoolExhausted` | Capacity planning failure |
| `.InternalError` | Programming error (panic in debug) |

**Public procedures (RULE-2):**
| Procedure | Purpose |
|-----------|---------|
| `Engine_Create(opts, allocator)` | Allocate and start Engine |
| `Engine_Destroy(engine)` | Drain and free Engine |
| `Engine_Get_Allocator(engine)` | Return the allocator provided at creation |
| `Engine_Get(engine, strategy, &m)` | Acquire Message from application pool; m^ ≠ nil on ok; m^ = nil if pool empty + poolOnly |
| `Engine_Put(engine, &m)` | Return Message to application pool; m^ = nil unconditionally |

**Dependencies:** `message_pool`, `reserved_pool`, `mailbox_router`, `reactor` (spawns Reactor thread), `channel_manager`, `tc_pool`, `dual_map`, `timeout_manager`.

**Allocator ownership:**
- Caller owns the Allocator. Engine holds a non-owning reference.
- `engine_destroy` completes all deallocations before returning.
- After `engine_destroy` returns, the caller may safely free the Allocator.
- `engine_get_allocator` is valid only while Engine is in `running` or `draining` state.

**Initialization order (success-flag pattern, MR-3):**
```
ok := false; defer if !ok { reverse-order cleanup }
1. Allocate Engine struct from allocator
2. Create message_pool (pre-populate with initial_count)
3. Create reserved_pool (pre-allocate options.reserved_messages)
4. Create tc_pool
5. Create reactor_inbox Mailbox
6. Initialize dual_map
7. Spawn Reactor thread
8. Wait for Reactor to signal running
ok = true
```

---

### Module: `channel_group`

**Responsibility:** Provide the application's cross-thread messaging handle. Wrap `mailbox_router.send_to_reactor` (for post) and `mailbox_router.wait_app` (for waitReceive). Provide `updateReceiver` for injecting messages into the receive queue from another thread (RULE-2). Encode channel control commands (connect/listen) into TextHeaders and post them via HelloRequest/WelcomeRequest Messages.

**Owned Entities:**
- The `ChannelGroup` struct is owned by Engine and managed by Engine via `channel_group` procedures. This module provides procedures that operate on ChannelGroup handles — it does not independently own the structs.
- The outbox `Mailbox` instance is owned by the ChannelGroup struct (allocated by Engine on `cg_create`; freed by Engine on `cg_destroy`).

**Dependencies:** `mailbox_router`, `notifier` (via `router_send_reactor` which calls `notifier_notify`), `reserved_pool` (for connect/listen control Messages), `engine` (for error type).

**Operations (RULE-2 — full tofu ChannelGroup API):**
| Procedure | Thread | Ownership effect |
|-----------|--------|-----------------|
| `cg_create(engine)` | Application | Engine allocates ChannelGroup + outbox Mailbox; returns non-owning handle |
| `cg_destroy(engine, cg)` | Application | Engine frees; application must have exited receive loop first |
| `cg_post(cg, &m)` | Application | Transfers Message ownership to Reactor inbox; m^ = nil on ok; m^ stays non-nil on error (TP-M2-fail) |
| `cg_wait_receive(cg, &m, timeout_ms)` | Application | Transfers Message ownership from outbox to application; m^ = non-nil on .Ok |
| `cg_update_receiver(cg, &m)` | Application (any thread) | Injects Message into CG outbox for waitReceive; m^ = nil on ok; can be called from any thread to wake or send info to the receiver |
| `cg_connect(cg, addr, &m)` | Application | Encodes HelloRequest with connect_to address in TextHeaders; calls cg_post |
| `cg_listen(cg, addr, &m)` | Application | Encodes WelcomeRequest with listen_on address in TextHeaders; calls cg_post |

**`cg_update_receiver` contract:**
```
If m^ is not nil: Engine sets BinaryHeader.proto.opCode = .Signal, origin = .engine;
                  m^ = nil after success; caller must not retain Message pointer.
If m^ is nil:     Engine creates a Signal notification from reserved_pool and injects it.
Returns error if Engine is draining/destroyed.
```

**Constraint (AI-10):** Exactly one application thread calls `cg_wait_receive` on any given ChannelGroup. This is enforced by convention, not by API. The ChannelGroup handle must be given to exactly one receiver thread.

**Constraint (V-CG1):** The ChannelGroup handle is a non-owning reference. Application must stop using all handles before calling `engine_destroy`.

---

### Module: `api_message`

**Responsibility:** Message construction and field access for application code. Provides: (1) type-safe accessors for all BinaryHeader fields (opcode, channel_number, status, message_id, origin, more), TextHeaders (HTTP-style key-value pairs), and Body; (2) `Message_Queue` — single-threaded intrusive FIFO for `*Message` (port of tofu's `IntrusiveQueue`); (3) address-TextHeader bridge (`Addr_To/From_Connect/Listen_Header`). All field operations require the Message to be in `owned-app` state.

Note: `api_message` maps to the top-level `message/` package (6 files) — not `otofu/`. See S3_fixed and S4_fixed.

**Owned Entities:** None. Stateless accessor layer. Operates on caller-owned Messages and caller-owned Message_Queues.

**Dependencies:** `types` (for OpCode, Address, ProtoFields).

**Operations (RULE-1 — full BinaryHeader access):**
| Procedure | Pre-condition | Post-condition |
|-----------|--------------|----------------|
| `msg_set_opcode(m, op)` | owned-app | bh.proto.opCode set |
| `msg_set_channel(m, num)` | owned-app | bh.channel_number set |
| `msg_set_message_id(m, id)` | owned-app | bh.message_id set |
| `msg_set_status(m, st)` | owned-app | bh.status set |
| `msg_set_more(m, flag)` | owned-app | bh.proto.more set |
| `msg_read_opcode(m)` | owned-app | Returns bh.proto.opCode |
| `msg_read_channel(m)` | owned-app | Returns bh.channel_number |
| `msg_read_message_id(m)` | owned-app | Returns bh.message_id |
| `msg_read_status(m)` | owned-app | Returns bh.status |
| `msg_read_origin(m)` | owned-app | Returns bh.proto.origin (application or engine) |
| `msg_read_more(m)` | owned-app | Returns bh.proto.more |
| `msg_write_thdr(m, name, val)` | owned-app | TextHeaders Appendable appended: "name: value\r\n" |
| `msg_read_thdr(m, name)` | owned-app | Iterates TextHeaders; returns value string or nil |
| `msg_thdr_iterator(m)` | owned-app | Returns TextHeaderIterator over bh.`<thl>` bytes |
| `msg_write_body(m, data)` | owned-app | Body Appendable appended; may realloc |
| `msg_body_slice(m)` | owned-app | Returns read-only []u8 view |

**Constraints:**
- After `engine_put(&m)` or `cg_post(&m)`, all buffer pointers (TextHeaders, Body) are invalid. Must not be accessed (L-M5, INV-08, INV-09).
- `msg_body_slice` returns a view that is valid only while the Message is in `owned-app` state. The caller must not retain this slice past `put` or `post`.
- `msg_write_thdr` / `msg_write_body` may trigger Appendable reallocation. The internal buffer address may change. Never cache the raw buffer pointer (L-B1).
- `bh.proto.origin`: application-set Messages always have origin = .application. Engine-generated Messages have origin = .engine. Application must not set origin = .engine.
- `bh.<thl>` and `bh.<bl>` are set by the framer, not by application code. `msg_write_thdr`/`msg_write_body` update the length tracking via Appendable; the framer serializes the final sizes.

---

## Split Analysis

Modules reviewed for mixed responsibility. No mandatory splits found. Justification:

| Module | Review | Verdict |
|--------|--------|---------|
| `channel_manager` | Manages Channel lifecycle + outbound queues + ChannelNumber pool. All are per-Channel state; tightly coupled. | Keep |
| `io_dispatch` | Handles READ / WRITE / ERROR / HUP / ACCEPT. All are event-type variants of the same responsibility: handle one Poller event. | Keep |
| `reactor` | Runs event loop + coordinates submodules. Coordinator pattern is one responsibility. Individual sub-actions are delegated. | Keep |
| `protocol` | Defines OpCode + dispatches. Dispatch IS the protocol definition. | Keep |
| `mailbox_router` | Routes App→Reactor and Reactor→App. Asymmetric directions but single responsibility: wire the cross-thread transfer. | Keep |
| `engine` | Lifecycle + Options + Error enum. Options and Error are type definitions, not behaviors. | Keep |
| `channel_group` | Lifecycle + post + waitReceive + updateReceiver + connect/listen. All are the same API surface for one ChannelGroup handle. | Keep |

---

## Ownership Summary

| Module | Owns (runtime instances) |
|--------|--------------------------|
| `matryoshka.poly` | Nothing (value/embed type) |
| `matryoshka.mailbox` | Mailbox struct + queued PolyNode items |
| `matryoshka.pool` | Pool struct + pooled PolyNode items |
| `poller` | OS multiplexer handle |
| `notifier` | read_fd + write_fd |
| `socket` | Socket struct + OS fd |
| `reactor` | Poller instance, Notifier instance, Dual_Map, TC_Pool, Channel_Manager, Timeout_Manager, per-iteration scratch |
| `dual_map` | chan_to_seqn HashMap + seqn_to_tc HashMap |
| `tc_pool` | Matryoshka Pool + all pooled TriggeredChannel instances |
| `channel_manager` | All Channel structs + outbound queues + recv_bufs + ChannelNumber bitmap |
| `io_dispatch` | Nothing (stateless) |
| `timeout_manager` | Deadline table per Channel |
| `message_pool` | Matryoshka Pool + all pooled Message instances + their Appendable buffers |
| `reserved_pool` | Matryoshka Pool + pre-allocated reserved Message instances |
| `mailbox_router` | reactor_inbox Mailbox instance; references to CG outboxes (owned by Engine) |
| `framer` | Nothing (stateless) |
| `protocol` | Nothing (type definitions + stateless dispatch) |
| `handshake` | Nothing (stateless; sequence state in Channel) |
| `engine` | Engine struct (root aggregate of all otofu state) |
| `channel_group` | Procedures only; ChannelGroup struct owned by Engine |
| `api_message` | Nothing (stateless accessor) |
