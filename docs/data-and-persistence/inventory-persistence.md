# Inventory persistence and mirror behavior

Inventory is both durable household data and an active gameplay input. A book taken from a shelf changes whether another Sim's menu test succeeds; an item can be carried, attached to a hand slot, moved between household storage and a Sim, or created at runtime with an ID that exists only in one game process. The current implementation has a narrow live membership mirror and book-specific resolution paths, not a general inventory-object replication system.

## Three different inventory mechanisms

| Mechanism | Data | Scope |
|---|---|---|
| Native save | Game-owned inventory/object persistence | Durable world reload |
| Household deep export | Property stream and optional inventory segments | Bounded export payload; deep import remains quarantined |
| `InventorySet` presentation line | Membership IDs plus optional item type names | Current inventory membership for watched owners on a client mirror |

Do not call the presentation membership line a save. It does not contain all item properties, item content, use state, every inventory class, or the complete Sim inventory serializer.

## `Iv` line format

`SimsOnlineServerHostMod/InventorySet.cs` implements the compact set representation. The host publishes the owner ID, a sorted unique list of item IDs, and, for newer lines, a parallel item-type list:

```text
Iv|7787|7,9,30|PhoneSmart,?,BookGeneral
```

Fields are:

| Position | Meaning |
|---:|---|
| 0 | Ring sequence when parsed from a SACS ring line |
| 1 | Kind `Iv` |
| 2 | Owner object ID |
| 3 | Comma-separated item IDs; `-` means empty |
| 4 | Optional comma-separated type names in the same order |

`InventorySet.Membership` sorts IDs ascending and removes duplicates. When type names are available, it preserves the matching type for each sorted item. It substitutes `?` for an item whose type is unavailable and sanitizes comma and pipe characters to underscores so they do not corrupt the delimiter format.

The type list is optional for compatibility. A legacy line with no fifth field still parses membership, but provides no type-based runtime-object pairing. If the number of type names does not equal the number of parsed IDs, the parser discards the type list rather than pairing ambiguous entries.

## Producer cadence and scope

`ServerHostMain.PublishInventories(Dictionary<ulong, bool> targets, bool resend)` publishes inventories for selected Sim owners. It gathers the current item membership, compares the serialized line against the previous value, and emits only changed content unless `resend` is requested. The watch set is tied to presentation targets, not a promise to export every Sim or every household container.

`InventorySet.MaxItems` is `200`; sets beyond the configured supported boundary are not intended to be streamed as an unbounded line. The source comment explicitly calls out large inventories such as a full fridge as unsupported by this line format. This limit is independent of the 512 KiB household export bulk slot.

The stream prioritizes reliable menu/presentation alignment without serializing every inventory item repeatedly. It is not a continuously polled full inventory database.

## Client reconciliation

`SimsOnlineServerHostMod/Presentation/Inventories.cs` applies the set to the local copy of the owner:

1. Parse the line and resolve the owner through `PresentationApi.ReplayObject`.
2. Resolve known server item IDs to local objects.
3. Compare current and target membership with `InventorySet.Diff`.
4. Remove only objects previously named by the server, using `TryToRemove(item, false)`.
5. Pair a server-missing runtime item with an unnamed local item of the same type when the optional type list makes that mapping possible.
6. Add resolved items using `TryToAddTemporary` so a remote Sim's NPC inventory policy does not destroy or purge the object.
7. Count missing, in-use, refused and paired cases for diagnostics.

The mirror does not create an object from a type name alone. It uses existing local objects where identity mappings exist or type pairing can safely associate them. A server ID missing on the client and no matching local object increments a missing counter; it is not fabricated into an object.

| Counter from `Inventories.Report()` | Meaning |
|---|---|
| `lines` | Parsed inventory lines processed |
| `+` / `-` | Items added/removed from mirror membership |
| `inUse` | Inventory operation could not take an object currently in use |
| `missing` | Host item has no local object or safe type pair |
| `refused` | Local inventory refused an add/remove operation |
| `noOwner` | Owner ID did not resolve to an object with an inventory |
| `paired` | Runtime server item mapped to an unnamed local item by type |
| `ownKept` | Unnamed local items were not removed by a server set |

## Why removal is conservative

The presentation stream also represents visible attachment and object operations. Removing an inventory item by deleting it or forcibly moving it can conflict with the animation/slot stream. `TryToRemove(item, false)` changes membership without asking the inventory operation to drop the item into the world. The server is responsible for sending the item's visual state/parenting separately.

Only items previously named by a server inventory set are removed. A client's unrelated item that is not in the host set is treated as local/unnamed and kept unless it can be paired to a missing server item of the same type. This prevents a partial or old server line from deleting arbitrary local inventory contents.

An item being used on the client is not forcibly moved. The `inUse` count reports a failed membership operation, leaving reconciliation to a later set rather than tearing an active client interaction apart.

## Book identities and `bookId`

Books need a stable resource identity because host `ObjectGuid` values do not necessarily exist in another process. The historical `inventory-book-sync.md` investigation documents the `BookGeneral` path: the interaction parameter `bookId` identifies the book's data definition, whereas a runtime `ObjectId` identifies one instantiated object.

In the current host mod, terminal replay and read-intent handlers can resolve a book by data identity, look for a matching local inventory item, and in supported paths create a local `BookGeneral` instance from its definition. This is separate from the `Iv` membership mirror. The protocol's `InteractionQueued.Parameters` can carry semantic strings such as `bookId`; the public `SimulationSnapshot` has no inventory contents.

| Identifier | Stable across separate processes? | Use |
|---|---|---|
| `ObjectId` / `ObjectGuid` | Not guaranteed | Resolve an existing instance in one process or through an explicit alias |
| `bookId` | Definition-level identity | Locate the corresponding book data/object when supported |
| Runtime type name | Coarse class identity only | Last-resort pairing for selected membership presentation |
| Display name | Not guaranteed unique | User-facing label only |

A type match between two runtime items is not semantic equality. It is a limited presentation pairing strategy and should not be used to merge item contents or ownership.

## Durable payload inventory segments

`HouseholdExportPayload` carries optional `SimInventories`, `SharedFridgeInventory`, and `SharedFamilyInventory` records. Each segment contains its own property bytes and remap tables. `HouseholdExportTransport` parses their descriptor sequence from the bulk payload and verifies that the declared segment sizes sum to the reported total.

That transport is not the same thing as the live `Iv` mirror. The deep household import is quarantined because a successful in-memory import did not establish cold-load safety, and `HouseholdExportValidator` rejects unproven nonempty object-ID remapping. See [Household export/import](household-export-import.md) and [Lot reconciliation](lot-reconciliation.md).

## Common failure signatures

| Observation | Likely explanation |
|---|---|
| Book menu choice remains enabled after host takes book | Membership line absent/stale or target owner excluded from watch set |
| `missing` rises for runtime items | Host IDs do not resolve; type metadata absent/mismatched or no local counterpart |
| `ownKept` rises | Local inventory contains items not named in server set; conservative policy retains them |
| `inUse` rises | Client item is currently participating in a local interaction |
| Correct inventory count, wrong displayed object | Membership reconciled but object alias/visual stream did not |
| Book action targets wrong instance | `bookId`/identity resolution and runtime ID were conflated |
| Full fridge has no matching line | `MaxItems` and line-size constraints intentionally limit broad containers |

## Tests and source map

The source-linked tests in `DedicatedServer.Tests/Program.cs` include:

- `InventorySet.SendsOneOrderAndReadsItBack`
- `InventorySet.DiffsTheClientsSetAgainstTheServers`
- `InventorySet.CarriesEachItemsTypeAndPairsRuntimeItems`
- `BridgeLineCodec.EncodesApplyInteractionQueueWithReadBookProgress`
- `InteractionQueuedCoalescer.StripsBookPrecursorsWhenReadBookPresent`
- `TwoMirror.ClientsForeignJenniferBookChainUsesInjectReadIntent`

The first three pin the membership algorithm and delimiters. The book tests cover selected interaction parameter flow and target-resolution policy; they do not prove every inventory class or object type can be mirrored. `SimsOnlineServerHostMod/InventorySet.cs`, `SimsOnlineServerHostMod/Presentation/Inventories.cs`, and `SimsOnline.Protocol/LotExport.cs` are the current implementation entry points.

## Related pages

- [Household export and import](household-export-import.md)
- [Lot reconciliation](lot-reconciliation.md)
- [Snapshot fields](snapshot-fields.md)
- [Presentation stream](../presentation/presentation-stream.md)

## Set examples and diff semantics

Input item IDs may arrive in any order and may contain duplicates. `InventorySet.Membership` normalizes them before formatting. For example:

```csharp
var ids = new ulong[] { 30, 7, 9, 7 };
var types = new string[] { "BookGeneral", "PhoneSmart", "PhoneSmart", "BookGeneral" };
string line = InventorySet.Format(7787, ids, types);
```

The output membership order is ascending by item ID. A client set `{7, 9}` and a server set `{7, 30}` produces one add and one remove. The diff is set membership; it is not a transaction log, so it cannot tell whether an item was transferred through an intermediate inventory.

| Current client | Target server | Membership action |
|---|---|---|
| Item named by server | Also named | Keep |
| Item named by server | Not named | Remove from inventory membership |
| Item not named by server | Named, ID resolves | Add resolved object |
| Item not named by server | Named, ID missing, type match found | Pair local runtime instance where supported |
| Item not named by server | No ID or safe type match | Keep unrelated local item and count missing |

## Aliases and type pairing safeguards

Type pairing is deliberately a fallback for process-local runtime IDs. It is only available when type metadata aligns positionally with the normalized ID list. If the optional type list is missing or its count does not equal the IDs, the parser declines pairing rather than guessing.

Type names are not a unique identifier. Two phones or books of the same runtime type can have different contents, ownership or state. Pairing is useful to maintain a displayable membership set; it must not be used to merge hidden item properties or to choose which item a destructive command consumes. A semantically sensitive action should use a definition-level id or a verified alias.

## Inventory limits and incomplete observations

A published line can be absent because the owner was not in the watched target set, because the membership did not change, because the stream was not requested/resend, or because the supported item bound was exceeded. Absence of a line is not equivalent to an empty inventory.

`MaxItems` is 200 for the compact membership line. A broad inventory such as a full fridge may exceed that bound and is not supported as an unbounded line. The implementation uses bounded lines to avoid flooding the recorder ring and bridge. Durable shared containers instead belong to the native save path or the structured household payload (whose deep import is quarantined).

## Producer state and reset behavior

The publisher caches previous line content to avoid emitting the same membership on every poll. A forced resend is needed after a mirror reset or a new observer joins because a client that did not see the earlier line cannot reconstruct membership from a change-only stream. Reset the publisher cache consistently when changing owner scope; otherwise an identical line can be incorrectly suppressed for a newly interested client.

The consumer should clear its per-session mirrored inventory state when the local host is reset or the player switches to a different world. A stale set from a previous save can refer to runtime IDs that now belong to other objects.

## Targeted test cases for changes

In addition to existing normalization/diff/type-pairing tests, a change to `Iv` should cover:

- empty set encoded as `-` and parsed back as zero items;
- duplicate IDs with different input type positions;
- missing fifth field (legacy line) and mismatched type count;
- delimiter characters sanitized in type names;
- maximum supported item count and overflow behavior;
- forced resend versus unchanged-line suppression;
- owner missing or owner without inventory;
- local object in use during removal;
- item alias missing with and without type candidate;
- unrelated local item retained when target set is partial;
- clear/reset after a new client world is loaded.

The source-linked tests in `DedicatedServer.Tests/Program.cs` pin primary cases; changes should add regression cases where the behavior is new.

## Avoiding stale writes

A membership line can arrive after the local player has already moved an item. The mirror should avoid deleting an in-use object and should prefer a subsequent authoritative set to reconcile again. If a local player action is allowed to mutate an owned Sim's inventory, the authority policy must decide whether to forward that intent or reassert server membership; a presentation set alone cannot resolve the conflict.

The publisher's cached line is an optimization, not a durable cursor. A client rejoin, save reload, mirror reset or target-scope expansion can require a resend even when the host membership is unchanged. Check that the host forces an initial observation for new targets and that the client clears stale local state when its world changes.

## Distinguishing membership from item content

The `Iv` line has IDs and optional types, but no quality, charges, book progress, ownership flags, content strings, interaction state, or location beyond owner membership. If a UI shows a book in inventory but the book has the wrong progress, membership may be correct while the item's content was not synchronized. Do not expand `InventorySet` into an ad hoc property serializer; use an explicit typed contract or the native save path for durable content.

When a user reports “item missing,” ask which layer is absent: membership line, local object resolution, local inventory membership, visible attachment, or semantic item data. The counters and source paths above map to distinct repair points.

## Container ownership

The owner ID in the `Iv` line is the Sim object whose inventory is mirrored. It is not necessarily the household ID or the object ID of a shared container. Shared fridge/family inventory belongs to the deep export descriptor model, not to an arbitrary Sim owner line. A future general container mirror must define an explicit container identity and membership policy.
