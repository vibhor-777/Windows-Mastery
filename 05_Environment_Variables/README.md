# Folder 05: Environment Variables and System PATH

## 4Ws Structure

- **Who:** All users — Developers (Vikshasak), System Administrators (Pranali Prashasak).
- **What:** Complete guide to Environment Variables (Parivesh Char) — system-wide and user-specific variables that configure the OS and application behavior.
- **Where:** System Properties, CMD, PowerShell, Registry (`HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment`).
- **Why:** Correct PATH and environment variable configuration is essential for application functionality, security, and scripting reliability.

---

## What Are Environment Variables?

Environment Variables (Parivesh Char) are named values that define the operating environment for processes. When a process is created, it inherits a copy of the environment block from its parent process. Changes to environment variables affect only the current process and its children — unless the system environment is modified through the API or registry.

### Two Scopes of Environment Variables

| Scope | Storage Location | Affects | Requires Admin |
|---|---|---|---|
| **System (Machine)** | `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment` | All users and services | Yes |
| **User** | `HKCU\Environment` | Current user only | No |

---

## Essential Windows Environment Variables

| Variable | Description | Default Value Example |
|---|---|---|
| `%SystemRoot%` | Windows directory path | `C:\Windows` |
| `%SystemDrive%` | Drive containing Windows | `C:` |
| `%COMSPEC%` | Path to command shell | `C:\Windows\System32\cmd.exe` |
| `%PATH%` | Executable search path | List of semicolon-separated directories |
| `%PATHEXT%` | Extensions treated as executable | `.COM;.EXE;.BAT;.CMD;.VBS;.JS;.PS1` |
| `%TEMP%` / `%TMP%` | Temporary file directory | `C:\Users\Username\AppData\Local\Temp` |
| `%USERPROFILE%` | Current user's profile path | `C:\Users\Username` |
| `%APPDATA%` | User's roaming AppData | `C:\Users\Username\AppData\Roaming` |
| `%LOCALAPPDATA%` | User's local AppData | `C:\Users\Username\AppData\Local` |
| `%ProgramFiles%` | 64-bit program install path | `C:\Program Files` |
| `%ProgramFiles(x86)%` | 32-bit program install path | `C:\Program Files (x86)` |
| `%ProgramData%` | Machine-wide app data | `C:\ProgramData` |
| `%WINDIR%` | Windows directory | `C:\Windows` |
| `%COMPUTERNAME%` | Machine hostname | `WORKSTATION01` |
| `%USERNAME%` | Current user name | `JohnDoe` |
| `%USERDOMAIN%` | Domain of current user | `CONTOSO` |
| `%USERDOMAIN_ROAMINGPROFILE%` | Domain for roaming profile | `CONTOSO` |
| `%LOGONSERVER%` | Domain controller used for logon | `\\DC01` |
| `%NUMBER_OF_PROCESSORS%` | Processor count | `8` |
| `%PROCESSOR_ARCHITECTURE%` | CPU architecture | `AMD64` |
| `%OS%` | Operating system identifier | `Windows_NT` |
| `%PUBLIC%` | Public user profile | `C:\Users\Public` |
| `%HOMEDRIVE%` | Drive of user home directory | `C:` |
| `%HOMEPATH%` | Path portion of home directory | `\Users\Username` |
| `%ALLUSERSPROFILE%` | All users profile path | `C:\ProgramData` |
| `%PSModulePath%` | PowerShell module search paths | Multiple paths |
| `%OneDrive%` | OneDrive sync folder | `C:\Users\Username\OneDrive` |

---

## Managing Environment Variables

### Via System Properties GUI

```
Win + R → sysdm.cpl → Advanced → Environment Variables
```

### Via CMD

```cmd
REM Display all environment variables
set

REM Display a specific variable
set SystemRoot

REM Set a session-level variable (lost when CMD closes)
set MY_VAR=HelloWorld

REM Use the variable
echo %MY_VAR%

REM Set a persistent system variable (requires elevation)
setx MY_SYSTEM_VAR "SystemValue" /M

REM Set a persistent user variable
setx MY_USER_VAR "UserValue"

REM Add a directory to PATH (system-wide, persistent, requires elevation)
setx PATH "%PATH%;C:\NewTool\bin" /M

REM Remove a session variable
set MY_VAR=
```

**Security Warning (Suraksha Chetavni):** `setx` truncates values at 1024 characters. Do not use `setx PATH "%PATH%;new_path" /M` if your PATH is already long — it will silently truncate it, breaking many system tools. Instead, edit PATH through System Properties GUI or PowerShell.

### Via PowerShell

```powershell
# View all environment variables
Get-ChildItem Env:

# Get a specific variable
$env:APPDATA
$env:PATH -split ";"  # Split PATH into separate lines for readability

# Set a session-level variable
$env:MY_VAR = "HelloWorld"

# Set persistent system environment variable
[System.Environment]::SetEnvironmentVariable("MY_SYSTEM_VAR", "SystemValue", "Machine")

# Set persistent user environment variable
[System.Environment]::SetEnvironmentVariable("MY_USER_VAR", "UserValue", "User")

# Add directory to system PATH persistently (safe method - no truncation risk)
$currentPath = [System.Environment]::GetEnvironmentVariable("PATH", "Machine")
$newPath = $currentPath + ";C:\NewTool\bin"
[System.Environment]::SetEnvironmentVariable("PATH", $newPath, "Machine")

# Remove an environment variable
[System.Environment]::SetEnvironmentVariable("MY_SYSTEM_VAR", $null, "Machine")

# Refresh environment variables in current session (after system changes)
$env:PATH = [System.Environment]::GetEnvironmentVariable("PATH", "Machine") + ";" + 
            [System.Environment]::GetEnvironmentVariable("PATH", "User")
```

---

## The PATH Variable: Security and Configuration

The `%PATH%` variable (Maarg Nirdeshak) is a semicolon-separated list of directories. When you type a command without a full path, Windows searches these directories in order for a matching executable.

### PATH Security Risks

1. **PATH Hijacking:** If a malicious directory appears before `C:\Windows\System32` in the PATH, a malicious executable named `cmd.exe` or `notepad.exe` in that directory will be executed instead of the legitimate one.

2. **Writeable PATH Directories:** If any directory in the system PATH is writable by non-administrators, an attacker can plant malicious executables there.

3. **DLL Search Order via PATH:** PATH also affects DLL search order when an application doesn't specify a full path for DLL loading.

```powershell
# Security audit: Check for writeable directories in system PATH
$systemPath = [System.Environment]::GetEnvironmentVariable("PATH", "Machine") -split ";"
foreach ($dir in $systemPath) {
    if (Test-Path $dir) {
        $acl = Get-Acl $dir
        $writeableByNonAdmin = $acl.Access | Where-Object {
            $_.FileSystemRights -match "Write|FullControl" -and
            $_.IdentityReference -notmatch "SYSTEM|Administrators|TrustedInstaller"
        }
        if ($writeableByNonAdmin) {
            Write-Warning "WRITEABLE PATH DIRECTORY: $dir"
            $writeableByNonAdmin | Select-Object IdentityReference, FileSystemRights
        }
    }
}

# View PATH entries one per line
($env:PATH).Split(";") | ForEach-Object { Write-Host $_ }
```

---

## Registry Storage of Environment Variables

```powershell
# View system environment variables from registry
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment"

# View user environment variables from registry
Get-ItemProperty -Path "HKCU:\Environment"

# Backup before modification
reg export "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" C:\Backups\SystemEnv_backup.reg
reg export "HKCU\Environment" C:\Backups\UserEnv_backup.reg
```

---

## PATHEXT: Controlling Executable Extensions

The `%PATHEXT%` variable defines which file extensions Windows treats as executable when searching the PATH. Default value: `.COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC`

**Security Warning (Suraksha Chetavni):** The inclusion of `.VBS`, `.JS`, `.WSF`, and `.WSH` in PATHEXT means that script files in any PATH directory are executable. Consider removing these extensions from PATHEXT in high-security environments and blocking execution via AppLocker or Windows Defender Application Control (WDAC).

```powershell
# View PATHEXT
$env:PATHEXT

# In a high-security environment, restrict PATHEXT (user-level, current session)
$env:PATHEXT = ".COM;.EXE;.BAT;.CMD"
```

---

## Practical: Diagnosing Missing Commands

When a command is "not recognized," it means the executable is not found in any PATH directory:

```powershell
# Find where a command is resolved from
Get-Command cmd.exe
Get-Command python.exe -ErrorAction SilentlyContinue

# Find all instances of an executable in PATH
where.exe python.exe

# Verify a tool is in PATH
$toolPath = Get-Command git -ErrorAction SilentlyContinue
if ($toolPath) {
    Write-Host "Git found at: $($toolPath.Source)"
} else {
    Write-Warning "Git not found in PATH. Install Git or add it to PATH."
}
```

> **Next:** Proceed to [Folder 06: CMD Masterclass](../06_CMD_Masterclass/README.md) for the comprehensive command reference.
