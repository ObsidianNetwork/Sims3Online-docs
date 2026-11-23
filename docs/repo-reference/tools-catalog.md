# Tools directory catalog

`git ls-files tools` reports 205 tracked files across test kits, native/managed analysis, interop experiments, research helpers, and local infrastructure. This is not a single installable product; dependencies and side effects vary. This catalog intentionally omits internal agent/coordination harness material. Read each tool's README, module docstring and current help before use.

## Main groups

| Group | Tracked files | Purpose and boundaries |
|---|---:|---|
| `tools/native_observatory/` | 90 | Evidence contracts, inventory, static analysis, anchor mapping, campaign orchestration, archive assembly/publication, comparisons and toolchain bootstrap. See [native-observatory reference](native-observatory-reference.md). |
| `tools/colab_offload/` | 21 | Bounded source-contract and analysis helpers, synthetic fixtures and config validation. `ghidra_to_rust_colab.py` may contact a configured remote endpoint and handles private research material. Its compiler gate proves syntax/layout checks, not behavioral equivalence. |
| `tools/decompile_lane/` | 19 | Experimental decompilation/recompilation adapters, worker/job helpers, candidate verification and benchmarks; configuration files name optional backends. Not a general-purpose decompiler service or proof of behavior. |
| `tools/tester_kit/` | 14 | Friend-side install/join/uninstall/log-collection scripts, command wrappers and fork-builder project. README states it is local test tooling, not a player distribution. |
| `tools/ide/` | 11 | Reference project files for game-facing .NET APIs. They are compile-time declarations/project scaffolding, not an engine implementation; redistribution rights are a separate question. |
| `tools/monohost/` | 7 | Native Mono-host prototype plus a logic test; requires CMake and native compiler toolchain. |
| `tools/managed_recompile/` | 5 | Managed IL effect modeling and recompile helper source; experimental outputs need independent verification. |
| `tools/simnav_route_probe/` | 5 | Rust route probe crate and tests, distinct from the main server runtime. |
| `tools/research/` | 3 | Notebook-oriented research helpers and tests; external service credentials, if used, stay local. |
| Small Rust codec probe | 3 | Standalone protocol-codec experiment; not the public server CLI. |
| `tools/managed_decompile_metadata/` | 2 | .NET metadata extraction helper; only use with authorized assemblies. |
| `tools/ghidra/` | 1 | Ghidra Java script; requires separately installed Ghidra and user-provided inputs. |
| `tools/probes/` | 1 | Focused probe source used by local tests, not a player-facing component. |
| Root helpers | 3 | Package marker and standalone notebook/usage utilities; inspect before running because they may invoke external services or alter local resources. |

## Tester kit

The `tools/tester_kit/` folder holds install, join, uninstall and log-collection scripts; `.cmd` files wrap PowerShell entry points. `wire_test.py` checks the kit connection/wire behavior. The root orchestrator is `scripts/tester_kit.py`; `python scripts/tester_kit.py --help` lists `build`, `verify`, `install-box`, `list` and `check`. Help was inspected only; no kit was built or installed. Generated kits are local artifacts, not files to commit; they may contain third-party or game-derived material.

## Colab/offload helpers

`ghidra_to_rust_colab.py` takes an input contract and requires `--output` and `--evidence-dir`; optional flags select endpoint config, metadata, symbol, authored contract, trace assertions and bounded repair passes. The module states that remote responses and derivative ABI output are private research evidence, and that compilation establishes syntax/layout safety, never behavioral equivalence. Related tests include `ghidra_to_rust_colab_test.py` and `extract_pilot_anchors_test.py`. Tests do not authorize a live endpoint or private source upload.

## Build and verification entry points

| Task | Entry point |
|---|---|
| Evidence archive CLI usage | `python -m tools.native_observatory.build_evidence_archive --help` |
| Route-probe Rust checks | `cargo test --manifest-path tools/simnav_route_probe/Cargo.toml` |
| Mono-host prototype | Inspect `tools/monohost/CMakeLists.txt` and its CMake test target; requires native toolchain. |
| Managed metadata helper | Build `tools/managed_decompile_metadata/ManagedRecovery.csproj` with .NET SDK; input must be authorized. |
| Native-observatory entry points | Use `python -m tools.native_observatory.<module> --help` so package imports resolve; do not invoke a campaign/validation/publish operation just to inspect usage. |

No live game, remote service, decompiler, archive publication or kit installation was executed for this catalog. `tools/ide/` reference declarations and native analysis tooling do not confer permission to redistribute proprietary input.

## General limits

Tools may access local processes, saved games, packages, analysis binaries, external endpoints or credentials. A structurally valid report can still lack behavioral evidence or use the wrong baseline. Platform boundaries matter: Windows process attachment and Linux/POSIX-safe evidence publication are different capabilities. Inspect every tool's input/output boundary before using it with private data.

## Related

- [Native observatory](native-observatory-reference.md)
- [Script catalog](scripts-catalog.md)
- [Testing](../testing.md)
- [Repository hygiene](repo-hygiene.md)
