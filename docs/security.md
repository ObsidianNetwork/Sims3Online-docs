# Security

Sims3Online is experimental development software. Use it only on systems and networks you trust.

## Network limitations

The client/server protocol does not currently provide authentication or encryption. The host
controller defaults to listening on loopback, but changing its listen address can make it
reachable to other systems. Do not expose it to the public internet, forward its port, or accept
connections from untrusted clients.

Local game-controller interfaces are intended for local development and must not be exposed
outside the host machine. The project is not a supported player release and should not be used
with data that must be protected from untrusted users.

## Game ownership and distribution

Users must obtain and install The Sims 3 themselves. Do not distribute game executables,
assemblies, saves, packages, decompiled game source, or other EA/Maxis content with Sims3Online.
The repository's own source is licensed under the
[MIT License](https://github.com/ObsidianNetwork/Sims3Online/blob/main/LICENSE).

## Reporting a vulnerability

Please do not post security vulnerabilities in public issues or discussions. Report them privately through a
[GitHub Security Advisory](https://github.com/ObsidianNetwork/Sims3Online/security/advisories/new)
or email [security@obsidiannetwork.au](mailto:security@obsidiannetwork.au), which is monitored.

Include the impact and enough information to reproduce the issue safely. Do not include game
files, saves, credentials, or personal data in a report.

Sims3Online is an independent project and is not affiliated with, endorsed by, or sponsored by
Electronic Arts or Maxis. The Sims and The Sims 3 are trademarks of Electronic Arts Inc.
