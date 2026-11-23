# Project status and evidence

Sims3Online is experimental multiplayer engineering software, not a player-ready release.
Use the [current status](../status.md) for the maintained capability summary and the
[compatibility matrix](compatibility-matrix.md) for tested environment/feature combinations.
This page explains how to describe evidence without turning an implementation detail or
test result into a broader support promise.

## Maturity terms

| Term | Minimum evidence | Does not establish by itself |
|---|---|---|
| Implemented | A current source path exists. | That the path works in a running game. |
| Tested | A named automated test or scenario passed under recorded inputs and conditions. | Compatibility outside that test's inputs and environment. |
| Partially verified | Some relevant cases have evidence, while important cases remain untested or fail. | General support or release readiness. |
| Supported | Maintainers have defined the environment and acceptance boundary and have evidence for it. | Compatibility with unspecified configurations. |
| Released | An identifiable version and its artifacts have been published. | That every feature is supported. |

Do not promote a claim from one level to another without additional evidence. For example,
a successful build proves compilation for its target; it does not prove that the game loaded
the mod or that a live interaction completed.

## Evidence boundaries

| Evidence source | Can support | Cannot prove alone |
|---|---|---|
| Source implementation | A code path or data contract exists in this revision. | Runtime success or broad compatibility. |
| Unit test | A bounded behavior passed under the test's conditions. | Game-process integration or user-visible behavior. |
| Integration test | Named components cooperated through a tested boundary. | Every game build, save, or hardware environment. |
| Live scenario | A named setup behaved as recorded. | Universal compatibility or security. |
| Release artifact | A versioned package was published. | That all repository features are included or supported. |
| Issue report | A reporter observed a condition. | That the condition is universally reproducible. |

For live-game evidence, name the game/mod build, scenario, and relevant setup. Keep saves,
game files, private logs, personal paths, and account data out of public reports.

## Writing a capability statement

- Describe the behavior and boundary that were actually tested.
- Name the test or scenario and its environment when making a verification claim.
- Keep known limitations close to the capability description.
- Distinguish current defaults from optional modes and experimental paths.
- Avoid unapproved dates, tracker identifiers, private run identifiers, and personal plans.
- Update the relevant status or compatibility entry when evidence changes.

Prefer a scoped statement such as “the protocol codec test covers resume-frame bounds” over
a broad claim such as “reconnect is reliable.” A single successful live run is evidence for
that setup, not a support guarantee.

## Where to record changes

Use the [current status page](../status.md) for a concise capability snapshot,
[known limitations](known-limitations.md) for constraints, and the
[compatibility matrix](compatibility-matrix.md) for environment-specific evidence. Link
from a status statement to the technical page or test that supports it. For setup limits,
see the [quickstart](quickstart.md); for test interpretation, see the
[test strategy](../guides/test-strategy.md).
