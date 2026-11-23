# Repository layout

The repository separates game-facing integration, controllers, shared protocol code,
tests, scripts, and public documentation. The project list may evolve; inspect the
current solution before adding a component.

| Path | Responsibility |
|---|---|
| `DedicatedServer/` | Server controller, session authority, networking, host bridge |
| `SimsOnline.Client/` | Client controller and mirror application |
| `SimsOnline.Protocol/` | Shared public message types and protocol encoding |
| `SimsOnline.TestClient/` | Test/support client for protocol/server scenarios |
| `DedicatedServer.Tests/` | Controller and server test suites |
| `SimsOnlineServerHostMod/` | In-game script mod and bridge/presentation integration |
| `SimLink/` | Native support component for game process |
| `scripts/` | Build, test, diagnostic, docs, repository tools |
| `scenarios/` | Scenario definitions and test fixtures |
| `schemas/` | Structured protocol/configuration schema artifacts |
| `docs/` | Public MkDocs documentation |
| `Assets/` | Original project artwork and documentation |
| `refs/sims3/` | Compile-time API reference materials; redistribution requires review |

## Runtime boundaries

Controllers are modern .NET applications; the host mod targets an older game scripting
environment. An API available to a controller cannot automatically be used inside the
game mod. Native support has a separate platform and ABI boundary.

## Where to trace

- Session commands/events: server controller and shared protocol project.
- Client application of host events: client controller.
- In-game bridge commands: host mod and server bridge code.
- Animation/motion mirror: host presentation code and client dispatch.
- Automated coverage: test projects, scenario definitions, documented scripts.

Add a project only for a real boundary with a clear owner, test strategy, and
deployment story. Keep shared protocol definitions centralized. Never add game
executables, original packages, saves, decompiled source, private profiles,
credentials, or output dumps from a game installation.

Related: [architecture](../architecture.md), [components](../components.md),
[development](../development.md), [licensing](licensing-and-assets.md).

## Areas to inspect by question

| Question | Start here |
|---|---|
| How does the controller communicate? | `SimsOnline.Protocol/` and controller network code |
| How are sessions managed? | `DedicatedServer/` |
| Where is host/client game integration? | `SimsOnlineServerHostMod/` and client mod projects |
| Where are shared DTOs? | Protocol and shared libraries; check project references |
| What is the native boundary? | `SimsOnline.Native/` and native build files |
| How are game tests run? | `SimsOnline.TestRunner/`, test projects, and testing docs |
| Where are assets documented? | `Assets/README.md` and licensing page |
| Where are site pages? | `docs/` and MkDocs configuration |
| How are setup scripts organized? | `scripts/` and `tools/` readmes/help |

Names and boundaries can evolve. Confirm project references and build files rather than relying on this table as a complete dependency graph.

## Repository navigation habits

Use solution/project files to see the actual build graph. Search from a named protocol type to find serializers and both ends of a message. Search test names and fixtures before changing behavior. For game-process work, distinguish controller code, injected mod code, and native code; they have different build/runtime assumptions. Keep build artifacts out of commits unless explicitly intended.

## Public vs generated content

Generated docs output, local logs, test saves, and binaries are not source documentation. Edit maintained Markdown, source projects, and scripts instead. A file under `refs/` can exist for compile-time API signatures and still be restricted; repository presence does not imply redistribution rights. See [licensing](licensing-and-assets.md).

## Investigating a change

1. Identify the public behavior or protocol type involved.
2. Locate its project and project references.
3. Find both producer and consumer paths.
4. Check validation, identity/ownership, and error handling at boundaries.
5. Find existing tests and whether they exercise negative cases.
6. Determine whether the change affects the game mod, native component, or controller.
7. Update canonical docs and run the relevant test/check commands.
8. Check `git status` for generated or unrelated files before proposing the patch.

## Reading project configuration

Solution and project files are the best starting point for build relationships. Package references and target frameworks can differ among components. Do not assume one root build command exercises all projects. The development and testing pages describe maintained commands and prerequisites; this table only helps locate likely areas.

Scripts can generate, validate, or package content. Read their help and source before running a destructive or broad operation. Avoid invoking scripts with paths from untrusted input. Keep generated output local unless the repository explicitly tracks it.

## Files that deserve extra care

Reference assemblies, binary assets, game saves, captured logs, runtime dumps, and test environment configuration may contain licensed or personal data. Their presence in a checkout does not mean they are suitable for public issue attachment or redistribution. Refer to the licensing page and repository ignore rules.

## Common navigation mistakes

| Mistake | Better approach |
|---|---|
| Search only for a UI label | Trace its command and host handler |
| Assume one solution builds all | Inspect project/solution configuration |
| Treat every test as live game test | Read test project and fixture setup |
| Modify generated output | Find the maintained source and generator |
| Copy a reference binary into release | Check redistribution rights first |
| Change client behavior only | Verify server-side authorization remains |
| Assume docs folders are complete | Check site nav and canonical pages |
| Run scripts from arbitrary cwd | Read script help and use documented root |

When navigating unfamiliar code, follow a concrete type or command across the boundary rather than reading every subsystem. Record where a request originates, where validation happens, where state changes, and how the result reaches clients. This provides a compact mental model for a review.

## Finding the source of truth

Prefer maintained source files and project configuration over copied notes or build output. If two pages disagree, follow the canonical contract page and report the discrepancy so it can be corrected. A generated artifact can help diagnose a build, but it should not be edited as if it were the source.

## Directory orientation notes

| Path | Typical contents |
|---|---|
| `docs/` | Maintained public guides and references |
| `scripts/` | Repository helper and validation scripts |
| `tools/` | Developer-facing utilities and supporting material |
| `refs/` | Reference material; inspect licensing before sharing |
| `Assets/` | Project asset inventory and attribution notes |
| `tests` or test projects | Automated assertions and fixtures |
| solution/project files | Build graph, targets, package references |
| generated output folders | Local artifacts; generally not source |

Folder names can differ over time. Use the actual checkout and project files as authority. Before running a script, inspect its usage and side effects; if it modifies files, ensure the target is a disposable or intentionally selected output location.

## Useful search strategy

Start with a user-visible phrase, then trace its command or event type. Search for serializer/deserializer usage and the host handler. Inspect tests near the implementation and test project configuration. If the feature crosses the game/mod boundary, identify the role-specific code paths and the source of authority. Stop once the behavior's producer, validator, state owner, and consumer are understood; broad repository browsing often obscures the contract.

## Generated and local-only data

Common local artifacts include build directories, test output, generated documentation, crash logs, and game configuration. They are useful for diagnosis but are not automatically source files. Check ignore rules and the build scripts before adding any such file. Never publish a log without reviewing it for identifying paths, names, chat, or connection details.

## Avoiding accidental scope expansion

A useful repository map helps a contributor find the owner of a behavior; it should not become a complete inventory of every file. For a particular task, update only the canonical source and its tests unless the build graph requires more. Before opening a pull request, inspect the changed-file list and remove unrelated modifications. Use the root documentation navigation to find maintained public guides instead of relying on generated site output. When unsure whether a directory is generated, inspect its readme, project references, and build scripts.

## Navigating tests and utilities

Test code may live beside a project or in a dedicated test project; inspect the solution rather than assuming a conventional path. Utility scripts can have side effects such as generating files or changing local state, so read their usage and implementation before execution. If a helper depends on local game installation data, keep the data private and report only sanitized outcomes.

| Navigation task | Reliable clue |
|---|---|
| Find a command handler | Protocol type references and server dispatcher |
| Find a client consumer | Event type references and controller subscription |
| Find a game callback | Mod project and role-specific handler |
| Find a native dependency | Project references and native build configuration |
| Find documentation source | Maintained Markdown plus site configuration |
| Find test scope | Test project, fixture, and assertions |

| Clue | Interpretation |
|---|---|
| Project reference | Build dependency or shared contract |
| Test assertion | Behavior the suite explicitly checks |
| Script help | Supported arguments and side effects |
| Readme note | Local ownership or setup guidance |
