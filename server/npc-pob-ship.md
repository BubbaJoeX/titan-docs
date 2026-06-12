# NPC-Controlled POB Ship Controller

An NPC-controlled POB ship that flies to scripted destinations and loads/unloads players via a linked terminal.

## Ship Setup

1. **Create or unpack a POB ship** (ship with interior) in an atmospheric flight scene.
2. **Attach the controller script:**
   - `attachScript(ship, "space.npc_pob_ship_controller")`
   - Or via CE: add `space.npc_pob_ship_controller` to the ship's scripts
3. On attach, the controller sets `npc_pob.controller` and enables public boarding for shuttle use.
4. Ensure the ship has **no pilot** (NPC-controlled = flies itself).

## Scripting Destinations

Send `npcPobFlyTo` to the ship with (x, z) coordinates:

```java
dictionary params = new dictionary();
params.put("x", 100.0f);   // world X
params.put("z", -200.0f);  // world Z
messageTo(ship, "npcPobFlyTo", params, 0, false);
```

The ship will use atmospheric autopilot to fly to the destination and land.

## Terminal Setup

### Boarding (Load) – Terminal at Stop

1. Place `object/tangible/terminal/terminal_npc_pob_shuttle.iff` on the ground at a shuttle stop.
2. As god, stand near the ship when it is parked, use the terminal, and choose **Link Terminal to Ship**.
3. When the ship lands at that stop, players within 128m can use **Board Shuttle**.

### Disembark (Unload) – Terminal Inside Ship

1. Place the same terminal inside the ship (via ship interior datatable or CE).
2. No linking needed; `getContainingShip` is used automatically.
3. Passengers use **Disembark** when the ship is parked and has no pilot.

### Linking (God Mode)

- **Link Terminal to Ship**: Use the shuttle terminal in atmospheric flight, with a POB ship within 128m of the terminal.

## Spawner (Naboo Shuttle Test)

A spawner controller spawns a Sorosuub Space Yacht and cycles through all Naboo city waypoints:
Theed → Keren → Moenia → Kaadara → Deeja Peak → (repeat).

### Spawner Setup

- **Template**: `object/tangible/ground_spawning/npc_pob_ship_spawner.iff`
- **Script**: `space.npc_pob_ship_spawner`
- **Buildout**: Placed in `naboo_2_7.tab` (Theed area) – invisible, GM-only
- **Ship**: `object/ship/player/player_sorosuub_space_yacht.iff`
- **Dwell**: 30 seconds at each stop before flying to next
- **Tick**: Polls every 60 seconds

### Terminal Linking

Place `terminal_npc_pob_shuttle` at each city. As god, use **Link Terminal to Ship** when the shuttle is parked at that stop.

## Technical Notes

- The ship uses `shipAutoPilotEngage` with `npcControlled=true` to bypass the owner check.
- Boarding uses `space_transition.boardShipFromGround`.
- Disembark uses `space_transition.disembarkShip`.
- Ship must be in an atmospheric flight scene and have no pilot.
