# Test map

The repository has three different .NET test/build surfaces and a Python tooling suite.
Choose the runner based on the project file, not just the directory name: the custom C#
runner is not VSTest, while `RustAuthoritySidecarTests.cs` is an NUnit/VSTest project.
Python tests use pytest's `*_test.py` naming rule.

## Quick commands

| Goal | Command | What it runs |
|---|---|---|
| Complete Python/PowerShell tooling checks | `python scripts/dev.py check` | The documented aggregate: lint, then test/self-check groups. |
| Python/tooling tests only | `python scripts/dev.py test` | CI's `python-tools` test command. |
| Python lint and PowerShell parsing | `python scripts/dev.py lint` | CI's lint command. |
| One Python test file | `python -m pytest scripts/bridgectl_test.py` | Example: one script's focused pytest module. Replace with an existing `_test.py` path. |
| All Python tests under scripts and tools | `python -m pytest scripts tools` | Explicit test roots; `pyproject.toml` configures `*_test.py`. |
| Custom .NET case runner | `dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter all` | All cases registered in the reflection runner. |
| Filter custom .NET cases | `dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter PublicLineCodec` | Any case name containing `PublicLineCodec`, case-insensitive. |
| Fail if selected tests skip | `dotnet run --project DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj -- --filter SimLink --fail-on-skip` | Used by Windows SimLink validation. |
| NUnit/VSTest project | `dotnet test DedicatedServer.Tests/DedicatedServer.Tests.csproj` | Runs the sidecar NUnit cases via test adapter. |

Install Python development dependencies from `requirements-dev.txt` before invoking pytest
or `scripts/dev.py`; the CI workflow installs that file. `pyproject.toml` sets `python_files
= ["*_test.py"]`, `pythonpath = [".", "scripts"]`, and `testpaths = ["scripts"]`. Tests in
`tools/` can be passed explicitly as a path/root.

### Custom runner behavior

`DedicatedServer.Tests/SimsOnline.DedicatedServer.Tests.csproj` builds an executable whose
`Program` discovers methods marked by its local `[Test("case.name")]` attribute. The runner
sorts names, executes cases in sequence, and prints `PASS`, `FAIL`, or `SKIP` followed by
`RESULT total=N failed=M skipped=K`. `--filter` matches a substring in the full test case
name; a typo that matches nothing exits 2. An unknown option also exits 2. A skipped case
does not fail by default; `--fail-on-skip` turns any selected skip into a failing run. This
distinction matters on Windows-only SimLink tests and in environments without the
game-native prerequisites.

`DedicatedServer.Tests/Program.cs` contains a large set of test methods and local fake host
implementations; it is not one conventional class per test file. Smaller C# files hold
focused cases. `DedicatedServer.Tests/DedicatedServer.Tests.csproj` disables default compile
items and includes only `RustAuthoritySidecarTests.cs`; its NUnit packages are separate from
the custom runner.

## C# test cases by feature

The listed file names are concrete tests, not inferred test namespaces. The custom suite
links pure source files from the client and host mod so tests exercise the same logic the
production projects use.

| Feature | Test files / test classes |
|---|---|
| TCP listener, line bounds, outbound and connection lifecycle | `OutboundListenerTests.cs`, `ConnectionCloseLogTests.cs`, `BridgeAdminListenerTokenTests.cs`, `BridgeEventDropTests.cs` |
| Public protocol and JSON compatibility | `PublicLineCodecTests.cs`, `ProtocolHardeningTests.cs`, `WireJsonPinningTests.cs`, `ClientWireLineTests.cs` |
| Client receive and mirror coordinator | `ClientReceiveDrainTests.cs`, `CoordinatorTimerTests.cs`, `OrderCompletionReassert` cases in `Program.cs` |
| Host bridge send/shutdown and response handling | `HostBridgeShutdownTests.cs`, `BridgeParityTests.cs`, `JoinTraceTests.cs` |
| Session, join, permissions and claims | `JoinCapabilityTests.cs`, `JoinWorldReuseTests.cs`, `LeaveConversationTests.cs`, session cases in `Program.cs` |
| Save/checkpoint/recovery | `CheckpointSaveTests.cs`, `LastGoodSaveTests.cs` |
| Presentation, join settling and state repair | `JoinSettlePlanTests.cs`, `IdleRepairPlanTests.cs`, `SelectionGuardTests.cs`, presentation cases in `Program.cs` |
| Interaction and server-host policy | `OnScreenWalkTests.cs`, `RepairEntryGateTests.cs`, `RepairLabelTests.cs`, `SwallowedErrorTests.cs`, interaction cases in `Program.cs` |
| Telemetry and diagnostics | `SentryTelemetryTests.cs`, `BridgeAdminListenerTokenTests.cs`, `ConnectionCloseLogTests.cs` |
| SimLink native/shared-memory boundary | `SimLinkTests.cs`, `DedicatedServer.Tests/Program.cs` SimLink cases |
| Rust authority adapter and routing | `RustAuthoritySidecarTests.cs` in the NUnit project; `RustRoadPlaceTests.cs`, `RustRouteSeamTests.cs`, `RustRouteWaitTests.cs` in the custom runner |
| Mod policy source-linked into tests | `AutonomyCommandGate`, `ClientCaptureFilter`, `UnclaimedRoutingEmitFilter`, `OutboundLaneQueue`, replay/cache, presentation, line codec, and snapshot cases in `Program.cs` |

Examples of case names from current sources:

| Case name | What it pins |
|---|---|
| `PublicLineCodec.AllRegisteredKindsAndConformanceFixtures` | Registered line-v1 commands/events and fixture conformance. |
| `ClientListenerServer.CloseLogNamesAnOversizedLine` | Oversized public input is classified and logged. |
| `HostBridgeServer.DisposeCancelsPendingSend` | Disposal cancels an outstanding correlated send. |
| `MirrorCoordinator.PostedWorkRunsInLineOrder` | Posted work executes on the drain loop in sequence. |
| `JoinCapability.ServerListsTheStreamOnlyWhenItRuns` | Capabilities describe active server streams. |
| `JoinWorldReuseTests` cases | Join reuse/checkpoint semantics. |
| `CheckpointSave.JoinWritesSeparateSlotAndLastGood` | Join checkpoint and recovery copies are distinct. |
| `SimLink` cases | Shared memory/open/reader behavior, including platform-gated tests. |
| `RustAuthoritySidecarTests` cases | Rust sidecar handshake and command adapter replies. |

A subset filter works on test case names rather than source file paths. For example,
`--filter BridgeParity` selects parity cases, `--filter ClientListenerServer` selects
listener cases, and `--filter JoinCapability` selects capability cases. `--filter all` is
the explicit full-suite selector.

## Python tests by area

There are many focused tooling tests; run the module named by the utility being changed
before using the aggregate. These representative current `*_test.py` modules under
`scripts/` and `tools/` are grouped by what they exercise.

| Area | Python test modules |
|---|---|
| Build and developer command helpers | `build_slot_test.py`, `dev_test.py`, `dotnet_warnings_test.py`, `fork_builder_test.py`, `install_nointro_test.py`, `mod_catch_test.py`, `mod_sources_test.py`, `monohost_test.py`, `reference_assembly_test.py`, `refpack_test.py` |
| Bridge, sidecar, and transport | `bridgectl_test.py`, `bridge_record_replies_test.py`, `sidecar_test.py`, `simlink_offline_gates_test.py` |
| Documentation and source paths | `check_docs_test.py`, `doc_paths_test.py`, `doc_refs.py` (helper), `scenarios_test.py` |
| Client input and integration probes | `client_pick_test.py`, `compare_client_test.py`, `probe_check_test.py`, `probe_social_session_test.py`, `join_fingerprint_test.py`, `join_race_watch_test.py`, `simctl_test.py`, `tester_kit_test.py` |
| State, saves, and DBPF tooling | `dbpf_grep_test.py`, `dbpf_layout_test.py`, `household_read_test.py`, `inventory_diff_test.py`, `object_diff_test.py`, `persist_cycle_test.py`, `saves_test.py` |
| Diagnostics, logs, and process helpers | `boot_profile_test.py`, `crash_catch_test.py`, `crash_ledger_test.py`, `diag_state_readers_test.py`, `diag_state_test.py`, `machine_load_test.py`, `proc_timeline_test.py`, `pump_ms_test.py`, `run_watchdog_test.py`, `stream_gaps_test.py`, `stream_kinds_test.py` |
| Mod/runtime and presentation tooling | `gen_boundary_recorder_test.py`, `idle_repair_probe_test.py`, `menu_diff_test.py`, `motion_track_test.py`, `motion_watch_test.py`, `presentation_compare_test.py`, `route_continuity_test.py`, `tune_test.py` |
| Native observatory/source analysis | `native_byte_classify_test.py`, `native_decompile_export_test.py`, `native_defect_repair_test.py`, `native_name_evidence_test.py`, `native_observatory_anchor_admission_test.py`, `native_observatory_anchor_test.py`, `native_observatory_archive_test.py`, `native_observatory_campaign_test.py`, `native_observatory_command_recorder_test.py`, `native_observatory_compare_test.py`, `native_observatory_ghidra_smoke_test.py`, `native_observatory_headless_test.py`, `native_observatory_icall_map_test.py`, `native_observatory_identity_test.py`, `native_observatory_inventory_test.py`, `native_observatory_normalization_test.py`, `native_observatory_preflight_test.py`, `native_observatory_schema_test.py`, `native_observatory_source_guards_test.py`, `native_observatory_toolchain_test.py` |
| Native/game data inspection | `game_decompile_inventory_test.py`, `gc_heap_test.py`, `gc_watch_test.py`, `jazz_dump_test.py`, `sacs_names_test.py`, `sacs_picks_test.py`, `sacs_random_test.py`, `sacs_timing_test.py`, `ts3src_test.py`, `ts3w_test.py`, `ts3_view_test.py` |
| Code comparison, corpus, and mapping | `decompile_lane_test.py`, `loop_variant_table_test.py`, `managed_decompile_recover_test.py`, `managed_recompile_test.py`, `seam_map_test.py`, `simnav_export_test.py`, `sim_act_test.py`, `sim_compare_test.py` |
| Session, trace, and replay tools | `arms_lib_test.py`, `arms_values_test.py`, `arm_readout_test.py`, `crash_ledger_test.py`, `diag_state_test.py`, `rejection_relay_probe_test.py`, `replay_check_test.py`, `rig_cargo_test.py`, `session_factor_test.py`, `stack_claim_test.py`, `stack_who_test.py`, `stackctl_pid_test.py` |
| Rust/kernel workflow fixtures | Data-driven suites also exercise preflight gating, archive admission, catalog/schema validation, mapping queries, reproducibility, managed/native inventory, registration, and scenario campaigns. Their module-level tests run through the same pytest command; use the focused files under `scripts/` when changing those utilities. |
| Miscellaneous structured data/notebook helpers | `notebook_knowledge_test.py`, `notebook_sync_test.py`, `build_slot_test.py`, `gc_heap_test.py`, `gc_watch_test.py`, `arm_readout_test.py`, `arms_lib_test.py`, `arms_values_test.py` |

The table is grouped by likely debugging entry point, not by dependency or test-order
guarantee. A shared module can appear in more than one topic; run `scripts/dev.py test` for
the maintained aggregate rather than assuming a table row is a complete regression boundary.

### `tools/` test modules

The current `tools/` test files are:

| Module | Area |
|---|---|
| `tools/colab_offload/colab_config_test.py` | Configuration handling for the offload helper. |
| `tools/colab_offload/extract_pilot_anchors_test.py` | Anchor extraction. |
| `tools/colab_offload/ghidra_to_rust_colab_test.py` | Ghidra export/offload conversion. |
| `tools/colab_offload/sync_sim96_architect_test.py` | Synchronization helper. |
| `tools/research/notebooklm_deep_research_test.py` | Research integration helper. |
| `tools/tester_kit/wire_test.py` | Tester-kit wire framing. |

Some scripts in `scripts/` and tools have self-test or `--help` checks without a matching
pytest file; these are driven by `scripts/dev.py`. Do not infer “untested” solely from
absence of a sibling `_test.py`.

## CI and project boundaries

`.github/workflows/build-and-test.yml` defines the maintained CI commands: it builds the
client, test client, and custom test project; runs the custom C# suite; installs Python
development requirements; then runs Python lint/test. The `sidecar-tests` job builds the Rust
authority daemon and runs the full NUnit sidecar project on Ubuntu for each pull request.
Separate jobs compile the full host
mod on Windows and build/run the SimLink tests on Windows. The standard custom test runner
does not compile or boot the proprietary game; mod compilation uses the compile-time
reference assemblies and live game testing is a separate capability.

`DedicatedServer.Tests/DedicatedServer.Tests.csproj` is specifically the VSTest/NUnit
adapter project for the sidecar adapter source. It compiles only
`RustAuthoritySidecarTests.cs` and uses NUnit, NUnit3TestAdapter, and
Microsoft.NET.Test.Sdk. The sibling `SimsOnline.DedicatedServer.Tests.csproj` is an
executable with a custom reflection runner and is not discoverable as a VSTest suite.

For source ownership and target frameworks see
[build-and-solution-layout.md](build-and-solution-layout.md). For controller behavior and
event delivery see [dedicated-server.md](dedicated-server.md). For client event/queue tests
see [client-and-test-client.md](client-and-test-client.md).

## Selected C# cases and what they protect

The custom suite has both pure policy tests and integration-shaped tests with fake
host/network objects. Representative named assertions include:

| Test name | Regression it detects |
|---|---|
| `BridgeAdminListener.ADashIsAnEmptyPartAndAnEscapedDashIsLiteral` | Admin command argument parsing distinguishes empty argument from literal dash. |
| `BridgeLineCodec.DroppedModEventLoggedOnceCountedEachTime` | A dropped host event is logged/counted with the intended cardinality. |
| `BridgeShared.ModLayoutConstantsMatchController` | Controller and mod shared-region layout constants remain aligned. |
| `BridgeShared.ModCrc32MatchesController` | CRC behavior matches across controller and mod. |
| `CheckpointSave.JoinWritesSeparateSlotAndLastGood` | Join checkpoint, alternate slot, and last-good save stay distinct. |
| `SimsOnlineClient.LineResumeKeepsCompletedCursorAndCommandIdentity` | A warm line reconnect keeps cursor and command identity. |
| `SimsOnlineClient.LineWorldFetchHandshakesAndReassemblesRawBulk` | World fetch handshakes and completes raw-bulk assembly. |
| `MirrorCoordinator.PostedWorkRunsInLineOrder` | External work is serialized through the event drain loop. |
| `MirrorCoordinator.JoinDrainRunsPostedWorkBetweenLines` | Join-time drain does not strand posted work. |
| `HostBridgeServer.DisposeWakesSendWaitingForWriteLock` | Disposal wakes a send waiting for the serialized writer. |
| `ClientListenerServer.CloseLogNamesTheSupersedingConnection` | Connection replacement is visible in close diagnostics. |
| `JoinCapability.ClientNeverStartsReplayWhenTheServerStreams` | Client capability handling does not start a conflicting replay path. |
| `JoinSettlePlan.NonSeatDriftSettlesAfterConsumedJoin` | Join settling distinguishes a consumed join from ongoing drift. |
| `IdleRepairPlan.StreamedIdleWithoutStateEntersStandingIdle` | A presentation repair has an explicit fallback when state is missing. |

When a file contains multiple methods, the runner's case names are often more precise than
the file name. Use `--filter <case fragment>` and confirm the printed `RESULT total=...`
count is nonzero before relying on the result.

## Python tooling test areas in more detail

The Python tests are script-level checks, not one monolithic application suite. The module
to run normally mirrors the utility name (`foo.py` → `foo_test.py`). The following groups
are concrete entry points:

| Utility area | Example test modules | Typical assertion boundary |
|---|---|---|
| Docs path/link validation | `check_docs_test.py`, `doc_paths_test.py` | Missing paths, anchors, and document references. |
| Public bridge CLI | `bridgectl_test.py`, `bridge_record_replies_test.py` | Argument parsing and recorded reply handling. |
| Sidecar build and availability | `sidecar_test.py`, `simlink_offline_gates_test.py` | Build command behavior and safe no-sidecar paths. |
| Save inspection and persistence | `saves_test.py`, `persist_cycle_test.py`, `dbpf_layout_test.py`, `dbpf_grep_test.py` | Save folder/index boundaries and repeatable read/diff results. |
| Households/inventory/object data | `household_read_test.py`, `inventory_diff_test.py`, `object_diff_test.py` | Structured extraction and comparison helpers. |
| State diagnosis | `diag_state_test.py`, `diag_state_readers_test.py`, `probe_check_test.py` | State report parsing and diagnostic checks. |
| Session/client utilities | `simctl_test.py`, `client_pick_test.py`, `compare_client_test.py`, `join_fingerprint_test.py` | Client/session input and comparison helpers. |
| Scenarios and test harnesses | `scenarios_test.py`, `tester_kit_test.py`, `quick_loop_test.py` | Scenario metadata and harness behavior. |
| Native evidence/source gates | `native_observatory_preflight_test.py`, `native_observatory_schema_test.py`, `native_observatory_source_guards_test.py`, `native_observatory_identity_test.py` | Admission, schema, source-safety and identity checks. |
| Native data helpers | `native_byte_classify_test.py`, `native_name_evidence_test.py`, `native_defect_repair_test.py`, `native_decompile_export_test.py` | Normalized evidence and helper outputs. |
| Process and timing diagnostics | `proc_timeline_test.py`, `machine_load_test.py`, `pump_ms_test.py`, `run_watchdog_test.py` | Time-series extraction and process monitoring. |
| Routing/presentation diagnostics | `route_continuity_test.py`, `presentation_compare_test.py`, `motion_track_test.py`, `motion_watch_test.py` | Presentation and route comparison tools. |
| Protocol/replay diagnostics | `replay_check_test.py`, `stream_gaps_test.py`, `stream_kinds_test.py`, `rejection_relay_probe_test.py` | Event sequence gaps, type counts, replay and rejection parsing. |
| Managed assembly tooling | `managed_decompile_recover_test.py`, `managed_recompile_test.py`, `reference_assembly_test.py` | Metadata extraction/recompile helper behavior. |
| Reports and local utilities | `crash_ledger_test.py`, `crash_catch_test.py`, `boot_profile_test.py`, `run_watchdog_test.py` | Report aggregation and startup/process probes. |
| Developer infrastructure | `dev_test.py`, `dotnet_warnings_test.py`, `build_slot_test.py`, `fork_builder_test.py` | The developer runner and build helper contracts. |
| Notebook and text processing | `notebook_knowledge_test.py`, `notebook_sync_test.py` | Input indexing and synchronization. |
| Low-level gameplay diagnostics | `sacs_names_test.py`, `sacs_picks_test.py`, `sacs_random_test.py`, `sacs_timing_test.py`, `gc_heap_test.py`, `gc_watch_test.py` | Parsers/reporters for the matching diagnostic artifacts. |

The `tools/` tests are separately discoverable by passing the tools directory to pytest.
They cover configuration and data extraction in `tools/colab_offload/`, notebook research
integration in `tools/research/`, and wire framing in `tools/tester_kit/`. These are not
required for a typical controller-only edit, but a change in those directories should run
the corresponding module.

## Subset strategy

Use one test command that covers the smallest boundary changed, then expand when the feature
crosses a project seam:

* A pure filter/gate edit: custom .NET `--filter` on the case name, followed by full suite
  if it changes accepted events or authority.
* A DTO/codec edit: `PublicLineCodec` / wire pinning subset, client wire subset, then full
  C# suite.
* A controller listener/queue edit: listener or outbound test case names, then the full
  custom suite.
* A Python utility: its sibling `_test.py`, then `python scripts/dev.py test` when shared
  utilities change.
* A CMake/C++ edit: Windows sidecar build and native `SimLink` tests with `--fail-on-skip`.
* A full mod edit: Windows compile gate; source-linked tests alone cover only pure policy
  helpers.

Do not interpret `SKIP` as a pass in platform-specific validation. The custom runner returns
a successful exit for skips unless `--fail-on-skip` is supplied. CI uses that stricter mode
for the SimLink-specific Windows job.

## Additional script test modules

Several focused tests fall outside the representative table above. These names are current
Python modules under `scripts/`; invoke each with `python -m pytest scripts/<module>`.

| Tool family | Test modules |
|---|---|
| Arm/trace value readers | `arm_readout_test.py`, `arms_lib_test.py`, `arms_values_test.py`, `motion_track_test.py`, `proc_timeline_test.py` |
| Recording and reply parsers | `bridge_record_replies_test.py`, `crash_ledger_test.py`, `stream_gaps_test.py`, `stream_kinds_test.py` |
| DBPF, export and package inspection | `dbpf_grep_test.py`, `dbpf_layout_test.py`, `game_decompile_inventory_test.py`, `jazz_dump_test.py`, `reference_assembly_test.py` |
| Installation and mod source tools | `install_nointro_test.py`, `mod_sources_test.py`, `mod_catch_test.py`, `fork_builder_test.py` |
| Network/session diagnostics | `join_race_watch_test.py`, `join_fingerprint_test.py`, `rejection_relay_probe_test.py`, `session_factor_test.py`, `simctl_test.py` |
| Build and runtime instrumentation | `boot_profile_test.py`, `machine_load_test.py`, `pump_ms_test.py`, `run_watchdog_test.py`, `dotnet_warnings_test.py` |
| Native boundary parsers | `sacs_names_test.py`, `sacs_picks_test.py`, `sacs_random_test.py`, `sacs_timing_test.py`, `gc_heap_test.py`, `gc_watch_test.py` |
| Data mapping and extraction | `managed_decompile_recover_test.py`, `managed_recompile_test.py`, `native_observatory_icall_map_test.py`, `native_observatory_inventory_test.py`, `simnav_export_test.py` |
| Policy/report comparisons | `compare_client_test.py`, `inventory_diff_test.py`, `object_diff_test.py`, `presentation_compare_test.py`, `route_continuity_test.py` |
| State and persistence readers | `diag_state_readers_test.py`, `household_read_test.py`, `persist_cycle_test.py`, `saves_test.py`, `ts3_view_test.py` |
| Local harnesses | `quick_loop_test.py`, `tester_kit_test.py`, `scenarios_test.py`, `build_slot_test.py` |
| Supporting repository checks | `check_docs_test.py`, `doc_paths_test.py`, `dev_test.py`, `stack_claim_test.py`, `stack_who_test.py`, `stackctl_pid_test.py` |

For data-heavy test helpers, a pytest pass validates the parser/transform contract against
fixtures. It does not claim that a native binary, live process, or game install was present.
Check the test's skip conditions and fixture setup when a live-engine assumption matters.

The full `python scripts/dev.py test` command also runs non-pytest self-checks in addition
to pytest. CI invokes that same entry point after lint. This is the intended broad
validation for `scripts/` and `tools/`; use pytest directly when iterating on one module.
