# 📦 Frame Package (Odin) — High-Level Design
Version: 1

---

# PART 1 — PROMPT (Reusable)

You are a software architect specializing in Odin, backend systems, and message-oriented architectures.

Design a **frame package in Odin** for a system with the following architecture:

## Core Rule

matryoshka operates ONLY on polynode items extracted from frame  
matryoshka NEVER sees or processes frame structures

---

## System Layers

- frame → transport ABI container
- polynode → structural primitive (from matryoshka)
- matryoshka → runtime substrate (structure only)
- runtime → orchestration layer
- plugins → decision logic

---

## Frame Definition

Frame is a transport container:

FrameMessage {
    Polynode (FIRST FIELD)
    HeaderBinary
    HeaderText
    Body
}

---

## Requirements

Produce a **high-level design only** (no implementation):

Include:

- file structure (Odin files)
- structs
- enums
- API signatures
- design decisions

Constraints:

- frame must NOT contain logic
- frame must NOT depend on matryoshka runtime
- frame must NOT interpret data
- frame must be ABI-oriented
- polynode comes from external module

---

## Important Design Freedom

You are NOT required to copy Zig message design.

You must decide:

- body representation
- header representation
- ownership model
- API minimalism

---

## Output

Return full Markdown document with:

- clean structure
- simple English
- no marketing language
- no implementation details

---

# PART 2 — RESULTING HIGH LEVEL DESIGN

---

# 1. Purpose

The **frame package** defines a minimal transport container.

It exists only to:

- carry polynode
- carry payload
- provide encode/decode boundary

It does NOT:

- execute logic
- interpret headers
- manage lifecycle
- know anything about runtime or plugins

---

# 2. Core Design Direction

This design intentionally avoids:

- builder APIs
- appendable abstractions
- hidden allocations
- semantic helpers

Reason:

> frame must stay a passive data structure

---

# 3. Package Structure

```

frame/
├── frame_message.odin
├── frame_types.odin
├── frame_headers.odin
├── frame_body.odin
├── frame_codec.odin
├── frame_errors.odin
├── frame_api.odin
└── frame_views.odin

````

---

# 4. Core Data Structures

---

## 4.1 FrameMessage

```odin
FrameMessage :: struct {
    node: polynode.Polynode, // MUST be first

    header_bin: []u8,
    header_txt: HeaderText,

    body: []u8,
}
````

Key decisions:

* no nested structs unless necessary
* binary header is raw bytes
* body is raw bytes
* everything is transport-level

---

## 4.2 HeaderText

Text headers are NOT maps.

They are ordered and allocation-free friendly.

```odin
HeaderText :: struct {
    entries: []HeaderEntry,
}
```

```odin
HeaderEntry :: struct {
    key:   string,
    value: string,
}
```

Reason:

* preserves order (important for protocols)
* avoids hashmap dependency
* simpler encoding

---

## 4.3 Body

```odin
Body :: distinct []u8
```

But inside FrameMessage we store:

```odin
body: []u8
```

Reason:

* no abstraction needed
* runtime decides meaning
* frame does not expose manipulation API

---

# 5. Enums

---

## 5.1 FrameError

```odin
FrameError :: enum {
    None,

    InvalidFormat,
    InvalidPolynode,
    InvalidHeaders,
    InvalidBody,

    DecodeFailed,
    EncodeFailed,
}
```

---

## 5.2 DecodeMode (important for runtime)

```odin
DecodeMode :: enum {
    Full,        // full materialization
    HeaderOnly,  // partial decode
}
```

---

# 6. API Design (Minimal)

Frame exposes only transport operations.

---

## 6.1 Decode

```odin
frame_decode :: proc(
    data: []u8,
    mode: DecodeMode,
    allocator: mem.Allocator,
) -> (FrameMessage, FrameError)
```

---

## 6.2 Encode

```odin
frame_encode :: proc(
    msg: FrameMessage,
    allocator: mem.Allocator,
) -> ([]u8, FrameError)
```

---

## 6.3 Zero-Copy Decode (critical for performance)

```odin
frame_decode_view :: proc(
    data: []u8,
) -> (FrameMessageView, FrameError)
```

---

# 7. View Model (Important Design Choice)

To avoid unnecessary allocations:

```odin
FrameMessageView :: struct {
    node: ^polynode.Polynode,

    header_bin: []u8,
    header_txt: HeaderTextView,

    body: []u8,
}
```

```odin
HeaderTextView :: struct {
    entries: []HeaderEntryView,
}
```

```odin
HeaderEntryView :: struct {
    key:   []u8,
    value: []u8,
}
```

Key idea:

> decoding does not require copying

Runtime may convert view → owned if needed.

---

# 8. What Frame DOES NOT Provide

This is critical.

Frame does NOT provide:

* header lookup helpers
* body append
* message builders
* routing logic
* validation beyond format

If you add these later, you are moving logic into the wrong layer.

---

# 9. Interaction with Runtime

Runtime is responsible for:

1. calling decode
2. extracting polynode
3. deciding what to do

Example flow:

```
bytes → frame_decode_view
      → extract node
      → pass to matryoshka
```

Frame is not involved further.

---

# 10. Interaction with Polynode

Frame only embeds:

```odin
node: polynode.Polynode
```

Rules:

* frame does not modify it
* frame does not inspect it
* frame does not validate semantics

---

# 11. Encoding Strategy (High Level)

Frame encoding is:

```
[polynode binary]
[header_bin length + data]
[header_txt encoded]
[body length + data]
```

Exact format is implementation detail.

---

# 12. Key Architectural Decisions

---

## 12.1 No Appendable Body

Zig design used Appendable.

Rejected because:

* leaks behavior into transport
* complicates ownership
* not needed in Odin

---

## 12.2 No Map Headers

Maps introduce:

* hashing
* allocator coupling
* non-deterministic order

We keep linear list.

---

## 12.3 View-Based Decode

This is important improvement over typical designs.

Benefits:

* zero-copy
* high throughput
* better for network servers

---

## 12.4 No Builder API

Builders are logic.

Frame must remain:

> passive container

---

# 13. Extension Points

Future additions (outside core):

* streaming decoder (FrameStream)
* compression wrapper
* encryption wrapper
* HTTP adapter (runtime layer, not here)

---

# 14. Constraints Checklist

✔ polynode is first field
✔ frame is transport only
✔ no runtime dependency
✔ no matryoshka dependency
✔ no business logic
✔ minimal API
✔ zero-copy capable

---

# 15. One-Line Definition

frame is a minimal binary container that transports polynode and raw payload without interpreting them

---

# 16. Links (Context)

* message (Zig):
  [https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/src/message.zig](https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/src/message.zig)

* message docs:
  [https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/message.md](https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/message.md)

* patterns:
  [https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/patterns.md](https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/patterns.md)

* polynode (base):
  [https://raw.githubusercontent.com/g41797/matryoshka/refs/heads/main/polynode.odin](https://raw.githubusercontent.com/g41797/matryoshka/refs/heads/main/polynode.odin)

* analysis:
  [https://github.com/g41797/tofusite/blob/main/root/otofu/analysis/S6_tofu_mapping_fixed.md](https://github.com/g41797/tofusite/blob/main/root/otofu/analysis/S6_tofu_mapping_fixed.md)
  [https://raw.githubusercontent.com/g41797/tofusite/refs/heads/main/root/otofu/analysis/S1_modules_fixed.md](https://raw.githubusercontent.com/g41797/tofusite/refs/heads/main/root/otofu/analysis/S1_modules_fixed.md)

---

# 17. Open Questions (Important)

Answer before implementation:

1. Should decode default to:

   * view (zero-copy) or
   * owned (safe but slower)?

2. Will polynode always be fixed-size?
   (affects binary layout strongly)

3. Do you need frame versioning?

4. Is header_txt required or optional?

---
