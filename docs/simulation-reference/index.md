# Simulation reference

How player intent becomes queued game work, how autonomy picks actions, and how routes, postures,
objects and social interactions are represented and mirrored between game processes. These pages
separate simulator behaviour from network acceptance and visual mirroring. Engine-facing
descriptions are interoperability guidance, not a guarantee that every expansion or game build
behaves identically.

| Page | Answers |
| --- | --- |
| [Interactions and lifecycle](interactions-and-lifecycle.md) | How is a player command queued, started, completed, cancelled or coalesced? What are the request parameters and emitted events? |
| [Autonomy and command gating](autonomy.md) | When may a Sim act on its own, how do autonomy gates change, and how are books, seats and other targets resolved semantically? |
| [Refusal reasons](refusal-reasons.md) | Why was a command refused or failed after acceptance, and what feedback does the player see? |
| [Postures and transitions](postures-and-transitions.md) | How are standing, sitting, sleeping and similar states modelled and transitioned? |
| [Terminals](terminals.md) | How do terminal states of an interaction or route get reported? |
| [Routing](routing.md) | How do route commands, `GoHere` and route replay work? |
| [Social interactions](social-interactions.md) | How do approach and paired actions work, and what is replayed versus mirrored? |
| [Carry, books, cooking and objects](carry-and-objects.md) | How are carried items, reading progress, cooking and object condition represented? |
| [Scope of mirrored state](scope-of-mirrored-state.md) | Which relationships, inventory, skills and life-domain data are mirrored, and through which streams? |

Related: the [protocol reference](../protocol-reference/index.md) for message shapes, the
[code reference](../code-reference/index.md) for where this lives in source, and the
[architecture deep dive](../architecture-deep-dive/index.md).
