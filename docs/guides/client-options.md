# Client controller options

`SimsOnline.Client/Program.cs` parses command-line options directly. It does not expose a conventional generated help listing; this page names options verified against its parser. `scripts/simctl.py` has a separate CLI and its options are not interchangeable.

## Session connection

| Option | Default | Notes |
|---|---|---|
| `--endpoint <host:port>` | `127.0.0.1:34019` | Public server endpoint. |
| `--wire <json|line>` | `json` | Select public wire codec. |
| `--player <n>` | 1 | Player ID sent by this connection. Use a distinct ID per client. |
| `--display <name>` | `client-scaffold` | Display name. |
| `--role <server|client>` | `client` | Declared role; only the literal `server` selects server role. |
| `--connect-timeout <seconds>` | 30 | Connection/handshake timeout. |
| `--claim-sim <id>` | None | Request a Sim claim after role setup. |
| `--authority-gate <off|observe|enforce>` | Environment setting or policy default | Client-side authority policy mode; it does not grant server authority. |

## Local mirror

| Option | Default | Effect |
|---|---|---|
| `--enable-mirror` | Off | Apply server events to the mirror engine. |
| `--client-ts3 <path>` and `--client-save <name>` | None | Configure the local game and its save. |
| `--client-world-from-server` | Off | Fetch the server's world; also configure `--client-saves-dir <path>`. |
| `--client-attach-pid <pid>` | `SIMSONLINE_CLIENT_ATTACH_PID` when valid | Attach to a running client game process. |
| `--client-host-bridge <endpoint>` | Host bridge default | Local game bridge endpoint. |
| `--client-host-timeout <seconds>` | 120 | Host operation timeout. |
| `--client-player-command-timeout <seconds>` | Session default (10 s) | Player-paced command timeout. |
| `--client-ts3-sandboxie-box <name>` / `--client-ts3-sandboxie-start <path>` | None | Windows process isolation settings. |
| `--client-ts3-wineprefix <path>` | None | Linux/Wine prefix. |

## Diagnostics and test controls

- `--state-file <path>` writes lifecycle state; `--json-stdout` emits structured log records.
- `--snapshot-log <path>` diverts full snapshot payload lines from the main log; the file can contain world data.
- `--bridge-trace` and `--bridge-record <path>` capture local bridge traffic.
- `--max-fold-per-cycle <n>` defaults to 8; `--resync-cooldown-seconds <n>` defaults to 5.
- `--max-events`, `--stall-first-dispatch-ms`, and `--exit-after-*` switches are integration-test controls, not normal operation settings.

## Failure interpretation

An invalid endpoint shape produces a clear error (`--endpoint must be ip:port`). A connected client can still fail role registration, receive no baseline, or run with mirror disabled. Diagnose those stages separately. Full snapshot logs and bridge recordings can expose world data; keep them local and share only sanitized summaries.

## Example

```powershell
dotnet run --project SimsOnline.Client -- --endpoint 127.0.0.1:34019 --player 2 --display tester-2 --wire json
```

## Related pages

[Run clients](run-client.md) | [Environment variables](environment-variables.md) | [Configuration](configuration.md) | [Logging and diagnostics](logging-diagnostics.md) | [simctl](simctl-reference.md)
