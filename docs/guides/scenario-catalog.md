# Scenario catalog

The checked-in `scenarios/` directory contains JSON-driven live scenarios. `scripts/scenarios.py` discovers their names and runs their steps against the local server/client game setup; these are not offline unit tests.

## Browse and run

```powershell
python scripts/scenarios.py list
python scripts/scenarios.py health
python scripts/scenarios.py run sit-dining
python scripts/scenarios.py history --last 5
python scripts/scenarios.py explain sit-dining --run latest
```

`run` accepts scenario names, repeat count, boot options, client count, watcher settings, environment overrides, rejoin mode, and soak controls. The exact options are in `python scripts/scenarios.py run --help`. Use `health` first: a broken host/client process makes subsequent scenarios blocked, not meaningful failures.

## Starter scenarios

Start a [first live multiplayer session](first-live-session.md) with one host and one mirror. Use a freshly loaded disposable save with the same household and starting area as the checked-in `scenarios/start/home.json`. The repository provides the scenario definitions and coordinates. Create or supply the compatible save locally.

| Case | Prepare in the test world | What the runner captures |
|---|---|---|
| `sit-dining` | Sims named Jennifer and Pablo, the client claim on Jennifer, and a reachable dining chair near the reset location. | Jennifer's route to the chair, seated state, and motion evidence from both games. |
| `phone-standing` | The same household and claim, with a compatible smartphone in Jennifer's inventory that offers Browse the Web. | Jennifer's standing phone interaction, held prop, and motion evidence from both games. |

```powershell
python scripts/scenarios.py health
python scripts/scenarios.py run phone-standing
python scripts/scenarios.py explain phone-standing --run latest
```

Run against the stack already started above so it uses your disposable save. Check the actual verdict in `.omo/run/scenarios/<run>/report.md`. The runner executes selected files in catalog order. In the JSON, `expect` describes the expected result for scoring. Read `explain` for a failed or blocked case.

### Observed results on the development fixture

The [verified Windows session](first-live-session.md#verified-session) ran these cases without the optional local conversation probes. `phone-standing` passed with matching end-state checks. `sit-dining` completed the host action and reached a seated state on both games, then failed the motion verdict for a client-only placement jump and turn. Run `python scripts/scenarios.py run sit-dining` to reproduce the check and `python scripts/scenarios.py explain sit-dining --run latest` to inspect the events.

The client bridge stopped responding during a follow-up `shower` run. The watchdog stopped the steps and recorded `CRASHED`. The planned `sleep-bed` case did not run. Recheck `health` before starting another case if a bridge goes silent.

Both cases ask the host to start the interaction and check the resulting presentation on the mirror. Use the [manual input check](../repo-reference/scenarios-catalog.md#manual-checks-for-input-and-game-time) to exercise client input capture separately.

Neither definition requests a maintainer-local probe. The shared reset also attempts to read conversation state with an optional local probe. A clean checkout can log `conversations … not read`. Start from a fresh save with both Sims idle and out of conversation to keep that optional check out of the acceptance path. Conversation and multi-client scenarios have their own additional prerequisites. Check both the `steps` and `after` lists for `scratch/` probe paths before running a different case.

## Examples from the repository

| Scenario | Coverage intent from its file/name |
|---|---|
| `sit-dining` | Host-driven seating and mirrored seated state. |
| `sleep-bed` | Bed interaction and sleep-state presentation. |
| `social-chat` | Social interaction and paired presentation. |
| `two-chairs` | Two players selecting distinct seats. |
| `two-same-chair` | Contention for one seat. |
| `late-join-bed` | Client joins during an already-running interaction. |
| `two-lots-late-join` | Join while participants are on separate lots. |

Use the scenario JSON for exact setup, waits, selectors, and assertions. When recording an observed result, include the fixture, game build, and repository revision that produced it.

## Result interpretation

- **Pass** means the configured assertions matched the evidence recorded for that run.
- **Fail** means at least one scenario check failed, including a step, assertion, motion watch, or server/client comparison; inspect `explain` and raw evidence before changing code.
- **Blocked** means a prerequisite or host-health check prevented a meaningful scenario execution.
- **Rescore** re-evaluates saved motion evidence with current detectors; it does not rerun the game.

The runner also offers `status`, `wait`, `rescore`, `during`, `arms`, and `exposure` for saved-run inspection and controlled comparisons. Do not treat repeated runs on a changed world/save as identical inputs.

## Add a scenario safely

Keep scenario JSON focused on one observable behavior. Declare the required actors, setup, operations, waits, and explicit success conditions. Use a disposable save and a stable start state. A scenario that relies on an untracked maintainer-local probe must say so without implying that the probe ships in this repository.

## Related pages

[Test strategy](test-strategy.md) | [Performance profiling](performance-profiling.md) | [Run clients](run-client.md) | [Debugging](debugging.md)
