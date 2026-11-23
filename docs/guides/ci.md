# Continuous integration

CI separates managed code, Python tooling, native platform gates, documentation, and tagged release packaging. A green job proves only the commands that job ran.

## Workflows

| Workflow | Trigger | Relevant checks |
|---|---|---|
| `build-and-test.yml` | Pull requests, pushes to `main`, manual dispatch | .NET builds and custom tests, Rust authority sidecar NUnit tests, Python lint and tests, changed-file proprietary-input guard, conditional Windows mod compile, and Windows x86 SimLink build and tests. |
| `docs-check.yml` | Pull requests and pushes | Public boundary scan, strict link/path checks, checker unit tests, `mkdocs build --strict`, site artifact upload. |
| `deploy-docs.yml` | Push to `main` that changes a documentation input, or manual dispatch on `main` | Reads the configured Pages URL, builds the site, publishes it through the self-hosted `docs` runner and GitHub Pages, and optionally syncs sanitized public docs to `ObsidianNetwork/SimsOnline-docs` when `PUBLIC_DOCS_DEPLOY_TOKEN` is configured. |
| `release.yml` | Tags matching `v*` | Builds/tests managed targets, publishes server for Windows/Linux and client for Windows, packages archives with SHA-256 sums, opens a draft release. |

## Equivalent local checks

The Python-tools job uses `python scripts/dev.py lint` and `python scripts/dev.py test`. The .NET custom test runner is invoked by `dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter all`. The docs workflow installs `requirements-docs.txt`, checks docs with strict anchors/path validation, tests `scripts.check_docs_test`, and runs the strict MkDocs build.

The `sidecar-tests` job builds the Rust authority daemon from the checkout and runs
`dotnet test DedicatedServer.Tests/DedicatedServer.Tests.csproj` with
`SIMSONLINE_SIDECAR_DAEMON` set to that executable. The tests use loopback. The active rules on `main` require this job to pass before a pull request can merge.

The Windows jobs matter: the full legacy mod compile uses the .NET Framework 2 compiler via
`scripts/check-mod-compile.ps1 -Strict` when all seven local references are available. Otherwise,
CI emits a skip report because the public checkout excludes those files pending rights verification.
The SimLink test job builds the x86 native sidecar and filters the .NET suite to SimLink cases with
`--fail-on-skip`.

## Interpreting failures

- A Linux pass does not replace the Windows-only mod compile or x86 SimLink tests.
- The docs artifact is a preview product; it is not a release package.
- The release job creates a draft; it does not itself publish a final release.
- The proprietary-input guard checks changed files against the target base; a failure is a release-policy issue, not a compiler error.
- A canceled pull-request run may be superseded by a newer commit; inspect the latest run for the branch.

## Documentation publishing

Edit the source pages, styles, and assets under `docs/` in a pull request. The publishing
workflow runs when a push to `main` changes `docs/`, `mkdocs.yml`, `requirements-docs.txt`,
or `.github/workflows/deploy-docs.yml`. Each deployment builds from the triggering commit.
To confirm a particular update is live, find the **Deploy documentation** run for that
commit and open the site URL from its successful `github-pages` deployment.

Keep the repository's Pages source set to **GitHub Actions** and the deployment workflow
enabled. The build reads the site URL from GitHub Pages so canonical links
and the sitemap use the configured domain and path.

### Deployment runner

The Pages workflow deliberately uses the repository self-hosted runner with both the `self-hosted`
and `docs` labels. It has no `pull_request` trigger, and both jobs run only for the main branch;
dispatching it against another branch leaves the jobs skipped. The branch guard applies to manual
dispatch too: selecting a feature branch in the Actions UI cannot deploy that ref.
The ordinary `docs-check.yml` workflow remains on GitHub-hosted runners and is the workflow that
validates untrusted pull requests.

Register and maintain the runner outside this repository using GitHub's runner setup flow. Use a
dedicated, non-root operating-system account and a runner instance restricted to this repository;
do not place registration tokens, SSH credentials, or deployment secrets in workflow files,
repository files, artifacts, or logs. GitHub Actions obtains the deployment identity through OIDC:
only the deploy job grants `pages: write` and `id-token: write`. The build job uses
`contents: read` for checkout and `pages: read` for `actions/configure-pages`, which passes the
configured base URL to MkDocs through `MKDOCS_SITE_URL`. The build job uploads the Pages site
with `actions/upload-pages-artifact`. The deploy job waits for that artifact with `needs: build`,
runs in the `github-pages` environment, and uses `actions/deploy-pages`. The action references
are pinned to full commit SHAs. Do not enable persistent credentials in checkout.

The PR preview is a separate artifact: `docs-check.yml` builds on a GitHub-hosted runner and
uploads `documentation-site` for seven days with `actions/upload-artifact`. It is not the
`github-pages` deployment artifact and cannot trigger deployment. Keep the Pages runner out of
all pull-request jobs and do not pass PR-produced artifacts or generated code into its deployment
job. GitHub's [custom Pages workflow guide](https://docs.github.com/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages)
documents the artifact dependency, environment, and minimum `pages: write` and `id-token: write`
permissions. Its [secure-use guidance](https://docs.github.com/actions/reference/security/secure-use)
warns that self-hosted runners are not isolated from workflow code; restrict this runner to trusted
deployment work, keep it patched, limit interactive logins, and treat trigger or runner-label
changes as security-sensitive.

The deployment environment is `github-pages`. Configure any required environment protection or
approvals in repository settings, not as embedded credentials. The runner must be online with the
`docs` label before a deployment can proceed. This guide describes repository workflow wiring; it
does not install, register, or configure the machine-side runner. See GitHub's guidance for
[using self-hosted runners in workflows](https://docs.github.com/actions/hosting-your-own-runners/using-self-hosted-runners-in-a-workflow)
for label-based routing and external registration steps.

## Related pages

[Test strategy](test-strategy.md) | [Build from source](build-source.md) | [Release and versioning](release-and-versioning.md) | [Release packaging](release-packaging.md)
