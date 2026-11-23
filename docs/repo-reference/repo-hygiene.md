# Repository hygiene and release boundaries

The repository combines source code with local game integration, native analysis and documentation. Hygiene controls reduce accidental publication of local paths, secrets, proprietary game inputs, broken links and build output. They are complementary: an ignore rule does not remove a file already tracked, and a clean source scan does not establish licensing rights.

## `.gitignore`

The root `.gitignore` excludes local analysis/runtime folders (`/_analysis/`, `/.omo/`, `/.local/`, `/.tools/`, `/Backup/`), package/save extensions, build output (`bin`, `obj`, `dist`, `site`, test results and IDE state), Python caches, local credentials/signing material, machine configuration, logs and Rust targets.

Ignore patterns do not retroactively untrack files. `git ls-files <path>` reports tracked items even if the same pattern would ignore a newly created file. Sensitive tracked content needs explicit review and removal/relocation; do not assume an ignore entry protects it.

## Source and proprietary-input guard

The public-source policy scanner is `scripts/native_observatory_source_guards.py`. CI runs it as the `proprietary-input-guard` job on changed files. Pull-request mode (`--base REF`) checks changed, staged, unstaged and untracked files for symlinks, game package/binary signatures, Ghidra projects, raw binary, bulk disassembly/pseudocode/hex output, private named-home paths and credential patterns. The previous exception for reference-only assembly binaries has been removed; all reference assemblies are rejected. Supported repository images have a separate PR-mode exception, so this is not a ban on every binary file. The ten named assembly files are ignored and absent from the public checkout. Do not stage a local reference assembly or infer redistribution permission from the code's ability to validate a reference-only stub.

Run a local scan against the intended base ref:

```sh
python scripts/native_observatory_source_guards.py --base origin/main
```

CI adds `--format github` for Actions annotations. Without `--base`, the script selects evidence-archive mode with different arguments and stricter absolute-path handling; do not use that mode as a substitute for the changed-file check.

To extend the scanner, add focused tests to `scripts/native_observatory_source_guards_test.py` for both a rejected input and the intended safe case. Keep exemptions scoped to one rule and exact source paths. The source policy is described in `CONTRIBUTING.md` and wired in `.github/workflows/build-and-test.yml`.

## Documentation checker

`scripts/check_docs.py` detects broken relative links, invalid local Markdown heading fragments, and stale repository paths mentioned in backticks. `--anchors` checks heading fragments; `--strict` turns stale path mentions from warnings into errors; `--json` emits a machine-readable report; optional paths narrow the scope. CI runs strict anchor checking over the public documentation and selected READMEs, then independently runs `mkdocs build --strict`.

Validate this section and the checker itself with:

```sh
python scripts/check_docs.py --anchors --strict docs/repo-reference/
python -m unittest scripts.check_docs_test
```

When adding a page, use relative links, verify every backticked repository path exists, and use exact heading text for fragments. For removed historical paths, describe the concept without marking it as a live repo path. Do not weaken strictness or add misleading placeholder files to silence findings.

## Build outputs, data and local captures

Build directories and local evidence/output trees are ignored, including generated artifacts under `dist/`, `site/`, `.local/` and `.omo/`. Before adding an output, check its provenance and contents. Saves, game binaries/packages, user profiles, logs and process captures may contain proprietary or personal data even when a repository script produced them. Use synthetic fixtures in tests and keep local captures outside public changes.

## Maintainer-local experiments

Engine and gameplay probe sources used during maintainer experiments (the `scratch/` directory) and local analysis caches (`.omo/`) are deliberately not part of the public repository; both are ignored by `.gitignore`. Some presentation scenarios and developer scripts reference such probes by name. Those scenarios need the maintainer-local probe sources and are not runnable from a clean public checkout; the rest of the toolchain does not depend on them.
