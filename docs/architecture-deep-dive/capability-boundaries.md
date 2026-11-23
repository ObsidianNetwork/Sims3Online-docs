# Capability boundaries

The game integration exposes a useful managed gameplay surface, but not
every engine subsystem is a replaceable module. A key design boundary is
between managed script logic and native engine services. Reading a managed
API or calling a facade does not mean the underlying native service can be
rewritten or safely intercepted.

## Capability map

| Area | General access | Architectural consequence |
|---|---|---|
| Managed gameplay assemblies | Inspectable and, in selected development paths, changeable through a compatible gameplay build | Useful for policy gates and managed behavior adjustments |
| Managed facades over engine services | Callable through exposed APIs | A facade can expose a method without exposing its implementation |
| Native game core | Not ordinary application source; includes rendering, routing, scheduler, and other engine internals | Treat as a dependency, not an assumed extension point |
| Controller code | Fully under repository control | Good place for session policy, protocol, filters, and coordination |
| Host integration mod | Runs in-process with access to supported seams | Must be conservative because failures can affect the game process |

The phrase “we can modify the game” is too broad. The managed gameplay layer
may be extensible in some configurations; the native core remains a
constrained dependency. See the [component reference](../components.md) for
what the repository builds and distributes.

## Extension strategy

Prefer the narrowest boundary that can express the behavior:

1. Controller policy for session membership, command validation, interest,
   and synchronization.
2. A documented mod command or observation for managed in-game integration.
3. A shared engine interface seam for pass-through recording.
4. A compatible gameplay hook only when required behavior is unavailable
   through a stable seam.
5. Native interception only with explicit platform, version, safety, and
   compatibility evidence.

This order limits invasive changes and clarifies who owns failures. A
controller policy can be unit tested; an in-process hook needs stronger
startup and live-game validation.

## Native calls are not ordinary managed calls

Managed exception handling cannot be assumed to contain a native access
violation. A facade may route directly into native code, and a bad argument
or invalid runtime state can terminate the game process. Probes should use
known-safe operations, narrow inputs, and a disposable test world where
possible.

## Bitness and address space

The game process is 32-bit. Large-address-aware behavior can raise the
available user address space in supported environments, but it does not make
the game 64-bit and does not increase bridge throughput or simulation
determinism. Memory headroom is a lifetime constraint to measure, not a
substitute for profiling the actual bottleneck.

## What an isolated runtime experiment proves

An offline or reduced harness may establish that a specific managed method
can execute under the harness conditions. It does not automatically
establish that:

- The full game world can initialize.
- Native services are present and initialized.
- Simulation scheduling and world time advance.
- Routing, save/load, rendering, or all internal calls are available.
- The harness can replace a live server process.

Each broader claim needs independent evidence against the behavior it
promises.

## Capability review checklist

- Is the needed behavior in repository-controlled controller code, managed
  gameplay, a facade, or native engine code?
- Is the seam public and stable, or dependent on reflected private state?
- Does the change run inside the game process?
- What happens on a version mismatch or missing hook?
- Can failures be contained, or can they terminate the game?
- Are asset, assembly, and redistribution rights resolved for public
  release?
- Is the claim tested in the actual game configuration it targets?

## Failure modes / limits

- A managed stub can expose a signature without implementing a useful native
  operation.
- A gameplay fork can be incompatible with a different game patch or
  expansion set.
- Native calls can fail outside managed exception handling.
- 32-bit address-space limits remain even when CPU usage is modest.
- Offline harness success does not prove a production server replacement.

## Related pages

- [Dedicated server and offline host](dedicated-server-and-offline-host.md)
- [Boundary seams](boundary-seams.md)
- [Capacity and performance](capacity-and-performance.md)
- [Components](../components.md)

## Managed and native boundary in practice

The managed gameplay assemblies contain many rules that can be inspected
and, in a controlled compatible build, changed. ScriptCore and SimIFace
provide managed call surfaces over engine functionality. Those APIs do not
imply that their underlying native implementations can be replaced.

| Question | Managed gameplay answer | Native engine answer |
|---|---|---|
| Can code be inspected? | Usually through managed assembly tooling | Not as ordinary source code |
| Can behavior be changed? | Sometimes by a compatible fork or script integration | Only through exposed call surfaces or carefully versioned native integration |
| Can a managed exception contain failure? | Often for managed failures | Not necessarily for native faults or access violations |
| Can a controller substitute a new implementation? | For controller-owned policy | Not for hidden engine internals without replacing their contracts |

This boundary makes an engine facade both valuable and limited: it lets
integrations invoke supported operations, but does not grant ownership of
the service implementation.

## Selecting an extension seam

Use this decision sequence before proposing a hook:

```text
Is the rule session/network policy? -> controller
Is the state available from a documented in-game API? -> host mod observation/command
Do multiple producers call a shared interface? -> pass-through recorder at that seam
Is behavior inaccessible through a stable seam? -> assess a managed gameplay hook
Does it require native internals? -> stop and define platform/version/safety evidence
```

A change at a lower layer has a larger compatibility and failure surface.
Prefer controller policy for ownership and filters; use in-process hooks
only when necessary to control actual game behavior.

## Native boundary risk model

An in-process managed mod shares the game process. A bug in a probe,
reflection assumption, hook installation, or marshaling call can corrupt or
terminate that process. Native faults are not reliably transformed into a
catchable managed exception. Therefore, use disposable worlds, validate
exact versions, keep the hook narrow, and provide a clean path when the
expected seam is absent.

The game is also 32-bit. Address-space pressure can cause failures unrelated
to CPU load or network throughput. Large-address-aware settings may change
available virtual address space on some platforms, but do not alter the
fundamental process bitness or remove memory leaks.

## Compatibility layers

Compatibility has several axes, not one version string:

- Game executable and patch version.
- Managed gameplay assembly version or fork overlay.
- Host mod build and bridge layout.
- Controller protocol and serializer version.
- Optional native sidecar build and patch compatibility.
- Save/world and content compatibility.

A deployment can pass one handshake and still be incompatible at another
layer. Report each version boundary separately where possible.

## Risk mitigation table

| Risk | Mitigation |
|---|---|
| Private field no longer exists | Detect absence and report unsupported seam |
| Hook installed twice | Make installation idempotent and verify wrapper identity |
| Wrong game build | Check version/hash policy before invasive operation |
| Native call faults | Restrict inputs; run disposable live validation; avoid risky probes in production |
| Fork mismatch | Fail clearly when expected gate/capability is absent |
| Memory exhaustion | Track process memory and run long-session tests |
| Optional sidecar mismatch | Keep feature default-off unless build and executable match |

## How to verify / test

- Unit-test controller logic without game internals where possible.
- Validate managed method signatures and bridge codec independently.
- Run startup checks against the expected game build before applying hooks.
- Test missing/reflection-failed seams and ensure a clear unsupported
  status.
- Use process health and crash evidence for native-facing changes.
- Record whether a result was static analysis, managed test, harness test,
  or live game observation.

## Failure modes / limits

A source-level hook or successful assembly build is not proof that a live
game call path reaches it. Conversely, an offline runtime can execute
managed code while lacking native dependencies required for full simulation.
State the evidence level for every capability claim.

## Related pages

- [Capability boundaries](capability-boundaries.md)
- [Boundary seams](boundary-seams.md)
- [Dedicated server and offline host](dedicated-server-and-offline-host.md)

## Capability decision examples

**Need to reject a command from a non-owner?** Implement this in the
controller session policy. It is ordinary repository-controlled logic and
does not require an engine hook.

**Need to observe a Sim's current motive?** Prefer a supported in-game probe
or snapshot extractor, then define how the value is represented and applied
on clients.

**Need to reproduce an animation request made by many gameplay systems?**
Identify a shared engine interface and record the call there, with an
explicit pass-through contract.

**Need to change the native pathfinding solver?** A managed `Route` facade
may expose calls into the native solver, but that does not make the solver
an ordinary managed extension point. Either use the available API or treat
replacement as a native research and compatibility effort.

This decision sequence is a way to locate the work, not a promise that every
desired behavior has a supported seam.

## Capability evidence record

For each capability claim, record:

| Field | Example value |
|---|---|
| Target layer | Controller / managed gameplay / facade / native core |
| Access method | Unit-tested class / bridge command / reflected seam / native sidecar |
| Evidence type | Static inspection / automated test / offline harness / live game |
| Version boundary | Game build, mod build, bridge layout, sidecar revision |
| Failure containment | Clear unsupported state / process-level risk |
| Known omissions | Not covered by current test |

This record prevents a method signature or a green build from being
presented as proof of end-to-end capability.

## Deployment compatibility

The capability surface depends on the exact game installation and
integration artifacts. A public package should identify required game
version and expansion/content assumptions without distributing proprietary
game binaries or extracted implementation. Compile-time reference stubs, if
present, must be separately reviewed for licensing and redistribution before
publication.

The code may support a default path without an optional sidecar, while a
gameplay fork remains behaviorally important for authority enforcement.
These requirements should be stated precisely: “optional build component”
and “optional behavior” are not equivalent.

## Failure reporting

When a capability is unavailable, the integration should distinguish among:

- The API or reflected seam is absent.
- The seam exists but has no current engine instance.
- The hook was not built or loaded.
- The bridge peer does not advertise the capability.
- The capability is present but the operation failed at runtime.
- The engine completed the operation but the observation path missed it.

Each state suggests a different remediation. Combining them into one generic
“not supported” message slows diagnosis.

## How to verify / test

- Inspect the target boundary and identify which side owns its behavior.
- Test supported and unsupported version paths.
- Confirm capability negotiation reports actual in-memory state, not only
  files on disk.
- Run an operation through the public controller path and observe its game
  effect.
- Ensure a failed native-facing probe does not corrupt a real save or
  publish false success.
- Validate distribution inputs and compile-time references before public
  packaging.

## Failure modes / limits

The evidence map can become stale as integration code changes. Recheck both
the caller and callee rather than citing a historical study as proof of
current behavior.

## Related pages

- [Runtime components](runtime-components.md)
- [Boundary seams](boundary-seams.md)
- [Development](../development.md)
