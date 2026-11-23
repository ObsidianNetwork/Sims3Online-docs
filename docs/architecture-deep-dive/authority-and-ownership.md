# Authority and ownership

The running C# session has separate owners for connection membership, Sim claims, game outcomes, and client presentation. The distinction matters because a claim authorizes input routing; it does not make the claiming client a second simulator.

## State ownership in the current runtime

| State | Owner | Current source evidence |
|---|---|---|
| Connection membership and claim map | `AuthoritativeSession` | `_claims` maps a Sim object id (`ulong`) to a player id (`int`). |
| Command permission | `AuthoritativeSession` | `GateOwnedActorCommand` compares the command player with the owner in `_claims`. |
| Interaction outcome and live world | Server game process | The controller forwards accepted requests through `IHostBridge`; later lifecycle observations describe execution. |
| Client mirror | Each client controller and local game | Server events and snapshots are applied for presentation. |
| Save contents | Server game/save path | Client render state is not a persistence source. |

The current-state distinction is that state display and claim ownership are not evidence of simulation authority. This page follows the current implementation where earlier architectural notes differ.

## Claim command path

For actor-scoped commands, `AuthoritativeSession.GateOwnedActorCommand` looks up the actor in `_claims`. If another player owns it, the command is rejected before the host operation. The check is ownership-aware rather than requiring every actor to have a claim: an unclaimed actor is not rejected by this particular owner-mismatch branch, although each command still has its own validation and host/game tests.

The claim implementation rejects a claim when a different owner already holds it, stores the mapping, registers the autonomy gate, and emits `SimClaimed` plus `SimAutonomyChanged`. Release verifies that the requesting player is the owner, removes the entry, clears the gate, and emits `SimReleased` and an autonomy change. Disconnect cleanup releases the departing player's claims.

```text
player 4 joins
claim(actor=0x100, player=4)     -> claim map: 0x100 => 4
queue(actor=0x100, player=5)     -> rejected: actor is claimed by player 4
release(actor=0x100, player=4)   -> claim removed; release/autonomy events
```

The identifiers above are illustrative. A player id is session identity; a Sim object id is a game-world reference. Neither is authentication for an internet-facing deployment; see [security](../security.md).

## Client gate is not server authorization

`ClientAuthorityPolicy` calculates local game gate state. Its `TryParseMode` accepts case-insensitive `off`, `observe`, and `enforce`; an unknown value returns `false` and the default mode (`Enforce`) through the out parameter. `DefaultMode` is `Enforce`, and the environment variable name is `SIMSONLINE_AUTHORITY_GATE`.

`Compute` only places claimed Sim ids in `NativeSimObjectIds` when `LotLocalAuthorityMirrorGate.OwnsNativeSimulation(lotMode)` returns true; it sorts ids before equality comparison. That policy state is distinct from the server's command check. The local gate depends on the matching gameplay hook being present. It does not authenticate a remote connection or validate a request sent to the server.

An important current limitation is explicit in the implementation: `GateReplacesAutonomySuppression` always returns `false`. The script-level interaction gate does not replace the separate mirror autonomy suppression path. Do not describe the gate alone as preventing all client-native autonomy.

## Simulation obligation versus interest

`SimulationScopePolicy.Compute` builds two sorted lists: lots derived from claimed Sims' current nonzero `LotId`s, and all claimed Sim object ids (including a claimed Sim absent from the current snapshot). The lot calculation delegates to `MirroringInterestScope.ComputeInterestLots`. This scope is sent to the server game; it does not alter claim ownership or choose which player can issue a command.

`MirroringInterestScope` separately filters each client's mirrored state. A claimed Sim is retained in filtered snapshots; non-claimed Sims on interest lots are retained only when active under the interaction-queue rule. The two policy classes therefore answer different questions: what the server should simulate at higher fidelity, versus what a particular client receives.

## Publication, retries, and wire fields

Both `SimulationScopePolicy` and `ClientAuthorityPolicy` publish on change and use a 30-second retry holdoff for the exact state that just failed. An initially unpublished empty state is sent, which clears host state left by an earlier controller. A changed state is not held back by a failure for a different state.

`SimScopeWire.Format` writes comma-separated unsigned decimal ids and uses `-` for an empty list. `Parse` accepts `-` or the empty string as empty, uses invariant-culture `NumberStyles.None`, and returns `null` if any item is malformed. The bridge command shapes are:

```text
SET_SIM_SCOPE|<correlation>|<lot-ids-or->|<sim-ids-or->
SET_AUTHORITY|<correlation>|off|observe|enforce|<owned-sim-ids-or->
```

These are local controller-to-mod bridge records, not public TCP message formats. For example, an empty scope is `SET_SIM_SCOPE|<correlation>|-|-`.

## Failure modes and verification

- A different player's command for a claimed actor must fail before the host bridge is called.
- A stale claim can block a legitimate owner until release or disconnect cleanup runs.
- `Enforce` is not effective when the expected game hook is absent; the controller cannot inject the hook into a stock assembly.
- A valid command acknowledgement does not mean the game interaction completed; execution lifecycle is a later boundary.
- Scope publication can be retried after rejection; a just-failed identical value is held for 30 seconds.
- `ServerHostMain.HandleSetSimScope` currently updates its cached `sScopeSims` when that field parses before it checks whether the lot field also parsed. A malformed lot field can therefore produce an error after this cached field changed; handler atomicity is not covered by the formatter/parser tests.

The named policy tests live in `DedicatedServer.Tests/Program.cs`: `SimulationScopePolicy.ComputesSortedClaimedLotsAndSims`, `SimulationScopePolicy.PublishesOnChangeWithRetryBackoff`, `ClientAuthorityPolicy.NativeSetFollowsLotMode`, and `ClientAuthorityPolicy.PublishesOnChange`. The `SimScopeWire` parser cases include the empty marker, large `ulong` values, and whole-field rejection for malformed lists. These are unit-level contracts, not live-game hook acceptance.

## Related pages

- [Scope hooks and authority modes](scope-hooks-and-authority-modes.md)
- [Scope, interest, LOD, and dormancy](scope-interest-lod-dormancy.md)
- [Server-authoritative simulation](server-authoritative-simulation.md)
- [Protocol](../protocol.md)

## Source anchors

| Claim in this page | Implementation anchor |
|---|---|
| Actor claim checks and command gating | `DedicatedServer/Authority/AuthoritativeSession.cs`, `GateOwnedActorCommand` |
| Server claim ownership and claim lifecycle | `DedicatedServer/Authority/AuthoritativeSession.cs`, claim/release handlers and disconnect cleanup |
| Client-local enforcement policy | `DedicatedServer/Mirroring/ClientAuthorityPolicy.cs`, `TryParseMode`, `Compute`, `GateReplacesAutonomySuppression` |
| Lot and Sim high-fidelity simulation scope | `DedicatedServer/Mirroring/SimulationScopePolicy.cs` |
| Client-specific interest filtering | `DedicatedServer/Mirroring/MirroringInterestScope.cs` |
| Lot authority ownership predicate | `DedicatedServer/Mirroring/LotLocalAuthorityMode.cs`, `OwnsNativeSimulation` |
| Bridge serialization of authority/scope records | `SimsOnlineServerHostMod/ServerHostMain.cs` and `SimsOnlineServerHostMod/SimScopeWire.cs` |

These are separate contracts: server command authorization does not delegate world simulation;
client authority mode does not authenticate a peer; and interest filtering does not decide which
player owns a Sim. The corresponding policy tests are in
`DedicatedServer.Tests/Program.cs`; a passing policy test does not prove that the matching game
hook was loaded in a particular Sims 3 process.
