[⬅️ Back to Main README](README.md)

---

## Table of Contents
- [Attach YubiKey to WSL](#attach-yubikey-to-wsl)
- [Initialize YubiKey OpenPGP](#initialize-yubikey-openpgp)
- [Generate Keys On-Device (Wizard Choices)](#generate-keys-on-device-wizard-choices)
- [Verify Key in Slot 1](#verify-key-in-slot-1)
- [Export Public Key (Paste into GitHub)](#export-public-key-paste-into-github)
- [Configure GPG and Git](#configure-gpg-and-git)
- [Daily Use & Post-Commit Fix](#daily-use--post-commit-fix)
- [Test Verification](#test-verification)

---
# YubiKey on WSL — GPG Signing Guide
*This guide covers adding a GPG key to **slot 1 (Signature key)** of the OpenPGP applet on your YubiKey for verified GitHub commits. The initial setup is in the main `README.md`; this doc is ancillary.*  

**In addition to verified GitHub commits, other uses of this GPG key pair include:**  
- Signing/verifying emails  
- Encrypting/decrypting files  
- Signing software releases  
- Authentication for supported services  

## Attach YubiKey to WSL
In an **Administrator PowerShell** on Windows:
```powershell
yk2wsl
```

---

## Initialize YubiKey OpenPGP
Inside WSL:
```bash
gpg --card-status
```
- Confirms OpenPGP applet is available.  
- Shows three slots (Signature = slot 1, Encryption = slot 2, Authentication = slot 3).

---

## Generate Keys On-Device (Wizard Choices)

Run:
```bash
gpg --edit-card
```
Then:
```
gpg/card> admin
gpg/card> generate
```

### First-Time Initialization
- On a brand-new applet, the wizard goes straight into key generation — no backup or overwrite prompts appear.  
- All three slots are initialized at once (Signature, Encryption, Authentication).  
- When prompted for **Real name, Email, Comment**, just press **Enter** to skip.  
- Default curves are applied:  
  ```
  Key attributes ...: ed25519 cv25519 ed25519
  ```
  - Slot 1 = ed25519 (Signature)  
  - Slot 2 = cv25519 (Encryption)  
  - Slot 3 = ed25519 (Authentication)  

### Regeneration (second run and later)
- When slots already contain keys, you’ll be prompted:  
  ```
  Make off-card backup of encryption key? (Y/n)
  gpg: Note: keys are already stored on the card!
  Replace existing keys? (y/N)
  ```
  - Choose **n** — there’s no need to back up initialized keys.  
  - Choose **N** (default) when asked about replacing keys, unless you explicitly want to overwrite all three slots.  

- During regeneration, when asked for identity info, fill it out to match your GitHub account:  
  - **Name** → your GitHub username, e.g., `userScooby12`  
  - **Email** → your GitHub noreply email, e.g., `3060702+userScooby12@users.noreply.github.com`  
  - **Comment** → optional; can be your org or left blank.  

⚠️ **Important for GitHub verification**  
- The **email must match exactly** one of the emails on your GitHub account.  
- The **name** does not affect verification — it’s just for display.  
- If the commit email and the GPG key email don’t match what GitHub has, the commit won’t show as “Verified.”  

---

## Verify Key in Slot 1
```bash
gpg --card-status
```
- Should display your **Signature key** in slot 1.  
- Public key material is also saved to your local GPG keyring.

---

## Export Public Key (Paste into GitHub)
Export the ASCII-armored public key:  
```bash
gpg --armor --export 3060702+userScooby12@users.noreply.github.com
```

- Copy the full output from stdout.  
- Paste it into GitHub → **Settings → SSH and GPG Keys → New GPG Key**.  
- Set the **title** to:  
  ```
  YUBIKEY - Auth
  ```

---

## Configure GPG and Git

Add this block to your shell config (`~/.zshrc`):  
```bash
# GPG signing
export GPG_TTY=$(tty)
gpgconf --launch gpg-agent
```

Apply it now:  
```bash
source ~/.zshrc
```

### Derive your KEYID
List your secret keys:  
```bash
gpg --list-secret-keys --keyid-format=long
```

Example output:  
```
sec>  ed25519/FFFCBFD975BC9CB0  created: 2025-09-13
      card-no: 0006 24996293
uid           [ultimate] userScooby12 <3060702+userScooby12@users.noreply.github.com>
```

- The part after the slash is the **KEYID**:  
  ```
  ed25519/FFFCBFD975BC9CB0
  ```
  → **KEYID = FFFCBFD975BC9CB0**

### Configure Git
Tell Git to use your key and sign commits:  
```bash
git config --global user.signingkey FFFCBFD975BC9CB0
git config --global commit.gpgsign true
```

Also set your Git identity to match GitHub:  
```bash
git config --global user.email "3060702+userScooby12@users.noreply.github.com"
git config --global user.name "userScooby12"
```

---

## Daily Use & Post-Commit Fix
- Run `yk2wsl` before using GPG in WSL.  
- For signed commits, Git will prompt for your YubiKey PIN.  
- On WSL, GPG signing and SSH can clash over card access. This is handled by the **post-commit hook fix** described in the main `README.md`. Without that hook, you’ll need to restart `pcscd` manually after each commit:  
  ```bash
  sudo service pcscd restart
  ```  

---

## Test Verification
Make a quick test commit to confirm GitHub shows “Verified.”  

If you don’t have an existing repo to push to, create an empty test repo on GitHub, clone it, and use that for the test.  

```bash
cd ~/some-repo
touch testfile.txt
git add testfile.txt
git commit -m "test signed commit"
git push
```

Then check the commit on GitHub:  
- A green **Verified** badge should appear.  
- If you see **Unverified**, check that your Git email matches your GitHub noreply email and that the correct public key is uploaded.