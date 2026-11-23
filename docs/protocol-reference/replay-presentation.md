# Replay, input, and presentation

Interaction events describe simulation work; `SimAnimation` carries presentation that should be replayed as animation, not re-executed as a new interaction. Client-captured inputs are represented as events for shared decoding, then converted back into authoritative commands by the client controller.

## Replay bundle

`InteractionReplayBundle` is versioned independently from the public protocol. Its current version is 1. Its positional fields are:

| Field | Type | Role |
|---|---|---|
| `Version` | int | Bundle format version |
| `ActorObjectId`, `InteractionId` | ulong | Server-side actor and interaction IDs |
| `InteractionKey` | string | Interaction definition key |
| `InteractionInstanceType`, `InteractionDefinitionType` | string? | Optional type identity for replay diagnostics/translation |
| `TargetObjectId` | ulong? | Optional target |
| `TargetObjectType` | string? | Optional target type label |
| `TargetPosition` | Vector3Dto? | Optional target point |
| `PriorityLevel` | string? | Captured queue priority |
| `IsUserDirected`, `IsAutonomous` | bool? | Optional captured origin flags |
| `IsTransitionInteraction` | bool | Whether it is a transition entry |
| `QueueRole` | InteractionQueueRole | Unknown, Head, Transition, Continuation, Current, or Running |
| `PostureType`, `PostureContainerObjectId` | string?, ulong? | Captured posture context |
| `HostTick` | long? | Tick at capture |
| `Parameters` | IReadOnlyDictionary<string,string> | Semantic replay parameters |
| `PolicyHint` | ReplayPolicy? | NeverReplay, ReplayExact, ReplayAsRoute, ForceAlign, or SlideOnly |

`InteractionReplayQueued(Bundle)` is the legacy interaction-replay event. It preserves semantic parameters such as book/progress/seat data instead of treating every queued interaction as an identical click.

## Forwarding client intent

`IntentCaptured` is a basic actor/target/key/position intent. `UserIntentCaptured` retains the menu-pick values `DefinitionType`, `Name`, `Path`, `Hit`, optional `HitPoint`, and `Picked`; `ToRequest()` maps them into a `QueueInteraction` request. The `Parameters` keys that identify the user pick are `ui.name`, `ui.path`, `ui.hit`, and `ui.picked`. A `TargetObjectId` of zero means terrain in the captured form and becomes null in the request.

`CancelIntentCaptured` becomes `CancelInteraction` with server IDs. `LeaveConversationIntentCaptured` becomes `LeaveConversation`. `SpeedIntentCaptured` is an internal record, not a registered public event: the client controller turns it into `SetGameSpeed` rather than sending it as an event discriminator.

## Presentation stream and capability

`SimAnimation.Lines` contains recorder lines in `seq|kind|...` form. Sequence zero can mark historical lines; live lines are sequenced. The server chunks line lists at `PresentationStream.MaxLinesPerEvent` before creating events. When `session_joined.capabilities` contains `presentation_stream`, the client disables legacy interaction replay immediately rather than waiting until its first `sim_animation` event. If the capability is absent or unknown, the client retains legacy behavior for compatibility.

The capability is advertised only when presentation streaming is enabled and a sync broadcast interval starts the broadcaster. `DedicatedServer.Tests\JoinCapabilityTests.cs` covers present, absent, and unknown capability behavior.

### Reassembling a line-v1 animation event

`SimsOnlineClient.LineHandshakeJoinBulkDeltaAndAnimation` sends a synthetic
`SimAnimation` with one recorder string, `1|S|2`. This excerpt uses the test's
two-chunk split of the encoded field:

```text
EV|3|5|1|SimAnimation|b3
BULK|3|0|2|WzMxN2M1
BULK|3|1|2|MzdjMzJd
```

| Wire part | Meaning |
|---|---|
| `EV|3|5|1|SimAnimation` | Event sequence 3, simulation tick 5, and join-command correlation 1 |
| `b3` | The `Lines` field comes from bulk ID 3 |
| `BULK|3|0|2` and `BULK|3|1|2` | Chunk indices 0 and 1 of two chunks for bulk ID 3 |
| `WzMxN2M1` and `MzdjMzJd` | Base64 chunks that assemble to the line-v1 list `[317c537c32]` |
| `317c537c32` | UTF-8 hex for the recorder string `1|S|2` |

The client decodes the `Lines` list once both chunks arrive and delivers one
`SimAnimation` event. The test checks that the event reaches client dispatch. Use the
[presentation scenarios](../presentation/presentation-scenarios.md) to check how a live
game applies the recorder data.

## Remote dialogs

`DialogRequested(RequestId, SimObjectId, Kind, Args)` goes to the owning player. `Kind` values documented by the server are `TwoButton`, `AcceptCancel`, `ThreeButton`, `StringInput`, and `Message`. The client displays the dialog in its own TS3 and sends `AnswerDialog`; a null `Answer` means keep the game's default. `DialogAnswerCaptured` is the input event decoded from the client bridge, not an additional command.

Related pages: [event catalogue](event-catalog.md), [command catalogue](command-catalog.md), [handshake and resume](handshake-and-resume.md).
