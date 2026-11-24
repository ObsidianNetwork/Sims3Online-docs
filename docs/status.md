# Current status

Sims3Online is experimental software under active development. It is not a finished or
supported multiplayer release.

The repository includes a server-authoritative session model, controller-to-game integration,
client input forwarding, and code for distributing and applying presentation updates.
Automated tests cover portions of the controller, client, and protocol. These implementation
facts do not mean every multiplayer flow or game feature works reliably.

The default operating model uses the host game for shared simulation outcomes. The code also
contains explicit client-native lot-authority and reconciliation support; behavior and save
requirements differ when that mode is enabled.

## Implemented capabilities

- **Host-authoritative session & bridge control:** Server-authoritative session lifecycle,
  controller-to-game shared-memory bridge, client input forwarding, multi-client join/rejoin
  history, and client-side lot-authority reconciliation when enabled.
- **Native sidecar & ring transport:** Cleanroom `SimLink` native sidecar (`v17+` on shared-memory
  layout `v1`), sequence-list exhaustion guard and finished-driver history-liveness filtering,
  bounded native ring ownership in the Rust front door accepting compatible `layout v1` builds
  (`native_version >= 17`) with diagnostic `SACS_RECORDER` pass-through, and automated source +
  CLR Mono safety checks targeting The Sims 3's `.NET 2.0` script runtime.
- **World & presentation sync:** Authoritative weather and lunar state capture/replay plus
  vehicle presentation, occupant binding, and route stream identity handoff, backed by automated
  controller tests and live multi-client verification scenarios.

## Limitations

- The game and mod setup is manual and requires the user to provide a compatible, legally
  acquired copy of The Sims 3.
- Compatibility with game versions other than the project's tested build is not established.
- Not all interactions, expansions, custom content, or third-party mods have been verified.
- The network protocol has no authentication or encryption; it must not be exposed to untrusted
  networks.
- Joining, persistence, recovery, and multi-client behavior remain under development.
- Presentation coverage is selective; a successful controller test does not establish that a
  live interaction or animation works correctly.

Project behavior changes as code evolves. Check the current source, tagged releases, and issue
tracker before relying on any particular capability.
