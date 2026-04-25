# `frame` Package (Odin) — Composed High-Level Design

Version: 1 (composed from `frame-chatgpt.md`, `frame-claude.md`, the
Zig `tofu` sources, the `matryoshka` Odin sources, and the `otofu`
`P*_` / `S*_` analysis set).

This document has two parts:

- **Part 1 — Prompt.** A self-contained prompt you can paste into any
  other AI or give to another architect to reproduce this design.
- **Part 2 — Resulting high-level design.** Files, structs, enums,
  and API signatures only. No implementation bodies.

No emojis. Simple English. Intended for developers whose first
language is not English.

---

## Part 1 — Prompt (reusable)

### Role

You are a software architect. You are an expert in Odin, backend
systems, message-oriented systems, wire protocols, and ownership
models. You design clean layers. You do not write code in the
design phase.

### Background

A Zig project named `tofu` provides messaging building blocks for
modular monoliths. Its central data type is `Message`
(binary header + text headers + body, with an intrusive list link
and two opaque pointers). A separate Odin project named
`matryoshka` provides the structural primitive `PolyNode` and an
ownership token `MayItem`. Every item that travels through
`matryoshka` infrastructure embeds `PolyNode` as the first field
at offset 0. The user is porting `tofu`'s message layer to Odin as
a new separate repository named `frame`, which will be used as a
starting template for Odin services.

### Polynode-centric architecture (paste verbatim if you re-ask this)

The system has five layers:

- `frame`      — transport ABI container.
- `polynode`   — structural primitive (from `matryoshka`), embedded in every item.
- `matryoshka` — runtime substrate operating on polynode items.
- `runtime`    — orchestration + transport adapters (HTTP, TCP, UDS).
- `plugins`    — user-defined decision logic.

Central axiom:

> `matryoshka` operates only on polynode items extracted from frames.
> `matryoshka` never sees or processes raw frame structures.

Dependency rules (strict, no exceptions):

- `runtime` depends on `frame` and `matryoshka`.
- `matryoshka` does NOT depend on `frame`.
- `frame` does NOT depend on `matryoshka` runtime. The only link is
  that `frame` imports the `PolyNode` type from `matryoshka` to embed
  it by value.
- `polynode` is an independent structural primitive.

Frame container shape:

```
FrameMessage {
    PolyNode        (FIRST FIELD, offset 0, required)
    BinaryHeader    (fixed 16-byte packed header, wire-faithful)
    TextHeaders     ("name: value\r\n" key-value pairs)
    Body            (raw bytes)
    app_ctx         (opaque pointer, caller-owned)
    eng_ctx         (opaque pointer, runtime-owned)
}
```

Execution path in the larger system (for context only; `frame` is not
involved after decode):

```
bytes on the wire
    -> runtime transport adapter
    -> frame_decode (or frame_decode_view for zero-copy)
    -> polynode extracted from the decoded frame
    -> runtime dispatch
    -> matryoshka.mailbox   (transports polynode items)
    -> matryoshka.pool      (emits lifecycle events)
    -> plugins              (user decisions)
    -> handler execution
    -> frame_encode
    -> bytes on the wire
```

### Design decisions already taken

These are fixed inputs to the design. Do not revisit them.

1. **Wire format is immutable.** The `BinaryHeader` is exactly 16 bytes,
   packed, big-endian on the wire. The 10 `OpCode` values and the
   `ProtoFields` bit layout match the Zig `tofu` source exactly. The
   text headers format is the HTTP-style `"name: value\r\n"` line format.
2. **Zero-copy decode is a first-class API.** In addition to the owned
   decoder that copies into an owned `FrameMessage`, there is a
   zero-copy `frame_decode_view` that returns slices into the caller's
   input buffer. Both are exported from the same package.
3. **Codec is inside the `frame` package but in its own file
   `frame_codec.odin`.** One import brings both container types and
   wire codec to consumers. The file boundary keeps the codec API
   visibly separate from the container types.
4. **Both `app_ctx` and `eng_ctx` are preserved on `FrameMessage`** as
   `rawptr`. `frame` never reads them. `frame_clone` copies `app_ctx`
   and does not copy `eng_ctx`.

### Design freedom

You choose, as architect:

- Field representations within the wire format constraints.
- Lifecycle API shape (create/destroy/reset/clone contract).
- Validation split between `check_and_prepare` and encode.
- How iterators are expressed in Odin idioms.
- Whether helpers sit on the `FrameMessage` or on its sub-structs.

### Constraints

- `frame` must not contain business logic.
- `frame` must not depend on `matryoshka` at runtime — type import of
  `PolyNode` only.
- `frame` must not interpret semantics beyond what the wire format
  requires.
- Every procedure that allocates must take an explicit allocator.
  No `context.allocator` inside this package.
- All wire encoding is big-endian. Native-endian conversion happens
  at the codec boundary only.

### Source material

Zig tofu message and docs:

- `https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/src/message.zig`
- `https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/message.md`
- `https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/patterns.md`

Odin matryoshka base:

- `https://raw.githubusercontent.com/g41797/matryoshka/refs/heads/main/polynode.odin`

otofu analysis set (state machines, ownership, module catalog,
boundaries, package layout, tofu mapping):

- `https://github.com/g41797/tofusite/tree/main/root/otofu/analysis`

Local checkouts (if available):

- `/home/g41797/dev/root/github.com/g41797/matryoshka/`
- `/home/g41797/dev/root/github.com/g41797/tofu/`
- `/home/g41797/dev/root/github.com/g41797/tofusite/root/otofu/analysis/P*.md`
- `/home/g41797/dev/root/github.com/g41797/tofusite/root/otofu/analysis/S*.md`

### Output

Return a single Markdown document with two sections:

1. **Prompt** — reusable, self-contained.
2. **Resulting high-level design** — files, structs, enums, API
   signatures. No procedure bodies. No implementation.

Simple English. No marketing language. No emojis. No AI-ish filler.

### Before writing

Confirm your understanding. Ask clarifying questions if any
requirement is ambiguous. Do not write the final document until the
open questions are resolved.

---

## Part 2 — Resulting High-Level Design

### 0. One-line definition

`frame` is a passive transport container that carries a `PolyNode`
plus a tofu-wire-format message (binary header, text headers, body),
and provides the exact encode/decode boundary used by the runtime.

### 1. Purpose

The `frame` package exists to:

- Define the in-memory shape of a frame message.
- Carry a `PolyNode` as the first field of that shape, so the
  `matryoshka` runtime can transport items without knowing the
  frame at all.
- Define the exact byte-for-byte wire format (identical to Zig tofu).
- Provide encode, owned decode, and zero-copy view decode.

It does NOT:

- Route messages, pool messages, or manage mailboxes.
- Decide protocol semantics (Hello/Bye/Welcome handshake, etc.).
- Parse or build network addresses.
- Perform I/O.
- Hold runtime state.

### 2. Package structure

One Odin package, `frame`. One directory.

```
frame/
    frame.odin                 -- package doc + re-exports
    frame_message.odin         -- FrameMessage + lifecycle
    frame_opcodes.odin         -- OpCode, MessageType, MessageRole, derived procs
    frame_proto.odin           -- ProtoFields bit-field, OriginFlag, MoreFlag
    frame_binary_header.odin   -- BinaryHeader + wire helpers
    frame_text_headers.odin    -- TextHeaders owned buffer, TextHeader, iterator
    frame_body.odin            -- body helpers over [dynamic]byte
    frame_status.odin          -- FrameStatus (u8 wire) + FrameError (API)
    frame_validate.odin        -- frame_check_and_prepare
    frame_views.odin           -- FrameMessageView, view iterator
    frame_codec.odin           -- encode, decode (owned), decode_view (zero-copy)
```

### 3. Internal dependency layering

```
frame_opcodes.odin          -> (none)
frame_proto.odin            -> frame_opcodes
frame_status.odin           -> (none)
frame_binary_header.odin    -> frame_proto, frame_status
frame_text_headers.odin     -> frame_status
frame_body.odin             -> frame_status
frame_validate.odin         -> frame_binary_header, frame_text_headers,
                               frame_body, frame_opcodes, frame_status
frame_message.odin          -> all above + matryoshka (PolyNode type only)
frame_views.odin            -> frame_binary_header, frame_text_headers
frame_codec.odin            -> frame_message, frame_views
frame.odin                  -> package doc; no imports beyond the above
```

No cycles. No upward calls.

### 4. Core types

#### 4.1 `FrameMessage` (frame_message.odin)

```odin
FrameMessage :: struct {
    using poly : matryoshka.PolyNode,   // offset 0 — required

    bhdr       : BinaryHeader,
    text_hdrs  : TextHeaders,
    body       : [dynamic]byte,

    app_ctx    : rawptr,   // caller escape hatch; frame never reads
    eng_ctx    : rawptr,   // runtime escape hatch; frame never reads
}
```

Rules:

- `PolyNode` must be the first field. `matryoshka` casts
  `(^FrameMessage)(poly_ptr)` and relies on offset 0. This is not
  checked by the compiler; it is enforced by convention and by code
  review.
- `app_ctx` and `eng_ctx` are opaque. `frame_clone` copies `app_ctx`
  and does not copy `eng_ctx`.
- `body` is a plain `[dynamic]byte`. No `Appendable` wrapper. Odin's
  built-in dynamic array already provides what Zig needed `Appendable`
  for.

#### 4.2 `BinaryHeader` (frame_binary_header.odin)

Wire format is identical to Zig tofu. 16 bytes, packed, big-endian.

```odin
ChannelNumber :: distinct u16
MessageID     :: distinct u64

BinaryHeader :: struct #packed {
    channel_number : ChannelNumber,   // u16 on the wire, big-endian
    proto          : ProtoFields,     // u8 bit-field
    status         : u8,              // FrameStatus raw
    message_id     : MessageID,       // u64 on the wire, big-endian
    text_hdr_len   : u16,             // filled by validate; not by caller
    body_len       : u16,             // filled by validate; not by caller
}

BINARY_HEADER_SIZE :: size_of(BinaryHeader)
#assert(BINARY_HEADER_SIZE == 16)
```

Fields `text_hdr_len` and `body_len` are engine-internal. Caller
must not set them directly. `frame_check_and_prepare` fills them
from the current buffer sizes.

#### 4.3 `ProtoFields`, flags (frame_proto.odin)

```odin
OriginFlag :: enum u8 {
    Application = 0,
    Engine      = 1,
}

MoreFlag :: enum u8 {
    Last = 0,
    More = 1,
}

ProtoFields :: bit_field u8 {
    op_code : u4,   // OpCode value
    origin  : u1,   // 0 application, 1 engine
    more    : u1,   // 0 last, 1 more chunks follow
    _a      : u1,   // engine internal
    _b      : u1,   // engine internal
}
```

#### 4.4 `OpCode`, `MessageType`, `MessageRole` (frame_opcodes.odin)

10 `OpCode` values, matching Zig tofu exactly.

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

MessageType :: enum u8 {
    Regular = 0,
    Welcome = 1,
    Hello   = 2,
    Bye     = 3,
}

MessageRole :: enum u8 {
    Request  = 0,
    Response = 1,
    Signal   = 2,
}
```

Note: `OpCode` is stored in a 4-bit slot of `ProtoFields`. The
enum's underlying type is widened to `u8` for Odin ergonomics, but
only values 0..9 are legal.

#### 4.5 `TextHeaders`, `TextHeader`, iterator (frame_text_headers.odin)

```odin
TextHeader :: struct {
    name  : string,
    value : string,
}

TextHeaders :: struct {
    buf : [dynamic]byte,   // raw bytes "name: value\r\n" repeated
}

TextHeaderIterator :: struct {
    src   : []byte,
    index : int,
}
```

`TextHeader` slices point into a backing buffer (either `TextHeaders.buf`
for owned data, or a caller-supplied `[]byte` for decode views). The
views stay valid until the backing buffer changes.

#### 4.6 `Body` (frame_body.odin)

No wrapper type. The body is a plain `[dynamic]byte` owned by the
`FrameMessage`. Helpers below operate on `^[dynamic]byte` directly.

#### 4.7 `FrameStatus` and `FrameError` (frame_status.odin)

```odin
// On-wire status byte in BinaryHeader.status.
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

// Return value for package procedures.
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

Two domains: `FrameStatus` is a wire field (peer-reported); `FrameError`
is a local API error. Conversion procedures are listed below.

### 5. View types (frame_views.odin)

Zero-copy decode. The view holds slices that point into the input
buffer. The caller keeps the buffer alive for the life of the view.

```odin
FrameMessageView :: struct {
    poly          : ^matryoshka.PolyNode,  // may be nil if not present
    bhdr          : BinaryHeader,          // copied by value (16 bytes)
    text_hdrs_raw : []byte,                // slice into input
    body          : []byte,                // slice into input
}

TextHeaderIteratorView :: distinct TextHeaderIterator
```

The same iterator shape is reused; the distinct type is only a label
to make the caller aware the slices are not owned.

### 6. API signatures (no bodies)

All signatures are grouped by file. Return types included. No
procedure bodies.

#### 6.1 Lifecycle (frame_message.odin)

```odin
frame_create  :: proc(allocator: mem.Allocator) -> (^FrameMessage, FrameError)
frame_destroy :: proc(msg: ^FrameMessage)
frame_reset   :: proc(msg: ^FrameMessage)
frame_clone   :: proc(msg: ^FrameMessage, allocator: mem.Allocator) -> (^FrameMessage, FrameError)
```

Contracts:

- `frame_create` allocates the `FrameMessage`, and initializes
  `text_hdrs.buf` and `body` with a small starting capacity.
- `frame_destroy` frees all owned memory. Safe on nil.
- `frame_reset` clears content; keeps allocations. After reset the
  message is not valid for send until headers are set.
- `frame_clone` makes a deep copy of `bhdr`, `text_hdrs.buf`, and
  `body`. Copies `app_ctx`. Does not copy `eng_ctx`. Does not copy
  the intrusive list linkage (the clone is detached).

#### 6.2 Validation (frame_validate.odin)

```odin
frame_check_and_prepare :: proc(msg: ^FrameMessage) -> FrameError
```

Order of checks:

1. `OpCode` must be a valid enum value.
2. Force `proto.origin = Application` and `status = Success`
   (engine sets these, not callers).
3. If role is Response: `message_id` must be non-zero.
4. If type is not Regular: `more` flag must not be set.
5. `channel_number` must be non-zero unless the OpCode is
   `WelcomeRequest` or `HelloRequest`.
6. `text_hdr_len = len(text_hdrs.buf)`; must fit in `u16`.
7. If `text_hdr_len == 0` and OpCode is `WelcomeRequest` or
   `HelloRequest`: fail with `WrongAddress`.
8. `body_len = len(body)`; must fit in `u16`.
9. If `message_id == 0`: assign `frame_next_id()`.

On any failure: set `bhdr.status` to the matching `FrameStatus`
value and return the corresponding `FrameError`.

#### 6.3 Codec (frame_codec.odin)

```odin
// Owned encode. Writes wire bytes into out_buf.
// Requires len(out_buf) >= BINARY_HEADER_SIZE + text_hdr_len + body_len.
// Calls frame_check_and_prepare internally if not already prepared.
// Returns number of bytes written.
frame_encode :: proc(msg: ^FrameMessage, out_buf: []byte) -> (int, FrameError)

// Owned decode. Copies into msg (which must already exist via frame_create).
// Parses BinaryHeader, then text_hdr_len bytes into text_hdrs.buf,
// then body_len bytes into body.
// Returns number of bytes consumed from in_buf.
frame_decode :: proc(msg: ^FrameMessage, in_buf: []byte) -> (int, FrameError)

// Zero-copy decode. Fills a view whose slices point into in_buf.
// Does not allocate. Does not copy. Caller must keep in_buf alive
// until the view is no longer used.
frame_decode_view :: proc(in_buf: []byte) -> (FrameMessageView, int, FrameError)

// Total wire size = BINARY_HEADER_SIZE + text_hdr_len + body_len.
// Valid only after frame_check_and_prepare or frame_decode.
frame_wire_size :: proc(msg: ^FrameMessage) -> int
```

#### 6.4 BinaryHeader helpers (frame_binary_header.odin)

```odin
// Encode bh into buf (big-endian, BINARY_HEADER_SIZE bytes).
binary_header_encode :: proc(bh: ^BinaryHeader, buf: []byte) -> FrameError

// Decode BINARY_HEADER_SIZE big-endian bytes from buf into bh.
// Validates op_code range.
binary_header_decode :: proc(buf: []byte, bh: ^BinaryHeader) -> FrameError

// Zero all fields.
binary_header_reset :: proc(bh: ^BinaryHeader)
```

#### 6.5 OpCode helpers (frame_opcodes.odin)

```odin
opcode_valid :: proc(oc: OpCode) -> bool
opcode_type  :: proc(oc: OpCode) -> MessageType
opcode_role  :: proc(oc: OpCode) -> MessageRole

// Returns the matching response OpCode for a request.
// Signal returns Signal. Response and other non-request values return
// FrameError.InvalidOpCode.
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

Owned buffer:

```odin
text_headers_init        :: proc(th: ^TextHeaders, allocator: mem.Allocator, capacity: int) -> FrameError
text_headers_destroy     :: proc(th: ^TextHeaders)
text_headers_reset       :: proc(th: ^TextHeaders)
text_headers_append      :: proc(th: ^TextHeaders, name: string, value: string) -> FrameError
text_headers_append_raw  :: proc(th: ^TextHeaders, raw: []byte) -> FrameError     // decode path
text_headers_len         :: proc(th: ^TextHeaders) -> int
text_headers_bytes       :: proc(th: ^TextHeaders) -> []byte
```

Iterator (works on either owned or view data):

```odin
text_header_iterator :: proc(raw: []byte) -> TextHeaderIterator
text_header_next     :: proc(it: ^TextHeaderIterator) -> (TextHeader, bool)
text_header_rewind   :: proc(it: ^TextHeaderIterator)
```

`TextHeaderIteratorView` reuses the same procedures via explicit cast
back to `TextHeaderIterator` at the call site.

#### 6.8 Body helpers (frame_body.odin)

```odin
body_append :: proc(body: ^[dynamic]byte, data: []byte) -> FrameError
body_set    :: proc(body: ^[dynamic]byte, data: []byte) -> FrameError
body_reset  :: proc(body: ^[dynamic]byte)
body_bytes  :: proc(body: ^[dynamic]byte) -> []byte
```

#### 6.9 BinaryHeader-derived readers on FrameMessage (frame_message.odin)

```odin
frame_opcode         :: proc(msg: ^FrameMessage) -> (OpCode, FrameError)
frame_is_from_engine :: proc(msg: ^FrameMessage) -> bool
frame_has_more       :: proc(msg: ^FrameMessage) -> bool

// Atomically generate the next unique MessageID.
// Package-level atomic u64 starting at 1, monotonic.
frame_next_id :: proc() -> MessageID
```

#### 6.10 Status conversion (frame_status.odin)

```odin
frame_status_to_wire :: proc(s: FrameStatus) -> u8
frame_wire_to_status :: proc(raw: u8) -> FrameStatus
```

### 7. Wire format

Single, fixed layout. Big-endian. Match tofu exactly.

```
offset  size  field
------  ----  ----------------
  0      2    channel_number   (u16 big-endian)
  2      1    proto            (ProtoFields bit_field)
  3      1    status           (FrameStatus raw)
  4      8    message_id       (u64 big-endian)
 12      2    text_hdr_len     (u16 big-endian)
 14      2    body_len         (u16 big-endian)
 16      -    text headers     (text_hdr_len bytes, "name: value\r\n" repeated)
  .      -    body             (body_len bytes)
```

Total prefix is 16 bytes. `#assert(size_of(BinaryHeader) == 16)`.

Maximum `text_hdr_len` and `body_len` are each `65535` bytes (u16).
Payloads larger than this must be split by the runtime using the
`more` flag — the split logic lives in the runtime, not in `frame`.

### 8. Views and ownership

- The owned `FrameMessage` owns its `text_hdrs.buf` and `body`
  buffers. Lifetime is bounded by `frame_destroy`.
- `FrameMessageView` owns nothing. Its slices point into the
  decode input buffer. If the input buffer is freed, reused, or
  overwritten, the view becomes invalid.
- Conversion from view to owned is the caller's responsibility
  (the runtime decides the policy). A helper is intentionally not
  provided here to keep `frame` free of allocation policies.
- `app_ctx` and `eng_ctx` are opaque. `frame` never reads, writes,
  frees, or inspects them. They just ride along.

### 9. Interaction with polynode and matryoshka

- `frame` imports the `matryoshka` package only for the `PolyNode`
  type. No `matryoshka` procedures are called from `frame`.
- `matryoshka` never imports `frame`.
- The offset-0 rule lets `matryoshka` transport `^PolyNode` through
  mailboxes and pools without knowing the concrete item type. When
  `runtime` needs the full frame, it casts `(^FrameMessage)(poly_ptr)`.
- `frame` does not call `polynode_reset` or any other `matryoshka`
  procedure. The runtime is responsible for that.

### 10. MessageID generation

- Package-level atomic `u64`, initialized to 1, monotonic order.
- Incremented by `frame_next_id`.
- Called automatically by `frame_check_and_prepare` when
  `message_id == 0`. Otherwise the caller's value is preserved.

This matches Zig `tofu`'s `next_mid` behavior.

### 11. Allocation rules

- Every procedure that allocates accepts an `Allocator` parameter
  explicitly.
- No `context.allocator` inside the `frame` package.
- `frame_create` allocates the `FrameMessage`, the `text_hdrs.buf`
  backing array, and the `body` backing array. All three are freed
  by `frame_destroy`.
- `frame_reset` keeps all three allocations; only clears content.
- `frame_clone` uses the allocator passed to it, not any stored
  allocator of the source.

### 12. What `frame` does NOT provide

This list is intentional. If any of these appear inside `frame`,
the layer boundary is broken.

| Concern                                          | Where it lives           |
|--------------------------------------------------|--------------------------|
| PolyNode list linking / unlinking                | `matryoshka.polynode`    |
| Message pools, recycling                         | `matryoshka.pool`        |
| Mailboxes, cross-thread transfer                 | `matryoshka.mailbox`     |
| Channel state machines (Hello/Bye/Welcome)       | `chanmgr` / `protocol`   |
| Address types and header helpers                 | `runtime` or sibling pkg |
| `~connect_to`, `~listen_on` header formats       | `runtime`                |
| HTTP / TCP / UDS adapters                        | `runtime`                |
| Business logic and routing decisions             | `plugins`                |

### 13. Error model

Two separate domains:

- **Wire status** (`BinaryHeader.status`, `FrameStatus`): what the
  peer reports on a frame. Preserved exactly from `tofu`.
- **API error** (`FrameError`): what a `frame` procedure returns to
  its caller. Covers local issues: allocation failure, buffer too
  small, decode malformed, validation failure, and so on.

`frame_check_and_prepare` links the two: on validation failure, it
sets `bhdr.status` to the matching `FrameStatus` and returns the
matching `FrameError`.

### 14. Concurrency

`frame` itself has no threading concerns. A `FrameMessage` is owned
by exactly one thread at a time. The only shared state in the whole
package is the `frame_next_id` counter, which is atomic.

The `matryoshka` layer (mailboxes, pools) is what moves items
across threads. `frame` does not participate.

### 15. Extension points (outside the scope of version 1)

- Streaming decoder that consumes a partial byte buffer across
  multiple feeds (for TCP reassembly in the runtime).
- Optional compression wrapper around the codec.
- Optional encryption wrapper around the codec.
- Matching set of wire tests that lock the format.

These are not part of version 1 of `frame`.

### 16. Constraints checklist

- [x] `PolyNode` is the first field of `FrameMessage`.
- [x] `BinaryHeader` is exactly 16 bytes, big-endian, wire-faithful.
- [x] 10 `OpCode` values, names and values match `tofu`.
- [x] Text headers use `"name: value\r\n"`.
- [x] Body is a plain `[dynamic]byte` (no `Appendable` wrapper).
- [x] `app_ctx` and `eng_ctx` preserved; never read by `frame`.
- [x] Owned decode AND zero-copy view decode both exported.
- [x] Codec in its own file `frame_codec.odin`.
- [x] No `matryoshka` runtime dependency; type import of `PolyNode`
      only.
- [x] No business logic, no routing, no pools, no mailboxes.
- [x] Explicit allocator on every allocating procedure.

### 17. One-line recap

`frame` is a small, passive Odin package: a polynode-first message
container with a wire-faithful codec, exposed as both owned and
zero-copy decode, with zero dependencies on the runtime it serves.
