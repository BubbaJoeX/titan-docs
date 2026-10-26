# God Client — automation bridge

The God Client (`SwgGodClient_r.exe`) can be driven by tools and agents through a small socket
of its own: select and place objects, set and apply transforms, undo, run console commands, aim
the god camera, take screenshots and paint terrain strokes, all without a human at the mouse.

It is modelled on the game client's DevBridge (see the SwgMayaMCP README, "Offline client
tools"). The framing is the same, but the God Client has its own port, config key and token file.
The bridge never touches the game client's `devbridge.json`, which game-client sessions pin
their tokens from.

## Turning it on

The bridge is off unless `SwgGodClient.cfg` (next to the exe in `exe/win64_rel`) sets a port:

```
[GodClient]
	bridgePort=7201
```

The God Client ignores `-s Section key=value` on its command line, so the cfg is the only switch.
Port 7201 keeps clear of the game-client DevBridge ports (7031–7101 so far; see the session
coordination notes). 7101 was the first choice, but a benchmark client took it. If the port is
busy, the bridge stays off: the God Client logs "cannot listen" and writes no token file. With the
key set and the port free, the God Client:

- listens on **127.0.0.1 only**, so other machines can't reach it, and serves one connection at a
  time (the newest one wins);
- writes `%LOCALAPPDATA%\SwgTitan\godclient_bridge.json` with `{port, token, pid}` at start-up,
  using a fresh random 128-bit token, and deletes the file on a clean exit;
- prints `God Client bridge: listening on 127.0.0.1:7201` in the Console dock.

Every request must carry the token. A wrong token closes the connection.

**Offline scenes in the release build.** Release builds compile with `PRODUCTION == 1`, which
hides the login screen's Load Scene (Local) button and compiles out the game client's DevBridge.
The God Client now keeps that button (`SwgCuiLoginScreen::performActivate` shows it when
`Game::isGodClient()`; the game client's release login is unchanged). The bridge's `loadscene`
does the same from a script. Offline, `create` places objects locally, and nothing answers
`console`.

**Known caveat: loading a scene stalls the release God Client for minutes.** This applies to the
button and to `loadscene` alike. The whole scene is built synchronously inside one frame, so the
editor and its bridge stop answering until it is done. Ping and state then time out, and new
connections are refused. What was observed on 2026-10-24:

- One load got through after Windows logged a recovered hang (`AppHangTransient`, 09:45). That
  editor was in its scene two minutes later.
- Three other loads were closed while stalled: after about 30 s, and twice after several minutes.
- The cause isn't known: no stack was captured during a stall. A watcher waiting for Windows'
  "Not responding" flag (`IsHungAppWindow`) never fired, because the stalled editor still pumps
  window messages. The next attempt should take its dumps when the bridge stops answering.

Until that is fixed, start the load and leave the window alone until `state` answers again with
`ready:true`.

Don't set `[ClientGame] devBridgePort` for the God Client. That key belongs to the game-client
bridge, which `Game::runGameLoopOnce` also runs in the editor, and it would overwrite
`devbridge.json`.

## Protocol

One JSON object per line in each direction, the same as DevBridge:

```
→ {"id":1,"cmd":"ping","token":"…"}
← {"id":1,"ok":true,"result":"pong"}
```

When a command is refused, the reply has `"ok":false` and the reason in `result`, or in `error`
for an unknown command or a bad token.

The editor answers between frames: `GameWidget::runGameLoop` polls the bridge right after
`Game::runGameLoopOnce`. A reply therefore takes about one frame, and longer when the window
isn't active (half the frame rate) or is minimised (1 frame per second). A command that opens a
modal dialog (for example, opening buildout files through Perforce) holds the bridge until
someone closes the dialog.

| Command | Fields | What it does |
|---|---|---|
| `ping` | | `"pong"` |
| `loadscene` | `terrain`, `avatar` | From the login screen, loads an offline single-player scene, as the Load Scene (Local) button does. Defaults: `terrain/simple.trn`, `shared_human_male`. Poll `state` until `ready`. |
| `state` | | Scene id, `single_player`, `ready` (a scene is loaded with its avatar), the god camera, the cursor's world point, the avatar's cell, the selection (first 50), ghost count, the copied transform, drop/align-to-terrain, transform space, and undo state for objects and terrain (armed tool, brush size and strength). |
| `selection` | | Every selected object: `id`, `template`, `name`, `cell`, `building_id`, `pos_w`, `pos_p`, `yaw`/`pitch`/`roll` (degrees), `scale`, and `ghost` (its pending transform, or null). |
| `select` | `ids` [..], `template`, `add`, `clear` | Select by network id and/or every loaded object of a template, replacing the selection unless `add`. Objects the editor won't select come back in `rejected` with the reason: no appearance, or a different cell from the avatar. |
| `objects` | `center` [x,y,z], `radius`, `template`, `limit` | The loaded objects nearest a point (default: the camera pivot). `template` is a substring filter. |
| `create` | `template`, `pos` [x,y,z], `yaw`, `select` | Places the template in the world cell, the same way a cursor placement does: Recent Templates, and buildout files if the user chose to open them, but never a dialog. Offline the object exists at once (`mode:"local"`). Connected, the server creates it (`mode:"server"`), and it is selected when it arrives. |
| `transform` | `pos`, `yaw`, `pitch`, `roll`, `scale`, `copy`, `paste`, `apply` | Gives every selected object a ghost and sets the parts you send. Details below. |
| `apply` | | Moves the selection to its ghosts (Ctrl+T / Space). To cancel ghosts instead, use `select` with `clear` (like Esc). `undo` doesn't cancel them, because only applied edits enter the history. |
| `undo`, `redo` | `target`: `auto`, `objects`, `terrain` | `auto` behaves like Ctrl+Z / Ctrl+Y: terrain strokes first while a terrain tool is armed, otherwise object edits. |
| `console` | `line`, `wait` (s), `target` | A Console dock line. Details below. |
| `log` | `since` | Console lines after a sequence number: server replies (with `message_id`) and the lines the bridge sent. |
| `camera` | `pivot`, `distance`, `yaw`, `pitch`, `smooth` | With no fields, reads the god camera (`FreeCamera::Info`, angles in degrees). With any of the fields, sets those and keeps the rest. `smooth` glides like a bookmark jump. |
| `screenshot` | `path` | The 3D view from the back buffer, so menus, docks and windows over it don't show. The engine adds the extension; the reply has the full path and `source`. Default: `screenshots/godbridgeNNNN`. Details below. |
| `terrain_stroke` | `points` [[x,z],..], `tool`, `size`, `strength`, `spacing` | One brush stroke through world x/z points, resampled every `spacing` m (default a quarter of the brush size), like a mouse drag. It is one terrain undo step. |

### `transform` fields

- `pos` [x,y,z] is in parent space: world coordinates outside buildings, cell coordinates
  inside, as in the Set Transform dialog. It moves **every** selected object to that point, so
  give each object its own call when they should stay apart.
- `yaw`, `pitch` and `roll` (degrees) replace the whole rotation. They are applied yaw, then
  pitch, then roll, in object space, and a missing angle counts as 0. The values in `selection`
  use the same convention, so they round-trip.
- `scale` is [x,y,z] or a single number.
- `copy: 1` copies the primary selection's transform. `paste` then gives the copied position,
  rotation, or all three (with scale) to every selected object. This is the transform clipboard
  (Ctrl+Shift+C/V).
- `apply: 1` applies straight away. Otherwise the ghosts wait for `apply`, so you can look at them
  first. With Drop to Terrain or Align to Terrain switched on, applying drops or aligns the
  objects too, as it does by hand.

### `console` fields

- Text without a leading `/` (`object get <id>`, `objvar list <id>`) goes to the server console.
  The server echoes the request's message id, so with `wait` the reply comes back in the same
  call once it arrives (`answered:true`), or when the wait runs out (`answered:false`, at most
  30 s).
- `/name args` runs a slash command on `target`, or on the primary selection if there is no
  `target`. Slash commands have no reply.
- Offline (single player) nothing answers.

### `screenshot` notes

`Graphics::screenShot`, which F12 also uses, copies the front buffer by default: the desktop at
the corner of the engine's window. In the God Client that window is the whole editor frame, so
the capture shows menus, docks and anything lying over them. The bridge forces the back buffer
instead (`Graphics::setForceScreenShotBackBuffer`). That works because a windowed device swaps
with `D3DSWAPEFFECT_COPY`, so the back buffer still holds the frame just presented. If the back
buffer has a format the engine can't save, the bridge falls back to the front buffer, and
`source` says which one it used.

### `terrain_stroke` fields

- `tool` is one of `raise`, `lower`, `flatten`, `smooth`, `noise`, `setheight`, `shader`, `flora`,
  `water` or `radial`, or empty to use the dock's armed tool.
- `size` (m) and `strength` (0–1) go on the Terrain dock sliders.
- Everything else comes from the dock: shader family, set-height target, falloff, raise/lower
  speed. Raise and lower add their speed on every dab, so a denser path builds up more.
- The stroke needs procedural terrain loaded in the Terrain dock. Save the `.trn` from the dock
  as usual.

## From tools

**MCP (SwgMayaMCP).** The `godclient_*` tools wrap every command. The God Client signs in with
the credentials in `user.cfg`, so agents never launch it: the user starts it, and the tools
attach to it.

| Tool | Command |
|---|---|
| `godclient_state` | `state` |
| `godclient_load_scene` | `loadscene` |
| `godclient_objects` | `objects` |
| `godclient_select` | `select` |
| `godclient_create` | `create` |
| `godclient_transform`, `godclient_apply` | `transform`, `apply` |
| `godclient_undo`, `godclient_redo` | `undo`, `redo` |
| `godclient_console`, `godclient_log` | `console`, `log` |
| `godclient_camera` | `camera` |
| `godclient_screenshot` | `screenshot` (returns the image) |
| `godclient_terrain_stroke` | `terrain_stroke` |

**Scripts.** `swg_maya_mcp.godclient.GodClientBridge` reads the token file and reconnects
whenever the editor restarts:

```python
from swg_maya_mcp.godclient import GodClientBridge
gc = GodClientBridge()
gc.call("ping")                      # "pong"
st = gc.call("state")
ids = [o["id"] for o in gc.call("objects", radius=30, template="frn_")["objects"]]
gc.call("select", ids=ids[:1])
gc.call("transform", yaw=90, apply=True)
gc.call("undo")
```

A typical agent loop is: `objects` to find ids, then `select`, `transform` without `apply`,
`selection` (or a screenshot) to check the ghost, `apply`, and `undo` if it looks wrong.

## Source

| File | What |
|---|---|
| `SwgGodClient/src/GodClientBridge.{h,cpp}` | Socket, token, commands. The protocol comment is at the top of the header. |
| `GameWidget::runGameLoop` / `~GameWidget` | `GodClientBridge::update()` after each game frame; `remove()` on shutdown. |
| `GameWidget::createObjectFromTemplateWithTransform` | Cursor placement without the cursor or dialogs, shared with `createObjectFromTemplateAtPoint`. |
| `TerrainDock::scriptedBrushStroke` | Syncs the dock's brush settings, then `GodClientTerrainEditor::beginBrushStroke` / `continueBrushStroke` / `endBrushStroke`. |
| `GodClientData::getSelectionGhost` | A selected object's ghost, for reporting pending transforms. |
| `ConfigGodClient` | `[GodClient] bridgePort`. |
| `support/MCP/SwgMayaMCP/swg_maya_mcp/godclient.py`, `server.py` | Python client and the `godclient_*` MCP tools. |

## Testing notes

- **What was checked:**
  - the build: Release|x64 on 2026-10-24, deployed to `exe/win64_rel/SwgGodClient_r.exe`. The
    previous exe, without the bridge, is `SwgGodClient_r.exe.pre-fps`;
  - the bridge's JSON parsing, in a standalone test of the same helper code: python-style
    spacing, escaped quotes inside strings, ids as strings and numbers, `null` meaning absent,
    and stroke resampling;
  - the Python client against a stub bridge: token file, framing, refused commands, a bad token
    and a missing token file;
  - that the MCP server registers all 13 tools.
- **Against the running God Client** (the user started it; 2026-10-24, release build):
  - at the login screen, `ping` answered `pong` in 41 ms and `state` answered in 29 ms
    (`ready:false`);
  - in the offline scene `godclient_test`, these all answered in well under a second: `state`
    (camera, avatar cell, a selected Atmo starport with its transform and `ghost:null`, undo and
    terrain-dock state), `objects`, `camera` (read), `log` and `screenshot`;
  - the first `screenshot` showed the editor's menus and docks. That is how the back-buffer fix
    above was found.
  - `loadscene` started the scene load, but the window was closed after about 30 s, during the
    synchronous load. See the Load Scene caveat above.
  - The back-buffer screenshot fix is in the deployed 09:53 build, checked in the binary. It
    hasn't been run in a live editor yet.
  - Not exercised live yet: `select`, `create`, `transform`/`apply`, `undo`/`redo`,
    `console`, `terrain_stroke`. A non-destructive test is written: select, ghost a transform,
    check the ghost, discard it by clearing the selection, never apply. It needs a loaded scene,
    and the stall prevented one.
  - Don't use `undo` to cancel a ghost: ghosts aren't in the undo history, so `undo` reverts the
    last real edit instead. Clearing the selection (`select` with `clear`) discards unapplied
    ghosts.
- **To repeat the round trip:** set `bridgePort`, start the editor, then run:

  ```
  support\MCP\SwgMayaMCP\.venv\Scripts\python.exe -c "from swg_maya_mcp.godclient import GodClientBridge as G; g=G(); print(g.call('ping')); print(g.call('state'))"
  ```

  from `support/MCP/SwgMayaMCP`. Ping and state work at the login screen; the editing commands need a scene.
