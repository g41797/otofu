# ✅ Correct understanding (your intended model)

## matryoshka

> **General-purpose execution substrate**

* operates on **polynode-based items**
* provides:

  * mailbox (communication)
  * pool (lifecycle / policy engine)
* used for:

  * pipelines
  * workflows
  * internal systems
* **no transport**
* fully usable standalone

---

## otofu

> **Transport/messaging runtime built on matryoshka**

* async inter-process / inter-machine communication
* frame/message-based (your binary header + payload model)
* internally uses:

  * matryoshka mailbox
  * matryoshka pool
* exposes:

  * engine (reactor)
  * channel group
  * channels
  * frames

👉 user does **not see internal matryoshka usage**

---

## User options (this is the key point)

User can choose:

### 1. Only matryoshka

* build pipelines
* build internal async systems
* no networking

---

### 2. Only otofu

* use messaging runtime
* no need to know matryoshka exists
* treat it as black-box engine

---

### 3. Both together

* advanced usage
* integrate:

  * pipelines (matryoshka)
  * transport (otofu)
* same conceptual base: **polynode**

---

# 🧠 Correct dependency graph

```text id="correct_model"
        matryoshka
           ↑
           │ (internal usage)
         otofu
           
User can depend on:
- matryoshka
- otofu
- or both
```

---

# ⚠️ Important nuance (now correctly framed)

This is NOT:

```text
otofu hides matryoshka completely
```

It is:

> otofu **encapsulates matryoshka for its own purposes**,
> but matryoshka remains a **first-class independent library**

---

# 🧠 Critical invariant you are enforcing

> **polynode is the universal structural primitive across the ecosystem**

Which means:

* matryoshka operates on polynode
* otofu frames are compatible with that model
* user can unify both worlds if needed

This is actually a very strong design.

---

# 🔥 Architectural consequence (important)

Because of this:

## Frame design implication

You were right earlier:

> Frame is based on polynode

So internally:

```odin
Frame :: struct {
    node: PolyNode,
    ...
}
```

And this is NOT accidental — it is the bridge.

---

# ⚠️ But here is the trade-off (you should be aware)

If you allow users to mix both worlds:

You implicitly allow:

```text
Frame ↔ matryoshka pipelines
```

Which means:

> frame is not just transport — it becomes a **pipeline-compatible item**

That’s powerful, but also:

* increases coupling
* requires discipline

---

# 🧠 Clean mental model (final)

## matryoshka

> “how items flow and are processed”

## otofu

> “how items move between processes/machines”

## frame

> “the unit that can travel and be processed”

---

# ✅ Final confirmation

Your statement:

> user can use just otofu, just matryoshka, or both

✔ confirmed
✔ consistent
✔ architecturally sound

---

# 🧠 One important recommendation (not correction)

Make this explicit in docs:

> otofu uses matryoshka internally, but does not require users to understand or use it unless they want deeper integration.

That keeps:

* beginner UX simple
* advanced usage powerful

---


Yes — this is now very clear, and it’s a strong, **intentional design choice**, not an accident.

Let me restate it precisely so we lock the contract.

---

# ✅ Confirmed model

## 1. Frame is a PolyNode

* `Frame` embeds **polynode as its structural base**
* therefore it is **natively compatible** with:

  * matryoshka mailboxes
  * matryoshka pool
  * pipelines

---

## 2. Cross-usage is allowed (by design)

Because of that:

> A `Frame` can move **outside otofu**, through:

* user-created mailboxes
* user-defined pipelines
* custom pool flows

This is **not restricted by otofu**.

---

## 3. Ownership & lifecycle constraint (important)

However:

> Frames originate from **otofu’s internal pool**

So:

* memory lifecycle is controlled by that pool
* returning to pool must happen correctly
* misuse can break system behavior

---

## 4. User decision model

You are explicitly giving the user a choice:

---

### Option A — Zero-copy / high-performance mode

* use `Frame` directly as pipeline item
* pass through mailboxes / pools
* no duplication

👉 pros:

* zero-copy
* maximum performance
* unified model (transport + processing)

👉 cons:

* must respect lifecycle rules
* tied to otofu pool semantics

---

### Option B — Decoupled mode

* convert `Frame` → user-defined object
* release frame back to pool immediately

👉 pros:

* safe
* independent lifecycle
* no coupling to otofu internals

👉 cons:

* copy cost
* extra allocation

---

# 🧠 This is the real design

> otofu does not enforce isolation — it provides **controlled escape hatch**

And explicitly says:

> “you can treat Frame as opaque transport unit OR as a first-class pipeline item”

---

# ⚠️ Critical implication (must be documented)

This creates:

## 👉 **shared-object boundary**

Between:

* transport layer (otofu)
* execution layer (matryoshka / user pipelines)

So you MUST document:

---

## Frame lifecycle contract (strict)

Something like:

* Frame must be returned to originating pool exactly once
* User must not:

  * double-return
  * leak
  * mutate after send (if already enqueued)
* If Frame leaves otofu domain:

  * user becomes responsible for lifecycle correctness

---

# 🧠 Clean wording (you can use in docs)

> Frames are polynode-based items and may be used directly in matryoshka pipelines.
> However, they are allocated from an internal pool and must obey its lifecycle rules.
> Users may either:
>
> * process frames in-place (zero-copy), or
> * convert them into application-owned objects and release them immediately.

---

# 🔥 Why this is actually a powerful design

You achieved:

* zero-copy pipeline + transport integration
* optional decoupling
* no forced abstraction penalty

This is something most systems **fail to balance**.

---

# ✅ Final confirmation

✔ Frame = PolyNode
✔ Can be used outside otofu
✔ Lifecycle is pool-bound
✔ User chooses between:

* zero-copy integration
* safe copy + detach

✔ Responsibility is explicitly shifted to user when crossing boundary

---

# 🧠 One small recommendation

Do NOT try to “protect” user too much here.

Instead:

> document clearly + provide examples of both modes

That’s enough for this level of system.

---
