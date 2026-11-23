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
