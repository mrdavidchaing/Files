# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A collection of standalone PowerShell scripts for Windows OS deployment automation, primarily used with Microsoft Deployment Toolkit (MDT). Scripts are not interconnected — each directory is a self-contained unit deployed to a specific MDT Task Sequence step or run independently by an admin.

## Repository Structure

- **`MDT/`** — Scripts that run as MDT Task Sequence steps during OS deployment (WinPE/online phase). Each subfolder is a self-contained deployment action (e.g., BIOS config, language packs, TPM upgrade).
- **`Win10/`** — Scripts applied post-deployment to configure and optimize Windows 10 (registry tweaks, privacy settings, scheduled task removal).
- **`Tools/`** — Admin workstation utilities for managing the MDT deployment share (driver import, language pack import, WIM servicing, ImageFactory automation).
- **`Misc/`** — One-off administrative wrappers (e.g., SAP silent install).

## Running Scripts

These are Windows-only PowerShell scripts. There is no build step, test framework, or package manager. Scripts are executed directly:

```powershell
# Standalone (admin workstation)
.\ScriptName.ps1

# MDT Task Sequence scripts — typically called by MDT with no arguments;
# they auto-detect the TS environment via COM object
.\ScriptName.ps1
```

Scripts under `Tools/` often require adjusting variables at the top of the file (staging paths, MDT share UNC paths, vendor lists) before running.

## Key Architecture Patterns

### MDT Task Sequence Context

Scripts in `MDT/` and `Win10/` detect whether they are running inside an MDT Task Sequence by instantiating the `Microsoft.SMS.TSEnvironment` COM object:

```powershell
$tsenv = New-Object -COMObject Microsoft.SMS.TSEnvironment
$logPath = $tsenv.Value("LogPath")
$OSDisk  = $tsenv.Value("OSDisk")   # offline OS drive letter during deployment
```

Some scripts (e.g., `ApplyWin10Optimizations.ps1`) wrap this in a `Try/Catch` to fall back to a standalone mode (`$env:TEMP` for logs), making them dual-mode.

### Logging

Two patterns are used across the codebase:

1. **`Start-Transcript`** — used when output is simple and linear (Dell/HP BIOS scripts, SAP wrapper).
2. **Custom `Logit` function** — appends timestamped lines to a `.log` file; used alongside `Write-Output`/`Write-Host` when finer control is needed.

Log files are always named after `$myInvocation.MyCommand` and placed in `$logPath` (from the TS environment) or `C:\temp\`.

### Offline Image Servicing

Scripts that operate on offline WIM images (under `Tools/WIM-Servicing/` and parts of `MDT/`) use DISM.exe via `Start-Process` rather than the `DISM` PowerShell module, capturing stdout/stderr explicitly:

```powershell
$result = Start-Process -FilePath "dism.exe" -ArgumentList $Arguments -NoNewWindow -Wait -Passthru
```

The WIM is mounted to `C:\temp\Mount` before servicing and committed/unmounted after.

### External Module Dependencies (Tools/ImageFactoryForVMwareWorkstation)

`VMware-ImageFactory.ps1` depends on three bundled PowerShell modules stored alongside it:
- `PsIni` — INI file parsing
- `vmxtoolkit` — VMware Workstation VM control
- `PSFTP` — FTP upload of boot images

These are not installed from a gallery; the module folders must be present in the script directory.

### File Encoding

Several scripts (particularly in `MDT/TPMUpgrade/`, `Tools/MDT-ImportDrivers/`, `Tools/WIM-Servicing/`) are saved as UTF-16 LE (BOM). Editors must preserve this encoding when modifying those files — saving as UTF-8 will break execution on some Windows systems.

### BIOS Configuration

- **Dell**: uses `cctk.exe` (Dell CCTK) with a `settings.cctk` config file and requires HAPI64 drivers (`hapint64.exe`) to be installed first. Password is Base64-encoded in the script.
- **HP**: uses `BiosConfigUtility64.exe` with model-specific `.REPSET` files (one per hardware model in `ConfigureHPBiosSettings/`).
