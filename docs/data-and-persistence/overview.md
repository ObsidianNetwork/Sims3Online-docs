# Persistence overview

The server game writes the authoritative native save. The controller reads the resulting save folder, retains a separately verified last-good copy, and can stream those files to a joining client. Other objects called â€œsnapshotâ€� or â€œreplayâ€� serve narrower purposes and must not be confused with a loadable world.

## Who owns what

| Component | Current responsibility | Relevant type or path |
|---|---|---|
| Host game | Executes the save operation and owns native save semantics | `SAVE_GAME`, dispatched by `SimsOnlineServerHostMod/ServerHostMain.cs` |
| Server controller | Serializes save/checkpoint requests, reads the save folder, and retains recovery bytes | `DedicatedServer/Authority/HeadlessSims3Host.cs` |
| Transfer model | Represents a save as named files and emits ordered file chunks | `WorldSave`, `WorldSaveFile`, `WorldSaveWriter` in `SimsOnline.Protocol/WorldSave.cs` |
| Recovery copy | Checks structural completeness, hashes copies, and restores a damaged target at startup | `LastGoodSave` in `DedicatedServer/Authority/LastGoodSave.cs` |
| Mirror state | Represents selected state at a tick | `SimulationSnapshot`, `SimSnapshot`, and `ObjectSnapshot` |
| Clock mirror | Carries an accepted day/hour/minute state to clients | `WorldTimeChanged` and `WorldTime` |

## State categories

### Durable native save

The native save is a set of files in the game's save folder. `WorldSave.ReadFolder` reads only the folder's direct files, sorts paths case-insensitively, and loads their bytes. The controller does not reconstruct the game's object graph. The public implementation boundary documented here is the `SAVE_GAME` bridge operation and the file set it produces.

### Last-good recovery copy

`LastGoodSave.Keep` validates the candidate, writes it into a staging directory, reads each file back to compare SHA-256, writes a manifest, and swaps the staged folder into the `.lastgood` name. A failed candidate is not allowed to replace a prior verified copy. At startup, `EnsureBootable` restores that copy only when the main folder is missing or structurally incomplete. If the main folder is structurally whole but differs from the controller copy, it is deliberately left alone: it may be a valid save made outside the checkpoint path.

### Live snapshot

`SimulationSnapshot` has a tick, world/save names, speed, Sim records, and object records. A snapshot is an observation used by the session/mirror flow, not a full serialization. Its Sim and object runtime IDs are scoped to a loaded game instance unless a separate identity is provided.

### Presentation and event history

`ServerEvent.SimAnimation` carries selected animation-record lines for client presentation. Its replay stream is neither a durable save nor a comprehensive action log. One-time effects and active animation drivers have different replay semantics; see the client presentation reference.

## Failure interpretation

| Observation | What it establishes | Next step |
|---|---|---|
| Engine save accepted | The bridge accepted the save operation | Inspect files and recovery result separately |
| `LastGoodSave.Problem` returns null | The file set passes the implemented structural checks | Cold-load a disposable copy for semantic validation |
| Manifest verification passes | The retained files match recorded lengths and SHA-256 values | Still validate the game load path if semantics changed |
| Client received every chunk | Transfer data reached the local writer | Check `WorldSaveWriter.IsComplete` and the normal load transition |
| Snapshot fields match | Selected values agree at the sampled ticks | Do not infer omitted object, queue, inventory, or interaction state |

## Code and tests

- `SimsOnline.Protocol/WorldSave.cs` contains file enumeration, chunk production, staging, and promotion.
- `DedicatedServer/Authority/LastGoodSave.cs` contains structural validation, hashing, verification, and startup restore behavior.
- `DedicatedServer/Authority/HeadlessSims3Host.cs` sequences checkpoint operations and optional join reuse.
- `SimsOnline.Protocol/SimulationSnapshot.cs` defines the typed snapshot fields.
- `DedicatedServer.Tests/LastGoodSaveTests.cs` covers cut/missing files, manifest verification, and restore.
- `DedicatedServer.Tests/CheckpointSaveTests.cs` covers save-slot and join-checkpoint behavior.

## Limits

The public controller does not include the full game serializer. DBPF checks verify index placement and record-count sizing; they do not parse every record or prove that the game will load the save. Hashes prove that the recovery copy matches bytes written from the candidate, not that those bytes are good. A world transfer can be complete but still fail during the game's load lifecycle.

## Related pages

- [Save/load pipeline](save-load-pipeline.md)
- [Snapshot fields](snapshot-fields.md)
- [World clock](world-clock.md)
- [Protocol](../protocol.md)

## Applicability guide

| Question | Start here |
|---|---|
| Do I need the client to load the authoritative world? | [Save/load pipeline](save-load-pipeline.md) |
| Am I trying to transfer household or Sim inventory state? | [Household export/import](household-export-import.md), then [Inventory persistence](inventory-persistence.md) |
| Does a client-native Sim need to reconcile before save? | [Lot reconciliation](lot-reconciliation.md) |
| Is a field visible in current protocol snapshots? | [Snapshot fields](snapshot-fields.md) |
| Does the displayed day/time differ between processes? | [World clock](world-clock.md) |

## Do not conflate these operations

A checkpoint is written by the host game; a world transfer copies its native files; a snapshot reports selected current state; reconciliation applies a limited subset of client-native state; inventory membership is a presentation set; and household deep export is a bounded payload whose import is quarantined. Each has different completion criteria.

The central operational rule is to name the artifact and the evidence level. “The client received a snapshot” is not “the client save is current.” “The server merged a lot export” is not “the household was imported.” “The DBPF structure validates” is not “the game cold-loads.”

## Relevant code map

- Protocol DTOs: `SimsOnline.Protocol/WorldSave.cs`, `SimulationSnapshot.cs`, `LotExport.cs`, and `ServerEvent.cs`.
- Save orchestration and recovery: `DedicatedServer/Authority/HeadlessSims3Host.cs`, `LastGoodSave.cs`, and `AuthoritativeSession.cs`.
- Client transfer writer: `SimsOnline.Protocol/WorldSaveWriter.cs`.
- Reconciliation: `DedicatedServer/Mirroring/LotReconcile.cs`.
- Host household/inventory paths: `DedicatedServer/HostBridge/HouseholdExportTransport.cs` and `SimsOnlineServerHostMod/InventorySet.cs`.

The protocol and controller tests can establish framing, policy and filesystem safeguards. They cannot establish engine-level graph integrity without a live cold-load test.

## Decision table by task

| Task | Primary representation | Required validation |
|---|---|---|
| Join a client to the current world | `WorldSave` chunks | Full transfer, marker-safe promotion, normal game load |
| Observe current Sim positions | `SimulationSnapshot` | Identity resolution, scope, tick and mirror apply |
| Correct an outdated client baseline | `FullSnapshot` / `ForceFull` | Valid ack policy, backlog discard, applied-tick ack |
| Sync current server calendar | `WorldTimeChanged` | Range validation and week-relative conversion |
| Capture client-native position before save | `LotExportBundle` | Ownership/lot filter, `MERGE_LOT_STATE`, freshness gate |
| Persist arbitrary household graph | Native save lifecycle | Cold load and repeated-cycle validation |
| Recover from truncated save | `LastGoodSave` | Structural check and copied-byte verification |

## What “saved” means here

A successful host `SAVE_GAME` response means the game reported its save operation completed. A structurally valid folder means key files and DBPF index bounds look consistent. A transferred folder means the network writer received and promoted the expected bytes. A cold-load success means the game reconstructed the world in that tested build. These statements are progressively stronger but not interchangeable.

## Diagnostic routing

Start with the exact missing result:

- **Files or client load:** `save-load-pipeline.md` and `WorldSaveWriter`.
- **Selected state field or ID mismatch:** `snapshot-fields.md` and mirror alias handling.
- **Time drift:** `world-clock.md` and host clock conversion.
- **Client-owned position not in server save:** `lot-reconciliation.md`.
- **Household stream/import question:** `household-export-import.md` (deep import remains quarantined).
- **Item appears in one process only:** `inventory-persistence.md`.

The controller's test suite proves policy, framing and synthetic host operations. Engine object graph safety, UI appearance and disk durability require live game evidence.

## Maintainability boundary

Protocol DTOs describe information that can cross the network. The host bridge describes game-specific operations. The native save format remains game-owned. Keep those layers separated: do not add a snapshot field as an undocumented substitute for an import; do not make a bridge command authoritative without a session ownership gate; and do not claim a save is portable because a DTO serialized successfully.

## Boundary-oriented troubleshooting

| First observation | Read next | Avoid concluding |
|---|---|---|
| `SAVE_GAME` returned error | Host checkpoint and save-slot selection | The filesystem copy is necessarily corrupt |
| Save files exist but client does not load | `WorldSaveWriter` finalization and game load result | Snapshot protocol is at fault |
| Snapshot JSON looks correct but Sim is absent | Alias, spawn lifecycle, interest and self filtering | The server omitted the Sim |
| Position changes then snaps back | Local route/autonomy writer and active-order strip | Snapshot tick is necessarily stale |
| Household decoder accepts payload | Quarantine policy and import lifecycle | The payload is safely imported |
| Inventory line matches server | Client object mapping and inventory operation result | Item content/state is synchronized |
| Checkpoint waits for export | Client-native mode, freshness, request debounce | The server save is frozen indefinitely |
| Day/time differs after join | Clock range/conversion and startup ordering | Local timezone is the cause |

## Reading test results correctly

A unit test that uses a fake host can prove the controller called `MergeLotStateAsync`; it cannot prove the Sims 3 host applied the operation correctly. A bridge codec test can prove a command line was encoded; it cannot prove the host mod parsed it. A file writer test can prove bytes were promoted; it cannot prove the game cold-loaded them.

For every issue, identify the narrowest failing boundary and use the smallest test that exercises that layer. Then add a higher-level validation only when the claim requires it. This prevents a documentation page from overstating what a synthetic test actually establishes.

## Documentation and compatibility checklist

When adding a field or operation:

1. Update the owning protocol/bridge page and this overview's representation table if the persistence boundary changes.
2. Add exact source paths and type names rather than internal tracker references.
3. Include a small synthetic JSON/wire example where useful.
4. Identify default, absent and malformed behavior.
5. Name the relevant test and its scope.
6. State whether a native cold-load is required and whether it was performed.
7. Check local anchor links and folder index rows.

This section is deliberately a navigation guide; detailed field definitions, wire examples and lifecycle constraints remain on the linked topic pages.

## Artifact-specific acceptance gates

### Snapshot event

Accept as a current-state update only after public decode, session/source validation, client self/interest filtering, tick/coalescing logic, bridge translation and local mapping. If any stage transforms or drops records, diagnose against the transformed event rather than comparing only raw server JSON.

### Household export

Accept as a decoded candidate only after metadata, bulk length, CRC, segment boundaries and payload validator pass. Keep the candidate unapplied while deep-import quarantine is active. A valid decode does not imply the host created or updated a household.

### Native save transfer

Accept as a staged transfer only after all file/chunk ranges match their declarations and the ownership marker permits final promotion. Accept as a loaded world only after the normal game load completes. Accept as semantically verified only after the cold-load inspection needed for the claim.

### Recovery copy

Accept as recoverable only after structural checks and byte-copy verification pass. Do not equate it with a user backup strategy or compatibility check for another game build.

## Persistence review checklist

For a change in this section, check the following layers in order:

1. Is the data durable, current-state-only, transient, or a request?
2. Which process is authoritative for writing it?
3. What identity scope does each ID have?
4. What is the wire/bridge representation and size bound?
5. How does a client joining late obtain the state?
6. How does a missing or stale value clear/reset?
7. Can a partial transfer leave the prior state intact?
8. Which errors are structural versus semantic?
9. Which test proves the exact claim?
10. Is cold-load evidence required?

A strong test suite can still leave engine semantics unproven. State the boundary instead of hiding the limitation.

## Data contracts to keep separate

`WorldSave` consists of named native files and transfer metadata. `SimulationSnapshot` consists of typed selected state. `HouseholdExportPayload` consists of property streams and remap tables. `LotExportBundle` consists of a lot-scoped client observation and optional deep export. `InventorySet` is a compact live membership report. These records solve different problems and should not be nested into one generic “save state” type.

| Contract | Primary identifiers | Typical failure |
|---|---|---|
| `WorldSave` | relative file path, chunk offset | partial/corrupt transfer |
| `SimulationSnapshot` | runtime object id, optional Sim description id | stale/missing alias or delta gap |
| `HouseholdExportPayload` | household id and remap tables | size/CRC/semantic validation failure |
| `LotExportBundle` | player id, lot id, client tick | ownership, freshness or conflict rejection |
| `InventorySet` | owner object id and item ids | missing local item or stale membership |
| `WorldTimeChanged` | day/hour/minute | invalid range or wrong clock conversion |

## Durability ladder

1. **Observed:** a host or client process reported a value.
2. **Encoded:** a protocol/bridge DTO serialized successfully.
3. **Delivered:** the intended connection received the event or bytes.
4. **Applied:** the target game handler completed its operation.
5. **Saved:** the authoritative game wrote its native save.
6. **Cold-loaded:** the game reconstructed the save after process restart.
7. **Repeated-cycle stable:** a second save/reload preserved the required graph.

A feature should make claims no higher on this ladder than its evidence. Most controller unit tests establish levels 2–3 with fake hosts; selected bridge tests establish level 4; only live save/load tests establish the later levels.

## Data migration and reset boundaries

A new save name, a game-version upgrade, a new client process and a mirror resync are four different lifecycle events. The first can create a new native save; the second can trigger game-owned migration; the third changes process-local IDs; the fourth only reestablishes selected protocol state. Do not run a mirror reset as a substitute for a save migration, and do not assume reusing a save folder preserves runtime aliases.

When a world is reset or reloaded, clear cached snapshot aliases, per-owner inventory line caches, pending lot exports and animation driver mappings that belong to the old runtime. Each subsystem should reset at the lifecycle boundary it owns; stale caches can make new IDs appear to refer to old entities.

## Final evidence summary

- Transfer integrity: writer/file tests.
- Controller policy: dedicated server/session tests.
- Bridge framing and role: codec/host tests.
- Local runtime application: live bridge and game observation.
- Durable semantics: cold load and repeated reload of a disposable save.

A useful incident report names both the last verified boundary and the first failed boundary. That is more actionable than saying only “the data didn't persist.”

## Escalation by claim

| Claim | Minimum evidence |
|---|---|
| “Snapshot codec is compatible” | Pinned JSON round-trip |
| “Client received save bytes” | Complete file/chunk receipt |
| “Client world loaded” | Normal game load completion |
| “Client state converged” | Readback of the fields under test |
| “Household export is safe to decode” | Bounds, CRC and validator tests |
| “Household import is durable” | Not established by current controller tests; cold-load lifecycle required |
| “Inventory is synchronized” | Membership and object/content scope explicitly stated |

Use the narrowest supported statement and link the responsible page. If a claim depends on a live game build, record that build and avoid generalizing to untested saves.

## Shared vocabulary

- **Checkpoint:** native save operation on the authoritative host.
- **Transfer:** copying the resulting native files to a client.
- **Snapshot:** selected current-state DTOs, not a save.
- **Reconcile:** bounded client-native state merge before checkpoint.
- **Recovery:** restore a verified last-good native save copy.
- **Cold load:** reconstruct a world after the game process has exited.

Use these terms consistently in bug reports and docs.

The controller's test suite proves policy, framing and synthetic host operations. It cannot establish engine-level graph integrity without a live cold-load test.
