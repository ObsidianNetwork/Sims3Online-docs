# Superseded approaches and why

This page records significant changes in direction so that older design notes
are not mistaken for current guidance. Superseded does not mean every fact in
the old analysis became false; it means the old proposal is no longer the
selected baseline.

## One simulator replaced client-native authority

An earlier synchronization direction considered clients running native
simulation on selected lots, or changing authority based on proximity and
activity. This could preserve local gameplay for nearby actors, but it
required handoff rules and reconciliation when Sims traveled or crossed
household boundaries. Multiple non-deterministic game processes also remained
capable of choosing different outcomes.

The selected baseline instead has one server game simulate the world while
clients provide input and present results. Some concepts from the old analysis
remain useful: recording the reason for an authority change, preserving state
across transfer, and distinguishing visible presentation from simulation.

## Presentation replay replaced interaction replay

Re-running the server's interaction locally seemed likely to reproduce
animation with less data. In practice, local routing, autonomy, object state,
timing and native decisions could differ. The replacement direction records
server outcomes at shared engine seams and replays those outcomes on the
client.

This does not imply a complete recorder exists for every game effect. Each
event family still needs coverage and live validation.

## Phased plan replaced milestone-specific assumptions

The early phases proved basic building blocks: a listener, ownership rules,
snapshot transport and client mirror paths. Those milestones were evidence
checkpoints, not a permanent architecture roadmap. Later work revisited
authority, stream semantics, clock ownership and presentation after obtaining
live measurements.

## Clean-room server is not a shortcut to full engine replacement

A Rust kernel can own bounded contracts such as authority state, routing, or
the public server boundary. Rehosting the game's managed assemblies without
their native runtime is not treated as a viable shortcut: the managed game
depends deeply on native callbacks and services, and routing data is not
simply a portable save artifact.

The kernel track therefore proceeds as a measured replacement of explicit
seams, not as an assumption that all gameplay can be switched over at once.

## Decision table

| Earlier approach | Why it was attractive | Replacement or disposition |
|---|---|---|
| Client-side authoritative zones | Local simulation, reduced server work | One simulator; clients present and send intent |
| Re-run each interaction on the client | Reuses game logic and animation | Record/replay actual server results at boundaries |
| Trust client-computed outcomes | Low latency | Server validation and ownership gate |
| Synchronize by lockstep | Simple conceptual model | Rejected because engine behavior is not deterministic across peers |
| Treat snapshot as durable save | One representation for network and storage | Native game save remains the durable artifact |
| Rehost managed game logic with a thin native shim | Reuse decompiled managed behavior | Feasibility analysis found the native dependency too broad for a thin shim |
| Replace all game behavior in one kernel | Fully headless server ambition | Staged clean-room work with differential contracts |
| Query server for every pie-menu item | Exact server test result | Prefer local state mirrors plus refusal feedback; avoid per-entry synchronous queries |

## How to read old records

- An archived proposal can contain useful measurements even after its
recommendation is superseded.
- A failed experiment is not automatically a failed architecture; it may
identify an invalid measurement or an unbuilt gate.
- A later design is not proven solely because it is newer. Check current code
and tests.
- When a claim matters operationally, use the public
[architecture](../architecture.md), [protocol](../protocol.md), and
[status](../status.md) pages as the current reference.

## Consequences

The project retains historical reasoning without preserving internal planning
identifiers or handoff instructions. Readers can understand which tradeoff was
selected while still seeing the remaining evidence gap.

## Failure modes and limits

- Calling a direction "superseded" can hide valid constraints discovered
during its analysis.
- A proposal can be mistaken for an implementation if the status boundary is
not stated.
- A newer transport or kernel does not automatically settle gameplay parity.

Related pages: [authority](decision-authority.md),
[presentation](decision-presentation.md), [clean-room
feasibility](clean-room-feasibility.md), [scope](decision-scope.md).


## Why early proposals were useful even when retired

An abandoned proposal often contains the best description of a constraint. The lot-sharding analysis makes travel ownership and merge costs visible. Interaction replay exposes which engine outputs were not being captured. The snapshot-only approach highlights the difference between state fields and transient presentation. Keeping these lessons helps contributors avoid repeating the same experiment while still allowing a new proposal when evidence or requirements change.

A rejected approach should be distinguished from a deferred one. Rejected means a specific assumption or mechanism was judged incompatible with the selected model. Deferred means the work may be revisited after a prerequisite or a separate contract exists. Neither label means that every implementation detail associated with it should be deleted immediately.

| Status | Meaning | Example phrasing |
|---|---|---|
| Selected | Current design baseline | ?The server owns gameplay outcomes.? |
| Rejected | Mechanism conflicts with evidence or trust boundary | ?Client lockstep is not assumed.? |
| Superseded | Earlier design replaced by a later decision | ?Lot-local authority gave way to one simulator.? |
| Deferred | Scope remains possible but lacks a prerequisite | ?Travel needs its own compatibility tests.? |
| Unresolved | Evidence does not distinguish options | ?The writer of the descriptor zero remains unknown.? |

## Reopening a settled decision

A decision may be revisited when an assumption changes: a new engine build exposes a usable seam, a public requirement changes, a measured failure demonstrates the current choice is inadequate, or the cost of the selected path becomes unacceptable. A reopening proposal should identify the old evidence and add the new discriminator; it should not merely restate the attractive part of a rejected alternative.

## Example: re-evaluating activity-gated authority

New interest-management data could show that server load is dominated by dormant lots. That alone would not prove client-local simulation is safe. A revisiting study would also need to measure authority handoff, client divergence, save ownership and behavior during travel. If the experiment only lowers network traffic, it may justify a more selective mirror rather than a new simulator owner.

## How to verify/test

- Check whether a cited proposal was only proposed or actually implemented.
- Compare its assumptions with current code and public docs.
- Preserve the old limitation if it remains true after the decision changed.
- Write a new bounded experiment instead of silently adopting a historical recommendation.
- Keep public references focused on technical history; omit private work tracking.


## Avoiding historical drift

A historical page should preserve its date and scope in a neutral sentence, but public technical pages should avoid copying internal phase labels or tracker references. When an old recommendation is superseded, state the replacement and which evidence remains useful. When a claim is unverified, use "not established" rather than presenting it as either pass or failure.

A review of a new proposal can use this short comparison:

| Check | Result to record |
|---|---|
| Does it assume more than one gameplay authority? | Identify any exception and its transfer rule. |
| Does it rerun server behavior on clients? | Name the divergence it may introduce. |
| Does it treat a snapshot as complete persistence? | Identify the native save boundary. |
| Does it bypass a measured failure? | Explain why the prior failure no longer applies. |
| Is it genuinely new? | State new evidence or requirements. |

## How to verify/test

Search the public docs and current code before reusing a historic decision. If the code contradicts the historical choice, update the current architecture page or explain the transitional state. Keep tracker labels, private roadmap detail and personal plan text out of public content.

Related pages: [principles](principles.md), [decision log topics](decision-scope.md), [status](../status.md).


## Mapping old terminology to current concepts

Historical architecture notes sometimes use milestone names or implementation labels that no longer describe the public design. For public readers, translate those references into the behavior: "client mirror" means applying server state to a local game copy; "resync" means a full snapshot repair after invalid or stale delta state; "authority handoff" means changing which process may mutate a gameplay actor. This keeps the explanation useful without importing internal tracker labels.

A change can supersede an implementation while retaining a valid constraint. For example, replacing a C# front door with a Rust process does not remove the need for the memory bridge or native game simulation. When documenting a transition, list what changed and what stayed invariant.

## How to verify/test

Search both source history and current code when an old statement seems contradictory. Confirm the replacement path is live before describing the older one as retired. If the state is uncertain, call it a transition or open question instead of declaring the migration complete.
