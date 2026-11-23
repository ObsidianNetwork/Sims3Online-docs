# Compatibility matrix

Labels are conservative: **Working** means implemented with relevant evidence;
**Partial** means only part of the contract has evidence; **Not yet** means
unsupported, absent, or insufficiently verified. Re-check the current branch before
deployment.

| Feature | Working | Partial | Not yet / unverified |
|---|---|---|---|
| Controller build | .NET projects build with configured toolchain | environment-specific integration | other SDK/runtime combinations |
| Automated tests | controller/protocol tests cover selected behavior | live game behavior narrower | full end-to-end reliability |
| Supported game | current status identifies tested build | specific expansion combinations | other patches/arbitrary expansions |
| Server authority | host is default shared simulator | authority gates need scenario verification | proof no client-side world update leaks |
| Client-native lot authority | code includes explicit lot-authority/reconciliation support | behavior and save requirements differ from default | broad acceptance and interchangeable operation |
| Player input | intent forwarding and server checks exist | definition/UI parity varies | every menu/immediate action |
| Presentation | selective state and visual channels exist | timing and object coverage | pixel-equivalent clients |
| Same-lot multiplayer | core session flows exist | save/scenario-specific behavior | universal acceptance guarantee |
| Long-distance travel | basic route presentation exists | vehicle/travel boundary | all worlds and travel types |
| Late join/rejoin | bootstrap and restore code exists | active driver/transient state | complete recovery for every interaction |
| Persistence | save/world integration exists | interruption and recovery cases | durability guarantee |
| Network security | local/trusted test operation | private network controls operator-managed | public Internet use/authentication |
| Mods/custom content | no broad claim | accidental combinations may work | compatibility certification |
| Packaging | development setup documented | manual local install | consumer installer |

The public status also describes explicit client-native lot-authority and
reconciliation code. This is an alternate behavior surface with different save
requirements, not a reason to describe the default host-authoritative flow as
client-simulated. Verify the mode, configuration, and relevant tests before making a
compatibility claim.

## Evidence labels

- **Working:** code path exists and has a test or direct validation relevant to that
  claim.
- **Partial:** useful evidence exists, but an important branch, duration, or
  environment is missing.
- **Not yet:** unavailable, unsafe, or not proven enough to claim.

A component test can pass while the end-to-end feature remains partial.

## Updating the matrix

Name user-visible capability, not internal work item. Cite public test evidence.
Distinguish unit, integration, and live scenario tests. Include game build, save
conditions, player count, and duration. Downgrade status when a test becomes flaky or
prerequisites change. Never mark an unrun test as passing because adjacent code
compiles.

See [testing](../testing.md) and [status](../status.md).

## Environment dimensions

| Dimension | Compatibility question | Evidence to record |
|---|---|---|
| Game patch | Do native hooks and APIs match? | Executable patch/build identifier |
| Expansion set | Are referenced types/resources present? | Installed content set |
| Save | Are Sims/lots/resources compatible? | Sanitized save provenance, not save file |
| Controller revision | Do client/server messages agree? | Repository revision for each |
| Mod/fork package | Are hooks and shell/module aligned? | Package/build identity |
| Native sidecar | Does architecture/patch match? | Architecture and sidecar build |
| OS | Are controller tools and game runtime supported? | OS version and architecture |
| Content mods | Do overrides change definitions or UI? | Mod list category, not proprietary files |
| Network | Does route satisfy security assumptions? | Private/trusted topology description |

## Current supported game statement

The public current status does not establish compatibility beyond the project's tested build. Historical engineering notes and native implementation refer to a specific Steam 1.67.2.024037 executable. Because native patches and internal-call maps are build-sensitive, do not generalize that evidence to other executables or storefronts. The server and clients should use matching game/content configuration for meaningful comparison.

## Marking a cell

A cell is **Working** only when a repeatable test covers that particular feature in the named environment. **Partial** is appropriate when core behavior exists but important routes, player counts, or recovery cases are missing. **Not yet** covers unsupported or untested combinations. Cite test artifacts in a public release note where possible; do not cite internal tracker identifiers or private machine reports.

## Matrix use cases

| User question | How to interpret this page |
|---|---|
| Can I use another game patch? | Treat as unverified unless a row specifically says otherwise |
| Can I add a third-party mod? | No general compatibility guarantee; test in isolation |
| Can I connect from outside the machine? | Only in a trusted, controlled private network; public exposure is unsupported |
| Can I use a Linux controller? | Controller portability is separate from Windows game-process compatibility |
| Can I join a running world? | Code exists but late join/rejoin is partial and scenario-sensitive |
| Does passing controller tests prove visuals? | No; a live game scenario is required for presentation |
| Are alternate lot authority modes identical? | No; save and reconciliation behavior differs |

## Change log discipline

When support changes, update the matrix and the linked status page in the same documentation change. State the exact tested combination, not a broad label such as ?Windows supported.? Keep the test name public and stable where possible, but do not publish private run folders, user names, or internal ticket IDs.

## Compatibility report template

```text
Project revision: [public commit/tag]
Controller/runtime: [version]
Game build/content: [sanitized supported identifiers]
Authority mode: [default/alternate, if applicable]
Participants: [count and roles]
Scenario: [public name or concise behavior]
Result: [pass/partial/fail/blocked/not run]
Limit: [what this evidence does not establish]
```

This template is an explanatory reporting format, not a protocol schema. Omit private paths and account/network details. Related: [status](../status.md), [testing](../testing.md), [troubleshooting](troubleshooting-extended.md).

## Reading per-area evidence

"Working" means there is a specific implementation and relevant positive evidence; it does not mean every environment is supported. "Partial" means an important role, edge case, or recovery path remains incomplete. "Not yet" means no reliable support claim is made. Use "Unverified" in prose when evidence is simply missing rather than implying the feature is planned.

| Area | Evidence to seek | Common overclaim to avoid |
|---|---|---|
| Basic connection | Connect/disconnect tests and trusted network setup | "Internet-ready" |
| Identity | Server association and invalid identity rejection | "Authenticated" |
| Actions | Host acceptance/refusal, actor ownership | "All game actions work" |
| Presentation | Recipient view and failure behavior | "Perfectly synchronized" |
| Dialogs | Request ID, answer, timeout, role validation | "Any dialog is supported" |
| Persistence | Save/reload under selected authority mode | "All clients share one save" |
| Native integration | Target architecture/build and fallback | "Portable" |
| Mods/custom content | Named combination and scenario evidence | "Mod compatible" |

Update evidence when the test harness or public support boundary changes. A single green run should be tied to a revision and scenario. If a capability is technically implemented but has no reliable evidence, keep the claim cautious and state what remains unverified.

## Environment dimensions

| Dimension | Why it can matter |
|---|---|
| Game executable/build | Internal hooks and UI behavior may vary |
| Installed packs | Objects and interactions may be unavailable |
| Save state | Existing world contents and ownership can differ |
| Windows architecture | Native modules require matching architecture |
| .NET runtime/SDK | Build and runtime requirements differ |
| Client/controller revision | Protocol and behavior can diverge |
| Network topology | Listener reachability and trust boundaries vary |
| Mods/custom content | Hooks, IDs, and interaction definitions can conflict |
| Authority mode | Simulation and persistence owners differ |
| Number of clients | Concurrency and recipient filtering are exercised |

A compatibility report should not claim support for dimensions it did not record. Test one variable at a time where practical. The matrix is a snapshot; current evidence and code may be newer than a published artifact, so state which revision the report concerns.

## Report scope boundaries

A report should distinguish "not tested" from ?tested and failed.? A row marked partial does not imply a feature is unusable in every scenario; it means no broad support claim is warranted. Conversely, one successful case cannot erase known edge cases. Keep exact conditions near each claim and link to a reproducible test or report where available.

## Confidence labels

Use a confidence label only when it clarifies the evidence. "Verified" should identify a test context; "partial" should name a missing boundary; "unknown" should not be disguised as a positive status. Include negative evidence when it changes the reader's decision, such as a known refusal path or unsupported mode.

## When no row applies

If the environment or feature is absent from the matrix, do not infer support from a neighboring row. Check the linked source and tests, then describe the result as unverified until there is evidence for that exact combination. New support claims should add a row only when it clarifies a meaningful, maintainable boundary.

If evidence is old or its environment cannot be reconstructed, mark the claim as historical or unknown rather than current support.
