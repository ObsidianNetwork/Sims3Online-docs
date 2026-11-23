# Component reference

The repository is organized around a .NET controller, game-process integrations, and test and
development tooling. The table describes current responsibilities; exact internal APIs are
defined by the code and tests.

| Project or directory | Responsibility |
|---|---|
| `DedicatedServer/` | Host controller, player TCP listener, authoritative session coordination, state synchronization, and local game bridge |
| `DedicatedServer/Authority/` | Session command handling, game-host abstraction, checkpoint/bootstrap behavior, and server authority policy |
| `DedicatedServer/Network/` | Client connections, bounded line framing, event delivery, and local administrative interfaces |
| `DedicatedServer/HostBridge/` | Controller-side bridge codec, shared-region layout, snapshot decoder, and game-process transport |
| `DedicatedServer/Mirroring/` | Client mirror lifecycle, event translation, synchronization, lot reconciliation, and presentation-stream handling |
| `SimsOnline.Client/` | Player-side controller; sends commands and applies server updates to its local game |
| `SimsOnline.Protocol/` | Shared command/event records, value types, and protocol contracts |
| `SimsOnlineServerHostMod/` | In-game script integration, command handlers, observations, and presentation recording |
| `SimsOnlineServerHostMod/Presentation/` | Modular recorders and presentation-related integration code |
| `SimLink/` | Optional native sidecar for local transport; not required for the public TCP protocol |
| `DedicatedServer.Tests/` | Reflection-discovered .NET tests for server and protocol behavior |
| `SimsOnline.TestClient/` | Minimal client used by automated protocol and connection tests |
| `scripts/` | Python development front doors, test helpers, scenario and repository utilities |
| `scenarios/` | Reproducible definitions for selected live-game checks |
| `refs/sims3/` | Local-only build-input location; the public checkout includes policy documentation, not the assembly files |

## Important interfaces

### Public session protocol

The public controller-to-controller boundary is the TCP JSON protocol documented in
[network protocol](protocol.md). Message types and their serializers live in
`SimsOnline.Protocol/` and `DedicatedServer/Protocol/`.

### Local host bridge

The dedicated server and its local game mod have a separate bridge. Both sides define matching
layout and codec contracts. A change to this boundary must be reviewed on both sides; the current
layout and snapshot format are described in [bridge and formats](bridge-and-formats.md).

### Mirror and presentation boundary

Server events are translated into operations for a player's local game. The mirror applies
state and selected presentation updates; it does not decide the server's ordinary gameplay
outcomes. See [simulation and presentation](simulation.md).

## Build and distribution caveat

Game integration requires locally supplied API reference assemblies under `refs/sims3/`. The
public repository excludes those files pending rights verification; see the
[reference assembly policy](https://github.com/ObsidianNetwork/Sims3Online/blob/main/refs/sims3/README.md). The mod compile CI step reports as skipped
when these local-only inputs are absent, so a green clean-checkout run does not validate the full
legacy mod compile.
