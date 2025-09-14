![status](https://img.shields.io/badge/status-docs--only-blue)
![license](https://img.shields.io/badge/license-MIT-green)
![platform](https://img.shields.io/badge/platform-Windows%2FWSL-orange)


## Table of Contents

- [YubiKey-WSL](#yubikey-wsl)
  - [Slots](#yubikey-slots)
  - [Remove PIN Defaults](#remove-yubikey-pin-defaults)
  - [Why PIV for SSH?](#why-piv-and-not-fido2-for-ssh)
- [Getting Started with SSH](#getting-started-with-ssh)
  - [Assumptions](#assumptions)
  - [Windows Preparation](#windows-preparation)
  - [Attach YubiKey to WSL](#attach-yubikey-to-wsl)
  - [WSL Setup](#wsl-setup)
  - [Initialize PIV Slot 82](#initialize-piv-slot-82)
  - [Extract SSH Public Key](#extract-ssh-public-key)
  - [Add Key to GitHub](#add-key-to-github)
  - [Configure SSH](#configure-ssh-in-wsl)
  - [Test & Daily Use](#test)
- [Troubleshooting](#troubleshooting)
  - [Broken SSH After Using GPG](#gpg-and-scdaemon-will-break-ssh)

--- 
> 💡 *__This guide focuses on SSH with YubiKey.__*  
>  
> Related guides:  
> • [YubiKey on WSL — GPG Signing Guide](./GPG_Signing_Guide.md)  
> • [YubiKey on WSL — GitHub App Key Guide (Part of the GitHub AI Workflow series)](./GitHub_App_Key_Guide.md) 
> • [GitHub AI Workflow series](GitHub_AI_Workflow.md)
  
---

&nbsp;
# Yubikey-WSL
A guide for working with the Yubikey 5 Series hardware keys in WSL2 on Windows 11. Many of these techniques can be applied to Linux, MacOS or Windows 11 as well. The baseline flow of this document will walk you through initial setup of the binaries and helpers required and then walk you through the generation of a private SSH key on the YubiKey itself. There are also sections further on in the guide for GPG signing for verified github commits and storage of Bot/service PEMs for GitHub Apps, etc..

For an added level of security, ideally all Yubikey adminstration would be done on an air-gapped system such as Tails running on the RAMDisk but this step is beyond the scope of this guide.

## YubiKey slots

The YubiKey 5 series runs several independent applets:  
- **OpenPGP** – 3 fixed key slots (Signature, Encryption, Authentication).  
- **PIV (Personal Identity Verification)** – Multiple slots (9a, 9c–9e, retired 82–95) for certificates/keys via PKCS#11.  
- **FIDO2 / U2F** – Modern standard for web login and 2FA.  
- **OATH** – TOTP/HOTP secrets, used with Yubico Authenticator.  
- **OTP** – Legacy one-time password / static password applet.  

This guide focuses on **OpenPGP** and **PIV**, which are needed for SSH, GPG signing, and certificates. **FIDO2**, **OATH**, and **OTP** are mainly for web authentication and will not be covered.  

> ### ⚠️ Note: 
> GPG can cause a lockout by taking exclusive control of the OpenPGP applet, interfering with PIV/PKCS#11 usage. The [workaround](#gpg-and-scdaemon-can-break-ssh) is to run a small helper shell function after using GPG so the key is freed again for SSH. 

---

On YubiKey 5 series devices, PIV slots starting at 82 are labeled “retired” by Yubico, but in practice they’re generic, safe slots well-suited for the following purposes:
- SSH keys (GitHub, GitLab, infra access)
- Bot/service PEMs (GitHub Apps, automation, TLS client certs)
- VPN certificates (OpenVPN, WireGuard with PKCS#11)
- Code-signing certificates (different repos/products)
- Per-environment separation (dedicate slots by project/org/env)

 YubiKey 5 series hardware keys also have 3 OpenPGP slots:

- **Slot 1 (Signature)** – Sign files, emails, Git commits.  
- **Slot 2 (Encryption)** – Decrypt emails/files.  
- **Slot 3 (Authentication)** – General login/authentication (not used for SSH in this guide, since SSH will be handled via PIV slots with PKCS#11 for better compatibility and slot management).  

Later in the guide we will use OpenPGP slot 1 for GitHub GPG-verified commits

## Remove YubiKey PIN defaults
If youhave not already done so, secure your YubiKey PIN and PUK

#### PIN

- **Purpose:** Required every time the YubiKey performs a private-key operation via PIV (e.g., SSH with PKCS#11).  
- **Format:** Numeric only, length **6–8 digits**.  
- **Default:** `123456`.

**Change the PIN:**
```bash
ykman piv access change-pin
```
- Enter current PIN (default `123456` if never changed).  
- Enter the new PIN (6–8 digits).  
- Confirm the new PIN.

---

#### PUK

- **Purpose:** Used to reset the PIN if it is forgotten. Also known as the  Admin PIN. 
- **Format:** Numeric only, length **8 digits**.  
- **Default:** `12345678`.

**Change the PUK:**
```bash
ykman piv access change-puk
```
- Enter current PUK (default `12345678` if never changed).  
- Enter the new PUK (8 digits).  
- Confirm the new PUK.

---

#### Recovery Scenarios

- **Forgot PIN:** Use the PUK to reset it.  
- **Forgot PUK but still know PIN:** Device remains usable for daily operations, but if the PIN is lost in the future, the PIV applet must be reset.  
- **Forgot both PIN and PUK:** The PIV applet must be reset, which **erases all keys** in PIV slots.

# &nbsp;
#### Best Practices

- PIN and PUK **must** be changed from defaults on first use.  
- Store both in the in a secure password vault such as KeepassXC, with a sealed paper backup held in another secure location or by your by IT/security department.  
- Use different values for PIN and PUK.  
- Never share PIN or PUK outside of approved storage methods.  

# &nbsp;
# Why PIV and not FIDO2 for SSH?
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
   
## PIV vs FIDO2 Key Storage
  
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

# &nbsp;
# Getting started with SSH
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

# &nbsp;
# WSL Setup

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

# Troubleshooting #
- `pcsc_scan` should list the token.
- `ssh-keygen -D /usr/local/lib/libykcs11.so` should show an `ssh-rsa` key.
- **After sleep/hibernate:** Windows may re-enumerate USB devices and reclaim the YubiKey. Run `ykstatus` to confirm. If it shows "Attached to Windows", run `yk2wsl` again.


# &nbsp;

<a id="gpg-and-scdaemon-will-break-ssh"></a>
# ⚠️ GPG and scdaemon will break SSH

When you use the YubiKey’s OpenPGP applet (for example, running `gpg --card-status`, generating keys, or listing keys), **GnuPG starts `scdaemon`** to manage the smart card. The blocking issue is that `scdaemon` takes exclusive control of the YubiKey, which prevents the PKCS#11 provider (`libykcs11.so`) from accessing the PIV slot.  

### Symptom
After using `gpg`, such as signing a `git` commit, you then try to push that commit to the repo and your SSH authentication suddenly fails — `ssh` can no longer see your YubiKey key.

### Solution
Use a git post-commit hook to free up the YubiKey for PKCS#11 provided SSH

## Git Post-Commit Hook for YubiKey/GPG
`gpg`/`scdaemon` will lock the smartcard and break PKCS#11 provided SSH. This hook restarts `pcscd` after signed commits so the YubiKey is free again. 

> ### ⚠️ Note:
> To to let the hook run unattended, add a sudoers rule so your user and the hook can restart the service without a password (security impact is low since it only grants control of `pcscd`)

• Use `visudo`:
```bash
sudo visudo -f /etc/sudoers.d/pcscd
```
• Add:
```bash
# /etc/sudoers.d/pcscd
<your-username> ALL=(root) NOPASSWD: /usr/sbin/service pcscd restart
```
• Put the git post-commit hook in your template so all new repos inherit it:

```bash
mkdir -p ~/.git-template/hooks
nano ~/.git-template/hooks/post-commit
```

• Paste in the hook and make it executable:
```bash
chmod +x ~/.git-template/hooks/post-commit
```
### Git Post-Commit Hook
#### `~/.git-template/hooks/post-commit`
```sh
#!/bin/sh
# Workaround: GPG/scdaemon can lock smartcard → restart pcscd on signed commits
set -eu
DEBUG="${DEBUG:-0}"

log_debug() {
  [ "${DEBUG:-0}" = "1" ] || return 0
  echo "[DEBUG] $*" >&2
}

SERVICE="$(command -v service || echo /usr/sbin/service)"
log_debug "SERVICE=$SERVICE"

[ -x "$SERVICE" ] || { echo "post-commit: missing $SERVICE" >&2; exit 1; }

# Unsigned? say so and stop.
if ! git cat-file -p HEAD | grep -q '^gpgsig '; then
  echo "commit is unsigned, no additional action taken"
  log_debug "HEAD is an unsigned commit, pcscd was NOT restarted"
  exit 0
fi
# Use sudo -n if not root (visudo rule covers this), else call directly.
if [ "$(id -u)" -eq 0 ]; then
  RUN=""
else
  RUN="sudo -n"
fi
log_debug "RUN='$RUN'"

if $RUN "$SERVICE" pcscd restart >/dev/null 2>&1; then
  echo "pcscd reset"
  log_debug "HEAD is a signed commit, pcscd restart successful"
else
  rc=$?
  echo "post-commit: failed to restart pcscd (exit $rc)" >&2
  log_debug "pcscd restart failed with exit code $rc"
  exit "$rc"
fi
```
## Update Existing Repos
If needed, copy the hook into repos created earlier (run inside the repo root):

```bash
cp ~/.git-template/hooks/post-commit .git/hooks/post-commit
chmod +x .git/hooks/post-commit
```

## Optional Helper
Add this to `~/.zshrc` or `~/.bashrc` for a quick one-liner to sync the hook into the current repo:

```bash
sync-hook() {
  cp ~/.git-template/hooks/post-commit .git/hooks/post-commit
  chmod +x .git/hooks/post-commit
  echo "Hook synced into $(git rev-parse --show-toplevel)"
}
```
## Quick Fix
You can also set this workaround up for times where you want to use it without a git hook (such as random troubleshooting).

Add this helper function to your shell config (`.zshrc` or `.bashrc`):

```bash
# Free YubiKey from scdaemon and restart pcscd so PKCS#11 can use it
yk-ssh() {
  gpgconf --kill scdaemon
  sudo service pcscd restart
}
```

# Entire Guide Tested On

![Windows 11 Pro 22H2](https://img.shields.io/badge/Windows-11%20Pro%2022H2-blue?logo=windows)
![WSL2 Ubuntu 22.04](https://img.shields.io/badge/WSL2-Ubuntu%2022.04-green?logo=ubuntu)
![YubiKey 5C NFC](https://img.shields.io/badge/YubiKey-5C%20NFC-yellow?logo=yubico)  

![OpenSSH 8.9p1](https://img.shields.io/badge/OpenSSH-8.9p1-lightgrey?logo=openssh)
![ykman 5.8.0](https://img.shields.io/badge/ykman-5.8.0-orange)
![usbipd--win 5.2.0](https://img.shields.io/badge/usbipd--win-5.2.0-purple)
