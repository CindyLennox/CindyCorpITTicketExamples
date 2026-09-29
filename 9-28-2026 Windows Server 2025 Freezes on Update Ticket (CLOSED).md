# Ticket 001 — DC01 Windows Update Causes Complete VM Freeze

**Date Opened:** 2026-09-28  
**Date Resolved:** 2026-09-29  
**Status:** Resolved  
**Priority:** Medium  
**System:** DC01  
**Platform:** Oracle VirtualBox  
**Operating System:** Windows Server 2025 Standard Evaluation (Desktop Experience)  
**Version:** 24H2  
**Role:** Planned Active Directory Domain Controller (AD DS not installed at time of incident)

---

## Issue Summary

DC01 experienced repeated complete guest operating system freezes while processing Windows security updates through Windows Update.

The issue was isolated to the Windows Update workload. Normal operation outside of Windows Update did not reproduce the failure.

---

## User/Service Impact

Windows Update could not complete successfully while the issue was occurring.

The guest became completely unresponsive during affected update attempts and required forced power-off through VirtualBox. Deployment of Active Directory Domain Services was postponed until the server could be demonstrated to be stable and fully serviced.

No production services were affected because DC01 was still in its initial lab deployment phase.

---

## Symptoms

During Windows Update processing:

- Windows Update began processing or downloading available updates.
    
- The entire guest operating system eventually became unresponsive.
    
- Ctrl+Alt+Delete produced no response.
    
- VirtualBox ACPI Shutdown produced no response.
    
- No virtual disk activity was observed after the freeze occurred.
    
- Recovery required a forced power-off of the VM.
    
- The behavior was reproducible during multiple Windows Update attempts.
    
- Equivalent freezes were not observed during normal guest operation.
    

The behavior was consistent with a complete guest hang rather than an isolated failure of the Windows Settings application.

---

## Environment

|Component|Configuration|
|---|---|
|Hypervisor|Oracle VirtualBox|
|Guest|Windows Server 2025 Standard Evaluation|
|Hostname|`DC01`|
|RAM|6144 MB|
|vCPU|2 at time of incident|
|Virtual Disk|~80 GB dynamically allocated|
|Firmware|EFI/UEFI|
|Secure Boot|Disabled|
|Network|NAT|
|Guest Additions|Installed|

---

## Installed Updates at Time of Incident

`Get-HotFix` reported:

|Type|KB|Reported Date|
|---|---|---|
|Security Update|KB5072725|2026-01-14|
|Security Update|KB5073379|2026-01-14|
|Update|KB5066131|2026-01-14|

Windows build at the time of investigation:

`26100.32230`

---

## Updates Detected During Investigation

Windows Update logging identified the following pending/detected updates:

- KB5122871 — 2026-09 Security Update for Windows Server 2025 / Server OS 24H2
    
- KB5126052 — 2026-09 .NET Framework Security Update
    
- KB5007651 — Windows Security platform update
    
- Microsoft Defender Antivirus Security Intelligence update
    

---

## Diagnostic Findings

A Windows Update diagnostic log was generated using:

```
Get-WindowsUpdateLog
```

The generated log was reviewed from:

```
C:\Users\Administrator\Desktop\WindowsUpdate.log
```

Repeated Windows Update Agent errors were observed:

```
*FAILED* [80240037] wuauengcore.dll
Client\Engine\Agent\networkcostmgr.cpp @221
```

Associated error:

```
0x80240037
WU_E_NOT_SUPPORTED
```

Windows Update also repeatedly logged Download Manager activity:

```
DownloadManager CreateDownloadJob DownloadType 0.
DownloadJob did not init now, but this may be expected.
```

Affected update GUID observed during troubleshooting:

```
B31EE448-ACE5-4715-95EA-2494974D2576
```

Example sequence:

```
Received completion event for update B31EE448...
Unlock the sandbox...
[80240037] networkcostmgr.cpp
CreateDownloadJob
DownloadJob did not init now
Uninit download job
Queueing update ... for download handler request generation
```

The `0x80240037` errors were correlated with the affected Windows Update workflow. No evidence established that these errors directly caused the complete guest freeze.

---

## Steps to Reproduce

1. Start DC01 with its original allocation of 2 vCPUs.
    
2. Allow Windows Server to reach the desktop and verify normal guest operation.
    
3. Open Windows Update.
    
4. Initiate installation of the available security updates.
    
5. Allow update processing to continue.
    
6. Guest eventually becomes completely unresponsive.
    
7. Ctrl+Alt+Delete and VirtualBox ACPI Shutdown fail to produce a response.
    
8. VM requires forced power-off.
    

The failure was reproduced during multiple Windows Update attempts prior to the configuration change.

---

## Troubleshooting Performed

- Confirmed DC01 operated normally outside the Windows Update workload.
    
- Reproduced the complete guest freeze during Windows Update.
    
- Verified that Ctrl+Alt+Delete did not recover the guest.
    
- Verified that VirtualBox ACPI Shutdown did not recover the guest.
    
- Observed absence of virtual disk activity following the freeze.
    
- Verified installed updates using `Get-HotFix`.
    
- Verified Windows version and build using `winver`.
    
- Generated and reviewed `WindowsUpdate.log`.
    
- Identified repeated `0x80240037` Windows Update Agent errors.
    
- Ejected the Guest Additions installation ISO.
    
- Increased VM processor allocation from **2 vCPUs to 4 vCPUs**.
    
- Retested Windows Update following the CPU allocation change.
    
- Windows Update subsequently completed successfully.
    
- Confirmed DC01 remained responsive throughout the update process.
    
- Confirmed the server reached a fully updated state and remained stable following reboot.
    

---

## Resolution

The VirtualBox configuration for DC01 was modified to increase the guest's processor allocation from **2 vCPUs to 4 vCPUs**.

Following the configuration change, Windows Update completed successfully without reproducing the complete guest freeze. DC01 remained responsive during servicing and operated normally following reboot.

The server is now fully updated and stable under observed operation.

The original freezes were therefore associated with the previous VM resource configuration, with insufficient CPU allocation considered the most likely contributing factor based on the successful retest. A definitive low-level cause for the guest hang was not established.

**Resolution Status:** Successful  
**Current vCPU Allocation:** 4  
**Windows Update:** Operational  
**Guest Stability:** Verified under subsequent update and normal workloads

---

## Verification

Post-resolution verification confirmed:

- Windows Server boots successfully.
    
- Windows Update completes successfully.
    
- Guest remains responsive during update servicing.
    
- Server remains stable following reboot.
    
- No recurrence of the complete guest freeze has been observed.
    
- DC01 is ready to proceed to the next stage of lab deployment.
    

---

## Deployment Impact

The temporary hold on DC01 deployment has been removed.

The system may now proceed with planned configuration as an Active Directory Domain Services and DNS server. A known-good pre-AD configuration snapshot may be created before server roles are installed.

Planned baseline snapshot:

`Clean Install - Pre-AD`

---

## Root Cause

**Probable Cause:** Insufficient VM CPU allocation during Windows servicing workload.

The VM was configured with 2 vCPUs when the freezes occurred. Increasing the allocation to 4 vCPUs was the material configuration change preceding successful completion of Windows Update.

Because the issue was not reproduced under controlled testing with alternative CPU allocations and no crash dump was obtained from the hard-hung guest, insufficient CPU allocation cannot be established as the definitive root cause.

---

## Closure Notes

DC01 is fully updated and stable following the increase from 2 to 4 vCPUs. Windows Update completed without further guest hangs.

The incident is considered resolved. If equivalent freezes recur during Windows Update or unrelated workloads, the ticket should be reopened and investigation expanded to VirtualBox configuration, guest drivers, host virtualization, storage, and Windows system integrity.