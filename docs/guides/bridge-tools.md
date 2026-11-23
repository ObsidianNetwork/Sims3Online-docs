# Local bridge diagnostics

`bridgectl.py` is an operator-side client for the game's local bridge-admin listener. Unlike `simctl.py`, it can send arbitrary bridge command tokens directly to a controller; it bypasses normal player/session authorization. Keep it on the machine that owns the game process and use a disposable world for any state-changing operation.

## Safe entry points

```powershell
python scripts/bridgectl.py --help
python scripts/bridge-health.py --help
python scripts/bridgectl.py commands
```

The first two commands only display CLI help. `commands` connects to the configured admin endpoint and lists the command names the running controller recognizes; its result can differ from another build. Do not treat a name in that list as read-only without checking its handler.

`bridgectl.py` accepts `--endpoint` (defaults to the server bridge endpoint), `--timeout` (seconds; default is 15 on the server), and `--json`. Its remaining tokens form a raw bridge request. The `hexenc` and `hexdec` helper verbs encode/decode text-valued wire fields. Other named forms include `capture`, `state`, `state-compare`, `reload`, `probe`, and `send-corpus`; their options are defined in `scripts/bridgectl.py` and the top-level help. In particular, `probe` compiles/uploads and executes code in a running game, while `reload` changes the active handler assembly.

A probe source must be a maintainer-local file with the expected public `Run()` entry point; probes are not shipped with the public repository. Do not add a local probe path, source, output, or game-derived result to public docs or issues.

## Health versus feature verification

`bridge-health.py` sends `PING` and `DIAG_STATE` requests. By default it checks the server bridge; `--client` adds the client bridge, `--server-endpoint` and `--client-endpoint` select endpoints, `--timeout` bounds a probe, and `--json` emits a machine-readable result. The documented exit result is 0 for healthy and 1 for an unresponsive, wedged, or not-in-world bridge. A healthy local bridge does not prove that the public listener accepts a player or that a client rendered the latest state.

```powershell
python scripts/bridge-health.py --json
python scripts/bridge-health.py --client --json
```

These contact a local listener; do not run them against an endpoint you do not own. For public-session behavior, use `simctl.py` or a client and keep admin operations out of the acceptance path.

## Interpreting bridge health

The health check combines a responsive `PING` with `DIAG_STATE` evidence. A reachable socket alone is insufficient: the bridge must also report an in-world state and positive tick/dispatch progress to be considered healthy. The optional client probe applies the same check to the client bridge. This is a local controller/game-pump signal, not a substitute for checking that the public listener accepted a player or that the client presented the expected state.

Use a small, read-only check before any targeted command:

```powershell
python scripts/bridge-health.py --json
python scripts/bridgectl.py --timeout 15 DIAG_STATE
```

The direct request above is still an admin operation, even though `DIAG_STATE` is observational. Do not copy returned identities or world data into a public report. In the line protocol, long-running commands can be given a per-command timeout prefix (for example, snapshot requests are marked heavy in the command metadata); honor that operation's documented bounds rather than retrying a timed-out request blindly.

## Classify the effect first

| Operation | Examples | Evidence to collect |
|---|---|---|
| Listing/health | `commands`, bridge health | Build/role and response status |
| Read-only inspection | `DIAG_STATE`, `state`, `state-compare` | Selected role, IDs, capture time |
| Local evidence capture | `capture` | Short duration and sanitized summary |
| Live code execution | `probe`, `probe --watch` | Source identity, exact role, bounded stop condition |
| Mutation | Arbitrary command or handler reload | Before/after state and explicit permission |

A timeout does not guarantee that a command stopped inside the game pump. Query state before retrying. Do not expose the admin listener through a LAN, public bind, or tunnel, and do not forward raw captures without reviewing names, IDs, local paths, and world data.

## Implementation references

`DedicatedServer/HostBridge/BridgeLineCodec.cs` defines command encoding and known-command metadata; `DedicatedServer/Network/BridgeAdminListener.cs` handles the local admin boundary. `scripts/bridgectl.py` owns the command-line token handling, and `scripts/bridge-health.py` owns the readiness check. Review the matching code revision before interpreting a command not covered by the current help.

## Related pages

[simctl](simctl-reference.md) · [Debugging](debugging.md) · [Live probes](live-probes.md) · [Logging](logging-diagnostics.md) · [SimLink sidecar](tool-reference-sidecar.md)
