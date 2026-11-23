# Extended FAQ

## Project and architecture

### What does Sims3Online do?
It experiments with multiplayer Sims 3 using one server-side game as shared simulator
and client games as input/presentation endpoints.

### Is it an official EA product?
No. It is unofficial and does not imply Electronic Arts endorsement.

### Does each player simulate a household?
No. One host simulation is authoritative. A claim controls input permission, not local
simulation.

### Is it ready for general players?
No. It is experimental software for developers. See [status](../status.md).

### Does connected mean synchronized?
No. A transport connection does not prove bootstrap, models, or presentation are
current.

### Are host and client states identical?
Not guaranteed. Clients apply selected state/presentation; coverage is incomplete.

### Why not run one simulator per player?
Independent game systems can choose different outcomes, so duplicate simulation
diverges.

### What is the controller?
A .NET process managing session and game integration through a mod/bridge.

### What is a claim?
An authorization association deciding who can command a Sim; it does not transfer
simulation.

## Setup and compatibility

### Do I need The Sims 3?
Game-integrated work needs a compatible copy obtained legally by the user; the
repository does not provide it.

### Which game versions are supported?
Support is narrow; check [status](../status.md) and
[compatibility](compatibility-matrix.md).

### Can I use arbitrary mods or custom content?
No general compatibility claim is made; content can change interactions and resources.

### Is there a consumer installer?
The project is developer-oriented. Check current release information; do not assume
one exists.

### Can I test without a live game?
Many unit/protocol tests need no game, but do not prove in-game behavior.

### Which SDK should I use?
Use versions documented in [development setup](../development.md) and repository
config.

### Why does a controller build but the mod fail at runtime?
The embedded game runtime is older and may lack APIs available to controllers.

## Multiplayer behavior

### Why does a click fail?
Host validates request against current game state and ownership; client menu may be
stale.

### Why can a menu show a refused choice?
Client may not share host object, inventory, posture, or relationship inputs.

### Why does client show another animation?
Coverage, competing local behavior, resource differences, timing, or identity mapping
may be involved.

### Does the server replay interactions on every client?
The current model favors one host simulation with selective updates; interaction
replay is historical.

### Can players meet on a community lot?
There are scenarios for shared places, but behavior remains scenario- and
save-dependent.

### Can a player join while the world runs?
Bootstrap paths exist, but late-join completeness is limited.

### What does "accepted" mean?
The host accepted/queued the request; it does not prove the action completed.

### Why does travel look wrong?
Vehicle and long-distance travel are more complex than local walking and remain
limited.

### Why does a refusal appear after a menu choice?
The host executes authoritative eligibility checks; local menu state can lag.

## Security, licensing, data

### Can I expose server on the Internet?
No. Protocol is unauthenticated and unencrypted; use a trusted isolated environment.

### Should I upload a save to a bug report?
No. Describe minimal reproduction and sanitize logs; saves may include
personal/licensed content.

### Can I distribute reference assemblies?
Do not assume rights; see [licensing and assets](licensing-and-assets.md).

### Does MIT license cover the game?
No. It covers project material as stated, not EA content.

### Where should I report security issues?
Follow security policy, not a public issue for undisclosed
vulnerabilities.

## Contributing and support

### How do I run checks?
Use commands in [testing](../testing.md) and report exact outcomes.

### What should a bug report include?
Expected/actual, reproduction, versions, sanitized logs, and reproducibility; see
[support](support-and-reporting-issues.md).

### Why is a scenario blocked?
A prerequisite process or environment failed before behavior could be judged. Blocked
is neither pass nor fail.

### What does partial support mean?
Only part of behavior has evidence; it is not a promise for untested cases.

### Where are contribution rules?
See [contribution workflow](contribution-workflow.md) and the root guide.

### Can I submit game files to help debug?
No. Provide a sanitized description and only material you may lawfully share.

### Why do docs links fail in CI?
Relative path or heading anchors may be wrong; run the documented checker.

### Is an old wiki or internal handoff authoritative?
No. Use current public documentation and source/tests.

Related: [limitations](known-limitations.md),
[troubleshooting](troubleshooting-extended.md), [glossary](glossary.md).

### What does the client mirror contain?
Selected state and presentation, not a complete byte-for-byte game clone. Consult the specific subsystem page.

### Why can a screenshot disagree with a state probe?
A probe can report logical posture while a local renderer or animation driver is still wrong. Record both kinds of evidence.

### Is a workaround proof the feature works?
No. It may bypass the untested boundary; report exactly what it excludes.

### Why should commands and test results be included in a report?
They let another contributor distinguish a setup failure from a product defect and reproduce the same check.

### Where are tool-specific instructions?
Use the public developer guides and each tool's documented help; do not rely on private setup notes.

### Why should I not include the VM image?
It can contain licensed game content, credentials, personal save data, and machine identifiers.

### Does the documentation guarantee every linked command works on every platform?
No. Platform prerequisites and current tool support are described by the relevant development guide.

### What should I do if two pages disagree?
Check the current source, tests, and status; report the discrepancy and update the canonical page through review.

### Is the default host-authoritative mode the only code path?
No. The status page also notes client-native lot authority/reconciliation support with different behavior and save requirements.

### Can I treat a public issue as a security disclosure channel?
No. Follow the private process in the repository security policy.

### What does a compatibility "Not yet" cell mean?
The combination is unsupported or lacks evidence; it does not necessarily mean a known defect has been reproduced.

### How should I describe a capability that passes one scenario?
Name the specific scenario conditions and avoid generalizing to other saves, content, player counts, or game builds.

### Can I copy snippets from the root docs into an external guide?
Follow the MIT license for project text/code where applicable and preserve notices; do not copy game-derived or third-party content.

### Who responds to support requests?
The repository maintainers and community contributors, with no guaranteed response time or commercial SLA.
