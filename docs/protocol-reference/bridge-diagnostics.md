# Bridge diagnostics

Diagnostic verbs run across the local process-memory bridge. Their reports are correlation-scoped bridge responses, not public TCP `ServerEvent` records. Read-only inspection commands are useful when a host appears alive but its mirrored state is stale; code-loading verbs are a separate, development-only surface.

## Request and report shape

`BridgeLineCodec` encodes a diagnostic request with a correlation ID. The mod returns a matching report prefix, status/kind, and hex-wrapped UTF-8 text. For example, `DIAG_STATE_REPORT|17|ok|...` means correlation 17 completed; the final field must be hex-decoded before interpreting report text. Similar typed report prefixes include `DIAG_QUEUE_REPORT`, `DIAG_RESOLVE_REPORT`, `DIAG_WALK_REPORT`, `DIAG_INTERACTION_BUNDLE_REPORT`, `DIAG_INTERACTION_STATE_REPORT`, `DIAG_BLOB_REPORT`, `DIAG_LOADTEST_REPORT`, and `DIAG_RUN_REPORT`.

## Diagnostic command catalog

| Command | Arguments | What it reports |
|---|---|---|
| `DIAG_STATE` | none | Host state and counters. The report is key/value text; stable fields include role, `inWorld`, tick, autonomy/command-gate state, pending outbound work, pump ticks, probe counts, dispatch, and commanded/observed speed. Additional in-world clock, inventory/use, and swallowed-error counters may be present. |
| `DIAG_CAPTURE_INTERACTION` | `<simId>` | Captures replay-bundle fields for the Sim's current interaction. |
| `DIAG_INTERACTION_STATE` | `<simId> [hexBookId|-]` | Reads posture/container, active interaction, and reading-progress state. |
| `DIAG_QUEUE` | `<simId>` | Dumps the interaction queue and exit reasons. |
| `DIAG_RESOLVE` | `<objectId>` | Reports which known object/Sim resolution paths find the identifier. |
| `DIAG_WALK` | `<objectId> <hexFieldPath|-> <depth:1-6> [maxNodes]` | Performs a bounded reflection walk of a live object; `-` means the object itself. |
| `DIAG_BLOB` | `<blobId> <index> <count> <hexChunk>` | Reassembles an uploaded binary blob. This is a code-upload capability, not a read-only diagnostic. |
| `DIAG_LOADTEST` | none | Loads the embedded test assembly to check that the runtime's assembly-load path works. |
| `DIAG_RUN` | `<hexTypeName>` | Loads the last uploaded assembly and invokes its `Run` entry point. |
| `DIAG_RERUN` | none | Invokes the previously resolved probe again without another assembly load. |
| `DIAG_HANDLERTEST` | none | Loads the last uploaded handler assembly and invokes its test entry point. |
| `RELOAD_HANDLERS` | `[hexEntryType]` | Attempts to hot-swap a handler assembly previously uploaded through the bridge. |

`DIAG_WALK` arguments are not arbitrary plain text: field paths are hex encoded. The report exposes the dump as text; it does not make the game's internal object graph a stable API.

## Reading `DIAG_STATE`

The `DIAG_STATE` response is semicolon-separated `key=value` text encoded inside its bridge report. Fields are looked up by key, not positional order; producers append fields to preserve older readers. `swallowed` is a count of exceptions caught by otherwise-empty host-mod catches. A steadily increasing count is a signal that a fault is being hidden, not proof of a protocol error. In-world reports can include clock reconciliation, inventory/use sampling, and refusal counters.

## Safety boundary

The bridge admin listener is loopback-only and has a 15-second default request timeout.
Treat `DIAG_BLOB`, `DIAG_LOADTEST`, `DIAG_RUN`, `DIAG_RERUN`, `DIAG_HANDLERTEST`, and
`RELOAD_HANDLERS` as privileged developer operations. `DIAG_BLOB` uploads/reassembles bytes;
it does not itself execute them. The other listed verbs load or invoke code inside the
game process. Do not expose the admin listener through a public socket, tunnel, or router
port, and do not upload untrusted assemblies. Public gameplay clients should use the
public command/event protocol, not raw bridge commands.

The exact registrations and report encoders live in `SimsOnlineServerHostMod\ServerHostMain.cs`, `SimsOnlineServerHostMod\BridgeCodec.cs`, `DedicatedServer\HostBridge\BridgeLineCodec.cs`, and `DedicatedServer\Network\BridgeAdminListener.cs`.

Related pages: [bridge verbs](bridge-verbs.md), [transport limits](transport-limits.md), [troubleshooting](../troubleshooting.md).
