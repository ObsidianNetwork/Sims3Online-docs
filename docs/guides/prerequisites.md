# Prerequisites and environment

The repository supports three distinct tasks: source-only work, game-mod compilation, and live multiplayer runs. Do not install or copy game files merely to build the managed controllers.

## Capability tiers

| Tier | Needed | Enables |
|---|---|---|
| T0 | .NET 8 SDK selected by `global.json` | Build and run the managed controllers and .NET tests. |
| T1 | Python 3.12+, `requirements-dev.txt`, PowerShell 7; Node.js is optional | Repository tooling, lint, helper scripts, and automated tool tests. |
| T2 | A legally acquired local game installation, a compatible .NET Framework 2 compiler, and user-supplied game assemblies when a task requires them | Compile the script mod and perform game-backed checks; packaging additionally needs the tools below. |

`python scripts/dev.py doctor` reports which tiers are available. Its output distinguishes required checks from optional ones; an absent optional Node installation does not make T0 unavailable. The checked-in `global.json` requests SDK 8.0.100 and permits `latestFeature` roll-forward.

The complete `make-mod.ps1` build/package path also needs a .NET Framework 4 compiler
(`Find-Csc40`) and s3pe's `s3pi.Settings`, `s3pi.Interfaces`, `s3pi.Package`,
`s3pi.ScriptResource`, and `s3pi.NameMapResource` assemblies. These requirements come
from `SimsOnlineServerHostMod/package-server-host-mod.ps1`; the T2 compiler check in
`doctor` does not verify them. On Linux, packaging runs the CLR4 builder through Wine
and expects the s3pe assemblies in the selected Wine prefix. Supply permitted tools
locally; do not add them to the public repository.

## Install the source-tooling dependencies

```powershell
python -m pip install -r requirements-dev.txt
python scripts/dev.py doctor
```

The manifest pins `pytest` and `ruff`, alongside optional tooling libraries used by analysis scripts. The .NET build does not depend on these Python packages. PowerShell parsing in `dev.py lint` runs only when `pwsh` is available.

## Game files and reference assemblies

A real game install is necessary only for workflows that launch the game, load its save, or compile against its embedded API. The public checkout's `refs/sims3/` directory contains the policy README but no DLLs. Redistribution rights for compile references are unverified: users must separately provide references they are legally permitted to use. Never put game binaries, saves, or locally supplied assemblies in a public issue or package.

On Windows, `SIMSONLINE_TS3_PATH` can point the tools at a non-default `TS3W.exe`; `scripts/ts3_paths.py` performs discovery. Linux live runs additionally depend on the user's Wine installation and prefix. Neither is required for source-only .NET testing.

## Setup failures

| Symptom | Check |
|---|---|
| Wrong .NET SDK selected | `dotnet --list-sdks` and `global.json`; do not change target frameworks to work around an absent SDK. |
| Python lint/test imports fail | Install `requirements-dev.txt` into the Python environment used to run `scripts/dev.py`. |
| Mod compiler unavailable | `doctor` checks for the legacy compiler; this is a T2 limitation, not a failure of `dotnet build`. |
| Game cannot be found | Check `SIMSONLINE_TS3_PATH` and the selected local install. Do not paste the local path into public reports. |
| Mod compiles but does not run in-game | On Windows, run `python scripts/check_mod_runtime.py`; this checker requires `WINDIR` and the desktop .NET 2 compiler, not Linux/Wine. Modern desktop API availability is not proof of game-runtime availability. |

## Related pages

[Build from source](build-source.md) | [Environment variables](environment-variables.md) | [Test strategy](test-strategy.md) | [Run a server](run-server.md)
