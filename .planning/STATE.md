# Project State

**Last updated:** 2026-03-27
**Status:** Phase 1, Plan 2 complete — RAW-01/RAW-02 implemented; Phase 1 complete

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-27)

**Core value:** A Python Handler subclass can define `onGet(self, op)` and have it called when a client executes `cxt.get()`, enabling demand-driven hardware reads.
**Current focus:** Phase 2 — Backend adapters, tests, and documentation

## Current Phase

**Phase 1** — COMPLETE (2 plans done)
- Plan 1 (CPP-01/CPP-02/CPP-03): DONE — GetInterceptSource, Cython wiring, _WrapHandler onGet
- Plan 2 (RAW-01/RAW-02): DONE — Handler.onGet docstring, SharedPV.get decorator, 18 tests green

## Phase Status

| Phase | Name | Status |
|-------|------|--------|
| 1 | C++ and Raw Python Layer | COMPLETE (2/2 plans done) |
| 2 | Backend Adapters, Tests, and Documentation | Pending |

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

## Performance Metrics

| Phase | Plan | Duration | Tasks | Files |
|-------|------|----------|-------|-------|
| 01 | 01 | 8min | 2 | 5 |
| 01 | 02 | 15min | 2 | 1 |

## Last session

**Stopped at:** Completed 01-02-PLAN.md (SharedPV.get decorator + full test suite green)
**Timestamp:** 2026-03-27T20:15:00Z
