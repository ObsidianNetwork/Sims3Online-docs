# Process-memory bridge verb reference

The table is transcribed from the earlier bridge verb catalog and checked against the current bridge/mod dispatch registrations. It is an inventory of names and ownership, not a promise that every verb is part of the stable public TCP API. `KnownCommands` means the controller's typed `BridgeCommand.KnownCommands` registry recognizes the verb; codec-special and mod-only entries are called out separately.

| Verb | Controller representation | In `KnownCommands` | Mod handler / dispatch | Notes |
|---|---|---:|---|---|
| `APPLY_AUTONOMY` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `APPLY_CASH` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `APPLY_FORCE_ALIGN` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `APPLY_INTERACTIONS` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `APPLY_INTERACTION_QUEUE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `APPLY_MOTIVE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `APPLY_POSITION` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `APPLY_POSITIONS` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `APPLY_SIM_DETAILS` | `DedicatedServer\HostBridge\BridgeCommand.cs` | no | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `APPLY_SIM_STATE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `APPLY_SKILL` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `BULK_REF` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `CANCEL_INTERACTION` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `CONNECT` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `DESPAWN_SIM` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `DIAG_BLOB` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `DIAG_CAPTURE_INTERACTION` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `DIAG_HANDLERTEST` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `DIAG_INTERACTION_STATE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `DIAG_LOADTEST` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `DIAG_QUEUE` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `DIAG_RERUN` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `DIAG_RESOLVE` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `DIAG_RUN` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `DIAG_STATE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `DIAG_WALK` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `DIALOG_ANSWER` | `DedicatedServer\HostBridge\BridgeCommand.cs` | no | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `DIALOG_AUTOANSWER` | no BridgeCommand record | no | `SimsOnlineServerHostMod\ServerHostMain.cs` | mod only |
| `DISABLE_AUTONOMY` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `DISCONNECT` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `ENABLE_AUTONOMY` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `EXPORT_HOUSEHOLD_CONTENT` | `DedicatedServer\HostBridge\BridgeCommand.cs` | no | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `EXPORT_SIM_DESCRIPTION` | `DedicatedServer\HostBridge\BridgeCommand.cs` | no | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `GATE_AUTONOMY` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `IMPORT_HOUSEHOLD_CONTENT` | `DedicatedServer\HostBridge\BridgeCommand.cs` | no | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `IMPORT_SIM_DESCRIPTION` | `DedicatedServer\HostBridge\BridgeCommand.cs` | no | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `INJECT_CLICK` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `INJECT_READ_INTENT` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `LEAVE_CONVERSATION` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `LOAD_SAVE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `LOAD_SAVE_MENU` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `MERGE_LOT_STATE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `MERGE_TRAVELLING_HOUSEHOLD` | `DedicatedServer\HostBridge\BridgeCommand.cs` | no | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `NOTICES` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `NOTIFY` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `PING` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `PROBE_ACTOR_LOOKUP` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `PROBE_HOUSEHOLD_CASH` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `PROBE_PRESENTATION` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `PROBE_SIM_AUTONOMY` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `PROBE_SIM_DETAILS` | `DedicatedServer\HostBridge\BridgeCommand.cs` | no | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `PROBE_SIM_INTERACTION_QUEUE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `PROBE_SIM_LIFECYCLE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `PROBE_SIM_MOTIVE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `PROBE_SIM_PRESENCE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `PROBE_SIM_SKILL` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | mod: swappable handler assembly |
| `PROBE_SIM_STATE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `PROBE_SIM_STATUS` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\Handlers\HandlerSet.cs` | no BridgeCommand record |
| `PROBE_STATE_INJECTION` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `QUEUE_INTERACTION` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `RELOAD_HANDLERS` | `DedicatedServer\HostBridge\BridgeLineCodec.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | no BridgeCommand record |
| `REQUEST_SNAPSHOT` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SACS_RECORDER` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SACS_REPLAY` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SAVE_GAME` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SET_ACTIVE_SIM` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SET_AUTHORITY` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SET_GAME_SPEED` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SET_ROLE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SET_SIM_ALIASES` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SET_SIM_SCOPE` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SET_WORLD_TIME` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SHOW_DIALOG` | `DedicatedServer\HostBridge\BridgeCommand.cs` | no | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `SPAWN_SIM` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |
| `UNGATE_AUTONOMY` | `DedicatedServer\HostBridge\BridgeCommand.cs` | yes | `SimsOnlineServerHostMod\ServerHostMain.cs` | - |

## Interpreting the inventory

- Many gameplay/state verbs have a typed record in `DedicatedServer\HostBridge\BridgeCommand.cs` and a line encoder in `DedicatedServer\HostBridge\BridgeLineCodec.cs`. A `no` in the registry column means the controller does not expose that entry as a typed, known command even though a mod-side dispatch path may exist.
- `CONNECT`, `BULK_REF`, `DIAG_*_REPORT`, and related framing/control entries are handled by bridge codecs rather than ordinary `BridgeCommand` records.
- The mod dispatch map in `SimsOnlineServerHostMod\ServerHostMain.cs` is the authoritative receiving side. Some newer handler groups are dispatched through `SimsOnlineServerHostMod\Handlers\HandlerSet.cs`.
- The table intentionally omits line numbers from the historical inventory: source line numbers move, while verb names and dispatch owners are the interoperability surface.

Related pages: [bridge layout and versions](bridge-layout.md), [bridge diagnostics](bridge-diagnostics.md), [world transfer and reconciliation](world-transfer-and-reconcile.md).
