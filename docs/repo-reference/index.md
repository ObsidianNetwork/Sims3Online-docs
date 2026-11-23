# Repository reference

This section maps the implementation-facing parts of the repository: Rust crates, in-game scenarios, evidence schemas, developer scripts and tools, the native-observatory pipeline, CI, and source hygiene. Descriptions are grounded in checked-in manifests and code; consult those sources when details change.

## Reference map

| Page | Answers |
|---|---|
| [Rust workspace](rust-workspace.md) | Crate purposes, public root APIs, relationship to the current server and Cargo commands. |
| [Scenario catalog](scenarios-catalog.md) | Scenario JSON shape, intended coverage, real runner commands and recorded expected verdicts. |
| [Schema catalog](schemas-catalog.md) | Evidence schemas, required/top-level fields, and producer/validator entry points. |
| [Script catalog](scripts-catalog.md) | Functional groups among 285 tracked scripts and safe command-line front doors. |
| [Tools catalog](tools-catalog.md) | Functional groups among 205 tracked files under `tools/`, plus notable test/build tools. |
| [Native observatory](native-observatory-reference.md) | Evidence identity, archive sealing, anchor checks, CLI, tests and platform limits. |
| [CI and workflows](ci-and-workflows.md) | Workflow triggers, jobs, artifacts and local reproductions. |
| [Repository hygiene](repo-hygiene.md) | Ignore rules, source/proprietary-input guard, docs checker, and the clean-checkout boundary for ignored maintainer-local probes. |

## Reading boundaries

Player/server message contracts are maintained separately; this section does not restate them. See the [protocol reference index](../protocol-reference/index.md). The JSON documents under `schemas/evidence/` describe analysis and validation artifacts, not player network messages.

Build and test commands are entry points, not proof that each subsystem is a supported product feature. Live scenarios need a configured local game stack. The Rust crates describe experimental work and do not replace the checked-in C# server path. A scenario's desired `expect` value is not a test result; use a recent runner verdict.

## Related

- [Development setup](../development.md)
- [Testing](../testing.md)
- [Tooling overview](../tooling.md)
