# Presentation scenario catalog

`scenarios/` contains 52 tracked files: a README, 49 runnable scenario definitions, and three start-position presets. The JSON cases exercise client/server presentation and local integration; they are not part of the ordinary offline unit suite. `scripts/scenarios.py` is the actual runner.

## JSON contract

| Field | Shape | Meaning |
|---|---|---|
| `name` | string | Runner-visible scenario key, usually the filename stem. |
| `summary` | string | Human-readable intent; it can also flag an expected gap or prerequisite. |
| `clients` | optional integer array | Participating client slots. |
| `household`, `start`, `player2`, `players` | optional setup values | Household, lot/position or player-to-Sim setup; structure varies by case. |
| `steps` | string array | Ordered action DSL interpreted by the runner (`act:`, `sleep:`, `cpick:`, `until:`, `hud:`, `probe:`, `capture:` and related commands). |
| `watch` | optional object | Observation window, commonly `sims`, `seconds`, `capture`. |
| `after` | optional string array | Post-run assertions and observations such as `sims:`, `drivers`, `nonotice:` and `capture:`. |
| `expect` | string | Desired/documented classification, typically `pass` or `known-failing: ...`; not a fresh test result. |

A checked-in example, `social-chat.json`, uses this shape:

```json
{
  "name": "social-chat",
  "summary": "Jennifer chats with Pablo (facing, balloons, social state)",
  "steps": ["act:Jennifer sim:Pablo social:Chat --wait 40", "sleep:10"],
  "watch": {"sims": ["Jennifer", "Pablo"], "seconds": 28, "capture": true},
  "after": ["sims:Jennifer Pablo --machines", "capture:Jennifer:2"],
  "expect": "pass"
}
```

The `start/home.json`, `start/langeraks.json` and `start/park.json` fixtures map Sim labels to four-number position/orientation arrays. These are scenario fixtures, not a general coordinate contract.

## Run with the real runner

First inspect available cases and local readiness, then consult current `run --help` for optional client count, boot/rejoin, repeat, watch, speed and tuning flags.

```sh
python scripts/scenarios.py list
python scripts/scenarios.py health
python scripts/scenarios.py run --help
python scripts/scenarios.py run sleep-bed
python scripts/scenarios.py explain sleep-bed
python scripts/scenarios.py status
```

Only help/list/health were used as read-only inspection here; no scenario was executed. A real `run` requires a compatible local game installation and configured stack, may start/alter local processes, and writes run evidence. `explain` reports saved evidence; it does not rerun a case. The runner's actual verdict is the observed result: `expect` is only the case's target/current metadata. `expect: "pass"` may fail on a new run, while `known-failing` does not mean every step fails.

## Manual checks for input and game time

Run these checks with a disposable save and a healthy local host and client. Record the build, save and claimed Sim with each result.

| Check | Steps | Evidence |
|---|---|---|
| Client input round trip | In the client game, issue a ground movement command for a claimed Sim. Watch both game windows. | Match the forwarded intent to the server action. Confirm that the server Sim starts moving, then compare the client's rendered movement and final position with the server. |
| Clock and speed mirroring | Change to another supported running speed in the host game, then restore the original speed. Watch both game clocks during each change. | Record the client's receipt of `GameSpeedChanged` and check its log for dispatch errors. Observe both clocks changing pace and returning to the original pace. |

Forwarding, queueing and event delivery each confirm one stage. Record direct observation of the rendered action or clock for an end-to-end result. See [Presentation scenarios](../presentation/presentation-scenarios.md) for evidence and reporting guidance.

## Single-client and local presentation cases

| File / runner name | Purpose | Recorded expectation |
|---|---|---|
| `burnt-floor.json` / `burnt-floor` | World floor change appears in client presentation. | pass |
| `carry-paper.json` / `carry-paper` | Newspaper pickup, carry and disposal visuals. | pass |
| `copy-flip-resend.json` / `copy-flip-resend` | Warm rejoin after an animation/pose copy changes. | pass |
| `dj-booth.json` / `dj-booth` | DJ-booth interaction and client playback/presentation. | pass |
| `fire-near.json` / `fire-near` | Nearby fire state and flame location. | pass |
| `fridge-snack.json` / `fridge-snack` | Fridge-to-counter snack flow and food-prop comparison. | known-failing: food props differ |
| `holiday-lights.json` / `holiday-lights` | Lot holiday-light presentation. | pass |
| `idle-repair-stand-walk.json` / `idle-repair-stand-walk` | Repair loop and return to standing/walking state. | known-failing: repaired idle reconstruction differs |
| `late-join-bed.json` / `late-join-bed` | Join while a Sim is in bed, then observe the exit. | pass |
| `late-join-phone.json` / `late-join-phone` | Join during a phone action and replay recent history. | known-failing: finished phone driver is replayed |
| `late-join-seated.json` / `late-join-seated` | Join while seated and replay the sit transition. | known-failing: seat-relative root motion diverges |
| `phone-seated.json` / `phone-seated` | Browse on a phone while seated. | pass |
| `phone-standing.json` / `phone-standing` | Standing phone action and hand-off. | pass |
| `puddle-sound.json` / `puddle-sound` | Puddle creation plus resulting sound/presentation. | pass |
| `routed_walk_multiroom_lot.json` / `routed_walk_multiroom_lot` | Multi-room route movement across a lot. | JSON expects pass; summary marks prerequisite blocked pending measured topology. |
| `shower.json` / `shower` | Shower interaction in its intended room. | pass |
| `sit-dining.json` / `sit-dining` | Walk to dining chair and sit. | pass |
| `sleep-bed.json` / `sleep-bed` | Bed entry and sleep posture. | pass |
| `social-chat.json` / `social-chat` | Paired-social facing, balloons and state. | pass |
| `sting-owner.json` / `sting-owner` | Owner-targeted career sting across two clients. | pass |
| `walker-copy-rejoin.json` / `walker-copy-rejoin` | Rejoin while a movement/animation copy is in progress. | pass |

## Multiplayer, routing and synchronization cases

| File / runner name | Purpose | Recorded expectation |
|---|---|---|
| `conv-client-cancel.json` / `conv-client-cancel` | Cancel a server conversation from a client. | pass |
| `conv-client-leave.json` / `conv-client-leave` | A participant leaves during a conversation. | pass |
| `conv-group-leave.json` / `conv-group-leave` | Group conversation when one participant leaves. | pass |
| `conv-save-stale.json` / `conv-save-stale` | Recover client/server stale conversation state. | pass |
| `park-menu-parity.json` / `park-menu-parity` | Compare interaction-menu choices with server refusal at a public lot. | known-failing: client offers a social the server refuses |
| `park-walk.json` / `park-walk` | Two players route from separate homes to the park. | known-failing: long-distance driving/presentation remains open |
| `three-park-meet.json` / `three-park-meet` | Three-player, multi-household park meeting and relationship state. | known-failing: long-term relationship state is not carried live |
| `two-chairs.json` / `two-chairs` | Two players choose different dining chairs simultaneously. | pass |
| `two-families-buy.json` / `two-families-buy` | Household purchase and currency change on home lots. | pass |
| `two-families-home.json` / `two-families-home` | Two households at home and client-state synchronization. | pass |
| `two-families-late-join.json` / `two-families-late-join` | One player's game rejoins while both households are at home. | pass |
| `two-families-visit.json` / `two-families-visit` | Household member visits another lot and introduces herself. | pass |
| `two-hud-server-away.json` / `two-hud-server-away` | HUD behavior when the server camera is away from a Sim. | pass |
| `two-hud.json` / `two-hud` | Per-player HUD queues and cancellation ownership. | pass |
| `two-late-join.json` / `two-late-join` | Late join during another client's phone action. | pass |
| `two-look-at.json` / `two-look-at` | Puppet gaze follows server target during social interaction. | pass |
| `two-lots-apart.json` / `two-lots-apart` | Separate-lot actions, cancellation and movement comparison. | known-failing: motion comparison still fails |
| `two-lots-late-join.json` / `two-lots-late-join` | Rejoin while a Sim is on a different lot. | pass |
| `two-lots-walk.json` / `two-lots-walk` | Vehicle travel between lots and movement synchronization. | known-failing: vehicle travel presentation differs |
| `two-nap-refused.json` / `two-nap-refused` | Server refusal when a shared seat is occupied. | pass |
| `two-paper-delivery.json` / `two-paper-delivery` | Service-created newspaper delivery replicated to the client. | pass |
| `two-paper-refused.json` / `two-paper-refused` | Refuse a newspaper action when it is stale/unavailable. | pass |
| `two-park-meet.json` / `two-park-meet` | Two players meet at a community park. | pass |
| `two-same-chair.json` / `two-same-chair` | Simultaneous attempt to claim the same chair. | pass |
| `two-shelf-read.json` / `two-shelf-read` | Shared bookshelf availability and read action. | pass |
| `two-social-cut.json` / `two-social-cut` | HUD cancellation and interruption of a paired social. | pass |
| `two-social.json` / `two-social` | Player 2 initiates a paired social with the other player's Sim. | pass |

## Fixtures, outputs and limits

The three start fixtures are listed above; unlike scenario definitions, they have no `steps` or `expect`. At this checkout, 9 definitions record `known-failing`; their text describes observed gaps, not permanent promises. The runner saves run evidence and verdicts locally, and commands such as `history`, `explain`, `status`, and `rescore` interpret existing results. A case that has not been run on the current stack has no current observed outcome.

Scenario command strings form a DSL rather than a formal schema. Changing a command parser in `scripts/scenarios.py` can invalidate JSON that still parses correctly. Tests for the runner live in `scripts/scenarios_test.py`; live presentation remains distinct from those offline tests.

## Related

- [Testing](../testing.md)
- [Development setup](../development.md)
- [Protocol reference](../protocol-reference/index.md)
