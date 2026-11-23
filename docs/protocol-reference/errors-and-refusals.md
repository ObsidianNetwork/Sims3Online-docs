# Errors and refusals

The line-v1 codec uses a closed `RefusalCode` enum for `ERR` frames. The JSON listener reports `command_rejected` for a decoded command refusal. For a malformed command within the line limit, including one with an unknown discriminator, it sends `command_rejected` if it can recover the `commandId` and `server_error` otherwise. It continues reading the connection in either case. If a handler throws, the listener sends `command_rejected` with the command ID, followed by `server_error`. A JSON `reason` is free text and should not be parsed as a line-v1 code.

## Line-v1 error frame

```text
ERR|<corr>|<code>|<Verb>|<hexText>
```

`corr=0` is used for a pre-command error such as hello rejection. `hexText` is lowercase hex of UTF-8. `PickFailed` is a special asynchronous line-v1 event kind for late failures of a pick; it is not one of the JSON `ServerEvent` discriminators.

| Wire token | Enum member | Meaning |
|---|---|---|
| `UnauthorizedClientInsertion` | `UnauthorizedClientInsertion` | The authority kernel refused a client-side insertion. |
| `ClientWorldServiceSuppressed` | `ClientWorldServiceSuppressed` | Client-side world service was suppressed by authority policy. |
| `native_oracle_owned` | `NativeOracleOwned` | A verb is still owned by the native oracle and cannot be forwarded. |
| `SlotAlreadyReserved` | `SlotAlreadyReserved` | The requested Sim/slot is already reserved. |
| `hello_rejected` | `HelloRejected` | A well-formed hello failed a version/capability gate; `hexText` contains a closed detail token. |
| `bad_line` | `BadLine` | Frame, UTF-8, identifier, or capability syntax was invalid or unsupported. |
| `ClientReselectForbidden` | `ClientReselectForbidden` | A client-role connection attempted a forbidden reselection. |
| `NotJoined` | `NotJoined` | The command requires a joined public connection. |
| `WrongPlayer` | `WrongPlayer` | The command's `playerId` does not match the joined connection. |
| `Superseded` | `Superseded` | A newer connection owns this player's lease. |
| `ClientNativeModesOff` | `ClientNativeModesOff` | Lot export is unavailable when client-native modes are disabled. |
| `IdSpaceFull` | `IdSpaceFull` | The injective public-ID to kernel-ID map has no free entry. |
| `TestFailed` | `TestFailed` | The server-side interaction test refused the pick; optional text may be a tooltip. |
| `ObjectInUse` | `ObjectInUse` | An accepted pick later failed because the target object was in use. |
| `RouteFailed` | `RouteFailed` | An accepted pick could not complete its route. |
| `FailedToStart` | `FailedToStart` | An accepted pick did not start. |
| `NotRun` | `NotRun` | An accepted pick never ran. |
| `line_too_long` | `LineTooLong` | Public line content exceeded 4,194,304 bytes. |
| `bulk_malformed` | `BulkMalformed` | A bulk ID, chunk sequence, size, or typed payload was invalid. |
| `SimNotFound` | `SimNotFound` | The acting Sim could not be resolved before the interaction test. |
| `TargetNotFound` | `TargetNotFound` | The target could not be resolved before the interaction test. |
| `NoMatchingInteraction` | `NoMatchingInteraction` | No matching interaction entry was found. |
| `PickedObjectNotFound` | `PickedObjectNotFound` | The object selected by the user pick could not be resolved. |
| `InteractionNotFound` | `InteractionNotFound` | A cancellation referred to no queued or running interaction. |
| `queue_rejected` | `QueueRejected` | The server interaction queue rejected the request. |
| `dialog` | `Dialog` | The dialog path refused the supplied answer. |

Only seven members use lower-snake wire tokens: `NativeOracleOwned`, `HelloRejected`, `BadLine`, `QueueRejected`, `Dialog`, `LineTooLong`, and `BulkMalformed`. All other tokens retain the PascalCase C# enum name. The names are defined in `SimsOnline.Protocol\PublicLineCodecV1.cs` and round-trip tested by `DedicatedServer.Tests\PublicLineCodecTests.cs`.

## Hello details and framing failures

The front door has exactly these well-formed-hello detail tokens (`hexText` is UTF-8 encoded):

| Detail token | Cause |
|---|---|
| `version_unsupported` | The HELLO version is not 1. |
| `json_hello` | A JSON object was sent as the first line to the line-v1 endpoint. |
| `legacy_sidecar_hello` | A `CONNECT` line or the legacy seven-field sidecar HELLO was sent to the public endpoint. |
| `MissingSimlinkRing` | The required `ring=1` capability is absent. |
| `MissingWorldFromServer` | The required `wfs=1` capability is absent. |

A rejected hello's `hexText` is the UTF-8-hex encoding of its detail token. The code distinguishes a well-formed hello that fails the gate from a malformed frame; do not map every connection close to `hello_rejected`. The line reader rejects embedded separators/newlines, strict UTF-8 errors, bad numbers, unknown command/event kinds, invalid typed fields, or malformed bulk references.

The JSON listener uses `server_error` with a message such as `Command line exceeds ... Closing the connection.` for an overlong input line and then closes that connection. A decoded command denied by identity, role, or session policy is instead a `command_rejected` result. `command_accepted` means the command handler accepted the request; an interaction may still later fail and produce an interaction failure event.

## Menu refusal compatibility

The `Rf` recorder form retains backward compatibility with older menu-refusal lines. Six-field form decodes as `TestFailed` with empty tooltip. Seven-field form treats the last field as the historical hex tooltip. The eight-field public form carries refusal code and tooltip separately; a clear sets the refusal flag to `0` and leaves code/text empty. `DecodeRecorderMenuRefusal` deliberately aliases the compatible decoder. Never reinterpret a seven-field tooltip as a new refusal code.

Related pages: [handshake and resume](handshake-and-resume.md), [command catalogue](command-catalog.md), [transport limits](transport-limits.md).
