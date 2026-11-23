# Server event catalogue

Every event discriminator registered for the public JSON decoder, with constructor fields and a structural example. `SpeedIntentCaptured` exists as an internal nested record but is deliberately absent from the `JsonDerivedType` registrations, so it is not a public event discriminator.

Source of truth: `SimsOnline.Protocol\ServerEvent.cs` plus `SimsOnline.Protocol\PublicProtocolCodec.cs`. The catalogue below is derived from each `[JsonDerivedType]` registration and positional record constructor; field order is declaration/JSON order. JSON examples are illustrative, omit nullable members set to null, and use safe sample identifiers.

## `session_joined` - `SessionJoined`

`playerId` (int); `displayName` (string); `notice` (string?); `capabilities` (IReadOnlyList<string>?).

```json
{"$type":"session_joined","playerId":7,"displayName":"Alice"}
```

## `command_accepted` - `CommandAccepted`

`commandId` (Guid).

```json
{"$type":"command_accepted","commandId":"11111111-1111-4111-8111-111111111111"}
```

## `command_rejected` - `CommandRejected`

`commandId` (Guid); `reason` (string).

```json
{"$type":"command_rejected","commandId":"11111111-1111-4111-8111-111111111111","reason":"example refusal"}
```

## `sim_claimed` - `SimClaimed`

`playerId` (int); `simObjectId` (ulong); `lotId` (ulong?).

```json
{"$type":"sim_claimed","playerId":7,"simObjectId":7}
```

## `sim_released` - `SimReleased`

`playerId` (int); `simObjectId` (ulong).

```json
{"$type":"sim_released","playerId":7,"simObjectId":7}
```

## `interaction_queued` - `InteractionQueued`

`actorObjectId` (ulong); `interactionId` (ulong); `interactionKey` (string); `targetObjectId` (ulong?); `targetPosition` (Vector3Dto?); `parameters` (IReadOnlyDictionary<string, string>?).

```json
{"$type":"interaction_queued","actorObjectId":7,"interactionId":7,"interactionKey":"Terrain.GoHere"}
```

## `interaction_replay_queued` - `InteractionReplayQueued`

`bundle` (InteractionReplayBundle).

```json
{"$type":"interaction_replay_queued","bundle":{"version":7,"actorObjectId":7,"interactionId":7,"interactionKey":"Terrain.GoHere","isTransitionInteraction":false,"queueRole":"unknown","parameters":{"example":"value"}}}
```

## `interaction_started` - `InteractionStarted`

`actorObjectId` (ulong); `interactionId` (ulong).

```json
{"$type":"interaction_started","actorObjectId":7,"interactionId":7}
```

## `interaction_completed` - `InteractionCompleted`

`actorObjectId` (ulong); `interactionId` (ulong).

```json
{"$type":"interaction_completed","actorObjectId":7,"interactionId":7}
```

## `interaction_failed` - `InteractionFailed`

`actorObjectId` (ulong); `interactionId` (ulong); `reason` (string).

```json
{"$type":"interaction_failed","actorObjectId":7,"interactionId":7,"reason":"example refusal"}
```

## `snapshot_delta` - `SnapshotDelta`

`snapshot` (SimulationSnapshot); `forceFull` (bool).

```json
{"$type":"snapshot_delta","snapshot":{"tick":7,"worldName":"Sunset Valley","saveName":"Online.sims3","speed":"normal","sims":[],"objects":[]},"forceFull":false}
```

## `full_snapshot` - `FullSnapshot`

`snapshot` (SimulationSnapshot).

```json
{"$type":"full_snapshot","snapshot":{"tick":7,"worldName":"Sunset Valley","saveName":"Online.sims3","speed":"normal","sims":[],"objects":[]}}
```

## `game_speed_changed` - `GameSpeedChanged`

`speed` (GameSpeed).

```json
{"$type":"game_speed_changed","speed":"normal"}
```

## `world_time_changed` - `WorldTimeChanged`

`time` (WorldTime).

```json
{"$type":"world_time_changed","time":{"day":0,"hours":0,"minutes":0}}
```

## `save_checkpointed` - `SaveCheckpointed`

`saveName` (string); `at` (DateTimeOffset).

```json
{"$type":"save_checkpointed","saveName":"Online.sims3","at":"2026-01-02T03:04:05+00:00"}
```

## `server_error` - `ServerError`

`message` (string).

```json
{"$type":"server_error","message":"Example server message"}
```

## `client_role_changed` - `ClientRoleChanged`

`playerId` (int); `role` (PlayerRole).

```json
{"$type":"client_role_changed","playerId":7,"role":"client"}
```

## `sim_autonomy_changed` - `SimAutonomyChanged`

`simObjectId` (ulong); `disabled` (bool).

```json
{"$type":"sim_autonomy_changed","simObjectId":7,"disabled":false}
```

## `intent_captured` - `IntentCaptured`

`actorObjectId` (ulong); `targetObjectId` (ulong); `interactionKey` (string); `targetPosition` (Vector3Dto?).

```json
{"$type":"intent_captured","actorObjectId":7,"targetObjectId":7,"interactionKey":"Terrain.GoHere"}
```

## `lot_local_authority_mode_changed` - `LotLocalAuthorityModeChanged`

`playerId` (int); `mode` (LotLocalAuthorityMode).

```json
{"$type":"lot_local_authority_mode_changed","playerId":7,"mode":"fullPuppetFallback"}
```

## `lot_export_requested` - `LotExportRequested`

`playerId` (int); `lotId` (ulong?); `reason` (string).

```json
{"$type":"lot_export_requested","playerId":7,"reason":"example refusal"}
```

## `lot_reconcile_acked` - `LotReconcileAcknowledged`

`playerId` (int); `lotId` (ulong); `version` (long).

```json
{"$type":"lot_reconcile_acked","playerId":7,"lotId":7,"version":7}
```

## `sim_skill_changed` - `SimSkillChanged`

`simObjectId` (ulong); `skillName` (string); `level` (int).

```json
{"$type":"sim_skill_changed","simObjectId":7,"skillName":"Cooking","level":7}
```

## `household_funds_changed` - `HouseholdFundsChanged`

`householdId` (ulong); `funds` (int).

```json
{"$type":"household_funds_changed","householdId":7,"funds":7}
```

## `sim_motive_changed` - `SimMotiveChanged`

`simObjectId` (ulong); `motiveName` (string); `value` (float).

```json
{"$type":"sim_motive_changed","simObjectId":7,"motiveName":"Hunger","value":1.25}
```

## `sim_state_changed` - `SimStateChanged`

`simObjectId` (ulong); `motives` (IReadOnlyList<float>); `buffs` (IReadOnlyList<SimBuffDto>).

```json
{"$type":"sim_state_changed","simObjectId":7,"motives":[],"buffs":[]}
```

## `sim_spawned` - `SimSpawned`

`simObjectId` (ulong); `simDescriptionId` (ulong); `position` (Vector3Dto).

```json
{"$type":"sim_spawned","simObjectId":7,"simDescriptionId":7,"position":{"x":1.25,"y":1.25,"z":1.25}}
```

## `sim_despawned` - `SimDespawned`

`simObjectId` (ulong); `simDescriptionId` (ulong).

```json
{"$type":"sim_despawned","simObjectId":7,"simDescriptionId":7}
```

## `sim_description_data` - `SimDescriptionData`

`simDescriptionId` (ulong); `propertyStream` (byte[]); `resKeyTable` (byte[]); `objectIdTable` (byte[]).

```json
{"$type":"sim_description_data","simDescriptionId":7,"propertyStream":"AQID","resKeyTable":"AQID","objectIdTable":"AQID"}
```

## `sim_details_changed` - `SimDetailsChanged`

`simObjectId` (ulong); `careerGuid` (ulong); `careerLevel` (int); `careerBranch` (string); `funds` (int); `skills` (string).

```json
{"$type":"sim_details_changed","simObjectId":7,"careerGuid":7,"careerLevel":7,"careerBranch":"General","funds":7,"skills":"-"}
```

## `world_file_chunk` - `WorldFileChunk`

`folderName` (string); `fileIndex` (int); `fileCount` (int); `fileName` (string); `offset` (long); `fileLength` (long); `data` (byte[]).

```json
{"$type":"world_file_chunk","folderName":"Online.sims3","fileIndex":0,"fileCount":7,"fileName":"world.nhd","offset":0,"fileLength":7,"data":"AQID"}
```

## `sim_animation` - `SimAnimation`

`lines` (IReadOnlyList<string>).

```json
{"$type":"sim_animation","lines":["1|start|123|456"]}
```

## `user_intent_captured` - `UserIntentCaptured`

`actorObjectId` (ulong); `targetObjectId` (ulong); `definitionType` (string); `name` (string); `path` (string); `hit` (string); `hitPoint` (Vector3Dto?); `picked` (string).

```json
{"$type":"user_intent_captured","actorObjectId":7,"targetObjectId":7,"definitionType":"Terrain.GoHere","name":"Alice","path":"Terrain.GoHere","hit":"terrain","picked":"Go Here"}
```

## `cancel_intent_captured` - `CancelIntentCaptured`

`actorObjectId` (ulong); `interactionId` (ulong).

```json
{"$type":"cancel_intent_captured","actorObjectId":7,"interactionId":7}
```

## `leave_conversation_intent_captured` - `LeaveConversationIntentCaptured`

`actorObjectId` (ulong).

```json
{"$type":"leave_conversation_intent_captured","actorObjectId":7}
```

## `dialog_requested` - `DialogRequested`

`requestId` (int); `simObjectId` (ulong); `kind` (string); `args` (IReadOnlyList<string>).

```json
{"$type":"dialog_requested","requestId":7,"simObjectId":7,"kind":"Message","args":["Example"]}
```

## `dialog_answer_captured` - `DialogAnswerCaptured`

`requestId` (int); `simObjectId` (ulong); `answer` (string?).

```json
{"$type":"dialog_answer_captured","requestId":7,"simObjectId":7}
```

## Reading the examples

- C# `ulong` identifiers are JSON integer literals; clients should not coerce them through IEEE-754 doubles.
- `byte[]` values are base64 strings in JSON. Enums use camel-case strings, and object/dictionary keys use camel case.
- Nullable values shown as absent are omitted by the codec (`WhenWritingNull`); empty strings, empty arrays and empty objects are distinct values.

Related pages: [JSON wire format](wire-formats.md), [handshake and resume](handshake-and-resume.md), [errors and refusals](errors-and-refusals.md).
