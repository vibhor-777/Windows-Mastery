# Folder 23: Package Management (Winget and Chocolatey)

## 4Ws Structure

- **Who:** Developers (Vikshasak), System Administrators (Pranali Prashasak), and DevOps Engineers.
- **What:** Comprehensive cheatsheet and reference for Windows package managers — Winget (built-in) and Chocolatey (community).
- **Where:** PowerShell or CMD (administrative privileges recommended for system-wide installs).
- **Why:** Package managers enable consistent, scriptable, and auditable software deployment — eliminating manual download-and-click installations and enabling reproducible environments.

---

## Winget: The Native Windows Package Manager

Winget (Windows Package Manager) is Microsoft's built-in package manager, available in Windows 10 (1809+) and Windows 11. It draws packages from the Windows Package Manager Community Repository (winget-pkgs on GitHub).

### Winget Installation

Winget is pre-installed on Windows 11. For Windows 10:
```powershell
# Install from Microsoft Store (App Installer)
Start-Process "ms-windows-store://pdp/?ProductId=9NBLGGH4NNS1"

# Or install via PowerShell (for automation)
$url = "https://aka.ms/getwinget"
Invoke-WebRequest -Uri $url -OutFile "$env:TEMP\AppInstaller.msixbundle"
Add-AppxPackage -Path "$env:TEMP\AppInstaller.msixbundle"
```

---

## Winget Command Reference

| Task | Winget Command |
|---|---|
| **Search for a package** | `winget search <App>` |
| **Install a package** | `winget install --id <PackageID>` |
| **Silent install** | `winget install --id <ID> --silent --accept-package-agreements --accept-source-agreements` |
| **Upgrade a package** | `winget upgrade --id <PackageID>` |
| **Upgrade all packages** | `winget upgrade --all` |
| **Uninstall a package** | `winget uninstall --id <PackageID>` |
| **List installed packages** | `winget list` |
| **Show package info** | `winget show <PackageID>` |
| **Export installed packages** | `winget export -o packages.json` |
| **Import packages from file** | `winget import -i packages.json` |
| **Add a source** | `winget source add --name <name> --arg <url>` |
| **List sources** | `winget source list` |
| **Verify integrity** | `winget validate <manifest.yaml>` |
| **Pin a package version** | `winget pin add --id <ID> --version <ver>` |
| **Unpin a package** | `winget pin remove --id <ID>` |

### Common Winget Package IDs

```powershell
# Developer Tools
winget install --id Git.Git
winget install --id Microsoft.VisualStudioCode
winget install --id Microsoft.WindowsTerminal
winget install --id Microsoft.PowerShell
winget install --id Microsoft.DotNet.SDK.8
winget install --id OpenJS.NodeJS.LTS
winget install --id Python.Python.3.12
winget install --id GoLang.Go
winget install --id Rustlang.Rust.MSVC
winget install --id Docker.DockerDesktop
winget install --id Kubernetes.kubectl
winget install --id Hashicorp.Terraform
winget install --id Amazon.AWSCLI
winget install --id Google.CloudSDK

# Productivity
winget install --id 7zip.7zip
winget install --id Notepad++.Notepad++
winget install --id Mozilla.Firefox
winget install --id Google.Chrome
winget install --id Greenshot.Greenshot
winget install --id VideoLAN.VLC
winget install --id Adobe.Acrobat.Reader.64-bit

# Security Tools
winget install --id Wireshark.Wireshark
winget install --id nmap.nmap
winget install --id SysInternals.SysinternalsSuite
winget install --id GnuPG.GnuPG
winget install --id OpenVPN.OpenVPN
winget install --id KeePass.KeePass
winget install --id Bitwarden.Bitwarden
```

### Bulk Installation Script

```powershell
# Automated silent bulk installation
$packages = @(
    "Git.Git",
    "Microsoft.VisualStudioCode",
    "Microsoft.PowerShell",
    "7zip.7zip",
    "Notepad++.Notepad++",
    "SysInternals.SysinternalsSuite"
)

$failed = @()
foreach ($pkg in $packages) {
    Write-Host "Installing: $pkg" -ForegroundColor Yellow
    $result = winget install --id $pkg --silent --accept-package-agreements --accept-source-agreements 2>&1
    if ($LASTEXITCODE -ne 0 -and $LASTEXITCODE -ne -1978335189) {  # -1978335189 = already installed
        Write-Warning "Failed to install: $pkg"
        $failed += $pkg
    } else {
        Write-Host "Installed: $pkg" -ForegroundColor Green
    }
}

if ($failed) {
    Write-Warning "Failed packages: $($failed -join ', ')"
}
```

---

## Chocolatey: The Community Package Manager

Chocolatey (Meetha Pakaj Prabandhan) is a community-driven package manager that predates Winget. It has a larger package catalog (10,000+) and enterprise features.

### Chocolatey Installation

```powershell
# Install Chocolatey (run as Administrator)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Verify installation
choco --version
```

### Chocolatey Command Reference

| Task | Chocolatey Command |
|---|---|
| **Search for a package** | `choco search <App>` |
| **Install a package** | `choco install <Package>` |
| **Silent install** | `choco install <Package> -y` |
| **Install specific version** | `choco install <Package> --version <ver>` |
| **Upgrade a package** | `choco upgrade <Package>` |
| **Upgrade all packages** | `choco upgrade all -y` |
| **Uninstall a package** | `choco uninstall <Package>` |
| **List installed packages** | `choco list --local-only` |
| **Show package info** | `choco info <Package>` |
| **Pin a package** | `choco pin add --name <Package>` |
| **Outdated packages** | `choco outdated` |

---

## Winget vs. Chocolatey Comparison

| Task | Winget Command | Chocolatey Command |
|---|---|---|
| Search | `winget search <App>` | `choco search <App>` |
| Install | `winget install --id <ID>` | `choco install <Package>` |
| Upgrade | `winget upgrade --all` | `choco upgrade all -y` |
| Uninstall | `winget uninstall --id <ID>` | `choco uninstall <Package>` |
| List installed | `winget list` | `choco list --local-only` |
| Show info | `winget show <ID>` | `choco info <Package>` |
| Export | `winget export -o file.json` | N/A (use choco list --local-only) |
| Import | `winget import -i file.json` | N/A |
| Source management | `winget source add` | `choco source add` |
| Enterprise features | Limited | Chocolatey for Business |

---

## Winget Configuration (YAML Deployment)

```yaml
# winget-config.yaml - Declarative configuration for a developer workstation
# Apply with: winget configure --file winget-config.yaml

# yaml-language-server: $schema=https://aka.ms/configuration-dsc-schema/0.2
properties:
  resources:
    - resource: Microsoft.WinGet.DSC/WinGetPackage
      id: GitInstall
      directives:
        description: Install Git
        allowPrerelease: false
      settings:
        id: Git.Git
        source: winget
    - resource: Microsoft.WinGet.DSC/WinGetPackage
      id: VSCodeInstall
      directives:
        description: Install VS Code
      settings:
        id: Microsoft.VisualStudioCode
        source: winget
  configurationVersion: 0.2.0
```

---

## Enterprise Package Management with SCCM/Intune

For large enterprise deployments, Winget and Chocolatey work alongside:

```powershell
# Deploy via Intune using PowerShell script
# Create a PowerShell script in Intune: Devices → Scripts → Add

$logPath = "C:\Windows\Logs\AppInstall.log"
function Write-Log { param($msg) "$((Get-Date).ToString('yyyy-MM-dd HH:mm:ss')) $msg" | 
    Add-Content $logPath }

$apps = @("Git.Git", "Microsoft.VisualStudioCode", "7zip.7zip")
foreach ($app in $apps) {
    Write-Log "Installing $app"
    winget install --id $app --silent --accept-package-agreements --accept-source-agreements
    Write-Log "Result: $LASTEXITCODE for $app"
}

Write-Log "Installation complete"
```

---

## Scoop: Lightweight User-Level Package Manager

Scoop is another Windows package manager that installs without admin rights and places everything in `~\scoop`:

```powershell
# Install Scoop
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod get.scoop.sh | Invoke-Expression

# Add extras bucket
scoop bucket add extras
scoop bucket add nerd-fonts

# Install tools
scoop install git vscode nodejs python
scoop install cascadiacode-nf    # Nerd Font for terminal

# Update all
scoop update *
```

> **Next:** Proceed to [Folder 24: WSL 2](../24_WSL2/README.md) for Linux integration on Windows.
