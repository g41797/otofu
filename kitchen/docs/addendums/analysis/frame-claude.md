# frame — High-Level Design

---

## Part 1: Prompt (for repeating this question with another AI)

### Context

This document was produced by asking an AI architect to design the `frame` package
for the Matryoshka/oTofu service runtime written in Odin.

To reproduce or extend the design, supply the following to your AI:

**Role:** software architect, expert in Odin, backend systems, message-oriented systems.

**Source files to fetch:**

| What | URL |
|------|-----|
| Zig message implementation | https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/src/message.zig |
| Zig message docs | https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/message.md |
| Zig patterns docs | https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/patterns.md |
| polynode.odin (base struct) | https://raw.githubusercontent.com/g41797/matryoshka/refs/heads/main/polynode.odin |
| System analysis S1 | https://raw.githubusercontent.com/g41797/tofusite/refs/heads/main/root/otofu/analysis/S1_modules_fixed.md |
| System analysis S6 | https://github.com/g41797/tofusite/blob/main/root/otofu/analysis/S6_tofu_mapping_fixed.md |

**Architecture spec (paste verbatim):**

The system has three layers: `frame` (transport ABI container), `polynode` (structural
core embedded in every item), and `matryoshka` (runtime substrate operating on polynode
items). On top: `runtime` (orchestration + adapters) and `plugins` (user decision logic).

Central axiom: **matryoshka operates only on polynode-based items, never on raw frame structures.**

Execution path:
```
HTTP POST → runtime HTTP adapter → frame.decode() → extract polynode item →
runtime dispatch → matryoshka.mailbox → matryoshka.pool → plugins →
matryoshka polynode processing → handler execution → frame.encode() → HTTP response
```

Dependency rules:
- `runtime` depends on `frame` and `matryoshka`
- `matryoshka` does NOT depend on `frame`
- `frame` does NOT depend on `matryoshka`
- `polynode` is an independent structural primitive

**Question to ask:**

> Using all information above, create a high-level design (not implementation) of the
> `frame` package. Include: list of Odin files, structs, enums, and API signatures —
> not internal implementation — sufficient to cover all functionality of message and
> related structs from the Zig source. Do not blindly copy the Zig API; think as an
> architect. frame is "based on" polynode.odin. Use simple English. No AI-ish filler.

---

## Part 2: Resulting High-Level Design

### 0. Design Principles

1. `FrameMessage` is a plain data container. It has no behavior beyond encoding/decoding.
2. `PolyNode` (from `matryoshka`) is the **first field** of `FrameMessage`. This is non-negotiable.
3. Binary header fields are fixed and machine-friendly. Text headers are dynamic key-value pairs.
4. The body is a plain dynamic byte buffer (`[dynamic]byte`). No custom `Appendable` type needed — Odin's built-in dynamic arrays do the job.
5. Text headers are stored as a `[dynamic]byte` buffer in `"name: value\r\n"` format. Parsing is done by an iterator — the buffer itself is not structured.
6. Allocation is explicit. Every procedure that allocates takes an `Allocator`. Nothing allocates silently.
7. All encode/decode is big-endian on the wire (matching the Zig implementation).
8. `frame` has zero dependency on `matryoshka`. It only imports `polynode.odin` because `PolyNode` is embedded by value.

---

### 1. Package Structure

```
frame/
    frame.odin          -- FrameMessage struct + lifecycle (create, destroy, reset, clone)
    opcodes.odin        -- OpCode, MessageType, MessageRole enums + classification procs
    binary_header.odin  -- BinaryHeader struct + encode/decode (big-endian wire format)
    text_headers.odin   -- TextHeaders buffer + TextHeader + iterator
    body.odin           -- body helpers (append, reset, slice view)
    status.odin         -- FrameStatus enum (wire status codes)
    validate.odin       -- check_and_prepare: validates + fills computed fields
```

---

### 2. opcodes.odin

```odin
package frame

// Wire role of a message. Derived from OpCode.
MessageRole :: enum u2 {
    Request  = 0,
    Response = 1,
    Signal   = 2,
}

// Lifecycle type of a message. Derived from OpCode.
MessageType :: enum u2 {
    Regular = 0,
    Welcome = 1,
    Hello   = 2,
    Bye     = 3,
}

// All valid wire operation codes.
OpCode :: enum u4 {
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

// Returns the MessageType for a given OpCode.
opcode_type :: proc(oc: OpCode) -> MessageType

// Returns the MessageRole for a given OpCode.
opcode_role :: proc(oc: OpCode) -> MessageRole

// Returns true if oc is a valid wire value.
opcode_valid :: proc(oc: OpCode) -> bool

// Returns the matching response OpCode for a request.
// Error if oc is not a request, or has no defined echo (e.g. Signal stays Signal).
opcode_echo :: proc(oc: OpCode) -> (OpCode, FrameError)
```

---

### 3. binary_header.odin

```odin
package frame

ChannelNumber :: u16
MessageID     :: u64

// Flags packed into one byte on the wire.
// Layout matches the Zig ProtoFields packed struct (8 bits total).
ProtoFlags :: bit_field u8 {
    op_code : u4,  // OpCode value
    origin  : u1,  // 0 = application, 1 = engine
    more    : u1,  // 0 = last, 1 = more chunks follow
    _a      : u1,  // engine internal
    _b      : u1,  // engine internal
}

// Fixed binary header. Encoded big-endian on the wire.
// Wire layout (16 bytes total):
//   channel_number : u16
//   proto          : u8  (ProtoFlags)
//   status         : u8
//   message_id     : u64
//   text_hdr_len   : u16  (filled by check_and_prepare, not by caller)
//   body_len       : u16  (filled by check_and_prepare, not by caller)
BinaryHeader :: struct {
    channel_number : ChannelNumber,
    proto          : ProtoFlags,
    status         : u8,
    message_id     : MessageID,

    // Filled automatically. Do not set manually.
    text_hdr_len   : u16,
    body_len       : u16,
}

BINARY_HEADER_SIZE :: size_of(BinaryHeader)  // 16 bytes

// Encode bh to buf in big-endian wire format.
// buf must be at least BINARY_HEADER_SIZE bytes.
binary_header_encode :: proc(bh: ^BinaryHeader, buf: []byte) -> FrameError

// Decode buf (big-endian) into bh.
// Returns error if buf is too short or OpCode is invalid.
binary_header_decode :: proc(buf: []byte, bh: ^BinaryHeader) -> FrameError

// Zero all fields.
binary_header_reset :: proc(bh: ^BinaryHeader)
```

---

### 4. text_headers.odin

```odin
package frame

// One parsed name/value pair. Views into the TextHeaders buffer — not copies.
TextHeader :: struct {
    name  : string,
    value : string,
}

// Storage for text headers.
// Raw format in the buffer: "name: value\r\n" repeated.
// The buffer owns the memory. Views (TextHeader) are valid until the buffer changes.
TextHeaders :: struct {
    buf : [dynamic]byte,
}

// Allocate buffer with initial capacity hint.
text_headers_init :: proc(th: ^TextHeaders, allocator: mem.Allocator, capacity: int) -> FrameError

// Free the buffer.
text_headers_destroy :: proc(th: ^TextHeaders)

// Remove all content. Keep allocation.
text_headers_reset :: proc(th: ^TextHeaders)

// Append one validated header. Trims whitespace. Returns error on empty name or value.
text_headers_append :: proc(th: ^TextHeaders, name: string, value: string) -> FrameError

// Append raw bytes without validation. For decode path only.
text_headers_append_raw :: proc(th: ^TextHeaders, raw: []byte) -> FrameError

// Current byte length of stored headers.
text_headers_len :: proc(th: ^TextHeaders) -> int

// Return a slice view of the raw buffer. Valid until next mutation.
text_headers_bytes :: proc(th: ^TextHeaders) -> []byte

// Iterator over parsed TextHeader values.
TextHeaderIterator :: struct {
    src   : []byte,
    index : int,
}

// Construct iterator over a raw byte slice.
text_header_iterator :: proc(raw: []byte) -> TextHeaderIterator

// Return next TextHeader, or (_, false) when exhausted.
text_header_next :: proc(it: ^TextHeaderIterator) -> (TextHeader, bool)

// Reset iterator to start.
text_header_rewind :: proc(it: ^TextHeaderIterator)
```

---

### 5. body.odin

```odin
package frame

// The body is a plain [dynamic]byte owned by FrameMessage.
// These helpers operate on it directly — no wrapper struct.

// Append bytes to body.
body_append :: proc(body: ^[dynamic]byte, data: []byte) -> FrameError

// Replace body content with data (clear then append).
body_set :: proc(body: ^[dynamic]byte, data: []byte) -> FrameError

// Clear body content. Keep allocation.
body_reset :: proc(body: ^[dynamic]byte)

// Return current content as a slice. Empty slice if nothing written.
body_bytes :: proc(body: ^[dynamic]byte) -> []byte
```

---

### 6. status.odin

```odin
package frame

// Wire status codes. Sent in BinaryHeader.status.
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

// Go-style error type for frame package API.
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
}

frame_status_to_wire :: proc(s: FrameStatus) -> u8
frame_wire_to_status :: proc(raw: u8) -> FrameStatus
```

---

### 7. frame.odin

This is the central file. It defines `FrameMessage` and the lifecycle API.

```odin
package frame

import matryoshka "../matryoshka"  // for PolyNode only
import "core:mem"

// FrameMessage is the transport container.
//
// PolyNode MUST be the first field (offset 0).
// matryoshka casts ^PolyNode back to ^FrameMessage using this assumption.
//
// Callers set: proto flags, channel_number, message_id, text headers, body.
// check_and_prepare fills: text_hdr_len, body_len, message_id (if 0), status.
//
// app_ctx: caller-owned pointer. frame never touches it.
// eng_ctx: runtime-owned pointer. do not touch.
FrameMessage :: struct {
    using poly : matryoshka.PolyNode,  // offset 0 — required

    bhdr       : BinaryHeader,
    text_hdrs  : TextHeaders,
    body       : [dynamic]byte,

    app_ctx    : rawptr,  // caller use
    eng_ctx    : rawptr,  // runtime internal — do not touch
}

// Allocate and initialize a FrameMessage.
// Allocates TextHeaders buffer and body buffer with default capacities.
frame_create :: proc(allocator: mem.Allocator) -> (^FrameMessage, FrameError)

// Free all owned memory.
frame_destroy :: proc(msg: ^FrameMessage)

// Reset to zero state. Keeps allocations. Clears all headers, body, flags.
// After reset, message is not valid for send until headers are set.
frame_reset :: proc(msg: ^FrameMessage)

// Deep copy. Allocates a new FrameMessage with same bhdr, headers, body.
// eng_ctx is NOT copied (it belongs to the runtime of the original).
frame_clone :: proc(msg: ^FrameMessage, allocator: mem.Allocator) -> (^FrameMessage, FrameError)

// Validate fields and compute derived fields (text_hdr_len, body_len, message_id).
// Must be called before encode.
// Sets bhdr.status to the first error encountered.
// Returns FrameError.None if valid.
frame_check_and_prepare :: proc(msg: ^FrameMessage) -> FrameError

// Encode msg to wire format into buf.
// buf must be at least: BINARY_HEADER_SIZE + text_hdr_len + body_len bytes.
// Calls check_and_prepare internally if not already done.
frame_encode :: proc(msg: ^FrameMessage, buf: []byte) -> (int, FrameError)

// Decode wire bytes into msg.
// msg must be already created (frame_create).
// Parses BinaryHeader, then reads text_hdr_len bytes of text headers,
// then reads body_len bytes of body.
frame_decode :: proc(msg: ^FrameMessage, buf: []byte) -> (int, FrameError)

// Total wire size of msg (BINARY_HEADER_SIZE + text_hdr_len + body_len).
// Valid only after check_and_prepare or decode.
frame_wire_size :: proc(msg: ^FrameMessage) -> int

// Get the OpCode from bhdr proto flags.
// Returns error if stored value is not a valid OpCode.
frame_opcode :: proc(msg: ^FrameMessage) -> (OpCode, FrameError)

// True if origin bit says engine. False = application.
frame_is_from_engine :: proc(msg: ^FrameMessage) -> bool

// True if more bit is set (more chunks follow for this message_id).
frame_has_more :: proc(msg: ^FrameMessage) -> bool

// Atomically generate the next unique MessageID.
// frame_check_and_prepare calls this automatically when message_id == 0.
frame_next_id :: proc() -> MessageID
```

---

### 8. validate.odin

`frame_check_and_prepare` runs these checks in order:

1. OpCode must be a valid enum value.
2. Force `origin = application` and `status = Success` (engine sets these, not callers).
3. If role is Response: `message_id` must be non-zero.
4. If type is not Regular: `more` flag must not be set.
5. `channel_number` must be non-zero unless OpCode is `WelcomeRequest` or `HelloRequest`.
6. `text_hdr_len = len(text_hdrs)`, must fit in `u16`.
7. If `text_hdr_len == 0` and OpCode is `WelcomeRequest` or `HelloRequest`: error (address is required in text headers).
8. `body_len = len(body)`, must fit in `u16`.
9. If `message_id == 0`: assign `frame_next_id()`.

On any error: set `bhdr.status` to the matching `FrameStatus` value, return the error.

---

### 9. What Is Not in frame

These things are intentionally absent from the `frame` package:

| Thing | Where it lives |
|-------|---------------|
| PolyNode list linking | `matryoshka` (polynode.odin) |
| Message pooling / recycling | `matryoshka` |
| Mailbox / routing | `matryoshka` |
| Channel management | `runtime` |
| Business logic | `plugins` |
| HTTP adapter | `runtime` |
| Connection lifecycle (Hello/Bye state machine) | `runtime` |

---

### 10. Design Notes

**Why `[dynamic]byte` for body and text headers?**
Odin's `[dynamic]byte` with `append` covers everything `Appendable` did in Zig.
No custom type. No wrapper. Direct slice access when you need a view.

**Why no `TextHeaders` struct with behavior?**
The raw buffer with an iterator is enough. The iterator produces `TextHeader` views
(string slices into the buffer) on demand. No allocation per header.

**Why is `PolyNode` embedded with `using`?**
`using` promotes `prev`, `next`, and `tag` to `FrameMessage` scope. More important:
the cast `(^FrameMessage)(poly_ptr)` is valid only because `PolyNode` is at offset 0.
matryoshka uses this cast. It is a convention enforced by position, not by the compiler.

**Wire format**
The wire byte stream is:
```
[ BinaryHeader (16 bytes, big-endian) ][ text headers (text_hdr_len bytes) ][ body (body_len bytes) ]
```
`text_hdr_len` and `body_len` inside the BinaryHeader tell the decoder how many
bytes follow each section. The decoder does not need a sentinel.

**MessageID generation**
A package-level `sync.Atomic_Int64` (or equivalent) starting at 1. `frame_next_id`
increments with monotonic ordering. Same approach as Zig's `uid.fetchAdd`.

**`app_ctx` and `eng_ctx`**
Two opaque pointers. `app_ctx` is for the caller. `eng_ctx` is for the runtime. frame
never reads or writes either; it just carries them. `frame_clone` copies `app_ctx`,
does not copy `eng_ctx`.

---

### 11. File Dependency Summary

```
opcodes.odin        -- no frame imports
status.odin         -- no frame imports
binary_header.odin  -- imports opcodes, status
text_headers.odin   -- imports status
body.odin           -- imports status
validate.odin       -- imports binary_header, text_headers, body, opcodes, status
frame.odin          -- imports all of the above + matryoshka (PolyNode only)
```

`matryoshka` is imported only for the `PolyNode` type. No matryoshka procedures are
called from `frame`. The dependency is structural (type embedding), not behavioral.
