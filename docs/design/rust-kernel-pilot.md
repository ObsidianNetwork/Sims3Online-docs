# Rust kernel pilot: what the early result did and did not show

This page separates a historical first pilot from the present Rust source tree. The original pilot report records a failed whole-crate check: the `routing` module was missing in that revision, and Clippy could not run because the toolchain component was absent. Its conclusion was therefore **not accepted as a completed kernel pilot**. A narrower standalone integer seed/path test did not turn the whole Cargo suite into a pass.

That report is historical, not the current checkout status. The tracked kernel now includes `routing_astar.rs`, `road_placement.rs`, `handles.rs`, and a default `kernel_default.rs`; optional private oracle modules are gated by `private-oracles`. The previous absent-module failure must not be copied as a present build defect, and an earlier isolated test must not be promoted into an end-to-end acceptance claim.

## What the initial pilot tried to isolate

The early work asked whether selected engine-facing contracts could be expressed as pure Rust interfaces and tested independently. The durable engineering lesson was to separate four evidence levels:

| Evidence | What it shows | What it does not show |
|---|---|---|
| Source/interface inspection | A type or function has a stated shape | It compiles or matches native behavior |
| Unit test | A bounded case satisfies an assertion | The whole crate builds or a live game agrees |
| Whole-crate build/test | Tracked crate components compile and run together | A live bridge/game session works |
| Live differential | Behavior agrees for the captured scenario/build | Universal lot, interaction, or version parity |

A missing local oracle or unavailable linter is a failed/unrun gate, not evidence that the implementation is correct. The original report was right to distinguish those outcomes.

## Current kernel surface

`rust/sim96_kernel_cleanroom/src/lib.rs` now declares the kernel modules and selects default implementations for `KernelRng` and `KernelSlots` when private-oracle modules are not compiled. The crate forbids unsafe code; its manifest denies the Clippy `all` lint group. `routing_astar.rs` implements authored `SIMNAV01` loading and deterministic A*, while `road_placement.rs` handles authored `SIMROAD01` lanes. These are separate contracts and neither is a full native engine replacement.

The current tests name the covered contracts directly. For example, `handles.rs` checks `generation_reuse_rejects_every_stale_access`; `routing_astar.rs` checks `tie_break_is_strict_and_reproducible`, `water_is_blocked_unless_allowed_then_doubled`, and `loader_is_panic_free_on_mutations`; `road_placement.rs` checks malformed data, canonical compilation, candidate ordering, and view constraints. See [routing obligations](rust-kernel-obligations.md) and [road placement](rust-road-placement.md) for precise inputs and limits.

## Re-run the current gate

From the repository root:

```powershell
cargo test --manifest-path rust/sim96_kernel_cleanroom/Cargo.toml
```

This command is the current crate test gate. A result from an earlier checkout, a standalone module test, or a stale internal report must not be substituted for a fresh result. Clippy is a separate command and only counts as run when the toolchain component is installed and the command actually completes.

## Failure modes and limits

- Historical pilot failure does not prove the current crate is broken; the source tree has since changed.
- Current unit tests do not establish native route parity or complete interaction coverage.
- The kernel is not the default public multiplayer server path.
- A clean build does not by itself establish the bridge's live process-memory behavior.
- The server and bridge crate READMEs continue to mark those components experimental and unsupported for player release.

## Related pages

- [Kernel overview](rust-kernel-overview.md)
- [Kernel ownership](rust-kernel-ownership.md)
- [Routing obligations](rust-kernel-obligations.md)
- [Road placement](rust-road-placement.md)
- [Architecture options](rust-architecture-options.md)
