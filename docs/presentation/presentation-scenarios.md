# Presentation verification scenarios

This page defines repeatable scenarios for evaluating what a player sees, rather than treating a passing controller test as a visual proof. The scenarios use repository concepts and test seams; any live-game result is scoped to the tested save/build/configuration. Use a disposable save and two local game instances when a scenario requires process-to-process evidence.

## Evidence levels

| Level | Evidence | Claim it supports |
|---|---|---|
| 1. DTO/codec | Protocol unit test round-trips the event/command | The wire representation encodes/decodes |
| 2. Controller | Session or `MirrorEngine` test observes targeted dispatch | Ownership, filtering and ordering policy works in the test host |
| 3. Bridge | Host bridge accepts a role-gated command | Command reached the local game mod handler |
| 4. State readback | Local snapshot/diagnostic observes resulting state | Selected local state changed as requested |
| 5. Visual comparison | Synchronized screenshots/video of server/client | The tested player-facing result looks equivalent at that moment |
| 6. Durability | Save and cold-load verify the result | The tested change survives native persistence |

A higher level does not automatically cover another feature. A correct screenshot of an animation says nothing about a save file, and a successful test-world save says nothing about a modal dialog's client UI.

## Scenario A: remote Sim movement while a player directs their own Sim

**Purpose:** verify that remote-owned positions continue to mirror while the local player's active directed route is not teleported out from under itself.

**Setup:** player A owns Sim A; player B owns Sim B; place them in the same interest lot. Capture initial aliases and a full snapshot. Enable a directed route for Sim A and observe the active-order window.

**Expected behavior:**

- A's `InteractionQueued` echo is skipped on A's client.
- B's client still receives and applies Sim A's supported remote route/presentation.
- Position mirroring strips A's Sim while A's order is active, then reasserts the final server position on completion/failure.
- Sim B remains mirrored on A's client; ownership filtering does not drop the entire mixed snapshot.

**Evidence:** `MirrorEventFilter.SelfOwnedInteractionQueuedSkips`, `ComputePositionMirrorStripSetStripsOnlyOwnedAndActive`, `FilterSnapshotSimsRemovesSelfOwnedFromSnapshotDelta`, and `OrderCompletionReassert` tests cover the pure/controller rules. A live readback and screenshot are required to establish motion quality.

## Scenario B: late join during an active animation

**Purpose:** prove that a new client gets the relevant active driver history before later live updates.

**Setup:** run one observable interaction on a target in the new player's interest scope; join a second client while the animation is active. Continue producing live SACS lines during history collection.

**Expected behavior:**

- The joiner gets a bounded current-driver history for targets it can see.
- Live lines for the joiner are held while its history is gathered and delivered afterward.
- Existing clients continue receiving live updates during the gather.
- A stale transient effect is not replayed just because it existed earlier in a ring buffer.

**Evidence:** `JoinTraceTests.AuthoritativeSession.HoldsTheJoinersStreamWhileItsHistoryGathers` covers controller ordering. Verify the client bridge logs and capture both windows to prove the visible state.

## Scenario C: server dialog reaches only the owner

**Purpose:** ensure a server-side question appears on the controlling player's client and cannot be answered by another player.

**Setup:** claim Sim A for player A, leave Sim B claimed by B or unclaimed, and trigger a supported dialog kind from an interaction running on A's Sim task.

**Expected behavior:**

- `DialogRequested` is routed only to player A's active connection.
- The client local game shows the dialog without translating its pause into a global speed change.
- Player A's answer is correlated by request id and accepted under the Sim ownership check.
- A forged/stale answer from player B is rejected; timeout falls back to default.

**Evidence:** `RemoteDialogs.EncodeDecodeAndGateAnswers` and `ClientListenerServer.SendsDialogToOwnerOnly` cover codec and target routing. A live test must verify actual dialog type rendering and server-side interaction outcome.

## Scenario D: client menu refuses stale target

**Purpose:** distinguish pre-pick menu honesty from rejection after a pick was submitted.

**Setup:** show the target and a supported menu interaction on both worlds, then remove the target or cause the authoritative test to reject it while leaving the client's menu state stale.

**Expected behavior:**

- A `MenuRefusal` record for a failed definition test greys only the matching actor/target/name on the owner client.
- A later clear restores the entry when the authoritative test passes.
- If the user already clicked and the server rejects the command, the pending command id is removed and a short notification is shown instead.
- A missing client alias does not grey an unrelated local object.

**Evidence:** `MenuRefusal.SendsARefusalAndItsClearAndReadsThemBack`, `MenuRefusal.CarriesTheTooltipInFieldSeven`, and `RejectionText` tests. Live verification requires observing the actual menu, including tooltip content.

## Scenario E: snapshot resync after a slow client

**Purpose:** confirm the slow client recovers from missed state deltas without corrupting event ordering.

**Setup:** use a client test harness or a constrained bridge dispatcher to create a valid ack baseline and delay its apply path. Continue sending snapshots and at least one non-snapshot event during the delay.

**Expected behavior:**

- Server lag policy targets the stale client rather than broadcasting a full snapshot to everyone.
- The client drain loop retains non-snapshot event order.
- Snapshot records in the same bounded drain batch coalesce latest-wins by whole Sim/object record.
- On `ForceFull`, stale queued delta backlog is discarded and the authoritative reset is applied.
- The full snapshot is acknowledged only after the dispatch path completes.

**Evidence:** `ResyncPolicy` and `MirrorCoordinator.ForceFullDrainsBacklogAndAppliesFullSnapshot` tests plus `SnapshotDelta.ForceFull.RoundTripsViaCodec`. A live test should read back final positions and record the per-player ack.

## Scenario F: inventory item on a second client

**Purpose:** check the narrow inventory membership mirror without implying general object replication.

**Setup:** choose a supported item type (book is useful), ensure its definition id and local counterpart can be resolved, and move it between an owner Sim's inventory and a supported container. Keep a separate unrelated local-only item in the client inventory.

**Expected behavior:**

- The `Iv` line has sorted, deduplicated item IDs and a parallel type list where available.
- The client adds/removes only names covered by server membership and retains unrelated local items.
- An item currently in use is not forcibly moved.
- A missing object id is paired by type only in the defined safe case; the type name alone does not instantiate arbitrary content.
- The book action uses definition identity (`bookId`) rather than assuming server and client runtime object IDs match.

**Evidence:** `InventorySet` tests for sorting/diff/type pairing and the two-mirror book chain test cover modeled behavior. Test the actual item/menu and then reload a disposable save only if testing durability.

## Scenario G: presentation replay versus interaction replay

**Purpose:** isolate which stream is responsible for a mismatch.

**Setup:** run an interaction with a visible animation and known target. Compare one mode with interaction replay enabled and one with it disabled while the SACS presentation stream remains enabled. Record server and client snapshots at matched times and capture both windows.

**Expected behavior:**

- Turning off interaction replay leaves position, alias, spawn and presentation streams available.
- The client may match an animation state while its interaction/posture object differs; that is a presentation-only match.
- Props or held items may not match if the object/slot stream lacks a corresponding record.
- `anim_state` reported by the recorder means the operation was accepted/recorded; visual comparison is separate evidence.

**Evidence:** `MirrorEngine.InteractionReplaySwitchDropsOnlyInteractionReplay`, recorder tests, and matched live captures. Report match rate by field; do not combine gameplay object state and animation appearance into one score.

## Scenario H: checkpoint after client-native activity

**Purpose:** verify the save gate honestly represents the server's canonical state.

**Setup:** use a client-native player Sim, submit a lot export, and request a checkpoint. Repeat with no export, stale export, successful position merge and explicit `AllowWithDataLossRisk` policy.

**Expected behavior:**

- Without a fresh merged export, default policy blocks and requests an export.
- A fresh accepted position export allows checkpoint within the 30-second freshness window.
- The request path is debounced for 10 seconds.
- A policy override can allow save while clearly accepting data-loss risk.
- No test result is described as deep household import success; that path remains quarantined.

**Evidence:** lot-reconciliation controller tests cover freshness, debounce, versioning and gate policy. Cold-load a disposable checkpoint for durability evidence.

## Recording results

For each scenario write down:

| Field | Record |
|---|---|
| Server/client build | Build stamp for both processes |
| Save/world | Disposable save identifier, no private save contents |
| Player/claim setup | Which player owns which Sim |
| Mode | Logging-only, local TS3 mirror, warm attach, or world transfer |
| Feature flags | Interaction replay and related options |
| Event evidence | Event name, target, tick/request id, and decision |
| Bridge evidence | Command handler result and whether it was queued/applied |
| Readback | Snapshot/diagnostic fields and timestamp |
| Visual evidence | Paired screenshots/video and camera framing |
| Result scope | Exact scenario/build/save tested; unsupported inference noted |

Do not record private household bytes, account tokens, personal file paths, or maintainer-local probes. A synthetic event fixture and minimal screenshots normally provide enough review evidence.

## Source map

- `DedicatedServer.Tests/Program.cs` — controller, filter, replay and resync tests.
- `DedicatedServer.Tests/JoinTraceTests.cs` — late-join history ordering.
- `DedicatedServer.Tests/CoordinatorTimerTests.cs` — receive/drain event order.
- `SimsOnline.Client/Program.cs` — runtime dispatch and feature switches.
- `DedicatedServer/Mirroring/PresentationStream.cs` — bounded SACS event batches.
- `DedicatedServer/Mirroring/LotReconcile.cs` — save-gate policy.
- `SimsOnlineServerHostMod/Presentation/Refusals.cs` — client refusal rendering data.

See [Client mirroring](client-mirroring.md), [Remote dialogs](remote-dialogs.md), [Menu honesty](menu-honesty.md), and [Recording and replay](recording-and-replay.md) for the corresponding feature contracts.

## Scenario evidence checklist

For every live scenario, capture enough information to replay the setup without retaining player data:

1. Use a disposable world and document whether it was loaded from server or local save.
2. Record client/server process roles and host bridge readiness before starting the action.
3. Confirm claim ownership and the active interest lot.
4. Note feature settings (interaction replay on/off, capture allowlist, resync config).
5. Record the event/command ids and simulation ticks, not full private payloads.
6. Capture controller decision and bridge result at the boundary under test.
7. Capture local state readback and paired visual evidence when relevant.
8. Repeat after reconnect to check late-join reset behavior.
9. State what the test does not prove (for example persistence or unrelated interaction classes).

Scenario reports should use plain-language conclusions such as “the owner-only controller route was verified; UI rendering was not tested” rather than a blanket “remote dialogs work.”

## Interpreting mixed results

A scenario may pass at one boundary and fail at another. For example, a remote dialog can be correctly owner-routed and accepted by `SHOW_DIALOG`, while the local game displays an unsupported kind as a fallback. An animation can match while held objects differ because object/slot records are a separate producer. A checkpoint gate can accept a fresh position export while household inventory remains outside the active merge surface.

Keep results field-specific. If only one of four tested scenarios fails, do not generalize to all gameplay. Conversely, a passing test fixture should not be presented as evidence of live visual correctness.

## What the end-to-end scenario suite cannot establish

The scenario matrix verifies representative paths, not all game interactions, object classes, save generations or UI dialogs. A successful GoHere scenario does not authorize arbitrary interaction capture. A book inventory test does not establish support for all inventory classes. One successful dialog kind does not prove every custom modal or picker can be serialized. A clean save load does not prove a separate household import is safe.

Use scenario results to prioritize missing coverage, and keep each claim attached to its exact event kind and test surface. For broader compatibility, repeat the scenario on each supported game build and save generation.

## Closing the test loop

After any behavior change, update the page table, run the focused controller tests for the changed boundary, run the docs checker for local anchors, and report any live-only behavior that remains unverified. Keep scenario output bounded to synthetic identities and minimal screenshots.
