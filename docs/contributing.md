# Contributing

Contributions to Sims3Online are welcome. This is an experimental project; changes that improve
correctness, testing, documentation, and developer experience are useful.

Before making a change, read the [overview](overview.md), [architecture](architecture.md), and
[current limitations](status.md). The host game is the authority for gameplay; clients send input
and present the host's results.

## Proposing a change

- Keep each change focused and explain its purpose.
- Add or update tests for behavior changes.
- Update public documentation when setup, supported behavior, configuration, or the protocol
  changes.
- Do not include game binaries, saves, packages, decompiled game source, credentials, or personal
  machine paths.

Run the relevant checks from the [testing guide](testing.md). In a pull request, summarize the
change, note any known limitations, and include the commands and results for checks you ran.

Use [GitHub Issues](https://github.com/ObsidianNetwork/Sims3Online/issues) for bugs and proposals,
or [Discussions](https://github.com/ObsidianNetwork/Sims3Online/discussions) for questions.
Report security issues privately using the process in [security](security.md). Contributors are expected to follow the
[Code of Conduct](https://github.com/ObsidianNetwork/Sims3Online/blob/main/CODE_OF_CONDUCT.md).

By contributing, you agree that your contributions are licensed under the repository's
[MIT License](https://github.com/ObsidianNetwork/Sims3Online/blob/main/LICENSE).
