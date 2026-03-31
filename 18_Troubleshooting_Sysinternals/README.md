# Folder 18: Troubleshooting with Sysinternals

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak), Malware Analysts, and Security Engineers.
- **What:** Sysinternals Suite diagnostic workflows — Process Monitor, Process Explorer, Autoruns, and supporting tools.
- **Where:** Sysinternals tools run on live Windows systems; download from `https://learn.microsoft.com/en-us/sysinternals/`.
- **Why:** Sysinternals tools provide unparalleled visibility into Windows internals — far beyond what Task Manager and Event Viewer offer. They are the primary tools used by Microsoft engineers for "Case of the Unexplained (Asamajhik mamlon ka nidan)" scenarios.

---

## Sysinternals Overview

The Sysinternals Suite (now owned by Microsoft) contains 70+ tools for monitoring, diagnosing, and troubleshooting Windows systems.

| Tool | Primary Use | Key Capability |
|---|---|---|
| **Process Monitor (Procmon)** | Real-time I/O monitoring | Captures file, registry, network, and process activity |
| **Process Explorer (Procexp)** | Advanced Task Manager replacement | DLL analysis, handle inspection, VirusTotal integration |
| **Autoruns** | Startup persistence analysis | 200+ ASEP locations |
| **PsTools** | Remote administration | PsExec, PsKill, PsInfo, etc. |
| **TCPView** | Network connection viewer | Real-time TCP/UDP connections with process |
| **Strings** | String extraction | Dumps printable strings from binaries |
| **Sigcheck** | File signature verification | Certificate chain, VirusTotal hash check |
| **Handle** | Open handle finder | "File In Use" diagnosis |
| **Disk2vhd** | Physical to virtual conversion | Disaster recovery |
| **RAMMap** | Physical memory analysis | Memory type breakdown |
| **VMMap** | Virtual memory analysis | Per-process virtual address space |
| **WinObj** | Windows object manager | Kernel namespace visualization |
| **AccessChk** | Permission auditing | Find misconfigured permissions |
| **Bginfo** | Desktop info display | System info on wallpaper |
| **DebugView** | Debug output capture | `OutputDebugString` messages |
| **NotMyFault** | Crash generation | Testing crash dumps |
| **RegJump** | Registry navigation | Jump to registry path in regedit |
| **ZoomIt** | Screen annotation | Presentations |

---

## Process Monitor (Procmon): The Swiss Army Knife

Process Monitor captures real-time file system, registry, process, and network activity for every process on the system. It is the definitive tool for diagnosing "Access Denied" errors, application crashes, and malware behavior.

### Procmon Interface

- **Event Classes:** Filter by File System, Registry, Network, Process/Thread, Profiling events.
- **Columns:** Process name, PID, Operation, Path, Result, Detail.
- **Result:** SUCCESS, ACCESS DENIED, PATH NOT FOUND, NAME COLLISION, etc.

### Technical Execution: Diagnosing "File In Use" Errors

**Who:** System Administrators.  
**What:** Use Procmon to identify which process holds a lock on a file.  
**Where:** Procmon running as Administrator.  
**Why:** "File In Use" errors prevent deletion, movement, or modification of files — Procmon pinpoints the locking process.

```
1. Open Procmon as Administrator
2. Set filter: Path contains "filename_in_use.txt" → Include
3. Observe which process is accessing the file
4. Note the operation: CreateFile with SHARE_WRITE denied = file lock
5. Close the offending process or wait for it to release the lock

Alternative: Use Sysinternals Handle.exe
handle.exe "filename_in_use.txt"    ← Shows which process has the handle

PowerShell alternative:
$handlePath = "C:\LockedFile.txt"
$processes = Get-Process | Where-Object {
    try { $_.Modules | Where-Object {$_.FileName -eq $handlePath} } catch { $false }
}
```

### Procmon Filtering Techniques

```
Key Filters for Malware Analysis:
- Category is Write → Show all write operations (file drops, registry changes)
- Result is ACCESS DENIED → Permission issues
- Path contains Run → Registry autostart entries
- Operation is RegSetValue → Registry modifications
- Process Name is powershell.exe → PowerShell activity

Save filter as: Malware_Analysis.PMF for reuse
```

### Process Tree Analysis

Procmon's Process Tree (`Ctrl+T`) visualizes parent-child process relationships — essential for detecting:
- `cmd.exe` spawned by `winword.exe` (Office macro execution)
- `powershell.exe` spawned by `wscript.exe` (script-based attack)
- `net.exe` spawned by unexpected parents (lateral movement)

---

## Process Explorer (Procexp): The Advanced Task Manager

Process Explorer provides a tree view of all running processes with their parent-child relationships, resource usage, and security details.

### Lower Pane: DLL and Handle Analysis

The Lower Pane (Nichla Bhag) shows additional detail for the selected process:

- **DLL View:** Shows all DLLs loaded by the selected process, including their paths, descriptions, and versions. Useful for detecting DLL hijacking (malicious DLL in unexpected location).
- **Handle View:** Shows all open handles (files, registry keys, events, mutexes) held by the process. Useful for diagnosing "File In Use" errors and understanding process behavior.

```
To use:
1. View → Lower Pane View → DLLs (Ctrl+D) or Handles (Ctrl+H)
2. Select a process in the upper pane
3. Lower pane populates automatically

DLL Hijacking Detection:
- Look for DLLs not in System32 or expected application directory
- Check for DLLs with no company name or unusual publishers
- Use Process Explorer's VirusTotal integration:
  Options → VirusTotal.com → Check VirusTotal.com
```

### Security Tab: Process Permissions Analysis

Right-click a process → Properties → Security tab shows:
- The process's access token (user, groups, privileges)
- Enabled/disabled privileges (SeDebugPrivilege, SeImpersonatePrivilege, etc.)
- Whether the process is elevated

**Suspicious Privilege Flags:**
| Privilege | Risk Level | Typical Use |
|---|---|---|
| SeDebugPrivilege | Critical | Allows debugging any process (including LSASS) |
| SeImpersonatePrivilege | High | Token impersonation (common in local privilege escalation) |
| SeTakeOwnershipPrivilege | High | Take ownership of any object |
| SeLoadDriverPrivilege | High | Load kernel drivers |
| SeBackupPrivilege | Medium | Read any file (backup bypass) |

### VirusTotal Integration

```
Options → VirusTotal.com → Check VirusTotal.com
- Procexp sends the SHA256 hash of each process executable to VirusTotal
- Results shown in "VirusTotal" column: 0/70 (clean), X/70 (detected)
- Right-click → Check VirusTotal.com → Opens browser with detailed results

Hacker Trick: Use this to quickly identify known malware masquerading as legitimate processes
```

---

## Autoruns: Persistence Analysis

Autoruns (Swachalan Vishleshan) is the most comprehensive tool for analyzing Autostart Extensibility Points (ASEPs - Swachalan Vistaar Bindu). It shows everything configured to run automatically in Windows.

### ASEP Categories (200+ Locations)

| Category | Key Locations |
|---|---|
| **Logon** | HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run, HKCU\...\Run, Startup folders |
| **Explorer** | Shell extensions, context menu handlers, namespace extensions |
| **Internet Explorer** | BHOs (Browser Helper Objects), toolbars |
| **Scheduled Tasks** | All task scheduler entries |
| **Services** | All Windows services |
| **Drivers** | All kernel-mode drivers |
| **Codecs** | Audio/video codec registrations |
| **Boot Execute** | Programs that run before Windows fully loads |
| **Image Hijacking** | Image File Execution Options debugger entries |
| **Known DLLs** | Known DLL list overrides |
| **Winlogon** | Notification packages, credential providers |
| **Print Monitors** | Print spooler extensions |
| **LSA Providers** | Authentication packages |
| **Network Providers** | Network credential providers |
| **AppInit DLLs** | DLLs injected into every GUI process |
| **WMI** | WMI subscriptions (common malware persistence) |
| **Office** | Office add-ins and templates |

### Malware Detection with Autoruns

```
Key Autoruns Techniques for Malware Hunting:

1. Options → Scan Options → Check VirusTotal.com
   → Red entries = known malware
   → Yellow entries = file not found (broken persistence or cleaned malware)

2. Filter to show only non-Microsoft entries:
   Options → Hide Microsoft Entries
   → Reduces noise; focus on third-party and unknown entries

3. Look for suspicious characteristics:
   → Entries with no Publisher or Description
   → Entries pointing to Temp, AppData\Roaming, ProgramData directories
   → Encoded command lines: powershell -enc <base64>
   → Randomly-named executables in user-writable directories

4. Critical ASEP: WMI subscriptions (often missed)
   → Autoruns → WMI tab
   → Look for any entries (should be empty on clean systems)
   → WMI persistence survives reboots and is difficult to detect

5. Check HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\BootExecute
   → Default value: autocheck autochk *
   → Any additional entries indicate rootkit-level persistence
```

---

## Strings: Finding Malware Artifacts

**Hacker Trick:** Use `strings2.exe` (or the Sysinternals `strings.exe`) to dump printable strings from process address spaces or binary files to find obfuscated commands, C2 URLs, and encryption keys.

```cmd
REM Extract strings from a suspicious executable
strings.exe -n 8 C:\Suspicious\malware.exe > C:\Analysis\malware_strings.txt

REM Live memory dump of a process (requires admin)
strings.exe -p <PID> > C:\Analysis\process_memory_strings.txt

REM Search for common IOCs in extracted strings
findstr /i "http:// https:// cmd.exe powershell HKLM\\ HKCU\\" C:\Analysis\malware_strings.txt

REM Find base64-encoded content (often used for obfuscation)
findstr "[A-Za-z0-9+/]\\{40,\\}" C:\Analysis\malware_strings.txt
```

---

## TCPView: Real-Time Network Analysis

```
TCPView shows all active TCP and UDP connections:
- Process name and PID
- Protocol (TCP/UDP)
- Local address and port
- Remote address and port
- State (LISTENING, ESTABLISHED, TIME_WAIT, etc.)

Color coding:
- Green = New connection
- Red = Closing connection
- Yellow = Modified (address changed)

Malware Hunting Tips:
1. Look for unknown processes with ESTABLISHED connections to unusual IPs
2. Right-click → WHOIS → Identify the organization behind remote IPs
3. Look for processes with LISTENING on unusual ports
4. Check for known malicious ports: 4444, 6666, 31337 (common Metasploit defaults)
```

---

## AccessChk: Permission Auditing

```cmd
REM Find services with writable service configurations (privilege escalation risk)
accesschk.exe -uwcqv "Authenticated Users" * /accepteula

REM Find writable directories in System PATH
accesschk.exe -uwdq C:\Windows\System32 /accepteula

REM Find files everyone can write to
accesschk.exe -uwf "Everyone" C:\Windows\System32 /accepteula

REM Check registry key permissions
accesschk.exe -kwuq HKLM\SYSTEM\CurrentControlSet\Services /accepteula

REM Find services running as SYSTEM that are modifiable
accesschk.exe -uwcqv "Users" * /accepteula
```

> **Next:** Proceed to [Folder 19: Error Codes and Fixes Database](../19_Error_Codes_Fixes/README.md) for systematic error resolution.
