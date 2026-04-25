# S6 — Tofu → Otofu Component Mapping (Fixed)

Sources: TOFU_REPO_PATH, S4_file_structure_fixed.md (with S5 corrections applied)
Pipeline position: S6 (source mapping — each tofu component traced to its otofu target)

---

## Corrections Applied

The original S6_tofu_mapping.md contained wrong decisions in the following mapping entries:

| Entry | Wrong decision | Fix |
|-------|---------------|-----|
| `ampe.zig` | Status: **REPLACED** — "VTables → opaque handles" implied all API methods changed | Changed to **ADAPTED**: VTables replaced by direct procedures, but ALL API methods preserved including `getAllocator` and `updateReceiver` |
| `message.zig` | Status: **SPLIT** — "Header restructured; TextHeaders replaced; OpCodes renamed" | Changed to **ADAPTED**: wire format preserved exactly; only framing extracted and message/ promoted to top-level public package |
| `address.zig` | Notes: "TCP only (UDS removed)" | Fixed: UDS retained; both TCP_Address and UDS_Address in union; ConnectToHeader/ListenOnHeader preserved |
| `status.zig` mismatch 3 | "Status not embedded in message header" | Corrected: status byte IS in BinaryHeader wire format at offset 3; Engine_Error is additional API-level error, does not replace the wire field |
| Per-component mismatches for `ampe.zig` | Items 2 and 3: "getAllocator removed", "updateReceiver removed" | Removed — both procedures are preserved in S4_file_structure_fixed.md |
| Per-component mismatches for `message.zig` | Items 1–5: TextHeaders replaced, status removed, message_id changed, OpCodes restructured | Removed — all RULE-1/RULE-3 violations; the wire format is preserved |
| Per-component mismatches for `address.zig` | Item 1: "UDS removed" | Removed — UDS is retained (RULE-4) |
| "Features without otofu equivalent" table | UDS, TextHeaders, status byte, message_id listed as absent | Removed those rows — all are present in corrected otofu |

The per-component mismatch sections for `message.zig` and `address.zig` are rewritten. Incorrect mismatch items (which were actually RULE violations, not legitimate mismatches) have been replaced with the actual architectural decisions that differ from tofu.

A new `## Legitimate Mismatches` section is added at the end, consolidating all intentional architectural differences between tofu and otofu.

---

## Rules Reminder

```
RULE-1  Wire format is immutable: BinaryHeader layout, ProtoFields encoding, TextHeader format
        ("name: value\r\n"), and body framing must be preserved exactly.
RULE-2  Public API is immutable: Ampe.get / put / create / destroy / getAllocator;
        ChannelGroup.post / waitReceive / updateReceiver.
RULE-3  OpCode enum must match tofu exactly: 10 values
        (Request, Response, Signal, HelloRequest, HelloResponse,
         ByeRequest, ByeResponse, ByeSignal, WelcomeRequest, WelcomeResponse).
RULE-4  UDS (Unix Domain Sockets) must be supported alongside TCP.
RULE-5  Matryoshka API is otofu-internal; client code has no dependency on Matryoshka types.
```

---

## Status Legend

| Status | Meaning |
|--------|---------|
| **DIRECT** | 1:1 structural port. Same concept, same structure. Odin syntax only. |
| **SPLIT** | One tofu file maps to two or more otofu files (responsibility was separated). |
| **MERGED** | Multiple tofu files collapse into one otofu file. |
| **ADAPTED** | Same concept, different structure. Ported with architectural adjustments. |
| **REPLACED** | Concept retained but mechanism changed fundamentally. |
| **REMOVED** | Concept dropped. No otofu equivalent. |
| **NEW** | otofu addition. No tofu equivalent. Required by Matryoshka or architecture. |

---

## Master Mapping Table

| Tofu file | Lines | otofu target(s) | Status | Notes |
|-----------|-------|----------------|--------|-------|
| `tofu.zig` | 29 | `otofu/engine.odin`, `otofu/channel_group.odin` | **ADAPTED** | tofu re-exports via VTable; otofu uses direct procedure calls; same API surface |
| `ampe.zig` | 171 | `otofu/engine.odin`, `otofu/channel_group.odin` | **ADAPTED** | VTables replaced by direct procedures; ALL 5 Ampe + 3 ChannelGroup methods preserved |
| `message.zig` | 837 | `message/message.odin`, `runtime/framer.odin`, `types/opcodes.odin` | **ADAPTED** | Wire format preserved exactly; BinaryHeader/TextHeaders/OpCodes unchanged; framing extracted to framer.odin; Message promoted to top-level `message/` package |
| `address.zig` | 410 | `types/address.odin`, `message/address_headers.odin` | **ADAPTED** | All four variants preserved (TCP_Client, TCP_Server, UDS_Client, UDS_Server); active format/parse behavior preserved; ConnectToHeader/ListenOnHeader preserved; header integration split to message/ to avoid import cycle |
| `status.zig` | 149 | `types/errors.odin` | **ADAPTED** | 35 AmpeStatus → 13 Engine_Error for API errors; status byte in BinaryHeader preserved on wire |
| `ampe/Reactor.zig` | 1268 | `reactor/reactor.odin`, `reactor/io_dispatch.odin` | **SPLIT** | 9-phase loop made explicit; I/O dispatch extracted; mutexes removed |
| `ampe/MchnGroup.zig` | 226 | `otofu/channel_group.odin`, `runtime/router.odin` | **SPLIT** | Mailbox wiring extracted to Router; CG is now a thin opaque handle |
| `ampe/channels.zig` | 340 | `chanmgr/channel.odin`, `chanmgr/numbers.odin`, `reactor/dual_map.odin` | **SPLIT** | Mutex removed; maps moved to Reactor-exclusive Dual_Map; Channel gets SM2 |
| `ampe/Pool.zig` | 196 | `runtime/message_pool.odin`, `runtime/reserved_pool.odin` | **SPLIT** | One pool → two; alerter callback replaced by Reserved Pool mechanism |
| `ampe/Notifier.zig` | 348 | `platform/notifier.odin`, `platform/notifier_unix.odin`, `platform/notifier_windows.odin` | **ADAPTED** | Alert type removed; wake is byte-only; socketpair replaces loopback TCP |
| `ampe/SocketCreator.zig` | 183 | `reactor/io_dispatch.odin` | **REMOVED** | Factory merged into io_dispatch connect/listen command handlers |
| `ampe/triggeredSkts.zig` | 1057 | `reactor/tc_pool.odin`, `chanmgr/channel.odin`, `reactor/io_dispatch.odin` | **SPLIT** | Fat union decomposed: TC = Poller token only; Channel = state + queue; dispatch = io_dispatch |
| `ampe/IntrusiveQueue.zig` | 97 | `message/queue.odin` | **ADAPTED** | Single-threaded FIFO for *Message; Matryoshka has no equivalent; ported to Odin as Message_Queue; uses PolyNode's embedded list.Node for intrusive linking |
| `ampe/vtables.zig` | 30 | (none) | **REMOVED** | VTable polymorphism dropped; otofu uses direct procedure calls |
| `ampe/internal.zig` | 25 | `platform/` package | **REMOVED** | Build-tag–based platform selection replaces comptime re-export |
| `ampe/poller.zig` | 41 | `platform/poller.odin` | **ADAPTED** | Comptime Backend selector → build-tag file selection |
| `ampe/poller/common.zig` | 63 | `types/identifiers.odin`, `types/flags.odin` | **MERGED** | Shared types moved to `types` package |
| `ampe/poller/core.zig` | 198 | `reactor/dual_map.odin`, `platform/poller.odin` | **SPLIT** | Maps separated from OS backend (critical architecture change) |
| `ampe/poller/triggers.zig` | 120 | `types/flags.odin`, `reactor/io_dispatch.odin` | **SPLIT** | Trigger flag type → types; interest flag logic → io_dispatch |
| `ampe/poller/epoll_backend.zig` | 103 | `platform/poller_linux.odin` | **DIRECT** | epoll → epoll |
| `ampe/poller/kqueue_backend.zig` | 144 | `platform/poller_darwin.odin` | **DIRECT** | kqueue → kqueue |
| `ampe/poller/poll_backend.zig` | 92 | (none) | **REMOVED** | Generic poll fallback not carried to otofu |
| `ampe/poller/wepoll_backend.zig` | 130 | `platform/poller_windows.odin` | **ADAPTED** | Legacy wepoll → native AFD_POLL (architectural upgrade) |
| `ampe/os/linux/Skt.zig` | 308 | `platform/socket.odin`, `platform/socket_unix.odin` | **SPLIT** | Common struct in socket.odin; POSIX syscalls in socket_unix.odin |
| `ampe/os/windows/Skt.zig` | 200 | `platform/socket_windows.odin` | **DIRECT** | Windows socket ops → socket_windows.odin |
| `ampe/os/windows/poller.zig` | 256 | `platform/poller_windows.odin` | **DIRECT** | IOCP+AFD_POLL → poller_windows.odin |
| `ampe/os/windows/afd.zig` | 149 | `platform/poller_windows.odin` | **MERGED** | AFD structures merged into poller_windows.odin |
| `ampe/os/windows/ntdllx.zig` | 125 | `platform/poller_windows.odin` | **MERGED** | ntdll declarations merged into poller_windows.odin |
| `ampe/testHelpers.zig` | 119 | (test/ directory, not S4) | **REMOVED** | Test utilities are outside the production file tree |
| — | — | `message/message.odin` | **NEW** | Top-level public package for Message type; no tofu equivalent (tofu has message.zig in src/) |
| — | — | `runtime/router.odin` | **NEW** | No tofu equivalent; required by two-mailbox pattern and Matryoshka wiring |
| — | — | `reactor/timeout.odin` | **NEW** | No tofu equivalent; tofu has timeout in Reactor.zig inline |
| — | — | `protocol/protocol.odin` | **NEW** | Explicit protocol dispatch layer; logic was inline in tofu Reactor.zig |
| — | — | `protocol/handshake.odin` | **NEW** | Explicit handshake module; logic was inline in tofu Reactor.zig |
| — | — | `chanmgr/outbound.odin` | **NEW** | Per-channel outbound queue; implicit in tofu IoSkt.send_queue |
| — | — | `types/options.odin` | **NEW** | Engine_Options extracted to types (S5 fix for circular import) |

---

## Per-Component Mismatch Analysis

### `tofu.zig` + `ampe.zig` → `otofu/engine.odin` + `otofu/channel_group.odin`

**tofu design:**
```
Ampe :: struct {
    vtable: *const AmpeVTable,   // get, put, create, destroy, getAllocator
}
ChannelGroup :: struct {
    vtable: *const CHNLSVTable,  // post, waitReceive, updateReceiver
}
```

`Ampe` and `ChannelGroup` are runtime-polymorphic handles. The application calls `ampe.vtable.get(ampe, strategy)`. All internal implementations (Reactor, pool, mailbox) are hidden behind function pointers.

**otofu design:**
```odin
Engine :: distinct ^engine_internal    // opaque
CG     :: distinct ^cg_internal        // opaque
```

Direct procedure calls: `Engine_Get(e, strategy)`. No function pointer indirection.

**Mismatches:**

1. **VTable removed.** Odin's type system does not require runtime dispatch for single-implementation interfaces. Direct procedures are cleaner and compile-time verifiable. The API surface (all 5 Ampe methods, all 3 ChannelGroup methods) is preserved identically.

*Note: The original S6 listed `getAllocator` and `updateReceiver` as removed. This was incorrect — both are preserved in S4_file_structure_fixed.md as `Engine_Get_Allocator` and `CG_Update_Receiver`.*

---

### `message.zig` → `message/message.odin` + `runtime/framer.odin` + `types/opcodes.odin`

**tofu `BinaryHeader` (16 bytes, packed, big-endian) — PRESERVED:**
```
channel_number: u16    (big-endian)
proto: ProtoFields     // u8: opCode(u4), origin(u1), more(u1), _A(u1), _B(u1)
status: u8             // operation result code
message_id: u64        // request-response correlation
@"<thl>": u16          // text headers length
@"<bl>": u16           // body length
Total: 16 bytes
```

**otofu `BinaryHeader` (identical wire layout):**
```odin
BinaryHeader :: struct #packed {
    channel_number: u16be,
    proto:          ProtoFields,
    status:         u8,
    message_id:     u64be,
    thl:            u16be,
    bl:             u16be,
}
#assert(size_of(BinaryHeader) == 16)
```

**Wire format: IDENTICAL to tofu. No mismatches in wire encoding.**

**tofu `OpCode` (10 values) — PRESERVED:**
```
Request=0, Response=1, Signal=2,
HelloRequest=3, HelloResponse=4,
ByeRequest=5, ByeResponse=6, ByeSignal=7,
WelcomeRequest=8, WelcomeResponse=9
```

**otofu `OpCode` (identical):**
```odin
OpCode :: enum u8 {
    Request=0, Response=1, Signal=2,
    HelloRequest=3, HelloResponse=4,
    ByeRequest=5, ByeResponse=6, ByeSignal=7,
    WelcomeRequest=8, WelcomeResponse=9,
}
```

**TextHeaders format — PRESERVED:** Both tofu and otofu use `"name: value\r\n"` line format for text headers. `TextHeaders.buffer` is the accumulation buffer. Header procedures parse and construct the format identically.

**Architectural differences (not wire-format changes):**

1. **Framing extracted.** tofu encodes/decodes in `message.zig` methods (`toBytes`, `fromBytes`, `check_and_prepare`). otofu extracts this to `runtime/framer.odin` (`Framer_Encode`, `Framer_Try_Decode`). The `Message` struct itself has no serialization knowledge. The wire encoding is identical — only the code location changed.

2. **`Message` promoted to top-level `message/` package.** In otofu, `*Message` appears in public API signatures. Odin has no re-export mechanism. `Message` must live in a package that clients can import directly. The `message/` package is the top-level public home for `Message`, `BinaryHeader`, `TextHeaders`, and their helper procedures.

3. **`@"<void*>"` and `@"<ctx>"` fields removed.** tofu reserves two pointer fields in `Message` for application and engine use. otofu replaces this pattern with Matryoshka's `MayItem` ownership token embedded as `PolyNode` at offset 0. Client code does not see the `PolyNode` field — it accesses only `bh`, `thdrs`, `body`.

---

### `address.zig` → `types/address.odin` + `message/address_headers.odin`

**tofu types (all preserved):**
```
TCPClientAddress  — connect address + port
TCPServerAddress  — bind address + port
UDSClientAddress  — Unix socket path (client)
UDSServerAddress  — Unix socket path (server)
Address           — tagged union of the above
ConnectToHeader   — "~connect_to" (address in TextHeader)
ListenOnHeader    — "~listen_on"  (address in TextHeader)
format methods    — each variant serializes itself to TextHeader value string
```

**otofu types (RULE-6/7 — all four variants and active behavior preserved):**
```odin
// types/address.odin
TCP_Client_Address :: struct { host: string, port: u16, version: IP_Version }
TCP_Server_Address :: struct { host: string, port: u16, version: IP_Version }
UDS_Client_Address :: struct { path: string }
UDS_Server_Address :: struct { path: string }

Address :: union {
    TCP_Client_Address,
    TCP_Server_Address,
    UDS_Client_Address,
    UDS_Server_Address,
}

ConnectToHeader :: "~connect_to"
ListenOnHeader  :: "~listen_on"

// Active behavior: "tcp|host|port" or "uds|path" — matches tofu format exactly
Addr_Format   :: proc(addr: Address, buf: []u8) -> (string, bool)
Addr_Parse    :: proc(s: string) -> (Address, bool)
Addr_Is_Client :: proc(addr: Address) -> bool   // → connect()
Addr_Is_Server :: proc(addr: Address) -> bool   // → listen()/accept()

// message/address_headers.odin (imports types + uses message.TextHeaders)
Addr_To_Connect_Header   :: proc(addr: types.Address, msg: ^Message) -> bool
Addr_To_Listen_Header    :: proc(addr: types.Address, msg: ^Message) -> bool
Addr_From_Connect_Header :: proc(msg: ^Message) -> (types.Address, bool)
Addr_From_Listen_Header  :: proc(msg: ^Message) -> (types.Address, bool)
```

**Mismatches:**

1. **Header integration split to `message/` package.** In tofu, `address.zig` contains both the address types AND the format/parse methods that write into TextHeaders. In otofu, `types/address.odin` contains only address types and the string format/parse procedures (no message import). The TextHeader integration procedures (`Addr_To_Connect_Header`, `Addr_From_Connect_Header`, etc.) live in `message/address_headers.odin`. This is required to avoid an import cycle: `types` ← `message` ← `types/address` would be circular if `types/address` imported `message`.

*Note: The original S6 listed "UDS removed", "client/server variants collapsed", and "address embedded in headers removed" as mismatches. All three were wrong decisions, not legitimate mismatches. All are corrected: UDS is retained (RULE-4), all four variants are preserved (RULE-7), and ConnectToHeader/ListenOnHeader are preserved and travel on the wire in TextHeaders (RULE-1).*

---

### `ampe/IntrusiveQueue.zig` → `message/queue.odin`

**tofu `IntrusiveQueue(T)` (generic, 97 lines):**
```
IntrusiveQueue(T) — generic FIFO intrusive linked list
  Requirement: T must have prev: ?*T and next: ?*T fields
  Operations: enqueue, pushFront, dequeue, empty, count, move
  Usage: only instantiated as IntrusiveQueue(Message) → MessageQueue
  Defined in message.zig line 709: pub const MessageQueue = IntrusiveQueue(Message)
```

Single-threaded. Zero allocation for queue metadata. Used as per-channel outbound send queue and reactor-phase scratch collections.

**otofu `Message_Queue` (specific to Message, in `message/queue.odin`):**
```odin
Message_Queue :: struct { first: ^Message, last: ^Message }

MQ_Enqueue    — add to back (FIFO)
MQ_Push_Front — add to front (priority; for protocol control messages)
MQ_Dequeue    — remove from front; nil if empty
MQ_Empty, MQ_Count
MQ_Move       — transfer all from src to dst
MQ_Clear      — dequeue + Msg_Destroy all
```

**Mismatches:**

1. **Generic → Message-specific.** tofu's `IntrusiveQueue(T)` is generic (comptime type parameter). Odin has comptime generics but `Message_Queue` only ever holds `*Message` — there is no other type it needs to queue. Specializing to `Message_Queue` is equivalent to tofu's single instantiation and removes unnecessary generality.

2. **Intrusive link field changed.** tofu uses `Message.prev: ?*Message` and `Message.next: ?*Message` directly as link fields. otofu's `Message` has `PolyNode` at offset 0, which embeds `list.Node` (from `core:container/intrusive/list`) with `prev`/`next`. `MQ_*` procedures access `msg._node.node.prev` / `msg._node.node.next` as the intrusive links. Semantics identical; field path differs due to Matryoshka embedding.

3. **Placed in `message/` package (not a separate utility).** tofu has `IntrusiveQueue.zig` in `ampe/` and re-exports the instantiation from `message.zig`. In otofu, `message/queue.odin` is directly in the `message` package because `MessageQueue` is part of the public message API — it is used by consumers of the `message` package. Matryoshka Mailbox is NOT a substitute: it is concurrent (mutex-based) and semantically different from a single-threaded per-channel queue.

---

### `status.zig` → `types/errors.odin`

**tofu `AmpeStatus` (35 values, partial list):**
```
ok, pool_empty, pool_exhausted, peer_disconnected, peer_aborted,
request_error, response_error, send_error, recv_error, protocol_error,
timeout, busy, shutting_down, not_found, ...
```

**otofu `Engine_Error` (13 values):**
```odin
None, WouldBlock, ConnectionRefused, ConnectionReset, TimedOut,
AddressInUse, ProtocolError, BackpressureExceeded, PoolEmpty,
EngineDraining, EngineDestroyed, ReservedPoolExhausted, InternalError
```

**Mismatches:**

1. **Request/response error distinction removed.** tofu has `request_error` and `response_error` as separate codes. otofu uses `ProtocolError` for all wire-level errors. Application-level error semantics are encoded in message content, not in Engine_Error.

2. **`peer_disconnected` / `peer_aborted` collapsed to `ConnectionReset`.** tofu distinguishes clean disconnect from abortive close. otofu uses a single `ConnectionReset`.

3. **Wire status byte preserved; API error type added.** tofu puts `AmpeStatus` in `BinaryHeader.status`. otofu preserves the `status: u8` field in the wire `BinaryHeader` at offset 3. Additionally, otofu returns `Engine_Error` at API call sites for local errors (pool exhaustion, drain state, etc.). The two error domains are separate: wire status (peer-reported, in every frame) and Engine_Error (local, returned from API procedures).

   *Note: The original S6 stated "Status not embedded in message header." This was incorrect. The `status: u8` field is in BinaryHeader and preserved exactly as in tofu.*

4. **`AmpeError` error set removed.** tofu maintains a parallel Zig error set (`AmpeError`) with bidirectional mapping to `AmpeStatus`. otofu uses a single `Engine_Error` enum; no parallel error set.

---

### `ampe/Pool.zig` → `runtime/message_pool.odin` + `runtime/reserved_pool.odin`

**tofu `Pool`:**
```
initialMsgs, maxMsgs, currMsgs: usize
first: ?*Message            // head of free list
mutex: Mutex                // thread-safe
alerter: ?AlerterFn         // callback: pool has freed memory
```
Thread-safe. Single pool. Alerter wakes Reactor when a message is returned, allowing Reactor to re-enable recv interest.

**otofu:**
- `Message_Pool`: Matryoshka Pool wrapper; on_get/on_put hooks; no mutex (Matryoshka thread-safe).
- `Reserved_Pool`: Separate pre-allocated pool; Reactor-owned; never exposed to application.

**Mismatches:**

1. **Alerter callback removed.** tofu's alerter fires when `pool.put()` restores a message, signaling the Reactor that it can re-enable `recv` interest on sockets that were paused due to pool exhaustion. otofu replaces this mechanism entirely: backpressure is per-channel outbound queue depth (`BackpressureExceeded`), not pool-based. The Reactor never pauses recv for pool exhaustion.

2. **One pool → two pools.** tofu has one pool shared between application and engine. otofu has two: `Message_Pool` (application) and `Reserved_Pool` (Reactor-owned, pre-allocated). The Reserved Pool solves the R6.3 deadlock (P3/P4): if the application holds all pool messages, the Reactor can still deliver `Channel_Closed` notifications from the reserved pool.

3. **Mutex replaced by Matryoshka.** tofu's Pool has its own mutex. Matryoshka Pool is already thread-safe via its own internal mechanism.

4. **on_get/on_put hooks replace reset logic.** tofu's `pool.get()` calls `msg.reset()` after retrieval. otofu's `on_get` hook handles reset. This moves reset responsibility into the pool hook rather than the caller.

---

### `ampe/Notifier.zig` → `platform/notifier.odin` + platform backend files

**tofu `Notifier`:**
```
Alert :: enum { freedMemory, shutdownStarted }
send_alert(alert: Alert)   — sends typed signal over socket pair
isReadyToRecv() → bool     — check if notification pending
```
Sends typed `Alert` values. The Reactor reads the alert type and takes specific action (re-enable recv for `freedMemory`; begin shutdown for `shutdownStarted`).

**otofu `Notifier`:**
```odin
Notifier :: struct {
    read_fd:  Handle,
    write_fd: Handle,
}
```
Sends any byte (1 byte). The content is irrelevant — the wake signal only tells the Reactor that the inbox has data. The Reactor drains the inbox Mailbox and processes commands.

**Mismatches:**

1. **Alert type removed.** tofu encodes semantic intent (`freedMemory`, `shutdownStarted`) in the Notifier signal. otofu's Notifier is a pure wake mechanism — semantic intent is carried in the reactor inbox Mailbox (via command Messages). The Notifier byte only says "check your inbox."

2. **`freedMemory` alert has no otofu equivalent.** In tofu, when a message is returned to the pool, the Pool notifies the Notifier which notifies the Reactor to re-enable recv. In otofu, recv is never disabled for pool exhaustion — only for per-channel outbound queue depth.

3. **loopback TCP on Unix replaced by socketpair.** tofu uses a loopback TCP connection for the Notifier on Unix platforms. otofu uses `socketpair(2)` — simpler, lower overhead, no TCP overhead, stays in-kernel.

4. **Windows approach unchanged in concept.** Both tofu and otofu use a socket-based Notifier on Windows (AF_UNIX or loopback TCP). The specific mechanism is platform-dependent in both.

---

### `ampe/triggeredSkts.zig` → `reactor/tc_pool.odin` + `chanmgr/channel.odin` + `reactor/io_dispatch.odin`

This is the most significant structural change in the mapping.

**tofu `TriggeredSkt` (tagged union):**
```
notification: NotificationSkt   — wake socket (Notifier fd)
accept:       AcceptSkt          — listening socket with backlog queue
io:           IoSkt              — connected socket, has send_queue + recv_buf
dumb:         DumbSkt            — placeholder (not yet connected)
```

Each variant contains both the OS file descriptor AND the associated send/recv queues AND the state machine for that variant. `TriggeredSkt` is a fat polymorphic object.

**otofu decomposition:**

| tofu concern | otofu location |
|-------------|---------------|
| Poller registration token (seqn, FD handle) | `TC` in `reactor/tc_pool.odin` |
| Channel state machine (SM2) | `Channel` in `chanmgr/channel.odin` |
| Per-channel send queue | `Outbound_Queue` in `chanmgr/outbound.odin` |
| Per-channel receive buffer (partial frames) | `recv_buf` in `chanmgr/channel.odin` (S5 addition) |
| OS socket operations | `platform/socket.odin` + backend files |
| I/O dispatch logic (READ/WRITE/ACCEPT/ERROR) | `reactor/io_dispatch.odin` |
| Notification socket handling | `platform/notifier.odin` (separate from TC/Channel system) |

**Mismatches:**

1. **Fat union decomposed into thin roles.** tofu's `TriggeredSkt` bundles identity + state + I/O queues into one union. otofu separates concerns strictly: `TC` is only the Poller token (seq, channel_num, flags). `Channel` holds state and queues. `io_dispatch.odin` contains the dispatch logic.

2. **`NotificationSkt` removed from TC/Channel system.** In tofu, the Notifier socket is a `TriggeredSkt::notification` variant — it participates in the same union and iteration as regular sockets. In otofu, the Notifier is classified separately in Phase 3 (`inbox_pending` flag) and never enters the resolved[] array. Notifier is a separate platform concern.

3. **Backpressure mechanism changed.** tofu `IoSkt.triggers()` calculates interest flags dynamically: if send queue is full → clear WRITE interest; if pool empty → clear READ interest. otofu does not clear READ interest for pool exhaustion. Interest flags are managed by `io_dispatch.odin` based on Channel state only.

4. **`dumb` variant removed.** tofu's `DumbSkt` represents a channel before a socket is assigned. otofu handles this with `Channel_State.Idle` — no special socket variant needed.

---

### `ampe/channels.zig` → `chanmgr/channel.odin` + `chanmgr/numbers.odin` + `reactor/dual_map.odin`

**tofu `ActiveChannels`:**
```
mutex: Mutex
chn_seqn_map: HashMap(ChannelNumber, SeqN)
seqn_trc_map: HashMap(SeqN, *TriggeredSkt)
recently_removed: FixedQueue(SeqN)     // ABA guard: prevents SequenceNumber reuse
```

**otofu decomposition:**
- `chanmgr/numbers.odin`: `Number_Pool` — ChannelNumber assignment and release
- `reactor/dual_map.odin`: `Dual_Map { by_seqn, by_chan }` — the two hash maps, mutex-free
- `chanmgr/channel.odin`: `Channel` struct with SM2 state machine

**Mismatches:**

1. **Mutex removed from dual-map.** tofu protects `ActiveChannels` with a mutex because both Reactor and application threads access it (e.g., `post()` triggers channel lookup). otofu's `Dual_Map` has no mutex — it is Reactor-exclusive. The only cross-thread data movement is through Mailboxes. Application threads never touch the dual-map.

2. **ABA guard changed.** tofu uses a `recently_removed` queue (fixed-size FIFO of recently released SequenceNumbers) to prevent immediate reuse of a SequenceNumber while old events may still be in the OS queue. otofu uses monotonic SequenceNumber assignment (never wraps within a session) and validates seqn on every event in Phase 6. Stale events for old seqn values are silently discarded when not found in `by_seqn`.

3. **`ActiveChannel` fields restructured.** tofu's `ActiveChannel` holds `(channel_number, message_id, proto, context)` — protocol-level fields. otofu's `Channel` holds `(number, state, cg_id, remote_number, socket, outbound)` — lifecycle-oriented fields. Protocol correlation (`message_id`) is not engine-maintained in otofu.

---

### `ampe/MchnGroup.zig` → `otofu/channel_group.odin` + `runtime/router.odin`

**tofu `MchnGroup`:**
```
msgs[0]: *Mailbox   // App → Reactor (send side)
msgs[1]: *Mailbox   // Reactor → App (receive side)
semaphore           // for post() completion signaling
```

**otofu:**
- `CG { id, router }` — thin handle
- `Router { reactor_inbox, wake_fn, cg_entries }` — owns all Mailboxes

**Mismatches:**

1. **Mailbox ownership moved to Router.** tofu's MchnGroup owns its own mailboxes. otofu's Router owns all mailboxes centrally. The CG handle has no Mailbox fields — it references the Router.

2. **Send-side mailbox per CG removed.** tofu has `msgs[0]` (send side) per MchnGroup — each CG has its own send queue to the Reactor. otofu has a single `reactor_inbox` shared by all CGs. This simplifies the Reactor drain (one inbox, one drain) at the cost of FIFO ordering across CGs.

3. **Semaphore removed.** tofu uses a semaphore for `post()` to synchronize acknowledgment. otofu's `CG_Post` is fire-and-forget: `mbox_send` to reactor_inbox, then `Notifier.notify`. No completion semaphore.

4. **`updateReceiver` implementation changed.** tofu's VTable `updateReceiver(msg?)` wakes the application thread blocked on `waitReceive()`. otofu's `CG_Update_Receiver` sends a `CG_Receiver_Update` message through the Router, which the Reactor delivers to the application's CG outbox Mailbox; Matryoshka's Mailbox wakes the blocked receiver. The API method is preserved; only the delivery mechanism changes.

---

### `ampe/poller/core.zig` → `reactor/dual_map.odin` + `platform/poller.odin`

**tofu `PollerCore(Backend)`:**
```
chn_seqn_map: HashMap(ChannelNumber, SeqN)   // in PollerCore
seqn_trc_map: HashMap(SeqN, *TriggeredSkt)   // in PollerCore
backend: Backend                              // epoll/kqueue/etc.
```

tofu's PollerCore combines the identity maps AND the OS event backend in one structure.

**otofu:**
- Maps → `reactor/dual_map.odin` (owned by Reactor)
- OS backend → `platform/poller.odin` + backend files (owned by platform package)

**Mismatch:**

**Maps separated from OS backend.** This is an architecture-level decision, not a naming change.

In tofu, the PollerCore owns the ChannelNumber↔SequenceNumber↔*TriggeredSkt maps. This couples the channel tracking data structure to the OS polling mechanism.

In otofu, the OS poller knows only about (SequenceNumber, TriggerFlags) pairs — it does not know about ChannelNumbers or TriggeredChannel pointers. The maps live in `reactor/dual_map.odin`, owned exclusively by the Reactor event loop. The Poller is pure OS event notification with no channel tracking.

**Why this matters:** tofu's PollerCore cannot deregister by ChannelNumber without going through its own map. otofu's Reactor can manipulate the dual-map independently of the Poller — insert in Phase 5, query in Phase 6, remove in Phase 8 — with no lock and no coupling to OS event timing.

---

### `ampe/poller/poll_backend.zig` → (removed)

tofu includes a `poll(2)` fallback for platforms without epoll/kqueue. This covers:
- FreeBSD before kqueue was added (historical)
- Any POSIX platform not covered by the main backends

otofu does not include a poll fallback. The three supported backends (epoll/kqueue/AFD_POLL) cover Linux, macOS/BSD, and Windows. Any platform outside these three is not in otofu's scope at baseline.

**Consequence:** otofu is not portable to platforms without epoll, kqueue, or AFD_POLL. This is an explicit scope reduction.

---

## Architecture-Level Mismatches

These are structural differences that affect multiple files simultaneously.

---

### M-1 — VTable Polymorphism Removed

**tofu:** `AmpeVTable` and `CHNLSVTable` provide runtime-polymorphic interfaces for `Ampe` and `ChannelGroup`. Application code calls through `vtable.*` function pointers.

**otofu:** Direct procedure calls. `Engine_Create`, `CG_Post`, etc. are top-level procedures, not method calls through function pointers.

**Why:** tofu's VTable exists to support potential future alternative implementations of the Ampe interface. otofu has exactly one implementation. VTables add indirection with no benefit.

**API impact:** None. All 5 Ampe methods (`get`, `put`, `create`, `destroy`, `getAllocator`) and all 3 ChannelGroup methods (`post`, `waitReceive`, `updateReceiver`) are preserved as top-level procedures.

**Impact:** `ampe/vtables.zig` is removed entirely. `tofu.zig` and `ampe.zig` facades are rewritten as direct procedure modules.

---

### M-2 — Matryoshka Replaces tofu's Intrusive Infrastructure

**tofu:** `IntrusiveQueue.zig` provides intrusive FIFO for linked `Message` nodes. `Pool.zig` has its own mutex + linked list. The Mailbox (from the `mailbox` external dependency) handles cross-thread queues.

**otofu:** All intrusive list and pool infrastructure is provided by Matryoshka:
- `PolyNode` + `MayItem` replaces `IntrusiveQueue`
- `Pool` (Matryoshka) replaces `Pool.zig`
- `Mailbox` (Matryoshka) replaces the `mailbox` external dependency

**Impact:** `ampe/IntrusiveQueue.zig` is removed. `ampe/Pool.zig` is rewritten as a Matryoshka Pool wrapper. The `mailbox` external dependency disappears (absorbed into Matryoshka).

---

### M-3 — Backpressure Mechanism Changed

**tofu backpressure:**
1. Pool runs low → Pool.alerter fires → Notifier sends `freedMemory`
2. Reactor receives `freedMemory` → re-enables `recv` on all paused sockets
3. Remote sender experiences `WouldBlock` on send → backs off

Pool exhaustion disables recv globally.

**otofu backpressure:**
1. Per-channel outbound queue reaches `outbound_queue_depth` limit
2. `CG_Post` returns `Engine_Error.BackpressureExceeded` to the application
3. Application retries or drops the message
4. No global recv-disable; no pool-based signaling

**Why changed:** tofu's pool-based backpressure creates a global coupling: one channel's receiver filling the pool pauses ALL channels' receives. otofu's per-channel depth limit is scoped — one channel's backpressure does not affect others.

**Impact:** `alerter` callback in `Pool.zig` is removed. `freedMemory` alert in `Notifier.zig` is removed. `BackpressureExceeded` error is added to `Engine_Error`.

---

### M-4 — Protocol Layer Extracted

**tofu:** Protocol logic (Hello/Bye/Welcome sequences, OpCode routing) is inline in `Reactor.zig` and partially in `message.zig`. There is no separate protocol module.

**otofu:** Explicit L4 protocol layer with two files:
- `protocol/protocol.odin` — exhaustive OpCode switch, dispatch to handlers
- `protocol/handshake.odin` — Hello/Welcome/Bye state sequences

**Why extracted:** otofu's 6-layer architecture (P7) requires strict layer separation. Protocol logic in the Reactor file (L2) would violate the L2/L4 boundary. Extraction enables the key design decision from S2: `io_dispatch` sets deadlines; protocol only drives Channel state.

---

### M-5 — Timeout Management Extracted

**tofu:** Timeout is handled inline inside `Reactor.zig`. There is no separate timeout module.

**otofu:** `reactor/timeout.odin` — dedicated `Timeout_Manager` with `Timeout_Manager.deadlines` map. Phase 1 queries it; Phase 4 collects expired entries; `io_dispatch.odin` sets/clears deadlines after state transitions.

**Why extracted:** P6's 9-phase loop requires timeout as an explicit Phase 1 input. Inline timeout handling makes the phase boundaries unclear and prevents the S2 rule: "L4 does not call timeout_manager."

---

### M-6 — Dual-Map Ownership Moved from Poller to Reactor

See per-component analysis of `ampe/poller/core.zig`.

The consequence: `platform/poller.odin` in otofu is simpler than tofu's `PollerCore`. It has no maps, no channel tracking. It only:
- Registers an FD with a SequenceNumber and interest flags
- Deregisters by SequenceNumber
- Waits and returns `(SequenceNumber, TriggerFlags)` pairs

All channel-to-event resolution happens in the Reactor's dual-map (Phase 6), not in the Poller.

---

## Features in tofu Without otofu Equivalent

| tofu feature | otofu status | Reason |
|-------------|-------------|--------|
| Pool `alerter` callback | **Absent** | Replaced by Reserved Pool + per-channel backpressure |
| `poll(2)` fallback backend | **Absent** | Not in scope; epoll/kqueue/AFD_POLL only |
| `recently_removed` ABA queue | **Absent** | Replaced by monotonic SequenceNumber validation |
| `TriggeredSkt` fat union | **Absent** | Decomposed into TC, Channel, io_dispatch |
| `DumbSkt` variant | **Absent** | Replaced by `Channel_State.Idle` |
| `AmpeVTable` / `CHNLSVTable` | **Absent** | Direct procedures; single implementation |
| `AmpeError` parallel error set | **Absent** | Single `Engine_Error` enum; no parallel set |
| `SocketCreator` factory | **Absent** | Merged into io_dispatch command handlers |

*Note: The original S6 listed UDS support, TextHeaders, status byte, and message_id as absent from otofu. All four are present in the corrected design — their absence was an error in the original S4/S6, not an intentional design decision.*

---

## Legitimate Mismatches

This section documents intentional architectural differences between tofu and otofu. These are not errors to be fixed — they are required by the Odin language, the Matryoshka integration, or the reactor design.

| Area | tofu | otofu | Why legitimate |
|------|------|-------|---------------|
| VTable polymorphism | `Ampe`/`ChannelGroup` use vtable function pointers | Direct procedures in Odin | Single-implementation; Odin has no need for runtime dispatch; API semantics identical |
| Intrusive list (generic) | `IntrusiveQueue(T)` generic | `message/queue.odin` `Message_Queue` (Message-specific) | Only ever instantiated for Message; specialization is equivalent; intrusive links via PolyNode's embedded list.Node |
| Pool alerter | `Pool.alerter` callback wakes Reactor on `pool.put()` | No alerter; Reserved Pool solves the deadlock differently | Backpressure model changed (per-channel outbound queue, not pool exhaustion) |
| Notifier typed alert | `Alert :: enum { freedMemory, shutdownStarted }` | Pure wake byte; semantic intent carried in Mailbox messages | Alert type removal is safe: otofu never needs `freedMemory` signal; shutdown uses Mailbox commands |
| `TriggeredSkt` fat union | Single union with FD + state + queues + dispatch logic | Decomposed: TC (Poller token), Channel (state + queue), io_dispatch (logic) | Strict ownership separation; one struct = one responsibility |
| Mutex on `ActiveChannels` | Protects shared channel map (app + reactor threads) | No mutex on `Dual_Map` (Reactor-exclusive) | Cross-thread data via Mailbox only; no shared mutable state |
| ABA prevention | `recently_removed` FixedQueue | Monotonic SequenceNumber; stale events discarded silently | Simpler; correct under Matryoshka ownership model |
| `SocketCreator` factory | Separate `SocketCreator.zig` | Merged into `io_dispatch.odin` | No factory object needed; Odin procedures handle creation inline |
| Poll fallback backend | `poll_backend.zig` (generic POSIX) | Not ported | otofu targets Linux/Darwin/Windows only; generic poll not needed |
| `vtables.zig` | Defines `AmpeVTable`, `CHNLSVTable` | Not ported | No vtable needed with direct procedures |
| AmpeStatus 35 values | Full error taxonomy as status codes AND in wire header | Engine_Error (13 values) for API; status byte in BinaryHeader preserved on wire | Wire format preserved; internal error codes simplified to otofu's actual failure modes |
| Thread model | Mutex-based synchronization | Matryoshka Mailbox ownership transfer | No shared mutable state across threads; ownership-based handoff only |
| UDS Notifier on Unix | Loopback TCP connection for cross-thread wake | `socketpair(2)` for cross-thread wake | Lower overhead; stays in-kernel; same semantics |
| `CG_Post` acknowledgment | Semaphore in MchnGroup for `post()` completion | Fire-and-forget `mbox_send` + Notifier wake | Matryoshka Mailbox delivery is ordered; semaphore unnecessary |
| Mailbox ownership | Each MchnGroup owns its own send/recv Mailboxes | Router owns all Mailboxes centrally | Centralized ownership simplifies drain (one inbox) and enables cross-CG ordering |
| Protocol layer | Inline in `Reactor.zig` | Explicit `protocol/` package | Required by P7 layering; enables strict L2/L4 boundary |
| Timeout management | Inline in `Reactor.zig` | Explicit `reactor/timeout.odin` | Required by P6 9-phase loop; Phase 1 needs explicit timeout as input |
| Map ownership | PollerCore owns channel maps | Reactor owns Dual_Map; Poller is stateless | Decouples OS event timing from channel lifecycle management |
| `message_id` correlation | Engine tracks request-response correlation in message header | Engine preserves the `message_id` field on wire but does not interpret it; correlation is application-layer concern | Simplifies engine; correlation semantics vary per application |
| `@"<void*>"` / `@"<ctx>"` | Reserved pointer fields in Message for app and engine | Replaced by Matryoshka `PolyNode` at offset 0 | Matryoshka ownership model subsumes the pointer-tagging pattern; RULE-5 compliant |
| `Message` package location | `message.zig` is in `src/` alongside other files | `message/message.odin` is a top-level package | Odin package system requires clients to import by package path; no re-export mechanism; top-level placement is the only option |
