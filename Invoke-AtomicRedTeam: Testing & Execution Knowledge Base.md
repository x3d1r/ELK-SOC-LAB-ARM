
# Invoke-AtomicRedTeam: Testing and Execution Guide

> **OPSEC Directive**: In offensive security evaluations, Operations Security (OPSEC) is critical. Never run `Invoke-AtomicTest All` blindly in a production or monitored environment. Always stage prerequisites, execute targeted tests, and immediately clean up artifacts to maintain full control of the engagement.

## Table of Contents
- [Glossary](#glossary)
- [1. Setup and Configuration](#1-setup-and-configuration)
- [2. Discovery and Reconnaissance](#2-discovery-and-reconnaissance)
- [3. Execution (Local)](#3-execution-local)
- [4. OPSEC and Artifact Cleanup](#4-opsec-and-artifact-cleanup)
- [5. Advanced Automation (Controlled Execution Loop)](#5-advanced-automation-controlled-execution-loop)
- [6. Troubleshooting Quick Reference](#6-troubleshooting-quick-reference)

---

## Glossary

| Term | Definition |
| :--- | :--- |
| **ART** | Atomic Red Team. A library of simple turnkey tests mapped to the MITRE ATT&CK framework. |
| **IART** | Invoke-AtomicRedTeam. The PowerShell execution framework used to run ART tests. |
| **LOLBins** | Living Off the Land Binaries. Native Windows executables (e.g., `certutil.exe`, `rundll32.exe`) used to perform malicious actions, often bypassing application whitelisting. |
| **PPL** | Protected Process Light. A Windows security feature that prevents even Administrator-level processes from reading the memory of protected processes like LSASS. |
| **GUID** | Globally Unique Identifier. A static ID assigned to each atomic test. Using GUIDs for execution is preferred over Test Numbers, as test numbers can change when the repository is updated. |
| **IoC** | Indicator of Compromise. Artifacts left behind after an attack (e.g., dropped files, registry keys, memory dumps). |
| **OPSEC** | Operations Security. The practice of hiding or managing indicators of an operation to prevent defenders from detecting the activity. |
| **YAML** | The file format used by Atomic Red Team to define test metadata, attack commands, dependencies, and cleanup routines. |

---

## 1. Setup and Configuration

Before executing any tests, the PowerShell environment must be prepared and the module must know where the test definitions (YAML files) are located.

```powershell
# Import the module (if installed via PowerShell Gallery)
Import-Module Invoke-AtomicRedTeam -Force

# Define the default path to the atomics folder (CRITICAL after a system reboot)
$PathToAtomicsFolder = "C:\AtomicRedTeam\atomics"

# (Optional) Set as the default parameter for the entire PowerShell session
# This prevents having to type -PathToAtomicsFolder in every subsequent command
$PSDefaultParameterValues = @{"Invoke-AtomicTest:PathToAtomicsFolder" = $PathToAtomicsFolder}
```

---

## 2. Discovery and Reconnaissance

Enumerating available atomic tests, viewing their mechanics, and checking dependencies before execution. This allows you to select the most OPSEC-friendly technique (e.g., choosing a LOLBin over downloading an external payload).

### List Tests Briefly
```powershell
# List all tests for a specific MITRE technique (e.g., T1003.001)
Invoke-AtomicTest T1003.001 -ShowDetailsBrief -PathToAtomicsFolder $PathToAtomicsFolder

# List all tests for a technique across all OS platforms (Windows, Linux, macOS)
Invoke-AtomicTest T1003.001 -ShowDetailsBrief -anyOS -PathToAtomicsFolder $PathToAtomicsFolder
```

### View Verbose Test Details
```powershell
# Inspect the exact commands, input parameters, prerequisites, and cleanup steps
Invoke-AtomicTest T1003.001 -ShowDetails -PathToAtomicsFolder $PathToAtomicsFolder
```

### Check Prerequisites
```powershell
# Verify what tools or files are missing for a specific test without downloading them yet
Invoke-AtomicTest T1003.001 -TestNumbers 1 -CheckPrereqs -PathToAtomicsFolder $PathToAtomicsFolder
```

---

## 3. Execution (Local)

Running the atomic simulation on the local machine. Always target specific test numbers or GUIDs to maintain strict OPSEC control.

### Execute by Test Number
```powershell
# Run specific test numbers (e.g., Test 1 and 2 of T1003.001)
Invoke-AtomicTest T1003.001 -TestNumbers 1,2 -PathToAtomicsFolder $PathToAtomicsFolder

# Shorthand syntax
Invoke-AtomicTest T1003.001-1,2 -PathToAtomicsFolder $PathToAtomicsFolder
```

### Execute by Test Name
```powershell
Invoke-AtomicTest T1218.010 -TestNames "Regsvr32 remote COM scriptlet execution" -PathToAtomicsFolder $PathToAtomicsFolder
```

### Execute by GUID (Recommended for Automation)
```powershell
# GUIDs never change, whereas test numbers and names might be updated in the repository
Invoke-AtomicTest T1003 -TestGuids 5c2571d0-1572-416d-9676-812e64ca9f44 -PathToAtomicsFolder $PathToAtomicsFolder
```

### Handle Hanging Processes (Timeouts and Interactive Modes)
```powershell
# Force kill the process and children if it runs longer than 15 seconds
Invoke-AtomicTest T1218.010 -TimeoutSeconds 15 -PathToAtomicsFolder $PathToAtomicsFolder

# Allow manual input (e.g., if a payload prompts for Y/N confirmation). 
# WARNING: This breaks output redirection to files.
Invoke-AtomicTest T1003 -Interactive -PathToAtomicsFolder $PathToAtomicsFolder
```

---

## 4. OPSEC and Artifact Cleanup

Removing artifacts, dropped files, registry keys, and scheduled tasks created by the atomic test. Leaving artifacts (like a memory dump in `C:\Windows\Temp`) is a primary Indicator of Compromise (IoC).

### Cleanup Specific Test
```powershell
Invoke-AtomicTest T1003.001 -TestNumbers 1 -Cleanup -PathToAtomicsFolder $PathToAtomicsFolder
```

### Cleanup Entire Technique
```powershell
# Runs the cleanup routine for ALL tests under T1003.001
Invoke-AtomicTest T1003.001 -Cleanup -PathToAtomicsFolder $PathToAtomicsFolder
```

### Manual Artifact Verification (Post-Cleanup)
```powershell
# Always verify the cleanup actually worked
Get-Item "C:\Windows\Temp\lsass_dump.dmp" -ErrorAction SilentlyContinue
Get-Process -Name "procdump" -ErrorAction SilentlyContinue
```

---

## 5. Advanced Automation (Controlled Execution Loop)

Running `Invoke-AtomicTest All` is noisy and reckless. This PowerShell script ensures every test gets its prerequisites, executes, pauses for log generation, and cleans up after itself before moving to the next.

```powershell
# Define the technique to test
$TechniqueID = "T1003.001"
$PathToAtomicsFolder = "C:\AtomicRedTeam\atomics"

# Parse all YAML files for the technique
$Techniques = Get-ChildItem -Path "$PathToAtomicsFolder\$TechniqueID" -Recurse -Include *.yaml | Get-AtomicTechnique

foreach ($Tech in $Techniques) {
    foreach ($Atomic in $Tech.atomic_tests) {
        # Filter for Windows and non-manual executors
        if ($Atomic.supported_platforms -contains "windows" -and $Atomic.executor.name -ne "manual") {
            
            Write-Host "========================================================" -ForegroundColor Cyan
            Write-Host "[*] Processing: $($Tech.attack_technique) - $($Atomic.name)" -ForegroundColor Cyan
            
            # 1. Stage Prerequisites (Download payloads safely)
            Write-Host "[+] Staging Prerequisites..." -ForegroundColor Yellow
            Invoke-AtomicTest $Tech.attack_technique -TestGuids $Atomic.auto_generated_guid -GetPrereqs -PathToAtomicsFolder $PathToAtomicsFolder
            
            # 2. Execute the Test
            Write-Host "[>] Executing Test..." -ForegroundColor Green
            Invoke-AtomicTest $Tech.attack_technique -TestGuids $Atomic.auto_generated_guid -PathToAtomicsFolder $PathToAtomicsFolder
            
            # 3. OPSEC Pause (Allow EDR/SIEM time to generate logs)
            Write-Host "[!] Pausing for 5 seconds for log generation..." -ForegroundColor Magenta
            Start-Sleep -Seconds 5
            
            # 4. Cleanup Artifacts
            Write-Host "[-] Cleaning up artifacts..." -ForegroundColor Red
            Invoke-AtomicTest $Tech.attack_technique -TestGuids $Atomic.auto_generated_guid -Cleanup -PathToAtomicsFolder $PathToAtomicsFolder
        }
    }
}
Write-Host "[*] Technique execution and cleanup complete." -ForegroundColor Cyan
```

---

## 6. Troubleshooting Quick Reference

| Symptom | Root Cause | Resolution |
| :--- | :--- | :--- |
| `Resolve-Path: Cannot bind argument... empty string` | The `$PathToAtomicsFolder` variable was lost (common after a system reboot). | Redefine the variable: `$PathToAtomicsFolder = "C:\AtomicRedTeam\atomics"` |
| `Access is denied (0x00000005)` on LSASS dumps | LSASS Protected Process Light (PPL) is enabled, blocking user-mode memory reads. | Run `Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "RunAsPPL" -Value 0 -Force` and **reboot the system**. |
| `Process Timed out after 120 seconds` | Payload download was blocked by AV/Network, or the payload hangs waiting for input. | Use `-TimeoutSeconds 300`, run with `-Interactive`, or manually download the payload to the `ExternalPayloads` folder. |
| `Cannot be loaded because running scripts is disabled` | The PowerShell Execution Policy is restricted. | Launch PowerShell with: `powershell.exe -ExecutionPolicy Bypass` |
| `Requested registry access is not allowed` | Tamper Protection is active, or UAC is filtering the Administrator token. | Disable Tamper Protection via the Windows Security GUI, or ensure you are running a fully elevated Administrator shell. |
```
