---
phase: 01-cpp-and-raw-python-layer
plan: 01
subsystem: server
tags: [pvxs, cython, cpp, sharedpv, onget, source]

requires: []
provides:
  - GetInterceptSource C++ struct in pvxs_sharedpv.cpp that intercepts GET via ConnectOp::onGet()
  - attachGetHandler(name, pv, handler) function returning shared_ptr<Source>
  - StaticProvider.add() uses custom source for PVs whose handler has onGet
  - Server.__init__ registers GetInterceptSource sources at higher priority than StaticSource
  - _WrapHandler.onGet dispatch in server/raw.py (conditional on real handler having onGet)
  - Handler base class documents onGet(self, pv, op) signature
  - Three green TestOnGet tests covering called, done-value, and no-handler fallback
affects:
  - 01-cpp-and-raw-python-layer/plan-02 (backend adapters — onGet dispatch through WorkQueue)

tech-stack:
  added: []
  patterns:
    - "Custom PVXS Source pattern: GetInterceptSource bypasses SharedPV::attach() to intercept GET at ConnectOp::onGet() level"
    - "Conditional _WrapHandler dispatch: instance method onGet added only when real handler has onGet attribute"
    - "StaticProvider dual registration: custom sources stored in _get_src_vec/names, registered to Server separately"

key-files:
  created: []
  modified:
    - src/pvxs_sharedpv.cpp
    - src/p4p.h
    - src/p4p/_p4p.pyx
    - src/p4p/server/raw.py
    - src/p4p/test/test_sharedpv.py

key-decisions:
  - "GetInterceptSource stores pvname explicitly — needed because onSearch must compare by name string, not SharedPV identity"
  - "Custom source registered at iorder-1 (higher priority than StaticSource) to claim the PV name before StaticSource does"
  - "onGet added to _WrapHandler as an instance method only when real handler has onGet — avoids always-true hasattr check"
  - "PVs with onGet are NOT added to StaticSource — GetInterceptSource handles all channel operations (GET, PUT, RPC, Subscribe)"
  - "CPP-03 fallback: PyObject_HasAttrString(handler, 'onGet') in C++ — if false, calls gop->reply(pv.fetch())"
  - "Pre-existing Cython 0.29 bug fixed: except+ nogil changed to nogil except+ in extern declaration"

patterns-established:
  - "onGet wiring pattern: attachGetHandler(name, pv.pv, pv.handler) called in StaticProvider.add() when hasattr(pv.handler, 'onGet')"
  - "Source registration: _get_src_vec/names in StaticProvider; Server iterates them after adding StaticSource"

requirements-completed: [CPP-01, CPP-02, CPP-03]

duration: 8min
completed: 2026-03-27
---

# Phase 1 Plan 01: C++ GetInterceptSource and Cython wiring Summary

**GetInterceptSource custom PVXS Source with ConnectOp::onGet() interception, Cython StaticProvider wiring, and _WrapHandler dispatch enabling Python onGet callbacks for server GET operations**

## Performance

- **Duration:** 8 min
- **Started:** 2026-03-27T19:41:13Z
- **Completed:** 2026-03-27T19:50:12Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments

- GetInterceptSource C++ struct intercepts GET operations via ConnectOp::onGet() without calling SharedPV::attach(), enabling Python-level onGet callbacks
- Cython StaticProvider.add() and Server.__init__ wiring registers custom sources at higher priority so they claim PV names before StaticSource
- _WrapHandler.onGet dispatch in server/raw.py correctly routes GET ops to user handler via WorkQueue (_exec), with fallback to current value when no onGet

## Task Commits

1. **Task 1: Write failing TestOnGet tests (Wave 0)** - `61fa903` (test)
2. **Task 2: Implement GetInterceptSource in C++ and wire Cython** - `6e1bdef` (feat)

## Files Created/Modified

- `src/pvxs_sharedpv.cpp` - Added GetInterceptSource struct and attachGetHandler() function
- `src/p4p.h` - Added attachGetHandler declaration
- `src/p4p/_p4p.pyx` - Added attachGetHandler extern, StaticProvider _get_src_vec/_get_src_names fields and add() logic, Server.__init__ loop to register custom sources
- `src/p4p/server/raw.py` - Added onGet to Handler base class and conditional onGet dispatch to _WrapHandler
- `src/p4p/test/test_sharedpv.py` - Added TestOnGet class with three tests

## Decisions Made

- GetInterceptSource stores pvname explicitly because onSearch must match by name string
- Custom source registered at iorder-1 (higher priority) so it claims the PV before StaticSource
- _WrapHandler.onGet added as instance method only when real handler has onGet — avoids hasattr always returning True after adding the method unconditionally
- PVs with onGet are NOT added to StaticSource — the custom source handles all operations
- CPP-03 fallback handled in C++ via PyObject_HasAttrString check, calling gop->reply(pv.fetch()) when no onGet

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Fixed pre-existing Cython 0.29 syntax error in _p4p.pyx**
- **Found during:** Task 1 (build attempt)
- **Issue:** Line 45 had `except+ nogil` which is invalid in Cython 0.29.34; Cython 3.x syntax only
- **Fix:** Changed to `nogil except+` which is the Cython 0.29 compatible form
- **Files modified:** src/p4p/_p4p.pyx
- **Verification:** Cython compilation succeeds, `_p4p.cpp` generated without error
- **Committed in:** `61fa903` (part of Task 1 commit)

**2. [Rule 1 - Bug] Revised StaticProvider.add() check to use _WrapHandler's dynamic onGet**
- **Found during:** Task 2 (first test run, "Get not supported" RemoteError)
- **Issue:** Plan put `onGet` unconditionally on `_WrapHandler`, making `hasattr(pv.handler, 'onGet')` always True even for handlers without onGet. The AttributeError fallback in onGet was hitting because _real didn't have onGet, not because _WrapHandler lacked it.
- **Fix:** Add `onGet` to `_WrapHandler` as an instance method only when `real` handler has `onGet`, using a closure. This makes `hasattr(pv.handler, 'onGet')` correctly reflect the real handler's capability.
- **Files modified:** src/p4p/server/raw.py
- **Verification:** `test_onget_no_handler` passes (no custom source for DummyHandler), `test_onget_called` and `test_onget_done_value` pass
- **Committed in:** `6e1bdef` (part of Task 2 commit)

**3. [Rule 1 - Bug] Fixed data::Value namespace in C++ (Cython namespace not valid in C++)**
- **Found during:** Task 2 (first C++ compile attempt)
- **Issue:** Plan code used `data::Value{}` and `data::Value&&` — these use Cython's pxd import names which don't exist in C++; `using namespace pvxs` in p4p.h means just `Value{}` and `Value&&`
- **Fix:** Replaced `data::Value{}` with `Value{}` and `data::Value&&` with `Value&&` in pvxs_sharedpv.cpp
- **Files modified:** src/pvxs_sharedpv.cpp
- **Verification:** C++ compilation succeeds without errors
- **Committed in:** `6e1bdef` (part of Task 2 commit)

---

**Total deviations:** 3 auto-fixed (1 blocking, 2 bugs)
**Impact on plan:** All auto-fixes necessary for correctness and compilation. No scope creep.

## Issues Encountered

- StaticSource.add() only accepts SharedPV (no shared_ptr<Source> overload) — addressed by storing custom sources separately in _get_src_vec/names on StaticProvider and registering them directly via Server::addSource() in Server.__init__

## Next Phase Readiness

- C++ GET interception and raw Python layer complete for thread backend
- Plan 02 can implement WorkQueue dispatch for asyncio/cothread backends and documentation
- onFirstConnect/onLastDisconnect do not fire for PVs using GetInterceptSource (known limitation; acceptable for Phase 1 scope)

---
*Phase: 01-cpp-and-raw-python-layer*
*Completed: 2026-03-27*
