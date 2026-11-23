# Wire formats and compatibility

Sims3Online has more than one message format. Each format has a distinct peer, version boundary, and failure behavior; a payload valid in one channel is not a valid frame in another.

## Channel map

| Channel | Framing and peer | Version mechanism | Code |
|---|---|---|---|
| Public JSON oracle | One UTF-8 JSON object per LF-terminated line between a client and `ClientListenerServer` | No protocol version field; `$type` is the discriminator | `SimsOnline.Protocol\PublicProtocolCodec.cs`, `DedicatedServer\Network\ClientListenerServer.cs` |
| Public line protocol v1 | LF-terminated UTF-8 fields separated by `|`, on the Rust public listener | `HELLO|1|...` and `WELCOME|1|...` | `SimsOnline.Protocol\PublicLineCodecV1.cs`, `rust\simlink_headless_server\src\wire_v1.rs` |
| Process-memory bridge | Fixed-size shared-memory slots containing short bridge lines and bulk references | Layout 11, bridge protocol 23, hello wire 6 | `DedicatedServer\HostBridge\BridgeSharedRegionLayout.cs`, `DedicatedServer\HostBridge\BridgeLineCodec.cs` |
| SimLink | Local shared-memory rings and named events between a game process and its controller | SimLink layout 1; native API version 15 | `SimLink\src\simlink_layout.h`, `DedicatedServer\HostBridge\SimLinkShadowReader.cs` |

The public line listener's version `1` does not negotiate the JSON oracle or memory bridge. The controller's sidecar handshake constants are also separate from the public `HELLO` version.

## JSON oracle encoding

`PublicProtocolCodec.JsonOptions` sets camel-case property names and dictionary keys, omits null-valued properties, writes compact JSON, and serializes enums as camel-case strings. `$type` names are the stable snake-case names registered with `JsonDerivedType`, for example `join_session` and `session_joined`.

```json
{"$type":"claim_sim","commandId":"11111111-1111-4111-8111-111111111111","playerId":7,"simObjectId":18446744073709551600}
```

`simObjectId` is a `ulong` JSON number. Consumers must preserve it as an integer; parsing through a binary64 `double` can round identifiers above 2^53. `byte[]` fields use base64 strings; `DateTimeOffset` uses an ISO 8601 string. Nulls are omitted, so absent nullable properties decode to null, whereas empty strings, lists, and dictionaries remain explicit values.

The decoder requires a known discriminator. An unknown `$type` causes a decode error. See [errors and refusals](errors-and-refusals.md) for the JSON listener's reply. A new optional/defaulted property is normally additive, but changing a field's name/type or changing a discriminator is breaking. JSON wire byte order is pinned by `DedicatedServer.Tests\WireJsonPinningTests.cs`.

## Line-v1 encoding

Line-v1 uses an explicit record grammar rather than JSON property names:

| Value | Line-v1 representation |
|---|---|
| Field separator | `|`; a field cannot contain literal `|`, CR, or LF |
| String | Lowercase hex of UTF-8; empty text is an empty field |
| Nullable absent | `~`; different from an empty string/list |
| Boolean | `0` or `1` |
| Enum | The C# enum member name, such as `Normal` |
| Record | `(field1;field2;...)` in positional constructor order |
| List | `[item1,item2,...]` |
| Dictionary | `[(hex-key;hex-value),...]` |
| Byte arrays | Hex inline; raw bytes when introduced as a bulk value |
| Floating point / time | Invariant round-trip representation; `DateTimeOffset` is UTF-8 hex of `O` format |

The public line codec has a 4,194,304-byte content limit (excluding LF) and validates UTF-8 strictly. One trailing CR before LF is accepted and removed. The same 4,194,304 count in the C# JSON reader is a character limit, not a byte limit; see [transport limits](transport-limits.md).

Command records carry `Guid` in the JSON oracle. On line-v1, a connection-local injective map converts GUIDs to `u64` command IDs without truncation. Retries reuse their mapped ID. A GUID whose top 64 bits are zero has a reversible direct representation; otherwise the caller must use `PublicLineCodecV1.CommandIds`.

## Compatibility limits

- Keep JSON discriminator spellings and field types stable. Unknown message discriminators are not forward-compatible.
- Line-v1 checks protocol version during `HELLO`; its positional fields are ordered by record constructor, so reorderings are wire-breaking even if JSON property names remain unchanged.
- Bridge `LayoutVersion`, `ProtocolVersion`, `HelloWireVersion`, and SimLink's native API version are independent. Bump and validate the version that belongs to the changed channel, not all of them indiscriminately.

## Failure modes

- A JSON object sent as the first line to line-v1 is not a hello and is refused.
- A line-v1 frame containing invalid UTF-8 or an embedded separator is rejected before typed decoding.
- A newly added JSON event type cannot be consumed by a peer whose `JsonDerivedType` table has not been updated.
- A memory bridge frame is capped by its slot; large payloads need the bridge bulk region, not a huge text frame.

Related pages: [command catalogue](command-catalog.md), [event catalogue](event-catalog.md), [handshake and resume](handshake-and-resume.md), [bridge layout](bridge-layout.md).
