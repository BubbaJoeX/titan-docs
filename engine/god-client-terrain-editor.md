# God Client Terrain Editor — User Guide

This document describes how to use the in-world terrain editing tools in **SwgGodClient** (Terrain dock + `GodClientTerrainEditor`). Source lives under:

`client/src/game/client/application/SwgGodClient/src/`

---

## Opening the editor

1. Launch the God Client and load into a scene with procedural terrain.
2. Open the **Terrain Editor** dock: use the **View** menu action **Terrain Editor** (with keyboard mnemonic **Te&rrain Editor**, see `ActionsWindow.cpp`), which toggles the dock visibility.
3. Confirm **Terrain File** shows a path or use **Refresh** / **Load** if the label reads “No terrain loaded”.

The 3D view (**GameWidget**) forwards mouse input to the terrain editor when a tool is active; keep **Brush Preview** enabled in **Visualization** while learning.

---

## Terrain file operations

| Control | Purpose |
|--------|---------|
| **Load...** | Open a `.trn` from disk (path updates the dock label). |
| **Save** / **Save As...** | Write the procedural terrain `.trn` **locally** only (autosave behaves the same). Does **not** notify connected game clients. |
| **Publish** | When logged into a session, uploads the terrain **from the on-disk `.trn`** you have configured (writes unsaved edits to that path first). Use this to distribute edits; use **Save** for local checkpoints between publishes. |
| **Refresh** | Resync dock state from the scene / template. |

---

## Height tools (brush)

Select one of **Raise**, **Lower**, **Flatten**, **Smooth**, **Noise**, or **Set Height**.

**Brush settings**

- **Size** — diameter of the brush in world units; updates the preview ring immediately when the slider moves.
- **Strength**, **Shape**, **Falloff**, **Feather** — propagate straight into `GodClientTerrainEditor` as you change sliders or combos (**Shape**: circle vs square; **Falloff**: linear, smooth, sharp, flat; **Feather**: edge softness toward the rim).

**How to paint**

1. Choose a height tool.
2. Click-drag on the ground in the 3D view (ray hits drive the brush; while moving the mouse, the dock uses **robust ground picking**—horizontal-plane and column fallbacks—so brush position and preview stay coherent when LOD rebuilds briefly drop mesh collision).
3. **SwgGodClient:** While you **hold** the mouse button, height (and other brush) **modifier data** still updates every sample, but **procedural mesh invalidation** is **deferred** until you **release**. That avoids repeatedly tearing down the terrain mesh mid-stroke (which used to make the whole layer look like it “phased out”). On **mouse-up**, the editor invalidates the stroke’s accumulated bounds and runs `flushTerrainChanges`, so the mesh then catches up to the edited heights. Camera **nudge** / DPVS refresh is still reserved for stroke end and other non-drag invalidates, not every tick while dragging.

**Flatten** blends terrain toward the **height at the point where the stroke starts** (first click), so it can raise or lower locally toward that plane. **Set Height** still uses the absolute target from the dock / tool configuration.

**Tip:** Negative world coordinates are supported; height keys use a floor grid plus interpolation so sub-meter samples still see edits.

---

## Shader and flora painting

- **Paint Shader** — selects a shader **family** from the **Shaders** tab: **Scene** list uses families already on the loaded terrain; **Global shaders** lists families discovered from scanned `.trn` files. When you pick a global row, the dock can **merge** that family (and its `.sht` children) into the live terrain **`ShaderGroup`** so painting uses real templates. If the scene already has the same **family id** but **different** child shader templates, the editor may **clone** the catalog family to a new unused id and switch your selection (console message) so existing terrain families are not overwritten. Live rendering samples `CityTerrainLayerManager::getModifiedShader` in `ClientProceduralTerrainAppearance_ClientChunk.cpp` (`createTileShader`); painted cells also refresh **priority** (family index) with **family id** so `ShaderCache` rows match the correct `.sht`.
- **Paint Flora** — **Flora** tab entries map to **family ids** from the active terrain’s `FloraGroup`. Live preview reads `getModifiedFlora`; static flora (collidable and non-collidable) applies the overlay during placement queries.

Ensure the intended family row is selected in the tab combo before painting (ids are **not** the same as combo row indices on all worlds). After picking a **global** catalog row, wait for any one-time mesh refresh if the dock merged new shader families into the scene.

---

## Dynamic (radial) flora and water

- **Place Radial** — paints radial-group overlay data that **dynamic** flora consumes. Registration mirrors shader/flora: `CityTerrainLayerManager::setExternalRadialModifierCallback` / `getModifiedRadial`, consumed from `ClientChunk::findDynamicFlora` (including ability to introduce radial coverage where baked maps were empty). Combo entries use **`RadialGroup` family ids** from the terrain.
- **Water** tab: set **water height**, optional **water shader**, then **Apply** or use **Place Water** brush mode as implemented in the dock.

---

## Advanced tools (Roads, Ribbons, Environment, Bitmap)

The **Advanced** tab (see `BaseTerrainDock.ui`) supports polylines and stamps.

### Roads and ribbons (polylines)

1. Choose **Road** or **Ribbon** (or **Begin Road** / **Begin Ribbon** if split).
2. Set **Width**, **Feather**, **Shader** (roads), and **Fixed heights** if needed.
3. **Add points** by **left-clicking** in the world. While the path is active you can also:
   - **Shift+click** — delete the nearest control point (within a pick radius tied to width).
   - **Ctrl+click** — drag the nearest point (move the mouse, **release** to drop; height follows terrain unless **Fixed heights** is on).
   - **Alt+click** — insert a new point on the closest segment.
4. **Finish** commits the polyline into editor state / `TerrainGenerator` layer creation (see checklist for export completeness).
5. **Cancel** discards the in-progress path.

A **polyline preview** (debug lines / control points) can be shown while editing if **Polyline preview** (or equivalent) is enabled in the dock.

### Environment zone

1. **Place Environment** / begin environment mode.
2. Pick **Environment family** and place polygon boundary points.
3. **Finish** / **Cancel** as above.

### Bitmap stamp

1. Choose **Stamp Bitmap** and a **bitmap** entry (family id aligned with terrain `BitmapGroup` where wired).
2. Adjust **rotation**, **scale**, and whether the stamp affects **height** and/or **shader**.
3. Apply at cursor per build (click or drag as implemented). **SwgGodClient:** dragged **Stamp Bitmap** strokes defer mesh invalidation the same way as height/shader brushes until **mouse-up**, then refresh.

---

## Region operations

1. **Select** (toggle tool, or **Select Region**) — **mousedown** anchors a corner and starts the rectangle; **drag** updates an **in-progress** yellow outline rubber-banded from anchor to cursor; **release** commits **min/max** bounds and clears the provisional overlay in favor of the committed selection rectangle.
2. **Copy** — samples heights into an internal clipboard (grid resolution scales with region size).
3. **Paste** — applies the clipboard heights into the **currently selected** region (uses live `GodClientTerrainEditor` modifications where applicable; dock undo entry may coexist with editor undo).
4. **Fill** — sets heights in the selection to the **Set Height** target / configured fill height.

Committed selection still renders the same yellow outline as during drag (`GodClientTerrainEditor::renderRegionSelectionOverlay`).

---

## Visualization

| Option | Purpose |
|--------|---------|
| **Wireframe Overlay** | Terrain tile–aligned mesh grid overlay (follows ground height). |
| **Chunk Grid** | Chunk-aligned grid; also forwards to `ClientProceduralTerrainAppearance::setShowChunkExtents` where supported (debug chunk boxes). |
| **Height Colors** | Height-tinted tile grid overlay (approximate false color by elevation). |
| **Brush Preview** | Brush circle or square at the cursor on terrain; toggles both the dock flag and `GodClientTerrainEditor::setBrushPreviewEnabled`; position tracks robust ground picks during move events. |

---

## Undo / history

**Undo** / **Redo** in `GodClientTerrainEditor` rewind **height** modifications recorded into the brush-stroke snapshots on the stacks. Shader, flora, radial overlay maps, water, etc. **are not rewound** by that path today—the paint data stays applied until cleared or superseded separately. Large sessions cap the undo stack (`MAX_UNDO_STROKES`). The Terrain dock maintains its **own** undo list for legacy height/water/UI operations where implemented.

---

## Technical notes (for operators)

- Live height edits route through **`CityTerrainLayerManager::getModifiedHeight`**, chaining the external height modifier registered in `GodClientTerrainEditor::install()`.
- Shader / flora / radial overlays route through **`getModifiedShader`**, **`getModifiedFlora`**, and **`getModifiedRadial`** on `CityTerrainLayerManager` respectively.
- **SwgGodClient mesh stability:** `rebuildShaderCacheFromGenerator` (e.g. after merging global shader families) rebuilds the client `ShaderCache` **without** `purgeChunks()`; old caches are deleted only after no chunk references them. `invalidateRegion` also marks procedural LOD dirty in God builds so the tree refills after invalidates.
- Do not include raw `<dinput.h>` in arbitrary God Client `.cpp` files; menu key scan codes should use constants or engine wrappers (see `GameWidget.cpp` and `clientDirectInput`).
- If you see STLport errors involving `_num_get.h`, God Client’s `FirstSwgGodClient.h` undefs conflicting Windows SAL / RPC macros before STL and Qt headers.

---

## Improvements and ideas

Recent work separates **disk saves** from **Publish** (server replication). Natural follow‑ups:

- **Region or layer‑scoped replication** — stream only invalidated chunks or exported layers instead of the full serialized `.trn` when networking supports it.
- **Publish UX** — progress bar, checksum confirmation, reconnect/retry queue if replication fails mid-transfer.
- **Staging names** — publish under a provisional template tree path before committing the planet binding.
- **Preflight validation** — run `prepare`/`write` checks and emit a concise error before any network traffic.

---

## Troubleshooting

| Symptom | Things to check |
|--------|------------------|
| No terrain / grey label | Scene has `TerrainObject`; use **Refresh**; confirm planet `.trn` path. |
| Brush does nothing | Tool not **None**; click hits terrain; try moving slightly—LOD may be rebuilding colliders. |
| Whole terrain or large patches **disappear** briefly while editing | **SwgGodClient:** Mid-stroke mesh invalidates are deferred until mouse-up; shader cache rebuild no longer purges the entire quad-tree. If it still happens, note which tool and whether **Refresh** / **Reload Terrain** was used. |
| Chunky updates / “frozen” mesh mid-stroke | **SwgGodClient:** Brush **mesh** updates mostly on **mouse-up** by design; the preview ring can move before the terrain mesh catches up. |
| Shaders/flora/radial don’t look painted | Confirm hooks in checklist; wrong **family id** vs combo row usually shows as painting “nothing” — use Refresh after `.trn` load and verify family combos repopulated. |
| Wrong family painted | Flora/radial/shader combos use **generator ids**; re-pick row after Refresh. |
| **Publish** refuses or warns | Requires a connected game session, a `.trn` already written via **Save** / **Save As**, and readable on-disk bytes. Console reports oversize (>32 MiB) or read/template-name failures. |

For development status and gaps, see **`GodClientTerrainEditor_Checklist.md`**.
