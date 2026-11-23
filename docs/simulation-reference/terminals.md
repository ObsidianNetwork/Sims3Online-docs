# Terminals

A terminal interaction is the part of an activity that acts on a particular world object: reading a book, using a phone, sitting on a chair, or interacting with a plumbing fixture. The word “terminal” describes a replay classification in this project; it does not mean every game activity is safe to recreate on a client. A mirror needs a resolvable local target, compatible object state, required posture, and a policy that permits the interaction.

This page consolidates the live replay registry with the unimplemented or partial families for needs, hobbies, and phones. The authoritative current classifier is `SimsOnlineServerHostMod\TerminalInteractionFilter.cs`; policy selection is in `DedicatedServer\Mirroring\InteractionReplayPolicyResolver.cs`; the serialized interaction types are in `SimsOnline.Protocol\ServerEvent.cs` and `SimsOnline.Protocol\InteractionReplayBundle.cs`.

## Classification is not execution

There are three stages to keep separate:

1. **Capture** observes an interaction on the host and may emit its definition key, target, posture data, parameters, and queue role.
2. **Policy** classifies the captured bundle. A known terminal with a target can be `ReplayExact`; a posture transition with a known container can be `ForceAlign`; other targeted activities can degrade to `ReplayAsRoute`.
3. **Application** resolves the target in the client game and either queues a local interaction or creates `Terrain.GoHere`. A classifier match alone does not guarantee target identity, `Test()` success, or matching side effects.

The resolver's order is material. `NeverReplay` wins; a host `PolicyHint` wins next; then posture alignment is considered; then routing, then known terminal replay. Any unknown bundle with a target degrades to a route, and one with no target becomes `SlideOnly`. This prevents an unfamiliar type name from being passed blindly into the game.

The server event `interaction_queued` carries an `InteractionKey`, optional `TargetObjectId` and `TargetPosition`, and an optional string-to-string `Parameters` map. The richer `interaction_replay_queued` event wraps an `InteractionReplayBundle`, including queue/posture context and `PolicyHint`. `InteractionQueued.Parameters` is additive and can be absent or null on older-shaped messages.

Example of the public event shape, with the type key and identifiers shown as example values:

```json
{
  "$type": "interaction_queued",
  "actorObjectId": 1001,
  "interactionId": 482,
  "interactionKey": "Sims3.Gameplay.Objects.ReadBook+Definition",
  "targetObjectId": 901,
  "parameters": {
    "bookId": "901",
    "progressNormalized": "0.25"
  }
}
```

The parameter map is intentionally textual because interaction-specific keys vary. Known keys used by the current bridge include `bookId`, `progressNormalized`, `seatArea`, and `seatContainmentSlot`. Do not assume that a missing key has a useful default or that every interaction producer stamps the same set.

## Current replay registry

The current `TerminalInteractionFilter` is deliberately small and prefix-based. The key on the wire is commonly the reflected interaction definition type, not a short menu label.

| Current family | Current classification behavior | What the mirror can reasonably claim |
|---|---|---|
| `ReadBook+` | Terminal family; book-chain logic may handle the sequence specially. | Book identity and progress are handled through the read-intent path when a chain is recognized; this is not generic state-machine replay. |
| `GetBook+`, `ReadBookChooser+`, `Bookshelf_ReadSomething+` | Recognized book-chain precursors. | The `BookChainReadIntentResolver` recognizes a coherent chain and can dispatch `INJECT_READ_INTENT`; target resolution can still fail on a client without the host's book object. |
| `PutBookDown+` | Listed terminal family. | Local target and inventory state still determine whether the engine accepts it. |
| `InteractionsShared.Sit+`, `InteractionsShared.Stand+` | Listed; a bundle with posture-container/transition context is classified as `ForceAlign` before generic exact replay. | Seat alignment uses the posture path documented in [Postures and transitions](postures-and-transitions.md). |
| `Phone+BrowseTheWeb+`, `Phone+StreamVideo+` | Listed terminal prefixes. | These are classified as candidates, not proven cross-host equivalent; phone animation and object state may differ. |
| `Actors.Sim+SocializeWith+` | Listed, but the client apply path treats it as an approach route instead of selecting a local social. | Approach is the supported outcome; conversation and social posture parity are not claimed. |
| `SocialInteractionA+`, `SocialInteractionB+` | Social routing definitions, not terminal replay. | They are reduced to a GoHere toward the other Sim. |
| `ActorSystems.ReactionInteraction+` | Explicit `NeverReplay`. | Engine-generated reactions are not replayed. |

Prefixes are namespaced to avoid accidentally matching another class with the same leaf name. The mod also excludes GoHere/routing definitions from `IsTerminalReplay`; the two policies are not interchangeable.

## Book chain: the implemented special case

Book interactions illustrate why a chain should be modeled as a unit. A host may select a book, pick it up, route to furniture, sit, and then read. `ReadBook` can require a non-standing posture; on the host the game can select and run a Sit transition before the activity. A client that receives only the final terminal may fail its insertion test or select another seat.

`BookChainReadIntentResolver` looks for the relevant read head and merges the book/seat parameters from the chain. For a recognized bundle sequence, `MirrorEngine` calls `INJECT_READ_INTENT` rather than independently replaying every precursor. The intent carries book identity and normalized progress; the script mod resolves the local book and applies the read state. If the target book does not exist on a second client, the current host logs a target-not-found degradation and leaves position presentation in place instead of treating the whole mirror dispatch as failed.

| Concern | Current handling | Limit |
|---|---|---|
| Book target | `bookId` / target object id, with client-side resolution. | Object ids can be local to a particular game process/save instance. |
| Reading progress | `progressNormalized` is a string-valued semantic parameter. | It does not recreate every animation/state-machine event. |
| Preceding Sit | Seat parameters can pass through the bundle; posture alignment happens before ordinary terminal requests when present. | A missing or mismatched slot can seat the client elsewhere. |
| Missing target | Read-intent path degrades to slide-only when the book cannot be resolved. | The client will not show the reading action. |

The special case is tested in `DedicatedServer.Tests\Program.cs` and the resolver implementation. It is not evidence that every other long-running object interaction can use the same path.

## Needs and motive-driven interactions

Need-driven activities often look simple in the pie menu but contain many host-local decisions. The interaction can choose a fixture, queue for it, wait for privacy or availability, change clothes, enter a state machine, update a motive over time, and clean up afterward. The current terminal prefix list does not include Eat, UseToilet, TakeShower, BedSleep, or general cooking definitions; those interactions must not be presented as exact-replay support.

| Family | Typical engine work | Current mirror stance | Main mismatch risks |
|---|---|---|---|
| `UseToilet` | Routes to a toilet slot, waits for access, may choose a standing/sitting animation, changes bladder motive, and may update cleanliness. | Not a recognized terminal; if the host emits a target, fallback can route toward it. | Toilet id/availability, queue ownership, cleanliness, and the interaction's seat parameters can differ. |
| `TakeShower` / bath | Routes to a shower/tub, checks broken/privacy state, can switch outfits, runs shower stages, and changes hygiene. | Not a recognized terminal; route-to-object is the safe fallback when target data exists. | Privacy/door situation, outfit state, broken fixture, tub variants, and motive state. |
| `Eat` | May route to food or a surface, sit first, pick up a serving, run an eating loop, then destroy or put down a plate. | Not a recognized terminal; do not queue it as if food identity were portable. | Spawned servings, carry/inventory state, recipe/quality, and seat choice. |
| `BedSleep` | Enters or promotes to a bed posture, runs sleep stages, changes energy, and wakes based on clock/motive conditions. | Not a recognized terminal; posture observation is not full sleep replay. | Bed part occupancy, `CanShareBed`, wake time, energy progression, and local autonomy. |
| Stove/fridge cooking | Multi-step selection, ingredients, object use, spawned servings, and an Eat follow-up. | Outside the terminal registry. | Multiple object identities and state transitions need to line up across hosts. |

A few concrete gates explain why these families diverge:

- Toilet interaction logic is not simply “bladder below a threshold”; an autonomous choice may be gated by the routing fixture and its current in-use state, while motive changes occur during the run.
- Autonomous shower choices can be rejected when hygiene is already full, the Sim is a robot, the fixture is broken, or a tub-specific condition blocks use. User-directed starts need not have identical autonomous tests.
- Eating depends on the specific edible object. A freshly spawned single serving or a carried plate is a weaker identity anchor than a stable fixture in a shared save.
- Sleeping runs for simulated time and can be interrupted by wake conditions. Replaying a single queue insertion would not mirror the sleep-stage loop or bed ownership.

These are engine behavior summaries from the historical interaction studies; the current mirror code's factual boundary is the smaller prefix list above. A family should only move into exact replay after the actual emitted key, target identity, parameters, and side effects have been verified on both instances.

## Hobbies and skill objects

Chess, sketching, sculpting, workout equipment, and similar activities usually combine an object target with a long-running interaction. Many also use a chair or station posture and update skill progress while the interaction runs. Multiplayer activities add a synchronized second actor. A route to the object is therefore not equivalent to “the Sim is practicing.”

The current classifier does not include general hobby definitions such as `PracticeChess`, `CreateSculpture`, or workout terminals. Unknown targeted keys can still reduce to a route-to-object, but the client will not receive a guarantee of matching skill progress, created props, score, or interaction state. The public snapshot's position and interaction key do not encode all of those fields.

A replayability assessment for a new hobby family should answer all of these before adding a prefix:

1. Is the target object the same on every participant, including objects spawned during the session?
2. Does the interaction require a seat, workstation slot, or paired Sim posture?
3. Are skill progress, object completion state, or temporary props expected to match?
4. Can the client run the same loop without advancing a second authoritative simulation?
5. What explicit fallback should run when the target or prerequisite is absent?

## Phone and social activity policy

Phone definitions are present in the terminal prefix list, but a prefix match is only classification. `TryApplyInteractionFields` still requires the target object to resolve locally before it can queue an exact terminal. Do not infer that every autonomous phone key or every runtime phone object is mirrored exactly just because the generic prefix exists.

Social interaction keys are more constrained. `SocializeWith` is recognized for classification but the client special-cases it to GoHere so its own local social-selection algorithm does not choose a different interaction. `SocialInteractionA` and `SocialInteractionB` match the route prefixes and are similarly reduced to approach movement. This protects the mirror from constructing a social state machine without the host's conversation context.

## Failure and degradation behavior

| Condition | Result |
|---|---|
| A known terminal has no target id. | Exact replay is rejected as missing its target; it is not guessed from the interaction name. |
| A known terminal target id does not resolve in the client. | The mod reports target-not-found; social approach with a position may still route to the Sim, otherwise no exact interaction is run. |
| Unknown key has a target position or object id. | Mirror apply degrades to GoHere-to-target rather than queuing an unknown definition. |
| Unknown key has no routable target. | Mirror apply acknowledges a no-op; position presentation remains the available representation. |
| A posture-requiring terminal is sent before alignment. | It can fail `Test()` or queue timing; the engine's live posture, not the bundle's string, decides. |
| Two clients do not share object identity/state. | One may route or play the terminal while the other cannot resolve the target. |

The `TargetPosition` fallback also has a coordinate requirement: the host mod snaps a route destination to terrain height when the reported Y differs from terrain by more than `0.5` world units. This avoids routing to a point below an elevated walk surface; see [Routing](routing.md).

## Verification

Classifier and dispatch behavior is covered by tests in `DedicatedServer.Tests\Program.cs`, including:

- `TerminalInteractionFilter.AllowsLiveBookChain`
- `TerminalInteractionFilter.NeverReplayRejectsReactionInteraction`
- `TerminalInteractionFilter.FailsClosedForRoutingAndGoHere`
- `InteractionReplayPolicyResolver.TerminalWithTargetUsesReplayExact`
- `InteractionReplayPolicyResolver.UnknownWithTargetUsesReplayAsRoute`
- `InteractionReplayPolicyResolver.UnknownWithoutTargetUsesSlideOnly`
- `EventToBridgeCommandTranslator.InteractionReplayQueuedForceAlignEmitsApplyForceAlign`

The tests verify policy selection and command shape. They do not establish live parity for food, bathroom fixtures, showers, sleep, hobbies, or phones.

## Related pages

- [Postures and transitions](postures-and-transitions.md)
- [Routing](routing.md)
- [Protocol](../protocol.md)
- [Simulation model](../simulation.md)