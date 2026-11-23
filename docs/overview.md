# Project overview

Sims3Online is an independent experimental multiplayer project for **The Sims 3**. It is not
affiliated with Electronic Arts or Maxis and is unrelated to EA's 2002 game *The Sims Online*.

!!! note "Naming compatibility"
    The project is branded **Sims3Online**. Solution, project, assembly, namespace, and
    environment-variable identifiers (`SimsOnline.sln`, `SimsOnlineServerHostMod`,
    `SimsOnline.Client`, `SimsOnline.Protocol`, `SIMSONLINE_*`) retain `SimsOnline*` for
    compatibility.

## Project goal

The project explores how multiple players can share one Sims 3 world while continuing to use
their local game clients. In the default operating model, a host-owned game supplies shared
simulation outcomes. Controllers forward player commands, maintain the session, and apply
server-originated state and selected presentation updates to each player's game.

This design reduces the need for separate game processes to independently reach the same
simulation result. It does not provide lockstep determinism, comprehensive feature parity, or
support for every game version and mod configuration.

## Session at a glance

1. The host starts the dedicated controller and its game integration.
2. A player controller connects to the host over the TCP session protocol.
3. The player joins and receives a world/session baseline.
4. The player submits commands; the server checks identity and ownership before acting.
5. Snapshots and events update the player's local game.

The separate local bridge carries controller/game traffic; it is not the player-facing network
protocol. See [architecture](architecture.md), [network protocol](protocol.md), and
[bridge formats](bridge-and-formats.md).

## Repository map

| Area | Purpose |
|---|---|
| `DedicatedServer/` | Host controller, session, network, bridge, and mirroring code |
| `SimsOnline.Client/` | Player-side controller |
| `SimsOnline.Protocol/` | Shared TCP command/event records |
| `SimsOnlineServerHostMod/` | In-game script integration |
| `SimLink/` | Optional native local-transport sidecar |
| `DedicatedServer.Tests/` | Automated .NET test suite |
| `scripts/` | Developer commands, test helpers, and maintenance tooling |
| `scenarios/` | Selected live-game scenario definitions |
| `docs/` | Public project and developer documentation |

See the [component reference](components.md) for a more detailed map.

## Important limitations

The project is not ready for general players. The public protocol has no authentication or
encryption. Game compatibility, interaction coverage, and behavior under custom content remain
limited. Review [current status](status.md) and [security](security.md) before running it.
