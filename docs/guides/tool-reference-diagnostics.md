# Diagnostics and measurement tools

Choose the smallest observer that answers the question. A live watcher can perturb the game, and a missing sample can reflect an observer failure rather than a game failure. Keep raw logs, crash reports, process identifiers, memory dumps, endpoints, and local paths private; publish only a reviewed summary.

## Tool map

| Question | Existing tool(s) | Evidence boundary |
|---|---|---|
| Find matching log events | `scripts/logq.py`, `scripts/stream_log.py` | Input log/run identity and filter determine what was searched. |
| Correlate a crash and run phase | `scripts/crash_catch.py`, `scripts/crash_ledger.py` | Crash dumps can contain process memory; keep them local. |
| Inspect process lifecycle | `scripts/proc_timeline.py`, `scripts/run_watchdog.py` | Process presence alone does not establish bridge/game health. |
| Measure bridge pump/transport | `scripts/pump_ms.py`, `scripts/transport_latency_report.py`, `scripts/transport_latency_ab.py`, `scripts/transport_latency_sweep.py` | Align roles, build, and capture windows before comparing. |
| Check snapshot delivery and acknowledgments | `scripts/syncmon.py` | Log mode reports stream delivery and acknowledgment lag. Client-side probes add presentation evidence. |
| Compare server and client presentation | `scripts/presentation_compare.py` | Live samples cover Sims on player-occupied lots. Read the sample count and coverage with the scores. |
| Observe movement | `scripts/motion_watch.py`, `scripts/motion_track.py` | Confirm actor, role, and whether state updates were captured. |
| Check machine contention | `scripts/machine_load.py` | A load spike is correlation, not attribution to a code path. |
| Inspect event records | `scripts/stream_gaps.py`, `scripts/stream_kinds.py` | A capture gap can resemble a missing server event. |
| Review managed/native runtime behavior | `scripts/gc_watch.py`, `scripts/gc_heap.py` | Optional live instrumentation can affect timing. |

These are repository files, not a promise that every command is available on every operating system or local setup. Use each tool's own parser/source and tests; a helper may be a saved-evidence analyzer or require a running local game.

## Examples of parser-backed interfaces

`motion_watch.py` accepts optional Sim names, `--ids`, `--seconds` (default 20), `--capture`, `--events`, `--rescore`, `--client`, and `--ticks`. Its live mode observes motion and may capture images; `--rescore` reads a saved run instead. Check `python scripts/motion_watch.py --help` before using it.

`transport_latency_report.py` takes log paths/glob patterns and supports `--json`, `--output PATH`, `--self-test`, and `--same-host`. It analyzes recorded latency evidence; it does not create a gameplay trace by itself.

`scripts/logq.py` accepts a run/archive directory and can produce a timeline, errors, or a card verification. Its query syntax is defined by the parser in that file; do not guess a filter option from another tool.

## Querying saved logs

`logq.py` reads a selected run or archive directory and has three subcommands: `timeline`, `errors`, and `verify CARD`. Timeline filters can be combined:

```powershell
python scripts/logq.py --dir <run-or-archive> timeline --interaction <id> --actor <id> --corr <correlation-id>
python scripts/logq.py --dir <run-or-archive> timeline --grep <text>
python scripts/logq.py --dir <run-or-archive> errors
python scripts/logq.py --dir <run-or-archive> verify <card>
```

The directory and filters select the evidence set; an empty result means no matching records were found in that input, not that the event never occurred. `verify` checks the saved card evidence offline. Use `python scripts/logq.py timeline --help` for the current filter syntax, and preserve the selected run identity when comparing output.

For archived logs from a different time zone, pass `--server-offset-hours` before the subcommand. Use the recording machine's UTC offset at capture time, such as `10` for UTC+10. The flag applies to old server bridge timestamps that contain only local time. Logq defaults to the analyzer's current local offset when the flag is omitted. Untimestamped lines inherit the nearest preceding timestamp in their own file, or the next timestamp if there is no preceding one. An inherited timestamp gives approximate ordering.

For a parser-only check of the latency report, `python scripts/transport_latency_report.py --self-test` runs its built-in test path without collecting a gameplay trace. For a live or saved `motion_watch` invocation, check `--help` first: capture options and `--rescore` have different evidence and side-effect boundaries.

## Checking live sync and presentation

```powershell
python scripts/syncmon.py --once
python scripts/presentation_compare.py --duration 120 --interval 3 --no-recorder --out .omo/run/presentation-smoke
```

`syncmon.py` reads server and client logs to show snapshot ticks and acknowledgment lag. Its default position columns come from the server's last snapshot. `--client-snapshot` adds a local bridge-admin probe per tracked Sim so you can compare that snapshot with the client-side position. Allow for sampling delay when interpreting the difference.

`presentation_compare.py` reads `PROBE_PRESENTATION` on both live admin listeners. It matches Sims by description ID and counts repeated mismatches for the same Sim and field over consecutive samples. Use a fresh output directory for `samples.jsonl`, `summary.json`, and `report.md`, and check the sample count before interpreting a score. The example uses `--no-recorder` to skip sending `SACS_RECORDER on`. The tool still compares `anim_state` when both probes return recorded states. For multiple running clients, `--clients` takes a comma-separated list of client numbers and writes a report for each.

## Measurement procedure

1. Identify the symptom and its role/time window.
2. Read the selected tool's help/source; decide whether it reads saved evidence or attaches live.
3. Use one observer and a short bounded interval first.
4. Record tool/build identity, role, sample count, units, and any dropped/invalid samples.
5. Correlate with one independent source only if it answers a distinct question.
6. Repeat with the same scenario and settings before attributing a regression.

A pump stall, transport delay, and client presentation lag are different boundaries. Do not combine durations from unrelated clocks or infer server state from client motion alone. If an observer stops sampling, validate the observer before interpreting the empty interval.

## Limits and failure modes

| Observation | Possible explanations |
|---|---|
| No samples | Wrong role, observer did not attach, no matching event, or incomplete capture. |
| One large latency maximum | Scheduling contention, a transient event, or a real stall; inspect distribution and timeline. |
| Flat motion | Wrong actor/client, no movement action, or state/presentation divergence. |
| Empty crash report | No crash captured, catcher started late, or a process exited another way. |
| Missing event | Capture loss, sequence gap, parser mismatch, or server omission. |

Do not stack a crash catcher, trace, motion watcher, and multiple polling tools unless their interaction is understood. Stop collecting once the relevant boundary is isolated or instrumentation changes process health.

## Related pages

[Logging](logging-diagnostics.md) · [Performance profiling](performance-profiling.md) · [Live probes](live-probes.md) · [Scenario CLI](tool-reference-scenarios.md) · [Script reference](script-reference.md)
