# Testing

The repository has several evidence levels. Unit and protocol tests can prove controller
behavior, while presentation and engine integration require the game itself. Report only the
level that was actually run.

## .NET controller, client, and protocol tests

Build the client and test projects, then run the repository's custom test runner:

```sh
dotnet build SimsOnline.Client/SimsOnline.Client.csproj
dotnet build SimsOnline.TestClient/SimsOnline.TestClient.csproj
dotnet build DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj
dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter all
```

The runner reports individual cases and a final `RESULT total=N failed=M skipped=K` line. A
skipped case is not a passing case; include the skip count when reporting results.

## Rust authority sidecar tests

The `sidecar-tests` pull-request job builds the Rust authority daemon from source
and runs the full NUnit project against it. These tests cover the loopback
handshake and bootstrap, forwarded commands and refusals, route results, and
the seed stream. The fixture starts and stops its own daemon on loopback.
You need .NET 8 and a stable Rust toolchain installed through rustup.

From the repository root on Linux:

```sh
sidecar_target="$PWD/rust/sim96_kernel_cleanroom/target-sidecar-tests"
cargo +stable build --release \
  --manifest-path rust/sim96_kernel_cleanroom/Cargo.toml \
  --bin simlink_authority_daemon --target-dir "$sidecar_target" &&
  test -x "$sidecar_target/release/simlink_authority_daemon" &&
  SIMSONLINE_SIDECAR_DAEMON="$sidecar_target/release/simlink_authority_daemon" \
    dotnet test DedicatedServer.Tests/DedicatedServer.Tests.csproj \
    --logger "console;verbosity=normal" -- RunConfiguration.TreatNoTestsAsError=true
```

On Windows PowerShell:

```powershell
$sidecarTarget = Join-Path $PWD 'rust/sim96_kernel_cleanroom/target-sidecar-tests'
cargo +stable build --release --manifest-path rust/sim96_kernel_cleanroom/Cargo.toml --bin simlink_authority_daemon --target-dir "$sidecarTarget"
if ($LASTEXITCODE -ne 0) { throw "The sidecar build failed." }
$sidecarDaemon = Join-Path $sidecarTarget 'release/simlink_authority_daemon.exe'
if (!(Test-Path -LiteralPath $sidecarDaemon -PathType Leaf)) { throw "The sidecar executable is missing." }
$env:SIMSONLINE_SIDECAR_DAEMON = $sidecarDaemon
dotnet test DedicatedServer.Tests/DedicatedServer.Tests.csproj --logger "console;verbosity=normal" -- RunConfiguration.TreatNoTestsAsError=true
```

Run the suite without a test filter to match CI. The NUnit summary reports how many tests
ran, passed, or were skipped, and the run fails if it discovers zero tests. Report that
summary alongside the platform and commit. This suite exercises the controller-to-daemon
boundary without loading the game.

## Windows SimLink checks

The `simlink-windows` job builds the x86 native SimLink bridge using CMake and MSVC. It
runs the Windows-specific named memory map and native ring tests in the custom .NET runner.
Use the [native sidecar build guide](guides/tool-reference-sidecar.md) to build the bridge
with your installed toolchain, then run the same test selection as CI:

```powershell
dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter SimLink --fail-on-skip
```

These checks exercise Windows native transport. For evidence from an actual game process,
run an [in-game scenario](#in-game-scenarios) separately.

## Python tooling checks

Python development checks require the dependencies in `requirements-dev.txt`:

```sh
python -m pip install -r requirements-dev.txt
python scripts/dev.py doctor
python scripts/dev.py lint
python scripts/dev.py test
python scripts/dev.py check
```

`python scripts/dev.py test --dotnet` also runs the managed builds and custom test runner.
`python scripts/dev.py test --all` attempts every available Python test, including checks that
may require local tools or evidence. Some tests are platform-specific or depend on local
installations; the native-observatory archive publication suites require Linux `dir_fd` and
`renameat2` filesystem semantics and are skipped on other platforms. Consult each skip reason and
the relevant script's help rather than treating an unavailable prerequisite as a product failure.

## In-game scenarios

Scenario tests require the compatible game installation and a configured local stack. They are
separate from the controller test command. First inspect available scenarios and local health,
then run only the target scenarios:

```sh
python scripts/scenarios.py list
python scripts/scenarios.py health
python scripts/scenarios.py run --help
```

Use `python scripts/scenarios.py explain <scenario>` for saved failure evidence and
`python scripts/scenarios.py status` to inspect a run already in progress. Do not report a live
PASS unless the scenario ran against the required game processes and its verdict says PASS.

## Documentation

Validate documentation links and build the site before changing published pages:

```sh
python scripts/check_docs.py --anchors --strict README.md CONTRIBUTING.md SECURITY.md docs
mkdocs build --strict
mkdocs serve
```

## Reporting results

Include the exact command, platform, game version for live tests, and the failing test name or
scenario verdict. Do not attach saves, game binaries, personal paths, or private logs to public
issues.
