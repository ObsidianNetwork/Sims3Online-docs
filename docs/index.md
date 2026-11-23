---
title: Home
hide:
  - navigation
  - toc
---

<div class="docs-home" markdown="1">
<div class="docs-hero" markdown="1">

<div class="docs-hero__copy" markdown="1">

# Sims3Online docs

Build, run, and understand an experimental multiplayer project for **The Sims 3**. The host
runs shared gameplay. Clients send player input, show the host's results, and can play local cosmetic idles.

!!! note "Naming compatibility"
    The project is branded **Sims3Online**. Solution, project, assembly, namespace, and
    environment-variable identifiers (`SimsOnline.sln`, `SimsOnlineServerHostMod`,
    `SimsOnline.Client`, `SimsOnline.Protocol`, `SIMSONLINE_*`) retain `SimsOnline*` for
    compatibility.

[Start with the Quickstart](project/quickstart.md){ .md-button .md-button--primary }
[Choose a runtime](project/runtime-modes.md){ .md-button }

</div>

<div class="docs-hero__art" aria-hidden="true" markdown="1">

![](assets/sims-online-512.png){ .docs-hero__mark width="240" height="240" }

</div>
</div>

## What are you working on?

<div class="docs-paths" markdown="1">

- [**Build and run**](development.md)

    Set up a clean checkout, build the controller and client, and prepare a local development session.

- [**Test and troubleshoot**](testing.md)

    Pick the right test for a change and learn what the result verifies.

- [**Understand the architecture**](architecture.md)

    Follow commands to the authoritative simulator and state back to each client.

</div>

!!! warning "Development software"
    Feature coverage is incomplete. The current network protocol has no authentication or
    encryption. Use a trusted network and review the [security guidance](security.md) and
    [current status](status.md) before hosting a session.

## Browse the reference

<div class="docs-reference" markdown="1">

<div markdown="1">

### Runtime and operations

- [Engineering guides](guides/index.md) and [tooling](tooling.md)
- [Server command line](server-cli.md) and [troubleshooting](troubleshooting.md)
- [Simulation](simulation-reference/index.md) and [client presentation](presentation/index.md)
- [Data, persistence, and world time](data-and-persistence/index.md)
- [Architecture deep dive](architecture-deep-dive/index.md)

</div>

<div markdown="1">

### Protocol and project

- [Network protocol](protocol.md) and [local bridge formats](bridge-and-formats.md)
- [Component reference](components.md) and [code reference](code-reference/index.md)
- [Design records](design/index.md) and [current status](status.md)
- [Project guide](project/index.md) and [documentation coverage](project/documentation-coverage.md)
- [FAQ](faq.md), [glossary](glossary.md), and [contributing](contributing.md)

</div>
</div>
</div>
