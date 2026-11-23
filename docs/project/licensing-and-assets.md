# Licensing and assets

This page describes the repository's current distribution boundaries. It is not legal
advice and does not grant rights beyond the relevant license or permission.

## Project source and artwork

The root [MIT License](https://github.com/ObsidianNetwork/Sims3Online/blob/main/LICENSE)
covers project source and the Sims3Online artwork in `Assets/`. The
[asset inventory](https://github.com/ObsidianNetwork/Sims3Online/blob/main/Assets/README.md)
records the artwork's provenance and intended use. Keep the license notice and the
unofficial-project statement with redistributed project material. The artwork is original
project branding; it does not use EA's exact plumbob, wordmark, or launcher art.

MIT does not grant rights to third-party packages, user contributions under separate terms,
The Sims 3, or other material the project does not own. When adding a dependency or asset,
identify its source and license and preserve any required notices.

## Game files and reference assemblies

Do not include game executables, packages, saves, extracted assets, decompiled game source,
or raw game captures in source releases or documentation. Users supply their own legally
acquired game installation and any local inputs they are permitted to use.

The public checkout intentionally excludes Sims 3 reference DLLs while redistribution
rights remain unverified. The
[reference-assembly README](https://github.com/ObsidianNetwork/Sims3Online/blob/main/refs/sims3/README.md)
documents local build inputs; it is not permission to obtain or redistribute them. Keep
those files out of commits, release archives, test artifacts, and issue attachments. This
is the project's current distribution boundary, not a legal conclusion about the assemblies.

## Native integration provenance review

The retained SimLink integration includes build-specific native addresses and byte
signatures; `SimLink/src/free_ring.cpp` explicitly records their disassembly provenance.
The documentation cleanup preserves that disclosure and the existing functional code.
Neither MIT licensing of project-owned work nor a passing source scan establishes rights
to every embedded snippet. Review these native integration inputs before public release;
do not treat removing a comment, renaming a file, or passing CI as provenance clearance.

## Other material

| Material | Check before publishing |
|---|---|
| Third-party dependency | Verify its license and include required notices. |
| Screenshot, log, or scenario fixture | Remove personal data and check for game-derived or save content. |
| Generated analysis output | Exclude game-derived IL, disassembly, pseudocode and other proprietary code; review for addresses, local paths, and private evidence. |
| Contributor submission | Confirm the contributor can license it under the project terms. |
| Unclear provenance or terms | Exclude it until the source and permission are established. |

Build success, a public download, or a file's presence in a checkout does not prove
redistribution permission. If an artifact's terms are unclear, do not package it; ask
maintainers to resolve the question without copying the questionable material into a
public report.

## Before making a release

1. Inspect the actual source archive or package contents, not only the build script.
2. Verify license and third-party notices for the material being shipped.
3. Confirm game binaries, reference assemblies, saves, local configuration, logs, and
   private evidence are absent.
4. Review screenshots and examples for identity, account details, and game-derived data.
5. Keep unresolved redistribution questions out of the release until they are resolved.

For contribution requirements, see [contributing](../contributing.md). For the
repository's security and network boundary, see [security](../security.md).
