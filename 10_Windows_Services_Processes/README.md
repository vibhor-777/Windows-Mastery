# Folder 10: Windows Services and Processes

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak) and Security Engineers.
- **What:** Complete documentation of Windows Services (Sewa) architecture, the Service Control Manager (SCM), and Svchost.exe process hosting.
- **Where:** Services.msc, SC.exe, PowerShell, and the SCM registry hive.
- **Why:** Services are the backbone of Windows functionality and a primary target for malware persistence. Understanding service architecture enables precise management, hardening, and threat detection.

---

## Windows Services Architecture

A Windows Service (Sewa) is a long-running executable that performs specific functions and can be managed independently of user sessions. Services can start before any user logs in (critical for servers) and continue running after users log off.

### Service States

| State | Description |
|---|---|
| Running | Service is actively executing |
| Stopped | Service is not running |
| Paused | Service is temporarily suspended (not all services support this) |
| Start Pending | Service is in the process of starting |
| Stop Pending | Service is in the process of stopping |
| Pause Pending | Service is in the process of pausing |
| Continue Pending | Service is resuming from paused state |

### Service Start Types

| Start Type | Description | Use Case |
|---|---|---|
| Automatic | Starts automatically during boot | Critical services |
| Automatic (Delayed) | Starts after boot completes (reduces boot time impact) | Non-critical services |
| Manual | Started only when explicitly requested | On-demand services |
| Disabled | Cannot be started | Security hardening |
| Boot | Loaded by OS loader before SCM initializes | Filesystem, storage drivers |
| System | Loaded by NT kernel during initialization | Core drivers |

---

## The Service Control Manager (SCM)

The Service Control Manager (SCM - Sewa Niyantran Prabandhan) is the Windows component responsible for managing the entire service lifecycle. It runs as a part of `services.exe`.

SCM Functions:
1. **Enumerate services:** Reads service configurations from `HKLM\SYSTEM\CurrentControlSet\Services`.
2. **Start/stop/pause services:** Sends control codes to service processes.
3. **Handle service failures:** Implements recovery actions (restart, run program, reboot).
4. **Maintain service database:** In-memory list of all service states.

```powershell
# View the SCM database (all services)
Get-Service | Select-Object Name, DisplayName, Status, StartType | Sort-Object DisplayName

# Start a service
Start-Service -Name "Spooler"

# Stop a service
Stop-Service -Name "Spooler" -Force

# Restart a service
Restart-Service -Name "Spooler"

# Get detailed service info
Get-Service "Spooler" | Select-Object *

# Set startup type
Set-Service -Name "Spooler" -StartupType Automatic
```

---

## Svchost.exe: The Service Host

Svchost.exe (Host Process for Windows Services - Windows Sewaaon ke liye Manch Prakriya) is a crucial architecture element. Instead of each service running as a separate executable, many services are implemented as DLLs and hosted within shared Svchost.exe processes. This optimizes memory usage by allowing multiple services to share a single process.

### Why Multiple Svchost Instances?

Services are grouped by privilege level and security context. Common groupings:

| Svchost Group | Typical Services Hosted | Privilege Level |
|---|---|---|
| `svchost.exe -k LocalServiceNetworkRestricted` | DHCP Client, DNS Client | Local Service |
| `svchost.exe -k LocalSystemNetworkRestricted` | BFE (Firewall), Dnscache | Local System |
| `svchost.exe -k netsvcs` | BITS, Schedule, SENS | Local System |
| `svchost.exe -k NetworkService` | CryptSvc, Dnscache | Network Service |
| `svchost.exe -k DcomLaunch` | Power, PlugPlay, RpcEpMapper | Local System |

**Windows 10 1703+ Change:** Microsoft split many services into individual Svchost processes on systems with >3.5GB RAM, making it easier to correlate resource usage to specific services.

```powershell
# Show all Svchost processes and their hosted services
Get-WmiObject Win32_Service | Where-Object {$_.PathName -like "*svchost*"} |
    Select-Object Name, DisplayName, ProcessId, PathName |
    Sort-Object ProcessId | Format-Table -AutoSize

# Get services in a specific svchost process
$pid = (Get-Process svchost)[0].Id
Get-WmiObject Win32_Service | Where-Object {$_.ProcessId -eq $pid} |
    Select-Object Name, State, DisplayName
```

---

## Service Security: Service Accounts

Services run under specific security contexts (user accounts):

| Account | Security Level | Network Access | Use Case |
|---|---|---|---|
| **SYSTEM** (LocalSystem) | Maximum — full local privileges | Computer account credentials | Core OS services |
| **LOCAL SERVICE** | Reduced privileges | Anonymous network credentials | Network-facing services with limited needs |
| **NETWORK SERVICE** | Reduced privileges | Computer account credentials | Network services needing domain authentication |
| **Managed Service Account (MSA)** | Domain account, auto-password rotation | Domain credentials | Domain-joined service accounts |
| **Group MSA (gMSA)** | Shared domain account across servers | Domain credentials | Load-balanced/clustered services |
| **Virtual Account** | Auto-created per-service | Limited | Per-service isolation |
| **Custom User Account** | Configured explicitly | Standard user rights | Custom applications |

**Security Warning (Suraksha Chetavni):** Running services as SYSTEM provides maximum privileges but maximum risk. If a SYSTEM-privileged service is exploited, the attacker gains full control. Apply the Principle of Least Privilege (Nyuntam Adhikar ka Siddhant) — use dedicated service accounts with only the required permissions.

```powershell
# Audit services running as SYSTEM
Get-WmiObject Win32_Service | 
    Where-Object {$_.StartName -eq "LocalSystem" -and $_.State -eq "Running"} |
    Select-Object Name, DisplayName, PathName |
    Sort-Object Name

# Check for services running as Administrator or elevated users (security risk)
Get-WmiObject Win32_Service |
    Where-Object {$_.StartName -notmatch "Local|Network|NT AUTHORITY" -and $_.StartName -ne $null} |
    Select-Object Name, DisplayName, StartName
```

---

## Service Registry Configuration

Each service's configuration is stored in: `HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>`

```powershell
# Backup service configuration before modification
reg export "HKLM\SYSTEM\CurrentControlSet\Services\Spooler" C:\Backups\Spooler_service_backup.reg

# View service registry entry
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Spooler"
```

Key registry values per service:

| Value Name | Type | Description |
|---|---|---|
| `ImagePath` | REG_EXPAND_SZ | Path to the service executable or DLL |
| `Start` | REG_DWORD | Start type (0=Boot, 1=System, 2=Auto, 3=Manual, 4=Disabled) |
| `Type` | REG_DWORD | Service type (0x1=Kernel driver, 0x10=Own process, 0x20=Shared process) |
| `ObjectName` | REG_SZ | Account name the service runs under |
| `DisplayName` | REG_SZ | Human-readable service name |
| `Description` | REG_SZ | Service description |
| `FailureActions` | REG_BINARY | Recovery actions for service failures |
| `DependOnService` | REG_MULTI_SZ | Services that must start first |

---

## Service Hardening

```powershell
# Disable unnecessary services (security hardening - adjust to your environment)
$servicesToDisable = @(
    "XblAuthManager",    # Xbox Live Auth Manager
    "XblGameSave",       # Xbox Live Game Save
    "XboxGipSvc",        # Xbox Accessory Management
    "RemoteRegistry",    # Remote Registry (high-risk)
    "Fax",               # Fax service
    "TelnetC",           # Telnet client (if installed)
    "SNMP",              # SNMP (if not needed)
    "SSDPSRV",           # SSDP Discovery (UPnP)
    "upnphost"           # UPnP Device Host
)

foreach ($svc in $servicesToDisable) {
    if (Get-Service -Name $svc -ErrorAction SilentlyContinue) {
        Stop-Service -Name $svc -Force -ErrorAction SilentlyContinue
        Set-Service -Name $svc -StartupType Disabled
        Write-Host "Disabled: $svc"
    }
}

# Configure service failure recovery
sc.exe failure Spooler reset= 86400 actions= restart/5000/restart/10000/restart/60000
```

---

## Critical Security Services to Monitor

| Service Name | Display Name | Security Role | Action if Stopped |
|---|---|---|---|
| `MpsSvc` | Windows Defender Firewall | Network packet filtering | Critical — restart immediately |
| `WdNisSvc` | Microsoft Defender Antivirus Network Inspection | Network-based threat detection | Critical |
| `WinDefend` | Microsoft Defender Antivirus Service | Real-time malware protection | Critical |
| `EventLog` | Windows Event Log | Security audit logging | Critical — may indicate tampering |
| `CryptSvc` | Cryptographic Services | Certificate validation | High |
| `lsass` | Local Security Authority Process | Authentication (not a service, but critical process) | System crash if stopped |
| `KeyIso` | CNG Key Isolation | Protects private keys | High |
| `PolicyAgent` | IPsec Policy Agent | IPsec enforcement | Medium-High |

```powershell
# Monitor critical security services
$criticalServices = @("MpsSvc", "WdNisSvc", "WinDefend", "EventLog")
foreach ($svc in $criticalServices) {
    $status = Get-Service -Name $svc -ErrorAction SilentlyContinue
    if ($status.Status -ne "Running") {
        Write-Warning "CRITICAL SERVICE NOT RUNNING: $($status.DisplayName)"
    }
}
```

> **Next:** Proceed to [Folder 11: Boot Process and UEFI/BIOS](../11_Boot_Process_UEFI_BIOS/README.md) for pre-OS boot architecture.
