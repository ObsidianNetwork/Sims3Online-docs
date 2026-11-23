# `simctl.py` reference

`simctl.py` is a command-line client for the dedicated server's session protocol. It is the right tool to exercise public player/session behavior; it is not interchangeable with `bridgectl.py`, whose local admin interface bypasses those policy checks.

## Connection defaults and safety

The parser defaults to endpoint `127.0.0.1:34019`, JSON wire mode, and a 120-second connect/response timeout. `--player` defaults to player 1 for JSON/admin requests. `--admin-endpoint` selects a local admin listener for eligible mutating commands; reads and name lookups continue to use `--endpoint`. Keep that endpoint loopback-only.

`--wire line` opts into the line protocol and its public join handshake. For public line commands, an explicit `--player` is required; joining a player already connected elsewhere can supersede that connection. Start with `python scripts/simctl.py --help` and never point a test command at an external or unowned listener.

## Commands

| Command | Arguments / effect |
|---|---|
| `sims [FILTER]` | List current Sims; optional name filter. |
| `claim SIM...` / `release SIM...` | Claim or release one or more names/IDs for the selected player. Names use substring matching. |
| `set-active SIM` | Select the active server Sim. |
| `cancel SIM INTERACTION_ID` | Cancel a queued/running interaction by ID. |
| `role server\|client` | Set the player role. |
| `save SAVE_NAME` | Request a server save; use only with an operator-approved disposable profile. |
| `speed pause\|normal\|double\|triple\|ultra` | Change world speed. |
| `worldtime [get\|set]` | Read current clock by default, or set it with `--day 0..6`, `--hours 0..23`, and optional `--minutes 0..59`. |
| `gohere SIM X Y Z` | Queue an authoritative terrain movement request. |
| `interact SIM KEY` | Queue an interaction key; optional `--target` and `--x/--y/--z` fields provide target/position. |
| `synth-click SIM X Y Z` | Inject a client-side player click via the client admin port; this is not a public player action. |
| `status` | Read session/tick/world/save/speed/claim summary. |
| `watch [--filter FILTER]` | Print events until interrupted. |

`--refresh` is available only on selected state/name-reading commands; it bypasses the local Sim snapshot cache. Inspect that command's help if cache freshness matters.

## Protocol and command behavior

JSON mode is the default and does not perform the public line-protocol join handshake. Line mode does join; the explicit player requirement exists because a new connection can replace that player's existing one. The admin endpoint is a separate route for eligible mutations, not a way to make public reads private: `status`, `sims`, and name resolution continue through the public endpoint. Keep a test client's public identity and local admin access distinct.

Name operands are resolved against the current Sim snapshot. Substring matches are convenient for unique names, but ambiguous matches are rejected for single-Sim operations; use an object ID or a more specific name. `--refresh` is available on selected state/name operations to bypass the cached snapshot. `interact` accepts a target ID or a complete x/y/z position. The handler only includes coordinates when all three are present, so pass all three or none to avoid silently omitting a partial position.

`worldtime` without an action reads a recent clock broadcast with a short bounded wait. For `worldtime set`, supply `--day` as well as `--hours` (and optionally `--minutes`); omitting the day is refused rather than implicitly rewinding the save to day zero. `gohere` queues a server-authoritative terrain action. `synth-click` instead sends an injected click to the client bridge, so it tests a client input path and should not be described as a public player request.

The offline `LineConnectionTests` in `scripts/simctl_test.py` cover JSON's no-join behavior, public line joining, admin no-join routing, snapshot chunk/event handling, and refusal to start line mode without an explicit player. These tests exercise protocol handling without requiring a live server.

## Read before mutating

```powershell
python scripts/simctl.py --endpoint 127.0.0.1:34019 status
python scripts/simctl.py --endpoint 127.0.0.1:34019 sims
```

For an authorized local test, use one mutation at a time, then query `status` or `sims` to confirm the postcondition. Release a claim when finished. Do not retry a timed-out save, claim, or time change until a fresh read confirms whether the first request took effect.

A successful transport exchange is not necessarily an accepted operation. Preserve the server's refusal/result and distinguish it from a connection failure. If public behavior is rejected, do not repeat it through the admin listener; fix the public path or report the rejection as the result.

## Failure interpretation

| Symptom | Check |
|---|---|
| Connection refused | Confirm local server readiness and the endpoint/codec. |
| Player/join conflict | Confirm the selected identity is not already connected; line mode can replace it. |
| Command refused | Check ownership, role, and server preconditions. |
| Timeout | Query state before retrying; execution may have completed asynchronously. |
| Command accepted but no visible change | Wait for the corresponding event and compare server state with client presentation. |

## Related pages

[Protocol](../protocol.md) · [Run clients](run-client.md) · [Server options](server-options.md) · [Bridge tools](bridge-tools.md) · [Scenario CLI](tool-reference-scenarios.md)
