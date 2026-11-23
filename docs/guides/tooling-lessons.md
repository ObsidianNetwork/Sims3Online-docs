# Tooling lessons from the repository

This page records patterns visible in the current repository tools, not general rules for every project. The most useful shared example is `scripts/dev.py`: it is standard-library-only for its command parser and setup diagnosis, while the tool dependencies are installed separately from `requirements-dev.txt`.

## Keep command boundaries explicit

`dev.py` uses required subcommands (`doctor`, `lint`, `test`, `check`). The managed controllers use hand-written parsers, and `stackctl.py`, `scenarios.py`, `simctl.py`, and `tester_kit.py` use `argparse`. Separate flags by owner: `stackctl boot-server --endpoint ...` and controller `--listen ...` are different arguments to different processes.

For user-facing scripts, prefer an inspectable `--help` contract and make it safe to execute without game files. `dev.py test` checks `--help` for tracked CLI scripts; it also runs `stackctl.py --test` and `transport_latency_report.py --self-test`. A CLI smoke check should test help/parsing without performing a live operation.

## Keep wrappers honest

A launcher should describe exactly what it builds, launches, waits for, or cleans up. `stackctl` distinguishes detached `launch-*` verbs from bounded `boot-*` verbs; boot exit codes distinguish ready, timeout, and boot failure. It also exposes `status --strict` so callers can fail on readiness rather than grepping human text.

Do not hide required work behind defaults. For example, `stackctl boot-server --build` explicitly rebuilds managed controllers, while the optional Rust front door is built separately and is never compiled implicitly. Boot commands can also synchronize/rebuild a stale mod automatically, independently of `--build`. Keep process ownership explicit so cleanup cannot terminate an operator-started game by accident.

## Make failure useful

The server/client state file provides lifecycle phases to a wrapper; a state transition can be consumed without parsing console output. Preserve a clear distinction between process-start, listener, bridge, join, snapshot, and mirror failures. When a tool writes a report, keep a stable machine-readable envelope and send human diagnostics to stderr or a separate channel.

## Test the interface and the implementation

- Add a self-test for argument conversion, output parsing, and exit-code behavior.
- Use a testable core for string/protocol transformations; `ClientCaptureFilter` and `UserIntentWire` are source-linked into the .NET tests.
- Keep an explicit CLI-help test so new entrypoints cannot regress on import or startup.
- Test file and process failures without requiring a live game where practical.
- Keep live-game scenarios separate from unit tests and mark unavailable prerequisites as blocked rather than passed.

The `dev.py` Python-test fallback runs test modules in bounded subprocesses so one platform-specific failure does not hide unrelated results. It reports modules that require pytest or could not load; incomplete test coverage is not silently described as complete.

## Keep output bounded and safe

Prefer one-line summaries, configurable log tails, and opt-in verbose traces. `stackctl logs` limits its tail to 40 lines by default. `transport_latency_report.py` parses only recognized `[transport-latency]` records and reports sample counts alongside distributions. Full snapshots and bridge recordings can contain world state and should never be emitted in an ordinary diagnostic summary.

## Related pages

[Developer CLI](dev-cli.md) | [Test strategy](test-strategy.md) | [Logging and diagnostics](logging-diagnostics.md) | [CI](ci.md) | [Configuration](configuration.md)
