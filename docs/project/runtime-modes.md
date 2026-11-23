# Choose a runtime

Choose the path that exercises the behavior you want to verify. Start with the
[quickstart](quickstart.md) to build a checkout and the [compatibility matrix](compatibility-matrix.md)
to check game prerequisites.

| Goal | Path | First useful result |
|---|---|---|
| Host a shared game and connect a player | [Game-backed .NET session](#game-backed-net-session) | A player joins the host-owned world on a trusted test network. |
| Test the controller's Rust authority adapter | [Rust authority daemon](#rust-authority-daemon) | The standalone NUnit suite passes against the loopback daemon. |
| Exercise the experimental Rust listener and session protocol | [Rust front door with a fake driver](#rust-front-door-with-a-fake-driver) | The loopback listener binds and a matching line-v1 client exercises session behavior. |
| Change the local native ring or its managed reader | [Windows x86 SimLink transport](#windows-x86-simlink-transport) | The selected Windows SimLink cases pass without skips. |

## Game-backed .NET session

The dedicated .NET controller coordinates players over newline-delimited JSON TCP and
communicates with its local Sims 3 process through the host bridge. In the default
authority mode the host game supplies shared gameplay outcomes. Each player's .NET
controller forwards intent and applies server updates to its local game.

Use the [server guide](../guides/run-server.md) and [client guide](../guides/run-client.md)
with your compatible, legally acquired game and a disposable profile and save. Keep the
listener on a trusted network because the protocol has no remote authentication or
encryption. Check the server and client processes, then verify the scenario's actual
join, state, or presentation outcome. The [testing guide](../testing.md) explains how
to record a live scenario verdict. See the [JSON protocol](../protocol.md) for wire
details and [current status](../status.md) for the project's compatibility limits.

## Rust authority daemon

The optional Rust authority daemon is a separate loopback process. The .NET controller
can opt in with `SIMSONLINE_RUST_SIDECAR=1` to consult it for selected authority
decisions. The NUnit fixtures start their own daemon instances on loopback and exercise
the managed adapter with real TCP messages.

Follow the [Rust authority test setup](../testing.md) to build
`simlink_authority_daemon`, point `SIMSONLINE_SIDECAR_DAEMON` to its executable, and
run `dotnet test DedicatedServer.Tests/DedicatedServer.Tests.csproj`. CI runs the
full NUnit project in `sidecar-tests` with zero discovered tests treated as a failure.
A successful run verifies the exercised adapter and daemon exchanges. Use an in-game
scenario for gameplay or client presentation evidence.

## Rust front door with a fake driver

The experimental `simlink_headless_server` owns its own listener and session state.
`SimsOnline.Client` defaults to JSON. To connect it to the Rust listener, pass
`--wire line` for the pipe-delimited line-v1 protocol and use the printed listener address
with `--endpoint`. See [client options](../guides/client-options.md).

To inspect the experimental listener and run it without starting a game, run these
server commands separately from the repository root:

```sh
cargo run --manifest-path rust/simlink_headless_server/Cargo.toml -- --bind-only
cargo run --manifest-path rust/simlink_headless_server/Cargo.toml -- --serve --fake-driver
```

The first command reports the bound addresses and exits. The second keeps a loopback
server running with an offline driver. Use Ctrl+C or send `QUIT` on standard input to stop
the resident process. This mode can exercise listener, framing, and session behavior.
Live game attachment is a separate experiment with its own prerequisites and evidence.
See [Rust server front door](../design/rust-kernel-front-door.md) and
[wire formats](../protocol-reference/wire-formats.md).

## Windows x86 SimLink transport

SimLink is an optional native C++ ASI loaded inside the 32-bit Windows game process.
Its shared-memory ring carries local game-to-controller records. The host bridge
continues to carry control traffic, and the public TCP interface is a separate boundary.

On Windows, `SimLink.ReaderDrainsWrappedRing` uses a managed writer and a named memory map.
The .NET build is sufficient for that case. Use the project below with
`--filter SimLink.ReaderDrainsWrappedRing`. For the full `SimLink` selection, build the
native x86 Release binaries first using the
[SimLink tooling guide](../guides/tool-reference-sidecar.md). The full selection includes
cases that launch `simlink_selftest.exe`, and `--fail-on-skip` requires every selected case
to run.

```powershell
dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter SimLink --fail-on-skip
```

These cases cover managed ring reading and the native self-test exchanges without booting
the game. For live plugin loading or measurement, use a compatible game and check the
registration and ring state using the [SimLink reference](../code-reference/simlink.md).
