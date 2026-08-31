STATE: SOURCE_READY

# Combat source handoff

## Delivered source boundary

`Plugins/QCombat` is the neutral runtime owner for Combat life, alive/dead transition, faction/mode/policy facts, authoritative damage acceptance and worker-safe value records.

- `UQCombatComponent` owns one atomic replicated state and one game-thread mutation funnel.
- Authority binds `OnTakePointDamage` for metadata only and `OnTakeAnyDamage` for the single mutation path. QWeapon/QAI/QModule continue to enter through Unreal `ApplyPointDamage`/damage APIs.
- Positive finite damage is range-safe at integer-life boundaries, including subnormal positive values and maximum finite float damage. Invalid state, amount, point-hit data, thread, authority, permission, repeated-dead and post-decision mutation refusals return explicit results.
- Alive/dead is derived from current life inside the same revision. A lethal commit emits one transition; later lethal requests return `AlreadyDead`. Every revive clears private damage provenance.
- Complete permission covers availability, self, life, modes, both safe-area sides, inclusive tags, both PvP sides, both PvE sides, player ownership, police/wanted and the directional 8x8 faction matrix.
- `Protected` and `Unavailable` both fail closed. QPolice's wanted mutation cooldown remains a separate QPolice rule.
- `LastDamageCauser` and damage type use `COND_OwnerOnly`; public state and current QWeapon target use normal replication.
- `FQCombatSnapshot` contains values only and is returned as a thread-safe shared pointer to const data. Build is game-thread-only, versioned, topology-stamped, sorted and rejects invalid/duplicate records.
- QCombat registers only with `UQWeaponTargetRegistrySubsystem`. It has no second roster, spatial index or actor resolver.
- Competing `UQCombatComponent` instances on one actor fail explicitly before registration/binding, preventing a second life owner.
- Diagnostics are bounded to one warning per second per component.

The source retains temporary Blueprint adapter names only. It does not reparent or save `CombatComponent_C`, change a consumer, add a damage RPC or claim parity.

## Exact owned files

- `Plugins/QCombat/.gitignore`
- `Plugins/QCombat/QCombat.uplugin`
- `Plugins/QCombat/Source/QCombat/QCombat.Build.cs`
- `Plugins/QCombat/Source/QCombat/Public/QCombat.h`
- `Plugins/QCombat/Source/QCombat/Public/QCombatTypes.h`
- `Plugins/QCombat/Source/QCombat/Public/QCombatPolicy.h`
- `Plugins/QCombat/Source/QCombat/Public/QCombatSnapshot.h`
- `Plugins/QCombat/Source/QCombat/Public/QCombatComponent.h`
- matching private `.cpp` files
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QCombatAutomationTests.cpp`
- `Documentation/BlueprintToCppMigration/09_COMBAT_MIGRATION.md`
- `HANDOFFS/07_COMBAT_AUDIT_READY.md`
- this handoff

The plugin-local `.gitignore` excludes only QCombat `Binaries/` and `Intermediate/`. A concurrently running Editor process generated/loaded a QCombat DLL while the source was being written; no generated output belongs in the handoff.

## Prompt 09 integration requests

### Dependency wiring, no cycles

1. Enable QCombat in the project/plugin graph.
2. Verify/retain the direct private `QCombat` and `QWeapon` dependencies now present in the shared dirty `QAutomatedTestSuite.Build.cs`, plus the QCombat/QWeapon plugin entries now present in `QAutomatedTestSuite.uplugin`, before compiling the Combat QATS source. Prompt 07 did not edit either shared file.
3. Add `QCombat` as a consumer dependency to QAI and QPolice. Add it to QModule only when QModule's typed adapters are migrated. Never add QAI, QPolice, QModule or QATS as a QCombat dependency.
4. Preserve `QCombat -> QWeapon + QGameManager`; this is the acyclic core direction.

### QWeapon, sole roster owner

1. Keep `IQWeaponTargetProvider`, `UQWeaponTargetRegistrySubsystem`, spatial queries, targeting RPC validation, ballistics and targeted-by lifecycle in QWeapon.
2. Add one narrow native, game-thread-only roster view to the existing QWeapon registry. Each entry must expose the registry's existing stable ID and provider component for immediate GT sampling; the view must be deterministically ordered.
3. Add a monotonically increasing QWeapon topology generation on register, unregister and stale-entry removal, plus a game-thread `StableId + ExpectedTopologyGeneration` resolver.
4. Do not pass roster entries/components to workers. QAI converts them immediately into QCombat value records.
5. Remove QWeapon's transitional Blueprint policy fall-through only after `CombatComponent_C` is natively reparented. Do not replace or duplicate its registry.

### QAI worker conversion

1. In the GT prepass, acquire the QWeapon roster view, call `UQCombatComponent::BuildSnapshotRecord` with each QWeapon stable ID, and publish exactly one `QCombatSnapshot::Build` result for the synchronous batch.
2. Snapshot every other acquisition fact currently read from mutable state, including recruited-squad, animal-role, vehicle/driver, stealth and distance inputs, or keep the corresponding decision on the game thread. No worker may call `GetWorld`, `GetSubsystem`, `IsValid(UObject)`, actor/component methods, QGameManager, interface `Execute_` functions, reflection or `ProcessEvent`.
3. Capture the const snapshot pointer before `ParallelFor`. Workers return only value decisions and stable IDs tagged with snapshot version/topology generation.
4. In GT PostProcess, reject results whose publication or QWeapon topology no longer matches, resolve IDs through QWeapon, revalidate current Combat permission, then commit through the typed QWeapon provider.
5. Keep authoritative AI damage on `ApplyPointDamage`. Never route server AI through `Lib_Combat.ClientRequestDamage`.
6. For a trusted scripted kill whose causer has no Combat component, call the explicit native `System` origin on the game thread. Normal weapons, melee and AI direct fire remain `Combat` point damage and must provide real instigator/causer/hit data.

### QPolice facts

1. Push `bPolice`, `bWanted` and effective player/driver ownership through `SetCombatPolicyState` on the game thread whenever wanted, possession or vehicle-driver state changes.
2. Preserve QPolice wanted points, decay, crimes, interfaces and the three-second `IsWantedLevelProtected` mutation cooldown. Do not map that cooldown to safe-area `Protected`.
3. Remove reflected police/wanted reads from QAI workers after the typed snapshot facts are live.

### Blueprint and gameplay adapters

1. Transactionally reparent `/Game/Systems/Combat/CombatComponent` to `UQCombatComponent`. Migrate max life, faction, mode, player ownership and inclusive tags; convert `E_Factions`/combat-mode pins to the native enums.
2. Rewire temporary adapters `SetLifeWhenNoStat`, `ResetLifePoints`, `SetFaction`, `SetPveEnabled`, `SetPvpEnabled`, `CheckTargetIsAllowedCombat`, `OnDamaged`, `OnDeath`, `OnAlive`, `OnFactionUpdate` and `CombatStateUpdate`.
3. Delete the superseded Blueprint `IsAlive`, previous-alive, current/max life, faction, mode, player, target, safe-zone and damage-provenance runtime owners and their mutation/replication graphs. Do not leave disabled duplicates.
4. Make physical stats a configuration/consumer adapter. Combat remains the only current-life owner; there must be no mirrored writable health.
5. Bind QAI/spawner unregister, drops, rewards, quests, ragdoll, death FX and other reactions to the committed transition and prove each server gameplay side effect occurs once.
6. Keep obituary/classification presentation downstream. Classify from the native accepted damage event on authority and send the existing presentation payload; do not make owner-only provenance public merely to preserve UI text.
7. Replace QModule medical/reflected life writes with typed life APIs. Preserve direct/radial point data and supply a valid Combat source or an explicit trusted origin rather than weakening fail-closed policy.
8. Route vehicle/turret/projectile and remaining legacy consumers as P2. Do not move their ownership into QCombat or QWeapon during P1.

## QATS source coverage

`QCombatAutomationTests.cpp` contains source tests for:

- all 64 faction pairs and invalid enum bounds;
- every permission dimension, including self, friendly/hostile, player-owned, police/wanted, both mode sides and both safe-area sides;
- wrong-thread/non-authority refusals, invalid state/hit data, zero/negative/NaN/infinite damage, fractional/subnormal/maximum damage and repeated lethal refusal;
- state canonicalization, limits, revision/no-change/exhaustion and replication-audience decisions;
- immutable old snapshots, monotonic versions, topology stamps, GT-only publication and invalid/duplicate record refusal;
- native provider registration/unregistration, fail-closed safe unavailability, server target revalidation and the real `ApplyPointDamage -> point metadata -> AnyDamage mutation` exactly-once funnel in a scratch world.

The file was added but not built or run in Prompt 07.

## Exact Prompt 09 validation gates

### Static/build

- non-unity Win64 Editor compile;
- Linux dedicated-server compile;
- focused `QATS.QCombat.*` automation;
- Blueprint compile/preflight after reparent with no duplicate variables, functions, delegates or enum pins.

### Standalone

- non-lethal, exact-lethal and repeated-lethal point damage; one damage event per accepted hit and one death transition;
- environmental/fall versus self/suicide classification with real causer/instigator/hit data;
- reset/revive clears private provenance and emits one alive transition;
- actual Blueprint actors cover every faction, PvE, PvP, inclusive, safe unavailable/protected, player-owned and police/wanted case;
- direct, radial and self ordnance preserve permission and hit metadata; drops/rewards/quests/spawner reactions occur once.

### Listen server plus one client

- one client weapon request produces one server acceptance and one coherent replicated transition per peer;
- client-local/forged damage and life mutations are refused;
- public life/alive/faction/mode revision is atomic on owner and non-owner;
- last causer/type reach only the owning connection;
- QWeapon target confirmation/correction does not duplicate targeted-by state;
- wanted and vehicle-driver changes affect the next accepted permission decision.

### Dedicated server plus two clients

- player weapon, QAI direct point damage and QModule ordnance mutate only on authority;
- QAI deals damage without a local player and never calls `ClientRequestDamage`;
- QAI acquisition workers read only captured values; stale snapshot version/topology results are rejected before GT resolution/commit;
- safe `Protected` and query `Unavailable` deny targeting and damage for direct, radial and projectile routes;
- one death produces one QAI unregister, spawner/drop/reward/quest notification and no dedicated presentation work;
- late join receives one coherent dead state without replaying gameplay death side effects.

## Verification status

Prompt 07 performed source/API/include/non-unity inspection and focused whitespace review only. It did not compile, run automation, control PIE, mutate assets, cook, package or validate runtime/network parity.
