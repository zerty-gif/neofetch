# Neofetch tool-call audit (2026-02-16)

This document records updates made after checking runtime command usage in `neofetch` against current Linux/Proxmox conventions and online references.

## Scope audited

Focused on command calls and data sources most affected by system/library evolution:

- Public IP retrieval (`get_public_ip`, `public_ip_host`)
- Local IP retrieval (`get_local_ip`)
- Distro identity source precedence (`get_distro`, `os-release` parsing)
- Proxmox detection (`pveversion`, `/etc/pve`) for current PVE behavior

## External references checked

- `ident.me` supports HTTPS and plain-text endpoint at `https://ident.me`.
- `os-release(5)` specifies `/etc/os-release` precedence over `/usr/lib/os-release`.
- Proxmox command-line docs and examples still use `pveversion` output shaped like `pve-manager/<version>/<build>`.

## Changes made

### 1) Public IP host default moved to HTTPS

**File:** `neofetch`

- Changed default:
  - From: `public_ip_host="http://ident.me"`
  - To: `public_ip_host="https://ident.me"`
- Updated nearby comment default text accordingly.

**Why:** modern default transport hardening; endpoint supports HTTPS.

---

### 2) Fixed `os-release` precedence order

**File:** `neofetch`

- In distro detection source loop, changed source priority to:
  1. `/etc/os-release`
  2. `/usr/lib/os-release`
  3. `/etc/openwrt_release`
  4. `/etc/lsb-release`

**Why:** aligns with `os-release(5)` precedence and avoids older `lsb-release` metadata overriding canonical OS identity.

---

### 3) Hardened local IP detection for modern systems

**File:** `neofetch`

- Added `ip` command existence checks before using it.
- Switched auto-route probe to IPv4-safe route check:
  - `ip -4 route get 1.1.1.1`
- Switched interface lookup to IPv4-specific address query:
  - `ip -4 addr show <interface>`
- Kept existing `ifconfig` fallback unchanged.

**Why:** reduces noisy failures when `ip` is absent and improves deterministic IPv4 extraction on mixed IPv4/IPv6 hosts.

## Proxmox path status

No change required in current Proxmox distro detection logic:

- `pveversion` command check remains valid.
- Fallback check for `/etc/pve` remains compatible.
- Existing parse of `pveversion` output is still coherent with current output format.

## Validation performed

- Script edit completed with targeted patching.
- Behavior remains backward-compatible with existing fallback chain (`ifconfig`, `curl`, `wget`, etc.).

## Follow-up options

If needed, we can do a second-pass audit for additional command paths (`xrandr`, `xdpyinfo`, package managers, GPU probing utilities) and add matrix-style compatibility notes by distro family.
