# P2 State Machines

Source: P2_raw.md
Prior context: P1_primitives.md, P2_ownership.md, P0_combined.md

Six machines extracted. Raw gave five lifecycle lines and implied a sixth (Mailbox).
Each machine is normalized independently, then cross-checked.

---

## Index

| # | Machine | Raw States | Normalized States | Missing | Race Risks |
|---|---------|-----------|-------------------|---------|------------|
| 1 | Engine | 4 | 5 | 2 | 4 |
| 2 | Channel | 4 | 8 | 5 | 5 |
| 3 | Message | 2 | 4 | 3 | 3 |
| 4 | Socket | 3 (merged) | 7 | 4 | 4 |
| 5 | TriggeredChannel | 3 | 7 | 5 | 3 |
| 6 | Mailbox | 0 (implied) | 4 | 1 | 3 |

---

## SM1 — Engine

**Raw (P2_raw.md line 57):** `created → running → draining → destroyed`

### States

| State | Description |
|-------|-------------|
| `starting` | Engine allocated; Reactor thread launching. Renamed from raw `created` — "created" implies completion; thread start is in progress. |
| `running` | Reactor thread active. All API operations available. |
| `draining` | `destroy()` called. No new operations accepted. Engine waits for in-flight Messages to return and Reactor thread to join. |
| `destroyed` | All owned primitives freed. All ChannelGroup handles invalid. |
| *(pre-create)* | Not a state — object does not exist. Not represented. |

### Transitions

| From | Event | Guard | To |
|------|-------|-------|----|
| — | `Engine.create(allocator, options)` | allocator non-nil | `starting` |
| `starting` | Reactor thread signals ready | thread start ok | `running` |
| `starting` | Reactor thread fails to start | thread start error | `destroyed` (cleanup inline) |
| `running` | `Engine.destroy()` called | — | `draining` |
| `draining` | all Messages returned to pool AND Reactor thread joined | no in-flight Messages | `destroyed` |

### Missing States

1. **`starting` absent in raw.** Raw lists `created → running` as direct. Thread startup is observable time with its own failure mode. Collapsing it hides the failure path.
2. **No `faulted` state.** If the Reactor thread panics or crashes after entering `running`, Engine has no state to express this. All owned resources are in undefined state. Current design has no recovery path.

### Race Risks

| # | Risk | Description |
|---|------|-------------|
| R1.1 | `post()` during `draining` | Application calls `post(msg)` after `destroy()`. Engine is draining. If `post()` succeeds, the message enters a pipeline that will never deliver. Ownership is lost. Must return error and leave ownership with caller. |
| R1.2 | `get()` during `draining` | Application calls `get()`. Pool may still have Messages. Returning one is safe if the caller will `put()` it back, but `post()` on it will hit R1.1. Need explicit policy: `get()` is legal during drain; `post()` is not. |
| R1.3 | `waitReceive()` during `draining` | Application is blocked. Engine drains Mailbox. Application must receive the `channel_closed` sentinel and terminate its receive loop. If application loops back into `waitReceive()` after all channels are closed, it blocks forever. |
| R1.4 | `destroy()` called twice | Second call on `destroyed` is undefined. Raw has no guard. Must be no-op or explicit error. |

---

## SM2 — Channel

**Raw (P2_raw.md line 58):** `unassigned → opened → ready → closed`

This is the most compressed raw state machine. `opened` hides three sub-paths and `ready → closed` hides the shutdown protocol.

### Channel Types

Two channel types exist in the system. They share states but follow different paths.

| Type | Created by | Role |
|------|-----------|------|
| **Listener** | `listen(addr)` | Accepts incoming connections. Does not carry message traffic directly. |
| **IO** | `connect(addr)` or `accept()` | Carries bidirectional message traffic. |

### States

| State | Applies To | Description |
|-------|-----------|-------------|
| `unassigned` | Both | Channel struct exists. No ChannelNumber assigned. Not registered in Engine maps. |
| `opened` | Both | ChannelNumber assigned (range 1–65534). Registered in Engine maps. Socket creation initiated. |
| `connecting` | IO (client) | `connect(addr)` called. TCP SYN in flight. Waiting for Poller readiness event. |
| `listening` | Listener | `bind(addr)` + `listen()` complete. Accepting incoming connections via Poller events. |
| `handshaking` | IO only | Socket connected. Protocol handshake in progress (HelloRequest / HelloResponse exchange). |
| `ready` | Both | Handshake complete. Message I/O active. For Listener: accepting new IO channels. |
| `closing` | IO only | ByeRequest sent or received. Waiting for ByeResponse. No new messages accepted. |
| `closed` | Both | ChannelNumber released. TriggeredChannel deregistered. Socket closing. Application must clear all state keyed on this ChannelNumber. |

### Transitions — IO Client

| From | Event | Guard | To |
|------|-------|-------|----|
| `unassigned` | `ChannelGroup.create()` | Engine `running` | `opened` |
| `opened` | `connect(addr)` | addr valid | `connecting` |
| `connecting` | Poller: connect ready | TCP ok | `handshaking` |
| `connecting` | Poller: connect ready | TCP error | `closed` |
| `connecting` | timeout | no readiness event | `closed` |
| `handshaking` | HelloResponse received | — | `ready` |
| `handshaking` | timeout or peer error | — | `closed` |
| `ready` | `post(ByeRequest)` | — | `closing` |
| `ready` | ByeRequest received | — | `closing` |
| `ready` | ByeSignal received | forced | `closed` |
| `ready` | network error | — | `closed` |
| `closing` | ByeResponse received/sent | — | `closed` |
| `closing` | timeout | no ByeResponse | `closed` (forced) |

### Transitions — IO Server (accepted)

| From | Event | Guard | To |
|------|-------|-------|----|
| `unassigned` | `accept()` completes | Listener `ready` | `opened` |
| `opened` | HelloRequest received | — | `handshaking` |
| `handshaking` | HelloResponse sent | — | `ready` |
| `handshaking` | timeout or peer error | — | `closed` |
| `ready` | *(same as IO client from `ready`)* | — | — |

### Transitions — Listener

| From | Event | Guard | To |
|------|-------|-------|----|
| `unassigned` | `ChannelGroup.create()` | Engine `running` | `opened` |
| `opened` | `listen(addr)` | addr valid | `listening` |
| `listening` | WelcomeResponse received | — | `ready` |
| `listening` | error | — | `closed` |
| `ready` | Poller: accept event | — | `ready` (spawn new IO Channel) |
| `ready` | `shutdown()` or Engine `draining` | — | `closed` |

### Missing States

1. **`connecting` absent in raw.** Raw jumps from `opened` to `ready`. The TCP handshake phase is observable and has its own failure mode (refused, timeout).
2. **`listening` absent in raw.** Same collapse. Listener activation is a distinct phase.
3. **`handshaking` absent in raw.** The Hello/Welcome protocol exchange between `connecting` and `ready` is the conversation-layer establishment. Its absence means there is no defined error handling for handshake failures.
4. **`closing` absent in raw.** Raw jumps from `ready` to `closed`. The ByeRequest/ByeResponse exchange (P2_raw.md line 63) is the shutdown protocol. Its absence means no defined timeout or forced-close path.
5. **No `error` state.** All failure transitions jump directly to `closed`. Error information is discarded. Application receives only a `channel_closed` notification; it cannot distinguish normal close from error close without an error code in the notification message.

### Race Risks

| # | Risk | Description |
|---|------|-------------|
| R2.1 | Simultaneous Bye | Both peers send ByeRequest at the same time. Both enter `closing`. Both will receive a ByeRequest and a ByeResponse. No tiebreaker defined. Either side may interpret incoming ByeRequest as a response and close; or may wait for ByeResponse that never comes. **Undefined behavior in raw.** |
| R2.2 | `post()` while `closing` | Application posts a Message after ByeRequest is sent. Message will be queued but peer has declared intent to close. Policy is undefined: reject at API level, or deliver and let peer discard? |
| R2.3 | ChannelNumber reuse with stale Mailbox items | Channel closes. ChannelNumber is recycled. New Channel opens with the same ChannelNumber. Mailbox still contains Messages from the old Channel tagged with that ChannelNumber. Application cannot distinguish old from new without application-level message IDs (message_id field). Raw notes this as a rule (line 42–43) but it is a race risk embedded in the state machine. |
| R2.4 | `waitReceive()` after `closed` notification | Application receives `channel_closed` Message. Application must clear all state keyed on ChannelNumber before the same number is reused. There is no ordering guarantee between the `channel_closed` delivery and the new Channel's first message delivery on the same ChannelGroup. |
| R2.5 | `connect()` or `listen()` while Engine `draining` | Channel transitions to `connecting` or `listening`. Engine is shutting down. The connection may succeed, but no Messages will be delivered. Resource is allocated and never used. |

---

## SM3 — Message

**Raw (P2_raw.md line 59):** `pooled → in-flight → pooled`

"In-flight" is a single raw state for what are actually two distinct ownership regimes.

### States

| State | Owner | Description |
|-------|-------|-------------|
| `nonexistent` | — | Not yet allocated, or after `destroy()`. |
| `pooled` | MessagePool | In pool. No thread may access. |
| `owned-app` | Application thread | Returned by `get()` or `waitReceive()`. Application has `MayItem != nil`. |
| `owned-engine` | Reactor thread | After `post()`. In Reactor dispatch pipeline. Caller's `MayItem` is nil. |

### Transitions

| From | Event | Guard | To |
|------|-------|-------|----|
| `nonexistent` | `Message.create(allocator)` | allocator valid | `owned-app` |
| `nonexistent` | `get(strategy=always)` | pool empty | `owned-app` (fresh allocation) |
| `pooled` | `get(strategy=poolOnly or always)` | pool non-empty | `owned-app` |
| `owned-app` | `post(msg)` | ChannelGroup valid, Engine `running` | `owned-engine` |
| `owned-app` | `put(msg)` | — | `pooled` |
| `owned-app` | `Message.destroy()` | — | `nonexistent` |
| `owned-engine` | `waitReceive()` returns | — | `owned-app` |
| `owned-engine` | Reactor dispatch: no receiver / error | — | `pooled` (silently returned) |

### Missing States

1. **`owned-engine` absent in raw.** Raw uses `in-flight` for both `owned-app` and `owned-engine`. The distinction matters: only the Reactor thread may access a Message in `owned-engine`; accessing it from the application thread after `post()` is undefined.
2. **`nonexistent` absent in raw.** Raw assumes all Messages come from the pool. `Message.create()` and `Message.destroy()` exist in the API (P1_primitives.md line 122) but the lifecycle path for directly allocated Messages is not represented.
3. **Silent pool return absent in raw.** When Reactor cannot deliver a Message (e.g., Channel closed mid-dispatch), the Message returns to the pool. This transition is not stated. The application never learns a Message was silently discarded.

### Race Risks

| # | Risk | Description |
|---|------|-------------|
| R3.1 | Raw pointer copy before `post()` | Application stores a raw pointer to the Message before calling `post()`. After `post()`, the `MayItem` is nil, but the raw copy is still non-nil and points to a Message now owned by the engine. The `^MayItem` convention only protects the `MayItem` variable, not raw pointer copies. Enforce: never store raw `^Message`; always use `^MayItem`. |
| R3.2 | `get()` returns nil; caller does not check | `get(strategy=poolOnly)` returns nil when pool is empty. If caller proceeds without checking, it dereferences nil. This is a usage rule, but it belongs in the state machine: the `pooled → owned-app` transition has a guard (pool non-empty) that can fail. The nil path must be explicit. |
| R3.3 | `owned-engine` Messages lost on Engine `destroy()` | Engine enters `draining`. Messages in `owned-engine` state are in the Reactor's dispatch pipeline. If the Reactor joins before delivering all in-flight Messages to Mailboxes, those Messages are neither returned to pool nor delivered. Ownership is lost. The `draining` state must guarantee all `owned-engine` Messages complete dispatch before join. |

---

## SM4 — Socket

**Raw (P2_raw.md line 61):** `created → connecting/listening → ready → closing → closed`

The merged `connecting/listening` is the primary problem. Two distinct paths, two distinct failure modes.

### States

| State | Description |
|-------|-------------|
| `created` | FD obtained from OS. Non-blocking flag set. No address bound. |
| `binding` | `bind(addr)` called. Address assigned. Listener path only. |
| `listening` | `listen(backlog)` called. Poller armed for accept events. |
| `connecting` | Non-blocking `connect(addr)` called. SYN in flight. Poller armed for write-readiness (EPOLLOUT / WRITE). |
| `connected` | TCP handshake complete. Poller armed for read/write as needed. |
| `closing` | Socket I/O complete. `SO_LINGER=0` applied (all platforms). FD about to be closed. |
| `closed` | FD closed. OS may reuse FD integer immediately. |

### Transitions — Listener path

| From | Event | Guard | To |
|------|-------|-------|----|
| `created` | `bind(addr)` | addr valid | `binding` |
| `binding` | `listen(backlog)` | bind ok | `listening` |
| `listening` | Poller: accept event | FD readable | `listening` (accept new Socket; stay) |
| `listening` | Channel.close() | — | `closing` |
| `closing` | `close(fd)` | SO_LINGER=0 applied | `closed` |

### Transitions — IO path (client connect)

| From | Event | Guard | To |
|------|-------|-------|----|
| `created` | `connect(addr)` non-blocking | addr valid | `connecting` |
| `connecting` | Poller: write-ready event | TCP ok (getsockopt) | `connected` |
| `connecting` | Poller: write-ready event | TCP error (getsockopt) | `closing` |
| `connecting` | Reactor timeout | no event | `closing` |
| `connected` | Poller: read-ready event | data available | `connected` (recv) |
| `connected` | Poller: write-ready event | buffer available | `connected` (send) |
| `connected` | Channel enters `closing` | bye sequence | `closing` |
| `connected` | Poller: error or HUP event | — | `closing` |
| `closing` | `close(fd)` | SO_LINGER=0 applied | `closed` |

### Transitions — IO path (server accepted)

| From | Event | Guard | To |
|------|-------|-------|----|
| `created` | `accept()` returns FD | Listener `listening` | `connected` (skip connecting) |
| `connected` | *(same as client from `connected`)* | — | — |

### Missing States

1. **`binding` absent in raw.** Bind failure is a distinct error mode (address in use, permission denied). Cannot be collapsed into `listening`.
2. **`connected` absent in raw.** Raw uses `ready` which is a Channel-level concept. At the Socket level, `connected` is the persistent I/O-capable state. Using `ready` here conflates two layers.
3. **`connecting` absent as distinct state.** Raw merges it with `listening`. They have different Poller arming (write-readiness vs read-readiness) and different completion events.
4. **No `error` state.** Network errors (RST received, ETIMEDOUT, ECONNREFUSED) all map to `closing`. Error code is lost unless explicitly threaded through the Channel notification Message.

### Race Risks

| # | Risk | Description |
|---|------|-------------|
| R4.1 | FD reuse (ABA) | Socket enters `closed`. OS recycles the FD integer. Poller has a pending event tagged with the old FD/SequenceNumber. If TriggeredChannel deregistration happens after `close(fd)`, the OS may already have assigned the FD to a new Socket. The new Socket's events could be misrouted. **Rule: deregister from Poller before calling `close(fd)`.** SequenceNumber validation provides a second layer of protection but deregistration-before-close is the primary guard. |
| R4.2 | `send()` on a `closing` Socket | Reactor receives a write-readiness event for a Socket that has transitioned to `closing` since the event was queued. Must check Socket state before dispatching send. |
| R4.3 | `connect()` timeout | Non-blocking `connect()` may never produce a readiness event if the peer is unreachable. No timeout state exists in raw. The Reactor event loop must enforce a connect timeout and drive the Socket to `closing` if exceeded. |
| R4.4 | `accept()` on a closing Listener | Listener Socket enters `closing`. A pending accept event is in the Poller queue. Reactor dispatches it. `accept()` returns a new FD. Engine must immediately close the new FD — it has no Channel to attach to. Unchecked, this creates a leaked FD. |

---

## SM5 — TriggeredChannel

**Raw (P2_raw.md line 60):** `registered → active → deregistered`

"Active" conflates three distinct sub-states. "Registered" conflates allocation with Poller registration.

### States

| State | Description |
|-------|-------------|
| `allocated` | Heap allocation complete. Stable address. Not yet in Poller maps. |
| `registered` | Added to Poller dual-map (ChannelNumber → SequenceNumber → `*TriggeredChannel`). Kernel interest armed. `TriggerFlags` = 0. |
| `idle` | Registered. No pending events. Poller not signaling. |
| `triggered` | Poller event fired. `TriggerFlags` non-zero. Queued for Reactor dispatch. |
| `dispatching` | Reactor is executing the event handler. `TriggerFlags` being processed. |
| `deregistering` | Channel close initiated. Removal from Poller maps in progress. May have pending events in OS queue. |
| `deregistered` | Removed from maps. All subsequent OS events will fail SequenceNumber validation. Memory still allocated. |
| `freed` | Heap memory released. Address no longer valid. No reference to this object may exist. |

### Transitions

| From | Event | Guard | To |
|------|-------|-------|----|
| — | Channel enters `opened` | — | `allocated` |
| `allocated` | `Poller.register(ch, seqn, flags)` | epoll_ctl / kevent ok | `registered` → `idle` |
| `allocated` | `Poller.register()` fails | OS error | `freed` (Channel enter `closed`) |
| `idle` | OS event fires | SequenceNumber matches | `triggered` |
| `idle` | OS event fires | SequenceNumber mismatch | `idle` (event discarded — ABA guard) |
| `triggered` | Reactor picks up event | — | `dispatching` |
| `dispatching` | Handler completes, re-arm | TriggerFlags processed | `idle` |
| `dispatching` | Channel.close() arrives (next loop) | dispatch complete | `deregistering` |
| `idle` | Channel.close() | — | `deregistering` |
| `deregistering` | `Poller.deregister()` | epoll_ctl DEL / kevent delete ok | `deregistered` |
| `deregistered` | No pending events confirmed | — | `freed` |

### Missing States

1. **`allocated` absent in raw.** Object exists before Poller registration. Poller.register() can fail; this failure path has no representation.
2. **`idle` vs `triggered` absent in raw.** Both collapsed into "active". They have different meanings for the Reactor loop and different guards.
3. **`dispatching` absent in raw.** While dispatching, the object must not be deregistered mid-dispatch. This state enforces that sequencing.
4. **`deregistering` (in-progress) absent in raw.** One-step deregistration is unsafe if events are in flight. The OS may deliver events after `epoll_ctl DEL` is called but before the FD is closed.
5. **`freed` absent in raw.** The memory release step is distinct from the logical deregistration. Objects must not be freed while any reference exists.

### Race Risks

| # | Risk | Description |
|---|------|-------------|
| R5.1 | OS event after `deregistering` | After `epoll_ctl DEL`, the OS may still deliver one final event batch. The Object Map lookup will find no entry (SequenceNumber is gone). Event is discarded. This is correct, but only if the deregistration removes the SequenceNumber from the Object Map atomically with the epoll_ctl call, within the same Reactor loop iteration. |
| R5.2 | Poller map mutation during iteration | Reactor iterates triggered events. A connect-accept event causes a new TriggeredChannel to be inserted into the Object Map. Standard hash map `swapRemove` invalidates existing pointers. This is avoided by the dual-map design (pointer indirection via SequenceNumber), but only if all existing `*TriggeredChannel` pointers are resolved before any map mutation. Must resolve pointers before dispatching handlers. |
| R5.3 | Deregistration during `dispatching` | Cannot happen concurrently (Reactor is single-threaded). Can happen on the next loop iteration if Channel.close() arrives via Mailbox during dispatch. The `dispatching` state must complete fully before `deregistering` begins. The Reactor must check the Mailbox after dispatch, not during. |

---

## SM6 — Mailbox

**Raw:** No explicit lifecycle states. Implied by P2_raw.md lines 33–36, 55–56, 70.

### States

| State | Description |
|-------|-------------|
| `open-empty` | Accepting sends. No items available to receive. Callers block on `mbox_wait_receive()`. |
| `open-nonempty` | Items queued. `mbox_wait_receive()` will return immediately. |
| `draining` | No new sends accepted. Existing items can still be received. Entered when Engine enters `draining`. |
| `closed` | No sends or receives. Any blocked `mbox_wait_receive()` returns `.Closed`. |

### Transitions

| From | Event | Guard | To |
|------|-------|-------|----|
| — | `Mailbox.create()` | — | `open-empty` |
| `open-empty` | `mbox_send(item)` | Mailbox open | `open-nonempty` |
| `open-nonempty` | `mbox_send(item)` | Mailbox open | `open-nonempty` |
| `open-nonempty` | `mbox_wait_receive()` | items remain | `open-nonempty` or `open-empty` |
| `open-empty` | `mbox_wait_receive()` | queue empty | blocks (stay `open-empty`) |
| `open-empty` | Engine enters `draining` | — | `draining` |
| `open-nonempty` | Engine enters `draining` | — | `draining` |
| `draining` | `mbox_wait_receive()` | items remain | `draining` |
| `draining` | last item received | queue empty | `closed` |
| `draining` | force-close (Engine.destroy timeout) | — | `closed` (items leaked) |
| `closed` | `mbox_send(item)` | — | error; item stays with caller |
| `closed` | `mbox_wait_receive()` | — | returns `.Closed` immediately |

### Missing States

1. **`draining` absent in raw.** P2_raw.md line 62 says "Engine drains in-flight Messages before final shutdown" but does not express this as a Mailbox state. Without `draining`, the transition from "Engine.destroy() called" to "Mailbox closed" is instantaneous in the raw model — which is incorrect. Items in the queue would be lost.

### Race Risks

| # | Risk | Description |
|---|------|-------------|
| R6.1 | `mbox_send()` after Engine.destroy() | Application thread calls `post()` → `mbox_send()` after Engine has entered `draining` or `destroyed`. Mailbox is `draining` or `closed`. `mbox_send()` must return an error and leave ownership with the caller. If it silently drops the item, the caller's `MayItem` is nil but the Message was lost — not pooled, not delivered. |
| R6.2 | Application blocked in `waitReceive()` during Engine.destroy() | Engine enters `draining`. Application is blocked on `mbox_wait_receive()`. Engine must send a sentinel `channel_closed` Message or rely on Mailbox returning `.Closed`. If application receive loop does not handle `.Closed` return, it loops forever. |
| R6.3 | Pool exhaustion deadlock | MessagePool is empty. Reactor needs a Message to deliver to the Mailbox (e.g., `channel_closed` notification). Application is blocked on `mbox_wait_receive()`. Application holds all Messages (received but not yet `put()` back). Reactor cannot deliver; application cannot unblock. **Circular dependency. Hard deadlock.** Mitigation: reserve a fixed count of Messages for engine-internal notifications; do not draw from the same pool the application depletes. |

---

## Cross-Machine Dependencies

State transitions in one machine trigger required transitions in another.
These are enforcement points, not options.

| Trigger | In | Required Action | In |
|---------|----|-----------------|----|
| Channel enters `closed` | SM2 | Deregister TriggeredChannel | SM5 |
| Channel enters `closed` | SM2 | Release ChannelNumber | SM2 |
| Socket enters `closed` | SM4 | Must happen after TriggeredChannel enters `deregistered` | SM5 |
| Engine enters `draining` | SM1 | Mailbox enters `draining` | SM6 |
| Engine enters `draining` | SM1 | No new `post()` accepted on any ChannelGroup | SM3 |
| Engine enters `destroyed` | SM1 | All TriggeredChannels enter `freed` | SM5 |
| Engine enters `destroyed` | SM1 | All Mailboxes enter `closed` | SM6 |
| TriggeredChannel enters `dispatching` | SM5 | Socket I/O executes (send/recv) | SM4 |
| Message enters `owned-engine` via `post()` | SM3 | Notifier.notify() wakes Reactor | (Notifier) |

**Critical ordering constraint (R4.1 and R5.1):**

```
Channel.close() called
  → SM5: TriggeredChannel enters deregistering
  → Poller.deregister() (epoll_ctl DEL)
  → SM5: TriggeredChannel enters deregistered
  → SM4: Socket.close(fd) called (SO_LINGER=0)
  → SM4: Socket enters closed
  → SM5: TriggeredChannel enters freed
```

Reversing steps 3 and 4 (closing FD before deregistering) exposes ABA risk R4.1.
The Socket FD must never be closed before the Poller deregistration completes.

---

## Summary of Detected Issues

### Missing States (13 total)

| SM | Missing State | Consequence if Absent |
|----|--------------|----------------------|
| SM1 | `starting` | Thread startup failures are invisible |
| SM1 | `faulted` | Reactor crash has no recovery path |
| SM2 | `connecting` | Connect failure has no defined error path |
| SM2 | `listening` | Listener activation failure has no defined error path |
| SM2 | `handshaking` | Hello/Welcome failure has no defined error path |
| SM2 | `closing` | Negotiated shutdown has no timeout or forced-close path |
| SM2 | `error` | All Channel failures are silent `closed`; application cannot distinguish normal from error |
| SM3 | `owned-engine` | Post-`post()` access is undefined but undetectable |
| SM3 | `nonexistent` | Direct-allocated Messages have no lifecycle representation |
| SM4 | `binding` | Bind failure (address in use) has no defined path |
| SM4 | `connected` | Channel-level `ready` and Socket-level connectivity are conflated |
| SM5 | `deregistering` | Deregistration is treated as atomic; OS may deliver one final event batch |
| SM6 | `draining` | Mailbox closure is treated as instantaneous; queued items would be lost |

### Race Risks (22 total)

| ID | SM | Name | Severity |
|----|-----|------|---------|
| R1.1 | Engine | `post()` during `draining` | High — ownership loss |
| R1.2 | Engine | `get()` during `draining` | Medium — policy gap |
| R1.3 | Engine | `waitReceive()` infinite block on drain | High — liveness failure |
| R1.4 | Engine | `destroy()` called twice | Low — undefined behavior |
| R2.1 | Channel | Simultaneous Bye | High — undefined protocol behavior |
| R2.2 | Channel | `post()` while `closing` | Medium — silent discard |
| R2.3 | Channel | ChannelNumber reuse with stale Mailbox | High — message misrouting |
| R2.4 | Channel | Application state clear ordering | High — stale state |
| R2.5 | Channel | `connect()` during Engine `draining` | Medium — resource leak |
| R3.1 | Message | Raw pointer copy before `post()` | High — use-after-transfer |
| R3.2 | Message | nil not checked after `get()` | Medium — nil dereference |
| R3.3 | Message | `owned-engine` lost on Engine `destroy()` | High — ownership loss |
| R4.1 | Socket | FD reuse (ABA) | Critical — misrouted I/O |
| R4.2 | Socket | `send()` on `closing` Socket | Medium — dropped I/O |
| R4.3 | Socket | `connect()` timeout | Medium — hung state |
| R4.4 | Socket | `accept()` on closing Listener | Medium — FD leak |
| R5.1 | TriggeredChannel | OS event after `deregistering` | Medium — ABA residual |
| R5.2 | TriggeredChannel | Poller map mutation during iteration | Critical — use-after-free |
| R5.3 | TriggeredChannel | Deregistration during `dispatching` | Medium — ordering |
| R6.1 | Mailbox | `mbox_send()` after `destroy()` | High — ownership loss |
| R6.2 | Mailbox | Application blocked through Engine drain | High — liveness failure |
| R6.3 | Mailbox | Pool exhaustion deadlock | Critical — hard deadlock |
