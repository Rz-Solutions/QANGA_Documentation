# Prompt 02 - QCamera/QSpring read-only integration audit

STATE: AUDIT_READY_WITH_ONE_SOURCE_GATE

OWNED FILES CHANGED:

- `Documentation/BlueprintToCppMigration/HANDOFFS/02_CAMERA_SPRING_INTEGRATION_AUDIT.md`
- No repository file, config, Build.cs, handoff owned by another worker, or Unreal asset was edited or saved.

EVIDENCE:

## Audit boundary

- Read completely: `agents.md`, `BLUEPRINT_TO_CPP_MIGRATION_PLAN.md`, `04_CAMERA_CONTROL_MIGRATION.md`, and `05_SPRING_ARM_MIGRATION.md`.
- Inspected the current worktree before using those notes. The latest relevant asset commits are `af6fab81527447dde415c00682945d80615d7a1b` (2026-08-28; ALS and QSpring wrapper) and `d2a0fb24a275163e1f915c3a784438e19c8444dc` (2026-08-28; ALS and AI_Cyborg). The native camera/spring sources, their QATS files, and migration documents are worktree-only/untracked.
- Current scoped worktree state: `ALS_Base_CharacterBP.uasset`, `QCameraControl_Component.uasset`, `Config/DefaultEngine.ini`, and `Config/EasyCookGenerated.ini` are modified; `QSpringArm_Component.uasset`, `AI_Cyborg.uasset`, and `IS_DroneBase.uasset` are tracked-clean. These concurrent changes were not touched.
- Final live play-state query before shutdown: stopped, not playing/simulating/paused, zero PIE worlds. The user then closed the Editor after all read-only evidence was collected; no further Editor access is needed for this audit.
- No compile, build, QATS, PIE, map launch/save, asset mutation, cook, package, stage, commit, or EasyCook rescan was performed. Live reflection proves only the currently running Editor state; it is not cold-build proof.

## Confirmed current reflected/native and Blueprint state

| Object | Confirmed current state |
|---|---|
| `/Script/QSystem.QCameraControlComponent` | Loaded reflected `Class` plus `Default__QCameraControlComponent` CDO in `/Script/QSystem`. Source derives from `USceneComponent`; constructor/`OnRegister` impose `bAutoActivate=false`, tick start disabled, `TG_DuringPhysics`, and no dedicated-server tick. Local `PlayerController` ownership gates tick; rotation is `FRotationMatrix::MakeFromXZ(ControlRotation.Vector, ActorUpVector)`. Cold load remains unproved. |
| `/Script/QSystem.QSpringArmComponent` | Loaded reflected `Class` plus CDO in `/Script/QSystem`, with loaded `QSpringArmOutLocationSignature__DelegateSignature`. Source derives from `USceneComponent`, self-activates once at valid BeginPlay, gates to a locally controlled player pawn, and forbids dedicated-server tick. Cold load and behavior parity remain unproved. |
| `/Game/Systems/QSpringArm/QCameraControl_Component` | Parent `QCameraControlComponent`; 0 variables, 0 functions, 0 macros, 0 events, 0 dispatchers, empty 0-node EventGraph. Asset Registry: 0 package referencers and one dependency, `/Script/QSystem`. |
| `/Game/Systems/QSpringArm/QSpringArm_Component` | Parent `SceneComponent`; 41 variables, 9 functions, 1 dispatcher, 2 events, 340 nodes. Asset Registry: 12 referencers and exactly 3 dependencies: `/Script/PhysicsCore`, `/Script/Cy_Trace`, `/Script/Cy_ASpringArm`. |
| `/Game/Systems/Character/Blueprints/CharacterLogic/ALS_Base_CharacterBP` | Parent `NinjaCharacter`; 133 variables, 32 own components, 87 functions, 5 macros, 4 event graphs, 4,424 nodes. Asset Registry: 38 referencers, 187 dependencies; relevant current dependencies include the QSpring wrapper and `/Script/QSystem`. |
| `/Game/AI/Cyborg/AI_Cyborg` | Parent `ALS_Base_CharacterBP_C`; 5 variables, 4 own components, 171 nodes. Asset Registry: 15 referencers, 28 dependencies, including ALS and the QSpring wrapper. |
| `/Game/Items/Drone/IS_DroneBase` | Parent `ItemScriptBase_C`; 10 variables, 8 components, 429 nodes. Asset Registry: 5 referencers, 42 dependencies, including the QSpring wrapper. |

`ALS_Base_CharacterBP_AILean` is a separate `NinjaCharacter` Blueprint at `/Game/AI/Lean/ALS_Base_CharacterBP_AILean`, currently 3,933 nodes and 8 own components; it has no QCamera/QSpring SCS component. It is not a current camera/spring dependency, although the migration document retains it as a parity compile gate.

## Current reference ledger: do not reduce this to one query category

### QCamera wrapper

- Asset Registry package referencers: **0**.
- Asset Registry dependencies: `/Script/QSystem` only.
- Exact current EasyCook seed occurrences: **0**.
- Fresh explicit Find-in-Blueprints on ALS reports two text hits for `QCameraControl_Component`: one getter in EventGraph and one in `OnViewModeChanged`. Both live getter pins resolve to `/Script/QSystem.QCameraControlComponent`; these are SCS variable-name hits, not wrapper-class/package dependencies.
- `EasyCookGenerated.ini:32966` still contains the commented wrapper path. Because the live seed has no QCamera entry and Asset Registry has no package referencer, this is a stale generated snapshot line, not proof of a live seed reference.
- Therefore the wrapper is a deletion candidate, but “global zero” is not claimed from Asset Registry alone. Cold load, historical-package warnings, explicit graph/type checks, seed/config cleanup, QATS, and runtime remain deletion gates.

### QSpring wrapper

Current Asset Registry referencers are exactly **12**:

1. `/Game/EasyCook/DA_EasyCookSeed_QANGA`
2. `/Game/AI/Cyborg/AI_Cyborg`
3. `/Game/Items/Drone/IS_DroneBase`
4. `/Game/Maps/Lobby/_OLD/L_Sub_MainMenu`
5. `/Game/Maps/Lobby/L_Lobby`
6. `/Game/Maps/Lobby/Lobby_Cyborg_Christmas/Lobby_Cyborg_Christmas`
7. `/Game/Maps/Lobby/Lobby_Cyborg_Halloween/Lobby_Cyborg_Halloween`
8. `/Game/Maps/Lobby/Lobby_Cyborg_V1/Lobby_Cyborg_V2`
9. `/Game/Systems/Character/Blueprints/CharacterLogic/ALS_Base_CharacterBP`
10. `/Game/Maps/ScreenshotShowdown/L_Showdown`
11. `/Game/Maps/Universe/_LevelTest/L_Persistent_DEV_2`
12. `/Game/Maps/Universe/_LevelTest/L_Persistent_DEV_3`

All eight map files are ignored by `.gitignore:228` (`/Content/Maps/`). The expected transactional ledger is 12 now, 9 after repairing/saving ALS + AI + Drone, 1 after resaving the eight exact map packages, then 0 after removing the one exact seed entry.

Fresh explicit Blueprint searches and node reflection add the graph category:

- ALS has exactly 8 `QSpringArm_Component` FiB hits: EventGraph bound event plus two getters; one getter each in `BPI_Get_3P_PivotTarget`, `QBuilder_GetCameraInfo`, `Combat Triggers`, `Camera Flip Side`, and `Camera Distance`.
- AI_Cyborg has one component getter feeding an explicit wrapper `Disabled` call.
- Drone has one `Get Component by Class` whose class/default return type is the wrapper, feeding the wrapper `Out_Location` bind.
- The eight maps are serialized package references, not evidence of Blueprint graph calls; the seed is a data entry. This is why no zero-reference conclusion may come from FiB alone.

## QCamera integration state

ALS SCS hierarchy is currently:

`CharacterMesh0 -> QSpringArm_Component (BP wrapper) -> QCameraControl_Component1 (native QCamera) -> CineCamera`

- `QCameraControl_Component1` is directly `/Script/QSystem.QCameraControlComponent`, not the wrapper. It is parented to `QSpringArm_Component`, has child `CineCamera`, `bAutoActivate=false`, and serialized tick text `(TickGroup=TG_DuringPhysics,bCanEverTick=True)`. Native `OnRegister` is intended to reimpose the complete runtime tick/activation contract.
- `CineCamera` remains `/Script/CinematicCamera.CineCameraComponent`, child of QCamera, `bAutoActivate=true`, with serialized `bCanEverTick=True,bAllowTickOnDedicatedServer=True`. This is a reason the broad QAI camera-class suppression cannot simply disappear yet.

The three live authored `SetActive` paths all target the native `QCameraControl_Component1` getter:

1. `OnViewModeChanged.K2Node_CallFunction_0`: Third Person enum output -> `SetActive(false, bReset=false)`.
2. `OnViewModeChanged.K2Node_CallFunction_1`: First Person enum output -> `SetActive(true, bReset=false)`.
3. `EventGraph.K2Node_CallFunction_94`: the QSpring `Out_Location` event, on its Third Person path, links `ToNear` into `bNewActive`; `bReset=false`.

The current migration redirect is correct and must remain:

`Config/DefaultEngine.ini:170`

`+ClassRedirects=(OldName="/Game/Systems/QSpringArm/QCameraControl_Component.QCameraControl_Component_C",NewName="/Script/QSystem.QCameraControlComponent")`

Historical evidence, not current runtime proof: `Saved/Logs/QANGA-backup-2026.08.30-19.07.39.log` recorded “different class; resave to fix” warnings for the same eight maps listed above. Current Asset Registry has no QCamera wrapper referencer, consistent with the documented resave, but only a cold load of those packages can close the warning gate. `QANGA-backup-2026.08.30-20.04.18.log` also contains earlier PIE `Accessed None ... QCameraControl_Component1` errors on `Set Active` and `Set World Location`; they must be explicitly absent in the new cold/runtime validation. The current newly opened log has no such hit, but no relevant runtime was run, so that absence is not a validation result.

QCamera deletion gates still open:

- full Editor build from disk and cold reflection/load of the class and redirect;
- compile QCamera wrapper and ALS at 0 errors / 0 warnings;
- cold-load all eight historical packages without the different-class warning;
- rerun Asset Registry, explicit Blueprint graph/type, live seed, and generated-config checks;
- pass `QATS.QSystem.CameraControl.PossessionLifecycle` from the cold process;
- pass the QCamera portions of `L_Dev_Rz`, including first/third, near activation, unpossess/repossess/vehicle exit, local-only ownership, no duplicate rotation, and no `Accessed None`;
- delete only after those gates; keep the ClassRedirect for ignored/older map copies and cold-load again after deletion.

## Exact QSpring Blueprint contract

### Variables and current CDO defaults

All 41 own variables were found. `Out_Location` is the dispatcher; the other 40 are data/state variables.

- `As Pawn: Pawn = null`
- `OwnerValid: bool = false`
- `Whiskers_H_Length: real = 200`
- `Whiskers_T_Length: real = 50`
- `Whiskers_T_Angle: real = 35`
- `Whiskers_H_Count: int = 8`
- `Whiskers_T_Count: int = 4`
- `Actor to Ingore: Array<Actor> = []`
- `Whiskers_Radius: real = 10`
- `Whiskers_Timer: TimerHandle = ()`
- `Origin: Vector = (0,0,0)`
- `Offset_Target: Vector = (0,0,0)`
- `Offset: Vector = (0,0,0)`
- `Interp_Offset_Speed: real = 8`
- `Length: real = 375`
- `Probe_Radius: real = 18`
- `Out_Location: multicast delegate = ()`
- `Relative_Location: Vector = (0,0,0)`
- `Interp_RLocation_Speed: real = 0`
- `LengthTarget: real = 0`
- `Current_Length: real = 0`
- `Enabled: bool = false`
- `Left_Right: bool = true`
- `CameraOffset: real = 35`
- `CameraOffsetInterp: real = 0`
- `CameraOffsetInterp_Target: real = 0`
- `CameraOffsetInterp_Speed: real = 48`
- `Local_Right_Vector: Vector = (0,0,0)`
- `Last_Start_QSpringArm: Vector = (0,0,0)`
- `SpringArmRightPlacement: Vector = (0,0,0)`
- `bIsCrouched: bool = false`
- `Player_Virtual_Velocity: Vector = (0,0,0)`
- `SpringArmNear: real = 0`
- `Target_Target: Vector = (0,0,0)`
- `Target: Vector = (0,0,0)`
- `Origin_Target: Vector = (0,0,0)`
- `OnScope: bool = false`
- `Interp_Distance: real = 0`
- `SwitchHitProbe_Interp: real = 0`
- `DefaultTargetLength: real = 1900`
- `PivotHeight: real = 0`

The Blueprint and native dispatcher signatures match exactly:

`Out_Location(Location FVector, ToNear bool, NearAlpha double, CameraRotation FRotator)`

Native declaration: `QSpringArmComponent.h:16-22`.

### Nine function graphs and behavior

| Function | Nodes | Current authored behavior |
|---|---:|---|
| `Init` | 25 | Dedicated-server branch does nothing. Otherwise owner -> Pawn; sets `As Pawn`, `OwnerValid=true`, component tick enabled, adds owner to ignore list, resolves `PivotHeight` from character `FP_Camera` socket or 150, then starts timer. Pawn-cast failure sets invalid/tick disabled. |
| `Whiskers_Test` | 31 | Calls Cy ASA horizontal and top whiskers using Query1, complex=false, ignore self, radius; horizontal origin includes 0.05 s velocity lead, top offset is scaled by 4; returns helper hit alphas/locations. |
| `Process_Origin` | 31 | VInterp `Offset` at `Interp_Offset_Speed=8`; VInterp `Target` at literal speed 10; sets `Origin`/`Origin_Target`. Debug spheres are disconnected; one interpolation getter is unused. |
| `Start_Timer` | 3 | Looping `SetTimerByFunctionName(Whiskers_Update, 0.2 s)` stored in `Whiskers_Timer`. |
| `Whiskers_Update` | 47 | Runs whiskers, then three pre-probes; maps summed alphas `[0,2] -> [0,1]` with ExpoInOut; updates camera-offset target, vector offset target, length target, and target target. Contains dead unlinked validity/select nodes. |
| `Update` | 117 | `OwnerValid` gate; Z compensation and origin processing; length interpolation at speed 3; Cy ASA probe Query1 with Query2 overlap recovery; target Query1 radius factor 1.2 whose location output is unused; scope trace only when `OnScope` (1250 length / default target 1900); computes near alpha/rotation and broadcasts. |
| `Disabled` | 5 | `Enabled=false`, clears/invalidates whisker timer, disables component tick. |
| `Process_Z_Compensator` | 48 | Interpolates `SpringArmRightPlacement`, sets relative location, records `Last_Start_QSpringArm`, normalizes/stores `Local_Right_Vector`. One VInterp uses literal speed 8; the other uses its configured interpolation input. |
| `Trace_PreProbe` | 16 | Cy Trace Query1 sphere from offset start/end, radius `Whiskers_Radius`, ignore self/list; maps hit distance from 25%-100% of total length to alpha 1->0. This uses the separate `Cy_Trace` plugin, not `Cy_ASpringArm`. |

EventGraph has exactly 12 nodes: BeginPlay -> `Init`, Tick -> `Update`, plus eight comments. Exact strip GUIDs:

- `K2Node_Event_0` BeginPlay: `E95C9656413214552A1C53ADDBCCAB79`
- `K2Node_Event_1` Tick: `23C8EED94C6C9F42CA0313B04C729BD9`
- `K2Node_CallFunction_0` Init: `06D253494FB5D8A8C0F4B79D2366C0F8`
- `K2Node_CallFunction_1` Update: `640040B24920EA7AC78C77A1A51DB94C`
- comment Whiskers H: `0A1187C4423FE4DB9665B0B853F7D57D`
- comment Offset Origin: `206FD8BC450D428E0493A28606E54729`
- comment Whiskers: `7D1584FA43446D2BE1BC06A9BAA9EB31`
- comment Probe: `2F8DFB094798ECCF5A53F1AC0D73A358`
- comment Probe Origin: `DE26BFD24019122486071D84330435B3`
- comment Whiskers T: `D41BD6D9460DD12619F07A883668CA27`
- comment Z Composator: `3139926D4DE6F7392876A99CBA8E57F4`
- second comment Offset Origin: `E3560E3E4693A8C95A2BBBB68C9170DD`

### ALS SCS, writers, and consumers

`QSpringArm_Component` is still the BP wrapper, parented to `CharacterMesh0`, with relative location approximately `(0,-0.0000009367,187.391887854)`, yaw 90, scale 1, Movable. Critical authored overrides are exactly `Length=210` and `CameraOffset=62`; also `Left_Right=true`, `OnScope=false`, `bIsCrouched=false`, velocity zero, `bAutoActivate=false`. Its serialized tick flags still say `bCanEverTick=true,bStartWithTickEnabled=true,bAllowTickOnDedicatedServer=true`, so native `OnRegister` and cold/runtime proof are mandatory.

External QSpring-owned writer/event nodes that must survive as native bindings:

- EventGraph `Set Actor to Ingore`, GUID `AA3F3DDC4E2C75CE21A7A0BEBFC83652`; value comes from attached actors, then its output feeds array additions.
- EventGraph `Set Left_Right`, GUID `28E837ED4F6C0807EABB2B86C55CA252`; input is `RightShoulder`.
- EventGraph `Set bIsCrouched`, GUID `BF55717E4D9C60C83F2193A951A9E657`.
- EventGraph `Set Player_Virtual_Velocity`, GUID `900B9B004E296E3C4B3132A6DE539291`.
- `Inputs.K2Node_Composite_11` / Camera Flip Side `Set Left_Right`, GUID `9FDB286C415C464C583C4DB8BFC027AB`.
- `Inputs.K2Node_Composite_6` / Combat Triggers `Set OnScope`, GUID `09DBC4674B78E5C45AE36F8FFF9AC7F1`.
- `Inputs.K2Node_Composite_12` / Camera Distance `Set Length`, GUID `96A034FB408F4AF8754B708D85D132CA`; its value is the just-set `ThirdPersonArmLength` and the current pin is `real/float` while native `Length` is `double`. The link/value conversion must be explicitly verified after refresh.
- EventGraph component-bound `Out_Location`, GUID `D106E3F24CCE4197A1B6FDAD85655DDE`; its owner is currently the wrapper class. `Location` drives camera/world locations, `ToNear` drives a branch and QCamera `SetActive`, `NearAlpha` drives a lerp, and `CameraRotation` drives CineCamera world rotation.

Standard component uses also remain: `BPI_Get_3P_PivotTarget` calls `SetRelativeLocation` on QSpring; `QBuilder_GetCameraInfo` reads its world location. They do not access native-private spring state.

`InterpCameraArmLength` is not a spring-length writer: its only live track output is `Alpha -> 3rdPersonCameraAlpha`. However the documentation is incomplete because the separate `Camera Distance` graph directly writes QSpring `Length`. Keep the timeline/name-based AI suppression and retarget the separate `Length` setter.

### Drone and AI

- Drone EventGraph `K2Node_CallFunction_75`, GUID `8B1A881F47A3DAB9DB3278A75B820246`, is `Get Component by Class` with class/default return type the QSpring wrapper. Its return feeds IsValid and `K2Node_AddDelegate_0`, GUID `C9347BFF4A0367B85EDD0DB2BA97C6C5`, whose `DelegateReference` is wrapper `Out_Location`.
- Drone custom event `Ligth`, GUID `42DD023247BD6FB134AA3993C107A8EA`, has the exact four dispatcher outputs. It consumes `Location` for scene/actor placement and look-at, and `CameraRotation` for world rotation; `ToNear` and `NearAlpha` are currently unused. Retype/reconstruct it; do not delete it.
- AI_Cyborg `K2Node_CallFunction_13`, GUID `87B4A1494C3E394355874781DDF84419`, calls wrapper function `Disabled` after IsValid. The target getter GUID is `AC400EF843909B7904845EB221AF789B`; IsValid GUID is `79D865CF4D0D57EA3F8BA8B1F210AC80`. The obsolete red comment GUID is `3A65AC644A256C8A5B50F5BAD363EAC2`. Native local-player gating makes this whole terminal branch redundant; remove all four nodes, not only the call.

## Confirmed source/integration defect owned by prompt 01

Hard gate before QSpring reparent/SCS replacement:

- Blueprint reflected contract and active ALS setter name: `Actor to Ingore`.
- Atomic reparent reference rewriting uses `FCppTypeMapper::SanitizeCppId`, which replaces every non-alphanumeric character with `_` (`Plugins/RzDirectMCP/Source/RzDirectMCP/Private/Codegen/CppTypeMapper.cpp:87-94`; use in `PendingReparentStore.cpp:646-659`). The expected inherited native property is therefore `Actor_to_Ingore`.
- Current native source instead declares `ActorToIngore` at `Plugins/QSystem/Source/QSystem/Public/Component/QSpringArmComponent.h:52-53`, used at `.cpp:164` and `.cpp:294`.
- Result: the existing ALS setter cannot automatically rebind to the current native property; the reparent tool would also emit redirects to nonexistent `Actor_to_Ingore` targets. `bp_refresh_all_nodes` cannot prove a contract that does not exist.

Prompt 01 must resolve this before mutation. The cleanest migration-compatible fix is a reflected native internal identifier `Actor_to_Ingore` (display metadata may remain human-readable) and updated C++ uses. The alternative is an explicit ALS node reconstruction to `ActorToIngore` plus hand-authored valid redirects and removal of the tool-generated invalid redirects. Do not keep an alias/fallback.

Separate tooling/operation trap: stripping obsolete `As Pawn` also makes the current reparent tool emit `As Pawn -> As_Pawn` redirects even though no native reflected `As_Pawn` exists. Those redirects would be stale and must never reach the next cold load. Either delete that dead internal variable through a non-redirecting cleanup path or remove the generated pair immediately after the successful transaction. No other confirmed QSpring source defect was found statically; collision/whisker parity remains untested.

## QAI name-based tick-disable cleanup

Current narrow block is `Plugins/QAI/Source/QAI/Private/Agent/QAI_AgentComponent.cpp:2948-2964`, with the condition at 2956-2959:

- `CompClass.Contains("SpringArm")`: completely redundant after QSpring native local-player/dedicated gating; remove this broad migration clause.
- `CompClass.Contains("Camera")`: its QCameraControl coverage becomes redundant, but the clause cannot be deleted wholesale because ALS still owns a tick-capable `CineCamera`. Narrow it to the actual remaining camera type/owner rather than retaining QCamera as a reason.
- `Comp->GetName().Contains("Camera")`: QCameraControl name coverage becomes redundant, but this must remain/narrow to exact `InterpCameraArmLength` because that player camera/view timeline is unrelated and still present.
- `CompClass.Contains("DynamicFlight")`: unrelated player-flight suppression; retain.
- `CompClass.Contains("QPlatform_2_Player")`: unrelated player-platform suppression; retain.

Also retain the separate latent-action cleanup at `QAI_AgentComponent.cpp:3139-3154` for `QPlatform_2_Player` and `DynamicFlight`, plus unrelated vehicle-movement clauses later in the file. The comment at 2950-2952 saying the timeline directly drives boom length is stale: live graph evidence shows it drives `3rdPersonCameraAlpha`; the separate Camera Distance graph writes spring `Length`.

## Every current `Cy_ASpringArm` dependency

Asset/graph dependency is concentrated in the QSpring wrapper:

- `/Script/Cy_ASpringArm` has exactly one Asset Registry referencer: `/Game/Systems/QSpringArm/QSpringArm_Component`.
- `Whiskers_Test`: `Cy ASA Whiskers Horizontal`, `Cy ASA Whiskers Top`, and one `Make Cy ASA Trace Sphere` (`FCyASA_TraceSphere`).
- `Update`: `Cy ASA Probe Tracing` and one `Make Cy ASA Trace Sphere`.
- No other QSpring function/EventGraph contains a `Cy ASA` node.
- `Cy Trace Sphere/Line Single by Channel` and `Cy Trace Ignore/Debug` nodes are from the separate `/Script/Cy_Trace` dependency. `Cy_Trace` has other production consumers and must remain.

Current C++/metadata evidence:

- Repository text search over `.h/.cpp/.cs/.uplugin/.uproject/.ini/.json`, excluding the plugin itself, found zero `Cy_ASpringArm`, `CyASA_`, `FCyASA_TraceSphere`, or `UCy_ASpringArm_Component` consumers.
- `QSystem.Build.cs`, `QSystem.uplugin`, and `QANGA.uproject` do not name the plugin.
- Live plugin metadata reports no declared dependencies and no dependent plugins.
- Descriptor is Runtime/Default, `CanContainContent=true`, but there is no project-descriptor entry to remove.
- Current plugin directory has 42 files, 11 tracked: descriptor, icon, Build.cs, four private sources, and four public headers. Binaries/Intermediate account for the other generated files.

Deletion conditions for `Plugins/Cy_ASpringArm`:

1. QSpring wrapper functions/struct nodes removed and wrapper compiled cleanly on native parent.
2. `/Script/Cy_ASpringArm` Asset Registry referencers = 0.
3. Paged/explicit Blueprint search = 0 for `Cy ASA`, `CyASA_`, `FCyASA_TraceSphere`, and `UCy_ASpringArm_Component`, with no index problems.
4. Source/build/plugin/project/config text search = 0 outside the plugin.
5. QSpring wrapper itself deleted only after its own multi-category zero-reference gate.
6. Cold QATS and `L_Dev_Rz` parity pass while the plugin still exists but has zero consumers.
7. Close Editor, remove the entire plugin directory (tracked source plus generated Binaries/Intermediate), perform another full Editor build, cold launch, and repeat reflection/load/QATS/runtime/log checks. Do not delete `Cy_Trace`.

## Redirect and EasyCook state

- Current migration-specific redirect: QCamera ClassRedirect at `DefaultEngine.ini:170`; valid and required.
- Current QSpring ClassRedirect: absent. Add it before wrapper deletion, after all live consumers are native:

  `+ClassRedirects=(OldName="/Game/Systems/QSpringArm/QSpringArm_Component.QSpringArm_Component_C",NewName="/Script/QSystem.QSpringArmComponent")`

- No current QSpring property redirect exists. Do not retain auto-generated `As Pawn -> As_Pawn`; retain an `Actor to Ingore` redirect only if its target is a real reflected native property.
- EasyCook seed current metadata: last scanned `2026.08.30-17.16.53`, plugin `1.1.2`, generated path `Config/EasyCookGenerated.ini`.
- Exact live seed entries: QCamera wrapper 0; QSpring wrapper 1, with `Source=CdoDefault` and `OriginContext="Dependency of /Game/Items/Drone/IS_DroneBase"`.
- `EasyCookGenerated.ini:32966-32967` lists both wrappers as comments. QCamera is stale now; QSpring is current now and becomes stale after Drone repair/seed removal. Do not rescan. Remove the QSpring seed entry through `RemoveDataAssetArrayEntries` with `array_property=Entries`, `match_field=AssetPath`, exact wrapper object path, and `expected_count=1`; then reconcile the two generated comment lines through the authorized config owner.

## Current versus stale/incomplete documentation

- Current: QCamera wrapper 0 nodes; any earlier “3 dead wrapper nodes” count is stale.
- Current: QSpring remains exactly 340 nodes / 41 variables / 9 functions; those document counts are still correct.
- Current: ALS is 4,424 nodes, not the 4,454 baseline still printed in the main migration plan/character document. AILean is 3,933, not 4,251. These are stale size baselines, not integration contracts.
- Camera documentation combines an earlier green lifecycle QATS/Live Coding checkpoint with unchecked recompile/cold-load gates for later `OnRegister` hardening. Treat only the unchecked cold gate as current; this audit did not rerun the historical tests.
- The documented QCamera Asset Registry/seed zero is still consistent with current live checks, but the generated INI still has its stale comment and the eight historical copies still require cold warning verification.
- The spring document’s statement that `InterpCameraArmLength` does not own spring length is narrowly true, but incomplete: Camera Distance directly writes QSpring `Length` and must be retargeted.

INTEGRATION REQUESTS:

## Ordered transactional operation plan for the main Codex

1. **Resolve ownership and cold prerequisites.** Wait for prompt 01 to resolve the `Actor to Ingore` reflected-name gate. Coordinate the currently modified ALS/config/QCamera packages with their owners. Ensure PIE stopped and every target package is clean in the Editor; never use broad “save dirty packages”. Close Editor, run:

   `"E:\UE573\Engine\Build\BatchFiles\Build.bat" QangaEditor Win64 Development -Project="G:\QANGA\QANGA.uproject" -WaitMutex`

   Launch only with:

   `"E:\UE573\Engine\Binaries\Win64\UnrealEditor.exe" "G:\QANGA\QANGA.uproject" -AutoDeclinePackageRecovery`

   From this cold process, recheck `/Script/QSystem` class/CDO/delegate reflection, source version, stopped play state, and all seven required assets. Do not use Live Coding as the cold gate.

2. **QCamera proof, then deletion.** Load/compile the empty QCamera wrapper and ALS at 0/0; recheck the two named getters are native typed and the three `SetActive` paths are intact. Cold-load the eight historical packages and require no different-class warning. Run the camera QATS and focused `L_Dev_Rz` scenarios below. Requery Asset Registry + explicit FiB/type + live seed + generated config. Remove the stale generated QCamera line through the config owner. Only then delete the wrapper with `delete_asset`, keep the current ClassRedirect, cold reload ALS and historical packages, and requery all categories.

3. **QSpring wrapper atomic reparent/strip.** On a clean package, call `reparent_blueprint` to `/Script/QSystem.QSpringArmComponent`, `save=true`, with:

   - `strip_variables` (40): `As Pawn`, `OwnerValid`, `Whiskers_H_Length`, `Whiskers_T_Length`, `Whiskers_T_Angle`, `Whiskers_H_Count`, `Whiskers_T_Count`, `Actor to Ingore`, `Whiskers_Radius`, `Whiskers_Timer`, `Origin`, `Offset_Target`, `Offset`, `Interp_Offset_Speed`, `Length`, `Probe_Radius`, `Relative_Location`, `Interp_RLocation_Speed`, `LengthTarget`, `Current_Length`, `Enabled`, `Left_Right`, `CameraOffset`, `CameraOffsetInterp`, `CameraOffsetInterp_Target`, `CameraOffsetInterp_Speed`, `Local_Right_Vector`, `Last_Start_QSpringArm`, `SpringArmRightPlacement`, `bIsCrouched`, `Player_Virtual_Velocity`, `SpringArmNear`, `Target_Target`, `Target`, `Origin_Target`, `OnScope`, `Interp_Distance`, `SwitchHitProbe_Interp`, `DefaultTargetLength`, `PivotHeight`.
   - `strip_dispatchers`: `Out_Location`.
   - `delete_function_graphs`: `Init`, `Whiskers_Test`, `Process_Origin`, `Start_Timer`, `Whiskers_Update`, `Update`, `Disabled`, `Process_Z_Compensator`, `Trace_PreProbe`.
   - `strip_event_node_guids`: the exact 12 GUIDs recorded above.
   - `strip_function_graphs`: empty. These nine functions are obsolete, not native UFUNCTION migrations; do not generate FunctionRedirects.

   Require tool success, save success, rollback not needed, parent native, 0 variables/functions/dispatchers/nodes, compile 0/0. Before any restart, inspect generated redirects: remove the invalid `As Pawn` pair and require the Actor target to exist exactly. If prompt 01 keeps `ActorToIngore`, do not use the payload unchanged; explicitly rebuild the ALS setter and replace invalid generated redirects with valid `ActorToIngore` redirects.

4. **ALS repair and SCS reclass.** Rebind/reconstruct the seven setter nodes and the bound event listed above against native QSpring; verify the `Length` source link survives float->double reconstruction. Use `blueprint_replace_component_class` on own component `QSpringArm_Component` to `/Script/QSystem.QSpringArmComponent`. Require `dropped_property_count=0`, `incompatible_property_count=0`, compile 0/0, save success, same variable GUID/name, same parent/child hierarchy, and retained `Length=210`, `CameraOffset=62`, transform, and QCamera child. Run `bp_refresh_all_nodes`; re-inspect every listed GUID/path and the two standard component consumers. The bound event owner and all writer `MemberParent` values must now be native.

5. **Drone and AI repair.** In Drone, change the exact `Get Component by Class` class to `/Script/QSystem.QSpringArmComponent`, reconstruct the `Out_Location` bind to the native delegate, and prove the `Ligth` event’s four pins and all current links are unchanged. In AI_Cyborg, remove the `Disabled` call, getter, IsValid macro, and obsolete comment by their four GUIDs; the branch is terminal, so do not invent a replacement call.

6. **Blueprint compile set.** Refresh/compile/save only the target packages and require 0 errors / 0 warnings: native-parent QSpring wrapper, ALS, AI_Cyborg, IS_DroneBase, and the document-required AILean parity compile. Recompile QCamera wrapper only if it has not already been deleted. Reinspect SCS/defaults/bindings after compilation, not only before.

7. **Historical package boundary.** Without opening/saving the current level, call targeted `save_asset`/package resave one at a time for the eight exact map packages. After each save, requery QSpring referencers and require that exact map to disappear; never save all dirty packages or save `L_Dev_Rz` merely because loading made it dirty. The count should go 9 -> 1.

8. **Seed and zero-reference gates.** After Drone no longer depends on the wrapper, atomically remove the one exact seed entry with `expected_count=1`, without rescan; require referencers 1 -> 0. Add the QSpring ClassRedirect shown above. Run a multi-category zero gate: wrapper Asset Registry zero, `/Script/Cy_ASpringArm` zero, prior-consumer explicit FiB/type zero, paged project search for wrapper/Cy ASA symbols with no index problems, live seed zero, generated config reconciled, and source/build/plugin metadata zero.

9. **Runtime/QATS before destructive cleanup.** Run both exact QATS filters and the full `L_Dev_Rz` matrix below while the empty QSpring wrapper and unused plugin still exist. This isolates behavior proof from deletion. Require no old import warning, missing class/function/property, `Accessed None`, duplicate camera owner/rotation, or Message Log warning/error.

10. **Delete wrapper, then plugin.** Delete QSpring wrapper only after step 9 and keep its ClassRedirect. Cold restart and load the historical/live consumers again. Requery all categories. Only then close Editor and remove the whole `Plugins/Cy_ASpringArm` directory. Repeat the full Editor build and cold launch, then repeat reflection, package-load, QATS, `L_Dev_Rz`, and log gates. Any nonzero reference or missing-class warning aborts deletion; do not hide it with another redirect/fallback.

11. **QAI cleanup after native proof.** Remove/narrow only the redundant SpringArm and QCamera portions described above. Retain/narrow CineCamera, exact timeline, DynamicFlight, QPlatform, latent cleanup, and unrelated vehicle clauses. Compile QAI in the next authorized source/build lane and repeat the AI/dedicated no-tick scenario.

TESTS FOR INTEGRATOR:

## Exact QATS

Run with `manage_automation_tests` from the cold-built Editor and collect final results:

- `QATS.QSystem.CameraControl.PossessionLifecycle`
- `QATS.QSystem.SpringArm.PossessionLifecycle`

The camera test covers serialized-default hardening, inactive/possess/unpossess/repossess, AI exclusion, and exact rotation. The spring test covers lifecycle/local possession/AI exclusion/config/first-frame interpolation. Neither spring QATS proves collision, whisker, overlap-exit, or visual parity; those remain runtime scenarios.

## Exact `L_Dev_Rz` runtime matrix

- Third person at rest and moving; crouch/uncrouch; shoulder flip; Camera Distance input changes `Length` with retained 210/62 authored baseline.
- Near obstacle, wall behind player, ceiling, direct overlap/recovery, and exit from overlap without pop or stuck offset.
- Scope on/off and first-person <-> third-person transitions; verify all three QCamera `SetActive` paths and QSpring `ToNear` behavior.
- Vehicle enter/exit and player unpossess/repossess; desired activation resumes with one camera-rotation owner only.
- Drone control/view: native component lookup, one delegate binding, location consumers, look-at, and rotation consumer all update.
- AI_Cyborg plus an unpossessed pawn, remotely controlled/distant pawn, and dedicated-server context: QCamera/QSpring ticks stay disabled without the deleted AI `Disabled` call. CineCamera/timeline/flight/platform suppression must remain correct.
- Stop PIE and require Message Log 0 errors / 0 warnings. Search the new cold/runtime log for `QCameraControl_Component`, `QSpringArm_Component`, `Cy_ASpringArm`, `Resolved import with ... different class`, `Accessed None`, `Could not find`, and missing property/function/class warnings.

REMAINING:

- Prompt 01 must resolve the confirmed `Actor to Ingore` / `ActorToIngore` reflected-name defect before QSpring mutation.
- Cold build/load, Blueprint compilation, QATS, PIE/runtime, historical resaves, seed removal, redirects, wrapper deletion, plugin deletion, QAI cleanup, and all post-deletion validation are intentionally unexecuted.
- QCamera has strong multi-category static deletion evidence but still lacks cold/runtime proof. QSpring has 12 live referencers and is not a deletion candidate yet.
- The current worktree/config is concurrently dirty; the main Codex must coordinate ownership rather than saving or overwriting those changes.
