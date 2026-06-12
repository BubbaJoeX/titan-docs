# TangibleDynamics

**Version:** 1.0 — Phase 1 complete  
**Status:** Ready for integration and testing

Physics for tangible objects: three independent force channels (push, spin, breathing) that can be combined on one object. Implemented across server scripts, shared C++, and client C++.

---

## Read order

1. **This page** — scope and checklist  
2. [overview.md](overview.md) — architecture, API, examples  
3. [quick-reference.md](quick-reference.md) — patterns while scripting  
4. [build-and-integration.md](build-and-integration.md) — build and deploy  
5. [files.md](files.md) — file locations

---

## What was built

| Layer | Files |
|-------|--------|
| Server scripts | `tangible_dynamics.java`, `tangible_dynamics_handler.java`, `tangible_dynamics_test.java`, `consts.java` (condition flag) |
| Shared C++ | `TangibleDynamics.h` / `.cpp`, public forwarding header |
| Client C++ | `ClientTangibleDynamics.h` / `.cpp` |

**Condition:** `CONDITION_MAGIC_TANGIBLE_DYNAMIC` (`0x80000000`)

**Force channels:**

- **Push** — linear velocity (world / parent / object space)  
- **Spin** — angular velocity (optional appearance-center pivot)  
- **Breathing** — min/max scale pulse  

---

## Quick start (scripting)

```java
import script.library.tangible_dynamics;

// Push upward for 3 seconds
tangible_dynamics.applyPushForce(target, 0.0f, 2.0f, 0.0f, 3.0f, tangible_dynamics.SPACE_WORLD);

// Spin on yaw for 5 seconds
tangible_dynamics.applySpinForce(target, 3.14f, 0.0f, 0.0f, 5.0f, false);
```

Attach `handler.tangible_dynamics_handler` on objects that should receive dynamics messages. Test interactively with `tangible_dynamics_test.java`.

---

## Validation checklist

Before production deploy:

- [ ] All implementation files present (see [files.md](files.md))
- [ ] `consts.java` defines `CONDITION_MAGIC_TANGIBLE_DYNAMIC`
- [ ] C++ and Java compile cleanly
- [ ] Test menu: push, spin, breathe, combined each work
- [ ] Condition flag sets and clears with effects
- [ ] No measurable perf regression at expected object counts

---

## Phase 2 (not implemented)

Velocity easing, damping, hierarchical propagation, client prediction, VFX hooks, preset combinations.
