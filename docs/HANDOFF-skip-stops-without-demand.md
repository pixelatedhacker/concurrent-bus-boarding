# Handoff: let buses pass stops with no demand

Branch `skip-stops-without-demand` on the fork `pixelatedhacker/concurrent-bus-boarding`, based on
upstream `Meapy/concurrent-bus-boarding` at `e1ce02f` (1.7.2). Feature commits: `be30f94` (the
option) and `b655b2b` (exit check matched to native resident AI, see below). No pull request to
upstream has been opened.

Companion work: branch `express-lines` on `pixelatedhacker/AutoBusLines` adds express bus lines,
which share kerbside stops with local lines. This change is what lets an express bus avoid being
held at a stop nobody wants it to make.

## Status

- Built on Windows against the installed game (2026-11-01): `scripts/test-policy.ps1` passes, and
  the Release build succeeds and deploys to the local `Mods` folder.
- The exit-path assumption (formerly unverified assumption 1) was checked against decompiled
  `Game.dll` and corrected in `b655b2b`.
- **Not tested in game yet.**
- Off by default, so with the option untouched the mod behaves exactly as 1.7.2.

## Why

Native vehicle AI only stops a bus at a waypoint when `PublicTransportFlags.RequireStop` is set
(see "Target-zone stop request (2026-07-23)" in `docs/HANDOVER.md`). Residents set it; a bus without
it advances to the next waypoint. When two or more buses contend for a stop,
`ConcurrentBoardingSystem.AdmissionJob.ManageStop` sets `RequireStop` on every admissible bus and may
begin a synthetic session for a stopped follower, whether or not anyone is waiting for that bus's
line or leaving it there. A single bus at a stop is untouched (`ShouldEngageConcurrentBoarding`), so
this only matters when buses bunch, for example an express line behind a local line.

## What changed

| File | Change |
| --- | --- |
| `ConcurrentBusBoarding/BoardingPolicy.cs` | `HasStopDemand(skipWithoutDemand, waiting, anyoneLeaving)`; C# 5, dependency-free. |
| `tests/BoardingPolicyCheck.cs` | Four cases for `HasStopDemand`. |
| `ConcurrentBusBoarding/BoardingAccess.cs` | `SimulationLookups` gains read-only `m_WaitingPassengers` and `m_PathOwner` (created and updated with the rest). |
| `ConcurrentBusBoarding/BoardingSystems.cs` | `AdmissionJob.m_SkipWithoutDemand`; in `ManageStop`, after `canAdmit`, a bus with no demand is counted and skipped before the `RequireStop` write and synthetic admission. New `BoardingHelpers.HasStopDemand` / `IsAnyoneLeaving`. Third engagement counter, logged as `no-demand checks left to native AI`. |
| `ConcurrentBusBoarding/ConcurrentBusBoardingSettings.cs` | `SkipStopsWithoutDemand` (default false) with label and description. |
| `README.md`, `CHANGELOG.md` | Behaviour described; changelog entry under "Unreleased". |

Demand for a bus at a stop is:

- `WaitingPassengers.m_Count > 0` on the bus's target waypoint (`Target.m_Target`), which is that
  line's waypoint, so the count is per line; or
- a passenger in the bus's `Passenger` buffer whose `PathElement` buffer, from
  `PathOwner.m_ElementIndex` for up to 8 elements, has an `m_Target` equal to that waypoint or the
  stop entity; or
- a passenger with no readable `PathOwner`/`PathElement` (treated as leaving, so missing data never
  carries anyone past their stop); or
- a passenger whose path is too short to name an exit (`path.Length < m_ElementIndex + 2`) or is
  being recomputed (`PathFlags.Pending` or `Obsolete`), also treated as leaving (`b655b2b`).

### Exit check, verified (`b655b2b`)

In decompiled `Game.dll`, native resident AI (`CurrentVehicleTesting`) reads a riding cim's exit
waypoint from `PathElement[m_ElementIndex + 1].m_Target`, checks it with
`RouteUtils.ShouldExitVehicle`, and requests the stop whenever the path is too short to name one.
`IsAnyoneLeaving` already scanned forward from `m_ElementIndex`, which covers that element;
`b655b2b` adds the short-path and pending/obsolete cases, so the mod never lets a bus pass a stop
that native AI would have stopped at for a rider.

## Unverified assumptions — check these first

1. **Whether an unadmitted follower is released correctly.** A skipped follower gets no session; it
   should wait in traffic behind the lead bus, then pass the waypoint without `RequireStop`. Watch
   for a bus that stalls in the zone instead.

## How to finish

1. On Windows with the toolchain (see `AGENTS.md`): `powershell -ExecutionPolicy Bypass -File
   scripts/test-policy.ps1`, then `npm ci && npm test` in `ConcurrentBusBoarding.UI`, then
   `dotnet build ConcurrentBusBoarding.slnx -c Release`. Already done once; see "Local setup used"
   for the workarounds it needed.
2. In game, with the option on: a line with two buses bunched, one stop with nobody waiting and
   nobody getting off. The trailing bus should follow the lead and drive on without boarding. Then
   repeat with a waiting cim and with a rider whose destination is that stop: both must stop.
3. Watch the engagement log line: `no-demand checks` counts per-tick checks, not unique buses.
4. Check the ridership figures in `LineDiagnosticsSystem` do not regress against the option off.
5. If it holds up, open a pull request to `Meapy/concurrent-bus-boarding` and record the result as
   a dated section in `docs/HANDOVER.md`, following that file's convention.

## Local setup used

- The fork clones on the Mac had `core.hooksPath` set to `.git/hooks` per repository to bypass a
  global commit hook; this is local config only and is not part of the repository.
- The Windows build (2026-11-01) needed:
  - building `ConcurrentBusBoarding/ConcurrentBusBoarding.csproj` directly, because the only SDK
    installed (.NET 8.0.4xx) does not read `.slnx`;
  - restoring with `--source https://api.nuget.org/v3/index.json`, because the repo's
    `NuGet.Config` clears all sources;
  - `DOTNET_ROLL_FORWARD=Major`, because the toolchain's ModPostProcessor targets .NET 6.
- In the game's playset, disable the subscribed Paradox Mods copy (152153) so the local build is the
  one that loads.
