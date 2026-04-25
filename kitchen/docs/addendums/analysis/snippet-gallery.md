# otofu Snippet Gallery

This gallery contains refactored Odin code snippets adapted from legacy porting notes. These snippets are strictly compliant with the **otofu Architecture (P7)** and **Implementation Plan (B0)**.

---

### Phase 0 / Step 0.2: OpCode Enum
*Adapted from: odin-message.md*

```odin
package types

OpCode :: enum u4 {
    Connect        = 0,
    Listen         = 1,
    Close          = 2,
    Drain          = 3,
    Hello          = 4,
    Welcome        = 5,
    Bye            = 6,
    Bye_Ack        = 7,
    Data           = 8,
    Channel_Closed = 9,
}
```

---

### Phase 1 / Step 1.1: Core Message Struct
*Adapted from: odin-message.md / zig-to-odin.md*

```odin
package runtime

import "matryoshka"
import "../types"

MESSAGE_ID :: 1

Header :: struct #packed {
    opcode:   types.OpCode,
    channel:  types.ChannelNumber,
    cg_id:    types.ChannelGroupId,
    meta_len: u16,
    body_len: u32,
}

Message :: struct {
    using poly: matryoshka.PolyNode, // MANDATORY: Offset 0 for Matryoshka integration
    header:     Header,
    meta:       Appendable,
    body:       Appendable,
}

// Verification Invariant
#assert(offset_of(Message, poly) == 0)
```

---

### Phase 2 / Step 3.1: Appendable Buffer (with MR-1/MR-4)
*Adapted from: zig-to-odin.md*

```odin
package runtime

import "core:mem"

Appendable :: struct {
    buf: [dynamic]u8,
    len: int,
}

// MR-1: Explicit Allocator Discipline
appendable_init :: proc(alc: mem.Allocator, cap: int = 0) -> Appendable {
    return Appendable{
        buf = make([dynamic]u8, 0, cap, alc),
        len = 0,
    }
}

// MR-4: Capacity Capping in Pool Hooks
appendable_reset_and_cap :: proc(a: ^Appendable, max_cap: int) {
    a.len = 0
    clear(&a.buf)
    
    if cap(a.buf) > max_cap {
        // Free oversized buffer to prevent memory bloating
        delete(a.buf)
        a.buf = make([dynamic]u8, 0, 0, a.buf.allocator)
    }
}
```

---

### Phase 6 / Step 6.2: Mailbox Router Snippet
*Adapted from: zig-to-odin.md*

```odin
package runtime

import "matryoshka"
import "core:sync"

// otofu uses Matryoshka MayItem as the transfer unit
Router :: struct {
    reactor_inbox: matryoshka.Mailbox,
    wake_fn:       proc(),
}

// LI-8: Strict Send-then-Notify ordering
router_send_reactor :: proc(r: ^Router, m: ^matryoshka.MayItem) -> matryoshka.SendResult {
    res := matryoshka.mbox_send(&r.reactor_inbox, m)
    if res == .Ok {
        r.wake_fn() // Only wake if ownership actually transferred
    }
    return res
}
```

---

### Phase 8 / Step 8.2: Handshake (Simultaneous-Bye Tiebreaker)
*Adapted from: shutdown_choreography_example.md*

```odin
package protocol

import "../types"
import "../chanmgr"

// R2.1: Simultaneous-Bye tiebreaker
// Lower channel number is the Responder (sends Bye_Ack immediately)
handle_simultaneous_bye :: proc(ch: ^chanmgr.Channel, ctx: ^Protocol_Context) {
    if ch.number < ch.remote_number {
        // We are Responder
        send_bye_ack(ch, ctx)
        chanmgr.ch_transition(ch, .Closed)
    } else {
        // We are Initiator
        // Stay in .Closing state and wait for peer's Bye_Ack
    }
}
```

---

### Phase 11 / Step 11.1: Engine VTable Interface
*Adapted from: odin-interfaces.md*

```odin
package otofu

import "core:mem"
import "matryoshka"
import "../types"

// PM-2: Public API via VTable
Engine_VTable :: struct {
    get: proc(e: rawptr, strat: types.Allocation_Strategy, m: ^matryoshka.MayItem) -> types.Engine_Error,
    put: proc(e: rawptr, m: ^matryoshka.MayItem),
}

Engine :: struct {
    impl:   rawptr,
    vtable: ^Engine_VTable,
}

engine_get :: proc(e: Engine, strat: types.Allocation_Strategy, m: ^matryoshka.MayItem) -> types.Engine_Error {
    return e.vtable.get(e.impl, strat, m)
}
```
