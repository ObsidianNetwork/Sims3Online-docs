# Transport limits

The repository has independent caps for public JSON, public line-v1, bridge control frames, bulk data, and queueing. Applying the wrong channel's limit can either reject a valid message or leave an unbounded reader.

## Framing and buffering

| Boundary | Code constant/default | Behavior at the limit |
|---|---:|---|
| JSON command line | `ClientListenerServer.DefaultMaxCommandLineChars = 4 * 1024 * 1024` characters | `BoundedLineReader` raises `LineTooLongException`; listener sends `server_error` and closes. |
| Line-v1 content | `PublicLineCodecV1.PUBLIC_LINE_MAX_BYTES = 4,194,304` UTF-8 bytes, LF excluded | Codec reports `line_too_long`; public line server closes on oversized input. |
| Line-v1 bulk chunk | `PUBLIC_BULK_CHUNK_RAW_BYTES = 3,145,632` bytes | Larger chunk is `bulk_malformed`; encoded base64 plus line header remains under the line cap. |
| Assembled line-v1 bulk | `PublicLineCodecV1.BulkReceiver` default `maxBulkBytes = 256 * 1024 * 1024` | Rejects before adding bytes beyond this cap. |
| Concurrent line-v1 bulks | `PublicLineCodecV1.BulkReceiver` default `maxInFlightBulks = 16` | An introduction over capacity is malformed. |
| Deferred line-v1 events | `PublicLineCodecV1.EventReceiver` default `maxPendingEvents = 1,024` | Further pending events fail as malformed rather than grow without bound. |
| World-save JSON chunk | `WorldSave.ChunkBytes = 256 * 1024` bytes | Each `WorldFileChunk.Data` event contains at most one chunk; empty files still produce one empty chunk. |
| Bridge control payload | `BridgeSharedRegionLayout.SlotPayloadMax = 4,080` bytes (`SlotSize - SlotHeaderSize`) | Large payloads must use the bridge bulk area or a chunked message. |
| Bridge bulk slot payload | `BridgeSharedRegionLayout.BulkSlotPayloadMax = 524,256` bytes (`BulkSlotSize - BulkSlotHeaderSize`) | Payload must fit a bulk slot; four slots are available. |
| Client event-history ring | `rejoin::CLIENT_EVENT_RING_MAX_BYTES = 67,108,864`; `wan_transport::RING_BUFFER_FRAMES = 2,048` | The byte or frame cap evicts the oldest range; a lease ends if the missing range is evicted. |

The JSON limit is a character count. UTF-8 output bytes can exceed the same numeric value for non-ASCII text. The line-v1 limit is measured in bytes and rejects invalid UTF-8. The C# v1 codec bounds assembled bulks; the JSON oracle's world-file events use their separate 256 KiB chunk size.

## Timeouts and pacing

| Operation | Default from code | Effect |
|---|---:|---|
| Public broadcast write | `ClientListenerServer.BroadcastWriteTimeout = 5 s` | A slow recipient cannot hold a broadcast forever; failure is reported in the outbound receipt. |
| Outbound delivery pump queue | `ClientListenerServer.DefaultOutboundQueueCapacity = 256` operations | One FIFO pump handles delivery; enqueue can fail immediately as `QueueFull`. |
| Player-paced host bridge command | `SessionConfig.DefaultPlayerCommandTimeout = 10 s` | Bounds interactive commands so one blocked bridge call does not hold the serialized bridge write lock indefinitely. |
| Force-align host command | `SessionConfig.DefaultForceAlignCommandTimeout = 30 s` | Allows posture/seat transitions more time than ordinary player commands. |
| Host bridge command timeout | `SessionConfig.Parse` default: `TimeSpan.FromSeconds(120)` | Long-running load/save/snapshot work may use the host timeout; `--host-timeout-seconds` can configure it. |
| Join history gather | `AuthoritativeSession.JoinGatherLimit = 30 s` | On expiry, the gather is abandoned and held live lines are delivered without the delayed history. |
| Wait for in-world state | `AuthoritativeSession.InWorldWait = 60 s` | Bounds the session's wait for the host world to become ready. |
| Sidecar I/O | `RustAuthoritySidecar.DefaultIoTimeout = 2 s` | A wedged authority sidecar fails as a transport failure instead of blocking the join indefinitely. |
| Warm-resume lease | `rejoin::RESUME_WINDOW_MAX_MS = 120,000` | Disconnected lease expires on monotonic elapsed time. |

These are separate clocks. The JSON listener's `ClientListenerServer.BroadcastWriteTimeout` is not the host bridge timeout; resume expiry is not a TCP socket timeout; a bridge command timeout does not enlarge a line frame.

## Outbound queue behavior

`OutboundDeliveryPump` is bounded and FIFO with a single reader. Accepted operations complete with an `OutboundDeliveryReceipt` that records intended, delivered, and failed recipients. The pump classifies zero recipients, queue-full, cancellation, partial delivery, delivery failure, and faults separately. A queued broadcast can therefore be accepted by the enqueue path and still fail during delivery; inspect the completion receipt rather than treating enqueue as delivery.

Backpressure should be visible rather than silently discarding protocol data. Tests in `DedicatedServer.Tests\OutboundListenerTests.cs` block a delivery deterministically to exercise the one in-flight operation and bounded waiting queue. `DedicatedServer.Tests\ProtocolHardeningTests.cs` covers overlong lines and verifies the connection closes after the error. The resume event ring additionally has a 2,048-frame cap (`rust\sim96_kernel_cleanroom\src\wan_transport.rs::RING_BUFFER_FRAMES`), independent of its 64 MiB per-client byte cap.

Related pages: [wire formats](wire-formats.md), [snapshots and resynchronization](snapshots-and-resync.md), [bridge layout](bridge-layout.md).
