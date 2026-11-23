# Build from source

There are separate build boundaries for managed .NET projects, the game script mod, and the optional native sidecar. A successful build validates only the selected boundary.

## Managed projects

`global.json` selects the .NET 8 SDK; `Directory.Build.props` sets `net8.0`, nullable analysis, and implicit usings for the five named stack projects. Start with a focused project, then run the test suite:

```powershell
dotnet build SimsOnline.Client/SimsOnline.Client.csproj
dotnet build DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj
dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter all
```

The test project references the dedicated server. The explicit project filename matters because `DedicatedServer.Tests` contains more than one project file. `python scripts/dev.py check` is the repository's Python-tooling check; `python scripts/dev.py test --dotnet` includes the managed build and test path.

## Game script mod

The mod uses a legacy compiler and runtime surface, not the managed projects' .NET 8 target. Build/package with PowerShell 7:

```powershell
pwsh -NoProfile -File scripts/make-mod.ps1 -SkipInstall
```

Packaging additionally requires the .NET Framework 4 compiler and s3pe assemblies listed
in [prerequisites](prerequisites.md); the .NET 2 compiler check alone is not sufficient.

`-SkipInstall` builds and packages without writing the package into a game profile.
The script's `-Reload` option is a different operation: it stops all controllers named
`SimsOnline.Client` and `SimsOnline.DedicatedServer`, but uses recorded PIDs for the TS3
game processes, then rebuilds/installs and normally starts the development stack again.
The controller stops are not restricted to this stack's tracked PIDs; coordinate with
other local runs first. `-Reload` and `-SkipInstall` are mutually exclusive. Do not
build over files held by a running game.

On Windows only, check runtime API compatibility before the next game launch:

```powershell
python scripts/check_mod_runtime.py
```

`check_mod_runtime.py` requires `WINDIR` and the Windows desktop .NET 2 compiler; it is
not the Linux/Wine build path. It compiles shell and presentation sources against the
game's runtime references, catching APIs absent from the embedded runtime. Passing
does not prove correct live behavior.

## Optional native sidecar

The SimLink native sidecar has a separate CMake/MSVC build. CI builds it on Windows x86 using `python scripts/sidecar.py build --generator "Visual Studio 18 2026"`; it is not implicitly built by `dotnet build` or by `stackctl --build`. Run this workflow only when changing the native component and its platform toolchain is installed.

## Reproducible validation

Record the source revision, target project, command, SDK/compiler version, and whether a game was involved. Keep generated binaries and local game material out of source changes. For mod changes, distinguish compile-time compatibility from a game-backed test; for a native change, distinguish managed tests from the Windows sidecar test.

## Common build failures

| Failure | Interpretation |
|---|---|
| SDK resolver or restore error | Toolchain/package-resolution layer; inspect the selected SDK and restore output first. |
| C# compile error | Source/API mismatch in the selected project. |
| `check_mod_runtime.py` reports missing API | The game runtime lacks an API available to the desktop compiler. |
| Output locked while installing | A running game may hold the assembly; stop the intended process before replacement. |
| Sidecar build not found by `stackctl` | Sidecar build and selection are explicit; compile it separately and select the supported front-door mode. |

## Related pages

[Prerequisites](prerequisites.md) | [Test strategy](test-strategy.md) | [CI](ci.md) | [Release packaging](release-packaging.md) | [Hot reload](hot-reload.md)
