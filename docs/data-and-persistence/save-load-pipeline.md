# Save and load pipeline

The save pipeline deliberately separates game serialization, controller recovery, and client transfer. The host game produces its native save; the controller reads and checks the resulting files; a joining client stages a copy before making it visible under the save name. None of the controller-side steps replaces the game's load path.

## Checkpoint lifecycle

`HeadlessSims3Host.CheckpointWorldAsync` is the current host entry point for a join checkpoint. It acquires `_saveGate`, so public saves and join checkpoints do not overlap. The checkpoint slot comes from `--checkpoint-save` or `SIMSONLINE_CHECKPOINT_SAVE`; when unset, the configured boot save is used. A join checkpoint is a distinct slot when configured, so checkpointing does not overwrite the boot save.

With join reuse enabled, the controller compares a saved-world fingerprint before deciding whether a previous checkpoint can be reused. `SIMSONLINE_JOIN_REUSE=0` disables that optimization and forces a fresh checkpoint for each join. A public save invalidates reusable bytes unless it is the checkpoint slot; the tests ensure a save cannot make the next join receive stale bytes.

| Condition | Behavior |
|---|---|
| No checkpoint slot configured | Use the configured boot save |
| Checkpoint slot configured | Save/read that slot for join transfer |
| `SIMSONLINE_JOIN_REUSE` is unset or not `0` | Reuse may be considered after fingerprint validation |
| `SIMSONLINE_JOIN_REUSE=0` | Always take a fresh checkpoint |
| Save/checkpoint in progress | Wait on the shared save gate rather than overlap |

`CheckpointSaveTests.cs` names executable cases such as `CheckpointSave.JoinWritesSeparateSlotAndLastGood`, `CheckpointSave.DisabledReuseStillUsesCheckpointSlot`, and `CheckpointSave.PublicSavesRefreshOrInvalidateJoinCache`.

## Save files and transfer chunks

`WorldSave` contains a `SaveName` and `IReadOnlyList<WorldSaveFile>`. Each `WorldSaveFile` is a filename plus its raw `byte[]` data. `FolderName` appends `.sims3` if needed. `ReadFolder` reads direct child files, sorting their paths with `StringComparer.OrdinalIgnoreCase` before assigning the ordered file indexes.

The default chunk size is `WorldSave.ChunkBytes`, exactly `256 * 1024` bytes. `ToChunks` emits one `ServerEvent.WorldFileChunk` per chunk, ordered by file and offset. The event fields are:

| Field | Meaning |
|---|---|
| `FolderName` | Destination save folder name, ending in `.sims3` |
| `FileIndex` / `FileCount` | Position of this file in the save and number of files |
| `FileName` | Name of the file inside the folder |
| `Offset` | Byte offset where this chunk belongs |
| `FileLength` | Expected full file length |
| `Data` | This chunk's bytes |

An empty file still produces one empty chunk, so the file's existence is represented. Example construction from the current API:

```csharp
var file = new WorldSaveFile("Meta.data", bytes);
var world = new WorldSave("Neighborhood", new[] { file });
foreach (ServerEvent.WorldFileChunk chunk in world.ToChunks())
{
    // Send chunk in sequence; client writer validates file and byte offsets.
}
```

This is an API example, not a standalone CLI command. The protocol event discriminator is `world_file_chunk` in `ServerEvent`.

## Client staging and overwrite protection

`WorldSaveWriter.Add` validates and writes each chunk to a sibling staging folder named `<target>.incoming`. It checks the destination folder suffix and invalid path characters, ensures the chunk's folder name stays consistent, requires file indexes to advance sequentially, and checks that each offset matches the current output stream position.

When the last file reaches its declared length, the writer closes the file, creates `simsonline-world.txt` in the staged folder, removes the existing destination only if that folder has the SimsOnline marker, and moves staging into place. A folder without the marker is never overwritten. Any exception aborts the partial staging folder and propagates to the caller; `IsComplete` is set only after promotion.

| Writer state | Observable property |
|---|---|
| Receiving | `IsComplete == false`; `FolderName == null` |
| Promoted | `IsComplete == true`; `FolderName` returns the finished folder |
| Failed | `Add` throws; partial `.incoming` directory is removed where deletion succeeds |

Do not infer that a complete transfer is a successfully loaded world. The caller still has to load the save through the game and wait for its normal world-ready path.

## Last-good copy

`LastGoodSave.Keep` accepts a `WorldSave` only when `Problem` returns null. The check requires an `.nhd` file, rejects empty files, and checks DBPF-like files beginning with `DBPF`: header length must be at least `0x60`; the index end must equal file length; index size must equal `4 + 32 * recordCount`, with the count and offsets read from the implemented header locations. Non-DBPF files are not treated as DBPF failures.

The method writes each data file to `<save>.lastgood.incoming` using `WriteThrough`, flushes to disk, rereads it, and compares SHA-256 with the input bytes. It writes `simsonline-lastgood.txt` containing file name, byte length, and lowercase SHA-256, then swaps the new staging directory into `<save>.lastgood`. If validation or writing fails, the previous retained copy is left intact.

At boot, `EnsureBootable`:

1. Checks the main folder with `Problem` and checks an available last-good manifest with `Verify`.
2. Leaves a structurally whole main save alone, including a whole save that differs from the retained copy.
3. If the main save is missing or incomplete and the last-good copy verifies, moves the broken target aside as `.broken-yyyyMMdd-HHmmss` (when present) and copies the retained data files back.
4. If there is no valid last-good copy, returns a diagnostic without claiming restore succeeded. A valid `.backup` is mentioned but is not automatically restored by this code.

Example manifest entry shape (the actual writer computes the digest):

```text
<64 lowercase hex characters> <byte length> <file name, spaces allowed>
```

The parser splits into three fields so names containing spaces remain intact.

## Failure cases and limits

| Symptom/result | Meaning | Safe interpretation |
|---|---|---|
| `no .nhd file` | Required world file absent | Do not retain as last-good |
| `DBPF index ends at ...` | File length does not match index end | Candidate is structurally incomplete |
| `does not fit ... records` | Index size and record count disagree | Candidate fails structural validation |
| `Verify` reports missing or changed file | Recovery copy no longer matches manifest | Do not restore it as verified |
| Main folder is whole but differs | It may be a save written outside the controller's last-good path | Code leaves it untouched |
| Main folder incomplete, no copy | No verified restoration source | Code reports the condition and does not claim a recovery |
| Transfer interrupted | Staging may be discarded by writer failure | Do not advertise a loadable save |

These checks do not decode all DBPF records and do not establish that the game can semantically load a save. Test a format or lifecycle change with a disposable copy and the real cold-load path.

## Verification

The focused implementation tests are in `DedicatedServer.Tests/CheckpointSaveTests.cs` and `DedicatedServer.Tests/LastGoodSaveTests.cs`. The latter includes `LastGoodSave.ProblemFindsCutAndMissingFiles`, `LastGoodSave.KeepWritesManifestAndReplacesTheCopy`, `LastGoodSave.KeepRefusesAPartialSave`, and `LastGoodSave.BootRestoresAMissingOrCutSave`. These use synthetic DBPF-shaped bytes and do not substitute for a live game load test.

## Related pages

- [Persistence overview](overview.md)
- [Snapshot fields](snapshot-fields.md)
- [Protocol](../protocol.md)
- [Troubleshooting](../troubleshooting.md)

## Checkpoint slot and join reuse in practice

`HeadlessSims3Host.CheckpointWorldAsync` is the checkpoint boundary for a joining client. The configured boot save and checkpoint save are separate names when `--checkpoint-save` is provided. If that option is omitted, `CheckpointSaveId()` falls back to the boot save name. The environment value `SIMSONLINE_CHECKPOINT_SAVE` is read by `SessionConfig`; command-line configuration is tested to override the environment value.

| Configuration | Effective checkpoint save |
|---|---|
| `--save Boot.sims3`, no checkpoint setting | `Boot.sims3` |
| `--save Boot.sims3 --checkpoint-save Join.sims3` | `Join.sims3` |
| `SIMSONLINE_CHECKPOINT_SAVE=Join.sims3`, no CLI checkpoint flag | `Join.sims3` |
| `--save Boot.sims3 --checkpoint-save Cli.sims3` and environment says `Environment.sims3` | `Cli.sims3` |

`CheckpointSave.ConfigDefaultEnvironmentAndCli` pins this precedence. `CheckpointSave.JoinWritesSeparateSlotAndLastGood` verifies that the boot save remains byte-for-byte unchanged when a separate checkpoint slot is used.

Join reuse is another, independent choice. `JoinReuseEnabled` is true unless `SIMSONLINE_JOIN_REUSE` is exactly `"0"`; its default therefore permits the optimization. With reuse disabled, `CheckpointWorldAsync` goes directly to `CheckpointFreshAsync`. With reuse enabled, a fingerprint scan decides whether the cached `ReusableWorld` still corresponds to the current world. `JoinWorldReuseTests.cs` covers the decision policy. Do not describe this as saving less often for durable backup: it is reusing transfer bytes for a joining client after state comparison.

A public save clears `_reusableWorld` first. Only a public save matching the configured checkpoint slot can seed subsequent reuse. `CheckpointSave.PublicSavesRefreshOrInvalidateJoinCache` tests saves to the boot, join, and manual slots; after the public save, the next join must not return older cached file bytes.

## Fresh checkpoint sequence

The host protects both a public save and a join checkpoint with `_saveGate`, a `SemaphoreSlim(1, 1)`. A fresh checkpoint arms a join fingerprint when enabled, calls `SaveGameCoreAsync`, reads the saved directory, checks whether the expected files are accessible, and retains last-good bytes asynchronously. The returned `WorldSave` is created from the post-save files, not from an earlier in-memory snapshot.

A simplified observed sequence is:

```text
CheckpointWorldAsync
  acquire _saveGate
  if reuse disabled: CheckpointFreshAsync
  else: decide by fingerprint; return reusable bytes only on a hit
  fresh path: arm fingerprint -> SAVE_GAME -> ReadFolder -> KeepReusableWorld
  retain LastGoodSave copy -> return WorldSave
  release _saveGate
```

The implementation has retry handling for save refusals that mean â€œnot ready yetâ€; arbitrary save errors are not silently reclassified as readiness. The readiness predicate `IsNotReadyToSave` is in `HeadlessSims3Host.cs`, and test `CheckpointSave` cases exercise retry behavior through the fake host. The game save still runs in the actual game process, so these unit tests simulate bridge replies rather than execute Sims 3 serialization.

## Full save-transfer payload example

The following JSON shows the field shape of a single protocol event after serialization. The `data` value is base64 as required by JSON encoding of `byte[]`; this small byte sample is illustrative of shape, not an actual save file.

```json
{
  "$type": "world_file_chunk",
  "folderName": "Neighborhood.sims3",
  "fileIndex": 0,
  "fileCount": 1,
  "fileName": "Meta.data",
  "offset": 0,
  "fileLength": 3,
  "data": "AQID"
}
```

`ServerEvent.WorldFileChunk` is the record type. `WorldSave.ToChunks` emits one event per file range. A real payload can involve multiple files and multiple chunks per file; receivers must use `fileIndex`, `offset`, and `fileLength`, not arrival order alone, to validate the stream.

| Receiver check | Why it matters |
|---|---|
| Folder name ends in `.sims3` | Prevent writing outside the game's save-folder naming scheme |
| File index increments by one | Prevent a later file being interleaved before the current file completes |
| First chunk of a file has offset 0 | Detect missing leading bytes |
| Next offset equals current stream position | Detect duplication, gap, or reorder |
| File name contains no invalid path characters | Keep transfer-relative names within the staging folder |
| Marker file is reserved | Avoid data overwriting the ownership marker |
| Destination already has writer marker | Protect a user's unrelated same-name save |

`WorldSaveWriter.Add` catches any failure, calls `Abort`, and rethrows. `Abort` disposes the active stream and tries to recursively delete staging; an `IOException` while deleting is swallowed because cleanup failure must not mask the original validation/write failure. It does not delete a destination save until final promotion.

## Readiness is more than file arrival

The transfer writer's `IsComplete` only means the declared file set was promoted. The next stage is the game load lifecycle. The controller's `LoadSaveViaMenuAsync` uses the normal menu path and invalidates reusable checkpoint data before loading. A `WorldFileChunk` stream completing cannot prove that the loaded game reached its normal in-world readiness event.

When diagnosing a join failure, keep these boundaries separate:

| Last confirmed boundary | Failure candidates after it |
|---|---|
| Server save returned successfully | Structural save validation, read-folder issue, recovery-copy failure |
| `WorldSave` was built | Event routing, serializer/write framing |
| Client accepted all chunks | Marker/promotion, game save discovery |
| Save appeared in menu | Load transition, missing/corrupt world data |
| Client entered world | Snapshot/identity bootstrap, presentation baseline |

A save might have copied all bytes but still reference data unavailable in the user's game installation. Do not publish save payload bytes or user world files as bug attachments; use a generated test fixture when demonstrating the chunk schema.

## Recovery result distinctions

`LastGoodSave.EnsureBootable` returns a `BootCheck(Restored, Detail)`. Consumers should preserve the distinction between `Restored == true` and a diagnostic that merely says a backup appears whole. The code restores only its manifest-verified `.lastgood` folder. A native `.backup` may be mentioned in `Detail` as a manual candidate; this is not an automatic restore.

| Situation | `Restored` | Expected consequence |
|---|---:|---|
| Main folder structurally whole and matches manifest | false | Load the main folder unchanged |
| Main folder structurally whole but differs from last-good | false | Leave it unchanged as a possible external/manual save |
| Main folder missing and verified copy exists | true | Restore the copy |
| Main folder incomplete and verified copy exists | true | Move incomplete folder aside, then restore |
| Main folder incomplete and copy invalid/missing | false | Report not whole; do not claim a restore |

This policy protects a valid save made outside the controller's last-good flow from being rolled back just because it differs from a retained checkpoint.

## What the tests prove

`LastGoodSaveTests.cs` uses `FakeDbpf` to create a DBPF-shaped file with a `DBPF` header, index count, index size and index start. It verifies:

- truncating the `.nhd` causes the computed index end to exceed the file length;
- removing the `.nhd` yields `no .nhd file`;
- zero-length files are rejected;
- an index size inconsistent with the record count is rejected;
- a manifest includes names with spaces and a 64-character SHA-256;
- an altered retained file fails verification;
- a cut save is moved aside and the verified copy restored;
- a whole but different target is preserved;
- a valid native `.backup` is named but not restored automatically.

`CheckpointSaveTests.cs` verifies controller sequencing using a pinned fake bridge. Neither test launches the commercial game. For engine semantic correctness, use a legally installed game and a disposable save; document game build, save origin, and the behavior inspected, without distributing the save itself.

## Transfer lifecycle details

A world transfer has several separate completion states. In the server host, a checkpoint creates or reuses a native save, then `WorldSave` enumerates files and produces metadata/chunks. In the client, the writer verifies the stream and stages files under an `.incoming` directory. Only after the last expected file and chunk are present does it promote the staging directory to the requested save name.

The `WorldSave` file record includes relative path and bytes; the client writer rejects unsafe paths, unexpected order, nonsequential offsets, inconsistent total length and checksum failures. A chunk size of `256 * 1024` bounds each chunk event. The chunk size is transport granularity; it does not cap total save size to 256 KiB.

| Transition | Required evidence |
|---|---|
| Game -> saved files | Host save response indicates native save completed |
| Files -> `WorldSave` | Folder enumeration finished and file metadata is stable |
| `WorldSave` -> chunk events | Files split into ordered chunks with declared length and hash |
| Chunk events -> `.incoming` | Every file chunk is sequential and in bounds |
| `.incoming` -> target save | All declared files complete; ownership marker check passes |
| Target save -> loaded game | Local game completes normal load path |

A transport success can end at any transition. For example, a clean chunk transfer does not prove that the local game successfully loaded the files.

## Checkpoint reuse and mutation detection

`HeadlessSims3Host.CheckpointWorldAsync` serializes save operations with `_saveGate`. Reuse is an optimization for joiners: the host can reuse a previous checkpoint when policy, requested slot and fingerprint conditions allow. The `SIMSONLINE_JOIN_REUSE=0` switch disables reuse. When the game state changes or the save fingerprint is no longer suitable, a fresh checkpoint is required.

A reuse decision should be evaluated against the current saved-folder identity and expected save slot, not only a file timestamp. A same-named save may be a different save generation. `CheckpointSaveTests.cs` covers slot selection, fresh checkpoint and invalidation cases.

## Last-good recovery mechanics

`LastGoodSave` retains a copied known-good save and checks structural completeness before using it. The validator requires the expected `.nhd` file and inspects DBPF index bounds; copy/restore includes byte hashes so a partial or changed copy is detected. Recovery restores a verified last-good directory only when the target fails the structural check.

This mechanism is intentionally conservative:

- It does not repair individual package records.
- It does not reinterpret or rewrite unknown DBPF resources.
- It does not prove the save's Sim graph or household data are semantically consistent.
- It does not bless a structurally valid but incompatible game version.
- It is not a substitute for keeping an independent backup before destructive testing.

A cold load remains the authoritative semantic test. If a cold load fails, preserve the failed copy for diagnosis and restore only a verified known-good save.

## Recovery failure table

| Condition | Safe conclusion |
|---|---|
| `.nhd` absent | Required save structure incomplete; do not promote as complete |
| DBPF index points past end of file | Structural corruption detected |
| Backup copy hash differs | Backup copy was not verified; do not restore it blindly |
| Structural validation passes | Byte/layout constraints pass only |
| Game cold load passes | This save loaded in this game build; not every future edit is safe |
| Household import returns success | Import API returned; no durable-load claim follows |
| Save copied after transfer | Transfer data is present; load has not yet been established |

## Save concurrency and failure cases

The host save gate prevents two checkpoint operations from competing for the same host state. The writer's `.incoming` folder separates incomplete transfer files from an existing target. The ownership marker prevents accidental replacement of a same-named user-created save folder. These safeguards are independent: a valid server transfer still should not overwrite a client folder that lacks the expected marker.

If a transfer is interrupted, the staging directory may remain incomplete. It should not be loaded or treated as a save. A retry should start a fresh incoming transfer or explicitly validate and resume according to `WorldSaveWriter` state; never concatenate chunks from two generations because matching paths alone do not prove a shared checkpoint.

The protocol must preserve ordering among `WorldFileStarted`, `WorldFileChunk`, and `WorldFileCompleted` events. A duplicate chunk, wrong offset, truncated final chunk, unexpected file, or declared-size mismatch is a failed transfer. The client should keep the old target intact until promotion succeeds.

## Operational checklist

Before a checkpoint:

1. Confirm the authoritative host is in the intended world and save slot.
2. Resolve client-native reconciliation requirements, or explicitly select the risk-accepting policy.
3. Ensure no concurrent checkpoint is underway.
4. Check whether join reuse is enabled and whether its fingerprint is still valid.
5. Keep a separate backup before destructive or experimental imports.

Before a client load:

1. Confirm the server save transfer reached final file completion.
2. Verify target folder is owned by the transfer marker if replacing an existing name.
3. Confirm no `.incoming` partial transfer is being mistaken for the final folder.
4. Load through the game normal path and wait for the world-ready transition.
5. Capture game version/build and load result for any compatibility report.

After an experimental import:

1. Save to a disposable slot, never the only known-good save.
2. Exit the game completely to force cold-load reconstruction.
3. Load the saved slot and inspect all affected Sims, relationships, inventories and object references.
4. Repeat a second save/reload cycle to catch stale object handles or deferred serialization errors.
5. Reject the resulting save if any required actor, relationship or inventory reference is missing.

## Tests and verification scope

`DedicatedServer.Tests/Program.cs` covers chunking and file reading; `DedicatedServer.Tests/Program.cs` covers staged writes and promotion; `DedicatedServer.Tests/LastGoodSaveTests.cs` covers structural backup verification/restoration; `DedicatedServer.Tests/CheckpointSaveTests.cs` covers checkpoint selection/reuse policy.

These tests use generated files and synthetic host behavior. They prove protocol and filesystem safeguards, not the native game engine's semantic load behavior. A cold-load test with a disposable Sims 3 save is necessary for semantic validation.
