# Rust server front door

The Rust front-door crate is no longer only a proposed topology: `simlink_headless_server` contains a TCP listener, v1 codec, per-connection session state, streams, admin surface, save/rejoin modules, and a bridge driver. Its README still calls the server experimental and not a supported multiplayer release. The current C# controller remains the established documented player path; this page describes the Rust code, not a cutover claim.

## Startup modes and listener posture

`src/main.rs` provides `--codec-stdio`, `--trace-stdio`, scorecard emission, bind-only probing, and resident/live operation. Resident operation is selected by `--serve`, `--resident`, `--fake-driver`, `--pid <pid>`, or `--attach-ts3w`; it requires a driver choice. `--fake-driver` cannot be combined with a live attach. `--attach-ts3w` is Windows-only and requires exactly one `TS3W.exe`; otherwise choose a process with `--pid`.

`--public <host>:<port>` is the only public bind opt-in. The default listener is `127.0.0.1` at `DEFAULT_WAN_PORT`. `--connections` defaults to 32. A non-loopback listener prints an exposure warning because this protocol has no authentication. The warning is not a security control: do not run the experiment on an untrusted network.

```powershell
cargo run --manifest-path rust/simlink_headless_server/Cargo.toml -- --bind-only
cargo run --manifest-path rust/simlink_headless_server/Cargo.toml -- --serve --fake-driver
```

The first command probes the default bind posture and exits. The second keeps the server resident with an offline driver; it is not a live-game deployment. The process accepts `QUIT` on stdin or Ctrl-C for shutdown.

## Layers and ownership

| Layer | Current implementation | Does not establish |
|---|---|---|
| Listener and transport | `bind.rs`, `serve.rs`, `transport.rs` | Authenticated internet service |
| Protocol | `messages.rs`, `wire_v1.rs` | Compatibility with the C# JSON protocol |
| Session | `session.rs` | Native game simulation |
| Per-client stream | `stream.rs`, `cadence.rs`, `events.rs` | Client render parity |
| Game driver | `bridge_driver.rs`, `simlink_bridge` | That every bridge verb is supported by every mod build |
| Kernel rules | `sim96_kernel_cleanroom` | Full native engine replacement |

The public Rust line protocol is version 1; the process-memory bridge contract is separately versioned (layout 11, protocol 23, HELLO 6). Public and bridge framing must not be conflated.

## Example command lifecycle

A public command is decoded into a `Command` record with `corr`, `verb`, `command_id`, `player_id`, and remaining `args`. The session gate checks whether the connection is joined, whether its player matches, whether the verb is allowed, and whether sequence/idempotency rules permit a new command. The semantic handler invokes the configured driver or kernel rule. The reply is cached for idempotent replay, while resulting events travel on the event stream with a sequence separate from `corr`.

```text
CMD|18|JoinSession|9001|7|<join-args>
ACK|18|JoinSession|...
CMD|19|ClaimSim|9002|7|<sim-id>
ACK|19|ClaimSim|...
EV|1|SimClaimed|...
```

Field details and acceptance rules belong to `wire_v1.rs`/`messages.rs`; the snippet indicates ordering rather than an application-ready join body. The tests prove that command correlation and event sequence are separate, a new `corr` must increase, and a duplicate command id replays the original reply bytes.

## IDs and replay

The public wire carries `u64` IDs, but kernel `SimId` is `u32`. `IdMap` maintains maps in both directions and allocates kernel ids in first-seen order; it skips the reserved `SERVER_PRIMARY_SIM` id and returns `IdSpaceFull` at exhaustion. This avoids silently truncating public identity.

Each `Connection` holds a 32-entry idempotency window (`COMMAND_ID_WINDOW`). The cache is bounded: once an old id is evicted, the byte-for-byte replay guarantee no longer applies. Correlation order, event `seq`, and command idempotency are separate state; a timeout after bridge dispatch may still leave an uncertain game-side outcome.

The test suite names these contracts directly in `rust/simlink_headless_server/tests/session.rs`: `replayed_command_id_returns_the_original_reply_byte_for_byte`, `seq_is_strictly_increasing_per_client_within_the_epoch`, `a_restart_starts_a_new_epoch`, `a_new_command_must_have_a_strictly_increasing_corr`, and `a_rejoin_supersedes_the_earlier_connection`.

## Bridge and game dependency

The game driver can attach to a PID through `BridgeGameDriver::attach_pid` or use `OfflineBridge` for fake/offline operation. The attached path communicates through `simlink_bridge`; it does not simulate Sims in Rust. `TS3W.exe` remains the gameplay engine for behavior not represented by explicit kernel contracts.

Layout constants are independently declared in the C# and Rust bridge implementations. Their current values are summarized in [kernel overview](rust-kernel-overview.md). The fake-mod suite exercises framing and slot state without requiring an actual process; live attach and game behavior require a separate controlled scenario.

## Failure modes and limits

- Bind-only success does not mean a resident game session is ready.
- A process can bind publicly, but the protocol lacks authentication and must not be treated as safe for untrusted access.
- Unsupported CLI arguments are printed as ignored warnings; verify the exact effective bind and driver in startup output.
- A bridge timeout after command dispatch can leave the result unknown; blindly resending a non-idempotent game action can duplicate effects.
- A successful `HELLO` does not prove that every verb, snapshot shape, or presentation record is compatible.
- Passing Rust tests does not establish a default-path migration or supported release.

## Related pages

- [Rust kernel overview](rust-kernel-overview.md)
- [Kernel ownership](rust-kernel-ownership.md)
- [Transport decision](decision-transport.md)
- [Protocol](../protocol.md)
- [Security](../security.md)
