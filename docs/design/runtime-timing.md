# Runtime timing, loop choices, and memory observations

## Purpose

Several investigations asked whether client/server animation differences came
from delivery latency, event timing, or independent native choices. Their main
lesson is methodological: identical high-level inputs do not prove that two
local game processes made the same internal decision.

## Shower-loop measurements

In repeated runs, the exit request occurred about 1.27?1.41 seconds after each
process entered the loop. Each process left at one of two later loop
boundaries: roughly 0.83?0.97 seconds or 1.83?1.97 seconds after the request.
Across 25 matched pairs, 12 pairs left a cycle apart. The request phase did
not predict which boundary a process selected; a tested marker cutoff
misclassified 17 of 50 observations.

A fixed playout delay would remove only delivery skew. It could not make two
independent local choices use the same loop count. State catch-up was also a
poor repair for this case: re-requesting a state could cause a client-only
restart rather than synchronize the decision.

## Outfit-build timing

A separate measured stretch during a shower animation normally covered about
48?50 simulation ticks (roughly 1.6 seconds), but one process in one boot took
about 63 ticks (roughly 2.1 seconds), an extra 14 ticks. Both processes
continued ticking steadily, so the observation was not explained by a frame
hitch in the measured interval. The event streams and final outfit category
matched in those runs.

One plausible cause was asynchronous outfit/model construction, but the
investigation did not establish the causal link. Per-process cache or
level-of-detail state remained possible. The right follow-up was to observe
build readiness and clip events at shared seams, not patch the shower
interaction itself.

## Native random draws and replay

If a loop count or animation variant is chosen by a local random draw outside
the recorded input stream, replaying the same request cannot guarantee the
same visible result. A robust design would need to capture the decision or a
stable seed at the boundary where it is consumed. Capturing every random call
is broader than capturing a gameplay-relevant choice; the decision must be
tied to a semantic event.

## Memory writer investigation

A separate interpreter-memory investigation found a zeroed descriptor
associated with a crash under route-heavy scenario load. A constrained offline
host reproduced frame placement patterns, but the exact store that zeroed the
descriptor was narrowed rather than named. Candidate sources included a
value-type return stored into an undersized slot, an evaluation-stack overflow
and a chunk with conflicting ownership. The writer remained an open forensic
question; no public claim should say that a specific call caused it.

## Evidence table

| Observation | Supported inference | Not established |
|---|---|---|
| Same request lines, different loop boundary | Internal per-process choice or timing remains | Exact random source or native transition mechanism |
| Stable 33 ms ticks during long clip | No large tick hitch in that measured segment | No timing anomaly elsewhere in the interaction |
| Same final outfit | End state converged | Same model-build path or same animation duration |
| Descriptor value became zero in loaded run | Memory corruption relates to exercised runtime path | Exact writer or universal trigger |
| Idle run did not reproduce corruption | Scenario load is relevant evidence | Idle safety or a fix |

## Practical test method

1. Record interaction and animation boundary events with timestamps and
simulator ticks.
2. Repeat several times per process and across fresh boots.
3. Keep the two processes' raw traces separate; compare event request, event
delivery and local state timing.
4. Test candidate fixes through a shared recorder seam or a single engine
choke point.
5. Report unproven hypotheses separately from measured facts.

## Failure modes and limits

- Asynchronous script callbacks can timestamp a native event a tick or two
late.
- A small sample may make a random outcome look deterministic.
- A process-wide cache can change timing between boots.
- Matching animation markers does not prove matching internal state.
- An observed corrupted descriptor does not identify its writer.

Related pages: [presentation decisions](decision-presentation.md), [probe
results](probe-results.md), [divergence
forensics](rust-divergence-forensics.md), [open problems](open-problems.md).


## Reproducible random decisions

A separate native-picks investigation distinguished three sources of
divergence: an engine path-finder tie-break, a Random node inside a decision
graph, and a clip-selection rule that remained stable per process boot. A
keyed-draw mechanism was built for the Random-node case, using a seed plus
semantic context such as graph/node identity, driver boundary and
evaluation/retry count. It was not live-verified in the handoff record. It
should therefore be described as a design/build result, not a proven fix for
the animation mismatch.

A useful replay key must be stable for the same logical decision and must not
depend on unrelated call order. If a node can be evaluated multiple times
before a state boundary, include enough context to distinguish those
evaluations. The record should also make retries explicit; otherwise a retry
can consume a different draw and desynchronize peers.

| Key component | Purpose |
|---|---|
| Session/driver seed | Separates independent simulation streams |
| Decision-graph identity | Prevents unrelated graphs sharing a draw sequence |
| Node identity | Identifies the semantic random choice |
| State-boundary count | Resets or scopes history at a meaningful transition |
| Evaluation count | Distinguishes repeated evaluations within a boundary |
| Retry index | Makes re-evaluation behavior explicit |

## Descriptor mitigation versus writer attribution

Later work reported a mitigation for the descriptor-zeroing crash under
route-heavy scenario load, using a reserved chunk and a mod-side change. That
does not identify the exact store that wrote the zero. The forensic question
remained whether a wide value-type return, an evaluation-stack overflow,
conflicting task-state ownership, or another write crossed the frame boundary.
Public documentation should keep the mitigation and causal attribution
separate until a reproduction names the writer.

Related pages: [social-session research](social-session-research.md), [open
problems](open-problems.md).


## Separating latency, phase, and local choice

For a streamed event, observed client time can be decomposed into server event time, transport delay, client processing delay and local animation/engine delay. A fixed playout buffer can compensate a stable transport term, but it does not synchronize a local random choice or a process-specific clip duration. Reports should record each component where possible rather than labeling all visible lag as ?network latency.?

| Term | Measurement point | Possible remedy |
|---|---|---|
| Server event time | Recorder or simulator tick | Correct event boundary capture |
| Delivery delay | Server send to client receive | Queue, cadence or transport adjustment |
| Callback processing | Client receive to script apply | Reduce work or schedule consistently |
| Native clip length | Clip start to local completion | Capture choice, seed or use state outcome |
| Loop boundary | State request to transition | Synchronize semantic choice, not only request time |
| Model readiness | Build start to ready notification | Prewarm or wait at a shared lifecycle boundary |

## Stable and variable measurements

A difference that persists for every run within one boot but flips between boots suggests a boot-scoped process property, such as registration or load order. A difference that varies run-to-run within one boot is more consistent with per-query timing or random choice. This is a hypothesis discriminator, not proof: caches can be invalidated mid-boot, and scenario inputs can differ subtly.

## Random decision capture

When a native path finder or graph node chooses among equivalent alternatives, replay needs the selected result or a shared deterministic seed at the semantic boundary. A keyed draw can use graph identity, node identity, per-driver boundary and evaluation index. It must preserve the game's own draw when the feature is disabled and fail safely if the graph or node identity is unknown. A draw log is useful for verification but should not turn into a large permanent trace of unrelated random calls.

## Memory mitigation and diagnosis

The later runtime record reports that a reserve adjustment plus a mod-side change stopped the sampled crash series under the tested scenario campaign. The earlier analysis did not identify a unique writer for the descriptor zero. These are different conclusions: a mitigation may improve observed reliability without proving the exact cause. Public bug reports should name the mitigation and residual forensic uncertainty separately.

## How to verify/test

- Run repeated scenarios within a boot and across fresh boots.
- Record event timestamps and simulator ticks on both processes.
- Compare random-pick keys and outcomes where the decision hook provides them.
- Check clip duration independently from event delivery.
- For a memory mitigation, compare matched workload durations and crash counts; report total exposure.
- Keep idle controls and scenario-load runs separate.

Related pages: [divergence forensics](rust-divergence-forensics.md), [probe results](probe-results.md), [presentation](decision-presentation.md), [open problems](open-problems.md).
