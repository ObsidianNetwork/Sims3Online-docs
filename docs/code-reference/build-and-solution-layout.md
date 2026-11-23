# Build and solution layout

The solution contains C# controller/client projects, design-time mod references, and
supporting C# tools. The repository also has independent Rust workspaces, a native CMake
sidecar, Python tooling, and a legacy-runtime mod build. These are related products but not
one uniformly buildable solution.

## Solution and project inventory

| Project / workspace | Kind | Target / build boundary | Main references |
|---|---|---|---|
| `DedicatedServer/SimsOnline.DedicatedServer.csproj` | .NET executable | .NET 8 via shared props | `SimsOnline.Protocol`; selected pure host-mod files source-linked. |
| `SimsOnline.Protocol/SimsOnline.Protocol.csproj` | .NET class library | .NET 8 via shared props | DTOs/codecs, no game runtime. |
| `SimsOnline.Client/SimsOnline.Client.csproj` | .NET executable | .NET 8 via shared props | DedicatedServer, Protocol, one linked mod filter. |
| `SimsOnline.TestClient/SimsOnline.TestClient.csproj` | .NET executable | .NET 8 via shared props | DedicatedServer, Protocol, linked `PublicWireConnection.cs`. |
| `DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj` | Custom test executable | .NET 8 via shared props | DedicatedServer, Protocol, numerous pure source-linked policy files. |
| `DedicatedServer.Tests/DedicatedServer.Tests.csproj` | NUnit test project | .NET 8 (set in project) | DedicatedServer, Protocol; compiles only `RustAuthoritySidecarTests.cs`. |
| `SimsOnlineServerHostMod/SimsOnlineServerHostMod.Ide.csproj` | Design-time library project | `netstandard2.0`, C# 5, x86, no standard library | Game API references / local IDE wrappers; not the production mod build. |
| `tools/managed_decompile_metadata/ManagedRecovery.csproj` | Utility executable | .NET 8 | Managed metadata recovery tooling. |
| `tools/managed_recompile/RecompileHelper/RecompileHelper.csproj` | Utility executable | .NET 8 | Recompile helper tooling. |
| `tools/native_observatory/managed_metadata/ManagedMetadataExtractor.csproj` | Utility executable | .NET 8 | Managed metadata extraction. |
| `tools/tester_kit/fork_builder/ForkTool/ForkTool.csproj` | Utility executable | .NET 8 | Fork/test harness helper. |
| `tools/ide/engine/*.csproj` (10 projects) | Design-time reference wrappers | Project-specific IDE inputs | `mscorlib`, `System`, `System.Xml`, `SimIFace`, `ScriptCore`, `Sims3Metadata`, `UI`, `Sims3GameplaySystems`, `Sims3GameplayObjects`, `Sims3StoreObjects`. |
| `rust/simlink_bridge/Cargo.toml` | Rust library + `simlinkctl` | Rust 2021; own workspace | Windows-only `windows-sys` dependency. |
| `rust/simlink_headless_server/Cargo.toml` | Rust library + executable | Rust 2021; own workspace | `simlink_bridge`, `sim96_kernel_cleanroom`, `mio`, `serde`, `serde_json`, `ctrlc`. |
| `rust/sim96_kernel_cleanroom/Cargo.toml` | Rust library + tools | Rust 2021; own workspace | Independent kernel crate and tooling. |
| `SimLink/CMakeLists.txt` | Native shared library/self-test | C++17, x86 only, MSVC/CMake | Builds `SimLink.asi`, `simlink_ring`, and `simlink_selftest`. |

`SimsOnline.sln` includes the controller, client/test client, custom C# tests, host-mod IDE
project, Protocol library, managed utility projects, and ten IDE engine-reference projects.
The Rust and CMake workspaces are built with their own tools rather than as .NET projects.

## Shared .NET settings

`global.json` requests SDK `8.0.100` and allows roll-forward to the latest feature band.
`Directory.Build.props` applies `net8.0`, implicit usings, nullable annotations, and
repository/package metadata only when `MSBuildProjectName` matches one of the five named
stack projects: DedicatedServer, Client, TestClient, DedicatedServer.Tests (custom runner),
or Protocol. The NUnit project has a different assembly/project identity and declares its
target framework itself.

The shared props file explicitly narrows its condition. This prevents unrelated tools and
generated projects from accidentally inheriting controller settings. When adding a .NET
stack project, check both its project name and the condition rather than assuming any
`.csproj` in the repository receives the defaults.

The controller project allows unsafe blocks because `SimLinkShadowReader` performs
compare-and-swap on a shared ring read position. The protocol library itself has no such
permission requirement. Release builds conditionally enable Sentry symbol/source upload only
when the expected build environment variables are present; this documentation does not
include credentials or their values.

## The mod is a separate build

`SimsOnlineServerHostMod.Ide.csproj` exists for language-server/code-navigation support. It
selects `LangVersion` 5, x86, disables implicit framework references, and has `NoStdLib`; it
is not the package builder's production compilation path. Its sources target the old game
runtime and compile against API references.

The actual host mod is built with scripts such as
`SimsOnlineServerHostMod/build-server-host-mod.ps1` and package/install scripts under that
directory. A full compile requires locally supplied reference inputs. CI checks whether these
inputs exist before invoking `scripts/check-mod-compile.ps1 -Strict`; on a clean public checkout
the gate is reported as unavailable and skipped. The .NET 8 test project source-links selected
pure files to test policy logic, but does not compile all engine-coupled mod source or prove live
game operation.

A few pure mod files are linked into other assemblies to avoid maintaining duplicate policy:
the controller links filters, wire record helpers, line-kind counts, and snapshot-related
code; the client links `OwnedHostDetourFilter.cs`; the custom test executable source-links
many policy helpers. The `.csproj` comments explain this split and warn against duplicate
compilation of the same public types.

## Reference assemblies and local-only folders

`refs/sims3/README.md` documents ten expected local filenames for **compile-only API references**.
The public checkout does not contain those files; redistribution rights have not been verified.
The names and declared assembly mapping are:

| Files | Declared reference assemblies |
|---|---|
| `gameplay_00.dll`–`gameplay_03.dll` | Store objects, gameplay objects, gameplay systems, and UI APIs. |
| `scripts_00.dll`–`scripts_02.dll` | SimIFace, ScriptCore, and Sims3Metadata APIs. |
| `simcore_00.dll`–`simcore_02.dll` | System.Xml, System, and mscorlib APIs. |

The README says users must obtain their own legal game copy and must not commit or
distribute original game assemblies, executables, packages, saves, or decompiled source. The
build references exist to compile against signatures; they do not bundle a playable game or
make these sources runnable on their own.

The root `.gitignore` ignores contents under `credentials/` and `signing/` except their
README placeholders. `credentials/README.md` says only that local configuration may be
stored there and that repository code does not read it. `signing/README.md` says local
signing keys/certificates are not read by repo code and releases are currently unsigned.
Only those README files are tracked; their ignored contents are intentionally not part of
this source tour. Do not copy, inspect, or document local secret/key values here.

## Native and Rust build boundaries

`SimLink/` is a C++17 ASI plugin, not a C# project. CMake configuration fails unless pointer
size is 32-bit; the supported target is x86 because it loads into the 32-bit game process.
`python scripts/sidecar.py build --generator "Visual Studio 18 2026"` is the current
CI-compatible entry point for the Windows build. The output includes an `.asi` library and a
`simlink_selftest` executable.

Each top-level Rust crate has its own `Cargo.toml` and `[workspace]`, so a single root
`cargo test` is not configured. `simlink_bridge` contains the process-memory bridge driver;
`simlink_headless_server` depends on the bridge and kernel; `sim96_kernel_cleanroom` is a
separate library/tool workspace. The current kernel manifest uses Rust 2021 and configures a
development profile; feature-gated utilities may need inputs not present in a public
checkout. Read each crate manifest before selecting a `cargo` command.

## CI build map

`.github/workflows/build-and-test.yml` is the authoritative routine CI sequence:

| Job | Platform | Commands / scope |
|---|---|---|
| `build-and-test` | Ubuntu | Build Client, TestClient, custom test project; run custom test cases with `--filter all`. |
| `sidecar-tests` | Ubuntu | Build the Rust authority daemon and run the full NUnit sidecar project with `dotnet test`. |
| `python-tools` | Ubuntu / Python 3.12 | Install `requirements-dev.txt`; run `python scripts/dev.py lint` then `python scripts/dev.py test`. |
| `proprietary-input-guard` | Ubuntu | Scan changed files for prohibited binary/private inputs; no game contents are distributed by this step. |
| `mod-compile-gate` | Windows | If all required local references are present, run `scripts/check-mod-compile.ps1 -Strict`; otherwise explicitly report that the mod compile was unavailable. |
| `simlink-windows` | Windows | Build x86 SimLink, then run `--filter SimLink --fail-on-skip` against native integration cases. |

The custom test project is built by reference from the test client build as needed, but CI
explicitly builds it. `dotnet test` is reserved for the separate NUnit adapter project; use
`dotnet run` for the custom test executable. The exact commands, feature filters, and skip
semantics are in [tests-map.md](tests-map.md).

## Practical build commands

The ordinary .NET projects use the .NET SDK declared by `global.json`:

```powershell
dotnet build SimsOnline.Client/SimsOnline.Client.csproj
dotnet build SimsOnline.TestClient/SimsOnline.TestClient.csproj
dotnet build DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj
dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter all
```

Run the optional NUnit project separately:

```powershell
dotnet test DedicatedServer.Tests/DedicatedServer.Tests.csproj
```

The full host-mod compile is Windows-specific and invoked by the repository's compile
script. A successful .NET controller build does not prove that the legacy mod compiles, that
a native ASI built, or that a local game installation can load the package. Those are
independent gates.

## Where changes belong

| Change | Edit / verify |
|---|---|
| New public command/event DTO | `SimsOnline.Protocol/`; update JSON and line-v1 fixtures/tests. |
| Controller startup flag | `DedicatedServer/SessionConfig.cs`, `DedicatedServer/Program.cs`; add focused parsing/validation test. |
| Game bridge verb | Controller `BridgeCommand`/`BridgeLineCodec` and mod `BridgeCodec`/`ServerHostMain`; add parity coverage. |
| Pure mod policy | `SimsOnlineServerHostMod/`; include source link only if other assemblies need to test the exact logic. |
| Native shared-memory layout | `SimLink/src/simlink_layout.h` and controller `SimLinkShadowReader`; bump/validate layout together. |
| Rust crate behavior | Owning crate under `rust/`; don't infer it is part of the .NET solution. |
| Secret/signing workflow | Keep values in external configuration; do not commit them or add values to docs. |

For source ownership and runtime flow see [index](index.md). For the controller's actual CLI
and host lifecycle see [dedicated-server.md](dedicated-server.md); for the native target see
[simlink.md](simlink.md).

## Project-reference details

The project-reference graph is intentionally small around the public runtime:

| Project | Direct references declared in source |
|---|---|
| `DedicatedServer` | `SimsOnline.Protocol`; selected mod wire/filter helpers are compile-linked. |
| `SimsOnline.Client` | `DedicatedServer`, `SimsOnline.Protocol`; `OwnedHostDetourFilter.cs` is linked once. |
| `SimsOnline.TestClient` | `DedicatedServer`, `SimsOnline.Protocol`; `PublicWireConnection.cs` is linked from the client. |
| Custom test executable | `DedicatedServer`, `SimsOnline.Protocol`; pure helper files linked from mod and client. |
| NUnit adapter project | `DedicatedServer`, `SimsOnline.Protocol`; one source file compiles the Rust authority adapter tests. |
| Host-mod design-time project | Engine wrapper projects or user-supplied local compile references, depending on local IDE configuration. |

Source links are deliberate. They let a test execute a pure policy implementation without
bringing the game runtime into the test process. They also mean an edit to a linked file can
affect more than one assembly, so verify all linked consumers and avoid adding the same
source file both directly and through a project reference.

`DedicatedServer/SimsOnline.DedicatedServer.csproj` references Sentry and Sentry.Profiling
6.12.1. It conditionally configures symbol/source upload in Release when the required
environment variables are supplied; repository docs never store the corresponding token
values. `SimsOnline.Client.csproj` has a similar conditional symbol/source-upload group.

## Compile-time API signature references

`SimsOnlineServerHostMod.Ide.csproj` maps the stubs to the managed assembly names documented
by `refs/sims3/README.md`:

| Expected local file | Declared API reference |
|---|---|
| `simcore_02.dll` | mscorlib |
| `simcore_01.dll` | System |
| `simcore_00.dll` | System.Xml |
| `scripts_00.dll` | SimIFace |
| `scripts_01.dll` | ScriptCore |
| `scripts_02.dll` | Sims3Metadata |
| `gameplay_00.dll` | Sims3StoreObjects |
| `gameplay_01.dll` | Sims3GameplayObjects |
| `gameplay_02.dll` | Sims3GameplaySystems |
| `gameplay_03.dll` | UI |

The corresponding IDE wrapper project files live under `tools/ide/engine/`. The project file
can use local game-derived source for IDE navigation when present, but that is a local
development configuration. The public compile-only references are the documented fallback
and do not contain runnable game logic.

## Build outputs are not source inputs

A normal .NET build writes to `bin/` and `obj/`; CMake writes its generated files and
artifacts under its configured build tree. These outputs are not part of project-reference
ownership. The repository's ignore rules also cover generated helper folders and local
game/save files. When the docs refer to project settings, inspect the tracked `.csproj`,
`.props`, `.sln`, CMake, or Cargo manifest rather than an artifact under a build output.

`SimsOnlineServerHostMod.Ide.csproj` sets its design-time output under `obj\ide-bin\` so it
does not collide with the legacy compiler's `bin\Release` output. This is one reason the IDE
project should not be treated as the package build command.

## A safe validation ladder

1. Run the focused custom test filter for the changed pure logic.
2. Build the consuming .NET project (and client/test-client if the public contract changed).
3. Run the full custom test executable when a bridge/session or codec contract changed.
4. Run `python scripts/dev.py lint` / focused pytest when touching Python, PowerShell, or
   Python-facing file layout.
5. Use the Windows full mod compile gate for engine-coupled mod edits when legally usable local references are available; otherwise record the compile as unverified.
6. Use the Windows CMake/native integration gate for SimLink layout/ABI changes.
7. Use a licensed local game only for a live boot or behavior that cannot be covered at the
   compile/protocol boundary.

This ladder separates cheap policy/DTO checks from platform-specific compile and
live-process checks. A successful lower step does not imply the later target was validated.

## Framework summary for toolchains

| Toolchain | Source of version/architecture requirement | Runtime use |
|---|---|---|
| .NET SDK | `global.json` requests 8.0.100 with latest feature-band roll-forward. | Controller/client/test client/protocol and named helpers. |
| Legacy mod compiler | Windows compile script and `check-mod-compile.ps1`. | Full in-game host mod compilation. |
| CMake/MSVC | `SimLink/CMakeLists.txt`; C++17 and 32-bit pointer-size assertion. | Native ASI and ring self-test. |
| Python | `pyproject.toml` targets Python 3.12; CI sets up Python 3.12. | Script/tool tests, lint, source guards, build entry points. |
| Cargo/Rust | Each manifest declares Rust 2021 and its own workspace. | Independent bridge, headless server and kernel tools. |

The `tools/` solution projects that declare .NET targets use `net8.0`: `ManagedRecovery`,
`RecompileHelper`, `ManagedMetadataExtractor`, and `ForkTool`. The ten `tools/ide/engine/`
projects are reference wrappers used for code navigation; their role is not the same as a
production binary dependency. The README in the API-reference directory, not the assembly
file names alone, defines the public redistribution description.

## Release/build credentials boundary

Conditional Sentry upload property groups are inert unless the build supplies the required
environment variables; this source tour records only that condition, never a value.
`credentials/` and `signing/` are local-only placeholders with ignored contents. Do not copy
secret values into command examples, build logs, test fixtures, or documentation. The
repository's signing README states that no release signing material is consumed by
repository code and current releases are unsigned.

An IDE or Windows build can resolve the placeholder folders differently from a CI runner,
but that does not make their ignored contents part of the application. Public docs should
explain where external configuration is needed without telling readers to check in tokens,
certificates, or private paths.
