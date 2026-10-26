# The Battle of Naboo, as told by Jar Jar Binks

A Historical heroic instance. Bombad General Jar Jar Binks narrates the Battle of the Great Grass Plains (Episode I)
his way, and the group plays it: they raise the Gungan shield, stop the droid army as it unfolds out of the
transports, hold the generators until the shield falls, take a captured AAT into the fight, and survive until the
droid control ship goes up and every droid on the field shuts down.

This page is the contract between the art, terrain, client and server pieces: names, hardpoints, positions and
phase logic. Change it here first when a name or a number changes.

| | |
|--|--|
| Instance | `heroic_naboo_battle` - "Heroic: The Battle of Naboo" |
| Scene | `adventure_heroic` (new): 16 km, 3 x 3 tiles of 3.8 km, 5 km apart, like stock `adventure2` (Hoth) |
| Tile | 05, the centre tile (world 0, 0): the Great Grass Plains. Other tiles are reserved for later Historical heroics |
| Group | 1-8 players, level 90 (stock heroic rules), daily lockout, 2 h time limit |
| Quest giver | Jar Jar Binks in Theed (Naboo), with the Historical marker: a spinning gold disc with the Aurebesh H (Herf) |

---

## 1. The story and the phases

Jar Jar's lines are in `string/en/theme_park/heroic_naboo_battle.stf` and play in the comm window with his head
(`commPlayers` with the Jar Jar server template as the appearance override), plus a chat copy. Every phase is also
a task in the ground quest `heroic_naboo_battle`, so the journal shows what to do.

**Direction.** Every objective is marked three ways (`naboo_battle_lib.setRally`):
- a **rally point** on each spot (`appearance/titan_hrc_nboo_rally_marker.apt`): a 24 m by 4 m gold light shaft,
  a dashed ring on the ground, and the Historical "H" emblem (1.6 m, two crossed discs) at head height;
- an active yellow **waypoint** for each player;
- an **objective line** in chat (`obj_*` strings).

Late joiners and cloned players get the current one (`syncRally`). The rally points move with the story:

| Rally point | When |
|-------------|------|
| Captain Tarpals | phase 1 starts |
| each generator still off | Tarpals gives the order |
| the Gungan line | phase 2 |
| the MTT drop zones | the racks unfold, for 25 s, then back to the line |
| the three generators | phase 3 |
| the captured AAT and both catapults | phase 4 |
| in front of the line | phase 5 |
| the line | the holdout |

**Cloning.** A player who dies and is not revived clones at the Gungan camp. The controller egg carries the
stock `structure.municipal.cloning_facility` script, as a `PRIVATE_INSTANCE` facility. Its row in
`cloning_facility_respawn` sets the spawn to (-120, 12.1, 6), and its row in `clone_mapping` maps
`adventure_heroic/heroic_naboo_battle` to itself.

| # | Phase | Players do | Ends when |
|---|-------|------------|-----------|
| 0 | **Meesa story** (Theed) | Talk to Jar Jar; the group is sent to the instance | the group enters |
| 1 | **Da Grand Army** | Rally with Captain Tarpals at the Gungan camp (45 s, or as soon as the group reaches him); switch on the three fambaa shield generators (radial *Calibrate Generator*, 8 s each, interrupted by damage) while STAP scouts harass | all three generators run: the shield dome rises |
| 2 | **Da Droid Army** | AATs shell the dome (impacts on the dome, no damage inside). Two MTTs land, open and run out their racks; folded B1s drop off folded (creature mood `titan_folded`) and power up after 2.5 s (another 6 s; passive and double damage until they stand). Kill them before they stand, then hold the line as squads walk through the shield | 30 droids destroyed, or 5 minutes |
| 3 | **Da Shield Falls** | Droid squads and a security detail go for the generators; defend the three fambaas (generator HP bars) | the scripted collapse: generator 1 at 0 %, the dome flickers and falls |
| 4 | **Da Tank** | A disabled AAT is recaptured: one player drives (strafe-capable hover), one mans the turret (plasma shells, `/titanAatFire`). Others work the Gungan catapults (radial *Launch Booma*) or fight on foot | 3 enemy AATs destroyed |
| 5 | **Da Big Boom** | The droid commander arrives in his command AAT with an escort and calls for surrender. Bring it down, then hold out | the commander's AAT dies and a 60 s hold-out timer ends: the control ship explodes in the sky, every droid freezes and powers down |
| - | **Epilogue** | Victory cheer; Jar Jar's wrap-up; rewards | instance clock set to 5 minutes |

Failure: everyone in the group incapacitated at once, or the clock runs out, resets the current phase (stock
heroic behaviour: the sequencer respawns the phase).

---

## 2. Scene and instance plumbing (server)

Stock pattern (see stock `echo_base` on `adventure2`): one buildout area per copy, an invisible controller egg
running `systems.instance.instance_manager` + `systems.dungeon_sequencer.sequence_controller` + the encounter
controller, a row in `instance_datatable.tab`.

| File | What |
|------|------|
| `support/MCP/SwgMayaMCP/examples/make_adventure_heroic.py` | builds `terrain/adventure_heroic.trn` + environment; `--register` registers the planet (`terrain/planet.py`) |
| `dsrc/.../sys.shared/.../datatables/buildout/buildout_scenes.tab` | add `adventure_heroic` |
| `dsrc/.../sys.shared/.../datatables/buildout/areas_adventure_heroic.tab` | `heroic_naboo_battle  -1900 -1900 1900 1900 ... light` (one copy: tile 05) |
| `dsrc/.../sys.server/.../datatables/buildout/adventure_heroic/heroic_naboo_battle.tab` | the controller egg at area-local (1900, 1900) = world (0, 0) |
| `dsrc/.../sys.shared/.../datatables/buildout/adventure_heroic/heroic_naboo_battle.tab` | the same row (stock keeps a one-egg stub here; the server reads `sys.server` first either way) |
| `data/.../sys.client/{compiled,built}/game/datatables/buildout/adventure_heroic/heroic_naboo_battle.iff` | the client format (`shared_template_crc`, no rows; the client FATALs without that column). Never in `D:/exported`: the server reads that before `sys.server` |
| `dsrc/.../sys.server/.../datatables/planet_server/preload_list.tab` | `adventure_heroic -100 -100 200 200 1 1`: keeps the tile-centre chunk (and so the controller) loaded with no player |
| `dsrc/.../sys.server/.../datatables/instance/instance_datatable.tab` | `heroic_naboo_battle  "-124,12,10,none"  ...  exit "<Theed>,naboo"  1 8 ... daily  vehicle_allowed 0` |
| `dsrc/.../sys.server/.../datatables/spawning/heroic/heroic_naboo_battle.tab` | the sequencer table (static dressing, waves, triggers), generated by `examples/naboo_battle_layout.py`: object rows need the full `.iff` template path, or the sequencer silently spawns nothing |
| `exe/linux/localOptions.cfg`, `multiserver.cfg`, `wsl/local.env TITAN_PLANETS` | start the scene (local server: needs a restart - ask first) |

World positions on tile 05, as offsets from the controller at the tile centre (y comes from the terrain; the
battlefield is 12-15 m). One source for all of them: `support/MCP/SwgMayaMCP/examples/naboo_battle_layout.py`
(spawn table, terrain pads; `naboo_battle_lib` mirrors DOME / LINE_R / GENERATORS). Everything a player walks
between is within ~250 m (playtest 2026-10-25: the first layout spread over 2.5 km).

| Point | x, z | Notes |
|-------|------|-------|
| Gungan camp | (-112, 0) | instance entry (-124, 10), Captain Tarpals, General Binks, the cloning point |
| shield dome | centre (-60, 0) | radius 90 m, 74 m high; pad flattened to y 12 |
| generators (fambaa) | G1 (-94, 0), G2 (-56, 40), G3 (-56, -40) | |
| Gungan line | an arc 78 m round the dome centre, z -56..56 | droids march to 16-26 m outside it |
| catapults | (-12, 58), (-12, -58) | |
| captured AAT | (-24, -30) | phase 4 |
| MTT pads | (150, -44), (150, 44) | MTTs face west and glide in from the droid rise (640, +-110); droids drop at (118, +-44) |
| STAP scouts | from (300..330, +-30..70) | phase 1, after the generators |
| enemy AATs | from (600..620, 0 / +-140); stations (190..210, 0 / +-78), phase 4 (120..140, 0 / +-66) | |
| command AAT | from (640, 0), station (170, 0) | phase 5 |

-------|------|-------|
| entry / staging hummock | (-1050, 0) | instance `enter_one` offset from the controller at (0, 0) |
| shield dome centre | (-420, 0) | dome radius 150 m, pad flattened to y 12 |
| generators (fambaa) | G1 (-480, 0), G2 (-360, 70), G3 (-360, -70) | |
| Gungan battle line | x -300, z -120..120 | inside the dome's front edge (x -270) |
| catapults | (-520, 50), (-520, -50) | |
| droid lines form | x +150..+300 | arrive from x +900 |
| MTT pads | (520, -160), (520, 160) | MTTs face west (heading 270), glide in from (1500, +-300) |
| enemy AATs | start (1100, +-300); stations (300, -260), (300, 260), (220, 0) | |
| captured AAT | (-150, -200) | phase 4 |
| command AAT | enters from (1300, 0) | phase 5 |

---

## 3. Assets

Built by `support/MCP/SwgMayaMCP` (geometry `maya/swg_mcp/naboo_battle.py`, materials
`examples/naboo_battle_materials.py`, terrain `examples/make_adventure_heroic.py`, export
`examples/make_naboo_battle.py`), written to `D:/exported`. Conventions: Maya y up, front +z, the exporter mirrors
X (so rotations about Y/Z change sign in engine space). Parts hung on hardpoints (`hp_dyn` "part" rows) are
modelled in their hardpoint's space; state 0 = as modelled, state 1 = moved by `amount`.

### 3.1 MTT (Multi-Troop Transport), ~30 m, khaki

| Appearance | Kind | Hardpoint on the hull | Motion |
|------------|------|------------------------|--------|
| `appearance/titan_hrc_nboo_mtt.apt` | hull (.lod, 3 levels, collision boxes) | - | - |
| `appearance/titan_hrc_nboo_mtt_lid.apt` | front cowl | `hp_lid` | rot about X, -100 deg, 2.5 s |
| `appearance/titan_hrc_nboo_mtt_chin.apt` | step plate under the rack | `hp_chin` | rot about X, +70 deg, 1.5 s |
| `appearance/titan_hrc_nboo_mtt_rack.apt` | droid rack, 112 folded B1s | `hp_rack` | move along +Z, 10 m, 6 s |
| `appearance/titan_hrc_nboo_mtt_rack_empty.apt` | the rack after deployment | `hp_rack` | swapped in at state 1 |

Other hull hardpoints: `muzzle_l1`, `muzzle_l2`, `muzzle_r1`, `muzzle_r2` (stacked twin cannons in the trunk
pods), `hp_dust` (under the hull), `hp_exhaust` (rear), `hp_deploy` (on the ground 22 m ahead: where the droids
drop off the rack).

Deploy sequence (server): glide in (TangibleDynamics push with ease-out) -> settle -> lid (0 s) -> chin (1 s) ->
rack (2 s, 6 s travel) -> droids drop in front of `hp_deploy` in two files, folded, and power up -> rack swaps to
empty. The MTT is attackable (vehicle armour); destroyed it plays the large explosion and smoke and is removed.

### 3.2 AAT (Armored Assault Tank), ~9.8 m, ochre

| Appearance | Kind |
|------------|------|
| `appearance/titan_hrc_nboo_aat.sat` | skinned hull on a small vehicle skeleton (`root`, `hull`) - creature and mount appearance |
| `appearance/titan_hrc_nboo_aat_turret.apt` | turret + main gun, `hp_dyn` part on `hp_turret`, rot about Y; state = yaw / 360 (server-driven traverse) |

Hull hardpoints: `hp_turret`, `muzzle_launcher_1..3` (the three front shell launchers), `muzzle_laser_l`,
`muzzle_laser_r` (side lasers), `hp_dust`, `seat_driver`, `seat_gunner`. Turret hardpoint: `muzzle_main`.

- NPC AATs: creature mobs (`niche=vehicle`, AT-ST pattern) with the `titan_hrc_aat_cannon` weapon; the
  encounter script turns the turret toward the target before each shell.
- The captured AAT: a two-seat strafe-capable mount (`mount.dm.*` seats, `canStrafe`): seat 1 drives, seat 2
  is the gunner. The gunner gets `/titanAatFire` (plasma shell at the look-at target: turret traverse, projectile
  `titan_hrc_nboo_plasma_shell`, 6 m splash) while seated.

### 3.3 Droids

| Mob (`creatures.tab`) | Template | Notes |
|-----------------------|----------|-------|
| `hrc_nboo_b1` | stock `battle_droid` | rifle; customisation pattern tan |
| `hrc_nboo_b1_pilot` | stock `battle_droid` | blue markings |
| `hrc_nboo_b1_security` | stock `battle_droid` | red markings, phase 3 detail |
| `hrc_nboo_b1_commander` | stock `battle_droid` | yellow markings; rides in the command AAT (boss) |
| `hrc_nboo_stap` | `titan_hrc_nboo_stap` | phase 1 scouts and the phase 5 escort; a B1 riding a STAP (below) |
| `hrc_nboo_aat` | `titan_hrc_nboo_aat` | NPC tank |
| `hrc_nboo_aat_command` | `titan_hrc_nboo_aat` | boss |

Heroic B1s use their own appearance, `appearance/titan_hrc_nboo_b1.sat`: the stock battle droid mesh and
skeleton with `appearance/lat/titan_hrc_nboo_b1.lat` (a copy of `battle_droid.lat` plus two rows), so stock
battle droids are untouched. The stock state graph (`all_b.ash`) already has the actions:

| Action | Logical | Animation | |
|--------|---------|-----------|--|
| `power_up` | `std_power_up` | `appearance/animation/titan_hrc_b1_unfold.ans` | 6 s: 3.5 s folded for transport (a boot twitch at the end), then 2.5 s unfolding - legs, torso, arms and head |
| `power_down` | `std_power_down` | `appearance/animation/titan_hrc_b1_power_down.ans` | 2 s: head drops, torso sags, knees give, arms hang |

**Deploy (`folded_droid`).** The droid spawns at its drop point with the creature mood `titan_folded`. A
mood is part of the creature's baselines, so every client that sees the droid sees it folded from its first frame.

The `.lat` wraps each standing loop (`loop_standing`, `loop_combat_standing`, the aimed loops) in a string selector
(SSAT) on the `mood` animation variable. `titan_folded` holds `titan_hrc_b1_folded.ans`; any other mood plays the
stock loop.

After 2.5 s it plays `power_up` and the mood returns to neutral underneath. It stays passive and takes double
damage until it stands, then normal AI takes over.

The first playtest fired `power_up` the moment the droid was created. A one-shot action sent before the client has
loaded the droid is lost, so the droids just stood there.

**Shutdown.** `power_down`, then stop AI and keep the droid invulnerable for the epilogue.

**STAP scouts.** `appearance/titan_hrc_nboo_stap.sat` is the stock B1 mesh on `titan_hrc_nboo_stap.lat`, which
maps every row except deaths and incapacitation to `titan_hrc_b1_stap_ride.ans`: standing on the foot pegs, hands
on the grips. The stock STAP mesh is a 1.4 KB placeholder like every stock vehicle `.mgn`, so the hull is the real
static `stap_speeder.msh` scaled 1.5 (`titan_hrc_nboo_stap_hull.apt`). `droid_unit` hangs it on the creature as an
hp_dyn `app` on `-`; a B1 skeleton has no `saddle`, so it stays at the object origin.

Built by `support/MCP/SwgMayaMCP/examples/make_b1_unfold.py` (CODE / STAP_CODE in headless Maya,
`write_lat_and_sat`, `write_stap_rider`).

### 3.4 Gungan Grand Army

| Piece | Asset |
|-------|-------|
| shield dome | `appearance/titan_hrc_nboo_shield_dome.apt` (r 150 m, rippling lilac membrane + bright ground rim), template `object/tangible/titan/heroic/naboo/shield_dome` |
| generator rig on the fambaa | `appearance/titan_hrc_nboo_shield_generator.apt`, `hp_dyn` "app" on hardpoint `root` of the stock fambaa (scale 1), no offset: the upright pose on its back is baked into the mesh (the root joint is turned); emitter hardpoint `hp_emitter` |
| warriors' energy shields | `appearance/titan_hrc_nboo_hand_shield.apt`, `hp_dyn` "app" on hardpoint `lWrist` of the stock Gungan skeleton (it has no hold_l), no offset: held in front, face forward, fitted in the standing idle |
| booma | `appearance/titan_hrc_nboo_booma.apt` (glowing ball) |
| catapult | `appearance/titan_hrc_nboo_catapult.apt`, template `object/tangible/titan/heroic/naboo/catapult` |
| mobs | `hrc_nboo_gungan_warrior`, `hrc_nboo_gungan_kaadu_rider`, `hrc_nboo_gungan_officer` (Captain Tarpals), `hrc_nboo_fambaa_generator` - stock Gungan/kaadu/fambaa templates, customised |

### 3.5 Jar Jar and the Historical marker

- `object/mobile/titan/heroic/naboo/jar_jar_binks` (stock `gungan_m.sat`, tan-orange skin, scale 1.12): the quest
  giver in Theed and the comm-window head.
- Marker: `appearance/titan_info_disk_historical.apt` (a gold medal disc, ivory Aurebesh Herf, full bright, same
  size and spin as the stock `info_disk`). Server: `setCondition(npc, CONDITION_INTERESTING)` plus an `hp_dyn`
  slot `kind=app, hp=@interesting, path=appearance/titan_info_disk_historical.apt`. Client: `TangibleObject`
  takes an `app` row whose hardpoint is `@interesting` as the override for the interesting marker (older clients
  ignore it and show the stock disc).

### 3.6 Effects

| Use | Effect |
|-----|--------|
| AAT shell flight | weapon effect `titan_aat_plasma` 0 (client `datatables/weapon/weapon.iff`): projectile `appearance/titan_hrc_nboo_plasma_shell.apt`, rocket muzzle flash, large explosion on impact |
| booma flight | weapon effect `titan_booma` 0: projectile `appearance/titan_hrc_nboo_booma.apt`, lightning fire and hit effects |
| hover wash | hp_dyn `fx` on `hp_dust`: stock `appearance/pt_hovering_landspeeder.prt` (AAT scale 3, MTT scale 6) |
| shell impact | stock `clienteffect/combat_explosion_lair_large.cef` |
| shell on the dome | hp_dyn `fx` stock `appearance/pt_droideka_shield.prt` + `app` `appearance/titan_hrc_nboo_shield_hit.apt` (a 9 m membrane flash, orientation-free) on the dome at the impact offset, cleared after 1.5 s |
| booma | booma glow + stock `pt_ball_electric_bolt_green.prt` on impact |
| control ship | stock `restuss_event_big_explosion.cef` high in the sky to the north-east |

---

## 4. Server scripts (dsrc `script/theme_park/heroic/naboo_battle/`)

| Script | Role |
|--------|------|
| `naboo_battle_controller.java` | phases, narration, quest signals, wave triggers, rewards, failure reset |
| `naboo_battle_player.java` | `player_script`: re-syncs quest state on login / late join |
| `mtt.java` | glide-in, door/rack sequence, droid drop, destruction |
| `folded_droid.java` | power-up window, double damage, then normal AI |
| `aat_npc.java` | turret traverse + shell fire for NPC AATs |
| `aat_captured.java` | two-seat mount, gunner command, turret |
| `shield_generator.java` | calibrate radial, HP, arc effect, collapse |
| `shield_dome.java` | rise, hit ripples, flicker and fall |
| `catapult.java` | *Launch Booma* radial |
| `conversation/jar_jar_binks_historical.java` | Theed conversation (STF-less server conversation), instance entry |

Strings: `string/en/theme_park/heroic_naboo_battle.stf` (narration, system messages), `string/en/instance.stf`
(`heroic_naboo_battle`), `string/en/quest/ground/heroic_naboo_battle.stf` (journal).

---

## 5. Verification

- Assets: `swg_check` / static .lod rails; offline client shots of every piece (`/scene load adventure_heroic`,
  `/scene objectCreate ...`), parts posed at state 0 and 1.
- Terrain: client height parity at the pads, the dome rim on the ground.
- Server: `titan.sh java-all` clean, scene loads, controller registers, a dry run of the sequencer (phases advanced
  by GM signal) with no player logged in.

Dry run (no player), from WSL with the scratch helper or `titan.sh console`:

    titan.sh console "game any runScript theme_park.heroic.naboo_battle.naboo_battle_console hrc adventure_heroic <cmd>"

`<cmd>` is `status`, `start`, `advance` (repeat: 0→1→2→3→4→5, then the holdout, then the control ship), `reset`, or
`probe [x z [template part]]`. `hrc naboo probe -5496 4396 jar_jar` checks Jar Jar in Theed.

Screenshots in `shots/`: the offline client (`client_*`: dome, shield warrior, fambaa generator, catapult, Jar
Jar with the Historical disk, spawned locally with `/scene objectCreate` + `/devtest vessel <id> hpdyn|condition`)
and Maya renders (`maya_*`: MTT, AAT, B1 sheet).

Playtest overhaul (2026-10-25), shots `shots/overhaul_*` from the offline client:
- `overhaul_camp_view`: the battlefield from the camp, inside the thinner dome, with rally points at the generators.
- `overhaul_rally_point`: a rally point beside a fambaa. Generator rally points stand 12 m on the camp side.
- `overhaul_b1_folded_vs_standing`, `overhaul_b1_unfolding`: the mood fold and the power-up.
- `overhaul_stap_rider`: a B1 on its STAP.
- `overhaul_line_view_east`: from the line to the MTT pad.

The SSAT value chunk is `VAL ` (TAG3); a `VALS` chunk is skipped silently and hides `DFLT` with it. Offline checks:
`/devtest vessel <id> mood titan_folded` and `/devtest vessel <id> anim appearance/animation/titan_hrc_b1_unfold.ans`.

Fixed by the 2026-10-24 dry runs:
- The buildout table published to `D:/exported` hid the server's controller row (see section 2).
- trial's `getObjectsInInstanceBy*` threw before the first spawn: use `naboo_battle_lib.children()` / `spawned()`.
- A stale phase-1 start (the 3-minute wait for players) restarted phase 1 in the middle of a run that a GM had
  advanced. `hrc_nbooStartPhase` now drops starts for a phase already reached.
