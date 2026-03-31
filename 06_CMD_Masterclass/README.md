# Folder 06: CMD Masterclass (A-Z Command Reference)

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak), Network Engineers, and Support Technicians.
- **What:** Comprehensive reference for CMD commands (Aadesh) — the legacy but powerful Windows command interpreter.
- **Where:** `cmd.exe` — Elevated (Run as Administrator) where noted.
- **Why:** CMD remains the primary tool for legacy scripting, diagnostics, and rapid system administration tasks. Many recovery scenarios require CMD with no GUI available.

---

## CMD Fundamentals

CMD (Command Prompt / Aadesh Sanket) is the Windows command-line interpreter (`cmd.exe`). While PowerShell has largely superseded CMD for scripting, CMD remains essential for:
- Recovery environments (WinPE, RE)
- Legacy batch scripts (.bat, .cmd)
- Fast, lightweight diagnostics
- Compatibility with older tools and automation systems

### CMD Batch Scripting Essentials

```cmd
@echo off
REM This is a comment
setlocal EnableDelayedExpansion

REM Variables
set MY_VAR=Hello
echo %MY_VAR%

REM Conditional
if "%1"=="" (
    echo No argument provided
) else (
    echo Argument: %1
)

REM Loop
for /l %%i in (1,1,10) do echo Count: %%i

REM For loop over files
for %%f in (*.txt) do echo File: %%f

REM Error handling
net start "NonExistentService" 2>nul
if %errorlevel% neq 0 echo Service failed to start

REM Function-like labels
goto :main

:MyFunction
echo Inside function
goto :eof

:main
call :MyFunction
```

---

## A-Z Command Reference

### A

**arp — Address Resolution Protocol**

**Who:** Network Administrators.  
**What:** `arp /a` or `arp /s <InetAddr> <EtherAddr>`  
**Where:** CMD.  
**Why:** Display or modify the ARP cache to diagnose connectivity issues and prevent ARP cache poisoning attacks.

```cmd
arp /a                            REM Display all ARP cache entries
arp /a 192.168.1.1               REM Display ARP entry for specific IP
arp /s 192.168.1.1 00-AA-BB-CC-DD-EE  REM Add static ARP entry
arp /d 192.168.1.1               REM Delete ARP entry
arp /d *                         REM Delete all ARP entries
```

**assoc** — Display or modify file extension associations
```cmd
assoc .txt          REM Show association for .txt
assoc .bat=batfile  REM Set .bat association
assoc              REM Show all associations
```

**attrib** — Display or change file attributes
```cmd
attrib +h +s C:\hidden_file.txt    REM Hide and mark as system file
attrib -h -r C:\file.txt           REM Remove hidden and read-only attributes
attrib /s /d C:\Folder             REM Apply to all files in folder recursively
```

### B

**bitsadmin** — Background Intelligent Transfer Service management

**Who:** Administrators, Security Analysts (abused by malware for persistence).  
**What:** Manage BITS download/upload jobs.  
**Where:** Elevated CMD.  
**Why:** Diagnose legitimate BITS transfers; detect malware using BITS for stealthy downloads.

```cmd
bitsadmin /list /allusers          REM List all BITS jobs
bitsadmin /info <job_id>           REM Get info on specific job
bitsadmin /cancel <job_id>         REM Cancel a BITS job
bitsadmin /reset /allusers         REM Cancel ALL BITS jobs (all users)
bitsadmin /rawreturn /list /allusers  REM Verbose listing
```

**Security Warning (Suraksha Chetavni):** Malware frequently uses BITS jobs for persistent, stealthy file downloads and execution. Regularly audit `bitsadmin /list /allusers` for unexpected jobs.

**bcdedit** — Boot Configuration Database editor
```cmd
bcdedit /enum all                  REM List all BCD entries
bcdedit /set safeboot minimal      REM Boot into Safe Mode
bcdedit /deletevalue safeboot      REM Remove Safe Mode boot flag
bcdedit /set nx AlwaysOn           REM Enable DEP for all processes
```

**bootrec** — Boot record repair (WinRE only)
```cmd
bootrec /fixmbr      REM Repair Master Boot Record
bootrec /fixboot     REM Repair boot sector
bootrec /rebuildbcd  REM Rebuild BCD
bootrec /scanos      REM Scan for Windows installations
```

### C

**certutil** — Certificate utility (and notorious LOLBin)
```cmd
certutil -store My                    REM List personal certificate store
certutil -hashfile file.exe SHA256    REM Calculate file hash
certutil -decode encoded.b64 out.exe REM Decode base64 (abused by malware)
certutil -urlcache -f http://...      REM Download file (abused by malware)
```

**chkdsk** — Check disk
```cmd
chkdsk C: /F /R /X    REM Fix errors, find bad sectors, force dismount
chkdsk D: /scan       REM Non-destructive online scan
```

**cipher** — Manage EFS encryption

**Who:** Users and Administrators.  
**What:** `cipher /e <Dir>` to encrypt a directory.  
**Where:** CMD (standard user for user-owned files).  
**Why:** EFS-encrypt sensitive directories to protect data at rest.

```cmd
cipher /e /s:C:\ConfidentialDocs    REM Encrypt directory and contents
cipher /d /s:C:\ConfidentialDocs    REM Decrypt
cipher /u /n                        REM List encrypted files
cipher /w:C:                        REM Overwrite deleted data on C: (secure wipe)
```

**cls** — Clear screen
**copy** — Copy files
```cmd
copy source.txt dest.txt           REM Copy file
copy source.txt \\server\share\    REM Copy to network share
copy *.log C:\Logs\                REM Copy all .log files
```

### D

**del** — Delete files
```cmd
del /f /q /s C:\Temp\*.*          REM Force delete all files in Temp, quiet, recursive
```

**dir** — List directory contents
```cmd
dir /a:h /s C:\          REM Show hidden files recursively
dir /o:d                 REM Sort by date
dir /r C:\file.txt       REM Show alternate data streams
```

**diskpart** — Disk partition management
```cmd
diskpart
list disk
select disk 0
list partition
```

**driverquery** — List installed drivers
```cmd
driverquery /v /fo csv > C:\drivers.csv    REM Verbose CSV output
driverquery /si                            REM Show signing information
```

### E

**echo** — Display messages or toggle command echoing
**expand** — Expand compressed .cab files
```cmd
expand -f:* C:\Windows\System32\en-US\*.mui C:\Expanded\
```

### F

**find** — Search for a text string in files
```cmd
find "error" C:\Logs\app.log
find /i /c "error" C:\Logs\*.log    REM Count occurrences, case-insensitive
```

**findstr** — Advanced string search (supports regex)
```cmd
findstr /i /r "error\|warning" C:\Logs\*.log
findstr /s /i "password" C:\*.*    REM Search all files in current dir recursively
```

**format** — Format a disk volume
```cmd
format D: /FS:NTFS /Q /L          REM Quick format as NTFS with extended label
```

**fsutil** — Filesystem utility
```cmd
fsutil fsinfo ntfsinfo C:          REM NTFS volume information
fsutil behavior query DisableLastAccess  REM Check last-access timestamp setting
fsutil behavior set DisableLastAccess 1  REM Disable (performance improvement)
```

### G

**gpresult** — Group Policy result
```cmd
gpresult /h C:\gp_report.html /F  REM HTML report of applied GPOs
gpresult /r                        REM Quick summary
```

**gpupdate** — Group Policy update
```cmd
gpupdate /force           REM Force reapplication of all GPOs
gpupdate /sync /force     REM Force sync and wait
```

### I

**icacls** — Integrity Control ACLs
```cmd
icacls C:\SensitiveData /grant Administrators:(OI)(CI)F /T
icacls C:\SensitiveData /inheritance:d    REM Disable inheritance
icacls C:\SensitiveData /save acls.txt /T REM Save ACLs
icacls C:\ /restore acls.txt             REM Restore ACLs
```

**ipconfig** — IP configuration
```cmd
ipconfig /all            REM Full network adapter details
ipconfig /flushdns       REM Flush DNS cache
ipconfig /registerdns    REM Re-register DNS
ipconfig /displaydns     REM Show DNS cache
ipconfig /release        REM Release DHCP lease
ipconfig /renew          REM Renew DHCP lease
```

### M

**manage-bde** — BitLocker management

**Who:** Administrators.  
**What:** `manage-bde -status` check BitLocker encryption state.  
**Where:** Elevated CMD.  
**Why:** Verify BitLocker protection status on all drives.

```cmd
manage-bde -status                 REM Show BitLocker status for all drives
manage-bde -status C:              REM Status for C: drive
manage-bde -on C: -RecoveryPassword  REM Enable BitLocker with recovery password
manage-bde -protectors -get C:     REM Show key protectors
manage-bde -off C:                 REM Disable BitLocker (decrypts drive)
```

**mklink** — Create symbolic links
```cmd
mklink link.txt target.txt           REM File symlink
mklink /d C:\LinkToFolder C:\Target  REM Directory symlink
mklink /h hardlink.txt target.txt    REM Hard link
mklink /j C:\Junction C:\Target      REM Junction (directory hard link)
```

**msiexec** — Windows Installer
```cmd
msiexec /i package.msi               REM Install
msiexec /i package.msi /quiet /norestart  REM Silent install
msiexec /x package.msi               REM Uninstall
msiexec /a package.msi               REM Administrative install
```

### N

**net** — Network management (comprehensive utility)
```cmd
net user                             REM List local users
net user username /add               REM Add user
net user username * /domain          REM Change domain user password
net localgroup Administrators        REM List Administrators group
net localgroup Administrators username /add  REM Add to Administrators
net share                            REM List network shares
net share ShareName=C:\Path /GRANT:Everyone,READ  REM Create share
net use * /delete                    REM Disconnect all mapped drives
net view \\server                    REM List shares on server
net start ServiceName                REM Start a service
net stop ServiceName                 REM Stop a service
net statistics workstation           REM Network session statistics
```

**netstat** — Network statistics
```cmd
netstat -ano                         REM All connections with PID
netstat -b                           REM Show executable for each connection
netstat -e                           REM Ethernet statistics
netstat -r                           REM Routing table
netstat -ano | findstr :443          REM Filter for HTTPS connections
```

**netsh** — Network shell
```cmd
netsh interface ip show config       REM IP configuration
netsh advfirewall show allprofiles   REM Firewall status
netsh advfirewall firewall add rule name="Block Port 23" protocol=TCP dir=in localport=23 action=block
netsh wlan show profiles             REM Show saved WiFi profiles
netsh wlan show profile "NetworkName" key=clear  REM Show WiFi password
netsh winsock reset                  REM Reset Winsock (requires reboot)
```

**nslookup** — DNS lookup
```cmd
nslookup microsoft.com               REM Basic A record lookup
nslookup -type=MX microsoft.com      REM MX records
nslookup -type=TXT microsoft.com     REM TXT records
nslookup microsoft.com 8.8.8.8       REM Use specific DNS server
```

### P

**ping** — ICMP connectivity test
```cmd
ping google.com                      REM 4 ICMP echo requests
ping -t google.com                   REM Continuous ping (Ctrl+C to stop)
ping -n 100 -l 1400 server          REM 100 pings with 1400-byte payload
ping -4 google.com                   REM Force IPv4
ping -6 ipv6.google.com              REM IPv6 ping
```

**pnputil** — Plug and Play utility (Driver store management)

**Who:** Administrators.  
**What:** `pnputil.exe -e` enumerate third-party INF files.  
**Where:** Elevated CMD.  
**Why:** Audit installed third-party drivers for unauthorized or malicious driver packages.

```cmd
pnputil /enum-drivers                REM List all third-party drivers
pnputil /add-driver C:\Driver\driver.inf /install  REM Add and install driver
pnputil /delete-driver oem5.inf      REM Remove driver package
pnputil /scan-devices                REM Trigger PnP scan for new devices
```

### R

**reg** — Registry command-line tool
```cmd
reg query HKLM\SOFTWARE /s /f "Password"  REM Search registry for "Password" string
reg export HKLM C:\backup_hklm.reg        REM Export full HKLM
reg import C:\backup_hklm.reg             REM Import registry file
reg add HKLM\Key /v Name /t REG_DWORD /d 1 /f
reg delete HKLM\Key /v Name /f
```

**robocopy** — Robust file copy
```cmd
robocopy C:\Source D:\Dest /E /COPYALL /LOG:C:\robocopy.log  REM Full backup with ACLs
robocopy C:\Source D:\Dest /MIR /R:3 /W:10                   REM Mirror sync, 3 retries
robocopy C:\Source D:\Dest /E /XA:H /XD "System Volume Information"  REM Exclude hidden/special
```

### S

**sc** — Service Control
```cmd
sc query                             REM List all services
sc query type= all state= all        REM All services including stopped
sc start ServiceName
sc stop ServiceName
sc config ServiceName start= disabled  REM Disable a service
sc qc ServiceName                    REM Query service configuration
sc create MyService binpath= "C:\MyService.exe" start= auto
sc delete MyService
```

**schtasks** — Scheduled Tasks
```cmd
schtasks /create /tn "TaskName" /tr "C:\script.bat" /sc daily /st 23:00 /ru SYSTEM
schtasks /query /fo LIST /v          REM List all tasks, verbose
schtasks /run /tn "TaskName"         REM Manually run a task
schtasks /delete /tn "TaskName" /f   REM Delete task
```

**sfc** — System File Checker
```cmd
sfc /scannow           REM Scan and repair all protected system files
sfc /verifyonly        REM Scan without repairing
sfc /scanfile=C:\Windows\System32\kernel32.dll  REM Scan specific file
```

**shutdown** — Shutdown/restart
```cmd
shutdown /s /t 60 /c "Maintenance"  REM Shutdown in 60 seconds with comment
shutdown /r /t 0                    REM Immediate restart
shutdown /a                         REM Abort pending shutdown
shutdown /l                         REM Logoff current user
shutdown /s /f /m \\RemotePC        REM Force shutdown remote PC
```

### T

**takeown** — Take file ownership
```cmd
takeown /f C:\ProtectedFile.exe /r /d y  REM Take ownership recursively
```

**tasklist** — List running processes
```cmd
tasklist /svc                        REM Show services in each process
tasklist /m kernel32.dll             REM Processes using specific DLL
tasklist /fi "status eq running"     REM Filter by status
tasklist /fo csv > C:\processes.csv  REM Export to CSV
```

**taskkill** — Terminate processes
```cmd
taskkill /pid 1234 /f               REM Force kill by PID
taskkill /im notepad.exe /f         REM Force kill by name
taskkill /im malware.exe /f /t      REM Kill process and all children
```

**tracert** — Traceroute
```cmd
tracert google.com                  REM Trace route to Google
tracert -d google.com               REM Skip DNS resolution (faster)
tracert -h 30 google.com            REM Maximum 30 hops
```

### W

**wbadmin** — Windows Backup Admin
```cmd
wbadmin start backup -backuptarget:D: -include:C: -quiet
wbadmin get versions
wbadmin start recovery -version:MM/DD/YYYY-HH:MM -itemType:File -items:C:\data.db -recursive -quiet
```

**wevtutil** — Windows Event utility
```cmd
wevtutil el                          REM List all event logs
wevtutil qe System /c:20 /rd:true /f:text  REM Last 20 System events
wevtutil cl Security                  REM Clear Security log (requires elevation)
wevtutil export-log System C:\sys.evtx  REM Export log to file
```

**wmic** — WMI command line
```cmd
wmic computersystem get Name,Domain,Manufacturer,Model
wmic os get Caption,Version,BuildNumber
wmic process list full
wmic product list brief              REM Installed software
wmic nicconfig where IPEnabled=TRUE get IPAddress
```

> **Next:** Proceed to [Folder 07: PowerShell Masterclass](../07_PowerShell_Masterclass/README.md) for advanced scripting.
