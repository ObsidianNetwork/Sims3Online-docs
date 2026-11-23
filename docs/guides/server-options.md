# Server controller options

These are options parsed by `SessionConfig.FromArgs` in `DedicatedServer/SessionConfig.cs`, not `stackctl` flags. The parser is a hand-written loop; it has no generated `--help` inventory and silently ignores unrecognized tokens, so verify a spelling in source before relying on it.

## Host and listener

| Option | Default | Effect |
|---|---|---|
| `--ts3 <path>` | Standard Windows game install location | Select local game executable. |
| `--save <name>` | `SimsOnlineModTempWorld.sims3` | Save loaded for the server session. |
| `--checkpoint-save <name>` | `SIMSONLINE_CHECKPOINT_SAVE`, if set | Target save used by checkpoint configuration. |
| `--listen <address:port>` | `127.0.0.1:34019` | Public session listener. Non-loopback binding is unauthenticated. |
| `--host-bridge <endpoint>` | `memory:TS3W` | Local host bridge transport. |
| `--host-timeout-seconds <n>` | 120 seconds | Timeout for host bridge operations. |
| `--player-command-timeout-seconds <n>` | 10 seconds | Default player-paced command timeout (`SessionConfig.DefaultPlayerCommandTimeout`). |
| `--attach-existing` | Off | Attach to a game already running instead of starting one. |
| `--attach-pid <pid>` | None | Identify the existing process for attach mode. |
| `--ts3-sandboxie-box <name>` | None | Launch the game in the named Windows sandbox. |
| `--ts3-sandboxie-start <path>` | None | Override the Sandboxie launcher. |

## Diagnostics and state

| Option | Effect |
|---|---|
| `--state-file <path>` | Write machine-readable controller lifecycle state. |
| `--bridgectl-listen <address:port>` | Enable the local administrative bridge listener; keep it loopback-only. |
| `--bridge-trace` | Enable bridge frame tracing. |
| `--bridge-record <path>` | Record bridge traffic to a local file. Treat it as sensitive game-state evidence. |
| `--host-probe`, `--probe`, `--snapshot-and-exit` | Run specialized probe/inspection modes instead of a normal long-lived session. |

## Replication controls

`--sync-broadcast-seconds <n>` accepts a positive integer; a non-positive value clears the override. `stackctl` explicitly passes `2` seconds. `--max-fold-per-cycle` defaults to 8, and `--resync-cooldown-seconds` defaults to 5 seconds. `--no-interest-scope` disables the default interest-scoped mirroring filter. The remaining `--resync-*` options override tick thresholds and maximum snapshot interval; defaults live in `ResyncThresholds.Default`.

Options prefixed `--probe-` are diagnostic entrypoints for particular game operations, not general deployment settings. Check the `SessionConfig.FromArgs` branches and their corresponding probe routines before invoking them. Do not run mutation probes against a valuable save.

## Example

```powershell
dotnet run --project DedicatedServer -- --save "Sunset Valley.sims3" --listen 127.0.0.1:34019 --host-timeout-seconds 120
```

## Related pages

[Run a server](run-server.md) | [Configuration](configuration.md) | [Environment variables](environment-variables.md) | [Logging and diagnostics](logging-diagnostics.md)
