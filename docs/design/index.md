# Design records and engineering evidence

This folder explains durable architecture decisions, bounded experiments, and the current experimental Rust implementation. For behavior and support status, prefer [overview](../overview.md), [architecture](../architecture.md), and [status](../status.md). A historical result describes its stated workload; a unit test is not a live-game acceptance result.

## Topic directory

| Topic | Pages | Questions answered |
|---|---|---|
| Principles and decisions | [Principles](principles.md), [authority](decision-authority.md), [transport](decision-transport.md), [persistence](decision-persistence.md), [presentation](decision-presentation.md), [tooling](decision-tooling.md), [scope](decision-scope.md) | Which durable trade-offs separate simulation, transport, persistence, and presentation? |
| Change history | [Superseded approaches](superseded-approaches.md), [two processes](milestone-two-process.md), [client mirroring](milestone-client-mirroring.md), [two-player gameplay](milestone-two-player-gameplay.md), [menu honesty](menu-honesty.md) | What did earlier implementations establish, and which constraints remain relevant? |
| Evidence and behavior | [Probe results](probe-results.md), [synchronization research](synchronization-research.md), [clean-room feasibility](clean-room-feasibility.md), [runtime timing](runtime-timing.md), [multi-household behavior](multi-household.md), [social sessions](social-session-research.md) | Which conclusions are measured, and what is not established by those tests? |
| Rust implementation | [Kernel overview](rust-kernel-overview.md), [front door](rust-kernel-front-door.md), [state and identity](rust-kernel-ownership.md), [routing obligations](rust-kernel-obligations.md), [road placement](rust-road-placement.md), [pilot evidence](rust-kernel-pilot.md) | What code exists in the experimental kernel/server, how are ids and topology represented, and which tests cover it? |
| Rust comparison and limits | [Architecture options](rust-architecture-options.md), [divergence forensics](rust-divergence-forensics.md), [open problems](open-problems.md) | Which migration boundaries and unresolved mismatches remain? |

## Reading current versus historical claims

Current code is the authority for exact names, defaults, and behavior. Historical design records are useful for rationale but can describe superseded proposals. For Rust, read the crate status files and current code before treating an architecture option as a deployed mode. In particular, the Rust front-door crate has an implemented listener and session model, but its README still marks it experimental and unsupported for player release.

The original architecture survey and front-door records in the repository history motivated a staged Rust boundary: move session/network responsibilities without claiming to replace the game simulation. Those source notes also contain private coordination details, so this public folder retains only technical rationale and verified implementation details.

## Reading paths

- **Current architecture decision:** [principles](principles.md) → [authority](decision-authority.md) → [transport](decision-transport.md) → [persistence](decision-persistence.md).
- **A synchronization mismatch:** [divergence forensics](rust-divergence-forensics.md) → [probe results](probe-results.md) → [runtime timing](runtime-timing.md).
- **A Rust change:** [kernel overview](rust-kernel-overview.md) → [front door](rust-kernel-front-door.md) or [state and identity](rust-kernel-ownership.md) → [routing obligations](rust-kernel-obligations.md) / [road placement](rust-road-placement.md). Run the tests for the changed crate; do not infer native parity from pure fixtures.

## Evidence levels

| Evidence | Supports | Does not establish by itself |
|---|---|---|
| Decision record | Rationale and an intended ownership boundary | Implementation exists |
| Unit test | A named contract for its fixture | Native engine compatibility |
| Fake bridge/server test | Simulated boundary behavior | A live process handshake or deployment readiness |
| Live scenario | Observed behavior in a stated build/setup | Universal compatibility |

## Maintenance

Keep this index synchronized with filenames and links. When code changes, revise the implementation section of the relevant page and retain historical conclusions only as dated, bounded evidence. Run `python scripts/check_docs.py --anchors docs/design/` after editing this folder.

Related pages: [Architecture](../architecture.md), [Status](../status.md), [Glossary](../glossary.md).
