# Client input capture pipeline

The client mod observes the active Sim's interaction queue and forwards only supported direct player commands as intent. It does not send every queue entry and does not run the authoritative action on the client as a fallback.

## Current capture boundary

`PollClientInputInterception` runs only when `sCurrentRole` is `"client"`. When `Sim.ActiveActor` changes, it unsubscribes from the prior actor's `InteractionQueue.QueueChanged`, subscribes to the new actor's queue, and stores the existing `InteractionList` as the baseline. This prevents pre-existing queue entries from being mistaken for fresh player input.

`OnActiveActorQueueChanged` uses `sIsProcessingQueueChange` as a re-entrancy guard. It compares current entries against `sClientQueueKnownInteractions`; only newly observed entries with `InteractionPriorityLevel.UserDirected` reach the definition filter. Priority is not sufficient proof of a human click: the game can queue reactions or other system work at that priority.

## Fail-closed allowlist

`SimsOnlineServerHostMod/ClientCaptureFilter.cs` is the pure decision function shared with `DedicatedServer.Tests`. It returns true only for the friendly key `Terrain.GoHere` or a reflected full name containing `Terrain+GoHere`. Null, empty, and unrelated keys are rejected. The server applies the corresponding allowlist when accepting the interaction, so adding an input requires keeping client and server checks aligned.

| Candidate | Forwarded? | Reason |
|---|---:|---|
| `Terrain.GoHere` | Yes | Supported terrain movement request |
| `Sims3.Gameplay.Core.Terrain+GoHere` | Yes | Reflected nested-type spelling accepted by the predicate |
| `ReactionInteraction+Definition` | No | System-generated reaction is not a captured player command |
| `BuffExhausted+PassOut` | No | User-directed queue priority does not make it a user choice |
| Null or empty key | No | Cannot safely reconstruct the operation |
| Any other definition | No | Not in the current supported player-command set |

The allowlist is intentionally narrow. It is not a generic interaction forwarding feature.

## Interception and event payload

For an accepted entry, `InterceptClientInteraction` removes the interaction from the local queue so the client does not execute the same action independently. It reads actor and target object IDs and the interaction definition's full type name. It attempts to read `Hit.mPoint`; if available, that point is included as the user target position. A reflection failure is counted through the mod's swallowed-error path; it does not synthesize a reliable hit position.

The mod emits an `EventIntentCaptured` bridge event with actor ID, target ID, interaction key, a position-present flag, and coordinates. The controller/session path treats this as a request; host validation and execution determine the outcome. A captured intent is not a success acknowledgement.

## Example classification trace

This trace is explanatory, not an exact logger contract:

```text
role=client; active actor queue changed
new entry priority=UserDirected; key=Sims3.Gameplay.Core.Terrain+GoHere
ClientCaptureFilter.ShouldForwardCapturedInteraction -> true
remove local queue entry; emit EventIntentCaptured(actor, target, key, position?)
server validates player and actor claim; host decides outcome
```

A new engine-pushed reaction instead gets `false`, stays in the queue, and emits the filter-drop diagnostic. It should not be misreported as a transport loss or host refusal.

## Queue edge cases

- Changing active actor detaches the prior queue listener and baselines the new actor's current list.
- An already-present interaction is not new input.
- A callback that reenters while queue processing is active returns without duplicating capture.
- Unsupported `UserDirected` entries remain in the local queue; the handler does not remove them as if they were player input.
- If `Hit.mPoint` is unavailable, the payload's position-present flag is false; consumers must not treat a fallback target position as equivalent to a click hit point.
- When the role is not client, polling exits without attaching the client capture path.
- `EventCaptureFilterDropped` is a diagnostic for an unsupported candidate, not an error result from the server.

## Tests and extension rule

`DedicatedServer.Tests/Program.cs` includes `ClientCaptureFilter.ForwardsGoHere`, `ClientCaptureFilter.DropsReactionInteraction`, and `ClientCaptureFilter.DropsOtherEngineUserDirectedAndNullEmpty`. These verify the shared predicate, not a real mouse click, callback timing, or end-to-end authorization.

To add a supported player operation, update the client's `ClientCaptureFilter`, the corresponding host-side accepted-command gate, and tests for both the accepted and rejected cases. Confirm the wire representation preserves the target/hit context required to reconstruct the user's choice. Do not broaden capture based only on priority or menu label.

Related: [Presentation ownership](presentation-ownership.md), [Presentation stream](presentation-stream.md), [Protocol](../protocol.md), [Testing](../testing.md).

## Intent validation after capture

`EventIntentCaptured` is an untrusted request boundary even when it came from the local game mod. The server validates the player connection, claimed actor, supported interaction key and target before executing the command. Client capture and server acceptance must remain in sync; the client allowlist is not authorization.

The client removes a captured GoHere interaction from its local queue to prevent a duplicate local route. If the server refuses it, the player receives a rejection and the local world should not silently resume an unsupported action. When adding another key, define whether it is safe to remove locally before server acceptance or whether a local prediction/rollback is necessary.

## Captured target semantics

The capture payload can include `Hit.mPoint` when the game exposes it. That coordinate represents the clicked point, not necessarily the target object's origin. If reflection cannot read the point, the position-present flag is false. Consumers must not treat a default `(0,0,0)` as a real click location.

Target object identity and hit point serve different purposes: the target ID identifies which object the player clicked, while the point can choose a spot on a larger object or terrain surface. The server may have to resolve them independently in its own world.

## Scope and trust tests

For each new intent type, add negative as well as positive tests:

| Question | Required negative test |
|---|---|
| Is the actor owned by this player? | Another player's actor is rejected |
| Is the definition supported? | Unknown key is rejected |
| Is the target present? | Missing target is rejected with player-readable feedback |
| Is the request fresh? | Duplicate/stale command id does not execute twice |
| Is the payload complete? | Missing required position/parameters is rejected |
| Is the client role valid? | Server-only path cannot be invoked from wrong role |

Tests for `ClientCaptureFilter` prove only the local predicate. Session authorization and host execution require separate tests. See [Menu honesty and refusals](menu-honesty.md) for what the player sees after validation.

## Capture and server acceptance are separate

`ClientCaptureFilter` is a local allowlist that minimizes unintended forwarding. The session/server still performs authorization and validation. A malicious or modified client can bypass the local filter and send a protocol command directly; the authoritative session must not rely on the client mod to enforce ownership or supported-command policy.

For GoHere, the session verifies the player owns the actor, checks the supported key, validates target/world context, and forwards a host command. The host can reject because the actor is absent, target is missing, engine test fails, or route cannot start. These are post-capture outcomes and should flow back as a command result or interaction status.

## Duplicate and callback behavior

Queue-change callbacks may fire for several mutations in one update. `sIsProcessingQueueChange` prevents reentrant capture, while `sClientQueueKnownInteractions` provides a baseline of already present queue entries. When switching active actor, the mod unsubscribes from the old queue and subscribes to the new one; otherwise an old actor callback could be attributed to the newly selected Sim.

The command ID is generated at the controller boundary and is used to correlate acceptance/rejection. Repeated queue callbacks should not create multiple independent commands for the same newly observed interaction. If adding a capture path, test callback repetition, active actor changes and queue removal ordering.

## Why only GoHere is accepted

GoHere is a compact intent that can be represented with actor, target and optional hit point. Many interactions depend on local inventory, object state, menu-picked row identities, hidden parameters, routing context or runtime objects that cannot be reconstructed on the server from a generic interaction key. A wide allowlist would make the client execute a local action while the server executes a potentially different one.

To expand the allowlist, define a dedicated semantic DTO with stable identifiers and explicit parameters, or prove that the full queue entry can be reconstructed safely in the server's own world. Do not serialize arbitrary reflection output or engine object references.

## Capture test matrix

`ClientCaptureFilter` unit tests should continue to cover the friendly key and reflected nested type spelling, plus reactions, user-directed system entries, null/empty keys and unrelated definitions. End-to-end tests should additionally cover:

- claimed versus unclaimed actor;
- connected versus stale player lease;
- valid versus missing target;
- hit point present versus absent;
- server queue accept versus test refusal;
- route start, completion and failure;
- client local queue removal and no duplicate execution;
- player-readable response for a rejected pick.

A predicate unit test alone cannot prove a real click was captured. A host test alone cannot prove the local queue did not execute a duplicate.

## Capture invariants by transition

| Transition | Required check | Failure classification |
|---|---|---|
| Active actor changes | Old queue handler detached; new queue baselined | Stale actor event or pre-existing action forwarded |
| Queue callback fires | Reentrancy guard honored | Duplicate callback/capture |
| New queue entry found | Entry absent from known baseline | Existing order mistaken for player input |
| Priority inspected | Only a candidate signal | System work mistaken for human choice |
| Definition inspected | Exact allowlist match | Unsupported interaction sent to server |
| Hit point read | Presence flag reflects actual read | Default coordinate misrepresented as click |
| Local entry removed | Only accepted capture removed | Unsupported action lost locally |
| Event sent | Actor/target/key retained | Server cannot reconstruct intent |
| Server validates | Claim and target rechecked | Client self-authorization |
| Host executes | Engine result recorded | Request misreported as success |

## Logging and privacy

Capture diagnostics should identify event kind, command id, definition key and failure category. Do not log full arbitrary reflection objects or player-created text. Coordinates are useful for route troubleshooting but should be treated as gameplay data and kept to the minimum needed in logs.

A dropped candidate is not necessarily an error. Reactions and unsupported entries are expected to remain local; counters should distinguish filtered entries from parser exceptions and server rejections.

## End-to-end trace template

Use this compact trace to diagnose a missing captured input without exposing full game state:

| Step | Evidence to collect |
|---|---|
| 1. Active actor | Whether `Sim.ActiveActor` matched the expected local Sim |
| 2. Queue baseline | Whether the entry was newly observed after subscription |
| 3. Candidate | Priority and interaction key only |
| 4. Filter | `ShouldForwardCapturedInteraction` decision and reason |
| 5. Removal | Whether the accepted entry was removed locally |
| 6. Event | Command/event id, actor/target ids and position-present flag |
| 7. Session | Claim validation and target existence result |
| 8. Host | Queue/route accepted or engine rejection category |
| 9. Client response | Completion/failure or final position reassert |

Never include arbitrary object `ToString()` dumps or raw reflected state in this report. Use the event id and narrow decision reason to correlate logs.

## Failure taxonomy

| Symptom | Likely boundary |
|---|---|
| Click does nothing locally or remotely | Local menu, active actor, or unsupported key |
| Local actor performs action and server also repeats it | Local queue removal/echo filter failure |
| No request event, drop counter increases | Intentional filter rejection; inspect key |
| Request exists but session rejects | Claim, target, lease or stale command validation |
| Host receives request but no route | Engine test/queue refusal or route failure |
| Server route works but observer absent | Snapshot/interaction/presentation stream scope |
| Action succeeds but player sees no feedback | Targeted event/notification path |

## Extension review policy

Keep the allowlist small and explicit. A new accepted key needs a protocol representation that captures the actual choice, server-side authorization, a host-side execution path, a typed result, and a negative test showing unsupported keys are still rejected. Avoid accepting interactions based on display strings, because names can be localized or duplicated.

If an operation needs a selected picker row, encode a stable domain key for the row and resolve it in the server's world. Do not send a client `ObjectId` and assume it identifies the same object on the host. If it needs a point, define coordinate space and whether the point is optional.

## Ownership and one-simulator modes

In a one-simulator authority model, the server runs shared gameplay and the client only forwards selected user choices. In client-native lot modes, the client can execute local behavior for its owned Sim, but the server still validates shared effects and later reconciliation is bounded. The input capture doc describes a supported request path, not a blanket claim that all local interactions are replicated.

## Capture queue replacement checklist

The queue listener subscribes to the active actor's queue. When the actor changes, confirm the prior handler is removed before the new baseline is stored. A race during the transition can otherwise forward an entry from the prior actor while labeling it with the new active actor.

Test a selection change while a callback is pending, an empty queue, and an actor with an already-populated queue. Baseline initialization should not forward old work. Queue comparison should use the engine's interaction identity/key semantics, not just list index, since queue entries can reorder or be replaced.

## Host response semantics

The public command acceptance event is not the final gameplay outcome. The engine can accept a command and later report interaction started, completed, failed, or canceled. UI feedback should distinguish immediate validation rejection from later route/interaction failure. The captured pick name can be used to make the explanation readable, but the result must remain correlated to the command id.

This distinction is especially important when client queue removal is optimistic: the local player sees the request disappear from their queue before the host result arrives. A late rejection must restore a coherent UI state or explain the refusal; it must not leave the player believing the action completed.

## Recommended capture regression scenarios

1. **No previous input:** baseline a queue containing an existing GoHere, then append a different GoHere; only the new entry should capture.
2. **Actor switch:** subscribe to actor A, switch to B, then mutate A's queue; no event should be attributed to B.
3. **Unsupported user-directed item:** candidate fails allowlist and remains in the local queue.
4. **Missing hit point:** event marks position absent instead of using zero coordinates.
5. **Server claim change:** command from a prior owner is rejected after ownership moves.
6. **Host late failure:** accepted command later fails route and produces player-readable feedback.

Each scenario should assert capture predicate, local queue effect, command identity and server result separately. A single “forwarded” assertion does not protect the end-to-end semantics.

## Command and result contract

The capture event is a request, and the server's command result is only immediate validation status. The subsequent interaction lifecycle events report whether the action started, completed or failed. A local UI should not turn command acceptance into an “action completed” notification.

For supported movement, the interaction key and target id let the server reconstruct a request; the hit point adds optional spatial context. For a future action with extra parameters, define a dedicated typed command rather than overloading the position field or serializing an engine object.

## Click-to-result timeline

```text
player click -> local queue observation -> allowlist decision
 -> local duplicate removal -> public command -> session ownership/target check
 -> host interaction queue -> interaction started/completed/failed
 -> owner-facing result and observer presentation
```

A gap between any adjacent steps has a distinct remedy. If no candidate was observed, inspect the active actor and queue subscription. If filtered, inspect the allowlist. If command rejected, inspect session authorization. If host accepted but no completion, inspect engine interaction and route status. If the owner saw no result, inspect targeted feedback.

## Client/server parity checks

The client and server gates should agree on supported intent keys, but they serve different purposes: client filtering prevents accidental forwarding; server validation protects authority. A parity test should feed the same accepted/rejected key set to both pure filters where possible. If one side uses a different representation (friendly name versus reflected type), document and test the normalization.

## Untrusted coordinates

An optional hit point arrives from the client process and must be treated as untrusted input. Validate finite numeric values and plausible world bounds before using it. A missing point should remain missing; the server can apply a documented fallback such as target-center routing, but that is a server policy and not equivalent to the clicked point.

## Minimal observability record

For one captured command, retain only the command correlation id, key, actor/target IDs, filter result and server response category. Avoid raw interaction parameter dumps unless the specific key contract requires them. This is enough to locate a boundary without retaining unrelated player state.

For live testing, use a synthetic click target in a disposable lot and repeat with an unsupported key. The accepted case should reach host validation; the negative case should remain local and should not produce a server-side action.

## Negative-case assertion

For every supported input, pair one unsupported or malformed input and verify it does not mutate the host. This guards the boundary against future allowlist expansion by accident.

## Supported-input proof

Capture correctness requires both a positive GoHere case and a negative reaction case; the server must independently enforce the same authority and key constraints.

Document the unsupported-key case alongside each new supported capture so the allowlist stays explicit.

Keep unsupported candidates in the local queue unless an explicit interaction-specific policy says otherwise.

Keep one explicit negative case beside every supported capture key.
