# Known limitations

Sims3Online is experimental. This page separates implementation from evidence and
unsupported behavior. A passing unit test or short scenario should not be generalized
beyond its stated scope.

| Area | Limitation | Consequence |
|---|---|---|
| Game setup | Installation and mod setup are manual | Requires compatible local game |
| Compatibility | Tested game build is narrow | Other patches/content unverified |
| Network security | Protocol lacks authentication/encryption | Do not expose to untrusted networks |
| Gameplay coverage | Not all interactions/services exercised | Some outcomes may differ |
| Presentation | Stream coverage is selective | Client may show incomplete motion/object state |
| Join/rejoin | Bootstrap remains sensitive | State and active drivers may be incomplete/stale |
| Persistence | Save/recovery behavior has limited validation | No production-grade recovery guarantee |
| Third-party content | Arbitrary mods/custom content untested | Resources and behavior may differ |
| Platform | Game integration depends on Windows/32-bit runtime | Controller portability does not imply game portability |

## Failure classes

| Class | Meaning | Reporting guidance |
|---|---|---|
| Reproduced defect | Known conditions cause incorrect behavior | Include minimal reproduction/evidence |
| Partial support | Narrow case works; broader matrix untested | State exact boundary |
| Unverified | No reliable evidence either way | Avoid claims of support |
| Unsupported by design | Outside current contract | Do not present workaround as support |
| Environment failure | Setup prevents judgment | Mark blocked, not pass/fail |

## Areas to retest

- Client-side simulation through systems other than interaction insertion.
- Client visual state competing with posture, overlays, or look-at behavior.
- Cross-household relationships and stale menu context.
- Long routes and vehicle travel causing position/posture discontinuities.
- Late join/rejoin with active animation drivers and object mappings.
- Runtime-created Sims, services, object state, and inventory.
- Long-session stability and resource use.

## Safety boundaries

The protocol must not be exposed directly to the Internet. Use a trusted isolated
network or approved private transport. Do not share credentials, private saves,
proprietary game files, or personal paths in public reports.

"Working" means code exists and some relevant verification exists, not that every
installation works. A mitigation reduces impact but may not remove root cause. A
workaround is an operator action, not a guarantee. See
[compatibility](compatibility-matrix.md), [status](../status.md),
[troubleshooting](troubleshooting-extended.md), and
[support](support-and-reporting-issues.md).

## Symptom-to-scope guide

| Symptom | Likely category | Why it matters |
|---|---|---|
| Sim missing on client | Interest/bootstrap | Could be visibility scope, spawn, or identity mapping |
| Sim shown but state differs | Model mirror | A snapshot or event family may be absent |
| Sim acts on client without input | Authority leak | More serious than visual mismatch |
| Animation differs, host state matches | Presentation coverage | Check record and local competing writer |
| Menu choice refused | Stale client inputs | Host remains authoritative |
| Client can connect remotely without login | Security boundary | Protocol does not authenticate peers |
| Same code behaves differently on another patch | Compatibility | Native offsets/resources are build-sensitive |
| Test never reaches in-world stage | Environment/setup | Mark blocked rather than product failure |

## Risk ownership

| Risk | Responsible boundary | What would reduce uncertainty |
|---|---|---|
| Unsupported game version | Release/configuration | Explicit tested-build policy and matrix |
| Client-side world mutations | Gameplay authority | Broad scenario coverage across tick/alarm paths |
| Presentation gaps | Recording/mirror | Field-specific tests and boundary inventory |
| Late-join stale state | Snapshot/event handoff | Join/rejoin acceptance with active interactions |
| Network takeover | Protocol security | Authentication, authorization hardening, encryption |
| Reference DLL distribution | Licensing review | Maintainer/legal determination or removal |
| Long-session instability | Runtime capacity | Controlled soak with memory and crash evidence |

## Reporting a limitation

State whether it is directly observed, inferred from missing coverage, or unsupported by policy. Give the smallest reproducible case and list what the evidence does not establish. If a fix is attempted, keep the old limitation until the new test has passed at the relevant evidence level. Related: [compatibility](compatibility-matrix.md), [troubleshooting](troubleshooting-extended.md), [status](../status.md).

## Separating confirmed failures from missing evidence

| Evidence state | Definition | Example wording |
|---|---|---|
| Reproduced | Same setup repeatedly triggers symptom | ?This route can teleport the client copy under these conditions.? |
| Intermittent | Symptom occurred but not on every attempt | ?Observed occasionally; repeat rate is unknown.? |
| Partially verified | Some cases pass, important cases remain | ?Basic same-lot flow passed; rejoin not covered.? |
| Unmeasured | No meaningful test exists | ?Long-session memory behavior is not established.? |
| Unsupported | Explicitly outside current contract | ?Public Internet use is unsupported.? |
| Mitigated | Control reduces symptoms but cause remains | ?Mitigation lowers observed incidence; root cause unresolved.? |

A public status claim should include the evidence class when there is a risk readers may treat a narrow pass as broad support. Do not preserve private run IDs or tracker references in this table; use public scenario names or concise behavior descriptions.

## Operational risk register

| Risk | Impact | Mitigation for users/contributors |
|---|---|---|
| Una authenticated peer can join or impersonate player | Session control and game state may be exposed | Keep listener local/private; do not port-forward |
| Game process is 32-bit | Address-space exhaustion in long runs | Treat long sessions as experimental; capture process health |
| Client mirror incomplete | Different pose, menu, or state on copies | Use status/scenarios and report exact field |
| Save state is inconsistent | Rejoin/recovery may fail | Use disposable, backed-up test saves |
| Game build mismatch | Hooks/resources may be invalid | Match tested build across host and clients |
| Unclear reference assembly rights | Distribution/legal risk | Hold distribution until reviewed |
| Local VM data exposure | Licensed or personal content may leak | Do not publish VM images or raw saves |

## When to downgrade a claim

Downgrade or qualify support when a required build disappears, a test becomes flaky, a security assumption changes, a regression appears in a previously passing scenario, or a reference dependency cannot be redistributed. A passing test that no longer runs in current CI is stale evidence; link the claim to the current source of verification.

## Limitation-to-impact map

| Limitation | User impact | Mitigation |
|---|---|---|
| No network authentication/encryption | A reachable listener is not safe for untrusted users | Restrict to a private trusted network; never expose publicly |
| Host/game version dependence | Client or mod may fail on another build | Use a known environment and record version |
| Incomplete presentation coverage | Some client effects may be absent or misleading | Treat host simulation as authoritative; report specific effect |
| Rejoin is partial | Reconnect may not restore a coherent session | Test with disposable state; restart rather than trust stale UI |
| Local game state can diverge | Client view may be stale or unowned | Distinguish host outcome from local display |
| Third-party mods unverified | Hooks or state may conflict | Reproduce on clean install |
| Save-mode differences | Authority modes can persist state differently | Back up and verify the chosen mode before use |
| Platform assumptions | Native/game path may be Windows-specific | Check project target frameworks and docs |

## Failure containment

If state appears inconsistent, stop sending actions that could mutate the world. Record only a minimal sanitized observation, preserve a backup of any local save, and restart from a known test scenario when safe. Do not repeatedly reconnect to an exposed listener to ?see if it works.?

If the failure occurs after an update, record the before/after revision and whether a clean configuration reproduces it. If it only occurs with a mod or custom content, remove that variable before filing a general compatibility claim.

## Questions to ask before relying on behavior

| Question | Why it matters |
|---|---|
| Which process owns the result? | Local display can diverge from host simulation |
| Is the listener reachable only by trusted peers? | Protocol transport lacks authentication/encryption |
| Does the test cover the exact game build? | Game hooks and UI are version sensitive |
| Is a feature tested with more than one client? | Single-player evidence misses recipient and conflict behavior |
| Was a disconnect tested? | Reconnection can leave stale or incomplete state |
| Is persistence part of the scenario? | In-memory success does not prove save/reload behavior |
| Are optional mods installed? | Third-party hooks can change results |
| Is test data disposable? | Avoid irreversible save changes |

## Unsupported does not mean impossible

A limitation records the current public support boundary, not a claim that the behavior can never be implemented. Conversely, a code path or experimental option does not imply support. Any expansion of support requires a clear owner, a safe operational boundary, and evidence appropriate to the behavior.

When a limitation is fixed, update this page and the compatibility matrix together. Preserve history in version control rather than leaving obsolete warnings that contradict current behavior. If only one edge case is fixed, describe the narrower change instead of declaring the whole area complete.

## Reporting a limitation

A useful report separates observation from hypothesis:

| Field | Example of useful detail |
|---|---|
| Observation | "Client B did not display the updated route" |
| Expected result | "A second participant should see the accepted route" |
| Authority result | Whether host accepted, refused, or did not receive request |
| Scope | Role, scenario, revision, and selected authority mode |
| Repeatability | Always, intermittent, or once |
| Safety impact | Whether listener or save was exposed/changed |
| Evidence | Sanitized relevant log excerpt or test result |

Do not label a suspected cause as established fact. A missing display update might be a presentation gap even when host simulation succeeded. State the difference; it helps maintainers route the report to the correct subsystem.
