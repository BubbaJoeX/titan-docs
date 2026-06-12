# TangibleDynamics — File map

Package: [README.md](README.md) · [overview.md](overview.md) · [quick-reference.md](quick-reference.md) · [build-and-integration.md](build-and-integration.md)

Paths are relative to the Titan monorepo (`dsrc`, `src`, `client` sibling repos).

---

## Server scripts (`dsrc`)

| File | Role |
|------|------|
| `dsrc/sku.0/sys.server/compiled/game/script/library/consts.java` | `CONDITION_MAGIC_TANGIBLE_DYNAMIC` |
| `dsrc/sku.0/sys.server/compiled/game/script/library/tangible_dynamics.java` | Public API |
| `dsrc/sku.0/sys.server/compiled/game/script/handler/tangible_dynamics_handler.java` | Message handler |
| `dsrc/sku.0/sys.server/compiled/game/script/test/tangible_dynamics_test.java` | Interactive tests |

Developer UI and RT-camera lock-to-parent: `dsrc/sku.0/sys.server/compiled/game/script/developer/bubbajoe/player_developer.java`.

---

## Shared C++ (`src`)

| File | Role |
|------|------|
| `src/engine/shared/library/sharedObject/src/shared/dynamics/TangibleDynamics.h` | Server dynamics class |
| `src/engine/shared/library/sharedObject/src/shared/dynamics/TangibleDynamics.cpp` | Implementation |
| `src/engine/shared/library/sharedObject/include/public/sharedObject/TangibleDynamics.h` | Public include wrapper |

---

## Client C++ (`client`)

| File | Role |
|------|------|
| `client/src/engine/client/library/clientGame/src/shared/dynamics/ClientTangibleDynamics.h` | Client dynamics |
| `client/src/engine/client/library/clientGame/src/shared/dynamics/ClientTangibleDynamics.cpp` | Client implementation |

Linked via `client/src/engine/client/library/clientGame/build/win32/clientGame.vcxproj` (see [build-and-integration.md](build-and-integration.md)).

---

## Documentation (this repo)

| File | Role |
|------|------|
| [README.md](README.md) | Package index |
| [overview.md](overview.md) | Architecture |
| [quick-reference.md](quick-reference.md) | Scripting API |
| [build-and-integration.md](build-and-integration.md) | Build / deploy |
| [files.md](files.md) | This file |

Published at [github.com/BubbaJoeX/titan-docs/tree/main/packages/tangible-dynamics](https://github.com/BubbaJoeX/titan-docs/tree/main/packages/tangible-dynamics).

---

## Dependency flow

```
tangible_dynamics.java
    → tangible_dynamics_handler.java (objvars + messages)
        → TangibleDynamics.cpp (server alter)
            → ClientTangibleDynamics.cpp (client visuals)
```

---

## Integration checklist

- [x] Condition constant in `consts.java`
- [x] Server API and handler
- [x] Shared and client C++ dynamics
- [x] Test script and developer tooling
- [x] Documentation in titan-docs
