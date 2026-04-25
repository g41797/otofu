# P6 Reactor Model

Sources: P2_state_machines.md, P2_ownership.md, P3_ownership.md, P4_matryoshka_mapping.md
Prior phases: P1–P5

---

## Overview

The Reactor is the single I/O thread in otofu. It owns every OS file descriptor from creation to close. No other thread touches socket state. No lock guards the I/O path.

The Reactor runs a deterministic phase loop. Each phase has a fixed input, a fixed output, and a fixed ordering constraint relative to adjacent phases. No phase is optional. No phase may be reordered. The loop exits only on Engine drain completion.

All cross-thread data movement uses the Mailbox. The Reactor never blocks on a Mailbox. It never blocks on a Pool. Its only blocking call is `Poller.wait`.

---

## Reactor Thread Structure

The Reactor owns the following exclusively:

| Component | Role | Notes |
|-----------|------|-------|
| `Poller` | OS event demultiplexer | epoll / kqueue / wepoll; one per Engine |
| `Notifier` | Socket-pair cross-thread wake | read socket registered with Poller |
| `TriggeredChannel Pool` | Allocation and recycling of TriggeredChannels | Reactor-internal; never exposed to application |
| `Reserved Message Pool` | Engine-internal notification Messages | Fixed pre-allocated count; never drawn from by application (V-MP2) |
| `Reactor Inbox Mailbox` | App→Reactor command and data queue | Drained non-blocking (`try_receive_batch`) only |
| Per-ChannelGroup `Outbox Mailbox` | Reactor→App delivery | `mbox_send` only; application calls `mbox_wait_receive` |
| `Dual-Map` | ChannelNumber→SequenceNumber + SequenceNumber→*TriggeredChannel | Reactor-internal hash maps; ABA guard |
| All `Socket` instances | OS FDs | Created, operated, and closed exclusively on this thread |
| All `TriggeredChannel` instances | Heap-pinned Poller registrations | Stable address from allocation to `pool_put` |
| All `Channel` instances | Per-ChannelGroup Channel lists | Reactor owns; application holds non-owning handle |

**Single-thread invariant:** No component in this list may be read or written by any other thread. The Mailboxes are the only structures shared between threads — and they are only accessed via their thread-safe API (`mbox_send`, `try_receive_batch`, `mbox_wait_receive`).

---

## Event Loop — Step by Step

### STARTUP (one-time, before loop entry)

```
S1. Allocate and initialize Poller (epoll_create / kqueue / wepoll)
S2. Create Notifier socket pair; set both sockets non-blocking
S3. Register Notifier.read_fd with Poller (READ interest)
S4. Pre-allocate TriggeredChannel Pool (Reactor-internal)
S5. Pre-allocate Reserved Message Pool (fixed count, V-MP2)
S6. Create Reactor Inbox Mailbox
S7. Initialize dual-map (two hash maps, empty)
S8. Initialize per-iteration scratch state:
    - io_events[]    ← raw (seqn, flags) pairs from Poller
    - resolved[]     ← *TriggeredChannel pointers, validated
    - pending_close[] ← Channels requiring close this iteration
    - inbox_pending  ← bool
S9. Signal Engine: Reactor ready (Engine transitions starting → running)
```

---

### LOOP ITERATION

Each iteration executes all nine phases in order. No phase is skipped.

---

#### Phase 1 — COMPUTE TIMEOUT

```
Scan all open Channels for earliest pending deadline:
  - Channels in state `connecting`:   connect_timeout deadline
  - Channels in state `handshaking`:  handshake_timeout deadline
  - Channels in state `closing`:      bye_timeout deadline

next_deadline = min(all active deadlines)
timeout_ms    = max(0, next_deadline - now())   ← clamped; 0 = no-wait poll

If Engine.state == `draining` and no open Channels and inbox empty:
  timeout_ms = 0   ← drain completion check; exit immediately
```

**Output:** `timeout_ms` for Phase 2.

---

#### Phase 2 — POLL

```
event_batch = Poller.wait(timeout_ms)
  ← blocks on epoll_wait / kevent / NtRemoveIoCompletion
  ← returns when: event fires, timeout expires, or interrupt
```

**Output:** Raw event batch. May be empty (timeout).

**Constraint:** This is the only blocking call in the Reactor. All other operations are non-blocking.

---

#### Phase 3 — CLASSIFY

```
inbox_pending = false
clear(io_events[])

For each raw_event in event_batch:
  if raw_event.fd == Notifier.read_fd:
    inbox_pending = true
    // do not add to io_events — Notifier is not a Channel
  else:
    // raw_event carries (seqn, flags) embedded by Poller at registration
    append (raw_event.seqn, raw_event.flags) to io_events[]
```

**Output:** `inbox_pending` flag; `io_events[]` populated.

**Note:** The Notifier may fire together with I/O events in the same batch. Both are handled in this pass. Separating them is mandatory — Notifier events are not I/O events.

---

#### Phase 4 — CHECK TIMEOUTS

```
now = current_time()

For each open Channel ch:
  switch ch.state:
    case `connecting`:
      if now >= ch.connect_deadline:
        append ch to pending_close[]
        ch.close_reason = .ConnectTimeout
    case `handshaking`:
      if now >= ch.handshake_deadline:
        append ch to pending_close[]
        ch.close_reason = .HandshakeTimeout
    case `closing`:
      if now >= ch.bye_deadline:
        append ch to pending_close[]
        ch.close_reason = .ByeTimeout   ← forced close; no ByeResponse needed
```

**Output:** `pending_close[]` may have initial entries from expired timers.

**Ordering rule:** Timeouts are collected before inbox drain. Commands processed in Phase 5 may also add to `pending_close[]` (e.g., application-initiated close). Both sources merge into the same list; Phase 8 processes all of them uniformly.

---

#### Phase 5 — DRAIN INBOX

Executed only if `inbox_pending == true`.

```
5a. Drain Notifier read socket:
    loop: recv(Notifier.read_fd, buf, MAX_DRAIN) until EAGAIN
    // discard all bytes — the socket is a wake-only mechanism

5b. Batch-receive from Reactor inbox:
    batch = try_receive_batch(reactor_inbox)
    // non-blocking; returns all currently queued Messages atomically

5c. For each Message m in batch (in FIFO order):
    dispatch on m.header.opcode:

    case HelloRequest:
      // Application requests new outbound connection
      ch = allocate_channel(m.channel_group_ref)
      sk = create_socket(SOCK_STREAM, nonblocking=true)
      tc = pool_get(tc_pool, TriggeredChannelId, .Available_Or_New)
      tc.seq = next_sequence_number()
      tc.channel_num = ch.number
      dual_map.insert(ch.number, tc.seq, tc)
      Poller.register(sk.fd, tc.seq, READ|WRITE|ET)
      sk.connect(m.address)   ← non-blocking; returns immediately
      ch.state = `connecting`
      ch.connect_deadline = now() + connect_timeout
      pool_put(reserved_pool, &m)   ← return control Message to reserved pool

    case WelcomeRequest:
      // Application requests Listener (passive open)
      ch = allocate_channel(m.channel_group_ref)
      sk = create_socket(SOCK_STREAM, nonblocking=true)
      tc = pool_get(tc_pool, TriggeredChannelId, .Available_Or_New)
      tc.seq = next_sequence_number()
      tc.channel_num = ch.number
      dual_map.insert(ch.number, tc.seq, tc)
      sk.bind(m.address)
      sk.listen(backlog)
      Poller.register(sk.fd, tc.seq, READ|ET)
      ch.state = `listening`
      pool_put(reserved_pool, &m)

    case ByeRequest:
      // Application initiates graceful close
      ch = channel_by_number(m.channel_num)
      if ch == nil or ch.state != `ready`:
        pool_put(reserved_pool, &m)
        break
      enqueue_outbound(ch, &m)   ← queue ByeRequest for send in Phase 7
      ch.state = `closing`
      ch.bye_deadline = now() + bye_timeout

    case Request / Response / Signal:
      // Application data: enqueue for outbound send
      ch = channel_by_number(m.channel_num)
      if ch == nil or ch.state != `ready`:
        pool_put(app_message_pool, &m)   ← silently discard (TP-M5)
        break
      enqueue_outbound(ch, &m)
      Poller.rearm(ch.tc.seq, READ|WRITE|ET)   ← arm WRITE to trigger send

    case EngineShutdown:
      // Engine.destroy() injected by application into inbox
      Engine.state = `draining`
      pool_put(reserved_pool, &m)
      // do not break loop; process remaining commands first
```

**Output:** Channels may have been opened, closed, or had outbound data queued. `pending_close[]` unchanged by inbox drain (close commands are queued via ByeRequest path, not added to pending_close yet).

**Ordering invariant (from P4):** The application filled the inbox BEFORE calling `Notifier.notify()`. By the time Phase 5 runs, all messages the application intended to send in this wake cycle are already in the inbox. `try_receive_batch` captures them atomically.

**No map mutation after Phase 5:** All dual-map insertions happen in Phase 5 (new channels) and Phase 8 (removals). Phase 6 and Phase 7 only read the map. This prevents R5.2.

---

#### Phase 6 — RESOLVE EVENTS

```
clear(resolved[])

For each (seqn, flags) in io_events[]:
  tc_ptr = dual_map.seqn_to_tc[seqn]
  if tc_ptr == nil:
    discard   ← ABA guard: stale event from deregistered channel
    continue
  if tc_ptr.state != `idle` and tc_ptr.state != `triggered`:
    discard   ← channel is closing; event is irrelevant
    continue
  tc_ptr.pending_flags = flags
  tc_ptr.state = `triggered`
  append tc_ptr to resolved[]
```

**Output:** `resolved[]` contains only `*TriggeredChannel` pointers that are valid and current.

**Critical rule:** ALL resolutions complete before ANY dispatch begins. This is R5.2: resolving pointers before any handler can insert or remove map entries prevents use-after-free on the hash map's internal storage. The dual-map must not be mutated between Phase 6 start and Phase 7 end.

---

#### Phase 7 — DISPATCH I/O

```
For each tc_ptr in resolved[]:
  tc = *tc_ptr
  ch = channel_by_number(tc.channel_num)
  tc.state = `dispatching`

  if tc.pending_flags has ERROR or HUP:
    ch.close_reason = .NetworkError
    append ch to pending_close[]
    tc.state = `idle`   ← will be deregistered in Phase 8
    continue

  // --- WRITE path ---
  if tc.pending_flags has WRITE:
    switch ch.state:

      case `connecting`:
        // Complete non-blocking connect
        err = getsockopt(ch.socket.fd, SO_ERROR)
        if err != 0:
          ch.close_reason = .ConnectFailed
          append ch to pending_close[]
          tc.state = `idle`
          continue
        // TCP handshake complete at socket level
        ch.state = `handshaking`
        ch.handshake_deadline = now() + handshake_timeout
        // Send HelloRequest to peer (from reserved pool)
        hello_msg = pool_get(reserved_pool, MessageId, .Available_Only)
        if hello_msg == nil:
          ch.close_reason = .InternalResourceExhausted
          append ch to pending_close[]
          tc.state = `idle`
          continue
        hello_msg.header.opcode = .HelloRequest
        enqueue_outbound_front(ch, &hello_msg)   ← front: HelloRequest before any app data
        Poller.rearm(tc.seq, READ|WRITE|ET)

      case `ready` or `closing`:
        // Drain outbound send queue
        loop while ch.outbound_queue not empty:
          m = peek_outbound(ch)
          n = send(ch.socket.fd, m.encoded_bytes_remaining)
          if n == EAGAIN: break   ← kernel buffer full; wait for next WRITE event
          if n < 0:
            ch.close_reason = .SendError
            append ch to pending_close[]
            break
          advance_send_cursor(ch, n)
          if send_complete(ch):
            pop_outbound(ch)
            pool_put(app_message_pool, &m)   ← message delivered to wire; return to pool
        if ch.outbound_queue empty:
          Poller.rearm(tc.seq, READ|ET)   ← disarm WRITE; no data pending

  // --- READ path ---
  if tc.pending_flags has READ:
    switch ch.state:

      case `listening`:
        // Accept new connection from Listener socket
        loop:
          new_fd = accept(ch.socket.fd)
          if new_fd == EAGAIN: break
          if new_fd < 0:
            // Accept error; Listener stays open
            break
          // Allocate IO Server Channel
          new_ch  = allocate_channel(ch.channel_group_ref)
          new_sk  = wrap_accepted_fd(new_fd, nonblocking=true)
          new_tc  = pool_get(tc_pool, TriggeredChannelId, .Available_Or_New)
          new_tc.seq  = next_sequence_number()
          new_tc.channel_num = new_ch.number
          dual_map.insert(new_ch.number, new_tc.seq, new_tc)
          Poller.register(new_fd, new_tc.seq, READ|WRITE|ET)
          new_ch.state = `opened`
          // Server waits for HelloRequest from peer; will arrive via READ
          new_ch.handshake_deadline = now() + handshake_timeout
          // NOTE: new_ch is added to dual-map NOW; Phase 6 resolve on next iteration

      case `handshaking`:
        recv and decode frame from socket
        on decode error:
          ch.close_reason = .ProtocolError
          append ch to pending_close[]
        on HelloRequest received (server role):
          // Send HelloResponse
          resp = pool_get(reserved_pool, MessageId, .Available_Only)
          resp.header.opcode = .HelloResponse
          enqueue_outbound_front(ch, &resp)
          Poller.rearm(tc.seq, READ|WRITE|ET)
          ch.state = `ready`
          // Deliver WelcomeResponse notification to app
          notif = pool_get(reserved_pool, MessageId, .Available_Only)
          notif.header.opcode = .WelcomeResponse
          notif.channel_num   = ch.number
          mbox_send(ch.channel_group.outbox, &notif)
        on HelloResponse received (client role):
          ch.state = `ready`
          // Deliver HelloResponse to app
          mbox_send(ch.channel_group.outbox, &m_decoded)

      case `ready`:
        recv and decode frame from socket
        on complete Message:
          mbox_send(ch.channel_group.outbox, &decoded_msg)
        on ByeRequest received:
          // Peer initiated graceful close
          ch.state = `closing`
          ch.bye_deadline = now() + bye_timeout
          // Send ByeResponse
          resp = pool_get(reserved_pool, MessageId, .Available_Only)
          resp.header.opcode = .ByeResponse
          enqueue_outbound_front(ch, &resp)
          Poller.rearm(tc.seq, READ|WRITE|ET)
          // Queue ByeSignal notification to app
          sig = pool_get(reserved_pool, MessageId, .Available_Only)
          sig.header.opcode = .ByeSignal
          sig.channel_num   = ch.number
          mbox_send(ch.channel_group.outbox, &sig)

      case `closing`:
        recv and decode frame from socket
        on ByeResponse received:
          // Our ByeRequest was acknowledged
          append ch to pending_close[]   ← graceful close complete
        on any other frame:
          discard   ← peer is still sending; drain and ignore

  tc.state = `idle`   ← dispatch complete for this tc
```

**Output:** `pending_close[]` contains all Channels that must be closed this iteration. Outbox Mailboxes have been populated with delivered Messages and notifications.

**No dual-map mutation in this phase:** accept() adds new entries (Phase 7 inserts new tc/ch). This is permitted — new entries are inserted, never removed here. The resolved[] list was built in Phase 6 before any inserts. Iterating `resolved[]` is safe because it is a fixed snapshot from Phase 6. New channels inserted in Phase 7 are not in `resolved[]` and will not be dispatched until the next iteration.

---

#### Phase 8 — PROCESS PENDING CLOSES

Executed for each Channel in `pending_close[]`. Each Channel is fully torn down before proceeding to the next.

```
For each Channel ch in pending_close[]:

  // Step 1: drain any remaining outbound data (optional; typically skipped on error close)
  if ch.close_reason == .Normal or ch.close_reason == .ByeComplete:
    // best-effort flush; limited iterations
    flush_outbound_queue(ch)
  else:
    // Error or timeout close: discard outbound queue
    while ch.outbound_queue not empty:
      m = pop_outbound(ch)
      pool_put(app_message_pool, &m)

  // Step 2: discard any partial receive buffer
  reset_recv_buffer(ch)

  // Step 3: deregister TriggeredChannel from Poller (MUST happen before close(fd))
  tc = ch.triggered_channel
  Poller.deregister(tc.seq)      ← epoll_ctl DEL / kevent DELETE
  dual_map.remove_seqn(tc.seq)   ← remove seqn → *tc mapping
  dual_map.remove_chan(ch.number) ← remove channel_num → seqn mapping
  tc.state = `deregistered`

  // Step 4: close OS file descriptor (safe now: FD no longer in kernel interest set)
  setsockopt(ch.socket.fd, SO_LINGER, {on=true, linger=0})
  close(ch.socket.fd)
  ch.socket.state = `closed`

  // Step 5: return TriggeredChannel to pool (safe now: deregistered, FD closed)
  tc.state = `freed`
  pool_put(tc_pool, &tc_item)   ← tc address is stable; pool links it into free-list

  // Step 6: deliver channel_closed notification to application
  notif = pool_get(reserved_pool, MessageId, .Available_Only)
  if notif != nil:
    notif.header.opcode = .ByeSignal   ← reuse ByeSignal as channel_closed token
    notif.channel_num   = ch.number
    notif.close_reason  = ch.close_reason
    mbox_send(ch.channel_group.outbox, &notif)
  // If reserved pool is empty: notification is lost — this is a capacity planning failure
  // (V-MP2: reserved pool must be sized to cover max simultaneous closes per loop)

  // Step 7: release ChannelNumber AFTER notification is sent
  // (app uses the number from the notification to clear its state)
  release_channel_number(ch.number)

  // Step 8: remove Channel from ChannelGroup list and free Channel struct
  ch.state = `closed`
  remove_from_channel_list(ch.channel_group, ch)
  free_channel(ch)
```

**Output:** All closing Channels are fully torn down. All dual-map entries are consistent. All reserved Message pool items used for notifications are in Outbox Mailboxes (owned by application after delivery). ChannelNumbers are available for reuse.

**Ordering is mandatory:** The sequence deregister → close(fd) → pool_put is the critical ordering from P2 (R4.1, R5.1). Reversing any step is a violation. Close before deregister = ABA exposure. pool_put before deregistered = stale pointer in Poller dispatch.

---

#### Phase 9 — DRAIN CHECK

```
If Engine.state != `draining`:
  clear scratch state (io_events, resolved, pending_close)
  goto Phase 1

// Engine is draining:

If open_channel_count() > 0:
  // Force-close all remaining open Channels
  For each open Channel ch:
    ch.close_reason = .EngineDraining
    append ch to pending_close[]
  execute Phase 8 for each newly added close
  // After Phase 8: all channels are closed

// All channels closed. Drain outbox Mailboxes.
For each ChannelGroup cg:
  mbox_close(cg.outbox)
  // mbox_close returns remaining queued Messages as list.List
  drain_and_free_list(returned_messages, reserved_pool)

// Drain Reactor inbox
mbox_close(reactor_inbox)
drain_and_free_list(returned_messages, app_message_pool)

// Return all remaining reserved pool Messages
pool_close(reserved_pool)
// pool_close returns all stored items; dispose each
free_reserved_pool_items(...)
matryoshka_dispose(&reserved_pool_item)

// Return all remaining TriggeredChannels (should be zero after channel close)
pool_close(tc_pool)
matryoshka_dispose(&tc_pool_item)

// Close Notifier and Poller
close(Notifier.read_fd)
close(Notifier.write_fd)
Poller.close()

// Signal Engine: Reactor thread exiting
Engine.state = `destroyed`
// Reactor thread exits
return
```

**Exit condition:** Only one path exits the loop — Phase 9 after all Channels are closed, all Mailboxes are drained, all Pools are disposed. The loop is not exited on error, timeout, or signal — only on clean drain completion.

---

## Loop at a Glance

```
STARTUP
  S1–S9: initialize Poller, Notifier, pools, maps, mailboxes
  signal Engine: running

LOOP:
  Phase 1:  Compute timeout from pending Channel deadlines
  Phase 2:  Poller.wait(timeout)           ← ONLY blocking call
  Phase 3:  Classify events (Notifier vs I/O)
  Phase 4:  Check expired timers → pending_close[]
  Phase 5:  Drain inbox (if Notifier fired)
              ← drain Notifier socket
              ← try_receive_batch
              ← process commands (open/close/send)
              ← dual-map INSERTs happen here (new channels)
  Phase 6:  Resolve I/O events via SequenceNumber
              ← ALL resolutions before ANY dispatch (R5.2)
  Phase 7:  Dispatch I/O per resolved TriggeredChannel
              ← accept / connect complete / recv / send
              ← deliver to app outbox Mailboxes
              ← pending_close[] populated on error/protocol close
  Phase 8:  Process pending closes
              ← deregister → close(fd) → pool_put (mandatory order)
              ← deliver channel_closed notification
              ← release ChannelNumber
              ← dual-map REMOVEs happen here
  Phase 9:  Drain check
              ← if draining: force-close remaining channels → exit
              ← else: clear scratch; loop to Phase 1
```

---

## Trigger System

### Components

```
Poller
  └── kernel interest set (epoll / kqueue / wepoll)
      ├── Notifier.read_fd   (READ interest; special-cased in Phase 3)
      └── Socket.fd × N      (READ|WRITE|ET interest, one per Channel)

TriggeredChannel (one per registered Socket)
  ├── PolyNode at offset 0     (Matryoshka Pool item)
  ├── seq: SequenceNumber      (u64 monotonic; ABA token)
  ├── channel_num: u16
  ├── pending_flags: TriggerFlags
  └── state: TC_State          (SM5)

Dual-Map (Reactor-internal, two hash maps)
  ├── channel_num → seqn        (for close path: look up TC from Channel)
  └── seqn        → *TC         (for dispatch path: validate and resolve)
```

### SequenceNumber Protocol

Every TriggeredChannel is assigned a new SequenceNumber at registration. The SequenceNumber is a u64 monotonic counter. It is never reused.

```
register:
  seqn = atomic_increment(global_seqn_counter)
  dual_map.insert(channel_num, seqn, tc_ptr)
  epoll_ctl ADD (fd, seqn embedded in epoll_data.u64)

deregister:
  epoll_ctl DEL (fd removed from interest set)
  dual_map.remove(seqn)
  dual_map.remove(channel_num)
```

Any OS event that arrives after `epoll_ctl DEL` carries the old SequenceNumber. Phase 6 looks up the SequenceNumber and finds no entry — the event is discarded. This is the ABA guard.

### TriggerFlags

```
TriggerFlags :: bit_set[Flag; u8]
Flag :: enum { READ, WRITE, ERROR, HUP }
```

Mapping to OS event flags:

| OS Flag | TriggerFlag | When |
|---------|-------------|------|
| EPOLLIN / EVFILT_READ | READ | Data or connection available |
| EPOLLOUT / EVFILT_WRITE | WRITE | Send buffer available or connect complete |
| EPOLLERR / EV_ERROR | ERROR | Socket-level error |
| EPOLLHUP / EV_EOF | HUP | Peer closed connection |

Edge-triggered mode (EPOLLET / EV_CLEAR) is required. Level-triggered mode causes spurious re-entry on partially drained buffers.

### Notifier

The Notifier is not a TriggeredChannel. It is not in the dual-map. It is not in any Pool.

```
Notifier.write_fd: fd   ← application calls write(1 byte) to wake Reactor
Notifier.read_fd:  fd   ← registered with Poller as READ; Reactor drains in Phase 5a
```

`Notifier.notify()` (called by application after `mbox_send`):
```
write(Notifier.write_fd, &byte, 1)
// write may fail with EAGAIN if the pipe buffer is full.
// This is not an error: an existing unread byte already wakes the Reactor.
// The Reactor will drain the inbox regardless of how many bytes are in the pipe.
```

Reactor drains:
```
loop:
  n = recv(Notifier.read_fd, buf, MAX_DRAIN)
  if n == EAGAIN: break
```

One write per `post()` call is not required. Any number of `post()` calls between Reactor wakeups produces a single drain pass in Phase 5. The inbox `try_receive_batch` captures all queued commands regardless of how many Notifier writes occurred.

---

## Channel Lifecycle in the Reactor

The Reactor drives all Channel state transitions from P2 SM2. The application never transitions Channel state directly.

### State Ownership

| State | Driven by | Phase |
|-------|-----------|-------|
| `unassigned` → `opened` | HelloRequest or WelcomeRequest command | Phase 5 |
| `opened` → `connecting` | Socket.connect() call | Phase 5 |
| `opened` → `listening` | Socket.bind()+listen() call | Phase 5 |
| `connecting` → `handshaking` | WRITE event, getsockopt OK | Phase 7 |
| `connecting` → pending_close | WRITE event, getsockopt error, or timeout | Phase 7 / Phase 4 |
| `listening` → `ready` | READ event, after accept and HelloResponse sent | Phase 7 |
| `handshaking` → `ready` | HelloResponse received (client) or HelloResponse sent (server) | Phase 7 |
| `handshaking` → pending_close | Timeout or protocol error | Phase 4 / Phase 7 |
| `ready` → `closing` | ByeRequest sent (app) or ByeRequest received (peer) | Phase 5 / Phase 7 |
| `closing` → pending_close | ByeResponse received, or timeout | Phase 7 / Phase 4 |
| any → pending_close | ERROR or HUP event; Engine draining | Phase 7 / Phase 9 |
| pending_close → `closed` | Phase 8 close sequence | Phase 8 |

### ChannelNumber Lifecycle

```
assign:
  channel_num = allocate from number pool (range 1–65534)
  (called when Channel enters `opened`)

release:
  mark channel_num available
  (called in Phase 8, AFTER channel_closed notification is sent)
```

**Rule:** ChannelNumber is not a permanent identity. The same number may be reused by a future Channel after the application clears its state on receiving the `channel_closed` notification (R2.3).

### Accepted Channel Spawning

When a Listener dispatches an `accept()` in Phase 7, the new IO Server Channel is added to the dual-map immediately (Phase 7 inserts). The new TriggeredChannel is NOT in `resolved[]` for this iteration — it was built after Phase 6. The new Channel will receive its first dispatch only on the next iteration.

This is safe: the new Socket is already armed for READ|WRITE. If the peer sends data immediately, a new Poller event will fire on the next `Poller.wait` call.

---

## Mailbox Interaction

### Two Mailboxes per Direction

```
App → Reactor:
  reactor_inbox:        Matryoshka Mailbox (DIRECT mapping)
  Notifier.write_fd:    wake mechanism (not Matryoshka mbox_interrupt — IMPOSSIBLE)

Reactor → App (per ChannelGroup):
  cg.outbox:            Matryoshka Mailbox (DIRECT mapping)
```

### App → Reactor: Send Path

```
Application thread:
  1. m = Engine.get(strategy)               ← pool_get; m^ != nil
  2. populate m (header, meta, body)
  3. ChannelGroup.post(&m)                  ← mbox_send(reactor_inbox, &m); m^ = nil
  4. Notifier.notify()                      ← write(Notifier.write_fd, 1 byte)
```

**Ordering is mandatory:** Step 3 before step 4. If step 4 precedes step 3, the Reactor may wake, call `try_receive_batch`, find an empty inbox, and return to `Poller.wait` before step 3 enqueues the Message. The wake would be lost.

**Ownership transfer:** After step 3, `m^` is nil. The caller has no access to the Message. Engine owns it until it is delivered, returned, or discarded (TP-M2, P3).

### Reactor → App: Receive Path

```
Application thread:
  loop:
    result = ChannelGroup.waitReceive(&m, timeout)
    switch result:
      .Ok:          process m; then put(&m) or post(&m)
      .Timeout:     check external state; loop
      .Closed:      all channels closed; exit loop — DO NOT loop back
      .Interrupted: check external state; loop (not used in baseline otofu)
```

**Rule:** The receive loop MUST handle `.Closed`. Looping back on `.Closed` causes R6.2 (infinite block through Engine drain).

**Rule:** Only one application thread calls `waitReceive` on any given ChannelGroup (INV-21). Matryoshka Mailbox is MPMC but otofu restricts to single-consumer per ChannelGroup by convention (V-CG2).

### Reactor Send to Outbox

```
Reactor thread (Phase 7 or Phase 8):
  mbox_send(cg.outbox, &m)
```

`mbox_send` is the ONLY Mailbox operation the Reactor calls on the outbox. The Reactor never calls `mbox_wait_receive`, `mbox_interrupt`, or `mbox_close` on the outbox during normal operation.

`mbox_close(cg.outbox)` is called only in Phase 9 (Engine drain).

### Reactor Inbox: Non-Blocking Only

```
Reactor thread (Phase 5):
  batch = try_receive_batch(reactor_inbox)
  // non-blocking; returns immediately whether inbox is empty or not
```

The Reactor never calls `mbox_wait_receive` on any Mailbox (V-R1). The Reactor never calls `pool_get_wait` on any Pool (V-R2). These calls would block the I/O thread.

### Reserved Message Pool Usage

The reserved Message pool (V-MP2) is used exclusively by the Reactor for internal notifications:

| Notification | OpCode | Pool used |
|-------------|--------|-----------|
| Channel connected (client) | HelloResponse | reserved |
| Channel connected (server) | WelcomeResponse | reserved |
| Channel closed (any reason) | ByeSignal | reserved |
| HelloRequest to peer | HelloRequest | reserved |
| HelloResponse to peer | HelloResponse | reserved |
| ByeResponse to peer | ByeResponse | reserved |

The application never calls `pool_get` on the reserved pool. The application does receive Messages from the reserved pool (via outbox Mailbox) and must `pool_put` them back — but `pool_put` returns them to the application's main pool (not the reserved pool), since the application cannot distinguish reserved from application Messages. The reserved pool is replenished only from pre-allocated stock created at startup. This is the R6.3 mitigation.

---

## Shutdown Sequence

Shutdown is initiated by the application calling `Engine.destroy()`. This injects an `EngineShutdown` Message into the Reactor inbox.

```
Application thread:
  1. Engine.destroy()
     a. inject EngineShutdown Message into reactor_inbox via mbox_send
     b. Notifier.notify()
     c. block: wait for Reactor thread to join

Reactor thread (Phase 5 processes EngineShutdown):
  2. Engine.state = `draining`

Reactor loop continues:
  3. Phase 4 each iteration: no new connect/listen accepted (rejected in Phase 5 if state is draining)
  4. Phase 9 each iteration: all remaining open Channels are force-closed (added to pending_close)
  5. Phase 8: teardown sequence per Channel (deregister, close FD, notify app, release number)
  6. Phase 9 final: all Channels closed
     a. mbox_close all ChannelGroup outboxes (remaining queued messages returned and freed)
     b. mbox_close reactor_inbox (remaining inbox items returned and freed)
     c. pool_close reserved pool; matryoshka_dispose
     d. pool_close tc pool; matryoshka_dispose
     e. close Notifier FDs
     f. Poller.close()
     g. Engine.state = `destroyed`
     h. Reactor thread returns
  7. Application's Engine.destroy() unblocks (Reactor thread joined)
  8. Engine struct freed
```

**Drain invariant:** The application must call `waitReceive` and handle `.Closed` to fully drain the outbox before the application-side resources are valid to free. If the application exits its receive loop before the Mailbox is closed, queued Messages remain in the Mailbox when `mbox_close` drains them — they will be freed by the Reactor during Phase 9, not returned to the application pool. This is correct behavior (not a leak), but the application loses access to any payload in those Messages.

---

## Loop Invariants

Numbered. No ambiguity.

| # | Invariant |
|---|-----------|
| LI-1 | Phase 2 (`Poller.wait`) is the only call that blocks the Reactor thread. All other operations are non-blocking. |
| LI-2 | All dual-map INSERTs occur in Phase 5. All dual-map REMOVEs occur in Phase 8. No mutation occurs in Phase 6 or Phase 7. |
| LI-3 | Phase 6 resolves all TriggeredChannel pointers before Phase 7 dispatches any of them. The resolved[] list is a fixed snapshot. |
| LI-4 | Phases 4 and 7 may add Channels to pending_close[]. Phase 8 processes all of them. No Channel is closed outside Phase 8 during normal operation. |
| LI-5 | Poller.deregister() is always called before close(fd). close(fd) is always called before pool_put(tc). This ordering is unconditional. |
| LI-6 | channel_closed notification is sent BEFORE the ChannelNumber is released. The ChannelNumber is not available for reuse until after the notification is enqueued. |
| LI-7 | The Reactor never calls mbox_wait_receive or pool_get_wait. |
| LI-8 | `mbox_send(reactor_inbox)` (by application) always precedes `Notifier.notify()`. This ordering is the application's obligation; the Reactor assumes it. |
| LI-9 | The reserved Message pool is never exposed to application pool_get. Application Messages and engine-internal notification Messages draw from separate pools. |
| LI-10 | A TriggeredChannel is not returned to its pool while registered in the Poller. Deregistered state must be reached before pool_put. |
| LI-11 | SequenceNumbers are never reused. Each new TriggeredChannel allocation gets a strictly higher SequenceNumber than any previous allocation. |
| LI-12 | The loop exits only through Phase 9's drain path. No other exit point exists. |
