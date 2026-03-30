# Folder 13: Networking in Windows (OSI Model)

## 4Ws Structure

- **Who:** Network Administrators (Neta Prashasak) and Security Engineers.
- **What:** Windows networking architecture mapped to the 7-layer OSI Model, core protocols (TCP/IP, SMB, DNS), and security advancements in SMB 3.1.1.
- **Where:** Windows Networking Stack, Netsh, PowerShell networking cmdlets.
- **Why:** Understanding Windows networking at the OSI level enables precise troubleshooting, protocol-level security hardening, and network forensics.

---

## The OSI Model: Windows Implementation

```
OSI LAYER                   WINDOWS IMPLEMENTATION
═══════════════════════════════════════════════════════════════
┌───────────────────┐       ┌─────────────────────────────────┐
│  7. Application   │  ←→   │  HTTP, HTTPS, SMB, DNS, RDP,    │
│                   │       │  WinRM, LDAP, Kerberos, SMTP    │
├───────────────────┤       ├─────────────────────────────────┤
│  6. Presentation  │  ←→   │  TLS/SSL (Schannel), Kerberos   │
│                   │       │  encryption, data formatting     │
├───────────────────┤       ├─────────────────────────────────┤
│  5. Session       │  ←→   │  NetBIOS, SMB sessions, RPC     │
│                   │       │  sessions, Named Pipes           │
├───────────────────┤       ├─────────────────────────────────┤
│  4. Transport     │  ←→   │  TCP, UDP, QUIC (HTTP/3)        │
│                   │       │  Windows Filtering Platform (WFP)│
├───────────────────┤       ├─────────────────────────────────┤
│  3. Network       │  ←→   │  IPv4, IPv6, ICMP, IPsec        │
│                   │       │  Routing (RRAS), ARP             │
├───────────────────┤       ├─────────────────────────────────┤
│  2. Data Link     │  ←→   │  Ethernet (802.3), Wi-Fi        │
│                   │       │  (802.11), NDIS drivers          │
├───────────────────┤       ├─────────────────────────────────┤
│  1. Physical      │  ←→   │  NIC hardware, cables, RF       │
│                   │       │  signals (abstracted by NDIS)    │
└───────────────────┘       └─────────────────────────────────┘
```

---

## TCP/IP Stack in Windows

The Windows TCP/IP stack is implemented in `tcpip.sys` and provides:
- Full IPv4 and IPv6 support.
- TCP, UDP, and ICMP protocol handlers.
- Integration with the Windows Filtering Platform (WFP) for firewall enforcement.
- QUIC protocol support (HTTP/3 and SMB over QUIC).

### Key TCP/IP Registry Settings

```powershell
# Backup before modification
reg export "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" C:\Backups\Tcpip_backup.reg

# View TCP/IP parameters
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters"

# Harden TCP/IP stack
# Reduce TCP retransmissions (faster timeout on dead connections)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "TcpMaxConnectRetransmissions" -Value 2 -Type DWord

# Enable SYN attack protection
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "SynAttackProtect" -Value 2 -Type DWord

# Disable source routing (IP spoofing protection)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "DisableIPSourceRouting" -Value 2 -Type DWord

# Disable ICMP redirects (routing attack prevention)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "EnableICMPRedirect" -Value 0 -Type DWord
```

---

## SMB (Server Message Block): Protocol Deep Dive

SMB (Server Message Block - Pratham Sandesh Khand) is the primary Windows file sharing and printer sharing protocol.

### SMB Version History

| Version | OS | Key Features | Vulnerabilities |
|---|---|---|---|
| SMBv1 | Windows 3.11 — Vista | Basic file sharing | EternalBlue (MS17-010), WannaCry vector — **DISABLE IMMEDIATELY** |
| SMBv2 | Windows Vista/Server 2008 | Reduced round trips, larger MTU | Improved security |
| SMBv2.1 | Windows 7/Server 2008 R2 | Client-side caching | |
| SMBv3 | Windows 8/Server 2012 | Multichannel, encryption | |
| SMBv3.1.1 | Windows 10/Server 2016 | AES-256-GCM, pre-auth integrity, SMB over QUIC | Current standard |

### SMB 3.1.1 Security Features

- **AES-256-GCM Encryption (AES-256-GCM Ambigukarana):** Full in-transit encryption, much stronger than legacy RC4.
- **Pre-authentication Integrity:** SHA-512 hash of the connection establishment messages, preventing man-in-the-middle attacks against authentication.
- **Cluster Dialect Fencing:** Prevents protocol downgrade attacks.
- **SMB over QUIC:** Allows encrypted SMB without a VPN over the internet, using TLS 1.3-based QUIC transport.

### SMB Hardening

```powershell
# CRITICAL: Disable SMBv1 (vulnerability to EternalBlue/WannaCry)
# Security Warning (Suraksha Chetavni): SMBv1 must be disabled in ALL environments.

# Check SMBv1 status
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol
Get-WindowsOptionalFeature -Online -FeatureName SMB1Protocol

# Disable SMBv1
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -NoRestart

# Require SMB encryption for all connections
Set-SmbServerConfiguration -EncryptData $true -Force

# Require SMB signing (prevents relay attacks)
Set-SmbServerConfiguration -RequireSecuritySignature $true -Force

# View current SMB configuration
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol, 
    EncryptData, RequireSecuritySignature, EnableSecuritySignature

# View active SMB sessions
Get-SmbSession | Format-Table -AutoSize

# View open SMB files
Get-SmbOpenFile | Format-Table -AutoSize

# View SMB shares
Get-SmbShare | Format-Table Name, Path, Description, EncryptData -AutoSize
```

---

## DNS: Name Resolution in Windows

The Windows DNS Client service resolves hostnames to IP addresses using a cache and stub resolver.

### DNS Resolution Order

1. Local hosts file (`%SystemRoot%\System32\drivers\etc\hosts`)
2. DNS Client cache (in-memory)
3. Primary DNS server (configured on network adapter)
4. Secondary DNS server(s)
5. WINS (legacy, if configured)

```cmd
REM Display DNS cache
ipconfig /displaydns

REM Flush DNS cache (fixes stale/incorrect entries)
ipconfig /flushdns

REM DNS lookup
nslookup google.com
nslookup -type=MX contoso.com 8.8.8.8

REM Query DNS from PowerShell
Resolve-DnsName google.com
Resolve-DnsName google.com -Type MX
Resolve-DnsName google.com -Server 8.8.8.8
```

### DNS Security (DoH and DoT)

Windows 11 supports DNS over HTTPS (DoH) natively:

```powershell
# Configure DoH via PowerShell
Add-DnsClientDohServerAddress -ServerAddress "8.8.8.8" -DohTemplate "https://dns.google/dns-query" -AllowFallbackToUdp $false
Add-DnsClientDohServerAddress -ServerAddress "1.1.1.1" -DohTemplate "https://cloudflare-dns.com/dns-query" -AllowFallbackToUdp $false

# Set as primary DNS with DoH
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "8.8.8.8","8.8.4.4"

# View DoH settings
Get-DnsClientDohServerAddress
```

---

## Windows Firewall: Windows Filtering Platform (WFP)

Windows Defender Firewall is built on the Windows Filtering Platform (WFP), which provides deep packet inspection at multiple layers of the network stack.

```powershell
# View firewall status for all profiles
Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction

# Enable firewall for all profiles
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True

# Block all inbound connections by default (high-security mode)
Set-NetFirewallProfile -Profile Public -DefaultInboundAction Block -DefaultOutboundAction Allow

# Create a specific allow rule
New-NetFirewallRule -DisplayName "Allow RDP" -Direction Inbound -Protocol TCP -LocalPort 3389 -Action Allow -Profile Domain

# Block an application
New-NetFirewallRule -DisplayName "Block Notepad" -Direction Outbound -Program "C:\Windows\notepad.exe" -Action Block

# View all firewall rules
Get-NetFirewallRule | Where-Object {$_.Enabled -eq "True" -and $_.Direction -eq "Inbound"} |
    Select-Object DisplayName, Action, Profile | Format-Table -AutoSize

# Export firewall rules
netsh advfirewall export C:\Backups\firewall_backup.wfw

# Import firewall rules
netsh advfirewall import C:\Backups\firewall_backup.wfw
```

---

## Network Diagnostics

```powershell
# Test network connectivity
Test-NetConnection google.com
Test-NetConnection 192.168.1.1 -Port 443
Test-NetConnection -ComputerName DC01 -CommonTCPPort RDP

# View routing table
Get-NetRoute | Format-Table -AutoSize

# View network adapters
Get-NetAdapter | Select-Object Name, Status, LinkSpeed, MacAddress

# View IP configuration
Get-NetIPAddress | Format-Table IPAddress, PrefixLength, InterfaceAlias -AutoSize

# View ARP table
Get-NetNeighbor | Format-Table IPAddress, LinkLayerAddress, State -AutoSize

# View active TCP connections with process
Get-NetTCPConnection | Where-Object {$_.State -eq "Established"} |
    Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess |
    ForEach-Object {
        $proc = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
        $_ | Add-Member -NotePropertyName ProcessName -NotePropertyValue $proc.Name -PassThru
    } | Format-Table -AutoSize
```

> **Next:** Proceed to [Folder 14: User Accounts, Groups, and Permissions](../14_User_Accounts_Groups_Permissions/README.md) for identity and access management.
