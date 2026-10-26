# Private vessels: Lambda shuttle, Corellian corvette, Nebulon-B frigate

Three boardable (POB) ships a player owns through a deed, lands next to themselves, walks aboard and flies from the
inside. They join the transport shuttle and SoroSuub yacht built on the same vessel framework
(`script.library.vessel`).

| Vessel | Template | Size | Interior |
|---|---|---|---|
| Lambda-class Shuttle | `object/ship/player/player_titan_lambda_shuttle.iff` | 26 m tall (fin up) | Stock cabin and cockpit; the ramp drops through a floor hatch under the cockpit corridor |
| Corellian Corvette | `object/ship/player/player_titan_corellian_corvette.iff` | 266 m | Stock bridge, halls and officers' quarters, extended aft: spine and cross halls, lounge, crew quarters, companionway, embarkation room, 29 m ventral ramp |
| Nebulon-B Frigate | `object/ship/player/player_titan_nebulon_frigate.iff` | 472 m | Embarkation room, a four-landing lift, medical bay, crew deck, bridge hall and a bridge with a four-pane forward viewport |

## Player starships (stock JTL path)

The three ships are player POB starships with their own chassis (`player_titan_lambda_shuttle`,
`player_titan_corellian_corvette`, `player_titan_nebulon_frigate`: rows in `ship_chassis.tab`, `shiptype.tab`,
`chassis_modifiers.tab`; hit points 9 000 / 60 000 / 90 000, handling tuned for atmospheric flight). They come fitted
out: `space_crafting.createChassisFromDeed` keeps their `shiptype.tab` components (`titan_*` types).

- **Starship deed** (`object/tangible/ship/crafted/chassis/titan_<name>_deed.iff`): use it to put the ship's control
  device (`object/intangible/ship/titan_<name>_pcd.iff`) in the datapad.
- **Datapad → Atmospheric Flight → Launch**: the ship appears 200 m up with the owner in the pilot seat, legs up.
- **Stations**: pilot seat (`shipcontrol_corvette`, the `ship_pilot_pob` slot) and operations seat in the cockpit /
  bridge, the starship management terminal, and an escape hatch ("Depart through Boarding Ramp") by the entrance.
- **Landing on the legs** (the atmospheric autopilot only hovers; these ships set down on their gear):
  - leave the pilot seat less than 120 m over level, open ground and the ship sets down;
  - Land Here on the helm console does the same from a hover;
  - the autopilot (landing pads, planet map) lowers the legs as its descent starts; a summoned ship (Starship Remote)
    sets down beside its owner with the ramp lowered.
  The ground check (level under the hull plan, no building, static or other hull on it) is `vessel.padIsClear`.
- **Taking off**: sit in the pilot seat of a landed ship; it raises the ramp, lifts to a hover, retracts the legs and
  hands over the controls (`vessel_controller.vesselPilotSeated`). Taking the seat during a scripted landing cancels it.
- **Datapad → Land Ship** stores it; anyone aboard steps off beside the hull.

## Called with a vessel deed (older path)

- **Deed** (`object/tangible/titan/vessel_deed/titan_<name>_deed.iff`, script `space.vessel.vessel_deed`), from the
  owner's inventory:
  - **Call Vessel** flies the ship in and sets it down broadside to the caller (22 / 55 / 75 m out). It tries ahead,
    then left, right and behind, and takes the first pad where the whole hull plan is level and clear of buildings,
    statics and other vessels' hulls; otherwise it asks the player to move to open ground. The ramp is down when the
    owner arrives. A ship parked or boarding elsewhere is brought round the same way; the first call creates it.
  - **Send Vessel Away** lifts an empty landed ship off and stows it until it is called again (POB ships are never
    destroyed in ground scenes).
  - **Lower / Raise Vessel Ramp** within 400 m (the hull's own radial is out of reach on the big hulls).
- **Helm console** inside (Lambda: aft cabin; corvette: mid-bridge; Nebulon: bridge command console) and the hull's
  radial: **Flight** (ramp, take off to a hover, land here), **Paint** (hull and trim colour from a 64-colour palette,
  previewed on the hull) and **Name Vessel**.
- **Logging out aboard** keeps the ship where it is and puts the owner back in the same spot on login.
- Lifts on the Nebulon use the stock elevator terminals.

## Admin and test commands

`titan.sh vessel <scene> ...` (WSL):

| Command | Does |
|---|---|
| `deed <deed template> <firstname>` | Puts a deed (vessel or starship deed) in a character's inventory |
| `datapad <firstname>` | Lists a character's datapad: ship control devices and the ship each holds |
| `paint <ship oid> hull\|trim <index>` | What the colour picker's OK sets |
| `warp <firstname> <x> <z>` | Moves a test character on the scene |
| `near <x> <z> <radius>` | Lists vessels and their state |
| `retick all` | Restarts every vessel's update loop on the scene |
| `restart <service oid>` | Resets a stuck shuttle service |

## Engine changes they rely on (client and server copies)

- **Floor hatches:** a ramp cell named `hatch*` lies under a deck people also walk over. Portals into it are crossed
  only while the cell admits the walker (`Portal::isPassable` and the position-change transition in
  `CellProperty.cpp`), and footprints keep no floor contact across a hatch (`Footprint::sweepContacts`).
- **Walker collision:** POB ship templates are filed as statics in the collision spatial database, and creatures
  collide with a ship only while in the world cell, so the hull's collision boxes stop walkers outside without
  blocking the ramp or the interior. The boxes cover only hull parts within 2 m of the ground, plus each landing leg.
- **Paint on parts:** hp_dyn parts (the Lambda's wings) wear the owner's customization (`TangibleObject`).

## Art pipeline

Built by `support/MCP/SwgMayaMCP/examples/make_titan_vessels.py lambda|corvette|nebulon` (Maya, templates,
tables, sync). See the SwgMayaMCP README, section "Lambda, corvette, Nebulon-B", for the build details: stock
hull re-export (with the UV and texture-coordinate-set fixes), paint shaders, room kit, landing gear (oleo legs,
truss towers for the Nebulon's engine supports), collision and the live test recipe. The package ships as
`client/releases/SwgMayaMCP-<date>.zip`.

## Known limits

- Ships called before a change to their interior table keep the old furniture; call from a new deed to see it.
- A hot script reload does not reload nested Java classes (`vessel.Config`): adding fields there needs a cluster
  restart.
