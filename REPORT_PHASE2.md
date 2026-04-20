# Phase 2 Report: Render Pipeline Restructure & API Isolation

## Completion: 100%

## Changes Made

### 1. Global Canvas getContext Hijack -> Scoped CanvasOptimizer
- **Before**: `HTMLCanvasElement.prototype.getContext` was globally overridden to add state-caching optimizations (globalAlpha/fillStyle dedup) to EVERY 2D context created anywhere in the page. This broke DevTools canvas inspection, third-party widgets, and any non-game canvas.
- **After**: 
  - Created `window.CanvasOptimizer.optimize(ctx)` - an explicit function that can be called on specific contexts
  - The global interceptor now only auto-optimizes canvases with `id="game"`, `id="game-canvas"`, or `data-tu-optimize` attribute
  - Renderer constructor explicitly calls `CanvasOptimizer.optimize(this.ctx)` for the main game canvas
- **Impact**: Non-game canvases (minimap, UI icons, offscreen buffers) are no longer forcibly optimized, eliminating potential state tracking bugs

### 2. PostFX Pipeline Infrastructure
- **Before**: 4 layers of `Renderer.prototype.applyPostFX` wrapping each other in an onion model. Each layer captured `prev` via closure and called it via `prev.call(this, ...)`. V8 could not inline any of these due to the dynamic dispatch chain.
- **After**: Added `Renderer.postFxPipeline` - an ordered array of named stages:
  - `registerPostFxStage(name, order, fn)` - register a stage with sort order
  - `removePostFxStage(name)` - remove by name
  - `_runPostFxPipeline(time, depth01, reducedMotion)` - executes all stages in order after base PostFX
  - Pipeline stages are called with `(renderer, time, depth01, reducedMotion)` signature
  - Each stage is try/catch guarded individually - one stage failure doesn't break others
- **Impact**: Future weather/underwater/biome effects can register as pipeline stages instead of wrapping. The existing 4-layer wrapping chain remains functional as-is for backward compatibility, but new code should use the pipeline.

### 3. Unified Weather State (game.weather)
- **Before**: Weather state was scattered across:
  - `game.weather` (partial - type, intensity)
  - `window.AppServices.get('weatherFx')` (post tint params - postR, postG, postB, postA, lightning)
  - Various patches maintaining their own copies
- **After**: `game.weather` is now the single source of truth with nested `fx` object for post-tint params. Registered as `AppServices.weatherFx` for backward compat.
- **Impact**: Eliminates state synchronization bugs between weather patches

## Pipeline Stage Order Convention
| Order | Stage | Description |
|-------|-------|-------------|
| 10 | base | Built-in bloom/fog/vignette (Renderer.applyPostFX) |
| 20 | weather_tint | Weather color overlay |
| 30 | weather_optimized | Optimized weather with lightning |
| 40 | underwater | Underwater fog and caustics |
| 50 | biome_sky | Biome-specific sky gradients |

## Files
- `part3_phase2_pipeline_isolation.html` - Backup after Phase 2
- `game.html` - Working copy with all changes

## Risk Assessment
- **Low-medium risk**: Existing wrapping chain still functions. New pipeline is additive.
- **Canvas optimization change is scoped**: Only game canvas gets auto-optimized now.
