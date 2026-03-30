# Folder 07: PowerShell Masterclass (Language Reference)

## 4Ws Structure

- **Who:** Developers (Vikshasak), System Administrators (Pranali Prashasak), and Security Engineers.
- **What:** Comprehensive PowerShell scripting language reference — operators, automatic variables, pipeline, remoting, and security.
- **Where:** PowerShell 5.1 (built-in on Windows 10/11) and PowerShell 7+ (`pwsh.exe`).
- **Why:** PowerShell is the definitive Windows automation and administration tool. Its object-oriented pipeline makes complex administration tasks simple, repeatable, and auditable.

---

## PowerShell Editions and Versions

| Edition | Executable | Platform | Notes |
|---|---|---|---|
| Windows PowerShell 5.1 | `powershell.exe` | Windows only | Built into Windows 10/11; uses .NET Framework |
| PowerShell 7+ (Core) | `pwsh.exe` | Cross-platform | Open source; uses .NET Core/.NET 5+ |

---

## The PowerShell Pipeline: Object Orientation

Unlike CMD which passes text between commands, PowerShell passes **objects**. This is the fundamental concept that makes PowerShell powerful.

```powershell
# CMD approach (text parsing):
tasklist | findstr "notepad"

# PowerShell approach (object pipeline):
Get-Process | Where-Object { $_.Name -eq "notepad" } | Select-Object Name, Id, CPU

# Objects retain their properties through the entire pipeline
Get-Service | Where-Object { $_.Status -eq "Running" } | 
    Sort-Object DisplayName | 
    Select-Object DisplayName, Status, StartType |
    Format-Table -AutoSize

# Export to CSV with full objects
Get-Process | Export-Csv -Path C:\processes.csv -NoTypeInformation
```

---

## Operators: Complete Reference

### Arithmetic Operators

| Operator | Description | Example |
|---|---|---|
| `+` | Addition / String concatenation | `5 + 3` = 8; `"Hello" + " World"` |
| `-` | Subtraction | `10 - 4` = 6 |
| `*` | Multiplication | `4 * 3` = 12 |
| `/` | Division | `10 / 3` = 3.3333... |
| `%` | Modulus (remainder) | `10 % 3` = 1 |

### Assignment Operators

| Operator | Description | Example |
|---|---|---|
| `=` | Assign | `$x = 5` |
| `+=` | Add and assign | `$x += 3` → `$x` becomes 8 |
| `-=` | Subtract and assign | `$x -= 2` |
| `*=` | Multiply and assign | `$x *= 4` |
| `/=` | Divide and assign | `$x /= 2` |
| `%=` | Modulus and assign | `$x %= 3` |
| `++` | Increment | `$x++` |
| `--` | Decrement | `$x--` |

### Comparison Operators

| Operator | Description | Example |
|---|---|---|
| `-eq` | Equal | `5 -eq 5` → True |
| `-ne` | Not equal | `5 -ne 4` → True |
| `-gt` | Greater than | `5 -gt 3` → True |
| `-ge` | Greater or equal | `5 -ge 5` → True |
| `-lt` | Less than | `3 -lt 5` → True |
| `-le` | Less or equal | `3 -le 5` → True |
| `-like` | Wildcard match | `"Hello" -like "He*"` → True |
| `-notlike` | Wildcard non-match | `"Hello" -notlike "He*"` → False |
| `-match` | Regex match | `"Hello123" -match "\d+"` → True |
| `-notmatch` | Regex non-match | `"Hello" -notmatch "^\d"` → True |
| `-contains` | Collection contains | `@(1,2,3) -contains 2` → True |
| `-notcontains` | Collection doesn't contain | `@(1,2,3) -notcontains 4` → True |
| `-in` | Value in collection | `2 -in @(1,2,3)` → True |
| `-notin` | Value not in collection | `4 -notin @(1,2,3)` → True |

### Logical Operators

| Operator | Description |
|---|---|
| `-and` | Logical AND |
| `-or` | Logical OR |
| `-not` / `!` | Logical NOT |
| `-xor` | Exclusive OR |

### Bitwise Operators

| Operator | Description | Example |
|---|---|---|
| `-band` | Bitwise AND | `0xFF -band 0x0F` = 15 |
| `-bor` | Bitwise OR | `0x10 -bor 0x01` = 17 |
| `-bxor` | Bitwise XOR | `0xFF -bxor 0x0F` = 240 |
| `-bnot` | Bitwise NOT | `-bnot 0` = -1 |
| `-shl` | Shift left | `1 -shl 4` = 16 |
| `-shr` | Shift right | `256 -shr 4` = 16 |

---

## Automatic Variables (Swachalit Char)

| Variable | Description | Common Use |
|---|---|---|
| `$_` / `$PSItem` | Current pipeline object | `ForEach-Object { $_.Name }` |
| `$?` | Success status of last command | Error checking |
| `$Error` | Array of recent errors | `$Error[0]` for last error |
| `$null` | Null value | Initialize variables; filter null |
| `$true` / `$false` | Boolean values | Conditions |
| `$args` | Array of script arguments | In functions without param block |
| `$MyInvocation` | Info about current command | Script path, name |
| `$PSScriptRoot` | Directory of current script | `$PSScriptRoot\config.json` |
| `$PSCommandPath` | Full path of current script | Logging |
| `$Home` | Current user's home directory | `C:\Users\Username` |
| `$Profile` | Path to PowerShell profile | Profile management |
| `$Host` | PowerShell host information | Version, UI |
| `$PSVersionTable` | PowerShell version information | Compatibility checks |
| `$PID` | Current process ID | Logging, IPC |
| `$PWD` | Current directory | Navigation |
| `$LastExitCode` | Exit code of last native command | `$LASTEXITCODE` |
| `$OFS` | Output Field Separator | Array-to-string conversion |
| `$ConfirmPreference` | Confirmation threshold | Automation |
| `$ErrorActionPreference` | Error action default | Set to "Stop" for strict scripts |
| `$VerbosePreference` | Verbose output control | Debugging |
| `$DebugPreference` | Debug output control | Debugging |
| `$ProgressPreference` | Progress bar control | `"SilentlyContinue"` for performance |
| `$MaximumHistoryCount` | Command history limit | Session |
| `$ExecutionContext` | Current execution context | Advanced scripting |
| `$input` | Pipeline input to function | `function f { $input | ... }` |
| `$Matches` | Regex match results | After `-match` operator |
| `$PROFILE` | PowerShell profile paths | Profile management |

---

## Control Flow

### Conditional Statements

```powershell
# if/elseif/else
$service = Get-Service -Name "Spooler"
if ($service.Status -eq "Running") {
    Write-Host "Print Spooler is running"
} elseif ($service.Status -eq "Stopped") {
    Write-Warning "Print Spooler is stopped"
} else {
    Write-Host "Print Spooler status: $($service.Status)"
}

# Switch statement (more powerful than if/elseif chains)
$errorCode = 5
switch ($errorCode) {
    0       { "Success" }
    1       { "General failure" }
    5       { "Access denied" }
    default { "Unknown error: $errorCode" }
}

# Switch with regex
$logLine = "[ERROR] Service crashed"
switch -regex ($logLine) {
    "ERROR"   { Write-Error $logLine }
    "WARNING" { Write-Warning $logLine }
    "INFO"    { Write-Verbose $logLine }
}
```

### Loops

```powershell
# for loop
for ($i = 0; $i -lt 10; $i++) { Write-Host $i }

# foreach loop
$services = Get-Service
foreach ($svc in $services) {
    if ($svc.Status -ne "Running") {
        Write-Host "Stopped: $($svc.DisplayName)"
    }
}

# ForEach-Object (pipeline)
Get-Process | ForEach-Object { "$($_.Name) uses $($_.WorkingSet64 / 1MB) MB" }

# while loop
$count = 0
while ($count -lt 5) { Write-Host $count; $count++ }

# do-while
do { $input = Read-Host "Enter 'quit' to exit" } while ($input -ne "quit")

# do-until
do { $result = Test-Connection google.com -Count 1 -Quiet } until ($result)
```

---

## Functions and Advanced Parameters

```powershell
function Get-SystemInfo {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$false)]
        [string]$ComputerName = $env:COMPUTERNAME,
        
        [Parameter(Mandatory=$false)]
        [ValidateSet("Basic","Full","Security")]
        [string]$Level = "Basic",
        
        [switch]$ExportCSV
    )
    
    begin {
        Write-Verbose "Starting system information collection for $ComputerName"
    }
    
    process {
        $os = Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName
        $cs = Get-CimInstance Win32_ComputerSystem -ComputerName $ComputerName
        
        $result = [PSCustomObject]@{
            ComputerName = $ComputerName
            OSVersion    = $os.Caption
            BuildNumber  = $os.BuildNumber
            Architecture = $os.OSArchitecture
            RAM_GB       = [math]::Round($cs.TotalPhysicalMemory / 1GB, 2)
            CPUCount     = $cs.NumberOfProcessors
        }
        
        if ($Level -eq "Full") {
            $disk = Get-PSDrive C
            $result | Add-Member -NotePropertyName "FreeSpace_GB" -NotePropertyValue ([math]::Round($disk.Free / 1GB, 2))
        }
        
        return $result
    }
    
    end {
        if ($ExportCSV) {
            $result | Export-Csv -Path "C:\SystemInfo.csv" -NoTypeInformation
        }
    }
}
```

---

## Error Handling

```powershell
# Try/Catch/Finally
try {
    $result = Get-Content "C:\nonexistent.txt" -ErrorAction Stop
}
catch [System.IO.FileNotFoundException] {
    Write-Error "File not found: $_"
}
catch {
    Write-Error "Unexpected error: $($_.Exception.Message)"
}
finally {
    Write-Verbose "Cleanup completed"
}

# -ErrorAction parameter
Get-Process -Name "FakeProcess" -ErrorAction SilentlyContinue
Get-Process -Name "FakeProcess" -ErrorAction Stop  # Converts to terminating error

# $ErrorActionPreference
$ErrorActionPreference = "Stop"  # All errors become terminating
```

---

## PowerShell Remoting (PSRemoting)

```powershell
# Enable PSRemoting (Elevated)
Enable-PSRemoting -Force

# Run command on remote computer
Invoke-Command -ComputerName Server01 -ScriptBlock { Get-Service Spooler }

# Create persistent remote session
$session = New-PSSession -ComputerName Server01 -Credential (Get-Credential)
Invoke-Command -Session $session -ScriptBlock { Get-Process }
Enter-PSSession $session   # Interactive remote session
Remove-PSSession $session

# Run script on multiple computers
$computers = @("Server01", "Server02", "Server03")
Invoke-Command -ComputerName $computers -FilePath C:\Scripts\audit.ps1 -ThrottleLimit 5
```

---

## PowerShell Security

### Execution Policy

```powershell
# View current execution policy
Get-ExecutionPolicy -List

# Set execution policy
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
# Policies: Restricted, AllSigned, RemoteSigned, Unrestricted, Bypass

# Bypass for a single script (does not change system policy)
powershell.exe -ExecutionPolicy Bypass -File C:\script.ps1

# Check if a script is signed
Get-AuthenticodeSignature -FilePath C:\script.ps1
```

### Script Block Logging (Suraksha Lekhan)

```powershell
# Enable Script Block Logging via registry (logs all executed PowerShell code)
reg export HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell C:\Backups\PS_Policy_backup.reg
$path = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-Item -Path $path -Force
Set-ItemProperty -Path $path -Name "EnableScriptBlockLogging" -Value 1

# View logged script blocks (Event ID 4104 in Microsoft-Windows-PowerShell/Operational)
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" |
    Where-Object { $_.Id -eq 4104 } |
    Select-Object TimeCreated, Message |
    Format-List
```

### Constrained Language Mode

```powershell
# Check current language mode
$ExecutionContext.SessionState.LanguageMode
# FullLanguage, ConstrainedLanguage, RestrictedLanguage, NoLanguage

# Constrained Language Mode is enforced when:
# - AppLocker/WDAC policies are active
# - PowerShell JEA (Just Enough Administration) is configured
```

---

## Useful PowerShell One-Liners for Administration

```powershell
# Find large files
Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue | 
    Where-Object {$_.Length -gt 100MB} | Sort-Object Length -Descending | 
    Select-Object FullName, @{N="SizeMB";E={[math]::Round($_.Length/1MB,2)}}

# List top 10 CPU processes
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name, Id, CPU

# Find all running services with their associated executables
Get-WmiObject Win32_Service | Where-Object {$_.State -eq "Running"} | 
    Select-Object Name, PathName, StartName | Sort-Object Name

# Test connectivity to multiple hosts
"8.8.8.8","8.8.4.4","1.1.1.1" | ForEach-Object { 
    [PSCustomObject]@{IP=$_; Reachable=(Test-Connection $_ -Count 1 -Quiet)} 
}

# Find all accounts with "Password Never Expires"
Get-LocalUser | Where-Object {$_.PasswordExpires -eq $null -and $_.Enabled -eq $true}

# Get last login times for local accounts
Get-LocalUser | Select-Object Name, LastLogon, Enabled | Sort-Object LastLogon
```

> **Next:** Proceed to [Folder 08: Windows Shortcuts](../08_Windows_Shortcuts/README.md) for the definitive keyboard shortcut database.
