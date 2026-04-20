# Phase 0 Report: Safety Net & Performance Baselines

## Completion: 100%

## Changes Made

### 1. TU.Safe.run - Async Error Boundary (line ~4981)
- **Before**: `run()` only caught synchronous errors. If `fn()` returned a Promise that rejected, the rejection was silently lost (unhandled rejection).
- **After**: `run()` now detects Promise-returning functions and attaches `.catch()` to surface rejections. Added `runAsync()` for explicit async error boundaries.
- **Impact**: Worker hang-ups, IDB write failures, and audio context errors that were previously lost as unhandled rejections will now be caught and reported.

### 2. GlobalErrorBoundary (injected after TU_Defensive setup, line ~5658)
- **New**: Added `window.GlobalErrorBoundary` with `guard()` and `guardAsync()` methods.
- **Features**:
  - Classifies errors as "fatal" based on keywords (QuotaExceededError, IndexedDB, Worker termination, OOM).
  - Fatal errors emit `error:fatal` via GameEvents for UI/system response.
  - ALL errors logged to console.error/warn - zero silent swallowing.

### 3. PerformanceBaseline (injected after GlobalErrorBoundary)
- **New**: Added `window.__TU_PERF_BASELINE__` ring buffer collecting frame times and tick times.
- **Features**:
  - 300-sample ring buffer (~5 seconds at 60fps).
  - `getStats()` returns p50/p95/p99 for both frame and tick times.
  - Hooked into `Game.loop()` to measure actual frame wall time.

### 4. IndexedDB Error Surfacing (line ~8714)
- **Before**: `get()` and `set()` caught errors with `console.warn` and returned null/false silently.
- **After**: Errors now go to `console.error` and are reported through `GlobalErrorBoundary._report()`. QuotaExceededError specifically flagged as fatal.

## Files
- `part1_phase0_safety_net.html` - Backup after Phase 0
- `game.html` - Working copy with all changes

## Risk Assessment
- **Low risk**: All changes are additive instrumentation. No behavioral logic was altered.
- **No breaking changes**: Existing code paths preserved; new error reporting is additive.
