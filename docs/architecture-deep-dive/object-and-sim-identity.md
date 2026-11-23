# Object and Sim identity

The game uses more than one identifier for a Sim or object. A process-local
object handle answers “which live object in this game process?” A persistent
or semantic identifier answers “which entity is this across copies or
restarts?” Confusing those domains causes missing remote actors, incorrect
attachments, or accidental matches.

## Sim identity

A Sim has an in-world object identity and a description identity. The object
identity is useful for engine operations inside one game process.
Description identity is the more stable cross-process key for save-born
Sims. The network mirror can maintain a mapping from the server's Sim object
id to the local client's Sim instance, using description identity to resolve
a local counterpart.

| Identifier | Scope | Appropriate use | Risk when misused |
|---|---|---|---|
| Sim object id | One game process / world instance | Bridge operation against a known local Sim | May differ for a Sim instantiated after load |
| Sim description id | Sim identity across copies when shared | Alias and snapshot correlation | Runtime-created descriptions may not exist on every peer |
| Player id | Session | Claim owner and command authorization | Not a substitute for a Sim identity |
| Lot id | World/save context | Interest and placement context | Not an entity identity by itself |

## Alias resolution

The mirror needs to translate server identities to local engine objects
before applying a command. Conceptually:

```text
server Sim object id
        │ alias entry
        ▼
Sim description id ── lookup ──> local Sim instance
```

When a full state baseline arrives, the mapping can be refreshed as a set.
Alias updates must be applied before other commands that reference those
Sims. If a local object shares a numeric id by coincidence but represents a
different Sim, resolution must use the alias semantics rather than blindly
trusting the raw id.

## Runtime-created Sims

A save-born Sim may have a shared description identity while its live object
handle differs across processes. A runtime-created Sim adds another edge
case: one game may create a description that the other game does not yet
know about. Identity alone cannot produce a local object when the
description is missing; description data and lifecycle events must be
transferred before a mirror can instantiate or resolve the Sim.

Spawn and despawn are lifecycle events, not just snapshot fields. The
receiver needs ordering rules for:

1. Description data becoming available.
2. Local instance creation or resolution.
3. Alias publication.
4. Initial state and presentation events.
5. Despawn notification and cleanup.

## Object identity

World objects also have process-local object ids. Static furniture may
happen to line up when processes load identical save copies, but that is not
a general identity contract. Props created during play, inventory objects,
and objects in different save copies can have different ids or exist only in
one process.

A cross-process object mapping can use several clues: a server-issued
identity, a resource/type key, known save identity, and—only as a
constrained fallback—position and type. Each method has trade-offs:

| Resolution approach | Strength | Limitation |
|---|---|---|
| Same raw object id | Fast for truly shared entities | Numeric id collisions are possible across processes |
| Stable semantic identity | Clear correlation when available | Runtime entities need identity propagation |
| Resource/type key | Can create a generic local counterpart | Does not carry all instance-specific contents |
| Type plus nearby position | Useful for matching copied furniture | Ambiguous with repeated objects and moving entities |
| Create a local presentation copy | Resolves missing props | A copy is not necessarily the authoritative object |

## Object lifetime and references

An event that references an object requires that the client can resolve it
before applying the event. An announced object may later disappear; a client
should remove only a mirror copy it owns, not destroy a local object merely
because an unrelated server id vanished. History snapshots must include
identities needed by a late joiner because it may never have received the
original creation event.

Hand attachments are especially difficult: objects can be nested, carried,
attached through engine parenting, or removed through multiple code paths. A
snapshot of what a Sim holds can repair visual attachment state, but it does
not replace inventory contents or object persistence.

## Safe implementation rules

- Keep server ids and local ids in distinct namespaces or explicit alias
  maps.
- Include semantic identity in snapshots and events when the receiver needs
  cross-process correlation.
- Define whether an entity is shared, matched, locally created, or
  unresolved.
- Do not use a raw integer collision as proof of identity.
- Order creation/description data before references and cleanup after final
  use.
- Make unknown-object behavior explicit and diagnosable.

## Failure modes / limits

- Runtime-created descriptions may be absent from a client's save and need
  an explicit transfer path.
- A generic object copy may omit name, inventory, contents, or custom state.
- Position matching can select the wrong object in a dense scene.
- Interest filtering can omit object descriptions needed by an event.
- Identity correlation does not guarantee that the local engine can replay
  the requested behavior.

## Related pages

- [Scope, interest, LOD, and dormancy](scope-interest-lod-dormancy.md)
- [Lots, households, and travel](lots-households-travel.md)
- [Boundary seams](boundary-seams.md)
- [Bridge and formats](../bridge-and-formats.md)

## Comparing server and client identity

Identity correctness should be evaluated by matching semantic entities, not
by requiring raw object ids to be equal. A client may represent the same Sim
with a different process-local object id, provided the alias map
consistently translates server references to the local object. Tests should
record both the semantic match and whether the local instance was found or
created.

| Check | Expected interpretation |
|---|---|
| Same description id, same object id | Convenient shared identifier for this instance pair |
| Same description id, different object id | Valid cross-process alias case |
| Server description absent locally | Requires description transfer or a supported fallback |
| Same object id, different description id | Collision; never assume identity from number alone |
| Object resource key matches, position differs | Candidate object type, not proof of same instance |

## Snapshot and event ordering

The server-to-client alias table must be established before commands that
refer to server Sim ids. A full snapshot can resend the alias set and reset
stale mappings. Incremental alias updates should replace entries when a
server id is reused or a Sim changes lifecycle. Commands arriving before
alias publication should be delayed or rejected in a bounded way rather than
applied to an arbitrary local entity.

For object identity, a creation announcement should precede the first
presentation record that names the new object. Late-join history must
include descriptions for all objects referenced by the retained presentation
state.

## Object resolution example

Suppose a server-created plate is attached to a Sim's hand. The client
cannot resolve the server's object id in its save. The server can provide a
resource/type description; the client may create a generic local copy,
assign an alias, and then apply attachment/visibility records. On despawn,
only that locally-created mirror copy should be destroyed. The copy may
still omit food contents or other instance-specific data, so it remains a
visual surrogate rather than a full semantic object.

This example illustrates why object resolution can return different
outcomes: **shared** local object, **matched** counterpart, **created**
mirror copy, or **failed** unresolved reference. Those states should be
counted distinctly.

## Lifetime ownership

The server owns authoritative entity creation and destruction. The client
owns the lifetime of copies created only to render the server. A client
should not destroy an object that merely happens to have the same numeric id
as a server object. Similarly, a destroyed client copy should not imply that
the server entity was destroyed.

| Event | Authoritative effect | Client mirror effect |
|---|---|---|
| Sim spawned | Server creates or activates Sim | Resolve/create local instance and alias |
| Sim despawned | Server removes Sim from world | Remove presentation and alias safely |
| Object announced | Server describes object referenced by stream | Resolve local counterpart or create surrogate |
| Object gone | Server no longer has the announced object | Drop mirror-owned copy; retain unrelated local objects |
| Full snapshot | Server establishes current visible set | Reset/rebuild applicable aliases and state |

## Handling missing identities

When an identity is unresolved, the consumer should stop only the affected
operation, emit a diagnostic with the entity category and correlation
context, and continue unrelated state application when safe. A single
missing prop should not corrupt every later record. However, skipping a
required actor or target can invalidate an entire interaction replay, so the
failure must not be hidden as success.

## How to verify / test

- Load two instances from the same save and compare semantic Sim identities
  rather than raw ids alone.
- Add an entity after world load and verify creation, alias, snapshot, and
  despawn ordering.
- Reuse an id and verify stale alias replacement.
- Construct a deliberate numeric-id collision and assert no wrong local Sim
  is selected.
- Test every object resolution outcome and ensure counters distinguish them.
- Join after an object was created and confirm history contains the required
  description.
- Verify that server removal does not destroy an unrelated client object.

## Failure modes / limits

Description identity is not a universal solution for objects, and a resource
key cannot reconstruct arbitrary contents. Semantic aliases solve routing to
the correct local instance only when that instance or a creation path
exists. They do not prove parity of the object's behavior.

## Related pages

- [Lots, households, and travel](lots-households-travel.md)
- [Boundary seams](boundary-seams.md)
- [Boundary coverage and mirror gaps](boundary-coverage-and-mirror-gaps.md)
- [Bridge and formats](../bridge-and-formats.md)

## Sim alias lifecycle example

A save-born Sim may have the same description identity on both sides but
different live object ids after one process instantiates it later. The
server snapshot carries the identity relationship; the client controller
prepares an alias from server object id to description id; the host mod
resolves that description to its local Sim; later mirror operations use the
alias. If a full snapshot replaces the visible set, the alias table can be
reset and resent before any dependent command.

```text
snapshot: serverObject=8401, description=291
alias:    8401 -> description 291
local:    description 291 resolves to clientObject=5027
apply:    server reference 8401 maps to local object 5027
```

These identifiers are illustrative only. They show why equal object ids are
not required for a correct mirror.

## Object matching confidence

Object resolution methods should be ordered by confidence. An exact identity
mapping is strongest. A save-derived match may be suitable when the two game
instances are known to share a world baseline. Resource-key creation can
make a generic visual surrogate. Position/type heuristics are weaker and
should never be used to perform a durable gameplay mutation on the wrong
object.

| Confidence | Evidence | Safe usage |
|---|---|---|
| High | Explicit server-to-client alias or stable semantic id | Apply supported mirror operation |
| Medium | Same known save object and compatible identity | Resolve with diagnostics |
| Lower | Same resource type near expected position | Presentation-only fallback with ambiguity handling |
| None | Numeric id alone or no local match | Reject/skip affected reference and report |

## Structured identity diagnostics

Useful counters include the number of shared, matched, created, failed, and
dropped objects, plus Sim aliases resolved locally and those that required a
different local object id. Counters should be scoped to a session or bounded
window and should avoid names or private save paths unless explicitly
required for local diagnostics.

A rising failed-object count during one interaction indicates a different
class of problem from a failed Sim alias. Keep entity categories distinct so
a developer can tell whether a route actor, target chair, or carried prop
failed resolution.

## Security and trust

An id is a reference, not authorization. A client-supplied object id must
not grant access to arbitrary server objects; server command handling still
checks the actor's claim and allowed interaction. Similarly, a semantic
alias should be generated or validated by the server, not trusted as a
client request to mutate a different entity.

## How to verify / test

- Exercise alias refresh before first command dispatch.
- Test duplicate and stale alias updates, including full snapshot reset.
- Compare object identity across same-save and intentionally different-save
  instances.
- Verify generic copies are used only for presentation and do not become
  save owners.
- Test carried-object reconciliation across hand, carry, nested-container,
  and floor transitions.
- Confirm diagnostics distinguish unresolved entity type and do not silently
  fall back to raw ids.

## Failure modes / limits

An identity map can be internally consistent yet semantically wrong if the
description or resource key was mapped incorrectly. Validate the entity's
type and lifecycle alongside its id. A visual surrogate must never be
treated as a full object replica.

## Related pages

- [Bridge and formats](../bridge-and-formats.md)
- [Protocol](../protocol.md)
- [Boundary seams](boundary-seams.md)
