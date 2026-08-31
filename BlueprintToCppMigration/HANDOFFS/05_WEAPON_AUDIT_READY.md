STATE: AUDIT_READY

# Weapon orchestration audit handoff

## Reference vertical

- Base asset: `/Game/Systems/Weapon/WeaponScript.WeaponScript` (`WeaponScript_C`).
- Reference weapon: `/Game/Items/Weapons/NashV2/Assault_Rifle/IS_NashV2_Assault_Rifle`.
- QATS already names this exact script class in `QuestTestSubsystem`.
- Shotgun rejected as reference: live inherited `Damage=0`; phases write unused `Damage_0`; base `PreImplementFire` sends `Damage`.

## Proposed source files

- `Plugins/QWeapon/Source/QWeapon/Public/QWeaponFireControlComponent.h`
- `Plugins/QWeapon/Source/QWeapon/Private/QWeaponFireControlComponent.cpp`
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QWeaponFireControlAutomationTests.cpp`

## Existing QWeapon files expected to change

- None.
- `Plugins/QWeapon/Source/QWeapon/QWeapon.Build.cs` remains unchanged unless implementation proves a missing direct dependency. No `QModule`, QInventory, QCombat, or QAI dependency is permitted.

## Native contract

`UQWeaponFireControlComponent` owns server-polled timestamp cadence, one authority mutation funnel, exact owner/item/equipped/active/Combat/fire-block validation, transactional magazine/reload state, exact `FireLocation` transform, one `UQWeaponBulletSubsystem::ServerFireBullet` call, and one immutable cosmetic multicast/presentation event per accepted shot.

Private ammo/reload/result state is owner-only replicated. Every failure is typed. Missing adapters fail explicitly. There is no actor-stat, reflection, trace, transform, or ballistics fallback.

## Cross-domain integration requests for prompt 09

- QModule/item domain: implement the context adapter using the exact item instance and `QModuleItemRack::GetStat`; do not use actor-only `QMOD_GetStat`. Avoid the `QWeapon -> QModule -> QAI -> QWeapon` module cycle.
- Inventory domain: implement compare-and-swap magazine persistence plus begin/commit/rollback reserve-ammo transactions and exact active/equipped/equipment-ready checks.
- Combat domain: expose typed fire permission for the exact owner pawn; do not move Combat into QWeapon.
- Presentation/animation: bind the single native fire/reload presentation events to existing recoil/audio/VFX and `QWeaponAnimInstance`; keep montages/notifies/magazine meshes there.
- QATS build owner: add direct private dependency `QWeapon` to `QAutomatedTestSuite.Build.cs` before compiling the new test source.
- Asset integration owner: add/configure the native component on the NashV2 AR, route trigger/reload input, then strip superseded cadence/RPC/ballistics/ammo nodes after runtime proof.
- Shotgun owner: repair the live `Damage` versus dead `Damage_0` tuning defect as a separate measured cleanup.
- P2 QAI/vehicle/turret owner: converge hitscan consumers only after the reference gate; give projectile consumers a distinct server-authoritative projectile executor.

## Audit document

Full evidence, contract, cleanup stages, QATS coverage, and prompt 09 gates are in `Documentation/BlueprintToCppMigration/07_WEAPON_ORCHESTRATION_MIGRATION.md`.

Phase 2 source implementation follows immediately. No compile, test, editor, PIE, asset mutation, cook, package, or EasyCook rescan is authorized in this lane.
