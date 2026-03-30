# Folder 16: Group Policy Masterclass (GPMC)

## 4Ws Structure

- **Who:** Domain Administrators (Domain Prashasak) and Security Engineers.
- **What:** Group Policy lifecycle management — creation, testing, deployment, WMI filtering, and disaster recovery.
- **Where:** Group Policy Management Console (GPMC), `gpupdate`, `gpresult`, PowerShell GroupPolicy module.
- **Why:** Group Policy is the primary enterprise mechanism for enforcing configuration consistency and security baselines across hundreds or thousands of Windows machines.

---

## Group Policy Architecture

Group Policy Objects (GPOs - Samuh Niti Vastu) are collections of configuration settings that are applied to users and computers based on their location in Active Directory. The lifecycle:

```
Test Environment → Stage → Pilot → Production → Validate → Monitor
```

### GPO Processing Order (LSDOU)

GPOs are applied in a specific order; later GPOs override earlier ones:
1. **L**ocal Group Policy (local machine)
2. **S**ite-linked GPOs
3. **D**omain-linked GPOs
4. **O**rganizational Unit (OU) GPOs (from parent OU to child OU)

**Key Rule:** Last applied policy wins (for non-conflicting settings). If "Block Inheritance" is set on an OU, site and domain policies are not applied. "Enforced" (No Override) GPOs bypass Block Inheritance.

### GPO Scope Components

| Component | Description |
|---|---|
| Computer Configuration | Applied during machine startup, refreshed every 90 min + 0-30 min random offset |
| User Configuration | Applied at user logon, refreshed every 90 min + 0-30 min random offset |
| Security Filtering | Applies GPO only to specific users/groups/computers |
| WMI Filtering | Applies GPO only if WMI query returns true |
| Delegation | Controls who can edit/link/read GPOs |

---

## GPO Versioning: The Version Number Myth

**Common Misconception:** Version numbers between the AD object and Sysvol don't need to match in Windows XP/Server 2003+.

**Reality:** The AD object stores the GPO version number. The Sysvol stores the actual GPO files. The version number in the AD object is incremented each time a setting is changed. The client checks if the AD version matches what it last applied (stored in registry under `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy`). If they match, the client skips re-applying the GPO (performance optimization).

**Content matters more than count:** The number of GPOs has minimal impact on performance compared to GPO content (e.g., logon scripts that run slowly).

---

## GPO Deployment Lifecycle

### Phase 1: Create and Test

```powershell
# Import GroupPolicy module (requires RSAT)
Import-Module GroupPolicy

# Create a new GPO
New-GPO -Name "Security_Hardening_v1" -Comment "Windows 11 hardening baseline"

# Link to test OU
New-GPLink -Name "Security_Hardening_v1" -Target "OU=TestComputers,DC=contoso,DC=com"

# Set security filter (apply only to test group)
Set-GPPermissions -Name "Security_Hardening_v1" -PermissionLevel GpoApply -TargetName "TestGroup" -TargetType Group

# Remove 'Authenticated Users' apply permission (required when using specific group filtering)
Set-GPPermissions -Name "Security_Hardening_v1" -PermissionLevel None -TargetName "Authenticated Users" -TargetType Group
```

### Phase 2: Configure Settings

```powershell
# Configure a registry-based setting
Set-GPRegistryValue -Name "Security_Hardening_v1" -Key "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -ValueName "ConsentPromptBehaviorAdmin" -Type DWord -Value 2

# Configure a security option
# Most security settings require direct GPO template modification or use of ADMX files
```

### Phase 3: Apply and Validate

```powershell
# Force GPO refresh on all computers in OU
$computers = Get-ADComputer -Filter * -SearchBase "OU=TestComputers,DC=contoso,DC=com"
$computers | ForEach-Object {
    Invoke-GPUpdate -Computer $_.Name -RandomDelayInMinutes 0 -Force
}

# Generate RSoP (Resultant Set of Policy) report
Get-GPResultantSetOfPolicy -ReportType Html -Path "C:\Reports\RSoP_$(Get-Date -Format yyyyMMdd).html"

# Check applied GPOs on a remote machine
Invoke-Command -ComputerName TargetPC -ScriptBlock { gpresult /r }

# Verify specific setting was applied
Invoke-Command -ComputerName TargetPC -ScriptBlock {
    Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" | 
    Select-Object ConsentPromptBehaviorAdmin
}
```

---

## WMI Filters: Conditional Policy Application

WMI Filters (Windows Management Instrumentation Chhanaka) allow GPOs to apply only when a WMI query returns true. This enables hardware-specific or OS-specific policy application.

```powershell
# Create WMI filter to check for NIC IPsec support
# Example: Apply IPsec policy only to machines with specific NIC
$wmiFilter = New-GPWmiFilter -Name "IPsec_Capable_NICs" -Namespace "root\cimv2" -Query "SELECT * FROM Win32_NetworkAdapter WHERE AdapterTypeId = 0 AND NetEnabled = True" -Description "Only PCs with enabled Ethernet adapters"

# Create WMI filter for Windows 11 machines only
New-GPWmiFilter -Name "Windows_11_Only" -Namespace "root\cimv2" -Query "SELECT * FROM Win32_OperatingSystem WHERE BuildNumber >= 22000"

# Create WMI filter for domain-joined machines with TPM 2.0
New-GPWmiFilter -Name "TPM2_Machines" -Namespace "root\cimv2" -Query "SELECT * FROM Win32_TPM WHERE SpecVersion LIKE '2%'"

# Link WMI filter to GPO
$gpo = Get-GPO -Name "Security_Hardening_v1"
$gpo.WmiFilter = $wmiFilter
```

---

## Critical GPO Security Settings

```powershell
# Configure password policy (domain-wide via Default Domain Policy)
$gpoName = "Default Domain Policy"
Set-GPRegistryValue -Name $gpoName -Key "HKLM\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters" -ValueName "MaximumPasswordAge" -Type DWord -Value 90

# Enable audit policies
# Create a dedicated audit GPO
$auditGPO = New-GPO -Name "Security_Audit_Policy"

# Configure audit settings (requires secedit template approach or LGPO tool)
# Key audit settings to enable:
$auditSettings = @"
[Unicode]
Unicode=yes
[Version]
signature="$CHICAGO$"
Revision=1
[Event Audit]
AuditSystemEvents = 3
AuditLogonEvents = 3
AuditObjectAccess = 3
AuditPrivilegeUse = 2
AuditPolicyChange = 3
AuditAccountManage = 3
AuditAccountLogon = 3
"@
$auditSettings | Out-File "C:\Temp\audit_template.inf" -Encoding Unicode
```

---

## GPO Backup and Disaster Recovery

**Security Warning (Suraksha Chetavni):** Always back up GPOs before making changes. A misconfigured GPO can lock users out of the domain.

```powershell
# Backup ALL GPOs
$backupPath = "C:\GPO_Backups\$(Get-Date -Format 'yyyyMMdd_HHmmss')"
New-Item -ItemType Directory -Path $backupPath

Backup-GPO -All -Path $backupPath
Write-Host "All GPOs backed up to: $backupPath"

# Backup a specific GPO
Backup-GPO -Name "Security_Hardening_v1" -Path $backupPath -Comment "Pre-change backup $(Get-Date)"

# List all backups in a location
Get-GPOBackup -All -Path "C:\GPO_Backups\"

# Restore a specific GPO from backup
Restore-GPO -Name "Security_Hardening_v1" -Path "C:\GPO_Backups\20241201_120000\"

# Restore ALL GPOs (disaster recovery)
Get-GPOBackup -All -Path "C:\GPO_Backups\" | ForEach-Object {
    Restore-GPO -BackupId $_.Id -Path "C:\GPO_Backups\" -WarningAction SilentlyContinue
}

# Import GPO to a different domain (migration)
Import-GPO -BackupGpoName "Security_Hardening_v1" -Path "C:\GPO_Backups\" -TargetName "Security_Hardening_v1" -MigrationTable "C:\migration.migtable"
```

---

## DCGPOFix: Last Resort Tool

**Warning:** DCGPOFix repairs the Default Domain Policy and Default Domain Controllers Policy. It should only be used as an absolute last resort — it unlinks existing policies and can cause significant disruption.

```cmd
REM DCGPOFix recreates the default GPOs
REM Only use if Default Domain Policy or Default Domain Controllers Policy is severely corrupted
REM Unlinks ALL linked policies first!
dcgpofix /target:Domain          REM Fix Default Domain Policy only
dcgpofix /target:DC              REM Fix Default Domain Controllers Policy only
dcgpofix /target:Both            REM Fix both (extreme caution)
```

---

## Useful Group Policy Reporting

```powershell
# Generate HTML report for all GPOs
Get-GPO -All | ForEach-Object {
    Get-GPOReport -Name $_.DisplayName -ReportType Html -Path "C:\GPO_Reports\$($_.DisplayName).html"
}

# View GPO link status
Get-GPInheritance -Target "DC=contoso,DC=com"

# Find orphaned GPOs (in AD but not linked)
$allGPOs = Get-GPO -All
$linkedGPOs = @()

# Check all OUs for links
Get-ADOrganizationalUnit -Filter * | ForEach-Object {
    $inheritance = Get-GPInheritance -Target $_.DistinguishedName
    $linkedGPOs += $inheritance.GpoLinks.DisplayName
}

$orphanedGPOs = $allGPOs | Where-Object {$_.DisplayName -notin $linkedGPOs}
Write-Host "Orphaned GPOs:"
$orphanedGPOs | Select-Object DisplayName, Id, CreationTime | Format-Table
```

> **Next:** Proceed to [Folder 17: System Optimization and Deployment](../17_System_Optimization_Deployment/README.md) for SOE and MDM configuration.
