# Phase 3 Report: Storage Resilience & Worker Rebuild

## Completion: 100%

## Changes Made

### 1. StorageAdapter with LS -> IDB Degradation (line ~5150)
- **Before**: Basic `{ get, set, remove }` object wrapping `localStorage` with silent error swallowing. QuotaExceededError was caught and ignored, causing "zombie saves" - the game thought it saved but data was lost.
- **After**: Full `StorageAdapter` with:
  - `lsAvailable` property - detects localStorage availability at startup
  - `quotaExceeded` flag - tracks if quota has been exceeded
  - **Explicit error reporting** for QuotaExceededError via GlobalErrorBoundary
  - **Automatic IDB fallback** for critical save data when LS quota is exceeded
  - `getAsync(k)` / `setAsync(k, v)` - async methods that try LS first, then IDB
  - Returns `{ lsOk, idbOk }` from async writes for caller to handle
- **Impact**: Save data is never silently lost. Quota errors are visible and trigger graceful degradation.

### 2. BlockRegistry with Palette Mapping (after BLOCK definition)
- **Before**: Block IDs were hardcoded and frozen (0-161). Patches that added new blocks used blind scanning for unused IDs. If patch load order changed, IDs shifted, corrupting old saves.
- **After**: `window.BlockRegistry` provides:
  - `getId(name)` / `getName(id)` - bidirectional lookup
  - `registerDynamic(name)` - assigns stable IDs (starting at 162) for patch-added blocks
  - `toPalette()` - generates `{ name: id }` map for save file headers
  - `buildRemapFromPalette(savedPalette)` - generates `Int32Array` remap table from old save to current IDs
  - Palette is now embedded in every save (`payload.palette`)
  - `applyToWorld()` checks palette and applies remap if IDs have changed
- **Impact**: Old saves from different patch load orders will have their block IDs correctly remapped on load. No more "ghost blocks" or world corruption.

### 3. Worker Generation Timeout (WorldWorkerClient.generate)
- **Before**: If the Worker hung during `generate()`, the Promise never resolved. Game was permanently stuck on loading screen with no error.
- **After**: 30-second timeout on world generation Promise. If exceeded:
  - Explicit error logged: `'Worker generation timeout (30s)'`
  - Promise rejects, allowing the existing fallback path (`_origGenerate.call(this, progressCb)`) to run on main thread
  - `clearTimeout` on successful resolve to prevent false positives
- **Impact**: Eliminates permanent loading screen hang from Worker issues.

## Files
- `part4_phase3_storage_worker.html` - Backup after Phase 3
- `game.html` - Working copy with all changes

## Risk Assessment
- **Medium risk**: StorageAdapter changes the sync API to expose errors that were previously swallowed. Existing code that relied on silent failures may need adjustment.
- **BlockRegistry remap**: Conservative - returns null (no remap) if all IDs match, avoiding unnecessary processing.
- **Worker timeout**: Fallback to main thread generation is already an existing code path.
