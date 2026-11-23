# Development setup

This guide covers the open-source controller and documentation workflow. Live game testing
requires the developer to supply a compatible, legally acquired installation of The Sims 3;
the game and its data are not included.

## Prerequisites

- .NET 8 SDK for the controller, client, and test projects.
- Python and the dependencies listed in `requirements-dev.txt` for repository checks.
- Platform-specific tools only for the component being built (for example, the native SimLink
  project or in-game script mod).
- A compatible game installation for live integration testing.

Use the scripts' `--help` output for exact platform and configuration requirements. A .NET build
does not require the game. The mod, native sidecar, and live scenarios have separate build and
installation steps.

The public checkout does not include Sims 3 reference assemblies. Building the in-game mod
requires locally supplied assemblies or API-only stubs that you are legally permitted to use; the
project does not grant redistribution rights. See the [reference assembly policy](https://github.com/ObsidianNetwork/Sims3Online/blob/main/refs/sims3/README.md).

## Clone and build

```sh
git clone https://github.com/ObsidianNetwork/Sims3Online.git
cd Sims3Online
dotnet build DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj
dotnet build SimsOnline.Client/SimsOnline.Client.csproj
dotnet build SimsOnline.TestClient/SimsOnline.TestClient.csproj
```

Note: The project is branded **Sims3Online**, while solution, project, assembly, and environment-variable identifiers (`SimsOnline.sln`, `SimsOnlineServerHostMod`, `SimsOnline.Client`, `SimsOnline.Protocol`, `SIMSONLINE_*`) retain `SimsOnline*` for compatibility.

The test-project build also builds its referenced server project. The three commands are shown
separately so a contributor can tell which boundary failed.

## Run repository checks

```sh
python -m pip install -r requirements-dev.txt
python scripts/dev.py doctor
python scripts/dev.py check
```

`dev.py check` runs the lint and Python test set configured for CI. To include .NET tests, use
`python scripts/dev.py test --dotnet`. For all locally available Python tests, including
tool-specific tests, use `python scripts/dev.py test --all`. See [testing](testing.md).

## Documentation site

The site uses MkDocs Material. From the repository root, install the pinned documentation
dependencies and start the live preview:

```sh
python -m pip install -r requirements-docs.txt
mkdocs serve
```

Open the local address printed by MkDocs (normally `http://127.0.0.1:8000/`). Changes to Markdown
are rebuilt automatically. Validate the deployable output with:

```sh
mkdocs build --strict
```

The generated `site/` directory is build output and should not be committed. CI uploads the
built site as an artifact for review without publishing it to Pages. The trusted Pages deployment
workflow runs only on `main` and requires the self-hosted runner labelled `docs`; it is not used for
pull-request validation.

## Live game work

Live work depends on a local game installation and configured controller/mod integration. Start
with the help for `python scripts/stackctl.py`, then use `python scripts/scenarios.py list` to
see available scenario names. Do not expose the unauthenticated server listener to an untrusted
network. Never commit game files, saves, extracted assets, credentials, or machine-specific
paths.
