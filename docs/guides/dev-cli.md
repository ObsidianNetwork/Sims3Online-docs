# Developer CLI: `dev.py`

`python scripts/dev.py` is the repository's contributor front door for environment inventory, lint, Python tests, and a combined check. It uses the Python standard library for dispatch. Read the current help before using options because it is generated from the checked-out parser.

```powershell
python scripts/dev.py --help
python scripts/dev.py test --help
```

## Commands and flags

| Invocation | Scope |
|---|---|
| `python scripts/dev.py doctor` | Report available tools and capability tiers; does not install dependencies. |
| `python scripts/dev.py lint` | Run Ruff and, when available, the PowerShell parse check. |
| `python scripts/dev.py test` | Run the standard Python test set. |
| `python scripts/dev.py test --dotnet` | Add the .NET build/test suite. |
| `python scripts/dev.py test --all` | Include Python tests normally skipped by CI. |
| `python scripts/dev.py test --force` | Permit the .NET test run during another session's quiet window. |
| `python scripts/dev.py check` | Run lint, then the standard test set. |

`check` also accepts `--dotnet`, `--all`, and `--force`; the parser permits extra pytest arguments after `--`. The exit code is the gate: a printed partial summary is not a pass. Inspect the stages to distinguish a failed check from an unavailable prerequisite or an intentional skip.

## A focused loop

```powershell
python scripts/dev.py doctor
python scripts/dev.py lint
python -m unittest scripts.dev_test
python scripts/dev.py check
```

The unittest command above targets the developer CLI's own test module. For a different utility, substitute its actual test module after confirming it exists. `doctor` is an inventory, not proof that a build or live game scenario works; `lint` is static validation, not runtime verification.

The repository's developer dependencies are listed in `requirements-dev.txt`. Install them in a virtual environment using the documented package-manager command only when a required tool is missing. The standard test command may report that a module could not run when its optional runner is absent; do not describe that module as passed.

## What each gate exercises

`doctor` groups prerequisites rather than installing them. T0 checks the .NET 8 SDK; T1 checks Python 3.12+, `requirements-dev.txt`, PowerShell 7 when available, and optional Node.js; T2 reports local game/compiler/reference inputs used for game-facing work. A ready tier only means its prerequisites were found. It does not assert that a live game starts or that all platform-specific tools work.

The standard `test` path runs the Python suite, CLI help checks, and repository-defined lightweight self-tests; optional components can be skipped when their runner or platform prerequisites are absent. `--dotnet` adds the .NET projects, and `--all` includes Python tests normally excluded from CI. `check` runs lint before tests and returns a failing status if either stage fails. Use the stage output to distinguish a failed test from an intentional skip.

Focused examples:

```powershell
python -m unittest scripts.dev_test
python -m unittest scripts.simctl_test
python -m unittest scripts.sidecar_test
python scripts/dev.py test --dotnet
```

The test modules are ordinary offline checks; they do not prove public-network, live-game, or end-to-end behavior. The developer CLI's own test covers its Python import-path fallback (`FallbackPythonPath` in `scripts/dev_test.py`). The repository's test harness may also execute script `--help` checks, so a help failure can reveal an import-time dependency even when the target command itself is not being run.

## Choosing the right gate

| Change | Minimum useful evidence |
|---|---|
| Python utility | Focused utility tests, then `dev.py test` |
| PowerShell helper | Parse check and a safe help/status invocation where available |
| .NET source | Affected build/test project; use `--dotnet` for the repo-defined suite |
| Native boundary | Relevant platform/toolchain tests plus a bounded local test if available |
| Game-facing behavior | Offline tests plus a separate live scenario; report each layer independently |
| Documentation | `python scripts/check_docs.py --anchors` and the site build in CI |

## Interpreting failures

A missing SDK or local game may block a particular tier without indicating a source failure. A failing `--help` check usually points to import or parser initialization; reproduce it with the specific script before broadening the run. When a combined command fails, preserve the first failing stage and exit code rather than replacing it with the result of a later cleanup action.

## Related pages

[Build from source](build-source.md) · [Test strategy](test-strategy.md) · [Script reference](script-reference.md) · [CI](ci.md)
