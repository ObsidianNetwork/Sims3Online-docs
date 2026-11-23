# Warm reattach workflow

Warm reattach restarts a controller while leaving its already-running Sims 3 process intact. It is not the same as a cold restart, and it depends on identifying the correct game process before the controller is stopped.

## Server reattach through stackctl

```powershell
python scripts/stackctl.py server-reattach --build
python scripts/stackctl.py status --strict
```

`server-reattach` checks the current mod build, identifies the tracked server game PID, stops the controller without intentionally stopping the warm game, rebuilds if requested, and launches the controller with `--attach-existing --attach-pid <pid>`. The wrapper's `reattach-server` verb adds a bounded readiness poll; its default timeout is 600 seconds. If the game cannot be identified or the installed mod is stale, the wrapper stops rather than guessing.

Direct controller attach uses `--attach-existing`; `--attach-pid` is valid only with that flag. `SessionConfig.ValidateProbeArguments` rejects incompatible combinations, including attach mode with Sandboxie launch options and several probes that require a fresh game host.

## Client attach

The client controller accepts `--client-attach-pid <pid>` or a positive `SIMSONLINE_CLIENT_ATTACH_PID` value. This attaches its local mirror host to an existing client game rather than launching a new game process. Confirm that the PID belongs to the intended client role and current profile; a stale PID can identify an unrelated process.

## Verify after attach

1. Confirm the game process remains alive before and after the controller transition.
2. Check the controller state moves through startup and session-start phases.
3. Confirm bridge handshake and the intended server/client role.
4. Check that the session resumes and the expected snapshot/mirror state arrives.
5. Compare the correct role logs if the controller is ready but the game is not.

A bound listener alone does not prove reattachment succeeded. `stackctl` can print bridge-trace diagnostics on failure; preserve those with both role logs.

## Failure modes

- PID missing or stale: verify the process identity and start time; do not attach by PID alone.
- Controller exits but game also exits: the game was not being treated as an externally owned warm host; investigate process ownership and launch mode.
- Bridge connects but HELLO is missing: verify bridge startup and compatible mod state.
- Mod freshness gate fails: reinstall/build the intended package, then retry; do not bypass the check without understanding the mismatch.
- Session does not return to world: use phase and bridge logs to distinguish game load from controller session startup.

## Related pages

[Run a server](run-server.md) | [Run clients](run-client.md) | [Server options](server-options.md) | [Debugging](debugging.md) | [Hot reload](hot-reload.md)
