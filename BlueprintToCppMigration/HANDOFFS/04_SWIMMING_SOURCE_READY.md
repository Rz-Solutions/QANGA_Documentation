STATE: SOURCE_READY

# SwimmingDetection source handoff

## Implemented source

- `Plugins/QSystem/Source/QSystem/Public/Component/QSwimmingDetectionComponent.h`
- `Plugins/QSystem/Source/QSystem/Private/Component/QSwimmingDetectionComponent.cpp`
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QSwimmingDetectionAutomationTests.cpp`
- `Documentation/BlueprintToCppMigration/06_SWIMMING_DETECTION_MIGRATION.md`

No ALS, AI, map, wrapper, EasyCook, config, plugin descriptor, Build.cs, main-plan, or unrelated QATS
file was edited by this lane.

## Native contract represented

`UQSwimmingDetectionComponent` is the single procedural-ocean force owner. It:

- enforces `TG_DuringPhysics`, a `0.05 s` interval, no dedicated-server tick, automatic activation,
  and no component replication, including against stale serialized Blueprint template values;
- binds the pawn controller-change delegate and enables tick only for a local `PlayerController` on
  an authority (standalone/listen host) or autonomous-proxy character role;
- resolves Ninja/ALS dependencies only after that local-player role gate, so AI, remote, unpossessed,
  and dedicated copies do no recurring dependency or water work;
- reads the exact `ALS_Character_BPI.BPI_GetAlsMovementState` contract only to preserve the authored
  `None`/vehicle/sitting exclusion;
- suppresses work inside QSystem's registered underground geometry;
- asks `UWorldscapeSubsystem` for the nearest authoritative root, then reads `bOcean` and native
  signed `GetDistanceFromWater` state directly;
- reads effective gravity from `UNinjaCharacterMovementComponent::GetGravity()`;
- applies the authored velocity damping plus gravity-relative procedural-ocean force through
  `UCharacterMovementComponent::AddForce`;
- emits neutral output for no ocean, surface/air, invalid dependencies, and non-finite input/result;
- throttles structural errors globally to at most one log per second for this source.

It does not cache water state, use `PlayerId`/TagLock, write ALS or CMC movement state, create a second
swimming mode, run for AI/remotes/dedicated servers, replicate state/RPCs, or retain the old
`AddMovementInput` fallback.

## Exact shared dependency integration for prompt 09

Before compiling:

1. Add exactly `"NinjaCharacter"` to `PrivateDependencyModuleNames` in
   `Plugins/QSystem/Source/QSystem/QSystem.Build.cs`.
2. Add exactly `{ "Name": "NinjaCharacter", "Enabled": true }` to the `Plugins` array in
   `Plugins/QSystem/QSystem.uplugin`.
3. Do not change `QAutomatedTestSuite.Build.cs` for this lane. It already directly depends on
   `QSystem`, `NinjaCharacter`, and `WorldScapeCore`; its current unrelated `AIModule` hunk belongs to
   another lane and must be preserved.

No other module or plugin dependency is required.

## Exact asset integration for prompt 09

1. Reparent `/Game/Systems/Character/Blueprints/CharacterLogic/SwimmingDetection` to
   `/Script/QSystem.QSwimmingDetectionComponent` transactionally.
2. Delete all 57 legacy EventGraph nodes and all five variables: `GravityAreaComponent`,
   `WorldVolumeManager`, `BaseCharacter BP`, `PlayerId`, and `CurrentDistanceFromWater`.
3. Keep exactly one inherited native-backed Swimming component on
   `/Game/Systems/Character/Blueprints/CharacterLogic/ALS_Base_CharacterBP` and apply the enforced
   native defaults.
4. Remove the unused Swimming component from
   `/Game/Systems/Character/Blueprints/CharacterLogic/AI_BaseCharacter`; do not add an AI path.
5. Inspect these eight map referencers individually for deliberate component overrides before
   refreshing them. Do not bulk-load them in one Editor session:
   - `/Game/Maps/Lobby/L_Lobby`
   - `/Game/Maps/Lobby/Lobby_Cyborg_Halloween/Lobby_Cyborg_Halloween`
   - `/Game/Maps/Lobby/Lobby_Cyborg_Christmas/Lobby_Cyborg_Christmas`
   - `/Game/Maps/Lobby/Lobby_Cyborg_V1/Lobby_Cyborg_V2`
   - `/Game/Maps/ScreenshotShowdown/L_Showdown`
   - `/Game/Maps/Lobby/_OLD/L_Sub_MainMenu`
   - `/Game/Maps/Universe/_LevelTest/L_Persistent_DEV_2`
   - `/Game/Maps/Universe/_LevelTest/L_Persistent_DEV_3`
6. Refresh/resave only owners or placed instances needed to clear stale legacy serialization.
7. Remove the obsolete `/Game/EasyCook/DA_EasyCookSeed_QANGA` reference only after the wrapper and
   owner dependency graph validates.
8. Leave ALS state graphs, Ninja movement, WorldScape roots, underground-volume content, config, and
   unrelated consumers unchanged.
9. Update the main migration plan only after source, compile, asset, and runtime gates pass.

The wrapper may remain as an empty native-backed compatibility asset. Delete it only if prompt 09
proves every serialized consumer can point directly at the native class.

## Focused automation source

After dependency integration and a successful compile, run exactly:

- `QATS.QSystem.SwimmingDetection.Decisions`
- `QATS.QSystem.SwimmingDetection.Lifecycle`

The decision test covers the surface boundary, half/full/beyond-full depth, ordinary/zero/negative
gravity, non-finite inputs, computed-force overflow, output neutralization, and the exact ALS state
gate. The lifecycle test covers stale template restoration, no replication/dedicated tick, retained
20 Hz during-physics cadence, and explicit neutral failure for a non-character owner.

No automation test was run by this lane.

## Prompt 09 compile, asset, and runtime gates

1. Compile `QSystem` and `QAutomatedTestSuite` after the two dependency edits above.
2. Run both focused QATS tests.
3. Reparent, strip, compile, and validate the wrapper plus both direct character owners.
4. Confirm the wrapper has zero legacy variables and zero EventGraph nodes, and that only ALS keeps
   the component.
5. Standalone/PIE: validate air/surface, shallow depth, full depth, water exit, no-ocean root,
   underground suppression, ordinary gravity, zero gravity, and negative gravity.
6. Possession: validate unpossessed, local player, AI, repossession, and vehicle/sitting ALS gates.
7. Listen server plus client: prove only the locally controlled copy applies the force, remote proxies
   remain idle, ALS movement state has no Swimming-owned write/RPC, and movement prediction does not
   introduce unacceptable correction.
8. Dedicated server: prove zero Swimming ticks and zero WorldScape water queries.
9. Conventional `bWaterVolume` content must continue through Ninja's native `MOVE_Swimming`,
   `ImmersionDepth`, and `PhysSwimming` path without intervention from this component.
10. Trace the retained local 20 Hz query. Change its cadence only with measured physics/perceptual
    parity; do not silently move it to a per-frame loop.

## Validation performed by this lane

- The Blueprint graph, class defaults, owner templates, referencers, and API contract were inspected
  before the Editor was closed; no asset was mutated or saved. The bulk map-instance override scan was
  not completed and remains the explicit per-map prompt 09 gate above.
- Current UE, NinjaCharacter, WorldScape, QSystem underground, ALS reflection, and component lifecycle
  signatures/access were checked against their headers and implementations.
- Focused static source review and owned-file whitespace checks completed cleanly.
- No compile, UHT, QATS, Editor command after closure, PIE, asset mutation/save, cook, package,
  EasyCook rescan, stage, or commit was performed.
