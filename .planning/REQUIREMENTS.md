# Requirements: p4p onGet Handler

**Defined:** 2026-03-27
**Core Value:** A Python Handler subclass can define `onGet(self, op)` and have it called when a client executes `cxt.get()`, enabling demand-driven hardware reads.

## v1 Requirements

### C++ / Cython Layer

- [x] **CPP-01**: `attachHandler()` wires up a GET intercept when the Python handler has an `onGet` attribute, using `ConnectOp::onGet()` via `ChannelControl::onOp()`
- [x] **CPP-02**: The GET `ExecOp` is wrapped as a `ServerOperation` Python object (reusing existing wrapping machinery)
- [x] **CPP-03**: If handler has no `onGet`, behavior is unchanged — PVXS default GET (return current value) is preserved

### Python Raw Layer

- [x] **RAW-01**: `server/raw.py` `Handler` base class documents `onGet(self, op)` method signature
- [x] **RAW-02**: `SharedPV._WrapHandler` dispatches `onGet` to user handler with proper exception isolation

### Thread Backend

- [x] **THR-01**: `server/thread.py` dispatches `onGet` through a `WorkQueue` (not called directly on pvxs network thread)
- [x] **THR-02**: `onGet` handler can call `op.done(value)` to complete the GET with a value
- [x] **THR-03**: `onGet` handler can call `op.done(error='msg')` to fail the GET

### Asyncio Backend

- [x] **ASIO-01**: `server/asyncio.py` dispatches `onGet` into the asyncio event loop via `loop.call_soon_threadsafe()`
- [x] **ASIO-02**: `async def onGet(self, op)` coroutines are supported

### Cothread Backend

- [x] **CTH-01**: `server/cothread.py` dispatches `onGet` via cothread callback mechanism

### Tests

- [x] **TEST-01**: Integration test: client `get()` triggers `onGet` in thread backend
- [x] **TEST-02**: Integration test: `onGet` calling `op.done(value)` returns the correct value to client
- [x] **TEST-03**: Integration test: `onGet` calling `op.done(error=...)` delivers error to client
- [x] **TEST-04**: Backward compat test: Handler without `onGet` continues to work (default PVXS behavior)
- [x] **TEST-05**: Reference leak test via `RefTestCase` passes (no C extension object leaks)

### Documentation

- [x] **DOC-01**: `documentation/server.rst` (or equivalent) documents `onGet` in Handler API reference
- [x] **DOC-02**: Example showing hardware-read pattern in `onGet`

## Out of Scope

| Feature | Reason |
|---------|--------|
| Modifying pvxslibs upstream | Separate project/PR; user needs this within p4p now |
| Qt backend onGet | Qt backend already has known gaps; not user's use case |
| DynamicProvider onGet | Different API with different semantics; separate feature |
| Auto-polling/subscription alternative | Explicitly excluded by user — on-demand only |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| CPP-01 | Phase 1 | Done (01-01) |
| CPP-02 | Phase 1 | Done (01-01) |
| CPP-03 | Phase 1 | Done (01-01) |
| RAW-01 | Phase 1 | Done (01-01) |
| RAW-02 | Phase 1 | Done (01-01) |
| THR-01 | Phase 2 | Done (02-01) |
| THR-02 | Phase 2 | Done (02-01) |
| THR-03 | Phase 2 | Done (02-01) |
| ASIO-01 | Phase 2 | Done (02-01) |
| ASIO-02 | Phase 2 | Done (02-01) |
| CTH-01 | Phase 2 | Done (02-01) |
| TEST-01 | Phase 2 | Done (02-01) |
| TEST-02 | Phase 2 | Done (02-01) |
| TEST-03 | Phase 2 | Done (02-01) |
| TEST-04 | Phase 2 | Done (02-01) |
| TEST-05 | Phase 2 | Done (02-01) |
| DOC-01 | Phase 2 | Done (02-02) |
| DOC-02 | Phase 2 | Done (02-02) |

**Coverage:**
- v1 requirements: 18 total
- Mapped to phases: 18
- Unmapped: 0 ✓

---
*Requirements defined: 2026-03-27*
*Last updated: 2026-03-27 after initial definition*
