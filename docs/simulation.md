# Simulation, interactions, and presentation

## From player intent to visible result

The project separates a player's request, the simulation's response, and the way that response
is shown in each local game:

1. The player's game reports an input intent to its controller.
2. The controller sends a command such as `queue_interaction` to the server.
3. The server associates the command with a joined player and checks the relevant ownership
   rules before invoking the authoritative game.
4. The server reports command acceptance or rejection and publishes supported interaction,
   snapshot, or presentation events.
5. Client controllers translate those events into work for their local game.

An accepted command means the server accepted the request; it does not guarantee that the
underlying game can complete the interaction. The game may reject, interrupt, or fail it later.
The protocol therefore distinguishes queued, started, completed, and failed interaction events.

## Ownership and claims

A Sim claim associates a Sim identifier with a player. Claim and release events communicate
ownership changes, and later commands are checked against the joined player's identity and the
simulation's ownership policy. A connection is not allowed to speak for another player merely
by putting a different player ID in a command.

The repository also contains an explicit client-native lot-authority mode. When enabled, the
server can request an owned-lot export and reconcile it into the host game. This mode has
separate validation and checkpoint rules; it is not the default rule that every client may
write arbitrary state.

## Routing and execution

Routing is one part of a game's interaction lifecycle, not a synonym for the entire interaction.
A destination request, a game's navigation plan, an interaction's execution state, and the
client's visible movement are different kinds of evidence. A route can be unavailable even
when the command was valid, and a client can display movement that has not yet converged with
the latest authoritative snapshot.

When diagnosing a problem, identify which layer failed:

- **Command rejected:** connection, identity, ownership, or command validation.
- **Interaction failed after acceptance:** the game could not carry out the requested work.
- **Snapshot or reconciliation mismatch:** a client is stale or applying incomplete state.
- **Presentation mismatch:** visible movement, posture, animation, or effects differ from the
  server's stream.

The integration relies on The Sims 3 for gameplay and routing. Documentation does not promise
that a route or interaction works in every expansion, custom-content setup, or game build.

## Mirroring and presentation

The server distributes simulation snapshots and selected events. The client mirror translates
those events into operations for the local game. The presentation stream can include animation
records; it is separate from the authoritative snapshot and must not be used as proof that the
client independently simulated the same outcome.

The client acknowledges applied snapshot ticks. If it falls behind, the server can signal that
a delta should be treated as a full replacement. This repairs a mirror's state view but does
not make the network protocol authenticated or encrypted.

## How to verify behavior

Controller tests are suitable for command validation, event translation, and state-machine
logic. Claims about actual game behavior need a live scenario with the required game processes.
See [testing](testing.md) and [tooling](tooling.md); report the scenario verdict rather than
extrapolating from a nearby test.
