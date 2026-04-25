# `frame` Package (Odin) — High-Level Design

**Version:** 3 (Full Specification / Embedded Allocator)  
**Status:** Ready for implementation  
**Date:** 2026-04-22

This document is the authoritative high-level design for the `frame` package. It replaces Version 2 to reflect the shift toward self-managed message lifecycles for cross-thread safety.

---

## Part 1 — Prompt (reusable)

### Role
You are a software architect. You are an expert in Odin, systems programming, and high-performance messaging. You design clean, minimal layers. You do **not** write implementation bodies during the design phase.

### Background
The project ports the Zig `tofu` messaging layer to Odin as `frame`. It integrates with the `matryoshka` structural primitive `PolyNode`. The system follows a strict layering where the transport container (`frame`) is distinct from the runtime substrate (`matryoshka`).

### Polynode-centric architecture
The system has five layers:
- `frame`: Transport ABI container.
- `polynode`: Structural primitive embedded in every item.
- `matryoshka`: Runtime substrate operating on `poly` items.
- `runtime`: Orchestration and transport adapters (TCP, UDS).
- `plugins`: User-defined logic.

**Central Axiom:**
`matryoshka` operates **only** on `PolyNode` items extracted from frames. It never sees or processes the raw `FrameMessage` structure.

**Frame container shape:**
```odin
FrameMessage {
    PolyNode        (Offset 0)
    BinaryHeader    (16-byte packed)
    TextHeaders     (Key-Value pairs)
    Body            ([dynamic]byte)
    allocator       (embedded mem.Allocator)
    app_ctx         (opaque rawptr)
    eng_ctx         (opaque rawptr)
}
```

---

## Part 2 — Resulting High-Level Design

### 0. Definition
`frame` is a passive, polynode-first transport container carrying the `tofu` wire-format message. It embeds its own allocator to ensure messages can safely traverse thread boundaries without "arena rug-pulls."

### 1. Purpose
- Define the in-memory shape of a frame message with `PolyNode` at offset 0.
- Provide 1:1 wire compatibility with Zig `tofu`.
- Provide an embedded allocator model where the message "carries its own cleanup kit."
- Separate owned decoding from zero-copy views.

### 2. Package Structure (File Granularity)
- `frame.odin`: Package entry point and re-exports.
- `frame_message.odin`: The `FrameMessage` struct definition and lifecycle procs.
- `frame_opcodes.odin`: `OpCode` (0–9), `MessageType`, and `MessageRole` enums.
- `frame_proto.odin`: `ProtoFields` bit-field (8-bit) and flag enums.
- `frame_binary_header.odin`: The 16-byte `BinaryHeader` struct and endian helpers.
- `frame_text_headers.odin`: `TextHeaders` storage and the `TextHeader` slice-view.
- `frame_body.odin`: Helpers for managing the `[dynamic]byte` payload.
- `frame_status.odin`: `FrameStatus` (u8) and `FrameError` (enum).
- `frame_validate.odin`: Logic to verify wire-data integrity before full decode.
- `frame_views.odin`: `FrameMessageView` for zero-copy operations.
- `frame_codec.odin`: The core `encode` and `decode` procedures.

### 3. Internal Dependency Layering
```text
[frame_opcodes] ← [frame_proto]
      ↑                ↑
[frame_status] ← [frame_binary_header] ← [frame_validate]
      ↑                ↑                      ↑
[frame_text_headers] ← [frame_message] ← [frame_codec]
      ↑                ↑                      ↑
[frame_body]         [matryoshka]        [frame_views]
```

### 4. Core Types

#### 4.1 FrameMessage
```odin
FrameMessage :: struct {
    using poly : matryoshka.PolyNode, // Offset 0

    bhdr       : BinaryHeader,
    text_hdrs  : TextHeaders,
    body       : [dynamic]byte,

    // The stable allocator (usually a malloc-style heap) used for 
    // text_hdrs and body. Essential for cross-thread handoffs.
    allocator  : mem.Allocator, 

    app_ctx    : rawptr, 
    eng_ctx    : rawptr, 
}
```

#### 4.2 BinaryHeader
```odin
BinaryHeader :: struct #packed {
    channel_number : u16,    // Big-Endian
    proto          : u8,     // Bit-field: OpCode(4), Origin(1), More(1), Reserved(2)
    status         : u8,     // FrameStatus
    message_id     : u64,    // Big-Endian
    text_hdr_len   : u16,    // Big-Endian
    body_len       : u16,    // Big-Endian
}
#assert(size_of(BinaryHeader) == 16)
```

### 5. Memory & Ownership Strategy
1. **Creation:** `frame_create` requires an explicit allocator. This allocator is stored in the struct.
2. **Growth:** If `text_hdrs` or `body` need to reallocate, they **must** use `msg.allocator`.
3. **Cloning:** `frame_clone` uses the source message's allocator. It does not take an allocator argument.
4. **Handoff:** When a thread receives a `FrameMessage` via a mailbox, it can safely modify or destroy it because the memory is backed by the embedded (stable) allocator, not a thread-local arena.

### 6. API Signatures

#### 6.1 Lifecycle (`frame_message.odin`)
```odin
frame_create  :: proc(allocator: mem.Allocator) -> (^FrameMessage, FrameError)
frame_destroy :: proc(msg: ^FrameMessage)
frame_clone   :: proc(msg: ^FrameMessage) -> (^FrameMessage, FrameError)
frame_reset   :: proc(msg: ^FrameMessage)
```

#### 6.2 Codec (`frame_codec.odin`)
```odin
// Returns bytes written/read
frame_encode      :: proc(msg: ^FrameMessage, out: []byte) -> (int, FrameError)
frame_decode      :: proc(msg: ^FrameMessage, in_data: []byte) -> (int, FrameError)
frame_decode_view :: proc(in_data: []byte) -> (FrameMessageView, int, FrameError)
```

#### 6.3 Headers & Body
```odin
text_header_set :: proc(msg: ^FrameMessage, key, value: string) -> FrameError
body_set        :: proc(msg: ^FrameMessage, data: []byte) -> FrameError
```

### 7. Wire Format Table
| Offset | Size | Field | Description |
| :--- | :--- | :--- | :--- |
| 0 | 2 | Channel | u16 (BE) |
| 2 | 1 | Proto | OpCode/Flags |
| 3 | 1 | Status | FrameStatus |
| 4 | 8 | MsgID | u64 (BE) |
| 12 | 2 | T-Len | Text Header Length |
| 14 | 2 | B-Len | Body Length |
| 16 | Var | T-Data | Key: Value\r\n pairs |
| Var | Var | B-Data | Raw body bytes |

### 8. Allocation & Concurrency Rules
- **Cross-Thread Safety:** `FrameMessage` instances are intended to be moved between threads. 
- **Arena Ban:** Never use a `temp_allocator` or a short-lived `Arena` for `frame_create` if the message is destined for a mailbox.
- **Context Ownership:** `app_ctx` is copied during `frame_clone`. `eng_ctx` is cleared (set to nil) during `frame_clone` as it is specific to the active runtime instance.

### 9. Protocol Evolution
The `ProtoFields` bit-field reserves 2 bits for future engine flags. The `OpCode` space (4 bits) is locked to the 10 values defined in the Zig `tofu` spec to maintain ABI compatibility.

### 10. What `frame` deliberately does NOT provide
| Concern | Responsibility |
| :--- | :--- |
| Thread-safe Mailboxes | `matryoshka` / `odin-mbox` |
| List Linking | `matryoshka.PolyNode` |
| TCP/UDP IO | `runtime` |
| Message Routing | `runtime` |
| Frame Pooling | `matryoshka` |

### 11. Constraints Checklist
- [x] `PolyNode` at offset 0 for direct casting.
- [x] Embedded `allocator` for self-contained cleanup.
- [x] Wire-compatible 16-byte header.
- [x] Big-endian enforcement in codec.
- [x] Separation of `FrameMessage` (owned) and `FrameMessageView` (borrowed).

### 12. Recap
`frame` provides the stable, self-managed transport container required for the `otofu` messaging engine. By embedding the allocator, it eliminates the risk of memory corruption during cross-thread handoffs while maintaining a minimalist "dev-style" API.
