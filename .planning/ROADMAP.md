# Roadmap: p4p onGet Handler

**Project:** Expose `onGet` in p4p's Python Handler interface
**Milestone:** v1 — Full onGet support across thread/asyncio/cothread backends
**Granularity:** Coarse

---

## Phase 1: C++ and Raw Python Layer

**Goal:** GET operations can be intercepted by a Python `onGet` method at the raw callback level, without breaking existing Handlers that don't define `onGet`.

**Scope:**
- Investigate and implement GET interception in `pvxs_sharedpv.cpp` using `ChannelControl::onOp()` + `ConnectOp::onGet()`
- Wrap GET `ExecOp` as `ServerOperation` (reuse existing machinery from `ServerOperation_wrap`)
- Update `_p4p.pyx` / `attachHandler` to wire in the new GET path
- Update `server/raw.py`: document `onGet(self, op)` in `Handler` base class, add `_WrapHandler.onGet()` dispatch with exception isolation
- Backward compat: if handler has no `onGet`, preserve PVXS default GET behavior (return current value)

**Deliverables:**
- Modified `src/pvxs_sharedpv.cpp` with GET interception
- Updated `src/p4p/_p4p.pyx` (if Cython changes needed for new C++ signatures)
- Updated `src/p4p/server/raw.py` with `onGet` in Handler and `_WrapHandler`

**Requirements covered:** CPP-01, CPP-02, CPP-03, RAW-01, RAW-02

**Plans:** 2 plans

Plans:
- [x] 01-01-PLAN.md — C++ GetInterceptSource + Cython wiring + failing tests (Wave 1) — DONE 2026-03-27
- [x] 01-02-PLAN.md — Python raw layer: Handler.onGet + SharedPV.get decorator (Wave 2) — DONE 2026-03-27

---

## Phase 2: Backend Adapters, Tests, and Documentation

**Goal:** `onGet` works correctly in all supported backends (thread, asyncio, cothread) and is fully tested and documented.

**Scope:**
- `server/thread.py`: dispatch `onGet` through `WorkQueue` (same pattern as `put`)
- `server/asyncio.py`: dispatch `onGet` via `loop.call_soon_threadsafe()`, support `async def onGet`
- `server/cothread.py`: dispatch `onGet` via cothread callback
- Integration tests in `src/p4p/test/` using `RefTestCase` + `Server(isolate=True)` pattern:
  - GET triggers `onGet`
  - `op.done(value)` delivers correct value to client
  - `op.done(error=...)` delivers error to client
  - Handler without `onGet` is unaffected
  - No C extension reference leaks
- Sphinx docs: update `documentation/` with `onGet` handler API reference and hardware-read example

**Note (from research):** No changes to thread.py, asyncio.py, or cothread.py are needed. The `_exec` dispatch mechanism already routes onGet through all backends correctly. Phase 2 work is tests and documentation only.

**Deliverables:**
- New tests in `src/p4p/test/test_sharedpv.py`, `asynciotest.py`, `cothreadtest.py`
- Updated `documentation/server.rst` with `onGet` docs

**Requirements covered:** THR-01, THR-02, THR-03, ASIO-01, ASIO-02, CTH-01, TEST-01 through TEST-05, DOC-01, DOC-02

**Plans:** 2 plans

Plans:
- [x] 02-01-PLAN.md — Missing tests: test_onget_error + asyncio TestOnGet + cothread TestOnGet (Wave 1) — DONE 2026-03-27
- [x] 02-02-PLAN.md — Documentation: onGet automethod + hardware-read example in server.rst (Wave 1) — DONE 2026-03-27

---

## Milestone Summary

| Phase | Goal | Key Files | Status |
|-------|------|-----------|--------|
| 1 | C++ + raw Python GET interception | pvxs_sharedpv.cpp, _p4p.pyx, server/raw.py | COMPLETE (2/2 plans done) |
| 2 | Backend adapters + tests + docs | test_sharedpv.py, asynciotest.py, cothreadtest.py, server.rst | COMPLETE (2/2 plans done) |

**Total requirements:** 18 v1 requirements, all mapped
**Estimated phases:** 2 (coarse)

---
*Roadmap created: 2026-03-27*
*Updated: 2026-03-27 — Phase 1 complete (Plans 01-01 and 01-02 done)*
*Updated: 2026-03-27 — Phase 2 planned (Plans 02-01 and 02-02 created)*
*Updated: 2026-03-27 — Phase 2 complete (2/2 plans done, 18/18 requirements verified)*
