STATE: SOURCE_READY

# Weapon orchestration source handoff

## Implemented source

- `Plugins/QWeapon/Source/QWeapon/Public/QWeaponFireControlComponent.h`
- `Plugins/QWeapon/Source/QWeapon/Private/QWeaponFireControlComponent.cpp`
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QWeaponFireControlAutomationTests.cpp`
- `Documentation/BlueprintToCppMigration/07_WEAPON_ORCHESTRATION_MIGRATION.md`

No existing QWeapon source file was edited. No asset, config, QInventory, QCombat, QAI, QModule, main migration plan, or QATS Build.cs file was edited.

## Native behavior now represented

`UQWeaponFireControlComponent` is the single authority mutation owner for:

- reliable trigger/reload/interruption requests;
- idle-disabled polling of absolute cadence and reload deadlines;
- exact weapon actor owner, pawn, item, equipment-ready, equipped, active-item, Combat and fire-block validation;
- exact `FireLocation` component location plus complete adapter-supplied pitch/yaw/roll;
- finite/infinite magazine transitions;
- compare-and-swap magazine persistence and reserve-ammo begin/commit/rollback;
- explicit fault state when compensation itself fails, with reload rollback retried by the interruption command;
- exactly one `UQWeaponBulletSubsystem::ServerFireBullet` call for an accepted hitscan sequence;
- one sequenced cosmetic multicast, one tracer dispatch and one presentation-adapter callback per rendering peer;
- private `COND_OwnerOnly` replication of initialization, ammo, reload, typed result and failure detail.

The three interop boundaries are generated typed interfaces:

- `IQWeaponFireControlContextAdapter`;
- `IQWeaponFireControlAmmoAdapter`;
- `IQWeaponFireControlPresentationAdapter`.

Their native defaults fail with `AdapterUnavailable` or do no presentation work. There is no reflection bridge, actor-stat fallback, local damage trace, timer delegate, or duplicate ballistics implementation.

## Exact build dependencies

### QWeapon

No Build.cs change is required. The new source uses existing direct public dependencies:

- `Core`;
- `CoreUObject`;
- `Engine`.

`QWeapon.Build.cs` remains clean. Do not add `QModule`, QInventory, QCombat, or QAI to QWeapon.

### QATS

Before compiling prompt 09 must add exactly `"QWeapon"` to `PrivateDependencyModuleNames` in:

`Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/QAutomatedTestSuite.Build.cs`

That file already has an unrelated dirty addition of `"AIModule"`. Preserve it and every other current hunk. This lane intentionally did not edit the file.

## Prompt 09 cross-domain implementations

1. QModule/item stat owner:
   - Provide a typed exact-item resolver for damage, fire delay, reload duration, magazine capacity, range, infinite ammo and tracer scale.
   - Resolve through `QModuleItemRack::GetStat(UObject* ItemInstance, FGameplayTag StatTag, float BaseValue)`.
   - Do not call actor-only `QMOD_GetStat`.
   - Implement the adapter from the QModule/integration side; never add a QWeapon-to-QModule dependency.
2. Inventory owner:
   - Prove `IsEquipmentReady`, exact item-instance-to-slot mapping and `GetActiveItem() == weapon`.
   - Implement revisioned `ReadWeaponMagazine`, `CommitWeaponMagazine`, `RollbackWeaponMagazine`, and reserve-ammo begin/commit/rollback atomically.
   - A failed adapter result must guarantee it did not partially mutate unless its matching rollback can compensate it.
3. Combat owner:
   - Supply the existing typed Combat fire permission for the exact owner pawn.
   - Do not move Combat state or damage dispatch into the component.
4. Fire-block/aim integration:
   - Resolve fire block authoritatively from the existing weapon bounds/wall semantics; do not trust a client-only bool.
   - Supply the current full aiming rotation on the authority.
   - Native code takes location only from the reference weapon's exact `FireLocation` component.
5. Presentation owner:
   - Bind `HandleWeaponFireCosmetic` to the existing AR recoil, `CallFired`, audio, muzzle VFX and weapon timeline nodes after their ammo/ballistics branches are removed.
   - Bind reload Started/Completed/Interrupted to `QWeaponAnimInstance` and existing reload audio.
   - Presentation must not call `ServerFireBullet`, `SpawnBulletTracer`, or mutate ammo.

## Reference asset integration and stripping

Exact reference:

- data asset: `/Game/Items/Weapons/NashV2/Assault_Rifle/IDA_NashV2_Assault_Rifle.IDA_NashV2_Assault_Rifle`;
- script class: `/Game/Items/Weapons/NashV2/Assault_Rifle/IS_NashV2_Assault_Rifle.IS_NashV2_Assault_Rifle_C`;
- current parent: `/Game/Systems/Weapon/WeaponScript.WeaponScript_C`.

Prompt 09 sequence:

1. Keep the current parent for the first proof. Reparenting now would discard shared attachment, aiming and presentation behavior that this vertical intentionally does not absorb.
2. Add one `QWeaponFireControlComponent` to the reference script and configure all three typed adapters before server initialization.
3. Initialize from the exact persisted magazine item instance/revision after owner/equipment/active-item state is ready.
4. Replace the reference `Combat_1stTrigger` body with `NativeSetTriggerHeld(Pressed)`.
5. Replace reference reload input with `NativeRequestReload`; call `NativeInterruptReload` on weapon swap, unequip, owner loss and real reload-montage interruption.
6. Move the existing aiming result into `ResolveWeaponFireRotation`; move the authoritative wall/bounds result into the context snapshot.
7. Rewire only presentation nodes to the presentation adapter callbacks.
8. Remove the reference active path through `AutomaticLoopFireDelay`, `CheckWeaponInsideWall -> CalcAimingAndTriggerFire -> TriggerFire`, `SV_TriggerFire`, `MC_TriggerFire`, `PreImplementFire`, `ConsumeMagazineAmmo`, delayed reload mutation, inventory consumption, and duplicate tracer/ballistics calls. Do not leave disconnected or disabled duplicates.
9. Retain inherited WeaponScript code only because other children still consume it. The reference asset must no longer schedule or call it. Reparent away from `WeaponScript_C` only after the shared presentation/attachment dependencies have their own proven owners or after the remaining children migrate; do not fork those systems merely to reparent this first vertical.
10. Preserve phase/designer tuning of active `Damage` and `FireDelay`, FireLocation, meshes/sockets, recoil/audio/VFX and animation assets.

The shotgun is separate cleanup: `IS_Shotgun` currently sends inherited `Damage=0` while phases write dead `Damage_0`. Do not copy or silently reinterpret that tuning during AR integration.

## Focused automation source

After adding the direct QATS dependency, run exactly:

- `QWeapon.FireControl.CadenceAndValidation`
- `QWeapon.FireControl.AmmoAndReload`
- `QWeapon.FireControl.TransformAndDispatchCardinality`

The source covers the pre-deadline/exact-deadline boundary, authority/owner/item/equipment/active/Combat/reload/fire-block rejection, finite depletion, infinite ammo, reload refill/interruption/rollback, full transform fidelity, one damage request, one cosmetic event and duplicate sequence rejection.

No automation test was run in this lane.

## Prompt 09 build and runtime gates

The editor is currently closed. Build first with:

`"E:\UE573\Engine\Build\BatchFiles\Build.bat" QangaEditor Win64 Development -Project="G:\QANGA\QANGA.uproject" -WaitMutex`

Then launch only when runtime integration is ready:

`"E:\UE573\Engine\Binaries\Win64\UnrealEditor.exe" "G:\QANGA\QANGA.uproject" -AutoDeclinePackageRecovery`

Required topologies:

1. Standalone:
   - exact NashV2 AR equip/init;
   - tap and held fire;
   - exact 0.15 s authored cadence;
   - miss and hit both consume one finite round;
   - empty-magazine rejection;
   - reload begin/deadline/complete and explicit interruption;
   - wall block, Combat block, swap and unequip;
   - exact FireLocation origin and full aim rotation;
   - one recoil/audio/VFX callback and one tracer per accepted sequence.
2. Listen server plus one client:
   - owner client request;
   - non-owner RPC rejection by ownership/context;
   - server damage exactly once;
   - owner-only ammo/reload/result replication;
   - one cosmetic callback on server renderer and client, with no owner duplicate;
   - swap/owner loss during reload rolls reservation back.
3. Dedicated server plus two clients:
   - no server presentation work;
   - one authoritative damage request;
   - one remote cosmetic/tracer per rendering peer;
   - Combat-disabled and invalid-owner requests reject without ammo mutation;
   - disconnect during reload leaves no committed reserve or magazine divergence.
4. QATS firearm topology:
   - use the existing `TestFirearmDataAssetPath` and `TestFirearmScriptClassPath` for the NashV2 AR in `QuestTestSubsystem`;
   - prove the native component is the only active ammo/cadence/damage owner;
   - inspect the reference graph after stripping for zero legacy fire/reload RPC or bullet/tracer calls.

Do not claim the source vertical complete until all four gates pass.

## P2 consumer routing

- `/Game/VehicleWeapons/MachineGun/VWP_MachineGun`: converge on the same hitscan execution contract with a vehicle ownership/overheat adapter; remove its client line-trace scheduling only during that migration.
- `/Game/GameplayActors/Turret/TurretBase` machine-gun mode: converge on the same hitscan executor with turret authority/target context and no Inventory adapter.
- QAI direct hitscan: converge on the accepted-shot executor while retaining QAI burst/cadence policy and explicit infinite-ammo context.
- `VehicleWeaponPawn.SV_Fire1/MC_Fire1` and `VehicleWeaponManager.SV_RequestFire/MC_Fire`: remove per concrete migrated weapon, not globally ahead of proof.
- NashV2 rocket/grenade launchers, vehicle rockets and turret missiles: share only validation/cadence/ammo/transform/presentation envelope later. Add a distinct server-authoritative projectile execution strategy; never route them through `ServerFireBullet`.
- Melee, recycler, bombs, flares and spotlight remain outside firearm ballistics consolidation.

No downstream consumer was edited in this lane.

## Validation actually performed

- Read-only live Blueprint/asset audit was completed before the editor closed.
- Current UE 5.3.2 header/API signatures and current QWeapon bullet implementation were statically checked.
- Focused `git diff --check -- <owned paths>` returned clean.
- Because every lane file is new/untracked, per-file `git diff --no-index --check -- NUL <file>` checks also reported no whitespace error; the only output was the repository's normal LF-to-CRLF autocrlf warning.
- No compile, UHT, test, Editor/PIE, asset save/mutation, cook, package, EasyCook rescan, stage or commit was performed.
