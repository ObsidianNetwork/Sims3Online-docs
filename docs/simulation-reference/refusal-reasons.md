# Refusal reasons and player feedback

A rejected pick can fail before the game sees it, during authoritative menu/target resolution, at queue insertion, or after the queue accepted it. Those stages produce different evidence and different player messages. The public protocol uses `command_rejected` for a request-level failure; the host mod also streams temporary menu-refusal state and reports later failures for the originating pick.

## Refusal stages

| Stage | What was refused | Where to find it | Is an interaction instance queued? |
|---|---|---|---|
| Connection gate | Command from an unjoined connection or a different player identity | `DedicatedServer/Network/ClientListenerServer.cs` → `CheckCommandAuthority` | No. |
| Session ownership | Player has not joined, or actor is claimed by someone else | `DedicatedServer/Authority/AuthoritativeSession.cs`, QueueInteraction branch | No. |
| Authority sidecar | Optional authority layer refuses a command | `ClientListenerServer.ConsultAuthoritySidecar` | No native host action when refusal is returned. |
| Pick reconstruction | Actor, target, selected object, or offered menu entry cannot be resolved | `SimsOnlineServerHostMod/ServerHostMain.cs` → `HandleQueueUserInteraction` | No. |
| Definition test | Server's game rejects the menu entry as currently unavailable | `InteractionQueue.Add` returns false; `NoteRefusal` can stream an `Rf` record | No. |
| Queue-capacity/queue-state refusal | Game queue rejects for a reason other than the interaction's definition test | Queue Add failure, but the definition test may pass | No; it is not represented as a greyed-out definition refusal. |
| Post-acceptance failure | Queue accepted a player pick, but it was in use, could not route/start, or never ran | `EVENT|PickFailed` converted to `command_rejected` for the originating command | Yes, acceptance preceded the failure callback. |
| Ordinary completion or cancel | Interaction ended normally, or player/server cancelled it | `InteractionCompleted` or normal cancellation path | Not a refusal. |

Do not label every `command_rejected` as a gameplay refusal. The connection and ownership gates are authorization failures; a definition-test refusal means the action is unavailable now; a `PickFailed` means the game had accepted the request and a later attempt failed.

## User pick round trip

A client that captures a player menu pick sends a `ClientCommand.QueueInteraction` with the `UserPick` parameter keys `ui.name`, `ui.path`, `ui.hit`, and `ui.picked`. The authoritative session checks that the player is joined and that the actor is not owned by another player. The server mod then rebuilds the menu entry against the server's own actor and target and queues it with user-directed flags.

Before queueing, the mod can reject with specific diagnostics:

| Mod-side reason | Example shape | What it means |
|---|---|---|
| Actor absent | `Sim not found: 1234` | The server cannot resolve the actor ID. |
| Target absent | `Target not found: 5678` | The target ID is not available in the server world. |
| Selected item absent | `Picked object not found: o5678` | A picker-selected object/SimDescription could not be resolved. |
| Entry absent | `No matching interaction: <definition> '<name>' (0 of that type on the target).` | The target's offered interactions do not match the forwarded type/name/path. |
| Instance creation failure | `CreateInstanceFromParameters returned null for <definition>.` | The server found a matching menu entry but could not instantiate it. |
| Queue insertion rejection | `InteractionQueue.Add refused <definition> '<name>'.` | The game queue declined insertion. The server appends an available tooltip as described below. |

For the menu lookup, the server selects from the relevant interaction list for the target: inventory, charred-object, or ordinary world interactions. It matches definition type, name, and path; when only one entry of that definition type exists, it can use that unique candidate. Thus “no matching interaction” is not the same as the definition's own test returning false.

## Four post-acceptance failure codes

`PickWatch` in `SimsOnlineServerHostMod/ServerHostMain.cs` attaches start/failure callbacks to a user pick. `PickFailureReason` classifies the failure in this exact precedence order:

| Code | Predicate | Player-facing text |
|---|---|---|
| `ObjectInUse` | The instance was in use, or the actor has `ExitReason.ObjectInUse` | `Can't do "Sit": someone else is using it.` |
| `RouteFailed` | Actor has `ExitReason.RouteFailed` | `Can't do "Go Here": can't get there.` |
| `FailedToStart` | Actor has `ExitReason.FailedToStart` | `Can't do "Chat" right now.` unless a definition tooltip is available. |
| `NotRun` | No earlier failure code and the start callback never ran | `Can't do "Chat" right now.` unless a definition tooltip is available. |

If the interaction started and its callback later reports no one of those categories, `PickFailureReason` returns null: that is not sent as a pick refusal. A cancelled interaction is also explicitly ignored by `PickWatch.OnFail`; a user cancelling their own order should not produce a refusal toast.

`ObjectInUse` and `RouteFailed` already explain the common cause, so they are sent without a second definition test. `FailedToStart` and `NotRun` do not tell the player why; for those two only, the mod re-runs the definition's test and appends its tooltip if the test still fails with one.

## Tooltip propagation

The game supplies a `GreyedOutTooltipCallback` to explain why an offered definition is unavailable. The queue's `Add` result preserves pass/fail, not that tooltip. On a failed Add, `PickRefusalTooltip` obtains the instance parameters, runs the same definition test again, and invokes the callback when it exists. If the test now passes, there is no tooltip; that indicates the queue may have rejected for a different queue-state reason. Exceptions are swallowed and counted rather than replacing the original refusal.

`SimsOnlineServerHostMod/UserIntentWire.cs` flattens CR/LF to spaces and appends non-empty text with the marker ` | tooltip: `. A new client finds that suffix with `TooltipOf`; `DedicatedServer/Mirroring/RejectionText.cs` presents it and truncates displayed reason text to 120 characters. Older servers without the marker continue through the existing generic fallback.

Examples covered by the current test harness:

```text
InteractionQueue.Add refused Sims3.Gameplay.InteractionsShared.Sit+Definition 'Sit'.
=> Can't do "Sit" right now.

InteractionQueue.Add refused Sims3.Gameplay.Objects.Bookshelf+Read+Definition 'Read'. | tooltip: Nothing to read on this shelf.
=> Can't do "Read": Nothing to read on this shelf.

NotRun | tooltip: Someone is using this.
=> Can't do "Nap": Someone is using this.
```

Tooltip strings are generated in the server game's language; the client does not translate them. An empty, missing, or throwing tooltip callback falls back to the reason-code text.

## Menu refusal memory (`Rf`)

When a pick is unavailable because its definition test fails, `NoteRefusal` keeps a server-side watch and records an `Rf` line in `SacsRecorder`. The record identifies actor, target, definition type, menu name, refused/clear flag, and the tooltip as hex-encoded UTF-8. A clear event has an empty tooltip. The client presentation module resolves server actor/target IDs to local IDs and holds refused entry names so the menu can grey a pick that the local copy would otherwise allow.

```text
Rf|7787|18000|<hex-definition-type>|<hex-name>|1|<hex-tooltip>
```

The `Rf` path has several distinct facts worth preserving:

- It updates the menu's temporary refusal memory; it is not itself a `command_rejected` response.
- Refusals are retested; when the condition clears, or actor/target disappears, a clear is sent.
- `MenuRefusal` sends the tooltip in its wire field, but `SimsOnlineServerHostMod/Presentation/Refusals.cs` currently calls the parser overload that ignores that field and returns the marker `"1"` to the presentation hook. The greyed menu therefore still uses the hook's generic text, not the actual server tooltip. The tooltip is shown for request rejection toasts, but not yet used by this menu-display hook.
- An old-format line without the tooltip field still carries the refused/clear state. Invalid tooltip hex loses the tooltip, not the refusal bit.

## Real counts and their meaning

The source exposes bounded-state sizes and live counters, not a checked-in population report. The repository does not contain a trustworthy session-wide histogram of which gameplay reasons occurred how often; do not present test fixture counts as live refusal frequency. The exact numeric limits and counters currently available are:

| Counter / limit | Current value or format | Interpretation |
|---|---:|---|
| Held server refusal watches | Capacity **32** | When full, the oldest held item is cleared before a new one is stored. |
| Sibling menu entries checked per refusal | Limit **12** | Other failing entries of the same definition type can be held too; the submitted entry is separate. |
| Refusal-watch lifetime | **240 sim minutes** (four sim hours) | Expiry clears stale refusal state even if the corresponding condition did not produce a clear event. |
| Server diagnostics | `held/sent:<n>/cleared:<n>/skipped:<n>/retests:<n>/errors:<n>` | `held` is current list size; the rest are cumulative counters in `RefusalReport()`. `skipped` includes a candidate whose test passes even though queue insertion failed. |
| Client menu diagnostics | `held/+<n>/-<n>/expired:<n>/missing:<n>/greyed:<n>` | Current entries, refusal/clear events, expired/missing local identities, and the hook's greyed count. A missing hook counter is shown as `-`. |
| Pick failure vocabulary | **4** codes | Exactly `ObjectInUse`, `RouteFailed`, `FailedToStart`, and `NotRun` in the current `PickFailureReason` classifier. |
| Toast text bound | **120 characters** | `RejectionText.MaxReasonLength`; truncation adds `...`. |

The counters are useful to distinguish no refusals from refusal-stream trouble, but they are not a durable analytics database and do not break failures down by reason code. To report frequencies, collect runtime records from a defined session and count each code at the appropriate stage.

## Worked traces

### The definition is not offered

The server receives `ui.name=Sit`, resolves the actor and target, but cannot find a matching offered entry. It returns a reason beginning `No matching interaction` (with the definition and candidate count). `NoteRefusal` is called with no pair: it can hold the named refusal and let the next poll determine whether the entry appears and passes, but it has no definition callback from which to obtain a tooltip. The player sees the generic “right now” phrasing because the missing entry has no test result.

### The definition test rejects insertion

The server finds a matching pair and `InteractionQueue.Add` returns false. `PickRefusalTooltip` reruns the definition test. If it returns a tooltip, the `command_rejected.reason` includes the suffix ` | tooltip: <text>` and the client renders `Can't do "<pick>": <text>`. If the test passes on recheck, no tooltip is attached; a queue-state rejection is not mislabeled as a definition refusal. The `Rf` watch can also grey the matching menu entry and up to 12 failing siblings.

### The action was accepted, then the route failed

The mod arms `PickWatch` only after `InteractionQueue.Add` succeeds. If the game later calls failure and the actor has `RouteFailed`, the mod emits `EVENT|PickFailed|<command-id>|<actor-id>|<interaction-id>|<hex-reason>`. The controller decodes it as `ServerEvent.CommandRejected` and `ClientListenerServer` routes it only to the player who submitted that command. The displayed text is “can't get there,” not a queue-insertion error.

## Operational edges

- A definition may change between the original queue test and the follow-up tooltip test. A missing tooltip does not prove that the original rejection had no cause; it means no current tooltip was recoverable.
- Some game tests fail without supplying a `GreyedOutTooltipCallback`. In that case the existing generic “right now” message is intentional.
- A reused server reason string may include a `tooltip:` marker. `RejectionText` checks the marker before mapping known codes or generic “refused” text.
- The client may lack an actor or target instance when replaying an `Rf` line. The menu refusal entry is not installed and `missing` increments; a later snapshot is not itself a guarantee that the old refusal is replayed.
- Menu refusal capacity/expiry and the toast length cap are independent. A long tooltip may be truncated for a toast while the menu record remains held until retest or expiry.
- The internal refusal stream can be disabled with `stream.refusals=0`; this does not change request-level `command_rejected` feedback.

## Verification

Focused cases are in `DedicatedServer.Tests/Program.cs` and `DedicatedServer.Tests/OutboundListenerTests.cs`:

- `RejectionText.IsShortAndReadable`
- `RejectionText.ShowsTheServersTooltip`
- `RejectionText.ShowsTheTooltipOfADroppedPick`
- `UserIntentWire.PickFailedTakesTheTooltipOnlyForTheSilentCodes`
- `MenuRefusal.SendsARefusalAndItsClearAndReadsThemBack`
- `MenuRefusal.RefusesMalformedLines`
- `MenuRefusal.CarriesTheTooltipInFieldSeven`
- `ClientListenerServer.SendsAFailedPickToItsPlayerOnly`
- `scripts/rejection_relay_probe_test.py` checks that named refusal replies are correlated to requests and that silent closes/timeouts do not pass as successful refusal delivery.

Key implementation paths: `DedicatedServer/Network/ClientListenerServer.cs`, `DedicatedServer/Authority/AuthoritativeSession.cs`, `DedicatedServer/HostBridge/BridgeLineCodec.cs`, `DedicatedServer/Mirroring/RejectionText.cs`, `SimsOnlineServerHostMod/ServerHostMain.cs`, `SimsOnlineServerHostMod/UserIntentWire.cs`, `SimsOnlineServerHostMod/MenuRefusal.cs`, and `SimsOnlineServerHostMod/Presentation/Refusals.cs`.

## Related pages

[Interactions and lifecycle](interactions-and-lifecycle.md), [autonomy](autonomy.md), [protocol](../protocol.md), and [testing](../testing.md).
