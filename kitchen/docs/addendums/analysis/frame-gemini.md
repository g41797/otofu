# Odin Service Framework: `frame` Package Design

## Part 1: Context & Intelligence Prompt
*This section preserves the architectural intent for future sessions or other AI models.*

**Role:** Software Architect (Systems Programming, Message-Oriented Middleware).
**Project Context:** Porting `tofu` (Zig-based modular monolith messaging) to Odin.
**Core Philosophy:** The system follows a "Matryoshka" nesting model.
1.  **polynode**: The basic structural unit (no logic).
2.  **frame**: The transport container (ABI/Wire format) that carries a `polynode`.
3.  **matryoshka**: The runtime substrate that processes `polynode` items.
4.  **runtime**: The orchestrator.
5.  **plugins**: The intelligence/business logic.

**Key Constraints:**
- **Zero Logic in Frame:** `frame` only handles packaging and transport encoding.
- **Polynode First:** The `polynode` struct MUST be the first field in the `Frame_Message` to allow pointer casting/extraction.
- **Memory:** Use Odin's `dynamic_array` and `slice` for payloads. Avoid complex interfaces; prefer simple data structures.
- **Independence:** `frame` depends on `polynode`, but never on `matryoshka` or `runtime`.

**Reference Links:**
- [Zig Tofu Message Source](https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/src/message.zig)
- [Tofu Patterns & Documentation](https://raw.githubusercontent.com/g41797/tofu/refs/heads/main/docs_site/docs/mds/patterns.md)
- [Polynode Definition (Odin)](https://raw.githubusercontent.com/g41797/matryoshka/refs/heads/main/polynode.odin)
- [Initial Mapping Analysis](https://github.com/g41797/tofusite/blob/main/root/otofu/analysis/S6_tofu_mapping_fixed.md)

---

## Part 2: High-Level Design (Odin `frame` Package)

### 1. Package Overview
The `frame` package defines the "on-the-wire" or "in-memory" container used to move data between services. It provides a stable ABI for Odin-based services while strictly adhering to the `polynode` structural core.

### 2. File Structure
```text
frame/
├── doc.odin          # Package documentation and usage examples
├── types.odin        # Enums, constants, and basic error types
├── message.odin      # The main Frame_Message struct and core API
├── headers.odin      # Logic for text and binary header manipulation
├── payload.odin      # Payload/Body handling
└── codec.odin        # Signatures for encoding/decoding (ABI layer)
```

### 3. Core Data Structures

#### `types.odin`
```odin
package frame

import "polynode"

Frame_Error :: enum {
    None,
    Invalid_Format,
    Payload_Too_Large,
    Header_Overflow,
    Allocation_Failed,
}

// Fixed-size binary header keys for speed
Header_Bin_Key :: enum u16 {
    None         = 0,
    Trace_ID     = 1,
    Service_ID   = 2,
    Priority     = 3,
    TTL          = 4,
    Custom_Start = 100,
}
```

#### `message.odin`
```odin
package frame

import "polynode"

// The physical container moved through the system
Frame_Message :: struct {
    // MANDATORY: First field for structural operations
    node:         polynode.Polynode, 
    
    // Binary headers: Fixed-size, fast access
    headers_bin:  [dynamic]Header_Binary_Field,
    
    // Text headers: Key-Value pairs for external metadata (HTTP style)
    headers_text: [dynamic]Header_Text_Field,
    
    // The raw data
    body:         [dynamic]u8,
}

Header_Binary_Field :: struct {
    key:   Header_Bin_Key,
    value: u64,
}

Header_Text_Field :: struct {
    key:   string,
    value: string,
}
```

### 4. API Signatures

#### Message Lifecycle (`message.odin`)
```odin
// Initialization and Destruction
message_create  :: proc(allocator := context.allocator) -> (^Frame_Message, Frame_Error) ---
message_destroy :: proc(msg: ^Frame_Message) ---

// Extraction (Cast helper)
// Returns the pointer to the internal polynode for matryoshka usage
as_polynode :: proc(msg: ^Frame_Message) -> ^polynode.Polynode ---
```

#### Header Manipulation (`headers.odin`)
```odin
// Binary Headers
set_bin_header :: proc(msg: ^Frame_Message, key: Header_Bin_Key, value: u64) ---
get_bin_header :: proc(msg: ^Frame_Message, key: Header_Bin_Key) -> (u64, bool) ---

// Text Headers
set_text_header :: proc(msg: ^Frame_Message, key: string, value: string) ---
get_text_header :: proc(msg: ^Frame_Message, key: string) -> (string, bool) ---
```

#### Payload Handling (`payload.odin`)
```odin
// Replaces the body with new data
set_body :: proc(msg: ^Frame_Message, data: []u8) -> Frame_Error ---

// Appends data to existing body (Odin dynamic array style)
append_body :: proc(msg: ^Frame_Message, data: []u8) -> Frame_Error ---

// Returns a slice view of the body
get_body_view :: proc(msg: ^Frame_Message) -> []u8 ---
```

#### Encoding / ABI (`codec.odin`)
```odin
// Serializes the whole frame into a byte slice for transport
encode :: proc(msg: ^Frame_Message, allocator := context.allocator) -> ([]u8, Frame_Error) ---

// Deserializes bytes into a Frame_Message (extracts polynode)
decode :: proc(data: []u8, allocator := context.allocator) -> (^Frame_Message, Frame_Error) ---
```

### 5. Key Implementation Decisions

1.  **Body as `[dynamic]u8`**: Instead of the Zig `Appendable` interface, we use Odin's built-in dynamic arrays. This is simpler, more performant, and follows the "raw and human" code style preferred by the community.
2.  **Text Header Views**: `Header_Text_Field` uses `string` (which is a `struct { data: ^u8, len: int }`). During `decode`, we point these strings into the original buffer to avoid small allocations, unless the user explicitly clones them.
3.  **No Logic**: The `frame` package does not check if a message is a "Request" or "Response". It doesn't know about `flows` or `subsystems`. It only knows how to store and move `bytes` + `polynode`.
4.  **Polynode Integration**: Because `Polynode` is the first field, the `runtime` can receive a `Frame_Message`, cast the pointer to `^Polynode`, and pass it directly to `matryoshka` without any memory copying.

### 6. Summary for Developers
- Use `frame` when you need to send data over a network or between isolated service boundaries.
- Use the `as_polynode()` helper when passing a message to the `matryoshka` engine.
- All memory allocated by the `frame` is owned by the `Frame_Message` and cleaned up in `message_destroy`.