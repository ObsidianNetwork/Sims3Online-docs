# Run a server

The dedicated server controller hosts one local Sims 3 process and exposes a public client listener. The default listener is `127.0.0.1:34019`; the protocol has no authentication. Keep the listener on loopback or a private network you control, and do not forward it from a router.

## Start and confirm readiness

If you will join a local mirror, follow the [first live session](first-live-session.md) and build both controllers before starting the server.

For the development stack, `stackctl` has bounded boot commands. `--build` requests a
managed-controller rebuild; independently, boot commands may run `make-mod.ps1`
automatically when workspace mod sources differ from the installed copy. See
`ensure_mod_synced` in `scripts/stackctl.py`; `--skip-mod-sync` bypasses that check
and can leave a stale mod installed.

```powershell
python scripts/stackctl.py boot-server --build
python scripts/stackctl.py status --strict
python scripts/stackctl.py logs --server --lines 80
```

`boot-server` returns 0 when ready, 1 on timeout, and 2 when startup failed. Its default wait is 600 seconds. `status --strict` returns 1 for failed readiness; it can return 3 when another active stack claim prevented bridge probes, which means listeners may be up but bridge health was not checked. Do not reinterpret that as a successful host probe.

To start the controller directly for a focused experiment:

```powershell
dotnet run --project DedicatedServer -- --save "SimsOnline Test.sims3" --listen 127.0.0.1:34019
```

Create a dedicated throwaway save with that name first, or back up an existing save
before selecting it: join checkpoints can write back to the selected save.
The argument parser is `SessionConfig.FromArgs` in `DedicatedServer/SessionConfig.cs`. The controller prints the selected save, listener, host bridge, and probe mode at startup. The `stackctl` wrapper also supplies its own state-file path, bridge-admin endpoint, and two-second sync interval.

## Readiness is layered

A listening socket is not proof that the game loaded the save. `stackctl wait-ready` checks controller state and listeners, and can call bridge health; use `--require-client` only when client readiness is part of the test. For a live client test, separately verify the server has entered the world and the intended public port is reachable.

| Observation | What it establishes |
|---|---|
| Controller process exists | The controller was launched. |
| Public listener responds | The transport endpoint is open. |
| Bridge health responds | The local host bridge answered its probe. |
| Session reached `session-started` | The controller completed session startup. |
| Client join and baseline observed | A client passed protocol and initial-state stages. |

## Stop and preserve evidence

`python scripts/stackctl.py down` stops development-stack processes and prints a receipt. Use `stop-controller` only when intentionally keeping the game warm. Before restarting, capture the relevant server/client log and state-file phase; a restart can erase the first useful failure context.

## Failure modes and security boundary

- A bind failure can mean the address or port is already in use.
- A controller can report a startup error after the process has launched; inspect `stackctl status` and logs.
- A live game with no bridge response is a host/mod problem, not a public-protocol problem.
- A non-loopback listener is unauthenticated. Never expose it to an untrusted network.
- A local bridge-admin listener is separate from the public listener and should remain loopback-only.

## Related pages

[Server options](server-options.md) | [Configuration](configuration.md) | [Run clients](run-client.md) | [Logging and diagnostics](logging-diagnostics.md) | [Debugging](debugging.md)
