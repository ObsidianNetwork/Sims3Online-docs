# Process-memory bridge layout and versions

The controller and host mod communicate through a process-memory bridge. The current shared layout uses a pipelined control ring plus a separate bulk region; the control slot still carries short UTF-8 protocol lines, while large state payloads travel through bulk slots.

## Version matrix

| Boundary | Current code value | Meaning |
|---|---:|---|
| Public line protocol | 1 | `HELLO`/`WELCOME` wire version; unrelated to the local bridge. |
| Bridge memory layout | 11 | Shared-region header/layout version. |
| Bridge protocol | 23 | Control-line protocol version. |
| Bridge hello wire | 6 | Handshake wire revision. |
| SimLink layout | 1 | Local sidecar header/record layout. |
| SimLink native API | 15 | Internal-call API version reported by the sidecar. |

The memory bridge constants are in `DedicatedServer\HostBridge\BridgeSharedRegionLayout.cs`, mirrored in `SimsOnlineServerHostMod\BridgeSharedRegionLayout.cs`. The controller also has a separate sidecar compatibility tuple in `DedicatedServer\Authority\RustAuthoritySidecar.cs`; do not treat it as the public line version.

## One contiguous region

`BridgeSharedRegionLayout` computes a 2,162,816-byte region:

| Region | Offset | Size | Shape |
|---|---:|---:|---|
| Control region | 0 | 65,600 | 64-byte region header plus sixteen 4,096-byte slots |
| Bulk region | 65,600 | 2,097,216 | 64-byte region header plus four 524,288-byte slots |
| Total | 0 | 2,162,816 | One region scanned/owned as a whole |

Within the 16 control slots, slots 0-7 are host-to-controller, and 8-15 are controller-to-host. The 32-byte control header at offset 32 stores layout version 11, slot count 16, slot size 4096, slot header size 16, and the directional bases/counts. The region begins with a 32-byte find marker; the header follows at offset 32.

A control slot's 16-byte frame header is `u64 senderId`, `u32 sequence`, `u32 payloadLength`; UTF-8 payload begins at byte 16. The slot budget is 4,080 bytes. Empty and in-progress slots use explicit sender-state markers; consumers acknowledge each consumed slot independently.

The bulk sub-region begins at byte 65,600. Its first 32 bytes are a distinct marker, followed by a 32-byte bulk-region header. Four bulk slots follow. Each bulk slot has a 32-byte header and up to 524,256 payload bytes. State markers distinguish empty, host-writing/ready/consumed, and controller-writing/ready/consumed. The control line contains a reference/metadata; it does not carry the bulk payload itself.

## Why two lanes exist

A single 4 KiB ping-pong slot forced the producer to wait on each consumed marker. With eight host-to-controller slots and eight controller-to-host slots, control lines can be pipelined independently in each direction while the command writer remains serialized. The bulk area prevents megabyte snapshots and diagnostic blobs from being encoded as huge control strings. A normal control payload that exceeds 4,080 bytes must be chunked or moved to bulk; it is not safe to increase the slot size unilaterally.

`DefaultHelloCaps` is `multi-slot=16,bulk=4x512k,layout=11,snapshot2=1`. The hello advertises those local bridge capabilities; it is not the public socket `HELLO|1`.

## Host outbound queue

Before writing to the shared region, the host's `OutboundLaneQueue` schedules pending
lines. Each drain step sends the oldest queued command reply, event, or other priority
line first. The oldest snapshot chunk is sent when the priority queue is empty.
Snapshot chunks keep their order even when replies are interleaved.
`DedicatedServer.Tests` exercises priority ordering and snapshot chunk order.

## Failure and compatibility behavior

- Attachment normally validates the control marker/header. If the control header was overwritten, a valid bulk marker and header can recover the region base. This is a validated recovery path, not permission to guess a base from an arbitrary matching pattern.
- The controller serializes command writes and waits for acknowledgement, but a timeout can leave a slot marked `ServerSenderId`. A later write may reuse and overwrite that unread frame. Timeout is therefore an unknown command outcome, not a guarantee of retention or safe retry.
- A mid-write sentinel means not ready; readers must wait for the final published sender marker.
- Bulk references must match a valid slot/length/CRC before the payload is consumed; a corrupt payload is discarded, not passed to typed decoding.
- `SNAPSHOT2` is a bulk payload detected by its numeric magic. The controller reader accepts versions 1 and 2; current writer version is 2. See [snapshot layout](snapshots-and-resync.md).

Related pages: [bridge verbs](bridge-verbs.md), [bridge diagnostics](bridge-diagnostics.md), [SimLink](simlink.md).
