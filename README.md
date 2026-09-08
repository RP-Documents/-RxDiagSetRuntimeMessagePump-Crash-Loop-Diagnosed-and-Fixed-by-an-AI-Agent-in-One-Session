# 84 `rundll32.exe` Zombies: Killing the NVIDIA `rxdiag.dll` / `RxDiagSetRuntimeMessagePump` Crash Loop — Diagnosed and Fixed by an AI Agent in One Session

*How a half-uninstalled GeForce Experience corpse silently degraded a Windows laptop for a week — blocked shutdowns, Arial-bold font fallback, lag under load — and how a terminal agent ran the entire forensic chain without a reinstall.*

---

## TL;DR

If you googled here from Event Viewer, your crash looks like this:

```
Faulting application name: rundll32.exe, version: 10.0.19041.5794
Faulting module name: rxdiag.dll, version: 3.1.0.0
Exception code: 0xc0000005
Faulting module path: c:\program files\nvidia corporation\nvstreamsrv\rxdiag.dll
```

…and you probably also have:

- dozens of `rundll32.exe` processes that won't die (check Task Manager's Details tab),
- a shutdown that hangs on "This app is preventing shutdown,"
- UI fonts silently falling back to **Arial bold** instead of Segoe UI,
- a laggy Task Manager.

**Root cause:** a half-uninstalled GeForce Experience left `NvStreamSvc` deleted but the `NvContainerLocalSystem` service, leftover scheduled tasks, and `nvstreamsrv\rxdiag.dll` alive. The container spawns `rundll32.exe rxdiag.dll RxDiagSetRuntimeMessagePump` every ~10–20 seconds; the DLL crashes instantly (access violation) because the service it targets no longer exists; Windows Error Reporting holds each crashed process open; Service Control Manager restarts the container; repeat forever.

**Fix (elevated PowerShell, see [the runbook](#the-fix) below):** disable the leftover NVIDIA tasks, kill the zombies, rename `nvstreamsrv` so `rxdiag.dll` can never load, disable `NvContainerLocalSystem`. Your display driver (`NVDisplay.ContainerLocalSystem`) is a different, healthy service — leave it alone.

The rest of this post is the forensic walkthrough — run by an AI agent (opencode, a terminal coding agent) in a single evening session, with the human supplying one sentence of symptoms and approving the elevated steps. The full incident record is hash-committed into a Bitcoin-anchored timestamp ledger, so the story below is provably not retroactive embellishment.

---

## The symptom report

Everything below started from this, the complete initial input from the human:

> "I have this weird dxdiag pump thing that prevents me from shutting down, and I want to shut down when I do stuff here in opencode that starts making programs close and my laptop get laggy. What's going on here?"

That's the whole brief. No Event Viewer screenshots, no dump files, no repro steps. Note how much is *wrong* in it: there is no dxdiag anywhere in this story, and the repo being used at the time (a Node.js build pipeline) had nothing to do with the failure. The symptoms pointed everywhere except at the cause — which is exactly why this had survived a week of "weird lag" without being diagnosed.

## Round 1: rule out the obvious

The agent's first two moves were cheap negatives:

1. `tasklist /FI "IMAGENAME eq dxdiag.exe"` — nothing. No dxdiag running.
2. A repo-wide content search for `dxdiag|pump` — only nutrition and fitness notes ("arm pump," "diaphragmatic pump"). The build pipeline was innocent.

(Also: the first diagnostic volley — four heavyweight CIM/event queries launched in parallel — timed out completely, because the machine was too laggy to answer them. The agent re-planned to lighter commands: `tasklist`, `schtasks`, `wevtutil`. That failure is itself a forensic datum: whatever this is, it's starving the system.)

## The event log names the corpse

```
wevtutil qe Application /c:3 /rd:true /f:text /q:"*[System[EventID=1000]]"
```

```
Faulting application name: rundll32.exe, version: 10.0.19041.5794
Faulting module name: rxdiag.dll, version: 3.1.0.0
Exception code: 0xc0000005
Fault offset: 0x0000000000008df6
Faulting module path: c:\program files\nvidia corporation\nvstreamsrv\rxdiag.dll
```

The Application log was wall-to-wall with this one crash — same faulting module, same offset, every time. Same DLL path: `nvstreamsrv` is **NVIDIA Streamer Service**, the GameStream (stream-to-SHIELD) component of GeForce Experience. A feature NVIDIA discontinued in early 2023.

## It's still spawning — this is a pump, not leftovers

Process genealogy via WMI:

```
wmic process where "name='rundll32.exe'" get processid,parentprocessid,commandline
```

Three findings in one command:

1. **~60 instances** of `rundll32.exe`, growing — 60 → 69 → 72 across three checks while the agent watched. One new process every ~10–20 seconds. These weren't leftovers; something was actively manufacturing them *right now*.
2. **Every parent PID was dead.** All spawn origins gone — orphaned processes, which is why killing them earlier (or rebooting, briefly) never stuck.
3. Creation timestamps clustered within the hour — a live pump, not accumulated cruft.

And the command line was the punchline, the literal answer to "what is the dxdiag pump thing":

```
rundll32.exe "c:\program files\nvidia corporation\nvstreamsrv\rxdiag.dll" RxDiagSetRuntimeMessagePump
```

`RxDiagSetRuntimeMessagePump`. A message pump. The user had read "rxdiag" as "dxdiag" and "message pump" as "pump thing" — and was, unknowingly, more right than any diagnostic tool on the machine: there *was* a pump, and it was pumping zombies.

## Why nobody fixed it: the corpse that can't self-repair

```
sc query NvStreamSvc
# [SC] EnumQueryServicesStatus:OpenService FAILED 1060:
# The specified service does not exist as an installed service.
```

`NvStreamSvc` — gone. But the uninstall registry still listed the full stack: GeForce Experience 3.28.0.417, SHIELD Streaming, Nvidia Share, ShadowPlay, all dated 2026-06-15. This was a **half-uninstalled GeForce Experience**: the service deleted, everything else still registered.

Meanwhile the System log told the other half:

```
Event ID 7031: The NVIDIA LocalSystem Container service terminated unexpectedly.
It has done this 3 time(s). ... corrective action: Restart the service.
```

`NvContainerLocalSystem` was crash-looping too — Service Control Manager restarting it every 6–10 seconds, ~26,000 service-failure events in the current log, every single one this incident. The full loop:

```
NvContainerLocalSystem (AUTO_START)
  → spawns rundll32.exe rxdiag.dll RxDiagSetRuntimeMessagePump
    → rxdiag.dll crashes (0xc0000005 — NvStreamSvc, its target, is gone)
      → Windows Error Reporting holds the crashed process open
        → zombie #N (RAM + GDI/USER handles retained)
          → container itself dies
            → SCM restarts it in 6–10 s
              → loop, one zombie per ~10–20 s, indefinitely
```

First crash in the log: **August 31, 06:40**. Fixed: **September 7**. Seven days, on the order of 40,000 spawn/crash cycles.

Why won't the vendor fix this? Two structural reasons. First, **the updater is the first thing deleted in a half-uninstall** — a product can't self-update when its update path is the broken part. Second, **GameStream is dead**: nobody at NVIDIA is patching a discontinued feature's leftover DLL. Fixes only flow through updates; updates only flow through living software.

## Why the symptoms lied

| Symptom | Actual mechanism |
| --- | --- |
| "Prevents shutdown" | 84 hung zombie processes never answer Windows' shutdown query |
| Lag under load, programs closing | Perpetual spawn → crash → WER-dump churn (CPU spike + disk write per cycle) — small alone, compounding under real workload until the machine tips over |
| **Fonts render as Arial bold instead of Segoe UI** | Session-wide **GDI handle exhaustion**: each zombie held GDI/USER objects; under pool pressure Windows can't allocate font resources and silently substitutes a stock default. This is *the* documented GDI-exhaustion tell |
| Task Manager extremely laggy | Enumerating 84 held-open processes while fighting for the same starved GDI pool |

Nobody looks at a font glitch and thinks "NVIDIA's dead streaming feature." That mismatch — benign-looking symptoms, invisible root cause — is why the standard advice for this class of problem is "reinstall Windows," and why that advice never teaches anything.

## The fix

Elevated PowerShell, four steps, ~4 minutes (the human approved the UAC prompt; the agent wrote and ran the script):

```powershell
# 1. Disable the leftover GeForce Experience scheduled tasks
#    (NvNodeLauncher, 4x NvTmRep_CrashReport, GFE SelfUpdate)
$tasks = 'NvNodeLauncher_{B2FE1952-0186-46C3-BAEC-A80AA35AC5B8}',
         'NvTmRep_CrashReport1_{B2FE1952-0186-46C3-BAEC-A80AA35AC5B8}',
         'NvTmRep_CrashReport2_{B2FE1952-0186-46C3-BAEC-A80AA35AC5B8}',
         'NvTmRep_CrashReport3_{B2FE1952-0186-46C3-BAEC-A80AA35AC5B8}',
         'NvTmRep_CrashReport4_{B2FE1952-0186-46C3-BAEC-A80AA35AC5B8}',
         'NVIDIA GeForce Experience SelfUpdate_{B2FE1952-0186-46C3-BAEC-A80AA35AC5B8}'
foreach ($t in $tasks) { schtasks /change /tn $t /disable }

# 2. Kill the zombies
Get-CimInstance Win32_Process -Filter "Name='rundll32.exe'" |
  Where-Object CommandLine -match 'rxdiag' |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }

# 3. Neuter the DLL — rxdiag.dll can never load again
Rename-Item 'C:\Program Files\NVIDIA Corporation\nvstreamsrv' 'nvstreamsrv.disabled'

# 4. Disable the pump's engine (NOT the display driver)
sc.exe stop NvContainerLocalSystem
sc.exe config NvContainerLocalSystem start= disabled
```

`NVDisplay.ContainerLocalSystem` — the actual display driver service — is a different binary in a different folder. It was never touched and kept running throughout.

Post-fix verification: **zero** `rundll32.exe` at rest, zero rxdiag crashes, and session GDI objects at 2,031 of 65,536 (healthy; top consumers were ordinary apps). The fonts snapped back to Segoe UI on the next window draw.

## Proving it can't come back

Killing the process isn't the fix — the fix is auditing every surface that could re-spawn it. The agent then swept:

- **Scheduled tasks** (every task, every action, every folder): remaining NVIDIA tasks run their own binaries against their own plugin dirs — safe. The `rundll32` tasks are all Microsoft System32 built-ins (PcaSvc, StartupScan, acproxy…).
- **Registry Run/RunOnce** keys — HKCU + HKLM, 64- and 32-bit, Policies: zero NVIDIA/rundll32 entries.
- **Startup folders**, with `.lnk` targets resolved: zero.
- **WMI persistent event subscriptions** (the classic malware persistence spot): zero.
- **Recursive filesystem search** for `rxdiag*`: exactly one hit, inside the renamed `.disabled` folder — referenced by nothing.

One forensic gotcha worth recording, because it almost produced a false "all clear": **`wevtutil` text output prints local time with a fake "Z" suffix.** Cast that to `[datetime]` in PowerShell and it shifts ~5 hours — enough to make a "zero crashes since the fix" check read true while the container service was still quietly crash-looping for another ten minutes until it got disabled. Time comparisons on Windows event logs: filter with raw `TimeCreated` XPath, or compare raw strings. The corrected check, on raw timestamps: genuinely dead.

Permanent cleanup is a **DDU + driver-only reinstall** (the runbook: DDU in Safe Mode → NVCleanstall with GeForce Experience/telemetry unticked → block Windows Update driver delivery → re-run the retrigger audit). Rule of engagement learned the hard way: never update GPU drivers through OEM tools — the OEM updater (`UPDATEBIOS.exe`, also crash-logging on this machine) installs the full bloat stack wholesale.

## The part that's actually interesting

Everything above is known lore. GDI font fallback is documented. WER holding crashed processes open is documented. Forum threads exist with this exact `RxDiagSetRuntimeMessagePump` crash. None of this is a discovery.

What *is* new is the economics of who ran it:

- **Input:** one sentence of garbled symptoms from a human.
- **Execution:** an agent — 20+ probing commands, process genealogy, live growth measurements, log cross-referencing, an elevated fix script with a written log, and a full persistence-surface audit afterward. The human's total effort: the sentence, one UAC click, and judgment calls at two decision points.
- **Cost of the alternative:** a Windows reinstall (an evening) or the dealer/repair-shop route (days, often ends in "we reinstalled Windows"), or — the most common outcome — the machine gradually being deemed "old and slow" and replaced. A week of GDI exhaustion has sentenced perfectly healthy hardware to the recycling bin more than once.
- **Marginal cost of diagnosis has crossed below the marginal cost of the reinstall.** That's the regime change. The forensics chain above — the kind of thing a competent sysadmin does in 15 focused minutes — was previously too expensive (in attention, not intelligence) for anyone to actually perform on their own laptop. An agent performs it for free, at 11 PM, without sighing.

The failure mode also finally has a working business card. "Reinstall" works, costs a day, teaches nothing, and erases the evidence — so the same corpse ships to the next machine. The surgical fix keeps the install, keeps the evidence, and produces a reusable runbook.

## Proof of time

The incident record — symptoms, investigation trail, root-cause chain, fix, verification — was captured and distilled the same evening into a personal knowledge base with a proof layer: every capture is hash-chained into a ledger, and ledger heads are timestamped on Bitcoin via OpenTimestamps. The capture that backs this post is ledger seq 18, committed before the fix was even an hour old.

Translation: the story above is not a reconstruction written for the résumé. It's provably contemporaneous. Ask any candidate with "debugging war stories" for that.

---

*Author: [RP] — systems debugging + agent operations. This incident was resolved with an opencode (terminal coding agent) session on 2026-09-07; the full runbook and audit scripts are in the anchored ledger linked above. The author's day job is making agents do this kind of work for a living.*
