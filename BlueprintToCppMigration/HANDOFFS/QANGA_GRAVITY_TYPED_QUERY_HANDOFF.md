# GravityScape typed world query / QAI resolver retirement

## Changed files

- `Plugins/GravityScape/Source/GravityScape/Public/QGravityArea.h`
- `Plugins/GravityScape/Source/GravityScape/Public/GravityScape_SubSystem.h`
- `Plugins/GravityScape/Source/GravityScape/Private/GravityScape_SubSystem.cpp`
- `Plugins/GravityScape/Source/GravityScape/Private/QGravityArea.cpp`
- `Plugins/QAI/Source/QAI/Public/Component/QAI_FloatingPawnMovement.h`
- `Plugins/QAI/Source/QAI/Private/Component/QAI_FloatingPawnMovement.cpp`
- `Plugins/QAI/Source/QAI/Private/Animation/QAI_AnimInstance.cpp`
- `Plugins/QAI/Source/QAI/Private/Agent/QAI_AgentComponent.cpp`
- `Plugins/QAI/Source/QAI/Private/Processor/QAI_MovementProcessor.cpp`
- `Plugins/QAI/Source/QAI/Private/Impostor/QAI_ImpostorSubsystem.cpp`
- Deleted: `Plugins/QAI/Source/QAI/Private/QAI_GravityAreaResolver.h`

## Ownership and result semantics

- `UGravityScape_SubSystem::QueryGravityAtWorldPosition` is the sole public native world-position query. It reads the canonical registry's published value snapshot under `FRWLock`; the public result contains no `UObject` pointer.
- Registry publication and the private `UQGravityAreaComponent` area-pointer bridge are game-thread asserted. The component now reuses the same priority/containment selector instead of performing an overlap scan.
- Selection uses exact scaled sphere and oriented-box bounds. Strictly higher priority replaces the current winner; equal priority retains the first snapshot entry.
- `Valid`: finite nonzero signed gravity, with normalized `GravityDirection` and `SurfaceUp`.
- `Directionless`: winning area with zero scale, zero fixed direction, or point-gravity queried at its center; direction outputs are zero.
- `Unavailable`: no winning area or non-finite/invalid input or contract; no actor-up/world-up value is fabricated.
- Animation clears aim offsets, impostors invalidate ground state, dormant movement zeros velocity, vehicle altitude hold is cleared, and other rotation/aim consumers deliberately skip writes on non-valid results.
- Movement-processor queries remain in the game-thread apply pass. `NativeThreadSafeUpdateAnimation` consumes only game-thread snapshots. J4 remains character gravity direction/scale owner; QAI/AILean remains AI rotation writer.

## Removed

- Resolver reflection into `Lib_GravityArea`, its per-actor refresh cache, radius-only approximation, actor/FPM/world-up fallbacks, and resolver-owned state/helpers.
- Gravity-area component overlap selection and its obsolete collision-channel resolution/log state.
- Production source references to `QAI_GravityAreaResolver`: zero. Two stale documentation references remain intentionally untouched by instruction.

## Remaining blockers / risks

- No migrated consumer is blocked.
- `FixedGravityDirection_Arrow` is movable and direct runtime transform changes do not call `NotifyQGravityAreaChanged`; the published worker snapshot can therefore retain the prior fixed direction until another provider mutation. No production C++ mutator was found, but this needs an explicit event/setter publication hook before claiming parity for arbitrary runtime arrow rotation.
- `CustomShape` removal is pre-existing dirty work from another lane, not introduced here. Current live source exposes only sphere/box, while historical assets may still serialize enum value `2`; the owning migration must reconcile that contract. This lane did not touch assets or restore that unrelated shape path.

## Static verification completed

- Independent static review plus IWYU/non-unity correction.
- `rg` over production `*.h`/`*.cpp`: zero resolver references.
- Targeted `git diff --check`: clean.
- No build, UHT, QATS, Editor/RzMCP, PIE, asset mutation, cook/package, stage, or commit was run. Compilation and runtime parity are not claimed.

## Tests Sol must run

- `QATS.GravityScape.QangaGravity.Sampling.ShapeBoundaries`
- `QATS.GravityScape.QangaGravity.Selection.PriorityAndZeroOverride`
- `QATS.GravityScape.QangaGravity.Registry.UnregisterDirtyAndDuplicate`
- `QATS.GravityScape.QangaGravity.Provider.LegacyComponentActivationAndTick`
- `QATS.GravityScape.QangaGravity.Provider.NativeLifecycleAndContract`
- `QATS.GravityScape.QangaGravity.Character.NativeDirectionAndScale`
- `QATS.GravityScape.QangaGravity.Character.SurfaceQuery`
- `QATS.GravityScape.QangaGravity.Configuration.NetModeGate`
- `QAI.Movement.DormantAnchorCatchUp`
- `QAI.Movement.DormantGroundAuthority`
- Runtime in standalone plus meshless dedicated server/client: sphere/rotated-box boundary and overlap priority/tie; no area; zero/negative scale; point center; zero and rotated fixed direction; NaN/Inf rejection; animation aim neutralization; FPM simulated-proxy facing; autonomous/drone facing; mounted vehicle weapon; vehicle altitude-hold clearing and target-local up; impostor ground-cache invalidation; dormant boundary crossing; cyborg J4 gravity ownership and QAI/AILean rotation ownership.
