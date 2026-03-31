# Folder 21: Advanced Registry and System Tweaks

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak) and Power Users.
- **What:** Registry-based performance tweaks and security hardening modifications — with mandatory backup commands.
- **Where:** Registry Editor (`regedit.exe`) or PowerShell. All modifications require elevation.
- **Why:** Direct registry modifications provide control over Windows behavior that is not exposed in any GUI — enabling precise performance tuning and security hardening.

---

## Mandatory Rule: Always Back Up Before Modifying

**Every registry modification in this folder MUST be preceded by the following backup command:**

```cmd
reg export <KeyPath> <BackupFile>.reg
```

Example:
```cmd
reg export HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System C:\Backups\UAC_Policy.reg
```

---

## Performance Tweaks

### 1. Disable Search Indexing for Encrypted Files

**Who:** Security-conscious administrators.  
**What:** Prevent Windows Search from indexing encrypted files.  
**Where:** Registry — `HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search`  
**Why:** Indexing encrypted files can create indexed copies of sensitive data in an unencrypted index database.

```cmd
reg export "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" C:\Backups\WindowsSearch_policy.reg

reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v "PreventIndexingEncryptedFiles" /t REG_DWORD /d 1 /f
```

### 2. Disable Last Access Timestamp

**Why:** Every file read updates the Last Access time on NTFS, generating additional write I/O. On servers with many reads, this significantly impacts storage performance.

```cmd
reg export "HKLM\SYSTEM\CurrentControlSet\Control\FileSystem" C:\Backups\FileSystem.reg

reg add "HKLM\SYSTEM\CurrentControlSet\Control\FileSystem" /v "NtfsDisableLastAccessUpdate" /t REG_DWORD /d 80000001 /f
```

```cmd
REM Alternative via fsutil
fsutil behavior set disablelastaccess 1
```

### 3. Machine Inactivity Limit (Auto Screen Lock)

**Who:** Administrators (security hardening).  
**What:** Configure automatic session lock after inactivity.  
**Where:** `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`  
**Why:** Unattended workstations with unlocked sessions are a major security risk.

```cmd
reg export "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" C:\Backups\System_Policy.reg

REM Lock workstation after 300 seconds (5 minutes) of inactivity
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v "InactivityTimeoutSecs" /t REG_DWORD /d 300 /f
```

### 4. Optimize TCP Window Auto-Tuning

```cmd
reg export "HKLM\SOFTWARE\Policies\Microsoft\Windows\QoS" C:\Backups\QoS.reg

REM Set TCP auto-tuning to normal (default)
netsh int tcp set global autotuninglevel=normal

REM For high-latency WAN environments, try:
netsh int tcp set global autotuninglevel=highlyrestricted

REM Enable TCP Chimney Offload (for supported NICs)
netsh int tcp set global chimney=enabled
netsh int tcp set global rss=enabled
```

### 5. Reduce Start Menu Search Delay

```powershell
# Backup
reg export "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Serialize" C:\Backups\Serialize_backup.reg

# Remove start delay for startup applications
Set-ItemProperty -Path "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Serialize" -Name "StartupDelayInMSec" -Value 0 -Type DWord -Force
```

### 6. Disable Hibernation (for SSD systems)

```cmd
REM Disable hibernation (removes hiberfil.sys, frees disk space equal to installed RAM)
REM Security Warning (Suraksha Chetavni): Disabling hibernation on laptops means
REM "Fast Startup" will still work but true hibernate will not.
reg export "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Power" C:\Backups\Power.reg

powercfg /hibernate off
```

---

## Security Hardening Registry Tweaks

### 7. Disable SMBv1 (Critical)

```cmd
reg export "HKLM\SYSTEM\CurrentControlSet\Services\mrxsmb10" C:\Backups\mrxsmb10.reg

REM Disable the SMBv1 driver
reg add "HKLM\SYSTEM\CurrentControlSet\Services\mrxsmb10" /v "Start" /t REG_DWORD /d 4 /f
```

### 8. Disable NTLM Authentication (Force NTLMv2 minimum)

```cmd
reg export "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" C:\Backups\LSA.reg

REM Force NTLMv2 and refuse LM and NTLMv1
REM Level 5 = Send NTLMv2 response only/refuse LM & NTLM
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "LmCompatibilityLevel" /t REG_DWORD /d 5 /f
```

### 9. Disable AutoRun and AutoPlay

**Why:** AutoRun on removable media is a common malware delivery vector (e.g., BadUSB attacks).

```cmd
reg export "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer" C:\Backups\Explorer_policy.reg
reg export "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer" C:\Backups\Explorer_user_policy.reg

REM Disable AutoRun for all drives (0xFF = 255 = all drives)
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer" /v "NoDriveTypeAutoRun" /t REG_DWORD /d 255 /f
reg add "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer" /v "NoDriveTypeAutoRun" /t REG_DWORD /d 255 /f
```

### 10. Enable LSA Protection (Credential Hardening)

**Why:** RunAsPPL (Protected Process Light) makes the LSASS process a Protected Process, preventing even Administrator-level processes from injecting into or reading LSASS memory — protecting credential hashes from tools like Mimikatz.

```cmd
reg export "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" C:\Backups\LSA_PPL.reg

REM Enable LSA Protected Process Light
REM Security Warning (Suraksha Chetavni): This may break some third-party antivirus products
REM that inject into LSASS. Test thoroughly before deploying.
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "RunAsPPL" /t REG_DWORD /d 1 /f
```

### 11. Disable WDigest Plain-Text Password Caching

**Why:** WDigest (Windows Digest Authentication) caches credentials in plain text in LSASS memory. Mimikatz exploits this. Disabling prevents plain-text credential extraction.

```cmd
reg export "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest" C:\Backups\WDigest.reg

reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest" /v "UseLogonCredential" /t REG_DWORD /d 0 /f
```

### 12. Configure RDP Security

```cmd
reg export "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" C:\Backups\TerminalServer.reg

REM Disable RDP if not needed
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v "fDenyTSConnections" /t REG_DWORD /d 1 /f

REM If RDP is needed, require Network Level Authentication (NLA)
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v "UserAuthentication" /t REG_DWORD /d 1 /f

REM Set minimum encryption level to High (128-bit)
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v "MinEncryptionLevel" /t REG_DWORD /d 3 /f
```

### 13. Disable Null Session Access

```cmd
reg export "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" C:\Backups\LSA_NullSession.reg

REM Restrict anonymous access (null sessions)
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "RestrictAnonymous" /t REG_DWORD /d 1 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "RestrictAnonymousSAM" /t REG_DWORD /d 1 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "EveryoneIncludesAnonymous" /t REG_DWORD /d 0 /f
```

---

## Comprehensive Security Hardening Script

```powershell
# Windows Security Hardening Script
# Run as Administrator
# Security Warning: Test in non-production environment first

function Backup-RegistryKey {
    param([string]$KeyPath, [string]$BackupDir = "C:\SecurityHardeningBackups")
    
    if (-not (Test-Path $BackupDir)) { New-Item -ItemType Directory -Path $BackupDir }
    $safeName = $KeyPath -replace "[:\\]", "_"
    $backupFile = "$BackupDir\$safeName_$(Get-Date -Format yyyyMMdd).reg"
    reg export $KeyPath $backupFile /y 2>$null
    Write-Verbose "Backed up: $KeyPath → $backupFile"
}

# Apply hardening settings
$hardeningSettings = @(
    @{ Key="HKLM\SYSTEM\CurrentControlSet\Control\Lsa"; Name="LmCompatibilityLevel"; Value=5; Type="DWord" },
    @{ Key="HKLM\SYSTEM\CurrentControlSet\Control\Lsa"; Name="RunAsPPL"; Value=1; Type="DWord" },
    @{ Key="HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest"; Name="UseLogonCredential"; Value=0; Type="DWord" },
    @{ Key="HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer"; Name="NoDriveTypeAutoRun"; Value=255; Type="DWord" },
    @{ Key="HKLM\SYSTEM\CurrentControlSet\Control\Lsa"; Name="RestrictAnonymous"; Value=1; Type="DWord" }
)

foreach ($setting in $hardeningSettings) {
    Backup-RegistryKey -KeyPath $setting.Key
    $regPath = "Registry::" + $setting.Key
    if (-not (Test-Path $regPath)) { New-Item -Path $regPath -Force }
    Set-ItemProperty -Path $regPath -Name $setting.Name -Value $setting.Value -Type $setting.Type
    Write-Host "Applied: $($setting.Key)\$($setting.Name) = $($setting.Value)"
}

Write-Host "`nHardening complete. Reboot required for some settings." -ForegroundColor Green
```

> **Next:** Proceed to [Folder 22: Modern Developer Setup](../22_Modern_Developer_Setup/README.md) for development environment configuration.
