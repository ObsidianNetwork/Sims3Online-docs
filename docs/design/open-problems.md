# Known open problems

This page collects technical areas that the historical design records did not
close. The list is deliberately neutral: an open question is not necessarily a
defect, and a proposed experiment is not an implementation commitment. Check
[current status](../status.md) before treating an item as active work.

## Multiplayer acceptance

The early milestones established listener, ownership, mirror and selected
two-client flows. A full acceptance claim requires a reproducible end-to-end
run that covers concurrent clients, gameplay, reconnect, sustained duration
and the intended network conditions. Same-host tests do not represent a second
machine or WAN latency.

## Presentation coverage

- Some interaction categories were under-sampled in comparisons, including
activities with few examples.
- Animation loop count, outfit/model readiness and random decisions can vary
between processes despite identical request streams.
- Paired social interactions need a full session and effect-persistence
contract; mirroring a single terminal state is insufficient.
- Client-side local code may still run for Sims outside the selected household
even when input authority is centralized.
- Vehicle behavior and selected routing cases remain areas requiring
scenario-level evidence.

## State and household scope

The active household and active lot can influence native simulation detail.
Separate player households need explicit verification for funds, home-lot
state, bills, opportunities, travel, object identity and late join. A player
claim is not the same as the game's active-household setting.

## Persistence and recovery

The game-produced save remains the durable artifact. Cold join, warm join,
save rotation, interrupted transfer, restart and last-good recovery need
explicit acceptance coverage. Snapshot correctness does not replace
save-format validation.

## Rust front door and bridge

The Rust bridge and headless server crates are experimental. A complete
replacement requires validated frame layout, live bridge verb coverage, public
protocol compatibility, event sequencing, idempotency, backpressure, world
transfer, persistence, operator diagnostics and differential acceptance. A
library test does not establish that the Rust server is the default or
supported runtime path.

## Clean-room routing

Authored `SIMNAV01` and road fixtures can test deterministic planning without
copying game data. Real-world geometry and lane topology still require an
allowed authoring path and live differential comparison. Route placement, door
behavior, dynamic obstacles and native result representation are distinct
contracts.

## Runtime memory investigation

The descriptor-zeroing investigation narrowed plausible store mechanisms but
did not identify a confirmed writer. A candidate source should not be
presented as the cause until a constrained reproduction demonstrates the store
and a fix passes the relevant runtime test.

## Operational safety

The current experimental network protocol does not provide authentication or
encryption. It should not be exposed to untrusted networks. Public bind
posture, connection limits, listener readiness, queue limits and shutdown
behavior need validation before any deployment beyond a trusted development
environment.

## Failure modes and limits

- Treating all open items as equally urgent obscures their evidence and user
impact.
- Copying a historical checklist into current status would reintroduce private
project management state.
- A successful narrow test can be mistaken for closure of a broader
compatibility surface.
- A design choice can remain settled while implementation or acceptance
evidence is incomplete.

Related pages: [status](../status.md),
[troubleshooting](../troubleshooting.md), [two-player
gameplay](milestone-two-player-gameplay.md), [front
door](rust-kernel-front-door.md), [divergence
forensics](rust-divergence-forensics.md).


## Prioritizing open questions

An open item is most useful when it names an observable risk and a discriminating next test. "Routing is incomplete" is too broad; ?authored placement fixtures do not establish native candidate ordering on a live lot? identifies a measurable gap. Avoid adding internal owners, dates, or work-item state to this public technical list.

| Area | Risk | Evidence that would reduce uncertainty |
|---|---|---|
| Multiplayer | Clients diverge after reconnect or sustained play | Repeated live session with forced disconnect and recovery |
| Presentation | Native loop or clip choice varies | Same semantic decision captured and replayed across clients |
| Households | Active household receives unique native services | Test funds, bills, opportunities and lot scope across households |
| Persistence | Partial save transfer accepted | Truncation/corruption and restart recovery tests |
| Bridge | Layout or verb mismatch | Live command corpus against supported game build |
| Routing | Authored topology differs from native routes | Paired live route differential and safe authored topology |
| Memory | Descriptor writer unidentified | Constrained reproduction that names the exact store |
| Security | Network listener not safe for untrusted peers | Authentication, encryption and abuse/backpressure design |

## Distinguishing settled design from incomplete proof

A design can be settled while its implementation evidence remains incomplete. For example, "the server owns outcomes" is an architecture decision; whether every client copy consistently suppresses autonomy is an implementation and acceptance question. Keep those statements separate in issue reports and documentation.

## How to verify/test

- Check the current status page and tests before carrying an old limitation forward.
- Name the exact observable that will count as evidence.
- Include a negative control so the test can detect the failure it seeks.
- Retain the result and its scope, including a failed or inconclusive result.
- Update this page only when the supported design or evidence materially changes.

Related pages: [status](../status.md), [tooling decision](decision-tooling.md).


## Reporting progress on an open question

When evidence improves, update the statement at the narrowest level. For example, "the bridge accepts a command" can be replaced by ?the command round trip is verified on the stated development build,? while save compatibility remains open. Avoid marking an entire subsystem complete when only one verb or scenario was exercised.

A useful status paragraph contains four parts: current claim, supporting evidence type, tested scope and remaining gap. If a concern is resolved, point to the current contract and test rather than preserving a stale limitation. If the resolution is an accepted design but not code, label that distinction.

## What would close major gaps?

| Gap | Minimum credible closure evidence |
|---|---|
| Sustained multiplayer | Multi-client run with duration, disconnect and recovery cases |
| UI parity | Menu and visible-state comparisons across representative postures/objects |
| Households | Separate-household live scenario covering services and durable state |
| Save recovery | Interrupted checkpoint, restart and last-good recovery test |
| Rust front door | Clean build, wire conformance, live bridge and end-to-end client test |
| Routing | Authored topology plus native differential and client walk acceptance |
| Memory writer | Reproducer identifies the exact store and mitigation passes regression |
| Network security | Threat model, authentication, encryption and abuse tests |

## How to verify/test

Before a release claim, check that each closure result is recent enough for the current code, that the relevant test is not skipped on the release platform, and that test data is public-safe. Keep issue tracking and personal plans out of this page.


## Evidence freshness and lifecycle

An open question should not live forever after its premise changes. Before carrying a finding forward, compare it with current code, tests and documented status. A failed historical experiment remains useful as history, but a later successful gate may narrow the concern. Conversely, code that now exists may still lack live acceptance and therefore does not close the product-level question.

A useful update preserves the original scope and adds new evidence rather than rewriting the past. Record whether the new result confirms the old hypothesis, contradicts it, or covers a different environment. If the question is resolved, point to the contract or test that resolves it; if it is only deferred, retain the limit in neutral language.

## Evidence record template

For a new public finding, state: the observable behavior, tested build or fixture class, steps or command used, expected and actual result, failure modes exercised, and remaining unknowns. Keep personal notes and operator handoffs out of the technical record. Avoid publishing raw game-derived data; use authored fixtures and summarized measurements where possible.

Related pages: [probe results](probe-results.md), [tooling](decision-tooling.md), [scope](decision-scope.md).
