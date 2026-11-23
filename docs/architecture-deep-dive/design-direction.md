# Design direction

This page describes design direction neutrally. It is not a statement that a
target server, replacement simulator, or deployment mode is currently
available. The [system overview](system-overview.md) and
[status](../status.md) describe the current runtime.

## Direction: preserve one authority while improving portability

The current architecture uses a full game process as its simulator. A
possible long-term direction is to reduce dependence on a visible,
single-process game host while preserving the behaviors that make the shared
world coherent. Any such design must replace contracts, not merely reproduce
type names or protocol shapes.

The direction is constrained by several facts:

- The game owns important simulation behavior behind managed and native
  boundaries.
- A full-world server is needed to keep world-level services coherent under
  the current engine model.
- The current process has finite address space and platform dependencies.
- Clients rely on a stable authority contract, snapshots, identity mapping,
  and presentation events.

## Possible evolution dimensions

| Dimension | Current shape | Directional question |
|---|---|---|
| Simulation host | Full Sims 3 process | Can required behavior run in another host without changing outcomes? |
| Presentation | Local game applies server state and selected recordings | Which boundary contracts are sufficient for complete, stable presentation? |
| Scope | Full authoritative world, client filtering | Can cost be reduced without breaking global services or world continuity? |
| Runtime portability | Game-specific process and integration | Which platform and version dependencies can be isolated? |
| Persistence | Server game/session commits the world | How would a replacement preserve save continuity and recovery? |

## Contracts a replacement must preserve

A replacement simulator would need evidence for each behavior it claims to
own, including:

- Session and player lifecycle.
- Claims and command authorization.
- World tick and clock semantics.
- Interaction acceptance, execution, and failure lifecycle.
- Autonomy and player-directed action ordering.
- Routing and lot transitions.
- Sims, objects, households, relationships, and inventory identity.
- Persistence, save recovery, and restart continuity.
- Snapshot bootstrap, late join, delta application, and resynchronization.
- Disconnect, timeout, and recovery behavior.

The public protocol alone is not a complete simulation contract. It
describes messages, not all of the game semantics behind them.

## Evidence gates

Before a new simulator or alternate host could be treated as current
architecture, it would need:

1. A versioned behavior contract for each replaced subsystem.
2. An authorized and reproducible evidence source for native-dependent
   behavior.
3. Differential tests against the current game for representative scenarios.
4. Save/load, restart, and recovery tests over realistic world state.
5. Multi-client and late-join tests under network delay and loss.
6. A stated compatibility matrix and a safe failure mode for unsupported
   behavior.
7. Clear ownership for protocol, persistence, and upgrade compatibility.

## Direction is not a promise

An experimental kernel or offline runtime, if present in a development
environment, is not the online session host unless the public runtime path
starts and validates it. A design sketch is not evidence that the game can
be replaced. This page intentionally avoids implementation timelines and
project scheduling language.

## Failure modes / limits

- Replacing the game requires reproducing semantics, not just data records.
- A partial simulator can create subtle divergence in social, travel, or
  persistence behavior.
- A smaller simulation scope may break global world services.
- Portability does not follow automatically from replacing one executable.
- A new host does not remove identity, synchronization, or presentation
  responsibilities.

## Related pages

- [Capability boundaries](capability-boundaries.md)
- [One-simulator decision](one-simulator-decision.md)
- [Server-authoritative simulation](server-authoritative-simulation.md)
- [Architecture overview](../architecture.md)
- [Status](../status.md)

## Directional options

A future host can evolve along more than one axis. The first is replacing
the game process entirely; the second is continuing to use the game but
improving deployment and observability; the third is using a hybrid where
selected contracts move outside while the game remains responsible for
behavior it still owns. These possibilities have different risk profiles and
should not be collapsed into a claim that “the Rust server” or another
target is already available.

| Option family | Potential benefit | Main unresolved burden |
|---|---|---|
| Improve current game host | Lowest behavioral gap | Platform, memory, native runtime constraints remain |
| Hybrid services | Isolate selected logic or orchestration | Cross-runtime consistency and ownership boundaries |
| Full replacement simulator | Portability and potentially different resource model | Reproduce gameplay semantics, saves, and compatibility |
| Multiple authoritative game hosts | Horizontal simulation resources | Partitioning, global services, cross-host travel, conflict handling |

## Behavioral contract examples

A replacement claiming to handle interaction queues must define more than an
`AddInteraction` API. It needs ordering, priority, cancellation, validity
checks, autonomous competition, route interruption, completion and failure
reporting, and persistence effects. Similarly, a clock needs pause and speed
semantics; an object identity system needs lifecycle and late-join behavior.

| Contract | Questions a conformance suite should answer |
|---|---|
| Tick/clock | What advances under pause, speed changes, and host stalls? |
| Interaction | When accepted, queued, started, canceled, or failed? |
| Autonomy | How are candidate actions selected and compared with user input? |
| Travel | Who owns route result, lot transition, and arrival state? |
| Identity | How do save-born and runtime-created entities correlate? |
| Persistence | What survives save/reload and crash recovery? |
| Sync | What is baseline state, and how are deltas repaired? |
| Compatibility | Which game/content versions are represented? |

## Design constraints from the current host

The current system's strongest architectural property is also a dependency:
the server game supplies gameplay behavior. A future host can preserve the
public command protocol yet still be behaviorally incompatible if it changes
queue priorities, autonomy, route results, object creation, or save
semantics. Compatibility must therefore be evaluated at the
player-observable contract level.

The current engine also presents global-world constraints. A reduced-lot
host cannot be assumed to preserve town progression or social services
merely because its network interest set is small. A future design needs
evidence that omitted world systems are either reproduced, intentionally
unsupported, or safely partitioned.

## Public communication principles

When describing direction to contributors or users:

- Label the current runtime separately from experiments.
- Say “design direction” for a target that is not the normal execution path.
- Describe open engineering questions as limits or unknowns, not scheduled
  work.
- Avoid implying that a shared protocol guarantees shared gameplay
  semantics.
- Keep compatibility and licensing constraints visible.

## How to evaluate a future host

1. Select an observable behavior such as a two-Sim social on a community
   lot.
2. Record the current game outcome and the required state transitions.
3. Implement the candidate behavior without importing proprietary game code.
4. Compare state, events, persistence, and recovery—not just the final
   screen.
5. Run repeated and adversarial timing cases.
6. Record unsupported cases explicitly.
7. Validate multi-client behavior and a cold restart.

## Failure modes / limits

A future direction can be technically plausible but product-incompatible.
Portability, a headless process, or memory safety alone does not establish a
playable replacement. The evidence burden rises with the amount of game
behavior being moved out of the current engine.

## Related pages

- [Capability boundaries](capability-boundaries.md)
- [One-simulator decision](one-simulator-decision.md)
- [Architecture overview](../architecture.md)
- [Status](../status.md)

## Current decision versus historical material

Architecture documents often combine three kinds of statement: a current
mechanism, an accepted decision, and an intended future model. Keep them
separated. A design page should use present tense only for code paths that
exist in the normal runtime. It should not describe a proposed replacement
as if users can select it today.

| Statement type | Safe wording | Avoid |
|---|---|---|
| Current mechanism | “The controller computes…” | “The architecture will compute…” when already implemented |
| Accepted invariant | “The system is designed to…” with implementation caveat | “Every path always…” without tests |
| Measured result | “In this test setup…” | “Supports…” as a general guarantee |
| Design direction | “A possible direction is…” | “The server now…” when it is not the default runtime |
| Known limitation | “The current path does not…” | Internal to-do or milestone language |

## Design evaluation matrix

| Evaluation axis | Questions |
|---|---|
| Behavioral fidelity | Which player-visible actions are reproduced? |
| Authority | Which process can mutate each durable value? |
| Performance | Which costs move to server, client, or transport? |
| Recovery | What happens after a process or network interruption? |
| Compatibility | Which game, mod, protocol, and save versions are supported? |
| Privacy | What user/world data is retained or transmitted? |
| Distribution | Are all binaries, references, and assets licensed for the intended release? |

## How to verify / test

- For each directional claim, point to current code or label it explicitly
  as a design proposition.
- Compare proposed behavior against the contract list and live scenarios.
- Include failure, recovery, and compatibility tests, not only a happy path.
- Review public licensing and data handling before describing distribution
  readiness.

## Related pages

- [Architecture overview](../architecture.md)
- [Capability boundaries](capability-boundaries.md)
- [Decision summaries](decision-summaries.md)
