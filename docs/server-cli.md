# Dedicated server command line

The .NET controller starts the authoritative session and connects it to a locally installed
The Sims 3 host. Run it from the repository root after building the projects described in
[development setup](development.md). A usable game installation is required; the controller
does not include or provide game files.

```text
dotnet run --project DedicatedServer -- [options]
```

`dotnet run` accepts `--` as the separator between its own options and the arguments passed to
the server. The examples below use placeholders; substitute paths and save names available on
your machine.

## Common options

| Option | Default | Description |
|---|---|---|
| `--ts3 <path>` | Standard Windows TS3 executable path | Path to `TS3W.exe`. Pass this explicitly when the game is installed elsewhere. |
| `--save <name>` | `SimsOnlineModTempWorld.sims3` | Save name used by the host. |
| `--checkpoint-save <name>` | `SIMSONLINE_CHECKPOINT_SAVE`, if set | Optional checkpoint save name. |
| `--listen <host:port>` | `127.0.0.1:34019` | Public client protocol listener. Keep it on loopback unless an intentional, separately secured deployment design exists; the protocol does not authenticate or encrypt connections. |
| `--host-bridge <endpoint>` | `memory:TS3W` | Local controller-to-game bridge endpoint. See [bridge and formats](bridge-and-formats.md). |
| `--host-timeout-seconds <seconds>` | `120` | Timeout for host bridge operations. |
| `--player-command-timeout-seconds <seconds>` | 10 seconds effective | Override the player-paced command timeout. If omitted, `SessionConfig.EffectivePlayerCommandTimeout` falls back to `SessionConfig.DefaultPlayerCommandTimeout` (10 seconds). |
| `--start-speed <value>` | `normal` | Initial world speed: `pause`, `normal`, `double`, `triple`, `ultra`, or `none`. |
| `--sync-broadcast-seconds <seconds>` | Unset | Set a periodic synchronization broadcast interval; non-positive values disable this override. |
| `--server-mod <path>` | Unset | Select a server mod package path. The file must be supplied locally and is not distributed with this project. |
| `--attach-existing` | Off | Attach to a game process started separately instead of launching a new host. |
| `--attach-pid <pid>` | Unset | Select the process for attach mode; requires `--attach-existing`. |
| `--bridge-trace` | Off | Enable host bridge tracing for diagnostics. |

Example:

```text
dotnet run --project DedicatedServer -- --ts3 "<path-to-ts3>\Game\Bin\TS3W.exe" --save "<save-name>.sims3" --listen 127.0.0.1:34019
```

The process reports its effective configuration at startup. `--player-command-timeout-seconds`
sets an optional override; the effective fallback is defined by
`SessionConfig.DefaultPlayerCommandTimeout` and `SessionConfig.EffectivePlayerCommandTimeout` in
`DedicatedServer/SessionConfig.cs`. Most `--probe-*` switches are diagnostic/test modes, not
normal server operation; their prerequisites are enforced by the argument parser in
`DedicatedServer/SessionConfig.cs`.

## Runtime environment switches

These settings are read by the dedicated server process:

| Variable | Behavior |
|---|---|
| `SIMSONLINE_CHECKPOINT_SAVE` | Supplies the default checkpoint save name when `--checkpoint-save` is omitted. |
| `SIMSONLINE_PRESENTATION_STREAM` | Enabled unless set to `0`, but runs only with a positive `--sync-broadcast-seconds` interval. Direct launches leave that interval unset; `stackctl` supplies two seconds. |
| `SIMSONLINE_CLIENT_NATIVE_MODES` | `0` disables client-native lot modes. `1` explicitly enables them when the presentation stream is active; otherwise the stream causes them to be disabled. |
| `SIMSONLINE_ROUTE_STREAM` | Set to `1` to enable route streaming; it only takes effect when the presentation stream is enabled. |
| `SIMSONLINE_RUST_ROUTE` | Set to `1` to enable the opt-in Rust route path. |

These are advanced switches, not compatibility guarantees. Their implementation and defaults
are defined in `DedicatedServer/Program.cs`; keep the documentation synchronized
when those settings change.

## Client configuration

Client options are parsed by the client application, separately from the dedicated server.
The host's client-mirror probe options (`--client-mirror-*`) configure diagnostic launches, not
ordinary player clients. See the [network protocol](protocol.md) for client connection behavior
and [testing](testing.md) for supported test commands.
