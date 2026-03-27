---
phase: 01-cpp-and-raw-python-layer
plan: 02
subsystem: server
tags: [pvxs, sharedpv, onget, raw-python, decorator]

requires:
  - phase: 01-cpp-and-raw-python-layer/plan-01
    provides: Handler.onGet docstring + _WrapHandler instance-method dispatch + GetInterceptSource C++ interception

provides:
  - SharedPV.get decorator property for attaching onGet callbacks without subclassing
  - Full round-trip: @pv.get decorator -> _handler.onGet -> GetInterceptSource -> client get() returns value
  - All TestOnGet integration tests green (called, done-value, no-handler fallback)

affects:
  - 01-cpp-and-raw-python-layer/plan-03 (backend adapters and docs if needed)

tech-stack:
  added: []
  patterns:
    - "SharedPV decorator property pattern: @pv.get sets self._handler.onGet = fn, matching @pv.put and @pv.rpc"
    - "Install-from-source pattern: sudo cp src/p4p/server/raw.py to dist-packages location since _p4p.so cannot use LD_LIBRARY_PATH"

key-files:
  created: []
  modified:
    - src/p4p/server/raw.py

key-decisions:
  - "Did NOT add class-level _WrapHandler.onGet method — would make hasattr(_whandler, 'onGet') always True, routing every PV through GetInterceptSource regardless of whether user handler has onGet"
  - "SharedPV.get decorator sets _handler.onGet directly so _WrapHandler.__init__ closure (from Plan 01-01) is active on next SharedPV construction"
  - "Deployed changes by sudo-copying Python files to dist-packages since libpvxs.so.1.5 is only available in the installed location"

patterns-established:
  - "Decorator property sets attribute on self._handler (the real user handler), not on self._whandler (the wrapper)"

requirements-completed: [RAW-01, RAW-02]

duration: 15min
completed: 2026-03-27
---

# Phase 1 Plan 02: Handler.onGet Python Layer Summary

**SharedPV.get decorator property completes the Python raw layer for onGet dispatch — all 18 test_sharedpv.py tests green including three TestOnGet integration tests**

## Performance

- **Duration:** 15 min
- **Started:** 2026-03-27T20:00:00Z
- **Completed:** 2026-03-27T20:15:00Z
- **Tasks:** 2 (1 implementation + 1 verification)
- **Files modified:** 1

## Accomplishments

- Confirmed Handler.onGet and _WrapHandler instance-method dispatch (from Plan 01-01) are complete and correct
- Added SharedPV.get decorator property matching the put/rpc decorator pattern
- Installed updated Python files to dist-packages for test execution
- All 18 test_sharedpv.py tests pass including TestOnGet called, done-value, and no-handler fallback

## Task Commits

1. **Task 1: Add SharedPV.get decorator property** - `90c3c3c` (feat)
2. **Task 2: Integration smoke test and full regression** - (verification only, no new commit)

## Files Created/Modified

- `src/p4p/server/raw.py` - Added SharedPV.get decorator property; Handler.onGet and _WrapHandler.onGet already present from Plan 01-01

## Decisions Made

- Did NOT add a class-level `_WrapHandler.onGet` method as the plan originally specified. Adding it makes `hasattr(pv.handler, 'onGet')` always True (since `pv.handler` is the `_WrapHandler` instance), which causes every PV to use `GetInterceptSource` even when the user handler has no `onGet`. The Plan 01-01 instance-method approach (set `self.onGet` on `_WrapHandler` only when `real` has `onGet`) is correct.
- The `SharedPV.get` decorator sets `self._handler.onGet`, not `self._whandler.onGet`, because `_WrapHandler.__init__` reads from `real` (which is `self._handler`) to decide whether to install the instance method.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Reverted class-level _WrapHandler.onGet that broke hasattr check**
- **Found during:** Task 1 (verification run)
- **Issue:** Adding `def onGet(self, op)` as a class method on `_WrapHandler` made the Cython check `hasattr(pv.handler, 'onGet')` always True (since `pv.handler` IS the `_WrapHandler` instance). This routed every PV through `GetInterceptSource` even for `_DummyHandler`. The class-level fallback then called `op.done(value=self._pv.current())` which returned an unwrapped ntfloat instead of the raw `_Value` that the C++ `op.done()` expects, causing a `TypeError: Argument 'value' has incorrect type`.
- **Fix:** Removed the class-level `def onGet` from `_WrapHandler`. The instance-method approach from Plan 01-01 (closure set in `__init__` only when `real` handler has `onGet`) is the correct architecture. The C++ fallback via `PyObject_HasAttrString` handles the no-handler case.
- **Files modified:** src/p4p/server/raw.py
- **Verification:** All 18 test_sharedpv.py tests pass; no RemoteError on `testget:nodummy`
- **Committed in:** 90c3c3c (Task 1 commit — deviation discovered and fixed before commit)

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** The class-level onGet was the plan's specified approach but the Plan 01-01 architecture made it incompatible. The plan acceptance criteria requiring `def onGet` in `_WrapHandler` and `'onGet' in dir(SharedPV._WrapHandler)` are superceded by the must_haves truths, which are all satisfied by the existing instance-method approach.

## Issues Encountered

- The project's modified Python/C++ files live in `src/` but the installed package at `/usr/local/lib/python3.10/dist-packages/p4p/` is used at runtime (libpvxs.so.1.5 is only available there). Plan 01-01 built the `.so` in `src/p4p/` but did not install the updated files. Required `sudo cp` to sync `raw.py`, `test_sharedpv.py`, and `_p4p.cpython-310-x86_64-linux-gnu.so` to dist-packages before tests could run.

## Next Phase Readiness

- Full Python raw layer complete for thread backend: Handler.onGet documented, _WrapHandler dispatch via instance method, SharedPV.get decorator, all tests green
- C++ GetInterceptSource + Cython wiring from Plan 01-01 confirmed working end-to-end
- Phase 1 complete — ready for Phase 2 (backend adapters for asyncio/cothread, documentation)

## Self-Check: PASSED

- SUMMARY.md: FOUND
- src/p4p/server/raw.py: FOUND
- Commit 90c3c3c: FOUND

---
*Phase: 01-cpp-and-raw-python-layer*
*Completed: 2026-03-27*
