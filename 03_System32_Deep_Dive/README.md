# Folder 03: System32 Deep Dive

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak), Security Analysts (Suraksha Vishleshan), and Malware Analysts.
- **What:** Identification and documentation of critical binaries in `%windir%\System32`, WOW64 redirection, and DLL analysis.
- **Where:** `C:\Windows\System32` and `C:\Windows\SysWOW64`.
- **Why:** System32 is the most critical directory in Windows. Understanding its contents enables threat detection, troubleshooting, and security hardening.

---

## System32: The Heart of Windows

`%windir%\System32` (typically `C:\Windows\System32`) contains the core executables, DLLs, and driver files that constitute the Windows operating system. On 64-bit Windows, this directory contains **64-bit** binaries. The naming is historical and intentionally counter-intuitive.

**Security Warning (Suraksha Chetavni):** System32 is a prime target for DLL hijacking and DLL side-loading attacks. Malware frequently masquerades as legitimate System32 binaries or places malicious DLLs in locations that are searched before System32 in the DLL search order.

---

## Critical System32 Binaries

| Binary | Description | Significance |
|---|---|---|
| `ntoskrnl.exe` | NT OS Kernel | Core kernel and executive |
| `hal.dll` | Hardware Abstraction Layer | Hardware mediation |
| `kernel32.dll` | Base API functions | Core Win32 subsystem |
| `ntdll.dll` | System service stubs | Executive function interface; lowest-level user-mode DLL |
| `user32.dll` | User interface logic | Window management, message passing |
| `gdi32.dll` | Graphics Device Interface | 2D graphics rendering |
| `advapi32.dll` | Advanced API | Security, registry, services APIs |
| `ws2_32.dll` | Windows Sockets 2 | Network communication |
| `lsass.exe` | Local Security Authority | Authentication; stores credential hashes (high-value target) |
| `csrss.exe` | Client/Server Runtime SS | Console management; Win32 subsystem |
| `svchost.exe` | Service Host | Hosts multiple Windows services in shared processes |
| `services.exe` | Service Control Manager | Manages Windows services lifecycle |
| `winlogon.exe` | Windows Logon | Manages user logon/logoff sessions |
| `smss.exe` | Session Manager | First user-mode process; initializes the system |
| `wininit.exe` | Windows Initialization | Starts services.exe, lsass.exe, lsm.exe |
| `cmd.exe` | Command Prompt | Legacy command interpreter |
| `powershell.exe` | Windows PowerShell | Modern scripting engine (Windows PowerShell 5.1) |
| `pwsh.exe` | PowerShell Core | Cross-platform PowerShell 7+ |
| `taskhost.exe` / `taskhostw.exe` | Task Host | Hosts .dll-based scheduled tasks |
| `explorer.exe` | Windows Shell | Desktop, taskbar, file browser (technically in Windows, not System32) |
| `regedit.exe` | Registry Editor | GUI registry management |
| `mmc.exe` | Microsoft Management Console | Administrative snap-in host |
| `eventvwr.exe` | Event Viewer | System event log GUI |
| `netstat.exe` | Network Statistics | Active connection display |
| `net.exe` | Network Command | User/group/share management |
| `sc.exe` | Service Control | Command-line service management |
| `reg.exe` | Registry CLI | Command-line registry management |
| `certutil.exe` | Certificate Utility | PKI management; **commonly abused** for file download/decode |
| `rundll32.exe` | Run DLL as App | Execute DLL functions; **commonly abused** by malware |
| `regsvr32.exe` | Register COM DLL | COM registration; **commonly abused** (Squiblydoo technique) |
| `mshta.exe` | HTML Application Host | Runs HTA files; **commonly abused** for script execution |
| `wscript.exe` | Windows Script Host | Runs VBScript/JScript; **commonly abused** |
| `cscript.exe` | Console Script Host | Console-mode script execution |
| `bitsadmin.exe` | BITS Admin | Manages download/upload jobs; **commonly abused** for download |
| `wmic.exe` | WMI Command Line | WMI management; **commonly abused** for lateral movement |
| `msiexec.exe` | Windows Installer | Package installation |
| `expand.exe` | Expand | Decompresses cabinet (.cab) files |
| `cipher.exe` | Cipher | EFS encryption management |
| `diskpart.exe` | Disk Partitioner | Advanced disk management |
| `format.exe` | Format Utility | Format disk volumes |
| `icacls.exe` | Integrity Control ACLs | Manage file/folder permissions |
| `takeown.exe` | Take Ownership | Change file ownership |
| `cacls.exe` | Change ACLs | Legacy ACL management (deprecated, use icacls) |
| `attrib.exe` | Attribute | Set/view file attributes (Hidden, System, Read-only) |
| `xcopy.exe` | Extended Copy | Advanced file copy |
| `robocopy.exe` | Robust File Copy | Enterprise-grade file synchronization |
| `sfc.exe` | System File Checker | Verify and repair system files |
| `dism.exe` | Deployment Image Servicing | OS image management |
| `bcdedit.exe` | BCD Editor | Boot configuration management |
| `bootrec.exe` | Boot Recovery | Repair boot records |
| `chkdsk.exe` | Check Disk | Filesystem integrity check |
| `defrag.exe` | Defragmenter | Disk defragmentation |
| `fsutil.exe` | Filesystem Utility | Advanced filesystem management |
| `ipconfig.exe` | IP Configuration | Network interface information |
| `ping.exe` | Ping | ICMP connectivity test |
| `tracert.exe` | Traceroute | Network path tracing |
| `nslookup.exe` | NS Lookup | DNS query tool |
| `netsh.exe` | Network Shell | Network configuration |
| `arp.exe` | ARP | ARP cache management |
| `route.exe` | Route | Routing table management |
| `tasklist.exe` | Task List | List running processes |
| `taskkill.exe` | Task Kill | Terminate processes |
| `at.exe` | AT Scheduler | Legacy task scheduler (deprecated) |
| `schtasks.exe` | Scheduled Tasks | Modern task scheduler |
| `wbadmin.exe` | Windows Backup Admin | Backup and recovery |
| `wevtutil.exe` | Event Utility | Event log management |
| `auditpol.exe` | Audit Policy | Manage audit policies |
| `secedit.exe` | Security Editor | Apply security templates |
| `gpupdate.exe` | GP Update | Refresh Group Policy |
| `gpresult.exe` | GP Result | Display applied Group Policy |
| `driverquery.exe` | Driver Query | List installed drivers |
| `pnputil.exe` | PnP Utility | Driver store management |
| `manage-bde.exe` | Manage BitLocker | BitLocker management |
| `sigcheck.exe` | Signature Check | Verify file signatures (Sysinternals) |

---

## LOLBins: Living Off the Land Binaries

Many legitimate System32 binaries are abused by attackers because they are trusted by Windows and security tools. These "LOLBins" (Living Off the Land Binaries - Zamin se jeene wale binary) allow attackers to execute malicious code without introducing their own executables.

```powershell
# Common LOLBin abuse detection - monitor these in your SIEM
$lolbins = @(
    "certutil.exe",   # File download, base64 decode
    "rundll32.exe",   # Execute DLL code
    "regsvr32.exe",   # Execute scriptlets (Squiblydoo)
    "mshta.exe",      # Execute HTA/VBScript
    "wscript.exe",    # Execute VBScript/JScript
    "cscript.exe",    # Execute scripts
    "bitsadmin.exe",  # File download
    "wmic.exe",       # Lateral movement, execution
    "msiexec.exe",    # Remote MSI execution
    "odbcconf.exe",   # DLL loading
    "ieexec.exe"      # Remote execution
)

# Check for suspicious processes with unusual parent processes
Get-WmiObject Win32_Process | Where-Object {$lolbins -contains $_.Name} | 
    Select-Object Name, ProcessId, ParentProcessId, CommandLine | Format-Table -AutoSize
```

---

## SysWOW64: The 32-bit System Directory

On 64-bit Windows, `%SystemRoot%\SysWOW64` contains the **32-bit** versions of system DLLs and executables. This directory is used by WOW64 to redirect 32-bit application calls.

```cmd
REM Verify a DLL's architecture (32-bit vs 64-bit)
dumpbin /headers C:\Windows\System32\kernel32.dll | findstr "machine"
dumpbin /headers C:\Windows\SysWOW64\kernel32.dll | findstr "machine"

REM List files in SysWOW64 that differ from System32
```

---

## System File Checker (SFC): Integrity Verification

```cmd
REM Verify all protected system files (requires elevation)
sfc /scannow

REM Scan but do not repair
sfc /verifyonly

REM Scan a specific file
sfc /scanfile=%windir%\System32\kernel32.dll

REM View SFC log
findstr /c:"[SR]" %windir%\Logs\CBS\CBS.log > %TEMP%\sfcdetails.txt
notepad %TEMP%\sfcdetails.txt
```

---

## DISM: Advanced Component Store Repair

When SFC cannot repair files (because the component store itself is damaged), use DISM:

```cmd
REM Check the health of the component store
DISM /Online /Cleanup-Image /CheckHealth

REM Scan and repair the component store (downloads from Windows Update)
DISM /Online /Cleanup-Image /RestoreHealth

REM Repair using a local source (offline media)
DISM /Online /Cleanup-Image /RestoreHealth /Source:E:\Sources\install.wim

REM After DISM repair, run SFC again
sfc /scannow
```

---

## DLL Search Order: A Security Critical Concept

When Windows loads a DLL, it searches directories in this order (standard mode):

1. The directory containing the application's executable.
2. `C:\Windows\System32`
3. `C:\Windows\System`
4. `C:\Windows`
5. Current directory.
6. Directories listed in the `PATH` environment variable.

**Security Warning (Suraksha Chetavni):** DLL hijacking exploits this search order. If an attacker can place a malicious DLL with the same name as a legitimate one in a directory that appears earlier in the search order (especially the application's own directory), Windows will load the malicious DLL instead. Always use "Safe DLL Search Mode" (enabled by default on modern Windows) and prefer absolute paths when loading DLLs in code.

```powershell
# Verify SafeDllSearchMode is enabled
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager" -Name SafeDllSearchMode
# Value of 1 = Enabled (default) - current directory is searched AFTER System32
# Value of 0 = Disabled - dangerous! Current directory searched BEFORE System32
```

> **Next:** Proceed to [Folder 04: Registry Architecture](../04_Registry_Architecture/README.md) for registry internals and forensics.
