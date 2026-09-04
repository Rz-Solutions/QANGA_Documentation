# Weapon orchestration migration

## Scope and ownership

This lane migrates the first player-firearm orchestration vertical from Blueprint to the existing `QWeapon` plugin. It does not migrate presentation and it does not create a second ballistics implementation.

- Native owner: `UQWeaponFireControlComponent` in `QWeapon`.
- First reference asset: `/Game/Items/Weapons/NashV2/Assault_Rifle/IS_NashV2_Assault_Rifle`.
- Existing ballistics owner: `UQWeaponBulletSubsystem`.
- Existing targeting owners remain `UQWeaponTargetRegistrySubsystem` and `UQWeaponTargetingComponent`.
- Inventory, Combat, QModule item stats, animation, audio, recoil, VFX, meshes, sockets, and designer tuning remain owned by their current domains.
- Phase 2 changes source only. Asset integration and Blueprint stripping are prompt 09 work.

## Audited baseline

### WeaponScript identity and topology

The exact base asset is `/Game/Systems/Weapon/WeaponScript.WeaponScript` with generated class `WeaponScript_C`.

- Parent: `ItemScriptBase_C`.
- Kind: replicated Actor.
- Interfaces: `QangaInputsInterface_C`.
- Components owned by the base include `PhaseComponent`, `FireLocation` (`USceneComponent`), and `CustomBoundsCheck`.
- Size: 41 variables, 40 functions, one 266-node EventGraph, 23 events, and 973 total nodes.
- Direct children include the NashV2 firearm family, the older AT56/FA62/MZ56/NASH families, sniper, shotgun, grenade, recycler, melee, AI and Noel variants.
- The live Asset Registry reports 106 referencers. They include item data/assets, animation/presentation users, projectile classes, QAI paths, and vehicle-facing systems. Bulk child conversion is explicitly out of scope.

Base defaults relevant to the contract are `BaseAmmo=1`, `CurrentAmmo=0`, `FireDelay=0.2`, `ReloadTime=2.0`, `Damage=35`, `bUsesHitscan=true`, `InfiniteAmmo=false`, `AutoReload=false`, `IsReloading=false`, `IsFireBlocked=false`, and `IsAutoFireEnable=true`. `CurrentAmmo`, `CurrentAmmoType`, and `CurrentAimpointTag` are replicated in Blueprint.

### Exact Blueprint entry points and current ownership

| Entry point | Exact audited signature / RPC flag | Current behavior and owner |
|---|---|---|
| `Combat_1stTrigger` | Interface function, input `Pressed: bool` | Child weapon input graph starts/stops `AutomaticLoopFireDelay`, then wall check and aim calculation. |
| `TriggerFire` | Local custom event, inputs `FireLocation: FVector`, `FireRotation: FRotator` | Sends `SV_TriggerFire`; locally controlled caller also runs `PreImplementFire` for immediate presentation. |
| `SV_TriggerFire` | Reliable client-to-server custom event, same two inputs | Immediately invokes `MC_TriggerFire`; it does not revalidate cadence, active/equipped item, reload, fire block, or owner context. |
| `MC_TriggerFire` | Server multicast custom event, same two inputs; not marked Reliable in the live graph | Owning locally controlled pawn skips the duplicate path; other peers invoke `PreImplementFire`. |
| `PreImplementFire` | Local custom event, same two inputs | If ammo is positive, calls `UQWeaponBulletSubsystem::ServerFireBullet` once, calls `SpawnBulletTracer` once, and calls the child `ImplementTriggerFire` hook. It stores `TriggerTime`. The empty-magazine path still calls the child hook for click feedback. |
| `ImplementTriggerFire` | Local child hook, same two inputs | Reference child decrements one round, emits `CallFired`, then owns recoil/audio/VFX/timeline presentation. |
| `TraceTest` | Function `(StartLocation: FVector, Direction: FRotator, Length: double) -> (Hit: bool, OutHit: FHitResult)` | Line trace used only by Blueprint aim construction. It is not the damage trace and must not be copied into native ballistics. |
| `GetInfiniteAmmo` | Function `() -> bool`; live output pin is misspelled `InfiteAmmo` | Returns the base infinite-ammo flag used by reload/ammo presentation. |
| `ConsumeMagazineAmmo` | Function `(QuantityToConsume: int32)` | Computes `CurrentAmmo - Quantity`, then calls `SetCurrentAmmo`. |
| `SetCurrentAmmo` | Function `(NewCurrentAmmo: int32)` | Authority-only clamp, item-instance `CurrentAmmo` persistence, notification, and item update. It has no typed failure or rollback. |
| `ReloadAmmo` | Function `(AvailableAmmo: int32) -> QuantityUsed: int32` | Authority-only magazine refill up to `BaseAmmo`. |
| `TriggerReload` / `SV_Reload` / `MC_Reload` | Local request; Reliable client-to-server; Reliable multicast | Uses `IsReloading` plus a latent `Delay(ReloadTime)`. It increments the magazine before consuming inventory ammo and does not roll the magazine back if inventory consumption fails. |
| Reload interruption | No gameplay interruption API exists | `QWeaponAnimInstance` can restore visual state when a montage ends/interruption occurs, but the WeaponScript reload delay and later ammo mutation continue independently. |
| Owner/equipped validation | `ItemScriptBase_C.OwnerPawn`, `TryGetOwnerPawn`; Inventory `IsEquipmentReady`, `GetSlotByItemInstanceId`, `GetActiveItem` | Fire currently checks Combat enable and hidden state in parts of the client graph. It does not perform one authoritative equality check across actor owner, owner pawn, exact item instance, equipped slot, active item, reload, and fire block. |
| `FireLocation` | Inherited `USceneComponent` | The base aim graph ultimately uses the component world location/rotation. The reference AR overrides its relative location to approximately `(52.622, 0, 10.175)`. Native fire must use the resolved component transform, never an actor-location approximation. |
| Cosmetic replication | `MC_TriggerFire` plus the locally controlled skip | Produces one presentation path per peer today, but ballistics and presentation are interleaved in `PreImplementFire`. |

`AutomaticLoopFireDelay` is the only macro in `/Game/Systems/Weapon/WeaponScriptMacros`. It uses a latent `Retriggerable Delay(FireDelay)` and refuses to fire while `GetIsReloading` is true. It is client-scheduled; the server does not enforce its deadline.

### Shotgun discrepancy resolved from live state

`/Game/Items/Weapons/ShotGun/IS_Shotgun` is not the reference vertical.

- Its inherited active `Damage` default is `0`.
- Its child-only `Damage_0` default is `60`.
- The phase graph writes `Damage_0` values `200`, `230`, `250`, and `270`.
- The only `Get Damage_0` node is unlinked.
- Base `PreImplementFire` passes inherited `Damage` to `ServerFireBullet`.

Therefore the current shotgun hitscan request carries zero damage. `Damage_0` is dead tuning, not an alternate active damage path. Prompt 09 must clean this asset deliberately; this lane does not silently reinterpret it.

### Reference weapon

The first vertical is `/Game/Items/Weapons/NashV2/Assault_Rifle/IS_NashV2_Assault_Rifle` because it is already the exact firearm class used by QATS (`QuestTestSubsystem`'s `TestFirearmScriptClassPath`) and its phase graph writes the active inherited properties.

- Parent: `WeaponScript_C`.
- Inherited defaults: `Damage=35`, `FireDelay=0.15`, `BaseAmmo=60`, `CurrentAmmo=0`, `CurrentAmmoType=6`, `ReloadTime=1.5`, `bUsesHitscan=true`, `IsAutoFireEnable=true`.
- Phase updates write active `Damage` and `FireDelay`, unlike the shotgun.
- Input flow is `Combat_1stTrigger -> AutomaticLoopFireDelay -> CheckWeaponInsideWall -> CalcAimingAndTriggerFire`.
- Its `ImplementTriggerFire` consumes one round and retains `CallFired`, recoil, audio, muzzle VFX, and weapon timeline presentation.

### Native APIs and dependency constraints

`UQWeaponBulletSubsystem` already exposes the required production boundary:

```cpp
bool ServerFireBullet(
    bool bUsesHitscan,
    APawn* ShooterPawn,
    FVector MuzzleLocation,
    FVector Direction,
    float Range,
    float Damage,
    FVector& OutImpactPoint,
    AActor*& OutHitActor);

void SpawnBulletTracer(
    bool bUsesHitscan,
    FVector MuzzleLocation,
    FVector Direction,
    float Range,
    APawn* ShooterPawn,
    float Damage,
    USceneComponent* MuzzleComponent,
    float TracerSizeScale = 1.0f);
```

The subsystem already owns trace filters, Combat damage dispatch, hitmarkers/camera feedback, and tracer pooling. The new owner performs exactly one `ServerFireBullet` call for each accepted hitscan shot and never performs a line trace.

`QModuleItemRack::GetStat(UObject* ItemInstance, FGameplayTag StatTag, float BaseValue)` is the exact-item stat API. `QWeapon` cannot add a direct `QModule` dependency: `QModule` privately depends on `QAI`, and `QAI` publicly depends on `QWeapon`, so that would create a module cycle. The integration must arrive through a typed adapter implemented in the owning domain. Actor-only `QMOD_GetStat` passthrough and reflected property guessing are forbidden.

`QWeaponAnimInstance` remains the reload montage/notify/magazine presentation owner. Its active-weapon lookup and visual interruption/restoration are not moved into fire control. The native owner emits typed reload presentation events; prompt 09 binds those to the existing animation path.

QAI currently owns a separate per-agent `NextShotTime` for its standalone direct-fire path and calls `UQWeaponBulletSubsystem` directly. It can converge later on the native request/execution contract with an AI-specific context/ammo adapter. QAI is not modified in this lane.

Relevant recent commits checked before fixing the contract:

- `3bd272b02`: native weapon target acquisition/registry migration. Those owners remain intact.
- `fc7c6cde8`: native reload presentation changes in `QWeaponAnimInstance`; presentation remains there.
- `d112e180d`: shared bullet/hitmarker behavior in `QWeaponBulletSubsystem`; the new fire owner routes through it.
- `14d000eb2`: item persistence work; magazine integration must use the exact item-instance contract.

## Native fire-control contract

### Single authority owner

`UQWeaponFireControlComponent` is the only runtime owner of these decisions for an integrated weapon:

1. trigger-held state and polled timestamp cadence;
2. authoritative actor-owner/pawn/item/equipment/active-item/Combat/fire-block validation;
3. current magazine and reload deadline/state;
4. exact muzzle component location plus the complete validated aim rotation;
5. one ballistics dispatch through `UQWeaponBulletSubsystem`;
6. one multicast fire-cosmetic payload and one typed presentation callback per peer.

All authority mutations enter one command funnel. A press, release, reload request, reload interruption, reload completion poll, and accepted shot are commands into that funnel; there is no second Blueprint mutation path.

The server stores absolute `double` deadlines from world time. `TickComponent` only polls an active fire or reload deadline and is disabled while idle. No timer delegate or latent Blueprint delay owns gameplay state.

### Typed results

Every request returns or records an `EQWeaponFireControlResult` rather than silently doing nothing. The result distinguishes at least success, non-authority, missing adapter, invalid owner, invalid item instance, equipment-not-ready, not-equipped, not-active, Combat blocked, fire blocked, reloading, cadence blocked, empty magazine, invalid transform/tuning, bullet subsystem unavailable, reload already active/not active, no reserve ammo, ammo transaction failure, and rollback failure.

No context failure falls back to actor stats, reflected fields, a synthetic muzzle, or a local trace.

### Typed adapters

The component depends on explicit `TScriptInterface` contracts. An unbound or failed adapter rejects the command.

1. `IQWeaponFireControlContextAdapter`
   - Resolves the exact owner pawn and exact item instance.
   - Proves actor owner equality, equipment readiness, equipped-slot mapping, active-item equality, Combat permission, and fire-block state.
   - Supplies effective hitscan tuning from the exact item/QModule stat path: damage, fire delay, reload duration, magazine capacity, range, infinite-ammo flag, and tracer scale.
   - Resolves the exact `FireLocation` component and complete aim rotation; native code preserves pitch, yaw, and roll while taking location only from that component.
2. `IQWeaponFireControlAmmoAdapter`
   - Loads the exact persisted magazine state and revision.
   - Commits compare-and-swap magazine changes.
   - Begins, commits, and rolls back a reserve-ammo reservation for reload.
   - Does not expose Inventory implementation types to `QWeapon`.
3. `IQWeaponFireControlPresentationAdapter`
   - Receives one fire payload and reload begin/complete/interrupted events.
   - Owns only montage/notifies, recoil, audio, VFX, meshes/sockets, and designer-facing presentation.

These interfaces are the only temporary Blueprint interop. They are named, typed, and signature-checked by generated interface dispatch; no `FindFunction`, string property lookup, or broad reflection bridge is permitted.

### Shot transaction

For each server-polled deadline:

1. Resolve and validate a fresh context snapshot.
2. Validate finite transform/tuning values and the exact muzzle component.
3. Reject if reload is active, cadence is early, or finite ammo is empty.
4. Verify `UQWeaponBulletSubsystem` exists before mutating ammo.
5. Compare-and-swap the persisted magazine from `N` to `N-1` for finite ammo, then update native owner state.
6. Call `ServerFireBullet` exactly once. Its boolean is a hit/miss result, not an execution-success flag; a miss still consumes the round.
7. Advance `NextAllowedFireTime = AcceptedTime + FireDelay`.
8. Multicast one immutable cosmetic payload. Each non-dedicated peer calls `SpawnBulletTracer` once and the presentation adapter once.

If an external magazine commit fails, native state and cadence remain unchanged. There is no damage request and no cosmetic event.

### Reload transaction and rollback

1. Validate the same authority/context invariants and reject a full/infinite magazine.
2. Begin a reserve-ammo reservation for exactly the missing rounds.
3. Store an absolute reload-complete deadline and multicast one reload-begin presentation event.
4. On the polled deadline, revalidate owner/equipped/active/Combat context.
5. Compare-and-swap the persisted magazine to include the reserved rounds.
6. Commit the reserve reservation.
7. Only after both succeed, publish the native magazine and clear reload state.

The reservation result must contain exactly the requested missing-round count. A partial or oversized reservation is rolled back immediately and the reload is rejected before any magazine mutation; reserve adapters cannot silently redefine the transaction quantity.

If reserve commit fails after magazine commit, the component restores the prior magazine through the ammo adapter and rolls the reservation back. Native magazine/reload state is restored from the pre-command snapshot. A failed compensating operation returns `RollbackFailed` and stops automatic fire; it is never hidden.

Explicit interruption rolls back any open reservation, clears the deadline, and emits one interrupted presentation event. Losing owner/equipped/active/Combat validity at completion uses the same interruption path.

### Replication

- Trigger requests are reliable owner-to-server RPCs; the server alone advances gameplay state.
- `CurrentAmmo`, `bIsReloading`, and the latest typed result are private replicated fields with `COND_OwnerOnly`.
- Fire and reload presentation use immutable multicast payloads/events. They do not mutate ammo or damage.
- A monotonically increasing accepted-shot sequence lets tests and presentation reject duplicate payloads without replaying damage/tracers.
- The owner receives the same authoritative cosmetic multicast; there is no separate predictive Blueprint damage or tracer path in the first vertical.
- If later prediction is introduced, it must be a separate reconciled presentation feature and cannot become a second mutation owner.

## Legacy consumer audit and P2 routing

### Can converge on the hitscan executor

- `/Game/VehicleWeapons/MachineGun/VWP_MachineGun` currently client-traces for a target location, RPCs through `VehicleWeaponPawn`, then calls `FireVehicleMachineGun`. It can share the accepted-shot/cadence/cosmetic executor with a vehicle-specific validation/overheat adapter. Its camera selection and vehicle ownership remain vehicle-domain concerns.
- `/Game/Systems/Vehicle/Weapon/VehicleWeaponPawn` currently routes `SendFire1ToServer -> SV_Fire1 -> MC_Fire1 -> Fire1`; `/Game/Systems/Vehicle/Weapon/BP_VehicleWeaponBase` routes weapon IDs through `VehicleWeaponManager.SV_RequestFire/MC_Fire`. These are transport/orchestration candidates, not alternate ballistics owners, and must be replaced only when a concrete vehicle weapon is migrated.
- `/Game/Systems/Vehicle/Weapon/VehicleCombatComponent` already derives from `QWeaponTargetingComponent`. Its migrated targeting provider/registry path remains intact and is not part of fire-control consolidation.
- `/Game/GameplayActors/Turret/TurretBase` machine-gun mode uses `AutomaticLoopFireDelay(0.5)` and `FireStaticDefenseMachineGun`. It can share the QWeapon hitscan executor with a turret authority/target adapter and no player Inventory adapter.
- QAI direct hitscan can share the same executor with its existing AI cadence/burst policy expressed as an AI request adapter. Infinite-ammo semantics remain explicit.

These paths must not be bulk-edited until the reference player firearm passes prompt 09 network/runtime gates.

### Distinct server-authoritative projectile model

- `/Game/Items/Weapons/NashV2/Rocket_Launcher/IS_NashV2_Rocket_Launcher` spawns `RocketLauncherMissileProjectile_C` with homing target, lifetime, radius, and damage.
- `/Game/Items/Weapons/NashV2/Grenade_Launcher/IS_NashV2_Grenade_Launcher` spawns `BP_GrenadeProjectile_C` with speed, ricochet, shrapnel, gravity, ignore list, radius, and damage.
- `/Game/VehicleWeapons/Rocket/VWP_RocketLauncher` and turret missile mode spawn `TurretMissileProjectile_C` with homing/ownership semantics.

These consumers may later reuse the validation, cadence, ammo transaction, transform, sequence, and presentation envelope, but they require a separate typed projectile execution strategy. They must never be routed through hitscan `ServerFireBullet`, and projectile spawn must become server-authoritative before convergence.

`MeleeWeaponBase`, recycler, flares, bombs, and spotlight are not firearm ballistics consumers and require separate contracts.

## Staged cleanup

### Phase 2 source in this lane

Create:

- `Plugins/QWeapon/Source/QWeapon/Public/QWeaponFireControlComponent.h`
- `Plugins/QWeapon/Source/QWeapon/Private/QWeaponFireControlComponent.cpp`
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QWeaponFireControlAutomationTests.cpp`

Existing QWeapon source expected to change: none. `Plugins/QWeapon/Source/QWeapon/QWeapon.Build.cs` is changed only if static source inspection proves a missing dependency; current `Core`, `CoreUObject`, and `Engine` dependencies already cover component/RPC/replication APIs, and the existing module already compiles replication code.

### Prompt 09 asset/integration phase

1. Add/configure `UQWeaponFireControlComponent` only on `IS_NashV2_Assault_Rifle`.
2. Provide exact context, ammo, and presentation adapters from the owning domains.
3. Initialize native magazine state from the exact item instance/revision.
4. Route `Combat_1stTrigger` and reload input to native requests.
5. Bind fire/reload presentation events to the existing child presentation graph and `QWeaponAnimInstance`.
6. Prove the native vertical, then remove the reference asset's superseded `AutomaticLoopFireDelay`, `TriggerFire`, `SV_TriggerFire`, `MC_TriggerFire`, `PreImplementFire`, ammo/reload mutation, and duplicate tracer/damage path. Disabling nodes is not cleanup.
7. Preserve `TraceTest` or aiming presentation only if another live reference consumer still requires it; otherwise remove it from the reference child integration path without changing other WeaponScript children.
8. Correct the shotgun's dead `Damage_0` contract separately after measuring its intended balance value. Do not copy the zero-damage defect into native tuning.

## QATS source coverage

`QWeaponFireControlAutomationTests.cpp` statically exercises the pure transition/decision layer used by the component:

- exact cadence boundary immediately before and at the deadline;
- invalid authority, owner, exact item, equipment-ready, equipped, active, reload, Combat, and fire-block states;
- finite and infinite ammo depletion;
- reload begin, completion, interruption, no-reserve, commit failure, and rollback result;
- partial and oversized reload reservations rejected with one exact rollback;
- exact muzzle location and full pitch/yaw/roll transform fidelity;
- exactly one damage request and one cosmetic/tracer event for one accepted sequence;
- no damage/tracer/cosmetic output for rejection and no duplicate output for a repeated sequence.

The central integration now retains the direct private `QWeapon` dependency in `QAutomatedTestSuite.Build.cs` and its plugin dependency. The current Editor discovers and passes the three `QWeapon.FireControl.*` tests with zero warning/error. This closes the native decision-core gate only; the NashV2 reference asset and its real context, ammo, and presentation adapters are still not integrated.

## Source phase result

The source phase implements the audited contract in:

- `Plugins/QWeapon/Source/QWeapon/Public/QWeaponFireControlComponent.h`;
- `Plugins/QWeapon/Source/QWeapon/Private/QWeaponFireControlComponent.cpp`;
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QWeaponFireControlAutomationTests.cpp`.

The component now contains the single authority command funnel, idle-disabled timestamp polling, owner-only replicated state, exact muzzle-location/full-rotation construction, typed adapter boundaries, one bullet-subsystem dispatch, sequenced cosmetic multicast, and the compare-and-swap/reservation rollback path described above. Unimplemented context/ammo interface defaults return `AdapterUnavailable`; unimplemented presentation defaults do no work. There is no implicit adapter behavior.

The shared target registry now snapshots registration identities before invoking provider callbacks, revalidates stable identity and topology afterward, and defers stale cleanup until no callback can invalidate the live map iterator. A topology mutation discards the sampled result instead of publishing a partial snapshot. The targeting component reports a change only when the canonical ordered target list actually differs. The existing Blueprint-Combat compatibility path in `QWeaponBulletSubsystem` remains intentionally live until `CombatComponent_C` is reparented and stripped; removing it at this source-only checkpoint would break production damage filtering.

`QWeapon.Build.cs` remains unchanged because the implementation uses only existing `Core`, `CoreUObject`, and `Engine` dependencies. `QAutomatedTestSuite.Build.cs` also remains untouched in this lane; it is already independently dirty and prompt 09 must add `QWeapon` without overwriting its existing changes.

Static review checked the current UE 5.7.3 `UActorComponent` owner-role/replication signatures, `DOREPLIFETIME_CONDITION(..., COND_OwnerOnly)`, unreliable multicast syntax, transform/quaternion validation APIs, and the live `UQWeaponBulletSubsystem` signature/return behavior. The central pass subsequently compiled/loaded this source and ran `QWeapon.FireControl` green at `3/3`. No production asset mutation or gameplay/runtime parity is claimed.

## Dedicated simulated-proxy presentation repair

The two-client dedicated log exposed remote weapon animation instances calling the Blueprint aimpoint mutation and consequently attempting the owner-only `SV_WeaponValues` RPC without an owning connection. `QWeaponAnimInstance` now refreshes local-control state before its per-update consumers and invokes `UpdateCurrentAimpointPosition` only for the locally controlled pawn. Simulated proxies still read and interpolate the replicated weapon values, so remote visuals are retained while their invalid mutation/RPC producer is removed. The source compiles through Live Coding; a fresh dedicated build with two clients remains required to prove both zero ownerless aimpoint RPCs and unchanged remote ADS presentation.

## Prompt 09 build, network, and runtime gates

The isolated native source/QATS gate is green. The integration owner must still run all of the following after adapters and the reference asset are integrated:

1. Add the direct QATS `QWeapon` module dependency and compile `QangaEditor` with the editor closed, or use RzMCP Live Coding only if the editor is already running.
2. Run the focused `QWeapon.FireControl` automation tests.
3. Standalone: tap fire, held automatic fire, exact cadence, empty click, reload completion, reload interruption, fire while blocked, weapon swap during reload, and exact muzzle/tracer origin.
4. Listen server plus one client: client-owned fire, server-owned/AI request, non-owner rejection, owner-only ammo/reload replication, one damage request, one tracer/cosmetic event per peer, and no duplicate owner event.
5. Dedicated server plus two clients: remote presentation, miss versus hit ammo consumption, Combat-disabled target, late join/equip, disconnect during reload, and no dedicated-server presentation work.
6. QATS firearm topology: spawn/equip the exact NashV2 AR data asset and script class already used by `QuestTestSubsystem`, exercise one accepted shot and reload, and confirm the native component is the only gameplay mutation owner.
7. Inspect reference Blueprint after stripping: no latent cadence, no fire/reload RPC duplicate, no `ServerFireBullet`/`SpawnBulletTracer` duplicate, and no legacy magazine mutation on the active path.

Only after those gates pass may P2 vehicle/turret/QAI hitscan routing or a typed projectile execution strategy start.
