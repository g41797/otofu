# OTofu Direction Notes

## 1. Real Question

This is not about sockets.

It is about layers.

Transport handles:

* read
* write
* readiness or completion

Conversation handles:

* who connects
* first message
* message ID
* progress
* shutdown

Tofu focuses on conversation.

That is the core idea.

---

## 2. Why Odin

Odin prefers:

* explicit code
* simple structures
* no hidden magic

This is good pressure.

If abstraction is too heavy, it will be obvious.

Otofu is not a port.
It is a test.

---

## 3. Not Advertising

Do not present a finished framework.

Show the process.

Show what breaks.
Show what simplifies.
Show what stays.

Ask questions during design.
Not after release.

---

## 4. Sparring Partner Model

Otofu is a sparring partner for tofu.

Goal:

* test assumptions
* remove accidental complexity
* keep essential parts

If something does not survive Odin style,
maybe it was not essential.

---

## 5. Reactor vs Proactor

Instead of waiting for Zig `Io.Evented`,
build Proactor model in Odin now.

Keep message layer the same.
Change only transport model.

Observe:

* state machine size
* shutdown logic
* partial write handling
* TLS integration
* backpressure

Compare with Reactor version.

---

## 6. Keep Invariants

Message rules must not change:

* message ID as job ID
* signal vs request
* structured shutdown
* channel as lifecycle boundary

If transport change affects these,
layering is wrong.

---

## 7. Why Not Wait

Waiting means designing around unknown API.

Building now gives:

* real data
* real state machines
* clear invariants

Later Zig async can adapt to proven design.

---

## 8. Final Direction

Build minimal Proactor core.
Add one stream.
Add framing.
Add message queue.
Test full lifecycle.

Do not optimize early.
Do not abstract too much.

Let the experiment show what matters.

This is engineering work.
Not language politics.
Not framework marketing.
