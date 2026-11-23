# Performance profiling

Measure a defined stage and preserve the conditions that produced it. The repository's transport trace is opt-in and reports boundary timings; it is not a general game-frame profiler and does not by itself measure what a player saw on screen.

## Transport trace

Set `SIMSONLINE_TRANSPORT_TRACE=1` for the controller process, run a bounded test, and analyze the resulting logs:

```powershell
$runDir = if ($env:SIMSONLINE_RUN_DIR) { $env:SIMSONLINE_RUN_DIR } else { Join-Path (Get-Location) ".omo\run" }
$serverLog = Join-Path $runDir "server.log"
$clientLog = Join-Path $runDir "client.log"
# Only when both logs were recorded on this same PC:
python scripts/transport_latency_report.py --same-host $serverLog $clientLog
python scripts/transport_latency_report.py --same-host --json $serverLog $clientLog
python scripts/transport_latency_report.py --self-test
```

Run from the repository root. Omit `--same-host` for cross-machine logs; correlation is
intentionally skipped without that explicit assertion. Trace rows use the
`[transport-latency]` prefix and identify a `stage`, with fields such as `elapsed_ms`,
`bytes`, and `lines`. No current trace emitter provides a `queue_ms` field. The report
summarizes distributions with min, mean, p50, p95, p99, and max.

## What the stages mean

The report recognizes `source_event`, `server_bridge_dump`, `client_network_receive`, and `client_game_apply`. Where sequence IDs and compatible clocks allow it, it correlates server dispatch to client socket receipt and source event to managed replay completion. The first is a same-host network-delivery hop including queueing and TCP. The latter ends when the in-game managed replay handler completes; it is not a rendered-frame timestamp.

A four-stage measurement is marked ready only when the expected stages, event identity, and high-resolution same-host timing evidence are present. Unrelated samples from separate stages cannot be combined into an end-to-end latency number. Filtered ring cursor advances are not proof of dropped events.

## Controlled comparison

For an A/B experiment, keep the save, game speed, client count, scenario, and observation window constant. Change one parameter at a time; save the exact environment switch and build identity. Compare distributions and sample counts, not a single maximum. Repeat with tracing disabled to detect observer overhead.

## Limits and failure modes

- No trace rows: the switch may not have reached the controller, or the exercised path did not emit that stage.
- Unmatched receives: no exact dispatch sequence pair was available; do not treat them as zero-latency samples.
- Cross-machine logs: clock alignment is not sufficient for same-host latency claims.
- Managed apply completed: proves replay handler completion, not the next animation frame or visible presentation.
- Small sample sets: p99 is mathematically computed but may not be a stable estimate.

## Related pages

[Logging and diagnostics](logging-diagnostics.md) | [Test strategy](test-strategy.md) | [Scenario catalog](scenario-catalog.md) | [Environment variables](environment-variables.md)
