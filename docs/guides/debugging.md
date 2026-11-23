# Debugging

Debug by locating the first boundary that did not complete. A visible symptom such as a frozen Sim can be caused by a server decision, a missing event, a client decode/apply failure, or the game's own presentation; restarting immediately destroys evidence that distinguishes them.

## Triage in boundary order

| Boundary | Evidence | Next check |
|---|---|---|
| Process start | `stackctl status`, controller log, lifecycle state file | Executable, arguments, startup exception. |
| Game host | Host bridge health and mod build stamp | Mod load, bridge endpoint, game process identity. |
| Public transport | Listener state and connection-close log | Address, port, codec, peer disconnect. |
| Session join | `JoinTrace` and join response | Role, player identity, capability negotiation. |
| Snapshot | Client snapshot log / optional snapshot side file | Decode, event ordering, resync. |
| Mirror | `MirrorEngine` and bridge response lines | Local object mapping, unsupported action, timeout. |
| Visible result | `PresentationStream`, animation and position evidence | Game-side actor selection, presentation handler, actual render. |

Start with `python scripts/stackctl.py status --strict` and `python scripts/stackctl.py logs --lines 100`. If status returns 3, bridge checks were skipped due to another active claim; it is not a clean health result.

## Compare server and client evidence

Use the same time window on both logs. `JoinTrace` captures join phases; `ClientListenerServer` records connection close context including player, peer, lifetime, byte counts, last operation, and close reason. A JSON-encoding failure is logged with the event type and any non-finite fields before the exception propagates. These details answer different questions; a successful TCP connection is not a completed join.

For a managed failure, reduce to one player and protocol mode before adding a game mirror. For a live failure, preserve the server and client logs, run-state summaries, exact commands, build identity, and whether tracing was enabled. Do not share full snapshots or bridge frames without reviewing them for world data and identifiers.

## Decision table

| Observation | Likely fault area |
|---|---|
| Server controller exits before listener bind | Host startup or configuration. |
| Listener is up, join rejected | Protocol mode, role/capability or session policy. |
| Join succeeds, client baseline absent | Snapshot publication or decode. |
| Baseline is received, mirror emits errors | Local host bridge or event application. |
| Mirror reports applied but animation differs | Presentation/actor state, not socket transport. |
| Error only appears with timing trace enabled | Instrumentation overhead may be changing the timing; repeat without it. |

## Related pages

[Logging and diagnostics](logging-diagnostics.md) | [Test strategy](test-strategy.md) | [Scenario catalog](scenario-catalog.md) | [Run a server](run-server.md) | [Run clients](run-client.md)
