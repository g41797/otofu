# P3 Ownership Model

Sources: P1_primitives.md, P2_ownership.md, P2_state_machines.md
Notation: Matryoshka `^MayItem` (`m^ != nil` = you own it, `m^ == nil` = you do not)

This document is the definitive per-object ownership specification for otofu.
Each section answers: who owns it, how it moves, when it is valid, and what is forbidden.

---

## Quick Reference

| Object | Owner | Transferable | Thread-Confined | Shared | Buffers Owned |
|--------|-------|-------------|----------------|--------|--------------|
| Engine | Caller of `create()` | No | No (handle only) | No | — |
| MessagePool | Engine | No | No | No | — |
| Reactor | Engine | No | Yes — Reactor thread | No | — |
| Poller | Reactor | No | Yes — Reactor thread | No | — |
| Notifier | Reactor | No | Yes — Reactor thread | No | — |
| Mailbox | ChannelGroup | No | No (boundary object) | No | Queue nodes |
| ChannelGroup | Engine | No (handle shared) | No | ⚠ WARN | — |
| Channel | ChannelGroup | No | Yes — Reactor thread for state | No | Address (value) |
| Socket | Reactor | No | Yes — Reactor thread | No | FD |
| TriggeredChannel | Poller | No | Yes — Reactor thread | No | — |
| Message | Exclusive — one owner at a time | Yes | No (crosses thread boundary) | No | Header, MetaHeaders, Body |
| Header | Message | No (embedded) | — | No | — |
| MetaHeaders | Message | No (travels with Message) | — | No | Appendable buffer |
| Body | Message | No (travels with Message) | — | No | Appendable buffer |
| Allocator | External (reference held by Engine) | No | No | ⚠ WARN | — |

---

## Object: Engine

### Owner
The caller of `Engine.create()`. The handle is a value or pointer held by the application's main context. Engine does not own itself.

### Created by
`Engine.create(allocator, options)` → `starting` → `running`

### Destroyed by
`Engine.destroy()` → `draining` → `destroyed`
Blocking call. Caller must not release the handle until `destroy()` returns.

### Lifetime Rules

| Rule | Description |
|------|-------------|
| L-E1 | Engine must outlive all ChannelGroups. ChannelGroup handles become invalid the moment `destroy()` is called. |
| L-E2 | Engine must outlive all Messages currently in-flight. The `draining` state enforces this: Engine blocks until all Messages are returned to pool. |
| L-E3 | Engine must outlive the Allocator it references. If the Allocator is external, the caller is responsible for destruction order. |
| L-E4 | Only one Engine instance per Allocator at a time is safe. Engine internal structures assume exclusive allocator access for their lifetime. |

### Transfer Points
None. Engine handle is not transferred. It is used in place.

### Invalid States

| Condition | Why invalid |
|-----------|-------------|
| Any API call after `Engine.destroy()` returns | Engine is `destroyed`. All owned primitives are freed. Calling any method is a use-after-free. |
| `Engine.create()` called on a non-nil engine handle | Double-create. Prior Engine and its owned resources are leaked. |
| `Engine.destroy()` called twice | Second call is on a `destroyed` engine. Undefined behavior. Must be a no-op or explicit error. |

---

## Object: Message

Message has the most complex ownership in the system. It crosses the thread boundary and travels between three owners.

### States and Owners

| State (SM3) | Owner | MayItem state | Thread access |
|-------------|-------|--------------|--------------|
| `nonexistent` | — | n/a | None |
| `pooled` | MessagePool | n/a (no MayItem) | None. Any access is forbidden. |
| `owned-app` | Application thread | `m^ != nil` | Application thread only |
| `owned-engine` | Reactor thread | `m^ == nil` (caller's) | Reactor thread only |

### Created by

Two paths. Both yield `owned-app`.

| Path | Mechanism | When to use |
|------|-----------|-------------|
| From pool | `Engine.get(strategy)` | Normal case. Prefer this. |
| Direct allocation | `Message.create(allocator)` | Only when pool is unavailable (testing, bootstrap). Requires explicit `Message.destroy()` — do NOT `put()`. |

### Destroyed by

| Path | Mechanism | Valid from state |
|------|-----------|-----------------|
| Return to pool | `Engine.put(&msg)` | `owned-app` only |
| Free directly | `Message.destroy()` | `owned-app` only; for directly-allocated Messages |
| Silent pool return | Reactor internal, no caller notification | `owned-engine`, on dispatch failure |

### Transfer Points

All four transfer points. No others are legal.

#### TP-M1: Pool → Application (`get`)

```
Before: Message is `pooled`. MayItem does not exist yet.
Call:   m := Engine.get(strategy)
After:  Message is `owned-app`. m^ != nil.
Guard:  strategy=poolOnly may return m^ == nil (pool empty). Caller must check before use.
Pointer: m^ is non-nil on success; nil on pool-empty with poolOnly strategy.
```

#### TP-M2: Application → Engine (`post`)

```
Before: Message is `owned-app`. m^ != nil.
Call:   ChannelGroup.post(&m)
After:  Message is `owned-engine`. m^ == nil. Caller has surrendered ownership.
Guard:  Engine must be `running`. post() during `draining` must fail and leave m^ non-nil.
Pointer: m^ set to nil on success. If post() returns error, m^ MUST remain non-nil (ownership stays with caller).
```

**Critical:** If `post()` fails, ownership must not transfer. The `^MayItem` convention makes this explicit at the call site — if `m^` is nil after an error, ownership was lost silently.

#### TP-M3: Engine → Application (`waitReceive`)

```
Before: Message is `owned-engine`. Reactor has dispatched it to Mailbox.
Call:   m, err := ChannelGroup.waitReceive(timeout)
After:  Message is `owned-app`. m^ != nil on success.
Guard:  timeout expiry returns m^ == nil (no message). Not an error, just empty.
        Mailbox.Closed return means Engine is `draining`; no more messages will arrive.
Pointer: m^ non-nil on success; nil on timeout or closed.
```

#### TP-M4: Application → Pool (`put`)

```
Before: Message is `owned-app`. m^ != nil.
Call:   Engine.put(&m)
After:  Message is `pooled`. m^ == nil.
Guard:  put() on nil MayItem is a no-op (double-put safe).
Pointer: m^ set to nil unconditionally.
```

#### TP-M5: Engine → Pool (silent, internal)

```
Before: Message is `owned-engine`. Reactor cannot deliver (Channel closed, error).
Action: Reactor returns Message to pool internally.
After:  Message is `pooled`.
Notification: None. Application does not learn this happened.
```

**Warning:** TP-M5 is silent. If application posted a Message that was never delivered (Channel closed), it receives no confirmation. Use `message_id` for correlation if delivery confirmation matters.

### Buffer Components

Message owns three components. They travel with the Message through all states.

| Component | Storage | Created | Reset on pool return | Freed on destroy |
|-----------|---------|---------|---------------------|-----------------|
| Header | Embedded struct (no heap) | With Message | `reset()` zeros fields | With Message struct |
| MetaHeaders | Appendable (heap, variable) | On first write | `reset()` clears content; capacity retained | `Message.destroy()` |
| Body | Appendable (heap, variable) | On first write | `reset()` clears content; capacity retained | `Message.destroy()` |

**Rule:** When a Message is `pooled`, all three components are logically empty but MetaHeaders and Body buffers retain their allocated capacity. This is intentional — it avoids reallocation on reuse.

**Rule:** When a Message is `owned-app`, the application owns the buffers. The application may read and write MetaHeaders and Body. It must not free them directly — `Message.destroy()` or `put()` handles that.

**Rule:** When a Message is `owned-engine`, no one may read or write the buffers except the Reactor dispatching that Message.

### Lifetime Rules

| Rule | Description |
|------|-------------|
| L-M1 | Every `get()` that returns non-nil must be paired with exactly one `put()` or `post()`. No exceptions. Use `defer Engine.put(&m)` immediately after `get()`. |
| L-M2 | `post()` and `put()` both satisfy the obligation from L-M1. Do not call both. |
| L-M3 | Never store a raw `^Message` pointer separately from the `^MayItem`. The `^MayItem` is the ownership token. Copying the raw pointer creates a second reference with no ownership signal. |
| L-M4 | After `post()`, treat `m` as gone. It is nil. Do not read it, do not `put()` it. |
| L-M5 | MetaHeaders and Body Appendable buffers must not be referenced after the Message is `put()` or `post()`. The backing memory may be reused by another thread immediately. |
| L-M6 | Messages returned by `Message.create(allocator)` must be freed with `Message.destroy()`, not `put()`. Putting a directly-allocated message into the pool leaks the allocator contract. |

### Invalid States

| Condition | Classification |
|-----------|---------------|
| Read/write Message after `post()` (m^ is nil, raw copy used) | Use-after-transfer. Undefined behavior. |
| `put()` after `post()` (m^ is nil) | No-op by convention (double-put safe), but indicates logic error. |
| `get()` result not checked for nil before use | Nil dereference. |
| Application thread accessing `owned-engine` Message | Thread-safety violation. Reactor owns it. |
| Reactor thread accessing `owned-app` Message | Thread-safety violation. Application owns it. |
| MetaHeaders or Body buffer access after `put()` | Use-after-free. Buffer backing may be reused. |

---

## Object: Channel

### Owner
ChannelGroup. Channel is created and destroyed by Engine on behalf of ChannelGroup.
Application thread holds no ownership — it holds a ChannelGroup handle (reference).

### Thread Ownership Split

Channel has a split-thread access model:

| Aspect | Thread | Rule |
|--------|--------|------|
| State machine (SM2) | Reactor thread | All state transitions driven by Reactor |
| ChannelNumber (read-only) | Application thread | Read from incoming Message headers |
| Address (embedded value) | Reactor thread during setup | Application provides at `connect()/listen()` |

Application never reads Channel state directly. It learns of state changes via Messages delivered to the Mailbox (e.g., `channel_closed` notification Message).

### Transfer Points
None. Channel ownership does not transfer. It is created and destroyed by Engine.

### Lifetime Rules

| Rule | Description |
|------|-------------|
| L-C1 | Channel lifetime is bounded by ChannelGroup lifetime. ChannelGroup owns the Channel; when ChannelGroup is destroyed, all its Channels are forcibly closed. |
| L-C2 | Channel creates a TriggeredChannel on entry to `opened`. Channel must deregister the TriggeredChannel before entering `closed`. |
| L-C3 | TriggeredChannel deregistration must complete before Socket FD is closed. See cross-machine ordering constraint in P2_state_machines.md. |
| L-C4 | ChannelNumber is released on `closed`. A new Channel may receive the same ChannelNumber. Application must treat ChannelNumber as a short-lived correlation token, not a permanent identity. |
| L-C5 | Application must clear all state keyed on ChannelNumber when it receives a `channel_closed` Message. There is no grace period — the number may be reused immediately. |

### Invalid States

| Condition | Classification |
|-----------|---------------|
| Application reads Channel.state directly | Unsupported. State is Reactor-private. Use notification Messages. |
| post() on a Channel in `closing` state | Ownership transferred, message silently dropped by peer. Undefined behavior. |
| ChannelNumber used as a persistent session ID | Logic error. Numbers recycle. |
| connect() or listen() while Engine is `draining` | Resource leak — connection may open but Engine will not deliver messages for it. |

---

## Object: Socket

### Owner
Reactor thread. Exclusively. For the entire Socket lifetime.

No other thread ever touches a Socket or its FD.

### Created by
Reactor, internally, when Channel enters `opened`:
- Listener: `socket() + bind() + listen()`
- IO Client: `socket() + connect()` (non-blocking)
- IO Server: FD returned by `accept()` on the Listener Socket

### Destroyed by
Reactor, internally, after TriggeredChannel is `deregistered`:
- `SO_LINGER=0` applied on all platforms
- `close(fd)` called
- FD released to OS

### Transfer Points
None. Socket FD never leaves the Reactor thread. Application has no handle to it.

### Lifetime Rules

| Rule | Description |
|------|-------------|
| L-S1 | Socket is created after ChannelGroup.create() dispatches to Reactor, and before Channel enters `opened`. |
| L-S2 | TriggeredChannel must be deregistered from Poller before Socket FD is closed. This is the primary ABA guard (R4.1). |
| L-S3 | On Windows: `SO_LINGER=0` (abortive close) must be applied before `close(fd)` on all IO Sockets. This prevents TIME_WAIT accumulation under high churn. |
| L-S4 | On all platforms: FD must not be closed until TriggeredChannel has entered `deregistered` state. After `deregistered`, no further Poller events reference this FD. |
| L-S5 | A Socket FD from `accept()` that has no Channel to attach to (Listener closing during accept event) must be closed immediately in the same Reactor loop iteration. It must never be stored. |

### Invalid States

| Condition | Classification |
|-----------|---------------|
| Application thread reads or writes any Socket field | Thread-safety violation. Socket is Reactor-exclusive. |
| `close(fd)` before `epoll_ctl DEL` / `kevent delete` | ABA risk (R4.1). New Socket may receive events for the old FD. |
| send() dispatched on a Socket in `closing` state | Logic error. Must check Socket state in Reactor event loop before dispatch. |

---

## Object: MetaHeaders (Buffer)

MetaHeaders is the optional CRLF key-value header section of a Message.
Backed by Appendable (resizable heap buffer).

### Owner
Whoever owns the Message. Owner of Message = owner of MetaHeaders.
There is no separate ownership token for MetaHeaders.

### Allocation
Lazy: Appendable is first allocated on the first write to MetaHeaders.
If a Message is retrieved and never has MetaHeaders written, no heap allocation occurs.

### Lifetime

| Event | Effect on MetaHeaders buffer |
|-------|------------------------------|
| `Engine.get()` | Buffer exists (from prior use) or nil (fresh allocation). Content is empty after `reset()`. |
| Application writes MetaHeaders | Buffer allocated if nil; data written. |
| `Engine.put(&m)` → `Message.reset()` | Buffer content cleared. Capacity retained. Buffer is NOT freed. |
| `Message.destroy()` | Buffer freed. Capacity gone. |

### Rules

| Rule | Description |
|------|-------------|
| L-B1 | MetaHeaders buffer is owned by the Message. Never hold a reference to the MetaHeaders buffer separately from the Message. The buffer's address may change if Appendable grows (realloc). |
| L-B2 | After `put()`, the buffer may be written by another thread (via a new `get()` on the same Message from pool). Never access MetaHeaders after `put()` or `post()`. |
| L-B3 | After `post()`, the Reactor may read MetaHeaders (e.g., for Address.parse on HelloRequest). Application must not access MetaHeaders concurrently. |

### Invalid States

| Condition | Classification |
|-----------|---------------|
| Pointer to MetaHeaders content held across `put()` | Dangling reference. Buffer content is reset. |
| Pointer to MetaHeaders content held across `post()` | Concurrent access. Reactor owns the Message. |

---

## Object: Body (Buffer)

Body is the optional payload section of a Message.
Backed by Appendable. Same ownership model as MetaHeaders.

### Owner
Whoever owns the Message. Identical to MetaHeaders.

### Allocation
Lazy: allocated on first write to Body.

### Lifetime
Identical to MetaHeaders. See L-B1, L-B2, L-B3 above.
Substitute "Body" for "MetaHeaders" in all three rules.

### Special Case: Pointer Embedding

Body may carry embedded pointers via `ptrToBody()` / `bodyToPtr()` operations.

| Rule | Description |
|------|-------------|
| L-Body-P1 | A pointer embedded in Body is NOT owned by the Message. The Message does not manage the lifetime of the embedded target. The pointed-to object must outlive the Message. |
| L-Body-P2 | If Body carries an embedded pointer, the receiving thread must dereference it before `put()`-ing the Message back to the pool. Once `put()`, the Body is reset and the pointer is gone. |
| L-Body-P3 | Embedded pointers must not cross thread boundaries unless the pointed-to object is itself thread-safe or the pointer target's ownership has been explicitly transferred. |

---

## Object: TriggeredChannel

### Owner
Poller (via the dual Object Map). Reactor thread exclusively.

### Created by
Reactor thread, when Channel enters `opened`. Heap-allocated. Address is stable for lifetime.

### Destroyed by
Reactor thread, after Channel enters `closed` and TriggeredChannel is `deregistered`.
Memory freed only after `deregistered` state is confirmed and no pending events remain.

### Transfer Points
None. TriggeredChannel never leaves Poller or Reactor thread.

### Lifetime Rules

| Rule | Description |
|------|-------------|
| L-TC1 | Must be heap-allocated. Must never be embedded in a resizable container (hash map value slot). The stable address is a hard requirement for Windows `IO_STATUS_BLOCK` and for Poller iterator safety. |
| L-TC2 | Deregistration from Poller maps must happen before Socket FD is closed. Poller maps hold the only references to `*TriggeredChannel`. After deregistration, the pointer in the Object Map is gone; the object can then be freed. |
| L-TC3 | Memory must not be freed until `deregistered` state is entered AND the Reactor has confirmed no events referencing this SequenceNumber are in the OS queue. In practice: free after `epoll_ctl DEL` / `kevent delete` returns and the current event batch is fully processed. |
| L-TC4 | SequenceNumber assigned to this TriggeredChannel must be consumed (looked up in Object Map) before deregistration removes it. After removal, incoming events with this SequenceNumber are discarded (ABA guard). |

### Invalid States

| Condition | Classification |
|-----------|---------------|
| TriggeredChannel freed while SequenceNumber still in Object Map | Dangling pointer in Object Map. Use-after-free when OS delivers event. |
| Socket FD closed before TriggeredChannel deregistered | ABA risk. New Socket with same FD receives stale events. |
| TriggeredChannel moved in memory (e.g., embedded in growing vector) | Pointer instability. Kernel-facing pointer becomes invalid. Memory corruption on Windows. |
| Any thread other than Reactor reads or writes TriggeredChannel | Thread-safety violation. |

---

## Object: ChannelGroup

### Owner
Engine. ChannelGroup is created and destroyed by `Engine.create()` / `Engine.destroy(cg)`.

### Application Access
Application thread receives a *handle* — a non-owning reference.
The application uses the handle to call `post()` and `waitReceive()`.
The application does not own the ChannelGroup; it cannot free it.

### ⚠ SHARED OWNERSHIP WARNING

ChannelGroup is accessed by two parties simultaneously:

| Party | Access type | Thread |
|-------|------------|--------|
| Engine | Owns; manages lifecycle | Engine/Reactor thread |
| Application | Non-owning reference; calls post/waitReceive | Application thread |

This is reference-sharing, not ownership-sharing. Engine owns the ChannelGroup; application holds a non-owning reference. However, it is a form of shared access.

**Risk:** If application calls `post()` or `waitReceive()` after Engine.destroy() returns, the ChannelGroup is freed. The application is accessing freed memory.

**Rule:** Application must stop using all ChannelGroup handles before calling `Engine.destroy()`. The application is responsible for this ordering — Engine.destroy() does not wait for application threads to stop using handles.

**Rule:** Only one application thread calls `waitReceive()` on any given ChannelGroup. Multiple threads calling `waitReceive()` concurrently on the same ChannelGroup is a race condition on the Mailbox.

### Lifetime Rules

| Rule | Description |
|------|-------------|
| L-CG1 | ChannelGroup lifetime is bounded by Engine lifetime. |
| L-CG2 | Application must stop calling `post()` on a ChannelGroup before `Engine.destroy()` is called. There is no API-enforced barrier. |
| L-CG3 | Exactly one thread calls `waitReceive()` on any given ChannelGroup. |
| L-CG4 | `Engine.destroy(cg)` is the only valid destruction path. Application must not free the ChannelGroup directly. |

### Invalid States

| Condition | Classification |
|-----------|---------------|
| `post()` after `Engine.destroy()` | Use-after-free. ChannelGroup is freed. |
| `waitReceive()` after `Engine.destroy()` | Use-after-free. Mailbox is freed. |
| Two application threads calling `waitReceive()` on same ChannelGroup | Race condition on Mailbox receive. |

---

## Object: Mailbox

### Owner
ChannelGroup. One Mailbox per ChannelGroup.
Application thread uses Mailbox via `waitReceive()` (indirectly).
Reactor thread uses Mailbox via `mbox_send()` (to deliver messages).

### Thread Boundary Role
Mailbox is the single legal mechanism for transferring Message ownership across the thread boundary.

| Direction | Sender | Receiver | Mechanism |
|-----------|--------|----------|-----------|
| App → Reactor | Application | Reactor | `mbox_send()` (via `post()`) + `Notifier.notify()` |
| Reactor → App | Reactor | Application | `mbox_send()` then `mbox_wait_receive()` (via `waitReceive()`) |

### Ownership at Mailbox

When a Message enters the Mailbox (via `mbox_send`):
- Sender's `MayItem` is nil
- Message is "in the Mailbox" — owned by the Mailbox queue node
- Receiver's `MayItem` becomes non-nil after `mbox_wait_receive()`

The Mailbox owns the Message while it is queued. Neither sender nor receiver owns it.

### Lifetime Rules

| Rule | Description |
|------|-------------|
| L-MB1 | Mailbox is created with ChannelGroup. Destroyed when ChannelGroup is destroyed. |
| L-MB2 | On `mbox_send()` to a `draining` or `closed` Mailbox, the send must fail and ownership must return to caller (caller's `MayItem` stays non-nil). |
| L-MB3 | On `mbox_wait_receive()` from a `closed` Mailbox, return `.Closed` status. Application receive loop must handle this and stop looping. |
| L-MB4 | During Engine `draining`, Mailbox enters `draining`. No new sends accepted. Application drains remaining items via `waitReceive()`. |
| L-MB5 | Items in the Mailbox queue at forced-close (Engine.destroy timeout) are leaked — not pooled, not delivered. This is a degenerate path only. |

### Invalid States

| Condition | Classification |
|-----------|---------------|
| `mbox_send()` to `closed` Mailbox returns success | Ownership lost. Message not pooled, not delivered. |
| Application loops on `waitReceive()` without handling `.Closed` | Infinite block after Engine drain. Liveness failure. |
| Pool exhaustion with application blocking on `waitReceive()` | Deadlock (R6.3 from P2_state_machines.md). See deadlock note below. |

**Deadlock (R6.3) — Full Description:**

```
Conditions for deadlock:
  1. MessagePool is empty (all Messages are owned-app or in Mailbox)
  2. Application thread is blocked on waitReceive()
  3. Reactor needs to deliver a channel_closed notification to Mailbox
  4. Reactor cannot allocate a Message for the notification (pool empty)
  5. Application cannot return a Message (it is blocked)

Resolution: Reserve N Messages for engine-internal use in a separate pool.
These Messages are never exposed to the application via get().
```

---

## Object: MessagePool

### Owner
Engine.

### Role
Holds all Messages not currently owned by application threads or in-flight in the Reactor.

### Lifetime Rules

| Rule | Description |
|------|-------------|
| L-MP1 | MessagePool is created by Engine during `starting` state. |
| L-MP2 | MessagePool is destroyed by Engine during `draining → destroyed` transition, after all in-flight Messages have been returned. |
| L-MP3 | Pool capacity may grow (strategy=always) but Messages added by direct creation are not returned to pool — they must be destroyed directly. |

### Invalid States

| Condition | Classification |
|-----------|---------------|
| `get()` when pool is empty and strategy=poolOnly | Returns nil. Not an error — caller must handle. |
| Calling `put()` on a Message created with `Message.create()` | Mixes allocator contracts. The allocator used for `Message.create()` must be used for `Message.destroy()`. |

---

## Object: Allocator

### Owner
**⚠ UNRESOLVED — Shared Ownership Warning**

Allocator is provided to Engine at creation. Engine holds a reference. Whether Engine owns the Allocator (and must not outlive it) or whether the caller owns the Allocator (and must not free it while Engine is alive) is not resolved in the current design.

| Scenario | Implication |
|----------|-------------|
| Engine owns Allocator | Caller must not free Allocator after `Engine.create()` returns. Engine frees it on `destroy()`. |
| Caller owns Allocator | Engine may not hold Allocator after `Engine.destroy()` returns. Caller may free Allocator after that. |
| Caller owns, shared reference | Both parties hold references. The last one to finish must free. This is shared ownership. |

**The third scenario is the current implicit behavior. It is the most dangerous.**

**Required decision:** Define Allocator ownership explicitly before any otofu code allocates through it. The destruction order of Engine vs Allocator must be deterministic and documented.

**Recommendation:** Caller owns Allocator. Engine holds a non-owning reference. `Engine.destroy()` must complete all deallocations before returning. After `destroy()`, caller may free Allocator safely.

---

## Ownership Transfer Catalog

All legal ownership transfer points in the system. No others are legal.

| ID | Object | From | To | Mechanism | Caller pointer after |
|----|--------|------|----|-----------|---------------------|
| TP-M1 | Message | MessagePool | App thread | `Engine.get(strategy)` | non-nil (or nil if pool empty + poolOnly) |
| TP-M2 | Message | App thread | Reactor/Engine | `ChannelGroup.post(&m)` | nil |
| TP-M2-fail | Message | (stays with App) | (no transfer) | `post()` returns error | non-nil — MUST stay with caller |
| TP-M3 | Message | Mailbox queue | App thread | `ChannelGroup.waitReceive()` | non-nil |
| TP-M4 | Message | App thread | MessagePool | `Engine.put(&m)` | nil |
| TP-M5 | Message | Reactor/Engine | MessagePool | Internal — no API | n/a (Reactor-internal) |
| TP-M6 | Message | (nonexistent) | App thread | `Message.create(allocator)` | non-nil |
| TP-M7 | Message | App thread | (nonexistent) | `Message.destroy()` | nil |
| TP-MB | Message | App/Reactor thread | Mailbox queue | `mbox_send()` | nil |
| TP-MB-fail | Message | (stays with sender) | (no transfer) | `mbox_send()` to closed Mailbox | non-nil |

There are no other legal transfer points.
Socket, Channel, TriggeredChannel, ChannelGroup, Mailbox, and Engine do not transfer ownership.

---

## Lifetime Ordering Rules

What must exist before what can be created. What must stop before what can be destroyed.

### Creation Order

```
1. Allocator (external)
2. Engine.create(allocator)      ← Engine refs Allocator
3. MessagePool                   ← created by Engine
4. Reactor                       ← created by Engine; thread starts
5. ChannelGroup.create()         ← created by Engine on app request
6. Channel                       ← created by Engine inside ChannelGroup
7. TriggeredChannel              ← created by Reactor when Channel opens
8. Socket                        ← created by Reactor when Channel opens
```

### Destruction Order

```
1. Application stops calling post() on all ChannelGroups
2. Application sends ByeRequest (or ByeSignal) on all IO Channels
   → Channels enter `closing` → `closed`
   → TriggeredChannels deregistered (before Socket FD closed)
   → Sockets closed (after TriggeredChannels deregistered)
3. Engine.destroy() called
   → Mailboxes enter `draining`
   → Application drains remaining Messages via waitReceive()
   → Application calls put() on all received Messages
   → Engine.destroy() unblocks when all in-flight Messages returned
4. Reactor thread joins
   → All TriggeredChannels freed
   → All Sockets confirmed closed
5. MessagePool destroyed
   → All pooled Messages freed
6. ChannelGroups destroyed
   → All Mailboxes closed
7. Engine struct freed
8. Allocator freed (by caller, if caller owns it)
```

**Rule:** Steps 3–8 are sequential and must not be reordered.
**Rule:** Step 2 (Channel shutdown) must complete before step 3 (Engine.destroy). If application skips step 2, Engine.destroy must force-close all Channels and deal with the resulting ByeSignal behavior.

---

## Invalid State Index

All forbidden conditions in one place.

| ID | Object | Condition | Consequence |
|----|--------|-----------|-------------|
| INV-01 | Message | Accessed after `post()` | Use-after-transfer. Undefined behavior. |
| INV-02 | Message | `put()` after `post()` | Caller's MayItem is nil — no-op. Indicates logic error. |
| INV-03 | Message | `post()` during Engine `draining` | Ownership transferred to dying Engine. Message lost. |
| INV-04 | Message | `get()` result unchecked for nil | Nil dereference on `poolOnly` with empty pool. |
| INV-05 | Message | Raw `^Message` pointer stored separately | Ownership escapes MayItem convention. Use-after-transfer possible. |
| INV-06 | Message | Two threads access `owned-app` Message | No synchronization. Data race. |
| INV-07 | Message | Reactor accesses `owned-app` Message | Thread-safety violation. |
| INV-08 | MetaHeaders | Reference held across `put()` or `post()` | Dangling reference. Buffer reset or reused. |
| INV-09 | Body | Reference held across `put()` or `post()` | Dangling reference. Buffer reset or reused. |
| INV-10 | Body | Embedded pointer target freed before Message `put()` | Dangling pointer in Body. |
| INV-11 | Channel | `post()` during `closing` state | Message silently dropped. Undefined protocol behavior. |
| INV-12 | Channel | ChannelNumber used as permanent identity | After recycle, stale number misidentifies new Channel. |
| INV-13 | Channel | Application reads Channel.state directly | Unsupported. State is Reactor-private. |
| INV-14 | Socket | Any application thread access | Thread-safety violation. Socket is Reactor-exclusive. |
| INV-15 | Socket | `close(fd)` before TriggeredChannel deregistered | ABA risk. New Socket receives stale events. |
| INV-16 | Socket | send() on `closing` Socket | Dropped I/O. Must gate on Socket state. |
| INV-17 | TriggeredChannel | Freed while in Poller Object Map | Dangling pointer. Use-after-free on next OS event. |
| INV-18 | TriggeredChannel | Moved in memory | Pointer instability. Kernel-facing pointer invalidated. |
| INV-19 | TriggeredChannel | Deregistered while `dispatching` | Out-of-order teardown. Reactor must complete dispatch first. |
| INV-20 | ChannelGroup | `post()` or `waitReceive()` after Engine `destroyed` | Use-after-free. ChannelGroup memory freed. |
| INV-21 | ChannelGroup | Two threads call `waitReceive()` on same group | Race on Mailbox receive. |
| INV-22 | Mailbox | `mbox_send()` succeeds on `closed` Mailbox | Ownership lost — not pooled, not delivered. |
| INV-23 | Mailbox | Receive loop does not handle `.Closed` return | Infinite block after Engine drain. |
| INV-24 | Engine | Any API call after `destroy()` returns | Use-after-free. |
| INV-25 | Engine | `create()` called on non-nil handle | Prior Engine leaked. |
| INV-26 | Engine | `destroy()` called twice | Undefined behavior. Must be no-op or error. |
| INV-27 | Allocator | Freed while Engine is alive | Engine deallocates through freed allocator. Corruption. |

---

## Shared Ownership Warnings

Two cases. Both are documented here with required mitigations.

---

### ⚠ WARNING 1: ChannelGroup Handle

**What:** Engine owns the ChannelGroup. Application thread holds a non-owning reference (handle).

**Risk:** Application calls `post()` or `waitReceive()` after Engine has destroyed the ChannelGroup. This is use-after-free. The API cannot detect this — both Engine and application hold "valid" references from their respective perspectives.

**Mitigation:** Application must coordinate its own shutdown before calling `Engine.destroy()`. The coordination protocol is:

```
1. Application sends ByeRequest on all IO Channels via post()
2. Application receives ByeResponse (or channel_closed) via waitReceive()
3. Application exits its receive loop on Mailbox.Closed or all channels closed
4. Application calls Engine.destroy()
```

Only after step 3 is complete should `Engine.destroy()` be called. Steps 1–3 are application responsibility. Engine.destroy() provides no barrier that waits for application threads.

---

### ⚠ WARNING 2: Allocator Reference

**What:** Allocator is provided externally to Engine. Both Engine (internally) and the caller (externally) may use the same Allocator.

**Risk:** Caller frees Allocator while Engine is still alive (Engine is `running` or `draining`). Engine then allocates or deallocates through a freed Allocator. Memory corruption.

**Risk:** Engine holds a pointer to Allocator after `Engine.destroy()` returns. If caller frees Allocator after destroy, this is safe — but only if Engine guaranteed all allocations were freed before destroy returned.

**Mitigation:**
- Define ownership explicitly: Caller owns Allocator.
- Engine must complete all deallocations inside `destroy()`.
- Caller must not free Allocator before `Engine.destroy()` returns.
- After `Engine.destroy()` returns, Allocator may be freed.

This is not currently enforced by the API. It is a convention the caller must follow.

---

## Summary

| Category | Count |
|----------|-------|
| Tracked objects | 13 |
| Transfer points | 10 (Message only) |
| Lifetime rules | 24 |
| Invalid states | 27 |
| Shared ownership warnings | 2 |
| Objects with no transfer | 12 (only Message transfers) |
| Thread-confined objects | 5 (Reactor, Poller, Notifier, Socket, TriggeredChannel) |
