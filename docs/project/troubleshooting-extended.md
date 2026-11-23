# Extended troubleshooting

Classify symptoms before changing code. A likely cause is a hypothesis, not a
diagnosis. Gather host/client evidence and reproduce with a resettable environment.

| Symptom | Likely area | First checks | Safe next action |
|---|---|---|---|
| Controller will not build | SDK/toolchain | Installed SDK and config | Follow development setup; capture error |
| Tests fail before running | Restore/dependencies | Restore output and target | Restore only after missing dependency confirmed |
| Client cannot connect | Endpoint/network | Listener, address scope, port | Keep network private; review security |
| Connects but cannot join | Session protocol | Join response/version | Compare supported protocol/build |
| Client joins but is blank | Bootstrap/interest | Snapshot and Sim mapping | Inspect host/client join logs |
| Click does not reach host | Input capture | Active actor and capture event | Trace intent client-to-host |
| Action is refused | Host validation | Claim, target, game-side test | Report refusal without retry spam |
| Action runs twice | Authority/retry | Local execution and duplicates | Check gate/idempotency |
| Wrong Sim gets input | Identity mapping | Stable ID/current alias | Rebuild map; reject stale aliases |
| Menu choice unavailable | Stale model/menu | Posture/inventory/relationship | Compare host/client inputs |
| Menu choice refused | Host/client mismatch | Host refusal and target state | Report exact context |
| Dialog on wrong player | Claim/dialog routing | Claim timeline/dialog identity | Invalidate stale prompt |
| Dialog answer ignored | Stale/invalid response | Dialog active and choice valid | Reopen prompt; capture correlation |
| Remote Sim acts locally | Authority leak | Queue, alarms, client events | Treat as high priority and isolate |
| Needs drift over time | Heartbeat/mirror cadence | Repeated values | Measure fixed interval |
| Client-only Sim appears | Service/alarm path | Identity and creation source | Reset and verify client role gates |
| Sim snaps at route end | Route correction/timing | Route and position sequence | Test local walk separately |
| Sim teleports during travel | Vehicle/travel coverage | Route type and posture timeline | Treat as partial support |
| Animation restarts after join | Stale history | Driver lifecycle/join baseline | Check live-driver reconstruction |
| Animation stays stuck | Missing release/end | Host termination/client apply | Report lifecycle records |
| Local idle overrides pose | Presenter/overlay | Local hook and incoming stream | Compare local calls and counters |
| Sim faces wrong way | Look-at/rotation owner | Host target and client writer | Isolate from route behavior |
| Held item missing | Slot/inventory mirror | Attachment events and mapping | Inspect relevant model/boundary |
| Object duplicates | Local side effect plus replay | Creation source per process | Check simulation suppression |
| Client-only world state | Alarms/services | Role-specific service controls | Preserve logs; avoid saving state |
| Join works only after restart | Stale session/process | Session IDs and controller logs | Reproduce clean lifecycle |
| Long session crashes | Runtime/resource pressure | Duration/memory/process logs | Stop safely; do not infer leak once |
| Test hangs | Blocked process/timing | Timeout and child state | Stop only identified process |
| Scenario is blocked | Prerequisite failure | Game alive and environment | Mark blocked, not pass/fail |
| Docs links fail | Relative path/anchor | Run docs checker | Fix target or heading |
| Package omits client | Artifact definition | Build/release manifest | Confirm intended scope |

## Evidence bundle

- Exact reproduction and expected/actual result.
- Project revision and relevant game/mod version, without account details.
- Player count and process roles.
- Sanitized timestamped logs.
- Stable Sim identity and sanitized save/scenario description.
- Tests passed/failed, with exact commands.
- Frequency and conditions.

Do not send proprietary game files, full saves, credentials, private addresses, or
personal paths. Related: presentation
failures,
[support](support-and-reporting-issues.md), [testing](../testing.md).

## Diagnostic workflow

1. Stop before repeating an action that may mutate a save or shared state.
2. Record the project revision, component, role, and exact action.
3. Separate connection, request acceptance, host simulation, and client presentation.
4. Check whether the issue occurs with one client or only with multiple clients.
5. Reproduce on a clean baseline before testing optional mods or custom content.
6. Confirm the selected authority mode and save-copy status.
7. Run the narrowest relevant unit or integration test.
8. Check sanitized logs around the failure, not entire log archives.
9. Record whether the issue is consistent, intermittent, or one-time.
10. Follow the relevant safety guidance before changing network settings.

## Symptom / cause / fix reference

| Symptom | Possible cause | First safe check or fix |
|---|---|---|
| Controller cannot start | Missing SDK/runtime | Confirm documented .NET prerequisites |
| Build reports missing project | Wrong working directory | Run from repository root |
| Build succeeds but test fails | Test-specific dependency or fixture | Read the test output and project setup |
| Python checker cannot import module | Optional docs dependency absent | Follow documented dependency install |
| Docs link checker reports no files | Untracked pages omitted by folder mode | Pass explicit Markdown file paths |
| MkDocs rejects navigation | Page path missing from configured nav | Check nav and exact file spelling |
| Game process is not detected | Unsupported launch or install path | Verify live setup and known game launch |
| Mod loads but no UI appears | Role/configuration mismatch | Confirm client/host role and expected entry point |
| Socket bind fails | Port already in use or permission issue | Check local listener and configured bind |
| Client cannot connect locally | Wrong host/port or listener state | Verify local endpoint before changing firewall |
| VM client cannot reach host | Isolated virtual network or route issue | Check private VM network configuration |
| Remote peer unexpectedly reaches listener | Interface/firewall exposed too broadly | Disable exposure and isolate immediately |
| Connection accepted, then closed | Invalid protocol/version or session policy | Compare client/server revisions and handshake evidence |
| Client appears under wrong player | Identity association issue | Disconnect; verify server assignment, do not trust UI |
| Request appears pending forever | Response lost or handler stalled | Avoid duplicate mutation; inspect host result first |
| Action refused | Ownership or game prerequisite not met | Read refusal and verify actor/target state |
| Action succeeds only on host | Client presentation path incomplete | Distinguish shared result from local view |
| Other client sees no update | Recipient routing or mirror gap | Check host event and target role |
| Client shows stale position | Delayed update or correction pending | Stop local assumptions and allow host correction |
| Route count differs between peers | Route mirror is partial or stale | Compare host route result and presentation counters |
| Inventory item missing in mirror | Unmapped object/item or ownership boundary | Check mapping support; do not fabricate local item |
| Object-use animation missing | Local effect not replicated | Confirm host action separately from animation |
| Dialog does not open | Client role/target mapping or unsupported dialog | Confirm request identity and supported dialog path |
| Dialog answer has no effect | Stale request ID or host-side validation | Do not assume acceptance; inspect returned outcome |
| Notification appears twice | Duplicate event or reconnect replay | Record event context and avoid reissuing action |
| Notification absent but action worked | Cosmetic presentation path incomplete | Report presentation separately from simulation |
| Menu item is disabled | Role, state, or prerequisite guard | Check feature state and current authority role |
| Menu claims success too early | Pending UI is not tied to host acceptance | Wait for authoritative response; report misleading feedback |
| Two clients overwrite state | Conflict resolution or ownership gap | Stop test; preserve save copy and reproduce minimally |
| Reconnect shows stale world | Snapshot/rejoin path incomplete | Restart controlled scenario; do not trust stale state |
| Host and client saves differ | Selected authority/persistence mode mismatch | Confirm mode and save owner before continuing |
| Crash after save | Game/runtime issue or invalid state | Preserve original backup; use disposable copy only |
| Native module fails to load | Architecture/runtime/ABI mismatch | Check target architecture and documented prerequisites |
| Native action returns no result | Boundary error not propagated or unsupported call | Inspect error result and target environment |
| Third-party mod changes behavior | Hook/content conflict | Reproduce on clean install, then isolate mod |
| Only one game build fails | Version-sensitive integration | Record exact build and narrow compatibility claim |
| Test passes but live scenario fails | Fixture does not cover game runtime | Capture minimal live scenario and report limitation |
| Live scenario passes once | Evidence is too narrow for broad support | Repeat controlled case and state exact scope |
| Logs contain private data | Logging boundary too broad | Do not share; sanitize locally and report safely |
| Issue cannot be reproduced | Missing revision or scenario detail | Record exact steps and distinguish observation from cause |

## Interpreting repeated failures

A symptom can arise at different layers. For example, "the other player did not see it" can mean the host rejected the action, the action succeeded but the event was not routed, the client could not map a game object, or presentation was intentionally local. Do not infer the layer from the visual result alone.

If the first safe check does not resolve the issue, consult the canonical [protocol](../protocol.md), [testing](../testing.md), security, or [compatibility](compatibility-matrix.md) page. For a report, follow [support and reporting](support-and-reporting-issues.md); include only sanitized evidence.

## Fast checks by subsystem

| Subsystem | First check |
|---|---|
| Build | SDK version, working directory, target project |
| Tests | Test filter, fixtures, and exact failing assertion |
| Docs | File existence, relative link, heading anchor |
| Network | Listener state and private route, without opening exposure |
| Protocol | Peer revision, message validation, response event |
| Identity | Server association and role assignment |
| Game mod | Process role, install, supported build |
| Presentation | Host outcome first, client view second |
| Persistence | Authority mode and disposable save copy |
| Native boundary | Architecture, ABI, runtime dependency |

Do not change several configuration values at once. Preserve the first failure output before retrying; later retries can hide the original cause. If a failure is security-related, stop using the affected network path and follow the private security guidance rather than posting detailed reproduction steps publicly.

## Distinguish layers before changing settings

A successful TCP connection only establishes reachability. Protocol acceptance requires a valid message; authorization requires an accepted identity and allowed action; simulation success requires the owning game process to accept the action; presentation requires delivery and client mapping; persistence requires a save path. Diagnose these stages in order, and do not claim a later stage from evidence of an earlier one.

## Closing the loop

After a workaround, verify whether it fixes the cause or only hides the symptom. Record the exact change and test once from a clean baseline when safe. If the underlying behavior remains unsupported, keep the limitation visible and update the issue report with that distinction.

If the behavior changes after a restart, report whether the restart was of the client, host, or both. Preserve the distinction between an observed recovery and a verified root cause.

A work-around that weakens a firewall, changes ownership, or suppresses an error is not an acceptable resolution for a security-sensitive failure.

| Investigation mistake | Why it misleads |
|---|---|
| Retry a non-idempotent request | May duplicate a state change |
| Change several settings together | Hides which variable mattered |
| Trust a single UI indicator | It may not reflect host authority |
| Share a full log | It may expose private data |
| Assume a passing unit test is live proof | The game runtime may not be involved |
