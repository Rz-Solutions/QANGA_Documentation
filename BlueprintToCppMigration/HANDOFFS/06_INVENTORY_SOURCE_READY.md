STATE: SOURCE_VERIFIED_INTEGRATION_NOT_STARTED

# Inventory source handoff

## Delivered boundary

`QInventory` is the neutral native gameplay Inventory core. The production `InventoryComponent_C` still owns live gameplay; this checkpoint intentionally stops before the first adapter/reparent so the tree contains no half-wired Inventory path. The core remains independent from QStorage, QModule, DynamicQuestSystem, DataManager, QBuilder, QWeapon, UI, and presentation modules.

## Current checkpoint delta

- Endpoint reconciliation is a strict locked compare-and-swap with explicit expected content version, exact metadata/identity validation, one-way read-only state, single version/generation increment on change, no-op preservation, overflow rejection, and byte-semantic state preservation on every failure.
- `Capacity` means bag slots only. Bag records use a concrete unique `[0, Capacity)` slot and no equipment key; equipped records use `SlotIndex == -1` plus a unique non-empty equipment key. Equipped records do not consume bag capacity, while total root records remain bounded to the schema-v1 limit of `4096`.
- Moving an equipped source record clears its equipment assignment and installs the same complete record into the target bag slot.
- The neutral durability journal and QStorage generation/drain integration remain source-only from the production Inventory point of view.
- DataManager now owns the bounded synchronous TempDB row capture/write/flush/raw-reload/readback/rollback primitive. Offline Tutorial consumes it and no longer carries a second reflection/save implementation.
- Live signatures captured for the next adapter are `TryMoveItemToAnotherInventory(Source, Target, Item, WorldContext) -> bool` plus reliable server events `SV_MoveItemToVault(ItemId)`, `SV_MoveItemToInventory(ItemId)`, `SV_InventoryToVehicle(ItemId, VehicleInventory)`, and `SV_VehicleToInventory(ItemId, VehicleInventory)`.
- Cold Editor build is green. QATS are green at Inventory `35/35`, QStorage `11/11`, DataManager `4/4`, Offline Tutorial `5/5`, and Combat `16/16`.

## Exact next resume boundary

Create the first live Inventory integration owner around one stable endpoint per `InventoryComponent_C`. Materialize bag and equipment from their distinct authoritative legacy maps, reconcile through the strict endpoint CAS, resolve both endpoints and access server-side, and route only the four transfer RPCs through the typed transaction/journal boundary. Do not start another Inventory writer in parallel. Delete the old remove/recreate graph only after the same-instance transfer, rollback, network-race, restart, and packaged gates pass.

Implemented source behavior:

- stable authority-owned `FGuid` identities; no ordinal, timestamp, array-index, slot, legacy-name, or BuilderID fallback;
- schema-v1 endpoint, root item, and complete embedded attachment records with explicit persistent/transient mode, ItemData key, stack, rarity, owner, slot/mount, customization, lossless extension data, and validity;
- pure semantic validator for schema/mode/GUID/key/quantity/rarity/owner/slot/capacity/version/generation/validity, root-slot uniqueness, and root/attachment identity uniqueness;
- bounded canonical UTF-8 binary codec with explicit magic/version/payload length, deterministic item/attachment/map ordering, CRC32, full-buffer consumption, and output assignment only after semantic success;
- stable endpoint mutexes and deterministic lexicographic GUID lock order independent of transfer direction;
- one authority/access-checked whole-record transfer funnel with source and target expected versions, same-endpoint rejection, no partial stack split, no merge, no overflow drop, and explicit result codes;
- controlled transfer changes only: target owner, target persistence mode, and target slot. Identity and all other payload survive unchanged;
- complete source/target snapshots, exact remove/add checks, post-mutation equality against `source snapshot - record` and `target snapshot + record`, single content-version and persistence-generation increments, plus mutation-backend `BeginTransaction`, `CommitTransaction`, and `RollbackTransaction` hooks for future legacy UObject adapter state;
- any begin/remove/add/finalize/postcondition failure invokes adapter rollback and restores both native endpoint snapshots under the same locks. Rollback failure is a distinct terminal result;
- no logging, reflection, fallback path, dead implementation, asset dependency, or presentation behavior in the core.

The codec, reconciliation, transaction, durability coordinator, QStorage seam, and DataManager durable-row primitive do not claim live Inventory Blueprint integration. The production adapter and its runtime/network/restart gates remain below.

## Exact owned files

- `Plugins/QInventory/QInventory.uplugin`
- `Plugins/QInventory/Source/QInventory/QInventory.Build.cs`
- `Plugins/QInventory/Source/QInventory/Public/QInventory.h`
- `Plugins/QInventory/Source/QInventory/Public/QInventoryTypes.h`
- `Plugins/QInventory/Source/QInventory/Public/QInventoryValidation.h`
- `Plugins/QInventory/Source/QInventory/Public/QInventoryCodec.h`
- `Plugins/QInventory/Source/QInventory/Public/QInventoryTransaction.h`
- `Plugins/QInventory/Source/QInventory/Private/QInventory.cpp`
- `Plugins/QInventory/Source/QInventory/Private/QInventoryTypes.cpp`
- `Plugins/QInventory/Source/QInventory/Private/QInventoryValidation.cpp`
- `Plugins/QInventory/Source/QInventory/Private/QInventoryCodec.cpp`
- `Plugins/QInventory/Source/QInventory/Private/QInventoryTransaction.cpp`
- `Plugins/QAutomatedTestSuite/Source/QAutomatedTestSuite/Private/QInventoryAutomationTests.cpp`
- `Documentation/BlueprintToCppMigration/08_INVENTORY_MIGRATION.md`
- `HANDOFFS/06_INVENTORY_AUDIT_READY.md`
- this handoff

No other source, module, config, asset, plan, or consumer file was edited by this lane.

## QATS source coverage

The current suite declares 35 `QATS.QInventory.*` tests, including the original transfer/codec cases plus strict reconciliation, bag/equipment semantics, and the durability journal. The focused checkpoint also covers 11 QStorage tests, 4 DataManager tests, and 5 Offline Tutorial contract tests.

The original source list included:

- `QATS.QInventory.Transfer.SuccessPreservesCompleteRecord`
- `QATS.QInventory.Transfer.AuthorityAndReadOnlyBoundaries`
- `QATS.QInventory.Transfer.TargetFullIsAtomic`
- `QATS.QInventory.Transfer.TwoEndpointVersionChecks`
- `QATS.QInventory.Transfer.SourceEqualsTarget`
- `QATS.QInventory.Validation.DuplicateIdentityFailsClosed`
- `QATS.QInventory.Validation.QuantityKeyAndSlot`
- `QATS.QInventory.Transfer.RemoveAddAndCommitFailureRollback`
- `QATS.QInventory.Concurrency.DeterministicEndpointOrdering`
- `QATS.QInventory.Persistence.DeterministicRoundTrip`
- `QATS.QInventory.Persistence.MalformedRecordsFailClosed`

The failure backend deliberately mutates source/target state before returning false, and the tests check exact snapshot restoration plus adapter rollback invocation. All current focused suites listed above were run from the cold-built Editor.

## Exact Prompt 09 integration requests

### Build and activation

1. Register `QInventory` in the eventual runtime consumer/project plugin dependency without overwriting other dirty plugin entries. It is not currently listed in `QANGA.uproject`; the QATS plugin dependency only activates it for that test module.
2. Parallel Prompt 09 changes now already list `QInventory` as a private dependency in `QAutomatedTestSuite.Build.cs` and as an enabled plugin dependency in `QAutomatedTestSuite.uplugin`. Prompt 06 did not edit those shared files. Prompt 09 must retain/review those entries and must not duplicate or alter the test source contract.
3. Compile/UHT only after the remaining runtime adapter dependencies are integrated, then run the focused QATS list above.

### `InventoryComponent_C` / `Lib_Inventory`

1. Give every live Inventory component one persistent native endpoint owner; do not rebuild version state from scratch per request. Replicate/expose the current content version only to clients authorized to issue a move.
2. Materialize complete records from the real slot maps and DataObject fields. Array position is not a slot or identity. Resolve every attachment identity to its complete attachment payload.
3. For a persistent legacy item missing its native GUID, generate one on authority and durably write it to the DataObject before admitting the endpoint for mutation. On failure or duplicate/malformed GUID, mark the endpoint read-only. A transient item receives one authority-generated lifetime GUID.
4. Implement one `IQInventoryMutationBackend` for the legacy façade. `BeginTransaction` snapshots both components' arrays/maps, equipment state, replicated-object registration, the exact root/attachment UObjects, and all changed DataObject fields. `RemoveExact` detaches without destroying. `AddExact` attaches the same UObjects without recreating, merging, consuming, or dropping. `RollbackTransaction` restores all captured state; `CommitTransaction` publishes only the finalized in-memory façade state.
5. Resolve both endpoints and access policy on the server from the authorized interaction/session. Never trust the vehicle component reference supplied by the client. Carry both expected versions and the full-stack quantity.
6. Replace the internals of `Lib_Inventory.TryMoveItemToAnotherInventory` with the typed funnel, then make all four RPC callers consume its result: `SV_MoveItemToVault`, `SV_MoveItemToInventory`, `SV_InventoryToVehicle`, `SV_VehicleToInventory`.
7. QBuilder resource-deposit begin/end/cancel must bracket one transaction and count only the committed quantity. Every failure closes/cancels the bracket. Preserve the existing first-add notification exactly once.
8. After parity is proven, delete the old remove/destroy/recreate-by-ID graph and any superseded helper branches. Disabling them is not cleanup.

### Persistence coordinator, QStorage, and DataManager

1. Add a neutral Inventory persistence coordinator for two-endpoint transaction journals. One journal unit contains `TransactionId`, both before/after encoded snapshots, expected versions, and both generations. Persist a prepare before endpoint writes and a commit marker after both. Recovery restores both before states for an uncommitted prepare or completes both after states for a committed unit.
2. QStorage remains a backend snapshot writer for containers. Add its one-way dependency on QInventory, store the complete encoded endpoint rather than a lossy `FQST_ItemStack` projection, validate with `FQInventoryCodec`, and require expected content version.
3. Change QStorage unregister to verify stable container identity and exact live adapter/actor identity. When QStorage is enabled, GUID resolution failure is explicit; remove every `BuilderID`/ordinal fallback.
4. Add per-endpoint generation ordering, stale-completion rejection, paired failure re-dirty, and shutdown draining/resolution. An older async write must never overwrite a newer or synchronous generation.
5. A malformed segment retains the last valid snapshot, marks both segment and endpoint read-only, and is never cleaned up, migrated, or overwritten automatically.
6. DataManager remains the player/vault legacy snapshot backend until P2. Its first P2 scope is only `DataManagerItemRecordCodec` plus one world-drop round trip. It must not mutate Inventory or become a bulk rewrite.

### Other mutation consumers

1. Replace `QModule_InventoryBridge` grant/consume/refund reflection with typed QInventory operations. Refund/rollback restores the exact original record and identity, including attachments/customization/extensions.
2. Replace NPC dialogue direct reflected `Stack` writes and ad-hoc removal, `QuestActionBase` reflected removal, and `QuestManagerSubsystem` ItemData/identity probing with typed record/consume results.
3. Bind QWeapon's exact-item ammo compare-and-swap and reserve-ammo reservation/rollback adapter to QInventory through a neutral integration owner. Remove `QWeaponBulletSubsystem` equipped-item reflection without creating a QWeapon/QInventory dependency cycle.
4. Keep `ItemsManagerGS` as GUID-keyed materialization/cache consumer and DataManager as persistence consumer. Do not migrate them as gameplay mutation owners.

## Prompt 09 validation gates

### Static/build

- enable plugin/dependencies and run UHT plus non-unity Win64 Editor build;
- build Linux server to validate the neutral runtime boundary;
- run all `QATS.QInventory.*` tests;
- compile the four Inventory RPC graphs and every changed adapter with zero Blueprint error/warning.

### Runtime/network

- standalone: player↔vault, player↔vehicle, attachments/customization, explicit slot, auto-slot, full/occupied target, and transient↔persistent mode;
- listen server plus one client: authorized owner transfers and forged source/target/component/version/quantity requests;
- dedicated server plus two clients racing the same item: exactly one commit and one stale typed result, no duplicate/loss;
- simultaneous opposite-direction transfers between the same endpoints: deterministic lock order and no deadlock;
- add/remove/finalize adapter failure after partial legacy mutation: exact native, UObject, map, equipment, replication, and DataObject rollback;
- quest deposit and other consumers observe one committed event and never a failed attempt;
- QModule grant/consume/refund and weapon reload failure preserve exact payload/revision.

### Restart/failure lifecycle

- player, vault, vehicle/container, equipment, attachments, customization, and persistent/transient behavior across process restart;
- transient patrol first-spawn loadout once, persistent patrol no regeneration;
- tutorial capture/restore remains exact and idempotent;
- crash after journal prepare, after either endpoint write, and after commit marker;
- failed write re-dirties the paired generations; out-of-order completion cannot regress either endpoint;
- shutdown during an in-flight paired write resolves the newest journal before exit;
- malformed/truncated/checksum-invalid segment loads the last valid state read-only and survives restart untouched;
- two QBuilder containers keep distinct GUID-owned content with no ordinal fallback.

## Verification state

- Full cold `QangaEditor Win64 Development` build completed successfully after UHT processed the new reflected contracts.
- `QATS.QInventory.*` `35/35`, `QATS.QStorage.*` `11/11`, `QATS.DataManager.*` `4/4`, `QATS.Quest.OfflineTutorial*` `5/5`, and `QATS.QCombat.*` `16/16` completed with zero failure.
- Both independent static reviews of endpoint reconciliation and bag/equipment semantics reported no finding.
- No Inventory asset was reparented, stripped, or saved; no PIE, cook, package, or EasyCook rescan was run.

This is a verified source checkpoint. It does not claim live Inventory adapter integration, network parity, process-restart parity, or packaged parity.
