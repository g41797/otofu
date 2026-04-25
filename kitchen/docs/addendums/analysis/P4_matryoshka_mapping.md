# P4 Matryoshka Mapping

Sources: P1_primitives.md, P3_ownership.md
Matryoshka: SRC_MATRYOSHKA_REPO, SRC_MATRYOSHKA_DOCS (Dolls 1–4)

---

## Mapping Classifications

| Symbol | Meaning |
|--------|---------|
| `DIRECT` | Full representation. No adaptation needed. |
| `PARTIAL` | Usable with constraints or gaps. Documented per primitive. |
| `IMPOSSIBLE` | Cannot be represented. Fundamental incompatibility. |
| `N/A` | Value type or external handle. No Matryoshka mapping required. |

---

## Summary Table

| Primitive | Category | Matryoshka Component | Mapping |
|-----------|----------|---------------------|---------|
| Message | Core | PolyNode + Pool | `DIRECT` |
| MessagePool | Derived | Pool | `DIRECT` |
| Mailbox (Reactor→App) | Core | Mailbox | `DIRECT` |
| Mailbox (App→Reactor) | Core | Mailbox (send only) | `PARTIAL` |
| TriggeredChannel | Derived | PolyNode + Pool | `DIRECT` |
| Engine | Core | Master (conceptual) | `DIRECT` |
| ChannelGroup | Derived | Master reference (conceptual) | `PARTIAL` |
| Channel | Core | PolyNode (list node only) | `PARTIAL` |
| Reactor | Derived | Special Master | `PARTIAL` |
| Notifier | Derived | — | `IMPOSSIBLE` |
| Socket | Core | — | `IMPOSSIBLE` |
| Poller | Derived | — | `IMPOSSIBLE` |
| Allocator | Core | `mem.Allocator` (native) | `N/A` |
| Address | Core | Value struct | `N/A` |
| Header | Artifact | Embedded value in Message | `N/A` |
| MetaHeaders | Artifact | Appendable (owns via Message) | `PARTIAL` |
| Body | Artifact | Appendable (owns via Message) | `PARTIAL` |
| ChannelNumber | Artifact | Embedded value | `N/A` |
| SequenceNumber | Artifact | Embedded value | `N/A` |
| TriggerFlags | Artifact | Value type | `N/A` |

The critical boundary: Message and Mailbox are fully covered by Matryoshka.
Socket, Poller, and Notifier are not — they are OS-level and must be user code.

---

## Doll 1 — PolyNode + MayItem

The foundation. Every mapping in this document rests on this.

### What PolyNode provides

- Intrusive link (`prev`, `next`) embedded at offset 0.
- Type discriminator (`id`, must be != 0).
- Single-list-membership guarantee: one `prev`, one `next`. An item cannot be in two lists simultaneously.
- No allocation: just a link and a tag.

### What MayItem provides

- `m^ != nil` — you own this item. You must transfer, recycle, or dispose it.
- `m^ == nil` — you do not own it.
- `m == nil` — nil handle. Programming error.
- Enforced by convention at every API boundary. Odin has no borrow checker.

### Offset 0 rule

Every otofu struct that travels through Matryoshka infrastructure MUST embed `PolyNode` via `using` as its **first field**. The cast `(^YourStruct)(node)` is valid only at offset 0. Matryoshka has no compile-time check. This is a convention you enforce.

---

## Primitive: Message

**Classification: `DIRECT`**

### Representation in Matryoshka

Message embeds `PolyNode` at offset 0.

```
Message :: struct {
    using poly: PolyNode,   // offset 0 — required
    header:     Header,     // fixed-size embedded value
    meta:       Appendable, // variable MetaHeaders buffer
    body:       Appendable, // variable Body buffer
    // ...
}
```

`PolyNode.id` is set to a fixed constant (e.g., `MessageId :: 1`) at allocation.

`MayItem` (`Maybe(^PolyNode)`) is the ownership token at every call site:

| otofu operation | Matryoshka operation | MayItem after |
|-----------------|---------------------|---------------|
| `Engine.get(always)` | `pool_get(pool, MessageId, .Available_Or_New, &m)` | non-nil |
| `Engine.get(poolOnly)` | `pool_get(pool, MessageId, .Available_Only, &m)` | non-nil or nil |
| `Engine.put(&m)` | `pool_put(pool, &m)` | nil |
| `ChannelGroup.post(&m)` | `mbox_send(mb, &m)` | nil on Ok |
| `ChannelGroup.waitReceive()` | `mbox_wait_receive(mb, &m)` | non-nil on Ok |

### AllocationStrategy → Pool_Get_Mode

| otofu AllocationStrategy | Matryoshka Pool_Get_Mode |
|--------------------------|--------------------------|
| `poolOnly` | `.Available_Only` — no `on_get`, no allocation |
| `always` | `.Available_Or_New` — `on_get` creates if pool empty |

Exact match. No adaptation needed.

### PoolHooks — on_get and on_put

`on_get` (called on every `pool_get` except `Available_Only`):

```
if m^ == nil:
    allocate new Message struct via ctx.allocator
    set poly.id = MessageId
    set m^ = ^poly
else:
    reset existing Message for reuse:
        clear Header fields
        reset MetaHeaders Appendable (keep capacity, clear content)
        reset Body Appendable (keep capacity, clear content)
```

`on_put` (called outside pool mutex on every `pool_put`):

```
if in_pool_count > POOL_MAX_MESSAGES:
    destroy MetaHeaders Appendable buffer
    destroy Body Appendable buffer
    free Message struct via ctx.allocator
    m^ = nil   ← pool discards, does not store
else:
    leave m^ non-nil   ← pool stores
```

### Compatibility

The otofu Message lifecycle maps directly to the Matryoshka Pool get/put cycle:

```
nonexistent ──on_get (m^==nil)──► owned-app
pooled      ──on_get (m^!=nil)──► owned-app   (reset via on_get)
owned-app   ──pool_put──────────► pooled       (reset/kept via on_put)
owned-app   ──mbox_send─────────► owned-engine
owned-engine──mbox_wait_receive─► owned-app
```

`pool_put` with nil `m^` is a no-op. This is exactly the "double-put is safe" invariant from P3 (INV-02).

`defer pool_put(pool, &m)` can be placed immediately after `m: MayItem`, before `pool_get`. On all early-return paths, the defer is a no-op if `m^` is nil (send succeeded or get failed). This is the Matryoshka `[itc: defer-put-early]` idiom. Apply it everywhere.

### Violations

**V-M1 — Appendable buffers are not intrusive.**
MetaHeaders and Body are heap-allocated Appendable buffers inside Message. They do not embed PolyNode. They are not intrusive. This is not a violation of Matryoshka's intrusive model — the Message is the intrusive node; the buffers are owned by the Message. The intrusive model applies to the traveling unit (Message), not to its internal contents.

**V-M2 — `pool_get_wait` is not safe in the Reactor.**
Matryoshka provides `pool_get_wait` to block until a Message is available in the pool. The Reactor must NOT call `pool_get_wait` — the Reactor is the single I/O thread and blocks on Poller, not on a pool. Any Message the Reactor needs must either be pre-reserved or obtained non-blocking (`Available_Only`). Pool exhaustion in the Reactor is an error, not a wait condition.

**V-M3 — Pool exhaustion deadlock (R6.3) is not resolved by standard Pool.**
Standard Matryoshka Pool has no reservation mechanism for system-internal items. The deadlock (application holds all Messages, Reactor cannot deliver `channel_closed` notification) requires a SEPARATE reserved pool for engine-internal Messages. This separate pool is NOT part of standard Matryoshka — it must be initialized with a fixed pre-allocated count and must never be exposed to `pool_get` from application code.

**V-M4 — Hook reentrancy forbidden.**
The `on_get` and `on_put` hooks must NOT call `pool_get` or `pool_put` on the same pool. Calling back into the pool from inside a hook causes silent state corruption. This constraint applies to all otofu hook implementations.

---

## Primitive: MessagePool

**Classification: `DIRECT`**

### Representation in Matryoshka

MessagePool IS a Matryoshka `Pool`.

```
Pool :: ^PolyNode   (Matryoshka handle type)
```

Engine creates the Pool during `starting` state. Engine destroys it (via `pool_close` → `matryoshka_dispose`) during the `draining → destroyed` transition.

Pool teardown order (required by Matryoshka):

```
1. pool_close(pool)         ← returns all stored items as list.List
2. process returned list    ← for each item: reset and free (do NOT re-put; pool is closed)
3. matryoshka_dispose(&m)   ← free pool struct itself
```

### Compatibility

Full match. No adaptation needed beyond the hook implementations described under Message.

### Violations

**V-MP1 — Closed pool with valid id leaves `m^` non-nil (does not panic).**
If application calls `Engine.put(&m)` after Engine has been destroyed (pool is closed), `pool_put` returns with `m^` still non-nil. This is correct Matryoshka behavior. In otofu, this maps to INV-20 (use after Engine destroyed). The returned non-nil `m^` tells the caller that something went wrong — the item is still theirs and they must dispose it. This is NOT a violation; it is the expected contract.

**V-MP2 — Reserved pool for engine-internal Messages.**
A separate Pool must be initialized with a small fixed count of pre-allocated Messages for engine-internal use (`channel_closed` notifications, error delivery). This pool shares the same `on_get`/`on_put` hooks as the application pool but is populated with `New_Only` mode during Engine startup. Its capacity must be bounded and enforced in `on_put`.

---

## Primitive: Mailbox (Reactor → App direction)

**Classification: `DIRECT`**

### Representation in Matryoshka

ChannelGroup's receive Mailbox IS a Matryoshka `Mailbox`.

```
Mailbox :: ^PolyNode   (Matryoshka handle type)
```

Reactor calls `mbox_send(mb, &m)` to deliver a Message to the application.
Application calls `mbox_wait_receive(mb, &m, timeout)` to receive.
Engine calls `mbox_close(mb)` during `draining`.
`matryoshka_dispose` frees the Mailbox struct after close.

### ChannelGroup.waitReceive → mbox_wait_receive

| otofu result | Matryoshka result | Meaning |
|-------------|-------------------|---------|
| message received | `.Ok`, `m^ != nil` | Application now owns Message |
| timeout | `.Timeout`, `m^ == nil` | No message within timeout |
| engine draining | `.Closed`, `m^ == nil` | Mailbox closed — exit receive loop |
| interrupted (future) | `.Interrupted`, `m^ == nil` | Wake without data — check external state |

Application receive loop MUST handle `.Closed`. Not handling it causes infinite block (R6.2).

### Compatibility

Full match. `mbox_interrupt` is available for future use (e.g., priority or OOB signaling). Not required for baseline otofu.

### Violations

**V-MB1 — `mbox_wait_receive` entry guard: `out^ != nil` returns `.Already_In_Use`.**
If application calls `waitReceive` with a non-nil MayItem (previous message not yet put), Matryoshka refuses the call. This is correct behavior — it prevents overwriting an owned item. Application must `put` or `post` the previous Message before calling `waitReceive` again.

**V-MB2 — Batch close returns linked list, not MayItems.**
`mbox_close` returns a `list.List` of `^list.Node`. Items are still linked to each other. The application (or Engine, during drain) must call `polynode_reset(poly)` on each node after `list.pop_front` before passing to `pool_put`. Failing to reset causes the pool's linked-list invariant check to panic in debug builds.

---

## Primitive: Mailbox (App → Reactor direction)

**Classification: `PARTIAL`**

### Representation in Matryoshka

A second Matryoshka Mailbox — the Reactor's inbound queue — carries Messages from application to Reactor.

Application calls `mbox_send(reactor_inbox, &m)` (via `ChannelGroup.post`).
Reactor drains with `try_receive_batch(reactor_inbox)` after waking on Notifier event.

This is the **two-mailbox pattern** from Matryoshka Doll 2 (mb_main + mb_oob), adapted.

### Where the partial mapping begins

In Matryoshka's pattern:
- Sender fills `mb_oob` with data, then calls `mbox_interrupt(mb_main)` to wake the receiver.
- Receiver wakes on `.Interrupted`, then drains `mb_oob` with `try_receive_batch`.

In otofu's Reactor model:
- Sender fills Reactor's inbox Mailbox, then calls `Notifier.notify()` (socket write).
- Reactor wakes on a Poller event (socket read-ready on Notifier's read socket), then drains inbox with `try_receive_batch`.

**The gap:** The wake mechanism is different.

| Matryoshka | otofu Reactor |
|-----------|--------------|
| `mbox_interrupt` → condition variable signal | `Notifier.notify()` → socket write |
| Receiver blocked on `mbox_wait_receive` | Reactor blocked on `epoll_wait`/`kevent` |

`mbox_interrupt` cannot wake a thread blocked on `epoll_wait`. They use different OS primitives.

### What Matryoshka provides

- Queue aspect: `mbox_send` enqueues the Message atomically. Ownership transfers. `DIRECT`.
- Batch drain aspect: `try_receive_batch` non-blocking drain is the correct Reactor pattern. `DIRECT`.
- Wake aspect: `mbox_interrupt` does NOT apply. Notifier (socket pair) is required. `IMPOSSIBLE`.

### Required ordering for App→Reactor

Critical ordering from Matryoshka OOB pattern (adapted):

```
1. mbox_send(reactor_inbox, &m)    ← fill inbox FIRST
2. Notifier.notify()               ← then wake Reactor
```

Reversing this (notify before send) is a race: Reactor may call `try_receive_batch` before the Message is enqueued and see an empty batch.

### Compatibility

The Mailbox structure is reused. The wake mechanism is user-provided (Notifier). This is the only place where Matryoshka's infrastructure is used but its built-in wake mechanism (`mbox_interrupt`) is not.

### Violations

**V-MAR1 — `mbox_interrupt` must not be used in place of Notifier.**
The Reactor does not block on `mbox_wait_receive`. Calling `mbox_interrupt` on the Reactor's inbox has no effect on the Reactor thread. It is not an error, but it is a no-op. Using it would create a false expectation that the Reactor woke.

**V-MAR2 — `try_receive_batch` returns `.Interrupted` if mailbox was interrupted.**
If someone incorrectly calls `mbox_interrupt` on the Reactor's inbox, the next `try_receive_batch` returns `.Interrupted` with an empty list. Items in the queue are NOT returned on `.Interrupted` — a second call is needed. Reactor must handle this result.

---

## Primitive: TriggeredChannel

**Classification: `DIRECT`**

### Representation in Matryoshka

TriggeredChannel embeds `PolyNode` at offset 0.
A dedicated Reactor-internal Pool manages allocation and recycling.

```
TriggeredChannel :: struct {
    using poly:   PolyNode,         // offset 0 — required
    seq:          SequenceNumber,   // u64 monotonic, ABA token
    channel_num:  ChannelNumber,    // u16
    trigger_flags: TriggerFlags,    // packed u8
    // platform IO state (e.g., IO_STATUS_BLOCK on Windows)
}
```

`PolyNode.id` set to `TriggeredChannelId` (a fixed constant != 0).

### Pool usage

`on_get`:
```
if m^ == nil:
    allocate new TriggeredChannel via ctx.allocator
    set poly.id = TriggeredChannelId
    set m^
else:
    zero TriggeredChannel fields (seq, channel_num, trigger_flags)
    zero platform IO state
```

`on_put`:
```
// No cap enforcement needed — number of TriggeredChannels is bounded by max connections.
// Leave m^ non-nil — pool stores.
```

Lifecycle in relation to Pool:

```
pool_get ──► allocated ──► Poller.register ──► active ──► Poller.deregister ──► pool_put
```

**Rule:** `pool_put` must only be called AFTER the TriggeredChannel is fully deregistered from Poller maps (`deregistered` state in SM5). Returning to pool before deregistration violates pointer stability (L-TC2, L-TC3 from P3).

### Pointer stability under Pool

Matryoshka Pool stores items via heap allocation. Objects do not move — `pool_put` links the PolyNode into the free-list, but the `TriggeredChannel` struct stays at its heap address. `pool_get` unlinks it (resets prev/next to nil) and returns the same address.

This is compatible with pointer stability requirements. The Poller's Object Map holds `*TriggeredChannel` (the heap address). That address is stable from `pool_get` until `pool_put`. During pooling (free-list), the address is still stable — but no external code should hold it, and none does (it's deregistered).

### Compatibility

Full match for allocation, recycling, and pointer stability.
The single-list-membership constraint (PolyNode in one list at a time) is satisfied: a TriggeredChannel is either in the Pool's free-list OR in active use (not in any list). It is never simultaneously in the Pool and in the Poller's map. The Poller's dual-map is a hash map (not an intrusive list), so it does not conflict with PolyNode's `prev`/`next`.

### Violations

**V-TC1 — Must not return to Pool while still registered.**
This is the most critical rule. It is NOT enforced by Matryoshka. It is an ordering contract (L-TC2, L-TC3, SM5 `deregistering` state). Violation: pool gives the object to a new Channel assignment before the old Poller registration is gone. Events for the old SequenceNumber could fire and be dispatched to the new Channel. Silent corruption.

**V-TC2 — Windows IO_STATUS_BLOCK.**
On Windows, the kernel holds a pointer to fields inside TriggeredChannel (the `IO_STATUS_BLOCK`). The kernel does not know about Matryoshka's ownership convention. An outstanding AFD_POLL operation holds the kernel-facing pointer. The TriggeredChannel must NOT be returned to Pool while a kernel operation is pending — the pool's `on_put` must wait for the completion before storing. This is a constraint on the Windows-specific implementation of `on_put`, not a Matryoshka violation.

---

## Primitive: Engine

**Classification: `DIRECT`** (conceptual mapping)

### Representation in Matryoshka

Engine is the top-level **Master** in Matryoshka terminology.

A Matryoshka Master:
- Runs on a thread (Reactor thread = I/O Master).
- Owns all its Mailboxes and Pools.
- Is the only participant that knows concrete types.
- Has `newMaster` / `freeMaster` as a paired lifecycle.

Engine maps to this directly:
- `Engine.create()` = `newMaster` — allocates Pool (MessagePool), starts Reactor thread.
- `Engine.destroy()` = `freeMaster` — follows Matryoshka's required teardown order.

### Teardown order (from Matryoshka Doll 3 `freeMaster` pattern)

```
1. pool_close(message_pool)          ← returns stored Messages as list.List
2. dispose remaining pooled Messages ← for each: destroy Appendable buffers, free struct
3. matryoshka_dispose(&pool_item)    ← free pool struct
4. mbox_close(each ChannelGroup.mb)  ← returns queued Messages as list.List
5. dispose remaining queued Messages ← same as step 2
6. matryoshka_dispose(&mb_item)      ← free each mailbox struct
7. join Reactor thread
8. free Engine struct
```

**Rule:** Pool must be closed before Mailboxes. The `on_get`/`on_put` hooks access `ctx` (Engine or Master state). Freeing the Master before closing the Pool causes use-after-free in hooks (this is the `freeMaster` ordering rule from Matryoshka Doll 3).

### Violations

**V-E1 — Allocator ownership (Open Issue #1 from P2_ownership.md).**
Matryoshka stores `mem.Allocator` inside each Pool and Mailbox struct. Each item owns its allocator reference. In otofu, Engine holds the Allocator reference. The `on_get` hook's `ctx` also carries the allocator. If the Allocator is freed while Pool's `on_put` is still executing (race during Engine.destroy), use-after-free occurs. Matryoshka's design assumes the allocator outlives the Pool/Mailbox that references it. This is the same rule in otofu (Allocator must outlive Engine). Not a violation of Matryoshka's model — it is enforced by convention.

---

## Primitive: ChannelGroup

**Classification: `PARTIAL`**

### Representation in Matryoshka

ChannelGroup maps to the **sub-Master** concept — a unit that owns a Mailbox and a list of Channels.

Application thread holds a non-owning reference (handle) to ChannelGroup. In Matryoshka, there is no built-in concept of a non-owning Master reference. The ChannelGroup handle is a raw pointer held by the application, outside the `^MayItem` convention.

### Where the partial mapping begins

Matryoshka's Master model: one thread, one Master, exclusive ownership. ChannelGroup is accessed by two parties:
- Reactor thread: writes to the Mailbox (`mbox_send`), manages Channels.
- Application thread: reads from the Mailbox (`mbox_wait_receive`), calls `post`.

This is a cross-thread shared reference pattern. Matryoshka provides the Mailbox as the safe cross-thread transfer mechanism, but the ChannelGroup handle itself (the non-owning reference) is outside Matryoshka's ownership model.

### What Matryoshka covers

- Mailbox ownership within ChannelGroup: `DIRECT` (Matryoshka Mailbox, owned by ChannelGroup).
- Message transfer through the Mailbox: `DIRECT`.
- ChannelGroup's Channel list (Reactor-internal): PolyNode intrusive list, `PARTIAL`.

### What Matryoshka does not cover

- The non-owning handle given to the application thread. This is a raw pointer outside `^MayItem`.
- Invalidation of the handle on Engine destroy. Matryoshka has no lifecycle hook for this.

### Violations

**V-CG1 — Application's ChannelGroup handle is not a `^MayItem`.**
The application holds a raw reference. There is no ownership token. The application cannot determine from the handle alone whether the ChannelGroup is still valid (Engine may have been destroyed). This violates the spirit of Matryoshka's explicit ownership model. Mitigation (from P3): application is responsible for the ordering protocol — it must stop using all handles before calling `Engine.destroy()`.

**V-CG2 — Only one thread calls `mbox_wait_receive` on any given ChannelGroup Mailbox.**
Matryoshka Mailbox is MPMC (multiple producers, multiple consumers). It permits multiple concurrent receivers. otofu forbids it (INV-21). This is a constraint that otofu imposes on top of Matryoshka's permissive model. Must be enforced by convention, not by API.

---

## Primitive: Channel

**Classification: `PARTIAL`**

### Representation in Matryoshka

Channel can embed `PolyNode` at offset 0 for use in intrusive lists within the Reactor.

```
Channel :: struct {
    using poly:    PolyNode,       // offset 0
    number:        ChannelNumber,
    state:         ChannelState,   // SM2 state machine
    socket_ref:    ^Socket,        // non-owning reference
    // ...
}
```

Channel instances live in the ChannelGroup's Channel list, managed as a Reactor-internal intrusive list.

### Where the partial mapping begins

Channel does NOT cross the thread boundary. It does not travel through Matryoshka Mailbox or Pool. It does not use `MayItem` as its primary ownership token — ownership is fixed in the hierarchy (ChannelGroup owns Channel; no transfer).

Matryoshka PolyNode is useful for:
- Intrusive list of active Channels within a ChannelGroup.
- Channel state machine (SM2) is entirely user code.

Matryoshka is not useful for:
- Channel lifecycle management (state machine transitions are Reactor logic).
- Channel → Socket linkage (non-owning reference; no Matryoshka construct).
- ChannelNumber assignment and recycling (user code).

### Violations

**V-CH1 — Channel `PolyNode.id` must distinguish channel types.**
If Listener Channels and IO Channels share the same `id`, the Reactor cannot dispatch on type without examining state. Assign distinct ids (e.g., `ListenerChannelId`, `IOChannelId`). This is the Matryoshka `id` dispatch pattern applied to Channels.

**V-CH2 — Channel cannot be in two lists simultaneously.**
PolyNode has one `prev`/`next`. A Channel can be in exactly one intrusive list. If otofu needs to track Channels in multiple collections (e.g., "all channels" list AND "closing channels" list), it cannot use the same PolyNode for both. Either use a separate tracking field, or accept single-list membership constraint.

---

## Primitive: Reactor

**Classification: `PARTIAL`**

### Representation in Matryoshka

Reactor is a specialized Master: it runs on a dedicated thread and owns Pools and Mailboxes. The ownership model matches Matryoshka's Master model directly.

The partial mapping arises from the blocking mechanism.

### Where the partial mapping begins

A Matryoshka Master blocks on `mbox_wait_receive`.
The Reactor blocks on `epoll_wait` / `kevent` / `NtRemoveIoCompletion`.

These are fundamentally different blocking mechanisms. The Reactor is the I/O thread — it cannot yield its OS event loop to block on a Mailbox condition variable.

### What Matryoshka covers

- All Pools owned by Reactor (TriggeredChannel Pool, internal Message Pool): `DIRECT`.
- Reactor's inbound Mailbox (App→Reactor): queue aspect is `DIRECT`; wake is Notifier.
- Reactor's outbound Mailboxes (one per ChannelGroup, Reactor→App): `DIRECT`.

### What Matryoshka does not cover

- The event loop itself (`epoll_wait`, `kevent`). This is user code.
- The Poller abstraction. This is user code.
- The Notifier wake mechanism. This is user code.

### Violations

**V-R1 — Reactor must not call `mbox_wait_receive`.**
The Reactor is the I/O thread. Calling `mbox_wait_receive` with infinite timeout would block the I/O thread on a condition variable, halting all socket I/O. The Reactor must use Poller as its primary blocking mechanism. All Mailbox operations in the Reactor are non-blocking (`try_receive_batch`).

**V-R2 — `pool_get_wait` must not be called by the Reactor.**
Same reason. Blocking on a pool condition variable halts the I/O thread. Reactor message allocation must use `.Available_Only` and treat pool exhaustion as an error, not a wait condition.

---

## Primitive: Notifier

**Classification: `IMPOSSIBLE`**

### Why impossible

Notifier is a socket-pair wake mechanism: the application writes a byte to the write socket; the Reactor's Poller detects read-readiness on the read socket and wakes.

Matryoshka's `mbox_interrupt` provides conceptually similar behavior (wake without data), but via a condition variable, not a socket. A thread blocked on `mbox_wait_receive` can be woken by `mbox_interrupt`. A thread blocked on `epoll_wait` cannot.

| Aspect | Matryoshka `mbox_interrupt` | otofu Notifier |
|--------|----------------------------|----------------|
| Blocking primitive | condition variable | epoll / kqueue event |
| Wake mechanism | signal condition | socket write |
| Receives data | No | No |
| Self-clearing | Yes (flag clears on next receive) | Yes (drain read socket) |
| Thread blocked | `mbox_wait_receive` | `epoll_wait` / `kevent` |

The Reactor blocks on a different OS primitive than Matryoshka's Mailbox. These are incompatible. No mapping is possible.

### Consequence

Notifier must be implemented entirely in user code as a socket pair:
- Create two connected sockets (UDS on Linux, loopback TCP on Windows pre-AF_UNIX).
- Register the read socket with Poller (armed for read-readiness).
- Application writes 1 byte to the write socket (`Notifier.notify()`).
- Reactor drains read socket and processes the application's Mailbox batch.

This is the existing tofu design and it must carry over to otofu unchanged.

---

## Primitive: Socket

**Classification: `IMPOSSIBLE`**

### Why impossible

Socket is an OS file descriptor (an integer). It is not a heap-allocated struct that can embed PolyNode. The OS manages the FD's lifetime; otofu manages the wrapper struct (Skt).

A Socket wrapper struct (`Skt`) could embed PolyNode if it needs to travel in intrusive lists within the Reactor. However:
- Skt never crosses a thread boundary (Reactor-exclusive).
- Skt never uses Pool (lifetime is Channel-driven, not recycled).
- Skt's address must be stable for `IO_STATUS_BLOCK` on Windows — same constraint as TriggeredChannel, but Socket is even more OS-entangled.

PolyNode could be embedded in Skt as a list-node convenience for Reactor-internal channel tracking, but this is incidental, not a Matryoshka mapping.

No Pool, no Mailbox, no MayItem — the core Matryoshka value is in managing ownership across boundaries. Socket never crosses any boundary.

---

## Primitive: Poller

**Classification: `IMPOSSIBLE`**

### Why impossible

Poller wraps OS event notification (epoll, kqueue, wepoll). It is a facade over OS APIs that have no Matryoshka analogue. Poller's dual-map (ChannelNumber → SequenceNumber → `*TriggeredChannel`) is a hash map, not an intrusive list. Poller's operations (`epoll_ctl`, `kevent`, `NtDeviceIoControlFile`) are OS syscalls.

No Matryoshka primitive applies.

---

## Buffer Artifacts: MetaHeaders and Body

**Classification: `PARTIAL`**

### Representation

MetaHeaders and Body are Appendable buffers embedded inside Message. They are NOT standalone PolyNode items — they do not travel independently. They travel with the Message.

Matryoshka applies to MetaHeaders/Body only indirectly:

- When Message is `pooled` (in Matryoshka Pool free-list), MetaHeaders and Body have cleared content but retained capacity. No Matryoshka action on the buffers.
- When `on_get` is called with `m^ != nil` (recycled Message), the hook resets the Appendable buffers. This is user code inside the hook.
- When `on_put` decides to drop the Message (over pool limit), the hook must free the Appendable buffers explicitly before freeing the Message struct.

### Violations

**V-BUF1 — Appendable is not intrusive.**
Appendable is a heap-allocated variable-size buffer. It is a separate allocation inside Message. This is not a violation of Matryoshka's intrusive model — the intrusive node is Message, not its contents. But it means the Appendable buffer must be managed manually in `on_get` and `on_put` hooks. There is no Matryoshka infrastructure for this.

**V-BUF2 — No size limit on Appendable by default.**
If MetaHeaders or Body grow during application use, `on_put` retains the enlarged buffer capacity. Under a sustained workload with large messages, pool memory grows without bound. `on_put` must check buffer capacity and free buffers that exceed a maximum size limit. This is the same `in_pool_count` / capacity policy as Pool item count limits.

**V-BUF3 — Buffer references outside MayItem convention.**
A raw pointer to MetaHeaders or Body content is not protected by `^MayItem`. If application retains such a pointer across `pool_put` or `mbox_send`, it has a dangling reference that Matryoshka cannot detect. (INV-08, INV-09 from P3.)

---

## Value Types: N/A Primitives

These primitives require no Matryoshka mapping. Documented for completeness.

| Primitive | Reason |
|-----------|--------|
| Allocator | Matryoshka uses `mem.Allocator` natively. Same type. |
| Address | Embedded value in Channel. Does not travel through Mailbox or Pool. |
| Header | Embedded struct in Message. No separate allocation. Travels with Message. |
| ChannelNumber | `u16` scalar. |
| SequenceNumber | `u64` scalar. |
| TriggerFlags | Packed `u8`. Passed by value. |

---

## Constraint Summary

Requirements imposed on otofu by Matryoshka's model that are not API-enforced.
These must be upheld by convention and enforced by review.

| ID | Constraint | Applies To | If Violated |
|----|-----------|-----------|-------------|
| C1 | `PolyNode` must be the first field (offset 0) in every traveling struct | Message, TriggeredChannel, Channel | Invalid cast. Silent corruption or panic. |
| C2 | `PolyNode.id` must be != 0 | All PolyNode structs | Immediate panic on any Matryoshka operation. |
| C3 | An item can be in exactly one intrusive list at a time | All PolyNode structs | Both lists corrupted. |
| C4 | `on_get` / `on_put` must not call `pool_get` or `pool_put` | Message Pool hooks, TC Pool hooks | Silent state corruption in Pool. No immediate error. |
| C5 | `polynode_reset` required after `list.pop_front` on batch returns | Mailbox close drain, Pool close drain | Debug panic on next `mbox_send` or `pool_put`. |
| C6 | TriggeredChannel must not return to Pool while registered in Poller | TriggeredChannel | Stale Poller events dispatched to new Channel. Silent misrouting. |
| C7 | Reactor must not call `mbox_wait_receive` or `pool_get_wait` | Reactor | I/O thread stalls. All socket I/O halted. |
| C8 | `defer pool_put` must be placed before `pool_get` (not after) | All Pool users | No safety net on early-return paths. |
| C9 | Allocator passed to Engine must outlive Engine | Engine, Pool, Mailbox internals | Use-after-free in pool/mailbox hooks. |
| C10 | `on_put` must check Appendable buffer capacity and free oversized buffers | Message Pool on_put | Pool memory grows without bound. |

---

## Mapping Coverage

| Matryoshka Doll | otofu Coverage |
|----------------|----------------|
| Doll 1 — PolyNode + MayItem | Message, TriggeredChannel, Channel (partial) |
| Doll 2 — Mailbox | ChannelGroup receive path (direct); Reactor inbound (partial — queue only, not wake) |
| Doll 3 — Pool | MessagePool (direct); TriggeredChannel Pool (direct) |
| Doll 4 — Infrastructure as Items | Not required for baseline otofu. Applicable if Mailbox or Pool ownership needs to transfer between Engine components at runtime. |

Dolls 1–3 are required. Doll 4 is optional.
