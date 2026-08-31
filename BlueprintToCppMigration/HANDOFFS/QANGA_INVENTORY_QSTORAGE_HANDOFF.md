# QStorage -> QInventory native persistence handoff

## Changed files

- `Plugins/QStorage/QStorage.uplugin`: enables the one-way `QInventory` plugin dependency.
- `Plugins/QStorage/Source/QStorage/QStorage.Build.cs`: adds the public `QInventory` module dependency.
- `Plugins/QStorage/Source/QStorage/Public/QStorage_Types.h`: adds schema-2 native endpoint records, stable identity/version/generation state, and journal endpoint descriptors.
- `Plugins/QStorage/Source/QStorage/Public/QStorage_Codec.h`
- `Plugins/QStorage/Source/QStorage/Private/QStorage_Codec.cpp`: encodes/decodes the complete versioned QInventory codec payload and retains the existing legacy crate record boundary.
- `Plugins/QStorage/Source/QStorage/Public/QStorage_Persistence.h`
- `Plugins/QStorage/Source/QStorage/Private/QStorage_Persistence.cpp`: implements bounded atomic segment decoding, exact failure preservation, ordered async persistence, and shutdown draining.
- `Plugins/QStorage/Source/QStorage/Public/QStorage_World_SubSystem.h`
- `Plugins/QStorage/Source/QStorage/Private/QStorage_World_SubSystem.cpp`: owns native endpoint registration, exact identity/version checks, semantic publication gates, and neutral journal coordination hooks.

No `Plugins/QInventory/**` file was changed.

## Persisted format and contracts

- Dependency direction is only `QStorage -> QInventory`; no reverse dependency was introduced.
- QStorage segment schema 2 retains legacy records through their existing `QSTCRATE` representation and adds native records encoded as `QSTINV<US>1<US>ContainerGuid<US>ContentVersion<US>PersistenceGeneration<US>Base64(FQInventoryCodec bytes)`, where `<US>` is byte `0x1F`.
- The native record carries the complete canonical QInventory endpoint codec payload, including item instances, attachments, customization, and extensions. It does not project through `FQST_ItemStack`.
- Decode is bounded and atomic. The QInventory codec checks framing/checksum/truncation/trailing data, then `FQInventoryValidator` checks semantic validity before publication. Envelope GUID, content version, persistence generation, and canonical re-encoding must also agree.
- A malformed, truncated, checksum-invalid, or semantically invalid primary segment never publishes partial/default data and is never auto-cleaned, migrated, or overwritten. A valid retained backup may remain the last valid snapshot, but the exact segment and its endpoints become read-only. A missing primary with `.bak` or `.tmp` present is treated as an interrupted rotation, not as a new empty writable segment.
- Native registration requires stable endpoint/container GUID, exact live anchor actor, exact live adapter object, authority/world ownership, and durable-record identity. Unregister requires the same GUID/anchor/actor/adapter tuple. The native path has no BuilderID, ordinal, array-index, or linear-search identity fallback.
- Initialization requires the caller's expected content version to equal both the durable record and decoded QInventory state. A mutation must provide the exact expected current version and advance content version and persistence generation by exactly one.
- Each endpoint has ordered persistence generations. A stale async completion cannot replace memory or clear a newer dirty revision. Failure keeps/re-dirties the newest generation and reports resolution to the journal coordinator. Shutdown stops new writes, drains the in-flight write, then resolves the newest dirty generation synchronously.
- Logging on these failure paths is throttled to at most one message per second per source.

## Legacy path retained

Unrelated QStorage consumers still use schema-1/schema-2 legacy crate records and the existing `FQST_ItemStack` APIs. Schema 1 remains readable and is not rewritten merely because it was read. Legacy content APIs are rejected only for a durable identity currently owned by a native QInventory endpoint; legacy-only consumers remain available. DataManager and Blueprint presentation/adapter work remain untouched.

## Two-endpoint durability blocker and coordinator hook

QStorage does not claim atomic transfer durability and will not issue two unordered endpoint writes. A mutation is accepted only after an `IQST_InventoryJournalCoordinator` confirms a durable prepare through:

- `HasDurablePrepare(TransactionId, Endpoint, Error)`, where `Endpoint` contains exact complete before/after bytes plus endpoint identity, content versions, and persistence generations.
- `OnEndpointWriteResolved(TransactionId, EndpointId, Generation, bSuccess, Error)`, delivered on the game thread for the coordinator to resolve both participants and re-dirty the pair on failure.

The remaining blocker is the later neutral coordinator itself. It must durably write the paired prepare, dispatch both storage backends, write a commit marker only after both endpoint resolutions succeed, and recover an uncommitted transaction after a crash. No in-memory success fallback or fake atomicity was added here.

## Validation required from the main agent

This worker ran static review and `git diff --check` only. It did not build, run UHT/QATS, open Unreal Editor, or perform persistence/runtime/network validation.

1. Run a cold UHT/non-unity Win64 Editor build and the applicable Linux server build; then run all `QATS.QInventory.*` tests.
2. Exercise schema-1 legacy load and deterministic schema-2 mixed legacy/native round-trip. Verify exact equality of the complete QInventory payload, including attachments, customization, and extensions.
3. Inject malformed Base64, truncation, checksum flips, trailing bytes, envelope GUID/version/generation mismatches, and semantic-invalid state. Confirm output remains unchanged, the exact segment/endpoints are read-only, and flush/restart does not alter the primary/backup bytes or create defaults.
4. Test missing primary with `.bak`/`.tmp`, valid backup retention, and no-valid-backup behavior across restart.
5. Reject register/unregister calls with any mismatched GUID, anchor, actor, or adapter; verify two containers with distinct GUIDs never alias. Reject stale expected versions, non-consecutive version/generation mutations, and mutation without a durable coordinator prepare. Confirm legacy-only consumers still work while legacy projection calls for a native-owned identity fail.
6. Delay write generation N, enqueue N+1, and confirm N cannot clear dirty state or regress durable state. Inject N/N+1 failures and confirm the newest generation stays dirty and the coordinator receives the exact failed endpoint resolution.
7. Begin shutdown with a write in flight and confirm it drains before the newest dirty generation is synchronously resolved and survives process restart.
8. After the neutral coordinator exists, crash-test after paired prepare, after either endpoint write, after both writes before commit, and after commit. Confirm recovery never exposes a one-sided durable transfer.
