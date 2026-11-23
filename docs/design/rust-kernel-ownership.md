# Rust state and identity ownership

The Rust tree uses several identity layers. The kernel's typed IDs, generic generation handles, public protocol IDs, and game-process object ids are separate contracts and must not be collapsed into a single “64-bit handle” description.

## Kernel ID wrappers

`rust/sim96_kernel_cleanroom/src/lib.rs` defines these wrappers:

| Type | Width | Intended role |
|---|---:|---|
| `ObjectGuid` | `u64` | Object GUID value |
| `SimId` | `u32` | Kernel Sim/arena identity |
| `LotId` | `u32` | Typed lot handle |
| `ObjectId` | `u64` | Typed object handle wrapper |

`SimArena<T>` packs a 16-bit slot index into the low bits and a 16-bit generation into the high bits of `SimId`. `insert` reuses a vacant slot only while its generation is below `u16::MAX`; `remove` saturating-increments the generation. A slot reaching the maximum generation is retired. `get` and `get_mut` return `None` for out-of-range, stale, or vacant handles.

`LotArena<T>` wraps `SimArena<T>` with `LotId`. `ObjectArena<T>` wraps the same arena with `ObjectId(u64::from(sim_id.0))`; reads first convert the `ObjectId` back to `u32`, so values above `u32::MAX` cannot resolve. This is a typed wrapper, not a separate 64-bit object allocation space.

## Generic generation handles are a separate API

`handles.rs` defines `GenHandle { index: u32, generation: u32 }`, with `#[repr(C)]` and compile-time size/offset assertions: size 8, `index` at offset 0, `generation` at offset 4. `HandleArena<T>` returns `Result` values and distinguishes unknown indices (`UnknownHandle`) from stale or vacant generations (`StaleObjectHandle`). The implementation retires a slot at `u32::MAX` rather than allowing generation wraparound.

The two models have different wire and storage shapes:

| Model | Packed? | Generation width | Stale lookup result |
|---|---:|---:|---|
| `SimArena<T>` / `SimId` | Yes, index+generation in one `u32` | 16 bits | `Option::None` |
| `HandleArena<T>` / `GenHandle` | No, two `u32` fields | 32 bits | `Err(KernelError::StaleObjectHandle)` |

Do not serialize one as the other or assume a `GenHandle` can be converted to `SimId` without checking the representation.

## Public wire id mapping

The Rust server accepts public `u64` IDs and uses `simlink_headless_server::IdMap` to allocate kernel `SimId(u32)` values. It keeps both `wire_to_kernel` and `kernel_to_wire` maps. The primary server Sim id is reserved and skipped; allocations are assigned in first-seen order. Exhaustion returns the named refusal `IdSpaceFull` rather than truncating an id.

A stable map is process-local state. `Server::new(epoch)` starts a new session server and its `IdMap`; do not treat a kernel id as a durable identity across process restarts. The epoch and explicit wire ID are needed to distinguish a new runtime lifetime from a stale local handle.

## Arena examples

For the `SimArena` test, removing a handle and inserting another value may reuse the same slot index, but increments the generation so the old id no longer resolves. For `HandleArena`, the stale operation is an explicit error:

```rust
let old = arena.insert("old")?;
arena.remove(old)?;
let new = arena.insert("new")?;
assert_eq!(arena.get(old), Err(KernelError::StaleObjectHandle));
assert_eq!(arena.get(new), Ok(&"new"));
```

This abbreviated example matches the `HandleArena` API. It does not imply that all higher-level state transitions are atomic or that game object handles are process-independent.

## Tests and edge cases

The kernel unit tests in `lib.rs` include `stale_handle_does_not_alias_reused_slot`, `generation_exhaustion_retires_slot`, `unknown_handle_is_absent`, and `typed_lot_and_object_handles_also_reject_stale_ids`. `handles.rs` includes `generation_reuse_rejects_every_stale_access` and `exhausted_generation_never_wraps_into_an_old_identity`. The server's `tests/session.rs` includes `idmap_is_injective`, `idmap_assigns_kernel_ids_in_first_seen_order_reserving_the_primary_sim`, and `a_full_idmap_is_refused_id_space_full`.

These tests cover local identity safety and mapping behavior. They do not prove that a saved-game identifier is stable across independent game builds, that every engine object has a semantic alias, or that a public client can use a kernel handle directly.

## Failure modes and limits

- Reusing a raw index without checking generation can alias a newer value.
- Assuming `ObjectId(u64)` allocates all 64 bits overlooks its `SimArena` conversion.
- Treating kernel IDs as durable across an epoch/restart risks stale references.
- A `u64` public ID outside the `IdMap` has no kernel identity until explicitly mapped.
- A numeric id is not authorization; command policy still checks connection/player and ownership.

## Related pages

- [Rust kernel overview](rust-kernel-overview.md)
- [Rust server front door](rust-kernel-front-door.md)
- [Authority and ownership](../architecture-deep-dive/authority-and-ownership.md)
- [Protocol](../protocol.md)
