# Tofu to Otofu: A Comprehensive Architectural Porting Guide

## Introduction

This document serves as the canonical guide for porting the `tofu` message-passing engine from Zig to Odin (`otofu`). It synthesizes the provided architectural notes, identifies and resolves contradictions, and establishes a clear philosophical and technical path forward.

The primary goal is not a line-for-line translation. Instead, this is an exercise in **architectural refinement**. `otofu` will act as a "sparring partner" for `tofu`, using Odin's philosophical preference for simplicity and explicitness to challenge and strengthen the core abstractions.

---

## Part 1: Architectural Philosophy & Guiding Principles

The port is guided by a set of core principles derived from the `otofu-direction.md` and `otofu.md` documents.

1.  **Layers, Not Sockets**: The fundamental concern is the clean separation of two distinct layers:
    *   **Conversation Layer**: Manages the "who, what, and why" of communication: identity, message lifecycles, state machines, and structured cooperation. **This is the core of tofu and must be preserved.**
    *   **Transport Layer**: Manages the "how" of communication: reading/writing bytes, readiness/completion events, and I/O primitives. This layer is the subject of our experiment.

2.  **The Sparring Partner Model**: `otofu` is not a replacement for `tofu` but a tool to test its design. If an abstraction from `tofu` feels awkward or overly complex in Odin, we must question if it was truly essential or merely a convenience of the original environment. The goal is to discover the **essential invariants** of the architecture.

3.  **The Proactor Experiment**: Instead of waiting for Zig's `Io.Evented` (a Reactor model), we will proactively build `otofu` on a **Proactor model** (e.g., using IOCP on Windows or `io_uring` on Linux). This is a methodological choice to test a core hypothesis: *What changes in the Conversation Layer when the Transport Layer's semantics change from readiness-based to completion-based?*

4.  **Preserve Core Invariants**: The message-based mindset is non-negotiable. The following invariants from the Conversation Layer must remain constant:
    *   Message IDs function as unique Job IDs.
    *   "Signals" (fire-and-forget) remain distinct from "Requests" (expecting a reply).
    *   Shutdown is a negotiated, structured process.
    *   Channel lifecycles define the boundaries of a conversation.

5.  **Process Over Product**: This is an open exploration, not the creation of a finished framework. The process, including challenges and simplifications, should be documented to foster dialogue and lead to a more robust final architecture.

---

## Part 2: Core Pattern Migration (Zig -> Odin)

This section provides technical guidance for translating specific Zig patterns to idiomatic Odin, resolving contradictions found in the source documents.

### 2.1. Ownership, Memory Management, and the `*?*Message` Idiom

The safe transfer of heap-allocated messages between threads is critical.

**Zig Idiom**: `*?*Message` with `ampe.get()` and `defer ampe.put(&msg)`.

**Analysis**: The `ownership.md` document proposes a direct translation to an `Owned_Message` struct with an `atomic.Bool valid` flag. While technically sound, this approach is a literal translation and misses an opportunity to embrace a more Odin-idiomatic pattern, as encouraged by the "sparring partner" philosophy. The RAII-style guard proposed in the same document is a much stronger, more idiomatic pattern.

**Recommended Odin Pattern**: **Scope-based Ownership with RAII Guards**

The `Owned_Message_Guard` is the preferred approach. It leverages `defer` to tie the lifetime of the message directly to a scope, which is a powerful and common pattern in Odin.

```odin
package tofu

// Simplified from ownership.md
Message :: struct { id: u64 }
Allocation_Manager :: struct { /* ... pool ... */ }

// The Guard struct holds the two things needed to manage the resource.
Owned_Message_Guard :: struct {
    ptr: ^Message,
    am:  ^Allocation_Manager,
}

// Acquires the resource.
guard_init :: proc(am: ^Allocation_Manager) -> (g: Owned_Message_Guard, err: Alloc_Error) {
    // ... logic to get a message from the pool ...
    // Returns a guard with the message and a pointer to the manager.
}

// Releases the resource. This is the key.
guard_destroy :: proc(g: ^Owned_Message_Guard) {
    if g.ptr != nil {
        // ... logic to return the message to g.am's pool ...
        g.ptr = nil // Prevent double-free.
    }
}

// Usage is clean, safe, and leverages `defer`.
main_thread :: proc() {
    am: Allocation_Manager
    // ... setup ...

    guard, err := guard_init(&am)
    if err != nil { return }
    defer guard_destroy(&guard) // COMPILE-TIME GUARANTEED CLEANUP

    // Safe to use guard.ptr within this scope.
    fmt.printf("Got message %d\n", guard.ptr.id)
    
    // When passing ownership to another thread (e.g., via a channel),
    // you must explicitly release the guard's control.
    // e.g., mailbox.send(release_guard(&guard))
}
```

This pattern is superior because:
-   It eliminates the need for a separate `valid` flag. Ownership is implicitly tied to the guard's lifetime.
-   It prevents accidental use-after-free within the same function scope.
-   It aligns perfectly with Odin's data-oriented philosophy of bundling data (`ptr`) with the context needed to manage it (`am`).

### 2.2. Error Handling

**Zig Idiom**: `error!T` error unions and `try`.

**Analysis**: The `zig-to-odin.md` file contained a significant contradiction. It correctly defined an error union (`Tofu_Error :: union { Pool_Empty, Invalid_Channel, None }`) but then used an incorrect and unsafe string-based return in its example function.

**Recommended Odin Pattern**: **Multi-value returns with an error union.**

This is the canonical and correct way to handle errors in Odin.

```odin
package tofu

Tofu_Error :: enum {
    None,
    Pool_Empty,
    Invalid_Channel,
}

// CORRECTED EXAMPLE
get_message_from_pool :: proc(pool: ^Pool) -> (^Message, Tofu_Error) {
    if pool.is_empty() {
        return nil, .Pool_Empty
    }
    // ... get message ...
    return msg, .None
}

process_message :: proc() {
    msg, err := get_message_from_pool(&pool)
    if err != .None {
        // Handle error
        switch err {
        case .Pool_Empty:       fmt.println("Pool exhausted")
        case .Invalid_Channel:  fmt.println("Bad channel")
        }
        return
    }
    // Use msg safely here
    fmt.println("Got message:", msg.id)
}
```

### 2.3. Polymorphism & Interfaces

**Zig Idiom**: `anytype` for duck-typing, explicit vtables, or `comptime` checks.

**Analysis**: The `odin-zig.md` and `odin-interfaces.md` documents were largely redundant but provided excellent analysis. The conclusion is that Odin's built-in features for polymorphism are highly ergonomic.

**Recommended Odin Patterns**: Choose based on the use case.

1.  **For Static, Known Types**: **Procedure Groups**.
    *   Use to create a named "interface" for a closed set of types. It's zero-cost and simple.
    *   *Example*: `draw :: proc { draw_sprite, draw_text }`

2.  **For Generic Algorithms**: **Parametric Polymorphism with `where` clauses**.
    *   This is Odin's powerful equivalent to static duck-typing. It's the most flexible and often-used pattern.
    *   *Example*: `render_all :: proc($T: typeid, items: []T) where draw: proc(^T)`

3.  **For Heterogeneous Collections & Plugins**: **Explicit Vtables**.
    *   The pattern is identical to Zig's. It's required for runtime polymorphism.
    *   *Example*: `Shape :: struct { vtable: ^Shape_VTable, data: rawptr }`

The library organization patterns suggested in `odin-interfaces.md` should be followed to keep implementations clean.

### 2.4. Data Structures

-   **Tagged Unions (Zig `union(enum)`)**: Use Odin's `union` and an explicit `enum` tag field. For runtime introspection of parametric unions, use `core:reflect` as shown in the documents, but prefer explicit tags for performance and clarity where possible.
-   **Dynamic Buffers (Zig `Appendable`)**: Use Odin's `core:dynamic_array` for most cases. It integrates with `context.allocator` and provides a robust, tested implementation. Only build a manual `[]u8` wrapper for specialized needs.
-   **Mailboxes (Zig `MailBox`)**: Use `core:sync.Blocking_Queue` as the foundation. It provides the necessary MPSC semantics. Build any extra features like `interrupt` or `close` semantics on top of it using `core:sync` primitives (`Mutex`, `Cond`).

---

## Part 3: The Proactor Experiment: First Milestone

The first major goal of the `otofu` project is to execute the Proactor experiment.

**Objective**: To understand the impact of a completion-based I/O model on the `tofu` Conversation Layer.

**Action Plan**:

1.  **Build a Minimal Proactor Core**: Implement a thin wrapper over the platform's native Proactor API (e.g., IOCP on Windows). This core will be responsible for submitting async read/write operations and dequeuing completed events.
2.  **Implement a Single Stream**: Create a `Stream` type that represents a single network connection, managed by the Proactor.
3.  **Add Framing**: Implement a simple message framing layer on top of the `Stream` to delineate message boundaries (e.g., length-prefixing).
4.  **Integrate Message Queue**: Connect the Proactor's completion events to the existing message-passing logic (`core:sync.Blocking_Queue`). A completed read event should result in a `Message` being pushed to a consumer queue.
5.  **Test the Full Lifecycle**: Write a test that covers:
    *   Connection establishment.
    *   Sending and receiving several messages.
    *   A structured shutdown sequence.
    *   Resource cleanup.

Throughout this process, we will continuously evaluate whether the core **Conversation Layer invariants** are being stressed or broken. Any such friction is a critical data point for the experiment.

---

## Part 4: High-Level Porting Checklist (Zig → Odin)

| Zig Construct | Odin Equivalent | Notes |
| :--- | :--- | :--- |
| **Philosophy** | | |
| `std.mem.Allocator` | `context.allocator` | Embrace the implicit context system. |
| `try` / `catch` | Multi-value return `(T, Error)` | See section 2.2. |
| `anytype` generics | Parametric poly (`$T: typeid`) + `where` | More ergonomic and better error messages. |
| `comptime` | `#packed`, procedure groups, `where` | Odin prefers dedicated features over a single meta-programming tool. |
| **Syntax & Types** | | |
| `pub fn` | `proc` | |
| `*T` (pointer) | `^T` | |
| `[]T` (slice) | `[]T` | Identical. |
| `[]const T` | `[]T` or `[^]T` | Odin's `[]T` is `const` by default for data, `[^]T` for pointer arrays. |
| `@ptrCast` | `cast(^T)` | |
| `defer` | `defer` | Identical. |
| `union(enum)` | `union` + explicit `enum` tag | More explicit control. |
| `error{...}` | `enum {...}` | Used in a union for error returns. |

This guide provides the necessary direction. The next step is to begin implementing the Proactor core as outlined in Part 3.
