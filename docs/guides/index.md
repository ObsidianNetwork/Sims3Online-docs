# Engineering guides

Use these guides to set up a checkout, build components, run local checks, diagnose
failures, and package reviewed outputs. Commands that touch a live game require a
compatible game installation and a disposable profile/save. The guide documents
distinguish offline checks from game-backed evidence.

## Common workflows

| Goal | Reading path |
|---|---|
| First source build | [Prerequisites](prerequisites.md) → [build from source](build-source.md) → [developer CLI](dev-cli.md) |
| Run a local session | [configuration](configuration.md) → [server](run-server.md) → [clients](run-client.md) |
| Choose a test and interpret it | [test strategy](test-strategy.md) → [scenario catalog](scenario-catalog.md) → [scenario CLI](tool-reference-scenarios.md) |
| Diagnose a failed run | [debugging](debugging.md) → [logging](logging-diagnostics.md) → [diagnostic tools](tool-reference-diagnostics.md) |
| Review a release artifact | [release packaging](release-packaging.md) → [release and versioning](release-and-versioning.md) |

## Setup and operations

| Topic | Pages |
|---|---|
| Environment and configuration | [Prerequisites](prerequisites.md), [configuration](configuration.md), [environment variables](environment-variables.md) |
| Process launch | [Run a server](run-server.md), [run clients](run-client.md), [warm reattach](reattach-workflow.md) |
| Automated checks | [Developer CLI](dev-cli.md), [test strategy](test-strategy.md), [CI](ci.md) |
| Live testing | [Scenario catalog](scenario-catalog.md), [scenario and user-test CLI](tool-reference-scenarios.md), [quick loop](quick-loop-reference.md), [tester kit](tester-kit.md), [private testing boundaries](private-beta.md) |

## Command and tooling reference

| Tool or interface | Reference |
|---|---|
| Dedicated server | [Server options](server-options.md), [top-level server CLI](../server-cli.md) |
| Player client | [Client options](client-options.md) |
| Local stack lifecycle | [stackctl](stackctl-reference.md), [stackctl option summary](stackctl-options.md) |
| Protocol smoke client | [simctl](simctl-reference.md) |
| Host bridge | [Bridge tools](bridge-tools.md) |
| Repository scripts | [Script reference](script-reference.md) |
| Analysis and diagnostics | [Analysis tools](tool-reference-analysis.md), [diagnostics](tool-reference-diagnostics.md) |
| Managed and native inspection | [Managed analysis](tool-reference-managed-analysis.md), [native analysis](tool-reference-native-analysis.md), [dnSpy](dnspy.md) |
| Optional sidecar | [SimLink sidecar](tool-reference-sidecar.md) |
| Iteration and patching | [Hot reload](hot-reload.md), [hot injection](hot-injection.md), [live probes](live-probes.md) |
| Performance | [Performance profiling](performance-profiling.md) |

## Packaging and engineering practice

- [Release packaging](release-packaging.md) and [release/versioning](release-and-versioning.md) describe what CI builds and what a draft tag does not establish.
- [Tooling lessons](tooling-lessons.md) explains reliability patterns used by repository utilities.
- [MonoPatcher assessment](monopatcher.md) records why a runtime patcher is not part of the supported build path.
- [Capture and input pipeline](capture-input-pipeline.md) follows player input into validation.

## Scope and safety

The project is not a hosted game service. Keep the unauthenticated client listener
inside a trusted network, keep the privileged bridge-admin endpoint local, and never
use a valuable save for probes. Users supply their own legally acquired game and any
permitted local reference assemblies; neither is distributed here.

An option listed in a reference is not proof that its feature is supported. For
state-changing commands, inspect current `--help` or source, confirm the target
process/profile, and review cleanup before execution. See
[security](../security.md), [troubleshooting](../troubleshooting.md), and
[contributing](../contributing.md) for cross-cutting guidance.
