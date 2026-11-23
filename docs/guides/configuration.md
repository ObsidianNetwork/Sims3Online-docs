# Configuration

Configuration is owned by individual processes, not by one project-wide settings file. The relevant defaults live in the consumer: `SessionConfig.FromArgs` for the server, `SimsOnline.Client/Program.cs` for the client, and the argument parser in `scripts/stackctl.py` for the development launcher.

## Choose the owner first

| Setting | Owner | Where to configure |
|---|---|---|
| Public TCP endpoint and server save | Server controller | `--listen`, `--save`; `stackctl boot-server --endpoint ... --save ...` forwards values. |
| Client player identity and protocol codec | Client controller | `--player`, `--wire`, or role-specific stack settings. |
| Local game executable/profile | Launch wrapper and host | `SIMSONLINE_TS3_PATH`, `--ts3-path`, client sandbox/prefix settings. |
| Host bridge endpoint and timeout | Controller | `--host-bridge`, `--host-timeout-seconds`; wrapper has `--host-timeout`. |
| Presentation/routing experiments | Controller environment | Explicit `SIMSONLINE_*` switches listed in [Environment variables](environment-variables.md). |
| Diagnostic output | Controller or tooling | state/record/log arguments and `SIMSONLINE_RUN_DIR` for stackctl-owned output. |

There is no precedence rule that spans every program. In general, a controller's explicit option initializes its value; specific environment variables are read only where the parser or runtime names them. A wrapper may then build a child command with its own defaults. Check the owning parser rather than assuming an environment variable overrides every CLI layer.

## Safe local setup

Keep ordinary development listeners loopback-bound. The public session listener does not authenticate clients; local bridge-admin listeners are a separate privileged interface and should also remain loopback-only. Do not commit machine-specific endpoints, save paths, telemetry DSNs, or personal profile details.

Create a dedicated throwaway save before launching, or back up the existing save
you select. A name does not make a save disposable: the host loads an existing
save and join checkpoints can write back to it. This example assumes you have
created `SimsOnline Test.sims3` for testing and uses the default loopback endpoint:

```powershell
python scripts/stackctl.py boot-server --save "SimsOnline Test.sims3" --build
python scripts/stackctl.py status --strict
```

For direct execution, controller options go after `--` in `dotnet run`; `stackctl`
options go after the subcommand, as shown above. Do not mix the two parsers' flag names.

## Restart and side effects

Network endpoint, save, game executable, and profile changes require a new process or a fresh launch. Changing environment variables in the parent shell does not change an already-running process. Do not change a save or profile during a live test unless the test specifically covers that transition. `SIMSONLINE_JOIN_REUSE=0` disables unchanged-world reuse when preparing a client's join-world checkpoint, forcing a fresh save on each request. It does not control server-process startup; record it when reproducing join/checkpoint behavior.

## Configuration failures

- A wrapper flag can be valid while the corresponding direct-controller flag has another spelling.
- An unknown server/client argument may be silently ignored because their parsers are hand-written; check source, then verify the effective startup line.
- A valid endpoint may still expose an unauthenticated listener beyond loopback.
- A client with a local save can join successfully but display a different world than the server.

## Related pages

[Environment variables](environment-variables.md) | [Server options](server-options.md) | [Client options](client-options.md) | [Run a server](run-server.md) | [Run clients](run-client.md)
