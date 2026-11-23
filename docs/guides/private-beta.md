# Private testing and beta boundaries

This repository documents local builds and tester-kit tooling; it does not publish an open enrollment page, a hosted game endpoint, or a supported public beta service. Any limited external test must be arranged by the maintainers and run against an explicitly authorized server.

## Scope before inviting testers

Use a disposable game profile and save. Decide which controller/mod build is being tested, who can reach the server, which test scenarios are in scope, and how the host will stop the session. A public client listener has no authentication; never treat an invite link or an obscure port as access control. Restrict connectivity to a private network you control and do not forward the listener to the public Internet.

The bridge-admin port is more privileged than the client session port. It must not be forwarded to a tester or exposed to the network. Testers should use the supported client path, not developer bridge controls.

## Tester material and provenance

The local kit builder supports `build`, `verify`, `check`, `list`, and `install-box`. Use `verify` to validate the manifest's file hashes. `check` additionally rejects a kit containing an EA-derived file. A kit that contains the gameplay fork is not safe to redistribute merely because it builds: rights must be confirmed first. Prefer the fork-builder mode only after reviewing all shipped third-party dependencies and the recipient's ability to supply their own game files.

Do not send game executables, saves, extracted assemblies, raw logs, or unreviewed crash dumps. The test target must supply their own legally acquired game and any local assemblies. Keep credentials and telemetry configuration outside the package.

## Collect useful feedback

Ask testers to report the build identifier, OS, the exact scenario name, the observed result, and the last confirmed join/mirror stage. Request a sanitized excerpt, not an entire save or bridge trace. An issue report should identify whether the failure reproduces without tracing and whether other participants were connected.

## Stop conditions

End a run if an unauthenticated service becomes reachable outside the intended private network, a test affects a non-disposable save, a tester sees another user's private state, or a crash suggests profile corruption. Preserve a minimal sanitized record, stop the server, and review before restarting or inviting additional testers.

## Related pages

[Tester kit](tester-kit.md) | [Release packaging](release-packaging.md) | [Logging and diagnostics](logging-diagnostics.md)
