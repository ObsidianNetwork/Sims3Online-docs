# Handshake and resume

The JSON oracle accepts a JSON command as its first non-blank line; it has no `HELLO`, welcome event, or wire version negotiation. The public line-v1 endpoint is different: every connection must begin with a versioned `HELLO` before it may send a `CMD`.

## Line-v1 hello gate

A line-v1 client sends `HELLO|1|<caps>`. The `caps` field is empty or a comma-separated sequence of `key=value` pairs. The parser preserves declaration order; the gate checks the values needed by the front door. Current public clients use the ring and world-file capabilities, optionally role and resume data:

```text
HELLO|1|ring=1,wfs=1,role=client
WELCOME|1|42|
```

`1` is the only supported public protocol version. `WELCOME` reports the server's `u32` event epoch and the server capability field (empty in the current front door). The `ring=1` and `wfs=1` declarations are checked by the kernel hello gate; they are not a negotiation to silently downgrade an unsupported peer. The role is optional and defaults to `client`; if supplied, accepted values are `client`, `client2`, or `client3`. Other role values are `bad_line`. The gate fills bridge and native SimLink versions from its own constants; those sidecar version numbers are not fields in this public hello.

The final `|` in `WELCOME|1|42|` opens the empty server capability field. `42`
identifies this server's event epoch. `SimsOnlineClient.LineHandshakeJoinBulkDeltaAndAnimation`
uses this epoch in a synthetic loopback handshake. The fixture reads `HELLO`, sends
`WELCOME`, and checks that the next decoded command is `JoinSession`. In
`SimsOnline.Client/Program.cs`, the line client awaits `PublicWireConnection.ConnectAsync`
before sending `JoinSession`. Each frame ends with LF on the socket.

Malformed UTF-8 or malformed capability syntax produces `ERR|0|bad_line|-|` and closes. A well-formed but rejected hello produces `ERR|0|hello_rejected|HELLO|<hex detail>` and closes. The complete closed detail-token set is `version_unsupported`, `json_hello`, `legacy_sidecar_hello`, `MissingSimlinkRing`, and `MissingWorldFromServer`; see [errors and refusals](errors-and-refusals.md).

## Join and command ordering

After `WELCOME`, a new client sends `CMD|corr|Verb|commandId|playerId|args...`. New `corr` values increase per connection; retrying the same command reuses its correlation and `commandId`. A command ID is deduplicated over the kernel's 32-entry `COMMAND_ID_WINDOW` (`rust\sim96_kernel_cleanroom\src\runtime_bridge.rs`), and a duplicate returns the cached reply bytes rather than applying the effect a second time. The line-v1 codec maps JSON GUIDs injectively to the unsigned wire ID; it never truncates them.

A successful join returns an `ACK` with a resume token in an additional field and emits the session events. Example fields are encoded according to the line codec (strings are UTF-8 hex):

```text
C> HELLO|1|ring=1,wfs=1,role=client
S> WELCOME|1|42|
C> CMD|1|JoinSession|1|7|616c696365|~
S> ACK|1|JoinSession|ok|resume_token=0123456789abcdef0123456789abcdef
S> EV|1|100|1|SessionJoined|7|616c696365|~|[70726573656e746174696f6e5f73747265616d]
C> CMD|2|ClaimSim|2|7|12345
S> ACK|2|ClaimSim|12345
S> EV|2|101|2|SimClaimed|7|12345|~
```

The sample epoch, IDs, event sequence and 32-hex token are illustrative. In the actual server the token is generated from 16 OS-random bytes. The `presentation_stream` value belongs to the `SessionJoined.capabilities` event field, not the HELLO capabilities. The Rust line backend currently includes it in the join event; the JSON server advertises it when its broadcaster is active.

`C>` and `S>` label the sender in this transcript. In `CMD|1|JoinSession|1|7|616c696365|~`,
the first `1` is the request correlation (echoed by the `ACK`), the second is the wire
command ID, and `7` is the player ID. The remaining fields are the UTF-8 hex for `alice`
and the null `OnScreen` value. The `EV` header carries its own delivery sequence, simulation
tick, and command correlation, in that order.

## Warm reconnect

A disconnect does not immediately discard an eligible lease. A client retains the token and last event sequence it completely received, then includes both in the next hello:

```text
C> HELLO|1|ring=1,wfs=1,role=client,resume=9.4,resume_token=0123456789abcdef0123456789abcdef
S> WELCOME|1|9|
S> RESUME_OK|9|5|1
S> EV|5|0|0|GameSpeedChanged|Pause
```

`RESUME_OK|epoch|from_seq|count` is followed by precisely `count` replayed events. The server requires the same epoch, an identified disconnected lease, an available gap in the bounded ring, no bulk-backed event in the gap, and an unexpired monotonic lease. The resume window is at most 120,000 ms; history is also bounded by 67,108,864 bytes and the kernel's event-ring frame count. A sequence cursor must be no later than the last sent sequence. The event consumer advances its cursor only after bulk assembly completes and the event is delivered to the consumer.

Here the client last delivered sequence `4` in epoch `9`. The server resumes from
sequence `5` and replays one event. The count is the number of replayed `EV` frames.
An intervening sequence can belong to an `ACK`, so the count does not imply consecutive
event sequence numbers. The managed test `SimsOnlineClient.LineResumeKeepsCompletedCursorAndCommandIdentity`
checks this cursor and credential, the warm result, and a fresh result after the epoch
changes. The Rust test `warm_replay_is_the_original_v1_gap_and_reconnect_cancels_the_disconnect_clock`
checks a replay gap with an intervening ACK sequence.

If the token, cursor, epoch, or retained history cannot resume safely, the server replies `RESUME_FRESH|epoch`. The client must clear its resume credential and perform cold bootstrap; it must not replay old commands against a fresh lease. The token is possession proof for a disconnected lease, not account authentication. Tokens are in-memory credentials: do not put them in saves, logs, or diagnostics.

An active connection cannot be resumed, and a second reconnect cannot bind the same disconnected lease after the first succeeds. An explicit leave, superseding join, lease timeout, server epoch change, evicted history, or a bulk event in the missing range ends warm-resume eligibility and releases the player's claims.

## Capability declarations are not JSON capabilities

The line-v1 `HELLO` `caps` describe client transport/role requirements. JSON `session_joined.capabilities` describe server behavior that affects mirror clients. The current JSON capability is `presentation_stream`: when present, the client must not start legacy interaction replay. Unknown JSON capability names are ignored, and an absent list means an older server. See [wire formats](wire-formats.md) and [replay and presentation](replay-presentation.md).
