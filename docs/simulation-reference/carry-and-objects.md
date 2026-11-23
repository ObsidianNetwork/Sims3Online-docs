# Carry, books, cooking, and object condition

Carry is a change in object containment, not just a pose. A Sim may route to an
item, move it into a hand or inventory, continue into a second interaction,
and later put it down or consume it. The public mirror exposes only portions
of that chain; the book path is the most explicit integration, while cooking
and object-condition state remain broader simulation behavior.

## Carry model

The game-side carry API is centered on `CarrySystem` and the marker interfaces
`ICarryable`, `ICustomCarryable` and `IStackingCarryable`. Standard items,
custom carry graphs and stackable servings do not have identical requirements.
The operations have these observable roles:

| Operation | State boundary | Common failure |
|---|---|---|
| `PickUp` | Moves a carryable from world or inventory into the actor's hand | Item cannot be resolved, is already in use, or the actor cannot carry another item |
| `PickUpWithoutRouting` | Begins pickup when routing is already satisfied | The actor is not actually adjacent or the item is invalid |
| `PickUpFromSimInventory` | Moves an inventory item into the carry state | Inventory membership differs or the requested item is absent |
| `PickUpAnother` | Adds a compatible stackable object | Type or stack capacity is incompatible |
| Custom pickup | Uses the object's own carry graph | Required graph/target state is unavailable |
| `PutDown` | Returns the item to a surface, slot or world position | Destination is unavailable or containment transition fails |

A successful-looking animation is not proof of containment. When diagnosing a
carry chain, inspect the actor's held-object/containment state and the item's
inventory or world membership separately.

## Object identity across processes

A world object's `ObjectId` is useful when the same object is present in both
simulation processes. An inventory object created during play may instead have
an id that cannot resolve on another process. The current inventory stream
handles some runtime objects by pairing a server item with a local item of the
same exact script type; this is a presentation alias, not proof that both
processes hold the same object instance or item contents.

Books have an additional stable identity: the book-data id carried as
`bookId`. The book resolver prefers this semantic key when present rather than
assuming a host-side object id is meaningful in the client process. See
[scope of mirrored state](scope-of-mirrored-state.md) for the `Iv` inventory
membership stream and its limits.

## Book reading chain

Reading can start from an inventory book, a world book or a chooser that selects
between those paths. The implementation separates the book's semantic data id
from its process-local object id.

| Stage | Current integration behavior |
|---|---|
| Capture | `ServerHostMain` can append `bookId` for a book target and `progressNormalized` for a `ReadBook` terminal |
| Resolve | `BookChainReadIntentResolver` chooses the `ReadBook` terminal from a batch, then uses a `bookId`, target object id or `GetBook` precursor target to locate the book |
| Inject | `MirrorEngine` dispatches a resolved book chain through `INJECT_READ_INTENT` |
| Client lookup | `TryResolveTerminalReplayTarget` tries an existing object, a matching inventory book and then a locally created book from its data id |
| Read mode | `INJECT_READ_INTENT` accepts `chooser` or `getbook`; the engine-side chooser or GetBook runs its ordinary continuation logic |
| Progress | `progressNormalized` is constrained to 0..1 and seeds the client's saved read position when terminal apply resolves a book |

A conceptual injected intent has this shape:

```text
actor: <sim-object-id>
mode: chooser | getbook
bookObjectId: 0 when resolving from bookId, otherwise a local target id
parameters:
  bookId=<book-data-id>
  progressNormalized=0.4
```

The bridge command is `INJECT_READ_INTENT`; its wire form carries a
correlation id, Sim id, book object id, hex-encoded mode and optional
hex-encoded parameters. It is accepted only by the client-role host mod. The
semantic example above is easier to inspect than the encoded frame and does
not imply that any arbitrary book id will resolve.

`BookChainReadIntentResolver.TryResolveFromBundles` and
`TryResolveFromInteractions` select the last matching `ReadBook` terminal in a
batch. When `bookId` is present the resolver sets the object id to zero so the
client uses semantic lookup; otherwise it prefers the terminal target and
then a `GetBook` precursor target. The resolver can also preserve seat hints
from a deferred sit transition so reading does not choose a different seat
based on the client Sim's position.

### Read progress is not the same as read animation

The engine's saved progress is per Sim and book-data id. `ReadBookData`
progress can be expressed as a fraction of the book's data length; the
wire-facing parameter is `progressNormalized`, not an absolute page number.
The host mod captures it for a `ReadBook` terminal and seeds the local record
on apply. Seeding resumes the record; it is not the same as running a read
loop. Without a live `ReadBook` interaction, engine loop callbacks that
normally advance pages do not run.

The batch path and the plain interaction-event path are not interchangeable.
`InteractionReplayBundle.Parameters` carries a parameter map; consult
`DedicatedServer/Mirroring/EventToBridgeCommandTranslator.cs` before assuming
that an ordinary `InteractionQueued` row contains every book hint or progress
field. If the client fails to resolve the book, `MirrorEngine` reports a
missing target and degrades the presentation rather than treating a route as a
successful read.

### Book identity and branch selection

| Client state | Likely result |
|---|---|
| Matching book already in the Sim inventory | Existing inventory object can be reused |
| Book data id is known but the host object id is local-only | Client can resolve or create its own inventory book from the data id |
| Only an object id is supplied and that id is absent locally | Target resolution fails |
| `ReadBook` singleton chosen for an inventory item while posture is unsuitable | Its local `Test` may reject the queue |
| Progress is seeded but no read loop runs | Page state is restored, but does not advance by itself |

The exact engine continuation remains local: an inventory choice and a
world-book pickup can start from different states. Avoid reducing them to a
single â€œread animationâ€ event when comparing traces.

## Cooking and serving objects

Cooking is a multi-step chain rather than one terminal interaction. The
historic engine study describes recipe choice, ingredient acquisition, a
station interaction, completion/quality state and a newly created serving or
plate. Eating the result is another interaction. A serving created during the
chain can have a different object identity from the fridge, stove or ingredient
that started it.

The current public snapshot types do not define recipe, cooking-progress,
quality, appliance-condition or serving-content fields. The current terminal
filter has explicit book prefixes but no cooking terminal family. An unknown
interaction with a target can be classified as `ReplayAsRoute`; that is a
movement approximation, not a cooking simulation. Do not infer recipe choice,
plate creation, quality or eating from a route reaching a stove.

| Cooking stage | State to observe independently | Mirror limitation |
|---|---|---|
| Select recipe | Actor choice and recipe identity | No recipe field in `SimulationSnapshot` |
| Acquire ingredients | Inventory membership and carried item | Inventory pairing is by local type, not full item identity |
| Work at appliance | Interaction key and appliance availability | A route does not run the cooking process |
| Create serving | New object and its type/containment | Runtime object identity may not match across processes |
| Place or carry | Surface slot or actor containment | Position alone does not establish placement |
| Eat | Serving state and actor motive | Separate terminal lifecycle; not implied by cooking completion |

For supported public state fields, start with
`SimsOnline.Protocol/SimulationSnapshot.cs` and
`SimsOnline.Protocol/LotExport.cs`. The latter defines inventory segments in a
household export, but `HouseholdExportValidator` rejects a non-empty object-id
table while the remap is unproven; the export shape is not a general object
identity guarantee.

## Cleanliness and repair state

The game distinguishes cleanable state (for example `Cleanable.NeedsToBeCleaned`)
from repairable state (`Repairable.Broken`). These are object-level conditions,
not Sim motives. Interactions may read these values when checking availability
or selecting an animation branch. A clean object can be broken, and a dirty
object can still be usable depending on the interaction's own test.

The current `ObjectSnapshot` fields are `ObjectId`, `ScriptClass`, `Position`
and `LotId`; they do not carry dirty level, visual-dirty state, broken state or
repair progress. `SimsOnlineServerHostMod/ObjectUse.cs` synchronizes object
occupancy/use references, not cleanliness or repair status. Do not confuse an
object-use record with a condition snapshot.

The practical consequence is that client-side menu eligibility or an
animation parameter may disagree with authoritative behavior if the object
condition differs. Treat an authoritative terminal result as stronger evidence
than the local menu or visual branch. Exact cleaning thresholds and per-object
tuning are not encoded in the public protocol types.

## Identity and chain failure modes

| Failure | Why it happens | Diagnostic distinction |
|---|---|---|
| Pickup target not found | Host object id has no client counterpart and no semantic item key was supplied | Resolution failure precedes carry execution |
| Book resolves but read does not start | Wrong chooser branch, missing posture, or a local test rejects the interaction | Distinguish item resolution from queue acceptance |
| Book animation plays but progress is static | A page value was seeded without a running read loop | Compare progress over time, not only at start |
| Serving is absent on the client | Cooking spawned a new runtime object that has no local counterpart | Do not use the stove id as the plate id |
| Object menu differs by endpoint | Dirty/broken state is not a `SimulationSnapshot` field | Compare state at the host and local object, then interpret the interaction's own test |
| Household inventory export is refused | Payload is malformed/oversized or contains an object-id table that lacks remap proof | `HouseholdExportRejectionReason` distinguishes `Malformed`, `Oversized` and `ObjectGuidRemapUnproven` |

## Verification in this repository

Relevant current implementation and tests include:

- `DedicatedServer/Mirroring/BookChainReadIntentResolver.cs` â€” target and parameter resolution.
- `DedicatedServer/Mirroring/MirrorEngine.cs` â€” book intent dispatch and missing-target fallback.
- `SimsOnlineServerHostMod/InteractionReplayApplySorter.cs` â€” book precursor/terminal recognition.
- `SimsOnlineServerHostMod/ServerHostMain.cs` â€” `bookId`, `progressNormalized`, target fallback and `INJECT_READ_INTENT` handling.
- `DedicatedServer.Tests/Program.cs` â€” `InteractionQueuedCoalescer.StripsBookPrecursorsWhenReadBookPresent`, book-bundle resolver fixtures, and bridge encoding tests for read progress.
- `SimsOnline.Protocol/LotExport.cs` â€” validation for deep household inventory segments.

## Related pages

- [Eating terminal](terminals.md)
- [Autonomy and target resolution](autonomy.md)
- [Interaction wire parameters](interactions-and-lifecycle.md)
- [Social interactions](social-interactions.md)
- [Scope of mirrored state](scope-of-mirrored-state.md)
- [Protocol](../protocol.md)

