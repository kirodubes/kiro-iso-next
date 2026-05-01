# CHANGELOG — kiro-iso-next

> Parallel "next" branch of kiro-iso. Diverges in nanorc config, simplified boot entries, and earlier OOMD integration.
> Daily rebuilds (version bump + mirrorlist only) are grouped.

---

## 2026-04-26 — `v26.04.26.01`
- **Packages added:** 2 new entries
- **Mirrorlist** — cleaned up (3 entries removed)

## 2026-04-16 — Major Overhaul Day

- **Added `OVERVIEW.md`** — full project structure documentation (214 lines)
- **Added `nanorc`** — full nano editor config baked into ISO (349 lines) *(not present in kiro-iso stable)*
- **`README.md`** — major expansion
- **OOMD** — reworked config (`oomd.conf` simplified), enabled `systemd-oomd` service/socket, added memory accounting, slice configs
- **`.bashrc`** — debranded ArcoLinux → edu/kiro aliases, PATH deduplication
- **`ananicy-cpp.service`** — enabled at boot
- **`up.sh`** — initial content added
- **Screenshots** moved to `images/`
- **`volatile-storage.conf`** and **`10-parallel-services.conf`** removed
- **EFI boot entries** simplified to 3 entries (`01-archiso-linux`, `02-nvidianouveau`, `03-nomodeset`)
- **`build-the-iso.sh`** — major rewrite (26 lines changed)

## 2026-04-15 — `v26.04.15.01`
- **`ntpd.service`** symlink removed
- **`10-parallel-services.conf`** added (systemd timeout tuning)
- **Package swaps:** 8 lines updated
- **`loader.conf`** updated

## 2026-04-12 — `v26.04.12.01`
- **Packages:** 11 lines updated (+2 added)
- **Added `kiro-ohmychadwm.jpg`** screenshot
- **`build-the-iso.sh`** — 10 lines reworked
- **README** update

## 2026-04-09 — `v26.04.09.01`
- **Package swap**, mirrorlist cleanup

## 2026-04-05 — `v26.04.05.01`
- **Packages:** 11 lines updated (+2 new)

---

## 2026-03-28
- **Network configs updated:** `20-ethernet.network`, `20-wlan.network`, `20-wwan.network` — simplified addressing
- **`autologin.conf`** updated
- **`automated_script.sh`** — 6 lines reworked
- **Mirrorlist** expanded (+10 entries)
- **Version bump** `v26.03.28.01`

## 2026-03-22 — `v26.03.22.01`
- **`nsswitch.conf`** — updated resolver entries

---

## 2026-02 — Rebuilds
- 2026-02-27 and 2026-02-08 — version bumps + mirrorlist updates

---

## 2026-01-30/31 — Build Script Expansion

- **`build-the-iso.sh`** — grew from ~1 line to 80+ lines across multiple commits (logging, error handling, validation)
- **Mirrorlist** — initial expansion (+27 entries), then later mass prune (350 → 4 entries)
- **`up.sh`** update

---

## 2025-12-09 — Rename `installation-scripts/` → `build-scripts/`

- Folder renamed to match kiro-iso stable convention

## 2025-12-14/21 — `v25.12.14/21`
- **Package added** (Dec 14), package swap (Dec 21)

---

## 2025-11-08/09 — Kernel & Boot Cleanup

- **Removed** `archiso/LICENSE`
- **`mkinitcpio.conf.d/archiso.conf`** — added then removed (simplified)
- **`loopback.cfg`** — content added (12 lines)
- **`bootstrap_packages`** — added
- **`linux.preset`** — kernel preset updated
- **`loader.conf`** updated, timeout tuned

---

## 2025-10-29 — `v25.10.29.01`
- **Package changes** (6 swapped)

---

## 2025-09-28 — Boot Entry Simplification

- **EFI entries** reduced from 5 → 3:
  - Removed: `02-no-nouveau`, `04-nvidianonouveau`, `05-nomodeset`
  - Renamed: `03-nvidianouveau` → `02`, `05-nomodeset` → `03`
- **Added `scdaemon.conf`** for GPG smartcard
- **Packages:** 22 entries removed (significant cleanup)
- **`automated_script.sh`** minor fix

---

## 2025-09-04 — `v25.09.04.01`
- **Mirrorlist** rebuilt, 1 package removed

## 2025-09-01
- **`networkd.conf.d/ipv6-privacy-extensions.conf`** added

---

## 2025-08-21
- **`up.sh`** — rewritten (42 lines changed)

---

## 2025-07-08
- **`pacman.conf`** — added Chaotic-AUR repo block
- **`build-the-iso.sh`** — updated

## 2025-07-03
- **`up.sh`** — full rewrite (29 lines changed)
- **Mirrorlist** — massive purge (173 → 15 entries)

---

## 2025-06-22 — **Initial Commit**

- **Full ISO bootstrapped** from kiro-iso stable snapshot (94 files, 4826 insertions)
- Base differences from kiro-iso at time of fork:
  - Included `pacman.conf.kiro`, `personal_repo/` already baked in
  - Syslinux splash updated
  - `installation-scripts/` folder (later renamed to `build-scripts/`)
  - `kiro` mkinitcpio preset (renamed from `arcolinux`)
  - Removed `arcolinux-release` reference

---

## 2025-06-23/24 — Debranding & Boot Config

- **`arcolinux-release`** removed from airootfs
- **`mkinitcpio.d/arcolinux`** → renamed to `kiro`
- **GRUB** and **syslinux** configs updated (kiro branding)
- **Syslinux splash** updated (`splash-arcolinux.png` added, `splash.png` replaced)
- **Package cleanup**
