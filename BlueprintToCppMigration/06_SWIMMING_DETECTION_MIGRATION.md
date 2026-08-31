# SwimmingDetection Blueprint-to-C++ migration

## Status

- Audit/contract complete on 2026-08-30.
- Native component, dependency integration, Blueprint/map conversion, and dead-wrapper cleanup complete
  on 2026-08-31.
- Cold `QangaEditor Win64 Development` build and `QATS.QSystem.SwimmingDetection.*` pass `2/2`.
- The eight serialized level instances reconstruct to the exact native component with no live or
  serializable legacy/trash component. One pre-existing garbage-only `TRASH_` object observed while
  loading `L_Persistent_DEV_2` is not serialized and disappears with the editor world.
- The sole EasyCook seed entry was removed without rebuilding the seed, the old class redirect was
  inserted byte-for-byte into the UTF-16LE config and proved idempotent, and the empty wrapper asset
  was deleted after its direct referencer count reached zero.
- `L_Dev_Rz` PIE proves the local player owns one registered/active native component ticking at
  `20 Hz` in `TG_DuringPhysics`, with replication and dedicated-server ticking disabled. PIE ended
  with zero Message Log warnings/errors.
- Listen/dedicated multi-process prediction and lifecycle validation remains open; no packaged parity
  or performance gain is claimed yet.

## Baseline

Asset: `/Game/Systems/Character/Blueprints/CharacterLogic/SwimmingDetection`

- Parent: `ActorComponent`.
- Interfaces: none.
- Graphs: one 57-node `EventGraph`; 65 nodes total.
- Entry paths: `BeginPlay`, `Tick`, and the pawn controller-change delegate.
- Variables:
  - `GravityAreaComponent : GravityAreaComponent_C`, editable, default `None`.
  - `WorldVolumeManager : WorldVolumeManager_C`, editable, default `None`.
  - `BaseCharacter BP : Pawn`, editable, default `None`.
  - `PlayerId : Name`, editable, default `Global`.
  - `CurrentDistanceFromWater : Real`, editable, default `0`.
- Component defaults/templates:
  - `TG_DuringPhysics`.
  - `bCanEverTick=true`.
  - `bAllowTickOnDedicatedServer=true`.
  - `TickInterval=0.05 s` (20 Hz).
  - `bAutoActivate=false`.
  - `bReplicates=true`, although the component declares no replicated property or RPC.
- `ALS_Base_CharacterBP` and `AI_BaseCharacter` both own a component template named
  `SwimmingDetection`. Their inspected Swimming template values are identical to the class defaults.

The five editable variables are runtime caches/context/output, not authored tuning. No native source
consumer and no node outside the Swimming Blueprint was found for `CurrentDistanceFromWater` in the
explicit owner/referencer search. The asset-registry direct dependency set contains no child Blueprint
that owns another Swimming component class reference; descendants receive it through their character
parent. A bulk read-only map-instance load was abandoned after it exhausted Editor memory, so prompt 09
must still inspect the listed placed instances before reparenting. This cannot change the native source
contract: every legacy editable field is removed, and the native lifecycle enforces its tick/network
invariants instead of accepting serialized overrides.

## Related history reviewed

- `f0ddc6111` (2026-01-15) is the initial `Content/Systems` import containing the asset.
- `6f50cf53c` and `bcb688c00` (2026-03-01) are the only later Swimming asset-history entries; both are
  broad binary refresh commits and declare no new Swimming contract.
- `3bd272b02` (2026-08-26) establishes the current native GravityScape/Ninja gravity ownership but
  adds no procedural-ocean force owner.
- `d2a0fb24a` (2026-08-28) adds the existing WorldScape dependency/use to QSystem but does not absorb
  Swimming behavior.

There is therefore no newer native implementation to rewire to or delete. QSystem is the smallest
existing owner because it already consumes WorldScape and owns the underground-volume registry.

## Exact Blueprint execution contract

### BeginPlay and possession

1. Cast the owner to `Character`, store it in the pawn-typed `BaseCharacter BP` cache, and obtain its
   `GravityAreaComponent` through `Lib_GravityArea`.
2. Bind the pawn's `ReceiveControllerChangedDelegate` to `OnController`.
3. On a controller change, disable component tick unless the new controller is both a
   `PlayerController` and local.
4. For the local player, enable tick and poll `Lib_GameInstance.GetLocalPlayerId`; when unavailable,
   retry after `0.2 s`. The resulting `PlayerId` is used only as the context key for
   `TagLock.TagHasLock(PlayerId, Underground)`.

Consequences:

- AI possession never enables the component, including standalone AI where the generic Engine notion
  of a local controller would otherwise be true.
- Remote proxies and dedicated-server copies do not perform the water query or force write after the
  controller gate runs.
- A listen-server's locally controlled player and an autonomous client do run it.
- `PlayerId` is not gameplay ownership. It is an indirection used to read a local presentation tag.

### 20 Hz gameplay path

The tick accepts ALS states `Grounded`, `In Air`, `Mantling`, `Ragdoll`, and `Swimming`; it rejects
`None`, `In Vehicle`, and `Sitting`.

For an eligible state it:

1. validates/caches `WorldVolumeManager`;
2. returns while the local `Underground` tag lock is held;
3. calls `WorldVolumeManager.GetDistanceFromGround` with:
   - the character location;
   - `GravityAreaComponent.CachedTagGravityArea` as the planet key;
   - `WaterNoiseHeight=true`;
4. writes `CurrentDistanceFromWater=0` when the key is absent, the mapped WorldScape root is invalid,
   or `AWorldScapeRoot::bOcean` is false;
5. otherwise writes WorldScape's signed `GetDistanceFromWater` result;
6. when the distance is strictly below `0 cm`, calls `BPI_Set_MovementState(In Air)` and applies:

   `Force = Velocity * (-30) * Clamp((-DistanceCm) / 200, 0, 1)`

   `        + ActorUp * ProjectGravityValue * (-3)`

   through `UCharacterMovementComponent::AddForce`.

The cast-failure branch calls `Pawn.AddMovementInput(ActorUp, 10)`. It is a fallback for a path that is
not eligible in the proved production contract and is deleted rather than migrated.

## Authoritative owners

### Water surface and ocean identity

`AWorldScapeRoot` is authoritative for both `bOcean` and signed water distance.
`GetDistanceFromWater` evaluates the water-noise surface and returns the signed point-to-surface
distance. `UWorldscapeSubsystem` owns the live root registry and already exposes
`GetNearestWorldScapeFromLocation` to QSystem consumers.

The Blueprint `WorldVolumeManager` does not calculate water. It maps the gravity-area name to a
WorldScape root and forwards the native call. Its key coupling is not retained. The native owner asks
the WorldScape subsystem for the geometrically authoritative nearest registered root, then asks that
root for water distance.

Atmosphere and gravity-area transitions therefore no longer manufacture a `0 cm` water result merely
because a presentation/registry key changed. Atmosphere is not water ownership. The closest actual
WorldScape ocean determines distance.

### UE/Ninja swimming

UE and `UNinjaCharacterMovementComponent` already own conventional swimming, immersion, fluid
friction, buoyancy, and `MOVE_Swimming` while the character is inside an `APhysicsVolume` whose
`bWaterVolume` is true.

WorldScape does not create an `APhysicsVolume`, does not set `bWaterVolume`, and creates ocean render
meshes with collision disabled. Consequently Ninja's `ImmersionDepth`, `PhysicsVolumeChanged`, and
`PhysSwimming` cannot represent a WorldScape procedural ocean. The procedural-ocean force is unique
behavior and cannot be deleted as a mirror of Ninja state.

The native Swimming component does not set `MOVE_Swimming`, override `ImmersionDepth`, or implement a
second swimming physics mode. It only applies the proved procedural-ocean force.

### Gravity and buoyancy direction

`UNinjaCharacterMovementComponent::GetGravity()` is the authoritative effective gravity vector. It
already composes the active direction, physics-volume scale, character scale, forced gravity, and
positive/zero/negative scale semantics.

The migrated force is:

`DampingAlpha = Clamp((-DistanceCm) / 200 cm, 0, 1)`

`Force = -VelocityCmPerSecond * 30 * DampingAlpha - GravityCmPerSecondSquared * 3`

This is equal to the authored formula for ordinary downward gravity, while fixing its invalid reliance
on a project gravity scalar plus actor up:

- positive gravity: buoyancy opposes gravity;
- zero gravity: buoyancy is neutral and only velocity damping remains;
- negative gravity: buoyancy reverses with the authoritative gravity vector;
- non-finite distance, velocity, gravity, or result: no force is emitted.

### ALS movement state

ALS owns the replicated `ALSMovementState`. Its existing character-movement callback maps walking to
`Grounded` and falling to `In Air`, while protecting higher-level states such as vehicle and sitting.
`BPI_Set_MovementState` also invokes the replicated-state path and, from an owning client, a reliable
server RPC.

The Swimming Blueprint always writes `In Air`, not `Swimming`. In the procedural ocean the CMC remains
falling, so the normal ALS/CMC owner already produces `In Air`. Repeating that write from water
detection creates a competing movement-state owner and can cancel protected states such as ragdoll.
The native component reads the current ALS state only to preserve the `None`/vehicle/sitting gate; it
never writes ALS or CMC movement state.

### Underground suppression and player identity

QSystem already owns a native registry of active underground shapes through
`UQInteriorPostProcessComponent::IsLocationInsideUndergroundVolume`. The migration queries that
authoritative geometric volume state at the character position. It removes the `PlayerId`, TagLock
lookup, and delayed local-ID retry loop entirely.

## Selected native owner and source files

Existing module: `QSystem`. A new plugin is not justified.

Owned files:

- `Plugins/QSystem/Source/QSystem/Public/Component/QSwimmingDetectionComponent.h`
- `Plugins/QSystem/Source/QSystem/Private/Component/QSwimmingDetectionComponent.cpp`
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QSwimmingDetectionAutomationTests.cpp`

Shared integration changes completed by the main integration lane:

- `NinjaCharacter` is present in `QSystem.Build.cs` private dependencies;
- the `NinjaCharacter` plugin dependency is present in `QSystem.uplugin`;
- do not change `QAutomatedTestSuite.Build.cs`: it already depends on `QSystem`, `NinjaCharacter`, and
  `WorldScapeCore`.

## Native runtime contract

- Owner must be an `ACharacter` with `UNinjaCharacterMovementComponent` and the exact
  `ALS_Character_BPI` getter signature.
- The component is non-replicated and emits no replicated state or RPC.
- It is active automatically but ticks only for a local `PlayerController` on an authority
  (standalone/listen host) or autonomous-proxy character role.
- AI, remote proxies, unpossessed pawns, and dedicated servers never tick it.
- Ninja/ALS dependencies are resolved only after the local-player role gate; ineligible copies do no
  water query and no recurring dependency work.
- Tick group remains `TG_DuringPhysics` and interval remains `0.05 s`.
- The 20 Hz cadence is retained because the authored force is applied through `AddForce` only on that
  cadence and WorldScape water noise is dynamic. Moving the force into a per-frame or post-movement
  callback would change effective acceleration/timing without runtime measurement. Prompt 09 may
  change it only after a trace and a perceptual/physics parity gate.
- Every tick re-derives WorldScape root, ocean state, signed distance, underground state, ALS gate,
  velocity, and Ninja gravity from their owners; it stores no duplicate water or movement state.
- Every failure path emits no force. Structural failures are explicit and throttled to at most one log
  per second for the source; ordinary absence of an ocean is not an error.

## Direct consumers and deletion plan

The exact direct referencers observed for `SwimmingDetection` are:

- `/Game/Systems/Character/Blueprints/CharacterLogic/ALS_Base_CharacterBP`
- `/Game/Systems/Character/Blueprints/CharacterLogic/AI_BaseCharacter`
- `/Game/Maps/Lobby/L_Lobby`
- `/Game/Maps/Lobby/Lobby_Cyborg_Halloween/Lobby_Cyborg_Halloween`
- `/Game/Maps/Lobby/Lobby_Cyborg_Christmas/Lobby_Cyborg_Christmas`
- `/Game/Maps/Lobby/Lobby_Cyborg_V1/Lobby_Cyborg_V2`
- `/Game/Maps/ScreenshotShowdown/L_Showdown`
- `/Game/Maps/Lobby/_OLD/L_Sub_MainMenu`
- `/Game/Maps/Universe/_LevelTest/L_Persistent_DEV_2`
- `/Game/Maps/Universe/_LevelTest/L_Persistent_DEV_3`
- `/Game/EasyCook/DA_EasyCookSeed_QANGA`

Completed integration:

1. Added the two QSystem dependency declarations above.
2. Cleared the wrapper graph and all five legacy variables, then transactionally reclassed the
   `ALS_Base_CharacterBP` component to `/Script/QSystem.QSwimmingDetectionComponent` while preserving
   its variable identity and hierarchy.
3. Removed the duplicate own-SCS component from `AI_BaseCharacter`. AI descendants that inherit the
   ALS composition still instantiate the native component, but its native controller/role gate keeps
   them inactive; they no longer own a competing Blueprint implementation.
4. Audited every listed placed instance: none contained authored business overrides. Reconstructed
   and saved only the eight affected maps, verifying the exact native class and no live/serializable
   legacy or trash object in each map.
5. Removed the one obsolete EasyCook seed entry after the wrapper referencer graph reached only that
   seed, added the exact class redirect, proved the second insertion is a no-op, then deleted the
   zero-referencer wrapper.
6. Left ALS state graphs, Ninja movement, WorldScape roots, underground volumes, and their unrelated
   consumers unchanged.

What remains Blueprint after integration:

- ALS movement-state/interface implementation and animation reactions;
- character/map composition that owns the component;
- WorldScape and underground-volume content authoring.

## Focused tests

Source tests declared in `QSwimmingDetectionAutomationTests.cpp`:

- exact surface/depth clamp and damping force;
- ordinary, zero, and negative gravity direction;
- non-finite input/result neutralization;
- exact ALS eligibility set;
- registration restores 20 Hz, during-physics, local-only, non-replicated lifecycle defaults;
- an invalid owner remains neutral and unticked rather than using a movement-input fallback.

Completed gates:

- cold-compiled `QSystem`, `QAutomatedTestSuite`, the editor target, and all affected Blueprint owners;
- passed `QATS.QSystem.SwimmingDetection.*` `2/2`;
- proved all five legacy variables and all legacy graph logic are gone before deleting the wrapper;
- proved the live `L_Dev_Rz` local-player lifecycle/default contract and a clean PIE Message Log.

Remaining runtime gates:

- PIE surface, shallow/full depth, exit, underground suppression, no-ocean planet, zero gravity, and
  negative gravity behavior;
- possession across unpossessed, AI, remote client proxy, listen host, and repossession;
- autonomous-client prediction/server correction with no per-tick RPC or replicated component;
- dedicated server: zero Swimming component ticks and WorldScape water queries;
- trace the retained local 20 Hz query and change cadence only with measured semantics/performance;
- confirm conventional physics-volume water still uses Ninja's native `MOVE_Swimming` path.
