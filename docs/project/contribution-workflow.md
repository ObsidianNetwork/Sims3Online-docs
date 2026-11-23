# Contribution workflow

Contributions should be focused, reproducible, and safe for a public repository. Use a
branch, explain the change, test it, document user-visible behavior, and submit it for
review.

## Before changing code

1. Read [architecture](../architecture.md), [status](../status.md), and the relevant
   subsystem page.
2. Identify the owner of the behavior and protocol boundary involved.
3. Search for existing tests and documentation before duplicating a mechanism.
4. Confirm the change does not require game files, saves, credentials, or private data
   in Git.
5. Choose the smallest change that establishes intended behavior.

| Phase | Practice |
|---|---|
| Scope | One coherent behavior per change |
| Design | State inputs, outputs, failure behavior, authority boundary |
| Code | Follow language/runtime constraints of target component |
| Test | Add regression coverage for behavior changes |
| Docs | Update setup, protocol, status, or user-facing docs |
| Review | Inspect complete diff, not only edited function |

For protocol changes, update producer and consumer and add compatibility tests. For
game-mod changes, check runtime constraints as well as compilation; the in-game
runtime is older than the controller runtime.

## Validation

Run focused tests first, then broader checks when shared infrastructure changes.
Report exact commands and results. A live game test should include reproducible setup
context, reset strategy, and evidence from host and client. If a live test cannot run,
say why.

## Public safeguards

Never add original game binaries, packages, saves, decompiled source, credentials,
personal paths, or private network details. If redistribution is uncertain, do not add
the file; ask maintainers. Keep unrelated local changes out of a pull request.

A pull request should explain user-visible problem, design boundary, implementation,
validation, limitations, and compatibility impact. Sanitize screenshots/logs before
attachment.

## Review checklist

- Does the change preserve host authority?
- Are errors explicit and bounded?
- Are retries/idempotency safe?
- Are tests updated?
- Are links and commands accurate?
- Did the diff include unrelated or sensitive artifacts?
- Does the summary distinguish tested facts from assumptions?

Related: [conventions](code-conventions.md), [docs style](docs-style-guide.md),
[support](support-and-reporting-issues.md).

## Choosing the right test layer

| Change type | Minimum evidence | Stronger evidence when practical |
|---|---|---|
| Pure helper/codec | Focused unit/round-trip tests | Property tests for malformed inputs |
| Session authorization | Accepted and rejected command cases | Network-level connection lifecycle |
| Bridge line | Encoder/decoder symmetry | Live mod/controller exchange |
| Client mirror | Handler tests and reset behavior | Two-process visual/state scenario |
| Game hook | Compile/runtime contract validation | Live test in disposable save |
| Native support | Build and static contract checks | Exact-build runtime test and rollback |
| Docs only | Link/anchor check | Strict site build when available |
| Packaging | Manifest/artifact inspection | Install/uninstall on clean environment |

Use the smallest relevant command from [testing](../testing.md). If a test is platform-specific or requires licensed game data unavailable to you, report the limitation, not an inferred pass.

## Pull request structure

A clear review submission answers: What behavior changes? Which component owns it? What alternatives were considered? What tests were run, with exact results? What was not tested? Does the change alter wire compatibility, supported game assumptions, or save behavior? Is there a rollback or migration concern?

Keep implementation and documentation consistent in the same change where possible. If a capability is only partially verified, write that directly rather than leaving status prose unchanged.

## Review and revision

Reviewers should be able to trace a public claim to code/test evidence. A review request may ask for narrower scope, additional negative tests, clearer failure behavior, or removal of risky artifacts. Address each request in the relevant code/docs and rerun checks. Do not copy private review comments or tracker details into public documentation.

## Before publishing

Run a diff review for unrelated files, secrets, private machine paths, saves, game binaries, decompiled source, or assets with unclear rights. Confirm the top-level contribution and security policies still apply. The branch should contain only the intended changes and associated tests/docs.

## Suggested pull request body

```markdown
## Summary
[What behavior changes?]

## Design boundary
[Which component owns the decision and why?]

## Validation
- `exact command`: passed / failed / skipped / not run

## Limitations
[What remains unverified?]

## Compatibility or data impact
[Protocol, save, game build, or none]
```

This is a suggested format, not a required template. If the repository provides a pull request template, preserve its sections and checklist. Never claim a test passed if it was skipped or blocked.

## Documentation and release communication

If a change alters a public command, supported behavior, security boundary, or setup requirement, update its docs before merging. For a release-bound change, ensure the status/compatibility text does not overstate the test evidence. A deprecation should name what replaces the old behavior and whether old clients can still connect.

## Contributor data safety

Use synthetic identifiers in examples. Sanitize screenshot windows and logs before attaching. Do not place game-derived source or save fragments in a test fixture. If a test needs licensed data, document the prerequisite and keep data local and ignored.

## Working with changes safely

Start from a clean branch and inspect status before editing. Keep unrelated local work untouched. Avoid adding generated binaries, logs, local configuration, game saves, or private test artifacts. Use repository conventions for project structure and test placement. If behavior changes, update the canonical documentation in the same contribution.

| Stage | Contributor action | Review signal |
|---|---|---|
| Scope | State user-visible behavior and exclusions | Focused patch |
| Design | Identify owner, trust boundary, and compatibility impact | Contracts remain clear |
| Implement | Add bounded code and tests | No unrelated formatting churn |
| Validate | Run targeted checks and report exact outcomes | Reproducible evidence |
| Document | Update canonical page and links | Claims match implementation |
| Review | Answer questions with evidence | No unsupported guarantee |

A reviewer may request a smaller diff, a negative test, a clearer authority boundary, or narrower docs claims. Explain tradeoffs and unresolved conditions rather than expanding the patch outside its stated scope.

## When a test cannot run

State why (missing SDK, unavailable game, hardware boundary, or environment failure), what alternative validation did run, and the remaining uncertainty. Never substitute "not run" with "passes locally" or omit a known failure. For live game tests, avoid checking in install paths or private content.

## Review-ready change checklist

- [ ] Scope and affected user behavior are explicit.
- [ ] Component ownership and trust boundary are understood.
- [ ] Existing public contracts remain compatible or migration is described.
- [ ] Positive and negative cases are tested where feasible.
- [ ] The exact test commands and outcomes are recorded.
- [ ] Unavailable validation is identified with a reason.
- [ ] Documentation links and anchors are checked.
- [ ] No generated output, credentials, personal paths, or game data were added.
- [ ] Status language does not exceed evidence.
- [ ] Any unresolved license question is called out before distribution.

## After review

Keep follow-up changes focused. If a review identifies a documentation mismatch, update the canonical page instead of adding a duplicate note. If a requested test is impossible in the current environment, explain what it would establish and what evidence is available instead. Do not collapse multiple validation results into an unqualified ?all tests pass.?
