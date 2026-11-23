# Frequently asked questions

## Is this ready for players?

No. Sims3Online is experimental development software. Feature coverage and compatibility are
limited, and the network protocol is unauthenticated and unencrypted.

## Does this repository include The Sims 3?

No. Users must obtain the game themselves. Do not add game binaries, saves, packages, extracted
assets, or decompiled game source to the repository or to public bug reports.

## Does the server decide the outcome?

The default shared-session model uses the host game for shared simulation outcomes. Clients
forward intent and mirror the server's result. The code also has an explicit client-native
lot-authority mode with a separate reconciliation contract; see
[architecture](architecture.md).

## Can I expose the server on the internet?

No. The current protocol provides no authentication or encryption. Keep it on trusted local
development networks and do not forward its port or accept untrusted connections.

## Does an accepted interaction mean it succeeded?

No. Acceptance means the server accepted the command. The game can later report that the
interaction started, completed, or failed.

## How do I preview documentation changes?

Install `requirements-docs.txt` and run `mkdocs serve` from the repository root. CI validates a
strict build and retains its generated site as a downloadable artifact.

## Why are game-specific builds not reproducible from a clean clone?

Some integration builds require locally supplied game API reference assemblies and an installed
game. The public repository includes neither the game nor the reference DLLs. Review
[`refs/sims3/README.md`](https://github.com/ObsidianNetwork/Sims3Online/blob/main/refs/sims3/README.md) and the licensing caveat in
[components](components.md) before distributing any game-related build inputs.
