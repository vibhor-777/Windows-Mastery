# Folder 19: Error Codes and Fixes Database

## 4Ws Structure

- **Who:** All technical users — Support Technicians, System Administrators (Pranali Prashasak), and Developers.
- **What:** Searchable index of critical Windows error codes with diagnostic steps and resolutions.
- **Where:** Windows Event Viewer, BSOD screens, application error dialogs, CMD/PowerShell.
- **Why:** Rapid error diagnosis reduces Mean Time to Resolution (MTTR) and minimizes system downtime.

---

## Error Code Categories

Windows uses multiple error code formats:
- **Win32 Error Codes:** Decimal or hex (e.g., 5 or 0x5 = Access Denied)
- **NTSTATUS Codes:** Hex starting with 0xC, 0x8, or 0x4 (kernel/driver layer)
- **HRESULT Codes:** Hex starting with 0x8007 (often Win32 errors wrapped in COM)
- **Bug Check Codes:** BSOD stop codes (e.g., 0x0000007E)
- **HTTP Status Codes:** For web/API errors

---

## Critical Win32 Error Codes

| Code (Hex) | Code (Dec) | Name | Solution |
|---|---|---|---|
| 0x00000000 | 0 | ERROR_SUCCESS | No error |
| 0x00000001 | 1 | ERROR_INVALID_FUNCTION | Invalid function call; check API usage |
| 0x00000002 | 2 | ERROR_FILE_NOT_FOUND | Verify file path; check if file was deleted |
| 0x00000003 | 3 | ERROR_PATH_NOT_FOUND | Verify directory path exists |
| 0x00000004 | 4 | ERROR_TOO_MANY_OPEN_FILES | Close unused file handles; check for handle leaks |
| **0x00000005** | **5** | **ERROR_ACCESS_DENIED** | **Verify permissions with icacls; check UAC elevation; verify registry permissions** |
| 0x00000006 | 6 | ERROR_INVALID_HANDLE | Handle is invalid or closed; restart service/app |
| 0x0000000E | 14 | ERROR_OUTOFMEMORY | Free memory; close applications; increase pagefile |
| 0x0000001F | 31 | ERROR_GEN_FAILURE | Hardware device failure; check Device Manager |
| 0x00000020 | 32 | ERROR_SHARING_VIOLATION | File in use; close other apps; use Handle.exe to find owner |
| 0x00000057 | 87 | ERROR_INVALID_PARAMETER | Check function parameters/API documentation |
| 0x0000006E | 110 | ERROR_OPEN_FAILED | Cannot open file; check path and permissions |
| 0x00000070 | 112 | ERROR_DISK_FULL | Free disk space; run cleanmgr or expand volume |
| 0x0000007B | 123 | ERROR_INVALID_NAME | Invalid filename characters; rename file/path |
| 0x000000B7 | 183 | ERROR_ALREADY_EXISTS | Object already exists; handle conflict in code |
| 0x000000E6 | 230 | ERROR_BROKEN_PIPE | Network connection dropped; retry or restart service |
| 0x00000459 | 1113 | ERROR_NO_UNICODE_TRANSLATION | Invalid Unicode; check encoding |
| 0x000004C7 | 1223 | ERROR_CANCELLED | User cancelled operation; retry if needed |
| 0x00000717 | 1815 | ERROR_RESOURCE_NOT_PRESENT | Missing DLL or resource; install redistributables |

---

## Error Code 0x80070005: Access Denied (Deep Dive)

**Error:** `0x80070005` — Access is denied.  
**HRESULT Breakdown:** 0x80070000 (COM error wrapper) + 0x0005 (Win32 error 5 = ACCESS_DENIED)

**Common Scenarios and Solutions:**

```powershell
# Scenario 1: Windows Update access denied
# Solution: Fix Windows Update components
net stop wuauserv
net stop cryptSvc
net stop bits
net stop msiserver
ren C:\Windows\SoftwareDistribution SoftwareDistribution.old
ren C:\Windows\System32\catroot2 catroot2.old
net start wuauserv
net start cryptSvc
net start bits
net start msiserver

# Scenario 2: Registry key access denied
# Solution: Take ownership of registry key
# Security Warning (Suraksha Chetavni): Always back up before modifying registry ownership
reg export "HKEY_USERS\S-1-5-20" C:\Backups\HKU_S-1-5-20_backup.reg
# Then use PSEXEC to run regedit as SYSTEM:
psexec -sid regedit.exe
# Navigate to key → Permissions → Add your account

# Scenario 3: File/folder access denied
icacls "C:\ProblemFolder" /grant "%USERNAME%":F /T

# Scenario 4: COM/WMI access denied
# Verify DCOM permissions
dcomcnfg.exe
# Navigate to Component Services → Computers → My Computer → Properties → COM Security
```

---

## Critical BSOD (Blue Screen of Death) Error Codes

| Stop Code | Name | Common Cause | Solution |
|---|---|---|---|
| **0x0000007B** | INACCESSIBLE_BOOT_DEVICE | Storage driver missing; NTFS corrupt; AHCI/IDE mode change | WinRE: bootrec, check storage drivers, run chkdsk |
| 0x0000007E | SYSTEM_THREAD_EXCEPTION_NOT_HANDLED | Buggy driver | Boot safe mode, identify and remove problematic driver |
| 0x0000007F | UNEXPECTED_KERNEL_MODE_TRAP | Hardware failure (RAM, CPU overclock) | Run memtest86, check for overheating |
| 0x00000024 | NTFS_FILE_SYSTEM | NTFS corruption | chkdsk /f /r, restore from backup |
| 0x00000050 | PAGE_FAULT_IN_NONPAGED_AREA | Driver or RAM corruption | Update/reinstall drivers, test RAM |
| 0x0000003B | SYSTEM_SERVICE_EXCEPTION | Driver or system file corruption | SFC /scannow, update drivers |
| 0x000000D1 | DRIVER_IRQL_NOT_LESS_OR_EQUAL | Buggy driver accessing invalid memory | Driver Verifier to identify, update/roll back driver |
| 0x000000EF | CRITICAL_PROCESS_DIED | Core system process (csrss, winlogon) crashed | Malware scan, SFC, check RAM |
| 0x000000C4 | DRIVER_VERIFIER_DETECTED_VIOLATION | Driver verifier found a driver bug | Disable driver verifier, update the flagged driver |
| 0xC000021A | STATUS_SYSTEM_PROCESS_TERMINATED | Winlogon or Csrss fatal error | SFC /scannow, DISM repair, check for malware |
| 0xC0000034 | STATUS_OBJECT_NAME_NOT_FOUND | Missing BCD or boot files | bootrec /rebuildbcd |
| 0xC0000225 | STATUS_NOT_FOUND | Boot file not found | bcdboot C:\Windows /s C: /f ALL |

### BSOD Analysis Workflow

```cmd
REM View recent BSOD information from Event Log
wevtutil qe System /q:"*[System[EventID=41]]" /f:text /c:5

REM List minidump files
dir C:\Windows\Minidump\

REM WinDbg analysis (if installed)
REM Open WinDbg → File → Open Crash Dump → Select .dmp file
REM !analyze -v    (automatic crash analysis)
REM !thread        (current thread)
REM !process 0 0   (list all processes at crash time)
REM lmvm <drivername>  (driver version info)
```

---

## Common Windows Update Error Codes

| Code | Description | Solution |
|---|---|---|
| 0x800F0831 | CBS_E_STORE_INTEGRITY_VIOLATION | DISM /RestoreHealth |
| 0x800F0922 | CBS_E_INSTALLERS_FAILED | Reboot and retry; check antivirus exclusions |
| 0x8007001F | ERROR_GEN_FAILURE | Restart Windows Update service |
| 0x80080005 | CO_E_SERVER_EXEC_FAILURE | Run Windows Update troubleshooter |
| 0x8024402F | WU_E_PT_ECP_SUCCEEDED_WITH_ERRORS | Network issue; check proxy settings |
| 0x80244022 | WU_E_PT_HTTP_STATUS_SERVICE_UNAVAIL | Windows Update service overloaded; retry later |
| 0x80073712 | ERROR_SXS_COMPONENT_STORE_CORRUPT | DISM /RestoreHealth |
| 0xC1900101 | DRIVER_NOT_FOUND | Incompatible driver; update or remove |
| 0x8007042B | UPDATE_E_UNINSTALL_BLOCKED | Previous update failed; run troubleshooter |

---

## Common PowerShell Error Handling

```powershell
# Convert error code to message
$errorCode = 0x80070005
[System.ComponentModel.Win32Exception]$errorCode

# Get last Win32 error
$win32Error = [System.Runtime.InteropServices.Marshal]::GetLastWin32Error()
[System.ComponentModel.Win32Exception]$win32Error

# Parse HRESULT
function Get-HResultMessage {
    param([uint32]$HResult)
    $win32Code = $HResult -band 0xFFFF
    [System.ComponentModel.Win32Exception]$win32Code
}
Get-HResultMessage 0x80070005

# Error code lookup using net helpmsg
net helpmsg 5     # Returns: "Access is denied."
net helpmsg 2     # Returns: "The system cannot find the file specified."
```

---

## Event Log Error Reference

| Event ID | Log | Description | Action |
|---|---|---|---|
| 1000 | Application | Application crash (Application Error) | Check application logs, update/reinstall |
| 1001 | Application | Windows Error Reporting | View report for crash details |
| 4625 | Security | Failed logon attempt | Investigate source; check for brute force |
| 4648 | Security | Logon with explicit credentials | Investigate lateral movement potential |
| 4698/4702 | Security | Scheduled task created/modified | Verify legitimacy; check for persistence |
| 4719 | Security | System audit policy changed | Verify authorized change; alert if unexpected |
| 4728/4732 | Security | Member added to privileged group | Verify authorization |
| 4769 | Security | Kerberos service ticket requested | Kerberoasting if many requests from one source |
| 4771 | Security | Kerberos pre-auth failed | Password spraying indicator |
| 6005 | System | Event Log service started (boot) | Normal boot |
| 6006 | System | Event Log service stopped (shutdown) | Normal shutdown |
| 6013 | System | System uptime | System availability tracking |
| 7031 | System | Service terminated unexpectedly | Service failure; check dependencies |
| 7034 | System | Service crashed unexpectedly | Investigate service binary |
| 7045 | System | New service installed | **Critical** — verify legitimacy; possible malware installation |

```powershell
# Monitor for critical events in real-time
Register-WmiEvent -Query "SELECT * FROM Win32_NTLogEvent WHERE Logfile='Security' AND EventCode=4625" -Action {
    $event = $Event.SourceEventArgs.NewEvent
    Write-Warning "Failed Logon: $($event.Message)"
}

# Export error analysis report
$errorEvents = Get-WinEvent -LogName Application -MaxEvents 100 |
    Where-Object {$_.LevelDisplayName -in @("Error","Critical")}

$errorEvents | Group-Object Id | Sort-Object Count -Descending | Select-Object -First 20 Name, Count |
    Format-Table -AutoSize
```

> **Next:** Proceed to [Folder 20: Hidden Features and God Mode](../20_Hidden_Features_God_Mode/README.md) for advanced Windows features.
