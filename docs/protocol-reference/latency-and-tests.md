# Latency and verification

A protocol latency number is meaningful only when the measured hop, clock, and event type are explicit. A server tick is simulation progress, not wall-clock time; a client mirror may be slow because of transport, a blocked host command, snapshot folding, or game-thread scheduling.

## Instrumented path

The SimLink record header carries a local QPC timestamp at publish. The controller's sidecar path can compare when it observed the corresponding bridge line and record an arrival timestamp. This supports same-host measurements of the game-to-controller hop. It does not establish synchronized time between machines. Use line sequence and correlation IDs to match records; do not infer packet loss from a gap in one counter without checking the other lane/history source.

The network line protocol has per-client `seq`, server `tick`, and command `corr`. Resume replays event sequence numbers from a bounded history. A bulk-backed event is delivered only after all referenced chunks assemble; the receive cursor should represent completed delivery rather than the first event line observed.

For outbound TCP delivery, `OutboundDeliveryPump` records intended/delivered/failed recipients and queue timestamps. It is a single FIFO, so a slow delivery operation can place later sends behind it until the bounded write timeout or cancellation completes. For gameplay commands, the host bridge timeout bounds the call; a 10-second player-command timeout is intentionally shorter than the default 120-second host timeout.

## What to inspect

| Symptom | Evidence to compare |
|---|---|
| One client mirror is behind | That player's acknowledged snapshot tick vs current broadcast tick; inspect `ack_lag` and resync action. |
| Snapshot parse stalls or fails | Bulk size, `SNAPSHOT2` magic/version, decoder result, and whether control lines are only metadata. |
| Animation appears late | SimLink QPC publish-to-arrival, bridge correlation, event sequence, and whether a history gather held live lines. |
| Public events disappear under load | Outbound queue fullness, per-recipient delivery receipt, write timeout, and connection-close reason. |
| A reconnect triggers a cold load | Epoch, token validity, cursor gap, history eviction/byte cap, bulk event in the gap, and disconnect age. |

## Tests that pin behavior

- `DedicatedServer.Tests\PublicLineCodecTests.cs`: line frames, refusal spellings, composite fields, malformed input, bulk assembly, ordering, and exact cap boundaries.
- `DedicatedServer.Tests\ProtocolHardeningTests.cs`: bounded line reader, connection closure on oversized line, player binding, exposure warnings, and lot-export rejection.
- `DedicatedServer.Tests\WireJsonPinningTests.cs`: serialized JSON discriminator/field ordering and byte representation.
- `DedicatedServer.Tests\JoinCapabilityTests.cs`: present/absent/unknown capability handling and presentation-stream behavior.
- `DedicatedServer.Tests\OutboundListenerTests.cs`: bounded outbound queue behavior and delivery receipts.
- `DedicatedServer.Tests\SimLinkTests.cs`: local ring wrap, native self-test interoperability, and reverse-direction consumption.

Run the documented repository test workflow in [testing](../testing.md). The native SimLink self-test is optional; its absence is reported as skipped unless the Windows sidecar CI job requires it. Tests validate framing and state transitions, not real-world latency on every machine.

Related pages: [transport limits](transport-limits.md), [snapshots and resynchronization](snapshots-and-resync.md), [SimLink](simlink.md), [troubleshooting](../troubleshooting.md).
