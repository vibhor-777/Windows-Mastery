# Folder 01: Windows Architecture and NT Kernel

## 4Ws Structure

- **Who:** System Administrators (Pranali Prashasak) and Advanced Developers (Uchch Vikshasak).
- **What:** Deep analysis of the Windows NT Kernel (Antarbhag), the User Mode (Upayogakarta Mode) vs. Kernel Mode (Antarbhag Mode) architecture, and the Hardware Abstraction Layer (HAL).
- **Where:** Applicable to all Windows NT-based systems (Windows 2000 through Windows 11).
- **Why:** Understanding kernel architecture enables precise troubleshooting, performance optimization, and informed security hardening decisions.

---

## The Windows NT Architecture Overview

Windows NT (New Technology) is a monolithic kernel with a modular design. Unlike pure microkernels (which run most OS services in user mode), the NT kernel runs critical components in kernel mode for performance while maintaining strict separation from user applications.

### The Two Worlds: User Mode vs. Kernel Mode

The fundamental division in Windows NT architecture is between User Mode (Upayogakarta Mode) and Kernel Mode (Antarbhag Mode):

**Kernel Mode (Ring 0):**
- Has unrestricted access to hardware and system memory.
- Runs the NT Executive, Kernel, HAL, and kernel-mode drivers.
- A bug or malicious code here can crash the entire system (Blue Screen of Death - BSOD).
- Protected by CPU privilege rings; user mode code cannot directly call kernel mode code.

**User Mode (Ring 3):**
- Restricted access; cannot directly address hardware.
- Each process runs in its own virtual address space, isolated from other processes.
- System calls (Pranali Aahvaan) transition to kernel mode via a gate mechanism (SYSCALL/SYSENTER instruction).
- Crashes in user mode are isolated to the faulting process.

---

## Core Architecture Components Table

| Component | Function | File Path |
|---|---|---|
| NT Executive | Core OS logic — I/O, Memory, Process management | `%SystemRoot%\System32\ntoskrnl.exe` |
| NT Kernel | Scheduling, synchronization, interrupt handling | `%SystemRoot%\System32\ntoskrnl.exe` |
| HAL | Hardware mediation and abstraction | `%SystemRoot%\System32\hal.dll` |
| Win32 Subsystem (Kernel) | Kernel-mode window and GDI management | `%SystemRoot%\System32\win32k.sys` |
| Win32 Subsystem (User) | User-mode Win32 API implementation | `%SystemRoot%\System32\csrss.exe` |
| Session Manager | Initializes the system after kernel boot | `%SystemRoot%\System32\smss.exe` |
| Local Security Authority | Authentication and security policy | `%SystemRoot%\System32\lsass.exe` |

---

## The NT Executive Sub-Components

The NT Executive is composed of several managers, each responsible for a specific OS resource:

| Manager | Responsibility |
|---|---|
| Object Manager (Vastu Prabandhan) | Creates, names, and manages all system objects (files, processes, threads, events) |
| Process and Thread Manager | Creates and terminates processes and threads |
| Virtual Memory Manager (VMM) | Manages virtual address spaces, paging, and working sets |
| I/O Manager (I/O Prabandhan) | Manages I/O requests between applications and device drivers |
| Security Reference Monitor (SRM) | Enforces access control and generates audit events |
| Cache Manager | Manages the file system cache for performance |
| Configuration Manager | Manages the Windows Registry (Panjikaran) |
| Plug and Play Manager | Detects and configures hardware devices |
| Power Manager | Controls power states (S0-S5) and device power management |

---

## The Hardware Abstraction Layer (HAL)

The HAL (Hardware Abstraction Layer - Yantra Saransh Stara) is a thin software layer that insulates the NT Kernel from hardware-specific details. This allows Windows to run on different hardware platforms without changing the core kernel code.

Key HAL functions:
- **Interrupt routing:** Translates hardware interrupts (IRQs) to system-level Interrupt Request Level (IRQL) values.
- **DMA management:** Manages Direct Memory Access operations.
- **System bus abstraction:** Hides differences between PCI, PCIe, and other bus architectures.
- **Timer abstraction:** Provides a consistent timer interface regardless of underlying hardware timers.

```powershell
# View HAL version and details
Get-Item "$env:SystemRoot\System32\hal.dll" | Select-Object Name, VersionInfo

# Check system architecture and HAL type
(Get-WmiObject Win32_OperatingSystem).OSArchitecture
```

---

## The Windows Kernel: Deep Components

### The Microkernel Layer

Within `ntoskrnl.exe`, the true microkernel handles:

1. **Thread Scheduling (Sutra Niyojak):** Uses a priority-based preemptive scheduling algorithm. 32 priority levels (0-31); real-time threads (16-31) preempt time-shared threads (0-15).
2. **Synchronization Primitives:** Spinlocks, mutexes, semaphores, events, and timer objects.
3. **Interrupt Handling:** Manages the IRQL hierarchy — from PASSIVE_LEVEL (user mode) through DISPATCH_LEVEL (DPC) to HIGH_LEVEL (hardware interrupts).
4. **Exception Dispatching:** Structured Exception Handling (SEH) framework.

### IRQL Hierarchy

| IRQL Level | Name | Description |
|---|---|---|
| 0 | PASSIVE_LEVEL | Normal user and kernel thread execution |
| 1 | APC_LEVEL | Asynchronous procedure calls |
| 2 | DISPATCH_LEVEL | Dispatcher and DPC queue |
| 3-26 | DIRQL | Device interrupt request levels |
| 27 | PROFILE_LEVEL | Performance profiling |
| 28 | CLOCK_LEVEL | System clock |
| 31 | HIGH_LEVEL | Non-maskable interrupts |

---

## Windows Subsystem Architecture

The Windows Subsystem provides the Win32 API to applications. It consists of:

1. **Csrss.exe (Client/Server Runtime Subsystem):** User-mode portion of the Win32 subsystem. Manages console windows, process/thread creation callbacks, and side-by-side (SxS) assembly loading.
2. **Win32k.sys:** Kernel-mode component handling window management (USER) and graphics device interface (GDI). This is a significant attack surface, as it has historically been a source of privilege escalation vulnerabilities.

```powershell
# View all running subsystem processes
Get-Process csrss | Format-List Id, Name, SessionId, Handles

# Inspect Windows kernel modules loaded
Get-CimInstance -ClassName Win32_SystemDriver | Where-Object {$_.State -eq "Running"} | 
    Select-Object Name, PathName | Sort-Object Name | Format-Table -AutoSize
```

---

## WOW64: Running 32-bit Applications on 64-bit Windows

WOW64 (Windows-on-Windows 64 - Sathi Windows par 64-bit Windows) is a subsystem that enables 32-bit applications to run on 64-bit Windows without modification.

| Component | Location | Purpose |
|---|---|---|
| WoW64.dll | `%SystemRoot%\System32\wow64.dll` | Core emulation layer |
| Wow64Win.dll | `%SystemRoot%\System32\wow64win.dll` | Win32 API redirection |
| Wow64Cpu.dll | `%SystemRoot%\System32\wow64cpu.dll` | CPU context management |

**Key WOW64 Behaviors:**

- **Registry Redirection:** 32-bit apps writing to `HKLM\SOFTWARE` are silently redirected to `HKLM\SOFTWARE\Wow6432Node`.
- **File System Redirection:** 32-bit apps accessing `%SystemRoot%\System32` are redirected to `%SystemRoot%\SysWOW64`.
- **SysWOW64 Directory:** Contains 32-bit versions of system DLLs. The naming is counter-intuitive — `SysWOW64` holds 32-bit files, while `System32` holds 64-bit files.

```cmd
REM Disable WOW64 file system redirection to access true System32 from a 32-bit process
REM This requires using the Sysnative alias:
dir %SystemRoot%\Sysnative\
```

---

## Kernel Debugging Fundamentals

```powershell
# Check kernel debugging status
bcdedit /dbgsettings

# Enable kernel debugging over network (requires reboot)
# Security Warning (Suraksha Chetavni): Only enable in isolated lab environments.
bcdedit /debug on
bcdedit /dbgsettings net hostip:192.168.1.100 port:50000

# View loaded kernel modules
Get-CimInstance Win32_SystemDriver | Select-Object Name, State, PathName | 
    Where-Object State -eq Running | Sort-Object Name
```

---

## Process Creation in NT Architecture

Understanding the process creation flow is essential for both development and security analysis:

1. `CreateProcess()` API call in user mode.
2. Kernel32.dll validates parameters and calls NtCreateProcess via SYSCALL.
3. The Process Manager in the NT Executive creates the process object.
4. Virtual Memory Manager creates the initial address space.
5. The executable image is mapped using the Image Section object.
6. A primary thread is created.
7. The Win32 subsystem (Csrss.exe) is notified.
8. The entry point is executed.

```powershell
# Monitor process creation in real-time (requires Sysinternals Procmon or WMI)
Register-CimIndicationEvent -Query "SELECT * FROM Win32_ProcessStartTrace" -SourceIdentifier "ProcessStart" -Action {
    $event = $Event.SourceEventArgs.NewEvent
    Write-Host "New process: $($event.ProcessName) (PID: $($event.ProcessID)) Parent: $($event.ParentProcessID)"
}
```

---

## Security Reference Monitor (SRM)

The SRM (Suraksha Sandarbh Nirikshaak) is the kernel component responsible for enforcing access control. It works in conjunction with the Local Security Authority (LSA) in user mode:

1. When a process opens a handle to an object, the SRM checks the process's Access Token against the object's Security Descriptor (containing the ACL).
2. If access is granted, an appropriate handle is returned.
3. All access checks generate audit events if auditing is configured.

```powershell
# View current process access token
[System.Security.Principal.WindowsIdentity]::GetCurrent() | Select-Object Name, Groups, Claims

# Check if running as administrator
([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]"Administrator")
```

> **Next:** Proceed to [Folder 02: File System Deep Dive](../02_File_System_Deep_Dive/README.md) for NTFS internals and ACL management.
