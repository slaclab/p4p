---
phase: 02-backend-adapters-tests-and-documentation
plan: 01
subsystem: testing
tags: [p4p, pvxs, SharedPV, onGet, asyncio, cothread, pytest]

# Dependency graph
requires:
  - phase: 01-cpp-and-raw-python-layer
    provides: GetInterceptSource C++ wiring, _WrapHandler.onGet dispatch, SharedPV.get decorator

provides:
  - test_onget_error in test_sharedpv.py::TestOnGet (RemoteError delivery via op.done(error=))
  - TestOnGet in asynciotest.py with async def onGet and await inside handler
  - TestOnGet in cothreadtest.py with cothread.Yield() inside handler

affects:
  - 02-backend-adapters-tests-and-documentation

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Self-contained test_onget_error: creates own StaticProvider+Server inside the test method for isolation"
    - "AsyncTest subclass TestOnGet: uses asyncSetUp/asyncTearDown pattern, async def onGet with await asyncio.sleep(0)"
    - "cothreadtest TestOnGet: follows same _sync/gc/assertSetEqual teardown convention as TestGPM"

key-files:
  created: []
  modified:
    - src/p4p/test/test_sharedpv.py
    - src/p4p/test/asynciotest.py
    - src/p4p/test/cothreadtest.py

key-decisions:
  - "test_onget_error uses isolated Server/provider (not setUp's) to avoid tearDown complexity with extra PVs"
  - "AsyncTest TestOnGet sets timeout=3.0 (not 1.0) to allow async yield round-trip"
  - "RemoteError added to asynciotest.py import for completeness even though not yet used in current tests"
  - "cothreadtest TestOnGet auto-skips when cothread absent — no try/except needed, bare import cothread at top handles it"

patterns-established:
  - "Isolated server tests: when test needs its own PV setup, create StaticProvider+Server inside the test body"
  - "Asyncio handler tests: async def onGet with at least one await to prove async dispatch"

requirements-completed: [THR-01, THR-02, THR-03, ASIO-01, ASIO-02, CTH-01, TEST-01, TEST-02, TEST-03, TEST-04, TEST-05]

# Metrics
duration: 2min
completed: 2026-03-27
---

# Phase 2 Plan 1: Backend Adapters Tests and Documentation Summary

**Three missing onGet test cases added: RemoteError delivery (THR-03), asyncio async handler (ASIO-01/ASIO-02), and cothread dispatch (CTH-01) — all 29 sharedpv+asyncio tests passing**

## Performance

- **Duration:** ~2 min
- **Started:** 2026-03-27T20:23:30Z
- **Completed:** 2026-03-27T20:25:13Z
- **Tasks:** 3
- **Files modified:** 3

## Accomplishments

- `test_sharedpv.py::TestOnGet` now has 4 tests (was 3); `test_onget_error` verifies `RemoteError` raised when `op.done(error=...)` called
- `asynciotest.py` gains `TestOnGet` with two tests covering asyncio dispatch and `async def onGet` with `await` — added to `__all__` so test_asyncio.py picks it up automatically
- `cothreadtest.py` gains `TestOnGet` with `cothread.Yield()` inside handler; auto-skips when cothread not installed

## Task Commits

1. **Task 1: Add test_onget_error to TestOnGet in test_sharedpv.py** - `f9919aa` (test)
2. **Task 2: Add TestOnGet class to asynciotest.py** - `c68d517` (test)
3. **Task 3: Add TestOnGet class to cothreadtest.py** - `bd26d5c` (test)

## Files Created/Modified

- `src/p4p/test/test_sharedpv.py` - Added `test_onget_error` method inside `TestOnGet` class (self-contained with own server)
- `src/p4p/test/asynciotest.py` - Added `RemoteError` import, `TestOnGet` to `__all__`, and `TestOnGet(AsyncTest)` class with two async test methods
- `src/p4p/test/cothreadtest.py` - Added `TestOnGet(RefTestCase)` class with `cothread.Yield()` handler and `test_onget_called`

## Decisions Made

- `test_onget_error` uses its own isolated `StaticProvider` + `Server` created inside the test method rather than reusing `self.sprov`/`self.server` — avoids complicating tearDown which checks weak refs on a specific set of objects
- `TestOnGet` timeout set to 3.0s in asynciotest.py to give room for the async yield round-trip under load
- `RemoteError` imported in asynciotest.py even though not used in current tests — keeps the import line consistent and ready for future error tests

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

- `testPVClose` in asynciotest.py has two identical `finally: sub.close() / await sub.wait_closed()` blocks, which caused an ambiguous `Edit` match when appending `TestOnGet`. Resolved by using the unique surrounding context (`await self.pv.close(destroy=True, sync=True)`) to identify the correct insertion point.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- All three backend-adapter test gaps are closed (THR-03, ASIO-01, ASIO-02, CTH-01)
- 29 tests passing in test_sharedpv.py + test_asyncio.py, no regressions
- Ready for Phase 2 Plan 2 (documentation update)

## Self-Check: PASSED

- FOUND: src/p4p/test/test_sharedpv.py
- FOUND: src/p4p/test/asynciotest.py
- FOUND: src/p4p/test/cothreadtest.py
- FOUND: .planning/phases/02-backend-adapters-tests-and-documentation/02-01-SUMMARY.md
- FOUND commit: f9919aa (test_onget_error)
- FOUND commit: c68d517 (asyncio TestOnGet)
- FOUND commit: bd26d5c (cothread TestOnGet)

---
*Phase: 02-backend-adapters-tests-and-documentation*
*Completed: 2026-03-27*
