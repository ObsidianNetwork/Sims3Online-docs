# Persistence and world time

## Checkpointing

A `save_game` command asks the host game to save through the local game bridge. When successful,
the session emits `save_checkpointed`. This event reports the save operation's result; it is not
a promise of transactional rollback or protection against every game-process or filesystem
failure.

When client-native lot authority is enabled, the checkpoint gate requires a fresh, merged
export for each client-native player. `LotReconcileTracker` stores the latest export per
player, not a complete reconciliation record for every lot they own. Each bundle declares
one lot; the established merge applies selected owned Sims' positions, not deep household
state. See [lot reconciliation](protocol-reference/world-transfer-and-reconcile.md) for
validation and bootstrap limits. A checkpoint may be rejected while an export is missing
or stale. Submit the requested export and retry; do not treat a rejection as a successful save.

## World bootstrap

A joining player can request the server's world before joining the live session. The host
controller obtains a world checkpoint and sends `world_file_chunk` events to the requesting
connection. Each chunk identifies a folder and file, its index/count, offset, total file length,
and data. The client reassembles and loads the world locally before joining.

These chunks can contain an entire game save. Do not publish, attach, or log them in public
issues. Use synthetic data when testing chunk framing.

## Clock and speed

The server game is the clock authority for a shared session. A `set_world_time` command carries
a day, hour, and minute; the current `WorldTime` representation uses Sunday through Saturday
as day values 0–6, hours 0–23, and minutes 0–59. The server broadcasts `world_time_changed`
so mirrors can apply the same absolute time. Late joiners receive the current time when
available.

Separate game processes advance their own clocks. The server periodically checks its current
time against its last broadcast and re-broadcasts authoritative time according to the
reconcile policy. It does not sample the mirrors' clocks. This is periodic convergence,
not deterministic lockstep. `set_game_speed` and `game_speed_changed` represent the game's
speed setting separately from wall-clock time.

## What this does not guarantee

The controller delegates actual save/load operations to the game. A checkpoint event or a
complete chunk transfer is not by itself proof of crash-safe atomicity, cross-version save
compatibility, or successful client loading. Verify those properties with the relevant live
scenario before making stronger claims.
