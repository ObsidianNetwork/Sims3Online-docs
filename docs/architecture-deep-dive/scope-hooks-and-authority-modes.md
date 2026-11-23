# Scope hooks and authority modes

The C# controller maintains two independent policies: server simulation scope and per-client mirroring interest. The in-game mod can apply scope to a fork-provided hook; the client authority gate is another bridge command. Neither policy is a substitute for command authorization.

## Three distinct sets

| Policy | Input and implementation | Result |
|---|---|---|
| Server simulation scope | `SimulationScopePolicy.Compute(claims, sims)` | Sorted `LotIds` plus every claimed `SimObjectId`. |
| Per-client interest | `MirroringInterestScope` | Snapshot/event filtering for a receiving client. |
| Client local gate | `ClientAuthorityPolicy.Compute(mode, lotMode, ownedIds)` | Gate mode and a sorted native-Sim set, derived from lot-local mode. |

`SimulationScopePolicy.Compute` delegates lot selection to `ComputeInterestLots`. A claimed actor absent from the sampled Sim list remains in the `SimObjectIds` list, while no lot can be derived for it until a nonzero `LotId` is observed. A claim is therefore not the same as a current lot assignment.

## Scope computation and change detection

`SimulationScope` contains read-only `LotIds` and `SimObjectIds` lists. `SameAs` compares both sequences, so callers sort before constructing a new policy state. `ShouldPublish` sends the first state (including an empty state), skips an equal accepted state, and suppresses only the exact failed value until the retry duration passes. The production constant `RetryAfterFailure` is 30 seconds.

```text
claims = { 200 -> player 9, 100 -> player 2 }
snapshot = [ { object=100, lot=700 }, { object=200, lot=500 } ]
computed scope = lots [500,700], sims [100,200]
```

The example shows the sorted value; the source policy does not use a player's household as the list key.

## Bridge representation

`BridgeLineCodec.EncodeCommand` maps the typed `SetSimScope` command to `SET_SIM_SCOPE|corr|lots|sims`. `SimScopeWire.Format` uses invariant decimal strings, commas between ids, and `-` for an empty sequence. `Parse` rejects signs, whitespace, empty interior elements, and values outside the `ulong` range by returning `null`.

```text
SET_SIM_SCOPE|<corr>|1000,2000|100
SET_SIM_SCOPE|<corr>|-|-        # explicit empty scope
```

`ServerHostMain.HandleSetSimScope` resolves each requested lot with `LotManager.GetLot` and drops missing or world lots; its response reports `droppedLots=N`. If the fork type `SimsOnline.Hooks.SimScopeHook` is absent, the response says `hook=absent` rather than claiming that the scope was enforced. This is a local integration contract, not part of the player-facing network protocol.

## Household and lifecycle behavior

The mod's scope handler passes the Sim and lot arrays to the gameplay hook when that type is found, then calls the lifecycle application path and reads the hook's description. `SimScopeWire.AnyClaimed` answers whether a household member id is in a claimed set; null/empty inputs and id zero do not match. This predicate is an input to household policy, not a general rule that every household member is player-owned.

The implementation has an atomicity edge to keep in mind: `HandleSetSimScope` assigns the cached `sScopeSims` when the Sim list parses, before it rejects the request if the lot list failed to parse. The hook itself is not called on that error path, but the cached Sim list may already have changed. The source tests pin wire parsing and pure scope calculations; they do not prove this bridge handler's malformed-request state is atomic.

## Client authority modes

The configured mode comes from `SIMSONLINE_AUTHORITY_GATE` and `TryParseMode` accepts `off`, `observe`, and `enforce` without case sensitivity. Invalid text returns `false` and supplies `DefaultMode`, which is `Enforce`. `Compute` sorts ids and includes owned Sims only if `LotLocalAuthorityMirrorGate.OwnsNativeSimulation(lotMode)` is true.

The bridge command is `SET_AUTHORITY|corr|mode|owned-list`. `HandleSetAuthority` validates the mode and complete id list; it reports `hook=absent` if the authority hook is unavailable. Even with the hook present, `ClientAuthorityPolicy.GateReplacesAutonomySuppression` returns `false`: queue-insert gating and mirror autonomy suppression remain separate operations.

## Tests and limits

`DedicatedServer.Tests/Program.cs` includes named contracts for sorted scope calculation, 30-second failure retry, mode parsing, native-set derivation, empty `-` encoding, and malformed wire lists. Test names include `SimulationScopePolicy.ComputesSortedClaimedLotsAndSims`, `SimulationScopePolicy.PublishesOnChangeWithRetryBackoff`, `ClientAuthorityPolicy.NativeSetFollowsLotMode`, and `MirroringInterestScope.FiltersIdleTownieOnInterestLot`.

Those tests establish managed behavior and serialization. A passing policy test does not establish that a specific fork is installed, that a hook resolved in the live game, or that all game lifecycle callbacks behave correctly. For live claims, inspect the host reply (`hook=present` versus `hook=absent`) and test the actual in-game behavior.

## Failure modes

- A claimed Sim without a snapshot lot remains claimed but contributes no lot.
- A missing or world lot in the requested list is dropped by the host rather than treated as a valid simulation lot.
- A bridge older than the current host handler can reject `SET_SIM_SCOPE` or `SET_AUTHORITY`; the 30-second retry prevents repeated requests for the same failed state on every broadcaster pass.
- Mode `Enforce` with a missing hook is configured policy, not enforcement.
- Scope, interest, and claim authorization must not be conflated.

## Related pages

- [Authority and ownership](authority-and-ownership.md)
- [Scope, interest, LOD, and dormancy](scope-interest-lod-dormancy.md)
- [Boundary coverage and mirror gaps](boundary-coverage-and-mirror-gaps.md)
- [Bridge and formats](../bridge-and-formats.md)
