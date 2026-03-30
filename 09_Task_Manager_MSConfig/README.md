# Folder 09: Task Manager and MSConfig Deep Dive

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak), Power Users, and Support Technicians.
- **What:** Comprehensive analysis of Task Manager features and MSConfig (System Configuration) for startup, boot, and service management.
- **Where:** Windows 10/11 Task Manager (`taskmgr.exe`) and MSConfig (`msconfig.exe`).
- **Why:** Effective use of Task Manager and MSConfig enables rapid performance diagnosis, startup optimization, and service troubleshooting without requiring command-line expertise.

---

## Task Manager: Modern Interface (Windows 11)

Windows 11 introduced a completely redesigned Task Manager with a cleaner sidebar navigation and improved data presentation. Access via: `Ctrl + Shift + Esc` or right-click Taskbar → Task Manager.

### Processes Tab

The Processes tab shows all running applications, background processes, and Windows processes organized in a tree view:

- **Apps:** User-launched applications.
- **Background Processes:** Services and utilities running without user interaction.
- **Windows Processes:** Core OS processes (svchost.exe, csrss.exe, etc.).

```powershell
# Equivalent PowerShell for process listing with resource usage
Get-Process | Sort-Object CPU -Descending | 
    Select-Object Name, Id, CPU, @{N="RAM_MB";E={[math]::Round($_.WorkingSet64/1MB,2)}}, 
    @{N="Threads";E={$_.Threads.Count}} | 
    Format-Table -AutoSize
```

### Performance Tab

Real-time graphs for CPU, Memory, Disk, Network, and GPU. Key metrics:

| Metric | Description | Concern Threshold |
|---|---|---|
| CPU Usage | % of CPU capacity used | Sustained >85% |
| Available Memory | Free RAM | <10% of total RAM |
| Commit Charge | Virtual memory committed | >80% of commit limit |
| Disk Active Time | % disk I/O utilization | Sustained 100% |
| Network Throughput | Mbps in/out | Near link capacity |
| GPU Engine | Which engine is active (3D/Compute/Video) | 100% on bottleneck engine |

### App History Tab

Tracks CPU time and network usage per application over time. Useful for identifying apps that consume resources in the background.

### Startup Tab: The Modern Replacement for MSConfig Startup

The Startup tab shows all programs configured to launch at user login. It includes:

- **Startup Impact (Prarambh Prabhav):** Windows measures the impact of each startup item on boot time — High, Medium, Low, or Not Measured. Calculated based on CPU time and disk I/O during startup.
- **Status:** Enabled or Disabled.
- **Publisher:** Helps identify legitimate vs. suspicious entries.

```powershell
# View startup items via PowerShell (Registry-based)
Get-CimInstance -ClassName Win32_StartupCommand | 
    Select-Object Name, Command, Location, User | Format-Table -AutoSize

# Get startup items from registry
Get-ItemProperty "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
```

**Security Warning (Suraksha Chetavni):** Always investigate unfamiliar startup entries. Malware commonly registers persistence via `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`. Cross-reference unknown executables with VirusTotal.

### Services Tab

Lists all Windows services with their status. Provides a simplified view compared to Services.msc.

### Details Tab

The most powerful tab for advanced users — shows all processes with full details:
- PID, Status, CPU, Memory
- UAC Virtualization status
- Data Execution Prevention (DEP) status
- Operating System Context (32-bit vs 64-bit)

Right-click options: End task, End process tree, Set affinity, Set priority, Open file location, Go to service(s), Create dump file, Analyze wait chain.

### Users Tab

Shows all logged-in users and the resources each is consuming. Allows disconnecting or logging off other sessions (requires elevation).

---

## Startup Impact: How It Is Calculated

Windows calculates Startup Impact (Prarambh Prabhav) using telemetry from the boot session:

1. **CPU Time:** Total CPU cycles consumed by the process during the boot period (from process start until the desktop is fully loaded and interactive).
2. **Disk I/O:** Total bytes read from disk during the startup period.
3. **Weighted Score:** Microsoft's algorithm weights CPU time more heavily than disk I/O.

**Impact Thresholds:**
- **High:** CPU > 1 second OR Disk > 3MB during startup.
- **Medium:** CPU 0.3-1 second OR Disk 1-3MB.
- **Low:** CPU < 0.3 second AND Disk < 1MB.

---

## MSConfig: System Configuration Utility

MSConfig (`msconfig.exe`) is the legacy startup management tool. While the Startup tab functionality has moved to Task Manager in Windows 8+, MSConfig retains unique features:

### General Tab

| Option | Description |
|---|---|
| Normal Startup | Load all drivers and services (default) |
| Diagnostic Startup | Load only basic drivers and services (troubleshooting) |
| Selective Startup | Choose which items to load (advanced troubleshooting) |

### Boot Tab

```
Options:
- Safe Boot (Minimal, Alternate Shell, Active Directory Repair, Network)
- No GUI Boot (skip the Windows logo during boot)
- Boot Log (record boot to C:\Windows\ntbtlog.txt)
- Base Video (load generic VGA driver)
- OS Boot Information (show driver names during boot)
- Make all boot settings permanent (use with caution)
- Timeout: Default boot entry wait time (seconds)
```

**Equivalent CMD/PowerShell:**
```cmd
REM Enable Safe Boot (equivalent to MSConfig → Boot → Safe Boot)
bcdedit /set safeboot minimal

REM Enable boot log
bcdedit /set bootlog yes

REM Set boot menu timeout
bcdedit /timeout 10
```

### Services Tab

Allows selective enabling/disabling of Windows services. "Hide all Microsoft services" checkbox is essential for isolating third-party service issues.

```powershell
# Equivalent: List and manage services
Get-Service | Where-Object {$_.StartType -eq "Automatic" -and $_.Status -ne "Running"} |
    Select-Object DisplayName, Status, StartType

# Disable a non-essential service
Set-Service -Name "SysMain" -StartupType Disabled
Stop-Service -Name "SysMain" -Force
```

### Tools Tab

MSConfig's Tools tab provides one-click access to 20+ administrative tools:
- Computer Management, Event Viewer, Performance Monitor
- Registry Editor, System Information, Internet Options
- UAC Settings, Action Center, Task Manager

---

## Resource Monitor: The Advanced Performance Tool

Resource Monitor (resmon.exe) provides detailed, real-time resource usage data beyond Task Manager:

| Tab | Data Provided |
|---|---|
| Overview | CPU, Disk, Network, Memory summary |
| CPU | Per-process CPU usage, handles, modules |
| Memory | Physical memory usage, Working Set, Commit, Page Faults |
| Disk | Per-process disk I/O, queue length, file access details |
| Network | Per-process network connections, addresses, I/O |

```powershell
# Open Resource Monitor
Start-Process resmon.exe

# Equivalent data via PowerShell
# CPU per process
Get-Process | Sort-Object CPU -Desc | Select-Object -First 20 Name, Id, CPU

# Memory per process
Get-Process | Sort-Object WorkingSet64 -Desc | Select-Object -First 20 Name, Id, @{N="MB";E={[math]::Round($_.WorkingSet64/1MB,2)}}

# Network connections per process
Get-NetTCPConnection | Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess |
    Sort-Object State | Format-Table
```

---

## Performance Monitor (PerfMon)

Performance Monitor (perfmon.exe) enables long-term collection of performance data:

```powershell
# Create a Data Collector Set via PowerShell
$dcSet = New-Object -ComObject PLA.DataCollectorSet
$dcSet.DisplayName = "System Health Monitor"
$dcSet.Duration = 3600  # 1 hour

# Add performance counters
$counter = $dcSet.DataCollectors.CreateDataCollector(0)
$counter.Name = "PerfCounterLog"
$counter.FileName = "SystemHealth"
$counter.FileFormat = 0  # Binary
$counter.LogAppend = $true

# Start collecting
$dcSet.Commit("System Health Monitor", $null, 3)
$dcSet.Start($false)
```

> **Next:** Proceed to [Folder 10: Windows Services and Processes](../10_Windows_Services_Processes/README.md) for service architecture deep dive.
