# Script and tooling reference

This page is an entry map to checked-in command-line tools, not an exhaustive listing of every Python file. A script filename alone does not make a supported operator CLI. Before using a new command, confirm the file and its parser, read its help when safe, and identify whether it reads evidence, writes files, launches processes, or mutates a game.

## Contributor and lifecycle commands

| Tool | Use | Safe discovery |
|---|---|---|
| `scripts/dev.py` | Environment inventory, lint, test, combined check | `python scripts/dev.py --help` |
| `scripts/stackctl.py` | Start/observe/stop owned local stack roles | `python scripts/stackctl.py --help` |
| `scripts/simctl.py` | Public session protocol client | `python scripts/simctl.py --help` |
| `scripts/bridgectl.py` | Local bridge-admin diagnostics and live probes | `python scripts/bridgectl.py --help` |
| `scripts/scenarios.py` | Scenario execution and saved-run analysis | `python scripts/scenarios.py --help` |
| `scripts/quick_loop.py` | Short local integration checks | `python scripts/quick_loop.py --help` |
| `scripts/sidecar.py` | Build/inspect/install optional native sidecar | `python scripts/sidecar.py --help` |

Help is a safe parser query for these commands. Most verbs beyond help have side effects: read each command's description before use.

## Evidence and analysis commands

| Tool | What it reads or checks | Important boundary |
|---|---|---|
| `scripts/check_docs.py` | Markdown paths and anchors | Does not validate product behavior. |
| `scripts/saves.py` | Local save backup/restore/swap operations | Some commands overwrite profile data; inspect its help and stop game processes first. |
| `scripts/dbpf_diff.py`, `scripts/dbpf_dump.py` | Local save/package container structure and records | Treat input and output as game-derived data; use copies and do not publish payloads. |
| `scripts/ts3src.py` | Indexed managed symbol/type navigation | Depends on user-supplied local assemblies and an index; static graph results can miss dynamic dispatch. |
| `scripts/reference_assembly.py`, `scripts/check_mod_runtime.py` | Compile-reference metadata and runtime API compatibility | A stub or static pass is not proof of game runtime behavior. |
| `tools/native_observatory/run_headless_analysis.py` | Bounded local native analysis orchestration | Requires explicit binary/toolchain inputs and produces local evidence; not required for normal builds. |

These entries intentionally name only files present in this checkout. Other analysis helpers may exist, but check their source and tests before documenting or using them. `--help` is not guaranteed for every utility; some small scripts print a usage form or accept positional input directly.

## Side-effect ladder

| Class | Typical effect | Operator practice |
|---|---|---|
| Help/parser inspection | Prints syntax | Safe first step. |
| Evidence read | Reads local logs, saves, assemblies, or reports | Work from copies; sanitize output. |
| File creation | Writes reports, backups, archives, or builds | Choose an isolated output and check overwrite behavior. |
| Process control | Starts, attaches to, or stops processes | Confirm role and ownership; use bounded waits. |
| Game mutation | Changes claims, speed, time, interactions, or world state | Obtain permission and use a disposable profile. |

When scripting a command, preserve its exit code and stdout/stderr separately. A zero transport/tool exit does not necessarily mean an application-level assertion passed. Never infer success from a newly created file without validating its schema and inputs.

## Choosing the narrowest utility

- Use `dev.py` for routine contributor checks; use the focused test module while iterating.
- Use `stackctl.py` for process lifecycle, `simctl.py` for the public session path, and `bridgectl.py` only for local privileged diagnostics.
- Use `scenarios.py` when setup, actions, and assertions should be recorded together; use `quick_loop.py` for a deliberately short local integration smoke.
- Use DBPF/save tools only against authorized local artifacts; use managed/native analyzers only when source or tests cannot answer the question.

## Related pages

[Developer CLI](dev-cli.md) · [stackctl](stackctl-reference.md) · [simctl](simctl-reference.md) · [Bridge tools](bridge-tools.md) · [Scenarios](tool-reference-scenarios.md) · [Analysis](tool-reference-analysis.md) · [Diagnostics](tool-reference-diagnostics.md)
