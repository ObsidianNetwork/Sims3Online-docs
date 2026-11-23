# Presentation ownership

The host simulates shared gameplay. The client presents selected host-observed results. This ownership split prevents a mirrored client from starting the same gameplay interaction independently and mutating its local world as if it were authoritative.

## Ownership by boundary

| Boundary | Owner | Public path | Meaning |
|---|---|---|---|
| Player choice | Client input capture, then server validation | `ClientCaptureFilter`, `EventIntentCaptured` | A request, not an outcome |
| Gameplay interaction | Authoritative host game | Host bridge and game interaction queue | Decides whether the operation starts, completes, or fails |
| Current selected state | Host snapshot producer/session | `SimulationSnapshot`, `FullSnapshot`, `SnapshotDelta` | Data for client-side state display/mirroring |
| SACS animation operation | Host recorder | `ISacsProxy` decorator and `SacsRecorder.RingLine` | Recorded presentation call, not gameplay logic |
| Animation replay | Client-role game instance | `SACS_REPLAY` / `ApplyReplayBatch` | Applies selected presentation operations on local copies |
| Durable save | Host game | `SAVE_GAME` and `WorldSave` transfer | Loadable native world, separate from animation replay |

An event labeled as an intent must not be described as a completed interaction. A replay batch must not be described as the client rerunning an interaction. The authoritative interaction events (`InteractionStarted`, `InteractionCompleted`, `InteractionFailed`) and selected visual `SimAnimation` events are distinct event types.

## One simulator, multiple presenters

`ServerEvent.SimAnimation` contains SACS recorder lines. The client bridge's `HandleSacsReplay` enforces `sCurrentRole == "client"`; a server-role process rejects the command. The recorder and replay handler therefore form a presentation path, not a second authority path.

This split is not a promise of full visual or simulation equivalence. A model snapshot omits detailed animation-machine history; a presentation stream omits many gameplay fields. The two inputs complement one another but do not jointly encode every hidden engine state.

## What the stream intentionally excludes

`SacsRecorder` classifies facial idle animation as cosmetic and does not stream or snapshot that layer. Each local game can play its own camera-dependent cosmetic idle. The recorder also maintains driver/actor data for presentation; it does not use this as a replacement for game persistence. These exclusions avoid competing local writers or replaying process-local effects as if they were shared gameplay.

For posture, transform, object/slot, route, or effects discrepancies, determine which producer owns that field and whether the client has a corresponding local object. Equal `PostureType` values alone do not prove equal animation, slot placement, or driver state.

## How to diagnose a mismatch

1. Identify the shared Sim/object by a stable mapping. Runtime `ObjectId` values can differ between independently loaded client games.
2. Compare host and client model fields at comparable ticks.
3. Check whether a recorder line was emitted for the relevant driver and whether the target selection included the Sim.
4. Check whether controller batching preserved order and whether the client-role replay handler accepted the batch.
5. Check whether the local game has a writer or cosmetic system competing with the replay.
6. If the question is persistence, save and cold-load separately; a successful animation replay says nothing about durable state.

These steps are a diagnostic sequence, not a checklist implying every layer is instrumented identically. Use counters or test traces that exist for the subsystem under investigation.

## Failure modes

| Symptom | Likely ownership boundary to inspect |
|---|---|
| Input is visible locally but host does not act | Capture allowlist, transport, or host authorization |
| Host action succeeds but client pose is absent | Recorder producer, target set, event routing, replay handler, local mapping |
| Client pose differs while model fields agree | Presentation record coverage, game build/resource, local writer |
| Client changes local gameplay state independently | Role gate or an unguarded local gameplay path |
| Reload restores state but animation differs | Save/load semantics versus transient presentation state |

## Source and verification

Current code references: `SimsOnlineServerHostMod/SacsRecorder.cs`, `SimsOnlineServerHostMod/ServerHostMain.cs`, `DedicatedServer/Mirroring/PresentationStream.cs`, `SimsOnline.Protocol/ServerEvent.cs`, and `SimsOnline.Protocol/SimulationSnapshot.cs`. `DedicatedServer.Tests/JoinTraceTests.cs` covers event ordering and join-history routing. Visual correctness still requires a live scenario with an explicit game build and save.

Related: [Presentation stream](presentation-stream.md), [Capture input pipeline](capture-input-pipeline.md), [Protocol](../protocol.md), [Troubleshooting](../troubleshooting.md).

## Authority is per field and operation

The phrase â€œserver authoritativeâ€� does not mean every client-visible field is continuously identical. It means the host decides shared outcomes and the client does not commit its own competing outcome. The client can still predict or locally animate a direct action while waiting for the server, but that local action must be treated as intent/presentation and reconciled with the authoritative result.

| Data/action | Authority rule | Client behavior |
|---|---|---|
| Supported player pick | Server validates claim, target and definition | Capture intent; do not claim success until host accepts/starts it |
| Interaction outcome | Server game executes | Mirror selected status/presentation; client does not create canonical result |
| Remote Sim position | Server snapshot | Apply correction unless active-order policy temporarily strips it |
| Owned Sim route | Server accepts command; owner may have local UI route | Suppress duplicate echo and reconcile final position |
| Animation calls | Host recorder observes server SACS | Client applies selected calls; no gameplay authority implied |
| Dialog choice | Player owner answers; server task interprets | Show local UI and return answer under claim gate |
| Save state | Host game writes native save | Client receives save files separately if requested |

## Not every display is replayed

The presentation stream excludes or localizes some cosmetic work, including selected facial idle behavior that can be driven by local camera/game context. A missing cosmetic idle does not imply the authoritative interaction failed. On the other hand, a missing route or held object may indicate an omitted recorder module or target object mapping.

Local UI notifications and modal dialogs also have different semantics. A notification can be one-way; a modal question has a response that affects server gameplay. The latter must be owner-routed and correlated. See [Remote dialogs](remote-dialogs.md).

## Convergence is bounded

Snapshots and presentation events provide selected convergence, not complete deterministic equality. Two game instances can differ in:

- local object IDs and aliases;
- loaded asset or game build behavior;
- autonomous background activity;
- inventory contents and process-local runtime objects;
- animation driver handles and transient effects;
- hidden state not represented by the public snapshot.

Use the source-of-truth boundary to decide which discrepancy matters. A position mismatch after an authoritative snapshot may be a mirror apply issue; a different local idle animation may be expected; a save mismatch is outside the presentation stream entirely.

## Detecting competing writers

Two writers can fight even when each one is locally correct. Examples include a position delta interrupting a local GoHere route, local autonomy advancing a remote puppet, or a stale animation history overwriting a newer state. `ActiveOrderTracker`, autonomy suppression, per-Sim snapshot filtering and ordered SACS replay address separate cases.

When a symptom is intermittent, record event order and local simulator state around the transition. A single final snapshot may hide whether the client briefly applied the right state before another local writer changed it.

## What an evidence statement should say

Use terms that identify the verified boundary:

- â€œThe server accepted the GoHere intentâ€� â€” command/session evidence.
- â€œThe client bridge applied the position commandâ€� â€” bridge dispatch evidence.
- â€œThe client snapshot readback was within the expected toleranceâ€� â€” state evidence.
- â€œThe captured client image showed the Sim seatedâ€� â€” visual evidence.
- â€œThe save cold-loaded with the relationship intactâ€� â€” durability evidence.

Avoid â€œthe Sim is synchronizedâ€� without naming field, time window, client mode and evidence level. Related data boundaries are in [Snapshot fields](../data-and-persistence/snapshot-fields.md) and [Lot reconciliation](../data-and-persistence/lot-reconciliation.md).

## State transition matrix

| Transition | Authority | What the other process learns |
|---|---|---|
| Player clicks supported choice | Player produces intent | Server receives actor, target and supported parameters |
| Server refuses choice | Server decides no | Owner gets refusal/feedback; no shared action starts |
| Server starts interaction | Server game decides yes | Clients receive accepted/status/presentation events as supported |
| Host animation call | Server game emits appearance operation | Recorder event may be replayed on clients |
| Client receives snapshot | Server reports current selected fields | Client mirror applies selected fields; does not decide outcome |
| Client saves local world | Local process writes its save | Does not modify canonical server save without explicit authority path |

A remote client's locally simulated autonomous activity is not a second shared outcome. It can be hidden/suppressed or corrected depending on the current mode. If local action becomes visible, inspect role gates and autonomy policy before adding another event stream.

## Review questions for new fields

Before adding a presentation field, determine whether it is durable state, instantaneous state, a one-shot effect, a player intent, or an authoritative outcome. Each category has different replay/recovery behavior.

| Category | Typical transport | Retry strategy |
|---|---|---|
| Durable state | Native save / explicit typed persistent DTO | Verify save/reload; avoid implicit event replay |
| Current state | Snapshot/delta | Latest-state coalescing; full resync |
| One-shot effect | Ordered event/record | Do not replay stale effects to late joiners |
| Player intent | Command with idempotency/correlation id | Validate and return accepted/refused result |
| Outcome/status | Server event | Preserve order and scope to affected players |

If a field changes both interaction logic and visual appearance, document which stream carries each part. Do not send a visual state write as a substitute for an authoritative gameplay decision.

## Mode differences

The same client can operate in different mirror modes. A client-native owned Sim may route locally while the server accepts the command and reconciles the final state. A presentation-only client can instead display host positions and SACS operations without rerunning the interaction. The config `SIMSONLINE_MIRROR_INTERACTIONS=0` affects only interaction replay; it does not switch off the network mirror wholesale.

Document the mode in bug reports. A correctness expectation for an owned Sim on a client-native lot may be opposite the expectation for a remote puppet on an observation-only client.

## Observable versus unobservable state

The controller can observe a protocol event or a bridge response, but many internal engine effects are not exposed as direct readback in a release build. Do not turn a successful command response into a claim about a hidden engine state. Use the closest independent measurement available: a client snapshot for position/presence, a diagnostic for applied command counters, or a visual capture for appearance.

The same recorder state field can be reported by both a host that naturally animated the Sim and a client that accepted replay. It is useful for comparing a requested state but is not an independent visual oracle. State exactly where the observation came from.

## Review examples

- **Correct statement:** “The server recorder emitted a state line and client `SACS_REPLAY` accepted its batch; the client image was not inspected.”
- **Incorrect statement:** “The animation is synchronized” based only on the accepted bridge response.
- **Correct statement:** “The client `SimulationSnapshot` reports the Sim within 0.5 m at rest.”
- **Incorrect statement:** “The Sim's interaction is synchronized” based only on matching position.
- **Correct statement:** “The local save cold-loaded after a checkpoint.”
- **Incorrect statement:** “The save is correct” based only on matching live snapshot fields.

## Escalation boundaries

A live observation that contradicts controller unit tests should be captured with build and mode details, then traced to the first boundary where the expected record changes. Avoid fixing a symptom in the client presentation module if the server never produced the corresponding state. Avoid changing server authority based only on a visual mismatch caused by a local runtime asset.

## Observable versus unobservable state

The controller can observe a protocol event or a bridge response, but many internal engine effects are not exposed as direct readback in a release build. Do not turn a successful command response into a claim about a hidden engine state. Use the closest independent measurement available: a client snapshot for position/presence, a diagnostic for applied command counters, or a visual capture for appearance.

The same recorder state field can be reported by both a host that naturally animated the Sim and a client that accepted replay. It is useful for comparing a requested state but is not an independent visual oracle. State exactly where the observation came from.

## Review examples

- **Correct statement:** “The server recorder emitted a state line and client `SACS_REPLAY` accepted its batch; the client image was not inspected.”
- **Incorrect statement:** “The animation is synchronized” based only on the accepted bridge response.
- **Correct statement:** “The client `SimulationSnapshot` reports the Sim within 0.5 m at rest.”
- **Incorrect statement:** “The Sim's interaction is synchronized” based only on matching position.
- **Correct statement:** “The local save cold-loaded after a checkpoint.”
- **Incorrect statement:** “The save is correct” based only on matching live snapshot fields.

## Escalation boundaries

A live observation that contradicts controller unit tests should be captured with build and mode details, then traced to the first boundary where the expected record changes. Avoid fixing a symptom in the client presentation module if the server never produced the corresponding state. Avoid changing server authority based only on a visual mismatch caused by a local runtime asset.

## Authority transition examples

### Pick and rejection

The owner clicks a local menu item. The client captures it as a supported intent and removes the local duplicate queue entry. The server validates current ownership and target state. If the server test fails, the host refusal stream can grey the item in future menus; if the request has already been submitted, a targeted rejection explains why it did not run. The two signals should not be conflated.

### Route and convergence

The server accepts a GoHere command and emits the authoritative route/presentation. The owner client suppresses its own echoed command to avoid a second queue entry. While its local directed route runs, position mirroring can be withheld. On completion, a final position reassert reconciles any drift. A remote observer still sees the host's route/position because that observer does not own the actor.

### Dialog choice

A modal opened by the server's interaction waits on the server task. The owner's client displays the question and returns an answer; the server interprets it. The client doesn't make the outcome authoritative simply by showing the dialog. If no owner is connected, timeout/default policy resumes the server task.

### Checkpoint

The host game writes the native save. If client-native lot changes exist, the server can require a recent supported export before save. A client-side snapshot, animation replay or inventory set is not evidence that the canonical save contains those changes.

## Ownership review for implementation changes

Before adding a new event or bridge handler, record:

| Question | Required answer |
|---|---|
| Who produces the value? | One process/module and the lifecycle point |
| Who is allowed to change it? | Authority/claim rule |
| Who receives it? | Broadcast, interest scope or owner-only target |
| Is it intent or outcome? | Command vs authoritative event |
| Can it be retried? | Idempotency and correlation behavior |
| Is it current state or transient? | Snapshot/replay and late-join policy |
| How is it reset? | Full snapshot, clear, timeout or new save |
| What proves success? | Test, bridge, readback, visual, or cold load |

If no single owner exists, the feature needs a conflict policy before implementation. If the receiver cannot resolve identity, sending more fields does not solve the ownership ambiguity.

## Ownership event ordering example

A `ClaimSim` request changes which player is authorized to direct an actor. The session emits claim/authority state; client authority gates update; subsequent intents and targeted dialogs use the new claim table. If a dialog is in flight during a claim transfer, the answer must be checked against the policy for the pending request, not blindly trusted because the sender previously owned the Sim.

Likewise, an interaction completion may release a directed-order position strip. The client should process completion and final-position reassert in order so the snapshot corrects after the route, not in the middle of it.

## Avoiding ownership ambiguity

When an operation can be initiated by more than one player, define requester identity separately from actor ownership. The owner of the Sim may control its actions, while another player may have initiated a social interaction target or object use. The targeted result should go to the right requester/owner according to the event's semantics, rather than broadcasting sensitive UI or refusal text.

## Ownership assertions for reviews

When reviewing a new feature, reject ambiguous wording such as “syncs the Sim” unless the page names which side owns the decision and what concrete data crosses. Useful review questions include:

- Is this message a command or a server event?
- Does it target one claimant or all clients in an interest scope?
- Does the owner see their own echo or is it skipped?
- Does a client locally predict the result, and if so how is failure handled?
- Is state replayed to a late joiner or only broadcast on change?
- Is the operation durable, and what cold-load evidence exists?

A precise answer to these questions avoids having separate modules silently compete to own the same field.

## Ownership during partial failure

If a snapshot batch applies positions to several Sims and one local object lookup fails, the server remains the source of truth for all Sims. The client should report the failed target and keep processing independent rows. It should not promote its partially applied local state as canonical or write it back without an explicit authoritative command.

A claim transfer, connection lease expiry or host-role change can invalidate a previously valid intent. Authorization must be evaluated at the time the command is processed. Presentation history for a newly owned Sim can be resent, but transient UI requests should be checked against their pending lifecycle.

## Authority during recovery

A full snapshot repairs the client's selected mirror state; it does not transfer a native save. A server checkpoint captures the host's current world; it does not automatically repair an out-of-date client. If the two disagree, choose the recovery path based on which process owns the intended outcome rather than selecting whichever local value is newest.

## Consistency requirement

Every user-visible result should have a named owner and a return path. If a local prediction is shown before a host decision, document how it is corrected or withdrawn on rejection. If no rollback exists, prefer a pending indication over wording that implies completion.

A decision that has no owner should be treated as an unresolved design boundary, not a reason to let both processes write concurrently.
