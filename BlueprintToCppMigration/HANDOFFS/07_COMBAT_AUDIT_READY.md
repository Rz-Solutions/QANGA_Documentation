STATE: AUDIT_READY

# Combat audit handoff

## Chosen owner

Create `Plugins/QCombat` as the neutral native Combat owner.

- QWeapon remains the sole spatial target registry, target selection and ballistics owner.
- QGameManager remains the safe-area spatial-query owner.
- QPolice remains the wanted/crime owner and publishes typed facts into Combat.
- QAI, QModule, vehicles, turrets, spawners, drops, rewards, quests and presentation consume Combat.

QAI is not a valid owner because it is a consumer and already depends on QWeapon. QPolice is specialized and depends on QAI. QWeapon cannot own life/faction without reversing the domain boundary. `QCombat -> QWeapon + QGameManager`, followed by future `QAI/QPolice/QModule -> QCombat`, has no cycle.

## Proved baseline

- `CombatComponent_C` currently splits life, alive/dead, faction, PvP/PvE, safe overlap state, target and damage provenance across replicated mutable Blueprint variables and physical-stat/QAI mirrors.
- Point damage already converges through Unreal from QWeapon, QAI and QModule. Engine broadcasts `OnTakePointDamage` before one `OnTakeAnyDamage`; only AnyDamage may mutate or point hits would be applied twice.
- `Lib_Combat.ClientRequestDamage` depends on a local-player requester/RPC path in branches that can no-op on dedicated server. Server QAI must continue to call `ApplyPointDamage` directly.
- Safe-area policy is tri-state: only `Clear` permits combat; `Protected` and `Unavailable` deny fail-closed.
- QPolice wanted protection is a separate three-second wanted-mutation cooldown, not safe-area `Protected`.
- The canonical 8x8 directional QAI faction matrix is only the pure relation. Complete policy also requires life, modes, inclusive tags, player-owned PvE, PvP on both players, wanted-police escalation and vehicle-driver facts.
- QAI target acquisition currently runs in `ParallelFor` while traversing actors/components, reading reflected Combat properties/safe state and committing a provider target. Every one of those UObject operations must move to game-thread pre/post passes.
- QWeapon's completed `IQWeaponTargetProvider`, registry, targeting RPC and targeted-by lifecycle remain intact. Combat implements that provider and supplies immutable value records without storing a second roster.

Full evidence, matrix, API, replication, cleanup and gates are in `Documentation/BlueprintToCppMigration/09_COMBAT_MIGRATION.md`.

## Native contract

- `UQCombatComponent`: one atomic replicated life/faction/policy state, one game-thread mutation funnel, one authoritative damage acceptance path, exactly-once alive/dead transition and typed QWeapon provider.
- `QCombatPolicy`: pure complete permission and damage/state decisions with explicit unavailable/fail-closed results.
- `QCombatSnapshot`: game-thread-only validation/build of immutable versioned value snapshots. QWeapon remains the sole roster/stable-ID/resolution owner.
- Public state and current target replicate to all; last causer/type are `COND_OwnerOnly`.
- Invalid compound mutations never commit. NaN/infinity/non-positive damage is rejected. Repeated lethal damage returns `AlreadyDead`.
- No property-name lookup, manual `ProcessEvent`, mutable UObject traversal or target commit occurs on a worker.

## Exact proposed files

- `Plugins/QCombat/QCombat.uplugin`
- `Plugins/QCombat/.gitignore`
- `Plugins/QCombat/Source/QCombat/QCombat.Build.cs`
- `Plugins/QCombat/Source/QCombat/Public/QCombat.h`
- `Plugins/QCombat/Source/QCombat/Public/QCombatTypes.h`
- `Plugins/QCombat/Source/QCombat/Public/QCombatPolicy.h`
- `Plugins/QCombat/Source/QCombat/Public/QCombatComponent.h`
- `Plugins/QCombat/Source/QCombat/Public/QCombatSnapshot.h`
- matching private `.cpp` files
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QCombatAutomationTests.cpp`
- Prompt 07 migration document and source handoff

## Dependencies

`QCombat` publicly depends on `Core`, `CoreUObject`, `Engine` and `QWeapon`; it privately depends on `QGameManager`. It must not depend on QAI, QPolice, QModule, QATS, UI or Blueprint gameplay modules.

## Prompt 09 integration requests

1. Verify/retain direct QCombat/QWeapon QATS module and plugin dependencies, then compile non-unity Win64 Editor and Linux server.
2. Transactionally reparent `CombatComponent_C`, migrate authored defaults and remove the Blueprint life/faction/mode/target mutation owners after rewiring.
3. Keep QWeapon's registry/targeting/ballistics; remove only its transitional Combat-policy fall-throughs after the native provider is live.
4. Add a narrow QWeapon native roster view exposing its existing stable IDs, topology generation and game-thread resolver. Publish one QCombat snapshot before QAI `ParallelFor`; workers use values/stable IDs only; resolve through QWeapon and commit on the game thread after version/generation validation.
5. Make QPolice push police/wanted/player-or-driver facts and preserve QPolice's own points/decay/cooldown/crime rules.
6. Replace reflected QAI/QModule/spawner/recruitment/medical life/faction accesses with typed Combat APIs.
7. Bind drops, rewards, quest kills, spawner/QAI unregister, ragdoll, FX and obituary classification to the committed transition and prove each fires exactly once.
8. Preserve `ApplyPointDamage` for weapon/QAI/QModule damage and remove dedicated-server AI use of `ClientRequestDamage`.
9. Route vehicle/turret/projectile/legacy consumers as P2; do not bulk-edit them during the first integration.

Phase 2 source implementation follows immediately. This handoff does not claim compilation, network behavior, Blueprint integration or runtime parity.
