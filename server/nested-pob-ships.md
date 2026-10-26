# POB Ships Parked Inside Another POB

A boardable POB ship (a Titan vessel: the starport transport, the yacht) can sit inside a cell of another POB
building, such as the Theed hangar or a bay of the Titan atmo starport. Players walk from the hangar cell up the ramp
into the ship's cells and back out, exactly as they would with a ship parked on the terrain.

## Limits

- **No flying through hangar doors.** A ship is created in the cell and stays in it. Neither the client nor the server
  moves a ship between the world cell and a building cell while it flies. A vessel in a hangar can only hover a few
  metres (`vessel.CELL_HOVER_HEIGHT`, 4 m) and set down again.
- One level of nesting is what is exercised: a ship in a building cell. The engine changes are written for "the pob
  that directly owns the cell", so deeper nesting should work, but nothing tests it.
- The ship hull does not block players walking in the hangar, as on the terrain: ships are dynamic objects and are
  not static collision.
- Retiring a vessel in a cell stows it in place, parked with the ramp up, for its service to reuse
  (`takeSpare` with that cell). It is not moved 7 km away like a vessel in the world.

## Engine support (C++)

The engine already handled most of this. `PortalProperty::addToWorld` attaches a pob's exterior portals to its parent
cell, which is a hangar cell here. Rendering, camera, lighting, `CellProperty::getDestinationCell(start, end)`,
`Footprint` and client walking already worked. The fixes:

| Area | File | Change |
|---|---|---|
| Move validation | `serverGame/.../PlayerCreatureController.cpp` `checkValidMove` | The source pob is the one that owns the creature's cell, not the topmost container. A move between a nested pob and the cell it is parked in passes, given a passable portal to the parent cell. A nested pob never connects straight to the world cell. |
| Expel | `PlayerCreatureController::handleInvalidMove`, `BuildingObject::expelObject` | Ship cells have no owner building, which used to dereference null. A creature refused by a ship cell is now put at the ship's ejection point in its parent cell. Ejection transforms use `o2p`, so they are in cell space for a building in a cell. |
| Sphere trees | `serverGame/.../ServerWorld.cpp` | An object inside a nested pob's cell has its extent computed in the topmost pob's space. This applies to objects and trigger volumes. |
| Lighting RPCs | `serverGame/.../CellObject.cpp` `findPortalStructureRoot` | The search stops at the pob that directly owns the cell and does not reach the outer building. |
| Ship flight | `serverGame/.../PlayerShipController.cpp` `realAlter` | A ship in a cell with no flight path is left alone. It no longer runs the world flight model or the terrain + 15 m floor. Flight paths are applied in parent (cell) space. |
| Visibility, interior preload | `serverGame/.../ShipObject.cpp` | A ship in a cell is visible to everyone. It gets the `PortalTriggerVolume`, so its interior loads before players board, on any ground planet. |
| Portal by id | `sharedObject/.../CellProperty.cpp` `getDestinationCell(object, portalId)` | An interior cell matches the exterior portals of a pob attached to it by floor owner, as the world cell does. |
| Adjacency | `CellProperty::areAdjacent` | A nested pob's cells are adjacent to the cell it sits in. Without this the server drops unreliable movement updates between them (`ServerObject.cpp`). |
| Collision | `sharedCollision/.../CollisionWorld.cpp`, `SpatialDatabase.cpp` | In an interior cell, `canMoveInCell` and `findLocators` also use the object floors of pobs parked in that cell. Other object floors in cells are still ignored. `queryFloors(cell, segment)` converts the cell-space segment to world space for the world-space floor tree. |
| Client lerp | `client/.../RemoteShipController.cpp` | Server-correction smoothing converts its world-space result back to parent space before calling `setTransform_o2p`. |

The shared files have two copies, `D:/titan/src/engine/shared` (server) and `D:/titan/client/src/engine/shared`
(client), and both carry the same changes.

## Script API (`script.library.vessel`)

| Call | What it does |
|---|---|
| `obj_id parkInCell(String template, obj_id cell, float x, float floorY, float z, float heading)` | Creates the vessel in `cell` (cell space), parked on its legs over the floor point and facing `heading`. It holds the vessel there with a one-key path and records the floor height in `vessel.cell.floor_y`. |
| `location padLocation(obj_id padObject)` | The pad point for `landedPose`. In the world it is the object's spot on the terrain. In a cell it is the object's own position, since it stands on the floor. Use it in place of `getHeightAtLocation` for a pad controller that may be in a hangar. |
| `float[] landedPose(Config c, location pad, float heading)` | Unchanged. It works in the pad's space: the world, or the cell. |
| `obj_id takeSpare(service, template, cell, x, y, z, heading, pitch)` | Reuses a stowed vessel only from the same cell, or from the world when `cell` is null. The old signature means "the world". |
| `obj_id getParkedCell(obj_id ship)`, `boolean isInCell(obj_id ship)` | The hangar cell the vessel sits in, or null when it is in the world. |
| `float groundHeight(ship, x, z)`, `location groundPoint(ship, x, z)` | The ground in the vessel's parent space: the terrain, or the hangar floor in its cell. |
| `void moveOffShip(ship, player)`, `void setDownPassengers(ship)` | Sets people down beside the vessel. In a hangar this is on the cell floor, using `isValidInteriorLocation` to pull in from a wall. |
| `Path holdPath(float[] pose)` | A one-key path that holds the vessel still at a pose. |
| `takeOffPath`, `landHerePath` | Cell-aware. In a cell they hover only up to `CELL_HOVER_HEIGHT` and land back on the hangar floor. |

`space_transition.disembarkShip` sets people down on the hangar floor when the ship is in a cell. The
`vessel_controller.vesselRetire` handler stows a vessel in a cell in place.

## Testing

From the server console (`titan.sh vessel ...`, handled by `titan.dev_console`):

```
vessel <scene> parkcell <template> <cell oid> <x> <floorY> <z> <heading>   # cell-space floor point
vessel <scene> parkat <template> <oid> [heading]                            # where <oid> stands (its cell or the terrain)
```

To test, stand a non-god character in a hangar cell, run `parkat` with that character's oid, step off the pad, then
walk up the ramp. Gods skip move validation, so use a non-god character to test it.
