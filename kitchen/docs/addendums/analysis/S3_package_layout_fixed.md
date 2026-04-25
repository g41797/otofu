# S3 — Package Layout (Fixed)

Sources: S1_modules_fixed.md, S2_boundaries_fixed.md, P7_otofu_architecture.md
Pipeline position: S3 (folder/package structure — derived from S1 module catalog and S2 boundary rules)

## Corrections Applied

| Rule | Violation in original | Fix |
|------|-----------------------|-----|
| Structural | `api_message` (module 21) placed in `otofu/message.odin` (root package) | Moved to `message/` — separate top-level package with 6 files. `*Message` is the public API type; it cannot be in an `internal/` path. Client code imports `message` directly. |
| RULE-1 | `internal/runtime/message.odin` held the Message struct with wrong Header | Remove from runtime; Message moves to `message/` package (6 files) |
| RULE-1/4/6/7 | `types/address.odin` described as TCP-only, passive, client/server collapsed | Four active variants (TCP_Client, TCP_Server, UDS_Client, UDS_Server); full Address type + behavior consolidated in `message/address.odin`; no `types/address.odin` |
| RULE-8 | `ampe/IntrusiveQueue.zig` mapped as REMOVED | Ported as `message/queue.odin` — single-threaded Message_Queue; Matryoshka Mailbox is NOT a substitute |
| RULE-3 | `types/opcodes.odin` implied wrong 10 opcode values | Clarified: 10 original tofu wire protocol values |
| RULE-2 | Public API list missing `CG_Update_Receiver` and `Engine_Get_Allocator` | Added |
| Structural | Package import graph missing `message` package | Added |
| Structural | `runtime` package import graph didn't import `message` | Fixed: runtime imports message (framer, pools operate on *Message) |
| Structural | `chanmgr` didn't import `message` | Fixed: chanmgr imports message (Channel recv_buf, outbound queue use *Message) |
| Structural | `protocol` didn't import `message` | Fixed: protocol imports message (reads BinaryHeader fields) |

---

This document defines the canonical Odin package structure for otofu.
Each folder is one package. Each package enforces one layer boundary.
Visibility is controlled by Odin identifier case and by the `internal/` directory convention.

---

## Package Model

Odin packages are directories. All `.odin` files in a directory share one package namespace.
Visibility rules:
- **Uppercase identifier**: exported — importable by any package that imports this one
- **Lowercase identifier**: package-private — accessible only within the same directory

The `internal/` directory convention signals that packages beneath it are otofu-internal.
Odin does not enforce this at the compiler level. It is enforced by the import rules documented below and by code review.

---

## Full Folder Tree

```
otofu/
├── engine.odin                      (Engine lifecycle, Engine_Options, Engine_Get_Allocator)
├── channel_group.odin               (ChannelGroup handle API: Post/WaitReceive/UpdateReceiver/Connect/Listen)
│
├── message/                         (package message — public; application imports this directly)
│   ├── binary_header.odin           (ChannelNumber, MessageID, BinaryHeader packed struct, BH_To_Wire/BH_From_Wire)
│   ├── text_headers.odin            (TextHeaders, Text_Header, TextHeader_Iterator, TH_* procedures)
│   ├── message.odin                 (Message struct, Msg_* lifecycle + field procedures, MessageType/MessageRole/Trigger)
│   ├── queue.odin                   (Message_Queue, MQ_* procedures — single-threaded intrusive FIFO; port of IntrusiveQueue.zig)
│   ├── address.odin                 (Address type + all behavior: variants, Addr_Format/Parse/Is_Client/Is_Server, Addr_To/From_Connect/Listen_Header; port of address.zig)
│   └── helpers.odin                 (Ptr_To_Slice, Struct_To_Slice, Actual_Len, etc.)
│
├── types/                           (package types — shared definitions, no layer)
│   ├── opcodes.odin                 (OpCode enum — 10 wire protocol values; ProtoFields, OriginFlag, MoreMessagesFlag)
│   ├── identifiers.odin             (ChannelNumber, SequenceNumber, ChannelGroupId)
│   ├── flags.odin                   (TriggerFlags, AllocationStrategy, DecodeResult, ChannelState, TC_State)
│   └── status.odin                  (Ampe_Status enum u8 — 31 values; Status_From_Raw, Status_To_Raw; replaces Engine_Error)
│
└── internal/
    │
    ├── platform/                    (package platform — L1: OS I/O primitives)
    │   ├── poller.odin              (Poller type, interface constants, common logic)
    │   ├── poller_linux.odin        (epoll backend; //+build linux)
    │   ├── poller_darwin.odin       (kqueue backend; //+build darwin)
    │   ├── poller_windows.odin      (wepoll/AFD_POLL backend; //+build windows)
    │   ├── notifier.odin            (Notifier type, notify/drain interface)
    │   ├── notifier_unix.odin       (socketpair; //+build !windows)
    │   ├── notifier_windows.odin    (AF_UNIX or loopback TCP; //+build windows)
    │   ├── socket.odin              (Socket struct, all socket_* procedures — TCP and UDS, OS error mapping)
    │   ├── socket_unix.odin         (POSIX socket syscalls; //+build !windows)
    │   └── socket_windows.odin      (Winsock2 syscalls; //+build windows)
    │
    ├── chanmgr/                     (package chanmgr — L2 channel state)
    │   ├── channel.odin             (Channel struct, SM2 state machine, ch_mgr_transition)
    │   ├── outbound.odin            (per-Channel outbound send queue of *Message)
    │   └── numbers.odin             (ChannelNumber bitmap; assign/release)
    │
    ├── reactor/                     (package reactor — L2 event loop)
    │   ├── reactor.odin             (9-phase loop, startup, drain, Dispatch_Context)
    │   ├── dual_map.odin            (ChannelNumber ↔ SequenceNumber ↔ *TC)
    │   ├── tc_pool.odin             (TriggeredChannel struct, Pool, hooks)
    │   ├── io_dispatch.odin         (per-TC I/O handler: READ/WRITE/ERROR/HUP/ACCEPT)
    │   └── timeout.odin             (per-Channel deadline tracking)
    │
    ├── runtime/                     (package runtime — L3 messaging infrastructure)
    │   ├── message_pool.odin        (application Pool: on_get/on_put hooks; operates on message.Message)
    │   ├── reserved_pool.odin       (engine-internal Pool, fixed pre-allocated count)
    │   ├── router.odin              (Mailbox wiring: reactor_inbox, per-CG outboxes)
    │   └── framer.odin              (wire encode/decode: BinaryHeader + TextHeaders + Body)
    │
    └── protocol/                    (package protocol — L4 conversation layer)
        ├── protocol.odin            (OpCode dispatch, exhaustive switch, state directives)
        └── handshake.odin           (Hello/Bye/Welcome sequences, simultaneous-Bye tiebreaker)
```

Note: `internal/runtime/message.odin` is REMOVED compared to the original S3 design. The `Message` struct now lives in `message/message.odin` within the top-level `message/` package (6 files). The `runtime` package imports `message` to access `*Message` from its pool hooks and framer. `message/queue.odin` provides `Message_Queue` (single-threaded FIFO), used by `chanmgr/outbound.odin` and reactor scratch collections.

---

## Module → Package Mapping

Every module from S1 maps to exactly one package and one file.

| Module (S1) | Layer | Package | File |
|-------------|-------|---------|------|
| `matryoshka.poly` | L0 ext | matryoshka | (external) |
| `matryoshka.mailbox` | L0 ext | matryoshka | (external) |
| `matryoshka.pool` | L0 ext | matryoshka | (external) |
| `poller` | L1 | platform | poller.odin + poller_{linux,darwin,windows}.odin |
| `notifier` | L1 | platform | notifier.odin + notifier_{unix,windows}.odin |
| `socket` | L1 | platform | socket.odin + socket_{unix,windows}.odin |
| `reactor` | L2 | reactor | reactor.odin |
| `dual_map` | L2 | reactor | dual_map.odin |
| `tc_pool` | L2 | reactor | tc_pool.odin |
| `channel_manager` | L2 | chanmgr | channel.odin + outbound.odin + numbers.odin |
| `io_dispatch` | L2 | reactor | io_dispatch.odin |
| `timeout_manager` | L2 | reactor | timeout.odin |
| `message_pool` | L3 | runtime | message_pool.odin |
| `reserved_pool` | L3 | runtime | reserved_pool.odin |
| `mailbox_router` | L3 | runtime | router.odin |
| `framer` | L3 | runtime | framer.odin |
| `protocol` | L4 | protocol | protocol.odin |
| `handshake` | L4 | protocol | handshake.odin |
| `engine` | L5 | otofu | engine.odin |
| `channel_group` | L5 | otofu | channel_group.odin |
| `api_message` | L5 | **message** | **message/** (top-level package; 6 files: binary_header, text_headers, message, queue, address, helpers) |

**Why `channel_manager` has its own package (`chanmgr`) while other L2 modules share `reactor`:**

`channel_manager` is the one L2 module that L4 (`protocol`, `handshake`) calls directly (state transitions). If `channel_manager` were in the `reactor` package, then `protocol` importing `reactor` would gain access to all reactor internals (`dual_map_insert`, `tc_pool_get`, `reactor_start`, etc.). Separating `chanmgr` gives `protocol` a minimal import surface for the one L2 procedure it legitimately calls.

**Why `api_message` has its own top-level `message/` package:**

`Message` is the primary type exchanged at the public API boundary. `Ampe.get()` returns `*Message`, `ChannelGroup.post()` takes `*Message`. If `Message` lived in `internal/runtime`, application code would need to import an internal package to use the type — Odin has no re-export mechanism. The `message/` package is public (no `internal/` prefix), accessible to both application code and all otofu internal packages.

---

## Package Import Graph

Directed. No cycles. Arrows point downward (caller → callee).

```
otofu ──────────────────────────────────────────► reactor
  │                                                    │
  ├──► message ◄─────────────────────────── runtime ◄──┤
  │       │                                    │        │
  ├──────────────────────────────► runtime     │        │
  │                                    │       │        ├──► protocol ──► chanmgr
  │       └──────────────────────────► types   │        │                    │
  │                                            │        │                 message
  │                                         matryoshka  │
  │                                                      ├──► platform ──► types
  │                                                      │
  │                                                      └──► types
  │
  ├──► types
  └──► matryoshka (for MayItem at API boundary)
```

Tabular form — each package and what it imports:

| Package | Imports |
|---------|---------|
| `types` | (nothing — pure type definitions) |
| `message` | `types`, `matryoshka` (PolyNode at offset 0) |
| `platform` | `types`, OS syscall packages |
| `chanmgr` | `types`, `message` (Channel.outbound, recv_buf use *Message), `platform` (Socket type ref), `matryoshka` (PolyNode in Channel) |
| `runtime` | `types`, `message` (Message struct for pool/framer), `matryoshka` |
| `reactor` | `types`, `platform`, `chanmgr`, `runtime`, `protocol`, `matryoshka` |
| `protocol` | `types`, `chanmgr`, `runtime`, `message` |
| `otofu` | `types`, `reactor`, `runtime`, `message`, `matryoshka` |

**Application code imports:**
- `otofu` — the only otofu root package application code must import for engine/cg operations
- `message` — imported by application code for `*Message` type and message field accessors
- `otofu/types` — shared type definitions (Engine_Error, OpCode, etc.)

Application code must NOT import any package under `otofu/internal/`.

**Why no circular imports:**
- `chanmgr` imports `platform` for Socket type reference. `platform` does not import `chanmgr`. No cycle.
- `chanmgr` imports `message` for *Message. `message` does not import `chanmgr`. No cycle.
- `reactor` imports `protocol`. `protocol` imports `chanmgr`. `chanmgr` does not import `reactor`. No cycle.
- `runtime` imports `message`. `message` does not import `runtime`. No cycle.
- `protocol` imports `message`. `message` does not import `protocol`. No cycle.

---

## Visibility Rules

### Three Visibility Levels

| Level | Rule | Scope |
|-------|------|-------|
| **public** | Uppercase identifier in `otofu/` or `message/` package | Any code that imports otofu or message |
| **internal** | Uppercase identifier in `otofu/internal/*` package | Any otofu package that imports the internal package; NOT application code |
| **private** | Lowercase identifier in any package | Same directory (same package) only |

### Public Identifiers (exported from `otofu` + `message` packages)

These are the only names an application should need.

From `engine.odin`:
```
Engine                 (handle type — opaque to application)
Engine_Options         (configuration struct)
Engine_Create          (proc)
Engine_Destroy         (proc)
Engine_Get_Allocator   (proc — returns the allocator provided at creation)
```

From `channel_group.odin`:
```
CG                     (ChannelGroup handle type — opaque)
CG_Create              (proc)
CG_Destroy             (proc)
CG_Post                (proc)
CG_Wait_Receive        (proc)
CG_Update_Receiver     (proc — inject message from any thread; wake receiver)
CG_Connect             (proc — encodes HelloRequest with connect_to TextHeader; calls CG_Post)
CG_Listen              (proc — encodes WelcomeRequest with listen_on TextHeader; calls CG_Post)
```

From `message/message.odin`:
```
Message                (struct — the primary message type; application holds *Message)
BinaryHeader           (struct — 16-byte packed wire header)
ProtoFields            (packed struct — opcode + flags in 1 byte)
TextHeaders            (struct — key-value pairs in "name: value\r\n" format)
TextHeader             (struct — single name+value pair)
TextHeaderIterator     (struct — stateful iterator)
Engine_Get             (proc)
Engine_Put             (proc)
Msg_Set_Opcode         (proc)
Msg_Set_Channel        (proc)
Msg_Set_Message_Id     (proc)
Msg_Set_Status         (proc)
Msg_Set_More           (proc)
Msg_Read_Opcode        (proc)
Msg_Read_Channel       (proc)
Msg_Read_Message_Id    (proc)
Msg_Read_Status        (proc)
Msg_Read_Origin        (proc)
Msg_Read_More          (proc)
Msg_Write_Thdr         (proc — append TextHeader "name: value\r\n")
Msg_Read_Thdr          (proc — find TextHeader value by name)
Msg_Thdr_Iterator      (proc — TextHeaderIterator over message headers)
Msg_Write_Body         (proc)
Msg_Body_Slice         (proc)
```

From `types/` (also public — application imports `otofu/types`):
```
Engine_Error           (enum — 13 values)
OpCode                 (enum — 10 wire protocol values)
ProtoFields            (packed u8 — opCode, origin, more flags)
OriginFlag             (u1 enum — application / engine)
MoreMessagesFlag       (u1 enum — last / more)
ChannelNumber          (distinct type)
ChannelGroupId         (distinct type)
SequenceNumber         (distinct type)
Address                (struct — TCP or UDS)
AllocationStrategy     (enum)
RecvResult             (type alias / enum)
```

### Internal Identifiers (exported from `otofu/internal/*` packages)

Uppercase = exported = callable by any otofu internal package that imports the package.
These are NOT meant for application code.

**`platform` package (callable from `reactor` only):**
```
Poller_Register    Poller_Deregister    Poller_Wait    Poller_Close
Notifier_Create    Notifier_Notify      Notifier_Drain  Notifier_Close
Socket_Create      Socket_Set_Nonblocking  Socket_Connect  Socket_Connect_Complete
Socket_Listen      Socket_Accept        Socket_Send     Socket_Recv
Socket_Set_Linger  Socket_Close         Socket_Create_UDS  (for UDS creation)
Poller             (type)
Notifier           (type)
Socket             (type)
```

**`chanmgr` package (callable from `reactor` and `protocol`):**
```
Channel            (type — struct)
Channel_State      (type — enum; subset of types.ChannelState)
Ch_Allocate        Ch_Assign_Number     Ch_Release_Number
Ch_Transition      Ch_Free
Ch_Enqueue_Outbound  Ch_Dequeue_Outbound  Ch_Outbound_Empty
Ch_Set_Socket      Ch_Set_TC            Ch_Set_Remote_Number
```

**`runtime` package (callable from `reactor`, `protocol`, `otofu`):**
```
Message_Pool       (type)
Reserved_Pool      (type)
Router             (type)
MP_Create          MP_Get          MP_Put          MP_Close
RP_Create          RP_Get          RP_Put          RP_Close
Router_Create      Router_Send_Reactor   Router_Drain_Inbox
Router_Send_App    Router_Wait_App       Router_Register_CG
Router_Drain_Outbox
Framer_Encode      Framer_Try_Decode
Decode_Result      (type)
```

Note: `Message` type is NOT re-exported from `runtime`. It is imported from the `message` package.

**`reactor` package (callable from `otofu.engine` only):**
```
Reactor_Start      (proc — spawns Reactor thread; called from engine_create)
```

All other identifiers in `reactor` are either:
- Lowercase (private): `dual_map_insert`, `tc_pool_get`, `io_dispatch_call`, `phase1_compute_timeout`, etc.
- Uppercase but documented as reactor-internal: `Dispatch_Context` (type used within reactor package only)

**`protocol` package (called by `reactor` only):**
```
Protocol_Dispatch_Inbound   (proc — called from io_dispatch)
Protocol_Dispatch_Command   (proc — called from reactor Phase 5)
Protocol_Context            (type)
```

### Private Identifiers (lowercase — package-scope only)

Examples of identifiers that must be lowercase (not exported):
```
reactor_state          (struct — internal Reactor loop state)
dispatch_context       (struct — passed within reactor package only; lowercase to enforce privacy)
engine_internal        (struct — backing store for Engine handle)
cg_internal            (struct — backing store for CG handle)
dual_map_insert        (proc)
tc_pool_get            (proc)
io_dispatch_call       (proc)
```
