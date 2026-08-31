STATE: AUDIT_READY

# SwimmingDetection audit handoff

## Decision

Unique behavior remains: WorldScape procedural oceans have no water `APhysicsVolume`, so UE/Ninja
native swimming cannot produce their immersion or force. Migrate only the procedural-ocean force.

Do not migrate the redundant/competing state:

- no `CurrentDistanceFromWater` cache;
- no `PlayerId` or delayed retry;
- no GravityArea/WorldVolume manager cache;
- no ALS/CMC movement-state write;
- no replicated component state;
- no AI, remote-proxy, or dedicated-server work;
- no `AddMovementInput` fallback.

Authoritative inputs are WorldScape root distance/ocean state, QSystem's native underground-volume
registry, the existing ALS getter used only as a gate, and
`UNinjaCharacterMovementComponent::GetGravity()`.

## Exact source ownership

Module: `QSystem`

- `Plugins/QSystem/Source/QSystem/Public/Component/QSwimmingDetectionComponent.h`
- `Plugins/QSystem/Source/QSystem/Private/Component/QSwimmingDetectionComponent.cpp`
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QSwimmingDetectionAutomationTests.cpp`
- `Documentation/BlueprintToCppMigration/06_SWIMMING_DETECTION_MIGRATION.md`
- `HANDOFFS/04_SWIMMING_SOURCE_READY.md`

The lane does not own or edit QSystem/QATS plugin descriptors, Build.cs integration, ALS, AI, maps,
wrapper assets, EasyCook, config, or the main plan.

## Shared dependency changes deferred to prompt 09

1. Add `NinjaCharacter` to `PrivateDependencyModuleNames` in
   `Plugins/QSystem/Source/QSystem/QSystem.Build.cs`.
2. Add `{ "Name": "NinjaCharacter", "Enabled": true }` to the `Plugins` array in
   `Plugins/QSystem/QSystem.uplugin`.
3. Do not change QATS Build.cs; it already contains `QSystem`, `NinjaCharacter`, and `WorldScapeCore`.

## Exact asset integration deferred to prompt 09

1. Reparent `/Game/Systems/Character/Blueprints/CharacterLogic/SwimmingDetection` to
   `/Script/QSystem.QSwimmingDetectionComponent`.
2. Delete the 57-node EventGraph and the five legacy variables.
3. Keep one native-backed component on `ALS_Base_CharacterBP`; remove the unused component from
   `AI_BaseCharacter`.
4. Inspect the eight listed direct map referencers for deliberate instance overrides before refreshing
   them; a previous bulk read-only load was aborted after exhausting Editor memory.
5. Clear stale map/child serialization and the EasyCook seed reference only after wrapper/owner
   validation.
6. Run the compile, QATS, asset, PIE, multiplayer, dedicated, and trace gates listed in
   `06_SWIMMING_DETECTION_MIGRATION.md`.

Read-only RzMCP Blueprint inspection was performed before the Editor was closed. No compile, test,
PIE, asset mutation/save, cook, package, stage, commit, or EasyCook rescan was performed by this lane.
