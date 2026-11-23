# Native analysis and evidence tools

The repository's native-analysis code is optional tooling for authorized local binaries. It is not required for normal builds, tests, or game launch. The checked-in `tools/native_observatory` code emphasizes input identity, a pinned toolchain contract, bounded analysis, and validated output; a successful command still does not make an interpretation true or an artifact redistributable.

## Headless analysis entry point

`scripts` and `tools` include distinct utilities; the formal headless entry point is `tools/native_observatory/run_headless_analysis.py`. Its parser requires these inputs:

```text
--binary-inventory FILE
--analysis-identity FILE
--signature-scan FILE
--input-binary FILE
--ghidra-install DIR
--ghidra-archive FILE
--java-home DIR
--java-archive FILE
--output FILE
```

`--project-dir DIR` is optional. Verify exact syntax without opening an artifact:

```powershell
python tools/native_observatory/run_headless_analysis.py --help
```

The controller checks that the binary identity and signature scan agree, the executable is x86 and matches the inventory digest/size, the toolchain is verified, and requested anchors have unique matches. It launches headless analysis with the repository's profile, then rejects missing, stale, malformed, or identity-mismatched outputs. The run is bounded by the profile's per-file timeout plus a fixed outer allowance; it does not accept a pre-existing formal project/output as a fresh result.

The tool writes local run evidence and project state. Do not paste a real binary path, private installation path, raw analysis project, or game-derived disassembly into a public report. Supply only authorized local inputs and do not distribute game binaries, full symbol tables, decompiler databases, or large pseudocode exports.

## Shared Ghidra maintenance

`scripts/ghidra_apply_icall_names.py` defaults to a dry run and reports aggregate action counts. It lists up to 40 conflicts and counts any remaining conflicts. Use `python scripts/ghidra_lock.py status` to check who holds the shared Ghidra GUI lock. Applying a reviewed update requires acquiring the lock and using `--apply --holder` with the same holder name. Release it when finished. GUI edits are local maintenance. Use the headless controller above when you need a reproducible run record.

## Evidence levels

| Artifact | What it establishes | What it does not establish |
|---|---|---|
| Binary inventory/digest | Identity of selected bytes under the recorded schema | License or behavior |
| Unique signature match | A selected byte pattern matched once in that input | Function purpose or runtime use |
| Analysis profile/toolchain record | Which configured toolchain and analysis settings were used | That a proposed name is correct |
| Exported metadata/call relationship | A static relationship in the selected build | That the path executes in a live game |
| Runtime observation | What happened in the observed run | General behavior across versions |

Keep observation, symbol mapping, inference, and uncertainty separate. An import, string, or candidate name is a navigation lead, not proof of game behavior. Corroborate important findings with project-owned source/tests or an authorized controlled observation.

## Resource and platform constraints

Headless analysis can consume substantial CPU, memory, and disk. Work against a disposable output/project location and a verified local toolchain; do not run broad analysis alongside an active game session unless resource impact is understood. If the required toolchain or identity files are missing, stop rather than substituting an unverified version or producing an empty success report.

## Tool choice

Use checked-in source and managed/native boundary metadata first. Select any third-party disassembler based on the analysis question and local policy; this repository does not require an assistant-specific command-line tool. Keep external tools, binaries, and their outputs local unless their licensing and the exact output have been reviewed.

## Related pages

[Managed analysis](tool-reference-managed-analysis.md) · [Diagnostics](tool-reference-diagnostics.md) · [CI](ci.md) · [Release packaging](release-packaging.md)
## SimLink sidecar CLI

`scripts/sidecar.py` manages the optional x86 SimLink ASI component for a local game installation. It is not required for a normal managed build. Installing or switching it changes the selected installation; use a disposable profile and stop every game process first.

```powershell
python scripts/sidecar.py --help
python scripts/sidecar.py build --help
```

The current commands are `build [--generator NAME]`, `install`, `restore`, `status`, `on`, and `off`. With no action, the script defaults to `status`. `build` configures CMake for Win32 and builds the Release artifact; the default generator is Visual Studio 2022. A generator override is accepted only as `--generator <name>`. If the build folder was configured with a different generator, the script removes the incompatible CMake cache/configuration before configuring again.

| Command | Effect |
|---|---|
| `build` | Compile from `SimLink/` and print the built artifact hash. |
| `status` | Report on/off/absent state, loader presence, and hashes/version where readable. Exit 1 when absent. |
| `install` | Copy the built artifact into the selected game's local binary folder and preserve a replaced sidecar for rollback when it differs. |
| `restore` | Restore the saved prior sidecar when present; fails if there is no prior file. |
| `on` / `off` | Rename the sidecar between enabled and disabled local filenames. |

`install` and `restore` refuse to proceed while a `TS3W.exe` process is running. Stop the owned stack through its lifecycle tool and recheck status before continuing. Do not overwrite a file held by a running game.

### Runtime opt-in and boundaries

The native binary and managed controller/mod are separate pieces. In the server shadow path, the controller opt-in is `SIMSONLINE_SIMLINK_SHADOW=1`; after the sidecar handshake registers internal calls, the controller asks the mod to mirror recorder lines through `SACS_RECORDER sidecar-on`. Without opt-in, this is not a substitute for the ordinary managed transport. Existing game processes retain the image/settings they loaded at startup, so changes require a clean restart to compare modes.

Treat the sidecar binary and captured native records as local-only. Do not install it for a tester without explicit consent, package game binaries, or claim sidecar validation based only on the file being present.

### Controlled comparison

1. Run the normal managed path and record the selected build/profile.
2. Stop the game and use `status` to confirm the sidecar/loader state.
3. If testing the optional path, build from the reviewed checkout and inspect its identity.
4. Install or enable it only for the selected local game copy and set the intended opt-in before launch.
5. Verify the controller accepted at least one valid sidecar record; a file or process alone is insufficient.
6. Stop the stack, restore/disable the component, and verify the baseline path again.

`scripts/sidecar_test.py` covers CLI and rollback behavior without substituting for a live integration check. Report offline tests and any live validation separately.

## Related pages

[Environment variables](environment-variables.md) · [Performance profiling](performance-profiling.md) · [Bridge tools](bridge-tools.md) · [Release packaging](release-packaging.md) · [Test strategy](test-strategy.md)
