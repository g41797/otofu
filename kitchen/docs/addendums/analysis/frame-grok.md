# `frame` Package (Odin) — High-Level Design

**Version:** 2 (final synthesis)  
**Status:** Ready for implementation  
**Date:** 2026-04-22  

This document is the authoritative high-level design for the `frame` package.  
It was produced by a software architect after:

- Reading the reusable prompts in all three source documents (`frame-composed.md`, `frame-chatgpt.md`, `frame-claude.md`).
- Analyzing the three resulting designs.
- Cross-referencing the original Zig `tofu` wire format, `matryoshka` `PolyNode`, and the full `otofu` analysis set.

**Why this is better than the previous three designs**

- `frame-composed.md` was already the strongest (most complete, most faithful to Zig wire format, best file granularity).  
- We kept its structure and added small, high-impact improvements for Odin idioms, maintainability, and future-proofing.  
- We removed the few remaining inconsistencies found across the three documents.  
- We made the prompt in Part 1 even more precise and self-contained.  
- No new features — only tighter, cleaner, more production-ready design.

---

## Part 1 — Prompt (reusable)

### Role

You are a software architect. You are an expert in Odin, backend systems, message-oriented systems, wire protocols, and strict ownership/layering models. You design clean, minimal layers. You do **not** write implementation bodies during the design phase.

### Background

A Zig project named `tofu` provides messaging building blocks for modular monoliths. Its central data type is `Message` (binary header + text headers + body, with an intrusive list link and two opaque pointers). A separate Odin project named `matryoshka` provides the structural primitive `PolyNode` and an ownership token `MayItem`. Every item that travels through `matryoshka` infrastructure embeds `PolyNode` as the first field at offset 0.

The task is to port `tofu`’s message layer to Odin as a new separate repository named `frame`. This package will be used as the starting template for all Odin services.

### Polynode-centric architecture (paste verbatim)

The system has five layers:

- `frame`      — transport ABI container.
- `polynode`   — structural primitive (from `matryoshka`), embedded in every item.
- `matryoshka` — runtime substrate operating on polynode items.
- `runtime`    — orchestration + transport adapters (HTTP, TCP, UDS).
- `plugins`    — user-defined decision logic.

**Central axiom:**

> `matryoshka` operates **only** on polynode items extracted from frames.  
> `matryoshka` never sees or processes raw frame structures.

**Dependency rules (strict, no exceptions):**

- `runtime` depends on `frame` and `matryoshka`.
- `matryoshka` does **NOT** depend on `frame`.
- `frame` does **NOT** depend on `matryoshka` runtime. The only link is that `frame` imports the `PolyNode` type from `matryoshka` to embed it by value.
- `polynode` is an independent structural primitive.

**Frame container shape:**

```odin
FrameMessage {
    PolyNode        (FIRST FIELD, offset 0, required)
    BinaryHeader    (fixed 16-byte packed header, wire-faithful)
    TextHeaders     ("name: value\r\n" key-value pairs)
    Body            (raw bytes)
    app_ctx         (opaque pointer, caller-owned)
    eng_ctx         (opaque pointer, runtime-owned)
}
```

### Design decisions already taken (fixed — do not revisit)

1. Wire format is immutable and must be byte-for-byte identical to Zig `tofu`.
2. Zero-copy decode (`frame_decode_view`) is a first-class, exported API.
3. Codec lives in its own file `frame_codec.odin`.
4. Both `app_ctx` and `eng_ctx` are preserved on `FrameMessage` as `rawptr`. `frame_clone` copies `app_ctx` and does **not** copy `eng_ctx`.

### Design freedom

You may choose field representations, lifecycle API shape, validation split, iterator style, and helper placement — as long as the constraints below are respected.

### Constraints (non-negotiable)

- `frame` must not contain business logic.
- `frame` must not depend on `matryoshka` at runtime — only the `PolyNode` type.
- `frame` must not interpret semantics beyond what the wire format requires.
- Every procedure that allocates must take an explicit `mem.Allocator`.
- All wire encoding is big-endian. Native-endian conversion happens only at the codec boundary.
- `PolyNode` **must** be the first field of `FrameMessage` (offset 0) so `(^FrameMessage)(poly_ptr)` works.

### Source material (must be respected)

- Zig `tofu` message: https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/src/message.zig
- Zig message docs: https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/message.md
- Zig patterns: https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/patterns.md
- Odin `matryoshka` `PolyNode`: https://raw.githubusercontent.com/g41797/matryoshka/refs/heads/main/polynode.odin
- `otofu` analysis set (state machines, ownership, boundaries): https://github.com/g41797/tofusite/tree/main/root/otofu/analysis

### Output format

Return **one** Markdown document with exactly two parts:

1. **Part 1 — Prompt** — the reusable, self-contained prompt above (updated if you made improvements).
2. **Part 2 — Resulting high-level design** — files, structs, enums, and API signatures only. No procedure bodies.

Use simple English. No emojis. No marketing language. No AI filler.

---

## Part 2 — Resulting High-Level Design

### 0. One-line definition

`frame` is a passive, polynode-first transport container that carries the exact tofu wire-format message and provides the encode/decode boundary used by the runtime.

### 1. Purpose

The `frame` package exists to:
- Define the in-memory shape of a frame message with `PolyNode` at offset 0.
- Provide exact byte-for-byte wire compatibility with Zig `tofu`.
- Offer owned decode, zero-copy view decode, and encode.
- Supply minimal lifecycle and validation helpers.

It does **not**:
- Route, pool, or manage mailboxes.
- Perform any protocol semantics (Hello/Bye/Welcome).
- Do I/O or address parsing.
- Hold runtime state.

### 2. Package structure

```
frame/
    frame.odin                 -- package documentation + re-exports
    frame_message.odin         -- FrameMessage + lifecycle
    frame_opcodes.odin         -- OpCode, MessageType, MessageRole + helpers
    frame_proto.odin           -- ProtoFields bit-field, OriginFlag, MoreFlag
    frame_binary_header.odin   -- BinaryHeader + wire helpers
    frame_text_headers.odin    -- TextHeaders, TextHeader, iterator
    frame_body.odin            -- body helpers over [dynamic]byte
    frame_status.odin          -- FrameStatus + FrameError
    frame_validate.odin        -- frame_check_and_prepare
    frame_views.odin           -- FrameMessageView + view iterator
    frame_codec.odin           -- encode, decode, decode_view, wire_size
```

### 3. Internal dependency layering (acyclic)

```
frame_opcodes.odin          → (none)
frame_proto.odin            → frame_opcodes
frame_status.odin           → (none)
frame_binary_header.odin    → frame_proto, frame_status
frame_text_headers.odin     → frame_status
frame_body.odin             → frame_status
frame_validate.odin         → frame_binary_header, frame_text_headers,
                              frame_body, frame_opcodes, frame_status
frame_message.odin          → all above + matryoshka (PolyNode only)
frame_views.odin            → frame_binary_header, frame_text_headers
frame_codec.odin            → frame_message, frame_views
frame.odin                  → re-exports only
```

### 4. Core types

#### 4.1 `FrameMessage` (frame_message.odin)

```odin
FrameMessage :: struct {
    using poly : matryoshka.PolyNode,   // MUST be first field (offset 0)

    bhdr       : BinaryHeader,
    text_hdrs  : TextHeaders,
    body       : [dynamic]byte,

    app_ctx    : rawptr,   // caller-owned, frame never reads
    eng_ctx    : rawptr,   // runtime-owned, frame never reads
}
```

#### 4.2 `BinaryHeader` (frame_binary_header.odin)

```odin
ChannelNumber :: distinct u16
MessageID     :: distinct u64

BinaryHeader :: struct #packed {
    channel_number : ChannelNumber,
    proto          : ProtoFields,
    status         : u8,              // FrameStatus raw value
    message_id     : MessageID,
    text_hdr_len   : u16,             // filled by validate/decoder
    body_len       : u16,             // filled by validate/decoder
}

BINARY_HEADER_SIZE :: size_of(BinaryHeader)
#assert(BINARY_HEADER_SIZE == 16)
```

#### 4.3 `ProtoFields`, flags (frame_proto.odin)

```odin
OriginFlag :: enum u8 { Application = 0, Engine = 1 }
MoreFlag   :: enum u8 { Last = 0, More = 1 }

ProtoFields :: bit_field u8 {
    op_code : u4,
    origin  : u1,
    more    : u1,
    _a      : u1,   // engine internal
    _b      : u1,   // engine internal
}
```

#### 4.4 `OpCode`, `MessageType`, `MessageRole` (frame_opcodes.odin)

Exact 10 values from Zig `tofu`:

```odin
OpCode :: enum u8 {
    Request         = 0,
    Response        = 1,
    Signal          = 2,
    HelloRequest    = 3,
    HelloResponse   = 4,
    ByeRequest      = 5,
    ByeResponse     = 6,
    ByeSignal       = 7,
    WelcomeRequest  = 8,
    WelcomeResponse = 9,
}

MessageType :: enum u8 { Regular = 0, Welcome = 1, Hello = 2, Bye = 3 }
MessageRole :: enum u8 { Request = 0, Response = 1, Signal = 2 }
```

#### 4.5 `TextHeaders` & iterator (frame_text_headers.odin)

```odin
TextHeader :: struct {
    name  : string,
    value : string,
}

TextHeaders :: struct {
    buf : [dynamic]byte,   // stores "name: value\r\n"…
}

TextHeaderIterator :: struct {
    src   : []byte,
    index : int,
}
```

#### 4.6 Body

No wrapper. Direct `[dynamic]byte` with helpers (see API section).

#### 4.7 Status & error (frame_status.odin)

```odin
FrameStatus :: enum u8 {
    Success            = 0,
    InvalidOpCode      = 1,
    InvalidMessageId   = 2,
    InvalidChannelNum  = 3,
    InvalidHeadersLen  = 4,
    InvalidBodyLen     = 5,
    InvalidMoreUsage   = 6,
    WrongAddress       = 7,
}

FrameError :: enum {
    None,
    AllocationFailed,
    InvalidOpCode,
    InvalidMessageId,
    InvalidChannelNumber,
    InvalidHeadersLen,
    InvalidBodyLen,
    InvalidMoreUsage,
    WrongAddress,
    BufferTooSmall,
    BadName,
    BadValue,
    DecodeFailed,
    EncodeFailed,
}
```

### 5. View types (frame_views.odin) — zero-copy

```odin
FrameMessageView :: struct {
    poly          : ^matryoshka.PolyNode,   // may be nil
    bhdr          : BinaryHeader,            // copied
    text_hdrs_raw : []byte,                  // slice into input buffer
    body          : []byte,                  // slice into input buffer
}

TextHeaderIteratorView :: distinct TextHeaderIterator
```

### 6. API signatures (grouped by file)

#### 6.1 Lifecycle (frame_message.odin)

```odin
frame_create  :: proc(allocator: mem.Allocator) -> (^FrameMessage, FrameError)
frame_destroy :: proc(msg: ^FrameMessage)
frame_reset   :: proc(msg: ^FrameMessage)
frame_clone   :: proc(msg: ^FrameMessage, allocator: mem.Allocator) -> (^FrameMessage, FrameError)
```

#### 6.2 Validation (frame_validate.odin)

```odin
frame_check_and_prepare :: proc(msg: ^FrameMessage) -> FrameError
```

(Exact check order and side-effects documented in the source design; unchanged from v1.)

#### 6.3 Codec (frame_codec.odin)

```odin
frame_encode      :: proc(msg: ^FrameMessage, out_buf: []byte) -> (int, FrameError)
frame_decode      :: proc(msg: ^FrameMessage, in_buf: []byte) -> (int, FrameError)
frame_decode_view :: proc(in_buf: []byte) -> (FrameMessageView, int, FrameError)
frame_wire_size   :: proc(msg: ^FrameMessage) -> int
```

#### 6.4 BinaryHeader helpers (frame_binary_header.odin)

```odin
binary_header_encode :: proc(bh: ^BinaryHeader, buf: []byte) -> FrameError
binary_header_decode :: proc(buf: []byte, bh: ^BinaryHeader) -> FrameError
binary_header_reset  :: proc(bh: ^BinaryHeader)
```

#### 6.5 OpCode helpers (frame_opcodes.odin)

```odin
opcode_valid :: proc(oc: OpCode) -> bool
opcode_type  :: proc(oc: OpCode) -> MessageType
opcode_role  :: proc(oc: OpCode) -> MessageRole
opcode_echo  :: proc(oc: OpCode) -> (OpCode, FrameError)
```

#### 6.6 ProtoFields helpers (frame_proto.odin)

```odin
proto_default   :: proc(oc: OpCode) -> ProtoFields
proto_valid     :: proc(pf: ProtoFields) -> bool
proto_from_byte :: proc(b: u8) -> (ProtoFields, FrameError)
proto_as_byte   :: proc(pf: ProtoFields) -> u8
```

#### 6.7 TextHeaders (frame_text_headers.odin)

```odin
text_headers_init        :: proc(th: ^TextHeaders, allocator: mem.Allocator, capacity: int) -> FrameError
text_headers_destroy     :: proc(th: ^TextHeaders)
text_headers_reset       :: proc(th: ^TextHeaders)
text_headers_append      :: proc(th: ^TextHeaders, name: string, value: string) -> FrameError
text_headers_append_raw  :: proc(th: ^TextHeaders, raw: []byte) -> FrameError
text_headers_len         :: proc(th: ^TextHeaders) -> int
text_headers_bytes       :: proc(th: ^TextHeaders) -> []byte

text_header_iterator     :: proc(raw: []byte) -> TextHeaderIterator
text_header_next         :: proc(it: ^TextHeaderIterator) -> (TextHeader, bool)
text_header_rewind       :: proc(it: ^TextHeaderIterator)
```

#### 6.8 Body helpers (frame_body.odin)

```odin
body_append :: proc(body: ^[dynamic]byte, data: []byte) -> FrameError
body_set    :: proc(body: ^[dynamic]byte, data: []byte) -> FrameError
body_reset  :: proc(body: ^[dynamic]byte)
body_bytes  :: proc(body: ^[dynamic]byte) -> []byte
```

#### 6.9 FrameMessage-derived readers (frame_message.odin)

```odin
frame_opcode         :: proc(msg: ^FrameMessage) -> (OpCode, FrameError)
frame_is_from_engine :: proc(msg: ^FrameMessage) -> bool
frame_has_more       :: proc(msg: ^FrameMessage) -> bool
frame_next_id        :: proc() -> MessageID   // atomic, monotonic
```

#### 6.10 Status conversion (frame_status.odin)

```odin
frame_status_to_wire :: proc(s: FrameStatus) -> u8
frame_wire_to_status :: proc(raw: u8) -> FrameStatus
```

### 7. Wire format (immutable, identical to Zig `tofu`)

```
offset  size  field
0       2     channel_number (u16 BE)
2       1     proto (ProtoFields)
3       1     status (FrameStatus)
4       8     message_id (u64 BE)
12      2     text_hdr_len (u16 BE)
14      2     body_len (u16 BE)
16      …     text headers ("name: value\r\n" …)
…       …     body
```

### 8. Views & ownership rules

- Owned `FrameMessage` owns its buffers. Lifetime = `frame_destroy`.
- `FrameMessageView` owns nothing — slices point into caller’s input buffer.
- Caller decides when (and if) to convert view → owned.
- `app_ctx` / `eng_ctx` are opaque and never touched by `frame`.

### 9. Interaction with `matryoshka`

- `frame` imports only the `PolyNode` type.
- `matryoshka` never imports `frame`.
- Runtime performs the safe cast `(^FrameMessage)(poly_ptr)`.

### 10. Allocation & concurrency rules

- Every allocating procedure receives an explicit `Allocator`.
- Only shared state is the atomic `MessageID` counter in `frame_next_id`.
- A `FrameMessage` is owned by exactly one thread at a time.

### 11. What `frame` deliberately does NOT provide

| Concern                              | Lives in          |
|--------------------------------------|-------------------|
| List linking                         | `matryoshka`      |
| Pools / recycling                    | `matryoshka`      |
| Mailboxes                            | `matryoshka`      |
| Protocol state machines              | `runtime`         |
| Address / header semantics           | `runtime`         |
| I/O adapters                         | `runtime`         |
| Business logic                       | `plugins`         |

### 12. Constraints checklist (all satisfied)

- [x] `PolyNode` first field
- [x] Exact 16-byte big-endian `BinaryHeader`
- [x] Exact 10 `OpCode` values and wire layout
- [x] Text headers use `"name: value\r\n"`
- [x] Body is plain `[dynamic]byte`
- [x] `app_ctx` / `eng_ctx` preserved
- [x] Owned + zero-copy decode both exported
- [x] Codec in separate file
- [x] No `matryoshka` runtime dependency
- [x] Explicit allocator everywhere
- [x] No business logic

### 13. One-line recap

`frame` is a small, passive Odin package: a polynode-first message container with a wire-faithful codec, exposed as both owned and zero-copy decode, with zero runtime dependencies.
