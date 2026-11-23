# Environment variables

The project does not define one global environment-variable registry. The entries below are verified at their consumers in the current code. Most are opt-in switches; set them only for the process whose behavior they configure.

## Game and session selection

| Variable | Consumer and behavior |
|---|---|
| `SIMSONLINE_TS3_PATH` | `scripts/ts3_paths.py` and `stackctl` game discovery; select a local `TS3W.exe` when auto-detection is unsuitable. |
| `SIMSONLINE_CHECKPOINT_SAVE` | Server `SessionConfig.FromArgs`; supplies the checkpoint save name when the corresponding option is absent. |
| `SIMSONLINE_SERVER_SAVES_DIR` | `HeadlessSims3Host`; overrides the server save-directory discovery. |
| `SIMSONLINE_CLIENT_SAVES_DIR` | `stackctl`; overrides the client Saves directory used for server-world transfer. |
| `SIMSONLINE_CLIENT_ATTACH_PID` | Client controller; default attach PID when it parses as a positive integer. An explicit `--client-attach-pid` takes precedence. |
| `SIMSONLINE_JOIN_REUSE=0` | `HeadlessSims3Host`; disables reuse of an unchanged join-world checkpoint. Default is reuse enabled. |

## Replication and authority switches

| Variable | Accepted setting | Effect |
|---|---|---|
| `SIMSONLINE_PRESENTATION_STREAM` | `0` disables; otherwise enabled | Server presentation stream; default on. |
| `SIMSONLINE_CLIENT_NATIVE_MODES` | `0` disables; `1` explicitly keeps on | Native client lot modes. With presentation streaming on, the default is off to avoid conflict between local simulation and streamed animation. |
| `SIMSONLINE_ROUTE_STREAM` | `1` enables | Optional route presentation stream; requires presentation streaming. |
| `SIMSONLINE_RUST_ROUTE` | `1` enables | Optional Rust route-plan seam. |
| `SIMSONLINE_RUST_SIDECAR` | `1` enables | Optional Rust authority sidecar; default off. `SIMSONLINE_RUST_SIDECAR_PORT` selects its port. |
| `SIMSONLINE_AUTHORITY_GATE` | `off`, `observe`, `enforce` | Client authority gate; CLI `--authority-gate` can select the mode. |
| `SIMSONLINE_MIRROR_INTERACTIONS` | `0` disables interaction replay | Client continues other mirror channels. |
| `SIMSONLINE_SIMLINK_SERVER_RING` / `SIMSONLINE_SIMLINK_CLIENT_RING` | `1` enables respective ring path | Select native SimLink presentation transport. |
| `SIMSONLINE_SIMLINK_SHADOW` | `1` enables shadow observation | Exercises the sidecar without selecting the ring as the live path. |

## Diagnostics and tooling

| Variable | Consumer | Effect |
|---|---|---|
| `SIMSONLINE_PUBLIC_ENDPOINT` | `stackctl`, `simctl` | Default public session endpoint. An explicit `--endpoint` takes precedence. |
| `SIMSONLINE_SERVER_BRIDGECTL` / `SIMSONLINE_CLIENT_BRIDGECTL` | `stackctl`, bridge and scenario tools, `simctl synth-click` (client only) | Override the respective loopback admin endpoints for the launch, probes and scenario subprocesses. |
| `SIMSONLINE_TRANSPORT_TRACE=1` | Server and client controllers | Emit opt-in transport stage timings for `scripts/transport_latency_report.py`. |
| `SIMSONLINE_RUN_DIR` | `stackctl` | Override the local directory used for stack logs, state files, and captures. |
| `SIMSONLINE_ARCHIVE_KEEP` | `stackctl archive` | Number of newest automatic run archives to retain when pruning. Default is 0, which retains all. |
| `SENTRY_DSN` | Telemetry configurator | Enables Sentry when set. `--sentry-dsn-env` can name a different environment variable. |

Sentry's `--sentry-send-default-pii` is off by default. A DSN is a secret: do not place one in source, command history, issue text, or logs. Review `DedicatedServer/Observability/SentryEnvironment.cs` only if telemetry scope details are relevant; do not infer that enabled telemetry is anonymous without checking the configured scope.

## Physical fork census

Inside a checkout, every launch path runs the physical fork census (`scripts/fork_census.py`) before it starts or attaches The Sims 3: `stackctl` and `quick_loop.py` in Python, and the C# controllers through `ForkCensusGuard.Enforce` in `HeadlessSims3Host.StartAsync` (`DedicatedServer/Authority/ForkCensusGuard.cs`). The census reports one of `fork`, `stock`, `unknown`, `mismatch` or `unreadable` for the server profile plus the configured Sandboxie box or Wine prefix and the client saves directory. Only `fork` lets the launch proceed unconditionally.

| Variable | Consumer | Effect |
|---|---|---|
| `SIMSONLINE_NO_FORK_OK` | `scripts/fork_census.py`, `scripts/quick_loop.py`, `scripts/stackctl.py` (including direct and front-door launches), `DedicatedServer/Authority/ForkCensusGuard.cs` | `1` permits a deliberately stock launch (same as `--no-fork-ok` after the launch verb). Without it a `stock` census exits 2 with `FORK-MISSING`, because stock gameplay lacks the authority hook the multiplayer gates need. The override only covers physically stock profiles with no installed receipt: `unknown`, `mismatch` and `unreadable` results always refuse with `FORK-UNSAFE` (exit 2). On attach, the census must also establish the box or `WINEPREFIX` the running game uses and refuses when it differs from the configured one. A controller with no checkout above its build (no directory holding both `.git` and `DedicatedServer/SimsOnline.DedicatedServer.csproj`) skips the census, since only a checkout holds fork receipts. |
| `SIMSONLINE_TS3_DOCUMENTS` | `scripts/fork_census.py` | The Sims 3 profile folder the census inspects as the server profile. Otherwise the census resolves Windows' redirected Documents folder, or on Linux the Wine prefix (`WINEPREFIX`, default `~/.sims3`). Setting it does not redirect the game itself. |

## Safe use

Set overrides in the launch environment for one test, then remove them. Keep experimental switches with the run record because they alter which subsystem is exercised. Do not use a server/client shell's environment to assume a separately started game inherited the same values; launch wrappers set selected role-specific variables themselves.

## Related pages

[Configuration](configuration.md) | [Client options](client-options.md) | [Server options](server-options.md) | [Performance profiling](performance-profiling.md) | [Logging and diagnostics](logging-diagnostics.md)
