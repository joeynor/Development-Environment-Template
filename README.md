# Secure Development Environment Setup
Default environment setup when working in Blockchain Projects.

1. Separate VM for smart contract work
- Use Ubuntu Server 24.04 minimal install.
- Disable shared clipboard and folders with host.
- Enable firewall with `ufw`.
- Install Foundry for smart contract development.
2. Hardware wallet only
- Use Ledger or Trezor for all signing operations.
- No private keys stored locally.
- Connect via Foundry using `--ledger`.
3. Secrets in encrypted vaults (1Password CLI)
- Install 1Password CLI in VM.
- Store all secrets in vault.
- Inject into runtime with `op run`.
4. No .env files with keys
- Enforce via pre-commit hooks and gitleaks.
- Only use ephemeral env vars from 1Password.
5. Extension whitelist only
- Use dedicated browser profile with only essential extensions.
- Blocklist all others via Chromium policy JSON.

Here’s a practical, step-by-step way to **implement** each part, with copy-pasteable snippets and some opinionated defaults that won’t slow you down.

# 0) Quick goals

* Clean separation: your smart-contract toolchain runs in its own locked-down VM.
* **All signing** happens on a hardware wallet.
* **Zero plaintext secrets at rest** (no `.env` files).
* Browser is locked to a **tiny allowlist** of extensions.

---

# 1) Separate VM for smart-contract work

**Pick one:** Ubuntu Server 24.04 LTS (minimal install) on VMware/VirtualBox/UTM/Parallels.
**Create two snapshots:** `base-install` and `toolchain-ready`.

**Hardening checklist (VM):**

* Create a *non-sudo* user for daily work; use `sudo` only when needed.
* Disable clipboard & shared folders with host. Disable drag-and-drop.
* Turn on the firewall and default-deny incoming:

  ```bash
  sudo ufw default deny incoming
  sudo ufw default allow outgoing
  sudo ufw enable
  ```
* Keep packages current:

  ```bash
  sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
  ```
* Install dev basics:

  ```bash
  sudo apt install -y build-essential git curl jq unzip pkg-config libssl-dev
  ```

**Toolchain (recommend Foundry for first-class hardware wallet support):**

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
cargo --version || curl https://sh.rustup.rs -sSf | sh -s -- -y   # if you plan to build extras
```

> Optional: Connect the VM to a private network (host-only) and route internet via your host’s VPN; or use Tailscale/WireGuard only to reach trusted infra.

---

# 2) Hardware wallet only (no hot wallets)

**Ledger works great with Foundry:**

* Install the **Ethereum** app on the Ledger.
* In your VM, install `ledger-udev` rules so the device is reachable (Linux):

  ```bash
  echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="2c97", MODE="0660", GROUP="plugdev", TAG+="uaccess"' | sudo tee /etc/udev/rules.d/20-hw1.rules
  sudo udevadm control --reload-rules && sudo udevadm trigger
  sudo usermod -aG plugdev $USER && newgrp plugdev
  ```

**Use Foundry with Ledger (no private keys in files):**

```bash
# Example: deploy a contract with Ledger signing
forge create src/MyToken.sol:MyToken \
  --rpc-url $RPC_URL \
  --ledger \
  --hd-path "m/44'/60'/0'/0/0"
```

**Verify address before you ship:**

```bash
cast wallet address --ledger --hd-path "m/44'/60'/0'/0/0"
```

**What this gives you:** all tx signing prompts on the Ledger screen; no private keys ever touch disk or memory beyond the device.

> Trezor works similarly (`--trezor`). If you ever need a local signer, use `clef` with a hardware wallet backing; don’t load mnemonic files.

---

# 3) Secrets in encrypted vaults (1Password CLI)

**Install & sign-in (inside the VM):**

```bash
curl -sS https://downloads.1password.com/linux/keys/1password.asc | \
  gpg --dearmor | sudo tee /usr/share/keyrings/1password-archive-keyring.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/1password-archive-keyring.gpg] https://downloads.1password.com/linux/debian/amd64 stable main' | \
  sudo tee /etc/apt/sources.list.d/1password.list
sudo apt update && sudo apt install -y 1password-cli
op account add --address my.1password.com --email you@example.com
op signin    # creates an eval-able session export in your shell
```

**Pattern to inject secrets at run-time (no files):**

* Put secrets in 1Password items, e.g.
  `Ethereum RPC (Alchemy)` → field `HTTP_URL`
  `Etherscan` → field `API_KEY`

* **Run commands with ephemeral env vars:**

  ```bash
  eval "$(op signin)"   # start a session for this shell only

  op run --env \
    --env-var RPC_URL="op://Dev/Ethereum RPC (Alchemy)/HTTP_URL" \
    --env-var ETHERSCAN_API_KEY="op://Dev/Etherscan/API_KEY" \
    -- forge script script/Deploy.s.sol:Deploy \
         --broadcast --ledger --verify
  ```

* **Local scripts can wrap this** so your team runs `./safe.sh forge ...` and never touches secrets.

**Pro tip:** add a shell function `safe()`:

```bash
safe () {
  eval "$(op signin --account my.1password.com --raw >/dev/null 2>&1 || true)"
  op run --env "$@"
}
# usage:
safe --env-var RPC_URL="op://Dev/Alchemy/HTTP_URL" -- forge test
```

---

# 4) No `.env` files with keys, ever

**Enforce it technically + culturally:**

* **.gitignore** still lists `.env` but you won’t use it for secrets—only non-sensitive toggles.

* **Pre-commit hooks** to block commits containing secrets:

  ```bash
  pipx install pre-commit
  pre-commit sample-config > .pre-commit-config.yaml
  # add gitleaks
  curl -sSL https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_linux_x64.tar.gz | tar -xz
  sudo mv gitleaks /usr/local/bin/

  cat <<'YAML' >> .pre-commit-config.yaml
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
  YAML

  pre-commit install
  ```

* **Block runtime access to any `.env` file for deploy/test commands** by *only* supporting the `op run` flow in your package scripts.

Example `package.json` (Hardhat) or `Makefile` (Foundry):

```make
deploy:
	@safe --env-var RPC_URL="op://Dev/Alchemy/HTTP_URL" \
	      -- forge script script/Deploy.s.sol:Deploy --broadcast --ledger
```

---

# 5) Extension whitelist only

**Approach A (simple): dedicated browser profile** inside the VM with *only*:

* 1Password extension (optional; CLI is primary)
* uBlock Origin (security)
* Hardware wallet bridge (if your device needs one; Ledger Live bridge is a desktop app, not a browser extension, so you may not need anything here)
* A trusted contract-dev helper (e.g., Etherscan Assistant) — **avoid wallet extensions** that store keys (MetaMask/Rabby) to stick to your “no hot wallets” rule.

**Approach B (strict via Chromium policies):**

* Create `policies/managed/extension_policy.json`:

  ```json
  {
    "ExtensionInstallWhitelist": [
      "aebbglbbacdhloejeddjmbfobidgbolc",  // 1Password (example ID)
      "cjpalhdlnbpafiamejdnhcphjbkeiagm"   // uBlock Origin
    ],
    "ExtensionInstallBlocklist": ["*"]
  }
  ```
* Place it under:

  * Debian/Ubuntu: `/etc/chromium/policies/managed/extension_policy.json`
  * For Google Chrome: `/etc/opt/chrome/policies/managed/extension_policy.json`
* Restart browser; only the allow-listed IDs can be installed. (Find an extension’s ID on its store page URL.)

**Browser hygiene:**

* Disable password saving & autofill.
* Disable “Allow sites to check if you have payment methods.”
* Turn off WebUSB/WebHID for general browsing; enable only when using Ledger (or run dev and general browsing in separate profiles).

---

## Putting it together: minimal Foundry project with safe deploy

**Scaffold:**

```bash
forge init my-project && cd my-project
```

**Simple deploy script** `script/Deploy.s.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/MyToken.sol";

contract Deploy is Script {
    function run() external {
        vm.startBroadcast();
        new MyToken("My Token", "MTK", 18);
        vm.stopBroadcast();
    }
}
```

**Deployment (Ledger + 1Password, no .env):**

```bash
safe --env-var RPC_URL="op://Dev/Alchemy/HTTP_URL" \
  -- forge script script/Deploy.s.sol:Deploy \
     --rpc-url "$${RPC_URL}" \
     --broadcast \
     --ledger \
     --hd-path "m/44'/60'/0'/0/0"
```

**Verification (no API key files):**

```bash
safe --env-var ETHERSCAN_API_KEY="op://Dev/Etherscan/API_KEY" \
  -- forge verify-contract <DEPLOYED_ADDRESS> src/MyToken.sol:MyToken --chain 1
```

---

## Team workflows & guardrails (recommended)

* **CODEOWNERS** require review from at least one “security owner” for anything touching deploy scripts.
* **CI** runs `gitleaks` and fails on any secret.
* **Branch protections** prevent direct pushes to `main`.
* **Release tags** are built in CI and **only** deployable via a **manual Ledger-signed step** from the VM.

---

## Recovery & backups

* Hardware wallet seed phrase: store with **Shamir backup** (if supported) across two sealed locations.
* 1Password: ensure **emergency kit** printed & stored offline.
* VM: export **encrypted** VM backups after the `toolchain-ready` snapshot; keep a separate **dev-data** disk for repo clones so you can rotate the OS quickly.
