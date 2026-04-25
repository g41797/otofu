# P5 Zig vs Odin — Architectural Comparison

Context: otofu (Odin port of tofu)
Constraint set: Reactor model, single I/O thread, multi-threaded clients via mailboxes, no coroutines, no shared mutable state.

Architecture only. No syntax.

---

## Alignment Summary

Before the differences: both languages share the same foundational philosophy for this project.

| Aspect | Zig | Odin | Alignment |
|--------|-----|------|-----------|
| No GC | Yes | Yes | FULL |
| Explicit allocation | Yes | Yes (with caveat) | PARTIAL |
| No hidden control flow | Yes | Yes | FULL |
| Manual memory management | Yes | Yes | FULL |
| No runtime | Yes | Yes | FULL |
| No built-in async | Yes (removed) | Yes (never had) | FULL |
| Thread model | OS threads only | OS threads only | FULL |
| No inheritance | Yes | Yes | FULL |

The alignment is strong. The differences are in discipline mechanisms, not philosophy.

---

## 1. Memory Model

### Zig Design

Allocator is always an explicit parameter. Every allocation call site names the allocator. The type system makes the allocator visible at every boundary. `errdefer` provides deterministic cleanup on error paths. `defer` on success paths.

tofu passes the Engine's allocator (`gpa`) explicitly to every subsystem that allocates. All allocations are traceable to one source. No allocation happens without the caller knowing which allocator is used.

### Odin Design

Allocator has two modes:

**Explicit:** Passed as a parameter, same as Zig. Used by Matryoshka — every Pool and Mailbox stores an explicit `mem.Allocator` internally.

**Implicit:** `context.allocator` — a per-call-stack implicit allocator inherited from the calling context. Any `new(T)` or `make(...)` call without an explicit allocator silently uses `context.allocator`. This is idiomatic Odin for application code.

The two modes coexist. A procedure called from the Reactor thread inherits the Reactor thread's `context.allocator`. A procedure called from an application thread inherits that thread's `context.allocator`. If those are different allocators, any code that switches between implicit and explicit paths may allocate from the wrong source.

**Odin has no `errdefer`.** Defer runs on all exits — success and failure equally. A boolean success flag must gate cleanup defers. This is more verbose and more prone to programmer error than Zig's `errdefer`.

### Key Differences

| Aspect | Zig | Odin |
|--------|-----|------|
| Allocator discipline | Enforced by function signature | Convention; `context.allocator` bypasses it |
| Error-path cleanup | `errdefer` — automatic on error exit | Manual bool-flag guard on `defer` |
| Hidden allocation risk | None — every call site names allocator | Any `new()`/`make()` without explicit alloc silently uses context |
| Allocator visibility | Compile-time visible at call site | Runtime context — invisible unless explicitly passed |

### Impact on tofu Design

tofu's design assumes all allocations are explicit and traceable. This assumption is correct in Zig — the type system enforces it. In Odin, the `context.allocator` escape hatch can silently violate it.

tofu also uses `errdefer` heavily in Engine and Channel initialization paths. Any failure during multi-step initialization triggers cleanup in reverse order. This ordering is the teardown sequence from P2_state_machines.md.

### Required Change in otofu

**MR-1 — Adopt Matryoshka's explicit allocator discipline universally.**
Every otofu allocation passes an explicit allocator. No code in otofu calls `new(T)` or `make(...)` without specifying the allocator parameter explicitly. `context.allocator` is never used inside otofu modules.

**MR-2 — The Reactor thread must override `context.allocator` at startup.**
When the Reactor thread starts, it inherits `context` from the spawning thread (Engine creation). If Reactor code calls any library that uses `context.allocator` internally, it would allocate from the spawning thread's allocator. The Reactor must set its `context.allocator` to the Engine's allocator before doing any work. This is a thread-proc entry discipline.

**MR-3 — Replace every `errdefer` with a success-flag guarded `defer`.**
Pattern: set `ok := false` on entry; `defer if !ok { cleanup }`. Set `ok = true` before all return paths that represent success. This must be applied to every multi-step initialization path: `Engine.create`, `ChannelGroup.create`, `Channel.open`.

**MR-4 — Define a pool budget for MetaHeaders and Body buffers.**
tofu's Zig allocator model makes oversized buffer retention visible at allocation sites. Odin's pool hooks (`on_put`) must actively check buffer capacity and free oversized buffers. There is no language-level safeguard. This is a runtime policy that Zig's allocator model implicitly encouraged tofu to think about; otofu must make it explicit in hook code.

### Risk

**HIGH**

The `errdefer`-to-flag-defer translation is error-prone. A missed success flag means cleanup runs on both success and failure paths. In the context of the ownership model (P3_ownership.md), this means an item may be freed twice (double-free) or freed before use (use-after-free). Every initialization path in otofu must be audited for correct flag placement.

The `context.allocator` escape is a silent failure mode. No compiler warning. No runtime error on misuse. Only symptoms downstream — memory corruption or allocations on the wrong heap. Enforce MR-1 and MR-2 by code review rule, not by trust.

---

## 2. Error Handling

### Zig Design

Errors are a language feature. Error sets are closed (the set of possible errors is known at compile time). Error unions (`!T`) force the caller to handle the error or propagate it. `try` propagates. Unused error values are a compile error.

The state machine transitions in tofu (P2_state_machines.md) are partially driven by error propagation. A connect failure propagates up, and the Channel transitions to `closed` as a direct consequence of the error unwinding.

### Odin Design

Errors are a convention, not a language feature. The idiomatic form is multiple return values — typically `(T, bool)` or `(T, ErrorEnum)`. `or_return` propagates the "failure" part of a multi-return, similar to `try`, but only for boolean-compatible results. There is no compile-time check that an error was handled — a caller can ignore the second return value silently.

Error sets do not exist. The complete set of errors a procedure can return is documentation, not a type-checked contract.

### Key Differences

| Aspect | Zig | Odin |
|--------|-----|------|
| Error type in type system | Yes — error union is a first-class type | No — convention only |
| Unhandled error | Compile error | Silent — ignored return value |
| Error propagation | `try` — automatic, with stack unwind | `or_return` — manual, limited to bool/error |
| Error set composition | Comptime — merge error sets | Not applicable — no error sets |
| Error state machine integration | Error unwind can drive state transitions | State transitions must be explicit |

### Impact on tofu Design

tofu's Reactor dispatches errors from Socket operations (connect failed, recv returned error, handshake timeout) into state machine transitions. The error type tells the state machine which transition to take: ECONNREFUSED → `closed` directly; EAGAIN → stay in current state; ETIMEDOUT → `closed` after timeout.

In Zig, the error union makes this dispatch natural — `switch (err) { error.ConnectionRefused => ..., error.WouldBlock => ... }`. The error set is closed, so the `switch` can be exhaustive.

In Odin, there is no closed error set. The Reactor cannot know at compile time whether it has handled all possible error cases. A new socket error introduced without updating the dispatch switch is silently ignored.

### Required Change in otofu

**EH-1 — Define a closed otofu error enum.**
otofu defines an explicit `Engine_Error` enum covering all conditions the Reactor and application need to distinguish. This enum is not extensible by convention — any new error condition requires adding to the enum and updating all dispatch sites. This is the Odin equivalent of a Zig closed error set.

**EH-2 — State machine transitions must be explicitly driven, not implied by error propagation.**
In tofu, an error propagating up the call stack can drive a Channel from `connecting` to `closed` via function returns. In otofu, `or_return` cannot drive state machine transitions — it only propagates a bool. Every transition triggered by an error must be an explicit call to a state transition function in the Reactor's event loop.

**EH-3 — All error returns at the Reactor boundary must be checked.**
Application-facing functions (`post`, `waitReceive`, `get`, `put`) must return explicit error results. The application must check them. Document which errors leave ownership with the caller (TP-M2-fail from P3_ownership.md: `post` failure must leave `m^` non-nil).

**EH-4 — Audit every call site where EAGAIN / EWOULDBLOCK is expected.**
Non-blocking I/O in the Reactor depends on distinguishing "operation would block" (normal, re-arm Poller) from "operation failed" (error, drive state transition). In Zig, `error.WouldBlock` is distinct from other errors. In Odin, the OS error code is a raw integer that must be compared explicitly. Every Socket I/O call site must handle this distinction.

### Risk

**HIGH**

The silent-ignore risk for error returns is the most dangerous difference. In the state machines (P2_state_machines.md), every missing states error (13 total, 22 race risks) was caused by implied state transitions that were not made explicit. Odin's error handling forces the programmer to be explicit, which is the right direction — but only if the programmer actually writes the explicit checks. Zig's type system would catch the omission; Odin's will not.

The absence of `errdefer` (covered under Memory Model) compounds this: an error that is not caught also skips cleanup. Double exposure.

---

## 3. Threading

### Zig Design

Standard OS threads. Explicit synchronization (`Mutex`, `Condition`, `Semaphore`). Atomics. No data race detection. tofu's threading model: one Reactor thread (I/O), N application threads. All cross-thread communication via mailboxes. No shared mutable state. This model is enforced by architecture, not the language.

### Odin Design

Same: standard OS threads, `core:sync`, `core:atomic`. Same absence of data race detection.

The structural difference is `context`. Every Odin procedure invocation carries an implicit `context` parameter. Context holds: allocator, logger, error handler, and user-defined fields. When a new thread is spawned, it inherits the parent's `context` at the moment of creation.

This creates a structural shared reference: the spawned thread holds a copy of the parent's `context` value. If that context contains a non-thread-safe allocator (e.g., an arena), both threads use the same allocator concurrently without any synchronization.

Additionally, `core:sync.Mutex`, `core:sync.RW_Mutex` are MPMC (multiple producers, multiple consumers) — the same as Matryoshka Mailbox. This is alignment, not conflict.

### Key Differences

| Aspect | Zig | Odin |
|--------|-----|------|
| Thread model | OS threads, explicit | OS threads, explicit |
| Context inheritance | Not applicable | Parent context copied to child thread at spawn |
| Allocator thread safety | Caller's responsibility; explicit | Implicit via context inheritance — hidden risk |
| Mailbox semantics | N/A (tofu's own) | Matryoshka MPMC — DIRECT mapping |
| Thread-local storage | `threadlocal` keyword | `@(thread_local)` attribute |

### Impact on tofu Design

tofu's Reactor thread is spawned by Engine.create. It receives `gpa` as an explicit parameter — no `context` inheritance risk. The explicit allocator model eliminates the context-inheritance problem.

### Required Change in otofu

**TH-1 — Reactor thread proc must set `context.allocator` immediately on entry.**
Reactor thread inherits the Engine-creation thread's context, including its allocator. If the Engine was created in a context using an arena or thread-unsafe allocator, the Reactor would silently use it. The Reactor thread proc sets `context.allocator = engine.explicit_allocator` before any other operation. This ensures all library calls that use `context.allocator` (even ones not written by otofu) are routed to the correct allocator.

**TH-2 — Only one thread calls `mbox_wait_receive` on any given ChannelGroup Mailbox (C-CG2 from P4).**
Matryoshka Mailbox is MPMC. Odin's threading model does not prevent two application threads from both calling `waitReceive` on the same ChannelGroup. This is invariant INV-21 from P3_ownership.md. It cannot be enforced by the API. It is enforced by convention and ownership structure: the ChannelGroup handle is given to exactly one thread.

**TH-3 — `@(thread_local)` for Matryoshka's hook reentrancy guard.**
Matryoshka's pool hook reentrancy guard (C4 from P4_matryoshka_mapping.md) requires a thread-local flag: a pool-in-hook boolean that prevents `pool_get`/`pool_put` from being called inside a hook. The guard must be `@(thread_local)` because multiple threads may use the pool concurrently — a global flag would incorrectly block other threads. This matches Matryoshka's documented `[itc: hook-reentrancy-guard]` pattern.

### Risk

**MEDIUM**

The context-inheritance issue (TH-1) is subtle. It is invisible in normal operation but becomes a data race under concurrent allocation. The Reactor and Engine-creation thread could simultaneously allocate through the same non-thread-safe allocator. The symptom is heap corruption — difficult to diagnose. MR-2 and TH-1 together close this hole.

The MPMC restriction (TH-2) is a convention violation risk, not a language-level risk. Matryoshka's API won't stop a second thread from calling `mbox_wait_receive`. The ownership model (one ChannelGroup handle, one receiver) is the only guard.

---

## 4. Polymorphism

### Zig Design

Two primary mechanisms:

**Comptime generics:** `PollerCore(Backend)` in tofu — a generic struct instantiated at compile time with the OS-specific backend as the type parameter. Zero runtime overhead. Backend is selected at build time via `build.zig` feature flags. The comptime system makes this natural and idiomatic.

**Explicit vtables:** `Ampe` and `ChannelGroup` in tofu expose public interfaces via vtable structs (function pointer tables). The vtable is constructed at comptime using `@ptrCast` and type information. The vtable pattern in Zig requires more boilerplate than Odin's procedure groups but is explicit about dispatch cost.

**Type-safe dispatch:** Zig's comptime system allows type-checked dispatch. The compiler validates that a backend implements the required interface at instantiation time.

### Odin Design

Three relevant mechanisms:

**Conditional compilation (`when`):** `when ODIN_OS == .linux` is the Odin equivalent of Zig's build-time feature flags for platform selection. Same result, different syntax. Used for Poller backend selection.

**Procedure groups:** A named group of procedures with the same logical name but different argument types. Dispatch is static (compile-time), resolved by type. Odin's idiomatic equivalent of Zig's vtable for static dispatch cases. No runtime overhead.

**`rawptr` + id dispatch:** The Matryoshka pattern. PolyNode carries an integer `id`. Receiver casts `rawptr` to the concrete type based on `id`. This is runtime dispatch with no type system checking. The id → type mapping is a convention, not a guarantee. Wrong id → wrong cast → silent corruption or panic.

Odin also has `any` (dynamic type erasure), but it is unsafe and not idiomatic for systems code.

### Key Differences

| Aspect | Zig | Odin |
|--------|-----|------|
| Platform backend selection | `build.zig` feature flags + comptime | `when ODIN_OS` conditional compilation |
| Static polymorphism | `comptime` generics with type checking | Procedure groups + `where` constraints |
| Runtime dispatch | Vtable struct (comptime-constructed) | Procedure groups (static) OR rawptr+id (runtime) |
| Type safety of dispatch | Compile-time checked at generic instantiation | Id-dispatch is not compile-time checked |
| Interface definition | Implicit (duck typing with `anytype`) | Explicit (procedure group or vtable struct) |

### Impact on tofu Design

**PollerCore(Backend):** tofu's generic Poller is instantiated at comptime — one concrete type per platform. No vtable, no runtime overhead. The comptime generic ensures that a missing backend method is a compile error.

**Ampe / ChannelGroup vtable:** tofu exposes public interfaces via vtable structs. The vtable construction uses Zig's comptime capabilities.

**OpCode dispatch in Reactor:** The Reactor switches on `OpCode` (an enum) to route incoming messages. This is not polymorphism — it is a closed enum switch. Zig enforces exhaustiveness on tagged union switches; Odin's `#partial switch` does not.

### Required Change in otofu

**PM-1 — Poller backend: use `when` conditional compilation.**
Zig's `PollerCore(Backend)` comptime generic becomes a `when ODIN_OS`-selected concrete type. The result is the same: one Poller type per platform, selected at compile time. No runtime dispatch. No vtable.

This is the simplest and most direct mapping. Parametric polymorphism (`$Backend: typeid`) is not needed — backend is always known at compile time.

**PM-2 — Public Engine/ChannelGroup interface: use vtable struct.**
The `Engine` and `ChannelGroup` public APIs are defined as vtable structs (function pointer structs). The concrete implementation sets the function pointers at creation. This matches Odin's idiomatic explicit vtable pattern and requires no language features beyond structs and procedure pointers.

Procedure groups are NOT used for the public API. Procedure groups dispatch on argument types; the public API dispatches on instance type (which concrete Engine implementation). These are different dispatch axes.

**PM-3 — OpCode dispatch: use exhaustive switch, not `#partial switch`.**
The Reactor dispatches on `OpCode` in message handling. In Odin, using `#partial switch` on `OpCode` means unhandled opcodes are silently ignored. Use `switch` with a `case:` (default) that panics on unknown opcodes. This preserves the tofu invariant: an unknown OpCode in the Reactor is a programming error.

**PM-4 — Id-dispatch type safety rule.**
Matryoshka's `rawptr` + `id` dispatch is the primary runtime polymorphism mechanism for traveling items (Messages, TriggeredChannels). The `id` → type mapping must be defined in one place and referenced everywhere. Hardcoded casts scattered across the codebase are a violation. Define a dispatch procedure per item type (analogous to Matryoshka's Builder `dtor`) that centralizes the cast.

### Risk

**LOW**

Both languages provide the needed polymorphism mechanisms. The patterns are different but architecturally equivalent. The main risk is PM-3 (partial switch on OpCode) — a trivial but consequential mistake. Procedure groups eliminate the vtable boilerplate of Zig's comptime approach, which reduces the surface for error in the public API.

The id-dispatch type safety gap (PM-4) already exists in tofu (it's inherent to the pattern) and is carried over. Not a new risk introduced by the language difference.

---

## 5. Async / Event Model

### Zig Design

Zig had `async`/`await`/`suspend`/`resume` in version 0.10. They were removed. The language currently has no built-in async. `Io.Evented` is a proposed future interface but is not stable.

tofu was designed explicitly to reject Zig async — the design documents state this repeatedly. tofu uses: one Reactor thread + `epoll`/`kqueue`/`wepoll` + mailboxes + Notifier socket pair. All I/O state is explicit state machines (SM2–SM5). No continuation, no coroutine, no implicit suspension point.

### Odin Design

Odin has never had async or coroutines. The language has no `async`, `await`, `suspend`, or `resume`. There is no planned async feature. The language is explicitly oriented toward explicit control flow.

`nbio` is a third-party Odin library providing non-blocking I/O abstraction with completion semantics (similar to io_uring on Linux, IOCP on Windows). It is NOT part of the standard library. It is NOT part of otofu.

`context` is not a coroutine mechanism — it is an implicit parameter stack, not a resumable frame.

### Key Differences

| Aspect | Zig | Odin |
|--------|-----|------|
| Language async | Removed | Never existed |
| Coroutines | None | None |
| Implicit suspension points | None | None |
| OS event API access | `std.posix` — raw syscall wrappers | `core:sys/linux`, `core:sys/darwin`, `core:sys/windows` — raw syscall access |
| Completion-based I/O | Not used by tofu | `nbio` available but not used by otofu |
| Readiness-based I/O | epoll/kqueue/wepoll (tofu) | Same APIs available |
| io_uring | Not used by tofu | `core:sys/linux` includes io_uring bindings |
| Context as coroutine | Not applicable | Not applicable — context is not a coroutine |

### Impact on tofu Design

tofu's explicit rejection of async is a design decision that survives the port unchanged. The Reactor model, single I/O thread, and state-machine-driven I/O are not language features — they are architectural choices. Both Zig and Odin support this architecture with equal capability.

The absence of async in Odin is an advantage over current Zig: there is no `Io.Evented` API to accidentally adopt, no async function coloring problem, and no temptation to use `suspend` for state machine suspension.

### Required Change in otofu

**AE-1 — Explicit rejection of `nbio`.**
`nbio` provides completion-based I/O (not readiness-based). Using `nbio` would change the threading model (completion callbacks execute on a different thread than the Reactor), violate the Reactor constraint, and require rewriting the entire Poller + TriggeredChannel + state machine layer.

otofu must explicitly document and enforce: `nbio` is not imported, not used, not evaluated. This is an architectural decision recorded here and in CLAUDE.md or equivalent project rules.

**AE-2 — Explicit rejection of `io_uring`.**
`core:sys/linux` includes io_uring bindings. io_uring is completion-based (Proactor). Using it would violate the Reactor constraint for the same reasons as `nbio`. otofu uses `epoll` on Linux exclusively.

**AE-3 — Notifier remains socket-pair based.**
The Matryoshka `mbox_interrupt` mechanism (condition variable wake) cannot substitute for the Notifier — the Reactor blocks on `epoll_wait`/`kevent`, not on `mbox_wait_receive`. This constraint is architectural, not a language limitation. It carries over unchanged from tofu.

**AE-4 — Poller backends: same OS APIs, same abstractions.**
| Platform | Zig (`std.posix`) | Odin (`core:sys/...`) |
|----------|------------------|----------------------|
| Linux | `epoll_create1`, `epoll_ctl`, `epoll_wait` | `core:sys/linux` equivalents |
| macOS/BSD | `kqueue`, `kevent` | `core:sys/darwin` equivalents |
| Windows | `wepoll` (C shim over AFD_POLL) | `wepoll` (same C shim via FFI) |

The Poller backend implementations are direct API translations. No architectural change.

### Risk

**LOW**

Odin's absence of async is a natural fit for the Reactor model. The risk is deliberate misuse (AE-1, AE-2), not accidental. Document the rejection clearly. The OS event APIs are equivalent across both languages.

---

## Cross-Cutting Concerns

Issues that span multiple areas above.

---

### CC-1: Type Safety of the Ownership Convention

**Zig:** `*?*Message` (the `^MayItem` equivalent) is partially type-safe. The outer pointer `*` and the optional `?*` give the compiler some information. `try` on an optional fails to compile if the error case is unhandled.

**Odin:** `^MayItem` (`^Maybe(^PolyNode)`) is a convention. The compiler does not enforce `m^ == nil` checks. Matryoshka documents: "Following it is on you." The language has no borrow checker.

**Impact:** The ownership model (P3_ownership.md — 27 invalid states) depends entirely on programmer discipline in Odin. In Zig, the type system catches some violations at compile time. In Odin, it catches none.

**Required change:** Every ownership transfer point (TP-M1 through TP-M7 from P3) must have a corresponding test. The test verifies that `m^` is nil after transfer and non-nil after receive. No static guarantee — runtime verification in tests is the only substitute.

**Risk: MEDIUM** — Cannot be fully mitigated by architecture alone. Test discipline is required.

---

### CC-2: Closed vs Open Dispatch Sets

**Zig:** Error sets are closed (comptime-known). Tagged union switches can be exhaustive — the compiler warns if a case is missing.

**Odin:** Error enums and union switches use `#partial switch` as the default non-exhaustive form. Exhaustive `switch` is possible but requires a default `case:` that panics or handles the catch-all.

**Impact:** tofu's Reactor dispatches on: OpCode (8 values), Socket errors, state machine transitions. Zig's exhaustive dispatch means a new OpCode or error condition is a compile error if not handled. Odin's `#partial switch` silently ignores it.

**Required change:** All dispatch in otofu uses exhaustive `switch` (not `#partial switch`) with an explicit `case:` that panics on unknown values. This matches Matryoshka's design principle: unknown id → panic. Apply it to OpCode, error codes, and any other dispatch enum.

**Risk: MEDIUM** — Disciplined use of exhaustive switch mitigates this entirely. The risk is not enforced by the language.

---

### CC-3: Build System and Platform Selection

**Zig:** `build.zig` is Zig code. It drives compilation, test execution, and platform configuration. Backend selection (`epoll_backend.zig` vs `kqueue_backend.zig` vs `wepoll_backend.zig`) is a `build.zig` decision.

**Odin:** No official build system. Platform selection is `when ODIN_OS`. Package import paths handle the rest.

**Impact:** tofu's four-mode test requirement (Debug, ReleaseSafe, ReleaseFast, ReleaseSmall) becomes four separate Odin compilation modes. There is no `build.odin` equivalent — this must be automated via scripts or CI configuration.

**Required change:** CI must run otofu tests in all applicable Odin optimization modes: `-vet`, `-debug`, `-o:none`, `-o:minimal`, `-o:speed`. Map tofu's four modes to Odin equivalents. Define this in RULES.md for otofu (equivalent of tofu's RULES.md).

**Risk: LOW** — Operationally different but architecturally equivalent.

---

### CC-4: Comptime vs Runtime Information

**Zig:** `comptime` can compute types, sizes, and dispatch tables at compile time. `@sizeOf(BinaryHeader) == 16` can be a compile-time assertion — the compile fails if the struct layout changes.

**Odin:** `size_of(Header) == 16` can be a compile-time `#assert`. Equivalent capability.

**Impact:** The BinaryHeader/Header wire-format correctness check (must be exactly 16 bytes) is directly portable. No architectural change.

**Required change:** None.

**Risk: NONE**

---

## Decision Record

Explicit architectural rejections for otofu. Not options to evaluate later.

| Decision | Rejected | Reason |
|----------|---------|--------|
| DR-1 | `nbio` | Completion-based (Proactor). Violates Reactor constraint. |
| DR-2 | `io_uring` (via `core:sys/linux`) | Completion-based. Violates Reactor constraint. |
| DR-3 | `context.allocator` for otofu-internal allocations | Hidden allocation. Violates explicit ownership invariant. |
| DR-4 | `#partial switch` on closed dispatch sets | Silent miss. All dispatch on OpCode, error, state uses exhaustive switch. |
| DR-5 | `mbox_interrupt` as Notifier substitute | Condition variable cannot wake a thread blocked on `epoll_wait`. Mechanistically incompatible. |
| DR-6 | Multiple threads calling `mbox_wait_receive` on one ChannelGroup | Race condition. INV-21 from P3_ownership.md. |
| DR-7 | `pool_get_wait` in Reactor thread | Blocks I/O thread. All Reactor pool access is non-blocking (`.Available_Only`). |
| DR-8 | `any` type for polymorphism | Unsafe type erasure. Use id-dispatch (Matryoshka) or procedure groups. |

---

## Risk Register

| ID | Area | Risk | Level | Mitigation |
|----|------|------|-------|-----------|
| R-MM1 | Memory | `context.allocator` used implicitly inside otofu | HIGH | MR-1: explicit allocator discipline rule |
| R-MM2 | Memory | `errdefer` → flag-defer translation misses cleanup paths | HIGH | MR-3: audit all multi-step init paths |
| R-MM3 | Memory | Appendable buffer capacity grows unbounded in pool | MEDIUM | MR-4: capacity check in `on_put` |
| R-EH1 | Error | Error return value ignored at call site | HIGH | EH-1: defined error enum; EH-3: audit all call sites |
| R-EH2 | Error | State transition not driven on error (implicit in Zig, explicit required in Odin) | HIGH | EH-2: every state transition is an explicit call |
| R-EH3 | Error | EAGAIN vs fatal error not distinguished | MEDIUM | EH-4: per-call-site OS error code check |
| R-TH1 | Threading | Reactor thread uses parent `context.allocator` | MEDIUM | TH-1: override context at thread entry |
| R-TH2 | Threading | Two threads call `waitReceive` on same ChannelGroup | MEDIUM | TH-2: ownership doc; single-receiver rule |
| R-PM1 | Polymorphism | `#partial switch` on OpCode silently ignores new values | MEDIUM | PM-3: exhaustive switch with panic default |
| R-PM2 | Polymorphism | Id-dispatch cast without central dispatch table | LOW | PM-4: centralize cast per type |
| R-OW1 | Ownership | `m^` not checked after transfer | MEDIUM | CC-1: ownership tests at every TP |
| R-AE1 | Async | `nbio` or `io_uring` introduced by contributor | LOW | DR-1, DR-2: architectural decision documented |
