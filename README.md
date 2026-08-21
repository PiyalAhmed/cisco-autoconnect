# 🚀 Cisco AnyConnect Touch ID Auto-Connect CLI

A highly secure, fully automated command-line interface for Cisco AnyConnect VPN on macOS. 

This tool completely eliminates the need to manually type your password or open your authenticator app every time you connect to your corporate VPN. It encrypts your credentials using GPG, stores your password in the macOS Keychain, and uses **Touch ID** to seamlessly decrypt your data and generate live 2FA/TOTP codes on the fly.

## ✨ Features
* **Biometric Security:** Uses macOS Touch ID to authorize VPN connections.
* **Zero Plaintext:** Credentials are encrypted at rest using GNU Privacy Guard (GPG).
* **Auto-TOTP Generation:** Automatically generates your 6-digit 2FA code in the background.
* **Smart Secret Extraction:** Paste your raw Base32 secret *or* the long `otpauth-migration://` export link directly from Google Authenticator, and the setup script will decode it for you.
* **Clean Terminal Output:** Hides Cisco's verbose connection logs and handles terminal cleanup automatically.

---

## 🛠 Prerequisites
* **macOS** (Intel or Apple Silicon)
* **Cisco AnyConnect Secure Mobility Client** (or its rebrand, **Cisco Secure Client**) installed via your organization — this tool automates the Cisco VPN CLI, it doesn't install it. The script checks for it at `/opt/cisco/anyconnect/bin/vpn` or `/opt/cisco/secureclient/bin/vpn` before doing anything else, and exits with a clear message if neither is found, rather than letting you complete the whole wizard and only fail later at `vpn -c`.

Everything else is handled for you. After you answer the Touch ID question, the script scans what's already installed (skipping `pinentry-touchid` from the scan entirely if you didn't opt into Touch ID), shows you exactly what's already present and what it's about to install — including [Homebrew](https://brew.sh) itself if missing — and asks **one** yes/no to proceed. No per-package prompts.

> Homebrew is the only supported installer here. There isn't a package manager that ships natively with Apple Silicon macOS capable of installing these tools — the realistic alternatives are [MacPorts](https://www.macports.org) or [Nix](https://nixos.org), both third-party like Homebrew. Given this is a personal single-VPN tool, it isn't worth maintaining multiple install backends, so the script standardizes on Homebrew and installs it for you if missing.

---

## 📦 Installation

You can install and configure the CLI tool using either `curl` or by cloning the repository directly.

### Option 1: Quick Install via cURL
Run this single command in your terminal to download and start the setup wizard immediately:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/piyalahmed/cisco-autoconnect/main/setup.sh)"
```
### Option 2: Clone the Repository
If you prefer to review the code or run it locally, you can clone the repository:

```bash
git clone https://github.com/piyalahmed/cisco-autoconnect.git
cd cisco-autoconnect
chmod +x setup.sh
./setup.sh
```

### The Setup Wizard

The wizard asks two yes/no questions — Touch ID, then a single "go ahead and install these?" — up front, then handles the rest without further confirmation prompts:

1. Choosing Touch ID (recommended) or plain password-based encryption.
2. Scanning for existing dependencies (skipping `pinentry-touchid` from the scan if you didn't pick Touch ID), showing what's already there vs. what it's about to install (including Homebrew itself, if missing), and asking to confirm once.
3. Installing everything that was missing, silently, no per-package prompts.
4. Generating a GPG key pair (if you don't already have one) — this one still needs you, since GPG key generation happens interactively in its own terminal.
5. Naming your custom CLI command (default is `vpn`), and entering your VPN Server, Username, Password, and TOTP Secret.
6. Verifying your live 6-digit 2FA code.
7. Binding GPG to your macOS Keychain, then switching to Touch ID if you opted in.

Once complete, the script will output:

> **Setup Complete!**
> Try it out by typing: `vpn -c` or `vpn -h` for help

---

## 💻 Usage

Once installed, you can control your VPN from anywhere in your terminal using the command name you chose during setup (e.g., `vpn`).

| Command | Description |
| --- | --- |
| `vpn -c` | **Connect to the VPN.** Triggers Touch ID, generates your OTP, and connects. |
| `vpn -d` | **Disconnect from the VPN.** |
| `vpn -s` | **Check Status.** Returns `Connected` or `Disconnected`. |
| `vpn -u` | **Update Password.** Safely updates your saved domain password without needing to re-enter your TOTP secret. |
| `vpn -h` | **Help.** Displays the help menu and available flags. |

### Example Workflow

```bash
$ vpn -c
# (Touch ID prompt appears)
Connecting to vpn.yourcompany.com...
Connected successfully.

$ vpn -s
Connected

$ vpn -d
Disconnecting...
Disconnected.

```

---

## 🔄 Updating Your Password

Corporate passwords expire. When you are forced to change your domain password, you do not need to run the entire setup script again. Simply use the update flag:

```bash
vpn -u

```

The tool will ask for your new password, silently rescue your existing TOTP secret, and securely re-encrypt both into a fresh GPG file.

---

## 🗑 Uninstallation

To cleanly remove the generated CLI tool and permanently delete your encrypted credentials from your machine, run `setup.sh` with the `-r` flag followed by your command name (replace `vpn` below if you chose a different name during setup).

### Quick Uninstall via cURL
```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/piyalahmed/cisco-autoconnect/main/setup.sh)" _ -r vpn
```

### From a Local Clone
```bash
./setup.sh -r vpn
```

*This includes safety checks to ensure it only removes files generated by this project.*

---

## 🧪 Testing a Fresh Install

Since this script bootstraps Homebrew, GPG, and Keychain/Touch ID state, a true "fresh machine" test needs a real macOS environment — Docker can't exercise Keychain or Touch ID. Cheapest to most thorough:

1. **Syntax/lint pass** (seconds, no side effects):
   ```bash
   bash -n setup.sh          # syntax check
   brew install shellcheck && shellcheck setup.sh   # static analysis
   ```
2. **Cold-start dependency path**, on your own machine, without touching your real Homebrew: temporarily hide it from `PATH`, e.g. `PATH=$(echo "$PATH" | tr ':' '\n' | grep -v homebrew | paste -sd: -)  ./setup.sh`, and watch it walk through the "Homebrew missing → bootstrap it" branch. Ctrl-C once you've confirmed the branch fires correctly — you don't need to let it actually reinstall Homebrew over itself.
3. **Full fresh-user run**, on real hardware (needed to genuinely exercise the GPG key generation, Keychain, and Touch ID prompts): create a new local admin user (System Settings → Users & Groups → Add Account), fast-user-switch into it, and run the exact curl one-liner from this README. Homebrew itself is machine-wide, so this won't test the "Homebrew isn't installed at all" branch — but it does have no cached GPG keys, no Keychain items, and no `/usr/local/bin/<cmd>`, so everything else behaves like a new teammate's first run.
4. **Scratch VM**, for the most faithful "another machine" test, including the actual "Homebrew isn't installed" bootstrap and Xcode Command Line Tools prompt: spin up a vanilla Apple Silicon macOS VM with [UTM](https://mac.getutm.app) or [Tart](https://github.com/cirruslabs/tart), and run the install one-liner there.
5. Afterward, test `./setup.sh -r <cmd>` to confirm uninstall cleans up correctly, then reinstall to make sure re-running isn't broken by leftover state.

---

## 🔒 Security Model

* **Data at Rest:** Your VPN password and TOTP secret are bundled into a temporary file, encrypted via `gpg` using your personal public key, and saved to `~/.<command_name>-creds.gpg`. The plaintext is then securely wiped from the disk.
* **Data in Transit:** When you connect, the generated bash script pipes the decrypted credentials directly into the Cisco AnyConnect binary via `printf`.
* **Keychain Integration:** The GPG private key passphrase required to decrypt the file is stored securely in the native macOS Keychain, guarded by the `pinentry-touchid` wrapper.