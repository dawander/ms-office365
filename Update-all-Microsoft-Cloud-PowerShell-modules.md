# Update all Microsoft Cloud PowerShell modules

## Detailed documentation for `o365-update.ps1`

This document replaces and expands the earlier wiki guidance for the module updater script in this repository.

It reflects the current behavior of [o365-update.ps1](o365-update.ps1) as of September 2026 (v2.15), including:

- PowerShell 7 only support
- friendly elevation handling
- connectivity checks
- deprecated module removal
- progress-aware installs for large modules
- core module conflict handling
- latest-only version maintenance and cleanup
- **new:** an optional forced, comprehensive cleanup mode for stubborn old module versions

---

## Purpose

`o365-update.ps1` is a maintenance script for Microsoft 365, Azure, and Microsoft Cloud PowerShell tooling. Its goal is to standardize and update the modules most commonly used for tenant administration and automation.

The script can:

- install missing modules
- update existing modules
- remove deprecated legacy modules
- remove older duplicate versions so only the latest version remains
- force a comprehensive removal of every old version of every installed module, even ones that normally resist cleanup
- check connectivity and prerequisites before processing
- optionally log its work to a transcript file

---

## Script requirements

Before running the script, make sure the following requirements are met:

### Required

- PowerShell 7.0 or later
- Administrator privileges
- internet access to PowerShell Gallery and Microsoft download endpoints
- enough free disk space for large modules such as `Az` and `Microsoft.Graph`

### Why PowerShell 7 is required

The script explicitly requires PowerShell 7 and will stop if it is run in Windows PowerShell 5.1. This is intentional because the current Microsoft Graph and Azure administration modules work more reliably on PowerShell 7.

---

## Modules managed by the script

The updater maintains the following module groups.

| Category | Modules |
|---|---|
| Graph | `Microsoft.Graph`, `Microsoft.Graph.Authentication` |
| Exchange | `ExchangeOnlineManagement` |
| Teams | `MicrosoftTeams` |
| Azure | `Az` |
| SharePoint | `PnP.PowerShell`, `Microsoft.Online.SharePoint.PowerShell` |
| PowerApps | `Microsoft.PowerApps.PowerShell`, `Microsoft.PowerApps.Administration.PowerShell` |
| Core | `PowerShellGet`, `PackageManagement` |
| Other | `Microsoft.WinGet.Client` |

### Deprecated modules removed automatically

The script also checks for and removes older or deprecated modules such as:

- `AzureAD`
- `AzureADPreview`
- `MSOnline`
- `AIPService`
- `aadrm`
- `SharePointPnPPowerShellOnline`
- `WindowsAutoPilotIntune`
- `O365CentralizedAddInDeployment`
- `MSCommerce`

---

## Parameters

| Parameter | Description |
|---|---|
| `-Prompt` | Prompts before installs and cleanup actions |
| `-CreateLog` | Starts transcript logging |
| `-LogPath` | Sets the target folder for logs |
| `-SkipDeprecatedCleanup` | Skips removal of deprecated modules |
| `-SkipVersionCleanup` | Preserves old versions instead of removing them |
| `-ForceVersionCleanup` | **New.** Forces a comprehensive, unconditional cleanup of every old version of every installed module (not just those in `-ModuleScope`), overriding `-SkipVersionCleanup` and skipping `-Prompt` confirmations. See [Forced comprehensive cleanup](#forced-comprehensive-cleanup) below. |
| `-CheckOnly` | Reports versions without making changes |
| `-CheckSessions` | Checks session conflicts and exits |
| `-TerminateConflicts` | Intended for automatic termination of conflicting sessions |
| `-ModuleScope` | Restricts processing to specific categories |
| `-SkipConnectivityCheck` | Skips network validation |
| `-Repository` | Uses an alternative PowerShell repository instead of `PSGallery` |

---

## Common usage examples

### Update everything automatically

```powershell
.\o365-update.ps1
```

### Review actions interactively

```powershell
.\o365-update.ps1 -Prompt
```

### Check versions only

```powershell
.\o365-update.ps1 -CheckOnly
```

### Update only Graph and Exchange modules

```powershell
.\o365-update.ps1 -ModuleScope Graph,Exchange
```

### Create a transcript log

```powershell
.\o365-update.ps1 -CreateLog -LogPath 'C:\Logs'
```

### Keep multiple versions intentionally

```powershell
.\o365-update.ps1 -SkipVersionCleanup
```

### Force a full cleanup of every old module version

```powershell
.\o365-update.ps1 -ForceVersionCleanup
```

Use this when routine cleanup has left old versions behind (see [Forced comprehensive cleanup](#forced-comprehensive-cleanup)).

---

## Execution flow in detail

This section explains exactly what happens when the script runs.

### 1. PowerShell version enforcement

The script begins with:

```powershell
#Requires -Version 7.0
```

If the host is not PowerShell 7 or later, the script will not run.

### 2. Parameter handling

At startup, the script reads all runtime switches such as prompt mode, log settings, cleanup behavior, module scope, and repository choice.

### 3. Friendly administrator check

The script checks whether it is running elevated. If it is not:

- it explains why admin rights are required
- it tells the user how to reopen PowerShell 7 as Administrator
- it prints a ready-to-copy elevation command
- it exits cleanly

This is easier to understand than a raw PowerShell error.

### 4. Optional transcript logging

If `-CreateLog` is used, the script starts a transcript and records console activity to a log file.

### 5. Header and runtime configuration display

The script prints a run summary showing:

- mode
- scope
- repository
- cleanup settings (including whether forced cleanup is enabled)
- timeout and parallel settings

### 6. Prerequisite validation

The script checks:

- PowerShell version
- administrator privileges
- execution policy
- PowerShell language mode

If a critical prerequisite fails, the script exits.

### 7. Connectivity checks

Unless skipped, the script validates outbound connectivity to:

- PowerShell Gallery
- Microsoft Download Center
- NuGet.org

If one or more checks fail, the user can choose whether to continue.

### 8. Optional session-check-only mode

If `-CheckSessions` is used, the script performs the session check workflow and exits without changing modules.

### 9. Module filtering

The script builds a filtered list of modules based on `-ModuleScope`. If no scope is supplied, all supported module categories are processed.

### 10. Deprecated module cleanup

If deprecated cleanup has not been skipped, the script searches for known legacy modules and tries to uninstall them.

### 11. Per-module processing

For each selected module, the script:

1. checks whether the module is installed
2. gets the highest local version
3. checks the latest available online version
4. determines whether install, update, or no action is needed
5. applies module-specific parameter rules where necessary
6. performs installation or update with progress output

### 12. Latest-only version enforcement

The current script is designed to maintain only the latest version of each processed module by default.

That means when duplicate versions are detected, the script now:

- lists the installed versions
- removes older copies
- keeps only the newest version available
- performs a final cleanup verification pass after the main processing loop

This behavior helps reduce ambiguity and avoid the version conflicts that can happen when multiple module versions remain installed.

If `-SkipVersionCleanup` is supplied, this cleanup is intentionally bypassed. If `-ForceVersionCleanup` is supplied, this standard cleanup is superseded by the more thorough forced pass described below.

### 13. Forced comprehensive cleanup (optional)

If `-ForceVersionCleanup` is supplied, a dedicated cleanup pass runs after the normal update loop finishes (regardless of whether individual module updates succeeded or failed). This pass is unconditional:

- it scans **every installed module**, not just the ones selected by `-ModuleScope`
- it overrides `-SkipVersionCleanup` — you'll get a cleanup even if you also passed that switch
- it does not ask for confirmation, even under `-Prompt`
- it looks for old versions in two places: modules registered through PowerShellGet (`Get-InstalledModule`) **and** modules only visible on `PSModulePath` (`Get-Module -ListAvailable`) that were never registered with PowerShellGet — for example, copies installed by another tool or extracted manually
- before removing a version, it unloads that specific version from the current session (`Remove-Module -Force`) so `Uninstall-Module` isn't blocked by "module is in use"
- if `Uninstall-Module` still can't remove a version, it falls back to deleting the module's version folder directly from disk
- it will never touch modules bundled inside `$PSHOME\Modules` (the PowerShell 7 built-in modules), even during the filesystem fallback

Use this switch when you've noticed older module versions persisting across normal runs — most commonly because a version was loaded in the session, installed outside PowerShellGet, or previously skipped.

### 14. Final summary and recommendations

At the end of the run, the script prints:

- completion status
- end time
- general recommendations such as restarting PowerShell and verifying installed modules
- confirmation of which cleanup mode ran (standard cleanup, forced cleanup, or skipped)

---

## How the script handles module updates

### Standard modules

For normal cloud modules, the script installs or updates the module and then removes older versions unless cleanup is skipped.

### Core modules

Special handling is used for:

- `PowerShellGet`
- `PackageManagement`

These modules are often loaded into the current session and may be locked by PowerShell. In those cases the script uses the safest available update path and may require a PowerShell restart before the newest copy is fully active.

This is normal and expected.

---

## Version cleanup behavior

### Default behavior

By default, the script now aims for this result:

- one module
- one active latest installed version
- no unnecessary older copies left behind

### When duplicates may still temporarily remain

Older versions may still persist temporarily if:

- the module is actively loaded in the current session
- Windows has locked files in use
- the module is a protected core dependency
- `-SkipVersionCleanup` was used
- the module was installed outside PowerShellGet and the standard cleanup pass never registered it as a duplicate

In those cases, a restart of PowerShell 7 as Administrator is usually enough to allow a clean follow-up run. Alternatively, run the script with `-ForceVersionCleanup`, which handles most of these situations directly without requiring a restart first.

### Forced cleanup vs. standard cleanup

| | Standard cleanup (default) | Forced cleanup (`-ForceVersionCleanup`) |
|---|---|---|
| Scope | Modules in the current `-ModuleScope` | Every installed module |
| Respects `-SkipVersionCleanup` | Yes | No — always runs |
| Requires confirmation with `-Prompt` | Yes, when triggered interactively | No |
| Detects PSModulePath-only duplicates (not registered with PowerShellGet) | No | Yes |
| Unloads module from session before removing | No | Yes |
| Filesystem fallback if `Uninstall-Module` fails | No | Yes |
| Touches PS7 bundled modules under `$PSHOME\Modules` | No | No |

---

## Progress and timing behavior

Large modules such as `Az` and `Microsoft.Graph` can take much longer to process than smaller modules.

The script includes:

- estimated size output
- estimated time output
- progress bars
- extra messaging for constrained or restricted environments
- warnings that verification can look like a hang even when work is continuing normally

---

## Constrained language mode and restricted environments

In environments managed by application control, WDAC, AppLocker, or similar policies, the script may run in `ConstrainedLanguage` mode.

In that situation:

- some optimizations may not be available
- installs can take much longer
- verification can appear slow
- policy restrictions may block or reduce some operations

The script tries to continue safely and prints more detailed troubleshooting information.

---

## Troubleshooting

### The script says PowerShell 7 is required

Install PowerShell 7 and run the script again in that host.

### The script says administrator privileges are required

Start PowerShell 7 with Run as administrator and rerun the script.

### A module says it is already in use

This usually means the current session or another PowerShell process has loaded the module. Close other PowerShell windows, reopen an elevated PowerShell 7 session, and run the script again. If the problem persists, try `-ForceVersionCleanup`, which unloads the specific old version from the session before attempting removal.

### Multiple versions are still shown after cleanup

If this happens, it usually means one of the installed copies was locked during removal, or it was installed outside PowerShellGet and the standard cleanup pass didn't recognize it as a duplicate. Run:

```powershell
.\o365-update.ps1 -ForceVersionCleanup
```

This performs a full scan (including non-PowerShellGet installs) and applies a filesystem fallback for versions that can't be removed the normal way. If it still can't remove a version, restart PowerShell and rerun.

### Installs are very slow

This is most common for `Az` and `Microsoft.Graph`. Check:

- internet connectivity
- disk space
- antivirus interference
- execution restrictions or corporate policy settings

---

## Recommended post-run verification

After the script finishes, verify your installed modules with:

```powershell
Get-Module -ListAvailable PowerShellGet,PackageManagement,Microsoft.Graph,Microsoft.Graph.Authentication,ExchangeOnlineManagement,MicrosoftTeams,PnP.PowerShell,Az |
    Select-Object Name, Version |
    Sort-Object Name, Version -Descending
```

If you want only the highest version for each module, the list should show a single current entry per module after cleanup completes successfully. If duplicates still appear, re-run with `-ForceVersionCleanup`.

---

## Operational notes for maintainers

- The script uses `AllUsers` scope for module installation and removal.
- Logging is optional and controlled by the transcript switch.
- Cleanup behavior is centralized through helper functions that remove old module versions: `Remove-OldModuleVersions` (single module) and `Remove-AllOldModuleVersions` (all modules), both of which accept a `-Force` switch used internally by `-ForceVersionCleanup`.
- The forced-removal path is implemented in a dedicated `Remove-ModuleVersionForced` helper that unloads the module from the session, attempts `Uninstall-Module`, and falls back to a direct filesystem delete — while explicitly refusing to remove anything under `$PSHOME\Modules`.
- Module-specific installation parameters are adjusted automatically because some modules do not accept every standard parameter.
- The script prioritizes safe execution and clear user messaging over silent failure.

---

## Source and prior documentation

This local guide was prepared as an updated replacement for the earlier repository wiki page:

- Update all Microsoft Cloud PowerShell modules
- Repository: https://github.com/directorcia/Office365
- Previous wiki reference: https://github.com/directorcia/Office365/wiki/Update-all-Microsoft-Cloud-PowerShell-modules

---

## Summary

Use this script when you want a single maintenance workflow that:

- updates your Microsoft cloud administration modules
- removes deprecated tooling
- reduces duplicate-version problems
- keeps your environment aligned to the latest supported modules

For most administrators, the recommended run pattern is:

```powershell
.\o365-update.ps1
```

Run it from an elevated PowerShell 7 session and allow version cleanup so the environment stays clean and predictable.

If old versions have accumulated or previous cleanups have left stragglers behind, run:

```powershell
.\o365-update.ps1 -ForceVersionCleanup
```

for a thorough, one-time sweep across every installed module.
