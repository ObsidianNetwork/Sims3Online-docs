# Scope of mirrored state: relationships, inventory, skills, and life domains

This reference distinguishes typed simulation snapshots, server events and the
host mod's boundary-state stream. They carry different slices of state. A
field's absence from `SimulationSnapshot` does not mean the project has no
other replication path, and an inventory membership record does not mean the
client has the same object instance.

## State surfaces at a glance

| Domain | Current representation | What is synchronized | Important boundary |
|---|---|---|---|
| Sim identity and pose | `SimSnapshot` | Object id, name, household id, position, current interaction, owner, lot, queue/posture metadata, description id | No typed pet species or career/skill list in the snapshot |
| World object | `ObjectSnapshot` | Object id, script class, position and lot id | No condition, contents, repair state or vehicle-seat graph |
| Relationships | Host-mod `Rl` boundary line | Pair state, short-term context, long-term fields and known traits when present | This is separate from the JSON `SimulationSnapshot` |
| Conversations | Host-mod `Cv` boundary line | Members, turn, who talked last and active topics | It mirrors conversation state, not the paired social animation |
| Watched inventory | Host-mod `Iv` boundary line | Item membership plus optional local script-type hints | Runtime items may be paired to local objects by type |
| Career and skills | `SimDetailsChanged`; `SimSkillChanged` | Career identity/branch/level and skill levels | Skill points and full career history are not in these event shapes |
| Pets | Ordinary `SimSnapshot` | General Sim identity/pose and interaction fields | No dedicated pet-state record in the public protocol |
| Vehicles | `ObjectSnapshot` plus route classification | Object pose and selected group-route legs | No typed driver/passenger or mount/seat state in the snapshot |
| Lot/household export | `LotExportBundle` / `HouseholdExportPayload` | Serialized household streams and optional inventory segments | Object-id remapping is guarded; this is not a general gameplay delta stream |

## Sim and object snapshots

`SimsOnline.Protocol/SimulationSnapshot.cs` defines:

- `SimulationSnapshot(Tick, WorldName, SaveName, Speed, Sims, Objects)`.
- `SimSnapshot(ObjectId, Name, HouseholdId, Position, CurrentInteractionId, OwningPlayerId, CurrentInteractionKey, LotId, InteractionQueueCount, PostureContainerObjectId, PostureType, SimDescriptionId)`.
- `ObjectSnapshot(ObjectId, ScriptClass, Position, LotId)`.

Optional trailing Sim fields allow older host mods to omit queue, posture or
description metadata; `SimDescriptionId` can therefore be null. These records
are useful for identity, placement and coarse activity, not a universal save of
all Sim and object state.

A small `SimSnapshot` example is:

```csharp
new SimSnapshot(
    ObjectId: 1001,
    Name: "Sim A",
    HouseholdId: 3001,
    Position: new Vector3Dto(12.5f, 0f, 8f),
    CurrentInteractionId: 42,
    OwningPlayerId: 7,
    CurrentInteractionKey: "Terrain.GoHere",
    LotId: 9,
    InteractionQueueCount: 1,
    PostureContainerObjectId: null,
    PostureType: "Standing",
    SimDescriptionId: 5001);
```

The values above illustrate the actual record fields. A real snapshot may omit
the optional trailing values, and `CurrentInteractionKey` is not a complete
interaction payload.

## Relationship records (`Rl`)

`SimsOnlineServerHostMod/RelationshipState.cs` serializes pair state onto the
host mod's ring/boundary stream. The body (without the ring sequence prefix) is:

```text
Rl|simA|simB|liking|stc|commodity|commodityValue|perceivedSim
   [|ltr|bits|stateStartedTicks|highestLiking|lastGreetedTicks|lastTalkedTicks
   [|traitsA|traitsB]]
```

| Field | Type in `RelationshipState` | Meaning |
|---|---|---|
| `simA`, `simB` | `ulong` | Pair endpoints; both must be non-zero |
| `liking` | `float` | Current long-term liking value sampled from the host |
| `stc` | `int` | Short-term context value |
| `commodity` | `int` | Current relationship commodity id |
| `commodityValue` | `float` | Its sampled value |
| `perceivedSim` | `ulong` | Sim identity associated with the short-term context |
| `ltr` | `int` | Long-term relationship state when extended fields are present |
| `bits` | `uint` | Relationship interaction bits |
| `stateStartedTicks` | `long` | Start time for the long-term state |
| `highestLiking` | `float` | Highest recorded liking value |
| `lastGreetedTicks`, `lastTalkedTicks` | `long` | Pair interaction timestamps |
| `traitsA`, `traitsB` | comma-separated `ulong` ids or `-` | Known traits from each endpoint's perspective |

The parser accepts three frame lengths: the original 9-field form, a 15-field
form with long-term fields, and a 17-field form with both trait lists. It rejects
partial long-term groups, zero Sim ids and malformed numeric/list values.
The host mod publishes relationship changes and periodic refreshes; a refresh
is useful for clients that join after an earlier update.

This supersedes the older assumption that relationship values had no mirror
path. The current mechanism copies sampled relationship fields to the local
Sim representation; it does not make the client authoritative for the host's
social outcomes.

## Conversation records (`Cv`)

`SimsOnlineServerHostMod/ConversationState.cs` carries transient conversation
state as a boundary line:

```text
Cv|simA|simB|inConversation|whoseTurn|whoTalkedLast|extraMembers|topics
```

| Field | Meaning |
|---|---|
| `simA`, `simB` | The two lowest-listed members used as the pair key |
| `inConversation` | `1` for active membership, `0` for an end/absence record |
| `whoseTurn` | Sim id whose turn is current, or zero when absent |
| `whoTalkedLast` | Last speaker id, or zero when absent |
| `extraMembers` | Additional members, comma-separated ids or `-` |
| `topics` | Comma-separated topic names or `name:paramSimId:activeSimId` entries, or `-` |

Topic names containing `|`, `,` or `:` cannot be represented and are left out.
The parser requires exactly the expected frame shape and rejects malformed
member ids, turn ids, or topic tuples.

The explicit absence form is:

```text
Cv|1001|1001|0|0|0|-|-
```

It tells a client that a Sim the host successfully inspected is not in a
conversation. The host publisher sends ended memberships before new ones and
uses an absence record for newly observed idle targets, so a client does not
keep an old conversation merely because it never saw its end. If the host
cannot read a target's conversation, it does not claim that target is absent.

Conversation replication and social action replay are independent. `Cv` can
carry turn and topic data while the client remains on the route-only social
interaction path. See [Social interactions](social-interactions.md).

## Inventory membership (`Iv`)

`SimsOnlineServerHostMod/InventorySet.cs` publishes watched inventory
membership in the host mod's boundary stream:

```text
Iv|ownerObjectId|itemId,itemId,...|typeName,typeName,...
```

An empty membership is `Iv|ownerObjectId|-`. Item ids are sorted and duplicates
removed. The owner id can name a streamed Sim or a nearby object with an inventory. The optional type list is in the same order; a type containing a comma
or pipe is normalized to underscores. A parser can still read older records
that do not have type hints.

Example:

```text
12|Iv|810|11,15|BookGeneral,PhoneSmart
```

Here `12` is the ring sequence, `810` is the owner, and the two server item ids
have corresponding local script-type hints. The presentation-side
`SimsOnlineServerHostMod/Presentation/Inventories.cs` can pair a missing
server-side runtime item with one unused local item of the exact same type.
This preserves useful inventory membership for menus and object lookup; it is
not a cross-process identity proof. Unknown type `?` is not paired.

Current bounds and behavior:

- `InventorySet.MaxItems` is 200. The server does not send an inventory above
  that cap; the client retains its existing local state for that inventory.
- Watched inventories are refreshed as nearby streamed Sims and objects are
  reconsidered; changes and periodic refreshes are published by
  `ServerHostMain`.
- Membership does not include arbitrary item condition, quality, recipe,
  progress or carry animation state.
- A semantic book id can resolve a book even when its host object id is local;
  ordinary inventory membership alone does not make every object targetable.

`LotExport.cs` defines a separate, deeper data shape. `HouseholdExportPayload`
can contain `SimInventories`, `SharedFridgeInventory` and
`SharedFamilyInventory` as serialized segments. The current validator caps the
combined payload at `512 * 1024` bytes and rejects a non-empty object-id table
with `ObjectGuidRemapUnproven`. The protocol comments also describe the deep
import path as quarantined pending a safe lifecycle. Do not equate this export
schema with unrestricted live inventory replication.

## Skills and careers

The public protocol now has explicit detail events rather than skill fields in
every position snapshot:

```csharp
new ServerEvent.SimDetailsChanged(
    SimObjectId: 1001,
    CareerGuid: 0,
    CareerLevel: 0,
    CareerBranch: "-",
    Funds: 2000,
    Skills: "Cooking=4+Logic=2");

new ServerEvent.SimSkillChanged(
    SimObjectId: 1001,
    SkillName: "Cooking",
    Level: 4);
```

`SimDetailsChanged` is the combined claimed-Sim detail update. Its fields are
`CareerGuid`, `CareerLevel`, `CareerBranch`, `Funds` and a compact `Skills`
string. The string is `-` when no levels are included, or sorted
`Name=level+Name=level` pairs; zero-level skills are omitted by
`SimsOnlineServerHostMod/SimDetailsWire.cs`. The event translator turns it into
`BridgeCommand.ApplySimDetails`.

`SimSkillChanged` is a single skill-level delta translated to
`BridgeCommand.ApplySkill`. These event types synchronize career identity/level
and skill levels; they do not carry the full career history, performance
metrics, skill points or every skill-specific internal field. The diagnostic
`SimSkillProbeReport` parses a separate report with `sim`, `skill`, `level` and
`points`; probe-only points are not fields of `SimSkillChanged`.

`DedicatedServer/Authority/AuthoritativeSession.cs` publishes changed details
for claimed Sims and periodically resends them. It does not publish an
unclaimed Sim's details through the same claimed-Sim path. This distinction
matters when comparing a visible town Sim with a player-owned Sim.

## Pets and vehicles

Pets are represented as Sims at the generic snapshot level: `SimSnapshot`
provides identity, household, position, interaction and posture fields, but no
`IsPet`, species, pet skill or pet-specific action payload. A dog or horse being
present in the Sim list therefore proves only the fields in that snapshot.
Pet interactions and carried pet/toy state are not promoted to a dedicated
public protocol type here.

Vehicles are represented, at most, by the generic `ObjectSnapshot` fields. The
current `SimsOnlineServerHostMod/RoutingInteractionFilter.cs` explicitly
recognizes the group-route keys:

- `Sims3.Gameplay.Objects.Vehicles.LeaderGoHereWith`
- `Sims3.Gameplay.Objects.Vehicles.FollowerMeetUpAtPoint`

These are route legs; they do not synchronize vehicle entry/exit, seat
occupancy, driver/passenger relationships or a mounted posture as vehicle
state. When a vehicle's object id does not resolve on the other process, a
route can still degrade to position movement, but that is not vehicle
reconstruction.

## How to verify the contracts

The current C# tests cover serialization, compatibility and state transitions
in `DedicatedServer.Tests/Program.cs`:

- `RelationshipState.SendsLongTermStateAndReadsItBack` and
  `RelationshipState.ReadsOldLinesAndRefusesMalformedOnes`.
- `RelationshipState.SendsTraitsAndReadsItBack` and
  `RelationshipState.RefusesABadTraitItem`.
- `ConversationState.SendsConversationAndReadsItBack`,
  `ConversationState.SendsEndedConversationAndReadsItBack`,
  `ConversationState.RefusesMalformedLines` and
  `ConversationState.AbsenceReachesTheTargetsClient`.
- `InventorySet.SendsOneOrderAndReadsItBack`,
  `InventorySet.DiffsTheClientsSetAgainstTheServers` and
  `InventorySet.CarriesEachItemsTypeAndPairsRuntimeItems`.
- `SimDetails.PublishOnChangeAndApplyOnClients` and
  `SimSkillProbeReport.ParsesValidReport`.
- Household export validation cases for size limits, malformed payloads and
  unproven object-id remapping.

The tests establish parser/publisher behavior. They do not prove that every
engine-specific pet, vehicle, career or object-condition field is present in
the public snapshot.

## Failure modes and limits

| Symptom | Likely boundary |
|---|---|
| Client menu still offers a book or item that the server removed | Inventory membership was not sent, exceeded the cap, lacked a usable type pair, or the inventory was not watched |
| Two local objects have the same script type | Type pairing is only a best-effort local alias; it cannot distinguish semantically different same-type items |
| Relationship values update but social animations differ | `Rl` state synchronization does not replay the paired interaction |
| A client remains in an old conversation | It missed an end/absence record, or the host could not read the target and correctly withheld absence |
| Skill level matches but progress differs | The event carries level, while probe `points` is not part of that event |
| Career fields differ for an unclaimed Sim | The combined detail publisher is scoped to claimed Sims, with separate late-join behavior |
| Vehicle moves but occupants do not match | Route and generic object positions are not a driver/seat relationship model |
| A household export is rejected | Check `HouseholdExportRejectionReason`; `ObjectGuidRemapUnproven` is intentional, not a generic parse error |

## Related pages

- [Social interactions](social-interactions.md)
- [Carry and objects](carry-and-objects.md)
- [Protocol](../protocol.md)
- [Persistence and time](../persistence-and-time.md)
- [Simulation](../simulation.md)

