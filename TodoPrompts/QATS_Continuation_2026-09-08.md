# QATS continuation prompt — 2026-09-08

## Request to resume

Continue the QATS infrastructure cleanup from checkpoint `527356918` in `G:\QANGA`.
Read the current `AGENTS.md` first, inspect the current worktree, and verify the specific remaining contracts below before editing. This document is a checkpoint, not an immutable design. Do not repeat the full infrastructure audit or rewrite QATS wholesale.

The user explicitly stopped the previous implementation at a compiling checkpoint. The original objective/executor and QATS navigation cleanup is **not finished**. The commit summary describes implemented changes, not full runtime acceptance.

Work in small, coherent milestones: finish the outstanding harness corrections, then integrate the production objective/navigation contracts, then address reproducibility and demonstrated hot-path overhead. Report completion and remaining work at each milestone instead of opening all lanes at once.

## Working constraints

- Preserve unrelated dirty work. At handoff, other work remained in DataManager, QInventory, QCombat, QWeapon, QAI Faction, player interaction, several RzDirectMCP files, migration documents, and data assets. Recheck live status; do not restore or stage these wholesale.
- Follow the current native-subagent ownership policy. A bounded builder owns non-trivial implementation; the main agent alone owns builds, editor commands, automation, PIE and integration. Do not assign simultaneous owners to `QuestTestSubsystem.cpp/.h`.
- Use RzDirectMCP semantic operations for assets. Never inspect `.uasset` bytes as graph evidence.
- RzMCP was authorized during the previous work; avoid PIE unless the specific remaining proof requires it. No cook, stage-build or package execution: the user handles those.
- Do not add fallback navigation, synthetic quest completion, alternate privileged spawn paths or extra timeout layers. QAI owns legal traversal; production quest logic owns completion; the existing admin command owns spawning.
- Do not restart the editor or save unrelated dirty assets casually. In the previous session, TempDB had intentional-looking unrelated changes; automatic map dirtiness is not permission to save maps.
- Do not commit unless asked. The previous checkpoint was staged by the agent and subsequently appeared in commit `527356918`.

## Implemented checkpoint — keep, do not redo blindly

### QATS lifecycle and result infrastructure

Primary files under `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/`:

- `Private/QuestTestSubsystem.cpp`, `Public/QuestTestSubsystem.h`
- `Private/QATSArtifactWriter.cpp/.h`
- `Private/QATSRunAll.cpp`, `Private/QATSTestInventory.h`
- `Private/QAutomatedTestSuiteModule.cpp`, `Private/QModuleTestRunner.cpp`
- `Private/OfflineTutorialHandoffSmoke.cpp`, `Private/OfflineTutorialProgressTests.cpp`

Implemented: checked atomic result writing, explicit terminal outcomes and child exit failures, declared campaign coverage, real tutorial handoff/persistence campaigns, and a scoped fixture release path. Restoration includes save suppression, quest-level override, admin state, InfiniteAmmo, original equipment, input ownership and neural vehicle speeds. Combat setup is capability-driven; dedicated-server capability discovery uses authoritative active objectives rather than local driver state.

The fixture rejects pre-existing foreign prepared prerequisites instead of clearing them. InfiniteAmmo capture reads existing parameter maps without creating a baseline; unsupported missing/uninitialized/non-Bool baselines fail explicitly. Asynchronous equipment restoration waits for the original loadout to settle. PIE run IDs include a GUID and competing single-process PIE runs are rejected before shared mutations.

These are source changes, not evidence that all cleanup paths have passed runtime validation. The stop API exists, but its console caller is still missing; see the next milestone.

### Production quest contracts

Under `Plugins/DynamicQuestSystem/Source/`:

- `UPlayerQuestComponent::GetActiveObjectiveIDsView`: short-lived const view; do not retain across mutations.
- `UPlayerQuestComponent::GetActiveDialogueWidget`: authoritative dialogue owner lookup.
- `ULocationObjective::GetCurrentLocationRequirement`: distinguishes enter, exit, guardian wait, satisfied and invalid; respects production visited/order state.
- `UScannerObjective::GetLocationRequirement`: production location and allowed distance.
- `UKillObjective::DoesKillEventMeetRequirements`: production kill qualification with failure reasons. Quest-spawn-only kills now require both the correct quest and objective identity.
- Quest-level override ownership snapshots the authoritative baseline, retains the acquired player identity, and releases idempotently on explicit restoration or component EndPlay. Do not replace this with a client-side approximation.
- `UOfflineTutorialProgressSubsystem::WasTransferDurablyCommitted`: evidence recorded after the actual durable write, not a transient UI/state observation.

Most new objective queries are **not yet consumed by the QATS executors**.

### QAI producer contracts

Under `Plugins/QAI/Source/QAI/`, the checkpoint modifies pathfinding, its native callers, and client navigation reporting; the dirty Faction files belong to other work.

Implemented: per-acquisition requester leases, connected-route field sharing, and `GetValidatedNavigationSnapshot(Handle, WorldLocation)` for traversal consumers. Raw retained snapshots are not unconditional steering authority. `ESampleReason::RouteInvalidated` is appended as value 9, preserving earlier numeric values.

Geometry changes are admitted against both mutable rebuild bounds and the immutable retained publication, with their respective padding. Unchanged corridors may remain usable. Authored geometry uses a spatial index with deadline-sliced bucket/member/oversized-component iteration; ready HISM uses its indexed overlap query, while plain or not-yet-ready HISM collection remains deadline-sliced.

Known-invalid client steering is revoked through the existing report batch: zero direction with `bHasDir` clears server direction and expiry. No new network report layout was introduced for that final change.

Independent static re-review found no remaining high-confidence defect in that producer lane. Spatial-bucket and ready/plain/outdated ISM/HISM paths still lack focused automated coverage. **QATS raw snapshot consumers and its independent recovery routing are still outstanding.**

### Correlated production admin spawning

Native ownership lives in the four new QSystem files for `QCommandExecutionComponent` and `QCommandExecutionScript`. Existing assets were semantically modified and saved:

- `/Game/Systems/Commands/CommandComponent`
- `/Game/Systems/Commands/CommandScriptBase`
- `/Game/Commands/SpawnClass_Command`

The first two now inherit the corresponding native component/script. `SV_ExecuteChatCommand` remains a Server RPC with its original `CommandLine` plus an optional string `RequestToken`.

One component owns one outstanding token and dispatch receipt; the exact attached script owns execution completion. Relevant native operations are `BeginExecutionRequest`, `CanDispatchExecutionRequest`, `AttachExecutionScript`, `PublishExecutionResult` and `AcknowledgeExecutionRequest`.

Preserve these invariants:

- Empty-token callers retain the existing command behavior.
- Token-bearing requests with missing permissions fail explicitly; they do not enter the legacy latent first-admin bootstrap. Later RPC calls can overwrite Blueprint event-frame parameters, so do not reintroduce token bookkeeping across that latent bootstrap.
- Attachment must succeed before executing a token-bearing script. Async continuations must still belong to a pending request before spawning.
- Success retains the exact spawned actor, after admin marking and vehicle bookkeeping; it is not inferred from nearby actors.
- The existing 30-second producer timeout reports token failure; terminal receipts remain until acknowledgement. Do not add a second consumer timeout as a substitute for this contract.
- Acknowledgement clears ownership before destroying the retained script. Explicit premature script destruction produces a durable failure receipt via `OnDestroyed`.

The asset `Content/Commands/SpawnClass_Command.uasset` was explicitly added despite the existing ignored parent directory; it is now part of the checkpoint commit. Do not assume ignored-folder discovery means it is untracked.

**QATS has not yet integrated this receipt contract.** Its old local-controller submission restriction and youngest-matching-pawn search still exist.

### Neural initialization and tooling

The RzNeuralNetwork C++/Python changes preflight durable lineage before shared writes, use a nonblocking process-lifetime OS writer lock, and require session-correlated manifest publication with matching schema/protocol/generation/digest fields. Session-unique transport files prevent a new editor from truncating a surviving writer's stream.

Python is the single shared curriculum/checkpoint writer. Curriculum input is checksummed and sequenced against accepted transition boundaries; shutdown drains both streams before durable acknowledgement. Preserve deliberate zero-step fresh restart behavior; do not broaden fresh-run rejection without evidence. Do not modify trained snapshots or launch training merely to inspect this code.

RzDirectMCP now returns an error when an automation selection matches nothing, instead of a false success. Its change is in `MCPEditorUtilityLibrary.cpp`; other dirty RzDirectMCP changes were unrelated.

## Next milestone — close these known harness gaps first

The final five review corrections were **not applied** before the user stopped the session:

1. `QATSRunAll.cpp`: parent runtime/tutorial deadlines can expire before valid cumulative child stages. Derive parent budgets from the actual existing child contracts; do not add arbitrary safety timeouts.
2. `QATSTestInventory.h`: reconcile explicit context inventories with current source. At checkpoint, editor exclusions omitted `QATS.DataManager.SlotWriter.FifoAcrossAsyncAndDurable`, `QATS.DataManager.SlotWriter.PairRollback`, `QATS.DataManager.SlotWriter.TeardownJoins`, and `QATS.DynamicQuestSystem.KillObjective.QuestSpawnIdentity`. The three SlotWriter additions belonged to concurrent dirty work, so establish their current committed state first.
3. `QAutomatedTestSuiteModule.cpp`: a higher-priority `t.IdleWhenNotForeground` value can hide a newly stored `SetByCode=0` override. Restoration must remove the run's stored override even if it never became effective, while preserving prior history/value/priority. Verify the engine CVar API rather than restoring only the effective integer.
4. The packaged relaunch appends `RzHoverDrive`; an earlier occurrence wins because `FParse::Value` selects the first match. Replace/canonicalize an existing argument instead of appending another.
5. Wire `quest.test.stop` to the existing public `StopRun`. At checkpoint the cancellation API had no console caller. Preserve asynchronous fixture cleanup rather than disabling the subsystem immediately.

Additional inventory integration trap: the committed inventory includes `QATS.QInventory.Transfer.PackedBagReindex`, but its test definition was still in the other task's unstaged `QInventoryAutomationTests.cpp` changes when this document was created. Reconcile this dependency before claiming the checkpoint independently covers its declared inventory. Do not stage unrelated implementation files to hide a mismatch.

Old counts (117 packaged entries, 104 editor exclusions, 22 runtime contracts) were a snapshot, not authoritative current counts. The five declared campaigns are Automation, Runtime, TutorialHandoff, TutorialPersistence and Quest. Missing declared coverage must not become green, and editor-only tests must not be silently represented as executed in a packaged process.

## Following milestone — production objective semantics and QATS navigation

Keep one implementation owner for `QuestTestSubsystem.cpp/.h`. Inspect the new production APIs before changing the executors.

- Location: being inside is not always Passive. Complete-on-exit requires leaving after entry; multi-location selection must use actual visited/order/guardian state, not `Progress % NumLocations`.
- Scanner: navigate to its required location before issuing scanner input when a location gate exists.
- Dialogue: closing a popup without objective progress must not permanently latch the executor into waiting. Re-resolve the production dialogue/quest state and respect rejection conditions.
- Kill: honor actual target/weapon/type/critical/stealth requirements; do not always melee a bound quest enemy. Explicitly classify unsupported configurations instead of inventing completion events. A previously observed temporal multi-kill timestamp-seeding issue was not repaired; do not claim timed kills supported without proving the production contract.
- Interaction: preserve the accepting component's anchor rather than replacing it with the owning actor's origin.
- Fast quest completion: reassess the early Completed path that can bypass the presentation assertions being claimed. Separate actual quest completion from required presentation evidence.
- Client/server verdict: observer setup is improved, but a combined network outcome still needs explicit ownership; standalone success is not a network verdict.
- Spawn consumption: generate a unique token, invoke the existing Server RPC on authority where appropriate, consume its exact native receipt, validate class/owner/admin marker, adopt the actor, then acknowledge. Authority can invoke a Server RPC directly; the helper's local-player restriction is artificial. Delete `FindProductionAdminTravelVehicle` and its youngest-existing-pawn heuristic once replaced.
- Steering: migrate QATS foot/jetpack/vehicle traversal consumers to validated current-route views and handle `RouteInvalidated`. Preserve useful unchanged corridors, not unconditional old snapshot steering or whole-field stalls.
- Delete the independent nearest-integrated-cell direct recovery. A walkable target cell does not prove the intervening route; do not replace QAI rejection with straight-line steering.
- Remove the disabled guide-bridge consumer's producers too: approach/landing/pursuit goal overrides, candidate calculations, storage, scheduling, states, helpers and tests that exist only for that disabled behavior. Do not merely leave a hard-coded false consumer.

Useful search anchors: `ResolveExecutor`, `ChooseLocationGoal`, `TickAuthoritativeTravelVehicle`, `SubmitProductionTravelVehicleSpawnCommand`, `FindProductionAdminTravelVehicle`, `FindNearestIntegratedCell`, `bGuideBridgeCrossingActive`, `DriveToward`, `GetFieldSnapshot`.

## Final planned milestone — reproducibility and targeted overhead

- Replace location seed mixing through `GetTypeHash(CurrentQuestID/CurrentObjectiveID)` with stable canonical identifier hashing. FName pool identities vary across processes; preserve FName's intended case semantics.
- Debug must be observational. `TravelVehicleDetailedFieldPublicationPlanningSeconds` is 3 seconds while the debug alternative is 10, selected by `Manager->bDrawDebug`; remove that behavior change and adjust affected tests.
- Use the short-lived active-objective view instead of cloning and rebuilding full quest instances every driver tick.
- Replace repeated global widget scans with correctly scoped ownership/lifecycle tracking where evidence supports it. Use the production active-dialogue getter. Check player/world, ancestor visibility and switcher ownership; viewport widget events do not automatically cover all nested widgets.
- Keep readiness/capability setup out of the hot path once settled; invalidate it only on relevant ownership/equipment changes. Keep necessary Blueprint reflection in validated bridge helpers rather than creating shadow native properties.
- The temporary notification-manager accessor considered for this phase was removed because it had no consumer. Add only what an actual scoped consumer needs.
- No FPS/millisecond improvement was measured. Profile a relevant state before making performance claims or expanding this work.

## Build and validation evidence — preserve the boundaries

The user compiled and restarted during the implementation. Later RzMCP Live Coding compiled the C++ but failed to link a newly added **concurrent inventory** export, `FQInventoryTransaction::PreflightMoveWholeRecord`, in QInventoryIntegration and the unrelated inventory automation changes. Do not patch the QATS feature around that unrelated linking boundary.

The user explicitly declined closing the editor and requested a `-LiveCoding` command-line compilation check. This exact command returned exit 0 / `Result: Succeeded` (43 compile actions, 36.55 seconds total):

```powershell
& "E:\UE573\Engine\Build\BatchFiles\Build.bat" QangaEditor Win64 Development -Project="G:\QANGA\QANGA.uproject" -LiveCoding -WaitMutex
```

That establishes the requested compile-only checkpoint. It does **not** establish successful injection into the running editor, a full cold linked build, packaged parity or runtime acceptance. Never use `-NoLiveCoding`. For actual running-editor injection, use RzMCP `compile_project`; for a normal build, the editor must be closed under the user's current authorization.

Previously completed checks, before the final compile-only checkpoint:

- Five focused Python tests passed using `G:\QANGA\Intermediate\PipInstall\Scripts\python.exe` from `Plugins/RzNeuralNetwork/Content/Python`, with ResourceWarnings treated as errors: partial curriculum stream/drain, cross-process writer lock, read-only fresh preflight, exact durable generation, and versioned inference export.
- Thirteen native `RzNeuralNetwork.Dreamer.*` matches passed, including the separately prefixed observation-schema test; five `QATS.Quest.OfflineTutorial*` tests passed; `QATS.DynamicQuestSystem.KillObjective.QuestSpawnIdentity` passed.
- The three command assets and six CommandComponent referencers compiled with zero errors/warnings during semantic integration. RequestToken was verified on the preserved Server RPC.
- `QSystem.CommandExecution.ScriptState` passed. `ComponentOwnership` initially failed on premature script destruction; its callback was changed from conditional OnEndPlay to OnDestroyed, but that final correction was **not rerun** afterward.
- No final QAI automation run, PIE, multiplayer behavior run, packaged campaign, trainer launch or performance measurement was performed. The editor was left open.

Focused next checks, after coherent implementation and the appropriate current build:

- QAI: `AuthoredGeometryCoverage`, `NavigationTopologyRevision`, `RequesterLeaseOwnership`, `PublishedRouteValidity` under `QAI.Pathfinding`.
- Command receipt: both `QSystem.CommandExecution.*` tests, then actual command dispatch/adoption when runtime execution is authorized.
- Lifecycle: passive-to-combat dedicated-server setup; cancellation during pending equipment restoration; exact admin/ammo/level/save/input release; foreign prerequisites untouched; same-second unique artifacts; second PIE run rejected before mutation; terminal artifact failure propagation.
- Objective scenarios: complete-on-exit, already visited unordered locations, location-gated scanning, rejected/closed dialogue, restricted kill targets, valid unchanged corridor versus changed route.
- Keep editor automation, live-patch validity, PIE, packaged behavior and user acceptance separate in the handoff. Do not call the complete QATS gate trustworthy solely because the source compiles.

## Handoff expectation

At each resumed milestone, state what was changed, what was actually checked, and what remains. Respect the user's current stopping point. If asked for staging, include only owned paths/hunks and check the cached diff; if asked for a French commit summary, describe actual behavior changes only. Do not launch another broad audit or accumulate replacement abstractions when a narrow existing owner can satisfy the requirement.
