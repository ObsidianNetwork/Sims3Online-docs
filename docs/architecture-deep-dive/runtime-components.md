# Runtime components

The checked-in C# runtime consists of session controllers plus in-game integration in each Sims 3 process. The Rust crates are an experimental parallel implementation; their existence does not make them the default player path.

## Process and contract map

| Component | Current responsibility | Primary source |
|---|---|---|
| `DedicatedServer/` | Server session, connection identity, command authorization, host orchestration, mirror publication | `DedicatedServer/Authority/AuthoritativeSession.cs` |
| `SimsOnline.Client/` | Client session, intent forwarding, server event/snapshot apply | `SimsOnline.Client/` |
| `SimsOnline.Protocol/` | Typed public client/server commands and events | `SimsOnline.Protocol/` |
| `DedicatedServer/HostBridge/` | Local controller-to-mod command and reply codec, process-memory region | `DedicatedServer/HostBridge/BridgeLineCodec.cs`, `BridgeSharedRegionLayout.cs` |
| `SimsOnlineServerHostMod/` | In-process command handlers, state extraction, recorder installation | `SimsOnlineServerHostMod/ServerHostMain.cs` and recorder classes |
| `DedicatedServer.Tests/` | Managed contracts for session, codecs, policy and process boundaries | `DedicatedServer.Tests/Program.cs` and focused test files |
| `rust/sim96_kernel_cleanroom/` | Pure/clean-room kernel experiments | `Cargo.toml`, `src/lib.rs` |
| `rust/simlink_bridge/` | Experimental process-memory bridge | `README.md`, `src/layout.rs`, `src/driver.rs` |
| `rust/simlink_headless_server/` | Experimental Rust listener/session/server | `README.md`, `src/main.rs`, `src/session.rs` |

## Two unrelated protocols

The public client/server protocol and the local game bridge are separate interfaces. Public message types are declared in `SimsOnline.Protocol/`; bridge verbs are encoded by `BridgeLineCodec` and handled in the in-game mod. A client JSON command such as `queue_interaction` is not itself a bridge command: session validation and translation occur between those contracts.

The current C# bridge copies its layout constants in both `DedicatedServer/HostBridge/BridgeSharedRegionLayout.cs` and `SimsOnlineServerHostMod/BridgeSharedRegionLayout.cs`; the Rust bridge mirrors the values in `rust/simlink_bridge/src/layout.rs`. Current values are:

| Field | Value |
|---|---:|
| `LayoutVersion` / Rust `LAYOUT_VERSION` | 11 |
| `ProtocolVersion` / Rust `BRIDGE_WIRE_PROTOCOL_VERSION` | 23 |
| `HelloWireVersion` / Rust `HELLO_WIRE_VERSION` | 6 |
| Control slot count | 16 total: 8 host-to-controller, 8 controller-to-host |
| Control slot size | 4096 bytes, 16-byte slot header, 4080-byte payload |
| Bulk slot count and size | 4 slots × 524288 bytes, each with a 32-byte slot header |
| Full region | 2,162,816 bytes |

The bridge layout uses a 64-byte control header and a separate 64-byte bulk-region header. The capability string is `multi-slot=16,bulk=4x512k,layout=11,snapshot2=1`. These values are duplicated contracts: a one-sided change is incompatible even if the public network protocol is unchanged.

## Authority and mirror responsibilities

`AuthoritativeSession` owns joined-player/session facts and the `_claims` map. Its command handlers check the player/actor relationship before forwarding an allowed command to the server game. The game remains responsible for running interaction validity and producing the outcome. The client mirror consumes server state; it is not a second authority.

`SimulationScopePolicy`, `MirroringInterestScope`, and `ClientAuthorityPolicy` have distinct jobs: simulation workload, per-client visibility, and local client gate configuration. See [authority and ownership](authority-and-ownership.md) and [scope hooks](scope-hooks-and-authority-modes.md) for exact behavior.

## Change ownership

| Change | Update together |
|---|---|
| Public message field or event | Protocol type, client/server codec/handler, wire tests, `docs/protocol.md` |
| Local bridge verb | `BridgeCommand`, `BridgeLineCodec`, host handler, capability behavior and tests |
| Shared-memory layout | C# layout copies, Rust `layout.rs`, bridge/mod consumers, layout and fake-mod tests |
| Snapshot field | Host extraction, bridge decoder, protocol record, mirror/filter path and compatibility tests |
| Presentation recorder | Host recorder, transport path, client replay decoder/apply and representative scenario |
| Authority/claim rule | `AuthoritativeSession`, pure policy, game hook integration, authorization regression tests |

Keep public wire compatibility tests distinct from shared-region/layout tests. A successful codec round-trip says nothing about a live game hook or client-side replay.

## Focused source tests

`DedicatedServer.Tests/Program.cs` includes `BridgeLineCodec.EncodesSetSimScope`, `SimScopeWire.FormatsAndParsesIdLists`, `SimulationScopePolicy.ComputesSortedClaimedLotsAndSims`, and `ClientAuthorityPolicy.NativeSetFollowsLotMode`. The Rust bridge has `rust/simlink_bridge/tests/driver_fake_mod.rs`, including `a_compat_write_keeps_the_v11_control_header` and `an_oversized_frame_is_refused_before_any_write`. These tests cover managed or simulated-memory behavior, not a live process compatibility matrix.

## Failure modes and limits

- Controllers may be responsive while a game process or bridge is stalled.
- A valid public command can still be rejected by the game when its own tests run.
- A missing recorder affects presentation without necessarily changing server world state.
- Layout drift can break local communication while the public TCP protocol remains valid.
- Rust README status remains experimental; compilation or tests do not establish supported player-release behavior.

## Related pages

- [Process model](process-model.md)
- [Boundary seams](boundary-seams.md)
- [Components](../components.md)
- [Bridge and formats](../bridge-and-formats.md)
- [Rust kernel overview](../design/rust-kernel-overview.md)
