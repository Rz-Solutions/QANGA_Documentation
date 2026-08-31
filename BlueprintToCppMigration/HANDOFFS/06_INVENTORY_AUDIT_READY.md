STATE: AUDIT_READY

# Inventory audit handoff

## Chosen owner

Create `Plugins/QInventory` as the native gameplay Inventory owner. QStorage remains a downstream persistence adapter for container snapshots and is not the mutation authority. `InventoryComponent_C` remains in production and will be adapted incrementally.

## Proved baseline

- `Lib_Inventory.TryMoveItemToAnotherInventory` returns only `bool`, removes/destroys the source replicated item, recreates it by legacy ID through `ItemsManagerGS`, then adds it to the target.
- The existing add path may merge/consume identity or drop overflow; the move has no rollback, endpoint versions, postconditions, or typed error.
- Its four exact callers are `SV_MoveItemToVault`, `SV_MoveItemToInventory`, `SV_InventoryToVehicle`, and `SV_VehicleToInventory` inside `InventoryComponent_C`. All are reliable client-to-server RPC paths and none carries endpoint versions.
- Legacy identity is timestamp-derived `FName`; payload spans `Obj_ItemInstance_C`, attachment instances, and DataObject keys. Persistent and transient inventory modes are both active production behaviors.
- QStorage's current stack projection cannot represent the full record. Its enabled GUID path still has an ordinal fallback, and its mutation/flush APIs do not yet provide expected-version, semantic fail-closed, generation ordering, failure redirty, or shutdown draining.
- QModule and NPC dialogue/quest paths currently declare mutation success without native postconditions and can lose payload during refund or partial reflected writes.

Full evidence, boundaries, contract, cleanup stages, and gates are in `Documentation/BlueprintToCppMigration/08_INVENTORY_MIGRATION.md`.

## Source contract

- Authority-generated `FGuid` identities only; no ordinal/timestamp fallback in the native domain.
- Versioned complete root record with explicit persistent/transient mode, ItemData key, stack, rarity, owner, slot, complete embedded attachments, customization, extensions, and validity.
- Endpoint schema explicitly distinguishes persistent/transient and carries stable inventory GUID, content version, persistence generation, capacity, owner, and read-only state.
- One deterministic two-lock funnel with expected versions, whole-stack transfer, no merge/drop, full rollback snapshots, exact postconditions, single version/generation increments, and typed results.
- Bounded deterministic codec; malformed/invalid decode never replaces output.

## Exact proposed files

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
- Prompt 06 migration document and source handoff

## Dependencies

`QInventory` depends only on `Core` and `CoreUObject`. It must not depend on QStorage, QModule, DynamicQuestSystem, DataManager, QBuilder, UI, or Blueprint presentation modules.

The QATS source will require Prompt 09 to add `QInventory` to `QAutomatedTestSuite.Build.cs` and the QATS `.uplugin` plugin list. Prompt 06 will not edit those shared files.

## Integration requests for Prompt 09

1. Enable QInventory in the project, add QInventory/QATS dependency declarations, and compile the new module/tests.
2. Implement an InventoryComponent adapter that builds/applies complete native endpoint snapshots through one mutation path; replace the remove/recreate graph only after parity is proven.
   Persistent legacy items require one authority-generated GUID durably stored before mutation; transient items get one lifetime GUID. Never derive identity from legacy names, slots, array order, or builder IDs. Slots come from the existing slot maps, not array ordinal.
3. Resolve RPC endpoints and access on the server; add both expected versions and consume typed results in all four `SV_*` paths.
4. Make QBuilder deposit attribution observe only committed transfer quantity and close/cancel its bracket on every result.
5. Extend QStorage adapter storage to the full encoded QInventory record; add expected-version mutation, semantic validation, actor-identity unregister, strict GUID failure, ordered generations, write-failure redirty, and shutdown drain/newest-generation protection. Persist each transfer through one neutral two-endpoint prepare/commit journal so cross-DataManager/QStorage restarts can never expose only half the move.
6. Replace QModule consume/grant/refund reflection with typed QInventory operations and full-record rollback.
7. Replace NPC dialogue/quest direct stack writes and reflected removal with typed native consume.
8. Bind QWeapon's exact-item ammo/reload adapter to QInventory compare-and-swap and reservation/rollback through a neutral integration owner; remove equipped-item reflection without creating a QWeapon/QInventory cycle.
9. Replace QuestManager item-record probing and QuestAction reflected removal with the typed QInventory record/transaction boundary.
10. Keep ItemsManagerGS and DataManager downstream until the native record and mutation adapter are stable; migrate only the first world-drop codec vertical in P2 without ordinal identity synthesis or a bulk DataManager rewrite.

Prompt 06 now continues directly into the declared source phase. This handoff does not claim compilation, live integration, or persistence parity.
