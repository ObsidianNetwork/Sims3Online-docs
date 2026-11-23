# Scripts catalog

`git ls-files scripts` reports 285 tracked files: command-line entry points, libraries, tests, PowerShell helpers and fixtures. Most Python tests follow `<module>_test.py`; many files exercise code rather than define separate user commands. This page groups the tracked scripts by function and documents the safest front doors, not every internal helper as a supported CLI.

## Maintained entry points

| Command | Purpose and inputs/outputs | Side effects |
|---|---|---|
| `python scripts/dev.py doctor` | Reports available development prerequisites. | Read-only environment report. The front door itself uses the standard library. |
| `python scripts/dev.py lint` | Ruff plus PowerShell parse check when available. | Reads source. |
| `python scripts/dev.py test` | CI-selected Python tests, self-tests and CLI help checks. | Test/build output only; unavailable local prerequisites are listed in `dev.py` exclusions. |
| `python scripts/dev.py check` | Lint followed by the standard test set. | Preferred pre-push check; does not replace .NET or live-game validation. |
| `python scripts/scenarios.py list` / `health` | Lists registered scenarios or inspects local stack readiness. | Read-only; `health` requires the configured game stack. |
| `python scripts/scenarios.py run --help` | Shows runner flags without starting a scenario. | Help is read-only. Scenario execution writes run evidence and can interact with game processes. |
| `python scripts/tester_kit.py --help` | Lists `build`, `verify`, `install-box`, `list`, and `check`. | Help is read-only. Build/install create or install local files. |
| `python scripts/stackctl.py --help` | Documents local stack lifecycle/status subcommands. | Start/stop/rebuild/install operations affect processes or files. |
| `python scripts/check_docs.py --anchors --strict README.md CONTRIBUTING.md SECURITY.md docs` | Checks Markdown links, fragments and stale backticked path mentions. | Read-only; optional paths narrow the scope. |
| `python scripts/native_observatory_source_guards.py --base origin/main` | Scans changed source against privacy/proprietary-input policy. | Read-only; `--format github` formats CI annotations. |
| `pwsh -File scripts/check-mod-compile.ps1 -Strict` | Compiles the full host-mod source set on Windows when the required local references are supplied. | Produces build output; needs the Windows compiler setup and local Sims 3 references. |
| `python scripts/sidecar.py --help` | Displays native SimLink build/test entry points. | Use help before selecting a build operation. |

Help was successfully checked for `scripts/dev.py`, `scripts/scenarios.py`, and `scripts/tester_kit.py`; the scenario runner's `run --help` reports repeat/boot/mod controls, client count, client watches, tuning, held state, rejoin mode, crash catcher, soak duration and simulation speed. Consult current help for exact optional flags.

## Functional groups

| Area | Representative tracked files | Typical inputs and outputs |
|---|---|---|
| Build, packaging and local distribution | `dev.py`, `check-mod-compile.ps1`, `make-mod.ps1`, `make-probe.ps1`, `build-presentation-module.ps1`, `bootstrap-client-prefix.ps1`, `sidecar.py`, `fork_builder.py`, `tester_kit.py`, `refpack.py`, `install_nointro.py`, `mod_sources.py` | Source, SDK/compiler installations and locally supplied game paths; build products, manifests or local packages. Keep generated binaries and game-derived data out of source control. |
| Stack, clients and scenarios | `stackctl.py`, `scenarios.py`, `simctl.py`, `client_pick.py`, `rig_cargo.py`, `arm_readout.py` | Local stack settings, scenario JSON, run state and client identifiers; status/logs, verdicts and local helper output. |
| Bridge, protocol and stream inspection | `bridgectl.py`, `bridge-health.py`, `bridge-regression.py`, `bridge_record_replies.py`, `ring_peek.py`, `stream_log.py`, `stream_gaps.py`, `stream_kinds.py`, `transport_source_audit.py`, `transport_latency_report.py` | Bridge replies, streams and local captures; decoded state, regression diffs, gap and latency reports. Some commands connect to a process. |
| Presentation, routing and multiplayer comparison | `compare_client.py`, `motion_track.py`, `motion_watch.py`, `presentation_compare.py`, `presentation_reload.py`, `late_join.py`, `route_continuity.py`, `replay_check.py`, `replay_matrix.py`, `join_fingerprint.py`, `join_race_watch.py` | Scenario run folders and client/server observations; motion, replay, join and route diagnostics. |
| Diagnostics and performance | `crash_catch.py`, `crash_ledger.py`, `proc_timeline.py`, `run_watchdog.py`, `gc_heap.py`, `gc_watch.py`, `pump_ms.py`, `machine_load.py`, `telemetry.py`, `syncmon.py`, `tune.py`, `sweep.py` | Process/log/timing observations and tuning values; event timelines, crash summaries, CPU/GC/pump measurements or experiment results. Captures may expose private machine data. |
| Save/package/engine-format inspection | `dbpf_dump.py`, `dbpf_grep.py`, `dbpf_diff.py`, `dbpf_layout.py`, `saves.py`, `ts3_paths.py`, `ts3_view.py`, `ts3src.py`, `ts3w.py`, `household_read.py`, `object_diff.py`, `inventory_diff.py`, `jazz_dump.py`, `sacs_names.py`, `sacs_picks.py` | Local user-supplied saves/packages or synthetic fixtures; resource inventories, format summaries and diffs. Never publish proprietary assets, saves or bulk extracted content. |
| Managed/native evidence | `native_observatory_preflight.py`, `native_observatory_source_guards.py`, `native_byte_classify.py`, `native_decompile_export.py`, `native_name_evidence.py`, `native_defect_repair.py`, `managed_decompile_recover.py`, `managed_recompile.py`, `reference_assembly.py`, `game_decompile_inventory.py`, `gen_boundary_recorder.py` and mapping/archive-focused tests | Authorized bounded inputs and synthetic fixtures; normalized evidence, reports and validation results. See [native-observatory reference](native-observatory-reference.md) before running campaign/publication commands. |
| Codec/kernel comparison | `simnav_export.py`, `loop_variant_table.py`, `sacs_timing.py`, `session_factor.py`, `seam_map.py` | Controlled input or recorded test evidence; codec tables, diffs, route exports and measurement reports. A comparison is only as good as its baseline. |
| Docs and repository state | `check_docs.py`, `doc_paths.py`, `doc_refs.py`, `dotnet_warnings.py`, `ide_projects.py`, `reference_assembly.py`, `mod_sources.py` | Repository source and docs; link findings, build/source stamps, warnings and reference-assembly checks. |

Tests appear beside many modules, including `scenarios_test.py`, `stackctl_pid_test.py`, `native_observatory_schema_test.py`, `native_observatory_archive_test.py`, `sidecar_test.py`, `tester_kit_test.py`, `dbpf_grep_test.py`, and `check_docs_test.py`. Use `python scripts/dev.py test` for the CI-selected set. Individual modules mix unittest and pytest styles; check the file before selecting a focused test runner.

## Notable behavior and limits

- `scripts/scenarios.py` interprets a command DSL (`act:`, `sleep:`, `until:`, `hud:`, `capture:`); it needs a compatible local game stack to execute. The JSON catalog documents all definitions.
- `scripts/tester_kit.py verify` reads a kit folder and compares manifest inventory, size and hashes; `check` adds provenance/EA-derived-file checks. `build` and `install-box` mutate local outputs or install in a sandbox.
- `check_docs.py` accepts `--anchors`, `--strict`, `--json` and optional paths. `--strict` elevates stale path mentions; `--anchors` verifies heading fragments.
- `native_observatory_source_guards.py --base REF` runs changed-file policy checks; without `--base` it selects its evidence-archive mode with different arguments.
- A `.py` filename is not a guarantee of harmless execution: tools may attach to processes, install packages, collect logs, run decompilers, or contact configured services. Read docstrings/help first and keep private traces and proprietary content local.

## Related

- [Tooling overview](../tooling.md)
- [Testing](../testing.md)
- [Scenario catalog](scenarios-catalog.md)
- [Native observatory](native-observatory-reference.md)
- [Repository hygiene](repo-hygiene.md)
