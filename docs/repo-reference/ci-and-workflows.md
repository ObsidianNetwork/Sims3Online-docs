# CI and workflow reference

Four workflow files define build/test, documentation preview, Pages deployment and tagged releases. Permissions are read-only by default; elevated contents/Pages permissions are scoped to release/deployment jobs.

## Workflow map

| Workflow | Triggers | Jobs / runner | Outputs |
|---|---|---|---|
| `.github/workflows/build-and-test.yml` | Pull requests, pushes to `main`, manual dispatch. Superseded PR runs cancel; main runs finish. | `build-and-test` (Ubuntu/.NET 8); `sidecar-tests` (Ubuntu/Rust/.NET 8); `python-tools` (Ubuntu/Python 3.12 and .NET 8); `proprietary-input-guard` (Ubuntu/full history); conditional `mod-compile-gate` (Windows); `simlink-windows` (Windows/Python/.NET/MSVC). | Build/test/guard results, no persistent build artifact. |
| `.github/workflows/docs-check.yml` | Every pull request and push. | `docs-check` on Ubuntu/Python 3.12: public boundary check, strict link/anchor checker, checker unit tests, strict MkDocs build. | Uploads built `site/` as `documentation-site` for 7 days, including feature branches. |
| `.github/workflows/deploy-docs.yml` | Push to `main` when docs/config/dependency/workflow paths change, or manual dispatch on `main`. | All jobs use the self-hosted `docs` runner. `build` reads the configured site URL and builds with MkDocs in strict mode, `deploy` publishes to `github-pages`, and `sync-public-docs` mirrors sanitized docs to `ObsidianNetwork/SimsOnline-docs` when `PUBLIC_DOCS_DEPLOY_TOKEN` is set. | Pages artifact, published site, and optional public-docs sync. The repository's Pages source must be GitHub Actions. |
| `.github/workflows/release.yml` | Push a tag matching `v*`. | `build-and-release` on Ubuntu/.NET 8 builds/tests and publishes server/client packages. | Windows ZIP and Linux tar.gz with license/readme, `SHA256SUMS.txt`, and a GitHub draft release. |

## Build and test jobs

The .NET job builds `SimsOnline.Client`, `SimsOnline.TestClient`, and `DedicatedServer.Tests` (which project-references `DedicatedServer`), then runs the custom test runner with `--filter all`.

The `sidecar-tests` job builds `simlink_authority_daemon` in the runner's temporary directory and
runs the entire NUnit project against it. A run that discovers no tests fails the job.

The Python job installs `requirements-dev.txt`, runs `python scripts/dev.py lint` and `python scripts/dev.py test`. The script's exclusion list documents CI omissions due to local evidence, tools or platform requirements. CI also sets up .NET because one managed-inventory test builds a small .NET tool.

The proprietary-input job fetches full Git history, chooses the pull-request base or push predecessor and calls `scripts/native_observatory_source_guards.py --base <ref> --format github` on changed files.

The Windows mod compile job checks for all required local Sims 3 reference assemblies. If they
are absent—as in a clean public checkout—it emits an explicit skip report; if present, it runs
`scripts/check-mod-compile.ps1 -Strict`. It does not require a live game. The Windows SimLink job
builds the x86 sidecar with the selected Visual Studio generator, builds the .NET tests, then runs
the SimLink cases with `--fail-on-skip` so Windows-only cases must execute.

Local equivalents for the portable build/test work:

```sh
python -m pip install -r requirements-dev.txt
python scripts/dev.py check
dotnet build SimsOnline.Client/SimsOnline.Client.csproj
dotnet build SimsOnline.TestClient/SimsOnline.TestClient.csproj
dotnet build DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj
dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter all
```

Windows-only gates require Windows compilers, PowerShell and native build tools. A Linux run cannot replace them.

## Documentation workflows

`docs-check.yml` runs `python scripts/check_docs.py --anchors --strict` over public docs and selected READMEs, tests the checker with `python -m unittest scripts.check_docs_test`, runs `mkdocs build --strict`, and uploads `site/` as a seven-day preview artifact. It runs on every push and pull request.

`deploy-docs.yml` builds and publishes the canonical docs from the triggering `main` commit. The
push filters cover `docs/` (including theme assets), `mkdocs.yml`, `requirements-docs.txt`, and the
deployment workflow. `actions/configure-pages` reads the configured base URL and passes it to
MkDocs as `MKDOCS_SITE_URL`. The deploy job publishes the artifact built in that workflow run.

Use the `documentation-site` artifact to preview a branch. The live version is recorded by the
successful `github-pages` deployment for `main`. See the [CI guide](../guides/ci.md#documentation-publishing)
for the repository settings and deployment runner.

Reproduce documentation checks locally:

```sh
python -m pip install -r requirements-docs.txt
python scripts/check_docs.py --anchors --strict README.md CONTRIBUTING.md SECURITY.md docs
python -m unittest scripts.check_docs_test
mkdocs build --strict
```

## Release workflow

The tagged workflow runs .NET builds/tests, publishes `DedicatedServer` for Windows x64 and Linux x64 and `SimsOnline.Client` for Windows x64, packages outputs with `LICENSE` and `README.md`, computes SHA-256 checksums, and creates a draft release with generated notes. The workflow does not create a Linux client archive or publish a live game installation.

## Failure interpretation

- `docs-check` green means link/anchor and MkDocs validation passed; it does not mean Pages deployed.
- SimLink tests that skip on Linux run on Windows with `--fail-on-skip`.
- Mod compilation is compile-only against checked-in reference declarations, not a live game test.
- A proprietary-input finding should be investigated at source, not hidden by broad exemptions.
- A tag run creates a draft, not an automatically published release.

## Related

- [Testing](../testing.md)
- [Development setup](../development.md)
- [Repository hygiene](repo-hygiene.md)
