# Project State

**Last updated:** 2026-03-27
**Status:** MILESTONE COMPLETE — all 18 v1 requirements verified, 29 tests green

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-27)

**Core value:** A Python Handler subclass can define `onGet(self, op)` and have it called when a client executes `cxt.get()`, enabling demand-driven hardware reads.
**Current focus:** Milestone v1 complete — onGet handler fully implemented

## Current Phase

**Phase 2** — COMPLETE (2 plans done)
- Plan 1 (THR-01/02/03, ASIO-01/02, CTH-01, TEST-01-05): DONE — test_onget_error, asyncio TestOnGet, cothread TestOnGet, 29 tests green
- Plan 2 (DOC-01/DOC-02): DONE — server.rst onGet automethod + hardware-read example

## Phase Status

| Phase | Name | Status |
|-------|------|--------|
| 1 | C++ and Raw Python Layer | COMPLETE (2/2 plans done) |
| 2 | Backend Adapters, Tests, and Documentation | COMPLETE (2/2 plans done) |

## Key Technical Context

- **PVXS version:** pvxslibs 1.5.1 (installed at `/usr/local/lib/python3.10/dist-packages/pvxslibs/`)
- **GET interception point:** `ChannelControl::onOp()` → `ConnectOp::onGet()` in `pvxs/source.h`
- **No `SharedPV::onGet()`** — must intercept at ChannelControl/ConnectOp level before SharedPV's internal GET handler
- **ExecOp wrapping:** Already implemented as `ServerOperation` in `src/p4p/_p4p.pyx:824`
- **Existing pattern:** `attachHandler()` in `src/pvxs_sharedpv.cpp:17` sets up put/rpc/onFirstConnect/onLastDisconnect

## Codebase Map

See `.planning/codebase/` for full analysis (created 2026-03-27)
- ARCHITECTURE.md — component overview and data flow
- STACK.md — Python/C++/Cython stack details
- CONCERNS.md — technical debt, security issues, TODOs

## Decisions

- Implement within p4p C++ layer, not pvxslibs upstream (user needs this now)
- Reuse `ServerOperation` for GET op object (consistent with put/rpc)
- Skip Qt backend (already has known gaps)
- GetInterceptSource stores pvname explicitly — onSearch must compare by name string
- Custom source registered at iorder-1 (higher priority) to claim PV before StaticSource
- _WrapHandler.onGet added as instance method only when real handler has onGet attribute
- PVs with onGet NOT added to StaticSource — GetInterceptSource handles all channel operations
- onFirstConnect/onLastDisconnect do NOT fire for PVs using GetInterceptSource (Phase 1 known limitation)
- Class-level _WrapHandler.onGet must NOT be added — makes hasattr(_whandler, 'onGet') always True, routing every PV through GetInterceptSource
- SharedPV.get decorator sets _handler.onGet (the real user handler), not _whandler.onGet
- Deployed Python changes via sudo cp to dist-packages (libpvxs.so.1.5 only in installed location)
- onGet automethod placed between rpc and onFirstConnect — logical ordering: operations before lifecycle hooks
- Hardware-read example uses read_hardware_register() as application-defined placeholder to show the pattern
- test_onget_error uses isolated Server/provider inside test method (not setUp's server) to avoid tearDown weak-ref complications
- TestOnGet timeout=3.0 in asynciotest.py to allow async yield round-trip
- cothreadtest TestOnGet auto-skips when cothread absent via bare import guard at top of file

## Performance Metrics

| Phase | Plan | Duration | Tasks | Files |
|-------|------|----------|-------|-------|
| 01 | 01 | 8min | 2 | 5 |
| 01 | 02 | 15min | 2 | 1 |
| 02 | 01 | 2min | 3 | 3 |
| 02 | 02 | 5min | 1 | 1 |

## Last session

**Stopped at:** Phase 2 complete — all 18 v1 requirements done, milestone v1 achieved
**Timestamp:** 2026-03-27T20:40:00Z
