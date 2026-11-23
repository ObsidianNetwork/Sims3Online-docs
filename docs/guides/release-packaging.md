# Release packaging

`.github/workflows/release.yml` is the authoritative managed-release packaging recipe. It runs only for tags matching `v*` and produces a GitHub draft release.

## Artifacts currently assembled

| Archive | Included controller output | Additional files |
|---|---|---|
| Windows x64 ZIP | Dedicated server and client | `LICENSE`, `README.md` |
| Linux x64 tarball | Dedicated server | `LICENSE`, `README.md` |

All three publish steps use `--self-contained false`; the target machine needs the .NET runtime. The workflow hashes each generated archive and writes the checksums in `SHA256SUMS.txt`. It does not package the script mod, game installation, save files, or native sidecar as part of these managed archives.

## Reproduce the build boundary

The release workflow builds/tests before publishing. The corresponding platform-specific commands are:

```powershell
dotnet publish DedicatedServer/SimsOnline.DedicatedServer.csproj -c Release -r win-x64 --self-contained false
dotnet publish SimsOnline.Client/SimsOnline.Client.csproj -c Release -r win-x64 --self-contained false
dotnet publish DedicatedServer/SimsOnline.DedicatedServer.csproj -c Release -r linux-x64 --self-contained false
```

The CI workflow writes outputs to staging directories; these example commands compile the same target products but do not by themselves create the tagged archive or checksum file.

## Do not mix local inputs into a release

A local tester kit is a separate tool and has stricter provenance concerns. `scripts/tester_kit.py` can include a gameplay fork package by default; that package is game-derived. Do not distribute it without verifying rights. The `--fork-builder` option can ship a builder instead, and `scripts/tester_kit.py check` checks a kit for EA-derived files. Users must supply their own legally acquired game assemblies/stubs; the repository does not grant redistribution rights for reference stubs.

## Review checklist

- Tag resolves to the intended commit and all required CI jobs passed.
- Archive contains only the intended platform binaries, README, and license.
- Runtime requirement is stated because the publish is framework-dependent.
- SHA-256 sums correspond to the final archives.
- No local config, telemetry DSN, save, game executable, or unauthorized game-derived content is present.
- Draft is reviewed before publishing; CI does not automatically make it public.

## Related pages

[Release and versioning](release-and-versioning.md) | [Tester kit](tester-kit.md) | [CI](ci.md) | [Prerequisites](prerequisites.md)
