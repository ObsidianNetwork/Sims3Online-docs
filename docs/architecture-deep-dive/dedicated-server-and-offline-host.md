# Dedicated server and offline host

“Dedicated server” describes the role of the server controller and its game
process in a multiplayer session. It does not imply that the simulation is
an independent lightweight executable. The current server path still relies
on a Sims 3 game process with the host integration loaded.

## Current dedicated-host shape

The dedicated controller can start or attach to a server-side game process,
connect to the in-game mod through a local bridge, load the shared world,
and coordinate player connections. The controller owns session mechanics;
the game process owns the simulation. See [process model](process-model.md)
and the [server CLI reference](../server-cli.md).

| Term | Meaning in this repository | Not implied |
|---|---|---|
| Dedicated controller | Session and network coordinator | A standalone game simulator |
| Server game | Authoritative Sims 3 process | A headless rewrite of gameplay |
| Hidden-window operation | The game window is not shown during operation | No desktop, no graphics runtime, or zero rendering work |
| Offline host experiment | Isolated runtime research outside a live shared session | The production multiplayer server |

## Hidden versus headless

A bounded measurement found that hiding the game window retained
approximately the visible-window simulation tick rate, while minimizing
reduced it. This supports hidden-window hosting as a practical operational
mode for the tested configuration. It does not establish that the game boots
without a desktop, GPU, assets, or normal interactive session; nor does it
establish behavior on every operating system or game build.

The term *headless* is therefore potentially ambiguous. In operational
documentation, say whether the game is hidden, minimized, or running without
a desktop session. Those are different environments.

## Offline interpreter research

An offline host can mean a small experimental process that loads part of the
game's managed runtime without starting a playable world. Such work can help
answer narrow questions about interpreter execution or API behavior. It is
not the runtime path used to host an online world.

The distinction is architectural:

| Property | Live server game | Offline experiment |
|---|---|---|
| Loads a playable world and its services | Yes | Not necessarily |
| Advances the shared simulation | Yes | No, unless a specific simulation harness is built and validated |
| Receives player session commands | Through controller | Not part of its purpose |
| Suitable as a server replacement | Current host role | No evidence of replacement compatibility |
| Useful for isolated runtime questions | Sometimes | Yes, within its declared harness coverage |

Offline execution of managed methods is not proof that the native services,
world services, save pipeline, routing, or gameplay loop are available. A
method can return a plausible value in an isolated runtime while depending
on stubs or missing engine services in a live game.

## Why a lightweight simulator is difficult

Gameplay logic is split between managed script assemblies and native game
services. A controller can call exposed managed APIs, but much of the
engine's behavior is not a freely replaceable module. A successful
experiment with one managed method does not recreate world initialization,
native routing, simulation scheduling, serialization, or all provider calls.

A new server implementation would need a behavioral contract for each
subsystem it replaces and evidence that it preserves observable game
behavior. This is a separate engineering project, not a launch option for
the current host.

## Operational health checks

Do not infer a healthy server from a listening port alone. Check separate
signals:

- Is the controller process alive?
- Is the server game process alive and in a world?
- Is the authoritative tick or world clock advancing?
- Does the local bridge answer a harmless diagnostic request?
- Are client acknowledgements and snapshots progressing?
- Is the game paused, blocked by a dialog, or otherwise not advancing?

See [troubleshooting](../troubleshooting.md) for practical diagnostics.

## Failure modes / limits

- A hidden window can still depend on desktop-session behavior;
  service-session operation is a separate claim.
- A minimized game may be throttled and should not be assumed equivalent to
  a hidden game.
- An offline interpreter harness does not prove a world can load or tick.
- A server game crash is outside ordinary controller exception recovery.
- Current dedicated hosting inherits the game process's platform and
  address-space constraints.

## Related pages

- [Process model](process-model.md)
- [Capability boundaries](capability-boundaries.md)
- [Capacity and performance](capacity-and-performance.md)
- [Server CLI](../server-cli.md)
- [Status](../status.md)

## Hidden-window operation versus a lightweight world

Hiding the server window changes presentation and potentially rendering
cost; it does not change the simulation architecture. The game still loads a
full save and runs its internal systems. A measured hidden-window test
retained nearly the visible baseline tick rate on one configuration.
Minimizing reduced rate in that test, so “headless” procedures should name
the actual window state rather than treating hidden and minimized as
synonyms.

| Mode | What changes | Evidence required before relying on it |
|---|---|---|
| Visible | Standard interactive window | Baseline for comparison |
| Hidden | Window is not shown | Verify clock advances after hiding and over sustained run |
| Minimized | Window is iconic/minimized | Measure throttling on target configuration |
| Off-screen | Window remains but is moved out of view | Check focus and rendering behavior separately |
| No desktop session | No normal interactive desktop | Dedicated boot and long-run test; hidden-window data is insufficient |

## Launch versus attach

Launch mode gives the controller responsibility for starting the game and
coordinating its initial load. Attach mode connects to a process that
already exists. This is useful for development and recovery from a
controller restart, but it changes assumptions about save loading and
shutdown ownership.

| Lifecycle | Launch mode | Attach mode |
|---|---|---|
| Process creation | Controller starts host | Operator/tool started it |
| Initial world load | Controller requests load | May reuse existing in-world session |
| Process identity | Known from launch result | Must be found or selected safely |
| Controller stop | May stop launched host per policy | Must not assume it owns external host lifecycle |
| Diagnostic interpretation | Startup phases include host start/load | Reattach phases may skip load |

The controller must avoid attaching to the wrong game process. Process
discovery, role selection, and bridge marker validation are safety checks,
not mere convenience.

## Offline interpreter hosts are a different tool

A small offline process can initialize a portion of the game's managed
runtime and invoke selected managed methods. Such a harness may make narrow
experiments faster and safer than repeated live-game runs. It is not a world
host: it can omit native internal calls, services, scheduler state, world
objects, save/load, and UI initialization.

A successful offline method invocation is evidence about the tested managed
method under the harness, not proof about an online gameplay scenario. The
experiment must report which services were initialized, which calls were
stubbed, and which outputs are meaningful.

## Health model

```text
controller alive?       -> process liveness
host game alive?        -> process liveness
bridge responsive?      -> local integration health
in-world?               -> world readiness
clock advancing?        -> simulation progress
clients applying?       -> mirror delivery health
```

These questions are intentionally separate. A responsive bridge can return
diagnostics while simulation is paused. A ticking server can be healthy
while one client is disconnected. Operator dashboards should not compress
them into one green light.

## Save ownership and recovery

The server game owns the shared world save. The controller can coordinate
saves, maintain recovery copies, and validate load completion, but should
not merge client-local saves into the authoritative world without a
separately designed conflict policy. On restart, confirm whether the server
was cleanly saved or recovered from a last known good copy before allowing
clients to assume continuity.

## How to verify / test

- Compare visible and hidden operation while measuring actual simulation
  tick advancement.
- Test boot-to-world from a hidden start, not only hiding after world load.
- Exercise attach mode with an already-loaded world and verify no duplicate
  load occurs.
- Kill/restart the controller while attached and verify intended game
  ownership behavior.
- Test bridge responsiveness independently from game clock progress.
- Confirm save checkpoint and restart behavior with a disposable copy.
- For offline harness results, document stubs and missing engine services.

## Failure modes / limits

The hidden-window data is bounded to a specific environment and does not
prove service-session, GPU-less, or cloud behavior. Offline interpreter
execution is not equivalent to loading a full world. See
[status](../status.md) for what has been validated in the current
repository.

## Related pages

- [Process model](process-model.md)
- [Capability boundaries](capability-boundaries.md)
- [Capacity and performance](capacity-and-performance.md)
- [Server CLI](../server-cli.md)
- [Troubleshooting](../troubleshooting.md)

## Startup diagnostics example

A health report should read like a sequence of evidence, not a single
“server up” flag:

```text
controller: running, mode=launch
host: process found, role=server
bridge: connected, layout compatible, mod capabilities received
world: in-world=true, tick advanced since previous sample
session: listener ready
client: joined=true, baseline applied=true, ack=current
```

This is a schematic example. Do not copy real process ids, machine paths,
save names, or user data into public documentation.

## Window state and graceful shutdown

A hidden window can remain responsive while the simulation runs. A minimized
window may be subject to platform throttling. Process shutdown also differs:
a hidden game's window may not respond to a normal close request in the same
way as an interactive window, so the host wrapper needs an explicit bounded
shutdown policy. Force termination should be a last resort and should be
reported because it can affect save recovery.

## Recovery sequence

After a server process failure, do not immediately reconnect clients to an
unknown world state. First determine whether the game saved cleanly, whether
a known-good checkpoint exists, whether the controller restored or loaded a
prior save, and whether the world clock/tick is advancing. Only then
establish the client baseline. A restored save can be valid but older than
the last observed in-memory state.

| Recovery step | Question |
|---|---|
| Detect | Which process failed, and when? |
| Preserve | Is there a safe checkpoint or diagnostic record? |
| Restore | Which save version was loaded? |
| Validate | Did the game reach a valid world and advance? |
| Rejoin | Are old claims cleared or intentionally restored? |
| Resynchronize | Did each client receive a fresh full snapshot? |

## How to verify / test

- Test graceful shutdown from visible and hidden states.
- Simulate server-game exit while the controller remains alive and confirm
  it is not shown as healthy.
- Restart after an interrupted save using a disposable world.
- Confirm the recovered world version is visible in diagnostics without
  exposing paths.
- Force a client to reconnect with stale local state and verify a full
  baseline.

## Failure modes / limits

Host recovery cannot guarantee recovery of actions that occurred after the
last durable save. A server-authoritative model resolves conflicts by
ownership but cannot reconstruct uncommitted in-memory state after a process
crash.

## Related pages

- [Persistence and time](../persistence-and-time.md)
- [Troubleshooting](../troubleshooting.md)
- [Status](../status.md)
