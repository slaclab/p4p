# p4p onGet Handler

## What This Is

An extension to the p4p Python library that exposes `onGet` as a callable Handler callback, allowing Python server code to intercept client `get()` operations and perform hardware reads (or any demand-driven action) before returning a value. Currently, GET operations are handled entirely within PVXS's C++ layer with no Python hook.

## Core Value

A Python Handler subclass can define `onGet(self, op)` and have it called — with a live operation object — whenever a client executes `cxt.get()` on the PV.

## Requirements

### Validated

- ✓ SharedPV server with onFirstConnect / onLastDisconnect / put / rpc callbacks — existing
- ✓ WorkQueue-based dispatch to thread/asyncio/cothread backends — existing
- ✓ ExecOp wrapped as ServerOperation (op.done(), op.error()) — existing
- ✓ ConnectOp::onGet() exists in PVXS C++ source.h API — existing

### Validated (Phase 2 — 2026-03-27)

- ✓ Python Handler interface gains an `onGet(self, op)` method
- ✓ `op` passed to `onGet` is a `ServerOperation` (same interface as put/rpc ops)
- ✓ Handler can call `op.done(value)` to respond or `op.done(error=...)` to fail the GET
- ✓ onGet dispatched through WorkQueue in thread backend (non-blocking network threads)
- ✓ onGet dispatched into asyncio event loop in asyncio backend
- ✓ onGet dispatched via cothread in cothread backend
- ✓ If handler has no `onGet` method, behavior is unchanged (default value returned)
- ✓ Tests cover the full roundtrip: client get() → onGet → op.done(value)
- ✓ Documentation updated in Sphinx docs

### Out of Scope

- Changing PVXS upstream (pvxslibs) — we implement within p4p's own C++/Cython layer
- Qt backend onGet support — Qt backend is already missing monitor `request=` support; scope creep
- DynamicProvider onGet — DynamicProvider has different semantics; separate concern
- Auto-polling or subscription-driven hardware reads — explicitly excluded by user requirement

## Context

p4p wraps the PVXS C++ library (pvxslibs 1.5.1 installed). The C++ `SharedPV` class exposes `onPut()` and `onRPC()` callbacks but has no `onGet()`. GET operations are handled internally by PVXS — it clones and returns the last `post()`ed value with no Python code invoked.

The lower-level `ConnectOp::onGet()` exists in pvxs/source.h and is used by p4p's gateway (pvxs_gw.cpp). `ExecOp` is already wrapped as `ServerOperation` in `_p4p.pyx`. The implementation requires intercepting GET at the `ChannelControl`/`ConnectOp` level before SharedPV's internal handler takes over.

User's application subclasses `p4p.server.thread.Handler` and needs `onGet` to perform on-demand hardware reads so that `cxt.get()` always returns a live hardware value without polling.

## Constraints

- **Compatibility**: Must not break existing Handler subclasses that don't define `onGet` — backward compatible
- **Threading**: GET callbacks arrive on pvxs network threads; must dispatch to WorkQueue before calling Python
- **PVXS API**: Cannot modify pvxslibs headers; work within `ConnectOp::onGet()` + `ChannelControl::onOp()` API
- **Python versions**: Must work for Python 2.7 and Python 3.x (project still carries py2 compat shims)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Implement in p4p C++ layer, not pvxs upstream | User needs this now; upstream change requires separate pvxs PR + release cycle | Done — GetInterceptSource in pvxs_sharedpv.cpp |
| Reuse ServerOperation for the GET op object | ExecOp wrapping already exists; consistent API with put/rpc | Done — ServerOperation_wrap() reused |
| Skip Qt backend | Already has known gaps; not user's use case | Done — out of scope confirmed |
| No backend changes needed for dispatch | _exec() already routes all callbacks through all backends | Done — Phase 2 validated by tests |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-03-27 — Milestone v1 complete (Phase 2 verified)*
