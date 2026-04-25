# Odin Research Tasks

## Questions to Answer

### Networking
- [ ] Does `core:net` support setting sockets to non-blocking mode?
- [ ] Is there a poll/select/epoll wrapper in Odin?
- [ ] How to do Unix domain sockets in Odin?
- [ ] Socket error handling patterns in Odin?

### Threading
- [ ] Thread-safe queue pattern in Odin?
- [ ] Inter-thread signaling (eventfd equivalent)?
- [ ] Thread lifecycle management?

### Memory
- [ ] Is `heap_allocator()` thread-safe?
- [ ] How to share allocator across threads?
- [ ] Pool allocator patterns in Odin?

### Community
- [ ] Existing async I/O libraries in Odin?
- [ ] Networking projects in Odin ecosystem?
- [ ] Odin Discord activity level?

---

## Resources to Check

### Official
- https://odin-lang.org/docs/
- https://github.com/odin-lang/Odin/tree/master/core
- Odin Discord

### Code to Study
- `core:net` source
- `core:sys/posix` source
- Any Odin HTTP server implementations

---

## Minimal Proof of Concept

Write this before committing to full port:

```odin
// poc.odin - Non-blocking TCP echo server with poll

package main

import "core:net"
import "core:sys/posix"
import "core:fmt"

main :: proc() {
    // 1. Create listener socket
    // 2. Set non-blocking
    // 3. poll() loop
    // 4. Accept connections
    // 5. Echo data back
}
```

If this works cleanly, proceed with port.
If this requires ugly workarounds, reconsider.

---

## Decision Criteria

**Go ahead if:**
- Non-blocking sockets work
- poll() or equivalent available
- Community shows interest

**Reconsider if:**
- Need heavy FFI for basic I/O
- No poll/select available
- Community indifferent
