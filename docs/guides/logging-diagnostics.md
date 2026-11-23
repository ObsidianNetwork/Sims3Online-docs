# Logging and diagnostics

The controllers write ordinary logs to standard output; `stackctl` captures role-specific server and client logs and state files under its local run directory. `SIMSONLINE_RUN_DIR` can relocate that directory. The generated run artifacts are local evidence, not release inputs.

## Capture a bounded run

```powershell
python scripts/stackctl.py logs --server --lines 120
python scripts/stackctl.py logs --client --lines 120
python scripts/stackctl.py status --strict --json
```

Use `--follow` when observing a live transition and stop it once the failure boundary is known. The `--json` status output is machine-readable. Controller `--state-file` output records lifecycle phases; it avoids depending on buffered console text for readiness.

## High-volume payloads and bridge traces

Client `--snapshot-log <path>` moves full snapshot/delta payload lines out of the main client log. `--bridge-trace` enables per-frame bridge logging; `--bridge-record <path>` records traffic. These files can contain world state and should remain private. Capture only the minimum interval necessary.

`SIMSONLINE_TRANSPORT_TRACE=1` adds `[transport-latency]` records. Analyze them after the run:

```powershell
$runDir = if ($env:SIMSONLINE_RUN_DIR) { $env:SIMSONLINE_RUN_DIR } else { Join-Path (Get-Location) ".omo\run" }
$serverLog = Join-Path $runDir "server.log"
$clientLog = Join-Path $runDir "client.log"
python scripts/transport_latency_report.py $serverLog $clientLog
python scripts/transport_latency_report.py --json $serverLog $clientLog
```

Run these commands from the repository root, as with `stackctl`. The report computes per-stage
distributions (min, mean, p50, p95, p99, max). Add `--same-host` only when both logs were
recorded on the same PC and their clocks/event identities support correlation. It cannot
claim end-to-end render latency merely from network receipt. `--self-test` exercises the parser without a game.

## Safe report contents

Share the exact command after redacting local paths and endpoints, component revision, observed lifecycle stage, sanitized exception text, and a short relevant log excerpt. Do not publish full snapshot files, bridge recordings, save data, credentials, telemetry DSNs, or game screenshots with identifying content. Preserve timestamps and event ordering while removing personal/world identifiers.

## Common collection failures

- Empty console output can be buffering; use the role's state file and controller log rather than assuming the process is idle.
- A missing snapshot side file means the option was not supplied or no snapshot was written there; it does not prove no snapshot arrived.
- Traces from different machines cannot be combined into a same-host latency estimate without a shared clock basis.
- Large captures can perturb timing and consume storage; reproduce once with tracing off.

## Related pages

[Debugging](debugging.md) | [Performance profiling](performance-profiling.md) | [Environment variables](environment-variables.md) | [Scenario catalog](scenario-catalog.md)
