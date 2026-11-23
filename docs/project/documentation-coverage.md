# Documentation coverage

This page maps the topic areas of the project's earlier internal knowledge base to the public pages that now cover them, and says honestly how deep each is. Purely internal material (automation rules, personal planning, issue-tracker state, credentials, local machine details) has no public counterpart by design. Page counts are not coverage; the "Depth" column reflects whether pages cite real types, files and schemas.

| Old topic | Where it lives now | Depth | Known gaps |
|---|---|---|---|
| Interaction queue, lifecycle, wait/wake, coalescing, active orders | [Interactions and lifecycle](../simulation-reference/interactions-and-lifecycle.md) | Deep, code-grounded | Per-expansion behaviour differences unverified |
| Autonomy gates, scoring, decision injection, target resolution | [Autonomy](../simulation-reference/autonomy.md) | Deep | Scoring internals described at contract level only |
| Refusal reasons and player feedback | [Refusal reasons](../simulation-reference/refusal-reasons.md) | Deep | None known |
| Postures, terminals, routing | [Postures](../simulation-reference/postures-and-transitions.md), [Terminals](../simulation-reference/terminals.md), [Routing](../simulation-reference/routing.md) | Deep | Path-planner internals not public |
| Social, carry, books, cooking, mirrored state streams | [Social](../simulation-reference/social-interactions.md), [Carry and objects](../simulation-reference/carry-and-objects.md), [Mirrored state](../simulation-reference/scope-of-mirrored-state.md) | Deep | Some life-domain streams only summarised |
| Wire protocol: commands, events, handshake, resume, snapshots, limits, bridge verbs, schemas | [Protocol reference](../protocol-reference/index.md) (16 pages) | Deep: all 18 commands, 37 events, 75 bridge verbs, 39 schemas inventoried | Binary frame diagrams could be richer |
| Solution layout, server, client, host mod, protocol library, tests | [Code reference](../code-reference/index.md) | Deep (new) | Not every class documented; module-level only |
| Rust workspace, scenarios, schemas, scripts, tools, CI, repo hygiene | [Repository reference](../repo-reference/index.md) | Catalogue-level (new) | Catalogues list and describe; usage walkthroughs live in guides |
| Architecture, authority, seams, identity, LOD | [Architecture deep dive](../architecture-deep-dive/index.md) | Deep (re-verified against code in the latest pass) | Some historic engine claims remain build-specific |
| Persistence, world clock, reconciliation, replay | [Data and persistence](../data-and-persistence/index.md) | Deep (18 pages across persistence and presentation share a re-grounded pass) | Live-game behaviour not re-run |
| Presenter mode, mirroring, input, remote dialogs | [Presentation](../presentation/index.md) | Deep (dialogs, VM mirroring, replay, scenarios) | Live-game behaviour not re-run |
| Native observatory tooling contracts | [Native observatory reference](../repo-reference/native-observatory-reference.md) | Tooling contracts only | Engine analysis and decompile notes are excluded from public docs |
| Design records, Rust kernel | [Design records](../design/index.md) | Moderate | Historic plans intentionally summarised, not reproduced |
| Build, run, test, debug, tooling guides | [Guides](../guides/index.md) | Deep (CLI options verified against argument parsers and scripts) | Live-game steps not re-run |
| Troubleshooting, support vocabulary, common questions | [Troubleshooting](../troubleshooting.md), [FAQ](../faq.md), [Glossary](../glossary.md) | Troubleshooting and protocol terms now distinguish local bridge, TCP/session, game execution, mirror, and presentation failures | Live-game diagnosis remains dependent on a user's local installation |
| Public policies and Pages publishing | [Security](../security.md), [Code of conduct](code-of-conduct.md), [CI and Pages](../guides/ci.md) | Current contact routes and deployment permissions/triggers cross-checked against repository files and GitHub's official workflow guidance | Runner registration and environment protection are configured outside the repository |
| Archive handoffs, session notes, roadmap, trackers | Not published | n/a | Excluded as internal working state |

## Changes in this refresh

- The [quickstart](quickstart.md) gives a clean-checkout route through managed builds
  and the first test commands, with a separate path into game-backed work.
- The [runtime guide](runtime-modes.md) explains which server path and test suite to use.
- The [testing guide](../testing.md) provides local Rust authority sidecar commands
  and identifies which test runner covers each boundary.

## GitHub Wiki reconciliation

We reviewed all 273 Wiki pages against the current documentation and implementation.
The site already covered the current material from 212 pages. Verified details from
27 pages were merged into the relevant guides. The remaining 34 pages—superseded plans,
internal notes, and build-specific engine evidence—are excluded from the public site.

- [Dedicated server reference](../code-reference/dedicated-server.md) covers connection
  leases, telemetry, and late bridge replies.
- [Bridge layout](../protocol-reference/bridge-layout.md#host-outbound-queue) explains
  how command replies and events are scheduled alongside snapshot chunks.
- [Recording and replay](../presentation/recording-and-replay.md#trace-missing-interaction-observations)
  explains which boundary to check when an interaction observation is missing.
- [Snapshot fields](../data-and-persistence/snapshot-fields.md#check-snapshot-population-during-world-load)
  explains how to check snapshot population and posture on a fresh load.
- [Scenario catalog](../repo-reference/scenarios-catalog.md#manual-checks-for-input-and-game-time)
  includes manual checks for client input and game-time mirroring.

Update the repository docs to correct or extend this material. Merges to `main`
that change documentation inputs automatically rebuild the hosted site. See
[Documentation publishing](../guides/ci.md#documentation-publishing).

## Follow-up verification

- The [first live multiplayer session](../guides/first-live-session.md#verified-session)
  records a Windows host and mirror boot, checkpoint transfer, client join, full snapshot
  acknowledgement, and a matching Sim comparison on game build `1.67.2.024037`.
- The [starter scenarios](../guides/scenario-catalog.md#starter-scenarios) identify the local
  game fixture and distinguish executed passes from observed presentation failures.
- The protocol reference now includes a [warm resume handshake](../protocol-reference/handshake-and-resume.md#warm-reconnect),
  a [36-byte SNAPSHOT2 example](../protocol-reference/snapshots-and-resync.md#worked-example-for-an-empty-version-2-snapshot),
  and a [chunked replay event](../protocol-reference/replay-presentation.md#reassembling-a-line-v1-animation-event).
  The worked examples use synthetic fixtures and existing codec tests.

## Remaining gaps by priority

| Priority | Missing detail or evidence | Next verification |
|---|---|---|
| 1 | Portable fixtures and probe coverage beyond the [starter scenarios](../guides/scenario-catalog.md#starter-scenarios). | Document the prerequisites and verify the next subset using user-supplied game inputs and a disposable world. |

Credentials, personal saves and paths, proprietary game material, and internal handoffs
stay out of the published corpus.
