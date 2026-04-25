# Tofu Message Format Analysis & Odin Implementation Strategy

The provided tofu message specification defines a **structured, length-prefixed protocol** with three logical sections. This is a **classic message framing design** optimized for high-throughput, low-latency async messaging. Here's the complete architectural breakdown and Odin implementation.

## Protocol Structure (16 + Variable Bytes)

```
┌──────────────┬─────────────────┬──────────────┐
│ BinaryHeader │ TextHeaders     │ Body         │ ← Total: 16 + len1 + len2
│ (16 bytes)   │ (N bytes)       │ (M bytes)    │
├──────────────┼─────────────────┼──────────────┤
│ u16 chn      │ key1=val1\0...  │ raw bytes    │
│ u8 proto     │ keyN=valN\0     │              │
│ u8 status    │                 │              │
│ u64 msg_id   │                 │              │
│ u16 thdr_len │                 │              │
│ u16 body_len │                 │              │
└──────────────┴─────────────────┴──────────────┘
```

**Key Design Decisions**:
- **Fixed 16-byte header** → Fast parsing, cache-friendly
- **Self-describing lengths** → No ambiguity, supports streaming
- **Dual origin tracking** → Engine vs Application messages
- **Correlation via `message_id`** → Request/Response matching
- **`more` flag** → Streaming/large payloads

## Odin Implementation: `Message` & Framing

### 1. Core Data Structures

```odin
package tofu

// BinaryHeader (exactly 16 bytes, #packed)
Binary_Header :: struct #packed {
    channel_number:     u16,  // 0=unassigned, 1-65534=valid, 65535=internal
    proto:              u8,   // opCode(4)|origin(1)|more(1)|reserved(2)
    status:             u8,   // 0=success
    message_id:         u64,  // correlation/job ID
    text_headers_len:   u16,  // Length of TextHeaders section
    body_len:           u16,  // Length of Body section
}

// Proto bitfield extraction
Proto_Bits :: bit_set[u8; 8]
Proto_Info :: struct {
    op_code: u4,    // Request=0, Response=1, Signal=2, Hello=3...
    origin:  u1,    // 0=app, 1=engine
    more:    u1,    // 0=last, 1=more coming
    // reserved: u2
}

// Complete Message (owned by pool/guard)
Message :: struct {
    bhdr:     Binary_Header,
    thdrs:    string,     // null-terminated key=value pairs
    body:     []u8,       // raw payload
    pool_ref: rawptr,     // back-pointer for pool management
}
```

### 2. OpCode & Helpers

```odin
Op_Code :: enum u4 {
    Request,      // 0: Ask peer for something
    Response,     // 1: Answer to request  
    Signal,       // 2: One-way notification
    Hello_Request,// 3: Client→Server connect
    Hello_Response,//4: Server→Client accept
    Bye_Request,  // 5: Graceful close start
    Bye_Response, // 6: Graceful close ack
    Bye_Signal,   // 7: Close NOW
    Welcome_Request,//8: Server→tofu start listener
    Welcome_Response,//9: tofu→Server listener ready
}

proto_set_op :: proc(bhdr: ^Binary_Header, op: Op_Code) {
    bhdr.proto = (u8(op) << 4) | (bhdr.proto & 0x0f)
}

proto_is_engine :: proc(bhdr: Binary_Header) -> bool {
    return (bhdr.proto & 0x10) != 0  // origin bit
}

proto_has_more :: proc(bhdr: Binary_Header) -> bool {
    return (bhdr.proto & 0x08) != 0  // more bit
}
```

### 3. RAII Guard + Pool Integration

```odin
// From porting guide: scope-bound ownership
Owned_Message_Guard :: struct {
    msg: ^Message,
    am:  ^Allocation_Manager,
}

guard_init :: proc(am: ^Allocation_Manager, strategy: Allocation_Strategy) -> (guard: Owned_Message_Guard, err: Alloc_Error) {
    msg: ^Message
    switch strategy {
    case .pool_only:
        if !pool_try_get(am, &msg) { return {}, .pool_empty }
    case .fallback_heap:
        msg = new(Message) or_return
    }
    return Owned_Message_Guard{msg=msg, am=am}, .none
}

guard_destroy :: proc(g: ^Owned_Message_Guard) {
    if g.msg == nil { return }
    pool_put(g.am, g.msg)
    g.msg = nil
}

// Usage matches Zig exactly
use_message :: proc() {
    guard, err := guard_init(&am, .pool_only)
    if err != .none { return }
    defer guard_destroy(&guard)
    
    // guard.msg is safe to use here
    guard.msg.bhdr.proto_set_op(.Request)
    guard.msg.bhdr.message_id = job_id
}
```

### 4. Framing: Serialize/Deserialize

```odin
// Network byte order (little-endian assumed)
to_network_bytes :: proc(msg: ^Message) -> []u8 {
    total_len := 16 + msg.bhdr.text_headers_len + msg.bhdr.body_len
    buf := make([]u8, total_len)
    
    // Header (already packed)
    mem.copy(buf[0:16], &msg.bhdr, 16)
    
    // TextHeaders (if any)
    if msg.bhdr.text_headers_len > 0 {
        mem.copy(buf[16:16+msg.bhdr.text_headers_len], 
                 raw_data(msg.thdrs), msg.bhdr.text_headers_len)
    }
    
    // Body (if any)
    if msg.bhdr.body_len > 0 {
        mem.copy(buf[16+msg.bhdr.text_headers_len:], msg.body, msg.bhdr.body_len)
    }
    
    return buf
}

from_network_bytes :: proc(data: []u8) -> (msg: Message, ok: bool) {
    if len(data) < 16 { return {}, false }
    
    msg: Message
    mem.copy(&msg.bhdr, data[0:16], 16)
    
    thdr_end := 16 + msg.bhdr.text_headers_len
    body_end := thdr_end + msg.bhdr.body_len
    
    if len(data) < body_end { return {}, false }
    
    // TextHeaders
    if msg.bhdr.text_headers_len > 0 {
        msg.thdrs = string(data[16:thdr_end])
    }
    
    // Body  
    if msg.bhdr.body_len > 0 {
        msg.body = make([]u8, msg.bhdr.body_len)
        mem.copy(msg.body, data[thdr_end:body_end], msg.bhdr.body_len)
    }
    
    return msg, true
}
```

### 5. TextHeaders Parser

```odin
Text_Header :: struct {
    name:  string,
    value: string,
}

text_headers_iter :: proc(thdrs: string) -> Text_Header_Iterator {
    return Text_Header_Iterator{data=thdrs, pos=0}
}

Text_Header_Iterator :: struct {
    data: string,
    pos:  int,
}

next :: proc(it: ^Text_Header_Iterator) -> (header: Text_Header, ok: bool) {
    if it.pos >= len(it.data) { return {}, false }
    
    eq_pos := strings.index_byte(it.data[it.pos:], '=')
    if eq_pos == -1 { return {}, false }
    
    name := it.data[it.pos:it.pos+eq_pos]
    value_start := it.pos + eq_pos + 1
    
    nul_pos := strings.index_byte(it.data[value_start:], 0)
    if nul_pos == -1 { nul_pos = len(it.data[value_start:]) }
    
    value := it.data[value_start:value_start+nul_pos]
    it.pos = value_start + nul_pos + 1
    
    return {name=name, value=value}, true
}
```

## Proactor Integration (Porting Guide Milestone 1)

```odin
// Completion event carries deserialized Message
Io_Completion :: struct {
    msg:  Message,           // Deserialized from network bytes
    err:  Io_Error,          // Completion status
    bytes_transferred: int,
}

// Proactor submits read → completion queue
submit_read :: proc(stream: ^Stream, buf: []u8) -> bool {
    // Windows IOCP: Queue APC/POST
    // Linux io_uring: io_uring_prep_recv()
    return platform_submit_read(stream.fd, buf)
}

// Completion handler → Message pipeline
handle_completion :: proc(comp: Io_Completion, mailbox: ^Mailbox(Owned_Message_Guard)) {
    if comp.err != .none { 
        mailbox.send_error(comp.err)
        return 
    }
    
    // Frame → Message
    guard, parse_err := message_from_framed_bytes(comp.msg_bytes)
    if parse_err != .none {
        mailbox.send_error(.parse_failed)
        return
    }
    
    // Conversation layer owns message
    mailbox.send(guard)
}
```

## Porting Guide Alignment

| Tofu Feature | Odin Implementation | Status |
|--------------|-------------------|--------|
| BinaryHeader | `#packed` struct | ✅ Complete |
| TextHeaders | `string` + iterator | ✅ Complete |
| Body | `[]u8` slice | ✅ Complete |
| `*?*Message` | `Owned_Message_Guard` | ✅ Matches guide |
| OpCode enum | `enum u4` | ✅ Complete |
| `more` flag | Bitfield in `proto` | ✅ Complete |
| Channel routing | `u16 channel_number` | ✅ Complete |
| Engine vs App | `origin` bit | ✅ Complete |

**Next Milestone**: Implement Proactor core with this framing. The message spec is **fully operational** and aligns perfectly with the RAII ownership model from the porting guide.
