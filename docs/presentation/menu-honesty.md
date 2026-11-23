# Menu honesty and refusal feedback

A pie menu is built by the local game using local copies of Sims, objects, inventory and interaction state. The authoritative server may evaluate the same choice against newer state and refuse it. A truthful multiplayer menu should therefore distinguish three outcomes: the choice is available and accepted, the server rejected the choice before queueing, or the queued interaction later failed. A local green choice is not a promise that the server accepted it.

This page documents the implemented refusal display and its boundaries; it does not claim that every client-side menu test is a replica of the server's test.

## Two user-visible refusal paths

### Server says the choice is unavailable before queueing

`MenuRefusal` records server refusals from the authoritative pie-menu test. It sends a compact line that identifies actor, target, definition type and entry name, plus an optional tooltip. The client presentation module maps server actor/target IDs to local objects and lets the presentation hook grey the matching entry. As state changes, the server sends a clear line when the test passes or a relevant object disappears.

### Server accepts the pick, then it cannot run

A forwarded choice may pass the initial test but fail later (for example target missing, object in use, or route failure). The controller sends a targeted `CommandRejected`/failure to the player who submitted it. `SimsOnline.Client.Program` removes the pending command mapping and calls `RejectionText.ForPick` to show a short notification. This is not a menu grey state: the player already picked the item, so feedback explains why it did not happen.

| Stage | Feedback | State |
|---|---|---|
| Menu test says unavailable | Greyed menu item, optional server tooltip | Choice remains visible but disabled |
| Pick command rejected | Short notification to the requester | Pending pick is cleared |
| Interaction started then failed | Targeted explanation from failure reason | Already-submitted order ended |
| No local mirror host | Best-effort notification cannot be shown | Server remains authoritative |

## Refusal wire shape

`SimsOnlineServerHostMod/MenuRefusal.cs` defines kind `Rf`. A ring line begins with sequence and kind, then contains actor ID, target ID, hex UTF-8 definition type, hex UTF-8 entry name, and a refused flag; a later optional field carries the server game's tooltip:

```text
sequence|Rf|actorId|targetId|hexDefinitionType|hexEntryName|1|hexTooltip
sequence|Rf|actorId|targetId|hexDefinitionType|hexEntryName|0|
```

The `0` line clears a previous refusal. The old form without a tooltip field still parses. `MenuRefusal.Parse` rejects malformed field count, wrong kind, invalid numeric IDs, zero IDs, bad hex, empty definition type, or a refused flag other than `0`/`1`. A malformed tooltip does not invalidate the grey state; it simply omits the explanation.

Example (illustrative, not a live Sim ID):

```text
105|Rf|7787|9001|53696d73332e47616d65706c61792e416374697669746965732e536974|536974|1|43616e277420736974207468657265
```

Hex encoding protects pipe-delimited fields from embedded delimiters and newlines. The tooltip text is flattened before encoding so the bridge line remains single-line.

## Server menu state is keyed carefully

`MenuRefusal.Key(actor, target, definitionType)` identifies one interaction definition on one actor-target pair. The displayed entry additionally includes the menu entry name. On the client, `Presentation/Refusals.cs` maps the IDs through `PresentationApi.ReplayObject`; it only applies a refusal when both local actor and target resolve. Missing aliases are counted and do not create an unscoped grey entry.

The presentation hook queries in two stages:

1. Ask whether any refusal exists for this actor/target/definition type.
2. If yes, ask whether the specific entry name is refused.

This prevents broad menu rewriting when the server has no refusal for that definition. Entries expire after 240 simulation minutes if no clear arrives, bounding stale state after disconnects or missed events. The producer's clear is still preferred; expiration is a backstop, not the normal synchronization mechanism.

## Feedback text policy

`DedicatedServer/Mirroring/RejectionText.cs` translates internal server reasons into a concise player message. Its `MaxReasonLength` is 120 characters. The priority is:

1. Prefer the server game's own tooltip extracted by `UserIntentWire.TooltipOf`.
2. Translate common failure identifiers (`ObjectInUse`, `RouteFailed`, `FailedToStart`, `NotRun`) to readable messages.
3. Translate claimed-owner conflict and missing-target language to player-meaningful text.
4. Use a short generic message for empty or implementation-oriented refusal text.
5. Truncate an otherwise useful reason to 120 characters plus ellipsis.

Examples from `RejectionText.ForPick`:

```text
Can't do "Sit": someone else is using it.
Can't do "Read": it isn't there any more.
Can't do "Go Here": that Sim belongs to another player.
Can't do "Eat": Can't reach that object.
```

The final example depends on the tooltip content; this function is not a localization catalog and does not guarantee translated game strings. Do not expose stack traces, private object dumps, or full internal exception strings in player-facing text.

## Why menu tests can disagree

A client and server each run the game's menu tests against their own process state. Some tests read state the current mirror does not carry, such as whether a specific newspaper instance has been read or whether a particular object part is currently in use. The local menu may therefore show an item that the server declines. A refusal stream makes the disagreement visible and actionable, but it does not make the underlying replicas equal.

Other causes include:

- The target object exists only in one process or no longer resolves through its alias.
- The client inventory contains an item the server has already consumed or removed.
- An object is in use, but the client has not received that interaction/presentation update yet.
- The menu label collides with another entry under the same actor/target/type key.
- The server's refusal clear was lost and the 240-minute expiry has not elapsed.
- The server accepted the choice, but later routing/interaction execution failed.

Different causes should retain different signals. A test failure is not proof of network loss; a late command rejection is not proof that the menu was greyed.

## Client state counters

`Refusals.Report()` exposes current count and lifetime counters: refused, cleared, expired, missing mapping, and hook-reported greyed entries. These are diagnostic counts, not a durable audit record. `Lookup` runs inside the menu loop and catches exceptions so a bad refusal entry does not crash the game's menu; in that case the game falls back to its own test.

| Counter | Use in diagnosis |
|---|---|
| `entries` | Number of retained actor-target-entry refusals |
| `refused` | Number of refused messages processed |
| `cleared` | Number of explicit clear messages applied |
| `expired` | Entries evicted by simulation-time TTL |
| `missing` | Actor/target alias not resolved locally |
| `greyed` | Counter exposed by the UI hook, when available |

A rise in `refused` with no rise in `greyed` suggests a hook lookup/display issue. A rise in `missing` suggests identity/scope trouble before menu rendering. High `expired` with a low `cleared` count points toward delivery or producer-clear issues.

## Edge cases and compatibility

- A definition refusal without tooltip still greys the item.
- A tooltip parse failure must not un-grey an otherwise valid refused item.
- A repeated refusal updates its timestamp but should not double-count the definition's retained-entry count.
- A repeated clear for an already-removed entry has no effect.
- Client state reset clears both maps and reinitializes the hook binding.
- A stale actor or target ID must not grey a different object after ID reuse; local replay mapping and active scope must be current.
- A server refusal only affects the client that owns the actor/choice; it must not leak another player's refusal state.
- `CommandRejected` removes the pick from the forwarded-command map before showing text, preventing a later echo from matching the same pick.
- A notification failure is logged and swallowed; it must not block processing the receive loop.

## Tests and implementation map

`DedicatedServer.Tests/Program.cs` pins:

- `MenuRefusal.SendsARefusalAndItsClearAndReadsThemBack`
- `MenuRefusal.RefusesMalformedLines`
- `MenuRefusal.CarriesTheTooltipInFieldSeven`
- `RejectionText.IsShortAndReadable`
- `RejectionText.ShowsTheServersTooltip`
- `RejectionText.ShowsTheTooltipOfADroppedPick`

Current implementation:

- `SimsOnlineServerHostMod/MenuRefusal.cs` — wire formatting, compatibility parsing and composite keys.
- `SimsOnlineServerHostMod/ServerHostMain.cs` — refusal producer and state-change clears.
- `SimsOnlineServerHostMod/Presentation/Refusals.cs` — client mapping, TTL, and UI-hook callback.
- `DedicatedServer/Mirroring/RejectionText.cs` — player-readable late-refusal translation.
- `SimsOnline.Client/Program.cs` — consumes rejection and displays best-effort notification.

Tests validate text, delimiter safety, expiry-supporting data structures and message routing. A live menu test is needed to prove the local hook displays the exact intended item greyed. For questions shown during an interaction, see [Remote dialogs](remote-dialogs.md); those are not pie-menu refusals.

## Consistency across menu updates

A menu can be rebuilt many times as the player opens it, changes category, or returns to a previous page. The refusal table therefore describes server state, not one particular menu object. `Refusals.Lookup` is called while the local fork evaluates an entry; it must be cheap, exception-safe and keyed to the local actor/target aliases that the menu is currently using.

When a target or actor leaves the world, a stale refusal entry should not attach to a new object that reuses a runtime id. The server sends clears when actor/target disappears, and the client resets refusal state when its mirror session resets. TTL provides an additional bound, but identity mapping is the first safety requirement.

The definition-level query first avoids touching menu text if the server never refused that definition. The entry-level query includes the menu entry's visible name because multiple choices may share the same definition type. If the UI hook's name normalization differs from `MenuRefusal.Key`, the row may remain enabled; this is a compatibility issue to test against the live menu hook.

## Refusal versus disabled-by-game

The game itself can disable an entry for its local test, independent of the server refusal table. The multiplayer hook should preserve the game's own disabled state and add server refusal state; it should not turn a locally unavailable item green simply because the server currently accepts it. The two test results are complementary constraints:

| Local test | Server refusal | Expected UI |
|---|---|---|
| Pass | No refusal | Enabled |
| Fail | No refusal | Locally disabled |
| Pass | Refused entry | Greyed by server feedback |
| Fail | Refused entry | Disabled; retain server reason if the hook can display it |
| Unknown target alias | Refusal exists remotely | No unrelated entry is modified |

A tooltip from the server is the game's own explanation for its failed test. It should not be replaced with a guessed message. If there is no tooltip, a generic notification after the user picks is preferable to fabricating a specific reason.

## Correlation and message lifecycle

For post-pick rejection, `s_forwardedPicks` associates public command IDs with the original pick name. On `CommandRejected`, the mapping is removed before notification. A dropped or duplicate rejection should not produce repeated notifications, and an unrelated command must not inherit the previous pick label.

The refusal ring messages are state-like and can be replayed as current refusal state; post-pick notifications are transient. Do not retain and replay a historical “can't do that” toast to a late joiner. Conversely, do not rely on a one-time refusal line to initialize a new client if the server only broadcasts changes; resend current refusal state on the join/reset path where required.

## Quality checks

When changing refusal behavior, test the encoding and decode path, actor/target aliasing, repeated refusal, clear, malformed tooltip, expiry and menu-hook callback. Then test a real local menu with both local/server test disagreement and a late failure after click. Record whether the behavior came from the refusal table or the post-pick notification path.

## Menu honesty telemetry and support reports

A refusal report is most useful when it separates the server test result from what the client displayed. Capture the definition type and entry name in a synthetic/appropriately redacted form, whether the server sent a tooltip, whether the client mapping resolved actor/target, and whether the fork hook reported the row greyed. For a late rejection, capture the command correlation and translated notification category.

Do not infer a missing clear from the player seeing an item enabled: the client's own test may also fail, or the menu may have been rebuilt from a different target. Likewise, a row being grey does not prove the server refused it; the local engine can disable the row for its own reason. The diagnostic should identify which source contributed the visual state.

## Behavior change compatibility

The protocol parser accepts older refusal lines without the optional tooltip field. A producer change must preserve this compatibility unless a deliberate protocol-version transition is introduced. Clearing a refusal should not require parsing a tooltip; field 7 may be empty.

The UI hook is a separate build artifact from the controller. A controller build can send valid `Rf` lines while a client running an older UI fork ignores them. Include the client UI/game mod build stamp in live bug reports.

## Notification and dialog distinction

A notification is one-way player information. It should not hold the server Sim task waiting for an answer. A modal dialog contains a decision that may change the authoritative interaction outcome and therefore needs request identity, owner routing, answer authorization and timeout behavior. The two are carried using related UI plumbing but have different semantics.

A client-side toast failure is best effort; a server-side pending modal failure must fall back so gameplay can continue. See [Remote dialogs](remote-dialogs.md) for the answer lifecycle.

## Reply lifecycle test cases

The refusal stream has both edge-triggered and state-like behavior. Tests should pin these cases:

| Input sequence | Expected retained state |
|---|---|
| Refuse an entry | One keyed entry and one by-definition count |
| Repeat same refusal | Updated timestamp, not duplicate key count |
| Clear existing entry | Entry and matching definition count removed |
| Clear unknown entry | No negative count or unrelated removal |
| Malformed hex | Line rejected without hook mutation |
| Tooltip malformed only | Valid refusal retained, tooltip absent |
| Expire one of several entries | Only that entry removed; remaining definition count retained |

If the hook callback throws, the menu loop returns to its normal game behavior; it should not crash the game or permanently disable unrelated menu entries.
