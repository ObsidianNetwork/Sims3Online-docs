# Tester kit

`scripts/tester_kit.py` assembles a local kit, verifies its manifest, and can install it into a local Sandboxie box for a host-side check. The README in `tools/tester_kit/` explicitly says this directory is not itself a public player distribution.

## CLI

```powershell
python scripts/tester_kit.py build --help
python scripts/tester_kit.py build --fork-builder
python scripts/tester_kit.py verify
python scripts/tester_kit.py check
python scripts/tester_kit.py list
```

`build` accepts `--no-mod-build`, `--no-publish`, `--no-fork`, `--fork-builder`, `--no-loader`, `--simlink <path>`, and `--loader <path>`. `install-box` requires `--box` and optionally accepts `--kit`, `--template`, `--player`, `--sim`, and `--server`. Use `python scripts/tester_kit.py --help` for the current full argument spelling.

## Provenance and verification

A kit manifest records a hash and byte size for each file, together with provenance information. `verify` detects missing, changed, and extra files. `check` additionally tests the kit for EA-derived files. These checks do not establish redistribution permission for third-party software or game-reference material.

The default build may include the gameplay fork package, which is game-derived. `--fork-builder` substitutes a builder path instead; the kit's builder mode requires the recipient to use their own game inputs. Users must supply their own legally acquired Sims 3 assemblies or stubs. Do not send a kit until the legal and third-party dependency review is complete.

## Host-side proof

`install-box` is a Windows/Sandboxie validation step; it needs a prepared box and profile. It does not create that isolated environment automatically. The installer checks the kit manifest before copying files. Use the same `verify`/`check` commands again after any packaging change.

## Safety boundaries

- Do not use a valuable personal save or profile as the test target.
- Do not include game files, saves, credentials, or telemetry DSNs.
- Keep bridge-admin endpoints local to the host; a remote player uses the client session endpoint only.
- Keep the installed package and generated archive outside Git unless release authorization and provenance are clear.

## Related pages

[Private testing](private-beta.md) | [Release packaging](release-packaging.md) | [Prerequisites](prerequisites.md)
