# Folder 22: Modern Developer Setup

## 4Ws Structure

- **Who:** Developers (Vikshasak) and DevOps Engineers.
- **What:** Integration of VS Code, Git, Windows Terminal, and the SymCrypt cryptographic library for a secure, productive Windows development environment.
- **Where:** Windows 11 with Windows Subsystem for Linux (WSL 2) integration.
- **Why:** A properly configured development environment with security controls from the start prevents the accumulation of technical debt and security vulnerabilities in both the development process and the software produced.

---

## Windows Terminal: The Modern Console

Windows Terminal (available from Microsoft Store or GitHub) is the recommended terminal for Windows developers. It supports multiple tabs, PowerShell, CMD, WSL distributions, and Azure Cloud Shell.

### Installation and Configuration

```powershell
# Install Windows Terminal via winget
winget install --id Microsoft.WindowsTerminal --source winget

# Install PowerShell 7+ (recommended over Windows PowerShell 5.1)
winget install --id Microsoft.PowerShell --source winget

# Open Terminal settings
# Settings → Open settings file (settings.json)
```

### Windows Terminal settings.json Configuration

```json
{
    "$help": "https://aka.ms/terminal-documentation",
    "$schema": "https://aka.ms/terminal-profiles-schema",
    "defaultProfile": "{574e775e-4f2a-5b96-ac1e-a2962a402336}",
    "copyOnSelect": false,
    "copyFormatting": "none",
    "profiles": {
        "defaults": {
            "colorScheme": "One Half Dark",
            "fontFace": "Cascadia Code PL",
            "fontSize": 12,
            "useAcrylic": true,
            "acrylicOpacity": 0.9,
            "startingDirectory": "%USERPROFILE%",
            "antialiasingMode": "cleartype"
        },
        "list": [
            {
                "guid": "{574e775e-4f2a-5b96-ac1e-a2962a402336}",
                "name": "PowerShell 7",
                "source": "Windows.Terminal.PowershellCore",
                "icon": "ms-appx:///ProfileIcons/pwsh.png"
            }
        ]
    },
    "keybindings": [
        { "command": { "action": "splitPane", "split": "auto" }, "keys": "alt+shift+d" },
        { "command": "closePane", "keys": "ctrl+shift+w" }
    ]
}
```

---

## Git: Version Control Setup

```powershell
# Install Git
winget install --id Git.Git --source winget

# Initial Git configuration (global)
git config --global user.name "Your Name"
git config --global user.email "your.email@company.com"
git config --global core.autocrlf true       # Convert LF to CRLF on Windows
git config --global core.editor "code --wait" # VS Code as default editor
git config --global init.defaultBranch main
git config --global pull.rebase false

# Configure Git credential manager
git config --global credential.helper manager

# Sign commits with GPG (security best practice)
git config --global commit.gpgsign true
git config --global user.signingkey YOUR_GPG_KEY_ID

# Useful Git aliases
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.st "status -sb"
git config --global alias.co "checkout"
git config --global alias.br "branch -vv"
git config --global alias.undo "reset --soft HEAD~1"
```

### Git Security Best Practices

```powershell
# Prevent accidentally committing sensitive data
# Install git-secrets or similar tool
winget install --id BFG.Repo-Cleaner

# Create a global .gitignore for sensitive files
@"
# Credentials and secrets
*.pem
*.key
*.pfx
*.p12
*.cer
.env
.env.*
secrets.json
appsettings.local.json
*_credentials*
*_secrets*

# Windows-specific
Thumbs.db
Desktop.ini
*.lnk

# Development artifacts
node_modules/
.vs/
*.user
*.suo
bin/
obj/
dist/
*.pyc
__pycache__/
"@ | Out-File "$env:USERPROFILE\.gitignore_global" -Encoding UTF8

git config --global core.excludesFile "$env:USERPROFILE\.gitignore_global"
```

---

## Visual Studio Code: Security-Conscious Setup

```powershell
# Install VS Code
winget install --id Microsoft.VisualStudioCode --source winget

# Install essential extensions via CLI
$extensions = @(
    "ms-vscode.PowerShell",          # PowerShell extension
    "ms-vscode-remote.remote-wsl",   # WSL integration
    "ms-vscode-remote.remote-ssh",   # SSH remote development
    "ms-vscode.cpptools",            # C/C++ (for SymCrypt development)
    "eamodio.gitlens",               # Git supercharged
    "ms-azuretools.vscode-docker",   # Docker support
    "redhat.vscode-yaml",            # YAML support
    "ms-vscode.vscode-json",         # JSON tools
    "streetsidesoftware.code-spell-checker",  # Spell check
    "ms-vsliveshare.vsliveshare"     # Live Share collaboration
)

foreach ($ext in $extensions) {
    code --install-extension $ext
}
```

### VS Code settings.json for Security

```json
{
    "editor.formatOnSave": true,
    "editor.suggestOnTriggerCharacters": true,
    "files.autoSave": "onFocusChange",
    "telemetry.telemetryLevel": "off",
    "extensions.autoUpdate": false,
    "git.enableSmartCommit": true,
    "git.confirmSync": false,
    "security.workspace.trust.enabled": true,
    "security.workspace.trust.untrustedFiles": "prompt",
    "powershell.scriptAnalysis.enable": true,
    "powershell.codeFormatting.preset": "OTBS"
}
```

---

## SymCrypt: Microsoft's Open-Source Cryptographic Library

SymCrypt (Sanket Gupti - Symbolic Cryptography) is Microsoft's open-source, production-quality cryptographic library. It provides the cryptographic primitives used across Windows, Microsoft 365, and Azure.

### SymCrypt Capabilities

| Category | Algorithms |
|---|---|
| Symmetric Encryption | AES-128/256 (GCM, CBC, ECB, CTR), ChaCha20 |
| Hash Functions | SHA-1, SHA-256, SHA-384, SHA-512, SHA-3 |
| MAC | HMAC-SHA256, HMAC-SHA512, Poly1305 |
| Asymmetric | RSA, ECDH, ECDSA (P-256, P-384, P-521), Ed25519 |
| Key Derivation | HKDF, PBKDF2, SP800-108 |
| TLS | All TLS 1.3 cipher suites |

### Building SymCrypt on Windows

```powershell
# Prerequisites
winget install --id Microsoft.VisualStudio.2022.BuildTools
winget install --id Kitware.CMake

# Clone and build SymCrypt
git clone https://github.com/microsoft/SymCrypt.git
cd SymCrypt

# Configure with CMake
cmake -B build -G "Visual Studio 17 2022" -A x64 `
    -DCMAKE_BUILD_TYPE=Release `
    -DSYMCRYPT_TARGET_ARCH=AMD64 `
    -DSYMCRYPT_USE_ASM=ON

# Build
cmake --build build --config Release

# Run unit tests
.\build\exe\Release\symcryptunittest.exe
```

### Using SymCrypt in Applications

```c
// Example: AES-256-GCM encryption using SymCrypt
#include "symcrypt.h"

void EncryptData(
    const BYTE* key,        // 32-byte AES-256 key
    const BYTE* nonce,      // 12-byte nonce (must be unique per encryption)
    const BYTE* plaintext,
    SIZE_T plaintextLen,
    BYTE* ciphertext,
    BYTE* tag               // 16-byte authentication tag
) {
    SYMCRYPT_GCM_STATE gcmState;
    SYMCRYPT_GCM_KEY gcmKey;
    
    SymCryptGcmExpandKey(&gcmKey, SymCryptAesBlockCipher, key, 32);
    SymCryptGcmInit(&gcmState, &gcmKey, nonce, 12);
    SymCryptGcmEncrypt(&gcmState, plaintext, ciphertext, plaintextLen);
    SymCryptGcmGetTag(&gcmState, tag, 16);
}
```

---

## .NET Cryptography for Windows Applications

```csharp
// Secure data encryption using Windows Data Protection API (DPAPI)
using System.Security.Cryptography;
using System.Text;

public class SecureStorage
{
    // DPAPI: Encrypts data tied to the current Windows user account
    public static byte[] Protect(byte[] data, byte[] entropy = null)
    {
        return ProtectedData.Protect(data, entropy, DataProtectionScope.CurrentUser);
    }
    
    public static byte[] Unprotect(byte[] encryptedData, byte[] entropy = null)
    {
        return ProtectedData.Unprotect(encryptedData, entropy, DataProtectionScope.CurrentUser);
    }
    
    // AES-256-GCM using .NET 6+ AES-GCM
    public static (byte[] Ciphertext, byte[] Tag, byte[] Nonce) EncryptAesGcm(
        byte[] plaintext, byte[] key)
    {
        var nonce = new byte[AesGcm.NonceByteSizes.MaxSize]; // 12 bytes
        var tag = new byte[AesGcm.TagByteSizes.MaxSize];     // 16 bytes
        var ciphertext = new byte[plaintext.Length];
        
        RandomNumberGenerator.Fill(nonce);
        
        using var aesGcm = new AesGcm(key, tag.Length);
        aesGcm.Encrypt(nonce, plaintext, ciphertext, tag);
        
        return (ciphertext, tag, nonce);
    }
}
```

---

## PowerShell Profile Setup for Developers

```powershell
# View/edit PowerShell profile
code $PROFILE

# Sample developer profile content:
@'
# Developer PowerShell Profile

# Modules
Import-Module posh-git          # Git status in prompt
Import-Module PSReadLine        # Enhanced readline

# PSReadLine configuration
Set-PSReadLineOption -PredictionSource History
Set-PSReadLineOption -PredictionViewStyle ListView
Set-PSReadLineKeyHandler -Key UpArrow -Function HistorySearchBackward
Set-PSReadLineKeyHandler -Key DownArrow -Function HistorySearchForward

# Useful aliases
Set-Alias -Name g -Value git
Set-Alias -Name k -Value kubectl
Set-Alias -Name tf -Value terraform

# Quick navigation functions
function dev { Set-Location "$env:USERPROFILE\Development" }
function docs { Set-Location "$env:USERPROFILE\Documents" }

# Git shortcuts
function gst { git status }
function gco { git checkout $args }
function gp { git push }
function gl { git pull }

Write-Host "Developer profile loaded." -ForegroundColor Green
'@ | Out-File $PROFILE -Encoding UTF8
```

> **Next:** Proceed to [Folder 23: Package Management](../23_Package_Management/README.md) for Winget and Chocolatey mastery.
