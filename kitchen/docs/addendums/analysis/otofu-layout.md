# 📦 otofu (final practical layout)

```text
otofu/
├── frame/                     ← 🔴 CORE (what you actually need now)
│   ├── frame.odin            ← main struct (starts with Polynode)
│   ├── header_binary.odin    ← binary header definition
│   ├── header_text.odin      ← HTTP-like headers
│   ├── body.odin             ← blob handling
│   ├── codec.odin            ← encode/decode (read/write)
│   └── doc.odin              ← format description (important)
│
├── types/                    ← shared constants used by frame
│   ├── opcodes.odin
│   ├── flags.odin
│   ├── identifiers.odin
│   └── errors.odin
│
├── internal/                 ← 🔒 not part of public API
│   └── (empty for now or minimal)
│
├── engine/                   ← 🚧 future (NOT used by starter)
│   ├── engine.odin
│   └── channel_group.odin
│
├── transport/                ← 🚧 future (optional)
│   ├── protocol.odin
│   └── handshake.odin
│
├── README.md
└── LICENSE
```

---

# 🧠 Key points (important)

## 1. frame is first-class and standalone

* everything needed for message format is inside `frame/`
* no runtime, no engine, no transport mixed in

---

## 2. polynode comes from matryoshka (via collections)

Inside `frame.odin`:

```odin
import pn "matryoshka/polynode"
```

And then:

```odin
Frame :: struct {
    node: pn.Polynode,   // first field (your invariant)
    header_bin: Header_Binary,
    header_txt: Header_Text,
    body: []u8,
}
```

---

## 3. strict boundary

Inside `frame/`:

### ✔ allowed:

* `matryoshka/polynode`
* `otofu/types`

### ❌ NOT allowed:

* mailbox
* pool
* runtime logic
* transport
* engine

---

## 4. types are separated

So frame doesn’t become messy with:

* flags
* opcodes
* identifiers

---

## 5. internal is intentionally minimal

* don’t move logic there “just in case”
* only add when something is truly private

---

# 📦 How it is used from your starter repo

```text
vendor/
  otofu/
    frame/
  matryoshka/
  odin-http/
```

Build with:

```bash
-collection:otofu=vendor/otofu
-collection:matryoshka=vendor/matryoshka
```

---

# 🧠 Why this layout works (practical)

* you work only in `frame/` now
* no confusion where ABI lives
* easy to vendor
* easy to evolve
* no fake modularity

---

# 🔥 What you intentionally delay

* engine complexity
* transport layer
* protocol negotiation
* channel management

All of that stays out until needed.

---

# 🧠 One-line truth

> `otofu` is a container, but `frame/` is the product — everything else can grow later without touching it.
