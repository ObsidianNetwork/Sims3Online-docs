# Decision record: project scope and replacement boundaries

## Context

There are two related but distinct questions: how to make the existing game
engine work as a multiplayer authority, and how much of that authority could
eventually be replaced by a clean-room kernel. The project found that
expanding scope too early obscured the achievable multiplayer product and
confused experiments with a replacement plan.

## Decision

- Keep the current baseline centered on one game-backed simulator and client
presentation.
- Treat the Rust kernel as an experimental staged track, not as evidence that
the game process can already be removed.
- Bound each kernel function by a testable contract, and only expand after
comparing output against the native behavior.
- Avoid describing deferred gameplay areas as delivered. Travel worlds,
editors, vehicles, pets, careers, and broad NPC behavior require their own
evidence and compatibility boundary.
- Preserve save, rendering and client game requirements as explicit concerns
even if server-side rules move into Rust.

## Scope matrix

| Area | Current status in the design | Evidence needed to widen scope |
|---|---|---|
| Core player input and shared interaction | Server-authoritative baseline | Multi-client acceptance across supported scenarios |
| Client visual state | Mirrored/replayed presentation | Complete event coverage and drift diagnosis |
| Same-world multiplayer | Target experience under active development | Repeatable live test with reconnect and sustained-session checks |
| Special travel worlds | Bounded compatibility concern | Explicit travel and save-transition matrix |
| Create-a-Sim / Edit Town editors | Outside ordinary interaction flow | Separate ownership and save contracts |
| Build/Buy | Not a generic per-Sim replay problem | Transactional shared-economy and editor-specific contract |
| Clean-room kernel | Experimental replacement path | Differential, persistence, runtime and client-render acceptance |

## Why a staged kernel?

A narrow kernel can provide explicit ownership, session state, routing
contracts or protocol handling without first reimplementing every compiled
interaction. This can improve headless operation and make some rules easier to
test. It does not erase the native client requirement, reproduce the entire
game save model, or solve animation replication.

A proposal should state whether it aims to match the original engine or define
new gameplay rules. Those goals have different parity costs and should not be
mixed into one acceptance test.

## Consequences

- A clean-room component is not automatically a replacement for its
game-backed counterpart.
- The C# controller can act as a differential oracle during a transition, but
must not become a hidden runtime dependency after a front-door migration.
- The game remains responsible for behavior that has not been explicitly
transferred and tested.
- Test fixtures must be authored and reproducible; they should not rely on
copying proprietary game data into the repository.

## Failure modes and limits

- A kernel can pass unit tests while omitting real-world content or engine
dependencies.
- A broader demo can appear to work while silently using the old controller
path.
- Scope expansion can confuse an experimental prototype with supported
behavior.
- Matching one game version does not guarantee compatibility with another
version or expansion set.

Related pages: [clean-room feasibility](clean-room-feasibility.md), [Rust
kernel overview](rust-kernel-overview.md), [open problems](open-problems.md),
[status](../status.md).


## Scope gate for a new subsystem

A proposed server-owned subsystem should declare whether it is simulation, coordination, persistence, or presentation. Those classes imply different correctness standards. Simulation replacement must account for native outcomes; coordination must account for connection and ownership lifecycle; persistence must preserve durable game semantics; presentation must remain ordered and address the correct local objects. One feature may cross multiple classes, but each boundary should be named.

| Gate | Questions before implementation | Evidence before claiming completion |
|---|---|---|
| Product intent | Match original behavior or define a new rule? | User-visible acceptance criteria |
| Dependencies | Which native service, save field or content asset is required? | Dependency inventory and portability check |
| Authority | Which process owns each mutation? | Positive and adversarial ownership tests |
| Compatibility | Which game/build/world variants are in scope? | Explicit compatibility matrix |
| Recovery | What happens after disconnect, crash or restart? | Injected failure and recovery scenario |
| Privacy/release | Does fixture contain game-derived or personal data? | Public-source and artifact review |

## Example: deciding whether to replace routing

A deterministic route planner can be attractive because it is easy to unit-test and can run outside the game. The scope gate asks whether its input topology can be authored, whether its route legs can be translated to the native API, whether obstacles and doors are represented, whether the server game accepts the result, and whether client presentation follows it. Until those boundaries are validated, the work is a kernel capability rather than a replacement for world routing. See [routing obligations](rust-kernel-obligations.md) and [road placement](rust-road-placement.md).

## Release language

Use "experimental" when the component is not a supported player path. Use "implemented" for a concrete code path, "tested" with the exact tier, and "accepted" only when the relevant acceptance gate has passed. "Headless" should describe the server process interface, not imply the original game has been removed.

## How to verify/test

- Check the current README and launch path, not only the crate tree.
- Trace one real command end to end through the selected process topology.
- Confirm save, reconnect and shutdown behavior for any replacement listener.
- State untested game modes and expansions explicitly.
- Verify public wording matches current [status](../status.md).


## Scope language for contributors

Use the following terms consistently in designs and review summaries. "Input capture" means the client observed a user's action. "Command accepted" means the service admitted the request for processing. "Interaction started" means the game began running it. "Completed" means the engine reported a terminal result. "Presented" means the client applied enough state to show the result. These are separate milestones and should not be collapsed into one green check.

| Term | Required evidence |
|---|---|
| Captured | Client produced a bounded intent frame |
| Authorized | Server claim/role check passed |
| Dispatched | Game or kernel received the operation |
| Observed | Authoritative result was returned or recorded |
| Mirrored | Client applied the state/event |
| Persisted | Game save boundary retained the state after reload |

## Scoping an acceptance claim

An acceptance claim should list the tested world class, client count, join type, interaction family, network conditions and duration. If a concern is out of scope, say so plainly. A local test can be valid evidence for its own gate while remaining insufficient for public release language. This precision reduces the chance that "two-player works" is interpreted as support for every world, household, interaction and network.

## How to verify/test

Review the claim against the current code path and test command. Check that the selected stage has no hidden fallback to an older controller, and that failures are reported at the right boundary. For a compatibility expansion, add a scenario that exercises the new condition rather than only adding a prose statement.

Related pages: [status](../status.md), [testing](../testing.md), [open problems](open-problems.md).


## Scope boundary examples

A system can be within scope for coordination but out of scope for simulation. For example, the server may know that a vehicle is traveling between lots while not yet reproducing the vehicle's full native behavior. A client can receive the authoritative household balance while Build/Buy transaction semantics remain outside generic interaction replay. Describe each capability at the layer that has actually been implemented.

A public design page should also distinguish constraints from goals. "One server process owns the session" is an architecture constraint; "every player can travel anywhere" is a product goal that requires compatibility coverage. A goal does not become a current capability by appearing in an old plan.

## Scope review template

For a proposed addition, answer: (1) which user-visible behavior changes; (2) which authority owns it; (3) what data and assets are required; (4) which clients and worlds are covered; (5) what happens when the operation fails; (6) how durable state is saved; and (7) which test demonstrates the claim. If any answer is unknown, record it as an open design question.

## How to verify/test

Check code and tests for the exact path named in the claim. Run a focused scenario and review the current status page. Avoid adding unsupported capabilities to the navigation page.


## Scope negotiation example

Suppose a proposal adds a second household to a shared session. This spans several boundaries: the session can coordinate membership, the native game still computes household services, the client displays the active lot, and persistence stores durable household state. The proposal should not be summarized as merely ?support another player.? It should identify which household each player controls, whether a claimed Sim can travel, which home-lot services run, and how the save/rejoin cycle behaves.

A review should start with the smallest observable capability. If the second player can join and claim a Sim but cannot use every household service, document that exact boundary. Do not silently promote a connection milestone into a full gameplay-compatibility claim.

| Review question | Example evidence |
|---|---|
| Who owns the action? | Claim table and server-side authorization test |
| What changes in the world? | Native interaction result and event trace |
| What does the client show? | Mirror test with two client instances |
| Is it durable? | Save, restart and reload scenario |
| What happens on failure? | Disconnect or refusal test with recovery |

## Scope and public documentation

The same discipline applies to documentation. Describe the component that exists, name the source of the behavior, and avoid implying an experimental tool is a supported launch path. Where a capability depends on local authored fixtures or a particular game build, record that qualification adjacent to the claim.

Related pages: [multi-household](multi-household.md), [authority](decision-authority.md), [open problems](open-problems.md).
