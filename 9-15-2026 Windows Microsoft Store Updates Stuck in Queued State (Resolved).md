
**Date:** 2026-09-15  
**System:** Windows 11  
**Category:** Windows / Microsoft Store / System File Repair  
**Status:** Resolved

## Issue

Microsoft Store application and runtime updates would not download or install.

Multiple packages remained indefinitely in a **Queued** state, including Microsoft Store applications and supporting runtime components.

Retrying the updates did not resolve the problem.

---

## Initial Troubleshooting

Attempted to use the Windows Store reset utility:

    wsreset.exe

PowerShell reported that `wsreset.exe` could not be found.

Verified whether the executable existed:

    Test-Path C:\Windows\System32\WSReset.exe
    Get-Item C:\Windows\System32\WSReset.exe

Result:

    False
    Get-Item: Cannot find path 'C:\Windows\System32\WSReset.exe'

Rather than downloading or manually replacing the executable, Windows servicing health was investigated.

---

## DISM Component Store Check

Ran:

    DISM.exe /Online /Cleanup-Image /ScanHealth

Result:

    No component store corruption detected.
    The operation completed successfully.

### What is DISM?

**DISM (Deployment Image Servicing and Management)** is a Windows servicing utility used to inspect, modify, and repair Windows images.

For a running Windows installation:

- `/Online` targets the currently running operating system.
- `/Cleanup-Image` selects servicing operations for the Windows image/component store.
- `/ScanHealth` performs a detailed scan for corruption in the Windows component store.

The component store, primarily located under `C:\Windows\WinSxS`, contains Windows components used for servicing and system-file repair.

In this case, DISM determined that the component store itself was healthy. Therefore, `DISM /RestoreHealth` was not necessary.

---

## SFC Verification

Next, attempted:

    sfc /verifyonly

Initial result:

    Windows Resource Protection could not start the repair service.

### What is SFC?

**SFC (System File Checker)** is a Windows utility that verifies the integrity of protected Windows system files.

Two relevant modes are:

    sfc /verifyonly

This checks protected system files without attempting repairs.

    sfc /scannow

This checks protected system files and attempts to repair detected integrity violations.

### DISM vs. SFC

DISM and SFC perform related but different jobs.

**DISM** can verify and repair the Windows component store used by Windows servicing.

**SFC** verifies the actual protected Windows system files in the installed operating system.

A healthy DISM component store does not necessarily mean that every active Windows system file is intact.

---

## Windows Modules Installer Investigation

Because SFC reported that it could not start the repair service, the Windows servicing-related services were checked:

    Get-Service TrustedInstaller,wuauserv,BITS,InstallService,AppXSvc,ClipSVC |
        Format-Table Name,Status,StartType

Important services were available and running, including:

- Windows Modules Installer (`TrustedInstaller`)
- Windows Update (`wuauserv`)
- Background Intelligent Transfer Service (`BITS`)
- Microsoft Store Install Service (`InstallService`)
- AppX Deployment Service (`AppXSvc`)

The Windows Modules Installer configuration was then checked:

    sc.exe qc TrustedInstaller

`TrustedInstaller` was correctly configured as a demand-start service.

The associated process was also confirmed to be running:

    Get-Process TrustedInstaller -ErrorAction SilentlyContinue

This indicated that the initial SFC error was not simply the result of the Windows Modules Installer service being disabled.

---

## CBS Log Investigation

The Component-Based Servicing log was inspected:

    Get-Content C:\Windows\Logs\CBS\CBS.log -Tail 120

The log showed that `TrustedInstaller` and `TiWorker` successfully initialized.

Relevant servicing activity included:

    TrustedInstaller service starts successfully.
    TiWorker starts successfully.
    TiWorker: Client requests SFP repair object.
    NonStart: Set pending store consistency check.
    IAdvancedInstallerAwareStore_ResolvePendingTransactions
    Poqexec successfully registered in 'SetupExecute'

This suggested that Windows had pending servicing work or transactions that needed to be processed.

Because some servicing operations are completed during system startup, the computer was restarted before continuing.

---

## SFC Verification After Restart

After restarting Windows, the verification was attempted again:

    sfc /verifyonly

This time SFC successfully completed its scan.

Result:

    Beginning system scan.
    Beginning verification phase of system scan.
    Verification 100% complete.

    Windows Resource Protection found integrity violations.

This confirmed that protected Windows system files had integrity problems even though DISM had determined that the Windows component store itself was healthy.

---

## Repair

Because the component store was healthy and SFC had confirmed integrity violations, SFC was allowed to repair the affected protected system files:

    sfc /scannow

SFC reported that it found corrupted files and successfully repaired them.

The computer was restarted after the repair.

---

## Verification

After the SFC repair and restart:

1. Opened Microsoft Store.
2. Returned to the Downloads/Updates interface.
3. Retried the pending updates.
4. Previously queued updates began downloading and installing normally.

**The original Microsoft Store update problem was resolved.**

---

## Root Cause

Windows had protected system-file integrity violations.

The Windows component store itself was healthy, so a DISM component-store repair was not required. SFC was able to repair the affected protected Windows system files.

After SFC completed the repairs and Windows was restarted, Microsoft Store updates resumed functioning normally.

The exact individual corrupted file responsible for the Microsoft Store failure was not isolated. Therefore, it cannot be conclusively stated that one specific corrupted file caused the Store queue problem.

However, the sequence of events provides strong evidence of a relationship:

    Microsoft Store updates stuck
                ↓
    Windows integrity investigation
                ↓
    DISM component store healthy
                ↓
    SFC detects system-file integrity violations
                ↓
    SFC repairs violations
                ↓
    System restarted
                ↓
    Microsoft Store updates work normally

---

## Commands Used

    Test-Path C:\Windows\System32\WSReset.exe
    Get-Item C:\Windows\System32\WSReset.exe

    DISM.exe /Online /Cleanup-Image /ScanHealth

    sfc /verifyonly

    Get-Service TrustedInstaller,wuauserv,BITS,InstallService,AppXSvc,ClipSVC |
        Format-Table Name,Status,StartType

    sc.exe qc TrustedInstaller

    Get-Process TrustedInstaller -ErrorAction SilentlyContinue

    Get-Content C:\Windows\Logs\CBS\CBS.log -Tail 120

    sfc /verifyonly

    sfc /scannow

---

## Key Takeaways

### DISM

**Deployment Image Servicing and Management (DISM)** operates on Windows images and servicing infrastructure.

In this incident:

    DISM.exe /Online /Cleanup-Image /ScanHealth

was used to determine whether the Windows component store was corrupted.

It reported no component-store corruption, which meant there was no evidence requiring `DISM /RestoreHealth`.

### SFC

**System File Checker (SFC)** verifies protected Windows system files.

The initial command:

    sfc /verifyonly

checked system-file integrity without making changes.

After a restart allowed pending Windows servicing activity to proceed, SFC successfully ran and detected integrity violations.

The repair command:

    sfc /scannow

then repaired the detected protected system-file problems.

### Troubleshooting Methodology

The troubleshooting process followed an evidence-based sequence:

    Observe the original symptom
              ↓
    Attempt targeted Store troubleshooting
              ↓
    Discover unexpected system condition
              ↓
    Check Windows component-store health
              ↓
    Check protected system-file integrity
              ↓
    Investigate servicing services and logs
              ↓
    Restart to process pending servicing work
              ↓
    Confirm integrity violations
              ↓
    Repair confirmed violations
              ↓
    Restart
              ↓
    Retest the original symptom
              ↓
    Confirm resolution

Rather than downloading replacement Windows files or immediately performing broad resets, the underlying Windows servicing state was investigated first.

This preserved diagnostic evidence and allowed the repair to be based on a confirmed system-file integrity problem rather than an assumption.