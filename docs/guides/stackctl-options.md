# `stackctl` option quick reference

This page is retained as a short index target; the full option definitions, lifecycle verbs, defaults, and safety semantics are in [the `stackctl.py` reference](stackctl-reference.md).

Use these parser-verified forms as starting points, then consult current help before running a lifecycle action:

```powershell
python scripts/stackctl.py --help
python scripts/stackctl.py status --help
python scripts/stackctl.py boot-server --help
```

The most common controls are `--build`, `--json`, `--endpoint`, `--save`, `--timeout`, `--require-client`, `--quick`, `--strict`, `--follow`, `--lines`, and the per-role client isolation flags. `--timeout` is a bounded readiness wait, not proof the game process has stopped when a caller times out. `archive --prune N` only removes data when paired with `--yes`; inspect the candidate list first. `down --force` has broader process impact and requires verified ownership.

## Related pages

[Complete stackctl reference](stackctl-reference.md) · [Run server](run-server.md) · [Run clients](run-client.md) · [Logging](logging-diagnostics.md)
