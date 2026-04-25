# 📦 otofu — Frame Lifecycle & Usage Contract

## Purpose

This document defines the **strict rules** governing the lifecycle, ownership, and safe usage of `Frame`.

`Frame` is:

* a transport unit (otofu)
* a structural item (`polynode`)
* a pooled resource (matryoshka)

Because of this, it has **strong constraints**.

---

# 1. Core Principles

## 1.1 Single-owner lifecycle

At any moment:

> A `Frame` has exactly **one logical owner**

Ownership transfers explicitly between:

* reactor (engine)
* channel group (user-facing)
* user code
* matryoshka pipelines (if used)

---

## 1.2 Pool-backed allocation

* All frames are allocated from a **matryoshka pool**
* Memory is **reused**
* User must **never allocate or free frame memory manually**

---

## 1.3 Exactly-once return

> Every frame MUST be returned to its originating pool **exactly once**

Violations:

* ❌ double return → undefined behavior
* ❌ leak (no return) → pool exhaustion → system stall
* ❌ foreign return → memory corruption

---

# 2. Lifecycle States

A frame moves through these states:

```text
ALLOCATED → FILLED → IN_FLIGHT → OWNED_BY_USER → RELEASED
```

---

## 2.1 ALLOCATED

* frame obtained from pool (by reactor)
* not yet visible to user

---

## 2.2 FILLED

* header decoded (BE → LE)
* text headers/body attached (opaque)

---

## 2.3 IN_FLIGHT

* frame is:

  * queued for send, OR
  * delivered via mailbox

**Rule:**

> Frame MUST NOT be mutated in this state

---

## 2.4 OWNED_BY_USER

* user receives frame via channel
* user now controls:

  * when to release
  * how to process

---

## 2.5 RELEASED

* frame returned to pool
* MUST NOT be accessed again

---

# 3. Ownership Transitions

## Receive path

```text
Reactor → Mailbox → ChannelGroup → User
```

Ownership:

* reactor → channel group → user

---

## Send path

```text
User → ChannelGroup → Reactor → Socket
```

Ownership:

* user → reactor → pool (on completion or failure)

---

# 4. Usage Modes

## 4.1 Mode A — Zero-copy (advanced)

User processes frame directly.

Example:

* pass frame into matryoshka pipeline
* reuse body/text buffers
* no duplication

### Requirements

* respect lifecycle strictly
* ensure eventual return to pool
* no mutation after enqueue

### Risks

* misuse breaks system
* pool corruption possible

---

## 4.2 Mode B — Copy & detach (safe)

User copies data out:

* extract payload
* convert to application object
* immediately release frame

### Requirements

* return frame as early as possible

### Benefits

* safe
* no coupling to pool
* easier reasoning

---

# 5. Mutation Rules

## Allowed

* mutation ONLY when:

  * frame is owned by user
  * frame is NOT enqueued

---

## Forbidden

* ❌ modifying frame after `channel_send`
* ❌ modifying frame inside reactor-owned state
* ❌ concurrent mutation

---

# 6. Cross-boundary Usage (matryoshka)

Because `Frame` is a `polynode`:

> It MAY be used outside otofu (mailboxes, pipelines, pools)

---

## When doing so:

User becomes responsible for:

* lifecycle correctness
* return-to-pool guarantee
* avoiding double ownership

---

## Strong recommendation

If unsure:

> Use **copy & detach mode**

---

# 7. Failure Semantics

otofu uses **negative completion model**

* no success ACK
* only failures are reported

---

## Possible statuses

* `DISCONNECTED`
* `POOL_EXHAUSTED`
* `DROPPED`

---

## Implication

User MUST NOT assume:

> successful send == delivered

---

# 8. Backpressure Interaction

Backpressure is enforced via pool:

* allocation failure blocks:

  * send
  * receive

---

## Consequences

* leaking frames → system stall
* slow release → throughput degradation

---

# 9. Anti-patterns (forbidden)

## 9.1 Frame caching

```text
store frame in global structure without release
```

→ pool exhaustion

---

## 9.2 Double return

```text
release(frame)
release(frame)
```

→ undefined behavior

---

## 9.3 Post-send mutation

```text
channel_send(frame)
frame.body[...] = ...
```

→ data corruption

---

## 9.4 Cross-pool misuse

```text
return frame to different pool
```

→ memory corruption

---

# 10. Recommended Practices

## Minimal safe pattern

```text
receive → process → release
```

---

## High-performance pattern

```text
receive → pipeline → release (inside pipeline)
```

---

## Decoupled pattern

```text
receive → copy → release → process copy
```

---

# 11. Debugging Guidelines

Symptoms of misuse:

* random crashes
* stalled system (no progress)
* pool exhaustion
* inconsistent message behavior

---

## First checks

* all frames returned?
* any double release?
* mutation after send?
* frames escaping ownership?

---

# 12. One-line contract

> A Frame is a pooled, single-owner, polynode-based object that must be returned exactly once, must not be mutated while in-flight, and may be used outside otofu only with full lifecycle responsibility assumed by the user.

---
