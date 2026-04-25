# Porting Zig Idioms to Odin: A Pattern Guide

This document outlines common patterns and best practices for translating core Zig idioms and concepts into their Odin equivalents, specifically for the `tofu` project.

## 1. Tagged Unions

-   **Zig Concept**: Tagged unions (`union(enum) { Foo: FooType, Bar: BarType }`) allow a single type to hold values of different types, discriminated by a tag.
-   **Odin Idiom**: Odin uses raw unions with explicit tag fields for maximum control. For runtime discrimination, pair unions with `typeid` or enum tags. `core:reflect` provides type introspection.

    **Example:**
    ```odin
    package tofu

    import "core:fmt"
    import "core:reflect"

    Message_Payload :: union($T: typeid) {
        Hello: Hello,
        Data:  Data,
    }

    Hello :: struct {
        name:    string,
        version: u32,
    }

    Data :: struct {
        content:  []u8,
        sequence: u64,
    }

    Message :: struct {
        payload: Message_Payload,
    }

    process_payload :: proc(payload: Message_Payload) {
        // Method 1: typeid discrimination (runtime)
        id := reflect.union_variant_index(payload.id)
        switch id {
        case 0: hello := cast(^Hello)reflect.union_variant_payload(&payload, 0); 
                fmt.printf("Hello from %s v%d\n", hello.name, hello.version)
        case 1: data := cast(^Data)reflect.union_variant_payload(&payload, 1);
                fmt.printf("Data seq=%d len=%d\n", data.sequence, len(data.content))
        }
    }

    // Usage
    main :: proc() {
        msg1 := Message{ payload = Message_Payload{Hello{"Alice", 1}} }
        msg2 := Message{ payload = Message_Payload{Data{{}, 123}} }
        process_payload(msg1.payload)
        process_payload(msg2.payload)
    }
    ```

## 2. Errors and Error Handling

-   **Zig Concept**: Error sets (`error{Foo, Bar}`) and error unions (`!u8`) for explicit error propagation.
-   **Odin Idiom**: Tagged unions with `or_return` for propagation. Context-aware error handling via `or_return_if_non_context`.

    **Example:**
    ```odin
    package tofu

    import "core:fmt"

    Tofu_Error :: union {
        Pool_Empty,
        Invalid_Channel,
        None,
    }

    get_message_from_pool :: proc(pool_status: string) -> string {
        switch pool_status {
        case "empty":
            return "Pool_Empty"
        case "invalid_channel":
            return "Invalid_Channel"
        }
        return "success_message"
    }

    process_message :: proc() {
        result := get_message_from_pool("empty")
        if result != "success_message" {
            switch result {
            case "Pool_Empty":    fmt.println("Pool exhausted")
            case "Invalid_Channel": fmt.println("Bad channel")
            }
            return
        }
        fmt.println("Got message:", result)
    }
    ```

## 3. Interfaces (Duck Typing)

-   **Zig Concept**: `anytype` + comptime interface checks or vtable structs.
-   **Odin Idiom**: Parametric polymorphism with `where` clauses for static duck typing, or explicit vtables. Procedure groups for simple overload sets.

    **Example (Allocator interface):**
    ```odin
    package tofu

    import "core:fmt"

    Allocator :: struct($T: typeid) {
        data: T,
    }

    // Duck typing via parapoly + where clause
    allocate :: proc($A: Allocator, size: int) -> ^u8
    where alloc_proc: proc(a: ^A, size: int) -> ^u8 {
        return alloc_proc(&a.data, size)
    }

    free :: proc($A: Allocator, ptr: ^u8)
    where free_proc: proc(a: ^A, ptr: ^u8) {
        free_proc(&a.data, ptr)
    }

    // Concrete implementation
    Pool_Allocator :: struct {
        pool: u8,
        used: int,
    }

    pool_alloc :: proc(a: ^Pool_Allocator, size: int) -> ^u8 {
        if a.used + size > len(a.pool) {
            return nil
        }
        ptr := &a.pool[a.used]
        a.used += size
        return ptr
    }

    pool_free :: proc(a: ^Pool_Allocator, ptr: ^u8) {
        // Simplified - real pool would track allocations
        fmt.println("Freed from pool")
    }

    // Usage
    main :: proc() {
        pool: Pool_Allocator
        alloc := Pool_Allocator{pool}
        
        ptr := allocate(alloc, 16) or_else nil
        if ptr != nil {
            free(alloc, ptr)
        }
    }
    ```

## 4. Structs

-   **Zig Concept**: `struct` for aggregate data types.
-   **Odin Idiom**: Identical `struct` syntax. Use `#packed` for bitfields, custom stringers via `Stringable` interface pattern.

    **Example (BinaryHeader):**
    ```odin
    package tofu

    import "core:fmt"
    import "core:mem"

    Binary_Header :: struct #packed {
        channel_number:      u16,  // 0-1
        protocol_version:    u8,   // 2
        status_code:         u8,   // 3
        message_id:          u64,  // 4-11
        text_headers_length: u16,  // 12-13
        body_length:         u16,  // 14-15
    }

    to_bytes :: proc(h: Binary_Header) -> u8 { [ziggit](https://ziggit.dev/t/video-about-zig-interfaces/10156)
        buf: u8 [ziggit](https://ziggit.dev/t/video-about-zig-interfaces/10156)
        mem.copy(&buf, &h, size_of(Binary_Header))
        return buf
    }

    from_bytes :: proc(data: u8) -> Binary_Header { [ziggit](https://ziggit.dev/t/video-about-zig-interfaces/10156)
        h: Binary_Header
        mem.copy(&h, &data, size_of(Binary_Header))
        return h
    }

    // Usage
    main :: proc() {
        header := Binary_Header{
            channel_number = 1,
            protocol_version = 1,
            message_id = 12345,
            text_headers_length = 50,
            body_length = 200,
        }
        
        bytes := to_bytes(header)
        fmt.printf("Header bytes: %x\n", bytes)
        
        decoded := from_bytes(bytes)
        fmt.printf("Decoded: %+v\n", decoded)
    }
    ```

## 5. Dynamic Byte Buffers (Appendable)

-   **Zig Concept**: `Appendable` - growable byte buffer with explicit capacity management.
-   **Odin Idiom**: `core:dynamic_array` or custom `[]u8` with manual growth. Context allocator integration.

    **Example (AppendableBuffer):**
    ```odin
    package tofu

    import "core:fmt"
    import "core:mem"

    Appendable_Buffer :: struct {
        data: []u8,
        len:  int,
    }

    appendable_init :: proc(allocator := context.allocator, initial_size: int = 0) -> Appendable_Buffer {
        buf: Appendable_Buffer
        if initial_size > 0 {
            buf.data = make([]u8, initial_size)
        }
        return buf
    }

    appendable_free :: proc(buf: ^Appendable_Buffer) {
        if buf.data != nil {
            free(buf.data)
        }
    }

    append :: proc(buf: ^Appendable_Buffer, data: []u8, allocator := context.allocator) {
        required := buf.len + len(data)
        if required > len(buf.data) {
            // Double capacity or fit required
            new_cap := max(required, len(buf.data)*2 or 16)
            new_data := make([]u8, new_cap)
            if buf.len > 0 {
                mem.copy(new_data[:buf.len], buf.data[:buf.len])
            }
            free(buf.data)
            buf.data = new_data
        }
        mem.copy(buf.data[buf.len:], data)
        buf.len += len(data)
    }

    appendable_body :: proc(buf: Appendable_Buffer) -> []u8 {
        return buf.data[:buf.len]
    }

    // Usage
    main :: proc() {
        arena: mem.Arena
        defer mem.arena_free_all(&arena)
        context.allocator = mem.arena_allocator(&arena)

        buf := appendable_init(10)
        defer appendable_free(&buf)

        append(&buf, "Hello")
        append(&buf, " World!")
        fmt.println(string(appendable_body(buf)))
    }
    ```

## 6. Thread-Safe Blocking Queues (Mailbox)

-   **Zig Concept**: `MailBox` - MPSC queue with interrupt/close semantics.
-   **Odin Idiom**: `core:sync` primitives (`Mutex`, `Cond`) + `dynamic_array`. Context-aware allocation.

    **Example (Mailbox):**
    ```odin
    package tofu

    import "core:fmt"
    import "core:sync"
    import "core:time"

    Mailbox_Error :: enum {
        Closed,
        Interrupted,
        Timeout,
    }

    Mailbox :: struct($T: typeid) {
        queue: sync.Blocking_Queue(T),
        mutex: sync.Mutex,
        cond:  sync.Cond,
        closed: bool,
        interrupted: bool,
    }

    mailbox_init :: proc($T: typeid) -> Mailbox(T) {
        return Mailbox(T){
            queue = sync.blocking_queue_init(T, 1024),
            mutex = {},
            cond  = {},
            closed = false,
            interrupted = false,
        }
    }

    mailbox_send :: proc($T: typeid, mb: ^Mailbox(T), letter: T) {
        sync.mutex_lock(&mb.mutex)
        defer sync.mutex_unlock(&mb.mutex)

        if mb.closed {
            return
        }
        sync.blocking_queue_push(&mb.queue, letter)
        sync.cond_signal(&mb.cond)
    }

    mailbox_receive :: proc($T: typeid, mb: ^Mailbox(T), timeout_ns: i64) -> (T, Mailbox_Error) {
        sync.mutex_lock(&mb.mutex)
        defer sync.mutex_unlock(&mb.mutex)

        start := time.now()
        for !sync.blocking_queue_pop(&mb.queue, nil) {
            if mb.closed {
                return {}, .Closed
            }
            if mb.interrupted {
                mb.interrupted = false
                return {}, .Interrupted
            }

            if timeout_ns > 0 {
                elapsed := time.diff_ns(start, time.now())
                if elapsed >= timeout_ns {
                    return {}, .Timeout
                }
                // Wait with timeout using cond.timed_wait
            } else {
                sync.cond_wait(&mb.cond, &mb.mutex)
            }
        }

        letter: T
        sync.blocking_queue_pop(&mb.queue, &letter)
        return letter, .None
    }
    ```

## 7. Allocators

-   **Zig Concept**: Explicit `Allocator` interface with `create/free`.
-   **Odin Idiom**: Context allocators (`context.allocator`) + `core:mem` helpers (`new`, `make`, `delete`, `free`).

    **Example:**
    ```odin
    package tofu

    import "core:fmt"
    import "core:mem"

    main :: proc() {
        // Arena allocator (Zig: GeneralPurposeAllocator)
        arena: mem.Arena
        defer mem.arena_free_all(&arena)
        context.allocator = mem.arena_allocator(&arena)

        // Zig: try alloc.create(Foo)
        foo := new(Foo)
        defer free(foo)

        // Zig: try alloc.alloc(Foo, 10)
        slice := make([]Foo, 10)
        defer delete(slice)

        // Zig: try alloc.dupe(u8, "hello")
        str := strings.clone_from_cstring("hello", context.allocator)
        defer delete(str)
    }
    ```

## Porting Checklist

```
ZIG → ODIN:
1. `pub fn` → `proc`
2. `*T` → `^T` (pointers)
3. `anytype` → `$T: typeid` + `where` clauses
4. `try` → `or_return` / explicit union checks  
5. `error.Foo` → tagged union variants
6. `defer` → `defer` (identical)
7. `std.mem.Allocator` → `context.allocator`
8. `[]T` → `[]T` (identical slices)
9. `comptime` → procedure groups / `#packed`
10. `@ptrCast` → `cast(^T)`
```

This guide preserves Zig's safety and explicitness while leveraging Odin's context-oriented design and superior duck typing ergonomics.
