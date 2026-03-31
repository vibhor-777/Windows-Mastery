# Folder 15: Security, Defender, and Hardware-Based Protection

## 4Ws Structure

- **Who:** Security Administrators (Suraksha Prashasak) and Security Operations Center (SOC) analysts.
- **What:** Windows 11 high-priority security features — TPM 2.0, Microsoft Pluton, VBS, HVCI, Credential Guard, and Attack Surface Reduction (ASR) rules.
- **Where:** Windows Security Center, Group Policy, PowerShell security cmdlets, Registry.
- **Why:** This folder represents the core defensive capability of a hardened Windows 11 system. Every feature documented here directly reduces the attack surface and limits the damage of successful intrusions.

---

## Windows 11 Security Feature Matrix

| Feature | Implementation | Threat Mitigated |
|---|---|---|
| TPM 2.0 | Hardware-based key storage | Key extraction, identity theft |
| Microsoft Pluton | Security processor in CPU | Bus-sniffing, firmware attacks |
| VBS | Virtualization-based security isolation | Kernel-level credential theft |
| HVCI | Memory integrity enforcement | Unsigned kernel code, rootkits |
| Credential Guard | Isolates LSA secrets in VTL1 | Pass-the-hash, pass-the-ticket |
| Secure Boot | Pre-OS boot verification | Bootkits, rootkits |
| BitLocker + TPM | Full disk encryption | Physical data theft |
| Microsoft Defender AV | Real-time malware protection | Malware, ransomware |
| Microsoft Defender SmartScreen | URL/app reputation | Phishing, malicious downloads |
| Controlled Folder Access | Ransomware protection | Unauthorized file modification |
| Attack Surface Reduction (ASR) | Rule-based exploit blocking | Script-based attacks, Office macros |
| Network Protection | IP/URL reputation blocking at kernel | C2 communication |
| Microsoft Defender for Endpoint | EDR platform | Advanced persistent threats |
| Windows Hello for Business | Biometric/PIN-based passwordless auth | Credential theft, phishing |
| Smart App Control | AI-based application control | Unknown malware |

---

## Checking Security Status

```powershell
# Comprehensive security status check
function Get-WindowsSecurityStatus {
    Write-Host "`n=== Windows Security Status Report ===" -ForegroundColor Cyan
    
    # Windows Defender / Antivirus
    $av = Get-MpComputerStatus
    Write-Host "`n[Antivirus]" -ForegroundColor Yellow
    Write-Host "Real-time Protection: $($av.RealTimeProtectionEnabled)"
    Write-Host "Last Definition Update: $($av.AntivirusSignatureLastUpdated)"
    Write-Host "Quick Scan Age: $($av.QuickScanAge) days"
    
    # Device Guard / VBS
    $dg = Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
    Write-Host "`n[Virtualization-Based Security]" -ForegroundColor Yellow
    Write-Host "VBS Status: $($dg.VirtualizationBasedSecurityStatus)"
    Write-Host "HVCI Status: $($dg.HypervisorEnforcedCodeIntegrityStatus)"
    Write-Host "Credential Guard: $($dg.SecurityServicesRunning -contains 1)"
    
    # Secure Boot
    try {
        $sb = Confirm-SecureBootUEFI
        Write-Host "`n[Secure Boot]" -ForegroundColor Yellow
        Write-Host "Secure Boot: $sb"
    } catch { Write-Host "Secure Boot: Not UEFI system" }
    
    # BitLocker
    Write-Host "`n[BitLocker]" -ForegroundColor Yellow
    Get-BitLockerVolume | Select-Object MountPoint, EncryptionMethod, ProtectionStatus, 
        EncryptionPercentage | Format-Table -AutoSize
    
    # Firewall
    Write-Host "`n[Windows Firewall]" -ForegroundColor Yellow
    Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction | Format-Table -AutoSize
    
    # TPM
    Write-Host "`n[TPM]" -ForegroundColor Yellow
    $tpm = Get-Tpm
    Write-Host "TPM Present: $($tpm.TpmPresent)"
    Write-Host "TPM Enabled: $($tpm.TpmEnabled)"
    Write-Host "TPM Ready: $($tpm.TpmReady)"
}

Get-WindowsSecurityStatus
```

---

## Windows Defender: Deep Configuration

### Real-Time Protection Management

```powershell
# View all Defender settings
Get-MpPreference | Format-List

# Configure exclusions (use with extreme caution)
Add-MpPreference -ExclusionPath "C:\TrustedApp\"
Add-MpPreference -ExclusionProcess "trustedapp.exe"
Add-MpPreference -ExclusionExtension ".log"

# Remove exclusions
Remove-MpPreference -ExclusionPath "C:\TrustedApp\"

# Run a quick scan
Start-MpScan -ScanType QuickScan

# Run a full scan
Start-MpScan -ScanType FullScan

# Run a custom path scan
Start-MpScan -ScanType CustomScan -ScanPath "C:\Downloads\"

# Update definitions
Update-MpSignature

# View threat history
Get-MpThreatDetection | Select-Object ThreatID, ThreatName, ActionSuccess, DetectionTime |
    Sort-Object DetectionTime -Descending | Format-Table -AutoSize

# View quarantined items
Get-MpThreat | Format-List

# Restore from quarantine (with caution - only for false positives)
# Use Windows Security GUI for safety
```

---

## Attack Surface Reduction (ASR) Rules

ASR (Hamla Satah Nyunata) rules are specific behavior-based detection rules in Microsoft Defender that block commonly abused techniques.

### Technical Execution: Enable ASR Rule for Script Blocking

**Who:** Security Administrators.  
**What:** Enable rule `d3e037e1-3eb8-44c8-a917-57927947596d` (Block JavaScript/VBScript from launching downloaded content).  
**Where:** Group Policy: `Computer Configuration\Policies\Administrative Templates\Windows Components\Microsoft Defender Antivirus\Microsoft Defender Exploit Guard\Attack Surface Reduction`.  
**Why:** Prevents malware from using legitimate Windows script engines (wscript.exe, cscript.exe) to execute malicious content downloaded from the internet.

```powershell
# Enable ASR rules via PowerShell
# Modes: 0=Disable, 1=Block, 2=Audit, 6=Warn

# Block JavaScript/VBScript from launching downloaded content
Add-MpPreference -AttackSurfaceReductionRules_Ids d3e037e1-3eb8-44c8-a917-57927947596d -AttackSurfaceReductionRules_Actions Enabled

# Comprehensive ASR rule deployment
$asrRules = @{
    "BE9BA2D9-53EA-4CDC-84E5-9B1EEEE46550" = 1  # Block executable content from email/webmail
    "D4F940AB-401B-4EFC-AADC-AD5F3C50688A" = 1  # Block Office from creating child processes
    "3B576869-A4EC-4529-8536-B80A7769E899" = 1  # Block Office from creating executable content
    "75668C1F-73B5-4CF0-BB93-3ECF5CB7CC84" = 1  # Block Office from injecting into other processes
    "D3E037E1-3EB8-44C8-A917-57927947596D" = 1  # Block JS/VBScript from downloading content
    "5BEB7EFE-FD9A-4556-801D-275E5FFC04CC" = 1  # Block execution of potentially obfuscated scripts
    "92E97FA1-2EDF-4476-BDD6-9DD0B4DDDC7B" = 1  # Block Win32 API calls from Office macros
    "01443614-CD74-433A-B99E-2ECDC07BFC25" = 1  # Block untrusted/unsigned processes from USB
    "C1DB55AB-C21A-4637-BB3F-A12568109D35" = 1  # Block persistence via WMI event subscription
    "9E6C4E1F-7D60-472F-BA1A-A39EF669E4B2" = 1  # Block credential stealing from LSASS
    "D1E49AAC-8F56-4280-B9BA-993A6D77406C" = 1  # Block process creation from PSExec/WMI
    "B2B3F03D-6A65-4F7B-A9C7-1C7EF74A9BA4" = 1  # Block untrusted Office add-ins
    "26190899-1602-49E8-8B27-EB1D0A1CE869" = 1  # Block Office comm apps from creating child proc
    "7674BA52-37EB-4A4F-A9A1-F0F9A1619A2C" = 1  # Block Adobe Reader from creating child processes
    "E6DB77E5-3DF2-4CF1-B95A-636979351E5B" = 1  # Block persistence via removable storage WMI
}

foreach ($rule in $asrRules.GetEnumerator()) {
    Add-MpPreference -AttackSurfaceReductionRules_Ids $rule.Key -AttackSurfaceReductionRules_Actions $rule.Value
}

# View current ASR rule states
(Get-MpPreference).AttackSurfaceReductionRules_Ids
(Get-MpPreference).AttackSurfaceReductionRules_Actions
```

---

## Controlled Folder Access (Ransomware Protection)

```powershell
# Enable Controlled Folder Access
Set-MpPreference -EnableControlledFolderAccess Enabled

# View protected folders
(Get-MpPreference).ControlledFolderAccessProtectedFolders

# Add a protected folder
Add-MpPreference -ControlledFolderAccessProtectedFolders "C:\ImportantData"

# Allow a trusted application
Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\TrustedApp\app.exe"

# Set to Audit mode (log without blocking - useful for initial deployment)
Set-MpPreference -EnableControlledFolderAccess AuditMode
```

---

## Credential Guard Configuration

```powershell
# Check Credential Guard status
$dg = Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
$dg.SecurityServicesRunning
# 1 = Credential Guard Running
# 2 = HVCI Running

# Enable VBS and Credential Guard via registry
reg export "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" C:\Backups\DeviceGuard_backup.reg
reg export "HKLM\SYSTEM\CurrentControlSet\Control\LSA" C:\Backups\LSA_backup.reg

# Enable VBS
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" -Name "EnableVirtualizationBasedSecurity" -Value 1 -Type DWord
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" -Name "RequirePlatformSecurityFeatures" -Value 3 -Type DWord

# Enable Credential Guard
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard" -Name "LsaCfgFlags" -Value 1 -Type DWord

# Note: Group Policy is the recommended method for enterprise deployment
# Computer Configuration\Windows Settings\Security Settings\Local Policies\Security Options
# "System Guard Secure Launch and SMM protection: Enabled"
```

---

## Windows Hello for Business

Windows Hello for Business replaces passwords with strong two-factor authentication using biometrics (fingerprint, face recognition) or a PIN backed by a TPM-protected asymmetric key pair.

```powershell
# Check Windows Hello status
Get-WmiObject -Namespace root\StandardCimv2 -Class MSFT_NetFWRule | 
    Where-Object {$_.DisplayName -like "*Hello*"}

# Verify Hello for Business is configured (requires Azure AD or AD domain)
dsregcmd /status | Select-String "Hello"

# Check PIN/biometric registration
Get-WmiObject -Namespace root\cimv2\mdm\dmmap -Class MDM_Policy_Result01_Authentication02
```

---

## Microsoft Defender Exploit Protection

```powershell
# View current exploit protection settings
Get-ProcessMitigation -System

# Enable DEP for all processes
Set-ProcessMitigation -System -Enable DEP

# Enable ASLR (high entropy, bottom-up)
Set-ProcessMitigation -System -Enable HighEntropyASLR, BottomUp

# Enable CFG (Control Flow Guard)
Set-ProcessMitigation -System -Enable CFG

# Application-specific mitigations
Set-ProcessMitigation -Name "winword.exe" -Enable DEP, SEHOP, MandatoryASLR, ASLR

# Export exploit protection configuration
Get-ProcessMitigation -RegistryConfigFilePath C:\Backups\ExploitProtection.xml

# Import exploit protection configuration (deploy via Group Policy)
Set-ProcessMitigation -PolicyFilePath C:\ExploitProtection.xml
```

> **Next:** Proceed to [Folder 16: Group Policy Masterclass](../16_Group_Policy_GPMC/README.md) for enterprise policy management.
