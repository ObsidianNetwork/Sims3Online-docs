# Documentation style guide

Public docs should help a contributor or technical reader without requiring internal
context. Write for someone who has just cloned the repository.

## Page structure

A technical page should normally provide a title and purpose, system boundary and
terminology, a table or diagram where helpful, safe accurate examples, failure
modes/limits, and links to related public pages. Do not force a section where it does
not apply; a broad project guide should provide enough detail to act.

## Voice and claims

- Use direct, neutral language.
- Describe observed behavior, not an internal plan or personal opinion.
- Distinguish implemented, tested, partially verified, and unsupported.
- Prefer specific limitations to vague reassurance.
- Label workarounds as workarounds.
- Avoid release guarantees without an explicit policy.

## Code, paths, commands

Use fenced blocks with language labels. Prefer commands that work from repository root
and state prerequisites. Mention repository paths only if they exist in current
checkout. Never include home-directory paths, usernames, machine names, tokens,
private URLs, or game-derived source.

## Tables, diagrams, links

Use tables for fields, responsibilities, compatibility, and symptom/cause/fix. Use
Mermaid for flows/states that are hard to describe linearly. Keep labels short. Use
relative Markdown links and verify every local link/anchor before publishing. Avoid
internal folders, generated/private corpora, personal plans, tracker cards, or
unreviewed external services.

## Privacy and licensing

Do not publish proprietary game assets, saves, decompiled code, binary dumps,
credentials, private test outputs, or identifying local paths. State a license only
when verified. Flag uncertainty for maintainer review rather than guessing.

## Maintenance

When code changes, update its behavior page and relevant FAQ/troubleshooting. Remove
stale examples rather than leaving contradictions. Run the documented docs checker and
fix broken links and anchors.

Related: [contribution workflow](contribution-workflow.md), [repository
layout](repository-layout.md), [testing](../testing.md).

## Examples and executable accuracy

A code block should be one of: an exact current command, a schematic example explicitly labeled illustrative, or pseudocode with clear boundaries. Do not mix these. For exact commands, check argument parsing and documented prerequisites. For JSON, validate field names against the current protocol model. Use synthetic identifiers and non-sensitive strings; avoid examples that look like credentials or machine addresses.

| Example type | Label | Validation |
|---|---|---|
| Current CLI invocation | "Run from repository root" | Confirm script exists and `--help` syntax |
| Wire message | "Synthetic example" | Confirm serializer/type fields |
| Pseudocode | "Conceptual flow" | Check it doesn't imply unimplemented behavior |
| Log snippet | "Illustrative" unless exact | Sanitize identifiers and paths |
| Build/test output | Exact command and result | State platform and skipped tests |

## Page maintenance checklist

- Is the page's purpose clear in its opening paragraph?
- Does it identify the component's authority boundary?
- Are statuses time/version qualified where needed?
- Are tables readable on mobile and not needlessly wide?
- Are Mermaid diagrams valid and useful?
- Are examples safely copyable or explicitly schematic?
- Are all relative links and anchors valid?
- Do code-formatted paths exist in the current checkout?
- Does the page avoid tracker IDs, personal plans, and internal coordination?
- Is there a failure/limit section for operational content?

## Navigation changes

A new page needs a useful local index entry and appropriate site navigation by the documentation owner. Do not edit navigation files when your assigned work is limited to a subsection; provide the page and link it from the section's index for the site owner to include. Avoid orphan pages and duplicate indexes that disagree.

## Versioning content

Avoid exact dates unless they matter to the claim. If a capability changes quickly, cite the stable public status page and describe the relevant branch/release context. Remove outdated commands rather than appending a correction that leaves both instructions plausible.

## Cross-page consistency

| If you change? | Revisit? |
|---|---|
| Authority model | Overview, architecture, status, FAQ |
| Protocol field/event | Protocol page, examples, compatibility guidance |
| Build prerequisites | Development setup, quickstart, troubleshooting |
| Test command | Testing guide and quickstart |
| Game version support | Status, matrix, client VM guidance |
| License/assets | License page and release notice |
| Public contact | Support and Code of Conduct summary |
| Page grouping | Folder index and site navigation owner |

This is a checklist rather than a rule to duplicate every detail. Prefer a canonical page and link to it from other pages.

## Accessibility and localization

Use descriptive link text rather than ?here.? Give images meaningful alt text. Avoid status that relies on color alone. Expand acronyms on first use when the target reader may not know them. Keep dialog and UI examples clear about which text is host-generated and which is local. Avoid idioms that translate poorly when a public audience is international.

## Review evidence

A docs-only contribution should still be read as if a new contributor will follow it literally. Run the link/anchor checker. For a site-wide navigation change, run the strict build when feasible. Report exact validation and do not claim a visual preview if none was opened.

## Page structure pattern

A durable topic page generally includes:

1. A title that names the user question.
2. A short purpose and scope statement.
3. A model or table that clarifies roles and boundaries.
4. Procedures or examples only where they are reproducible.
5. Failure and limitation cases.
6. Links to the canonical related pages.
7. A note about evidence or version-sensitive behavior when needed.

Use a table when comparing explicit categories, not as a substitute for explanation. Use numbered steps only when order matters. Keep code examples minimal, accurate, and clearly labeled if schematic. Never present illustrative pseudocode as a guaranteed wire format.

## Link and anchor hygiene

Use relative links inside the docs tree. Prefer stable heading text and avoid manual anchors unless required for compatibility. When changing a heading, search for incoming links and run the anchor checker. Link to the exact section when it materially helps navigation; do not make a whole page a link farm.

## Status language

Use "implemented" for source capability, "tested" for exercised behavior, "supported" only when an intentional support boundary exists, and "released" only for a published artifact. State unknowns plainly. Avoid ?complete,? ?secure,? ?works everywhere,? or "production-ready" unless there is evidence that justifies the scope.

## Tables and examples

Keep tables narrow enough to scan, with concise cells and descriptive headers. Put procedural nuance outside the table. Use fenced code blocks with a language identifier when syntax is real; label illustrative or schematic payloads clearly. Remove placeholder values that could be mistaken for real credentials or addresses.

| Element | Use when | Avoid when |
|---|---|---|
| Table | Comparing stable dimensions | Prose needs qualification |
| Checklist | Reader must verify conditions | It implies unsupported guarantees |
| Numbered steps | Order matters | Sequence is arbitrary |
| Bullets | Items are independent | Every item needs a precise status |
| Callout | Safety or important caveat | It repeats nearby text |

## Maintenance

Review docs when APIs, commands, project layout, support labels, or security assumptions change. Search for the old term/path before renaming. Test relative links from the source file rather than assuming site navigation will repair them. Remove obsolete examples instead of leaving conflicting guidance alongside newer procedures.

## Security-sensitive instructions

Review a page from the perspective of a new reader who may execute every command literally. Never include a real password, access token, private endpoint, or user-specific absolute path. State whether a listener is local-only or network reachable, and do not imply that changing a UI option is authorization. For steps affecting saves or installed content, recommend a backup or disposable copy.

A page should also identify whether examples are exact commands, schematic snippets, or conceptual descriptions; readers must not confuse one for another.
