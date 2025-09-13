![status](https://img.shields.io/badge/status-docs--only-blue)
![license](https://img.shields.io/badge/license-MIT-green)
![platform](https://img.shields.io/badge/platform-Windows%2FWSL-orange)
# Yubikey-WSL-SSH
Step list for getting SSH to work from a Yubikey in WSL2 on Windows 11.

On YubiKey 5 series devices, PIV slots starting at 82 are labeled “retired” by Yubico, but in practice they’re generic, safe slots well-suited for SSH keys.

## Why PIV and not FIDO2?
 1. **Private key never leaves the YubiKey**
    - With PIV, the keypair is generated on the YubiKey itself in slot 82.
    - The private key is non-exportable by design — it can’t ever be written out to the host machine.
    - This aligned with any requirement where the private key must never touch the local machine.
 2. **SSH client compatibility**
    - OpenSSH supports PKCS#11 providers (ssh-keygen -D /usr/local/lib/libykcs11.so).
    - That means you can use the YubiKey PIV slot like a traditional SSH key without any patches or special drivers.
    - FIDO2 (sk-ecdsa / sk-ed25519 keys) is supported in newer OpenSSH, but requires libfido2 and uses a different key format (sk-...). It’s less universally compatible in environments/tools compared to RSA keys from PIV.
 3. **Predictable slot management**
    - PIV gives you numbered slots (82 = “Retired Key 1”), so you can document exactly where a key lives.
    - That makes it easy to standardize across employees (e.g., “all GitHub keys go in slot 82”).
    - FIDO2 doesn’t expose slot numbers in the same way, rather, its system is more opaque.
   
### PIV vs FIDO2 Key Storage
  
| Feature                  | PIV (PKCS#11)                                | FIDO2                                   |
|---------------------------|----------------------------------------------|-----------------------------------------|
| Key storage model         | Fixed slots (24 total: 0x9A, 0x9C–0x9E, plus 20 retired slots 82–95) | Credentials bound to relying parties (e.g., github.com) |
| Number of keys supported  | ~24 independent keypairs                     | Typically 25–50 resident credentials (model/firmware dependent) |
| Slot/credential clarity   | Each slot has a fixed ID (e.g., slot 82 = GitHub) | Credentials are opaque, no simple slot mapping |
| Key formats               | RSA, ECC via PIV                            | `sk-ecdsa` / `sk-ed25519` (FIDO2/WebAuthn) |
| SSH support               | Works via PKCS#11 provider (`libykcs11.so`) | Supported in newer OpenSSH, requires `libfido2` |
| GitHub compatibility      | Standard `ssh-rsa` public key accepted       | Supported, but different key type (`sk-...`) and less common |
| Private key handling      | Generated and locked in-slot, never exportable | Generated on token, never exportable     |
  
**Summary:**  
PIV provides predictable, slot-based key management with more independent slots, making it easier to standardize usage across teams.  
FIDO2 is credential-based with fewer total slots, less transparency, and requires newer OpenSSH tooling, though it offers modern security benefits.


## Assumptions
- Windows 10/11 + WSL2 (Ubuntu/Debian).
- You want RSA-2048 in PIV slot 82 via PKCS#11.
- You already have a YubiKey inserted into the USB port of your computer or device.

---

## Windows Preparation

### Install usbipd-win
Open an elevated PowerShell (Run as Administrator):

```powershell
winget install --id=Microsoft.usbipd-win
```

Verify installation:

```powershell
usbipd --version
```

### Install YubiKey Manager
Download the installer from Yubico’s official page:  
<https://www.yubico.com/support/download/yubikey-manager/>

Verify the installer’s digital signature **before running it**:

```powershell
Get-AuthenticodeSignature "$HOME\Downloads\yubikey-manager-latest.exe"
```

Confirm that the `SignerCertificate` shows **Yubico AB** and `Status` is `Valid`.

Then install YubiKey Manager.

Add permanently to PATH:

```powershell
$ykPath = 'C:\Program Files\Yubico\YubiKey Manager'

# Permanent: add YubiKey Manager to your user PATH for all future sessions
[Environment]::SetEnvironmentVariable('Path', $env:Path + ";$ykPath", 'User')

# Immediate: update current PowerShell session so you can use ykman right now
$env:Path += ";$ykPath"
```

Start a new PowerShell and confirm:

```powershell
ykman --version
```

### Load usbipd Helpers
A YubiKey can only be accessed by **either Windows or WSL at a given time**, never both simultaneously. These helpers provide quick commands to move the YubiKey between Windows and WSL, check status, and print usage.

1. Save the following code to:
```
C:\Users\<USER NAME>\Documents\WindowsPowerShell\YubiKey-WSL-Helpers.ps1
```

```powershell
# === YubiKey PowerShell Helpers (Clean Version) ===

function Test-Admin {
    $id = [Security.Principal.WindowsIdentity]::GetCurrent()
    $p  = New-Object Security.Principal.WindowsPrincipal($id)
    if (-not $p.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
        throw "Run PowerShell as Administrator."
    }
}

function YK-ToWSL {
    param([string]$Distribution)
    try {
        Test-Admin
        $busid = (usbipd list | Select-String -Pattern '^\s*([0-9-]+)\s+1050:').Matches[0].Groups[1].Value
        usbipd bind --busid $busid | Out-Null
        if ($Distribution) {
            usbipd attach --wsl --busid $busid --distribution $Distribution | Out-Null
            Write-Host "YubiKey attached to WSL (BUSID $busid) -> $Distribution"
        } else {
            usbipd attach --wsl --busid $busid | Out-Null
            Write-Host "YubiKey attached to WSL (BUSID $busid)"
        }
    } catch {
        Write-Host "Error: $($_.Exception.Message)"
    }
}

function YK-ToWindows {
    try {
        Test-Admin
        $busid = (usbipd list | Select-String -Pattern '^\s*([0-9-]+)\s+1050:').Matches[0].Groups[1].Value
        usbipd detach --busid $busid 2>$null | Out-Null
        usbipd unbind --busid $busid  2>$null | Out-Null
        Write-Host "YubiKey returned to Windows (BUSID $busid)"
    } catch {
        Write-Host "Error: $($_.Exception.Message)"
    }
}

function YK-Status {
    $toWslCmd = "yk2wsl"
    $toWinCmd = "yk2windows"

    try {
        $busid = (usbipd list | Select-String -Pattern '^\s*([0-9-]+)\s+1050:').Matches[0].Groups[1].Value
    } catch {
        Write-Host "No YubiKey (VID 1050) detected."
        Write-Host "Hint: plug it in, then run $toWslCmd to use in WSL or keep using it in Windows."
        return
    }

    $listLine = usbipd list | Select-String -Pattern "^\s*$busid\s"
    if (-not $listLine) {
        Write-Host "Could not read usbipd state for BUSID $busid."
        return
    }

    $line = $listLine.Line

    if ($line -match 'Attached') {
        Write-Host "Attached to WSL (BUSID $busid)."
        Write-Host "Next: use it inside WSL."
        return
    }

    if ($line -match 'Shared') {
        Write-Host "Shared (export enabled) but NOT attached to WSL (BUSID $busid)."
        Write-Host "Windows can still use the YubiKey right now."
        Write-Host "To move it into WSL: $toWslCmd"
        Write-Host "To keep it in Windows exclusively, run: $toWinCmd"
        return
    }

    Write-Host "Attached to Windows (BUSID $busid)."
    Write-Host "To use from WSL: $toWslCmd"
}

function YK-Help {
    $lines = @(
        "YubiKey <-> WSL quick commands",
        "",
        "Aliases:",
        "  yk2wsl      = YK-ToWSL      (ADMIN) Attach YubiKey to WSL",
        "  yk2windows  = YK-ToWindows  (ADMIN) Return YubiKey to Windows",
        "  ykstatus    = YK-Status     Show where the YubiKey is attached",
        "  ykhelp      = YK-Help       Show this help message",
        "",
        "Tips:",
        "  Open Windows Terminal as Administrator for yk2wsl / yk2windows",
        "  BUSID changes if you move USB ports, auto-detected each time"
    )
    $lines | ForEach-Object { Write-Host $_ }
}

# Aliases
Set-Alias yk2wsl     YK-ToWSL
Set-Alias yk2windows YK-ToWindows
Set-Alias ykstatus   YK-Status
Set-Alias ykhelp     YK-Help
```

2. Source it in your PowerShell profile so it loads each session:  

Profile location (typical):
```
C:\Users\<USER NAME>\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
```

Append:
```powershell
. "$HOME\Documents\WindowsPowerShell\YubiKey-WSL-Helpers.ps1"
```

Reload profile:
```powershell
. $PROFILE
```

---

## Attach YubiKey to WSL

```powershell
yk2wsl
```

---

## WSL Setup

### Update and install required packages
```bash
sudo add-apt-repository ppa:yubico/stable
sudo apt update
sudo apt install -y   yubikey-manager yubico-piv-tool   pcscd pcsc-tools libccid   build-essential cmake pkg-config git   libssl-dev libpcsclite-dev libcbor-dev
```

### Start smart card daemon and check reader
```bash
sudo service pcscd start
pcsc_scan   # press Ctrl+C to stop after confirming the YubiKey is visible
```

### Verify ykman inside WSL
```bash
ykman --version
```
Confirm it shows **>= 5.0**.

### Build and install libykcs11.so from source
The PKCS#11 module `libykcs11.so` is the bridge that lets SSH and other software talk to your YubiKey’s PIV interface. Generic PKCS#11 libraries can sometimes see smart cards, but they often miss YubiKey-specific capabilities and edge cases. Building and installing Yubico’s own module ensures **complete YubiKey feature coverage and the best compatibility** for real-world use.

```bash
cd ~
git clone https://github.com/Yubico/yubico-piv-tool.git
cd yubico-piv-tool
mkdir build && cd build
cmake ..
make -j"$(nproc)"
sudo make install

# Add /usr/local/lib to the system library search path so libykcs11.so is always found
echo | sudo tee /etc/ld.so.conf.d/yubico-piv-tool.conf >/dev/null <<'EOF'
/usr/local/lib
EOF
sudo ldconfig
```

---

## Initialize PIV Slot 82

With the YubiKey visible to Windows, generate an RSA-2048 keypair and a self-signed certificate. Use company convention for subject metadata:  
`CN=<employee name>,O=LearnStream,OU=<division>`  
(where *employee name* = your username and *division* = your department, e.g. Engineering).

```powershell
ykman piv keys generate -a RSA2048 82 - | ykman piv certificates generate 82 - "CN=<username>,O=LearnStream,OU=<division>"
```

---

## Extract SSH Public Key

```bash
ssh-keygen -D /usr/local/lib/libykcs11.so 2>/dev/null | grep 'Retired Key 1'
```

---

## Add Key to GitHub
- Go to GitHub → Settings → SSH and GPG keys → New SSH key
- Title: `YUBIKEY - slot 82`
- Paste contents of the output from the previous command

---

## Configure SSH in WSL

`~/.ssh/config`:
```
Host github.com
  User git
  IdentitiesOnly yes
  PKCS11Provider /usr/local/lib/libykcs11.so
```

---

## Test
```bash
ssh -T git@github.com
```

Expected:
You will be prompted for your Yubikey PIN. If you did not set a PIN then it will be the default 123456. After entering the PIN you should see:
`Hi <your_user>! You’ve successfully authenticated, but GitHub does not provide shell access.`

---

## Daily Use Notes
- Run `yk2wsl` before starting work in WSL.

---

## Troubleshooting
- `pcsc_scan` should list the token.
- `ssh-keygen -D /usr/local/lib/libykcs11.so` should show an `ssh-rsa` key.
- **After sleep/hibernate:** Windows may re-enumerate USB devices and reclaim the YubiKey. Run `ykstatus` to confirm. If it shows "Attached to Windows", run `yk2wsl` again.

## Tested On

![Windows 11 Pro 22H2](https://img.shields.io/badge/Windows-11%20Pro%2022H2-blue?logo=windows)
![WSL2 Ubuntu 22.04](https://img.shields.io/badge/WSL2-Ubuntu%2022.04-green?logo=ubuntu)
![YubiKey 5C NFC](https://img.shields.io/badge/YubiKey-5C%20NFC-yellow?logo=yubico)  

![OpenSSH 8.9p1](https://img.shields.io/badge/OpenSSH-8.9p1-lightgrey?logo=openssh)
![ykman 5.8.0](https://img.shields.io/badge/ykman-5.8.0-orange)
![usbipd--win 5.2.0](https://img.shields.io/badge/usbipd--win-5.2.0-purple)
