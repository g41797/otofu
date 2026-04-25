# Analysis of Existing Porting Files for otofu

This document evaluates files from `/tofu/porting/odin/port/` against the definitive `otofu` architecture (**P7**) and implementation plan (**Revision B0**).

## 1. Summary of Useful Components

The following files contain logic that can be adapted for specific steps in the `otofu` implementation plan, provided they are refactored to meet strict constraints (Explicit Allocator, Matryoshka Layout, Reactor Model).

| Target Phase/Step | Source File | Useful Logic/Snippets |
| :--- | :--- | :--- |
| **0.2 (OpCodes)** | `odin-message.md` | `Op_Code` enum definitions (10 values). |
| **0.4 (Flags)** | `odin-message.md` | Bit-shifting logic for `proto` byte (OpCode/Origin/More). |
| **1.1 (Message)** | `odin-message.md` | `#packed` struct layout for `Binary_Header` (16 bytes). |
| **2.0 (Appendable)** | `zig-to-odin.md` | `Appendable_Buffer` pattern (manual growth logic). |
| **6.2 (Router/Mbox)** | `zig-to-odin.md` | `sync.Blocking_Queue` wrapper for the `Mailbox` struct. |
| **8.2 (Handshake)** | `shutdown_example.md` | Conceptual Bye/ByeAck logic for negotiated shutdown. |
| **11.1 (Public API)** | `odin-interfaces.md` | Vtable and Procedure Group code patterns for the Engine handle. |

## 2. Mandatory Refactoring Rules

Any code extracted from these files **MUST** be modified according to the following `otofu` mandates:

1.  **Matryoshka Integration (AI-11)**: All pooled/queued structs (Message, TC, Channel) must add `using poly: matryoshka.PolyNode` as the **first field** (offset 0).
2.  **Explicit Allocator (MR-1)**: Every procedure that performs allocation (Appendable growth, Pool creation) must accept an explicit `mem.Allocator` parameter. Remove all uses of `context.allocator`.
3.  **Reactor Core (DR-1)**: Discard all logic related to **Proactor**, **Completion**, **IOCP**, or **io_uring**. The engine is strictly a single-threaded **Reactor** (readiness-based).
4.  **Ownership (P3)**: Discard RAII-style "Guards". Use the **Matryoshka `MayItem`** as the sole ownership token.
5.  **Success-Flag Defer (MR-3)**: Replace RAII-based cleanup with the `ok := false; defer if !ok { ... }` pattern for multi-step initializations.

## 3. Obsolete or Invalid Files

| File | Status | Reason |
| :--- | :--- | :--- |
| `otofu.md` | **OBSOLETE** | Primarily discusses the Proactor experiment, which has been rejected in favor of the Reactor model. |
| `otofu-direction.md` | **OBSOLETE** | Early-stage brainstorming superseded by P7 Architecture. |
| `tofu_to_otofu_porting_guide.md`| **INVALID** | Recommends `context.allocator` and RAII Guards, violating P5 and P3 constraints. |

## 4. Final Recommendation

Do not import these files directly. Instead, use them as a "snippet gallery" for specific Odin idioms (especially Vtables and packed struct bitfields) while implementing the `otofu` plan from the bottom up.
