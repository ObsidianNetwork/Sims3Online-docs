# Social interactions: approach, paired actions, and mirrored state

This page separates three related but distinct behaviors: moving a Sim toward a
social target, running a paired social in the game engine, and synchronizing
relationship or conversation state. The current mirror supports the first and
has explicit state streams for the third; a route event alone does not replay
the paired social.

## The three layers

| Layer | What it represents | Current mirror surface |
|---|---|---|
| Approach | A Sim moving toward a target Sim or target position | `GoHere`-style route replay |
| Paired action | The action selected between two participants, including each side's queue and animation | Not reconstructed by route replay |
| Social state | Relationship data and active conversation membership, turn, and topics | `Rl` and `Cv` boundary-state records |

A client can therefore show two Sims converging and can receive relationship
and conversation state without running the same `SocialInteractionA` / `B`
instances or reproducing their SACS animation sequence. Treat these as separate
claims when inspecting a trace.

## Interaction classification

`SimsOnlineServerHostMod/TerminalInteractionFilter.cs` is the authoritative
mod-side classifier. The controller applies a second policy in
`DedicatedServer/Mirroring/InteractionReplayPolicyResolver.cs`.

| Interaction key family | Filter result | Client-side result |
|---|---|---|
| `Sims3.Gameplay.Actors.Sim+SocializeWith+...` | Terminal and social-approach candidate | The client apply path deliberately falls through to the route arm, even if the target Sim resolves locally |
| `Sims3.Gameplay.Socializing.SocialInteractionA+...` | Route replay and social-approach candidate | Route toward the target; do not create the paired action |
| `Sims3.Gameplay.Socializing.SocialInteractionB+...` | Route replay and social-approach candidate | Route toward the target; do not create the linked response action |
| `Sims3.Gameplay.ActorSystems.ReactionInteraction+...` | Never replay | Drop the engine-pushed reaction from replay |

The `SocializeWith` terminal classification is not a promise that the client
runs the same social. `ServerHostMain` checks the client role and
`IsSocialApproachReplay` before terminal application; the social case takes the
route path instead of asking the client to select an action independently.
That avoids treating a local social choice as authoritative.

The controller's policy enum is `ReplayPolicy`:

| Value | Meaning |
|---|---|
| `NeverReplay` | Do not apply this interaction |
| `ReplayExact` | Attempt the terminal interaction |
| `ReplayAsRoute` | Apply movement toward a target |
| `ForceAlign` | Align to an authoritative posture container |
| `SlideOnly` | Leave movement to the position stream |

A route policy requires a usable target: `InteractionReplayPolicyResolver`
uses `TargetPosition` or a non-zero `TargetObjectId`. A missing target falls
back to `SlideOnly` rather than proving that a social began.

## What an interaction event proves

`SimsOnline.Protocol/ServerEvent.cs` defines
`ServerEvent.InteractionQueued` with an actor id, interaction id and key,
optional target object id, optional target position, and optional parameters.
A representative shape is:

```csharp
new ServerEvent.InteractionQueued(
    ActorObjectId: 1001,
    InteractionId: 42,
    InteractionKey: "Sims3.Gameplay.Actors.Sim+SocializeWith+Definition",
    TargetObjectId: 2002,
    TargetPosition: new Vector3Dto(12.5f, 0f, 8f));
```

This says the host queued the named interaction for the actor and supplied a
target hint. It does **not** say the client selected the same action, that the
partner accepted it, or that a conversation turn completed. The richer
`InteractionReplayBundle` also carries queue role, transition/posture metadata,
host tick and a `Parameters` map, but those fields do not serialize the
engine's full paired-social object graph.

Use the evidence at the layer that owns it:

| Evidence | Supports | Does not establish |
|---|---|---|
| `InteractionQueued` with target | Host queue change and an approach hint | Client route success or conversation membership |
| `ReplayAsRoute` dispatch | Client was asked to move toward a target | Social action choice, animation or relationship mutation |
| `Rl` record | Relationship values were sampled and sent | That the last route event caused those values |
| `Cv` record | Conversation membership/turn/topic state was sent | That a client ran the same social interaction or animation |
| A finished interaction observation | That one local queue entry ended | That both participants unwound cleanly |

## Batch coalescing

`DedicatedServer/Mirroring/InteractionQueuedCoalescer.cs` collapses redundant
route changes per actor. If a batch contains `SocializeWith`, it also removes
that actor's `SocialInteractionA` and `SocialInteractionB` route rows. The
current test `InteractionQueuedCoalescer.StripsSocialRoutesWhenSocializeWithPresent`
in `DedicatedServer.Tests/Program.cs` asserts this behavior.

This is a lossy boundary if the `SocializeWith` row has no usable target
position and an A/B row was the only movement hint. When debugging a Sim that
stays still, inspect the batch **after** coalescing and check both the target
object id and target position; a raw capture before coalescing is not sufficient
evidence of what the client received.

## Relationship and conversation state are separate streams

The host mod publishes relationship samples as `Rl` records and conversation
samples as `Cv` records. These are boundary-state updates, not social
interaction replay commands.

A relationship record starts with `Rl|simA|simB|liking|stc|commodity|commodityValue|perceivedSim`.
Newer forms can append long-term state, interaction bits, state-start ticks,
highest liking, last-greeted and last-talked ticks, plus the two Sims' known
trait lists. The parser accepts the older short record and rejects malformed
or partial extended records. This is implemented by
`SimsOnlineServerHostMod/RelationshipState.cs`.

A conversation record has this shape:

```text
Cv|simA|simB|inConversation|whoseTurn|whoTalkedLast|extraMembers|topics
```

`inConversation` is `1` or `0`; member ids are comma-separated or `-`. Each
topic is a name or `name:paramSimId:activeSimId`. The explicit absence record
`Cv|simId|simId|0|0|0|-|-` tells a client that a readable target is no longer
in a conversation. `ConversationPublisher` sends ended memberships before
new memberships so a Sim that switches conversations does not have the old
membership applied after the new one. `ConversationState` and its publisher
live in `SimsOnlineServerHostMod/ConversationState.cs`.

For the full field contracts, compatibility shapes and inventory boundaries,
see [Scope of mirrored state](scope-of-mirrored-state.md).

## Social-session probe tooling

`DedicatedServer/Mirroring/SocialProbeSessionGuard.cs` contains a TTL-based,
session-keyed lifecycle guard with phases such as `HostApproach`,
`HostSocialRunning`, `HostUnwind`, `Settled` and `Failed`. Its source explicitly
marks it **unwired**: production capture, coalescing and position-apply paths
do not call it. Do not infer suppression or cleanup behavior from the class
existing in the repository.

`scripts/probe_social_session.py` is a separate probe runner. Its `dry-run`
mode generates and archives probe material without contacting a bridge; the
live modes are guarded by required arguments and preflight checks. The
companion `scripts/probe_social_session_test.py` covers CLI validation,
report parsing, no-bridge dry runs and failure/cleanup classifications. The
runner's tests are not evidence that a live paired social was reproduced.

Useful test names include:

- `TerminalInteractionFilter` social classification assertions in
  `DedicatedServer.Tests/Program.cs`.
- `SocializeWithInteractionQueued.WithTargetTranslatesToApplyInteractionQueue`.
- `SocialInteractionAQueued.WithTargetTranslatesToApplyInteractionQueue`.
- `InteractionQueuedCoalescer.StripsSocialRoutesWhenSocializeWithPresent`.
- `SocialProbeSessionGuard` lifecycle and expiry tests in the same C# test file.
- The Python suite in `scripts/probe_social_session_test.py`.

## Failure modes and limits

| Symptom | Likely explanation | What to check |
|---|---|---|
| Sims meet but do not perform a matching social | Route replay is the current fallback, not paired-action playback | Compare route dispatch with `Rl` / `Cv` state; do not infer action identity from proximity |
| Target is visible but client action does not start | Social approach is intentionally routed instead of terminally selected | Check whether the client received `GoHere` intent and a valid target hint |
| One participant has stale posture or queue state | A route update cannot repair every local interaction lifecycle | Inspect current posture/queue snapshot and the mod's orphan-posture repair path |
| Sim does not move after a social batch | Coalescing may remove A/B route rows when `SocializeWith` is present | Inspect the coalesced batch for a remaining position or object target |
| Relationship or turn differs from animation timing | State records and animation/interaction execution are separate | Compare the authoritative `Rl` / `Cv` samples separately from the presentation stream |

The public `SimulationSnapshot` contains position, current interaction key,
queue count and posture metadata for Sims, but it does not make an interaction
key a social transcript. See `SimsOnline.Protocol/SimulationSnapshot.cs` and
[protocol](../protocol.md) for the network snapshot boundary.

## Related pages

- [Socializing posture](postures-and-transitions.md)
- [Interaction lifecycle](interactions-and-lifecycle.md)
- [Interaction coalescing](interactions-and-lifecycle.md)
- [Scope of mirrored state](scope-of-mirrored-state.md)
- [Carry and objects](carry-and-objects.md)
