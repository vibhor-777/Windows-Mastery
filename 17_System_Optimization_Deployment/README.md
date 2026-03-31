# Folder 17: System Optimization and Deployment

## 4Ws Structure

- **Who:** IT Operations teams, Desktop Engineers (Mej Abhiyanta), and MDM Administrators.
- **What:** Standard Operating Environment (SOE) creation, image deployment, and MDM policy management including Config Refresh.
- **Where:** Microsoft Deployment Toolkit (MDT), Windows Autopilot, Microsoft Intune, and Windows Configuration Designer.
- **Why:** A well-defined SOE ensures that every managed device starts from a consistent, secure, and optimized baseline, dramatically reducing support overhead and security gaps.

---

## Standard Operating Environment (SOE)

A Standard Operating Environment (SOE - Manak Parichalan Parivesh) is a standardized, tested Windows image that includes:
- Base OS (Windows 11 Enterprise, specific build)
- Security baseline (DISA STIG or CIS Benchmark applied)
- Required applications pre-installed
- Standard configuration (time zone, language, desktop)
- Management agents (Defender for Endpoint, Intune agent)

### SOE Design Principles

| Principle | Implementation |
|---|---|
| **Thin image** | Minimal pre-installed software; deploy apps separately |
| **Security baseline** | Apply CIS/DISA hardening during image creation |
| **Repeatability** | Fully automated, unattended installation |
| **Auditability** | Document every customization with justification |
| **Updateability** | Image updated monthly with cumulative updates |

---

## Windows Image Capture with DISM

```powershell
# Capture a reference Windows image (from reference PC)
# Boot reference PC to WinPE, then:

# Capture to WIM file
DISM /Capture-Image /ImageFile:D:\Images\Win11_SOE.wim /CaptureDir:C:\ /Name:"Windows 11 SOE" /Description:"Standard Operating Environment - Build 23H2 - $(Get-Date -Format yyyyMM)"

# Add to existing WIM (multi-index)
DISM /Append-Image /ImageFile:D:\Images\Win11_SOE.wim /CaptureDir:C:\ /Name:"Windows 11 SOE v2"

# List images in WIM
DISM /Get-ImageInfo /ImageFile:D:\Images\Win11_SOE.wim

# Mount WIM for modification
DISM /Mount-Image /ImageFile:D:\Images\Win11_SOE.wim /Index:1 /MountDir:C:\Mount

# Add drivers to mounted image
DISM /Image:C:\Mount /Add-Driver /Driver:C:\Drivers\ /Recurse

# Add language pack
DISM /Image:C:\Mount /Add-Package /PackagePath:C:\LangPacks\Microsoft-Windows-Client-Language-Pack_x64_en-us.cab

# Apply Windows updates to mounted image (offline servicing)
DISM /Image:C:\Mount /Add-Package /PackagePath:C:\Updates\Windows11-KB5027231-x64.msu

# Remove bloatware/provisioned apps
DISM /Image:C:\Mount /Remove-ProvisionedAppxPackage /PackageName:Microsoft.Xbox.TCUI_1.23.28002.0_neutral_~_8wekyb3d8bbwe

# Save and unmount
DISM /Unmount-Image /MountDir:C:\Mount /Commit

# Optimize WIM (reduce size)
DISM /Export-Image /SourceImageFile:D:\Images\Win11_SOE.wim /SourceIndex:1 /DestinationImageFile:D:\Images\Win11_SOE_optimized.wim /Compress:maximum
```

---

## Windows Autopilot: Zero-Touch Deployment

Windows Autopilot enables zero-touch deployment — devices arrive pre-configured without IT needing to touch them.

```powershell
# Get hardware hash for Autopilot registration (run on the device)
# Install the required module
Install-Script -Name Get-WindowsAutoPilotInfo -Force

# Generate hardware hash
Get-WindowsAutoPilotInfo -OutputFile C:\AutoPilot_Hash.csv

# The CSV can then be uploaded to Intune → Devices → Enroll → Windows enrollment → Windows Autopilot → Devices

# Or use the following to directly upload to Intune via PowerShell
Install-Module -Name Microsoft.Graph.Intune -Force
Connect-MSGraph
Import-AutoPilotCSV -CSVFile C:\AutoPilot_Hash.csv -GroupTag "SOE-Standard"
```

---

## Microsoft Intune: MDM Policy Management

Intune (previously Microsoft Endpoint Manager) provides cloud-based MDM policy management for Windows 11.

### Key Policy Areas in Intune

| Policy Area | Examples |
|---|---|
| Configuration Profiles | Settings catalog, ADMX templates, scripts |
| Compliance Policies | Require BitLocker, minimum OS version, Defender enabled |
| Security Baselines | Microsoft Security Baseline for Windows 11 |
| App Policies | Win32 app deployment, Microsoft Store |
| Endpoint Security | ASR rules, Firewall, Exploit protection |
| Conditional Access | Block non-compliant devices |

---

## Config Refresh: Preventing MDM Policy Drift

Config Refresh (Config Naveenikaran) is a Windows 11 feature that automatically re-applies MDM policies every 90 minutes without requiring a full MDM sync cycle. This prevents configuration drift — where local changes override centrally managed policies.

```powershell
# Config Refresh is controlled via MDM CSP (Configuration Service Provider)
# Policy path: ./Device/Vendor/MSFT/Policy/Config/Experience/ConfigureConfigRefresh

# Check Config Refresh status via registry
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device\Experience" | 
    Select-Object ConfigureConfigRefresh*

# In Intune, configure via Settings Catalog:
# Experience > Configure Config Refresh: Enabled
# Experience > Config Refresh Cadence (minutes): 90 (default)

# Monitor MDM sync events
Get-WinEvent -LogName "Microsoft-Windows-DeviceManagement-Enterprise-Diagnostics-Provider/Admin" |
    Where-Object {$_.Id -in @(1800, 1801, 1802)} |
    Select-Object TimeCreated, Id, Message | Format-List
```

---

## Performance Optimization Techniques

### System Performance Tuning

```powershell
# Set power plan to High Performance (for workstations)
powercfg /setactive SCHEME_MIN
powercfg /list  # View all power schemes

# Disable memory compression if RAM is plentiful and compression causing CPU overhead
Disable-MMAgent -MemoryCompression

# Optimize visual effects for performance
$uiRegPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\VisualEffects"
Set-ItemProperty -Path $uiRegPath -Name "VisualFXSetting" -Value 2  # 2 = Adjust for best performance

# Adjust paging file
# Manual paging file (disable auto-management for consistent performance)
$cs = Get-WmiObject Win32_ComputerSystem
$cs.AutomaticManagedPagefile = $false
$cs.Put()

$pf = Get-WmiObject Win32_PageFileSetting
if ($pf) { $pf.Delete() }

$totalRamGB = [math]::Round((Get-WmiObject Win32_ComputerSystem).TotalPhysicalMemory / 1GB)
$pagefileSizeMB = $totalRamGB * 1024 * 1.5  # 1.5x RAM

Set-WmiInstance -Class Win32_PageFileSetting -Arguments @{
    Name = "C:\pagefile.sys"
    InitialSize = $pagefileSizeMB
    MaximumSize = $pagefileSizeMB
}
```

### Storage Optimization

```powershell
# Run Storage Sense (automated cleanup)
# Enable via Settings → System → Storage → Storage Sense

# Manual disk cleanup
cleanmgr /sageset:65535 /sagerun:65535  # Cleanup all categories

# DISM cleanup of superseded update packages
DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase

# Defragment (for HDDs only - SSDs use TRIM)
Optimize-Volume -DriveLetter C -Defrag -Verbose

# TRIM for SSDs
Optimize-Volume -DriveLetter C -ReTrim -Verbose

# Enable TRIM if not enabled
fsutil behavior set DisableDeleteNotify 0
```

---

## Application Deployment with Winget and SCCM

```powershell
# Winget bulk installation (deploy during SOE build)
$apps = @(
    "Microsoft.Edge",
    "Microsoft.VisualStudioCode",
    "Git.Git",
    "Microsoft.Teams",
    "7zip.7zip",
    "Notepad++.Notepad++"
)

foreach ($app in $apps) {
    winget install --id $app --silent --accept-package-agreements --accept-source-agreements
    Write-Host "Installed: $app"
}

# Winget export all installed apps
winget export -o C:\Apps_manifest.json

# Winget import (restore apps on new machine)
winget import -i C:\Apps_manifest.json --accept-package-agreements
```

---

## Windows Update Management

```powershell
# Check Windows Update status
Get-WindowsUpdate  # Requires PSWindowsUpdate module

# Install all available updates
Install-WindowsUpdate -AcceptAll -AutoReboot

# Configure Windows Update via PowerShell (MDM managed via Intune)
# Key registry settings:
reg export "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" C:\Backups\WU_policy_backup.reg

# Defer quality updates by 7 days
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name "DeferQualityUpdates" -Value 1 -Type DWord
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name "DeferQualityUpdatesPeriodInDays" -Value 7 -Type DWord

# View installed updates
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 20

# View update history
Get-WinEvent -LogName "Microsoft-Windows-WindowsUpdateClient/Operational" |
    Where-Object {$_.Id -in @(19, 20, 43)} |
    Select-Object TimeCreated, Message | Format-List
```

> **Next:** Proceed to [Folder 18: Troubleshooting with Sysinternals](../18_Troubleshooting_Sysinternals/README.md) for diagnostic workflows.
