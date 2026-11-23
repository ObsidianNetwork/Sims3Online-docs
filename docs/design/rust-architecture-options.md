# Rust architecture options and current boundary

The design records considered several boundaries for moving controller responsibilities out of the C# runtime. The current repository now contains an experimental Rust front door, so options should be read as architectural comparisons—not as a list of unimplemented proposals or a release migration plan.

## Responsibility options

| Shape | Rust owns | Game process owns | Main constraint |
|---|---|---|---|
| Clean-room kernel library | Explicit pure rules and typed state helpers | Full simulation, native services, presentation and saves | A module can exist without being wired into a runtime path. |
| Rust process-memory bridge | Discovery and bridge frame exchange | Game behavior and mod-side handlers | Layout/version conformance and platform-specific memory access. |
| Rust front door | Experimental TCP listener, session state, codec, streams and bridge driver | Native gameplay behavior not replaced by a kernel contract | Must prove full join, recovery, save and client compatibility before being a supported path. |
| Full simulation replacement | All world/gameplay behavior | Client rendering only | Requires broad behavioral parity not established by current tests. |

The tested Rust server deliberately keeps a game-process driver in its architecture. It is not a standalone simulation engine. `simlink_headless_server` and `simlink_bridge` READMEs mark the crates experimental and not supported player releases.

## Why split the layers

A clean-room kernel can make an explicit rule deterministic and testable without loading the game. A bridge can isolate process-memory operations. A front door can own the experimental network/session boundary. Those are different improvements: a passing A* fixture does not establish a correct save system; an accepted bridge frame does not establish a playable session; a bound TCP port does not mean the service is secure or ready.

Dependency direction is part of the design. `simlink_headless_server` depends on `simlink_bridge` and `sim96_kernel_cleanroom`; the kernel has no dependency on either. `simlink_bridge` confines unsafe code to `ffi`, while the kernel and server forbid unsafe code.

## Current code and migration boundary

The Rust front-door binary has `--bind-only`, `--serve`/`--resident`, `--fake-driver`, `--pid`, and Windows-only `--attach-ts3w` paths. The public bind defaults to loopback; `--public` opts into another address and prints an exposure warning. `--connections` defaults to 32. The public line protocol is version 1 and distinct from the existing C# JSON protocol and the layout-v11 game bridge.

The server's `session.rs` owns per-connection ordering, idempotency, join/supersede state, and an id map. These are real code surfaces. However, the supported C# player path and the Rust experiment must not be described as if cutover already occurred. The `simlink_headless_server/README.md` explicitly says it is not a supported multiplayer release.

## One-writer rule

During any transition, designate one mutating owner for each state. If Rust compares against the C# implementation, the C# side is an oracle for that comparison, not a simultaneous writer to the same claim or gameplay operation. A timeout after bridge dispatch may leave the game-side outcome uncertain; idempotency can replay a cached reply but cannot make every native side effect automatically safe to repeat.

| Boundary | Current responsibility | Evidence to require before relying on migration |
|---|---|---|
| Public wire | Separate v1 codec/session | C# client interoperability and malformed-input tests |
| Game bridge | Layout-v11 Rust/C# implementations | Live handshake and verb-by-verb compatibility |
| Kernel rule | Explicit Rust module | Unit contract plus integration into the server path |
| Save/rejoin | Server-side experimental modules | Restart and interrupted-save recovery scenarios |
| Presentation | Client apply plus recorded game data | Representative live visual scenarios |

## Failure modes and limits

- A static kernel module is mistaken for a path reached by the live server.
- A new listener is described as a full headless simulator despite the game-process dependency.
- Two implementations both mutate claims or execute the same command.
- A green fake-driver test is presented as live bridge acceptance.
- A loopback-safe default is mistaken for authenticated public-network operation.

## Related pages

- [Kernel overview](rust-kernel-overview.md)
- [Front door](rust-kernel-front-door.md)
- [Kernel pilot](rust-kernel-pilot.md)
- [Clean-room feasibility](clean-room-feasibility.md)
- [Status](../status.md)
