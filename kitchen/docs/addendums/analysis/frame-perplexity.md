# 📄 Odin `frame` Package — High‑Level Design  - perplexity

*(Polynode‑Centric, matryoshka‑based, tofu‑inspired)*

This document serves as both:

- A **self‑contained prompt** for another AI (all context, links, and architectural constraints included).
- A **high‑level design** of the Odin `frame` package, without internal implementation details.

---

## 0. Context and Goal

You are building an **Odin Service Runtime** based on the **polynode‑centric model** from `matryoshka` and inspired by your Zig‑based `tofu` messaging library.

Your current Zig `tofu` defines a `Message` and related structures for asynchronous messaging in modular monoliths.  
You want to port the **core message and related concepts** into a **separate Odin `frame` package**, which will be part of a **template repository** for building Odin services.

The `frame` package is **solely responsible for the transport ABI** and **carrying polynode‑based items**; it must **not** depend on `matryoshka` or `runtime` logic, and must **not** implement decision logic or business semantics.

---

## 1. Architectural Rules Recap

Copy‑ready bullets you can paste into another AI prompt:

- The system is composed of four conceptual layers:
  - `frame` → transport ABI container
  - `polynode` → structural core embedded in every item
  - `matryoshka` → runtime substrate operating **only** on polynode items
  - `runtime` → orchestration + adapters + user behavior
  - `plugins` → decision logic layer (user code)

- **Core axiom**:  
  `matryoshka` operates **only** on polynode‑based items, **never** on raw frame structures.  
  Flow:  
  `frame` → polynode extraction → `matryoshka` operates on polynode items.

- **Dependencies**:
  - `runtime` → `frame`, `matryoshka`, `plugins`
  - `matryoshka` **does not** depend on `frame` or `runtime`
  - `frame` **does not** depend on `matryoshka` or `runtime`
  - `polynode` is an independent structural primitive

- **Logic placement**:
  - `polynode` has no behavior, no metadata semantics.
  - `pool` is only an evented registry emitting lifecycle events without decision logic.
  - `plugins` are the **only** source of intelligence and decisions.
  - `frame` is **pure transport ABI**.
  - `runtime` is the **orchestrator** (parsing, dispatch, handlers, transports).

- `frame` responsibilities:
  - encoding / decoding
  - packaging polynode + payload
  - language‑agnostic ABI
  - **does not** define execution, runtime logic, or policies.

- `frame` must be **based on** the `polynode` struct from `matryoshka` (see link below).

---

## 2. Relevant Links

Include these in prompts to other AIs:

- Zig `tofu` `message.zig` (source of current message API):  
  `https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/src/message.zig`

- Tofu docs: message concepts:  
  `https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/message.md`

- Tofu docs: patterns (messaging patterns, flows):  
  `https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/patterns.md`

- `matryoshka` polynode definition (Odin):  
  `https://raw.githubusercontent.com/g41797/matryoshka/refs/heads/main/polynode.odin`

- Architecture analysis mapping `tofu` ↔ `matryoshka` / `runtime` (S6):  
  `https://github.com/g41797/tofusite/blob/main/root/otofu/analysis/S6_tofu_mapping_fixed.md`

- Initial module mapping (`S1_modules_fixed.md`):  
  `https://raw.githubusercontent.com/g41797/tofusite/refs/heads/main/root/otofu/analysis/S1_modules_fixed.md?token=GHSAT0AAAAAADUVB5DSWH7CC23NVUQCOM262PIUYJQ`

These links give context for the **existing Zig `tofu` message API**, the **polynode model**, and how they map into the Odin service‑runtime architecture.

---

## 3. Prompt for Another AI

You can copy‑paste this block into another AI:

> You are a software architect, expert in Odin, backend systems, and message‑oriented systems.  
>  
> I have a **Zig‑based project `tofu`** (async messaging for modular monoliths).  
> The main item is `Message` defined here:  
> `https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/src/message.zig`  
> Docs:  
> - `https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/message.md`  
> - `https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/patterns.md`  
>  
> I am now porting the `message` and related “objects” into a **separate `frame` package** based on **Odin**, which will be part of a **template repository** for Odin services.  
>  
> The overall architecture is **polynode‑centric** and described here:  
>  
> - Central Axiom: `matryoshka` operates **only** on polynode‑based items, **never** on raw frame structures.  
> - Layers: `frame` → `polynode` → `matryoshka` → `runtime` + `plugins`.  
> - `frame` is **only** transport ABI; it must **not** depend on `matryoshka` or `runtime`.  
> - All behavior and decisions are in `plugins`.  
> - `polynode` is an independent structural primitive (see: `https://raw.githubusercontent.com/g41797/matryoshka/refs/heads/main/polynode.odin`).  
> - Use this mapping document for context:  
>   - `https://github.com/g41797/tofusite/blob/main/root/otofu/analysis/S6_tofu_mapping_fixed.md`  
>   - `https://raw.githubusercontent.com/g41797/tofusite/refs/heads/main/root/otofu/analysis/S1_modules_fixed.md?token=GHSAT0AAAAAADUVB5DSWH7CC23NVUQCOM262PIUYJQ`  
>  
> Your task:  
>  
> Design a **high‑level** Odin `frame` package that provides **all functionality** of the Zig `tofu` `Message` and related structures, but **adapted to the Odin model and polynode‑centric rules**.  
>  
> Requirements:
> - Do **not** force a copy of the Zig `tofu` API; rethink if it is wrong (for example, do not necessarily export `Appendable`‑style body; consider using Odin dynamic arrays; text headers may be a “view” on a second dynamic array, not part of the core message API).  
> - The `frame` package must be **based on** the `polynode` struct from `matryoshka`.  
> - Design should list:
>   - Odin source files (names, one per file).
>   - Top‑level structs and enums.
>   - Public API signatures (no implementation details).
>  
> Use **simple English** and **idiomatic Odin** style. Avoid “AI‑speak”; this is for non‑native English developers.

---

## 4. High‑Level Odin `frame` Package Design

We design the `frame` package as a **pure transport layer** that carries `polynode`‑based items and optional payloads (headers, body). The internal body representation is **not exposed**; it may be implemented via Odin dynamic arrays or views, but that is an internal detail.

The API surface is **lean** and focused on:

- creating and owning frames
- reading and writing frames
- extracting the embedded `polynode`
- attaching binary and text headers and a body

### 4‑1. Package Shape and Files

A minimal, single‑purpose `frame` package:

- `frame.odin` – main module; public API
- `frame_types.odin` – internal types, layout details (optional, if you split types)
- `frame_io.odin` – encoding/decoding helpers (optional, if you expose them)

For now, assume **one file**:

- `frame.odin` – single module with all public API.

You can later split into `types`, `io`, `view`, etc., if you want.

---

### 4‑2. Core Framed Message Type

Follow the matryoshka rule: **`polynode` must be the first field** of any framed item.

```odin
// IMPORTS you can assume:
import "core:mem"
import "core:bytes"
import "core:slice"
import "core:string"

import "g41797:matryoshka" // assuming polynode is here
```


#### 4‑2‑1. `FrameMessage` (polynode‑based)

```odin
// FrameMessage is the transport ABI container.
// It embeds a polynode as its first field.
FrameMessage :: struct {
    // Polynode is the structural core (mandatory, first field).
    // Based on:
    // https://raw.githubusercontent.com/g41797/matryoshka/refs/heads/main/polynode.odin
    polynode: matryoshka.Polynode,

    // Binary header (optional transport metadata).
    header_bin: []u8,

    // Text header (optional, human‑readable or structured text).
    // Currently modeled as a string view, not owned by the frame.
    header_text: string,

    // Body payload (opaque bytes).
    // May be implemented as a dynamic array internally;
    // public API does not expose "Appendable"‑style.
    body: []u8,
}
```

This struct:

- matches the architecture rule: `polynode` as first field.
- is **pure data**, no methods, no logic.
- expects that **encoding/decoding** and **view / append helpers** are in procedures, not in the struct.

---

### 4‑3. Enums and Flags

Optional enums to tag frame semantics (without semantics, just tags):

```odin
// FrameKind distinguishes logical frame types
// at the transport layer (request, reply, event, etc.).
// No semantics: only runtime and plugins decide meaning.
FrameKind :: enum u32 {
    request = 1,
    reply   = 2,
    event   = 3,
    stream  = 4,
    custom0 = 1000, // allow user‑defined extensions
}

// FrameFlags (optional) for transport‑level properties.
// Again, no semantics; only runtime/plugins decide meaning.
FrameFlags :: enum u32 {
    is_stream = 1 << 0,
    is_final  = 1 << 1,
    has_reply = 1 << 2,
}
```

These types are optional exports; they can be used in sidecar helpers or helpers in `runtime`, but are **not** part of the `FrameMessage` itself.

---

### 4‑4. Public API Signatures

The `frame` package exposes **value‑oriented** and **borrow‑oriented** APIs:

- `create` / `from` constructors
- accessor helpers (no mutation of internals)
- encode/decode helpers (if you want to test wire‑format)

We use **simple idiomatic Odin** (no `#` prefixes, no export by default; assume `using` or explicit `frame.`).

---

#### 4‑4‑1. Construction and Zero‑Initialization

```odin
// Frame_zero initializes a zero FrameMessage.
// Polynode is zeroed; headers and body are nil.
Frame_zero :: proc() -> FrameMessage;

// Frame_from_polynode creates a frame that carries the given polynode.
// Headers and body are initially empty.
Frame_from_polynode :: proc(polynode: matryoshka.Polynode) -> FrameMessage;
```

These are **factory functions**, not methods.

---

#### 4‑4‑2. Header and Body Accessors

We **do not** expose internal `Appendable`‑style APIs; instead, expose slices and views.

```odin
// Frame_set_body replaces the frame’s body with a new slice.
// Does not copy; the caller owns the slice.
Frame_set_body :: proc(frame: ^FrameMessage, body: []u8);

// Frame_view_body returns a view of the body (read‑only).
Frame_view_body :: proc(frame: ^const FrameMessage) -> []u8;

// Frame_set_header_bin replaces the binary header.
Frame_set_header_bin :: proc(frame: ^FrameMessage, header: []u8);

// Frame_view_header_bin returns a view of the binary header.
Frame_view_header_bin :: proc(frame: ^const FrameMessage) -> []u8;

// Frame_set_header_text replaces the text header.
// Frame does not own the string; it’s a view.
Frame_set_header_text :: proc(frame: ^FrameMessage, text: string);

// Frame_view_header_text returns the text header.
Frame_view_header_text :: proc(frame: ^const FrameMessage) -> string;
```

These allow:

- consuming bodies and headers as slices or strings.
- runtime/plugins to append or replace as needed.

---

#### 4‑4‑3. Polynode Access and Extraction

Per the axiom, `matryoshka` only works on `polynode` items, not on raw frames.

```odin
// Frame_polynode returns a pointer to the embedded polynode.
// This is how matryoshka can operate on it.
Frame_polynode :: proc(frame: ^FrameMessage) -> ^matryoshka.Polynode;

// Frame_polynode_view returns a const view of the polynode.
Frame_polynode_view :: proc(frame: ^const FrameMessage) -> ^const matryoshka.Polynode;
```

This is the **only** way `matryoshka` should obtain `polynode` items from a frame.

---

### 4‑5. Encoding / Decoding API (Optional)

If you want `frame` to also expose wire‑format helpers (e.g., for tests or simple transports):

```odin
// Frame_encode writes the frame to a byte buffer.
// Uses the provided allocator and returns the encoded bytes and an error flag.
Frame_encode :: proc(frame: ^const FrameMessage, out: mem.Allocator) -> (bytes: []u8, ok: bool);

// Frame_decode parses a frame from a byte buffer.
// Uses the provided allocator and returns a FrameMessage and an error flag.
Frame_decode :: proc(data: []u8, out: mem.Allocator) -> (frame: FrameMessage, ok: bool);
```

These procedures:

- do **not** contain runtime logic.
- only encode/decode the ABI (polynode, headers, body) in a canonical format.

You can later move wire‑format details into a separate `frame_wire` or `frame_bin` package if you want to swap formats (e.g., for JSON‑like vs compact‑binary).

---

### 4‑6. Optional “View” Types for Zero‑Copy Usage

If you want to avoid copying for HTTP‑style or streaming usage:

```odin
// FrameView is a zero‑copy view of a FrameMessage.
// It does not own the underlying data.
FrameView :: struct {
    polynode:     ^const matryoshka.Polynode,
    header_bin:   []u8,
    header_text:  string,
    body:         []u8,
}

// Frame_as_view converts a FrameMessage into a FrameView.
Frame_as_view :: proc(frame: ^const FrameMessage) -> FrameView;

// Frame_from_view creates a FrameMessage from a FrameView,
// copying the data into a new frame.
// Uses the provided allocator.
Frame_from_view :: proc(view: FrameView, alloc: mem.Allocator) -> FrameMessage;
```

This is **optional**; you can drop it if you do not need zero‑copy views.

---

### 4‑7. Optional Kind / Flag Helpers (Sidecar)

You can expose:

```odin
// FrameKindOf extracts the kind from the polynode or a sidecar field.
// This is optional; it can live in runtime or plugins.
FrameKindOf :: proc(polynode: ^const matryoshka.Polynode) -> FrameKind;

// FrameFlagsOf extracts the transport flags.
FrameFlagsOf :: proc(polynode: ^const matryoshka.Polynode) -> FrameFlags;
```

These helpers are **not part of the core `frame`**; they show how you can extend the model without bloating `frame`.

---

## 5. Summary of Public API Surface

Below is a compact list of what the `frame` package exposes at the signature level only.

```odin
//
// --- Types ---
//

FrameMessage :: struct {
    polynode:      matryoshka.Polynode,
    header_bin:    []u8,
    header_text:   string,
    body:          []u8,
}

FrameView :: struct {
    polynode:      ^const matryoshka.Polynode,
    header_bin:    []u8,
    header_text:   string,
    body:          []u8,
}

FrameKind :: enum u32 {
    request = 1,
    reply   = 2,
    event   = 3,
    stream  = 4,
    custom0 = 1000,
}

FrameFlags :: enum u32 {
    is_stream = 1 << 0,
    is_final  = 1 << 1,
    has_reply = 1 << 2,
}

//
// --- Construction ---
//

Frame_zero ::

```

