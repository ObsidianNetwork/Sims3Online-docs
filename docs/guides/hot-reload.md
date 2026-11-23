# Hot reload

The repository has two distinct no-full-game-restart loops: swapping the client presentation module and reloading handler logic through the local bridge. They have different effects and should not be conflated with rebuilding the main script-mod shell.

## Reload client presentation code

`scripts/presentation_reload.py` compiles `SimsOnlineServerHostMod/Presentation/*.cs` into a uniquely named assembly, uploads it through the client bridge, then requests a presentation-module load. The unique assembly name matters because the game's .NET 2 runtime cannot unload an already loaded assembly by name.

```powershell
python scripts/presentation_reload.py
python scripts/presentation_reload.py --full
python scripts/presentation_reload.py --watch
```

The default reload asks the module to keep the shell's broader presentation state. `--full` also resets shell presentation state and warm-rejoins so the stream can rebuild it. `--watch` repeats the default reload after presentation source changes; Ctrl+C stops the watcher. `--endpoint` overrides the client bridge endpoint.

This path requires a built shell, a running client game, a reachable local bridge, and the legacy compiler/runtime reference setup. The helper keeps only a few recent build outputs locally; each assembly already loaded in the running game remains resident until that game restarts.

## Reload other code

The script mod's main shell is not replaced by the presentation reload. A shell or engine change needs a full mod build/install and a restart of the game processes. `scripts/make-mod.ps1 -Reload` performs that more disruptive sequence; `-NoAutoUp` suppresses its automatic stack restart. `scripts/bridgectl.py` also exposes a handler-reload command for the separate handler assembly. Verify the current command implementation before using it, and do not assume it applies to shell code.

## Choose the right loop

| Change | Appropriate path |
|---|---|
| `Presentation` module implementation | `presentation_reload.py` |
| Presentation constants in `Tunables` | Live tuning tooling, no assembly rebuild if the setting supports it. |
| Handler assembly | Bridge handler reload operation. |
| Shell, bridge protocol, or engine-facing code | Rebuild/install mod and restart game. |

## Failure modes

A compile failure is distinct from upload failure, bridge rejection, or a load report that lacks the expected module identity. Check each phase's output. If the bridge is not responding, do not keep retrying upload. On Windows, run `python scripts/check_mod_runtime.py` for C# runtime API compatibility before the next game launch. That checker requires `WINDIR` and desktop .NET 2 `csc.exe`; it does not support Linux/Wine.

## Related pages

[Hot injection](hot-injection.md) | [Reattach workflow](reattach-workflow.md) | [Build from source](build-source.md) | [Logging and diagnostics](logging-diagnostics.md)
