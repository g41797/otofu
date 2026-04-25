# otofu API Skeletons (Concise)

**Constraints:** Explicit allocators (MR-1), MayItem ownership (L0), Engine_Error mapping (EH-1).

---

### L5 — Public API
```odin
engine_create       :: proc(alc: mem.Allocator, opt: Options) -> (Engine, Engine_Error)
engine_destroy      :: proc(e: Engine)
engine_get          :: proc(e: Engine, strat: Strategy, m: ^MayItem) -> Engine_Error
engine_put          :: proc(e: Engine, m: ^MayItem)

cg_create           :: proc(e: Engine) -> (CG, Engine_Error)
cg_destroy          :: proc(e: Engine, cg: CG)
cg_post             :: proc(cg: CG, m: ^MayItem) -> Engine_Error
cg_wait_receive     :: proc(cg: CG, m: ^MayItem, timeout_ms: i64) -> RecvResult
```

### L4 — Protocol Layer
```odin
framer_encode       :: proc(m: ^Message, buf: ^[dynamic]u8)
framer_try_decode   :: proc(buf: []u8, m: ^Message) -> Decode_Result
protocol_dispatch   :: proc(m: ^Message, ch: ^Channel, ctx: ^Dispatch_Context)
```

### L3 — Messaging Runtime
```odin
router_send_reactor :: proc(r: ^Router, m: ^MayItem) -> Send_Result
router_send_app     :: proc(r: ^Router, cg_id: ID, m: ^MayItem) -> Send_Result
router_drain_inbox  :: proc(r: ^Router, batch: ^[dynamic]MayItem) -> Recv_Result
```

### L2 — Reactor Core
```odin
reactor_start       :: proc(e: ^Engine, alc: mem.Allocator, opt: Options)
dual_map_insert     :: proc(dm: ^Dual_Map, chn: u16, seq: u64, tc: ^TC)
dual_map_lookup     :: proc(dm: ^Dual_Map, seq: u64) -> ^TC
ch_mgr_transition   :: proc(ch: ^Channel, state: State)
io_dispatch         :: proc(tc: ^TC, ch: ^Channel, flags: Flags, ctx: ^Context)
```

### L1 — OS / Poller
```odin
poller_register     :: proc(p: ^Poller, fd: Handle, seq: u64, flags: Flags) -> Engine_Error
poller_deregister   :: proc(p: ^Poller, seq: u64) -> Engine_Error
poller_wait         :: proc(p: ^Poller, timeout_ms: i64) -> ([]Event, Engine_Error)
notifier_notify     :: proc(n: ^Notifier)
socket_connect      :: proc(s: ^Socket, addr: Address) -> Engine_Error
socket_accept       :: proc(s: ^Socket) -> (^Socket, Engine_Error)
socket_send         :: proc(s: ^Socket, buf: []u8) -> (int, Engine_Error)
socket_recv         :: proc(s: ^Socket, buf: []u8) -> (int, Engine_Error)
```
