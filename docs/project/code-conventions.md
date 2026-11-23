# Code conventions

Use conventions that make authority, lifecycle, and failure behavior visible. This
summarizes project-level expectations; local style and stronger component guidance
take precedence.

## Principles

- Keep responsibilities at the boundary that owns them.
- A client must not decide shared simulation outcomes.
- Validate wire data and external input before use.
- Make state transitions explicit and test failure paths.
- Prefer shared typed protocol definitions to duplicated string parsing.
- Keep blocking/long work away from game tick callbacks.
- Make diagnostics useful without exposing private data.

| Component | Constraint |
|---|---|
| .NET controllers | Use SDK and language version configured by repository |
| Host game mod | Target older .NET/Mono-compatible APIs and syntax |
| Native component | Match supported executable architecture and ABI |
| Python scripts | Follow repository format/lint/test configuration |
| Documentation | Use relative links and current checkout paths |

A successful build does not prove a game API exists in the embedded runtime. For mod
changes, verify target framework references and use live/integration checks when
practical.

## Protocol handling

Treat incoming values as untrusted. Validate identity, authorization, ranges, and
collection sizes. Reject malformed messages explicitly and avoid partial mutation.
Preserve ordering/correlation where required. Keep encoding/decoding symmetric and
tested. Make retries safe against duplicate execution.

## Errors and logging

Include enough context to locate the failed boundary without credentials, private
paths, raw save data, or sensitive identifiers. Separate user-facing explanations from
diagnostic detail. Do not swallow data-loss or authority failures; catch only where a
safe fallback is defined.

## Tests and review

Test success, rejection, and edge behavior. For distributed flows test
producer/consumer codecs, authorization, ordering, and stale identity. Use
deterministic coordination rather than timing assumptions. Live tests are narrower
evidence than broad automated coverage.

Ask: does new state have a lifetime/reset path? Are client commands checked on server?
Can reconnect duplicate an action? Is an object ID process-local? Can work block a
game pump? Do tests assert behavior?

See [workflow](contribution-workflow.md), [testing](../testing.md), and
[protocol](../protocol.md).

## Data ownership and state lifecycle

For every mutable field, identify who writes it, who reads it, its reset condition, and whether it is durable. A field mirrored from the host should not also be mutated by client autonomy. A request object should not be reused across sessions unless it carries a generation/identity boundary. A collection fed by network or game callbacks needs size limits and stale-entry cleanup.

| State kind | Ownership question | Cleanup question |
|---|---|---|
| Session map | Which connection/player owns it? | What happens on leave/reconnect? |
| Sim/object alias | Which world generation does it reference? | How is despawn/reload invalidated? |
| Pending dialog | Which owner may answer? | What happens on timeout/disconnect? |
| Presentation driver | Which client module started it? | What record stops it? |
| Refusal cache | Which actor/target/definition key? | How are clear/expiry handled? |
| Snapshot cursor | Which baseline does it follow? | How does full resync reset it? |

## Public API and code clarity

Choose explicit names for authority-sensitive methods (`Apply...`, `Request...`, `Validate...`) and avoid generic helpers that hide role changes or write side effects. Keep protocol record types and serialization logic centralized. Avoid exposing implementation-only details as stable public API unless compatibility is intentional.

## Error surfaces

- Return a typed result or explicit failure reason at boundaries.
- Include correlation and stage when reporting failed requests.
- Ensure a failed client presentation does not stop host simulation.
- Never silently convert parse failure to a plausible gameplay value.
- Keep log messages bounded and avoid per-frame unbounded output.
- Do not leak private user data into public protocol/events.

## Formatting and repository settings

The repository's `.editorconfig` sets UTF-8, LF line endings, spaces, and final newlines. Python lint/test configuration lives in `pyproject.toml`; .NET 8 stack projects use shared settings in `Directory.Build.props`. Other tool projects can have their own configuration. Do not apply a broad formatter to unrelated files; format the touched code consistently with its local conventions.

## Review examples

**Prefer:** the server validates actor ownership, resolves target identity, and returns a typed rejection before any game mutation.

**Avoid:** a client-only guard that hides the command button but leaves the server command handler unrestricted.

**Prefer:** a snapshot baseline plus ordered deltas with an explicit cursor.

**Avoid:** "send state periodically" without defining staleness, reset, or missed-event recovery.

**Prefer:** a bounded catch that records which presentation line failed and allows safe subsequent processing.

**Avoid:** an empty catch around an entire event batch that makes partial application invisible.

These examples describe design qualities, not required names or exact implementations. The right boundary depends on the component; link the code change to the protocol/architecture contract it affects.

## Security-sensitive code

Any code that accepts a network message, applies a lot export, maps an actor, answers a dialog, or loads a native module needs explicit validation and a negative test. Keep privileged local bridge commands separate from public session commands. Do not rely on obscurity, loopback defaults alone, or client UI behavior as authorization.

## Change-scope checklist

Before implementation, identify the component that owns the behavior and inspect both callers and consumers. A protocol change often touches serializer, command validation, event production, client handling, tests, and docs; a presentation-only change should not alter authoritative state. Keep the patch focused and avoid renaming neighboring types unless needed for correctness.

| Change type | Review boundary |
|---|---|
| New command | Parse, validate, authorize, execute, respond, test invalid input |
| New event | Produce, serialize, route, handle, tolerate unknown/stale data |
| Game action | Game role, actor/target mapping, refusal, persistence |
| Native boundary | ABI, lifetime, architecture, error propagation, unload behavior |
| UI behavior | Role visibility, pending/refusal/success, disconnect states |
| Save behavior | Ownership mode, compatibility, backup/reload scenario |
| Test helper | Determinism, cleanup, no licensed fixture data |

## Tests as executable contracts

A test should assert externally meaningful behavior rather than duplicate implementation details. Use deterministic inputs, explicit cleanup, and a failure message that identifies the scenario. Prefer narrow tests for identity and serialization, then add an integration scenario for interactions across process boundaries. A passing unit test does not prove a live game hook, mod combination, or network deployment is supported.

## Error handling and logging

Validate untrusted input at the boundary, return actionable typed errors where practical, and include enough structured context for diagnosis without logging secrets. Avoid logging packet bodies, authentication material, private chat, or absolute user paths. Preserve exceptions when translating errors so they remain diagnosable.

## Naming and modularity

Choose names that reveal the domain role and authority, not merely the storage type. Keep protocol DTOs distinct from game objects and UI view models. A deep component should expose a small contract and own its invariants rather than spreading partial checks across callers.

| Prefer | Avoid |
|---|---|
| `Try...` when failure is an expected result | Throwing for routine refusal |
| Explicit result/enum for known outcomes | Parsing error text to infer a state |
| Immutable message shape | Mutating a DTO after it is queued |
| Narrow interfaces at process boundaries | Exposing implementation classes everywhere |
| A named timeout/cancellation path | Waiting indefinitely |
| Explicit role checks | Assuming a caller is always host/client |

When a method spans transport, authority, game mutation, and presentation, consider whether responsibilities belong in separate components. Do not split code mechanically: preserve a coherent transaction boundary and make it possible to reason about failures.

## Compatibility hygiene

For independently versioned peers, think about unknown fields, unknown enum values, optional data, and old clients. Define behavior for malformed and unsupported versions. Avoid changing the meaning of an existing field without a compatibility plan. Tests should include one legacy or unknown-data case where relevant.

## Small-change review questions

Before finishing, ask whether the behavior has one clear owner, whether invalid input fails closed, and whether state mutation is separated from presentation. Check cancellation and disposal paths for asynchronous work. Look for unbounded queues, retries, or waits. Verify that tests clean up sockets, processes, and temporary game state. Confirm log messages are useful without containing private payloads. These checks complement language-specific style and do not replace the repository's configured analyzers.
