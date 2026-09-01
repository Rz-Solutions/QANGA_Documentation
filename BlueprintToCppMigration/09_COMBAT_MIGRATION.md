# Combat native migration contract

## Status, scope and ownership

This document records the audited Combat baseline, native contract and current production-integration checkpoint. The native vertical owns life, damage acceptance, the alive/dead transition, faction and combat permission, and immutable worker-safe publication.

- Native owner: a new runtime plugin, `QCombat`.
- Runtime state owner: `UQCombatComponent`.
- Pure policy owner: `QCombatPolicy`.
- Worker value-model owner: `QCombatSnapshot`.
- Existing target roster, stable-ID, spatial registry and target-selection owner: QWeapon. Combat creates no second registry or actor resolver.
- Existing safe-area spatial truth owner: `UQGameManager_World_SubSystem`.
- Wanted-system owner: QPolice. QPolice publishes typed facts into Combat; Combat does not own wanted accumulation or decay.
- Drops, rewards, spawners, quest notifications, death FX, ragdolls, audio, obituary text and presentation remain downstream consumers.
- Vehicle, turret and remaining legacy damage paths are P2 consumers. They are not modified in this lane.

The production wrapper is now reparented to `UQCombatComponent`. Native life/faction/mode/target/provenance state and mutation are the live owner; the wrapper retains only presentation, reward, drop and downstream notification seams. The affected Blueprint consumers compile against the native component type, including the QAI death map. The wrapper-authored Combat mode and server-kill damage type are valid after a cold Editor restart.

Current validation is deliberately bounded: `QATS.QCombat.*` passes `16/16`, the complete `/Game` Blueprint sweep passes `4706/4706` with no compile warning, error or load failure, and a Standalone `L_Dev_Rz` run proved authoritative non-lethal damage, reset, lethal server kill and revive with a clean PIE Message Log. Listen-server, dedicated-server and packaged Development/Shipping parity remain open and are not implied by this checkpoint.

### Reparent contract corrections (retry2)

The retry2 pass corrected the reparent contract to use the real QWeapon owner for targeted-by bookkeeping:

- **Authority targeted-by routing through QWeapon**: `DispatchCommittedState` calls `RemoveTargetedBy(Owner)` on the old target's `UQWeaponTargetingComponent` and `NotifyTargetedBy(Owner)` on the new target's component when the replicated target changes on authority. This runs before `ReceiveTargetChangedBookkeeping` so deferred mutations do not publish premature side effects. No second targeted-by store or event is created in QCombat.
- **Authority EndPlay teardown**: when the owner has authority and a valid `CurrentTarget`, `EndPlay` removes the owner from the target's `UQWeaponTargetingComponent` targeted-by set before unregistering the provider.
- **Reentrancy preserved**: no targeted-by side effect fires before the state actually commits. The target transition is observed inside `DispatchCommittedState`, which runs only after `CommitComponentState` has written the new state.

## Evidence inspected

The contract was derived from current source, read-only live Blueprint inspection, cached Find-in-Blueprints data, Unreal Engine headers/implementation, and recent related commits.

- Project rules and the complete migration set through Prompt 08 were read before choosing an owner.
- `/Game/Systems/Combat/CombatComponent`, `/Game/Systems/Combat/Lib_Combat`, `E_Factions`, `E_CombatState`, `WV_SafeArea`, weapon targeting providers, QAI faction/combat/state processors, QPolice interfaces/subsystem, QModule ordnance and QGameManager safe queries were inspected.
- Engine `AActor::TakeDamage` was verified in `E:/UE573/Engine/Source/Runtime/Engine/Private/Actor.cpp`: point damage broadcasts `OnTakePointDamage` first, then every non-zero damage event broadcasts `OnTakeAnyDamage` exactly once.
- Recent commits checked include `3bd272b02`/`49c84e75a` for completed QWeapon registry/targeting ownership, `c769b7ef5` for fail-closed safe-area enforcement, `7f333015d` for static-defense hitscan, `482c10ac3` for point-damage hit/instigator routing, `347ea3417` for environmental death classification, and `5e3caae55` for vehicle/driver Combat decoupling.
- The project-wide Find-in-Blueprints scope contained 4,704 Blueprints. Its cached scope included deferred/out-of-date entries, so the consumer list below is a proved minimum, not a false claim of exhaustive index completeness. Explicit Combat assets and native consumers were inspected directly.
- The current live Asset Registry reports `525` direct package referencers for `/Game/Systems/Combat/CombatComponent`, `22` for `/Game/Systems/Combat/E_Factions` and `62` for `/Game/Systems/Combat/Lib_Combat`. The lists include Blueprint classes, maps, optimized level products and the EasyCook seed. This proves that reparenting or deleting the shared Blueprint contract before source adapters and a transactional per-family migration would break serialized consumers; no bulk asset mutation is authorized by the native core alone.

No `.uasset` was searched as text.

## Audited production baseline

### `CombatComponent_C`

`/Game/Systems/Combat/CombatComponent` is currently a Blueprint `UActorComponent` implementing `QWeaponTargetProvider`. It has approximately 565 nodes and 41 functions. Its important runtime state is split across public mutable Blueprint variables:

- `IsAlive` and `PreviousIsAlive`;
- `CurrentLife` and `LifeWhenNoStat`;
- `Faction` (`E_Factions` byte values 0 through 7);
- `CombatState` (`Disabled`, `PvEOnly`, `PvPOnly`, `PvE+PvP` values 0 through 3);
- `IsPlayerOwner`;
- `CombatInclusiveFilterEnabled` and `CombatInclusiveTags`;
- `ActorTarget` and `PreviousTarget`;
- `InsideSafeZones`;
- `LastDamageCauser` and replicated `LastDamageCauserName`;
- drop/reward/death-presentation data.

The live class defaults are alive with 100/100 component life, `Faction=None`, `CombatState=PvE+PvP`, no target and no last causer. `IsAlive`, `CurrentLife`, `IsPlayerOwner`, `ActorTarget` and the causer text are replicated in the Blueprint. The state is not one atomic replicated contract and the private damage provenance is not owner-only.

At BeginPlay the component:

1. registers itself in QWeapon's native target registry;
2. binds its owner `OnTakeAnyDamage`;
3. either mirrors a Blueprint physical-stat script or initializes `LifeWhenNoStat`;
4. derives player ownership from pawn/controller state;
5. applies faction setup.

At EndPlay it unregisters from QWeapon. The QWeapon interface currently forwards target actor/location, alive state, permission, current target and target commit into Blueprint graphs.

### Current life and death flow

`OnDamage` records the causer/classification and mutates `CurrentLife` only when the Blueprint physical-stat script is absent. Damage is converted into the integer life model without finite/range validation. `OnZeroLife` builds `DeathInfo`, flips `IsAlive`, mirrors optimized QAI state, notifies systems and emits the global pawn-death event on authority.

`OnRep_IsAlive`/`CallUpdateState` drives the visible transition. Alive invokes `OnAlive` and clears damage provenance. Dead invokes `OnDeath`, then downstream drop/reward/AI-stop behavior. `OnRep_CurrentLife` invokes `OnDamaged` for non-zero changes. This division means life, replicated transition and side effects currently have competing graph owners.

The native contract replaces that with one atomic state and one transition comparison. A state can cross `Alive -> Dead` only once for one revision. Repeated or re-entrant lethal events see `Dead` and return `AlreadyDead` without another transition.

### Current death and damage consumers

Proved direct consumers include:

- QAI `UQAI_AgentComponent`, spawners, sensing, recruitment UI, movement and optimized/impostor mirrors;
- QModule stat notification and medical-drone reflected life writes;
- QPolice drone/police reaction and wanted/crime paths;
- base player/AI characters, autonomous/lean AI families, vehicles and police vehicles;
- Sangline/flyer spawners and mission actors bound to `OnDeath`;
- drops, rewards, quest kill dispatch, tracker removal, ragdoll, FX, audio and obituary/presentation graphs;
- vehicle destruction paths reading `LastDamageCauser`.

These stay consumers. Prompt 09 must bind them to native typed state/events and remove each reflected property/delegate bridge only after its replacement is live.

## The real damage funnels

### Player weapons and static-defense hitscan

`UQWeaponBulletSubsystem::ServerFireBullet` runs on authority, performs the authoritative hit trace, preserves the `FHitResult`, resolves controller/causer and calls:

```cpp
UGameplayStatics::ApplyPointDamage(
    Target,
    Damage,
    ShotDirection,
    Hit,
    EventInstigator,
    Shooter,
    UDamageType::StaticClass());
```

QWeapon currently has transitional faction/None/player-owned fall-through logic around the Blueprint provider. That logic may choose whether to call point damage, but it cannot be the final acceptance owner. The native Combat component revalidates the complete policy when the Unreal damage event reaches the target. Prompt 09 removes QWeapon's now-obsolete policy fall-throughs only after the native provider and acceptance funnel are integrated; QWeapon keeps registry, targeting, traces and ballistics.

### QAI and dedicated-server AI

`UQAI_CombatProcessor::ApplyDamageToTarget` is server-only. It verifies target life, queries target safe-area state fail-closed, rejects friendly damage, resolves the instigator controller and calls the same `ApplyPointDamage` entry with the real hit. QAI drone/spacecraft hitscan also routes through QWeapon. Radial nanite/rocket paths use Unreal radial damage and the QGameManager safe-area ignore/query APIs.

This path must never be routed through `Lib_Combat.ClientRequestDamage`. That Blueprint function branches through a local-player requester/RPC path in cases that have no local player on a dedicated server and can silently no-op. Prompt 09 replaces QAI's reflected life/faction checks with QCombat snapshots and keeps the server `ApplyPointDamage` call.

### QModule and other gameplay

`UQModule_RackComponent::Authority_ApplySplashDamage` is authority-gated. It uses point damage for the direct target, `ApplyRadialDamageWithFalloff` for splash, and explicit point damage for self-damage because Unreal radial damage excludes the causer. It preserves controller, causer, impact point and direction. Generic environmental and QAI safe-area lethal paths use `ApplyDamage`, which still reaches `OnTakeAnyDamage`.

Legacy Blueprint `ClientRequestDamage` callers remain in AI utility/base graphs, player/AI base characters and vehicle collision/destruction/friction graphs. Those are integration/P2 debt. They are not copied into QCombat and do not become an alternate native funnel.

### Standalone, listen and dedicated behavior

- Standalone actors have authority; the native component initializes and mutates locally, while the same transition code emits consumer events.
- Listen server authority mutates once. The owning and remote client components observe the atomic replicated state and derive the same transition once per revision.
- Dedicated server owns every gameplay mutation and emits server-side consumer events. It performs no Combat presentation work. Clients receive public state; private damage provenance is owner-only.
- A client-local `ApplyDamage`/`ApplyPointDamage` event is rejected as `NonAuthority` and cannot alter life.

## Permission policy

### Canonical faction matrix

The existing QAI matrix is the audited pure faction relation. Row is source, column is target; `1` is hostile.

| Source \\ Target | None | IcLabs | Dissidence | Infected | Pirate | Animal | Rogue | Voss |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| None | 0 | 0 | 0 | 1 | 1 | 0 | 1 | 1 |
| IcLabs | 0 | 0 | 1 | 1 | 1 | 0 | 1 | 1 |
| Dissidence | 0 | 1 | 0 | 1 | 0 | 0 | 1 | 0 |
| Infected | 1 | 1 | 1 | 0 | 1 | 1 | 1 | 1 |
| Pirate | 1 | 1 | 0 | 1 | 0 | 1 | 1 | 0 |
| Animal | 0 | 0 | 0 | 1 | 1 | 0 | 1 | 0 |
| Rogue | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
| Voss | 1 | 1 | 0 | 1 | 0 | 0 | 1 | 0 |

The matrix is directional and Rogue-versus-Rogue is intentionally hostile. It is not the whole permission policy.

### Ordered complete decision

`QCombatPolicy::EvaluateTargetPermission` is pure and consumes no UObject. It evaluates in this order:

1. both subjects are explicitly available and their state is valid;
2. self-target is rejected;
3. source and target are alive;
4. source and target Combat modes are not disabled;
5. source and target safe-area facts are both exactly `Clear`;
6. a matching source inclusive tag may authorize the target after the safety/life gates;
7. player-owned versus player-owned requires PvP enabled on both subjects;
8. every mixed/NPC case requires PvE enabled on both subjects;
9. a player-owned source may engage a non-player-owned target under PvE, including police; crime/wanted consequences remain QPolice consumers;
10. a police source may engage a player-owned target when that player/driver is wanted, even where the pure `IcLabs -> None` matrix is friendly;
11. remaining NPC-to-player and NPC-to-NPC cases use the directional faction matrix;
12. every refusal returns a typed reason. Missing state never becomes `None` faction or allowed damage.

`bPlayerOwned` is a policy fact, not merely `APawn::IsPlayerControlled()`. Vehicle/turret adapters must publish the controlling driver's ownership and wanted state explicitly. Police identity is also a typed fact; the current QAI police marker and QPolice interfaces are adapters, not a second policy owner.

### Safe-area and wanted semantics

QGameManager exposes the authoritative tri-state result:

- `Clear`: Combat may continue to later gates;
- `Protected`: deny targeting and combat damage;
- `Unavailable`: deny targeting and combat damage fail-closed.

The safe-area `Protected` value is unrelated to `UQPoliceSubsystem::IsWantedLevelProtected`. The latter is a three-second cooldown preventing wanted-level re-add after a clear. QPolice continues to own that mutation rule. Combat only consumes the current typed `bWanted` fact.

Explicit self, environmental and trusted system damage are separate damage origins. They still require authority, game-thread execution, a valid finite positive amount and an alive target, but do not masquerade as faction combat. Auto-classification accepts environment only when there is no instigator and the causer is null or the damaged actor itself; an unknown external causer fails closed.

## Native component contract

### Atomic state

`FQCombatReplicatedState` is the only runtime life/faction/policy state:

- initialization flag;
- `MaxLife`, `CurrentLife` and derived `LifeState`;
- serialized-compatible `Faction` and `Mode` values;
- `bPlayerOwned`, `bPolice`, `bWanted`;
- inclusive-filter flag and bounded tag list;
- monotonically increasing `Revision`.

Authoring defaults configure the first authority initialization but are not a second runtime owner. Every accepted mutation copies the complete state, applies the requested compound change, validates every invariant, increments one revision, then commits. Any invalid candidate leaves the old state byte-for-byte unchanged.

### Source API

The P0 API is intentionally narrow:

| API | Thread/authority contract | Purpose |
|---|---|---|
| `GetCombatStateSnapshot`, `GetIsAlive`, `GetCurrentLife`, `GetLifeWhenNoStat`, `GetFaction`, `GetCombatState`, `GetIsPlayerOwner`, `IsCombatEnabled`, `IsPveEnabled`, `IsPvpEnabled` | Read on game thread for UObject callers; workers use `FQCombatSnapshot` only | Typed public state reads |
| `SetMaximumLifeAndCurrent`, `ResetLife`, `HealLife` | Game thread plus authority | Life configuration/heal through the single compound mutation funnel |
| `SetCombatFaction`, `SetCombatMode`, `SetCombatPolicyState` | Game thread plus authority | Atomic faction/mode/player/police/wanted/inclusive facts |
| `ApplyNativeDamage` | Game thread plus authority; no RPC | Result-bearing native entry behind Unreal damage delegates and explicit trusted native origins |
| `CheckTargetIsAllowedCombat` | Game thread | Temporary Blueprint permission adapter with an explicit provider-present output |
| `BuildPolicySubject`, `BuildSnapshotRecord` | Game thread; record requires current QWeapon registration and a caller-supplied QWeapon stable ID | UObject sampling boundary before worker execution |
| `IQWeaponTargetProvider` implementations | Game thread | Preserve QWeapon registry/selection ownership and server revalidation |
| `QCombatPolicy::*` | Pure/value-only | State, mode, faction, complete permission, damage and replication decisions |
| `QCombatSnapshot::Build` | Game thread | Validate/sort and return one immutable versioned publication |

Temporary Blueprint mutation adapters are exactly `SetLifeWhenNoStat`, `ResetLifePoints`, `SetFaction`, `SetPveEnabled` and `SetPvpEnabled`. Temporary dispatchers are exactly `OnDamaged`, `OnDeath`, `OnAlive`, `OnFactionUpdate` and `CombatStateUpdate`, each with the Combat component parameter. Prompt 09 rewires Blueprint enum pins to `EQCombatFaction`/`EQCombatMode` while reparenting; the Blueprint `E_Factions` asset is not a second native policy type.

`CurrentLife` is in `[0, MaxLife]`, `MaxLife` is positive, enum values are bounded, tag lists are bounded/non-empty, and `LifeState` must agree with zero/non-zero life. A positive finite float damage is converted with the existing integer-life boundary: its positive amount is rounded up to whole life points (equivalent to flooring exact remaining life), so even the smallest positive value applies one point; if it reaches/exceeds current life, life becomes zero. NaN, infinity, zero and negative damage are refused. Point-hit location/direction and an optional hit component must also be valid before mutation.

### One damage-acceptance funnel

The owner actor delegates are used as follows:

- `OnTakePointDamage` only captures the synchronous point metadata stack: instigator, causer, damage type, hit component, bone, location and shot direction.
- `OnTakeAnyDamage` is the only mutation entry. It consumes the matching point metadata when present and calls the same native decision/mutation function used by explicit native requests.

Binding both delegates as mutation owners would double damage because Engine broadcasts both for one point event. QCombat never does that.

The acceptance decision verifies game thread, authority, initialization, finite positive damage, alive state, origin and complete permission. Only an accepted decision writes life or private provenance. A lethal accepted decision commits life zero and emits one death transition. Re-entrant/repeated lethal events return `AlreadyDead`; a post-decision commit refusal returns `MutationRefused` with the exact `EQCombatMutationResult`.

`ApplyPointDamage` remains the common Unreal entry for QWeapon, QAI and direct gameplay during transition. The component does not call `Lib_Combat.ClientRequestDamage` and does not introduce a second RPC damage route.

### Notifications and temporary Blueprint seams

Native detailed delegates are emitted after a committed transition. Exact temporary Blueprint-facing dispatchers retain the existing `OnDamaged`, `OnDeath`, `OnAlive`, `OnFactionUpdate` and `CombatStateUpdate` names with the component parameter. They are post-commit adapter seams only; policy, damage decisions and worker reads contain no property-name lookup, `FindFunction` or manual `ProcessEvent`.

Prompt 09 rewires current consumers to those native seams, then deletes the superseded Blueprint state/mutation graphs. Disabling old graphs is not completion.

### Replication

- Atomic public Combat state: `COND_None`, one RepNotify comparison by revision.
- QWeapon current target: `COND_None`; QWeapon remains request validation/targeted-by owner.
- Last damage causer and last damage type: `COND_OwnerOnly`.
- Damage requests are not new Combat RPCs. Existing authoritative Unreal damage calls remain the entry.
- Server emits gameplay consumer events immediately after commit. Each client emits the corresponding local transition once when the newer atomic revision is applied.
- Dead/alive is derived and validated with current life inside the same struct; replication cannot expose a new-life/old-alive split.

## QWeapon provider integration

`UQCombatComponent` directly implements `IQWeaponTargetProvider`:

- target actor is the component owner;
- aim location uses the owner's native target-location API;
- active state is initialized, alive and Combat-enabled;
- `CanTargetActor` resolves another `UQCombatComponent` by type and runs the complete pure policy;
- missing native provider returns `bHasTargetProvider=false` and denies;
- current-target commit remains game-thread-only and is accepted on authority or through QWeapon's confirmed client mirror path.

BeginPlay/EndPlay register/unregister this component in `UQWeaponTargetRegistrySubsystem`. No QCombat spatial query, octree or replacement target registry is created.

## Immutable QAI snapshot contract

### Data model

`FQCombatSnapshotRecord` contains only values:

- stable non-zero Combat ID;
- Combat state revision;
- finite position and aim location;
- complete `FQCombatPolicySubject` including faction/mode/life/player/police/wanted/tags/safe state.

It contains no raw pointer, weak pointer, `TObjectPtr`, `UObject`, interface or reflected field. `FQCombatSnapshot` contains a monotonically increasing publication version, a topology generation and records sorted by stable ID.

### Publication lifecycle

1. Components register/unregister only with QWeapon on the game thread. QWeapon remains the roster owner and must expose its existing stable IDs, a monotonic topology generation and game-thread resolution through a narrow native integration API in Prompt 09.
2. Before a QAI parallel batch, QAI asks QWeapon for one game-thread roster view and asks each native Combat provider to build a value record with the QWeapon-owned stable ID.
3. `QCombatSnapshot::Build` runs only on the game thread, validates/sorts the records, increments the publication version and returns a new `TSharedPtr<const FQCombatSnapshot, ESPMode::ThreadSafe>` carrying QWeapon's topology generation.
4. QAI captures that shared pointer before launching its synchronous `ParallelFor`. Workers read only the captured const object; no concurrently mutable global pointer or lock is needed.
5. A worker returns stable IDs plus the snapshot version/generation. It never returns or traverses an actor/component.
6. QAI PostProcess accepts a result only when it still refers to the captured/current publication and QWeapon topology generation, then asks QWeapon to resolve the stable ID and commits through the typed provider on the game thread.

An old shared pointer remains immutable after a later publication. Version/generation exhaustion, duplicate IDs, non-finite positions and invalid records fail explicitly. QCombat owns value validation only; it does not store membership, resolve actors, acquire targets or duplicate QWeapon's spatial index.

### QAI violations to remove in Prompt 09

The current `ParallelFor` acquisition path still touches `AActor`, pawn/controller state, actor locations, reflected `CombatComponent.Faction`/`IsAlive`, QAI safe-area collections, and QWeapon provider mutation from worker code. Prompt 09 must move all of those reads to the publication/prepass and all target commits to PostProcess. Merely replacing faction reflection while leaving actor traversal is not worker-safe.

## Module boundary and dependency graph

No existing module is the correct neutral owner:

- QWeapon owns ballistics/target registry and must consume Combat policy, not own life/faction.
- QAI owns an interim pure matrix but already depends on QWeapon and contains the worker consumers being migrated.
- QPolice specializes crime/wanted and depends on QAI; making it the core owner would create the wrong dependency direction and cycles.
- QGameManager owns safe-area spatial truth, not life/damage.
- QModule and physical stats are consumers/configuration adapters, not Combat state owners.

The source dependency direction is:

```text
QGameManager     QWeapon
       \          /
           QCombat
              ^
              |
       QAI / QPolice / QModule / Blueprint adapters (Prompt 09/P2)
```

`QCombat` publicly depends on QWeapon because its public component implements `IQWeaponTargetProvider`, and privately depends on QGameManager for game-thread safe-area sampling. It does not depend on QAI, QPolice, QModule, QATS, UI or Blueprint gameplay modules.

## Staged cleanup

### P0 source boundary, completed

- Add QCombat types, pure policy, atomic component, single damage funnel and immutable snapshot value model.
- Add one isolated QATS Combat source.
- Add this contract and Prompt 07 handoffs.
- Do not modify any live consumer or shared build/config file.

### P1 production integration

- QCombat and its direct QATS/QWeapon dependencies are wired into the project/plugin graph.
- `CombatComponent_C` is transactionally reparented to `UQCombatComponent`; authored max life, faction, mode, player ownership, inclusive tags and server-kill damage type are native defaults.
- The superseded Blueprint runtime state and mutation graphs are removed. Retained Blueprint fields are limited to death presentation, rewards, drops and downstream payloads.
- Character, static-NPC, match and QAI-spawn-zone consumers use the native component contract. The QAI death map and delegate parameter are typed as `UQCombatComponent`.
- Production reads of life/alive use typed native functions; no writable mirrored health compatibility state was added.
- The saved infected animation Blueprint was refreshed so its Combat-state delegate binding cold-loads without stale compiler errors.

The remaining P1 proof is runtime rather than another ownership layer: exercise the retained death/reward/drop/quest consumers and the permission matrix on actual actors, then close the listen-server, dedicated-server and packaged gates below. Any consumer found outside the typed funnel must be migrated and its superseded path deleted before this phase is declared complete.

### P2 consumers

- vehicle collision/friction/destruction and `VehicleCombatComponent`;
- QWeapon vehicle weapons and static-defense turret adapters;
- projectile/radial/gas/nanite consumers with explicit safe-area and origin contracts;
- spawner destructibility and physical-stat configuration adapters;
- legacy character/environmental damage classification and obsolete `Lib_Combat` functions;
- presentation-only death/drop/reward assets after their typed consumer seam is proven.

## QATS source coverage

`QCombatAutomationTests.cpp` covers the source contract without claiming live integration:

- all 64 faction matrix entries, including Rogue self-hostility and directional pairs;
- unavailable subjects, self, alive/dead, disabled mode and inclusive tags;
- both PvP flags and both PvE flags;
- source/target safe-area `Unavailable` and `Protected` fail-closed results;
- friendly/hostile NPCs, player-owned PvE override, player-versus-player, police-versus-wanted/non-wanted player, and player-versus-police;
- non-authority/wrong-thread, NaN/infinite/non-positive damage, integer-life boundaries, lethal transition and repeated lethal exactly-once refusal;
- compound state validation/revision and rollback-by-no-commit;
- public versus owner-only replication field decisions;
- immutable old snapshot after new publication, monotonic version, topology generation, duplicate/invalid-record rejection;
- QWeapon provider registration/unregistration and caller-owned Combat snapshot lifecycle in a scratch world.

The test source has direct `QCombat` and `QWeapon` dependencies in `QAutomatedTestSuite.Build.cs` and matching plugin entries. The integrated suite currently passes `16/16`, including the committed-state bridge and the authored wrapper-default contract. This proves the native state/policy/funnel and the Standalone asset bridge; it does not replace the remaining network and packaged gates.

## Prompt 09 hard gates

The source, asset integration, full Blueprint compile, focused QATS and Standalone smoke gates are closed. The remaining production gates are:

### Static/build

- Linux server compile to preserve the meshless server boundary;
- packaged Development and Shipping compile/load validation;
- cold-load Blueprint preflight after any further Combat asset edit, with no duplicate native/Blueprint variables or functions.

### Standalone

- non-lethal, exact-lethal and repeated-lethal point damage with one `OnDamaged` and one `OnDeath` transition as specified;
- generic environmental/fall damage classification distinct from suicide;
- reset/revive clears private provenance and emits one alive transition;
- every faction/PvE/PvP/inclusive/safe-area/wanted/police case against actual Blueprint actors;
- direct, radial and self ordnance paths; drops/rewards/quests/spawner reactions exactly once.

### Listen server plus one client

- client weapon request produces one server acceptance and one replicated transition per peer;
- forged/client-local damage cannot mutate life;
- atomic life/alive/faction/mode revisions are coherent on owner and non-owner;
- last causer/type reach only the owning connection;
- current QWeapon target confirms/corrects without duplicate targeted-by state;
- wanted and vehicle-driver changes update permission before the next accepted damage.

### Dedicated server plus two clients

- player weapon, QAI direct point damage and QModule ordnance all mutate only on server;
- QAI never routes through `ClientRequestDamage` and still damages with no local player;
- QAI worker acquisition reads only the captured immutable snapshot and PostProcess rejects stale version/generation IDs;
- safe `Protected` and query `Unavailable` both prevent target/damage for all QAI direct/radial/projectile paths;
- one death causes one QAI unregister, one spawner/drop/reward/quest notification and no dedicated presentation work;
- late join receives a coherent atomic dead/alive state and no replayed gameplay death side effects.

## Exact source files

Prompt 07 owns only:

- `Plugins/QCombat/QCombat.uplugin`;
- `Plugins/QCombat/Source/QCombat/QCombat.Build.cs`;
- `Plugins/QCombat/Source/QCombat/Public/QCombat.h`;
- `Plugins/QCombat/Source/QCombat/Public/QCombatTypes.h`;
- `Plugins/QCombat/Source/QCombat/Public/QCombatPolicy.h`;
- `Plugins/QCombat/Source/QCombat/Public/QCombatComponent.h`;
- `Plugins/QCombat/Source/QCombat/Public/QCombatSnapshot.h`;
- matching private `.cpp` files;
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QCombatAutomationTests.cpp`;
- this document and Prompt 07 handoffs.

The current integration additionally owns the wrapper asset, its directly affected Blueprint consumers, the committed-state bridge tests and the dependency/config changes needed to make the native owner live. P2 consumers remain explicitly outside this checkpoint until their typed path and runtime proof are complete.
