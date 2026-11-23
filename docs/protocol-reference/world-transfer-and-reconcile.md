# World transfer and lot reconciliation

World bootstrap is a one-way transfer of the authoritative server's saved world. Lot reconciliation is a separate, explicitly scoped upload from a client-native lot owner. A `RequestWorld` file stream is not the lot-export protocol, and a lot export must not carry global state.

## RequestWorld and chunk assembly

`ClientCommand.RequestWorld` can be sent before a player joins and before the client TS3 has loaded anything. The server checkpoints or reuses its current save according to its join-save policy, then emits `WorldFileChunk` records to the requesting connection only. A chunk carries:

| Field | Type | Meaning |
|---|---|---|
| `FolderName` | string | Save folder name ending in `.sims3` |
| `FileIndex` / `FileCount` | int | Ordered file position and number of files |
| `FileName` | string | Flat filename within the save folder |
| `Offset` / `FileLength` | long | Current offset and complete file length |
| `Data` | byte[] | Chunk bytes; JSON base64-encodes this field |

`WorldSave.ChunkBytes` is 256 KiB. Files are emitted in order; an empty file still emits one empty chunk. The client-side `WorldSaveWriter` writes to a staging folder named `<save>.incoming`, checks folder/file names, requires contiguous file indices and offsets, then writes a marker file and moves the staging folder into place only after the final file is complete. If an existing target save lacks the SimsOnline marker, the writer refuses to replace it. A failure aborts and removes staging output without altering the existing user save.

```json
{"$type":"world_file_chunk","folderName":"Online.sims3","fileIndex":0,"fileCount":1,"fileName":"world.nhd","offset":0,"fileLength":3,"data":"AQID"}
```

This is a schematic three-byte example; production chunks can be much larger. The server does not send world chunks as unsolicited broadcasts.

## Client-native lot exports

The lot authority enum carried by `LotLocalAuthorityModeChanged` is:

| Value | Mode | Mirror behavior |
|---:|---|---|
| 0 | `FullPuppetFallback` | Client mirrors its owned Sims through apply events. |
| 1 | `SoloNative` | The lone player runs owned Sims natively; their own Sims are not puppeted. |
| 2 | `ProxMirror` | Owned Sims stay native while nearby foreign player Sims are mirrored. |

`SubmitLotExport` carries a `LotExportBundle`: `PlayerId`, `LotId`, `ClientTick`, a list of `LotSimExport(ObjectId, Position, CurrentInteractionKey?)`, and optional `DeepExport`. The client submits it after `LotExportRequested` and may send a best-effort export before a graceful leave in a client-native mode.

`LotExportValidator.FilterToOwnedLot` in `DedicatedServer\Mirroring\LotReconcile.cs`
always checks each Sim against the player's claims. With a populated controller lot
index, it also rejects unknown Sims and Sims whose indexed lot differs from the declared
`LotId`. During empty-index bootstrap, that lot check is unavailable: owned Sims can pass
without independent verification of the declared active lot. This is not a general
guarantee that every export comes from the player's currently active lot. The verified
minimal merge applies positions; deep household state remains quarantined.

`ClientLastReconciledVersion` is the version last acknowledged to that client; a mismatch means the client may have missed a server merge and affects field conflict classification.

`LotReconcileAcknowledged(PlayerId, LotId, Version)` tells the client which monotonic version to retain. Reconcile requests include a reason tag such as `checkpoint_freshness` or `mode_exit`; it is diagnostic context, not a permission claim.

## Deep household payload boundary

`HouseholdExportPayload` contains `HouseholdId`, property stream bytes, resource-key and object-ID tables, entry counts, and optional per-Sim/fridge/family inventory segments. `HouseholdExportValidator.MaxBulkSlotBytes` is 512 KiB. It rejects null/malformed/inconsistent payloads, oversized totals, and non-empty object-ID tables while ObjectGuid remapping remains unproven. `HouseholdExportRejectionReason` enumerates `Accepted`, `Oversized`, `CrcMismatch`, `Malformed`, `Inconsistent`, and `ObjectGuidRemapUnproven`.

The existence of a `DeepExport` field is not proof that all deep imports are enabled: current code comments keep the requested deep-export path quarantined pending a safe reload lifecycle. The established minimal path is the independently validated position-level lot export. When client-native modes are off, a submitted lot export is rejected and no merge reaches the server host; `DedicatedServer.Tests\ProtocolHardeningTests.cs` pins that rule.

Related pages: [snapshots and resynchronization](snapshots-and-resync.md), [bridge layout](bridge-layout.md), [errors and refusals](errors-and-refusals.md).
