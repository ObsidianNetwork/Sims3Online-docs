# Test strategy

Use the cheapest test that exercises the changed boundary, then advance toward a live game only when necessary. The repository has separate Python tooling, managed .NET logic, Windows-native, and game-backed validation; report them as separate evidence rather than calling one a full integration test.

## Local entrypoints

```powershell
python scripts/dev.py doctor
python scripts/dev.py lint
python scripts/dev.py test
python scripts/dev.py test --dotnet
python scripts/dev.py check
```

`dev.py test` runs the Python suite (pytest when installed, otherwise a bounded `unittest` fallback), script self-tests, the stackctl self-test, the transport-latency report self-test, Node tests when Node is installed, and a `--help` check for CLI scripts. `--all` includes Python tests normally excluded from CI; `--dotnet` adds the .NET builds and reflection test runner. `--force` permits that optional .NET portion when the shared build window refuses it.

To focus the .NET runner, use the project filename explicitly:

```powershell
dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter PublicLineCodec
```

The runner accepts `--filter all` for the full set; test class names correspond to files such as `PublicLineCodecTests.cs`, `ProtocolHardeningTests.cs`, `JoinTraceTests.cs`, `ClientReceiveDrainTests.cs`, and `SimLinkTests.cs`.

## Validation layers

| Layer | Example evidence | Limitation |
|---|---|---|
| Python/unit tooling | `python scripts/dev.py test` | Does not run the full .NET integration suite unless requested. |
| Managed controller/protocol | .NET build and test runner | Does not prove game runtime or UI behavior. |
| Rust authority sidecar | Ubuntu `sidecar-tests` job and NUnit suite | Exercises the controller adapter against a loopback daemon. |
| Native x86 SimLink | Windows CI build and SimLink-filtered tests | Platform-specific; Linux skips are not equivalent to a Windows pass. |
| Mod compile compatibility | `python scripts/check_mod_runtime.py` | Static compile check, not an in-game run. |
| Scenario/live game | A named scenario and its recorded verdict/evidence | Requires a local game installation and disposable test state. |

## Scenario evidence

The scenario runner is stateful and game-backed. Start with `python scripts/scenarios.py list` to inspect names, and use `python scripts/scenarios.py health` before interpreting a run. A scenario can be blocked by unavailable game processes; a blocked result is not a pass. Saved runs can be inspected with `history`, `status`, `explain`, and `rescore` without replaying the game.

## Report precisely

A useful test report names the exact command, pass/fail/blocked outcome, target layer, platform, and whether it used a real game. Include the first failing assertion and sanitized logs. Do not turn skipped platform cases into passes or infer live behavior from compilation.

## Related pages

[CI](ci.md) | [Scenario catalog](scenario-catalog.md) | [Build from source](build-source.md) | [Debugging](debugging.md)
