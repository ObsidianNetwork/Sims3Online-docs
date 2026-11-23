# Live probes

A live probe is a bounded observation performed against a running local game through the bridge. Prefer existing diagnostics and scenario evidence first. A probe should answer one question, avoid mutating the world, and stop promptly if the game is unhealthy.

## Repository-supported path

The tracked `tools/probes/SocialSessionProbe.cs` is an unexpanded template, not a
compile-ready probe. `scripts/probe_check.py` renders recognized Python-module templates
offline, but compiles an explicitly supplied `.cs` file as-is. Do not pass an unexpanded
template to that compiler path. `scripts/make-probe.ps1` compiles a concrete local C#
library; `scripts/bridgectl.py` sends the bridge operation. Maintainer-local source may
be used for a temporary investigation, but it is not shipped or promised by this repository.

```powershell
python scripts/probe_check.py --list
python scripts/probe_check.py sim_act --render-only
python scripts/bridgectl.py commands
```

The first command lists available templates. The second fills the `sim_act` templates
with sample values and checks placeholders without compiling or opening the game.
To compile those rendered templates, omit `--render-only` after supplying the legacy
toolchain and permitted local reference assemblies. The final command lists the bridge
command vocabulary; it does not mutate the game.

## Probe contract and report shape

A source can expose a public type with `public static string Run()`. The compiler targets the game's legacy x86 runtime. `bridgectl` recognizes a report envelope containing fields such as `loaded`, `asm`, `type`, and `run`; the probe's output can carry a semicolon-separated `key=value` header followed by detail lines. Keep field names stable enough for reproducible comparisons, and emit a failure rather than a misleading partial success.

## Safe observation budget

A probe runs in the game process and can add work to the simulation pump. Avoid waits, unbounded object traversal, repeated reflection scans, and writes to persistent world state. Capture only the minimum fields needed. Check elapsed time and repeat the observation without the probe before attributing a timing-sensitive symptom to the game.

## Evidence and limits

A successful compile does not prove runtime safety. A bridge response does not prove the observation is complete or that the visible UI matches. Record the source revision, target process role, exact probe name, response, and whether a trace was enabled. Do not share raw world snapshots or sensitive local paths.

## Related pages

[Hot injection](hot-injection.md) | [Debugging](debugging.md) | [Logging and diagnostics](logging-diagnostics.md) | [Performance profiling](performance-profiling.md)
