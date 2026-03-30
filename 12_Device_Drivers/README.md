# Folder 12: Device Drivers and Hardware Integration

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak) and Hardware Engineers.
- **What:** Driver auditing using `driverquery`, signed driver enforcement via WHCP, and driver store management with `pnputil`.
- **Where:** Device Manager, Driver Store (`C:\Windows\System32\DriverStore`), Elevated CMD/PowerShell.
- **Why:** Kernel-mode drivers run at the highest privilege level. Malicious or vulnerable drivers are among the most powerful attack vectors in Windows. Driver hygiene is a critical security discipline.

---

## Driver Architecture in Windows

A device driver (Yantra Chalak) is kernel-mode code that interfaces between the OS and hardware. Drivers run at kernel privilege (Ring 0) — meaning a buggy or malicious driver can crash the entire system or gain complete control.

### Driver Types

| Type | Description | Example |
|---|---|---|
| Kernel-Mode Driver | Runs in kernel space; full hardware access | Network adapter, disk, USB |
| User-Mode Driver | Runs in user space (UMDF); isolated from kernel | USB webcam, printer |
| WDM (Windows Driver Model) | Legacy model; uses IRPs for I/O | Many Windows 7-era drivers |
| KMDF (Kernel-Mode Driver Framework) | Modern framework; simplified WDM | Most modern kernel drivers |
| UMDF (User-Mode Driver Framework) | User-mode; crash-isolated | USB HID, simple peripherals |
| Filter Driver | Intercepts IRP flow above/below another driver | Antivirus filesystem filter |
| Mini-driver | Hardware-specific portion of a driver pair | NDIS miniport |

---

## Driver Signing: Windows Hardware Compatibility Program (WHCP)

Windows enforces driver signature requirements to prevent malicious code from being loaded into the kernel. The enforcement levels:

| Mode | Description | Risk Level |
|---|---|---|
| **Kernel Mode Code Signing (KMCS)** | All kernel drivers must be signed | Default on 64-bit Windows |
| **HVCI / Memory Integrity** | Drivers must pass strict code integrity checks | Highest security |
| **Test Signing Mode** | Allows unsigned test drivers | Development only — never production |

```cmd
REM Check if test signing is enabled (security risk if yes)
bcdedit /enum | findstr "testsigning"
REM Result: "testsigning    Yes" = DANGEROUS - test mode enabled

REM Disable test signing (production systems)
bcdedit /set testsigning off

REM Check secure boot and code integrity status
powershell -Command "(Get-CimInstance -Namespace root\Microsoft\Windows\DeviceGuard -ClassName Win32_DeviceGuard).CodeIntegrityPolicyEnforcementStatus"
```

---

## driverquery: Auditing Installed Drivers

```cmd
REM List all installed drivers
driverquery

REM Verbose output with file paths and descriptions
driverquery /v /fo table

REM CSV export for analysis
driverquery /v /fo csv > C:\AuditReports\drivers_audit.csv

REM Show driver signing information
driverquery /si

REM Query a remote computer
driverquery /S RemoteComputer /U DOMAIN\Admin /P password
```

```powershell
# PowerShell equivalent - richer data
Get-WindowsDriver -Online | Select-Object Driver, OriginalFileName, ProviderName, 
    Date, Version, ClassDescription | Sort-Object Date -Descending | Format-Table -AutoSize

# Find unsigned drivers (security risk)
Get-WindowsDriver -Online | Where-Object {$_.IsInstalled -eq $true} | 
    Where-Object {$_.BootCritical -eq $false} | 
    ForEach-Object {
        $sig = Get-AuthenticodeSignature $_.OriginalFileName -ErrorAction SilentlyContinue
        if ($sig.Status -ne "Valid") {
            Write-Warning "Unsigned driver: $($_.OriginalFileName)"
        }
    }
```

---

## The Driver Store

The Driver Store (`C:\Windows\System32\DriverStore`) is the centralized repository for all driver packages. Drivers are staged here before installation. The staging process verifies the driver package's authenticity.

### Driver Store Structure

```
C:\Windows\System32\DriverStore\
    └── FileRepository\
            ├── printer.inf_amd64_xxxxxxxx\     ← Staged printer driver
            ├── net\                            ← Network drivers
            ├── usb\                            ← USB drivers
            └── ...
```

---

## pnputil: Driver Store Management

`pnputil.exe` (Plug and Play Utility - Lagao aur Chalao Upayogita) manages the driver store:

```cmd
REM List all third-party (non-Microsoft) driver packages
pnputil /enum-drivers

REM Add a driver package to the store (stage it)
pnputil /add-driver C:\Drivers\network\netdrv.inf

REM Add and immediately install
pnputil /add-driver C:\Drivers\network\netdrv.inf /install

REM Remove a driver package from the store
pnputil /delete-driver oem5.inf

REM Force-remove even if in use (requires driver not active)
pnputil /delete-driver oem5.inf /force

REM Scan for hardware changes (trigger PnP detection)
pnputil /scan-devices

REM Export all driver packages to a folder
pnputil /export-driver * C:\DriverBackup\
```

---

## Device Manager Operations

```powershell
# Get all installed devices
Get-PnpDevice | Select-Object Status, Class, FriendlyName | Sort-Object Class | Format-Table

# Get only problem devices (error code != 0)
Get-PnpDevice | Where-Object {$_.Status -ne "OK"} | 
    Select-Object Status, Class, FriendlyName, InstanceId

# Get device details including driver
Get-PnpDevice | Where-Object {$_.FriendlyName -like "*network*"} | 
    Get-PnpDeviceProperty | Select-Object KeyName, Data

# Disable a device
Disable-PnpDevice -InstanceId "PCI\VEN_8086&DEV_xxxx..." -Confirm:$false

# Enable a device
Enable-PnpDevice -InstanceId "PCI\VEN_8086&DEV_xxxx..." -Confirm:$false

# Uninstall a device (driver remains in store)
pnputil /remove-device "PCI\VEN_8086..."
```

---

## WDAC and Driver Blocklist

Windows Defender Application Control (WDAC) includes a Microsoft-maintained driver blocklist that prevents known vulnerable and malicious drivers from loading. This is critical for preventing "Bring Your Own Vulnerable Driver" (BYOVD) attacks.

```powershell
# Check if WDAC driver blocklist is enabled
(Get-CimInstance -Namespace root\Microsoft\Windows\DeviceGuard -ClassName Win32_DeviceGuard).CodeIntegrityPolicyEnforcementStatus

# Enable WDAC driver blocklist via Microsoft recommended policy
# This requires creating and deploying a WDAC policy — see Folder 15 for full WDAC guidance

# View currently blocked drivers
Get-WinEvent -LogName "Microsoft-Windows-CodeIntegrity/Operational" |
    Where-Object {$_.Id -eq 3076 -or $_.Id -eq 3077} |
    Select-Object TimeCreated, Message | Format-List
```

**Security Warning (Suraksha Chetavni):** BYOVD (Bring Your Own Vulnerable Driver) attacks are increasingly common. Attackers load a legitimately signed but vulnerable kernel driver to gain kernel-level code execution. Always enable the WDAC driver blocklist and keep it updated.

---

## Driver Troubleshooting

### Blue Screen Analysis for Driver Issues

```powershell
# View recent crash dumps
Get-ChildItem "C:\Windows\Minidump\" -ErrorAction SilentlyContinue | 
    Sort-Object LastWriteTime -Descending | Select-Object -First 10

# Parse crash dump (requires WinDbg or similar)
# Install WinDbg from the Windows SDK or Microsoft Store

# Basic BSOD information from Event Log
Get-WinEvent -LogName System | 
    Where-Object {$_.Id -eq 41 -or $_.Id -eq 1001} |
    Select-Object TimeCreated, Message | Format-List

# Check for driver verifier errors
verifier /query
verifier /reset     REM Disable Driver Verifier
```

### Driver Verifier (for testing)

Driver Verifier stresses drivers to expose bugs during development/testing:

```cmd
REM Enable Driver Verifier for specific driver (development/test only)
verifier /standard /driver netdrv.sys

REM Enable for all non-Microsoft drivers
verifier /standard /all

REM View verification settings
verifier /querysettings

REM Disable all verification (requires reboot)
verifier /reset
```

**Security Warning (Suraksha Chetavni):** Never enable Driver Verifier on production systems. It will likely cause crashes if drivers have bugs. Use only in isolated test environments.

> **Next:** Proceed to [Folder 13: Networking in Windows](../13_Networking/README.md) for OSI model mapping and protocol documentation.
