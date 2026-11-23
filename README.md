# Sims3Online Public Documentation

Public documentation for **Sims3Online**, an experimental multiplayer project for *The Sims 3*.

This repository publishes the public documentation site at [https://obsidiannetwork.github.io/Sims3Online-docs/](https://obsidiannetwork.github.io/Sims3Online-docs/) using MkDocs Material.
Documentation source changes are maintained in [`ObsidianNetwork/Sims3Online`](https://github.com/ObsidianNetwork/Sims3Online) and mirrored here for public hosting.

> **Naming compatibility note:** The project is branded **Sims3Online**. Solution, project, assembly, namespace, and environment identifiers (`SimsOnline.sln`, `SimsOnlineServerHostMod`, `SimsOnline.Client`, `SimsOnline.Protocol`, `SIMSONLINE_*`) retain `SimsOnline*` for compatibility.

## Local preview

```sh
python -m pip install -r requirements-docs.txt
mkdocs serve
```

To verify the strict static site build:

```sh
mkdocs build --strict
```

## License

Documentation and original artwork in `docs/assets/` are released under the [MIT License](LICENSE).
Sims3Online is an unofficial fan project and is not affiliated with, endorsed by, or sponsored by Electronic Arts or Maxis.
