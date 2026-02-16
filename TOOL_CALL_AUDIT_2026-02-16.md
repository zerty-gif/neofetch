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

---

## Second-pass audit (display, GPU, packages)

### Display probing (`get_resolution`)

**Finding:** X11 probing (`xrandr`, `xwininfo`, `xdpyinfo`) is solid, but Wayland-only sessions can report less accurately when relying only on `/sys/class/drm` fallback.

**Change made:**

- Added a Wayland-first probe using `wlr-randr` when `WAYLAND_DISPLAY` is set.
- Parses only modes marked as current.
- Preserves existing `refresh_rate` behavior:
  - `on`: emits `<WxH> @ <Hz>`
  - `off`: emits `<WxH>`

**Impact:** Better resolution fidelity on wlroots-based compositors without affecting existing X11 logic.

### GPU probing (`get_gpu` on Linux)

**Finding:** Linux GPU path assumed `lspci` availability; on minimal installations without `pciutils`, GPU output can be empty.

**Change made:**

- Added guard for missing `lspci`.
- Added fallback to `glxinfo -B` renderer string when available.

**Impact:** Graceful degraded behavior on minimal Linux installs while keeping existing `lspci` parsing path unchanged.

### Package counting (`get_packages`)

**Finding:** Existing DNF optimization uses `/var/cache/dnf/packages.db` (DNF4-specific). DNF5 ecosystems may not provide this path, but script already falls back cleanly to `rpm -qa`.

**Change made:** No code change required in this pass.

**Why no change now:** Current fallback remains correct and stable across RPM-based systems; introducing DNF5-specific fast path would require distro/version-specific handling beyond this focused maintenance patch.

---

## Third-pass update: DNF5 fast path

### What changed

**File:** `neofetch`

- Added a DNF5-specific package counting branch in `get_packages`:
  - `dnf5 repoquery --installed --qf '%{name}' --quiet`
- Kept existing DNF4 optimization (`/var/cache/dnf/packages.db`) as second priority.
- Kept `rpm -qa` fallback as final path.

### Why

- DNF5 deployments may not provide the DNF4 sqlite cache path previously used for fast counts.
- Direct `dnf5 repoquery --installed` is a more coherent primary path on modern Fedora/RHEL-family systems that have moved to DNF5.

### Compatibility behavior

Order now is:

1. `dnf5` installed → use DNF5 query path
2. Else if `dnf` + `sqlite3` + `/var/cache/dnf/packages.db` → use DNF4 sqlite count
3. Else fallback to `rpm -qa`

This preserves backward compatibility while improving correctness on DNF5-based environments.

---

## Additional deprecations/enhancements identified

### 1) `wmic` usage on Windows (deprecation risk)

**Status:** Not changed yet (recommendation).

- Multiple Windows code paths still call `wmic`/`wmic.exe` for OS, model, kernel, battery, GPU and resolution queries.
- WMIC has been deprecated by Microsoft and may be absent/disabled on newer Windows installs.

**Recommended enhancement:** introduce a centralized Windows query helper that prefers PowerShell CIM (`Get-CimInstance`) and falls back to `wmic` when available.

### 2) `gconftool-2` legacy path

**Status:** Hardened in this pass.

- In `Metacity` theme detection, the legacy `gconftool-2` call is now guarded by `type -p gconftool-2`.
- Prevents noisy failures when legacy GNOME2 tooling is absent.

### 3) `ifconfig` fallback behavior

**Status:** Hardened in this pass.

- Linux local-IP fallback now checks for `ifconfig` presence before invoking it.
- Avoids command-not-found stderr on modern systems that only ship `iproute2`.

### 4) Optional future quality improvements

- Add a Windows CIM abstraction layer (single helper function) to reduce duplicated `wmic` handling.
- Expand Wayland resolution support beyond wlroots (`wlr-randr`) to compositor-specific tools where available.
- Add a CI smoke test mode that stubs external tools to validate fallback ordering without platform-specific runtime dependencies.
