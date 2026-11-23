# SimLink sidecar

SimLink is an optional native sidecar for local game-to-controller traffic. It is not the public network protocol and it does not replace the process-memory bridge. The x86 sidecar runs inside TS3W; the .NET controller opens the named shared-memory section and drains the ring. The bridge remains the control/command path, while SimLink provides a local data path for shadow/presentation records and measurement.

## Layout v1

`SimLink\src\simlink_layout.h` defines a section header followed by one single-producer/single-consumer byte ring:

| Header field | Offset | Type | Purpose |
|---|---:|---|---|
| Magic | 0 | 8 bytes | `SIMLINK1` signature |
| Layout version | 8 | u32 | Current layout 1 |
| Header size | 12 | u32 | 256 bytes |
| Process ID | 16 | u32 | Owning game process |
| State / error | 20 / 24 | u32 | Created, register requested, registered, or failed; error reason |
| Ring offset / size | 28 / 32 | u32 | Ring begins after header; default ring is 4 MiB and power-of-two |
| Managed string length offset | 36 | u32 | Runtime-probed string layout |
| QPC frequency | 40 | i64 | Converts native QueryPerformanceCounter stamps |
| Write / read position | 48 / 56 | u64 | Monotonic producer/consumer byte positions |
| Published / dropped / contended / bad args | 64-88 | u64 counters | Health and loss counters |
| Native API version | 96 | i32 | Current API 15 |

Each record is 8-byte aligned. Its 24-byte header contains total padded size (`u32`), kind (`u16`), lane (`u16`), sequence (`i32`), UTF-16 character count (`u32`), and QPC (`i64`), followed by UTF-16 code units. A `Pad` record fills the tail when a record wraps; consumers skip it. The producer owns `writePos`; the consumer advances `readPos` only after a complete record has been accepted.

The ring's `Publish` operation never blocks or allocates. A full ring, oversized record, or concurrent second writer increments a drop/contended counter rather than stalling the game thread. The .NET reader uses the same offsets in `DedicatedServer\HostBridge\SimLinkShadowReader.cs`.

## Registration lifecycle

At load, the sidecar creates the shared section and named control/data events, but it does not call into Mono. The controller requests registration after the host bridge is alive and Mono is initialized. The native sidecar validates the target runtime's internal-call registration entry point, registers the API, and publishes `StateRegistered` only after the calls are ready. The host mod does not call unregistered internal calls; a mismatch fails closed with `StateFailed` and an error code.

The mod probes the managed string/array layout instead of assuming offsets. `SimLinkNative.Version`, `Probe`, `Publish`, `InboundPoll`, `NoteBridge`, `ProbeArray`, and `InboundRead` are declared as internal calls in `SimsOnlineServerHostMod\SimLinkNative.cs`. Call sites are guarded until sidecar activation because an unregistered Mono internal call is fatal.

## Lanes and measurements

The C++ ring defines lane 1 for replay lines and lane 5 for batch-end markers; the client shadow path also uses sample lanes to compare data arriving through SimLink and the bridge. Records carry a QPC at publish, allowing same-host hop timing when the controller and game share a QPC frequency. This is local timing evidence, not a cross-machine clock.

## Failure and test coverage

A missing section or wrong magic/layout prevents reader attachment; failed runtime probes
or native API checks can prevent registration/writer activation. The native producer uses
a power-of-two ring size. `SimLinkShadowReader` copies `RingSize` from the shared header
without validating that assumption before activation and uses `position & (RingSize - 1)`
when draining. Do not interpret reader activation as validation of an arbitrary section's
ring size. Ring-full and concurrent-writer conditions drop records and are observable
through header counters. The core game/controller projects do not require building SimLink;
the native project is optional and platform-specific.

`DedicatedServer.Tests\SimLinkTests.cs` contains `SimLink.ReaderDrainsWrappedRing`, `SimLink.ReaderDrainsNativeSelftest`, and `SimLink.NativeConsumerDrainsDotnetWriter`. Native integration cases require Windows and the built sidecar self-test; the managed ring-layout reader test exercises wrap and pad records.

Related pages: [bridge layout](bridge-layout.md), [latency and verification](latency-and-tests.md), [testing](../testing.md).
