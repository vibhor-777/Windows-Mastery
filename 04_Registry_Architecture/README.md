# Folder 04: Registry Architecture (Panjikaran)

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak), Forensic Analysts (Nyayik Vishleshan), and Security Engineers.
- **What:** The Windows Registry (Panjikaran) — a hierarchical database of configuration settings stored in binary hive files.
- **Where:** Physical hive files on disk + in-memory registry managed by the Configuration Manager.
- **Why:** The registry is the single most important configuration database in Windows. Understanding its architecture enables precise configuration, forensic investigation, and malware detection.

---

## Registry Overview: The Hierarchical Configuration Database

The Windows Registry (Panjikaran) is a centralized hierarchical database that stores configuration settings for the operating system, applications, services, hardware, and user preferences. It replaces the legacy INI file approach used in Windows 3.x.

### Registry Structure: Keys, Values, and Data

- **Key (Kunji):** A container node in the hierarchy, analogous to a folder.
- **Value (Mulya):** A named data entry within a key, analogous to a file.
- **Data (Data/Aankda):** The actual configuration data stored in a value.

### Registry Data Types

| Type | Identifier | Description | Example |
|---|---|---|---|
| REG_SZ | 1 | Null-terminated string | `"C:\Windows"` |
| REG_EXPAND_SZ | 2 | String with environment variables | `"%SystemRoot%\System32"` |
| REG_BINARY | 3 | Raw binary data | Security descriptors |
| REG_DWORD | 4 | 32-bit unsigned integer | `0x00000001` |
| REG_DWORD_BIG_ENDIAN | 5 | 32-bit integer, big-endian | Rare, legacy |
| REG_LINK | 6 | Symbolic link to another key | Internal use |
| REG_MULTI_SZ | 7 | Array of null-terminated strings | Environment paths |
| REG_RESOURCE_LIST | 8 | Hardware resource list | Device Manager |
| REG_QWORD | 11 | 64-bit unsigned integer | Large numeric values |

---

## The Five Root Keys (Panch Muul Kunji)

| Root Key | Abbreviation | Contents |
|---|---|---|
| HKEY_LOCAL_MACHINE | HKLM | Machine-wide settings; hardware, OS, software |
| HKEY_CURRENT_USER | HKCU | Settings for the currently logged-in user |
| HKEY_USERS | HKU | Settings for all user profiles (SID-based) |
| HKEY_CLASSES_ROOT | HKCR | File associations and COM class registrations (merged view of HKLM\SOFTWARE\Classes and HKCU\SOFTWARE\Classes) |
| HKEY_CURRENT_CONFIG | HKCC | Current hardware profile (alias for HKLM\SYSTEM\CurrentControlSet\Hardware Profiles\Current) |

---

## Registry Hives: Physical Storage (Chhatte)

Registry data is stored in binary "Hive (Chhatta)" files on disk. These are not text files; they use a proprietary binary format called the Registry Hive Format.

| Hive Name | File Path | Contents |
|---|---|---|
| SYSTEM | `%SystemRoot%\System32\config\SYSTEM` | System configuration, services, drivers |
| SOFTWARE | `%SystemRoot%\System32\config\SOFTWARE` | Installed software, machine-wide settings |
| SAM | `%SystemRoot%\System32\config\SAM` | Security Accounts Manager; local user credentials |
| SECURITY | `%SystemRoot%\System32\config\SECURITY` | Security policy, LSA secrets |
| DEFAULT | `%SystemRoot%\System32\config\DEFAULT` | Default user profile template |
| NTUSER.DAT | `%UserProfile%\NTUSER.DAT` | Per-user settings (HKCU when user is logged in) |
| USRCLASS.DAT | `%LocalAppData%\Microsoft\Windows\UsrClass.dat` | Per-user COM classes and file associations |
| BCD | `\Boot\BCD` (EFI) / `C:\Boot\BCD` | Boot Configuration Database |

**Security Warning (Suraksha Chetavni):** The SAM hive contains hashed local account passwords. It is locked by the OS during operation. Offline access to SAM (e.g., from a bootable USB) allows password hash extraction. Protect physical access to systems.

---

## Registry-in-Memory: The Configuration Manager

The Configuration Manager (CM) is the NT Executive component that manages the registry. It maintains the registry in memory using the following internal structures:

- **_CMHIVE:** Represents a complete registry hive. Contains the HHIVE structure and CM-specific metadata.
- **HHIVE:** The core hive data structure. Contains the `BaseBlock` (hive header with signature `regf`), allocation bitmaps, and the actual cell storage.
- **Cell (Kosh):** The fundamental unit of registry storage. Cells can be key nodes (`nk`), values (`vk`), security descriptors (`sk`), or big data (`db`).
- **Cell Index (Kosh Suchi):** A 32-bit value used as an offset within the hive to locate cells. The Configuration Manager uses these to map cells to virtual addresses.

---

## Registry Operations with reg.exe

```cmd
REM Always back up a registry key before modifying it
reg export HKLM\SOFTWARE\Policies C:\Backups\Policies_backup.reg

REM Query a specific value
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion /v ProgramFilesDir

REM Add a new key and value
reg add HKCU\Software\MyApp /v Version /t REG_SZ /d "1.0" /f

REM Modify an existing value
reg add HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters /v TcpMaxConnectRetransmissions /t REG_DWORD /d 2 /f

REM Delete a value
reg delete HKCU\Software\MyApp /v OldSetting /f

REM Delete a key and all subkeys
reg delete HKCU\Software\OldApp /f

REM Compare two registry keys
reg compare HKCU\Software\App HKCU\Software\App_backup /s

REM Import a .reg file
reg import C:\Backups\Policies_backup.reg
```

---

## PowerShell Registry Management

```powershell
# Navigate registry like a filesystem
Set-Location HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion

# Get all values in a key
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion"

# Get a specific value
(Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion" -Name "ProgramFilesDir").ProgramFilesDir

# Create a new key
New-Item -Path "HKCU:\Software\MyCompany\MyApp" -Force

# Set a value
Set-ItemProperty -Path "HKCU:\Software\MyCompany\MyApp" -Name "Setting" -Value "Enabled" -Type String

# Remove a key
Remove-Item -Path "HKCU:\Software\MyCompany\MyApp" -Recurse

# Search for a value across the registry (slow on large hives)
Get-ChildItem -Path "HKLM:\SOFTWARE" -Recurse -ErrorAction SilentlyContinue | 
    Where-Object { $_.GetValueNames() -contains "ImagePath" }
```

---

## Forensically Significant Registry Keys

### Persistence Mechanisms (Malware Sthirata)

| Key Path | Purpose | Forensic Significance |
|---|---|---|
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` | Auto-start at machine boot | **High** - Common malware persistence |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce` | Run once at next boot | Medium |
| `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` | Auto-start for current user | **High** - Common malware persistence |
| `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` | Logon hooks (Shell, Userinit) | **High** - Modifying Userinit or Shell = persistence |
| `HKLM\SYSTEM\CurrentControlSet\Services` | Windows services | **High** - Malware as a service |
| `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options` | Debugger injection | **High** - IFEO hijacking |
| `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\BootExecute` | Pre-boot execution | **Critical** - Rootkit persistence |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders` | Shell folder locations | Medium |

### System Information Keys

| Key Path | Information |
|---|---|
| `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion` | OS version, build, install date |
| `HKLM\SYSTEM\CurrentControlSet\Control\ComputerName` | System hostname |
| `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters` | Network configuration |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RecentDocs` | Recently accessed files |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU` | Run dialog history |
| `HKLM\SYSTEM\CurrentControlSet\Enum\USBSTOR` | USB device history |

---

## Registry Backup Strategies

```powershell
# Export entire HKLM to a file (large - may take minutes)
reg export HKLM C:\Backups\HKLM_full.reg

# Backup specific hive files using Volume Shadow Copy
# Security Warning (Suraksha Chetavni): Always use VSS for live hive backup, 
# as direct file copy will fail due to locks.
$shadow = (Get-WmiObject -List Win32_ShadowCopy).Create("C:\", "ClientAccessible")
$shadowPath = (Get-WmiObject Win32_ShadowCopy | Sort-Object InstallDate | Select-Object -Last 1).DeviceObject
cmd /c mklink /d C:\ShadowMount "$shadowPath\"
Copy-Item "C:\ShadowMount\Windows\System32\config\SAM" C:\Backups\SAM_backup
Remove-Item C:\ShadowMount

# Automated registry backup with PowerShell
$backupPath = "C:\RegistryBackups\$(Get-Date -Format 'yyyyMMdd_HHmmss')"
New-Item -ItemType Directory -Path $backupPath
reg export HKLM\SYSTEM "$backupPath\SYSTEM.reg"
reg export HKLM\SOFTWARE "$backupPath\SOFTWARE.reg"
reg export HKCU "$backupPath\HKCU.reg"
```

> **Next:** Proceed to [Folder 05: Environment Variables](../05_Environment_Variables/README.md) for PATH and system variable management.
