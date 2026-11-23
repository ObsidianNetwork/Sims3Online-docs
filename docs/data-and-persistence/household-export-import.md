# Household export and import

This page separates the implemented byte transport for household exports from the unsafe deep-state mutation path. `HouseholdExportPayload` describes a property stream and its remap tables; it is not equivalent to a native world save, and current session reconciliation intentionally does not promote arbitrary deep imports into the authoritative game.

## Engine concept and payload boundary

The historical engine investigation found the `Household` object implements the game's `IExportableContent` persistence interface. Its export operation writes to an `IPropertyStreamWriter` using `ResKeyTable` and `ObjectIdTable`; import consumes the corresponding reader and mapping tables. Other persistable game entities use the same style of property-stream serialization. This explains why the payload is richer than a `SimulationSnapshot`, but it does not make the API safe to apply into an already-instantiated live household.

The repository-level record is `HouseholdExportPayload` in `SimsOnline.Protocol/LotExport.cs`:

| Field | Role |
|---|---|
| `HouseholdId` | Household the client says it exported |
| `PropertyStreamBytes` | Serialized household properties |
| `ResKeyTableBytes` | Resource-key remap metadata |
| `ObjectIdTableBytes` | Object-identity remap metadata |
| `EntryCount` | Count reported by the host exporter |
| `ResKeyCount` | Number of resource-key entries |
| `ObjectIdCount` | Number of object-id entries |
| `SimInventories` | Optional per-Sim inventory segments |
| `SharedFridgeInventory` | Optional shared fridge inventory segment |
| `SharedFamilyInventory` | Optional family inventory segment |

Each `InventoryExportPayload` has `SlotIndex`, its own property stream and two tables, and an item count. The documented slot convention is `0..N-1` for Sim descriptors, `-1` for shared fridge, and `-2` for shared family inventory.

## Why a household stream is not a world save

`Household.ExportContent` captures a domain object using the engine's object persistence protocol. It does not write all world-level DBPF records, native routing/world state, or the complete set of references expected during a normal world load. `WorldSave` instead transfers the entire set of files from a game save folder. Therefore:

- Use a native `SAVE_GAME` checkpoint and `WorldSave` when the purpose is to give a client a loadable world.
- Treat `HouseholdExportPayload` as a bounded serialization artifact for a specific import/reconcile path.
- Do not infer cross-save safety just because an export and import both return success.
- Do not interpret snapshot DTOs as an alternate deep export.

## Controller and bridge flow

The current types define two layers:

1. `LotExportBundle` carries lot metadata and selected Sim records; its optional `DeepExport` contains household bytes.
2. The bridge bulk transport carries a `HouseholdExportPayload` outside the small control frame. `HouseholdExportTransport` verifies metadata and decodes that blob into property/table sections and inventory segments.

A client command example, with fabricated one-byte data, is:

```json
{
  "$type": "submit_lot_export",
  "commandId": "<command-guid>",
  "playerId": 42,
  "clientLastReconciledVersion": 0,
  "bundle": {
    "playerId": 42,
    "lotId": 300,
    "clientTick": 1234,
    "sims": [
      { "objectId": 500, "position": { "x": 1.25, "y": 2.0, "z": 3.0 }, "currentInteractionKey": "Chat" }
    ],
    "deepExport": {
      "householdId": 600,
      "propertyStreamBytes": "AQID",
      "resKeyTableBytes": "BAU=",
      "objectIdTableBytes": "Bg==",
      "entryCount": 1,
      "resKeyCount": 1,
      "objectIdCount": 1,
      "simInventories": [],
      "sharedFridgeInventory": null,
      "sharedFamilyInventory": null
    }
  }
}
```

This is wire-shape documentation only. `DedicatedServer.Tests/WireJsonPinningTests.cs` pins the concrete `submit_lot_export` encoding, including the camel-case `deepExport` fields and base64 representation of byte arrays.

## Bulk transport bounds and validation

The bridge's shared bulk region is larger than a control frame, but an export is still constrained to one bulk slot. The protocol declares `HouseholdExportValidator.MaxBulkSlotBytes` as `512 * 1024`. `HouseholdExportTransport.DecodePayload` additionally requires declared lengths to agree with the byte array actually read.

| Validation boundary | Rejection case |
|---|---|
| Household identity | `metadata.HouseholdId` differs from requested ID |
| Total payload | Nonpositive or exceeds shared bulk-slot maximum |
| Base stream | Property stream length is zero, entry count nonpositive, or tables absent |
| Table counts | Negative count, or positive table count with zero bytes |
| Arithmetic | Combined lengths overflow `Int32` |
| Inventory descriptors | Slot/length fields malformed or segment extends beyond blob |
| Payload lengths | Household + inventory-described bytes differ from metadata or actual bulk bytes |
| CRC | `BridgeCrc32.Compute(bulkPayload)` differs from metadata checksum |
| Semantic validator | Decoded payload fails `HouseholdExportValidator.Validate` |

The transport result distinguishes `HouseholdExportTransportError` values such as `HouseholdMismatch`, `PayloadLengthOutOfRange`, `TableMetadataMalformed`, `PayloadLengthMismatch`, `PayloadCrcMismatch`, and `ValidatorRejected`. Preserve the specific error when diagnosing malformed input; collapsing every case to “bad export” makes it impossible to tell size mistakes from corruption.

The metadata separates the main household triplet sizes from optional inventory descriptor lengths. The single bulk array is ordered as household property stream, resource table, object table, then inventory triplets in descriptor order. Length checks happen before the final payload object is accepted.

## Import safety and quarantine

The old engine analysis documented a critical distinction: stock travel/bin import creates a fresh household, performs object and Sim remapping, runs post-import fixups, then merges selected travel information into existing `SimDescription` instances. A direct `Household.ImportContent` call on an already-live target can leave member collections and actor references inconsistent. The observed failure was not visible from in-memory field comparisons; a later cold load failed.

The public repository reflects that boundary in `LotExportBundle.DeepExport` comments and `LotReconcileCoordinator`: the deep import path remains quarantined until there is an engine-supported lifecycle with cold-reload proof. Although bridge codec/host methods exist for export and import transport, their existence is not an endorsement to wire them as production reconciliation. The session keeps requested deep responses from silently becoming a durable merge.

Do not treat `HouseholdImportResult.Success` as a durability certificate. It only reports the import operation and summary; a successful `SAVE_GAME` response after an unsafe import would also be insufficient. Re-enablement requires at least one save/reload cycle, then repeated-cycle validation, with references, identities, relationships and inventories checked after load.

## Cross-save identity and inventory caveats

`ObjectIdTable` exists to translate object identities, but `HouseholdExportValidator.ObjectGuidRemapRequiresProof` is `true`. If `ObjectIdCount > 0`, validation returns `ObjectGuidRemapUnproven`. This prevents the controller from assuming a successful byte transfer also remapped every live `ObjectGuid` safely.

The inventory segments are separate because household contents can reference Sim inventories and shared household containers. A bare household property stream may include references to inventory records without carrying the actual inventory resources. The current payload model can describe those segments, but a caller must confirm they are present, complete, and compatible with the matching import path.

## Failure modes

| Symptom | Interpretation |
|---|---|
| `HouseholdExportTransportError.PayloadCrcMismatch` | Bulk bytes differ from metadata checksum; reject before use |
| `PayloadLengthMismatch` | Metadata, segment descriptors and actual bytes disagree |
| `ObjectGuidRemapUnproven` | Object table is nonempty and its remap behavior is not accepted |
| Export accepted, no inventory items appear | Inventory segments or matching import policy may be absent |
| Import appears correct until restart | In-memory equivalence did not prove persistence consistency |
| Cold load fails after import | Treat imported save as invalid; retain previous verified save |
| Cross-process item cannot resolve | Runtime object identity belongs to another game instance |

## Verification

- `DedicatedServer.Tests/WireJsonPinningTests.cs` contains `PublicProtocolCodec.RoundTripsSubmitLotExport` and pins the nested deep-export wire representation.
- `DedicatedServer.Tests/Program.cs` exercises `HouseholdExportTransport`/host export decoding and lot-reconcile protocol behavior.
- `DedicatedServer.Tests/ProtocolHardeningTests.cs` includes `AuthoritativeSession.RejectsLotExportWithoutClientNativeModes`.
- `DedicatedServer.Tests/LastGoodSaveTests.cs` verifies the separate structural backup path; it does not validate household import semantics.

The unit suite can prove bounds, encoding, identity checks and session policy. It cannot prove the game object's registry consistency or cold-load semantics. Do not copy real household bytes into issues or public docs; test with generated small fixtures.

## Related pages

- [Save/load pipeline](save-load-pipeline.md)
- [Inventory persistence](inventory-persistence.md)
- [Lot reconciliation](lot-reconciliation.md)
- [Protocol](../protocol.md)

## Payload composition example

A fully described payload has separate lengths for the household triplet and each inventory triplet. For example, if the household data consists of 900 property bytes, 80 resource-table bytes and 40 object-table bytes, the household portion is 1,020 bytes. If two inventory descriptors contribute 140 and 55 bytes, the full bulk payload is 1,215 bytes. Metadata must report those exact sections; the decoder cannot infer a missing table from total length.

The simplified `HouseholdExportPayload` JSON shape is not the exact bridge bulk envelope: byte arrays serialize as base64 in public JSON, but the host bridge transfers metadata plus a bulk reference and then reads the bulk bytes. Use `HouseholdExportTransport` codec tests as the authority for bridge metadata order. Never hand-build a live bulk reference from a public JSON example.

| Piece | Expected ordering | Count source |
|---|---|---|
| Household property stream | First | `PropertyStreamBytes` length |
| Household resource-key table | Immediately after property stream | `ResKeyTableBytes` length |
| Household object-id table | Immediately after resource table | `ObjectIdTableBytes` length |
| Sim inventory triplets | After household triplet, descriptor order | Each descriptor's three lengths |
| Shared container triplets | Included only when described by payload model | Segment descriptors/payload fields |

## Remap validation and semantic checks

Byte-level validation protects the decoder from malformed lengths and corruption. It does not prove that the remap tables are internally consistent with the household stream. The validator checks reported entry counts and table sizes against its conservative rules; `ObjectGuidRemapRequiresProof` prevents a nonempty object mapping from being treated as safe without supporting evidence.

If a future engine-supported import path is added, validation should also prove:

1. Every referenced object/Sim identity resolves after remapping.
2. No two source identities unintentionally collapse to one live identity.
3. Household member lists agree with the imported Sim descriptions.
4. Inventory owners and item objects are reachable from the correct live inventories.
5. Relationship endpoints point to the intended Sim descriptions.
6. Save/reload reconstructs the same graph with no dangling references.
7. A second save/reload cycle remains stable.

These are semantic proof obligations, not current guarantees of the transport validator.

## Safe caller behavior

A caller receiving a valid decoded payload should preserve it as an untrusted export candidate until the session policy approves its use. It should not:

- call the live import solely because decode succeeded;
- save the world immediately and publish the save as verified;
- merge a household by matching display names;
- assume all inventory segments are present because a household stream exists;
- re-use runtime object IDs between game processes;
- replace a last-good native save with an import experiment.

When policy quarantines a deep export, report that it was received/validated but not applied. The user-facing and operator-facing wording should make that state explicit.

## Compatibility and operational limits

`HouseholdExportTransport` requires the bridge BULK region and rejects payloads larger than the fixed shared slot. A host running an older bridge protocol may not support this path even if the public protocol can represent `DeepExport`. The bridge version and the server/client game mod build must be recorded with a failure report.

The export also depends on the loaded world containing the requested household and a valid in-world session. A missing household is a host-state error, not a byte-decoding problem. A payload that passes CRC and bounds but fails the semantic validator is a distinct policy rejection.

For debugging, use synthetic payloads with small arrays and known CRCs. Do not copy real household bytes to logs or issue comments: those bytes can contain player-created and personal game data.

## Decoder test fixture design

A transport test should make lengths visible rather than using opaque large blobs. Build each section with a different byte pattern, compute CRC over the concatenated bulk data, then assert that the decoded household triplet and each inventory triplet exactly match the source ranges. A one-byte offset error should fail deterministically.

Test both a payload with no inventory descriptors and a payload whose inventory segments are present. Include a descriptor that ends exactly at the payload boundary and one that exceeds it by a byte. Verify that integer overflow and negative lengths reject before any allocation based on those lengths.

The validator should be tested independently from the bridge reader so the same policy applies to public `DeepExport` DTOs and host bulk transport. Keep malformed input errors specific enough to identify length, CRC, table and semantic-validator failures.

## Release decision gate

A future request to activate deep import should be treated as a feature change, not a documentation-only toggle. Before making it default, require a code-reviewed lifecycle using the game's supported import path, explicit identity remapping and cold-load tests on clean fixtures. Keep position-only reconciliation active unless all deep-state obligations are met.

## Final transport checklist

Before declaring a household export feature ready, verify the public DTO shape, bulk transport, CRC/bounds validation, owner/household authorization, inventory segment correspondence, bridge role gates and cold-load semantics. Until then, document successful decoding separately from import and keep deep import quarantined.
