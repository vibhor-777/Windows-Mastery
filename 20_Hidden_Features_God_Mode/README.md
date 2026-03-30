# Folder 20: Hidden Features and "God Mode"

## 4Ws Structure

- **Who:** Power Users (Shakti Upayogakarta) and System Administrators (Pranali Prashasak).
- **What:** Enabling Windows God Mode, advanced administrative folders, Windows Sandbox, and other hidden features.
- **Where:** Windows 11 desktop, Settings, and optional feature management.
- **Why:** These hidden features provide centralized access to advanced settings, enable safe malware testing environments, and unlock productive workflows that are not exposed in the standard UI.

---

## Windows "God Mode": The Ultimate Control Panel

God Mode (Ishwar Sthiti) is a special folder that provides access to all Windows administrative settings from a single location — over 200 settings shortcuts organized by category.

### Creating God Mode

```cmd
REM Method 1: CMD
mkdir "%USERPROFILE%\Desktop\GodMode.{ED7BA470-8E54-465E-825C-99712043E01C}"

REM Method 2: PowerShell
$godModePath = "$env:USERPROFILE\Desktop\GodMode.{ED7BA470-8E54-465E-825C-99712043E01C}"
New-Item -ItemType Directory -Path $godModePath
```

The folder name uses a CLSID (Class Identifier) that Windows recognizes as the All Tasks shell namespace extension. Double-clicking opens a comprehensive settings view.

### Other Useful CLSID Folders

| Name | CLSID | Contents |
|---|---|---|
| God Mode (All Tasks) | `{ED7BA470-8E54-465E-825C-99712043E01C}` | All 200+ Windows settings |
| Action Center | `{BB64F8A7-BEE7-4E1A-AB8D-7D8273F7FDB6}` | Security and maintenance |
| Administrative Tools | `{D20EA4E1-3957-11d2-A40B-0C5020524153}` | All admin tools |
| Fonts | `{BD84B380-8CA2-1069-AB1D-08000948F534}` | Font management |
| Network Connections | `{7007ACC7-3202-11D1-AAD2-00805FC1270E}` | Network adapters |
| Programs and Features | `{7b81be6a-ce2b-4676-a29e-eb907a5126c5}` | Installed software |
| User Accounts | `{60632754-c523-4b62-b45c-4172da012619}` | Account management |

```powershell
# Create all useful CLSID folders at once
$desktop = "$env:USERPROFILE\Desktop\AdminFolders"
New-Item -ItemType Directory -Path $desktop -Force

$folders = @{
    "God Mode" = "ED7BA470-8E54-465E-825C-99712043E01C"
    "Action Center" = "BB64F8A7-BEE7-4E1A-AB8D-7D8273F7FDB6"
    "Network Connections" = "7007ACC7-3202-11D1-AAD2-00805FC1270E"
    "Power Options" = "025A5937-A6BE-4686-A844-36FE4BEC8B6D"
    "Programs and Features" = "7b81be6a-ce2b-4676-a29e-eb907a5126c5"
    "User Accounts" = "60632754-c523-4b62-b45c-4172da012619"
}

foreach ($name in $folders.Keys) {
    $path = "$desktop\$name.{$($folders[$name])}"
    New-Item -ItemType Directory -Path $path -ErrorAction SilentlyContinue
}
```

---

## Windows Sandbox: Safe Malware Testing

Windows Sandbox (Ret-ka-dibba - Sand Box) provides a lightweight, isolated virtual environment for safely running untrusted applications. The Sandbox uses hardware virtualization and Windows Container technology.

### Key Sandbox Properties

| Property | Description |
|---|---|
| **Disposable** | Deletes all data when closed — no persistent malware |
| **Hardware isolated** | Uses Microsoft Hypervisor for complete isolation |
| **Clean snapshot** | Each session starts from a clean Windows 11 image |
| **No persistence** | Cannot affect the host system |
| **Clipboard** | Optional bidirectional clipboard sharing |
| **Network** | Optional network access |

### Enabling Windows Sandbox

```powershell
# Enable Windows Sandbox (requires Windows 11 Pro/Enterprise, Virtualization support)
Enable-WindowsOptionalFeature -FeatureName "Containers-DisposableClientVM" -Online -All

# Verify it's enabled
Get-WindowsOptionalFeature -Online -FeatureName "Containers-DisposableClientVM"

# Alternative: via DISM
DISM /Online /Enable-Feature /FeatureName:Containers-DisposableClientVM /All /NoRestart
```

### Custom Sandbox Configuration (.wsb files)

```xml
<!-- Save as sandbox_config.wsb -->
<Configuration>
    <vGPU>Disable</vGPU>
    <Networking>Disable</Networking>
    <MappedFolders>
        <MappedFolder>
            <HostFolder>C:\SandboxShare</HostFolder>
            <SandboxFolder>C:\Users\WDAGUtilityAccount\Desktop\SharedFiles</SandboxFolder>
            <ReadOnly>true</ReadOnly>
        </MappedFolder>
    </MappedFolders>
    <LogonCommand>
        <Command>C:\Users\WDAGUtilityAccount\Desktop\SharedFiles\setup.cmd</Command>
    </LogonCommand>
    <MemoryInMB>4096</MemoryInMB>
</Configuration>
```

```powershell
# Create a sandbox configuration for malware analysis
$wsbContent = @"
<Configuration>
    <vGPU>Disable</vGPU>
    <Networking>Disable</Networking>
    <MappedFolders>
        <MappedFolder>
            <HostFolder>C:\MalwareSamples</HostFolder>
            <SandboxFolder>C:\Users\WDAGUtilityAccount\Desktop\Samples</SandboxFolder>
            <ReadOnly>true</ReadOnly>
        </MappedFolder>
    </MappedFolders>
    <MemoryInMB>4096</MemoryInMB>
</Configuration>
"@

$wsbContent | Out-File "C:\MalwareAnalysis.wsb" -Encoding UTF8

# Open the sandbox
Start-Process "C:\MalwareAnalysis.wsb"
```

---

## Hidden Windows Features and Optional Components

```powershell
# List all optional features
Get-WindowsOptionalFeature -Online | 
    Select-Object FeatureName, State | 
    Sort-Object FeatureName | Format-Table -AutoSize

# Enable useful optional features
$featuresToEnable = @(
    "Microsoft-Hyper-V-All",         # Hyper-V virtualization
    "Containers-DisposableClientVM", # Windows Sandbox
    "Microsoft-Windows-Subsystem-Linux", # WSL
    "VirtualMachinePlatform",        # WSL 2 support
    "TelnetClient",                  # Telnet (diagnostic use only)
    "TFTP"                           # TFTP client
)

foreach ($feature in $featuresToEnable) {
    Enable-WindowsOptionalFeature -Online -FeatureName $feature -NoRestart -ErrorAction SilentlyContinue
}
```

---

## Hidden Diagnostic Tools

```cmd
REM Windows Diagnostic Infrastructure (WDI)
perfmon /report         REM Generate system performance report
perfmon /sys            REM Open System Diagnostics

REM Windows Memory Diagnostic
mdsched.exe             REM Schedule memory test on next boot

REM DirectX Diagnostic
dxdiag                  REM DirectX and hardware info

REM System Information
msinfo32                REM Comprehensive system info
msinfo32 /nfo C:\sysinfo.nfo  REM Save to file

REM Advanced Network Diagnostics
netsh trace start capture=yes   REM Start packet capture
netsh trace stop                 REM Stop and save to ETL file

REM Reliability Monitor
perfmon /rel            REM Shows reliability history and problem events

REM Windows Experience Index
winsat formal           REM Run Windows System Assessment Tool
winsat disk -drive c    REM Disk performance assessment
winsat cpuformal        REM CPU performance assessment
```

---

## Power User: Customizing Windows Explorer

```powershell
# Show hidden files and file extensions in Explorer
$explorerPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced"

# Backup first
reg export "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced" C:\Backups\Explorer_Advanced_backup.reg

# Show hidden files
Set-ItemProperty -Path $explorerPath -Name "Hidden" -Value 1
# Show file extensions
Set-ItemProperty -Path $explorerPath -Name "HideFileExt" -Value 0
# Show protected OS files (use with care)
Set-ItemProperty -Path $explorerPath -Name "ShowSuperHidden" -Value 1
# Show full path in title bar
Set-ItemProperty -Path $explorerPath -Name "FullPath" -Value 1
# Open File Explorer to This PC instead of Quick Access
Set-ItemProperty -Path $explorerPath -Name "LaunchTo" -Value 1

# Restart Explorer to apply changes
Stop-Process -Name explorer -Force
Start-Process explorer
```

> **Next:** Proceed to [Folder 21: Advanced Registry and System Tweaks](../21_Advanced_Registry_Tweaks/README.md) for performance and security registry modifications.
