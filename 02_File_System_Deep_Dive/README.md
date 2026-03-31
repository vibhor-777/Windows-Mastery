# Folder 02: File System Deep Dive (NTFS, exFAT, FAT32)

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak) and Storage Engineers.
- **What:** Exhaustive documentation of NTFS (New Technology File System - Naveen Takniki Sanchika Pranali), exFAT, and FAT32 file systems, including Access Control Lists (ACLs - Pravesh Niyantran Suchi).
- **Where:** All Windows storage environments — internal drives, external storage, network shares.
- **Why:** File system knowledge is foundational to data security, performance tuning, and disaster recovery operations.

---

## NTFS: The Foundation of Windows Storage

NTFS (Naveen Takniki Sanchika Pranali) has been the primary Windows file system since Windows NT 3.1. It provides features that FAT32 fundamentally cannot: security, reliability, and efficiency at scale.

### NTFS Core Features

| Feature | Description | Security Benefit |
|---|---|---|
| Access Control Lists (ACLs) | Per-file/folder permissions | Granular access control |
| Journaling (Lekhankaran) | Transaction log for metadata changes | Recovery from unexpected shutdown |
| Encryption (EFS) | File-level encryption using user certificates | Data protection at rest |
| Compression | Transparent file compression | Storage efficiency |
| Hard Links | Multiple directory entries for one file | Efficient storage of duplicates |
| Symbolic Links | Redirect file/folder access | Flexible path management |
| Quotas | Limit disk space per user | Resource governance |
| Sparse Files | Efficient large file storage | Storage optimization |
| Alternate Data Streams (ADS) | Hidden metadata streams | Used by Zone.Identifier for download tracking |
| USN Journal | Change tracking for all file operations | Forensic audit trail |

---

## The Master File Table (MFT)

The Master File Table (MFT - Mukhya Sanchika Talika) is the heart of NTFS. It is a special file (`$MFT`) that contains at least one entry for every file and folder on the volume.

Each MFT record is 1KB by default and contains:
- **Standard Information:** Timestamps (Created, Modified, Accessed, MFT Changed), file attributes, security ID.
- **File Name:** One or more name attributes (supporting different name formats).
- **Data:** For small files (<750 bytes), the file data is stored directly in the MFT record (resident data). For larger files, the MFT record contains pointers (run list) to the data clusters on disk (non-resident data).
- **Security Descriptor:** Reference to the file's security descriptor (DACL, SACL, Owner, Group).

```powershell
# View MFT information using fsutil
fsutil fsinfo ntfsinfo C:

# Check MFT size and fragmentation
fsutil volume diskfree C:

# View file record for a specific file
fsutil file queryEA C:\Windows\System32\ntoskrnl.exe
```

---

## Access Control Lists (ACLs): Security at the Filesystem Level

ACLs (Pravesh Niyantran Suchi) are the primary security mechanism in NTFS. Every file and folder has a Security Descriptor containing:

1. **Owner (Swami):** The user account that owns the object (typically has implicit full control).
2. **DACL (Discretionary ACL - Vivekadhin Pravesh Niyantran Suchi):** A list of Access Control Entries (ACEs) specifying who can do what.
3. **SACL (System ACL - Pranali Pravesh Niyantran Suchi):** Specifies which access attempts should generate audit log entries.
4. **Group:** The primary group (used for POSIX compatibility in WSL).

### DACL Access Control Entries (ACEs)

Each ACE contains:
- **SID (Security Identifier - Suraksha Pahchankarta):** Identifies the user or group.
- **Access Mask:** A 32-bit value specifying the access rights (Read, Write, Execute, Delete, etc.).
- **ACE Type:** Allow or Deny.
- **Inheritance Flags:** Whether child objects inherit this ACE.

**Security Warning (Suraksha Chetavni):** Deny ACEs take precedence over Allow ACEs. Avoid using Deny ACEs when possible as they can create complex troubleshooting scenarios. Prefer restricting access by removing Allow ACEs.

### icacls: The Primary ACL Management Tool

```cmd
REM View permissions on a file or folder
icacls C:\SensitiveData

REM Grant Administrators full control (recursive)
icacls C:\SensitiveData /grant Administrators:(OI)(CI)F /T

REM Remove a specific user's permissions
icacls C:\SensitiveData /remove "DOMAIN\JohnDoe" /T

REM Reset permissions to inherit from parent
icacls C:\SensitiveData /reset /T

REM Save ACLs to a file (for backup)
icacls C:\SensitiveData /save C:\Backups\SensitiveData_ACLs.txt /T

REM Restore ACLs from saved file
icacls C:\ /restore C:\Backups\SensitiveData_ACLs.txt
```

**Permission Flags:**
| Flag | Meaning |
|---|---|
| F | Full Control |
| M | Modify |
| RX | Read and Execute |
| R | Read |
| W | Write |
| D | Delete |
| (OI) | Object Inherit (files inherit this ACE) |
| (CI) | Container Inherit (folders inherit this ACE) |
| (NP) | No Propagate (do not inherit further down the tree) |

---

## Technical Execution: Filesystem Integrity Check

**Who:** System Administrators.  
**What:** `chkdsk C: /R`  
**Where:** Elevated CMD (Utchatam CMD).  
**Why:** To scan and repair bad sectors or invalid filesystem metadata, preventing data corruption and filesystem inconsistencies.

```cmd
REM Schedule chkdsk on next reboot (for system drive)
chkdsk C: /F /R /X

REM Run chkdsk on an unmounted volume immediately
chkdsk D: /F /R /X

REM View chkdsk results from Event Log (after reboot)
wevtutil qe Application /q:"*[System[Provider[@Name='Microsoft-Windows-Chkdsk']]]" /f:text /rd:true /c:10
```

**chkdsk Flags:**
| Flag | Description |
|---|---|
| /F | Fix errors on the disk |
| /R | Locate bad sectors and recover readable information (implies /F) |
| /X | Force the volume to dismount first if necessary |
| /B | Re-evaluate bad clusters on the volume (implies /R) |
| /C | Skip checking of cycles within the folder structure |

---

## Alternate Data Streams (ADS): Security Implications

NTFS supports multiple data streams per file. The primary stream (the one you see in Explorer) is `FileName:$DATA`. Additional streams can be attached:

```cmd
REM Create a hidden ADS
echo "Secret data" > C:\public.txt:hidden_stream

REM View ADS on a file
dir /R C:\public.txt

REM Read ADS content
more < C:\public.txt:hidden_stream

REM Find all ADS on a volume (using Sysinternals Streams)
streams.exe -s C:\

REM Delete a specific ADS
more < nul > C:\public.txt:hidden_stream
```

**Security Warning (Suraksha Chetavni):** Malware commonly uses ADS to hide payloads. The `Zone.Identifier` stream (added by Windows when downloading files from the internet) is a legitimate ADS used to mark untrusted files for SmartScreen. Always scan for unexpected ADS on sensitive directories.

---

## EFS: Encrypting File System

EFS (Encrypting File System - Ambigukarana Sanchika Pranali) provides transparent file encryption at the NTFS layer. Files are encrypted with a symmetric key (FEK - File Encryption Key), which is then protected by the user's RSA public key.

```cmd
REM Encrypt a directory (and all files within)
cipher /e /s:C:\ConfidentialDocs

REM Decrypt a directory
cipher /d /s:C:\ConfidentialDocs

REM View encryption status
cipher /u /n

REM Back up EFS certificate (CRITICAL - do this before using EFS)
certmgr.msc
REM Navigate to: Personal > Certificates > Right-click EFS cert > All Tasks > Export
```

**Security Warning (Suraksha Chetavni):** EFS-encrypted files are accessible only to the encrypting user's account. If the account is deleted without backing up the EFS certificate, the data is permanently inaccessible. Always configure a Data Recovery Agent (DRA) in enterprise environments.

---

## File System Comparison

| Feature | NTFS | exFAT | FAT32 |
|---|---|---|---|
| Max Volume Size | 256 TB (theoretical) | 128 PB | 32 GB (Windows limit) |
| Max File Size | 16 EB (theoretical) | 128 PB | 4 GB |
| ACL Security | Yes | No | No |
| Journaling | Yes | No | No |
| Compression | Yes | No | No |
| Encryption (EFS) | Yes | No | No |
| Snapshots (VSS) | Yes | No | No |
| Use Case | Windows system/data drives | USB drives, SD cards | Legacy compatibility |

---

## Volume Shadow Copy Service (VSS)

VSS (Volume Shadow Copy Service - Chhaya Naqal Sewa) provides point-in-time snapshots of NTFS volumes, enabling backup and recovery operations without taking volumes offline.

```powershell
# List existing shadow copies
Get-WmiObject Win32_ShadowCopy | Select-Object ID, VolumeName, InstallDate, Caption

# Create a new shadow copy
$shadow = (Get-WmiObject -List Win32_ShadowCopy).Create("C:\", "ClientAccessible")

# Mount a shadow copy to access previous versions
cmd /c mklink /d C:\ShadowMount \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\

# Delete old shadow copies
Get-WmiObject Win32_ShadowCopy | Where-Object {$_.VolumeName -eq "\\?\Volume{GUID}\"} | Remove-WmiObject
```

---

## USN Journal: Filesystem Change Tracking

The Update Sequence Number (USN) Journal tracks all changes to files and directories on an NTFS volume, providing a forensic audit trail.

```cmd
REM Query the USN journal
fsutil usn queryjournal C:

REM Read USN journal entries
fsutil usn readjournal C: csv > C:\usn_journal.csv

REM Create USN journal (if not exists)
fsutil usn createjournal m=1000 a=100 C:
```

> **Next:** Proceed to [Folder 03: System32 Deep Dive](../03_System32_Deep_Dive/README.md) for critical binary analysis.
