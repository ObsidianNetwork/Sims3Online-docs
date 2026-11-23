# Road placement and authored road topology

`road_placement.rs` implements a bounded `SIMROAD01` file loader, canonical JSON compiler, and deterministic lane-placement query. It is an authored-data contract; neither the compiler nor the tests export or establish a complete copy of the game's private road database. This is distinct from the `SIMNAV01` pedestrian route planner described in [routing obligations](rust-kernel-obligations.md).

## Public data model

| Type | Important fields |
|---|---|
| `RoadSet` | `world_key: u64`, `content_hash: u64`, private ordered lane list and selected vehicle-lane component |
| `RoadView` | `eye_cm`, forward/up vectors in parts per million, `fov_mdeg`, `aspect_ppm`, near/far clip distances |
| `RoadPlaceRequest` | `obj`, `position_cm`, `lane_kind`, `distance_cm`, `radius_cm`, optional `view` |
| `RoadPlaceAnswer` | `found`, `position_cm`, `heading_mdeg`, `lane_id`, `onscreen`, `place_hash` |
| `RoadPlaceError` | `RoadDataMissing`, `LaneKindUnsupported`, `BadRequest` |

Coordinates are integer centimetres. The fixed binary header is 56 bytes; `RoadNode` is 16 bytes, `RoadLane` is 32 bytes, and `RoadVertex` is 12 bytes. `size_of` and `offset_of` compile-time assertions pin the record layout.

## Input format and limits

The loader `load_simroad` requires the 9-byte magic `SIMROAD01`, zeroed reserved bytes, version 1, flags 0, reserved word 0, and a matching FNV-1a 64-bit content hash computed with the hash field treated as zero. File length must exactly equal the header plus all declared records.

| Validation | Current bound/rule |
|---|---|
| Nodes | At most 65,536, strictly increasing nonzero IDs, coordinates within ±1,000,000 cm |
| Lanes | At most 65,536, strictly increasing IDs; supported lane kind 0, 1, 2, 3, or 5 |
| Vertices | At most 1,000,000 total; every lane has at least 2 contiguous vertices |
| Lane width | 1 through 200,000 cm |
| Lane flags | Only bits 0–1 may be set; reserved bytes/word must be zero |
| Endpoints | `from`/`to` nodes exist and match first/last lane vertices |
| Geometry | Bounded coordinates; adjacent vertices may not have identical X and Z |

The largest weakly connected component of kind-0 lanes carrying flag bit 1 (the lane's `flags & 2 != 0` test) is selected for vehicle placement. Equal-size component ties retain the component reached from the lowest ordered root, because map/set traversal is ordered and only a strictly larger component replaces the current selection.

## JSON authoring and compile command

`compile_simroad_json` accepts only objects, arrays, and canonical integer values. It rejects unknown or duplicate keys, fractional values, excess nesting, and trailing data. The top-level object must contain exactly `world_key`, `nodes`, and `lanes`. Node objects require `id` and `position_cm`; lane objects require `id`, `kind`, `flags`, `from`, `to`, `width_cm`, and `vertices_cm`. The compiler sorts nodes and lanes by id before writing bytes and computes the content hash.

```powershell
cargo run --manifest-path rust/sim96_kernel_cleanroom/Cargo.toml --bin simroad_compile -- authored.simroad.json authored.simroad
```

The tool prints `content_hash=<16 lowercase hex digits>` on success. A read/write failure exits 1; rejected JSON exits 2. The source comment explicitly limits this utility to authored JSON and says it does not read game packages or saves.

## Placement request validation

`place_on_road_offscreen` returns a `RoadPlaceError` for malformed numeric input, unsupported requested lane kinds above 5, or missing/empty road data. A valid request with no eligible sample returns `Ok(RoadPlaceAnswer { found: false, ... })`; “no candidate” is not a separate error variant.

The request rejects coordinates outside the same ±1,000,000 cm bound, `distance_cm` outside 0–1,000,000, `radius_cm` outside 0–100,000, nonzero radius when `obj == 0`, and an invalid supplied view. View validation checks normalized direction/up vectors in ppm, near < far, `far_cm <= 4,000,000`, FOV in 1..180,000 millidegrees, and aspect in 1..=10,000,000 ppm.

For lane matching, requested kind 4 is a selector for lanes of kind 2 or 3. Other kinds match exactly. A candidate is rejected when twice the requested radius exceeds lane width. A kind-0 request additionally requires the caller position to be within the width corridor of a lane in the selected vehicle component.

## Deterministic candidate selection

The minimum placement distance is 100 cm. Each segment is sampled every 100 cm and includes its final endpoint. The distance error from the requested location must be within ±100 cm of the requested distance. When a view is supplied, `visible` tests the candidate against the supplied frustum.

The winner key is `(onscreen, error, lane_id, arc_distance)`, compared lexicographically in ascending order. Since `false < true`, a candidate considered off-screen is preferred to an on-screen one; ties are broken by smaller distance error, then lane id, then distance along the lane. There is no separate “fallback pass” in the current implementation. With no view, every candidate has `onscreen=false`, so no claim about an actual player's camera can be made.

The answer heading is derived from the segment's X/Z delta and normalized to 0–359,999 millidegrees. `place_hash` hashes the `found` bit, position, heading, lane id, and on-screen bit; it is a reproducibility aid, not a signature.

## Tests and verification

Focused tests are in `rust/sim96_kernel_cleanroom/src/road_placement.rs`: `canary_33_golden_cases`, `loader_rejects_noncanonical_tables_and_corruption`, `compiler_canonicalizes_and_rejects_bad_authored_input`, `component_uses_lane_count_then_smallest_node_and_weak_connectivity`, `sampling_uses_outgoing_segments_final_endpoint_and_three_dimensions`, and `request_precedence_and_frustum_limits`.

Run the crate test suite with:

```powershell
cargo test --manifest-path rust/sim96_kernel_cleanroom/Cargo.toml
```

These tests exercise the authored format and deterministic rule implementation. They do not prove parity with all native road networks, vehicles, service arrivals, or camera behavior.

## Failure modes and limits

- A header/count/hash/reference error rejects the road file as `BadRequest`; it is not silently repaired.
- Missing road data is distinct from a valid road set with no candidate.
- Lane type 4 is a query alias, not a stored lane kind.
- View-dependent off-screen status is only as meaningful as the caller-provided `RoadView`.
- The road graph and pedestrian navmesh are separate datasets and answer different questions.
- A deterministic fixture result is not native-engine equivalence.

## Related pages

- [Rust kernel overview](rust-kernel-overview.md)
- [Routing obligations](rust-kernel-obligations.md)
- [Kernel pilot](rust-kernel-pilot.md)
- [Open problems](open-problems.md)
