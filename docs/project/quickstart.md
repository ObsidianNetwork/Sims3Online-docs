# Quickstart

Start with a clean checkout and verify the controller and tooling before booting the game.
The first three steps use the repository alone. [Development setup](../development.md) covers
platform-specific build and mod requirements.

## 1. Prepare your tools

| Tool | What it enables |
|---|---|
| Git and access to the repository | Clone the source and identify the revision you test |
| .NET 8 SDK | Build the controller, clients, and managed tests |
| Python 3.12 or newer | Install and run the repository tooling checks |
| PowerShell 7 | Run PowerShell-based checks and development tools |

Node.js is optional for the tooling's Node test coverage. See the [prerequisite tiers](../guides/prerequisites.md)
for which checks need a local game installation or a legacy compiler.

## 2. Clone and build

Run from a terminal with Git and `dotnet` on your path. Stay in the repository root for
the remaining commands.

```sh
git clone https://github.com/ObsidianNetwork/Sims3Online.git
cd Sims3Online
dotnet --version
dotnet build SimsOnline.Client/SimsOnline.Client.csproj
dotnet build SimsOnline.TestClient/SimsOnline.TestClient.csproj
dotnet build DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj
```

`dotnet --version` should report an 8.0 SDK. Each build should exit successfully. The test
project builds the dedicated server through its project reference. If an SDK cannot be
selected, check `dotnet --list-sdks` and the SDK selection in `global.json`. (The project is
branded **Sims3Online**, while solution, project, assembly, and environment identifiers retain
`SimsOnline*` for compatibility.)

## 3. Run the first checks

Use a Python environment for this checkout. The install and check commands must use the
same Python interpreter.

```sh
python --version
python -m pip install -r requirements-dev.txt
python scripts/dev.py doctor
python scripts/dev.py check
dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter all
```

Look for `ALL PASSED` from the tooling tests and `RESULT total=N failed=0 skipped=K`
from the managed test runner. Record any skipped cases with their reasons. The
`doctor` report shows which source and live-game prerequisites it found. A source-only
checkout can proceed with the source checks when it lacks the local game inputs.

Use the [testing guide](../testing.md) to run the separate Rust authority sidecar suite,
Windows SimLink cases, or documentation build. For CLI failures, start with the
[troubleshooting guide](troubleshooting-extended.md).

## 4. Prepare a disposable live session

1. Check the [current status](../status.md) and [compatibility matrix](compatibility-matrix.md)
   for the scope of the project's game-build and multiplayer evidence.
2. Configure your own compatible game installation using the
   [live prerequisites](../guides/prerequisites.md). Back up the save and select a disposable
   copy. Use separate local profiles for separate game processes.
3. Follow the [first live multiplayer session](../guides/first-live-session.md) to verify
   the host, client join, and applied baseline on loopback.
4. Use a matching fixture to run the [starter scenarios](../guides/scenario-catalog.md#starter-scenarios).
   Record the scenario verdict and the host and client revisions.

The transport lacks authentication and encryption. Follow the [security guide](../security.md)
for trusted network use. If you need help, include the command, sanitized error, and last
completed step in your [support report](support-and-reporting-issues.md). Keep saves, game
files, credentials, and private logs out of the report.
