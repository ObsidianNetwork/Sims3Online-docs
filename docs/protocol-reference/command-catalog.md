# Client command catalogue

The public JSON command discriminator and field inventory. Each command carries `commandId` (`Guid`) and `playerId` (`int`) from the base `ClientCommand`; the two optional tracing properties `sentryTrace` and `sentryBaggage` are omitted when null.

Source of truth: `SimsOnline.Protocol\ClientCommand.cs` plus `SimsOnline.Protocol\PublicProtocolCodec.cs`. The catalogue below is derived from each `[JsonDerivedType]` registration and positional record constructor; field order is declaration/JSON order. JSON examples are illustrative, omit nullable members set to null, and use safe sample identifiers.

## `join_session` - `JoinSession`

`commandId` (Guid); `playerId` (int); `displayName` (string); `onScreen` (string?).

```json
{"$type":"join_session","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"displayName":"Alice"}
```

## `leave_session` - `LeaveSession`

`commandId` (Guid); `playerId` (int).

```json
{"$type":"leave_session","commandId":"11111111-1111-4111-8111-111111111111","playerId":7}
```

## `claim_sim` - `ClaimSim`

`commandId` (Guid); `playerId` (int); `simObjectId` (ulong).

```json
{"$type":"claim_sim","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"simObjectId":7}
```

## `release_sim` - `ReleaseSim`

`commandId` (Guid); `playerId` (int); `simObjectId` (ulong).

```json
{"$type":"release_sim","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"simObjectId":7}
```

## `set_active_sim` - `SetActiveSim`

`commandId` (Guid); `playerId` (int); `simObjectId` (ulong).

```json
{"$type":"set_active_sim","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"simObjectId":7}
```

## `queue_interaction` - `QueueInteraction`

`commandId` (Guid); `playerId` (int); `interaction` (InteractionRequest).

```json
{"$type":"queue_interaction","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"interaction":{"actorObjectId":7,"interactionKey":"Terrain.GoHere","parameters":{"example":"value"}}}
```

## `cancel_interaction` - `CancelInteraction`

`commandId` (Guid); `playerId` (int); `actorObjectId` (ulong); `interactionId` (ulong).

```json
{"$type":"cancel_interaction","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"actorObjectId":7,"interactionId":7}
```

## `leave_conversation` - `LeaveConversation`

`commandId` (Guid); `playerId` (int); `actorObjectId` (ulong).

```json
{"$type":"leave_conversation","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"actorObjectId":7}
```

## `answer_dialog` - `AnswerDialog`

`commandId` (Guid); `playerId` (int); `requestId` (int); `simObjectId` (ulong); `answer` (string?).

```json
{"$type":"answer_dialog","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"requestId":7,"simObjectId":7}
```

## `request_sim_description` - `RequestSimDescription`

`commandId` (Guid); `playerId` (int); `simDescriptionId` (ulong).

```json
{"$type":"request_sim_description","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"simDescriptionId":7}
```

## `request_world` - `RequestWorld`

`commandId` (Guid); `playerId` (int).

```json
{"$type":"request_world","commandId":"11111111-1111-4111-8111-111111111111","playerId":7}
```

## `set_game_speed` - `SetGameSpeed`

`commandId` (Guid); `playerId` (int); `speed` (GameSpeed).

```json
{"$type":"set_game_speed","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"speed":"normal"}
```

## `set_world_time` - `SetWorldTime`

`commandId` (Guid); `playerId` (int); `time` (WorldTime).

```json
{"$type":"set_world_time","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"time":{"day":0,"hours":0,"minutes":0}}
```

## `request_snapshot` - `RequestSnapshot`

`commandId` (Guid); `playerId` (int); `sinceTick` (long?).

```json
{"$type":"request_snapshot","commandId":"11111111-1111-4111-8111-111111111111","playerId":7}
```

## `set_client_role` - `SetClientRole`

`commandId` (Guid); `playerId` (int); `role` (PlayerRole).

```json
{"$type":"set_client_role","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"role":"client"}
```

## `save_game` - `SaveGame`

`commandId` (Guid); `playerId` (int); `saveName` (string).

```json
{"$type":"save_game","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"saveName":"Online.sims3"}
```

## `ack_snapshot` - `AckSnapshot`

`commandId` (Guid); `playerId` (int); `tick` (long).

```json
{"$type":"ack_snapshot","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"tick":7}
```

## `submit_lot_export` - `SubmitLotExport`

`commandId` (Guid); `playerId` (int); `bundle` (LotExportBundle); `clientLastReconciledVersion` (long).

```json
{"$type":"submit_lot_export","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"bundle":{"playerId":7,"lotId":7,"clientTick":7,"sims":[]},"clientLastReconciledVersion":7}
```

## Reading the examples

- C# `ulong` identifiers are JSON integer literals; clients should not coerce them through IEEE-754 doubles.
- `byte[]` values are base64 strings in JSON. Enums use camel-case strings, and object/dictionary keys use camel case.
- Nullable values shown as absent are omitted by the codec (`WhenWritingNull`); empty strings, empty arrays and empty objects are distinct values.

Related pages: [JSON wire format](wire-formats.md), [handshake and resume](handshake-and-resume.md), [errors and refusals](errors-and-refusals.md).
