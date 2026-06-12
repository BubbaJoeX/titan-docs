# God Client Terrain Editor — Implementation Checklist

This tracks what the Terrain dock + `GodClientTerrainEditor` are intended to do, what is implemented in-tree, and what still needs work. Paths use the repo layout under `client/src/…` unless noted otherwise.

---

## Core architecture

| Item | Status | Notes |
|------|--------|--------|
| `GodClientTerrainEditor` singleton (`install` / `remove`) | **Done** | `game/client/application/SwgGodClient/src/GodClientTerrainEditor.cpp` |
| `TerrainDock` + `BaseTerrainDock.ui` (Qt3) | **Done** | Dock registered from `MainFrame` / `ActionsWindow` |
| `GameWidget` input + preview forwarding | **Done** | Mouse to dock / editor; avoid `#include <dinput.h>` in God Client TUs |
| External **height** modifier → `CityTerrainLayerManager` | **Done** | `getModifiedHeight` chains callback; used from procedural terrain appearance / chunks |
| External **shader** modifier API | **Done** | `setExternalShaderModifierCallback`, `getModifiedShader` |
| External **flora** modifier API | **Done** | `setExternalFloraModifierCallback`, `getModifiedFlora` |
| External **radial** modifier API | **Done** | `setExternalRadialModifierCallback`, `getModifiedRadial` — God Client registers on `install()` |
| Shader / flora / radial callbacks **used during terrain render** | **Done** | `ClientChunk::createTileShader` samples `getModifiedShader`; static flora queries apply `getModifiedFlora`; **`findDynamicFlora`** applies `getModifiedRadial` after sampling radial maps (including paint where base family was 0). |

---

## Live editing: invalidation + picking

| Item | Status | Notes |
|------|--------|--------|
| Rolling **localized** mesh invalidation during brush stroke | **Done** | `invalidateTerrainMeshesForLiveBrushSample`: throttled (`REALTIME_INVALIDATION_INTERVAL`), unions last interim invalidate center with current sample so fast drags don’t leave gaps; avoids growing the invalidate rect to the **entire** stroke every tick (reduces chunk-queue thrash / DPVS glitches). **SwgGodClient only:** while `m_brushStrokeActive`, that path **defers** `invalidateRegion` calls (`shouldDeferTerrainMeshInvalidationForGod`) so the procedural mesh is not torn down every sample; modifiers / dirty rect still accumulate. |
| **Full stroke AABB** invalidation on stroke end | **Done** | `endBrushStroke` uses accumulated `m_dirtyRegion*` then `flushTerrainChanges` (flush also skips mid-stroke invalidates when the same God defer guard is active). |
| Interim stroke: **no** per-tick `nudgeGodClientCameraToRefreshDpvs` | **Done** | Nudge reserved for non-stroke invalidates and stroke finalize to reduce camera jitter. |
| Dock → editor **live** brush params | **Done** | Size, strength, shape, falloff, feather sliders/combos push to editor setters immediately (not only `syncGodClientEditorBrushSettings`). |
| Robust **ground pick** during mouse move (`pickTerrainGroundForLiveEdit`) | **Done** | `TerrainDock::handleMouseMove`: plane + vertical probe fallbacks seeded from collide / cached Y / player height so brush preview and `continueBrushStroke` keep valid XZ when mesh collision flakes during LOD rebuild. Press still uses strict `getTerrainPositionFromScreen` before starting a stroke. |

---

## Mesh visibility + shader cache (God Client)

| Item | Status | Notes |
|------|--------|--------|
| Avoid **whole-terrain** blanking when rebuilding `ShaderCache` | **Done** | `ClientProceduralTerrainAppearance::rebuildShaderCacheFromGenerator` **no longer** calls `purgeChunks()` (which cleared the quad-tree and made the whole layer vanish). Rebuild allocates a new `ShaderCache`, full-map `invalidateRegion` drives leaf swap via existing worker + `replaceLeafChunkInPlace`. |
| Safe **deferred** delete of replaced `ShaderCache` | **Done** | Old cache pointers are queued in `m_deferredShaderCacheDeletes`; `serviceDeferredShaderCacheDeletes()` (from `alter()` after `retrieveCompletedChunkCreationRequests`) deletes each entry only when **no** `ClientChunk` still references it (`anyClientChunkReferencesShaderCache` + `ClientChunk::referencesShaderCache`). |
| God: **LOD dirty** on invalidate | **Done** | `invalidateRegion` calls `m_levelOfDetail->setDirty(true)` when `Game::isGodClient()` so refills schedule promptly after invalidate bursts (`ClientProceduralTerrainAppearance_LevelOfDetail.cpp`). |
| `calculateLod` vs `m_lockTerrainLevelOfDetail` | **Documented** | Comment in `ClientProceduralTerrainAppearance::calculateLod`: lock skips LOD selection during procedural rebuild; God stroke deferral reduces overlapping invalidate storms. |

---

## Height editing

| Item | Status | Notes |
|------|--------|--------|
| Tools: Raise, Lower, Flatten, Smooth, Noise, Set Height | **Done** | `ToolMode` enum |
| Brush shape / falloff | **Done** | Circle/square; linear/smooth/sharp/flat |
| Integer grid samples + bilinear lookup for sub-meter | **Done** | Reduces “wrong cell” / plateau artifacts |
| Undo / redo for brush strokes (**height samples only**) | **Partial** | `BrushStroke::modifications` drives undo/redo today; **`shaderModifications` / `floraModifications` structs exist but are not populated** — shader/flora/radial/water overlay maps **do not** rewind with editor Undo. |

---

## Shader / flora / water / radial

| Item | Status | Notes |
|------|--------|--------|
| Shader list + paint tool | **Done** | Live preview consumes `getModifiedShader` in `ClientChunk::createTileShader`; after overlay paint, **`chooseShader(fid)`** refreshes `ShaderGroup::Info::priority` with `familyId` so `ShaderCache::getTextures` indexes the correct `.sht` row (priority = family index). Nearest-painted-dab linkage uses large world link radius (shared constant band with flora/radial). |
| **Global** shader catalog → live `ShaderGroup` | **Done** | `TerrainDock::ensureLiveTerrainShaderFamilyForPaint` copies family + children from catalog `.trn` via `SamplerProceduralTerrainAppearanceTemplate` / `terrainDockCopyFamilyIntoShaderGroup`; on id collision with **different** child templates, clones to a free family id and updates selection; calls `rebuildShaderCacheFromGenerator` + invalidates. Scene list path unchanged when family already exists. |
| Flora list + paint | **Done** | `getModifiedFlora` via **`getModifiedFloraInternal`** (nearest dab within link radius); dock passes **`FloraGroup::getFamilyId(i)`**, not combo index. |
| Water boundary / height UI + place-water brush path | **Done** | Per `TerrainDock`; editor `placeWaterBrushDab` / local water table helpers |
| Radial group UI + **Place Radial** tool | **Done** | `modifyRadialPaint` + `RadialModificationMap`; **`getModifiedRadial`**; dock uses **`RadialGroup` family ids** |
| `exportModificationsToLayer`: radial paint → layer | **Not done** | Height/shader/flora layers exist in export; radial affector analogue not wired (would need TerrainGenerator parity). |

---

## Region selection UI

| Item | Status | Notes |
|------|--------|--------|
| Commit selection on mouse up | **Done** | `m_region*` + `GodClientTerrainEditor::setRegionSelection` |
| **Rubber-band** overlay while dragging | **Done** | `TerrainDock::updateFrame` renders `renderRegionSelectionOverlay` from anchor vs `m_regionDragCur*` when `m_regionDragActive` |

---

## Advanced: roads, ribbons, environment, bitmaps

| Item | Status | Notes |
|------|--------|--------|
| Polyline state (roads vs ribbons) | **Done** | `ActivePolyline`, `ControlPoint` |
| `AffectorRoad` / `AffectorRibbon` creation hooks | **Done** (in editor) | **Verify** layers land in live `TerrainGenerator` and serialize to `.trn` |
| Environment polygon zone | **Done** (structure) | **Verify** boundary + affector wiring + export |
| Bitmap stamp (height/shader) | **Done** (structure) | **Verify** IFF/asset load and application |
| Polyline preview rendering | **Done** | `renderPolylinePreview()` from `TerrainDock::updateFrame` when enabled |
| Advanced tab UI + signal wiring | **Done** | `BaseTerrainDock.ui` + `TerrainDock::initializeUI()` |
| Polyline shader combo family ids | **Done** | `m_polylineShaderFamilyIds` (row index ≠ id guard) |

---

## File I/O and TerrainGenerator

| Item | Status | Notes |
|------|--------|--------|
| Load / Save / Save As `.trn` | **Done** (UI) | **Regression-test** with complex layers after road/ribbon export |
| **Publish** (procedural `.trn` → live session) | **Done** | `TerrainDock::onPublishTerrainToLiveServer` uploads the **on-disk** file (flushes edits first via `writeCurrentTerrainTemplateToFile` when dirty). **Save** / autosave are **local-only** — no implicit network push after write. |
| `exportModificationsToLayer` / layer tracking | **Done** (code present) | Height/shader/flora branches; radial export gap above |
| Polyline import/export (e.g. `.poly`) | **Done** (if slots implemented) | Test round-trip |

---

## Build / toolchain hygiene (God Client)

| Item | Status | Notes |
|------|--------|--------|
| STLport `_num_get.h` / SAL `__in` conflict | **Fixed** | `FirstSwgGodClient.h` undefs SAL + `small` around STL/Qt |
| DirectInput / `dinput.h` in `GameWidget.cpp` | **Fixed** | Use numeric scan code constant; `clientDirectInput` includes DI safely in its own lib |
| `QSpinBox` includes for new UI | **Done** | `TerrainDock.cpp` includes `qspinbox.h` |
| Qt3 `BaseTerrainDock.ui` — no illegal XML comments in widgets | **Fixed** | Earlier uic corruption issue |

---

## Product / parity goals (“Noggit-like”)

| Item | Priority | Notes |
|------|----------|--------|
| Reliable **live** shader preview | High | Tile blend poles consult `getModifiedShader`. |
| Reliable **live** flora preview | High | Static paths + **`getModifiedFlora`** nearest-dab resolver. |
| Reliable **live** dynamic (radial) flora preview | Medium | **`getModifiedRadial`** in `findDynamicFlora`. |
| Multi-select / edit control points (move/delete/insert) | Medium | **Done (UX)** — Road/ribbon: **Shift/Ctrl/Alt** gestures (see user guide). |
| Chunk-local vs world-space clarity | Medium | Brush/paint keys are **world** meters; mesh regen = rolling invalidates + stroke finalize + `CityTerrainLayerManager` hooks. |
| Server authority / replication | Low (typical) | Operators keep working locally; **Publish** explicitly replicates the current disk `.trn` to connected clients. Ordinary **Save** / autosave do not replicate. |
| Performance profiling on large planets | Medium | Many `getModified*` scans over paint maps on hot paths; profile if needed. |

---

## Suggested verification pass

1. Build **SwgGodClient** (Debug and Release).
2. Height: small brush, negative X/Z, confirm smooth height; drag long stroke — **SwgGodClient:** mesh regen should **not** flash the whole terrain away mid-drag; release and confirm full refresh + height catch-up.
3. Shader / flora / radial: pick non-trivial family ids, paint, confirm visible change after chunk rebuild.
4. Region select: drag rectangle, confirm yellow outline **during** drag and after release.
5. Road: place 3+ points, Finish, save `.trn`, reload scene / cold start, confirm road persists.
6. Ribbon / environment / stamp: same persistence smoke test.
7. Undo: confirm **height** strokes undo; confirm shader/flora/radial **do not** undo via editor (expected gap until stroke captures overlay samples).
8. **Publish:** after edits, confirm **Save** alone does **not** change remote terrain; confirm **Publish** with a joined client observes updated procedural terrain once replication completes.

---

## Improvements and ideas

Following the explicit **Publish** workflow (disk save vs distributing to the session):

- **Incremental replication** — send height/shader deltas for an AOI instead of chunked full `.trn` uploads when protocol support exists.
- **Publish telemetry** — last success time, byte count, checksum or server ack, visible progress/retry UI.
- **Conflict handling** — optional lock/version when multiple GMs edit the same planet concurrently.
- **Dry-run / diff** — compare in-memory generator vs disk file vs last published snapshot without transmitting.

---

## Key source files

- `client/src/game/client/application/SwgGodClient/src/GodClientTerrainEditor.{h,cpp}` — brush stroke, `shouldDeferTerrainMeshInvalidationForGod`, `flushTerrainChanges`, `invalidateTerrainMeshesForLiveBrushSample`
- `client/src/game/client/application/SwgGodClient/src/TerrainDock.{h,cpp}` — global shader catalog, `ensureLiveTerrainShaderFamilyForPaint`, `rebuildShaderCacheFromGenerator` caller
- `client/src/game/client/application/SwgGodClient/src/shared/ui/BaseTerrainDock.ui`
- `client/src/game/client/application/SwgGodClient/src/GameWidget.{h,cpp}`
- `client/src/game/client/application/SwgGodClient/src/FirstSwgGodClient.h`
- `client/src/engine/client/library/clientTerrain/src/shared/core/CityTerrainLayerManager.{h,cpp}`
- `client/src/engine/client/library/clientTerrain/src/shared/appearance/ClientProceduralTerrainAppearance.cpp` — `rebuildShaderCacheFromGenerator`, deferred cache delete, `calculateLod` notes
- `client/src/engine/client/library/clientTerrain/src/shared/appearance/ClientProceduralTerrainAppearance_LevelOfDetail.cpp` — `invalidateRegion` (God LOD dirty)
- `client/src/engine/client/library/clientTerrain/src/shared/appearance/ClientProceduralTerrainAppearance_ClientChunk.cpp` — `createTileShader`, `referencesShaderCache`

---

## User documentation

See **`GodClientTerrainEditor.md`** for operator-facing steps and troubleshooting.
