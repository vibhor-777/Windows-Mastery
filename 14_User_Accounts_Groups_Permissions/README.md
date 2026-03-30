# Folder 14: User Accounts, Groups, and Permissions

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak) and Identity Management specialists.
- **What:** Architecture of the Security Accounts Manager (SAM), user/group management, and NTFS permission management via `icacls` and `takeown`.
- **Where:** Local Security Policy, SAM, Active Directory, PowerShell, CMD.
- **Why:** Identity and access management (Parichay aur Pravesh Prabandhan) is the foundation of Windows security. Misconfigured permissions are among the most common causes of security breaches and data loss.

---

## The Security Accounts Manager (SAM)

The Security Accounts Manager (SAM - Suraksha Khata Prabandhan) stores local user account credentials. It is a registry hive located at `%SystemRoot%\System32\config\SAM`.

### SAM Security Features

- **Locked during operation:** The SAM file cannot be read while Windows is running (protected by a kernel lock).
- **Password storage:** Modern Windows uses NTLMv2 hashes. The original LM (LAN Manager) hash is disabled by default since Windows Vista.
- **SAM encryption:** The SAM is encrypted with a SYSKEY derived from the SYSTEM hive, providing additional protection against offline attacks.
- **Credential Guard:** In VBS-enabled environments, even kernel-level access cannot retrieve credential hashes from the in-memory credential store (they are in VTL1).

### User Account Architecture

Each local user account has:
- **SAM Account Name:** The username (e.g., `JohnDoe`).
- **Security Identifier (SID - Suraksha Pahchankarta):** A unique identifier in the format `S-1-5-21-<domain>-<RID>`. The RID (Relative Identifier) is unique per account.
- **NTLM Hash:** One-way hash of the password used for authentication.
- **User Account Control (UAC) flags:** Account status, password policy flags.
- **Profile path:** Location of the user's profile directory.

**Well-Known SIDs:**

| SID | Account |
|---|---|
| S-1-1-0 | Everyone |
| S-1-2-0 | Local |
| S-1-5-18 | SYSTEM (LocalSystem) |
| S-1-5-19 | LOCAL SERVICE |
| S-1-5-20 | NETWORK SERVICE |
| S-1-5-32-544 | Administrators (local group) |
| S-1-5-32-545 | Users (local group) |
| S-1-5-32-546 | Guests (local group) |
| S-1-5-32-551 | Backup Operators |
| S-1-5-32-580 | Remote Management Users |

---

## Built-in User Accounts

| Account | Description | Default Status |
|---|---|---|
| **Administrator** | Full local administrative access | Disabled by default on domain-joined machines |
| **Guest** | Minimal access | Disabled by default |
| **DefaultAccount** | System-managed account | Disabled |
| **WDAGUtilityAccount** | Windows Defender Application Guard | Disabled (enabled when WDAG is used) |

**Security Warning (Suraksha Chetavni):** The built-in Administrator account should be renamed and kept disabled. Create a separate named administrator account for administrative tasks and use a standard account for daily operations (Principle of Least Privilege).

---

## User Management Commands

### Local User Management

```powershell
# List all local users
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordExpires, Description

# Create a new local user
$password = ConvertTo-SecureString "ComplexP@ssw0rd!" -AsPlainText -Force
New-LocalUser -Name "ServiceAccount" -Password $password -Description "Service account for AppX" -PasswordNeverExpires $false

# Disable a user account
Disable-LocalUser -Name "OldEmployee"

# Enable a user account
Enable-LocalUser -Name "NewEmployee"

# Set account expiration
Set-LocalUser -Name "TempUser" -AccountExpires (Get-Date).AddDays(90)

# Delete a user
Remove-LocalUser -Name "OldEmployee"

# Force password change at next logon
Set-LocalUser -Name "JohnDoe" -PasswordRequired $true
# Note: This doesn't force change at next logon via PS alone;
# use: net user JohnDoe /logonpasswordchg:yes
net user JohnDoe /logonpasswordchg:yes
```

### Group Management

```powershell
# List local groups
Get-LocalGroup | Select-Object Name, Description

# Create a group
New-LocalGroup -Name "ITDepartment" -Description "IT Department Users"

# Add a member to a group
Add-LocalGroupMember -Group "Administrators" -Member "DOMAIN\JohnDoe"
Add-LocalGroupMember -Group "Administrators" -Member "S-1-5-21-..."  # Using SID

# Remove a member from a group
Remove-LocalGroupMember -Group "Administrators" -Member "DOMAIN\JohnDoe"

# View group members
Get-LocalGroupMember -Group "Administrators"

# List all groups a user is member of (via WMI)
$user = "JohnDoe"
$groups = ([ADSISEARCHER]"(&(ObjectCategory=User)(SamAccountName=$user))").FindOne().GetDirectoryEntry().memberOf
$groups | ForEach-Object { ($_ -split ",")[0] -replace "CN=","" }
```

---

## Technical Execution: Take Ownership of a Protected Folder

**Who:** Administrators.  
**What:** `takeown /f C:\Windows\System32 /r /d y` followed by `icacls C:\Windows\System32 /grant Administrators:F /T`  
**Where:** Elevated CMD.  
**Why:** To resolve stubborn access issues during critical system repairs. Used when TrustedInstaller or SYSTEM owns files that need to be modified.

```cmd
REM Step 1: Back up the current ACLs before any changes
icacls C:\Windows\System32 /save C:\Backups\System32_ACLs.txt /T

REM Step 2: Take ownership
takeown /f C:\Windows\System32 /r /d y

REM Step 3: Grant Administrators full control
icacls C:\Windows\System32 /grant Administrators:F /T

REM Step 4: Perform your repair operation...

REM Step 5: CRITICAL - Restore original ACLs after repair
icacls C:\ /restore C:\Backups\System32_ACLs.txt

REM Step 6: Restore TrustedInstaller ownership (system files should be owned by TI)
icacls C:\Windows\System32 /setowner "NT SERVICE\TrustedInstaller" /T
```

**Security Warning (Suraksha Chetavni):** Taking ownership of System32 significantly reduces system security. Always restore original permissions and ownership immediately after completing repairs. Never leave System32 writable by Administrators in production.

---

## User Account Control (UAC)

UAC (User Account Control - Upayogakarta Khata Niyantran) is Windows' mechanism for limiting administrative access to only when explicitly needed. Even administrators run with a standard token; a separate elevated token is created when elevation is approved.

### UAC Architecture

1. **Standard Token:** Created at logon for all users, including admins. Has standard user privileges.
2. **Elevated Token:** Created for admin users when UAC elevation is approved. Has full administrative privileges.
3. **UAC Prompt:** Dialog that appears when an action requires elevation. Types:
   - **Credential Prompt:** Standard users must provide admin credentials.
   - **Consent Prompt:** Administrators confirm they want to elevate (Secure Desktop).
   - **Auto-elevation:** Trusted system binaries can elevate without a prompt.

```powershell
# Check current UAC level
$uacKey = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System"
Get-ItemProperty -Path $uacKey | Select-Object ConsentPromptBehaviorAdmin, 
    ConsentPromptBehaviorUser, EnableLUA, PromptOnSecureDesktop

# UAC ConsentPromptBehaviorAdmin values:
# 0 = Elevate without prompting (NOT RECOMMENDED)
# 1 = Prompt for credentials on secure desktop
# 2 = Prompt for consent on secure desktop
# 5 = Prompt for consent for non-Windows binaries (Default)

# Enable most secure UAC (always prompt on secure desktop)
reg export "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" C:\Backups\UAC_backup.reg
Set-ItemProperty -Path $uacKey -Name "ConsentPromptBehaviorAdmin" -Value 2 -Type DWord
Set-ItemProperty -Path $uacKey -Name "PromptOnSecureDesktop" -Value 1 -Type DWord

# Check if current process is elevated
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]"Administrator")
```

---

## NTFS Permissions: Advanced Management

```powershell
# View effective permissions for a user on a path
$path = "C:\SensitiveData"
$user = "DOMAIN\JohnDoe"
$acl = Get-Acl $path
$acl.Access | Where-Object {$_.IdentityReference -like "*$user*"}

# Create a new folder with specific permissions
$path = "C:\AppData\SecureApp"
New-Item -ItemType Directory -Path $path -Force

# Set ACL from scratch (remove inheritance, set explicit permissions)
$acl = New-Object System.Security.AccessControl.DirectorySecurity

# Disable inheritance
$acl.SetAccessRuleProtection($true, $false)

# Add SYSTEM full control
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "SYSTEM", "FullControl", 
    "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.AddAccessRule($rule)

# Add Administrators full control  
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "Administrators", "FullControl",
    "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.AddAccessRule($rule)

# Add specific user read-only
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "DOMAIN\AppUser", "ReadAndExecute",
    "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.AddAccessRule($rule)

Set-Acl -Path $path -AclObject $acl
```

---

## Password Policy Management

```powershell
# View local password policy
net accounts

# Configure password policy via secedit
secedit /export /cfg C:\Temp\secpolicy.cfg
# Edit the cfg file, then:
secedit /configure /db C:\Temp\secpol.sdb /cfg C:\Temp\secpolicy.cfg

# Key password policy settings in secpolicy.cfg:
# MinimumPasswordLength = 14
# PasswordComplexity = 1
# MaximumPasswordAge = 90
# MinimumPasswordAge = 1
# PasswordHistorySize = 24
# LockoutBadCount = 5
# ResetLockoutCount = 15
# LockoutDuration = 15
```

> **Next:** Proceed to [Folder 15: Security, Defender, and Hardware-Based Protection](../15_Security_Defender/README.md) for the core security deep dive.
