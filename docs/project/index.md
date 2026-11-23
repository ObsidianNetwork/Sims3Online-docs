# Project guide

This section is the project-level guide for technical readers, contributors, and
reviewers. It explains scope, compatibility, contribution expectations, and the
public-support boundary. For exact implementation and protocol contracts, use the
linked code and protocol references rather than treating this overview as an API.

## Choose a path

| Reader | Start here |
|---|---|
| New to Sims3Online | [What is Sims3Online?](what-is-simsonline.md) → [quickstart](quickstart.md) → [architecture](../architecture.md) |
| Contributor | [Contribution workflow](contribution-workflow.md) → [code conventions](code-conventions.md) → [testing](../testing.md) |
| Documentation author | [Docs style guide](docs-style-guide.md) → [documentation coverage](documentation-coverage.md) |
| Bug reporter | [Troubleshooting](troubleshooting-extended.md) → [support and reporting](support-and-reporting-issues.md) |
| Release reviewer | [Compatibility](compatibility-matrix.md) → [known limitations](known-limitations.md) → [licensing and assets](licensing-and-assets.md) |

## Reference map

| Topic | Page |
|---|---|
| Purpose and repository layout | [What is Sims3Online?](what-is-simsonline.md), [repository layout](repository-layout.md) |
| Compatibility and evidence | [Compatibility matrix](compatibility-matrix.md), [known limitations](known-limitations.md), [project status](project-status-and-direction.md) |
| Contribution and documentation | [Contribution workflow](contribution-workflow.md), [code conventions](code-conventions.md), [docs style](docs-style-guide.md), [coverage](documentation-coverage.md) |
| Support and common questions | [Support](support-and-reporting-issues.md), [extended FAQ](faq-extended.md), [extended troubleshooting](troubleshooting-extended.md) |
| Legal and conduct | [Licensing and assets](licensing-and-assets.md), [code of conduct](code-of-conduct.md) |
| Shared terminology | [Project glossary](glossary.md), [main glossary](../glossary.md) |

## Claims and support

Sims3Online is experimental software. Code presence, a passing unit test, a live
scenario, and a published release are different evidence levels. A page may explain
an implemented or experimental path without promising compatibility or support.
Check [current status](../status.md) and [security](../security.md) before running
network-facing or game-mutating workflows.

The host game normally supplies shared simulation outcomes. Client controllers send
player intent and apply selected host-originated state and presentation. The
architecture also contains a client-native lot authority/reconciliation path; its
ownership and save requirements differ. Start with the [architecture overview](../architecture.md)
and [protocol reference](../protocol.md) for those boundaries.

## Public-release boundaries

- Users provide their own compatible, legally acquired game installation.
- The client-facing protocol has no authentication or encryption; public-network
  exposure is unsupported.
- Reference assemblies and game-derived material are not included. Verify rights
  before considering redistribution.
- Report a suspected vulnerability through the private contact in
  [the security policy](../security.md), not a public issue.

Use the [repository layout](repository-layout.md) to locate implementation owners,
the [code reference](../code-reference/index.md) for source paths, and
[documentation coverage](documentation-coverage.md) for the mapping from legacy
technical topics to current public pages.
