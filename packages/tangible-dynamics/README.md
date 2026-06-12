# TangibleDynamics

Server-driven motion and effects for tangible objects, mirrored on the client for visuals.

**Authority:** `dsrc/sku.0/sys.server/compiled/game/script/library/tangible_dynamics.java`  
**Handler:** `handler.tangible_dynamics_handler`  
**Condition:** `CONDITION_MAGIC_TANGIBLE_DYNAMIC` (`0x80000000` in `consts.java`)

---

## Read order

1. **This page** — scope and checklist  
2. [overview.md](overview.md) — architecture and integration  
3. [quick-reference.md](quick-reference.md) — methods, patterns, troubleshooting  
4. [build-and-integration.md](build-and-integration.md) — compile and deploy  
5. [files.md](files.md) — source locations in `dsrc`, `src`, `client`

---

## Effect families

| Family | Examples | Java entry points |
|--------|----------|-------------------|
| **Linear / spin** | Push (with drag), spin, combined push+spin+breathe | `applyPushForce`, `applySpinForce`, `applyCombinedForces` |
| **Scale / motion** | Breathing, bounce, wobble, shake, sway | `applyBreathingEffect`, `applyBounceEffect`, `applyWobbleEffect`, … |
| **Paths** | Orbit, carousel, ferris wheel, conveyor | `applyOrbitEffect`, `applyCarouselEffect`, `applyConveyorEffect` |
| **Anchored** | Hover, float, follow target, lock to parent | `applyHoverEffect`, `applyFloatEffect`, `applyFollowTargetEffect`, `applyLockToParentEffect` |
| **Rides** | Mountable platforms, hover platform, carousel seat | `makeMountable`, `createHoverPlatform`, `createCarouselRide` |
| **Easing** | Ease in/out on force changes | `setEasing` |

Shortcuts (`push`, `hover`, `levitate`, `carousel`, …) are defined at the bottom of `tangible_dynamics.java`.

---

## Quick start

```java
import script.library.tangible_dynamics;

// Attach handler on the object template or at runtime:
// handler.tangible_dynamics_handler

tangible_dynamics.applyPushForce(target, 0.0f, 2.0f, 0.0f, 3.0f, tangible_dynamics.SPACE_WORLD);
tangible_dynamics.applyHoverEffect(target, 1.5f);  // bob at 1.5m
```

Interactive tests: `dsrc/.../script/test/tangible_dynamics_test.java` and developer tooling in `player_developer.java`.

---

## Validation checklist

- [ ] Handler attached where effects are used  
- [ ] `CONDITION_MAGIC_TANGIBLE_DYNAMIC` sets/clears with active effects  
- [ ] C++ and Java compile in `src` and `dsrc`  
- [ ] Client `ClientTangibleDynamics` linked in `clientGame`  
- [ ] Spot-check: push, spin, breathe, hover, lock-to-parent (RT camera follow uses lock-to-parent)  
- [ ] No perf regression at expected concurrent effect counts  

---

## Documentation in this package

| File | Role |
|------|------|
| [overview.md](overview.md) | Design and integration |
| [quick-reference.md](quick-reference.md) | API patterns |
| [build-and-integration.md](build-and-integration.md) | Build steps |
| [files.md](files.md) | File inventory |
