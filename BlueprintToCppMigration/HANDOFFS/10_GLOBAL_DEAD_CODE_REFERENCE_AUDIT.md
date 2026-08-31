# Prompt 10 - migration-wide dead-code, reference and duplicate-owner audit

> Read-only audit snapshot from the shared dirty checkout on 2026-08-30. This is not a completion claim or a deletion authorization. No repository source, asset, config, documentation, build file, seed, Editor state, test state, staging area, or commit was changed by this worker.

## 1. Executive verdict

The migration currently has three materially different situations:

1. Weapon targeting, player interaction, gravity provider/application, player gravity rotation, and QCameraControl have a native runtime owner. Their remaining Blueprint surface is either a living consumer/facade or serialized compatibility debt.
2. QSpringArm, SwimmingDetection, Weapon fire orchestration, Inventory, and Combat now have native source candidates, but the production Blueprint assets have not been integrated into those candidates. In those systems the Blueprint remains the runtime authority; calling the new C++ code the owner would be false.
3. Only the already-removed `/Game/Systems/GravityArea/SM_CylinderCollision` has a complete multi-category deletion chain in the current migration record. Every other wrapper, graph, asset, plugin, redirect, or bridge remains gated.

The most dangerous current duplicates are not cosmetic:

- `QAIGravityAreaResolver` still performs a second reflected/manual gravity query with a non-pruned static cache and a fallback chain, alongside the typed GravityScape character path.
- `QWeaponBulletSubsystem`, `QAI_Faction`, QPolice, the Blueprint `CombatComponent`, and the new dormant `QCombat` policy all currently encode overlapping combat permission/faction/life rules.
- QAI and QModule directly mirror or write Blueprint Combat life.
- QPolice uses real `TakeDamage(1.0f)` calls as AI provocation events.
- the native SpringArm, Swimming, Weapon, Inventory, and Combat implementations exist beside authoritative Blueprint producers but have no production asset owner yet.

The shared checkout advanced during this audit. The previously observed QSystem/QATS declaration holes were then filled in the dirty worktree:

- QSystem now declares NinjaCharacter in both Build.cs and its plugin descriptor;
- QATS now declares QWeapon, QInventory and QCombat in Build.cs and its plugin descriptor;
- `QCombatAutomationTests.cpp` now exists with six source tests.

These are current source declarations, not build/test proof. `QInventory` is still disabled by default and both QInventory and QCombat remain absent from the explicit `QANGA.uproject` plugin list; QATS currently pulls them in for the developer-tool graph, but intended production enablement must be decided explicitly. QWeapon's stable IDs are still private implementation data and no public topology-generation/ID-resolution roster API exists for the planned QCombat/QAI worker snapshot integration.

## 2. Audit boundary and evidence quality

### 2.1 Read sources

The audit read:

- `G:/QANGA/agents.md`;
- every document under `Documentation/BlueprintToCppMigration`, including the central plan and documents 01 through 10;
- the current dirty status and diffs;
- recent migration-related commits, including `3bd272b02`, `49c84e75a`, `976971f95`, `495d08110`, `10d984fdc`, `90f0ba1e4`, `0d57d4dc8`, and `dc587c453`;
- the ten root prompt files under `C:/Users/Rz/Desktop/QANGA_BP_CPP_PARALLEL`.

Other workers' handoff results were not used as audit evidence and this worker did not wait for them.

### 2.2 Blueprint and reference capture

Before the Editor was closed, read-only live queries captured:

- a project index generated at `2026-08-30T15:17:16.899Z` with 58,428 indexed assets;
- only `/Game` content in that index, not plugin content;
- a global Find-in-Blueprints population of 4,704 Blueprints;
- 2,146 deferred entries, 570 out-of-date entries, and 2,558 entries still pending the full index;
- a hard result-page ceiling of 1,000.

Therefore a zero Find-in-Blueprints result from this snapshot is not deletion proof. Plugin Blueprints require separate plugin-scoped inspection. No rescan was run.

Asset Registry reference queries were captured separately for hard, soft, manage, search, and dependency categories. Manage and search references were zero for the audited assets, but a zero in one category was never treated as sufficient.

The final attempted `blueprint_get_components` call on `ALS_Base_CharacterBP` hung the Editor. The user closed the Editor and cannot reopen it yet. No retry was made. Consequently:

- final SCS/direct-child enumeration is an open deletion gate;
- current Blueprint compilation, cold class load, Message Log, PIE, and topology checks are not available from this worker;
- all live counts below are the last captured pre-close snapshot and must be refreshed if assets change during integration.

### 2.3 EasyCook evidence

`Config/EasyCookGenerated.ini` is an informational generated snapshot, not the live seed authority. It contains 76,661 listed entries and still contains the old QCamera wrapper at line 32966. The live seed read before Editor closure contained 76,660 entries and Asset Registry reported zero incoming references to QCamera. The generated QCamera line is stale output; it must not be manually reintroduced into the seed.

## 3. Required state taxonomy

Every conclusion below uses these four states:

| State | Meaning |
|---|---|
| **AUTH** | Living authoritative producer. It currently makes or mutates production gameplay state/output. |
| **FACADE** | Living presentation/facade consumer. It may retain input, UI, quest, animation, tuning, or adapter work but must not reproduce the migrated state. |
| **COMPAT** | Compatibility wrapper required only by serialized assets, redirects, historical maps, or exact Blueprint signatures. It may not contain a second producer. |
| **DEAD-GATED** | No proved living responsibility. It is deletable only after the listed cold-load, reference, resave, build, test, topology, and package gates. |

`DORMANT` is an additional warning, not a fifth deletion state: source exists, but no production asset/caller instantiates it. A dormant native candidate is not AUTH and is not automatically dead.

## 4. Current owner map

| System | Current authoritative producer | Living facade/presentation | Serialized compatibility | Dormant/duplicate/dead state |
|---|---|---|---|---|
| Weapon target acquisition | `UQWeaponTargetRegistrySubsystem`, `IQWeaponTargetProvider`, `UQWeaponTargetingComponent` in QWeapon | `VehicleCombatComponent_C` and `HomingLocker_C` retain weapon-specific eligibility/presentation; `GS_CombatManager_C.EnabledPvP` remains live | Legacy Combat provider Blueprint remains loaded by QWeapon tests | `SAT_FilterCombatComponents` and old manager registry were removed. `UQCombatComponent` is a dormant future provider until the Combat asset is reparented. |
| Player interaction/focus/dispatch | `UQPlayerControllerInteractComponent` owns local detection, focus state, hold state, server validation and dispatch | `PlayerControllerInteractComponent_C` owns input/presentation/quest/highlight/sit behavior; widgets remain consumers | `Interact_Interface_C` and exact reflected signatures remain serialized/live | No wrapper deletion is currently safe. Five reflection-binding maps and several cross-plugin reflection calls remain migration debt. |
| Gravity provider | `AQLevelGravityArea`, `UQGravityAreaComponent` and the GravityScape subsystem | Level environment setup and other non-gravity manager consumers | `GravityAreaComponent_C` and `LevelGravityArea_C` are highly referenced wrappers | The legacy GlobalGravityAreaManager reflection bridge remains live. `SM_CylinderCollision` is already deleted after complete gates. |
| Character gravity | Authored `UQGravityCharacterComponent` on ALS and AILean; it owns direction contract and the sole composed `GravityScale` write | QModule sends only the typed cushion multiplier; QPlatform sends its external vector/alpha override | Native component preserves historical Ninja defaults/child deltas at runtime | No second QModule scale writer remains. Legacy QAI world-position resolver is still separate and live for non-cyborg consumers. |
| Player rotation | `UNinjaCharacterMovementComponent::UpdateSmoothGravityAlignment` in `OnMovementUpdated` | ALS view/gait/animation remains Blueprint | Historical child defaults remain serialized | Empty override `SetGravityDirection` is disabled-but-live because deleting it would re-enable the base virtual implementation. |
| QAI rotation | `UQAI_AgentComponent` owns AILean cyborg body rotation and consumes typed `QuerySurfaceUp` | AnimBP consumes locomotion/rotation results | Other QAI movement, vehicle, floating-pawn and impostor paths still depend on the legacy resolver | `QAIGravityAreaResolver` is a duplicate provider/cache/reflection bridge and eventual DEAD-GATED code, not safe now. |
| Locomotion outputs | `UQGravityLocomotionComponent` owns `Acceleration`, `Speed` and `IsMoving` publication | ALS AnimBP and `PawnNoiseEmitter` consume those historical properties | Reflected property publication is an intentional temporary bridge | Old Blueprint calculation/state was removed, but cold Editor/QATS/PIE validation remains open. |
| QCameraControl | `UQCameraControlComponent` in QSystem; ALS SCS was observed using the native class directly | ALS calls `SetActive` and consumes rotation | CoreRedirect protects old serialized wrapper class paths | Wrapper has zero incoming references and is DEAD-GATED, not safe solely from that zero. |
| QSpringArm | `QSpringArm_Component_C` is still AUTH: 340 nodes, per-frame camera output and 5 Hz sweeps | ALS and drone consume `Out_Location` | None yet; the asset has not been reparented | `UQSpringArmComponent` is DORMANT. `Cy_ASpringArm` remains required by the BP asset. |
| SwimmingDetection | `SwimmingDetection_C` is still AUTH: 65 nodes and 20 Hz movement/force behavior | ALS and AI base characters consume it | None yet; the asset has not been reparented | `UQSwimmingDetectionComponent` is DORMANT. Its NinjaCharacter dependency is now declared but has not been compiled by this audit. |
| WeaponScript fire orchestration | `WeaponScript_C` and `WeaponScriptMacros_C` own cadence, ammo, reload, aiming and RPC orchestration | weapon children, QWeapon AnimInstance, QAI, QATS and UI are consumers | exact Blueprint fields/functions are reflected by native consumers | `UQWeaponFireControlComponent` is DORMANT. Its source is not reached by production callers. |
| Inventory/ItemsManager/DataManager | `InventoryComponent_C`, `Obj_ItemInstance_C`, `Lib_Inventory_C`, `ItemsManagerGS_C` and DataManager content remain AUTH | QModule, DynamicQuestSystem, QAI recruitment, shops/quests/UI are consumers | many exact reflected Inventory/DataManager names are live | QInventory records/transaction/codec are DORMANT. DataManager's empty C++ scaffold is DEAD-GATED; the content plugin is not. |
| Combat life/damage/faction/permissions | `CombatComponent_C` and `Lib_Combat_C` remain runtime AUTH; QAI/QPolice/QWeapon contain live transitional policy and mirrors | death/drop/reward/UI/AI reactions are consumers | QWeapon tests and hundreds of assets require the BP class path | `UQCombatComponent`/policy/snapshots are DORMANT source. QAI/QWeapon/QPolice/QModule are current second owners that must be rewired before BP deletion. |

## 5. Blueprint, Asset Registry and cook snapshot

All counts are incoming package references captured before Editor closure. “+ seed” means the EasyCook seed was the soft reference. Manage/search were zero for every row.

| Asset | Live shape | Incoming references | EasyCook/current implication |
|---|---|---:|---|
| `/Game/Systems/Vehicle/Weapon/VehicleCombatComponent` | native child, 7 vars, 11 functions, 156 nodes | 73 hard + 1 soft = 74 | live facade |
| `/Game/Systems/Vehicle/Weapon/HomingLocker` | native child, 5 vars, 134 nodes | 13 hard + 1 soft = 14 | live facade |
| `/Game/Systems/Combat/CombatComponent` | BP ActorComponent, 26 vars, 41 functions plus interface graphs, 565 nodes | 524 hard + 1 soft = 525 | authoritative; cannot delete |
| `/Game/Systems/Combat/GS_CombatManager` | `EnabledPvP` plus a tiny graph | 2 hard + 1 soft = 3 | manager registry removed, remaining contract live |
| `/Game/Systems/Character/Blueprints/PlayerController/PlayerControllerInteractComponent` | native child, 4 vars, 205 nodes | 3 hard + 1 soft = 4 | active facade |
| `/Game/Systems/Character/Blueprints/Interaction/Interact_Interface` | six interface functions | 136 hard + 1 soft = 137 | active serialized API |
| `/Game/Systems/GravityArea/GravityAreaComponent` | empty native wrapper | 189 hard + 1 soft = 190 | COMPAT, not deletable |
| `/Game/Systems/GravityArea/LevelGravityArea` | empty native wrapper plus `EnvironmentSetupComponent` | 195 hard + 24 soft = 219 | COMPAT; maps and soft paths remain |
| `/Game/Systems/QSpringArm/QCameraControl_Component` | 0 vars, 0 nodes, native reparent | 0 hard/soft/manage/search | DEAD-GATED; stale generated cook line remains |
| `/Game/Systems/QSpringArm/QSpringArm_Component` | BP SceneComponent, 41 vars, 9 functions, 340 nodes | 11 hard + 1 soft = 12 | authoritative BP and Cy_ASpringArm consumer |
| `/Game/Systems/Character/Blueprints/CharacterLogic/SwimmingDetection` | BP ActorComponent, 5 vars, 65 nodes | 10 hard + 1 soft = 11 | authoritative BP |
| `/Game/Systems/Weapon/WeaponScript` | 41 vars, 40 functions, 23 events, 973 nodes | 105 hard + 1 soft = 106 | authoritative BP |
| `/Game/Systems/Weapon/WeaponScriptMacros` | one 23-node automatic-loop macro | 26 hard + 1 soft = 27 | live scheduler |
| `/Game/Systems/Item/InventoryComponent` | 44 vars, 67 functions, 30 events, 13 dispatchers, about 1,595 nodes | 124 hard + 2 soft = 126 | seed plus `DA_QuestSystemPrimaryAsset` soft reference |
| `/Game/Systems/Item/Obj_ItemInstance` | 18 vars, 35 functions, 282 nodes | 89 hard + 1 soft = 90 | authoritative mutable item |
| `/Game/Systems/Item/Lib_Inventory` | 20 functions, 303 nodes | 81 hard + 1 soft = 82 | live mutation helper |
| `/Game/Systems/Item/ItemsManagerGS` | 21 vars, 33 functions, 547 nodes | 19 hard + 1 soft = 20 | live global materializer/map |
| `/DataManager/GameDataManager` | content BP component | 25 hard + 1 soft = 26 | live; has nine hard outgoing dependencies |
| `/DataManager/DataObject` | content object | 72 captured referencers | live persistence contract |
| `/DataManager/PersistentDataComponent` | content component | 194 captured referencers | live persistence contract |
| `/DataManager/DataManagerLib` | content library | 52 captured referencers | live lookup facade |
| `/Game/Systems/Combat/Lib_Combat` | seven functions, 171 nodes | 61 hard + 1 soft = 62 | live faction/damage facade |
| `/Game/Systems/GravityArea/SM_CylinderCollision` | file absent/deleted | zero in all captured categories | only fully proved deletion in this audit |

Important outgoing dependencies:

- QCamera's only outgoing hard dependency was `/Script/QSystem`.
- QSpring has hard dependencies on PhysicsCore, Cy_Trace and `Cy_ASpringArm`.
- Inventory's extra soft referencer is `/Game/Systems/QuestSystem/DA_QuestSystemPrimaryAsset`.
- QSpring, Swimming, Weapon, Inventory, Combat and both gravity wrappers still have live EasyCook entries.

## 6. System audit

### 6.1 Weapon target acquisition

Current source ownership is coherent:

- `Plugins/QWeapon/Source/QWeapon/Public/QWeaponTargetRegistrySubsystem.h:24` owns registration and spatial query;
- `Plugins/QWeapon/Source/QWeapon/Public/QWeaponTargetProvider.h:16` is the typed provider contract;
- `Plugins/QWeapon/Source/QWeapon/Public/QWeaponTargetingComponent.h:20` owns acquisition, selection, request validation and targeted-by state.

The removed `SAT_FilterCombatComponents` symbol is absent. The old `GS_CombatManager` add/remove registry functions are asserted absent, while `EnabledPvP` remains live. Do not delete `GS_CombatManager`.

Deletion blocker omitted from the high-level owner claim: QWeapon's embedded tests hard-load legacy assets:

- `QWeaponTargetRegistrySubsystem.cpp:479-482` loads `CombatComponent_C`;
- `QWeaponTargetRegistrySubsystem.cpp:574-589` loads `GS_CombatManager_C` and `CombatComponent_C` and asserts the old manager functions are absent;
- `QWeaponTargetingComponent.cpp:911-961` exercises the real Blueprint Combat provider.

Those tests are real serialized consumers. Combat integration must either retain the same wrapper path or migrate the fixtures before deleting the BP class.

`AsyncBlueprintsExtension` is not a deletion candidate. It remains declared by `QANGA.uproject`, `EssentialMacros.uplugin`, and `DataManager.uplugin`.

When `CombatComponent_C` is reparented to `UQCombatComponent`, strip the BP BeginPlay/provider registration in the same transaction. The new native component registers with QWeapon at `QCombatComponent.cpp:131-147`; retaining a Blueprint registration path would create a second lifecycle owner.

### 6.2 Player interaction/focus/dispatch

`UQPlayerControllerInteractComponent` is the production owner. The BP wrapper is not dead: its 205 nodes still handle input and presentation. `Interact_Interface` has 137 references and is an active serialized contract.

Reflection bridges still live in `Plugins/QSystem/Source/QSystem/Private/Component/QPlayerControllerInteractComponent.cpp`:

- exact `HasInteraction` signature/binding cache around lines 346-481, static map at 415;
- `InteractServer` signature/binding cache around 495-581, static map at 535;
- `InteractLost` dispatch around 585-598;
- ALS view mode lookup/cache around 649-686;
- drivable-vehicle library cache around 875-915;
- pawn-in-vehicle library cache around 984-1026;
- interface load and signature checks around 1073 and 1266-1280;
- live interaction invocation around 1756-1860;
- focus/dispatch/lost paths around 2045-2051, 2245, and 2307-2367.

The five static `TMap<TWeakObjectPtr<UFunction>, ...>` maps at lines 415, 535, 649, 875 and 984 do not visibly prune invalid weak keys. They are live compatibility caches, not safe deletions, but they need bounded cleanup or class-reinstance invalidation proof.

Cross-module reflection remains:

- `QTrainSeatActor.cpp:630` and `:686` invoke `SV_Interact`;
- `QTrackElevator.cpp:384-393` invokes `SetHasInteraction`;
- `RequesterOptimizedState.cpp:465-472` invokes `InteractServer`.

These bridges cannot be removed just because the QSystem implementation is typed. They cross current module/Blueprint boundaries and need either a neutral typed interface or a proved dependency direction first.

No interaction wrapper/plugin deletion is safe now. Required gates are cold packaged class loading, the complete interaction BP compile set, repeated instant/hold/vehicle-exit behavior, Listen and Dedicated topology, forged-request rejection, clean Message Log, and user packaged validation.

### 6.3 Gravity provider, character gravity, rotation and locomotion

#### Native owner now live

The old memory/checkpoint that said the character component was not authored is stale. Current docs/assets state:

- `ALS_Base_CharacterBP` and `ALS_Base_CharacterBP_AILean` contain `UQGravityCharacterComponent`;
- the replaced Blueprint direction/scale writers and `UpdateNinjaGravityDirection` were removed;
- QModule now calls `SetGravityCushionMultiplier` at `QModule_LegacyFacade.cpp:238-260` and no longer writes character `GravityScale`;
- QAI caches the typed component at `QAI_AgentComponent.cpp:2807` and its cyborg path queries `QuerySurfaceUp` around `4063-4154`.

`UQGravityCharacterComponent` owns the composed scale write at `QGravityArea.cpp:1180-1835`. `UQGravityLocomotionComponent` owns the three legacy locomotion outputs at `QGravityArea.cpp:1840-2055`.

Player rotation is native:

- `UNinjaCharacterMovementComponent::OnMovementUpdated` at `NinjaCharacterMovementComponent.cpp:3875-3898` selects the smooth or immediate writer;
- `UpdateSmoothGravityAlignment` is at `3901-3948`;
- `GetComponentDesiredAxisZ` at `6739-6747` returns the current axis when smooth alignment owns the update, preventing the immediate path from becoming a second writer.

QAI owns AILean rotation; the typed cyborg path intentionally leaves directionless/unavailable contracts unchanged rather than inventing world up.

#### Living duplicate gravity provider

`Plugins/QAI/Source/QAI/Private/QAI_GravityAreaResolver.h` remains a second gravity implementation:

- lines 236-248 load `/Game/Systems/GravityArea/Lib_GravityArea.Lib_GravityArea_C`;
- lines 251-280 find and invoke `Gravity_CheckGravityAtLocation` via `ProcessEvent`;
- lines 283-347 manually read radius, mode, arrow and gravity scale and recompute surface up;
- line 360 owns a static `TMap<TWeakObjectPtr<AActor>, FAreaCache>`;
- lines 365-390 retain a 0.25 s per-actor cache with no invalid-key pruning;
- lines 401-437 fall through from the BP resolver to QAI floating movement, actor up, world gravity and finally world up.

Live callers include:

- `QAI_AnimInstance.cpp:23`;
- `QAI_FloatingPawnMovement.cpp:741`;
- `QAI_MovementProcessor.cpp:1467, 1522, 1789, 2433, 2733`;
- `QAI_ImpostorSubsystem.cpp:1842`;
- `QAI_AgentComponent.cpp:4184` and `:5332`.

This is not safe to delete because non-cyborg movement, vehicle, autonomous and impostor paths still call it. The correct deletion gate is a typed GravityScape world-position query with explicit valid, directionless and unavailable results, migration of every caller, zero source/FIB references to `Lib_GravityArea` from QAI, and movement/vehicle/impostor runtime validation. Do not preserve its fallback chain.

#### Other gravity debt

- `QGravityArea.cpp:70-78` and `:133-245` retain the legacy GlobalGravityAreaManager reflected add/remove bridge. It is still required while the Blueprint manager owns other consumers.
- `UQGravityLocomotionComponent::ResolveOwnerBridgeProperties` at `QGravityArea.cpp:1883-1970` is an intentional COMPAT publisher for `Acceleration`, `Speed` and `IsMoving`. Remove it only when the AnimBP/noise consumers are typed.
- `UNinjaCharacterMovementComponent::SetGravityDirection` at `NinjaCharacterMovementComponent.cpp:2191-2194` is an empty override marked “Disabled”. No direct source caller was found, but deleting the override changes virtual dispatch by re-enabling the base implementation. It is disabled-but-live, not dead.
- `ANinjaPhysicsVolume` remains required by rope/cable consumers. Its name is not deletion evidence.
- `GravityAreaComponent_C` and `LevelGravityArea_C` have 190 and 219 incoming references. They are COMPAT, not deletion candidates.

The J3 deletion of `SM_CylinderCollision` is the model to follow: 133 worlds, 368 instances, 42 BP referencers, zero authored CustomShape, regeneration of 21 QLevel optimized products, exact seed removal, zero references in every category, wrapper/component inspection, QATS/PIE/Message Log and Linux non-unity evidence preceded deletion.

Documentation drift:

- `03_CHARACTER_GRAVITY_MIGRATION.md:3` says the J3 Linux gate is finished, while line 189 still calls it remaining;
- lines 206-218 describe J4 in future tense, while lines 220-232 describe it as integrated;
- current J4/J5/J6 source was statically revised after older successful probes, so those probes do not validate the present revision.

### 6.4 QCameraControl and QSpringArm

#### QCameraControl

`UQCameraControlComponent` is the current owner:

- lifecycle/activation begins in `QCameraControlComponent.cpp:10-92`;
- local-controller ownership is resolved around `122-165`;
- rotation output is `ApplyCameraRotation` at `168-183`.

The wrapper has zero hard, soft, manage and search references. ALS was observed using `/Script/QSystem.QCameraControlComponent` directly with the CineCamera child. However the final SCS query hung the Editor, so this must be rechecked cold before deletion.

Keep `Config/DefaultEngine.ini:170`:

`+ClassRedirects=(OldName="/Game/Systems/QSpringArm/QCameraControl_Component.QCameraControl_Component_C",NewName="/Script/QSystem.QCameraControlComponent")`

The redirect protects excluded/historical map copies. The stale line `Config/EasyCookGenerated.ini:32966` is not a live seed reference.

QCamera wrapper deletion gates:

1. cold UHT/class load of QSystem and the redirect;
2. direct SCS/component-template and child-index verification on ALS;
3. compile ALS plus the three `SetActive` call sites;
4. cold load/resave boundary for the eight historical map packages or equivalent redirect proof;
5. zero references in every Asset Registry category and current FIB;
6. focused QATS, `L_Dev_Rz` possession/view-mode/near-camera PIE and topology;
7. clean Message Log, Windows/Linux non-unity build, and user packaged validation.

#### QSpringArm

`QSpringArm_Component_C` is still the authoritative producer. It remains a plain `SceneComponent`, not a child of `UQSpringArmComponent`. Its 340 nodes and 12 incoming references are live.

`UQSpringArmComponent` is source-ready but DORMANT:

- constructor/lifecycle starts at `QSpringArmComponent.cpp:52`;
- local/dedicated ownership gates are around `61-181` and `626-665`;
- output broadcast is around `383-399`;
- its 5 Hz work uses `NextWhiskerUpdateTime` and a polled deadline around `403-410`;
- there is no `FTimerManager` scheduler in the native component.

The Blueprint macro/timer logic therefore remains the runtime scheduler until asset integration.

`Cy_ASpringArm` is not yet deletable:

- the BP SpringArm asset has a hard dependency on it;
- `UCy_ASA_FLibrary` still supplies sphere/probe/whisker functions;
- `UCy_ASpringArm_Component` enables tick in its constructor, but `BeginPlay` and `TickComponent` only call `Super`;
- `Plugins/Cy_ASpringArm/Source/Cy_ASpringArm/Private/Cy_ASAc_Detector.cpp` is a 38-byte include-only translation unit;
- no external C++/Build/project dependency was found outside the plugin, but serialized content dependency is decisive.

The QAI optimization block at `QAI_AgentComponent.cpp:2941-2960` scans every component by class/name fragments (`SpringArm`, `Camera`, `DynamicFlight`, `QPlatform_2_Player`) and disables ticks. Native Camera/Spring already self-gate. This broad name-based producer must be replaced with exact typed ownership and removed after Spring integration; otherwise it remains a second tick owner and can disable unrelated future components.

Required asset cleanup:

- reparent `/Game/Systems/QSpringArm/QSpringArm_Component` to `UQSpringArmComponent`;
- reclass the ALS SCS node and retype ALS/drone bindings;
- strip `Update`, `Whiskers_Update`, probe helpers, duplicate state, disconnected debug islands and the dead `Enabled` contract;
- compile `ALS_Base_CharacterBP`, AILean, `AI_Cyborg` and `IS_DroneBase`;
- resave all 11 hard referencers and remove the exact seed only after live zero;
- then cold-load and delete the wrapper;
- only afterward prove zero C++/asset/plugin references and delete `Cy_ASpringArm`.

### 6.5 SwimmingDetection

The BP `/Game/Systems/Character/Blueprints/CharacterLogic/SwimmingDetection` remains AUTH: parent `UActorComponent`, five variables, 57 EventGraph nodes, 65 total nodes, 11 incoming references.

`UQSwimmingDetectionComponent` is DORMANT:

- declaration at `QSwimmingDetectionComponent.h:41`;
- tick/cadence defaults at `QSwimmingDetectionComponent.cpp:122-130`;
- lifecycle hardening at `153-228`;
- local 20 Hz force path at `226-299`;
- dependency resolution at `312-407`;
- ALS reflection and `ProcessEvent` around `441-477`.

It includes `NinjaCharacterMovementComponent.h`. During the audit, `QSystem.Build.cs:24` and `QSystem.uplugin:34-35` gained the required NinjaCharacter declarations. That closes the static dependency omission, but no compile/UHT result is claimed.

Exact currently known serialized referencers requiring cleanup/resave:

- `/Game/Systems/Character/Blueprints/CharacterLogic/ALS_Base_CharacterBP`;
- `/Game/Systems/Character/Blueprints/CharacterLogic/AI_BaseCharacter`;
- `/Game/Maps/Lobby/L_Lobby`;
- `/Game/Maps/Lobby/Lobby_Cyborg_Halloween/Lobby_Cyborg_Halloween`;
- `/Game/Maps/Lobby/Lobby_Cyborg_Christmas/Lobby_Cyborg_Christmas`;
- `/Game/Maps/Lobby/Lobby_Cyborg_V1/Lobby_Cyborg_V2`;
- `/Game/Maps/ScreenshotShowdown/L_Showdown`;
- `/Game/Maps/Lobby/_OLD/L_Sub_MainMenu`;
- `/Game/Maps/Universe/_LevelTest/L_Persistent_DEV_2`;
- `/Game/Maps/Universe/_LevelTest/L_Persistent_DEV_3`;
- `/Game/EasyCook/DA_EasyCookSeed_QANGA`.

Do not remove the AI base component merely because its template matches defaults. Reparent and prove local-player eligibility first, then inspect every map instance override. Required gates are QSystem/QATS dependencies, cold compile/UHT, BP reparent and strip, exact SCS/direct-child topology, all referencer compiles, QATS, player/AI/dedicated topology, water-entry/exit parity, clean Message Log, Linux non-unity and packaged validation.

### 6.6 WeaponScript fire orchestration

`UQWeaponFireControlComponent`, typed ammo/context/presentation adapters and its server RPCs exist in:

- `Plugins/QWeapon/Source/QWeapon/Public/QWeaponFireControlComponent.h`, interfaces around lines 173, 201 and 291, component at 413, RPCs around 498-504, replicated state around 521-533;
- `QWeaponFireControlComponent.cpp`, tick at 468-489, RPCs at 594-604, shot execution from 861, native bullet submission around 1017, presentation around 1557 onward.

No production source caller or serialized owner was found. Only QATS includes the header. QATS now declares QWeapon in its Build.cs and plugin descriptor, but the test has not been run here. Therefore the new component remains DORMANT, while `WeaponScript_C` and `WeaponScriptMacros_C` remain AUTH.

Living legacy consumers/bridges:

- `QuestTestSubsystem.cpp:14564` invokes `GetInfiniteAmmo` by name;
- `QuestTestSubsystem.cpp:22677-22717` requires an authored `FireLocation` component;
- `QWeaponAnimInstance.cpp:1620-1622` invokes `UpdateCurrentAimpointPosition`;
- `QWeaponAnimInstance.cpp:1642-1646` reads `CurrentAimpointScene`;
- `QWeaponAnimInstance.cpp:1663-1668` reads `DistanceFromCamera`;
- `QAI_CombatProcessor.cpp:157` owns static `BPTriggerLatched` for the legacy `Combat_1stTrigger` path;
- QAI class-name/muzzle probing occurs at `QAI_AgentComponent.cpp:3177` and `:6225-6245` and `QAI_CombatProcessor.cpp:222-284`;
- `Combat_1stTrigger` dispatch remains at `QAI_CombatProcessor.cpp:736, 1329, 1337, 3579-3639`.

`BPTriggerLatched` and its release calls become DEAD-GATED only after QAI uses a typed trigger adapter. Until then they prevent a stuck held input and are live.

The first valid integration asset is:

`/Game/Items/Weapons/NashV2/Assault_Rifle/IS_NashV2_Assault_Rifle`

Only after its production adapters and topology pass may its superseded cadence, ammo, reload, RPC, tracer and damage graphs be removed. `WeaponScriptMacros` can be deleted only after all 27 references are zero.

Separate asset defect:

`/Game/Items/Weapons/ShotGun/IS_Shotgun` has live inherited `Damage=0` while its child `Damage_0` tuning (60 and phase values 200/230/250/270) has no linked getter. `Damage_0` is dead data, but its deletion/value correction is gated on an explicit balance decision and a real shotgun shot validation.

QAI direct hitscan is a distinct server-authoritative agent ballistics path. Do not delete it merely because player WeaponScript orchestration migrates.

### 6.7 Inventory, ItemsManager and DataManager

QInventory currently contains value records, validation, atomic two-endpoint transfer and a versioned codec:

- `QInventoryTypes.h:103, 151, 206, 265`;
- `FQInventoryEndpoint` and backend contract in `QInventoryTransaction.h:16-77`;
- `FQInventoryTransaction::TryMoveWholeRecord` at `QInventoryTransaction.h:90`;
- codec entry points at `QInventoryCodec.h:12` and `:17`.

It has no production caller. The only external source consumer is `QInventoryAutomationTests.cpp`. `QInventory.uplugin` is `EnabledByDefault=false`; QATS now explicitly enables it, but `QANGA.uproject` has no explicit production entry and no gameplay asset/caller uses it. QInventory is a DORMANT migration core, not the runtime Inventory owner and not a deletion candidate while this migration is intended.

Living reflected/mutation bridges include:

- `QModule_InventoryBridge.cpp:14-24` exact class/function/property names;
- `GetInventoryItems` invocation at approximately 119-125, 187-195, 239-246 and 547-555;
- `ItemDataAsset` reflection at 157, 217, 267, 535 and 585;
- `ServerConsumeItem` invocation at 286-315;
- manual full-target/stack guard at 331-342;
- dynamic `GenerateNewItemInstance` resolution at 359-446;
- `AddItemToInventory` invocation at 463-476;
- DynamicQuestSystem NPC dialogue direct `InventoryItems` access around 603 and `GetInventoryItems` around 635-644;
- direct stack write/remove/manual refresh/save in that path around 861-975;
- scanner soft class and item-tag reflection in `ScannerObjective.cpp:78` and approximately 335-1066;
- offline tutorial Inventory/DataManager reflection and direct array operations;
- QAI recruitment `GenerateNewItemInstance` and add/slot/replace calls around `QAI_AgentComponent.cpp:1064-1145`.

The new value contract does not make those bridges obsolete until a production adapter materializes a complete BP state and commits through one verified writer. In particular, no Inventory or item graph is safe to strip now.

DataManager:

- the plugin contains 17 live content assets;
- `GameDataManager` has 26 incoming references, `DataObject` 72, `PersistentDataComponent` 194 and `DataManagerLib` 52;
- QModule and DynamicQuestSystem locate `GameDataManager_C` by class name and call its Blueprint API;
- `DataManager.uplugin` still declares VaRest, EssentialMacros and AsyncBlueprintsExtension.

The DataManager plugin is not deletable. Its C++ scaffold is a separate DEAD-GATED candidate:

- `UDataManagerBPLibrary` at `DataManagerBPLibrary.h:28-32` declares no functions;
- its cpp only has an empty constructor at lines 7-11;
- `FDataManagerModule::StartupModule/ShutdownModule` are empty at `DataManager.cpp:7-18`;
- no external Build.cs/uplugin dependency on the DataManager module was found.

Before converting DataManager to content-only or deleting the library/module, require plugin Asset Registry and FIB coverage for `/Script/DataManager`, cold-load all 17 content assets, compile their Blueprint dependency closure, verify no serialized native class/import, run Windows/Linux non-unity, and validate a user package. Current project index excludes plugin content, so this proof is open.

QStorage remains a separate persistence-mechanics owner and is not a deletion candidate.

### 6.8 Combat life, damage, faction, permissions and snapshots

At the final inspected snapshot, QCombat source contains:

- `FQCombatReplicatedState` and policy/value types;
- pure `QCombatPolicy`;
- UObject-free `QCombatSnapshot` value records and builder;
- `UQCombatComponent` implementing `IQWeaponTargetProvider`.

The component is non-ticking, binds `OnTakePointDamage` only for metadata and `OnTakeAnyDamage` as the single mutation funnel, registers only in QWeapon, uses owner-only damage provenance, and samples QGameManager safe state. Relevant locations:

- lifecycle and damage delegates: `QCombatComponent.cpp:51-92`;
- replication: `97-104`;
- QWeapon provider registration: `148-181`;
- mutation API and commit/rep-notify path: `205-505`;
- safe-area cache, policy subject and snapshot record: `508-581`;
- typed permission: `585-620`;
- damage-source resolution and damage funnel: `623-847`;
- QWeapon provider decisions/commit: `861-910`.

`QCombatSnapshot::Build` validates/sorts immutable records but deliberately owns no roster. QWeapon already assigns private stable IDs in `FRegisteredTarget`/`FTargetSnapshot` (`QWeaponTargetRegistrySubsystem.h:50-70`), yet exposes no public stable-ID roster view, monotonic topology generation, or game-thread ID resolver. That missing narrow API is an explicit integration hole; creating a second QCombat registry would violate ownership.

This source is not AUTH yet:

- `CombatComponent_C` still has parent `ActorComponent`, not `UQCombatComponent`;
- no production caller of `UQCombatComponent` was found;
- the BP has 525 references and remains the life/damage/faction/target authority;
- QCombat has no content and no project asset integration;
- `QCombatAutomationTests.cpp` now contains six source tests, and QATS now declares QCombat/QWeapon dependencies;
- none of those tests, the module, or the asset integration was executed by this worker.

Current duplicate policy/state owners:

1. `QAI_Faction` mirrors the BP enum and reads `CombatComponent.Faction` by class/property name at `QAI_Faction.cpp:25-93`. Its hostility matrix is at `:102-149`. The comment claiming diagonal factions are always friendly is inconsistent with the Rogue self-entry.
2. `QWeaponBulletSubsystem.cpp:714-736` treats missing/unreadable alive state as alive. Lines 752-796 probe legacy Combat health. Lines 799-939 combine provider checks, safe-zone checks, None-faction exceptions, player-owned same-faction exceptions and a fallback faction policy. Lines 954-983 apply point damage.
3. QPolice applies its own faction/crime decisions in `QPoliceLibrary.cpp:30-49` and `:178-187` and `QPoliceSubsystem.cpp:1226-1261`.
4. `UQAI_AgentComponent` binds BP `OnDamaged`/`OnDeath` by name at `QAI_AgentComponent.cpp:4700-4763`, caches the component at `4767-4789`, reflects life into `ReplicatedCurrentLife/MaxLife` at `4792-4835`, and directly writes `CurrentLife` for follower regeneration at `4838-4886`.
5. `QModule_MedicalDroneActor.cpp:185-238` reflects and directly writes BP `CurrentLife`, then forces the QAI mirror refresh.
6. `CombatComponent_C` and `Lib_Combat_C` still own the production damage/life/faction/safe-area graphs.
7. Dormant `QCombatPolicy` now duplicates those semantics in source.

QPolice contains concrete dead and dangerous legacy work:

- `QPoliceSubsystem.cpp:4537-4545` finds `CombatComp` and never reads it;
- `:4551-4559` finds `CombatComponent` and never reads it;
- `:4574` assigns `SpawnedClassName` and never reads it;
- `:4561-4572` asks for the first arbitrary `UActorComponent`, then conditionally treats `InsideSafeZones` as an `FObjectProperty`. The BP field is a collection, so this branch does not perform the intended clear;
- `:4581-4582` applies real one-point damage solely to trigger `OnDamaged`;
- `:7461` repeats the same real-damage provocation for peaceful-to-aggressive conversion.

The unused locals and non-effect property branch are source-cleanup candidates. The real damage calls are not dead: removing them without a typed provocation/aggression event would remove current behavior. Replace the behavior first; do not preserve damage as an event bus.

The recurring proximity timer at `QPoliceSubsystem.cpp:4534` and the two-second spawn-protection timer at `4584-4597` are live. No unused timer was proved here.

`IQCombatant` under `Plugins/QPolice/Source/QPolice/Public/Interfaces/QCombatant.h` has no C++ implementer/caller beyond `QInterfaceQueryLibrary` template/query functions. Blueprint implementations are unknown because plugin FIB/SCS inspection is unavailable. It is a DEAD-GATED contract candidate, never a zero-source-reference deletion.

Combat deletion order must be:

1. review the newly staged QCombat/QATS dependencies and source tests, and decide the production project enablement;
2. cold compile/UHT;
3. reparent `CombatComponent_C` transactionally and preserve the existing class path;
4. remove duplicate BP state/functions/events only as each consumer becomes typed;
5. move QWeapon, QAI worker snapshots/commits, QPolice, QModule and legacy `ClientRequestDamage` callers;
6. replace real-damage provocation;
7. run life/death exactly-once, owner/non-owner replication, Standalone/Listen/Dedicated two-client, QATS, topology and Message Log gates;
8. only then consider stripping `Lib_Combat` or the wrapper.

## 7. Module/plugin/build dependency audit

| Item | Current evidence | Consequence |
|---|---|---|
| `QANGA.uproject` | NinjaCharacter at line 61, QWeapon at 527, QATS at 531; no QCombat, QInventory, QSystem or Cy_ASpringArm explicit entries | project enablement must be resolved by the integrator, not assumed |
| `QSystem.Build.cs` | NinjaCharacter is now a private dependency at line 24 | static Swimming dependency omission closed; cold compile still required |
| `QSystem.uplugin` | NinjaCharacter is now enabled at lines 34-35 | descriptor updated; runtime/package load still unproved |
| QATS Build.cs | QCombat at 35, QInventory at 46 and QWeapon at 49 | source graph is declared; no build or test result is claimed |
| QATS uplugin | QCombat at 30, QInventory at 74 and QWeapon at 102 | developer-tool plugin graph updated; production enablement is a separate decision |
| `QInventory.uplugin` | content false, `EnabledByDefault=false` | dormant until explicitly enabled/integrated |
| `QCombat.uplugin` | content false, `EnabledByDefault=true`; QGameManager and QWeapon plugin deps | may be discovered by default, but no serialized/runtime owner and no explicit project entry |
| `QCombat.Build.cs` | public QWeapon, private QGameManager | dependency direction is coherent; production consumer graph and runtime gates remain |
| Cy_ASpringArm | no external native Build/project dependency found | serialized QSpring dependency still blocks plugin deletion |
| DataManager | no external native module dependency found | only the empty C++ module is a candidate; content/plugin dependencies remain live |

At the final 23:16 check, another worker had removed `Plugins/QCombat/Intermediate`, added a plugin-local `.gitignore`, and left one ignored `Plugins/QCombat/Binaries/Win64/UnrealEditor-QCombat.dll`. The DLL is timestamped 23:02, while `QCombatComponent.cpp` and `QCombatAutomationTests.cpp` changed through approximately 23:14, so that binary is stale relative to the final inspected source. It is not proof of a clean current build, test pass, Linux/non-unity coverage, or asset integration.

## 8. CoreRedirect and serialized-compatibility audit

Current migration redirects:

- `DefaultEngine.ini:170` QCamera wrapper class to `/Script/QSystem.QCameraControlComponent`;
- `:171-172` gravity `Forced Location` property redirects to `ForcedLocationValue`;
- `:174-175` Blueprint gravity enums to native GravityScape enums.

Do not remove any of them now. Current tracked assets are not the complete serialized universe: the QCamera document explicitly mentions eight excluded maps, gravity has hundreds of map/Blueprint referencers, and packaged/historical content may not be present in source control.

Redirect removal requires:

1. cold load of every tracked and explicitly excluded/historical package boundary;
2. Blueprint compile with no unresolved pins/imports;
3. targeted resave of the authoritative packages, not arbitrary map saves;
4. zero old class/property/enum paths in Asset Registry, FIB, config, seed and package logs;
5. Windows and Linux non-unity;
6. user packaged validation.

## 9. Deletion ledger

### 9.1 Safe on current source evidence, but not changed by this audit

| Candidate | Why source semantics are dead | Minimum remaining gate |
|---|---|---|
| `Plugins/Cy_ASpringArm/.../Cy_ASAc_Detector.cpp` | include-only 38-byte TU, defines no symbol or registration | delete in owning lane, then normal Win/Linux non-unity build |
| QPolice locals at `QPoliceSubsystem.cpp:4537-4545`, `4551-4559` and `4574` | values are never consumed | focused diff review and compile |
| QPolice `InsideSafeZones` block at `4561-4572` | arbitrary component lookup plus wrong property class makes it non-effective | confirm live field class once Editor returns, remove, compile; do not replace with another reflection hack |
| already removed `SM_CylinderCollision` | zero all-category references after owner-product regeneration and full documented gates | preserve deletion; final user package remains the outer release gate |

### 9.2 DEAD-GATED: do not delete yet

| Candidate | Current blocking evidence | Required proof before deletion |
|---|---|---|
| `QCameraControl_Component` wrapper | zero refs, but final SCS query hung; redirect protects historical maps | cold class/redirect load, SCS child/template, BP compile set, eight-map boundary, zero all categories, QATS, PIE/topology, Message Log, Linux/non-unity, package |
| `QSpringArm_Component` wrapper | still AUTH, 340 nodes, 12 refs | native reparent/strip, overrides preserved, exact referencer resave, zero categories/FIB/SCS, QATS/PIE/network, Linux/package |
| `Cy_ASpringArm` plugin | hard serialized dependency from QSpring | wrapper zero first, plugin Asset Registry/FIB/source/build zero, cold BP load, Linux/non-unity, package |
| `SwimmingDetection` wrapper | still AUTH, 11 refs | fix QSystem Ninja deps, reparent/strip, exact 10 referencers plus seed, SCS, QATS/topology/PIE, Linux/package |
| gravity wrappers | 190/219 refs | typed/resaved consumers, exact map boundary, zero all categories, redirects/cold load, QATS/PIE/Linux/package |
| GlobalGravityAreaManager bridge | other manager consumers remain | typed replacement, zero calls/strings/FIB, manager BP compile, gravity QATS/runtime |
| `QAIGravityAreaResolver` | many live callers | typed tri-state world query, migrate every caller, delete cache/reflection/fallback, QAI runtime/topology |
| empty Ninja `SetGravityDirection` override | deletion changes virtual dispatch | base-contract/call hierarchy, non-unity build, gravity/QPlatform/cable runtime |
| `WeaponScriptMacros` and old fire graphs | 27/106 refs and authoritative runtime | production Nash adapters, child compile closure, QATS/Listen/Dedicated, zero refs, seed/resave, package |
| QAI `BPTriggerLatched` | prevents stuck legacy trigger | typed QAI trigger owner, source zero, combat runtime |
| Inventory BP variables/functions/libraries | still authoritative, hundreds of refs | enabled native adapter, single mutation funnel, persistence/transfer rollback, complete consumer migration, BP/network/package gates |
| DataManager C++ library/module | plugin index excludes content; serialized imports unknown | plugin AR/FIB, cold-load all 17 assets, BP compile, no `/Script/DataManager` imports, Win/Linux/package |
| Combat BP state/functions/`Lib_Combat` | 525/62 refs; all production callers still legacy | QCombat integration, typed consumer migration, QATS, exactly-once damage/death, replication topology, zero refs, package |
| QPolice `IQCombatant` | no C++ user, Blueprint users unknown | plugin FIB/AR/SCS/direct-child query, cold BP compile/load, source zero, package |
| QPolice real `TakeDamage(1)` provocation | currently causes aggression behavior | typed event replacement plus behavior parity; only then delete damage path |
| `/Game/VehicleWeapons/Rocket/Missile_VehicleRocketLauncher` | doc 10 reports zero refs and seven nodes, asset is local/ignored | fresh all-category AR, FIB, direct children, cold load/compile, cook/package; do not trust doc-only zero |
| `/Game/_QData/Proto_Quest` | doc 10 reports 77 disconnected nodes and zero refs, asset local/ignored | same fresh all-category/FIB/cold/package proof |
| shotgun `Damage_0` data | unlinked but intended balance unknown | explicit design value and real weapon validation |

## 10. Cleanup missing or under-specified in migration documents

These exact locations are not adequately carried by the existing per-system cleanup lists:

1. `Plugins/QAI/Source/QAI/Private/QAI_GravityAreaResolver.h:236-437` and all listed call sites: duplicate provider, manual gravity math, fallback chain and unpruned cache.
2. `Plugins/NinjaCharacter/Source/NinjaCharacter/Private/NinjaCharacterMovementComponent.cpp:2191-2194`: disabled-but-live virtual override.
3. `Plugins/QAI/Source/QAI/Private/Agent/QAI_AgentComponent.cpp:2941-2960`: broad class/name camera-tick owner that becomes redundant after QCamera/QSpring/Swimming self-gating.
4. QWeapon's source-embedded hard loads of `CombatComponent_C` and `GS_CombatManager_C`: deletion blockers for Combat wrappers.
5. `QWeaponAnimInstance.cpp:1620-1668`: exact WeaponScript reflection to remove after typed fire integration.
6. `QAI_CombatProcessor.cpp:157` and `3579-3639`: legacy trigger cache and release product.
7. `QModule_InventoryBridge.cpp`, DynamicQuestSystem NPC/offline/scanner paths and QAI recruitment: exact Inventory second writers/bridges, not merely generic “consumer integration”.
8. DataManager's empty native library/module versus its live 17-asset content plugin.
9. `QAI_Faction.cpp`, `QWeaponBulletSubsystem.cpp:714-939`, `QAI_AgentComponent.cpp:4700-4886` and `QModule_MedicalDroneActor.cpp:185-238`: exact Combat policy/mirror/writer removals.
10. `QPoliceSubsystem.cpp:4537-4600` and `:7461`: dead locals, impossible safe-zone reflection and real-damage provocation.
11. QPolice `IQCombatant`: no C++ production use, plugin Blueprint use still unproven.
12. `Cy_ASpringArm_Component.cpp` empty tick and `Cy_ASAc_Detector.cpp` dead TU.
13. missing all-category/current direct-referencer names for QSpring and the unnamed eight historical QCamera maps.
14. the newly staged shared QSystem/QATS dependency changes still require central review and cold validation; production QInventory/QCombat enablement remains undecided.
15. QWeapon exposes no public stable-ID/topology/resolver API for the planned QCombat/QAI worker snapshot.
16. `Config/EasyCookGenerated.ini:32966` stale QCamera output versus live zero.

## 11. Stale or conflicting documentation

- `BLUEPRINT_TO_CPP_MIGRATION_PLAN.md` stops at detailed Camera/Spring checkpoints and does not link or reconcile documents 06-10.
- Its graph/reference snapshots mix older and newer totals: VehicleCombat, Inventory, Combat and indexed Blueprint counts have drifted.
- Docs 06, 07, 08 and 09 repeatedly assign build/asset/consumer integration to “Prompt 09”. Root prompt 09 is explicitly a read-only performance/P2 audit and cannot own those changes.
- `09_COMBAT_MIGRATION.md` now matches the existence of `QCombatAutomationTests.cpp`, but neither the tests nor the newly changed shared dependency graph have been executed by this audit.
- `09_COMBAT_MIGRATION.md:308` and `:342` still say Prompt 09 must add QCombat/QWeapon QATS dependencies even though those declarations appeared in the shared worktree during this audit.
- `03_CHARACTER_GRAVITY_MIGRATION.md:3` and `:189` conflict on the Linux J3 gate.
- The Gravity document interleaves pre-implementation future-tense analysis with later integrated state. Current code/assets must win.
- `05_SPRING_ARM_MIGRATION.md:13` says native “becomes” the unique owner, but the asset remains a plain BP SceneComponent. That sentence is a target state, not current evidence.
- `04_CAMERA_CONTROL_MIGRATION.md` reports wrapper zero and SCS native. Reference zero was recaptured, but final SCS confirmation is open because the query hung.
- `EasyCookGenerated.ini` still lists QCamera after the live seed/reference removal; generated output is stale.
- Find-in-Blueprints counts in docs cannot be treated as exhaustive while 2,146 entries are deferred and 570 out of date.

## 12. Other-worker ownership overlap/conflict from root prompts

This section uses only the root prompt contracts, not their results.

| Prompt | Intended lane | Overlap/conflict |
|---|---|---|
| 01 QSpring source | QSpring C++ and focused QATS source | Shares QSystem/QATS modules with 04/08; must not own shared Build.cs/uplugin or assets. |
| 02 Camera/Spring integration audit | read-only asset/SCS/reference evidence | Overlaps 01 only in evidence; 01 owns source, 02 owns no edits. |
| 03 Gravity/locomotion source | GravityScape, Ninja, QAI, gravity QATS | Shares QAI performance evidence with 09 and QATS/shared build graph with other lanes. |
| 04 Swimming source | QSystem Swimming files and focused QATS | Exact source is disjoint from 01/08. Shared QSystem/QATS dependency files changed during the run and require central ownership/review. |
| 05 Weapon orchestration | QWeapon fire-control source and focused QATS | Meets 06 at ammo/Inventory and 07 at damage/Combat; production adapters cannot be owned independently. |
| 06 Inventory core | new QInventory source/test | Leaves QStorage, QModule, DQS, assets and shared build integration to “prompt 09”, which cannot do it. |
| 07 Combat core | new QCombat source/test | Meets 05 through QWeapon provider/damage and 09 through QAI worker snapshots; its new test/dependency files now exist but are unvalidated. |
| 08 Interaction | QSystem interaction source/test | Same module as Camera/Spring/Swimming but disjoint files; shared QSystem/QATS dependency changes remain unowned. |
| 09 Performance/P2 | read-only doc/audit | Cannot legally perform the integration work assigned to it by docs 06-09. |
| 10 Global deletion audit | read-only report | Must not resolve any source/asset conflict itself. |

The main integrator must explicitly own:

- `QANGA.uproject`;
- `QSystem.Build.cs` and `QSystem.uplugin`;
- QATS Build.cs/uplugin;
- shared config/redirects/EasyCook;
- all Blueprint reparent/strip/resave operations;
- cross-domain Weapon-Inventory-Combat adapters;
- builds, tests, Editor, PIE, topology, Message Log and packaged validation.

## 13. Shortest ordered cleanup sequence for the main Codex

1. **Freeze and reconcile the ten source snapshots.** Re-read current status after workers stop. Resolve shared-file ownership once; do not combine stale handoff assertions.
2. **Review and prove the staged build graph before touching assets.** Verify the newly added QSystem/Ninja and QATS QWeapon/QInventory/QCombat declarations, decide explicit production enablement for QInventory/QCombat, and inspect the new Combat tests.
3. **Cold source gate.** Run cold UHT/Editor build plus Windows and Linux non-unity. Run focused pure/lifecycle QATS. Fix source defects before any reparent.
4. **Close the smallest wrapper first: QCamera.** Cold-load redirect and historical packages, recheck ALS SCS/child topology, compile its BP closure, run possession/view-mode runtime, then delete only the zero-reference wrapper. Keep the redirect until packaged history is proven.
5. **Integrate QSpring and Swimming transactionally.** Reparent, preserve authored overrides, strip replaced graphs rather than disable them, compile exact referencers/maps, remove QAI's broad tick-disabler only after typed self-gating, then zero/resave/seed. Delete Cy_ASpringArm last.
6. **Close Gravity J4/J5/J6 current revision.** Cold-load ALS/AILean descendants, run all 15 QATS and QPlatform/cable/player/AI runtime. Add the typed world-position GravityScape query, migrate every QAI resolver caller, then delete the reflected/manual resolver and its cache. Keep wrappers/manager bridge until their own reference sets are zero.
7. **Integrate foundational Combat and Inventory ownership.** Reparent Combat while preserving its path; enable and adapt Inventory through one state/materialization boundary. Remove QAI/QWeapon/QPolice/QModule second writers only as typed replacements go live. These two domains must be authoritative before deleting old WeaponScript ammo/damage orchestration.
8. **Integrate one production Weapon vertical.** Use the NashV2 assault rifle, real Inventory/Combat adapters and QWeapon bullet owner. Prove shot/reload/rollback and network topology, strip only its superseded graphs, then expand to compatible children. Delete the macro only after all 27 refs are gone.
9. **Remove P2/hack debt.** Replace QPolice real-damage provocation, remove dead locals/impossible reflection, resolve IQCombatant, DataManager scaffold, QAI trigger cache and conditional dead asset candidates.
10. **Run the final deletion proof in order.** Fresh FIB, all Asset Registry categories, SCS/direct children, module/plugin/config/redirect/seed searches, authoritative package resaves, Blueprint compile set, QATS, Standalone/Listen/Dedicated topology, clean Message Log, Windows/Linux non-unity, then user packaged validation. Only after a second zero sweep should wrappers/assets/plugins/redirects be deleted.

## 14. Terminal handoff

This audit deliberately leaves work open. It identifies current owners, duplicate writers, compatibility debt, dead-source candidates and the exact gates required to turn candidates into safe deletions. No source or asset deletion was performed.
