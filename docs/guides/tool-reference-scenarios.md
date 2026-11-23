# Scenario and user-test CLI

`scripts/scenarios.py` records repeatable scenario runs and offers read-only commands for inspecting prior evidence. `scripts/ut.py` runs a separate set of human-assisted test cards. Read each tool's help before a live action:

```powershell
python scripts/scenarios.py --help
python scripts/scenarios.py run --help
python scripts/ut.py --help
```

## Scenario commands

The current top-level subcommands are `list`, `home`, `run`, `health`, `history`, `rescore`, `exclude`, `arms`, `exposure`, `explain`, `status`, `wait`, and `during`. `list`, `home`, `health`, `history`, `explain`, `status`, `wait`, `exposure`, and saved-run views inspect definitions or evidence; `run`, `arms`, and `exclude` can change live setup or saved history. `rescore` changes the interpretation of saved evidence, not the recorded gameplay.

For run options, the parser currently defines:

| Option | Current contract |
|---|---|
| `--repeat N` | Run the selected set N times; default 1, each sample has its own record. |
| `--boot`, `--mod`, `--no-fork-ok` | Boot first; `--mod` builds/installs before launch; the last option permits boot without the local gameplay fork. |
| `--bridge-trace` | Enable bridge tracing during boot. |
| `--clients {1,2,3}`, `--watch-clients 1,2` | Start or compare selected client roles; game profiles must be isolated. |
| `--tune [SIDE:]NAME=VALUE` | Repeatable scenario-scoped tunable, restored after each scenario where supported. |
| `--held [SIDE:]NAME=VALUE` | Record a caller-owned tunable; it is checked but not restored. |
| `--held-process JSON` | Associate held values with game process identity. |
| `--no-repair` | Skip the between-scenario reset repair step. |
| `--rejoin {warm,paused,cold}` | Select the rejoin mode for scenarios that use rejoin. |
| `--catcher`, `--catcher-side {both,server,client}` | Attach passive crash collection to the chosen role(s); this adds overhead. |
| `--soak MIN`, `--speed {Normal,Double,Triple}`, `--soak-sims {free,held}` | Run a bounded idle exposure instead of named scenarios. |
| `--server-env K=V` | Repeatable extra server-controller environment setting when booting. |

These are live/integration options, not harmless parser flags. Use an owned disposable stack, identify what the runner will change, and do not pass credentials or private endpoint data in shell history.

## A repeatable workflow

```powershell
python scripts/scenarios.py list
python scripts/scenarios.py explain <scenario-name>
python scripts/scenarios.py run <scenario-name> --repeat 2
python scripts/scenarios.py history <scenario-name> --last 5
```

`list` supplies current scenario names; `explain` describes the selected scenario and expected evidence; `run` executes; `history` and `explain --run latest` review saved results. `health` is a prerequisite check, not a feature assertion. A failure during setup means the scenario behavior may never have been exercised; distinguish blocked setup, failed assertion, incomplete capture, and skipped evidence in the report.

The `arms` command has its own extensive interface; inspect `python scripts/scenarios.py arms --help` before using it. It can repeatedly vary startup/tunables and has recovery controls. `exclude` alters whether a prior run participates in history, so use only with an explicit reason and keep the original evidence.

## Human-assisted cards

`ut.py` exposes `list`, `run CARD`, and `verify CARD [--dir DIR]`, with top-level `--endpoint`, `--client-endpoint`, and `--player`. `run` requires a live local stack and a human for physical actions. `verify` judges saved logs offline where the card's assertions permit it. A physical-movement assertion is not satisfied by an offline verification that cannot observe movement; report unavailable checks as such rather than as passes.

## Reading saved evidence

Use `explain` to inspect a scenario's expected evidence before running it; `explain --run <run>` describes a saved execution. `history` compares prior records for a scenario, while `status` and `wait` inspect current runner state. `rescore [run]` recalculates interpretation from saved evidence; it is not a replay and should not overwrite the original record. The selected run can be made explicit when reviewing multiple samples:

```powershell
python scripts/scenarios.py explain <scenario-name>
python scripts/scenarios.py explain <scenario-name> --run <run-id>
python scripts/scenarios.py rescore <run-id>
python scripts/scenarios.py status
```

The offline tests in `scripts/scenarios_test.py` exercise gates, boot recovery, multi-client evidence, watchdogs, and run-status handling. In particular, test coverage is not a live acceptance result; report setup refusal, a failed assertion, missing evidence, and a passing run as distinct outcomes.

## Results and evidence

Keep scenario definition, sample record, run report, relevant logs, and build identity together locally. A rescore should preserve the original record and be treated as a new detector interpretation. Compare the selected assertions and skipped checks; a green summary for a subset is not a full-suite result. Use unique run identities and never overwrite an earlier failure to hide it.

## Related pages

[Scenario catalog](scenario-catalog.md) · [Quick loop](quick-loop-reference.md) · [Test strategy](test-strategy.md) · [Logging](logging-diagnostics.md) · [Performance profiling](performance-profiling.md)
## Quick loop

`scripts/quick_loop.py` runs short, controlled local integration checks. Its top-level verbs are `fork-install`, `fork-restore`, `fork-cycle`, `boot-times`, `active-household`, and `run`; use the checked-out help as the syntax authority:

```powershell
python scripts/quick_loop.py --help
python scripts/quick_loop.py run --help
```

`run` can optionally build/install the mod and boot the local stack before running ordered `--step` values. It supports `--clients`, `--parallel` / `--sequential`, repeatable `--server-env` and `--client-env`, `--sims`, `--bridge-trace`, `--live-needs`, and `--no-gc-watch`. `--mod` has an effect only with `--boot`; `--clients` is also a boot option. These options can launch or alter real local game processes, so only use an operator-owned disposable profile.

The step runner recognizes the following forms; no maintainer-local probe filenames or paths are documented here:

| Prefix | Form | Behavior |
|---|---|---|
| `sleep` | `sleep:<seconds>` | Wait for the specified duration. |
| `wait` | `wait:<role>:<sim>:<regex>:<seconds>` | Poll presentation state; timeout is reported but does not fail the run by itself. |
| `until` | `until:<role>:<sim>:<regex>:<seconds>` | Same poll, but timeout fails the step. |
| `server`, `client` | `<role>:<bridge-command> [arguments]` | Send a bridge command to that local role; may mutate the game. |
| `snap` | `snap[:label]` | Capture selected household Sims' state on both sides. |
| `compare` | `compare:<seconds>[:label]` | Compare client presentation over a bounded interval. |
| `diag` | `diag` | Read client replay/presentation diagnostics. |
| `act` | `act:<sim_act arguments>` | Invoke the simulation action helper against the server. |
| `rejoin` | `rejoin[:player][:paused]` or `rejoin:warm` | Restart a client while keeping the server alive; `warm` is client 1 only. |
| `sims`, `motion` | `<prefix>[:helper arguments]` | Run state-comparison or motion observer helpers. |
| `cpick`, `hud`, `notice`, `nonotice`, `notices:mark`, `menudiff`, `objectdiff` | `<prefix>:<arguments>` | Exercise the corresponding client input, HUD, notice, menu, or object comparison helper. |
| `capture` | `capture:<sim>[:pairs[:interval]]` | Focus cameras and capture paired frames. |

The `probe` step can execute a maintainer-local C# probe in a selected role; probe source is not shipped with the public repository. Historical fork install/restore steps also depend on local deployment materials and are not a reproducible public setup workflow.

A minimal read/observation example still requires a configured local stack:

```powershell
python scripts/quick_loop.py run --step sleep:1 --step snap:baseline --step diag
```

A quick-loop pass covers only the selected steps. It is not a substitute for unit tests, the reusable scenario suite, persistence tests, or multi-client acceptance. Record whether the command booted the stack, installed the mod, enabled tracing, and how many clients participated. After interruption, inspect stack status and verify cleanup before retrying.

## Related pages

[Scenario CLI](tool-reference-scenarios.md) · [stackctl](stackctl-reference.md) · [Test strategy](test-strategy.md) · [Run clients](run-client.md) · [Performance profiling](performance-profiling.md)
