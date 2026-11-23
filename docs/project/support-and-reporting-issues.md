# Support and reporting issues

Sims3Online is an experimental community project rather than a supported commercial
service. Reports are most useful when they distinguish a reproducible defect from
setup trouble or unverified behavior.

Use the repository's public issue tracker for reproducible bugs and focused feature
requests. Use public discussion for general questions when available. For security
concerns, follow security policy and do not publish sensitive
exploit details before maintainers assess them.

## Bug report checklist

1. **Summary:** one sentence describing failure.
2. **Expected behavior:** what should happen and why.
3. **Actual behavior:** exact visible/logged result.
4. **Reproduction:** shortest sequence, player count, process role.
5. **Environment:** OS, project revision, controller/runtime, game build as
   applicable.
6. **Evidence:** sanitized logs and relevant test output.
7. **Frequency:** always, intermittent, or once.
8. **Workaround:** clearly labeled if any.

Before attaching logs, remove usernames, home-directory paths, tokens, account
details, machine names, private IPs, and personal save data. Never upload game
executables, original packages, copyrighted saves, decompiled source, or credentials.
If a private save is essential, describe its structure and ask maintainers for a safe
route.

A bug describes mismatch with documented/intended behavior. A feature proposal
explains use case, affected users, constraints, and alternatives. For both, identify
which process owns the decision.

Maintainers may request a smaller reproduction or classify behavior as
unsupported/partial. A report that cannot be reproduced is still useful if evidence is
precise, but is not a confirmed defect. There is no guaranteed response time,
compatibility promise, or production support SLA.

Related: [troubleshooting](troubleshooting-extended.md),
[limitations](known-limitations.md), [code of conduct](code-of-conduct.md).

## Report quality checklist

Before submitting, check that the issue is reproducible and that the report does not disclose secrets or licensed game content.

| Include | Keep safe |
|---|---|
| Expected vs observed behavior | No passwords, tokens, or session secrets |
| Sanitized project revision | No private repository or local absolute paths |
| Exact command and result | No raw save files or game assets |
| Minimal reproduction steps | No public IPs, port forwards, or router screenshots |
| Relevant sanitized log excerpt | Remove user names, account IDs, and chat |
| Whether this is a live or unit test | Do not upload proprietary assemblies |

A small code snippet authored for the report may help if it contains no copied game material. Prefer a public test case or typed protocol description. Do not attach full logs by default; include only the relevant window and retain the original locally.

## Triage expectations

A report may need follow-up information before it can be reproduced. Maintainers may ask for a project revision, command, or scenario; do not send credentials or enable remote access to your machine. Reports are reviewed as time allows, and an issue is not a promise of a fix or release date.

| State | Meaning |
|---|---|
| Needs reproduction | Essential safe evidence is missing |
| Confirmed | A maintainer reproduced or verified the condition |
| Planned | Work is intended but not necessarily started |
| In progress | A change is being developed |
| Fixed | A change exists; verify the indicated revision |
| Closed/no action | Out of scope, duplicate, or not reproducible with available evidence |

Use the Code of Conduct contact only for conduct or safety matters, not ordinary bug triage.

## Reproduction checklist

1. Use the smallest scenario that still exhibits the behavior.
2. Start from a known revision and note it.
3. Record the role and action that triggered the result.
4. Separate server outcome from client presentation.
5. Repeat only if safe and avoid repeating non-idempotent actions.
6. Remove optional mods or custom content to isolate the baseline.
7. Capture only the relevant sanitized output.
8. State whether the result is consistent, intermittent, or one-time.
9. Include the exact validation command where relevant.
10. Describe the expected outcome without assuming implementation details.

## Privacy and consent

Before attaching media, inspect every visible window, title bar, username, chat, desktop notification, path, and network address. Crop or redact sensitive portions locally. Do not share a VM image, save, game assets, or full directory archive. Ask other participants before including their names, messages, or screenshots.

A maintainer should not need a password or remote-control session to triage an ordinary issue. Decline requests to expose a service publicly for debugging. Use the private security channel for a suspected vulnerability and avoid posting exploit steps in a public issue.

## What not to send

Do not attach passwords, access tokens, private keys, account/session credentials, save files, full VM images, game installation archives, proprietary assemblies, private chat logs, or unredacted desktop captures. Avoid public IP addresses and router/firewall screenshots. Maintainers should not ask you to disable your security controls or expose a listener publicly to reproduce an issue.

## Follow-up and closure

If asked for more detail, provide the narrowest safe additional evidence. If you cannot reproduce on the latest revision, say so and include the revision where it occurred. If the issue is resolved, close or update the report with the validating version and scenario; avoid claiming all similar cases are fixed unless tested.

A missing response does not imply acceptance, rejection, or a target date. The issue tracker is for actionable project work, not emergency support. Keep the report respectful and avoid publishing another person's details without consent.

## Choosing an issue type

A concise bug report describes a defect in existing behavior. A feature request describes a desired behavior and the problem it solves. A question belongs in the project's documented discussion route when one exists. Security and conduct reports should use their private channels, not a public issue. If uncertain, omit sensitive detail and ask maintainers for safe routing.

## A concise report outline

```text
Title: [component + observable failure]

Expected:
[What should happen?]

Observed:
[What happened instead?]

Reproduction:
1. [safe step]
2. [safe step]

Context:
- Revision:
- Role(s):
- Test type: automated / live scenario
- Authority mode, if relevant:

Validation:
- `exact command`: [result]

Privacy note:
[Confirm attached evidence has been sanitized]
```

Do not paste this template with empty sections; include only relevant details. If the issue is a security concern, do not use a public reproduction outline that exposes exploit mechanics. Contact maintainers through the private security instructions.

## Respectful follow-up

Keep comments focused on the technical issue. Avoid tagging individuals repeatedly or demanding a timeline. If you have new evidence, add it as a concise update with the revision and scenario. If the original report contained accidental private data, contact maintainers to remove it and avoid quoting it in follow-up.

## If the issue is about a dependency

Confirm whether the symptom reproduces without the optional dependency, record the exact version, and check its public documentation. Keep the report focused on the interaction observed in this project; do not copy private dependency support conversations or post credentials from either system.

For an intermittent problem, provide a small count of attempts and outcomes (for example, two reproductions in five controlled runs) rather than ?sometimes.? Do not repeat a scenario if doing so could expose the listener or damage a save.

A report with no reproduction can still be useful when it describes a clear regression or user impact; state explicitly that it is not reproduced and avoid guessing at a fix.

| Report state | Contributor action |
|---|---|
| New evidence available | Add a short revision-linked update |
| Cannot reproduce now | State this and retain the original observation |
| Fixed in a revision | Verify the relevant scenario before closure |
| Personal data was exposed | Request removal and avoid quoting it |
| Security risk suspected | Stop public detail and use private reporting |
| No response yet | Wait; do not repeatedly tag individuals |
