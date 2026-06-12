# Titan documentation

Canonical specs for the Titan monorepo. Published mirror: **[github.com/BubbaJoeX/titan-docs](https://github.com/BubbaJoeX/titan-docs)**.

Local copy lives in **`docs/`** at the monorepo root (`D:\titan\docs` when using the full tree). Keep **`titan-docs`** in sync when you change these files.

For code touchpoints and feature inventory, see **[FEATURES.md](../FEATURES.md)** in the monorepo.

---

## Layout

| Path | Contents |
|------|----------|
| **[client/](client/)** | Client features, Maya, migration, gameplay design |
| **[server/](server/)** | Server scripts (`dsrc`) |
| **[engine/](engine/)** | Shared engine systems (C++ / cross-tier) |
| **[packages/](packages/)** | Multi-tier feature packages |
| **[content/](content/)** | Content authoring (farming, templates, …) |

---

## Client (`client/`)

| Document | Subject |
|----------|---------|
| [planning.md](client/planning.md) | Running Titan: Magic Painting, God Client, x64 inventory |
| [x64-migration.md](client/x64-migration.md) | Client x64 migration, `SwgTitan` checklist |
| [engine-improvements.md](client/engine-improvements.md) | D3D9 instancing, occlusion |
| [texture-url-resolving.md](client/texture-url-resolving.md) | Magic painting pipeline |
| [rt-camera-system.md](client/rt-camera-system.md) | RT camera / screen |
| [atmo-landing-points.md](client/atmo-landing-points.md) | Atmospheric landing points (UX) |
| [bounty-hunter-overhaul.md](client/bounty-hunter-overhaul.md) | Bounty Hunter plan |
| [crafting-overhaul.md](client/crafting-overhaul.md) | Crafting QoL, schematic library, BOM |
| [city-gameplay-landscaping.md](client/city-gameplay-landscaping.md) | City systems design |
| [city-implementation-summary.md](client/city-implementation-summary.md) | City update — files touched |
| [calendar-database.md](client/calendar-database.md) | Oracle calendar tables |
| [in-game-calendar-plan.md](client/in-game-calendar-plan.md) | In-game calendar plan |
| [story-companions.md](client/story-companions.md) | Story companion datatables |
| [developer-commands.md](client/developer-commands.md) | Developer commands |
| [interior-structure-creation.md](client/interior-structure-creation.md) | Buildings / POB ships |
| [maya-pob-from-scratch.md](client/maya-pob-from-scratch.md) | Author a `.pob` in Maya |
| [maya-kitbash-import-combine.md](client/maya-kitbash-import-combine.md) | Maya kitbash workflow |

**Client repo guides (not in `titan-docs`):** `COMPILE_GUIDE_CLIENT.md`, `STREAMING-GIF-ETC.md`, `VLC.md`, `MODELING.md` in the **`client`** repository.

---

## Server (`server/`)

| Document | Subject |
|----------|---------|
| [atmospheric-flight.md](server/atmospheric-flight.md) | Ground-scene ship flight |
| [npc-pob-ship.md](server/npc-pob-ship.md) | NPC POB ship controller |
| [open-face-vendor.md](server/open-face-vendor.md) | Open-face shelf vendor |

---

## Engine (`engine/`)

| Document | Subject |
|----------|---------|
| [dynamic-mounts.md](engine/dynamic-mounts.md) | Dynamic mount maker |
| [god-client-terrain-editor.md](engine/god-client-terrain-editor.md) | God Client terrain editor |
| [god-client-terrain-editor-checklist.md](engine/god-client-terrain-editor-checklist.md) | Terrain editor checklist |

---

## Packages (`packages/`)

### TangibleDynamics — [packages/tangible-dynamics/](packages/tangible-dynamics/)

| Document | Subject |
|----------|---------|
| [README.md](packages/tangible-dynamics/README.md) | Overview and checklist |
| [overview.md](packages/tangible-dynamics/overview.md) | Architecture, API, examples |
| [quick-reference.md](packages/tangible-dynamics/quick-reference.md) | Scripting patterns |
| [build-and-integration.md](packages/tangible-dynamics/build-and-integration.md) | Build and deploy |
| [files.md](packages/tangible-dynamics/files.md) | File inventory |

---

## Content (`content/`)

| Document | Subject |
|----------|---------|
| [farming-ranch/template-map.md](content/farming-ranch/template-map.md) | Farming / ranch template map |

---

## Related (monorepo)

| Path | Subject |
|------|---------|
| [FEATURES.md](../FEATURES.md) | Feature index with code paths |
| [README.md](../README.md) | SWG Source getting started |
| [titan-prepare/README.md](../titan-prepare/README.md) | Linux server install |
