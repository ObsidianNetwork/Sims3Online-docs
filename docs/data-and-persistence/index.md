# Data and persistence

This section covers durable native saves, transfer and recovery; selected protocol snapshots; the authoritative world clock; and the narrower client-native reconciliation and inventory paths. It distinguishes a loadable world save from a snapshot, presentation mirror, or bounded household export. Deep household import remains quarantined pending cold-load-safe evidence.

| Page | Question answered |
|---|---|
| [Persistence overview](overview.md) | Which representation should I use, and what does each success result prove? |
| [Save/load pipeline](save-load-pipeline.md) | How are native checkpoints transferred, staged, validated and recovered? |
| [Snapshot fields](snapshot-fields.md) | Which current-state fields and identities cross in the snapshot protocol? |
| [World clock](world-clock.md) | How is authoritative day/time validated and synchronized? |
| [Household export/import](household-export-import.md) | What does the household payload contain, and why is deep import quarantined? |
| [Inventory persistence](inventory-persistence.md) | How do inventory membership lines, item identities and native inventory differ? |
| [Lot reconciliation](lot-reconciliation.md) | What client-native data is currently merged before a checkpoint, and when is save blocked? |
| [Recording and replay](../presentation/recording-and-replay.md) | How do transient recorder events differ from snapshots and saves? |

## Choose the representation

| Need | Use | Do not substitute |
|---|---|---|
| Load a world in the game's normal path | Native save plus `WorldSave` transfer | `SimulationSnapshot` |
| Observe selected current state | Snapshot/delta stream | Household property-stream export |
| Recover from truncated files | Verified `LastGoodSave` copy | Snapshot or replay ring |
| Sync current calendar time | `WorldTimeChanged` | Local wall clock |
| Reconcile client-native lot changes | Fresh position-oriented lot export | Assumed household/inventory merge |
| Display inventory membership | Bounded `Iv` set | Full inventory content or durable save |

A successful operation proves only the stage named by that operation. Structural validation is not semantic loadability; an accepted export is not a full household import; a client mirror is not a canonical save.
