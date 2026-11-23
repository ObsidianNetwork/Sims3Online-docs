# Project glossary

Core terms used across the public docs. For process relationships see
[architecture](../architecture.md); for wire-level concepts see
[protocol](../protocol.md).

| Term | Definition |
|---|---|
| **Authority** | Right to determine a shared-world result; gameplay authority resides with the host. |
| **Authoritative game** | Server-side Sims 3 process whose outcomes define the session. |
| **Authoritative session** | Controller-managed participants, claims, commands, and event distribution. |
| **Alias mapping** | Association between a wire identity and a process-local game object. |
| **Animation driver** | Running state-machine instance responsible for animation calls. |
| **Boundary recording** | Capture a game operation at a shared engine boundary for client presentation. |
| **Bridge** | Local communication channel between controller and in-game mod. |
| **Bulk slot** | Larger bridge region for payloads too large for a control frame. |
| **Bootstrap** | Initial world/session data sent to a joining client. |
| **Claim** | Session association granting a player permission to command a Sim; not simulation authority. |
| **Client** | Player's controller and game process; forwards intent and presents host results. |
| **Controller** | .NET process managing session/network behavior and communicating with a game mod. |
| **Control ring** | Fixed-size bridge slots for framed command and response traffic. |
| **Cross-process identity** | Identifier mapping the same Sim/object concept between separate game processes. |
| **Delta snapshot** | Snapshot update advancing state from an earlier baseline. |
| **Dedicated server** | Session host/controller paired with the authoritative game; not necessarily a hosted service. |
| **Event** | Message reporting command result, state transition, or presentation update. |
| **Full snapshot** | Complete baseline used to initialize or replace mirrored state. |
| **Fork** | Modified build of a game integration component containing hooks; not the game itself. |
| **Gameplay** | State-changing simulation such as interactions, autonomy, relationships, or household effects. |
| **Host** | Authoritative server-side simulation and controller. |
| **Interest scope** | Sims/objects a client currently needs to receive. |
| **Intent** | Player-requested action forwarded for host validation. |
| **Interaction replay** | Historical method executing host interactions again on clients; not the current default. |
| **Mirror** | Client application of host state and presentation records. |
| **Model mirror** | Synchronization of durable system fields rather than replay of gameplay operation. |
| **Puppet** | Client representation of a host-owned Sim, constrained to presentation. |
| **Presentation** | Visible/audible output shown by a client to reflect host simulation. |
| **Presentation stream** | Ordered records of selected host calls applied to a client. |
| **Refusal** | Host response that a request did not pass validation or game-side tests. |
| **Sim claim** | Player-to-Sim ownership record used to authorize commands. |
| **Sim identity** | Stable identity across processes, distinct from local runtime object ID. |
| **Snapshot** | State baseline or update sent from host to client. |
| **State machine** | Game mechanism driving animation or other stateful behavior. |
| **Transport** | Delivery layer for controller messages or bridge frames. |
| **User-directed input** | Direct player choice distinguished from autonomous actions and continuations. |
| **World bootstrap** | Transfer of current host world/session baseline to a joining client. |

## Common distinctions

- **Accepted** means the host accepted a request; it does not mean the action
  completed.
- **Connected** means a transport exists; it does not mean the client is synchronized.
- **Mirrored** means a representation was applied; it does not mean every game system
  is identical.
- **Claimed** means one player is authorized to command a Sim; simulation remains on
  host.
- **Snapshot** and **presentation stream** are complementary channels.

See [status](../status.md) and [limitations](known-limitations.md) for current scope.

## Runtime and protocol terms

| Term | Definition |
|---|---|
| **Active actor** | Sim selected as the player's current actor in the local UI. |
| **Answer dialog** | Client response tied to the host's outstanding dialog request. |
| **Bridge command** | Framed request sent between a controller and its in-game mod. |
| **Bridge event** | Mod-originated report returned through the local bridge. |
| **Client role** | Mod/controller operating as a player's endpoint, not the shared simulator. |
| **Command** | Request from a controller to perform an operation, such as joining or queuing. |
| **Correlation ID** | Identifier used to associate a request with its bridge response. |
| **Dialog request** | Host-created prompt routed to the player who owns the associated Sim. |
| **Dialog answer** | Choice returned by the client and validated by the host-side waiting task. |
| **Full snapshot** | Complete state baseline for a joining or resynchronized client. |
| **Host role** | Mod/controller mode paired with the authoritative game process. |
| **Join** | Session operation that authenticates a participant in current session state; it is not game-account login. |
| **Line codec** | Encoder/decoder mapping bridge text frames to typed controller events and commands. |
| **Notification** | Non-modal message displayed to a user; it may be forwarded separately from a dialog answer. |
| **Object picker** | UI dialog whose rows/tabs are presented to a player and whose selection is returned as an answer. |
| **Public protocol** | Controller-to-controller message format; distinct from the local mod bridge. |
| **Request ID** | Identifier for one dialog interaction, used to reject or correlate stale answers. |
| **Role gate** | Check restricting a bridge operation to server or client operation. |
| **Server event** | Typed event distributed from the session host to a client. |
| **Server role** | Mod/controller mode responsible for host-side simulation integration. |
| **Snapshot delta** | Incremental update that assumes a known earlier snapshot baseline. |
| **Transport acknowledgment** | Confirmation that a message was received/accepted at a layer; not proof gameplay completed. |
| **Wire identity** | Identifier serialized across a process boundary, often resolved through an alias table. |

## State and presentation terms

| Term | Definition |
|---|---|
| **Boundary state** | Latest recorded state for a keyed presentation property, used to reconstruct a current view. |
| **Driver lifecycle** | Sequence from animation-machine creation through state changes and release. |
| **Event-like record** | One-time occurrence that should not be replayed as durable state. |
| **Interest set** | Current set of Sims/objects the server distributes to a particular client. |
| **Late join** | Client entering after the host world has already been running. |
| **Local object ID** | Handle meaningful within one game process; it may not identify the same runtime object elsewhere. |
| **Model state** | Durable or current values representing a subsystem, such as funds or posture. |
| **Overlay** | Animation layer used for reaction or other additional visual behavior. |
| **Posture container** | Object/location associated with a Sim posture, such as the seat being used. |
| **Presentation batch** | Bounded group of ordered records applied to a client game. |
| **Presentation baseline** | Current live state needed for a client to begin presenting an already-running scene. |
| **Presentation correction** | Explicit update used to repair a client representation; not a new simulation decision. |
| **Replay cursor** | Sequence position separating already-applied records from later records. |
| **Route record** | Data observed from host movement and used to present that movement on a client. |
| **State-like record** | Record describing a current value that remains relevant until replaced or ended. |
| **Transient effect** | Short-lived visual/audio event that may not be represented in a snapshot. |
| **World model** | Set of shared Sim/object/session state represented across host and clients. |

## Testing and support terms

| Term | Definition |
|---|---|
| **Acceptance scenario** | End-to-end test with explicit setup, action, observation, and pass criteria. |
| **Blocked test** | Test unable to reach a verdict because a prerequisite or environment failed. |
| **Compatibility claim** | Explicit statement of tested/supportable versions and configurations. |
| **Deterministic test** | Test whose expected result does not depend on uncontrolled timing or random order. |
| **Failure mode** | A known way a capability can fail, described with symptoms and likely evidence. |
| **Integration test** | Test across two or more software boundaries, such as controller and bridge. |
| **Known limitation** | Documented unsupported, partial, or unverified behavior. |
| **Live test** | Scenario exercising a running game process rather than only isolated code. |
| **Reproduction** | Minimal, repeatable steps that cause an observed result. |
| **Sanitized log** | Diagnostic output with personal, secret, and private-network data removed. |
| **Unit test** | Focused test of a component or function without proving the full running system. |
| **Verification evidence** | Test result tied to a command, environment, and behavior claim. |

## State words in user interfaces

| Word | Use it when | Do not imply |
|---|---|---|
| **Sent** | Client transmitted a request | Host accepted or executed it |
| **Received** | A transport boundary accepted data | Game-side validation passed |
| **Accepted** | Host accepted the command | Interaction completed |
| **Queued** | Host placed work in an interaction queue | Work will necessarily finish |
| **Refused** | Host rejected the request | The client has a complete explanation |
| **Completed** | Authoritative completion was reported | Every client has already rendered the result |
| **Unknown** | The client cannot establish outcome | Success or failure |

Definitions describe public project usage, not every internal type name. See
[protocol](../protocol.md), presentation, and
[limitations](known-limitations.md).

## Configuration and lifecycle terms

| Term | Definition |
|---|---|
| **Active Sim** | Local UI-selected Sim used as the current player actor. |
| **Authority mode** | Configuration determining whether the host or a supported alternate mode owns simulation. |
| **Bridge frame** | One command/response unit transported between local controller and mod. |
| **Capability** | Server-advertised feature name; not proof the client has a compatible implementation. |
| **Client-native lot mode** | Alternate mode in which a client owns limited lot simulation data and reconciles it with host under a distinct contract. |
| **Connection** | Live transport channel, not equivalent to a joined session or authenticated identity. |
| **Event cursor** | Sequence/tick boundary used to order updates or resume after a snapshot. |
| **Game speed** | Shared simulation speed value; client UI input must be routed through server policy. |
| **Join baseline** | Snapshot/state used to initialize a new participant. |
| **Lot reconciliation** | Validated merge of supported client-native lot state into host. |
| **Mirror generation** | A lifecycle boundary that invalidates prior client-applied records. |
| **Protocol discriminator** | `$type` value naming a JSON command or event. |
| **Rejoin** | New connection/session participation after disconnect; old socket state is not assumed valid. |
| **Resync** | Replacing a possibly stale client mirror with an authoritative baseline. |
| **Server push** | Event distribution initiated by the server, independent of client polling. |
| **Session role** | Participant's declared server/client behavior within a connected session. |
| **World clock** | In-game day/hour/minute state, distinct from host wall clock. |

## Terms for test evidence

| Term | Definition |
|---|---|
| **A/B comparison** | Interleaved comparison of two configurations under controlled conditions. |
| **Baseline** | Reference result or state captured before a change. |
| **Cold start** | Starting the game/process from a fresh process state rather than a live reload. |
| **Evidence artifact** | Sanitized log, report, or sample set supporting a claim. |
| **Flake** | Intermittent test behavior not reliably attributable to a code change. |
| **Live stack** | Running server and client game/controller processes used for integration checks. |
| **Pass rate** | Number of passes divided by clearly stated eligible attempts. |
| **Probe** | A narrow diagnostic query or controlled operation used to observe behavior. |
| **Scenario reset** | Returning world/process state to known reproducible conditions. |
| **Timeout** | A bound after which a wait ends; it does not by itself prove a defect. |
| **Trace** | Ordered records linking an input to processing and outcome. |

## Words to avoid without qualification

- **Real-time:** give measured latency and workload if this claim matters.
- **Synchronized:** state which fields and what time window matched.
- **Supported:** identify version, platform, configuration, and acceptance evidence.
- **Safe:** name the threat model; the public protocol remains unauthenticated.
- **Complete:** define the feature boundary and tested cases.
- **Deterministic:** do not use to mean ?usually looks similar.?

This glossary describes the public vocabulary, not every internal type or historical work label. See [compatibility](compatibility-matrix.md), [status](../status.md), and [protocol](../protocol.md).

## Release, documentation, and community terms

| Term | Definition |
|---|---|
| **Artifact** | Build output produced by CI or a local build; inspect contents before distribution. |
| **Code of Conduct** | Community participation policy in the repository root. |
| **Compatibility matrix** | Summary of feature/environment combinations and their evidence level. |
| **Contribution** | Proposed source, test, documentation, or tooling change under repository terms. |
| **Docs checker** | Script that verifies selected Markdown links, anchors, and repository path mentions. |
| **Documentation build** | MkDocs validation/rendering of the public site. |
| **Issue report** | Public description of a bug or feature proposal; should be sanitized. |
| **MIT License** | License in the repository root for material it covers; not a game-content license. |
| **Non-endorsement notice** | Statement that Sims3Online is unofficial and not affiliated with EA/Maxis. |
| **Public release** | Version/artifacts intentionally made available with stated support boundaries. |
| **Reference assembly** | Compile-time API signature material; technical purpose does not itself settle redistribution rights. |
| **Strict build** | Documentation build that treats warnings as errors. |
| **Third-party notice** | License/attribution text required for incorporated third-party material. |
| **Unverified** | No sufficient test evidence for the stated scope. |
| **Workaround** | Temporary way to avoid a symptom that does not necessarily fix its cause. |

## Directory vocabulary

| Directory | Public-facing meaning |
|---|---|
| `DedicatedServer/` | Host controller and session implementation |
| `SimsOnline.Client/` | Player-side controller |
| `SimsOnline.Protocol/` | Shared TCP message model |
| `SimsOnlineServerHostMod/` | Game scripting integration |
| `SimLink/` | Optional native integration component |
| `DedicatedServer.Tests/` | Controller/server automated tests |
| `scripts/` | Repository development and maintenance tools |
| `scenarios/` | Live game scenario inputs |
| `docs/` | Public technical documentation |
| `Assets/` | Project artwork materials |
| `refs/sims3/` | Reference assemblies whose rights need separate review |

Repository paths can change; see [layout](repository-layout.md) before relying on them. This glossary deliberately omits private tooling names, internal tracker conventions, and local-only paths.
