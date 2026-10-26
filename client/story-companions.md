# Adding new story companions

Story companions are **datatable-driven NPC pets** (SWTOR-style roster): one combat pet out at a time, control device on the **datapad**, shared rules with normal NPC pets (`pet_lib` / `callable`).

Primary code: `script.library.companion_lib`  
Primary data: `datatables/companion/story_companions.iff` (source `.tab` under `dsrc/.../datatables/companion/story_companions.tab`)

---

## 1. Add a row to `story_companions`

| Column | Purpose |
|--------|---------|
| **companion_id** | Stable string key (e.g. `companion_my_hero`). Used in code and objvars (`companion.storyId`). |
| **companion_name** | Display / default name. |
| **creature_name** | Row in **`datatables/mob/creatures.iff`** — defines the spawned mob template, stats baseline, and species. Ignored at grant for **`full`** tier (player species template used instead). |
| **level** | `0` = effective level follows the owner (capped like other pets). Positive = cap level from table vs player. |
| **role** | `tank`, `healer`, `dps` (or `heal`) — default **combat stance** (`companion.stance` on PCD/pet). |
| **companion_weapon** | Optional weapon template `.iff` path granted on first summon when no stored player weapon exists (e.g. `object/weapon/ranged/pistol/pistol_cdef.iff`). |
| **companion_favorite_gifts** | Optional comma-separated item names for gift/influence systems. |
| **companion_speed** | Movement scale hint (default `1.2` in table). |
| **companion_pet_bar_abilities** | **Non–human-skeleton** companions only: comma-separated **`beast_specials`** `ability_name` entries (e.g. `bm_slash_2`). **Human-shaped** companions use the humanoid pet bar (weapon toggle + taught player commands); this column is ignored for them at runtime. |
| **companion_customization** | `none` (default), `celebrity` (rename only), or `full` (rename + appearance wearables + palvar persistence). |
| **grant_message** | System message when granted via `grantStoryCompanionToDatapad` (can be empty). |

After editing the `.tab`, rebuild the **`.iff`** as your pipeline requires.

### Customization tiers

| Tier | Rename (`companion.displayName`) | Appearance wearables | Player species template at grant |
|------|-----------------------------------|----------------------|----------------------------------|
| **none** | No | No | No (uses `creature_name`) |
| **celebrity** | Yes | No | No |
| **full** | Yes | Yes (mannequin-style SUI) | Yes (`object/creature/player/{species}_{gender}.iff`) |

Example rows in repo: demos = `none`, Greeata = `celebrity`, Associate (`companion_commoner`) = `full`.

---

## 2. Creature choice: humanoid vs non-humanoid

- **Human skeleton** (`ai_lib.isHumanSkeleton(pet)` — see `datatables/ai/species.iff` **Skeleton** = `human`):  
  - Pet bar: **melee/ranged toggle** + **four taught slots** + three core bar slots.  
  - **Faction** on summon: aligned to owner via `applyStoryCompanionFactionFromOwner`.  
  - Teach abilities from **`getCommandListingForPlayer`** (must exist in `command_table` with a **scriptHook**).

- **Other creatures** (e.g. beasts, aliens without human skeleton):  
  - Pet bar abilities come from **`companion_pet_bar_abilities`** and must validate against **`beast_specials`**.

Pick **`creature_name`** accordingly. For **`full`** tier humanoid associates, the grant uses the owner's player species template automatically.

---

## 3. Weapon persistence

- Drag a weapon onto an **active** story companion (`pet.java` / `handleStoryCompanionWeaponGift`).
- On **store**, player-gifted weapons move to a hidden gear-hold container on the owner datapad (`companion.gearHold` on PCD, `companion.storedWeapon` objvar).
- On **summon**, stored weapons are re-equipped; otherwise **`companion_weapon`** datatable default is applied when set.
- **Companion Options** hub (PCD radial): view weapon status, clear weapon, return weapon to player inventory.

---

## 4. Naming and appearance

- Custom name: **`companion.displayName`** objvar on PCD; hub **Rename Companion** for `celebrity` and `full` tiers.
- **`full`** tier: hub **Customize Appearance** add/remove wearables (same item types as mannequin: armor, clothing, jewelry, cybernetics). Wearables persist in gear hold when stored; palvars persist via `pet_lib` customization objvars on PCD.
- Summon applies display name, palvars, wearables, and weapon via `companion_lib.finishStoryCompanionSummon`.

---

## 5. Granting the companion to a player

Runtime API:

```java
obj_id cd = companion_lib.grantStoryCompanionToDatapad(player, "companion_id");
```

- Creates/spools the pet, builds the **control device** on the datapad, sets **`companion.storyId`**, default stance, **`companion.taughtAbilities`**, attaches **`systems.companion.companion_story_pcd`**, stores the pet, ensures **`ai.pet_master`** on the player.  
- **`full`** tier: sets **`companion.spawnTemplate`**, grants from player species template.  
- Fails if the player already owns that `companion_id`, pet storage limits block, or creation fails.

Call this from **quests**, **conversation** scripts, or **GM tools**.

---

## 6. Hire flow (world NPC → join roster)

For a **world recruiter** that talks to the player:

1. Implement a **conversation** script (see `conversation.companion_greeata`).
2. Wire the companion in **`companion_lib.resolveHireConversationScript(String storyCompanionId)`** — return your `conversation.*` script name (e.g. `"conversation.companion_my_npc"`).
3. Optionally handle **`applyMakeHireableToNpc`** / **`prepareHireConversationNpc`** for designer `/developer` setup (strip AI, attach only hire dialog). Greeata also has **`companion_lib.GREEATA_WORLD_MOBILE_TEMPLATE`** for a safe client mesh.

In the hire branch, call:

- `companion_lib.playerOwnsStoryCompanion(player, STORY_ID)` to avoid duplicates  
- `companion_lib.grantStoryCompanionToDatapad(player, STORY_ID)`  
- Optional: `companion_lib.modifyInfluence(player, STORY_ID, delta)`

**`resolveHireConversationScript`** must return **non-null** for `applyMakeHireableToNpc` to succeed for that id.

---

## 7. Control device radial (Companion Options hub)

Script: **`systems.companion.companion_story_pcd`**

- **Combat Role** — Tank / Healer / Damage  
- **Rename Companion** — `celebrity` and `full` tiers  
- **Customize Appearance** — add/remove wearables (`full` tier only)  
- **Weapon** — status line, clear, return to inventory  
- **Train / Clear Ability Slot** — humanoid pet bar taught commands  
- **Train / Clear Core Bar Slot** — attack / follow / stay style slots  

Requires **`companion.storyId`** on the PCD. Hub title uses custom display name when set.

---

## 8. Key PCD objvars

| Objvar | Purpose |
|--------|---------|
| `companion.storyId` | Datatable row key |
| `companion.displayName` | Owner-assigned name |
| `companion.spawnTemplate` | Player creature template for `full` tier respawn |
| `companion.storedWeapon` | Persisted player weapon while stored |
| `companion.gearHold` | Hidden datapad inventory for weapons/wearables |
| `companion.stance` | Combat role |
| `companion.taughtAbilities` | Humanoid bar taught commands |
| `companion.coreBarCommands` | Core bar slot commands |
| `ai.pet.palvar.*` | Customization palvars (via `pet_lib`) |

---

## 9. Code touchpoints

| Task | Where |
|------|--------|
| Grant / summon / store logic | **`companion_lib`** (`finishStoryCompanionSummon`, `saveStoryCompanionStateOnStore`) |
| Summon hook | **`ai.pet_control_device`** |
| Store hook | **`pet_lib.storePet`** |
| Spawn template override | **`pet_lib.createPetFromData`** |
| Hub UI | **`systems.companion.companion_story_pcd`** |
| Client Companion tab | **`SwgCuiCompanionTab`**, **`SwgCuiCompanionCustomize`**, `/companionTab` |
| Weapon drag-drop | **`ai.pet`** `OnGiveItem` → **`handleStoryCompanionWeaponGift`** |
| New hire conversation | **`companion_lib.resolveHireConversationScript`** |

---

## 10. Client Companion tab

The **Companion** tab (`SwgCuiCompanionTab`, `/pda.CompanionTab`) mirrors the player Appearance tab but targets your **active story companion** combat pet (detected via `PlayerObject::getPetId()` and `getCompanionPetStanceUi()`).

Open from the client with `/companionTab` (same pattern as `/appearanceTab`).

### Tab behavior

| Feature | Client | Server command | Server handler |
|---------|--------|----------------|----------------|
| Wearable drag from inventory | Companion tab slot drop | `companionEquipWearable` | `player_beastmaster.companionEquipWearable` -> `companion_lib.equipCompanionWearableFromPlayer` |
| Unequip (double-click slot item) | Companion tab | `companionUnequipWearable` | `player_beastmaster.companionUnequipWearable` -> `companion_lib.removeCompanionWearableToPlayer` |
| Rename button | Companion tab | `companionRename` | `player_beastmaster.companionRename` -> SUI or `companion_lib.setCompanionDisplayName` |
| Image Design button | Opens `SwgCuiCompanionCustomize` (`/pda.companioncustomize.designer`) | `companionSaveCustomization` on OK | `companion_lib.applyCompanionCustomizationParams` + PCD customization objvars |

The tab auto-closes when no active story companion is out. Server-side tier checks still apply (`celebrity`/`full` for rename, `full` for wearables and Image Design).

Wearables persist through the existing gear hold and appearance inventory pipeline (`equipCompanionWearableFromPlayer`, `saveStoryCompanionAppearanceOnStore`, `restoreStoryCompanionAppearanceOnSummon`).

Image Design saves palette/ranged customization objvars on the PCD via `pet_lib.savePetInfo` (same path as beast/pet palvar persistence).

Canonical sources: `client/` for C++; `dsrc/` for server scripts and datatables. Do not mirror changes into `gr/` (legacy) or `exe/win64_godclient/`.

---

## 11. Quick checklist

1. Add **`creature_name`** to creatures table if it does not exist.  
2. Add **`story_companions`** row with columns above (set **`companion_customization`** tier).  
3. Rebuild **story_companions.iff**.  
4. Grant via script: **`grantStoryCompanionToDatapad`**, or implement hire + **`resolveHireConversationScript`**.  
5. Test: datapad PCD, call out pet, store/recall with gifted weapon, rename (celebrity/full), wearables (full), combat role, train/clear abilities, client Companion tab (`/companionTab`) drag-equip, Image Design save.

---

## 12. Reference IDs in repo

Example rows: `companion_demo_bodyguard`, `companion_demo_scoundrel`, `companion_demo_medic`, `companion_greeata`, `companion_commoner` in `story_companions.tab`.  
Reference conversation: `conversation/companion_greeata.java`.
