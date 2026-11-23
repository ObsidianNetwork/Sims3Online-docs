# First live multiplayer session

Start one host game and one game-backed mirror on a Windows development machine. This exercise uses the .NET controller and loopback. The first milestone is a client that joins and applies the host's state. The [starter scenarios](scenario-catalog.md#starter-scenarios) exercise behavior after that milestone.

## Prepare the two profiles

Use the [live prerequisites](prerequisites.md), including your compatible game, the locally supplied reference assemblies listed in `refs/sims3/README.md`, and a prepared Sandboxie box with its own Saves and Mods directories. The full mod package builds the presentation assembly as well as the host shell. The presentation assembly needs all ten references, including the three `simcore` assemblies. Use the same mod package in both profiles and a verified gameplay fork installation.

If you plan to run the starter scenarios, prepare the matching household and objects described in the [scenario catalog](scenario-catalog.md#starter-scenarios) first. Close both games before copying or installing files. Make a new copy of a known-good save in the host's Saves folder and give it a unique `.sims3` name. Set a separate disposable checkpoint name so the world transfer cannot overwrite your source save. The client fetches that world into its own profile.

On a shared development machine, check `python scripts/stack_claim.py status` and acquire the claim with `python scripts/stack_claim.py take` and your session and purpose before changing the mod or starting the games. The command's `--help` describes the supported session settings.

## Build and install before starting the host

Run from the repository root in PowerShell 7. Supply the executable, game profile paths, and Sandboxie box from your local setup. Each profile is the Sims 3 user folder that contains `Saves` and `Mods`.

```powershell
$gameExe = Read-Host 'Path to TS3W.exe'
$hostProfile = Read-Host 'Host Sims 3 user folder'
$clientProfile = Read-Host 'Isolated client Sims 3 user folder'
$clientBox = Read-Host 'Sandboxie box name'
$save = Read-Host 'Disposable host save folder name, including .sims3'
$saveBase = [IO.Path]::GetFileNameWithoutExtension($save)
$env:SIMSONLINE_CHECKPOINT_SAVE = "$saveBase-checkpoint.sims3"
$env:SIMSONLINE_WORLD_FROM_SERVER = '1'
$env:SIMSONLINE_SIMLINK_CLIENT_RING = '1'
$publicEndpoint = '127.0.0.1:34019'

dotnet build DedicatedServer/SimsOnline.DedicatedServer.csproj
dotnet build SimsOnline.Client/SimsOnline.Client.csproj
pwsh -NoProfile -File scripts/make-mod.ps1 -SkipInstall
pwsh -NoProfile -File SimsOnlineServerHostMod/install-server-host-mod.ps1 -Sims3UserFolder $hostProfile
pwsh -NoProfile -File SimsOnlineServerHostMod/install-server-host-mod.ps1 -Sims3UserFolder $clientProfile
```

Check each command for success before moving on. Build both managed projects before starting either controller. On Windows, a client build can try to copy the running server's executable through the project reference and fail with `MSB3021` or `MSB3027`. Boot with the binaries and game packages you just built.

### When a loopback port is in use

Set all three endpoints before boot in the same PowerShell session. Pick unused loopback ports. Launchers, health probes, scenario tools, and their subprocesses need to use the same endpoints.

```powershell
$env:SIMSONLINE_PUBLIC_ENDPOINT = '127.0.0.1:34119'
$env:SIMSONLINE_SERVER_BRIDGECTL = '127.0.0.1:34129'
$env:SIMSONLINE_CLIENT_BRIDGECTL = '127.0.0.1:34130'
$publicEndpoint = $env:SIMSONLINE_PUBLIC_ENDPOINT
```

## Start the host, then join from the client

```powershell
python scripts/stackctl.py boot-server --skip-mod-sync --save $save --ts3-path $gameExe --endpoint $publicEndpoint
python scripts/simctl.py --endpoint $publicEndpoint sims
$simId = Read-Host 'Object ID of a Sim to claim in the disposable world'
python scripts/stackctl.py boot-client --skip-mod-sync --save $save --ts3-path $gameExe --sandboxie-box $clientBox --endpoint $publicEndpoint --claim-sim $simId
python scripts/stackctl.py status --strict --endpoint $publicEndpoint
```

Confirm that the host is ready before starting the client. `--skip-mod-sync` uses the package already installed above. If boot reports `FORK-MISSING` or `FORK-UNSAFE`, check the game profiles and the local fork installation. See [compatibility](../project/compatibility-matrix.md) for the deliberately stock path. For a customized Sandboxie setup, set `SIMSONLINE_CLIENT_SAVES_DIR` to the isolated Saves directory if automatic discovery cannot find it.

If a boot command reports `listeners_not_ready` while its game is still starting, run `python scripts/stackctl.py status --strict --endpoint $publicEndpoint` before relaunching. For the host gate, require a live public listener and host bridge with the host in-world. For the client gate, require both games in-world, followed by the join and comparison below. An early timeout can leave the processes running. Read the relevant `.omo/run/server.log.err` or `.omo/run/client.log.err` if the controller exited or failed to bind.

### Confirm the join and baseline

Check these gates in order.

| Gate | Where to verify it |
|---|---|
| Host session is serving | Host state and log report the active session, with the public listener on the selected loopback endpoint. |
| Both game processes entered the world | `stackctl status --strict` probes both game bridges. Look for both loaded games with live ticks. |
| Intended player joined | The client log shows a join for its player ID, and the host confirms the accepted player. |
| Client applied the host baseline | Match the tick in `received_snapshot` (`full_snapshot`) and `snapshot_ack_sent`. Check the intervening log for `mirror_dispatch_failed` or `mirror_dispatch_partial_failure`. An acknowledgement confirms consumption. Compare the Sim in both games to verify application. |

The default run files are under `.omo/run/`. Use the complete client startup log to find the first join and snapshot once more events have arrived.

```powershell
Select-String -Path .omo/run/client.log -Pattern 'session_joined|received_snapshot|snapshot_ack_sent|mirror_dispatch_(failed|partial_failure)'
$simName = Read-Host 'Name of the claimed Sim'
python scripts/sim_compare.py $simName
```

Confirm that both games show the Sim in the same world with a comparable position and posture. Use `python scripts/simctl.py --endpoint $publicEndpoint sims` to check the claim before a player-input exercise.

## Verified session

On 23 November 2026, Windows hosted one game and one Sandboxie mirror using game build `1.67.2.024037`, controller and mod revision `f151c99b`, and a verified fork. The host loaded a disposable save, the client received a separate checkpoint, and both bridge probes reported `inWorld=True`. Player 1 joined and claimed the test Sim. The client's first full snapshot at tick `4239` had a matching acknowledgement and no recorded dispatch failures in the join check. The two-game comparison showed the same position, facing, and standing posture with zero visible mismatches.

## Stop and record the result

Run the appropriate [starter scenarios](scenario-catalog.md#starter-scenarios) while both games are up. Then run `python scripts/stackctl.py down`. On a shared rig, release the claim after confirming the stack stopped. Keep the logs and saves local. A useful [support report](../project/support-and-reporting-issues.md) records the component revision, in-game version, last successful gate, and observed scenario verdict.
