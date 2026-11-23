# Rust navigation contract and verification limits

The clean-room planner in `rust/sim96_kernel_cleanroom/src/routing_astar.rs` separates authored nav topology from the game engine. It defines a bounded `SIMNAV01` format, maps it to a graph, and returns deterministic route plans for that input. Its comments identify native route geometry as an oracle that remains necessary for live differential; fixture agreement does not make the game engine unnecessary.

## `SIMNAV01` records

The file begins with a 64-byte `NavHeader`: magic `SIMNAV01`, format version 1, flags, `lot_id`, `world_seed`, counts for rooms/cells/portals/obstacle templates/vertices, a zero-reserved word, and a 64-bit content hash. The parser reads little-endian records and checks record sizes with compile-time layout assertions.

| Record | Size | Contract fields |
|---|---:|---|
| `NavRoom` | 16 bytes | `room_id`, `level`, portal range, cell range, zero-reserved field |
| `NavCell` | 32 bytes | `cell_id`, room id, vertex/portal ranges, centroid in cm, flags |
| `NavPortal` | 48 bytes | adjacent cells, unsigned `path_type`, door id, jambs, Y, width, lock key |
| `ObstacleTemplate` | 32 bytes | authored object type key, min/max bounds, scale ×1000 |
| `NavVertex` | 12 bytes | X/Y/Z integer centimetres |

`NavHeader.flags` permits `NAV_FLAG_HAS_OBSTACLE_TEMPLATES`, `NAV_FLAG_MULTI_LOT`, and `NAV_FLAG_PORTALS_BIDIRECTIONAL`; bidirectional portals must be set. Cells are limited to 65,536, portals to 65,536, and a loaded nav must contain at least one cell. Each record table must fit the exact file length. IDs use dense canonical ordering; references and ranges are validated. `content_hash` uses FNV-1a 64 with the hash bytes zeroed. Malformed format returns `KernelError::NavFormatUnsupported`; a bad content hash returns `NavContentHashMismatch`.

Loaded polygons must have 3–8 vertices, canonical counter-clockwise convex order, and a centroid matching the stored centroid. Only `CELL_NO_SPAWN`, `CELL_TERRAIN_WATER`, and `CELL_OUT_OF_BOUNDS` are valid cell flags. Obstacle templates require `scale_x1000 == 1000`. These checks reject malformed or ambiguous data rather than guessing a nearest valid interpretation.

## Planning inputs and costs

`NavGraph::plan_route(start_cell, goal_cell, sim_id)` plans with no blockers or water permission. `plan_route_blocked` adds a `FootprintBlocker`; `plan_route_with` additionally accepts flags, currently only `FLAG_ALLOW_WATER` (bit 0). Any other request flag returns `NavFormatUnsupported`.

The planner uses integer millimetre costs and three-dimensional octile distance (`ast_octile_mm`) over cell centroids. A `PortalEdge.base_cost_mm` below its geometric lower bound is rejected by `ast_validate_edges`; the A* heuristic is the octile lower bound, and the 250-mm door surcharge applies to edge cost, not the heuristic. Water is not traversable unless `FLAG_ALLOW_WATER` is set; allowed water edges are multiplied by `WATER_COST_MULT_X1000 == 2000` (2×). The priority order is ascending `(f_cost, g_cost, cell_id)`, and adjacency is sorted by portal id for deterministic ties.

A portal with the high bit set in unsigned `path_type` is treated as a door. The `lock_key` is resolved against held keys when `SimNav::to_graph` constructs the graph; a locked door is not silently traversed. Ordinary edges emit `SubPath`; a door crossing emits `Approach`, `Through`, and `Exit`. `RoutePlan` carries `cells`, `total_cost_mm`, ordered door crossings, legs, and `path_hash`. `route_path_hash` hashes the emitted route shape, excluding `cost_mm`.

## Dynamic footprint blockers

`FootprintBlocker::from_obstacles` expands authored AABBs by `HALF_WIDTH_CM == 25` on each side and tests cell centroids/vertices. `footprint_blocked_cells` maps cells from the lot-edit registry onto nav cells by lot and grid coordinates; cells not present in the graph block nothing. `plan_route_blocked` rejects a blocked start as `RouteStartOutOfBounds`; a blocked goal is deliberately allowed to remain reachable, according to the current source comment.

The obstacle path is explicit but limited: templates use a fixed scale of 1000; unsupported scale is a format refusal rather than a live scaling approximation. The graph and blockers must agree on coordinate origins and units.

## Route output boundary

`NavGraph::to_path_data` converts a plan into `kernel_iface::PathData`, whose `path_type` remains `u32` and whose `portal_bits` is `[u32; 3]`. The adapter uses the named boundary rather than exposing internal graph records as a game API. `boundary_route_and_seat_handoff` returns a `BoundaryRoute` plus a separate seat-handoff trailer. `SeatHandoff` is not part of `RoutePlan::legs` or its hash.

Named refusal values include `RouteStartOutOfBounds`, `RouteGoalUnreachable`, `PortalLocked`, `NavFormatUnsupported`, and `NavContentHashMismatch`. A caller should preserve these distinctions; a zeroed route would otherwise hide whether the graph was invalid, the start was invalid, or the goal was unreachable.

## Authored compiler

The `simnav_compile` binary accepts exactly an input and output path:

```powershell
cargo run --manifest-path rust/sim96_kernel_cleanroom/Cargo.toml --bin simnav_compile -- authored.simnav.json authored.simnav
```

It calls `SimNav::from_canonical_json`, serializes with `to_bytes`, and reloads with `SimNav::load` before writing. Success prints `content_hash=<16 hex digits>`. Usage or I/O errors exit 1; rejected content exits 2. Source comments limit it to authored fixtures; it does not read game files. The original route-design record established deterministic planning and route-leg requirements; this implementation is the current code contract, not an extracted native navmesh.

## Tests that define current coverage

`routing_astar.rs` contains focused unit tests including `sunset_fixture_routes`, `tie_break_is_strict_and_reproducible`, `water_is_blocked_unless_allowed_then_doubled`, `simnav_content_hash_mismatch_is_named`, `loaded_door_matches_the_kdiff_door_row`, `astar_dynamic_footprint_blocks`, `loader_is_panic_free_on_mutations`, and `kdiff_canary31_fuzz_10000`. `tests/live_navigation.rs` tests `cross_lot_requires_explicit_physical_connector_and_actor_mapping`, `overlapping_interiors_fail_instead_of_choosing_nearest`, and atomic replacement/binding behavior.

Run the crate suite with:

```powershell
cargo test --manifest-path rust/sim96_kernel_cleanroom/Cargo.toml
```

These are authored topology and kernel tests. They do not prove that the authored geometry matches every game lot or that a native route solver and the clean-room planner produce equivalent results for all interactions.

## Failure modes and limits

- A content hash proves internal integrity, not origin or trust.
- A valid authored graph may still omit a native obstacle, door rule, or route case.
- A deterministic tie-break proves repeatability for identical inputs, not native route optimality.
- A game-facing adapter must preserve unsigned `path_type` values; signed conversion can lose the high-bit door class.
- A bridge success does not establish that the game's live navigation contract matches the authored planner.

## Related pages

- [Rust kernel overview](rust-kernel-overview.md)
- [Road placement](rust-road-placement.md)
- [Kernel pilot](rust-kernel-pilot.md)
- [Divergence forensics](rust-divergence-forensics.md)
- [Clean-room feasibility](clean-room-feasibility.md)
