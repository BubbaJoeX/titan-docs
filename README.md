# Titan documentation

Published documentation for **Bubba Joe's flavor of Star Wars Galaxies (Titan)**.

| | |
|--|--|
| **This repo** | [github.com/BubbaJoeX/titan-docs](https://github.com/BubbaJoeX/titan-docs) |
| **Game code** | Separate repos: `client`, `dsrc`, `src` (siblings under the Titan monorepo when using the full tree) |
| **Feature index** | [FEATURES.md](https://github.com/BubbaJoeX/titan/blob/main/FEATURES.md) in the monorepo root (code paths and touchpoints) |

Paths below are relative to **this repository**. In the monorepo, the same tree is often mirrored at `docs/` next to `client`, `dsrc`, and `src`.

---

## Repository layout

```
README.md                          ← you are here
client/                            ← client-facing specs (SwgTitan, Maya, UX)
server/                            ← server scripts (dsrc)
engine/                            ← shared C++ / cross-tier engine
packages/tangible-dynamics/        ← TangibleDynamics multi-tier package
content/farming-ranch/             ← content authoring
tools/                             ← launcher and support tooling (see below)
```

**Support tooling (monorepo `support/`):** [Kaadu Launcher](../support/launcher_redux/docs/README.md) — player launcher for SWG: Titan (manifest downloads, auto-login, SFTP CDN).

**Build guides that stay in the `client` repo** (not duplicated here): `COMPILE_GUIDE_CLIENT.md`, `MODELING.md`, `VLC.md`, `STREAMING-GIF-ETC.md`.

---

## Client (`client/`)

| Document | Subject |
|----------|---------|
| [planning.md](client/planning.md) | Running notes: Magic Painting, God Client, x64 status |
| [x64-migration.md](client/x64-migration.md) | Client x64 migration, `SwgTitan` checklist |
| [engine-improvements.md](client/engine-improvements.md) | D3D9 instancing, occlusion |
| [texture-url-resolving.md](client/texture-url-resolving.md) | Magic painting (URL textures) |
| [rt-camera-system.md](client/rt-camera-system.md) | RT security camera / screen |
| [atmo-landing-points.md](client/atmo-landing-points.md) | Atmospheric landing points (map UX) |
| [bounty-hunter-overhaul.md](client/bounty-hunter-overhaul.md) | Bounty Hunter plan |
| [crafting-overhaul.md](client/crafting-overhaul.md) | Crafting QoL, schematic library, BOM |
| [city-gameplay-landscaping.md](client/city-gameplay-landscaping.md) | City systems design |
| [city-implementation-summary.md](client/city-implementation-summary.md) | City update — files touched |
| [calendar-database.md](client/calendar-database.md) | Oracle calendar persistence |
| [in-game-calendar-plan.md](client/in-game-calendar-plan.md) | In-game calendar plan |
| [story-companions.md](client/story-companions.md) | Story companion datatables |
| [developer-commands.md](client/developer-commands.md) | `player_developer.java` commands |
| [god-client-pen-tablet.md](client/god-client-pen-tablet.md) | God Client pen tablet (Kamvas 13): pressure brushes, eraser, barrel camera |
| [god-client-productivity.md](client/god-client-productivity.md) | God Client shortcuts: command palette, terrain undo/brush keys, recent templates, transform copy/paste, console input |
| [god-client-bridge.md](client/god-client-bridge.md) | God Client automation bridge: localhost socket + `godclient_*` MCP tools for offline scene load, selection, placement, transforms, undo, console, camera, screenshots, terrain strokes; Load Scene stall caveat |
| [interior-structure-creation.md](client/interior-structure-creation.md) | Buildings / POB ships |
| [maya-pob-from-scratch.md](client/maya-pob-from-scratch.md) | Author a `.pob` in Maya |
| [maya-kitbash-import-combine.md](client/maya-kitbash-import-combine.md) | Maya kitbash workflow |

---

## Server (`server/`)

| Document | Subject |
|----------|---------|
| [atmospheric-flight.md](server/atmospheric-flight.md) | Ground-scene ship flight |
| [npc-pob-ship.md](server/npc-pob-ship.md) | NPC POB ship controller |
| [nested-pob-ships.md](server/nested-pob-ships.md) | POB ships parked inside another POB's cell (hangars) |
| [vessel-fleet.md](server/vessel-fleet.md) | Private boardable ships: Lambda shuttle, Corellian corvette, Nebulon-B (deeds, helm, paint) |
| [open-face-vendor.md](server/open-face-vendor.md) | Open-face shelf vendor |

---

## Engine (`engine/`)

| Document | Subject |
|----------|---------|
| [dynamic-mounts.md](engine/dynamic-mounts.md) | Dynamic mount maker (`mount.dm`, `hp_dyn`) |
| [god-client-terrain-editor.md](engine/god-client-terrain-editor.md) | God Client terrain editor |
| [god-client-terrain-editor-checklist.md](engine/god-client-terrain-editor-checklist.md) | Terrain editor checklist |

---

## Packages

### TangibleDynamics — [packages/tangible-dynamics/](packages/tangible-dynamics/)

Script-driven forces on tangibles (push, spin, hover, orbit, rides, lock-to-parent, …). Start at [README.md](packages/tangible-dynamics/README.md).

| Document | Subject |
|----------|---------|
| [overview.md](packages/tangible-dynamics/overview.md) | Architecture and core API |
| [quick-reference.md](packages/tangible-dynamics/quick-reference.md) | Scripting patterns (authoritative API list) |
| [build-and-integration.md](packages/tangible-dynamics/build-and-integration.md) | Compile and deploy |
| [files.md](packages/tangible-dynamics/files.md) | Source file map |

---

## Content (`content/`)

| Document | Subject |
|----------|---------|
| [farming-ranch/template-map.md](content/farming-ranch/template-map.md) | Farming / ranch template map |

---

## Tools (`tools/`)

| Document | Subject |
|----------|---------|
| [kaadu-launcher.md](tools/kaadu-launcher.md) | Kaadu player launcher — index to `support/launcher_redux` docs |

---

## Monorepo path convention

When reading code paths in these docs:

| Repo | Root | Example |
|------|------|---------|
| Server scripts | `dsrc/` | `dsrc/sku.0/sys.server/compiled/game/script/library/tangible_dynamics.java` |
| Shared / server C++ | `src/` | `src/engine/shared/library/sharedObject/...` |
| Client C++ | `client/` | `client/src/engine/client/library/clientGame/...` |

Set `TITAN_ROOT` (or use your monorepo root) when running build commands that reference sibling repos.

---

## Keeping docs in sync

1. Edit markdown in this repo (`titan-docs`) or the monorepo `docs/` mirror.
2. Commit and push `titan-docs` so README links in `client`, `dsrc`, and `src` stay valid.
3. Update [FEATURES.md](https://github.com/BubbaJoeX/titan/blob/main/FEATURES.md) when adding a new major system.
