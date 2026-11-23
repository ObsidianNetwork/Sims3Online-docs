# `stackctl.py` reference

`stackctl.py` owns the local server/client lifecycle. It launches role-specific controllers and game processes, records logs/receipts, and offers bounded readiness checks. Begin with the current help; unlike `--help`, lifecycle verbs may launch or stop a game.

```powershell
python scripts/stackctl.py --help
python scripts/stackctl.py status --help
python scripts/stackctl.py boot-server --help
```

## Verb families

| Family | Current verbs | Purpose |
|---|---|---|
| Interactive lifecycle | `up`, `server`, `client`, `client2`, `client3` | Start all, server, or a particular client role. |
| Recycle/reattach | `client-restart`, `server-reattach`, `reattach-server`, `reload-mod`, `stop-controller` | Keep a warm game where possible while restarting/reloading its controller or mod. |
| Role shutdown | `client2-down`, `client3-down`, `down` | Stop one later client or the stack. `down --force` has broader impact. |
| Bounded launch | `boot-full`, `boot-server`, `boot-client`, `boot-client2`, `boot-client3`, `boot-client-restart`, `reattach-server` | Launch and wait for a terminal readiness result. Default timeout is 600 seconds. |
| Detached launch | `launch-full`, `launch-server`, `launch-client`, `launch-client2`, `launch-client3`, `launch-client-restart`, `launch-reattach-server` | Start and return without a readiness result; poll afterward. |
| Observe/manage | `status`, `logs`, `archive`, `wait-ready` | Inspect current roles, read logs, inspect/prune archives, or poll readiness. |

The help text and parser in `scripts/stackctl.py` are authoritative for the current checkout. A process launch receipt is not a readiness result. Use `status --strict` or a bounded boot/wait command before scenario work.

## Global flags

These flags are attached to the CLI's subcommands, but some apply only to specific verbs; parser acceptance does not prove a verb forwards or uses an option.

| Flag | Meaning |
|---|---|
| `--json` | Emit structured output where implemented. |
| `--build` | Rebuild controllers before launch. |
| `--front-door` / `--front-door-bin PATH` | Opt into the alternate Rust server and select its executable; the launcher does not build it. |
| `--bridge-trace` | Enable local bridge tracing. |
| `--detached` | Launch without waiting. |
| `--skip-mod-sync` | Skip automatic mod synchronization. |
| `--no-auto-up` | Keep `reload-mod` from automatically bringing the stack back up. |
| `--save NAME`, `--endpoint HOST:PORT` | Select the local save and public listener. |
| `--claim-sim ID`, `--claim-sim2 ID`, `--claim-sim3 ID` | Select initial local Sim claims; zero disables later-player claims. |
| `--ts3-path PATH` | Override the game executable selection. |
| `--host-timeout SEC`, `--client-launch-delay SEC`, `--player-command-timeout SEC` | Bound or pace controller/game operations. Defaults are 600, 8, and 0 respectively; zero player timeout means the default 10 seconds. |
| `--minimize-server-ts3` | Minimize the server game's window where supported. |
| `--client-wineprefix`, `--client2-wineprefix`, `--client3-wineprefix` | Select per-client Wine prefixes. |
| `--sandboxie-box`, `--sandboxie-box2`, `--sandboxie-box3`, `--sandboxie-start PATH` | Configure per-client Windows isolation. |

The public endpoint defaults to loopback `127.0.0.1:34019`; the client bridge defaults to `127.0.0.1:34030`. Do not change a bind to expose admin or public test listeners to a network without a separate security review.

## Verb-specific controls

- `status --quick` / `status -q` skips the `DIAG_STATE` probe. `status --strict` returns failure when required readiness checks fail. `status --probe` forces a bridge probe even while another session holds the stack claim.
- `logs` supports `--follow`, role selectors `--server`, `--client`, `--client2`, `--client3`, and `--lines N` (default 40). Follow mode is interactive.
- `archive --prune N` lists older automatic archives; deletion occurs only with `--yes`. Review candidates before confirming.
- Bounded `boot-*` verbs accept `--timeout N` (default 600), `--interval N` (default 0, meaning the implementation's normal poll cadence), and `--require-client` where applicable. `wait-ready` also accepts `--probe-timeout N` (default 15).
- `down --force` on Windows can sweep all `TS3W.exe` processes, not only tracked ones. Use only after confirming process ownership.
- `down --ignore-claim` bypasses another session's active stack claim. Without it,
  shutdown refuses before stopping processes, even with `--force`. Coordinate with
  the claim owner before using this override; it can interrupt their live run.

## Readiness and polling details

`status --strict` reports failure when required readiness checks fail. Exit code 3 specifically means another session holds the stack claim, so the bridge probe was skipped; it is not equivalent to a failed health response. `status --probe` requests the bridge check despite that claim, so use it only when probing the shared local stack is appropriate. `status --quick` omits `DIAG_STATE` and therefore gives less evidence.

For bounded boot and `wait-ready`, `--interval 0` selects the implementation's cadence: 15 seconds in the ordinary readiness path and 5 seconds when a client is required. A nonzero interval overrides that cadence. `wait-ready --probe-timeout` controls the individual bridge probe timeout, separate from the overall `--timeout`. Readiness is evaluated from local process/listener and bridge health evidence; it does not prove a client rendered the expected UI or that a gameplay assertion passed.

When a required client is included, the launch sequence waits for the server before starting that client. A timeout is a terminal result for the command, not proof that all started processes were stopped; re-check `status` and role logs before retrying.

## Typical bounded sequence

```powershell
python scripts/stackctl.py boot-server --build --timeout 600
python scripts/stackctl.py status --strict
python scripts/stackctl.py logs --server --lines 80
```

A timeout can leave the process running. Query status and inspect the role receipt/log before starting again. Use `down` for the stack-owned processes; do not terminate by broad executable name. Preserve relevant evidence before pruning an archive.

## Alternate front door

`--front-door` selects an already-built alternate server binary; `--front-door-bin` overrides its location. `--build` builds the managed controllers, not this Rust binary. Treat the mode as experimental and verify public protocol readiness, client compatibility, and local bridge/admin health separately.

## Related pages

[stackctl options](stackctl-options.md) · [Run server](run-server.md) · [Run clients](run-client.md) · [Warm reattach](reattach-workflow.md) · [Logging and diagnostics](logging-diagnostics.md)
