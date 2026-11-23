# Capacity and performance

Capacity is a property of the whole path: the game simulation, controller,
bridge, network serialization, client mirror, and local presentation. A
single “players supported” number cannot be inferred from a controller
benchmark or a short live run.

## Work dimensions

| Dimension | Examples of scaling cost | How to observe |
|---|---|---|
| World simulation | Active Sims, lots, autonomy, global services | Server ticks, game time, CPU, autonomy backlog |
| Bridge | Snapshot extraction, command cadence, bulk payloads | Bridge latency, timeouts, queue depth |
| Network | Connected clients, encoded deltas, event fanout | Bytes/sec, per-client send queue, delivery latency |
| Mirror | State application, alias resolution, local engine calls | Client apply acknowledgements and lag |
| Presentation | Animation, routes, objects, audio, effects | Live visual comparison and client frame rate |
| Lifetime | Memory growth, save size, long-lived caches | Virtual/private memory and multi-hour soak |

The authoritative server runs a full world. Filtering what a client receives
primarily reduces mirror and network work; it does not remove world-wide
server simulation obligations.

## Bounded simulation measurement

A capacity study on a test save varied the number of specially scoped lots.
At normal speed, the game held roughly its nominal 30 simulation ticks per
second even at the largest tested scope. At faster speed, a modest decline
appeared only at the largest tested scope, and the autonomy queue first
showed backlog there. This suggests spare simulation headroom for a handful
of lots in that configuration; it is not a universal capacity guarantee.

Memory grew by several megabytes per additional scoped lot in that test. The
game process is 32-bit, so address-space pressure and long-session growth
remain relevant even when per-tick CPU appears acceptable.

## What the measurement did not include

- Connected clients and the cost of fanout to several players.
- Dense community lots, heavy object populations, or unusual content.
- Long-session memory behavior sufficient to establish a safe operating
  duration.
- Multiple machines, network latency, packet loss, or reconnect churn.
- Every game build, expansion set, save, or mod combination.

Do not turn this bounded run into a “supports N players” statement. Report
the workload, game configuration, duration, and observable pass criteria.

## Hidden-window behavior

A separate test found that hiding the server window maintained approximately
the visible-window tick rate in its tested setup; minimizing was measurably
slower. This is useful operational evidence for a hidden game window, not
proof of no-desktop hosting or cloud scaling.

## Bottleneck diagnosis

1. Measure authoritative game ticks and game-clock progress.
2. Measure bridge request latency and backlog separately from simulation
   ticks.
3. Compare server send progress with each client's applied acknowledgement.
4. Check whether interest filtering is enabled and how many entities each
   client receives.
5. Inspect local game frame rate and presentation queue behavior.
6. Track memory over time; a stable short run does not prove a stable long
   session.

A low client acknowledgement rate with healthy server ticks points toward
delivery or apply throughput, not automatically toward simulation CPU. A
live port with a frozen game clock points toward simulation pause or
game-loop health, not necessarily a dead bridge.

## Capacity acceptance recipe

A reproducible capacity report should define:

- Exact build and enabled integration components.
- World/save and population characteristics without private paths or
  personal data.
- Number of clients, claims, lots of interest, and active interactions.
- Game speed and whether the server window is visible, hidden, or minimized.
- Warm-up and measurement intervals.
- Tick rate, CPU, memory, queue depth, bridge latency, and per-client lag.
- Failure thresholds and what counts as a successful session.

## Failure modes / limits

- CPU-only results can hide a memory or bridge bottleneck.
- A short run cannot establish memory stability.
- A few scoped lots do not represent large or object-heavy lots.
- Hidden-window behavior is platform- and configuration-dependent.
- There is no supported public capacity number until end-to-end multi-client
  testing establishes one.

## Related pages

- [Scope, interest, LOD, and dormancy](scope-interest-lod-dormancy.md)
- [Process model](process-model.md)
- [Dedicated server and offline host](dedicated-server-and-offline-host.md)
- [Status](../status.md)
- [Testing](../testing.md)

## Reading benchmark results responsibly

A benchmark answers only the question represented by its workload. The
scoped-lot study compared several scope sizes in one save with no connected
clients. Its tick-rate result shows that this workload did not materially
reduce normal-speed simulation throughput. It does not answer how many
clients can join, how much bandwidth each receives, how fast presentation
queues drain, or whether memory reaches a stable plateau.

The observed memory increase with additional scoped lots is not a clean
per-lot cost model: population changed during the run, private bytes drifted
with time, and the sample size was limited. Report the measured values as
observations and distinguish them from extrapolation.

## Bottleneck signatures

| Signal combination | Candidate bottleneck | Next measurement |
|---|---|---|
| Server ticks slow; CPU high | Game simulation or rendering work | Repeat hidden; compare autonomy backlog and lot load |
| Server ticks healthy; bridge calls slow | Bridge cadence or serialization | Measure per-command latency and bulk traffic |
| Server sends quickly; client ack lags | Client apply or local game throughput | Batch count, apply queue, client tick/frame rate |
| Memory grows with elapsed time | Leak, cache growth, or world accumulation | Multi-hour memory samples and process dump on failure |
| Only one client lags | Per-client network or local presentation | Compare that client's send backlog and apply metrics |
| Port responsive; clock stopped | Paused/stalled game rather than dead controller | Inspect in-world state and tick delta |

## Approximate resource model

A useful first-order model separates costs rather than hiding them in one
number:

```text
server load = world simulation + snapshot extraction + event recording
            + protocol encoding + per-client fanout
client load = local game rendering + mirror apply + presentation replay
            + client-side network decoding
```

Some terms are shared and others scale per client. An interest filter can
reduce bytes and client apply work while leaving the world simulation term
largely unchanged. A batch operation may reduce controller/game bridge round
trips without reducing engine simulation cost.

## Capacity test matrix

| Axis | Low-load example | Stress case |
|---|---|---|
| Claims | One active player Sim | Several claims across distant lots |
| Actors | Mostly idle town | Many active interactions and visitors |
| Lots | Small residential lots | Large community lot and dense object set |
| Clients | One local client | Several clients with different interest sets |
| Time | Short smoke test | Multi-hour soak through save cycles |
| Transport | Loopback | Delayed/lossy network on separated hosts |
| Window state | Visible baseline | Hidden and minimized comparisons |
| Content | Base known setup | Declared expansion/mod compatibility matrix |

## Memory and address-space interpretation

Track private bytes and virtual size separately. A rising private-byte curve
can reflect committed allocations; a virtual-address ceiling can be reached
even when physical memory is available. The game's 32-bit process makes
fragmentation and address-space availability important. One sample near a
peak is not enough to distinguish transient allocation from sustained
growth.

A soak test should record at a fixed cadence, include save/load and client
reconnects, and note the first hour separately from later behavior. If a
process fails, capture the failure evidence without exposing private machine
paths or unrelated user data.

## How to verify / test

- Use a warm-up phase before measuring steady-state performance.
- Keep game speed, window state, world, player count, and content fixed
  across comparisons.
- Collect server tick rate, client acknowledgements, CPU, memory, queue
  depth, and bridge latency together.
- Repeat noisy measurements and report variance rather than selecting a
  favorable run.
- Include one run with an active interaction and one mostly idle run.
- Run a long soak past expected failure duration and include save/rejoin
  events.
- State explicitly when a proposed threshold is a service objective rather
  than a measured limit.

## Failure modes / limits

Do not infer safety from a quiet CPU graph if world time is paused or the
client is falling behind. Do not extrapolate player counts linearly from a
one-client test. Capacity reports need representative topology and live
session evidence.

## Related pages

- [Scope, interest, LOD, and dormancy](scope-interest-lod-dormancy.md)
- [Testing](../testing.md)
- [Status](../status.md)
- [Troubleshooting](../troubleshooting.md)

## Measurement table and interpretation

The historical scoped-lot run varied up to eight residential lots and
measured the server without connected clients. At normal speed, tick rate
remained near the game's nominal rate across those runs. At triple speed,
the largest tested scope showed a modest decline relative to the smaller
scopes. The autonomy queue showed a backlog only at the largest scope and
much higher game speed.

The measurement is evidence that a handful of additional lots did not
measurably reduce normal-speed ticks in that setup. It is not a scaling law.
The run used one save and one machine; Sim counts changed as town characters
entered and left; visible rendering added CPU noise; and the observed memory
change cannot be cleanly attributed to lot count alone.

| Result | What it supports | What it does not support |
|---|---|---|
| Normal tick rate stable over tested scope sizes | This workload had simulation headroom | Unlimited lots or players |
| High-speed tick rate declined at largest scope | Dense scope can increase work | A universal break point |
| Autonomy backlog appeared in the largest high-speed case | Scheduling demand can exceed pick throughput | Normal-speed capacity threshold |
| Private memory rose with scope/time | Memory must be tracked | A fixed per-lot memory formula |

## Per-client costs

The server's world simulation cost is not the only expense. Each additional
client can add serialized state, filtering work, transport queues, and event
fanout. Each client also applies mirror updates through its own game
process. These costs can scale differently: adding a client may increase
outbound bandwidth without changing server tick rate, while a slow client
can accumulate lag even when the server remains healthy.

A useful experiment varies one axis at a time: fixed world and clients while
changing active interaction count; fixed workload while adding a client;
fixed count while moving clients to separate lots; and fixed topology while
changing event rate. Avoid changing all of these in one benchmark.

## Saturation and backpressure

Bounded queues prevent unbounded memory growth, but when a queue fills the
system must choose whether to reject, drop, coalesce, or force
resynchronization. The right behavior depends on data type. A command should
not disappear silently; a stale position update may be coalesced; a
presentation stream may need to report lost records and reset a client's
baseline.

| Data | Appropriate overload response to define |
|---|---|
| Player command | Reject or report overload; preserve correlation |
| Snapshot delta | Replace with full snapshot when safe |
| Position updates | Coalesce to latest state where semantics allow |
| Lifecycle event | Preserve ordering; do not silently discard |
| Presentation record | Bounded loss reporting and repair strategy |
| Diagnostic sample | May be sampled, but report sampling/drop count |

## Soak-test design

A meaningful long-session test should run longer than the suspected failure
window and capture regular samples of server and client memory, tick
advancement, bridge latency, recorder counts, queue depths, client
acknowledgements, save outcomes, and reconnect behavior. Include both a
quiet baseline and a session that continuously produces presentation events.
If resource growth appears only with recorders enabled, run a controlled
comparison rather than attributing it from correlation alone.

The first hour can behave differently from later hours due to caches and
world population. Report slope by interval, not just start and end points.
Include process virtual size as well as private memory for a 32-bit process.

## How to verify / test

- Reproduce the baseline run with a documented environment and fixed
  workload.
- Add clients and active interaction rate as separate experiments.
- Compare visible, hidden, and minimized server states.
- Sample memory and queue depth at fixed intervals over a multi-hour run.
- Include save, restart, and late-join events in at least one soak.
- Fail the test if world ticks stop even when bridge diagnostics still
  answer.
- Publish only aggregated measurements; exclude private run directories and
  machine identifiers.

## Failure modes / limits

The existing figures come from a bounded study and should be treated as
historical measurements, not public service guarantees. Performance
characteristics must be re-measured after changes to the game fork,
recorders, bridge, mirror batching, or save lifecycle.

## Related pages

- [Scope, interest, LOD, and dormancy](scope-interest-lod-dormancy.md)
- [Dedicated server and offline host](dedicated-server-and-offline-host.md)
- [Testing](../testing.md)
- [Status](../status.md)
