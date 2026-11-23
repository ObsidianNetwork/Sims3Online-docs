# Release and versioning

The release workflow is tag-triggered and creates a draft rather than publishing automatically. A tag matching `v*` starts the workflow; the workflow's build is the source of the released managed artifacts.

## Release sequence implemented by CI

1. Build the client and the .NET test project with the .NET 8 SDK.
2. Run the custom .NET test runner with `--filter all`.
3. Publish the dedicated server for `win-x64` and `linux-x64`, and the client for `win-x64`, using `--self-contained false`.
4. Package the selected outputs with `LICENSE` and `README.md`.
5. Generate `SHA256SUMS.txt` and create a GitHub draft release with generated release notes.

The exact workflow is `.github/workflows/release.yml`. The repository does not enforce a SemVer parser for tag names: the trigger is the `v*` pattern. Do not claim that a tag is a compatible release merely because it matches the trigger.

## What the tag does not build

The workflow builds managed server/client products. It does not build the legacy game mod or SimLink native sidecar, and it does not run an in-game acceptance test. Those need their own checks. The published managed artifacts are framework-dependent; consumers need the matching .NET runtime installed.

## Release evidence

Before promoting a draft, check the exact tag and commit, all workflow jobs, platform artifacts, archive hashes, and the intended release notes. Keep the draft until package membership and legal/provenance checks are complete. A passing test run is not proof that optional local game files can be distributed.

## Compatibility notes

A release may contain several independently versioned layers: controller binaries, protocol schema, script mod, and optional native sidecar. Check compatibility across those layers from actual manifests/build stamps and tests. Do not infer a single compatibility guarantee from the Git tag alone.

## Related pages

[Release packaging](release-packaging.md) | [CI](ci.md) | [Test strategy](test-strategy.md) | [Build from source](build-source.md)
