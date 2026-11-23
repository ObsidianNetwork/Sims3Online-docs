# Save and package analysis

These tools inspect user-owned save/package files and local evidence. They are for offline analysis, not a way to repair a live world. Work on a copy, preserve source hashes, and keep binary payloads, names, IDs, and game-derived data out of public reports.

## Save lifecycle utility

`scripts/saves.py` exposes `backup`, `verify`, `restore`, `swap`, `list`, `arm`, and `arm-after`. The usage text is available with `python scripts/saves.py --help`; read it before invoking any operation because restore/swap replace profile save contents. The save operations are designed around hashes and local receipts, but that does not make the destination safe to overwrite. Stop game processes and verify the target profile before any replacement.

- `backup` creates a backup and records file hashes.
- `verify` compares the current save folders with a named backup.
- `list` reports available local backups.
- `swap` changes the selected boot save and explicitly requires games to be down.
- `restore` replaces the affected save folders from a named backup and checks hashes.
- `arm` and `arm-after` are scenario-runner hooks, not general save repair commands.

Never use an analysis helper to repair the only copy of a save. If a staged operation leaves an unexpected folder, stop and inspect it rather than deleting by wildcard.

## DBPF-style container inspection

`scripts/dbpf_diff.py` accepts two positional save/package paths and reports record-level differences keyed by resource type/group/instance. Its usage text is printed with no arguments; the tool reports exit 0 for identical inputs and 1 when it finds differences. Run it on copies and treat record names or decoded values as game-derived content.

`scripts/dbpf_dump.py` reports header/index structure and resource-type counts for a selected local container. It can expose payload details; do not publish raw dumps. Unknown or unsupported records should remain labeled as such rather than being assigned guessed semantics.

A byte-level or record-level difference is not proof of a gameplay effect. Serialization metadata, record ordering, and version changes can all matter. Compare matching builds and corroborate an important finding with a test, saved event record, or controlled runtime observation.

## Offline command examples

These commands read their inputs and write or print analysis results; use copies of valuable files and keep output local:

```powershell
python scripts/dbpf_diff.py <before-package-or-save> <after-package-or-save>
python scripts/dbpf_dump.py <package-or-save>
python scripts/saves.py list
```

`dbpf_diff.py` compares records and returns exit code 1 when it finds differences, so that code is a difference signal rather than necessarily a parser crash. `dbpf_dump.py` is an inspection report, not a repair tool. The format-level tests in `scripts/dbpf_layout_test.py` cover locating/parsing the indexed records, including the alternate TravelDB index layout; they do not establish the gameplay meaning of an unknown record.

## Local RefPack diagnostics

`scripts/refpack.py` inspects a supported local `.nhd` save layout. It reports decoded, raw, and failed records, decompressed-size mismatches, and extracted strings by resource type. For an unfamiliar container layout, start with `dbpf_dump.py`. Keep decoded strings local, and run the synthetic decoder tests when checking the implementation.

```powershell
python scripts/refpack.py --help
python -m unittest scripts.refpack_test
```

## A repeatable comparison

1. Select two authorized inputs and preserve immutable originals.
2. Record the tool version, input digests, and selection scope in local notes.
3. Run the narrow read-only comparison first.
4. Review changed and unparsed record counts; note format/version mismatches.
5. Validate any behavior claim using independent project-owned evidence.
6. Share a concise sanitized summary, not the save, package, dump, or a full symbol/resource inventory.

A useful report separates **observed** differences from **inferred** meaning and states the parser limitations. If the source build or format is unknown, stop at the byte/record observation.

## Related pages

[Managed analysis](tool-reference-managed-analysis.md) · [Persistence and time](../persistence-and-time.md) · [Scenario CLI](tool-reference-scenarios.md) · [Debugging](debugging.md) · [Release packaging](release-packaging.md)
