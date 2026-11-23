# dnSpy and managed assembly inspection

The repository includes a local PowerShell launcher for interactive inspection of managed assemblies. Use it only with assemblies you are authorized to inspect. A static decompiler view is useful for signatures and IL navigation; it is not a supported method for redistributing game code or altering a shared installation.

## Launcher interface

`scripts/dnspy.ps1` supports `-Status`, `-Install`, `-Engine`, `-Mod`, `-Attach`, `-TargetPid <pid>`, `-Dll <name-or-path>`, and `-Extra <argument>`. The install helper `scripts/install-dnspy.ps1` supports `-Force`. Its current parameters can be read from those script files; these PowerShell scripts are not Python argparse CLIs.

```powershell
pwsh -NoProfile -File scripts/dnspy.ps1 -Status
pwsh -NoProfile -File scripts/dnspy.ps1 -Engine
pwsh -NoProfile -File scripts/dnspy.ps1 -Dll <assembly-name.dll>
```

`-Status` reports the local installation and process discovery. `-Engine` opens the engine assemblies found in the user's local assembly folder; `-Dll` opens a selected assembly from that folder (or an explicit existing path). `-Mod` opens the built project mod together with local engine assemblies and therefore requires a successful mod build. `-Attach` starts the GUI with attach guidance; it does not itself establish that the embedded game runtime can be debugged. `-TargetPid` disambiguates a process when more than one candidate exists. `-Extra` passes additional arguments to the GUI.

The installer downloads a pinned dnSpyEx distribution and checks its SHA-256 before extraction. `-Force` replaces the local installation. Inspect the installer source and upstream license/provenance before using it; a hash check establishes file identity against the pin, not publisher identity. Generated tool files and local manifests are not project outputs to commit.

## Static versus live evidence

| Question | Appropriate first step | Limitation |
|---|---|---|
| Where is a project-owned call made? | Read checked-in source and tests | Does not establish which game assembly loaded |
| What signature exists in a local engine build? | Inspect that authorized local assembly | Applies only to the inspected version |
| What callers/callees are indexed? | `scripts/ts3src.py` | Static index can miss dynamic dispatch and callbacks |
| What does the game report now? | Supported bridge diagnostics | Does not expose every internal method path |
| Can a method be stepped in the live game? | Treat attach as optional and runtime-specific | TS3 is a 32-bit native process with embedded Mono; attach may be unavailable |

Prefer a type/member signature and a concise behavior summary over copied decompiler text. Verify important conclusions against the project call site, a test, or a bounded runtime observation. Do not publish decompiled bodies, game assets, full symbol inventories, binary files, or debugger databases.

## Local input and licensing boundary

The project does not package engine assemblies for this workflow. Users supply their own legally acquired Sims 3 assemblies or stubs. Compile-only reference stubs are not runtime implementations and their redistribution rights are unverified; do not instruct readers to redistribute them. Keep locally supplied assemblies and generated exports out of commits, test artifacts, and issue attachments.

## Failure modes

- **No assembly is found:** local inputs are absent; this does not imply the type is absent from every game build.
- **Static open works but attach fails:** the embedded runtime/architecture may not be supported by the debugger.
- **A member is missing:** check the inspected build and assembly version before inferring an API change.
- **Decompiler output looks unlike source:** names and control flow may be reconstructed; check metadata/IL and project-owned call sites.
- **Installer checksum differs:** stop; do not extract or bypass the pin.

## Related pages

[Managed analysis tools](tool-reference-managed-analysis.md) · [Live probes](live-probes.md) · [Build from source](build-source.md)
## Why not a runtime patcher

MonoPatcher is an external method-replacement project, not a dependency of the supported SimsOnline build, server, tester kit, or normal development workflow. This guide makes no installation recommendation. Runtime replacement sits below the managed mod/bridge boundary and can be coupled to an exact game build, runtime, architecture, and loader placement; a failure may be a native process crash rather than a controller-visible error.

For observation, prefer checked-in source, `scripts/ts3src.py`, or the supported bridge tools. For a project-owned behavior change, use the maintained managed mod/controller path and its tests. A future evaluation would require a separately approved disposable installation, verified external provenance, version/architecture identity, demonstrated rollback, and a server/client test plan. Do not bundle third-party binaries, bypass integrity/access controls, or make a method patcher a contributor prerequisite.

## Related pages

[Managed analysis tools](tool-reference-managed-analysis.md) · [Hot injection](hot-injection.md) · [Release packaging](release-packaging.md)
