# Folder 11: Boot Process and UEFI/BIOS

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak), Security Engineers, and Recovery Specialists.
- **What:** Complete documentation of the Windows 11 boot sequence from UEFI firmware to the loaded desktop, including Secure Boot and Measured Boot.
- **Where:** Pre-OS firmware environment (UEFI), Windows Boot Manager, and Windows Boot Configuration Database (BCD).
- **Why:** Understanding the boot process is essential for troubleshooting boot failures, hardening against bootkits, and implementing measured boot attestation for Zero Trust architectures.

---

## BIOS vs. UEFI: The Fundamental Shift

| Feature | Legacy BIOS | UEFI |
|---|---|---|
| Interface | 16-bit real mode | 64-bit protected mode |
| Max disk size | 2 TB (MBR) | 9.4 ZB (GPT) |
| Boot partition | MBR | GPT with ESP |
| Secure Boot | Not supported | Supported |
| Network boot | Limited | Built-in PXE + HTTPS Boot |
| Driver model | Proprietary ROM | UEFI drivers (EFI applications) |
| Pre-boot environment | None | UEFI Shell, DXE, PEI |
| Boot time | Slower | Faster (parallel initialization) |

---

## The Complete Windows 11 Boot Sequence

### Phase 1: UEFI Firmware Initialization

1. **SEC (Security) Phase:** First code to execute. Validates the UEFI firmware itself against a trusted signature stored in ROM. Establishes a temporary stack in CPU cache (Cache as RAM — CAR).

2. **PEI (Pre-EFI Initialization) Phase:** Initializes minimum hardware (CPU, chipset, RAM). Loads PEI modules (PEIMs) from the firmware volume.

3. **DXE (Driver Execution Environment) Phase:** Loads UEFI drivers for storage, network, and display. Builds the system table with all protocol interfaces.

4. **BDS (Boot Device Selection) Phase:** Evaluates the boot order from UEFI NVRAM. Loads the EFI boot application — for Windows this is `\EFI\Microsoft\Boot\bootmgfw.efi`.

5. **Secure Boot Verification:** Before loading any EFI application, UEFI Secure Boot (Surakshit Boot) checks the application's signature against keys stored in:
   - **db (Signature Database):** Authorized keys/certificates.
   - **dbx (Forbidden Signature Database):** Revoked keys/certificates.
   - **KEK (Key Exchange Key):** Used to update db and dbx.
   - **PK (Platform Key):** The root of trust for the firmware.

---

### Phase 2: Windows Boot Manager (bootmgfw.efi)

The Windows Boot Manager (`bootmgfw.efi`) reads the Boot Configuration Database (BCD) and presents the boot menu if multiple OS entries exist.

```cmd
REM View BCD contents
bcdedit /enum all

REM View default boot entry
bcdedit /enum {default}

REM Key BCD settings
bcdedit /set {default} description "Windows 11 Pro"
bcdedit /set {default} nx AlwaysOn    REM Enable DEP
bcdedit /set {default} pae ForceEnable REM Physical Address Extension
bcdedit /timeout 5                     REM Boot menu timeout seconds
```

**BCD Store Location:**
- UEFI systems: `\EFI\Microsoft\Boot\BCD` on the EFI System Partition (ESP).
- Legacy BIOS: `C:\Boot\BCD`.

---

### Phase 3: Windows OS Loader (winload.efi)

The OS Loader (`winload.efi`) loads the NT kernel and its dependencies:

1. Reads hibernation file (`hiberfil.sys`) if resuming from hibernation.
2. Loads `ntoskrnl.exe` (NT Kernel + Executive).
3. Loads `hal.dll` (Hardware Abstraction Layer).
4. Loads boot-start drivers (from `HKLM\SYSTEM\CurrentControlSet\Services` with Start=0).
5. Transfers control to the NT Kernel.

**Measured Boot (Mapi gayi boot):** During this phase, the OS Loader measures (hashes) each loaded component and extends the measurements into TPM PCRs (Platform Configuration Registers):
- PCR 0: Core UEFI firmware
- PCR 2: Option ROMs
- PCR 4: Boot Manager
- PCR 5: Boot Configuration Data (BCD)
- PCR 7: Secure Boot state
- PCR 11: BitLocker

These PCR values form the Static Root of Trust for Measurement (SRTM), which can be used to attest device health to remote services.

---

### Phase 4: NT Kernel Initialization

1. **Phase 0 (Interrupts Disabled):** Kernel initializes critical data structures. Single processor operation.
2. **Phase 1 (Interrupts Enabled):** All processors initialize. Kernel opens the registry, starts the session manager (`smss.exe`).

---

### Phase 5: Session Manager (smss.exe)

The first user-mode process. Responsibilities:
- Creates the paging file (`pagefile.sys`).
- Creates Windows subsystems (Win32).
- Initializes the registry (loads hives).
- Creates the Winlogon session.

---

### Phase 6: Windows Initialization (wininit.exe) and Winlogon

`wininit.exe` starts key components:
- `services.exe` — Service Control Manager.
- `lsass.exe` — Local Security Authority Subsystem.
- `lsm.exe` — Local Session Manager.

`winlogon.exe` presents the logon screen (Ctrl+Alt+Del → Logon UI via LogonUI.exe), authenticates credentials through LSASS, and creates the user session.

---

### Phase 7: User Session Initialization

1. User profile loaded (NTUSER.DAT hive mounted).
2. `userinit.exe` launches (runs logon scripts, then exits).
3. `explorer.exe` launched (desktop, taskbar).
4. Startup programs from Run/RunOnce registry keys launch.
5. Startup folder programs launch.

---

## Secure Boot: Implementation Details

Secure Boot (Surakshit Boot) prevents unauthorized code from running before the OS loads. It is enforced by UEFI firmware.

```powershell
# Check Secure Boot status
Confirm-SecureBootUEFI
# Returns: True (enabled), False (disabled), or throws if UEFI not supported

# Alternative check
(Get-CimInstance -Namespace root/microsoft/windows/deviceguard -ClassName 
    Win32_DeviceGuard).SecureBootState
# 0=Off, 1=On

# View Secure Boot policy
Get-SecureBootPolicy | Format-List
```

**Secure Boot Key Hierarchy:**
```
PK (Platform Key) — set by OEM/Enterprise
    └── KEK (Key Exchange Key) — Microsoft + OEM
            ├── db  (Signature DB) — authorized bootloaders/drivers
            └── dbx (Forbidden DB) — revoked keys
```

---

## BitLocker and the Boot Sequence

BitLocker (Suraksha Tala) uses the TPM to protect the Volume Master Key (VMK). During boot:

1. BitLocker verifies that the boot environment matches the sealed PCR values.
2. If PCR values match (Secure Boot passed, boot components unchanged), TPM releases the VMK.
3. VMK decrypts the Full Volume Encryption Key (FVEK).
4. FVEK decrypts the drive.

If PCR values don't match (indicating tampering), BitLocker enters recovery mode, requiring the 48-digit recovery key.

```powershell
# Check BitLocker status
Get-BitLockerVolume

# Enable BitLocker with TPM and recovery password
Enable-BitLocker -MountPoint "C:" -TpmProtector
Add-BitLockerKeyProtector -MountPoint "C:" -RecoveryPasswordProtector

# Backup recovery key to AD (enterprise)
Backup-BitLockerKeyProtector -MountPoint "C:" -KeyProtectorId (Get-BitLockerVolume C:).KeyProtector[1].KeyProtectorId

# Save recovery key to file
(Get-BitLockerVolume C:).KeyProtector | Where-Object {$_.KeyProtectorType -eq "RecoveryPassword"} |
    Select-Object -ExpandProperty RecoveryPassword | 
    Out-File "C:\SecureLocation\BitLockerRecovery.txt"
```

---

## Boot Troubleshooting

### Common Boot Failures

| Error | Cause | Solution |
|---|---|---|
| `INACCESSIBLE_BOOT_DEVICE` (0x0000007B) | Storage driver missing or corrupt NTFS | Boot WinRE, use `bootrec`, check storage drivers |
| `BOOTMGR is missing` | BCD or boot partition damaged | `bootrec /fixmbr` + `bootrec /rebuildbcd` |
| `Winload.efi not found` | EFI partition damaged | Repair EFI partition from WinRE |
| `Secure Boot violation` | Unsigned bootloader | Check Secure Boot keys, update keys |
| Endless reboot loop | Driver crash during boot | Boot to Safe Mode, remove problematic driver |

```cmd
REM Boot Recovery Commands (from WinRE CMD)
bootrec /fixmbr          REM Repair MBR
bootrec /fixboot         REM Repair boot sector
bootrec /rebuildbcd      REM Rebuild BCD store
bootrec /scanos          REM Scan for Windows installations

REM If BCD is corrupted, rebuild manually
bcdboot C:\Windows /s C: /f ALL   REM Repair BCD for both UEFI and BIOS

REM Access Windows Recovery Environment
shutdown /r /o /t 0      REM Restart to Recovery Environment

REM Repair UEFI boot files
diskpart
list vol
select vol X  (EFI volume)
assign letter=S
exit
bcdboot C:\Windows /s S: /f UEFI
```

> **Next:** Proceed to [Folder 12: Device Drivers](../12_Device_Drivers/README.md) for driver architecture and management.
