# otofu — Implementation Plan

Sources: P3, P4, P5, P6, P7, S1, S2, S3, S4, S5, S6
S5 corrections applied throughout (Engine_Options in types/, reserved_pool Reactor-owned, channels collection in reactor_state, Channel.socket embedded, recv_buf added, deadline_ms removed from Channel).

This document defines the strict, dependency-aware build sequence for a human developer.

---

## Package Structure Rationale

```
otofu/           ← package otofu   — engine.odin + channel_group.odin  (public lifecycle API)
message/         ← package message — Message, Address, BinaryHeader, TextHeaders, Queue
types/           ← package types   — wire enums: Ampe_Status, OpCode, ProtoFields, identifiers, flags
internal/        ← platform, reactor, chanmgr, runtime, protocol  (not importable by clients)
```

**Why engine.odin and channel_group.odin are at the root (not in a subfolder):**
- Odin package name = directory name. Files in `otofu/` are `package otofu`.
- Client writes: `import otofu "github.com/g41797/otofu"` → `otofu.Engine_Create(...)` — clean entry point.
- Moving them to `otofu/api/` would make them `package api`, changing the import path.
- The root directory IS the library entry point; these two files ARE `package otofu`.

**Why message/ is top-level (not internal/):**
- `Message` is a public type. Clients declare `^message.Message` variables directly.
- Odin has no re-export. If Message were under `internal/`, clients could not name the type.
- Engine and ChannelGroup are opaque handles — clients never inspect their fields.

**Client import pattern:**
```odin
import otofu    "github.com/g41797/otofu"          // Engine_Create, CG_Create, etc.
import message  "github.com/g41797/otofu/message"  // Message, Address, BinaryHeader, etc.
```

---

## Preliminary Plan — Learning-Aware Phase Order

Porting tofu (Zig) to otofu (Odin) has two parallel challenges:
1. **Engineering** — correct architecture, wire format, ownership, concurrency
2. **Language learning** — Odin syntax, idioms, allocators, build system, testing

The phase order below respects both. Each phase introduces one new Odin concept while
producing a compilable, independently testable increment.

```
Phase A  — Pure Odin types (no logic, no allocators)        ← learn syntax + build system
Phase B  — Structs + simple procedures (string ops, math)   ← learn procs, enums, tagged unions
Phase C  — Allocators + Message lifecycle                    ← learn mem.Allocator idiom
Phase D  — Matryoshka integration (Pool, Mailbox)           ← learn external package usage
Phase E  — Channel state machines + protocol logic           ← learn complex Odin patterns
Phase F  — OS / platform layer (sockets, epoll/kqueue)      ← learn OS syscall bindings
Phase G  — Reactor event loop (concurrency, threads)        ← learn Odin threading
Phase H  — Public API + integration tests                   ← learn Odin test framework
```

### Phase A — Pure Type Definitions (Zero Logic)
**Odin:** package structure, `enum`, `distinct` types, `odin build`
**Files:** `types/status.odin` (Ampe_Status enum u8), `types/opcodes.odin`, `types/identifiers.odin`, `types/flags.odin`
**Test:** package compiles; enum switch exhaustiveness check.
**Risk: NONE**

### Phase B — Message Wire Format (Structs + String Ops)
**Odin:** `#packed` structs, big-endian fields, tagged unions, `strings` package
**Files:** `message/binary_header.odin`, `message/text_headers.odin`, `message/address.odin`, `message/helpers.odin`
**Test:** BinaryHeader encode→decode round-trip; TextHeader write→find; Address format→parse.
**Risk: LOW**

### Phase C — Message Lifecycle (Allocators)
**Odin:** `mem.Allocator`, `defer`, `sync/atomic`
**Files:** `message/message.odin` (Msg_Create/Destroy/Clone), `message/queue.odin` (Message_Queue, MQ_*)
**Test:** Message create→fill→destroy; enqueue 10→dequeue→FIFO order.
**Risk: LOW-MEDIUM**

### Phase D — Matryoshka Integration (Pools + Mailbox)
**Odin:** external package import, Matryoshka Pool/Mailbox/PolyNode API
**Files:** `runtime/message_pool.odin`, `runtime/reserved_pool.odin`
**Test:** pool get→put→pool-empty behavior.
**Risk: MEDIUM** (first external dependency)

### Phase E — Channel + Protocol Logic
**Odin:** complex state machines, multi-file packages
**Files:** `chanmgr/` (Channel, SM2), `runtime/framer.odin`, `protocol/` (dispatch, handshake, Bye)
**Test:** state machine transitions; frame encode→decode round-trip; dispatch completeness.
**Risk: MEDIUM**

### Phase F — Platform / OS Layer
**Odin:** `foreign` bindings, OS-specific `when` blocks
**Files:** `platform/socket.odin`, `platform/poller_linux.odin`/`poller_macos.odin`, `platform/notifier.odin`
**Test:** local socket connect→send→recv; poller fd registration→trigger.
**Risk: HIGH** (OS-specific, first `foreign` usage)

### Phase G — Reactor Event Loop
**Odin:** `thread.create`, `sync.Mutex`, `sync/atomic`, reactor pattern
**Files:** `reactor/dual_map.odin`, `reactor/tc_pool.odin`, `reactor/io_dispatch.odin`, `reactor/reactor.odin`
**Test:** start reactor → HelloRequest → connect → WelcomeResponse in outbox.
**Risk: HIGH** (concurrency, all layers integrated)

### Phase H — Public API + Integration
**Odin:** test framework, package integration
**Files:** `otofu/engine.odin` (Engine_Create/Destroy/Get/Put), `otofu/channel_group.odin` (CG_*)
**Test:** full echo client/server mirroring tofu cookbook recipes.
**Risk: LOW** (thin wrappers over Phase G)

### Key Principles
1. Never start a phase until the previous is tested
2. Use tofu source as the reference — translate literally first, optimize later
3. Matryoshka is a black box — use its API, don't port it

### Status
- [ ] A — type definitions
- [ ] B — wire format
- [ ] C — message lifecycle
- [ ] D — pools (Matryoshka)
- [ ] E — channels + protocol
- [ ] F — platform/OS
- [ ] G — reactor
- [ ] H — public API

---

## Detailed Reference Plan

> **Note:** This section predates Round 4 corrections. The following substitutions apply throughout:
> - `types/errors.odin` / `Engine_Error` → **does not exist**; replaced by `types/status.odin` / `Ampe_Status`
> - `types/address.odin` → **does not exist**; replaced by `message/address.odin` (full Address type + behavior)
> - `message/address_headers.odin` → merged into `message/address.odin`
> - `internal/runtime/message.odin` → **does not exist**; Message lives in top-level `message/` package
> - Step 0.1 (`types/errors.odin`) is superseded — skip it; start from Step 0.2

---

## Ground Rules

1. **Bottom-up only.** A file is not written until all its import dependencies exist.
2. **Ownership-first.** No message handling before pool rules are validated. No channel I/O before SM2 is correct.
3. **No public API before runtime is stable.** L5 (`otofu/`) is written last.
4. **Each phase produces a compilable, testable increment.** If a phase cannot be tested in isolation, it is too large.
5. **S5 corrections are authoritative.** The S4 structures as corrected by S5 are the implementation target.
6. **P5 discipline applies throughout:**
   - MR-1: Every allocation uses an explicit `mem.Allocator` parameter. No `context.allocator` inside otofu.
   - MR-2: Reactor thread sets `context.allocator = engine.allocator` as its first statement.
   - MR-3: Multi-step init uses success-flag `defer` instead of `errdefer`.
   - MR-4: Pool `on_put` hooks check Appendable buffer capacity and free oversized buffers.

---

## Phase Overview

```
Phase 0 — Shared Type Definitions
Phase 1 — Core Traveling Struct (Message)
Phase 2 — Platform Struct Layer (L1 types)
Phase 3 — Memory & Ownership (Pools, Queues)
Phase 4 — OS/Platform Implementations (L1 procs)
Phase 5 — Channel State Management (chanmgr)
Phase 6 — Runtime Messaging Infrastructure (L3)
Phase 7 — Reactor Support Structures (L2 non-loop)
Phase 8 — Protocol Layer (L4)
Phase 9 — I/O Dispatch (L2 bridge)
Phase 10 — Reactor Event Loop (L2 core)
Phase 11 — Public API (L5)
Phase 12 — Integration & Hardening
```

---

## Phase 0 — Shared Type Definitions

**Goal:** All shared enums, distinct types, and value structs exist. No logic. No allocation. No imports needed.

Every subsequent phase depends on this phase. It must be complete before any other work starts.

---

### Step 0.1 — `types/errors.odin`

**Implements:**
- `Engine_Error` enum (13 values: None, WouldBlock, ConnectionRefused, ConnectionReset, TimedOut, AddressInUse, ProtocolError, BackpressureExceeded, PoolEmpty, EngineDraining, EngineDestroyed, ReservedPoolExhausted, InternalError)

**Depends on:** Nothing.

**Reason:** Every other file returns or receives `Engine_Error`. Must exist first.

**Ownership rules:** N/A — pure enum definition.

**Becomes testable:** `types` package compiles.

**Risk: LOW**

---

### Step 0.2 — `types/opcodes.odin`

**Implements:**
- `OpCode` enum (10 values: Connect, Listen, Close, Drain, Hello, Welcome, Bye, Bye_Ack, Data, Channel_Closed)

**Depends on:** Nothing.

**Reason:** OpCode is embedded in `Header` (runtime/message.odin) and used in protocol dispatch. Must precede both.

**Ownership rules:** N/A.

**Becomes testable:** `types` package compiles.

**Risk: LOW**

---

### Step 0.3 — `types/identifiers.odin`

**Implements:**
- `ChannelNumber :: distinct u16` (0 = invalid)
- `SequenceNumber :: distinct u64`
- `ChannelGroupId :: distinct u32`

**Depends on:** Nothing.

**Reason:** Used by chanmgr, reactor, runtime. Distinct types prevent silent interchange at call sites — must be defined before any struct embeds them.

**Ownership rules:** ChannelNumber is a short-lived correlation token (P3 L-C4). Must not be used as a persistent session ID.

**Becomes testable:** Compiler rejects `ChannelNumber + SequenceNumber` assignments.

**Risk: LOW**

---

### Step 0.4 — `types/flags.odin`

**Implements:**
- `Trigger_Flag` enum u8 + `Trigger_Flags :: bit_set[Trigger_Flag; u8]` (Read, Write, Error, Hup, Accept)
- `Allocation_Strategy` enum (Pool_Only, Always)
- `Decode_Result` enum (Ok, Incomplete, Invalid)
- `Channel_State` enum (Idle, Connecting, Handshaking, Open, Closing, Closed)
- `TC_State` enum (Allocated, Registered, Active, Deregistering, Deregistered)
- `Engine_State` enum (Starting, Running, Draining, Destroyed)
- `Recv_Result` enum (Ok, Timeout, Closed, Interrupted)

**Depends on:** Nothing.

**Reason:** `Channel_State` is required by chanmgr/channel.odin. `TC_State` is required by reactor/tc_pool.odin. `Trigger_Flags` is required by platform/poller.odin. All must precede the files that embed them.

**Ownership rules:** `Channel_State` is Reactor-thread-only (P3 L-C: "All state transitions driven by Reactor"). Application must never read Channel.state directly.

**Becomes testable:** `types` package compiles.

**Risk: LOW**

---

### Step 0.5 — `types/address.odin`

**Implements:**
- `IP_Version` enum (V4, V6)
- `Address` struct (`ip: [16]u8`, `port: u16`, `version: IP_Version`)

**Depends on:** Nothing.

**Reason:** Used by platform/socket.odin (Socket fields) and otofu/channel_group.odin (CG_Connect/CG_Listen parameters).

**Ownership rules:** Value type. No ownership.

**Becomes testable:** `types` package compiles.

**Risk: LOW**

---

### Step 0.6 — `types/options.odin` *(S5 addition)*

**Implements:**
- `Engine_Options` struct (8 fields: max_messages, reserved_messages, max_channels, outbound_queue_depth, max_appendable_capacity, connect_timeout_ms, handshake_timeout_ms, bye_timeout_ms)

**Depends on:** Nothing.

**Reason:** Moved from `otofu/engine.odin` to `types/` by S5 P-01 fix. Both `otofu` and `reactor` need Engine_Options. If it stayed in `otofu`, reactor importing it would create a cycle (otofu→reactor→otofu). In `types`, both can import without a cycle.

**Ownership rules:** Value type. Copied into `reactor_state` at startup (by value).

**Becomes testable:** `types` package compiles. Compiler rejects options access in reactor if this step is skipped.

**Risk: LOW** — but skipping this causes an immediate cycle that prevents Phase 10 from compiling.

---

**Phase 0 complete:** `types` package compiles with all 6 files. No behavior. No tests needed beyond compilation.

---

## Phase 1 — Core Traveling Struct

**Goal:** `runtime/message.odin` compiles with the correct Matryoshka layout. This is the single most critical file in the entire codebase. If PolyNode is not at offset 0, all pool operations silently corrupt memory.

---

### Step 1.1 — `runtime/message.odin`

**Implements:**
- `MESSAGE_ID :: 1` — PolyNode discriminator constant
- `Header` struct (opcode: OpCode, channel: ChannelNumber, id: ChannelGroupId, meta_len: u16, body_len: u32)
- `Appendable` struct (buf: [dynamic]u8, len: int)
- `Message` struct:
  ```odin
  Message :: struct {
      using poly: matryoshka.PolyNode,  // MUST be first — offset 0
      header:     Header,
      meta:       Appendable,
      body:       Appendable,
  }
  ```

**Depends on:**
- Phase 0 (types: OpCode, ChannelNumber, ChannelGroupId)
- Matryoshka (external: PolyNode)

**Reason:** Message is the primary traveling unit. Every other runtime file (message_pool, reserved_pool, framer) depends on this struct. The PolyNode offset-0 constraint (P4 C1) must be verified here before any pool code is written.

**Ownership rules (P3 Message):**
- Every `get()` that returns non-nil must be paired with exactly one `put()` or `post()` (L-M1)
- Never store a raw `^Message` pointer separately from `^MayItem` (L-M3)
- After `post()`, treat the MayItem as gone (L-M4)
- MetaHeaders and Body buffers must not be referenced after `put()` or `post()` (L-M5)

**Validation required before proceeding:**
- Write a test that casts `^Message` to `^matryoshka.PolyNode` and back. Verify `(^Message)(node) == original_message_ptr`. If this fails, the offset-0 constraint is violated. **Do not proceed to Step 1.2 until this test passes.**

**Becomes testable:** Cast validation. Struct layout check via `offset_of(Message, "poly") == 0`.

**Risk: HIGH** — Silent corruption if offset-0 is wrong. Validate first.

---

**Phase 1 checkpoint:** Confirm `offset_of(Message, "poly") == 0` compiles and passes before continuing.

---

## Phase 2 — Platform Struct Layer

**Goal:** All L1 type definitions exist so that chanmgr and reactor can reference `platform.Socket`, `platform.Poller`, `platform.Notifier`, and `platform.Event`. Procedure bodies are NOT required yet — only struct definitions and type shells.

These files define types. Platform backend files (Step 4.*) add procedure bodies.

---

### Step 2.1 — `platform/socket.odin` (struct only)

**Implements:**
- `Handle :: distinct uintptr`
- `Socket` struct (fd: Handle, local: Address, remote: Address, nonblock: bool)
  *With S5 correction: Socket is embedded by value in Channel — its address must remain stable within Channel's lifetime.*

**Depends on:**
- Phase 0 (types: Address)

**Reason:** `chanmgr/channel.odin` (Phase 5) embeds `platform.Socket` by value. The struct must exist before Channel can be defined.

**Ownership rules (P3 Socket):**
- Reactor thread exclusively owns all Socket instances (L-S1 through L-S5)
- Application thread must never touch any Socket field
- FD must not be closed before TriggeredChannel is deregistered (L-S4, R4.1)

**Becomes testable:** `chanmgr` package can import `platform` and reference `Socket` struct fields.

**Risk: LOW**

---

### Step 2.2 — `platform/poller.odin` (struct only)

**Implements:**
- `Event` struct (seqn: SequenceNumber, flags: Trigger_Flags)
- `Poller` struct (fd: Handle — plus platform-specific fields completed by backend files)
- Procedure signature declarations for: `Poller_Register`, `Poller_Deregister`, `Poller_Wait`, `Poller_Close`

**Depends on:**
- Phase 0 (types: SequenceNumber, Trigger_Flags)
- Step 2.1 (Handle)

**Reason:** `reactor/reactor.odin` embeds `platform.Poller`. The struct shell and procedure signatures must exist before reactor compiles. Backend files complete the implementation.

**Ownership rules:** Poller owned by Reactor thread exclusively.

**Becomes testable:** `reactor` package can import `platform` and reference `Poller` and `Event`.

**Risk: LOW**

---

### Step 2.3 — `platform/notifier.odin` (struct only)

**Implements:**
- `Notifier` struct (read_fd: Handle, write_fd: Handle)
- Procedure signature declarations for: `Notifier_Create`, `Notifier_Notify`, `Notifier_Drain`, `Notifier_Close`

**Depends on:**
- Step 2.1 (Handle)

**Reason:** `reactor/reactor.odin` embeds `platform.Notifier`. Struct must exist before reactor. `Notifier_Notify` is referenced in `runtime/router.odin` (via Wake_Fn callback — but Wake_Fn is a proc type, not a direct Notifier reference, so this is not a hard compile dependency).

**Ownership rules:** Notifier owned by Reactor thread. Both FDs created and closed by Notifier procedures. Application thread calls only Notifier_Notify (the write side).

**Becomes testable:** `reactor` package compiles the reactor_state struct definition.

**Risk: LOW**

---

**Phase 2 complete:** `platform` package has all struct definitions. No procedure bodies yet. `chanmgr` and `reactor` can reference platform types.

**Parallel opportunity:** Steps 2.1, 2.2, 2.3 have no interdependency. Write in parallel.

---

## Phase 3 — Memory & Ownership Infrastructure

**Goal:** Message pools with on_get/on_put hooks and the per-channel outbound queue are fully implemented and testable. This phase validates the Matryoshka pool integration before any I/O code.

---

### Step 3.1 — `runtime/message_pool.odin`

**Implements:**
- `msg_pool_ctx` struct (allocator: mem.Allocator, max_appendable_cap: int)
- `Message_Pool` struct (pool: matryoshka.Pool, ctx: msg_pool_ctx)
- `MP_Create`, `MP_Get`, `MP_Put`, `MP_Close` procedures
- `on_get` hook: if m^ == nil → allocate new Message (explicit allocator, MR-1); else reset Header, clear meta/body Appendable content (keep capacity)
- `on_put` hook: if meta/body capacity > max_appendable_cap → free buffers (MR-4); else retain

**Depends on:**
- Step 1.1 (Message, Appendable)
- Phase 0 (types: Allocation_Strategy)
- Matryoshka (Pool)

**Reason:** Message ownership lifecycle (P3 TP-M1 through TP-M5) depends on correct pool behavior. Validate before any messaging code is written.

**Ownership rules (P3 MessagePool):**
- MessagePool is created during Engine `starting` state (L-MP1)
- MessagePool destroyed after all in-flight Messages returned (L-MP2)
- on_get/on_put hooks must NOT call MP_Get/MP_Put on the same pool — reentrancy forbidden (P4 C4)
- `defer MP_Put(pool, &m)` must be placed BEFORE the call to `MP_Get` — not after (P4 C8)

**Validation required:**
- Confirm `on_get` with nil m^ allocates a new Message with `poly.id == MESSAGE_ID`.
- Confirm `on_get` with non-nil m^ resets content but does not allocate.
- Confirm `on_put` clears `m^` to nil (caller loses ownership).
- Confirm oversized Appendable buffer is freed in `on_put`.

**Becomes testable:** Pool get/put cycle; double-put safety (no-op on nil); capacity enforcement.

**Risk: MEDIUM** — on_get/on_put reentrancy constraint (P4 C4) is easy to violate accidentally. The hook must not call back into the pool.

---

### Step 3.2 — `runtime/reserved_pool.odin`

**Implements:**
- `reserved_pool_ctx` struct (allocator: mem.Allocator, fixed_count: int)
- `Reserved_Pool` struct (pool: matryoshka.Pool, ctx: reserved_pool_ctx)
- `RP_Create`, `RP_Get`, `RP_Put`, `RP_Close` procedures
- `on_get` hook: same as message_pool; allocates via explicit allocator
- `on_put` hook: never frees items (fixed capacity — always stores)
- `RP_Create` pre-allocates exactly `fixed_count` Messages into the pool using `.New_Only` mode

**Depends on:**
- Step 1.1 (Message)
- Step 3.1 (same hook pattern — implement after message_pool is validated)

**Reason:** Reserved pool solves R6.3 deadlock (P4 V-M3): application holds all Messages and blocks on waitReceive; Reactor needs a Message to deliver Channel_Closed. Must be separate from application pool.

**Ownership rules:**
- Never exposed to application code. Application never calls RP_Get.
- `reserved_messages >= max_channels * 2` (AI-14) — validated at `RP_Create` time
- Capacity is fixed: `on_put` always stores; pool never grows beyond `fixed_count`

**Validation required:**
- Confirm exactly `fixed_count` items are pre-allocated.
- Confirm RP_Get on an empty pool returns nil (no allocation — `.Available_Only` mode only).
- Confirm RP_Put always stores (never frees).

**Becomes testable:** Reserved pool isolation; R6.3 scenario simulation (exhaust app pool, Reactor still gets from reserved).

**Risk: MEDIUM** — The pre-allocation step in RP_Create (multiple `.New_Only` gets to seed the pool) is not standard Matryoshka usage. Verify Matryoshka `.New_Only` mode forces on_get with nil m^ even when the pool has capacity.

---

### Step 3.3 — `chanmgr/outbound.odin`

**Implements:**
- `Outbound_Queue` struct (items: [dynamic]matryoshka.MayItem, depth_limit: int)
- `Ch_Enqueue_Outbound` (returns BackpressureExceeded if at depth_limit)
- `Ch_Dequeue_Outbound` (returns MayItem; nil if empty)
- `Ch_Outbound_Empty` (bool predicate)

**Depends on:**
- Phase 0 (types: Engine_Error.BackpressureExceeded)
- Matryoshka (MayItem)

**Reason:** Outbound_Queue is embedded in Channel (Phase 5). It must exist before Channel. Implementing here validates the bounded-queue backpressure mechanism independently before Channel integrates it.

**Ownership rules:**
- Items in queue are owned-engine (ownership transferred from application on post, before routing to channel)
- On channel close (Phase 8), all remaining items must be returned to the reserved pool via MP_Put, not dropped (P3 TP-M5)
- `depth_limit` set once at Channel allocation from `Engine_Options.outbound_queue_depth`

**Becomes testable:** Enqueue up to limit, confirm BackpressureExceeded on overflow. Dequeue in FIFO order. Empty predicate.

**Risk: LOW**

---

**Phase 3 complete:** Core memory and ownership infrastructure is validated. Pool hooks work correctly. Outbound queue enforces backpressure. These are prerequisites for all subsequent phases.

**Parallel opportunity:** Steps 3.1, 3.2, 3.3 can be written in parallel (no interdependency). However, 3.2 should be validated after 3.1 since the hook pattern is identical — write 3.1 first, validate, then write 3.2 following the same pattern.

---

## Phase 4 — OS/Platform Implementations

**Goal:** At least one platform has fully working Poller, Notifier, and Socket procedure bodies. This is the earliest point where real OS behavior can be tested.

Write for the primary development platform first. Other platform backends follow the same pattern.

---

### Step 4.1 — `platform/socket_unix.odin` (Linux/macOS)

**Implements:**
- `Socket_Create` (socket() syscall, set SOCK_NONBLOCK)
- `Socket_Set_Nonblocking` (fcntl O_NONBLOCK)
- `Socket_Bind`, `Socket_Listen`, `Socket_Accept` (non-blocking accept)
- `Socket_Connect` (non-blocking connect → returns WouldBlock immediately)
- `Socket_Connect_Complete` (getsockopt SO_ERROR — called on WRITE event)
- `Socket_Send` (send() → returns WouldBlock on EAGAIN)
- `Socket_Recv` (recv() → returns WouldBlock on EAGAIN, ConnectionReset on ECONNRESET)
- `Socket_Set_Linger` (SO_LINGER = 0 — abortive close)
- `Socket_Close` (close() after SO_LINGER)
- OS error → Engine_Error mapping table

**Depends on:**
- Step 2.1 (Socket struct, Handle)
- Phase 0 (types: Address, Engine_Error)

**Reason:** Socket is the lowest-level I/O primitive. All other platform work builds on this. Test socket operations in isolation before adding Poller.

**Ownership rules (P3 Socket):**
- `Socket_Close` must only be called after TriggeredChannel is deregistered (L-S4) — but this ordering is enforced by reactor, not by Socket itself
- `Socket_Set_Linger` must be called before `Socket_Close` on all IO sockets (L-S3)
- FD from `Socket_Accept` that has no Channel to attach to must be `Socket_Close`-d immediately (L-S5)

**Becomes testable:** Loopback TCP connect/accept/send/recv. Non-blocking error codes (EAGAIN → WouldBlock).

**Risk: LOW** — Standard POSIX; well-understood.

---

### Step 4.2 — `platform/notifier_unix.odin`

**Implements:**
- `Notifier_Create` (socketpair(AF_UNIX, SOCK_STREAM, 0) — both sockets set non-blocking)
- `Notifier_Notify` (write 1 byte to write_fd — idempotent if pipe buffer full)
- `Notifier_Drain` (read all bytes from read_fd until EAGAIN)
- `Notifier_Close` (close both FDs)

**Depends on:**
- Step 2.3 (Notifier struct, Handle)
- Step 4.1 (Socket_Set_Nonblocking pattern, error mapping)

**Reason:** Notifier is the Reactor's wake mechanism. Must work correctly before Poller integration, since Phase 10 (reactor loop) depends on Notifier events.

**Ownership rules:** Notifier owned by Reactor. `Notifier_Notify` is the ONLY Notifier call made by application threads (via `wake_fn` callback). Reactor calls `Notifier_Drain` during Phase 5 after classifying the event.

**Critical ordering rule (LI-8, P4):** Caller must `mbox_send` to reactor_inbox BEFORE calling `Notifier_Notify`. This is not enforced here — it is enforced in `runtime/router.odin`. Document this constraint in the Notifier file as a comment.

**Becomes testable:** Write from one thread, read from another. Confirm drain clears all buffered wake signals. Confirm idempotency (double-notify does not block or error).

**Risk: LOW**

---

### Step 4.3 — `platform/poller_linux.odin` (or darwin/windows)

**Implements (Linux — epoll):**
- `Poller_Create` (epoll_create1(EPOLL_CLOEXEC))
- `Poller_Register` (epoll_ctl ADD — embeds SequenceNumber in epoll_data.u64)
- `Poller_Modify` (epoll_ctl MOD — re-arm with updated interest flags)
- `Poller_Deregister` (epoll_ctl DEL)
- `Poller_Wait` (epoll_wait → returns []Event{seqn, flags})
- `Poller_Close` (close epoll fd)
- epoll flags → Trigger_Flags conversion

**Depends on:**
- Step 2.2 (Poller struct, Event)
- Phase 0 (types: SequenceNumber, Trigger_Flags, Engine_Error)
- Step 4.1 (socket FDs to register)

**Reason:** Poller is the blocking call that drives the entire Reactor. Must be validated before the event loop is written.

**Critical constraint (P3 L-TC3, P6 LI-5):** `Poller_Deregister` must complete before any subsequent `Socket_Close`. This ordering is enforced by the caller (reactor Phase 8), not by Poller itself. Document in the file.

**Becomes testable:** Register a socket FD, write to the other end, call Poller_Wait, confirm Event arrives with correct seqn and READ flag. Register/deregister cycle.

**Risk: MEDIUM** — SequenceNumber embedding in epoll_data.u64 must survive the round-trip through the kernel. Verify alignment. On 32-bit platforms, u64 in epoll_data is split — test explicitly.

---

### Step 4.4 — `platform/socket_windows.odin` *(optional for Linux-first development)*

**Implements:** Winsock2 equivalents of all Socket procedures.

**Depends on:** Step 2.1, Phase 0.

**Risk: MEDIUM** — Winsock2 non-blocking semantics differ from POSIX (WSAGetLastError, WSAEWOULDBLOCK). `base_handle` field for AFD operations requires separate attention.

---

### Step 4.5 — `platform/poller_windows.odin` *(optional for Linux-first development)*

**Implements:** AFD_POLL backend (NtDeviceIoControlFile, AFD structures, IO_STATUS_BLOCK).

**Depends on:** Step 2.2, Step 4.4.

**Risk: HIGH** — Windows AFD_POLL requires kernel-facing IO_STATUS_BLOCK inside TC. The TC must not be freed while an AFD operation is pending (P4 V-TC2). This constraint is enforced by reactor/tc_pool.odin's on_put hook — but the Windows-specific platform file must document the required handshake.

---

**Phase 4 complete (primary platform):** Real OS I/O works. Loopback tests pass. Notifier wake works. Poller event delivery verified.

**Parallel opportunity:** Steps 4.1, 4.2, 4.3 can be written in parallel on the same platform. Steps 4.4, 4.5 are the Windows path and can proceed in parallel with 4.1-4.3 if a separate developer handles Windows.

---

## Phase 5 — Channel State Management

**Goal:** The `chanmgr` package is complete. Channel SM2 is correct. ChannelNumber allocation works. The full Channel struct (with embedded Socket, Outbound_Queue, recv_buf) is defined.

---

### Step 5.1 — `chanmgr/numbers.odin`

**Implements:**
- `Number_Pool` struct (used: [dynamic]bool, max: u16, next: u16)
- `Ch_Assign_Number` → (ChannelNumber, bool) — scans bitmap for free slot; 0 is invalid
- `Ch_Release_Number` — marks slot as free; resets `next` hint

**Depends on:**
- Phase 0 (types: ChannelNumber)

**Reason:** ChannelNumber assignment is prerequisite for Channel creation. This module has no other dependencies and can be written and tested early.

**Ownership rules (P3 L-C4):** ChannelNumber released on Channel `Closed`. A new Channel may receive the same number. The Number_Pool makes this recycling explicit. `Ch_Release_Number` must be called in Phase 8 AFTER `Channel_Closed` notification is sent (AI-5, P3 L-C5).

**Becomes testable:** Allocate max_channels numbers; confirm failure on overflow. Release; confirm reuse.

**Risk: LOW**

---

### Step 5.2 — `chanmgr/channel.odin`

**Implements:**
- `CHANNEL_ID :: 2`, `LISTENER_ID :: 3`
- `Channel` struct (S5-corrected):
  ```odin
  Channel :: struct {
      using poly:    matryoshka.PolyNode,  // offset 0; id = CHANNEL_ID or LISTENER_ID
      number:        types.ChannelNumber,
      state:         types.Channel_State,
      cg_id:         types.ChannelGroupId,
      remote_number: types.ChannelNumber,
      socket:        platform.Socket,     // embedded by value (S5 P-05 fix)
      outbound:      Outbound_Queue,      // embedded (from outbound.odin)
      recv_buf:      [dynamic]u8,         // partial-frame accumulation (S5 P-07 fix)
  }
  ```
- SM2 procedures: `Ch_Allocate`, `Ch_Transition`, `Ch_Free`
- `Ch_Set_Socket`, `Ch_Set_Remote_Number`

**Depends on:**
- Step 2.1 (platform.Socket — embedded by value)
- Step 3.3 (Outbound_Queue — embedded)
- Step 5.1 (Number_Pool — called by Ch_Allocate)
- Phase 0 (types: ChannelNumber, Channel_State, ChannelGroupId)
- Matryoshka (PolyNode at offset 0)

**Reason:** Channel is the core data structure of chanmgr. All protocol and I/O dispatch operates on it. Must be complete before protocol or reactor work starts.

**Ownership rules (P3 Channel):**
- Channel is created and destroyed by Reactor (P3: "Channel — Owner: ChannelGroup, managed by Engine/Reactor")
- Application never reads `Channel.state` directly (P3: invalid state)
- `Ch_Transition` is the ONLY valid state-change path. No direct field assignment.
- SM2 must reject invalid transitions (e.g., Open → Connecting is illegal). Use exhaustive switch with panic on invalid transition (P5 PM-3, DR-4).

**Validation required:**
- Confirm `offset_of(Channel, "poly") == 0`.
- Confirm SM2 rejects all invalid transitions.
- Confirm Ch_Allocate assigns a valid ChannelNumber (not 0).

**Becomes testable:** Channel lifecycle: allocate → transition through states → free. SM2 exhaustive transition tests.

**Risk: MEDIUM** — SM2 correctness is critical. An invalid transition (e.g., from Closing back to Open) would cause indefinite resource leaks. Exhaustive switch with panic (not silent ignore) on invalid transitions.

---

**Phase 5 complete:** `chanmgr` package fully compiles. Channel lifecycle works. PolyNode offset-0 verified for Channel.

---

## Phase 6 — Runtime Messaging Infrastructure

**Goal:** Message routing (Router), wire framing (Framer) are complete. Cross-thread mailbox wiring works.

---

### Step 6.1 — `runtime/framer.odin`

**Implements:**
- `frame_wire_header` struct #packed (total_size, opcode, channel, cg_id, meta_len, body_len)
- `FRAME_HEADER_SIZE :: size_of(frame_wire_header)`
- `Framer_Encode`: writes frame_wire_header + meta bytes + body bytes into a caller-provided `^[dynamic]u8`
- `Framer_Try_Decode`: reads from `[]u8`, populates `^Message`, returns `types.Decode_Result`

**Depends on:**
- Step 1.1 (Message, Header, Appendable)
- Phase 0 (types: Decode_Result, OpCode, ChannelNumber, ChannelGroupId)

**Reason:** Framer is called by `reactor/io_dispatch.odin` on every READ (decode) and WRITE (encode) event. It must be complete before I/O dispatch. Testing framing in isolation catches endianness errors before they cause silent protocol failures.

**Ownership rules:**
- `Framer_Encode` reads from `^Message` — caller retains ownership
- `Framer_Try_Decode` writes into a caller-provided `^Message` — caller owns the Message
- Framer does NOT allocate Messages. It reads/writes Message fields only.

**Validation required:**
- Round-trip: encode a Message, decode from the resulting bytes, verify all fields match.
- Incomplete frame: truncate encoded bytes to n-1; confirm Decode_Result.Incomplete.
- Invalid frame: corrupt header; confirm Decode_Result.Invalid.
- Empty meta/body: confirm zero-length fields encode/decode correctly.

**Becomes testable:** Encode/decode round-trip. Partial-frame detection. Invalid frame rejection.

**Risk: MEDIUM** — Endianness errors in packed struct. Byte-level field layout must exactly match on all platforms. Test with fixed byte sequences.

---

### Step 6.2 — `runtime/router.odin`

**Implements:**
- `Wake_Fn :: #type proc()`
- `CG_Entry` struct (id: ChannelGroupId, mb: matryoshka.Mailbox)
- `Router` struct (reactor_inbox, wake_fn, cg_entries, allocator)
- `Router_Create`, `Router_Register_CG`, `Router_Unregister_CG`
- `Router_Send_Reactor` — `mbox_send` to reactor_inbox THEN call `wake_fn` (ordering is mandatory, LI-8)
- `Router_Drain_Inbox` — `try_receive_batch` on reactor_inbox (non-blocking)
- `Router_Send_App` — `mbox_send` to `cg_entries[id].mb`
- `Router_Wait_App` — `mbox_wait_receive` on `cg_entries[id].mb` (blocking; application thread only)

**Depends on:**
- Phase 0 (types: ChannelGroupId)
- Matryoshka (Mailbox, MayItem)

**Reason:** Router owns all Mailboxes. It must be complete before the Reactor can receive commands or deliver messages. The wake ordering constraint (LI-8) is the single most important correctness invariant in this file.

**Ownership rules (P3 Mailbox):**
- `Router_Send_Reactor`: sender's MayItem becomes nil on success. If Mailbox is draining/closed, MayItem stays non-nil — ownership remains with caller (P3 L-MB2).
- `Router_Wait_App`: application blocks; receives non-nil MayItem on message. Returns Closed when Engine is draining (P3 L-MB3). Application MUST handle Closed.
- `Router_Drain_Inbox`: Reactor-only. Non-blocking (C7: Reactor must not call mbox_wait_receive).

**Critical ordering validation:**
- Verify that `Router_Send_Reactor` calls `mbox_send` before `wake_fn`. This ordering must be enforced in the procedure body — it is a correctness invariant (P4 V-MAR2 consequence).

**Becomes testable:** Send from app thread, drain from reactor thread. Register/unregister CG. Closed-mailbox behavior (send fails, ownership returned). Wake ordering (send before notify).

**Risk: MEDIUM** — Wake ordering bug (notify before send) is a race condition that causes Reactor to miss messages. Silent failure under low concurrency; intermittent failure under load.

---

**Phase 6 complete:** L3 messaging infrastructure is validated. Framing works. Cross-thread routing works.

**Parallel opportunity:** Steps 6.1 and 6.2 have no interdependency. Write in parallel.

---

## Phase 7 — Reactor Support Structures

**Goal:** The three reactor-internal data structures that the event loop depends on are complete and independently tested.

---

### Step 7.1 — `reactor/tc_pool.odin`

**Implements:**
- `TC_ID :: 4`
- `TC` struct:
  ```odin
  TC :: struct {
      using poly:  matryoshka.PolyNode,  // offset 0; id = TC_ID
      seq:         types.SequenceNumber,
      channel_num: types.ChannelNumber,
      flags:       types.Trigger_Flags,
      tc_state:    types.TC_State,
  }
  ```
- `TC_Pool` struct (pool: matryoshka.Pool, allocator: mem.Allocator, seq_next: SequenceNumber)
- `TC_Pool_Get` — calls `pool_get(.Available_Or_New)`; assigns monotonic seq_next; returns `^TC`
- `TC_Pool_Put` — calls `pool_put`; ONLY valid when `tc.tc_state == .Deregistered` (P4 V-TC1, C6)
- `on_get` hook: allocate new TC or zero existing TC fields; set `poly.id = TC_ID`
- `on_put` hook: leave m^ non-nil (pool retains)

**Depends on:**
- Phase 0 (types: SequenceNumber, ChannelNumber, Trigger_Flags, TC_State)
- Matryoshka (PolyNode at offset 0, Pool)

**Reason:** TC is the Poller registration token. Its stable heap address is a hard requirement (P4 L-TC1). Must be validated before io_dispatch can use it.

**Ownership rules (P3 TriggeredChannel):**
- Must be heap-allocated (L-TC1). Pool.put() does not free — keeps at heap address.
- `TC_Pool_Put` must ONLY be called after `tc_state == .Deregistered` (L-TC2, L-TC3, C6)
- SequenceNumber assigned at `TC_Pool_Get` is monotonic. seq_next must not wrap to a previously-used value within an Engine session.

**Validation required:**
- Confirm `offset_of(TC, "poly") == 0`.
- Confirm TC pointer is stable: `TC_Pool_Get`, store pointer, `TC_Pool_Put`, `TC_Pool_Get` again — same heap address returned (P4 pointer stability).
- Confirm `TC_Pool_Put` with `tc_state != .Deregistered` panics (in debug builds).

**Becomes testable:** TC allocation/recycling. Pointer stability. Monotonic seqn.

**Risk: HIGH** — PolyNode offset-0 for TC is critical (same risk as Message in Step 1.1). Pointer stability is the hard invariant — heap address must not change between pool_get and pool_put.

---

### Step 7.2 — `reactor/dual_map.odin`

**Implements:**
- `Dual_Map` struct (by_seqn: map[SequenceNumber]^TC, by_chan: map[ChannelNumber]SequenceNumber)
- `Dual_Map_Insert` (seqn, chan_num, ^TC — Phase 5 only)
- `Dual_Map_Lookup_Seqn` (seqn → ^TC or nil — Phase 6 only)
- `Dual_Map_Lookup_Chan` (chan_num → SequenceNumber or 0 — Phase 8 only)
- `Dual_Map_Remove` (seqn + chan_num — Phase 8 only)

**Depends on:**
- Step 7.1 (TC — map value type)
- Phase 0 (types: SequenceNumber, ChannelNumber)

**Reason:** Dual_Map is the ABA guard for Poller events. Phase 6 of the reactor loop resolves ALL events through this map before Phase 7 dispatches any. Must be complete before reactor.odin.

**Ownership rules:**
- Dual_Map is Reactor-exclusive. No locks. No cross-thread access (P7 AI-3).
- ALL inserts happen in Phase 5. ALL removes happen in Phase 8. Phases 6 and 7 are read-only (P6 LI-2).
- `Dual_Map_Lookup_Seqn` returning nil means the event is stale (ABA). Caller must discard the event.

**Validation required:**
- Insert, lookup by seqn → returns TC pointer.
- Insert, lookup by chan → returns seqn.
- Remove, lookup by seqn → returns nil (stale event discarded).

**Becomes testable:** Insert/lookup/remove cycle. ABA guard: after remove, lookup returns nil.

**Risk: LOW** — Simple hash map operations. The correctness constraint (Phase 5/8 only mutation) is architectural, enforced by the reactor loop structure, not by the dual_map code itself.

---

### Step 7.3 — `reactor/timeout.odin`

**Implements:**
- `Timeout_Manager` struct (deadlines: map[ChannelNumber]i64)
- `Timeout_Set` (chan_num, deadline_ms i64 — overwrites any existing entry)
- `Timeout_Clear` (chan_num — removes entry; no-op if not present)
- `Timeout_Next_Ms` (returns minimum deadline across all entries; 0 if empty)
- `Timeout_Collect_Expired` (returns []ChannelNumber with deadlines <= now_ms; removes from map)

**Depends on:**
- Phase 0 (types: ChannelNumber)

**Reason:** Phase 1 of the reactor loop calls `Timeout_Next_Ms` to compute the poll timeout. Phase 4 calls `Timeout_Collect_Expired`. These must be correct before the reactor loop is written.

**Ownership rules:**
- Timeout_Manager is Reactor-exclusive. No locks. No application thread access.
- `Channel.deadline_ms` field was removed (S5 P-06). Timeout_Manager is the single source of truth for deadlines.
- `Timeout_Set` is called by `io_dispatch.odin` after Channel state transitions: Connecting → set connect_timeout_ms; Handshaking → set handshake_timeout_ms; Closing → set bye_timeout_ms.
- `Timeout_Clear` is called when the corresponding state resolves (e.g., Connecting → Handshaking clears the connect timeout, sets the handshake timeout).

**Becomes testable:** Set/clear/next; expired collection; multiple entries (min selection).

**Risk: LOW**

---

**Phase 7 complete:** All reactor support structures are independently validated. Dual_Map, TC_Pool, Timeout_Manager are correct before the event loop uses them.

**Parallel opportunity:** Steps 7.1, 7.2 are sequential (7.2 needs TC from 7.1). Step 7.3 is independent and can be written in parallel with 7.1.

---

## Phase 8 — Protocol Layer

**Goal:** Full protocol dispatch and handshake sequences are correct and independently testable — without any Reactor or OS I/O.

---

### Step 8.1 — `protocol/protocol.odin`

**Implements:**
- `Protocol_Context` struct (channel: ^chanmgr.Channel, router: ^runtime.Router, reserved_pool: ^runtime.Reserved_Pool, allocator: mem.Allocator)
- `Protocol_Dispatch_Inbound` — exhaustive OpCode switch on decoded Message; routes to handlers
- `Protocol_Dispatch_Command` — exhaustive switch on command Message (Connect, Listen, Close, Drain opcodes)
- All unrecognized OpCodes → panic (P5 PM-3, P7 DR-4: exhaustive switch with panic default)

**Depends on:**
- Step 5.2 (chanmgr.Channel)
- Step 3.2 (runtime.Reserved_Pool)
- Step 6.2 (runtime.Router)
- Phase 0 (types: OpCode)

**Reason:** Protocol layer (L4) must be compiled before `reactor/io_dispatch.odin` (Phase 9). Testing it independently confirms correct OpCode routing without needing a running Reactor.

**Ownership rules:**
- `Protocol_Context` does NOT contain Timeout_Manager or Dual_Map — those are reactor-internal (S5 P-02 fix). Protocol calls only chanmgr and runtime.
- Protocol calls `Ch_Transition` to advance Channel state. It does NOT set timeouts — that is `io_dispatch.odin`'s responsibility (S2 boundary rule).
- All outbound Messages (protocol responses) come from the reserved pool, not the application pool (P4 V-M2: Reactor must not call pool_get on app pool).

**Validation required:**
- Confirm Protocol_Dispatch_Inbound routes each OpCode to the correct handler.
- Confirm unrecognized OpCode panics (not silently ignored).
- Confirm Protocol_Context has no Timeout_Manager field (S5 P-02 check).

**Becomes testable:** OpCode routing with mock Protocol_Context. Panic on unknown OpCode.

**Risk: MEDIUM** — Exhaustive switch correctness is critical. A missing OpCode case causes silent message drops if a panic-default is not in place.

---

### Step 8.2 — `protocol/handshake.odin`

**Implements:**
- `Bye_Role` enum (Initiator, Responder)
- Hello sequence handler (initiator: send Hello; listener: receive Hello → send Welcome)
- Welcome sequence handler (initiator receives Welcome → transition to Open)
- Bye sequence handler:
  - Initiator: send Bye → wait for Bye_Ack → close
  - Responder: receive Bye → send Bye_Ack → close
  - Simultaneous-Bye tiebreaker: lower `channel.number` = Responder (sends Bye_Ack immediately)
- `Ch_Set_Remote_Number` called during Welcome exchange

**Depends on:**
- Step 8.1 (Protocol_Context)
- Step 5.2 (chanmgr.Channel, Ch_Transition, Ch_Set_Remote_Number)
- Step 6.2 (runtime.Router, Router_Send_App)

**Reason:** Handshake is the most protocol-critical code. The simultaneous-Bye tiebreaker (S2 decision) must be explicitly tested before the Reactor wires it into the event loop.

**Ownership rules:**
- `Channel_Closed` notification Message comes from reserved pool, not application pool
- `Router_Send_App` must be called with a reserved-pool Message — application pool may be exhausted
- `Channel_Closed` notification is sent BEFORE ChannelNumber is released (AI-5, P3 L-C5)

**Critical rule — simultaneous-Bye tiebreaker (S2):**
- Both sides receive each other's Bye in the same reactor iteration
- Side with lower `channel.number` acts as Responder: sends Bye_Ack immediately
- Side with higher `channel.number` acts as Initiator: transitions to Closing and waits

**Validation required:**
- Hello→Welcome round-trip (mock both sides)
- Bye→Bye_Ack sequence (initiator path)
- Simultaneous-Bye: both sides send Bye; lower ChannelNumber sends Bye_Ack; higher waits
- Verify `Channel_Closed` notification is sent before ChannelNumber release

**Becomes testable:** Complete handshake sequence tests. Simultaneous-Bye tiebreaker.

**Risk: MEDIUM** — Simultaneous-Bye tiebreaker is a two-party edge case. Easy to implement asymmetrically. Test both sides in the same test with deterministic ChannelNumbers.

---

**Phase 8 complete:** L4 protocol layer is fully validated. Handshake sequences are correct without a running Reactor. The simultaneous-Bye tiebreaker is tested.

---

## Phase 9 — I/O Dispatch

**Goal:** `reactor/io_dispatch.odin` is complete. This is the integration point between all layers (L1, L2, L3, L4). It is the most complex single procedure in the codebase.

---

### Step 9.1 — `reactor/io_dispatch.odin`

**Implements:**
- `io_result` struct (private: pending_close: bool, error: Engine_Error)
- `io_dispatch_call` (main entry: takes ^TC, ^Channel, Trigger_Flags, Dispatch_Context → io_result)
- READ handler: `Socket_Recv` into `channel.recv_buf`; call `Framer_Try_Decode`; if complete frame → call `Protocol_Dispatch_Inbound`
- WRITE handler: if Channel not yet connected → `Socket_Connect_Complete`; else `Ch_Dequeue_Outbound`, cast MayItem to `^Message`, `Framer_Encode`, `Socket_Send`
- ACCEPT handler: `Socket_Accept` → allocate new Channel + TC → `Dual_Map_Insert` (note: insert in Phase 5 context; accept-produced channels inserted during Phase 7 but wait for next iteration per P6 R5.2)
- ERROR/HUP handler: initiate Channel close → add to pending_close
- After state transition: call `Timeout_Set` or `Timeout_Clear` based on resulting `Channel_State`

**Depends on:**
- Step 7.1 (TC)
- Step 7.2 (Dual_Map — for ACCEPT channel insert)
- Step 7.3 (Timeout_Manager — set/clear after transitions)
- Step 8.1 (Protocol_Dispatch_Inbound)
- Step 5.2 (Channel, Ch_Transition)
- Step 6.1 (Framer_Encode, Framer_Try_Decode)
- Step 6.2 (Router_Send_App)
- Steps 4.1-4.3 (Socket_Recv, Socket_Send, Socket_Accept, Socket_Connect_Complete)
- Phase 0 (types: Trigger_Flags, Engine_Error, Channel_State)

**Reason:** io_dispatch integrates every layer. It must come after all its dependencies are validated independently. Testing it requires mock platform, mock protocol, mock chanmgr — all of which are already written by this point.

**Ownership rules (P3 — cross-cutting):**
- After `Socket_Recv`, bytes are in `channel.recv_buf` (owned by Channel, Reactor-exclusive)
- After `Framer_Try_Decode` succeeds: the decoded Message must come from `reserved_pool` (Reactor-internal allocation); application pool is not used for inbound decoding
- After `Framer_Encode`: the encoded bytes are in a caller-provided buffer, not owned by io_dispatch
- ACCEPT-produced socket from `Socket_Accept`: if no Channel is available to attach it to, call `Socket_Close` immediately (P3 L-S5)
- After io_dispatch returns `pending_close = true`: Channel enters Phase 8 of the reactor loop

**Dispatch context (S5 P-02 fix):**
- `dispatch_context` (lowercase, package-private) is constructed in `reactor.odin` and passed to `io_dispatch_call`
- io_dispatch constructs `protocol.Protocol_Context` from `dispatch_context` before calling `Protocol_Dispatch_Inbound`
- `Protocol_Context` does NOT contain Timeout_Manager or Dual_Map

**Becomes testable:** Unit test each handler (READ/WRITE/ACCEPT/ERROR) with mock socket and mock channel state. Verify timeout set/clear after each state transition.

**Risk: HIGH** — io_dispatch is the largest single file in the reactor package. It calls 5+ packages. Phase ordering bugs (e.g., inserting into dual_map during Phase 7 instead of Phase 5 for ACCEPT channels) are subtle. Test each handler in isolation before integration.

---

**Phase 9 complete:** Full I/O dispatch is validated per handler. All layer interactions are tested.

---

## Phase 10 — Reactor Event Loop

**Goal:** The complete 9-phase event loop runs on a dedicated thread. Engine can be created and destroyed.

---

### Step 10.1 — `reactor/reactor.odin`

**Implements:**
- `reactor_state` struct (S5-corrected):
  ```odin
  reactor_state :: struct {
      poller, notifier, tc_pool, reserved_pool, router,
      dual_map, timeout_mgr, number_pool,
      channels: [dynamic]^chanmgr.Channel,  // S5 P-04 fix
      io_events, resolved, pending_close,
      eng_state: ^types.Engine_State,       // S5 P-01 fix
      options:   types.Engine_Options,      // copied by value at startup
      allocator: mem.Allocator,
  }
  ```
- `Reactor_Start` procedure:
  - First statement: `context.allocator = rs.allocator` (MR-2)
  - STARTUP: S1-S9 (init poller, notifier, tc_pool, reserved_pool, router, dual_map, etc.)
  - Signal engine: set `eng_state^ = .Running`
  - LOOP: Phase 1 through Phase 9, sequential, no skipping
  - On loop exit: close reserved_pool, close poller, close notifier; signal `eng_state^ = .Destroyed`
- All 9 loop phase procedures (private): `phase1_compute_timeout` through `phase9_drain_check`
- Phase 8: unconditional close order: Poller_Deregister → Socket_Close → TC_Pool_Put → Router_Send_App (Channel_Closed) → Ch_Release_Number

**Depends on:** ALL previous steps.

**Reason:** Reactor is the capstone of all L2 work. Every prior step was prerequisite.

**Ownership rules (P3 — Reactor lifecycle):**
- First statement: `context.allocator = engine.explicit_allocator` (MR-2, TH-1)
- Reactor.start() = engine_internal.allocator (P7 AI-13: caller owns allocator; Engine holds non-owning reference)
- Phase 8 close order is UNCONDITIONAL (P6 LI-5): deregister, then close(fd), then pool_put. Any other order is a correctness violation.
- `Channel_Closed` notification sent BEFORE `Ch_Release_Number` (AI-5)
- Drain check (Phase 9): `len(channels) == 0 AND inbox empty AND eng_state == .Draining`

**S5 corrections applied here:**
- `reactor_state.engine_state` removed; replaced by `^types.Engine_State` (P-01 fix)
- `reactor_state.reserved_pool` is the ONLY reserved pool (removed from engine_internal) (P-03 fix)
- `reactor_state.channels` is the primary channel collection (P-04 fix)
- `dispatch_context` is private (lowercase); not passed to protocol (P-02 fix)

**Validation required:**
- Engine starts: `eng_state` transitions Starting → Running
- Engine drains: drain signal received; Reactor processes all remaining work; `eng_state` transitions Draining → Destroyed
- Phase ordering: insert in Phase 5, NO mutations in Phase 6/7, remove in Phase 8 (LI-2)
- Thread context: `context.allocator` is engine allocator on Reactor thread

**Becomes testable:** Create Engine → send Connect command → receive Channel_Closed → destroy Engine. Single-threaded test with real sockets. Phase ordering with debug instrumentation.

**Risk: HIGH** — Phase ordering violations (AI-2: mutations in wrong phase), drain sequence errors, and context.allocator discipline (MR-2) are all high-risk correctness issues that manifest as rare/intermittent bugs.

---

**Phase 10 complete:** Reactor event loop runs end-to-end on a real OS thread with real sockets.

---

## Phase 11 — Public API

**Goal:** Application-facing API is complete. Opaque handles. Clean error contracts. No internal types exposed.

---

### Step 11.1 — `otofu/engine.odin`

**Implements:**
- `Engine :: distinct ^engine_internal`
- `engine_internal` struct (message_pool, router, options, allocator, state) — NOTE: no reserved_pool (S5 P-03 fix)
- `Engine_Create` (allocator, options → Engine, Engine_Error):
  - MR-3: `ok := false; defer if !ok { cleanup all }`
  - Initialize message_pool, router
  - Call `reactor.Reactor_Start(engine_ref, allocator, options)` on new thread
  - Wait for `eng_state == .Running`
  - Set `ok = true`
- `Engine_Destroy` (Engine):
  - Signal drain: post Drain command to reactor_inbox; call wake_fn
  - Wait for `eng_state == .Destroyed` (Reactor thread join)
  - Close message_pool (P4 Teardown Order: pool first, then mailboxes)
  - Close Router mailboxes
  - Free engine_internal

**Depends on:**
- Step 10.1 (Reactor_Start)
- Step 6.2 (Router)
- Step 3.1 (Message_Pool)
- Phase 0 (types: Engine_Options, Engine_Error, Engine_State)

**Reason:** Engine is the lifecycle owner of all internal systems. It is the correct point to wire teardown ordering.

**Ownership rules (P3 Engine):**
- Caller owns the allocator; Engine holds a non-owning reference (P7 AI-13, P3 L-E3)
- Engine must outlive all ChannelGroups (P3 L-E1)
- `Engine_Destroy` is blocking: caller must not release allocator until Destroy returns (P3 L-E2)
- Teardown order is mandatory (P4): pool_close → pool drain → matryoshka_dispose; mbox_close → mbox drain → matryoshka_dispose; join Reactor thread; free engine_internal (Steps 1-8 of P4 teardown)
- `Engine_Destroy` called twice → must be no-op or explicit error (P3: invalid state)

**Becomes testable:** Engine create/destroy cycle. Allocator lifetime rule (create Engine, free allocator early → crash; correct ordering → clean).

**Risk: MEDIUM** — Teardown ordering. Allocator lifetime. The `ok` defer-guard pattern (MR-3) is more verbose than Zig's errdefer; easy to miss a cleanup path.

---

### Step 11.2 — `otofu/channel_group.odin`

**Implements:**
- `CG :: distinct ^cg_internal`
- `cg_internal` struct (id: ChannelGroupId, router: ^runtime.Router)
- `CG_Create` (Engine → CG, Engine_Error): allocate cg_internal; call `Router_Register_CG`
- `CG_Destroy` (Engine, CG): call `Router_Unregister_CG`; free cg_internal
- `CG_Post` (CG, ^MayItem → Engine_Error): encode opcode; call `Router_Send_Reactor`
- `CG_Wait_Receive` (CG, ^MayItem, timeout_ms → Recv_Result): call `Router_Wait_App`
- `CG_Connect` (CG, Address → Engine_Error): allocate Message from app pool; set opcode=Connect; encode Address in meta; call `CG_Post`
- `CG_Listen` (CG, Address → Engine_Error): same but opcode=Listen

**Depends on:**
- Step 11.1 (Engine, engine_internal)
- Step 6.2 (Router, Router_Register_CG, Router_Send_Reactor, Router_Wait_App)
- Phase 0 (types: ChannelGroupId, Recv_Result)

**Reason:** CG depends on Engine for the Router reference. Must follow 11.1.

**Ownership rules (P3 ChannelGroup):**
- CG handle is non-owning: Engine owns the ChannelGroup; application holds a handle (P3: "shared ownership warning")
- Application must stop all CG calls BEFORE `Engine_Destroy` (P3 L-CG2)
- Exactly one application thread calls `CG_Wait_Receive` on any given CG (P3 L-CG3)
- `CG_Post` on draining Engine must fail and return ownership to caller (P3 L-MB2): if `Router_Send_Reactor` returns error, `m^` must still be non-nil

**Becomes testable:** CG create/destroy. Post → drains to reactor inbox. WaitReceive → blocks until message or timeout. Post on draining Engine → ownership returned.

**Risk: LOW** — CG is a thin wrapper over Router operations.

---

### Step 11.3 — `otofu/message.odin`

**Implements:**
- `Engine_Get` (Engine, Allocation_Strategy, ^MayItem → Engine_Error): call `MP_Get`; no new types
- `Engine_Put` (Engine, ^MayItem): call `MP_Put`
- `Msg_Set_Opcode`, `Msg_Read_Opcode`, `Msg_Set_Channel`, `Msg_Read_Channel`, `Msg_Set_Id`, `Msg_Read_Meta`, `Msg_Write_Meta`, `Msg_Write_Body`, `Msg_Body_Slice`: cast `m^ (^PolyNode)` to `^runtime.Message`; access Header fields

**Depends on:**
- Step 11.1 (Engine, engine_internal.message_pool)
- Step 1.1 (runtime.Message, Header, Appendable)

**Reason:** Message accessors are the final application surface. They depend on the Message struct being finalized and the Engine providing the pool.

**Ownership rules (P3):**
- Every `Engine_Get` that returns non-nil must be paired with one `Engine_Put` or `CG_Post` (L-M1)
- `defer Engine_Put(e, &m)` placed BEFORE `Engine_Get` call (P4 C8)
- Application must not access meta/body Appendable buffers after `CG_Post` or `Engine_Put` (L-M5)
- `Msg_Body_Slice` returns a `[]u8` slice into the Message's body buffer. The application must not retain this slice across `CG_Post` or `Engine_Put`.

**Becomes testable:** Get → set fields → post. Get → put (double-put safety). Get → access body slice → verify contents.

**Risk: LOW**

---

**Phase 11 complete:** Full public API is available. Applications can create engines, post messages, and receive replies.

**Parallel opportunity:** Steps 11.2 and 11.3 can be written in parallel after 11.1.

---

## Phase 12 — Integration & Hardening

**Goal:** End-to-end correctness, concurrency behavior, and edge cases validated.

---

### Step 12.1 — Basic Echo Test

**Implements:** Two Engines on the same machine. Client CG_Connect → server receives Hello → responds with Welcome → both sides open → Data round-trip → CG_Post Bye → graceful close.

**Validates:** Full message lifecycle from application layer to wire and back.

**Risk: MEDIUM** — First integration test. Expect timing-dependent failures.

---

### Step 12.2 — Multi-CG Test

**Implements:** One Engine, multiple CGs, each with independent channels. Verify channel multiplexing.

**Validates:** ChannelNumber assignment and recycling (P3 L-C4, L-C5). CG isolation.

**Risk: LOW**

---

### Step 12.3 — Drain Sequence Test

**Implements:** Engine_Destroy while channels are open. Verify:
- Drain command processed
- All channels receive Channel_Closed
- All Mailboxes return Closed to waiting receivers
- Engine_Destroy blocks until Reactor exits
- No memory leaks

**Validates:** P4 teardown order; Phase 9 drain-check; `eng_state` transition.

**Risk: HIGH** — Drain sequence interacts with all ownership rules simultaneously.

---

### Step 12.4 — Backpressure Test

**Implements:** Fill one channel's outbound queue to `outbound_queue_depth` limit. Verify `BackpressureExceeded` returned. Release items. Verify queue accepts again.

**Validates:** Outbound_Queue depth enforcement; BackpressureExceeded error; no other channels affected.

**Risk: LOW**

---

### Step 12.5 — Reserved Pool Validation (R6.3 Deadlock Prevention)

**Implements:** Exhaust application message pool (all messages owned by application). Trigger Channel_Closed event. Verify notification is delivered from reserved pool.

**Validates:** P4 V-M3 fix; Reserved_Pool isolation; R6.3 deadlock prevention.

**Risk: MEDIUM** — Requires precise pool exhaustion timing.

---

### Step 12.6 — Simultaneous-Bye Test

**Implements:** Both sides post Bye in the same reactor iteration. Verify tiebreaker: lower ChannelNumber sends Bye_Ack; higher waits.

**Validates:** S2 simultaneous-Bye rule; handshake.odin Bye_Role logic.

**Risk: MEDIUM**

---

### Step 12.7 — Timeout Tests

**Implements:** Connect to unreachable address → verify connect_timeout_ms fires → Channel transitions to Closed → Channel_Closed delivered.

**Validates:** Timeout_Manager correctness; Phase 1 and Phase 4 interaction; timeout-triggered close sequence.

**Risk: MEDIUM**

---

### Step 12.8 — Allocator Discipline Audit

**Implements:** Run all tests with a tracking allocator. Verify:
- No `context.allocator` use inside otofu (MR-1)
- All allocations traceable to engine's explicit allocator
- No memory leaks after Engine_Destroy

**Validates:** MR-1, MR-2, P7 AI-13.

**Risk: LOW** (audit, not implementation)

---

## Critical Path Summary

```
Step 0.1 (errors)
  └→ Step 0.3 (identifiers)
       └→ Step 1.1 (Message — PolyNode offset-0)  ← CRITICAL CHECKPOINT
            └→ Step 3.1 (Message_Pool)
                 └→ Step 6.1 (Framer)
                      └→ Step 9.1 (io_dispatch) ←──────────────────────────────┐
                                                                                 │
Step 0.6 (options) ──────────────────────────────────────────────────────────── │
  └→ Step 2.1 (Socket struct)                                                   │
       └→ Step 4.1 (socket_unix.odin — Socket procs)                           │
            └→ Step 4.3 (poller_linux.odin)                                     │
                 └→ Step 7.1 (TC_Pool — PolyNode offset-0) ← CRITICAL CHECKPOINT│
                      └→ Step 7.2 (Dual_Map) ──────────────────────────────────►Step 9.1
                                                                                 │
Step 5.2 (Channel SM2) ──────────────────────────────────────────────────────► │
  └→ Step 8.1 (Protocol_Context)                                                │
       └→ Step 8.2 (Handshake) ──────────────────────────────────────────────► │
                                                                                 │
Step 6.2 (Router) ───────────────────────────────────────────────────────────► │
  └→ Step 9.1 ────────────────────────────────────────────────────────────────►▼
       └→ Step 10.1 (Reactor loop)
            └→ Step 11.1 (Engine)
                 └→ Step 11.2 + 11.3 (CG + Message API)
                      └→ Phase 12 (Integration)
```

**Two critical checkpoints that must PASS before any subsequent work:**
1. `offset_of(Message, "poly") == 0` — after Step 1.1
2. `offset_of(TC, "poly") == 0` AND TC pointer stability — after Step 7.1

---

## Parallel Work Opportunities

These groups have no interdependency and can be worked simultaneously:

| Group | Steps | Can parallelize with |
|-------|-------|---------------------|
| A — types | 0.1 → 0.6 | Each other (all standalone) |
| B — platform structs | 2.1, 2.2, 2.3 | Each other; after Phase 0 |
| C — pool pair | 3.1, 3.3 | Each other (3.2 follows 3.1) |
| D — platform impls (Linux) | 4.1, 4.2, 4.3 | Each other; after Phase 2 |
| E — platform impls (Windows) | 4.4, 4.5 | Group D; developer split |
| F — runtime pair | 6.1, 6.2 | Each other; after Step 1.1 |
| G — reactor support | 7.1, 7.3 | Each other; 7.2 follows 7.1 |
| H — protocol pair | 8.1, then 8.2 | Group G (independent of reactor); 8.2 follows 8.1 |
| I — public API | 11.2, 11.3 | Each other; after 11.1 |
| J — integration tests | 12.1-12.8 | Some can be parallel after 12.1 passes |

**Do NOT parallelize:**
- Step 1.1 and 7.1 critical checkpoints (must verify before continuing)
- Step 9.1 (io_dispatch) before all its dependencies are complete
- Step 10.1 (Reactor) before 9.1
- Step 11.1 (Engine) before 10.1

---

## High-Risk Areas

| Risk | Step | Issue | Guard |
|------|------|-------|-------|
| **CRITICAL** | 1.1 | `Message.poly` not at offset 0 → all pool operations silently corrupt | `offset_of(Message, "poly") == 0` assertion + cast round-trip test |
| **CRITICAL** | 7.1 | `TC.poly` not at offset 0 → Poller events misrouted | `offset_of(TC, "poly") == 0` + pointer stability test |
| **HIGH** | 10.1 | Phase ordering violation: dual_map mutated in Phase 6 or 7 | Debug instrumentation: log which phase each dual_map call occurs in |
| **HIGH** | 10.1 | `context.allocator` not set in Reactor thread before any allocation | MR-2: first statement in `Reactor_Start` must be context override |
| **HIGH** | 9.1 | Stale event not discarded: dual_map returns nil but dispatch proceeds | Phase 6: check for nil in resolved[]; skip nil entries in Phase 7 |
| **HIGH** | 4.5 | Windows TC holds kernel IO_STATUS_BLOCK pointer; freed too early | P4 V-TC2: `on_put` hook must verify no outstanding AFD operation |
| **MEDIUM** | 3.1/3.2 | Hook reentrancy: `on_put` calls `MP_Put` on same pool | P4 C4: hooks must never call pool_get/pool_put on the same pool |
| **MEDIUM** | 8.2 | Simultaneous-Bye tiebreaker inverted (both sides act as Responder or both as Initiator) | Unit test with deterministic ChannelNumbers where both sides receive each other's Bye |
| **MEDIUM** | 6.2 | Wake ordering: `Notifier_Notify` called before `mbox_send` | LI-8: message must be in inbox before wake is sent. Review every call to `Router_Send_Reactor` |
| **MEDIUM** | 10.1 | Drain exits before all Channel_Closed notifications consumed | Phase 9 check must include Router in_flight count (S5 P-09 — optional hardening) |
| **MEDIUM** | 6.1 | Endianness errors in `frame_wire_header` #packed | Cross-platform encode→decode test with fixed byte sequence |
| **MEDIUM** | 11.1 | Teardown order wrong: mailboxes closed before pool | P4 teardown order: pool_close first, then mbox_close |

---

## Early Failure Detection Points

These steps will surface design errors immediately:

| Step | Design error detected |
|------|-----------------------|
| 1.1 | PolyNode offset-0 constraint feasibility in Odin's struct layout |
| 3.1 | Matryoshka Pool hook contract (on_get/on_put can't call pool back) |
| 4.3 | SequenceNumber embeds in epoll_data.u64 without loss on 32-bit |
| 5.2 | SM2 transition table exhaustiveness — missing transitions discovered |
| 6.2 | Wake ordering constraint: `mbox_send` before `wake_fn` in the same procedure |
| 7.1 | TC pointer stability assumption — Matryoshka Pool must not move heap objects |
| 8.2 | Simultaneous-Bye tiebreaker: requires both sides to have exchanged ChannelNumbers before Bye can be compared |
| 9.1 | ACCEPT handler: new channels inserted during Phase 7 must NOT be in Phase 6's resolved[] (R5.2 from P6) |
| 10.1 | Drain: `eng_state^` shared between two threads without atomic/mutex — Odin's `atomic` package required |

---

## Notes on `eng_state` Thread Safety

Step 10.1 uses `eng_state: ^types.Engine_State` — a pointer to a field in `engine_internal` shared between the application thread (Engine_Create/Destroy) and the Reactor thread (transitions Starting→Running and Draining→Destroyed).

This is a shared mutable variable. Two options:
1. Use Odin's `atomic` package: `atomic.load/store(&eng_state, .Running)`. Simple; no mutex.
2. Use a condition variable or OS event: Engine_Create blocks until `sync.Cond` signals Running.

Recommendation: Use `sync.Cond` (wait/signal) for the startup synchronization (Engine waits for Reactor to signal Running). Use atomic store/load for the drain state check (Reactor stores Destroyed; Engine checks after thread join). This avoids busy-wait.

This is an implementation detail not decided in the architecture documents — it must be resolved at Step 10.1 before coding starts.
