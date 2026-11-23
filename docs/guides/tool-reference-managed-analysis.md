# Managed source and assembly analysis

Start from checked-in project source and tests. Use managed-assembly inspection only when the unresolved question is about an external API or the exact local game build. All external inputs remain user-supplied, legally acquired local files; this repository does not grant redistribution rights.

Type lookup prefers the read-only corpus when it contains the same namespace and type name as the bulk cache. The duplicate cache entry is omitted from the results.

## `ts3src.py` command map

The parser in `scripts/ts3src.py` exposes these commands. Run `python scripts/ts3src.py --help` for the current interface. The index command reads the configured local source/assembly corpus; it does not make those assemblies part of the repository.

| Command | Query |
|---|---|
| `index [--no-external]` | Build or rebuild the local symbol index; `--no-external` skips raw-DLL type listings. |
| `find TYPE` | Locate a type. |
| `outline TYPE [--public]` | List members; `--public` restricts the display to public/internal members. |
| `body TYPE MEMBER [--nth N]` | Show a selected member body; `--nth` selects an overload using a 1-based number. |
| `callers TARGET [--limit N]` | Find indexed call sites; default limit is 40. |
| `callees TARGET` | Inspect calls made by a member. |
| `hierarchy TYPE [--depth N]` | Inspect a type hierarchy; default depth is 3. |
| `refs PATTERN [--limit N]` | Search references; default limit is 25. |
| `decompile TYPE [--dll NAME] [--head N]` | Decompile a type from a selected local assembly; default output head is 120 lines. |
| `notes TYPE [--full]` | Show project-local notes when available. |
| `native QUERY [--binary PATH]` | Query the checked-in internal-call map and optional local executable identity. |
| `assemblies` | Summarize indexed and locally listed assemblies. |

Example queries are read-only after the index exists:

```powershell
python scripts/ts3src.py find <TypeName>
python scripts/ts3src.py callers <TypeName.Member>
python scripts/ts3src.py hierarchy <TypeName> --depth 2
```

A graph or call-site match is navigation evidence, not proof that a dynamic callback executes. Reflection, virtual dispatch, event registration, native calls, and generated code may not be represented.

## Runtime/API compatibility checks

`scripts/check_mod_runtime.py` checks the project mod against the modeled game-runtime API surface; `--force` permits a run during another session's quiet window. `scripts/reference_assembly.py` takes one or more assembly paths and verifies that the metadata is reference-only. These checks complement compilation but cannot prove a game runtime has the same implementation or that a behavior works in a live session.

The repository also contains source/navigation helpers such as `scripts/seam_map.py`, `scripts/codegraph_engine.py`, `scripts/probe_source.py`, and `scripts/probe_check.py`. Treat their output as a lead and check each helper's current source and tests before relying on its interface.

## Legal and provenance constraints

Users supply their own legally acquired Sims 3 assemblies or stubs. Compile-only stubs model signatures; they are not executable implementations, and their redistribution rights are unverified. Do not rely on shipping or redistributing those files. Keep local binaries, decompiler databases, exported code, and generated indexes outside commits and issue attachments.

Record the inspected assembly identity and exact type/member in private notes. In public documentation, describe the interoperability contract in original prose and cite project-owned call sites. Do not paste decompiled bodies, game assets, or broad symbol inventories.

## Evidence strength

| Evidence | Supports | Does not establish |
|---|---|---|
| Checked-in source/test | Project-owned implementation and tested contract | What a particular local game build loaded |
| Assembly metadata/signature | API shape in the inspected build | Runtime behavior or redistribution permission |
| Static caller/callee result | Candidate source path | That dynamic dispatch takes the path at runtime |
| Runtime load check | Loadability in that environment | Correct behavior in every scenario/build |
| Bounded live observation | Behavior in that run | General correctness across versions |

## Related pages

[dnSpy](dnspy.md) · [Native analysis](tool-reference-native-analysis.md) · [Analysis and saves](tool-reference-analysis.md) · [Live probes](live-probes.md) · [Build from source](build-source.md)
