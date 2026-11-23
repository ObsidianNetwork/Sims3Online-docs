# One-simulator decision

The architectural decision is that one server-side Sims 3 game simulates
every Sim in the shared sessionâ€”including the Sim a player controls. Client
games present server-produced outcomes and forward player input; they are
not additional simulators for local player characters.

This decision sharpens â€œserver authoritative.â€ A weaker interpretation would
allow the server to own the save while clients still run their own Sims and
later reconcile changes. That creates two sources of gameplay outcomes for
one world. The one-simulator model removes that ambiguity for the default
session path.

## Decision summary

| Question | Decision |
|---|---|
| Who simulates shared Sims? | The server-side game process |
| What does a client do? | Captures player intent and presents server-originated state |
| What does a Sim claim mean? | Input routing and ownership policy, not simulation transfer |
| Who commits shared persistence? | The server-side world/session |
| Is client-native lot simulation the default? | No; it is not the current authority model |

## Rationale

### Avoid dual authority

If a client and server both execute the same Sim's interactions, they can
disagree about route, posture, object state, or whether the action was
valid. A reconciliation layer must then choose which state wins and explain
the consequences. Making the server the only simulator gives the system one
answer to preserve.

### Use the engine for gameplay

The server's game already contains the world and its gameplay systems. The
controller can inject a validated request into that engine and observe the
result. Clients use their game for input and local rendering, not as
replicas expected to make matching gameplay decisions.

### Prefer measured presentation

Re-running interactions on a client can fail when its world, object state,
or engine scheduling differs. Recording shared engine-boundary operations
gives the client a more direct presentation path. It still requires identity
mapping and coverage; it is not a complete serialization of the game.

## Consequences

- A player's action incurs a network round trip before the authoritative
  game begins it.
- Client UI must mirror relevant queue, need, money, and world information
  that local gameplay would otherwise provide.
- Client-side autonomy and other gameplay side effects must be suppressed
  for mirrored Sims where required by the integration.
- Only the server copy is persistent for shared world progress.
- Interest filtering can reduce what each client receives without
  transferring simulation authority.
- An in-game hook may be needed to enforce presenter-only behavior inside a
  client process.

## Alternative that is not the default

A lot-local model can let clients simulate some Sims and reconcile changes
to the server. It may reduce server work or network traffic, but it creates
handoff rules, conflict resolution, save reconciliation, and transition edge
cases. The current model does not use this as its normal path. This summary
does not claim that every vestigial compatibility path has been deleted.

## Guardrails for implementation

- Do not infer simulation authority from a Sim claim.
- Do not treat client observations as the authoritative outcome.
- Do not commit client state as a shared save without an explicit ownership
  mode.
- Keep input intent and server result as distinct protocol concepts.
- Keep client presentation records separate from gameplay-state snapshots.
- Test that an accepted player request runs on the server and is not
  independently re-run by the client.

## Current implementation anchors

`DedicatedServer/Authority/AuthoritativeSession.cs` owns the claim map and checks actor ownership before forwarding queue/cancel/dialog operations. `DedicatedServer/Mirroring/ClientAuthorityPolicy.cs` computes local gate state; its default mode is `Enforce`, but it explicitly keeps autonomy suppression separate. The game-side handlers for `SET_AUTHORITY` and `SET_SIM_SCOPE` are in `SimsOnlineServerHostMod/ServerHostMain.cs`.

Managed tests include `ClientAuthorityPolicy.NativeSetFollowsLotMode` and `ClientAuthorityPolicy.PublishesOnChange` in `DedicatedServer.Tests/Program.cs`. They establish policy computation and publication, not live fork-hook coverage. For the separate current-state contract, see [authority and ownership](authority-and-ownership.md).
## Failure modes / limits

- The round trip is visible as latency; local speculative effects must not
  be mistaken for outcomes.
- A client without the required gameplay fork may not enforce all native
  simulation gates.
- Not every gameplay state is mirrored yet; one simulator does not imply
  visual parity.
- Old client-native logic can remain in the codebase without being part of
  the default session path.

## Related pages

- [Authority and ownership](authority-and-ownership.md)
- [Server-authoritative simulation](server-authoritative-simulation.md)
- [Scope, interest, LOD, and dormancy](scope-interest-lod-dormancy.md)
- [Decision summaries](decision-summaries.md)

## Decision consequences by subsystem

| Subsystem | Expected server behavior | Expected client behavior |
|---|---|---|
| Interaction queue | Validates and executes queue changes | Shows the authoritative queue and forwards picks/cancels |
| Autonomy | Decides autonomous behavior for the world | Suppresses competing autonomous work for mirrored Sims |
| Needs and motives | Advances gameplay values | Applies server values for UI and display |
| Money | Owns household/world transactions | Displays applied values; does not commit shared funds |
| Sim lifecycle | Creates/despawns authoritative actors | Resolves aliases and applies supported spawn/despawn state |
| World clock | Advances canonical time | Displays synchronized time and speed |
| Save/load | Commits shared world state | Local copies are not shared-world commits |
| Animation and routes | Produces the actual outcome | Replays supported presentation data |

These are architectural responsibilities; exact coverage can vary by
integration build. See [status](../status.md) before relying on any
subsystem as complete.

## Client presentation still has local game state

â€œClient simulates nothingâ€ is a statement about shared gameplay outcomes,
not a claim that the process is inert. The local game still runs its
renderer, applies mirrored operations, receives input, and may perform
cosmetic idle or interface behavior. The integration must distinguish
permitted presentation work from gameplay side effects such as autonomous
interactions, need updates, aging, or local spawning.

This distinction is why a controller-only mirror filter cannot enforce the
architecture by itself. A local game may continue to run native code unless
the required gameplay hook is installed and configured.

## Claims are not leases on world truth

A claim can be treated as an input permission with a lifetime. It is not a
lease that transfers the Sim to a client's process or changes which save is
canonical. Claim release does not reverse server outcomes that already
happened. It only changes future authorization and derived scope.

## Failure classification matrix

| Symptom | Authority question | Diagnostic owner |
|---|---|---|
| Client Sim autonomously queues action | Did the local gate actually enforce? | Client fork/mod and authority policy |
| Player command rejected | Is connection joined and claim current? | Session state and network logs |
| Server action accepted but absent | Did the game enqueue it? | Host bridge and game lifecycle |
| Server and client motive differ | Did client apply current state? | Snapshot/mirror and timing |
| Animation differs only | Is the seam recorded and replayed? | Recorder and presentation path |
| Shared save differs | Which side committed it? | Server persistence boundary |

## Migration from any dual-authority mode

A transition from client-native execution to one simulator must handle more
than a mode flag. It needs to disable client inserts and autonomy, route
player picks to the server, supply enough state for the client UI, map
identities across processes, and ensure the client does not commit its local
world. It also needs a clean rule for any pre-existing client-side state
that was already mutated.

A safe transition is observable: the client reports its authority mode and
native Sim set, the server reports claims and scope, and tests verify that
an attempted local gameplay insert is blocked or forwarded rather than
executed.

## How to verify / test

- Inspect the configured authority mode and whether the matching in-game
  hook is present.
- Verify an empty native Sim set in the presenter-only default.
- Trigger a player-directed menu pick and observe the server execution
  lifecycle.
- Check that local autonomy does not create a second action for the same
  Sim.
- Verify that disconnect and reconnect do not transfer the save owner.
- Run a live scenario with both claimed and unclaimed Sims in the same
  client view.

## Failure modes / limits

The decision chooses a source of truth, not a complete feature set. Missing
mirror fields can still produce poor UI or a visually incorrect client. An
implementation bug can violate the decision even when the design record is
clear.

## Related pages

- [Authority and ownership](authority-and-ownership.md)
- [Decision summaries](decision-summaries.md)
- [Boundary coverage and mirror gaps](boundary-coverage-and-mirror-gaps.md)

## Test matrix for the decision

The one-simulator invariant should be tested at multiple layers:

| Test | Setup | Pass condition |
|---|---|---|
| Controller authorization | Joined player sends a valid claimed-Sim command | Server accepts only under current ownership policy |
| Cross-player isolation | One connection names another player's identity | Request is rejected before game dispatch |
| Client gate state | Client runs presenter mode | Reported native Sim set is empty and gate capability is present |
| Local pick forwarding | Player selects an interaction | Local insert is captured/blocked and server receives intent |
| Server execution | Accepted request reaches the server game | Game lifecycle reports queue/start/outcome |
| No duplicate client action | Same Sim is visible on client | Client does not independently execute the shared interaction |
| Late join | Client joins after world changes | Snapshot establishes current server state |
| Persistence | Server saves and restarts | Shared state survives server save/load, not client local save |

A unit test cannot prove a game-process hook is installed and effective. At
least one live scenario should exercise the authority gate and actual
user-directed input.

## Latency and user experience

The consequence of remote execution is a round trip before the authoritative
game starts. UI can show a pending command, but must be able to withdraw
that pending state on rejection or later failure. Immediate local animation
can be tempting as a prediction strategy, yet it risks making an unconfirmed
outcome look authoritative. If prediction is introduced, it needs a rollback
or correction policy and must remain presentation-only until server
confirmation.

## State completeness and menus

A client can correctly obey authority while still presenting an incomplete
menu because relationship, inventory, household, or target state is stale.
Such a defect is not evidence that the client should simulate the
interaction. The state-mirroring contract should be completed or the UI
should communicate the unsupported option.

## Migration guard

When retiring a client-native execution path, verify all the coupled gates:
interaction inserts, autonomy, needs, money, lifecycle creation, world
services, and save ownership. A setting that blocks only player-directed
queue insertion may leave autonomously generated client actions active. The
gate report should identify which behavior categories are actually
controlled.

## Failure modes / limits

The architecture can be violated by a missing fork hook, stale client state,
or an unguarded local engine subsystem. Treat â€œone simulatorâ€ as an
invariant to assert and observe, not as an assumption that follows from the
process diagram.

## Related pages

- [Boundary coverage and mirror gaps](boundary-coverage-and-mirror-gaps.md)
- [Authority and ownership](authority-and-ownership.md)
- [Server-authoritative simulation](server-authoritative-simulation.md)
- [Testing](../testing.md)
