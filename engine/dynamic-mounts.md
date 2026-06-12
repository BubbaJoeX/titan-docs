# Dynamic mounting (mount maker)

This document describes the **dynamic mount** stack in Titan: designer-authored rider geometry and optional client-side attachments, **containment-based** riding like production mounts, and known gaps.

## Goals

- Author **per-creature** mount metadata (`mount.dm.*`) without relying solely on IFF saddle tables or pet control devices.
- Optional **visual overlays** (`hp_dyn.*`) for saddle meshes, lights, and FX, replicated to the client as packed state.
- **Ride / drive** uses normal **`mountCreature`** containment plus **`mountMakerPossessionEnter`** so input follows the mount.
- **Safety**: logout / emergency paths clear possession and riding; GM tooling stays gated.

## Architecture (split responsibilities)

| Layer | Responsibility |
|--------|----------------|
| **Server scripts** | Objvars, designer session (invuln, combat ignore), preset file I/O, calling `makeDynamicMountable`, `mountCreature`, `mountMakerPossessionEnter` / `Leave`. |
| **Server C++** | `C_mount`, `mount.dm` bypass in `getMountabilityStatus`, **rider SlottedContainer** repair (`ensureDynamicMountRiderContainmentSlots`), `mountCreature` transfer, `TangibleObject::updateDynamicHardpointsFromObjvars` packing for baselines. |
| **Client C++** | Consumes `m_dynamicHardpointsState`, `mount_dm` tail in same pack, `HardpointObject` / skeletal hardpoint defaults for `hp_dyn` apps (see client `TangibleObject.cpp`). |
| **Client scripts** | None required for core ride; possession is engine/client messaging. |

## Data: `mount.dm.*`

- **`mount.dm.active`** (int): marks a scripted dynamic mount; affects `getMountabilityStatus()` and triggers server **rider-slot ensure** when set.
- **`mount.dm.capacity`**: 1–8 seats (script + engine cap).
- **`mount.dm.seat.<n>.pose`**, **`.ox` / `.oy` / `.oz`**: per-seat rider pose and local offset (authoring; replicated to client via packed `mount_dm` line in `m_dynamicHardpointsState`).

Authoring UI: `creature_dynamic_mount` listbox + `base_player` SUI handlers; session helpers in `mount_maker`.

## Data: `hp_dyn.*`

Nested list slots (e.g. `hp_dyn.<slot>.kind` = `app` | `light` | `fx`, plus fields documented in `TangibleObject.h` near `updateDynamicHardpointsFromObjvars`).

- Server packs objvars into **`m_dynamicHardpointsState`** (`TangibleObject::updateDynamicHardpointsFromObjvars`).
- Client applies attachments in **`TangibleObject::applyDynamicHardpointsState`**.
- **`hp` empty or `-`**: on **skeletal non-vehicle** creatures the client resolves to the **`saddle`** hardpoint so visuals track the skeleton (not the static object root).

Unified authoring library: `dynamic_hardpoint.java`; GM terminal `gm_dynamic_hardpoint.java` delegates into it. Slot selection objvar: `mount_maker.hp_dyn_slot` (`dynamic_hardpoint.OV_HP_SLOT`).

## Presets

- **`mount_maker`**: `exportObjVarsToFile` / `applyPresetFromFile` / `applyPresetToSpawn` — text lines `objvar.path=value`.
- **Preset directory** is currently **hard-coded** in `mount_maker.java` as `PRESET_DIR` (Linux path under `swg-main`). **Needs work** for other install roots / Windows dev trees.

## Riding and possession

Typical server order (`mount_maker.possessionEnter`):

1. Ensure **`mount.dm.active`** (and capacity defaults if missing).
2. **`makeDynamicMountable(mount)`** — sets **`C_mount`** and ensures rider containment slots when `mount.dm.active` is set (see below).
3. **`doesMountHaveRoom`**, **`mountCreature(designer, mount)`**.
4. Invulnerability, then **`mountMakerPossessionEnter`**.

**Unmount / cleanup**: `possessionLeave`, `emergencyUnmountAll`, `onPlayerLogoutCleanup` (`base_player` logout paths), slash command **`/mountMakerExit`** (`command_table_ground.tab` → `cmdMountMakerExit`).

**Authorization** (designer tooling): scripts use **`isGod` or `test_center`** objvar on the character; C++ possession entry matches for **`mountMakerPossessionEnter`** (see `Client.cpp` / `ScriptMethodsMount.cpp`).

**Possession template gate** (`Client::mountMakerPossessionEnter`): templates with **`getCanCreateAvatar()`** normally cannot be possessed (anti-abuse). If **`mount.dm.active`** is set on the mount, that check is **skipped** so authoring creatures can reuse avatar-style templates.

**Script TRIG gate** (`ContainerInterface::checkTransferScripts`): when **`mount.dm.active`** and the item is a **`CreatureObject`** rider entering a mountable mount, **`TRIG_ABOUT_TO_LOSE_ITEM`** (source), **`TRIG_ABOUT_TO_RECEIVE_ITEM`** (mount), and **`TRIG_ABOUT_TO_BE_XFERRED`** (rider) **SCRIPT_OVERRIDE** results are ignored so cells/buildings/ai scripts do not cancel **`mountCreature`**; C++ **`onContainerAboutTo*`** callbacks still run.

**`mount_maker.possessionEnter`**: if **`mountMakerPossessionEnter`** fails after **`mountCreature`**, the script **dismounts** and sends a system message (stale client/server mismatch hint).

## Server containment (critical for `mountCreature`)

Dynamic mounts can be marked mountable via **`mount.dm.active`** even when the creature template has **no rider slots** or **`CT_none`**.

- **`CreatureObject::makeDynamicMountable`**: sets **`C_mount`**, then **`ensureDynamicMountRiderContainmentSlots()`**.
- **`CreatureObject::ensureDynamicMountRiderContainmentSlots()`** (runs when **`mount.dm.active`** is set): merges **`rider` / `riderN`** into an existing **`SlottedContainer`**, or allocates a **new** `SlottedContainer` if the object has **no** container property and **no** `VolumeContainer`. Idempotent.
- **JNI** (`ScriptMethodsMount.cpp`): **`makeDynamicMountable`** (early return if already mountable), **`doesMountHaveRoom`**, and **`mountCreature`** all call **`ensureDynamicMountRiderContainmentSlots`** so older or persisted mounts still get slots before room/transfer checks.

**Failure modes** (logged on channel **`mount-script-interface`** for JNI `mountCreature` when applicable):

- Mount has **only a volume container** — cannot add a parallel `SlottedContainer` under current object rules; **containment ride will not work** until template/container type changes.
- **`mountCreature`** transfer fails after a free slot exists — usually rider **arrangement / `mayAdd`** / script transfer hooks; investigate `ContainerInterface` logs and rider template.

## Client visuals vs classic saddle

When **`mount_dm`** is active in the packed state, **`CreatureObject::onJustBecameMountable`** skips **`SaddleManager::addRiderSaddleToMount`** / dressing so **`hp_dyn`** can own the saddle look (`TangibleObject` / `CreatureObject` client sources).

## Key source files (non-exhaustive)

| Area | Path |
|------|------|
| Session + preset + ride | `dsrc/.../script/library/mount_maker.java` |
| Authoring listbox | `dsrc/.../script/creature/creature_dynamic_mount.java` |
| hp_dyn UI / GM terminal | `dsrc/.../script/library/dynamic_hardpoint.java`, `.../terminal/gm_dynamic_hardpoint.java` |
| Player SUI / logout / command | `dsrc/.../script/player/base/base_player.java`, `.../datatables/command/command_table_ground.tab` |
| JNI | `src/.../serverScript/.../ScriptMethodsMount.cpp` |
| Mount logic + slots | `src/.../serverGame/.../CreatureObject_Mounts.cpp`, `CreatureObject.h` |
| hp_dyn pack | `src/.../serverGame/.../TangibleObject.cpp` / `.h` |
| Slotted merge | `src/.../sharedObject/.../SlottedContainer.{h,cpp}` (+ client mirror if used) |
| Client hp_dyn apply | `client/.../clientGame/.../TangibleObject.cpp` |
| Possession gate | `src/.../serverGame/.../Client.cpp` (and JNI above) |

---

## What is in good shape

- Single library **`mount_maker`** for presets + ride session (avoids missing **`dynamic_mount`** class on deploy).
- **Designer session** objvars, invuln during ride, **`mountMakerPossessionEnter`** after **`mountCreature`**.
- **Rider slot repair** for `mount.dm.active` mounts and JNI logging for “no room” vs “transfer failed”.
- **Client `hp_dyn`** following skeletal **`saddle`** when `hp` is blank/`-`, plus inverse scale on hardpoint nodes for creatures.
- **Emergency** `/mountMakerExit` and logout cleanup hooks.

---

## What is left / needs work

### High priority

1. **`PRESET_DIR` portability** — replace hard-coded Linux path with config, env, or `data` root discovery so presets work on Windows and other installs.
2. **Volume-only creatures** — no containment mount without a larger design (template `CT_slotted` / `CT_ridable`, or engine support for dual container types — high risk).
3. **`mountCreature` transfer failures** — if logs show transfer failed with slots present, trace **`SlottedContainmentProperty`** arrangements for the rider, **`checkTransferScripts`**, and mount-specific script hooks.

### Medium priority

4. **Multi-seat polish** — capacity up to 8 exists in data and slots; UI and possession assumptions are mostly **seat 0** focused. Verify passenger slots, camera, and dismount order for each seat.
5. **Persistence** — confirm DB save/load restores **`mount.dm.*`**, **`hp_dyn.*`**, and **`C_mount`** consistently; re-apply presets on spawn if using template-only spawns.
6. **Production gating** — today **`test_center`** + god paths are intentional for tooling; decide prod rules (CSR only, quest flag, etc.) and enforce consistently in script + C++.
7. **Mount validity table** — `getMountabilityStatus` bypasses scale table when `mount.dm.active`; designers can author impossible scales. Optional: soft warnings in UI or stricter checks.

### Lower priority / nice-to-have

8. **Automated tests** — JNI round-trip for `makeDynamicMountable` + `doesMountHaveRoom` + `mountCreature` on synthetic creatures (CT_none vs slotted).
9. **Client-only `hp_dyn` in cells** — FX path was simplified when wrapping in `HardpointObject`; watch for interior / portal edge cases vs old `setParentCell` behavior.
10. **Documentation for artists** — which appearance templates expose **`saddle`** / rider hardpoints; link to CDF/skeleton docs in art pipeline.
11. **`docs/README.md`** (or [titan-docs](https://github.com/BubbaJoeX/titan-docs)) — add a one-line link to this file in the master doc index.

---

## Quick troubleshooting

| Symptom | Check |
|---------|--------|
| Ride / `mountCreature` fails | Server log **`mount-script-interface`**: no empty rider slot vs transfer failed. **`mounts-error`**: volume-only slot ensure. If transfer fails with slots present, **`CreatureObject::onContainerAboutToGainItem`** used to run **`isAppearanceEquippable`** on the **rider** (wearables table vs **mount** species) and could return **`CEC_SilentError`** — that path is now **skipped when the destination is mountable and the item is a `CreatureObject`** (rider containment). |
| Saddle floats | Client build includes **`resolveHpDynHardpointCrc`** behavior; set explicit **`hp`** if not using creature **`saddle`** bone. |
| Possession denied | **`test_center`** / god on character; **`Client::mountMakerPossessionEnter`** gate. |
| Stuck riding after UI loss | **`/mountMakerExit`**, relog (logout cleanup), or GM **`emergencyUnmountAll`**. |

---

*Last updated to match the Titan tree as of the dynamic mount + `hp_dyn` + containment slot work; adjust dates and paths in your branch as needed.*
