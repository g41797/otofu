# 📦 otofu — Repository Layout (Final)

## Overview

This repository implements **otofu**, a standalone async messaging system.

Design principles:

* frame-oriented (not stream-based)
* reactor-driven (single-threaded event loop)
* built on matryoshka (polynode + mailbox + pool)
* no application logic
* no recipes
* no transport abstraction leakage
* strict separation between public API and internal implementation

---

# 🗂️ Top-Level Structure

```
otofu/
├── engine/              # Public API: reactor (engine) + channel group
├── frame/               # Public API: frame structure + binary header
├── types/               # Public constants, enums, flags, identifiers
├── internal/            # Private implementation (reactor, IO, routing)
├── docs/                # Architecture + usage docs
├── tests/               # Unit + functional tests
├── examples/            # Minimal usage examples (NOT recipes)
├── vendor/              # Git submodules (matryoshka)
├── README.md
├── LICENSE
└── odin.mod             # Odin module definition
```

---

# 📦 1. engine/ (Public API)

```
engine/
├── engine.odin          # Reactor lifecycle (create/run/stop)
├── channel_group.odin   # User-facing API (send/receive/connect/listen)
├── channel.odin         # Channel handle (opaque ID + helpers)
├── config.odin          # Engine configuration (limits, tuning)
└── doc.odin
```

## Responsibilities

* expose minimal API:

  * create engine
  * create channel group
  * send frame
  * receive frame
  * connect / listen

* NO internal logic here

* acts as façade over internal reactor + mailboxes

---

# 📦 2. frame/ (Public API)

```
frame/
├── frame.odin           # Frame struct (polynode embedded first)
├── header.odin          # Binary header definition (16 bytes)
├── encode.odin          # BE ↔ LE conversion only
├── decode.odin          # Header read logic (no payload parsing)
└── doc.odin
```

## Responsibilities

* define **frame ABI**
* define binary header fields:

  * kind
  * flags
  * sizes
  * identifiers

## Rules

* text headers = opaque bytes
* body = opaque bytes
* no parsing beyond header

---

# 📦 3. types/ (Public API)

```
types/
├── kinds.odin           # Message kinds (Hello, Welcome, Bye, Control)
├── flags.odin           # Bit flags for header
├── channel_id.odin      # Channel identifiers
├── errors.odin          # Error/status codes
└── doc.odin
```

## Responsibilities

* shared constants between:

  * frame
  * engine
  * user code

---

# 🔒 4. internal/ (Private Implementation)

```
internal/
├── reactor/
│   ├── reactor.odin         # Main event loop
│   ├── loop.odin            # Poll loop orchestration
│   ├── dispatch.odin        # Frame routing inside reactor
│   └── state.odin           # Reactor state (connections, channels)
│
├── io/
│   ├── socket.odin          # Cross-platform socket abstraction
│   ├── reader.odin          # Frame read (header + payload)
│   ├── writer.odin          # Frame write (queue → socket)
│   └── acceptor.odin        # Listener (WelcomeRequest handling)
│
├── platform/
│   ├── poller_linux.odin    # epoll
│   ├── poller_darwin.odin   # kqueue
│   ├── poller_windows.odin  # wepoll
│   ├── poller.odin          # unified interface
│   └── notifier.odin        # wakeup mechanism
│
├── channel/
│   ├── channel_state.odin   # per-channel queue + metadata
│   ├── channel_table.odin   # channel lookup
│   └── lifecycle.odin       # Hello/Bye handling
│
├── queue/
│   ├── outbound.odin        # per-channel outbound queue
│   └── oob.odin             # OOB control queue (ByeSignal)
│
├── control/
│   ├── hello.odin           # connect logic
│   ├── welcome.odin         # listen logic
│   ├── bye.odin             # disconnect logic (OOB)
│   └── control.odin         # generic control frames
│
├── runtime/
│   ├── mailbox_bridge.odin  # matryoshka mailbox integration
│   ├── frame_pool.odin      # pool usage helpers
│   └── backpressure.odin    # pool-driven pressure handling
│
└── doc.odin
```

## Responsibilities

* everything real happens here:

  * reactor loop
  * IO
  * queueing
  * routing
  * lifecycle handling

## Rules

* NOT exposed publicly
* may change without breaking API

---

# 📦 5. vendor/

```
vendor/
└── matryoshka/              # git submodule
```

## Usage

* polynode → embedded in frame
* mailbox → reactor internal comms
* pool → frame allocation + backpressure

---

# 📦 6. tests/

```
tests/
├── unit/
│   ├── frame/
│   ├── engine/
│   └── channel/
│
├── functional/
│   ├── connect_disconnect_test.odin
│   ├── fifo_order_test.odin
│   ├── backpressure_test.odin
│   └── multi_engine_test.odin
```

---

# 📦 7. examples/

```
examples/
├── minimal_send_receive/
├── client_server/
└── multi_engine/
```

## Rules

* minimal only
* no recipes
* no business logic

---

# 📦 8. docs/

```
docs/
├── architecture.md
├── frame_spec.md
├── reactor_model.md
├── channel_model.md
└── backpressure.md
```

---

# 🔑 Key Design Rules (enforced by layout)

* frame is isolated and stable
* engine API is minimal and clean
* internal complexity is hidden
* matryoshka is used, not reimplemented
* no application concerns inside otofu

---

# 🧠 One-line structure summary

> Public surface = engine + frame + types
> Everything else = internal reactor system built on matryoshka

---
