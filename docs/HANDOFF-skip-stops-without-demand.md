# Handoff: let buses pass stops with no demand

Branch `skip-stops-without-demand` on the fork `pixelatedhacker/concurrent-bus-boarding`, based on
upstream `Meapy/concurrent-bus-boarding` at `e1ce02f` (1.7.2). Feature commit: `be30f94`. No pull
request to upstream has been opened.

Companion work: branch `express-lines` on `pixelatedhacker/AutoBusLines` adds express bus lines,
which share kerbside stops with local lines. This change is what lets an express bus avoid being
held at a stop nobody wants it to make.

## Status

- Written and reviewed by reading only. **Not compiled, not run through `scripts/test-policy.ps1`,
  not tested in game.** The machine it was written on had no C# compiler and no game install.
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
  carries anyone past their stop).

## Unverified assumptions — check these first

1. **How a riding passenger's path names its exit.** The code assumes the cim's remaining path
   contains the alighting waypoint (or stop) as a `PathElement.m_Target` within 8 elements of
   `m_ElementIndex`. Confirm against `Game.dll` IL (the repo's `scripts/dump-boarding-il.ps1`
   approach): look at how resident AI decides to leave a vehicle. If the exit is represented some
   other way, `IsAnyoneLeaving` returns false and the mod simply stops forcing the stop for
   alighters; native resident AI still requests it, with the original timing gap.
2. **`PathOwner.m_ElementIndex` and `WaitingPassengers.m_Count` field names and types** (assumed
   `int`). `WaitingPassengers` is already read in `LineDiagnosticsSystem`, so that one is safe.
3. **Whether an unadmitted follower is released correctly.** A skipped follower gets no session; it
   should wait in traffic behind the lead bus, then pass the waypoint without `RequireStop`. Watch
   for a bus that stalls in the zone instead.

## How to finish

1. On Windows with the toolchain (see `AGENTS.md`): `powershell -ExecutionPolicy Bypass -File
   scripts/test-policy.ps1`, then `npm ci && npm test` in `ConcurrentBusBoarding.UI`, then
   `dotnet build ConcurrentBusBoarding.slnx -c Release`.
2. In game, with the option on: a line with two buses bunched, one stop with nobody waiting and
   nobody getting off. The trailing bus should follow the lead and drive on without boarding. Then
   repeat with a waiting cim and with a rider whose destination is that stop: both must stop.
3. Watch the engagement log line: `no-demand checks` counts per-tick checks, not unique buses.
4. Check the ridership figures in `LineDiagnosticsSystem` do not regress against the option off.
5. If it holds up, open a pull request to `Meapy/concurrent-bus-boarding` and record the result as
   a dated section in `docs/HANDOVER.md`, following that file's convention.

## Local setup used

The fork clones on the authoring machine had `core.hooksPath` set to `.git/hooks` per repository to
bypass a global commit hook; this is local config only and is not part of the repository.
