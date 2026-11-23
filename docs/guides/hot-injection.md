# Hot injection for local diagnostics

The bridge supports loading a diagnostic assembly into a running game. Loaded assemblies
remain resident in the game's .NET 2 AppDomain until the game restarts; distinct uploads
accumulate even if each probe finishes quickly. This is a maintainer debugging technique, not a gameplay API or an external tester feature. The repository does not ship a general collection of personal probes; any maintainer-local probe must be treated as untracked local material.

## Compile before loading

Use `scripts/probe_check.py` to list and validate supported script templates without opening a live game:

```powershell
python scripts/probe_check.py --list
python scripts/probe_check.py --render-only
python scripts/probe_check.py
```

The default check renders templates and compiles them with the same legacy C# 2 toolchain used for a probe. A render-only check validates placeholders but does not compile. `--force` bypasses the repository's build-quiet guard, so avoid it during another measurement/build operation.

For an explicitly authored local diagnostic, `scripts/make-probe.ps1` compiles a C# source file as a library; `scripts/bridgectl.py` provides the bridge probe operation. The probe contract is a public type with a static `Run()` method returning a string. The compiler targets x86 and the game's own reference surface. Users must supply their own legally acquired reference assemblies where required; the repository does not authorize redistribution.

## Runtime constraints

The in-game runtime is not desktop .NET 8. Keep diagnostic code within the supported legacy language/API subset, do not block the simulation thread, and avoid changing persistent world state. The probe executes inside the game process, so a bug can hang or destabilize that game. Prefer read-only observations and a disposable save.

Probe results are text returned through the bridge. A structured report can use a first-line
set of semicolon-separated `key=value` fields (for example, `ok=1;count=2`) and subsequent
detail lines; `bridgectl.py` splits recognized report headers on semicolons when rendering
JSON. Preserve raw output when the shape is unrecognized.

## What injection does not prove

Successful compilation proves only that the source fits the selected compiler references. Successful upload proves transfer, not that the probe executed safely or produced a correct observation. Compare its result with a known source-side state and repeat without instrumentation if timing matters.

## Related pages

[Live probes](live-probes.md) | [Hot reload](hot-reload.md) | [Debugging](debugging.md) | [Prerequisites](prerequisites.md)
