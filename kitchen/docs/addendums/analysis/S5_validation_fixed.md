# S5 — Structure Validation (Fixed)

Sources: S4_file_structure_fixed.md, P3_ownership.md, P7_otofu_architecture.md
Pipeline position: S5 (validation — checks S4 against ownership rules, layering constraints, and architecture invariants)

---

## Corrections Applied

The original S5_validation.md was based on S4_file_structure.md, which contained critical violations of the five immutability rules. The following problems were identified as missing from the original validation and have been added here as P-08 through P-14:

| Rule Violated | New Problem | Summary |
|--------------|-------------|---------|
| Architecture | P-08 CRITICAL | `Message` struct in `internal/runtime` — clients cannot use `*Message` without importing internal package |
| RULE-3 | P-09 CRITICAL | OpCode set entirely replaced — wire protocol broken at the enum level |
| RULE-1 | P-10 CRITICAL | BinaryHeader structure changed — `status`, `message_id`, `ProtoFields` fields lost |
| RULE-1 | P-11 CRITICAL | TextHeaders replaced by raw `meta` buffer — wire text-header protocol broken |
| RULE-4 | P-12 SIGNIFICANT | UDS support removed — TCP-only `Address` struct; UDS peers cannot connect |
| RULE-2 | P-13 SIGNIFICANT | `updateReceiver` removed from ChannelGroup — public API diverges from tofu |
| RULE-2 | P-14 SIGNIFICANT | `getAllocator` removed from Engine — public API diverges from tofu |

The original P-01 through P-07 (structural/ownership problems in S4) are carried forward unchanged.
The original P-08 (framing layering) and P-09 (drain accounting) are renumbered P-15 and P-16.

---

## Rules Reminder

```
RULE-1  Wire format is immutable: BinaryHeader layout, ProtoFields encoding, TextHeader format
        ("name: value\r\n"), and body framing must be preserved exactly.
RULE-2  Public API is immutable: Ampe.get / put / create / destroy / getAllocator;
        ChannelGroup.post / waitReceive / updateReceiver.
RULE-3  OpCode enum must match tofu exactly: 10 values
        (Request, Response, Signal, HelloRequest, HelloResponse,
         ByeRequest, ByeResponse, ByeSignal, WelcomeRequest, WelcomeResponse).
RULE-4  UDS (Unix Domain Sockets) must be supported alongside TCP.
RULE-5  Matryoshka API is otofu-internal; client code has no dependency on Matryoshka types.
```

---

## Severity Legend

| Severity | Meaning |
|----------|---------|
| **CRITICAL** | Prevents compilation or creates a silent correctness bug. Must be resolved before any code is written. |
| **SIGNIFICANT** | Design gap or structural ambiguity that will cause implementation errors or undefined behavior at runtime. |
| **MINOR** | Inconsistency with a prior phase document. Negligible runtime risk but creates drift between specification and implementation. |

---

## Problem List

---

### P-01 · CRITICAL · Circular Import — `reactor_state` references `engine_state`

**Location:** `reactor/reactor.odin` — `reactor_state.engine_state: ^engine_state`

**Problem:**

`engine_state` (renamed `engine_internal` in S4) is defined in `otofu/engine.odin`, which is in the `otofu` root package. `reactor/reactor.odin` is in the `reactor` package.

The import graph established in S3 is:
```
otofu → reactor   (otofu imports reactor to call Reactor_Start)
```

If `reactor` holds a field of type `^engine_internal`, then `reactor` must import `otofu`. That creates:
```
otofu → reactor → otofu   ← CYCLE
```

Odin does not permit circular package imports. This will not compile.

**Evidence in S4:**

```odin
reactor_state :: struct {
    ...
    engine_state: ^engine_state,   // defined in otofu/engine.odin; imported indirectly
    ...
}
```

The comment "imported indirectly" does not resolve the cycle. There is no indirect import mechanism in Odin.

**Root cause:** The Reactor needs some Engine state (drain flag, options, allocator) after startup. Passing a back-pointer to `engine_internal` was used as a shortcut, but creates a cycle.

**Fix:**

Remove `engine_state: ^engine_internal` from `reactor_state`. Replace with the specific fields the Reactor actually needs:

```odin
reactor_state :: struct {
    ...
    // From engine — passed at startup, owned by engine_internal, valid for Reactor lifetime
    options:   Engine_Options,           // copied by value at startup; no pointer needed
    allocator: mem.Allocator,            // already present; keep
    eng_state: ^types.Engine_State,     // pointer to types-level enum only; no otofu import
    ...
}
```

`Engine_Options` must be moved from `otofu/engine.odin` to `types/` (e.g., `types/options.odin`), or passed to `Reactor_Start` as a parameter and stored by value. Either way, `reactor` never holds a pointer into `otofu`-defined structures.

The `^types.Engine_State` field holds a pointer to a field in `engine_internal`, but the field type is from `types` — a package `reactor` already imports. No cycle.

---

### P-02 · CRITICAL · Import Impossibility — `Dispatch_Context` crosses from `reactor` to `protocol`

**Location:** `reactor/reactor.odin` (defines `Dispatch_Context`) and `protocol/protocol.odin` (calls `Protocol_Dispatch_Inbound`)

**Problem:**

S4 states:
> `Protocol_Dispatch_Inbound` is called by `reactor/io_dispatch.odin` with a decoded inbound Message and a `Dispatch_Context`.

`Dispatch_Context` is defined in `reactor/reactor.odin`. The import graph is:
```
protocol imports: types, chanmgr, runtime   (NOT reactor)
```

`protocol` cannot use `reactor.Dispatch_Context` in its procedure signatures without importing `reactor`. Importing `reactor` from `protocol` would create:
```
reactor → protocol → reactor   ← CYCLE
```

This is both a semantic violation (S3: protocol must not import reactor) and a compile-time impossibility.

**Root cause:** Two separate context structs were designed (`Dispatch_Context` and `Protocol_Context`) but the boundary between them was not enforced in the procedure signatures.

**Fix:**

`Protocol_Dispatch_Inbound` and `Protocol_Dispatch_Command` must take `^Protocol_Context`, not `^Dispatch_Context`.

The conversion from `Dispatch_Context` → `Protocol_Context` is performed in `reactor/io_dispatch.odin` before calling protocol procedures. `io_dispatch.odin` is in the `reactor` package and can see both types.

`Dispatch_Context` is reactor-internal only. It must never appear in a procedure exported from `protocol`. Remove `^Dual_Map` and `^Timeout_Manager` from `Dispatch_Context` entirely — those fields are never needed by the protocol layer, and their presence only creates the temptation to pass `Dispatch_Context` directly.

Revised `Dispatch_Context` (reactor-internal only, not exported):
```odin
dispatch_context :: struct {   // lowercase: package-private
    router:        ^runtime.Router,
    reserved_pool: ^runtime.Reserved_Pool,
    channel:       ^chanmgr.Channel,
    engine_state:  types.Engine_State,
    allocator:     mem.Allocator,
}
```

This type is never passed to `protocol`. `io_dispatch.odin` constructs a `protocol.Protocol_Context` from it.

---

### P-03 · CRITICAL · Dual Ownership — `Reserved_Pool` in both `engine_internal` and `reactor_state`

**Location:** `otofu/engine.odin` and `reactor/reactor.odin`

**Problem:**

S4 defines `reserved_pool: runtime.Reserved_Pool` as a field in both:
- `engine_internal` (in `otofu/engine.odin`)
- `reactor_state` (in `reactor/reactor.odin`)

P7 is unambiguous:
> "Reactor owns all L2 submodules... and the Reserved Message Pool."

Having it in `engine_internal` contradicts this. Two instances of `Reserved_Pool` cannot exist simultaneously — they would be different pools. If Engine initializes one and Reactor initializes another, which one does `RP_Get` draw from? The pools are not synchronized.

**Root cause:** Engine needs to perform Matryoshka teardown of all pools (P4 Teardown Order 1-8), so `RP_Close` must be called somewhere. Engine.destroy was incorrectly identified as the caller, so a pool reference was added to `engine_internal`.

**Fix:**

Remove `reserved_pool` from `engine_internal`. The Reserved_Pool is owned exclusively by the Reactor and lives only in `reactor_state`.

Teardown sequencing:
- `Engine_Destroy` signals drain (`eng_state = .Draining`) and joins the Reactor thread.
- The Reactor, upon completing the drain phase, calls `RP_Close` internally, then returns from `Reactor_Start`.
- After the Reactor thread joins, all pools are already closed. Engine then closes `message_pool` and frees `engine_internal`.

This preserves the P4 teardown order and keeps Reserved_Pool Reactor-owned.

---

### P-04 · CRITICAL · Missing Channel Ownership Collection in `reactor_state`

**Location:** `reactor/reactor.odin` — `reactor_state` struct

**Problem:**

`reactor_state` has no primary collection of owned Channel pointers:
```odin
reactor_state :: struct {
    ...
    pending_close: [dynamic]^chanmgr.Channel,  // scratch only; populated per-iteration
    ...
}
```

P7 requires the Reactor to visit all open Channels during:
- Phase 1: scan all open channels for earliest deadline
- Phase 4: collect expired-deadline channels
- Phase 9: drain check requires "no open channels"

Without a persistent channel collection, the Reactor cannot enumerate channels across iterations. `pending_close` is per-iteration scratch — it is cleared at the start of each iteration and populated during Phase 4/7. It cannot serve as the primary collection.

**Root cause:** The channel ownership collection was not included in the `reactor_state` definition in S4.

**Fix:**

Add a channel list to `reactor_state`:

```odin
reactor_state :: struct {
    ...
    channels: [dynamic]^chanmgr.Channel,   // all open channels; Reactor owns their lifetime
    ...
}
```

Channels are appended on open (Phase 5 / Phase 7 accept), removed on close (Phase 8). This is the canonical ownership list. Phase 1 iterates it for deadline scan. Phase 4 iterates it for timeout collection. Phase 9 checks `len(channels) == 0`.

Note: `pending_close` remains as per-iteration scratch, populated from `channels` when a channel needs closing.

---

### P-05 · SIGNIFICANT · Socket Ownership Undocumented — `Channel.socket: ^platform.Socket`

**Location:** `chanmgr/channel.odin` — `Channel.socket` field

**Problem:**

S4 describes `Channel.socket` as a "non-owning reference; lifetime tied to Channel." No structure in S4 claims ownership of the Socket. The comment is self-contradictory: a lifetime-tied reference is effectively ownership.

P3 is unambiguous:
> "Socket — Owner: Reactor thread. Exclusively. For the entire Socket lifetime."

P7:
> "Socket struct owned by Reactor. FD owned by Socket struct."

But `reactor_state` has no Socket collection. Who allocates the Socket? Who frees it? `io_dispatch.odin` calls `Socket_Create` and `Socket_Accept`, but where is the returned `^platform.Socket` stored with ownership?

**Root cause:** Socket ownership was described at the level of "the Reactor thread owns it" without being mapped to a concrete struct field in `reactor_state`.

**Two valid fixes:**

**Option A — Embed Socket directly in Channel (cleanest):**

```odin
Channel :: struct {
    using poly:    matryoshka.PolyNode,
    number:        types.ChannelNumber,
    state:         types.Channel_State,
    cg_id:         types.ChannelGroupId,
    remote_number: types.ChannelNumber,
    socket:        platform.Socket,    // embedded by value; Channel owns it
    outbound:      Outbound_Queue,
}
```

Channel is Reactor-owned. Embedding Socket by value makes Channel the owner. Freed when Channel is freed (Phase 8). No separate ownership tracking needed.

**Option B — Add socket table to `reactor_state`:**

```odin
reactor_state :: struct {
    ...
    sockets: map[types.ChannelNumber]^platform.Socket,
    ...
}
```

Channel holds a non-owning pointer; reactor_state holds the owning map. Freed in Phase 8 when channel is closed.

Option A is preferred: it eliminates a separate map lookup and makes the ownership unambiguous. The downside (chanmgr imports platform) already exists.

---

### P-06 · SIGNIFICANT · Dual Tracking of Deadline — `Channel.deadline_ms` and `Timeout_Manager.deadlines`

**Location:** `chanmgr/channel.odin` and `reactor/timeout.odin`

**Problem:**

S4 defines deadline data in two places:
1. `Channel.deadline_ms: i64` (in `chanmgr/channel.odin`)
2. `Timeout_Manager.deadlines: map[types.ChannelNumber]i64` (in `reactor/timeout.odin`)

These are two separate representations of the same datum. Two sources of truth will diverge.

P7 assigns timeout management exclusively to `timeout_manager`:
> "timeout_manager — Purpose: Track per-Channel deadlines."

S2 boundaries for `channel_manager`:
> "channel_manager does NOT own deadlines."

**Fix:**

Remove `deadline_ms: i64` from `Channel` struct. All deadline data lives exclusively in `Timeout_Manager`. The `Channel` struct does not know about deadlines — it knows only about state.

`io_dispatch.odin` sets/clears deadlines in `Timeout_Manager` after calling `Ch_Transition`. Phase 1 reads `Timeout_Manager` to compute the poll timeout. Phase 4 reads `Timeout_Manager` to collect expired channels. No deadline data is stored in Channel.

---

### P-07 · SIGNIFICANT · Missing Per-Channel Receive Buffer

**Location:** `chanmgr/channel.odin` and `reactor/reactor.odin`

**Problem:**

TCP is a byte stream. The framer (`Framer_Try_Decode`) requires accumulating bytes until a complete frame is available. Between Reactor iterations, a partial frame may be buffered. There is no per-channel receive buffer in any struct in S4.

`reactor_state` has no `recv_bufs` field. `Channel` has no `recv_buf` field.

Without a receive buffer:
- `Socket_Recv` writes into a temporary buffer (where? the stack?)
- If the read yields a partial frame, there is nowhere to store the unconsumed bytes
- Next iteration starts fresh and loses the partial frame
- `Framer_Try_Decode` never receives a complete frame for multi-iteration messages

This is a completeness gap, not an oversight — P6 and P4 both describe framing but neither specifies where the accumulation buffer lives.

**Fix:**

Add a per-channel receive buffer to `Channel`:

```odin
Channel :: struct {
    ...
    recv_buf: [dynamic]u8,   // partial-frame accumulation; reset to len=0 on channel open
    ...
}
```

Lifetime: allocated on first recv, freed when Channel is freed (Phase 8). Reset to `len = 0` (not freed) between frames to avoid reallocation.

Alternatively, store in `reactor_state` as `recv_bufs: map[ChannelNumber][dynamic]u8`. The Channel field is simpler and avoids a map lookup on every read.

This adds an import or a type from `chanmgr` — `chanmgr` can use `[dynamic]u8` directly without any new import.

---

### P-08 · CRITICAL · `Message` Struct in Internal Package — Public API Inaccessible (NEW)

**Location:** `internal/runtime/message.odin` (original S4)

**Violated rule:** Architecture constraint (Odin has no re-export mechanism)

**Problem:**

The original S4 placed `Message` struct in `internal/runtime/message.odin`. This makes `Message` part of an internal package. Yet `Message` is the central type of the public API:

- `Engine_Get(strategy) -> ?^message.Message`  — returns `*Message`
- `Engine_Put(msg: ^message.Message)`  — takes `*Message`
- `CG_Post(chnls, msg: ^?^message.Message) -> BinaryHeader`  — takes `**Message`
- `CG_Wait_Receive(chnls, timeout_ns) -> ?^message.Message`  — returns `*Message`

In Odin, `internal/` packages are accessible only to code within the module. Client applications are outside the module. If `Message` lives in `internal/runtime`, client code cannot name the type `runtime.Message` — the package path is unreachable.

Zig has `pub usingnamespace`; Odin has no equivalent. There is no re-export mechanism. A client cannot do:

```odin
import "github.com/.../otofu"
msg := otofu.Engine_Get(engine, .always)   // returns ?^runtime.Message
// ERROR: client cannot name ^runtime.Message — internal package is opaque
```

The API is structurally impossible to use.

**Fix (implemented in S4_file_structure_fixed.md):**

Move `Message` and all its sub-types to a new top-level package `message/`:

```
otofu/
├── message/
│   └── message.odin    ← Message, BinaryHeader, TextHeaders, ProtoFields (PUBLIC)
├── internal/
│   └── runtime/
│       └── ...         ← framer, pools, router (internal; no message.odin here)
```

Client code imports `message` directly:
```odin
import msg "github.com/.../otofu/message"

m := engine.get(.always)   // returns ?^msg.Message
```

`PolyNode` is embedded in `Message` at offset 0 (RULE-5: internal implementation detail), but clients access only `bh`, `thdrs`, `body` — the Matryoshka field is never documented in the public API.

**Required before:** Any code is written.

---

### P-09 · CRITICAL · OpCode Enum Replaced — Wire Protocol Broken (NEW)

**Location:** `types/opcodes.odin` (original S4)

**Violated rule:** RULE-3

**Problem:**

The original S4 defined a 10-value OpCode enum with these values:
```odin
OpCode :: enum u8 {
    Connect, Listen, Close, Drain, Hello, Welcome, Bye, Bye_Ack, Data, Channel_Closed,
}
```

None of these match the tofu wire protocol opcodes. The actual tofu opcodes (`message.zig` lines 22–31) are:

```
Request=0, Response=1, Signal=2,
HelloRequest=3, HelloResponse=4,
ByeRequest=5, ByeResponse=6, ByeSignal=7,
WelcomeRequest=8, WelcomeResponse=9
```

The OpCode is encoded in `ProtoFields.opCode` (u4 field of the wire BinaryHeader). Any otofu instance using the wrong enum:
- Sends messages with opcodes tofu peers do not recognize
- Interprets received opcodes incorrectly
- Cannot complete handshake (`HelloRequest`/`HelloResponse`/`WelcomeRequest`/`WelcomeResponse`) or shutdown (`ByeRequest`/`ByeResponse`/`ByeSignal`)

Every inter-process frame exchanged with a tofu peer will be misinterpreted. This is not a minor divergence — it breaks the protocol at byte 1 of every frame.

**Fix (implemented in S4_file_structure_fixed.md):**

```odin
// types/opcodes.odin
OpCode :: enum u8 {
    Request        = 0,
    Response       = 1,
    Signal         = 2,
    HelloRequest   = 3,
    HelloResponse  = 4,
    ByeRequest     = 5,
    ByeResponse    = 6,
    ByeSignal      = 7,
    WelcomeRequest = 8,
    WelcomeResponse = 9,
}
```

`Connect`, `Listen`, `Close`, `Drain`, `Channel_Closed` are not wire opcodes. They are internal reactor commands, conveyed via Mailbox messages between threads — not encoded into the BinaryHeader.

**Required before:** Any code is written.

---

### P-10 · CRITICAL · BinaryHeader Structure Changed — Wire Framing Broken (NEW)

**Location:** `internal/runtime/message.odin` (original S4) and `reactor/framer.odin`

**Violated rule:** RULE-1

**Problem:**

The original S4 defined:
```odin
Header :: struct {
    opcode:   u8,
    channel:  u16,
    id:       u32,
    meta_len: u16,
    body_len: u32,
}
```

The actual tofu BinaryHeader (`message.zig` lines 164–176) is:
```
channel_number: u16    (big-endian)
proto:          u8     (ProtoFields packed: opCode u4 + origin u1 + more u1 + _A u1 + _B u1)
status:         u8
message_id:     u64    (big-endian)
<thl>:          u16    (big-endian — text header length)
<bl>:           u16    (big-endian — body length)
Total: 16 bytes packed, no padding
```

Differences:
| Field | tofu wire | original S4 | Impact |
|-------|-----------|-------------|--------|
| `proto` (ProtoFields u8) | byte 2 | `opcode: u8` (just opcode, no flags) | opCode packing wrong; origin/more flags lost |
| `status` | byte 3 | absent | status byte lost from wire |
| `message_id` | u64 (8 bytes) | `id: u32` (4 bytes) | message ID truncated; frame desync |
| `<thl>` | u16 (text header len) | `meta_len: u16` | naming only — this one matches |
| `<bl>` | u16 (body len) | `body_len: u32` | body length field doubled in size; frame desync |
| Total | 16 bytes | 11 bytes | framer reads wrong number of bytes |

The framer reads exactly 16 bytes for the fixed header, then parses them. With the wrong struct, every frame boundary is calculated incorrectly. Partial reads and frame splits would corrupt all subsequent frames in the stream.

**Fix (implemented in S4_file_structure_fixed.md):**

```odin
// message/message.odin
FRAME_HEADER_SIZE :: 16

BinaryHeader :: struct #packed {
    channel_number: u16be,
    proto:          ProtoFields,
    status:         u8,
    message_id:     u64be,
    thl:            u16be,   // text header length in bytes
    bl:             u16be,   // body length in bytes
}

ProtoFields :: bit_field u8 {
    op_code:    OpCode : 4,
    origin:     OriginFlag : 1,
    more:       MoreMessagesFlag : 1,
    _internal_a: u8 : 1,
    _internal_b: u8 : 1,
}
```

**Required before:** Any code is written.

---

### P-11 · CRITICAL · TextHeaders Replaced by Raw Buffer — Wire Header Protocol Broken (NEW)

**Location:** `internal/runtime/message.odin` (original S4)

**Violated rule:** RULE-1

**Problem:**

The original S4 `Message` struct had:
```odin
Message :: struct {
    using poly: matryoshka.PolyNode,
    header:     Header,       // wrong BinaryHeader (see P-10)
    meta:       Appendable,   // raw byte buffer — NOT TextHeaders
    body:       Appendable,
}
```

The `meta` field is a plain byte buffer. tofu TextHeaders (`message.zig`) use a structured key-value encoding:

```
name: value\r\n
name: value\r\n
...
```

Each header is a line terminated by `\r\n`. Names and values are separated by `: `. Specific headers like `~connect_to` (ConnectToHeader) and `~listen_on` (ListenOnHeader) carry address information.

A raw `Appendable` has no awareness of this encoding. Code using `meta` directly would need to:
1. Know the format (undocumented)
2. Parse it manually (no helper procedures)
3. Construct it correctly (no builder procedures)

Result: otofu instances cannot construct or parse TextHeaders correctly. The handshake (`HelloRequest` carries `~connect_to`/`~listen_on` headers) will fail. Any message with metadata will fail.

**Fix (implemented in S4_file_structure_fixed.md):**

```odin
// message/message.odin
Message :: struct #align(align_of(matryoshka.PolyNode)) {
    using _node: matryoshka.PolyNode,   // offset 0 — required by Matryoshka C1
    bh:          BinaryHeader,
    thdrs:       TextHeaders,
    body:         Appendable,
}

TextHeaders :: struct {
    buffer: Appendable,
}
```

`TextHeaders` provides typed procedures for adding and reading headers in `"name: value\r\n"` format. Client code uses `Msg_Add_Text_Header`, `Msg_Get_Text_Header`, `Msg_Set_Connect_To`, `Msg_Get_Connect_To`, etc.

**Required before:** Any code is written.

---

### P-12 · SIGNIFICANT · UDS Support Removed — TCP-Only Address (NEW)

**Location:** `types/address.odin` (original S4) and `platform/socket_unix.odin`

**Violated rule:** RULE-4

**Problem:**

The original S4 defined:
```odin
Address :: struct {
    ip:      string,
    port:    u16,
    version: enum { IPv4, IPv6 },
}
```

This is TCP-only. tofu (`address.zig`) explicitly supports both TCP and UDS (Unix Domain Sockets):

```zig
pub const TCPClientAddress = struct { ... };
pub const TCPServerAddress = struct { ... };
pub const UDSClientAddress = struct { ... };
pub const UDSServerAddress = struct { ... };
```

Both TCP and UDS addresses are passed via TextHeaders (`ConnectToHeader = "~connect_to"`, `ListenOnHeader = "~listen_on"`):
- TCP format: `"tcp|127.0.0.1|7099"`
- UDS format: `"uds|/tmp/7099.port"`

With only TCP support, otofu cannot:
- Connect to a tofu peer listening on a UDS socket
- Accept connections from tofu clients using UDS
- Implement any UDS-only deployment (common in same-host service meshes)

**Fix (implemented in S4_file_structure_fixed.md):**

```odin
// types/address.odin
Address_Kind :: enum { TCP, UDS }

TCP_Address :: struct {
    host:    string,
    port:    u16,
    version: enum { IPv4, IPv6 },
}

UDS_Address :: struct {
    path: string,   // e.g. "/tmp/7099.port"
}

Address :: union {
    TCP_Address,
    UDS_Address,
}

ConnectToHeader :: "~connect_to"
ListenOnHeader  :: "~listen_on"
```

`platform/socket_unix.odin` implements `SOCK_STREAM AF_UNIX` alongside `SOCK_STREAM AF_INET/AF_INET6`.

**Required before:** Any integration test with UDS peers.

---

### P-13 · SIGNIFICANT · `updateReceiver` Removed from ChannelGroup — Public API Diverges (NEW)

**Location:** `otofu/channel_group.odin` (original S4)

**Violated rule:** RULE-2

**Problem:**

The original S4 `otofu/channel_group.odin` defined only two ChannelGroup procedures:

```odin
CG_Post(chnls: ^ChannelGroup, msg: ^?^message.Message) -> BinaryHeader
CG_Wait_Receive(chnls: ^ChannelGroup, timeout_ns: i64) -> ?^message.Message
```

tofu's `ChannelGroup` public API (`ampe.zig` line 99) has three methods:

```zig
pub fn post(self: *ChannelGroup, msg: *?*Message) BinaryHeader
pub fn waitReceive(self: *ChannelGroup, timeout_ns: i64) ?*Message
pub fn updateReceiver(self: *ChannelGroup, update: anytype) void
```

`updateReceiver` is used by the application to change the active receiver mid-session. Without it:
- Client applications that dynamically swap message receivers cannot port to otofu
- Cross-language clients expecting the full tofu API will not work
- The Odin API is a strict subset of the Zig API — a behavioral divergence

**Fix (implemented in S4_file_structure_fixed.md):**

```odin
// otofu/channel_group.odin
CG_Update_Receiver :: proc(chnls: ^ChannelGroup, update: CG_Receiver_Update)

CG_Receiver_Update :: struct {
    channel_number: types.ChannelNumber,   // 0 = all channels
    recv_proc:      CG_Recv_Proc,
    recv_ctx:       rawptr,
}
```

This is the Odin equivalent of `updateReceiver(update: anytype)`. The `anytype` in Zig accepts a struct with a `receive` method; in Odin, we use an explicit function pointer + context pointer pair.

**Required before:** Any application porting work.

---

### P-14 · SIGNIFICANT · `getAllocator` Removed from Engine — Public API Diverges (NEW)

**Location:** `otofu/engine.odin` (original S4)

**Violated rule:** RULE-2

**Problem:**

The original S4 `otofu/engine.odin` defined four Engine procedures:

```odin
Engine_Create(options: Engine_Options) -> (^Engine, Error)
Engine_Destroy(eng: ^Engine)
Engine_Get(eng: ^Engine, strategy: Allocation_Strategy) -> ?^message.Message
Engine_Put(eng: ^Engine, msg: ^message.Message)
```

tofu's `Ampe` public API (`ampe.zig` line 52) has five methods:

```zig
pub fn get(self: *Ampe, strategy: AllocationStrategy) ?*Message
pub fn put(self: *Ampe, msg: *Message) void
pub fn create(self: *Ampe) ChannelGroup
pub fn destroy(self: *Ampe, chnls: *ChannelGroup) void
pub fn getAllocator(self: *Ampe) Allocator
```

`getAllocator` is used by application code that needs the Engine's allocator for creating auxiliary buffers compatible with the Message pool. Without it:
- Applications cannot allocate auxiliary data using the same allocator as the Engine
- Code that constructs `Appendable` bodies using the Engine's backing allocator cannot port
- The Odin API is a strict subset of the Zig API

**Fix (implemented in S4_file_structure_fixed.md):**

```odin
// otofu/engine.odin
Engine_Get_Allocator :: proc(eng: ^Engine) -> mem.Allocator
```

`engine_internal` stores the `allocator: mem.Allocator` field (already present per P-01 fix). This procedure simply returns it.

**Required before:** Any application porting work.

---

### P-15 · MINOR · Framing Responsibility Conflicts with P7

*(formerly P-08 in original S5_validation.md)*

**Location:** `reactor/io_dispatch.odin` vs. P7 L4 specification

**Problem:**

P7 states:
> "L4 calls L3 (to obtain Messages from Reserved Pool for responses, **to frame outbound messages**)."

S4 assigns framing to L2 (`reactor/io_dispatch.odin`):
> "Called by `reactor/io_dispatch.odin` on WRITE (encode outbound) and READ (decode inbound) events."

In S4, `io_dispatch.odin` calls `Framer_Encode` and `Framer_Try_Decode` directly. L4 (`protocol`) does not call the framer.

This is a layering deviation from P7 but not a correctness bug. The current S4 placement is arguably cleaner: framing is a wire-format concern (L2 I/O boundary) rather than a protocol concern (L4 semantics). L4 works only with decoded Messages.

**Decision required:**

Either:
- Accept the deviation and update P7 to reflect that framing is L2, not L4. Document the rationale.
- OR move framing calls to protocol (L4) and have io_dispatch pass raw byte slices to protocol.

Recommendation: Accept the deviation. Framing at L2 (at the I/O boundary) is consistent with the single-responsibility principle — L4 handles semantics, L2 handles wire encoding.

**Action:** Update P7 L4 description to remove "to frame outbound messages." Update L2 description to include framing as an L2 responsibility.

---

### P-16 · MINOR · Protocol Direct-Send Bypasses Reactor Drain Accounting

*(formerly P-09 in original S5_validation.md)*

**Location:** `protocol/handshake.odin` — calls `runtime.Router_Send_App`

**Problem:**

`protocol/handshake.odin` calls `runtime.Router_Send_App` directly to notify the application of channel open/close events. This is L4 → L3 (permitted by P7). However, the Reactor (L2) is unaware of these sends.

During drain (Phase 9), the Reactor checks: "no open channels and inbox empty." But messages sent by the protocol layer are in the outbox Mailboxes, not tracked by the Reactor's drain check.

If the Reactor exits the drain phase before protocol-sent messages are delivered to the application, the application may miss notifications.

**Risk assessment:** Low. Matryoshka `mbox_send` is synchronous for the sender — the message is in the Mailbox queue immediately. The application can drain the Mailbox after Engine drain completes. The messages are not lost. The concern is timing, not correctness.

**Fix (if tightening drain semantics):**

Route all protocol notifications through a counter in `router_state`:
```odin
Router :: struct {
    ...
    in_flight: int,   // atomic; incremented by Router_Send_App; decremented on mbox_wait_receive
    ...
}
```

Phase 9 checks `router.in_flight == 0` as a drain-complete condition. This is optional for the baseline.

Alternatively, document explicitly: "Engine drain guarantees inbox empty and no open channels. It does not guarantee all protocol notifications have been consumed by the application. Application must drain its ChannelGroup Mailboxes after Engine_Destroy." This aligns with the Matryoshka drain model (V-MB4).

---

## Summary Table

| ID | Severity | Location | Issue | Fix Required Before Code |
|----|----------|----------|-------|--------------------------|
| P-01 | **CRITICAL** | `reactor/reactor.odin` | `engine_state: ^engine_internal` creates reactor→otofu circular import | Yes |
| P-02 | **CRITICAL** | `reactor/reactor.odin`, `protocol/protocol.odin` | `Dispatch_Context` defined in reactor but used in protocol procedure signatures; protocol does not import reactor | Yes |
| P-03 | **CRITICAL** | `otofu/engine.odin`, `reactor/reactor.odin` | `Reserved_Pool` declared in both `engine_internal` and `reactor_state`; dual ownership, dual pool | Yes |
| P-04 | **CRITICAL** | `reactor/reactor.odin` | No primary Channel collection in `reactor_state`; Reactor cannot enumerate channels for drain, deadline scan, or Phase 9 check | Yes |
| P-05 | **SIGNIFICANT** | `chanmgr/channel.odin` | `Channel.socket` described as non-owning but no owner declared; Socket allocation and freeing path undefined | Yes |
| P-06 | **SIGNIFICANT** | `chanmgr/channel.odin`, `reactor/timeout.odin` | `Channel.deadline_ms` and `Timeout_Manager.deadlines` both track the same datum; two sources of truth | Yes |
| P-07 | **SIGNIFICANT** | `chanmgr/channel.odin`, `reactor/reactor.odin` | No per-channel receive buffer; framer cannot accumulate partial frames across Reactor iterations | Yes |
| P-08 | **CRITICAL** | `internal/runtime/message.odin` | `Message` in internal package; clients cannot use `*Message` in API calls; Odin has no re-export; API is unusable | Yes |
| P-09 | **CRITICAL** | `types/opcodes.odin` | OpCode set entirely replaced with non-tofu values; every frame exchanged with a tofu peer is misinterpreted | Yes |
| P-10 | **CRITICAL** | `internal/runtime/message.odin`, `reactor/framer.odin` | BinaryHeader structure wrong: `status` absent, `message_id` u32 not u64, `body_len` u32 not u16, total 11 bytes not 16; frame desync | Yes |
| P-11 | **CRITICAL** | `internal/runtime/message.odin` | TextHeaders replaced by raw `meta: Appendable`; header encoding/decoding undefined; handshake headers (`~connect_to`) cannot be constructed | Yes |
| P-12 | **SIGNIFICANT** | `types/address.odin`, `platform/socket_unix.odin` | TCP-only `Address` struct; UDS peers cannot connect; `Address_Kind` union required | Yes |
| P-13 | **SIGNIFICANT** | `otofu/channel_group.odin` | `CG_Update_Receiver` absent; ChannelGroup API incomplete; cross-language clients break | Yes |
| P-14 | **SIGNIFICANT** | `otofu/engine.odin` | `Engine_Get_Allocator` absent; Engine API incomplete; application cannot access Engine's allocator | Yes |
| P-15 | MINOR | `reactor/io_dispatch.odin` | Framing assigned to L2; P7 assigns it to L4. Inconsistency in spec, not in S4 structure | No (update P7) |
| P-16 | MINOR | `protocol/handshake.odin` | Protocol direct-sends to Router bypass Reactor drain accounting; application may not have consumed all notifications at drain completion | No (document) |

---

## Corrected Struct Summaries

The following replacements resolve P-01 through P-07. These are structure-level corrections, not implementations.

---

### `reactor/reactor.odin` — `reactor_state` (corrected)

Resolves: P-01, P-03, P-04

```odin
reactor_state :: struct {
    // OS I/O
    poller:        platform.Poller,
    notifier:      platform.Notifier,

    // Matryoshka infrastructure (Reactor-owned only)
    tc_pool:       TC_Pool,
    reserved_pool: runtime.Reserved_Pool,   // Reactor-owned; removed from engine_internal

    // Routing
    router:        ^runtime.Router,

    // Tracking
    dual_map:      Dual_Map,
    timeout_mgr:   Timeout_Manager,
    number_pool:   chanmgr.Number_Pool,

    // Channel ownership (PRIMARY collection)
    channels: [dynamic]^chanmgr.Channel,    // ADDED; Reactor owns all Channel lifetime

    // Per-iteration scratch (zeroed/reset at start of each iteration)
    io_events:     [dynamic]platform.Event,
    resolved:      [dynamic]^TC,
    pending_close: [dynamic]^chanmgr.Channel,

    // Engine coordination (no pointer into otofu package)
    eng_state: ^types.Engine_State,         // CHANGED: types.Engine_State not engine_internal
    options:   types.Engine_Options,        // CHANGED: copied by value; Engine_Options moved to types/
    allocator: mem.Allocator,
}
```

---

### `reactor/reactor.odin` — `Dispatch_Context` (corrected)

Resolves: P-02

```odin
// Package-private (lowercase). Not exported. Never passed to protocol.
dispatch_context :: struct {
    router:        ^runtime.Router,
    reserved_pool: ^runtime.Reserved_Pool,
    channel:       ^chanmgr.Channel,
    engine_state:  types.Engine_State,
    allocator:     mem.Allocator,
}
```

`Dispatch_Context` (uppercase) is removed from the public package interface. `protocol.Protocol_Dispatch_Inbound` takes `^protocol.Protocol_Context`. The conversion is performed in `io_dispatch.odin`.

---

### `otofu/engine.odin` — `engine_internal` (corrected)

Resolves: P-01, P-03

```odin
engine_internal :: struct {
    message_pool:  runtime.Message_Pool,   // Engine-owned; application pool
    // reserved_pool removed — Reactor-owned only
    router:        runtime.Router,
    options:       types.Engine_Options,   // Engine_Options moved to types/
    allocator:     mem.Allocator,
    state:         types.Engine_State,     // Reactor holds ^state for signaling
}
```

---

### `chanmgr/channel.odin` — `Channel` (corrected)

Resolves: P-05, P-06, P-07

```odin
Channel :: struct {
    using poly:    matryoshka.PolyNode,
    number:        types.ChannelNumber,
    state:         types.Channel_State,
    cg_id:         types.ChannelGroupId,
    remote_number: types.ChannelNumber,
    socket:        platform.Socket,        // CHANGED: embedded by value; Channel owns
    outbound:      Outbound_Queue,
    recv_buf:      [dynamic]u8,            // ADDED: partial-frame accumulation buffer
    // deadline_ms removed — owned by Timeout_Manager only
}
```

---

### `message/message.odin` — `Message` (corrected)

Resolves: P-08, P-10, P-11

```odin
// Top-level public package: message/
Message :: struct #align(align_of(matryoshka.PolyNode)) {
    using _node: matryoshka.PolyNode,   // offset 0 — Matryoshka C1 (internal; not in public API docs)
    bh:          BinaryHeader,
    thdrs:       TextHeaders,
    body:         Appendable,
}

BinaryHeader :: struct #packed {
    channel_number: u16be,
    proto:          ProtoFields,
    status:         u8,
    message_id:     u64be,
    thl:            u16be,
    bl:             u16be,
}
#assert(size_of(BinaryHeader) == 16)

ProtoFields :: bit_field u8 {
    op_code:     OpCode : 4,
    origin:      OriginFlag : 1,
    more:        MoreMessagesFlag : 1,
    _internal_a: u8 : 1,
    _internal_b: u8 : 1,
}
```

---

## Required File Addition

### `types/options.odin` — NEW FILE

Resolves: P-01 (Engine_Options must be accessible by both `otofu` and `reactor` without a cycle)

**Purpose:** Engine configuration struct. Moved from `otofu/engine.odin` to `types/` so that both `otofu` and `reactor` can reference it without importing each other.

```odin
Engine_Options :: struct {
    max_messages:            int,
    reserved_messages:       int,
    max_channels:            int,
    outbound_queue_depth:    int,
    max_appendable_capacity: int,
    connect_timeout_ms:      i64,
    handshake_timeout_ms:    i64,
    bye_timeout_ms:          i64,
}
```

Updated import graph consequence: `reactor` already imports `types`. Moving `Engine_Options` to `types` requires no new import.

---

## Impact on S3 Package Layout

| Change | Affected file | Impact |
|--------|--------------|--------|
| `Engine_Options` moved to `types/` | New file `types/options.odin` | File tree updated; no import graph change |
| `engine_internal.reserved_pool` removed | `otofu/engine.odin` | Struct simplified |
| `reactor_state.engine_state` removed; `eng_state: ^types.Engine_State` added | `reactor/reactor.odin` | No import change (`types` already imported) |
| `reactor_state.channels` added | `reactor/reactor.odin` | No import change |
| `Dispatch_Context` made private | `reactor/reactor.odin` | No import change; just identifier case change |
| `Channel.socket` changed from `^platform.Socket` to `platform.Socket` | `chanmgr/channel.odin` | No import change |
| `Channel.deadline_ms` removed | `chanmgr/channel.odin` | Simplification |
| `Channel.recv_buf` added | `chanmgr/channel.odin` | No import change |
| `Message` moved from `internal/runtime/` to top-level `message/` | `message/message.odin` (new) | Import graph: all packages that use `Message` import `message` not `runtime` |
| `BinaryHeader` corrected to 16-byte packed struct | `message/message.odin` | All framer code uses correct field layout |
| `TextHeaders` restored as struct with `buffer: Appendable` | `message/message.odin` | Header encode/decode procedures use typed struct |
| `OpCode` corrected to 10 tofu wire values | `types/opcodes.odin` | All protocol code uses correct opcode values |
| `Address` changed to TCP + UDS union | `types/address.odin` | Address parsing and socket creation updated |
| `CG_Update_Receiver` added | `otofu/channel_group.odin` | Public API complete |
| `Engine_Get_Allocator` added | `otofu/engine.odin` | Public API complete |

No circular imports are introduced by any of these changes.
No new packages are required beyond:
- `message/` (new top-level public package)
- `types/options.odin` (new file in existing package)

---

## Validation Result

| Category | Status |
|----------|--------|
| Ownership violations | 3 found (P-03, P-05, P-06) — all fixable within existing packages |
| Module coupling issues | 1 found (P-06 dual tracking) — resolved by removing Channel.deadline_ms |
| Circular dependencies | 2 found (P-01, P-02) — both resolved by type relocation and context separation |
| Missing structures | 2 found (P-04, P-07) — resolved by additions to reactor_state and Channel |
| Wire protocol violations | 4 found (P-09, P-10, P-11, P-12) — all resolved in S4_file_structure_fixed.md |
| Public API violations | 3 found (P-08, P-13, P-14) — all resolved in S4_file_structure_fixed.md |
| Layering violations | 1 minor (P-15) — spec drift, not structural; resolved by updating P7 |
| Drain accounting gap | 1 minor (P-16) — acceptable for baseline; document the contract |

**All CRITICAL and SIGNIFICANT issues are resolvable within the corrected package structure.**

**New files required:**
- `message/message.odin` — new top-level public package (resolves P-08, P-10, P-11)
- `types/options.odin` — moves `Engine_Options` out of `otofu` into `types` (resolves P-01)

**S4_file_structure_fixed.md applies all corrections for P-08 through P-14.**
**S4_file_structure_fixed.md is the input to S6 (tofu mapping).**
