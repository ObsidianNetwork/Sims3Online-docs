# Developer tooling reference

Prefer the repository's maintained command-line front doors over reproducing their setup logic
in ad hoc scripts. Check the current `--help` output before using less common options; flags
change as the tools evolve.

## Repository checks

| Command | Purpose |
|---|---|
| `python scripts/dev.py doctor` | Report which development tiers have their prerequisites |
| `python scripts/dev.py lint` | Run the Python lint and optional PowerShell parse check |
| `python scripts/dev.py test` | Run the Python tests and CLI help checks used by CI |
| `python scripts/dev.py test --dotnet` | Include the .NET build and test suite |
| `python scripts/dev.py test --all` | Attempt all locally available Python tests |
| `python scripts/dev.py check` | Run the standard lint and test checks |
| `dotnet build DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj` | Build server and test project |
| `dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter all` | Run the custom .NET test runner |

Install Python development dependencies with `python -m pip install -r requirements-dev.txt`.
Name the test project's `.csproj` explicitly: `DedicatedServer.Tests/` contains more than one
project, so selecting the directory alone is ambiguous.

## Local stack lifecycle

`python scripts/stackctl.py --help` documents the stack lifecycle CLI. Its commands cover
starting the server and client mirrors, checking readiness, rebuilding a controller, reloading
the mod, viewing logs, and stopping stack-owned processes. Use the command's help for required
game paths and platform options. The stack is local development tooling; its listener is not
intended to be exposed to untrusted networks.

Useful read-only checks include:

```sh
python scripts/stackctl.py status
python scripts/stackctl.py wait-ready --require-client
python scripts/stackctl.py logs
```

Starting, restarting, or stopping game processes can affect an active local session. Do that
only in an environment you control.

## Live scenarios

`python scripts/scenarios.py --help` lists the scenario runner's operations. Start with
`list`, `health`, and `history`; use `run` only when the required game stack is configured.
`explain` and `status` help interpret saved evidence without rerunning the game.

Scenario results are meaningful only when the stated prerequisites and target client are
available. A controller unit test is not a substitute for a live presentation check. See the
[scenario guide](guides/scenario-catalog.md) and [test strategy](guides/test-strategy.md) for
the evidence and prerequisite boundaries.

## Documentation

```sh
python -m pip install -r requirements-docs.txt
mkdocs serve
mkdocs build --strict
```

The local preview rebuilds on edits. The strict build is the CI check; CI also uploads the
generated `site/` directory as a preview artifact. It is separate from the trusted Pages
deployment described in [CI and deployment](guides/ci.md).
