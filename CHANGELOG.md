# CHANGELOG

> Complete history of the KIRO ISO project — newest first. Each entry explains not just what changed, but why it was done and what benefit it brings. Daily rebuilds (version bump + mirrorlist refresh only) are grouped into a single line.

---

## 2026-06-06 — One-command build (`./build.sh`) + host-prep extracted to a sourced helper

**What Changed**
- Added **`build.sh`** at the repo root as the single entry point for building the ISO. It is a thin, template-conformant wrapper that hands off to **`build-scripts/build-the-iso.sh`**, so the command is identical on every machine: **`./build.sh`**. A builder no longer needs to know the internal script layout or `cd` into `build-scripts/`.
- Extracted the host-preparation helpers (**`ensure_package`**, **`setup_chaotic`**, and the new **`setup_cachyos`**) out of `build-the-iso.sh` into a new **`build-scripts/host-prep.sh`**, which `build-the-iso.sh` now **sources**. `host-prep.sh` is a function-only library (no `main()`) with a load-once guard, keeping all "make the host ready to build" logic in one place.
- Wired **`setup_cachyos`** into `main()` alongside `setup_chaotic`. It trusts the CachyOS signing key, enables the `[cachyos]` CDN77 geo-mirror in `/etc/pacman.conf` if absent, and installs `cachyos-keyring` + `cachyos-mirrorlist` — idempotently (already-configured hosts are detected and skipped).

**Why**
- The build pulls `linux-cachyos` (the default live kernel) from `[cachyos]`, and `prepopulate_keyring` runs `pacman-key --populate cachyos`; a host lacking the cachyos keyring/mirrorlist fails the build. Folding that prep into the sourced helper makes the build **self-contained on any Arch-based host** (Arch, Kiro, EndeavourOS, CachyOS, Garuda) with no manual setup.
- Splitting host-prep from the build pipeline keeps each file focused — `build-the-iso.sh` is the pipeline, `host-prep.sh` is the environment — and means the same one command (`./build.sh`) works everywhere, matching the "users never need to know the internal layout" goal.

**Files Modified**
- `build.sh` (new)
- `build-scripts/host-prep.sh` (new)
- `build-scripts/build-the-iso.sh`

## 2026-06-06 — Reorganize `packages.x86_64` into risk tiers + drop paid app (community-ISO hygiene)

**What Changed**
- Reorganized **`archiso/packages.x86_64`** from repo-origin grouping into **three risk tiers**, grouped by function within each tier, with banner comments that make the "never remove" packages unmistakable:
  - **TIER 1 — FROZEN**: archiso base (preserved verbatim in upstream order), archiso-extra, graphics/xorg, NVIDIA, build essentials, repo keyrings + mirrorlists, installer/Calamares, live guest tools, display manager, default session.
  - **TIER 2 — KIRO CORE**: audio, bluetooth, network, file-management, printing, thumbnails, desktop integration, theming engines, system tuning, Ohmychadwm runtime, shell/terminal, kiro-* branding, icons/cursors/themes, fonts.
  - **TIER 3 — USER-CHANGEABLE**: browsers, media, editors, CLI utilities, system info, package tools, archive tools, misc extras.
- Removed **`spotify`** — a paid streaming app has no place pushed onto users of a community ISO.
- Pruned the trailing commented-out "uncomment-to-enable" optionals (`#flat-remix`, `#colloid-cursors-git`, `#dex`, `#ckb-next-git`, `#discord`, `#telegram-desktop`, `#tlp`) — personal-taste suggestions that belong in the arcolinux-nemesis post-install scripts, not seeded as hints on the ISO.

**Why**
- The list had grown with no signal telling a builder *which packages are safe to remove without breaking the build/boot/install* and which are load-bearing. The tiering makes the blast radius explicit and pushes the freely-editable apps to the end.
- Kiro is now a **community** ISO, not a personal one — paid apps (spotify) and personal optionals come off; users make up their own mind and **`sublime-text-4`** stays (the unlicensed nag is liveable) as does **`claude-code`** (free to install).
- The reorg is **build-safe**: `build-the-iso.sh` operates on a copy and only needs `nvidia-*` and `linux-cachyos`/`-headers` as plain column-0 lines (verified intact); pacman ignores order. A token-set diff confirmed the reshuffle lost/toggled **zero** packages before the paid-app removal.
- Net: **396 active** packages (was 397), 36 commented (was 43, all upstream archiso + xf86-video stubs kept). Structure mirrored to production **`kiro-iso`** is a follow-up once validated here.

**Files Modified**
- `archiso/packages.x86_64`

---

## 2026-06-04 — Ship the wedge-fixed `-nemesis` Plymouth theme (encrypted-boot LUKS card)

**What Changed**
- **`archiso/packages.x86_64`**: swapped **`plymouth-theme-kiro-logo`** → **`plymouth-theme-kiro-logo-nemesis`** under the PLYMOUTH section.

**Why**
- The graphical LUKS passphrase **card** (dark box + Kiro-blue border, lock icon, typed characters as blue dots, "enter passphrase" label) and the corrected self-assembling-K **wedge timing** live only in the **`-nemesis`** variant. Production `plymouth-theme-kiro-logo` (`26.06-01`) still has the old slow wedge — the green never appears inside the brief splash window — and lacks the card images, so a fresh `-next` install would have rendered the old prompt. Shipping `-nemesis` is what makes a fresh encrypted install actually exercise the fix that was proven (hand-patched) on the Kiro-next VM.
- `-nemesis` is published in **nemesis_repo** (`26.06-08`, db-indexed) with the wedge fix (`FADE_END=8 / SLIDE_BEG=3 / SLIDE_END=18`) and all five card images (`box`/`bullet`/`lock`/`logo-body`/`logo-wedge`). It `conflicts` (not `replaces`) the original, so this is an opt-in swap on the **beta** ISO only — production `kiro-iso` is untouched.
- Pairs with `kiro-calamares-config-next`'s `useSystemdHook: true`: `sd-encrypt` asks via `systemd-ask-password`, which Plymouth's built-in password agent renders using this theme. Plymouth reaches the install target via `unpackfs` (this very package), so stock `detect_plymouth()` adds the `plymouth` hook **before** `initcpiocfg` runs — no `kiro_plymouth`, no `plymouthcfg` needed (the package `.install` runs `plymouth-set-default-theme kiro-logo`).
- Ships in **v26.06.04**. Next: build the `-next` ISO, then run a fresh **encrypted** install *and* a plain **unencrypted** install (the systemd-hook switch affects all installs and drops the busybox recovery shell) to confirm end-to-end.

**Files Modified**
- `archiso/packages.x86_64`

---

## 2026-05-31 — Three NVIDIA boot options: proprietary modern / proprietary auto-detect (mirrored from production)

**What Changed**
- Mirrors the production kiro-iso change: the NVIDIA boot entry splits into **"NVIDIA proprietary, modern"** (`driver=nonfree`, keeps the baked `nvidia-open-dkms`, no chwd) and a new **"NVIDIA proprietary, auto-detect"** (`driver=nonfreechwd`, runs chwd), across systemd-boot / GRUB / syslinux. Open entry unchanged (`driver=free`).

**Why**
- A chwd-free express lane to the baked driver for modern Turing+ GPUs; chwd stays for older cards. Driver-mode logic lives in [kiro-calamares-config-next](../kiro-calamares-config-next).

**Files Modified**
- `archiso/efiboot/loader/entries/02-nvidianouveau.conf` (relabel) + `02b-nvidiachwd.conf` (new)
- `archiso/grub/grub.cfg`, `archiso/syslinux/archiso_sys-linux.cfg`

## 2026-05-29 — Sync committed skel `.bashrc` with the renamed kiro-* helpers

**What Changed**

Updated the committed **[archiso/airootfs/etc/skel/.bashrc](archiso/airootfs/etc/skel/.bashrc)** so its aliases point at the renamed `kiro-*` helper scripts instead of the old `edu-*` names (and dropped the dead `rvariety`/`rkmix`/`rconky` aliases, whose `edu-remove-*` scripts were removed, not renamed). Purely a sync change — at build time `build-the-iso.sh` fetches the live `.bashrc-latest` from `erikdubois/edu-shells` into the build tree, so the *shipped* `.bashrc` always comes from edu-shells. The point of this edit is the **Phase 2c consistency check** (`files_are_identical` against the local edu-shells `.bashrc-latest`): without it, that check would print NOK once edu-shells is pushed with the renamed aliases. Real fix lives in (and must be pushed from) `edu-shells`.

**Technical Details**

- Renames (old → new): `edu-which-vga` → `kiro-which-vga`; `edu-fix-pacman-databases-and-keys` → `kiro-fix-pacman-keys` (7 alias variants); `edu-fix-pacman-conf` → `kiro-fix-pacman-conf`; `edu-fix-pacman-gpg-conf` → `kiro-fix-gpg-conf`; `edu-fix-archlinux-servers` → `kiro-fix-mirrors`; `edu-probe` → `kiro-probe`. File still parses clean (`bash -n`).

**Files Modified**

- [archiso/airootfs/etc/skel/.bashrc](archiso/airootfs/etc/skel/.bashrc)

---

## 2026-05-29 — Dark Calamares installer: ship KiroDark Kvantum theme

**What Changed**

The Calamares installer now renders **dark** (navy + sky-blue, matching the website) instead of the light-grey default. The ISO ships a custom **KiroDark** Kvantum theme for root, which the installer (run as root via pkexec) picks up.

**Technical Details**

- New: `airootfs/root/.config/Kvantum/KiroDark/{KiroDark.kvconfig,KiroDark.svg}` — KiroDark theme (ArcDark remapped to Kiro's navy/sky-blue palette, fully opaque, white button text).
- New: `airootfs/root/.config/Kvantum/kvantum.kvconfig` → `theme=KiroDark`.
- `packages.x86_64`: added `kvantum` (qt6 style plugin) explicitly — it was only present as a dependency of `kvantum-qt5`; Calamares is Qt6 and needs the qt6 Kvantum style, so this makes it a first-class requirement.
- Paired with `kiro-calamares-config-next` (dark branding) and the KIRO-PKG-BUILD `calamares-next` launcher change (`-style kvantum`).

**Files Modified**
- `archiso/airootfs/root/.config/Kvantum/KiroDark/KiroDark.kvconfig` (new)
- `archiso/airootfs/root/.config/Kvantum/KiroDark/KiroDark.svg` (new)
- `archiso/airootfs/root/.config/Kvantum/kvantum.kvconfig` (new)
- `archiso/packages.x86_64`

---

## 2026-05-29 — Fix: cachyos repo "unknown trust" on the live ISO

**What Changed**

The freshly-enabled `[cachyos]` repo failed every `pacman -Sy` on the live ISO with a `signature from "CachyOS ..." is unknown trust` error, aborting the whole sync (and breaking `kiro-fix-pacman-keys`, which couldn't install `archlinux-keyring`). The previous push enabled the repo and shipped `cachyos-keyring`/`cachyos-mirrorlist`, but never populated the cachyos key into the prebuilt live keyring — so the repo was untrusted from first boot.

**Technical Details**

- **[build-scripts/build-the-iso.sh](build-scripts/build-the-iso.sh)** — `prepopulate_keyring()` now also runs `pacman-key --populate cachyos`, alongside the existing `archlinux` and `chaotic` populates. Installing `cachyos-keyring` only drops `cachyos.gpg` into `/usr/share/pacman/keyrings/`; it does not sign the key into `/etc/pacman.d/gnupg`, which is built once at ISO-build time — so the key must be populated here, exactly as chaotic is. Build host already carries the key material.
- **`SigLevel` deliberately left off `[cachyos]`** — it inherits `Required DatabaseOptional`, matching chaotic. With the key now populated this is the secure, correct config; not dropped to `SigLevel = Never`.
- Added the missing trailing newline to both **[archiso/airootfs/etc/pacman.conf](archiso/airootfs/etc/pacman.conf)** and **[archiso/pacman.conf](archiso/pacman.conf)**.

**Files Modified**

- [build-scripts/build-the-iso.sh](build-scripts/build-the-iso.sh)
- [archiso/airootfs/etc/pacman.conf](archiso/airootfs/etc/pacman.conf)
- [archiso/pacman.conf](archiso/pacman.conf)

---

## 2026-05-29 — Live ISO boot now shows the K splash

**What Changed**

Made the `kiro-logo` Plymouth splash render during the **live ISO boot** (mirrored from production `kiro-iso`). The theme already shipped on the ISO and showed on installed systems, but never drew at live boot because the live environment lacked the two prerequisites Plymouth needs: the `plymouth` initramfs hook and the `splash` kernel parameter.

**Technical Details**

- **[archiso/airootfs/etc/mkinitcpio.conf](archiso/airootfs/etc/mkinitcpio.conf)** — added the `plymouth` hook after `udev` in the live `HOOKS` line (KMS hook already present; `mkarchiso` rebuilds the initramfs at build time).
- **`quiet splash`** added to the KMS boot entries across all three bootloaders — Plymouth needs both tokens or it falls back to the text theme:
  - systemd-boot (**[archiso/efiboot/loader/entries/](archiso/efiboot/loader/entries/)**): `01-archiso-linux`, `02-nvidianouveau`, `04-fallback-zen` (had `quiet loglevel=3`, so only `splash` added).
  - GRUB (**[archiso/grub/grub.cfg](archiso/grub/grub.cfg)**) and syslinux (**[archiso/syslinux/archiso_sys-linux.cfg](archiso/syslinux/archiso_sys-linux.cfg)**): had neither, so `quiet splash` appended to the free / NVIDIA / zen-fallback entries.
- **`03-nomodeset` left untouched** in every bootloader — no KMS means Plymouth can't render, so the safe-graphics fallback stays bare and verbose.
- Theme selection unchanged (`kiro-logo`, set at build time by the package `.install`).

**Files Modified**

- [archiso/airootfs/etc/mkinitcpio.conf](archiso/airootfs/etc/mkinitcpio.conf)
- [archiso/efiboot/loader/entries/01-archiso-linux.conf](archiso/efiboot/loader/entries/01-archiso-linux.conf)
- [archiso/efiboot/loader/entries/02-nvidianouveau.conf](archiso/efiboot/loader/entries/02-nvidianouveau.conf)
- [archiso/efiboot/loader/entries/04-fallback-zen.conf](archiso/efiboot/loader/entries/04-fallback-zen.conf)
- [archiso/grub/grub.cfg](archiso/grub/grub.cfg)
- [archiso/syslinux/archiso_sys-linux.cfg](archiso/syslinux/archiso_sys-linux.cfg)

---

## 2026-05-28 — Hardware-aware install via **chwd**: package list prepared (paired with `kiro-calamares-config-next`)

First step of the install-time driver-selection experiment. The companion change in [kiro-calamares-config-next](../kiro-calamares-config-next/) adds a Calamares Python module that runs `chwd --autoconfigure` inside the chroot — this repo's job is to make sure every package chwd might want to install, and every firmware/microcode/detection helper it depends on, is already present on the live ISO.

### What Changed

Four edits to **[archiso/packages.x86_64](archiso/packages.x86_64)** to sync this repo with production's hardware-detection baseline:

- **Enabled `b43-fwcutter`** (was commented) — Broadcom B43/B43legacy firmware extractor. Older Broadcom Wi-Fi chipsets need the firmware unpacked at runtime; without this, those laptops have no wireless on first boot.
- **Enabled `broadcom-wl-dkms`** (was commented) — Broadcom proprietary `wl` kernel module (DKMS). Covers the BCM43xx generations that need the closed-source blob rather than the open `b43` driver. chwd has a dedicated `broadcom-wl` profile (priority 1) that expects this package to be available.
- **Added `chwd`** — CachyOS's [Hardware Detection Tool](https://github.com/CachyOS/chwd) (Rust, GPL-3.0). Pulled from `nemesis_repo`. Inspects PCI/USB hardware, matches it against TOML profiles, and installs the right driver bundle. Same tool the live ISO will use during install and that users can re-run post-install (`sudo chwd -a`) after kernel or hardware changes.
- **Added `hwdetect`** — console hardware-detect helper (CachyOS ships it on their live ISO too). Useful for diagnostics from the live env, complements `hwinfo` / `inxi` / `hw-probe`.

### Why

Kiro previously chose the NVIDIA driver **at build time** via the `nvidia_driver` variable in `build-the-iso.sh` (`open` / `580xx` / `390xx`) — one ISO per choice. With chwd, the live ISO can ship a single sensible default (`nvidia-open-dkms`, modern GPUs) and **chwd picks the right variant at install time** from the detected device IDs. Same ISO covers Turing+ (nvidia-open), Maxwell/Pascal (proprietary 580xx), and older legacy hardware (470xx). Hybrid graphics laptops automatically get the `.prime` variant with `switcheroo-control` + RTD3 wiring.

The existing GRUB-menu `driver=free|nonfree` choice (handled by `kiro_remove_nvidia`) is preserved: chwd only runs on `driver=nonfree`. Users who explicitly pick the open-source path at boot still get nouveau, untouched.

### Validation pre-merge

Smoke-tested on the dev host (`hq`, Intel iGPU): `chwd-arch-git` from AUR (shorin fork) detected the Intel GPU and matched the `intel` profile (priority 4) and `fallback` (priority 3) — exactly what upstream's TOML says it should pick. Bare `chwd` (no args) only lists matches; `chwd -a` triggers actual install. The companion `chwd-kernel` binary correctly enumerated every kernel in the enabled repos (linux-cachyos, linux-zen, hardened, rt, lts variants).

### Architecture decisions

- **`build-the-iso.sh` `inject_nvidia_packages` left as-is for now.** The default `nvidia_driver=open` keeps `nvidia-open-dkms` baked into the live ISO so NVIDIA hardware boots cleanly into the installer. chwd swaps in the proper variant during install if needed. The per-build switch will be retired in a follow-up once chwd's selection is validated in the field.
- **`kiro_remove_nvidia` kept.** It owns the `driver=free` cleanup path; chwd owns the `driver=nonfree` smart-pick path. Complementary, no overlap.

### Pairs With

- [kiro-calamares-config-next](../kiro-calamares-config-next/) — new `chwd` Python module in `usr/lib/calamares/modules/chwd/` and corresponding entry in `settings.conf`'s exec sequence (between `kiro_remove_nvidia` and `initcpiocfg`).

**Files Modified**

- **[archiso/packages.x86_64](archiso/packages.x86_64)** — uncommented `b43-fwcutter` (line 5), uncommented `broadcom-wl-dkms` (line 11), added `chwd` and `hwdetect`.

---

## 2026-05-28 — Live-boot fallback kernel: `linux-zen` entries added to UEFI / BIOS-syslinux / GRUB menus (synced from production)

Mirror of the same-date `kiro-iso` change. A 4th menu entry, **"fallback kernel linux-zen"**, was added to each live boot menu so a user whose hardware refuses `linux-cachyos` can pick `linux-zen` at the boot screen rather than being stranded before Calamares. New file [archiso/efiboot/loader/entries/04-fallback-zen.conf](archiso/efiboot/loader/entries/04-fallback-zen.conf) (UEFI, sort-key 04); new `LABEL arch_fallback_zen` block in [archiso/syslinux/archiso_sys-linux.cfg](archiso/syslinux/archiso_sys-linux.cfg); new `menuentry id='kirofallback'` in [archiso/grub/grub.cfg](archiso/grub/grub.cfg), all wrapped in `# >>> KIRO_ZEN_FALLBACK_BEGIN/END <<<` markers. `apply_kernel()` in [build-scripts/build-the-iso.sh](build-scripts/build-the-iso.sh) gained a strip-step that deletes the UEFI file and sed-removes the marker-wrapped blocks when `linux-zen` isn't in the user's `kernel=` list — keeps the build robust to non-default kernel selections without leaving broken boot entries. Pattern informed by CachyOS's own live ISO (their main kernel + cachyos-lts fallback in the same menu).

---

## 2026-05-28 — Default kernel: `linux-lqx` → `linux-cachyos` (synced from production)

Mirror of the same-date `kiro-iso` change. Both `build-scripts/build-the-iso.sh` (lines 369-370) and every load-bearing archiso template were updated: `KERNEL_CANDIDATES` dropped `linux-lqx`, `CANONICAL_KERNEL` set to `linux-cachyos`, and all boot/initramfs templates (3 efiboot entries, 2 syslinux configs, 2 grub configs, 2 mkinitcpio.d presets, plus `packages.x86_64`) now reference `linux-cachyos`. Cachyos variants (`-bore`, `-lts`, `-rc`) continue to be discovered dynamically from the enabled repos at picker time. `LIQUORIX.md` is retained as a historical record with a banner noting the switch. The previous bug — picker pre-selecting `linux-lqx` and the builder's auto-rewrite (line 526) being a no-op for canonical picks, so default-path ISOs shipped lqx unchanged — is fixed by aligning canonical with the cachyos decision.

---

## 2026-05-28 — squashfs compression L6 → L3 (synced from production)

Mirror of the same-date `kiro-iso` change. `archiso/profiledef.sh` (line 19) now uses `-Xcompression-level 3` instead of `6`, with the old L6 line preserved as a commented fallback right above the active line. Trade: ~3.4% ISO size growth (5.9 → 6.1 GB) for faster squashfs decompression during the Calamares `unpackfs` phase. Validated on the production ISO already — unpackfs at ~2 min 13 s on a VirtualBox VM, install total ~3 min 10 s.

The beta-only divergence in `profiledef.sh` (`iso_name="kiro-next"`, `iso_label="kiro-next-v26.05.28"`) is preserved — only the compression line was touched.

**Files Modified**
- `archiso/profiledef.sh`

---

## 2026-05-28 — launcher trust moved out of airootfs; kernel selector hardening

Mirrors the production `kiro-iso` change of the same date.

- **Launcher trust out of airootfs.** The autostart helper shipped `644` (the overlay doesn't preserve git's `100755`), so the "Untrusted application launcher" prompt persisted. Helper **removed from airootfs** (`archiso/airootfs/usr/local/bin/kiro-trust-desktop-launchers` deleted); trust now ships from the **`calamares-next`/`calamares` package** as a systemd **user** service — `ExecStart=/bin/bash …` (exec-bit-proof), `WantedBy=default.target` (XFCE doesn't activate `graphical-session.target`).
- **Kernel selector hardening** in `build-the-iso.sh`: strict `picker=` validation (bad value errors with the valid set), kernel-name validation (typo errors with the valid names), `detect_available_kernels` skipped for a fixed kernel (only two local-DB lookups), and `auto` resolves to **dialog-first**.

**Files Modified**

- **[build-scripts/build-the-iso.sh](./build-scripts/build-the-iso.sh)** — picker/kernel validation, no scan on fixed kernel, `auto` dialog-first.
- **archiso/airootfs/usr/local/bin/kiro-trust-desktop-launchers** — removed.

## 2026-05-27 — kernel selector: `picker=` toggle + broader dynamic discovery

Two refinements to the `kernel="ask"` selector:

**1. `picker=` config var** (`auto` | `gum` | `dialog`). `auto` (default) uses gum if installed, else dialog; set it explicitly to force one. Previously the choice was implicit (`command -v gum`).

**2. Broader, fully-dynamic kernel discovery.** Offers every kernel in the **first four families** the repos provide, discovered dynamically: the mainstream set plus all **CachyOS**, **XanMod**, and **pinned-LTS** flavors. Static candidates are just the mainstream names (`linux`, `-lts`, `-zen`, `-hardened`, `-rt`, `-rt-lts`, `-lqx`, `-mainline`); families matched via `^(linux-cachyos|linux-xanmod|linux-lts[0-9])`.

**What we deliberately leave out, and why.** CPU-microarch builds (`linux-x64v2/v3/v4`, `linux-znver2…5`) and niche kernels (`linux-cjktty`, `-nitrous`, `-tachyon`, `-vfio`) are **excluded by design**: low demand, and the microarch ones are **dangerous on a general ISO — they silently fail to boot on the wrong CPU level** (`x64v4` needs AVX-512, `znver5` needs Zen 5). A user can still set `kernel="linux-znver4"` directly.

| Bucket | Kernels | Offered? |
|---|---|---|
| Mainstream | `linux`, `-lts`, `-zen`, `-hardened`, `-rt`, `-rt-lts`, `-lqx`, `-mainline` | ✅ |
| CachyOS | `linux-cachyos`, `-bore`, `-lts`, `-rc` | ✅ |
| XanMod | `linux-xanmod-lts`, `-rt`, `-x64v2`, `-x64v3`, `-edge-x64v3` | ✅ |
| LTS pins | `linux-lts515`, `-lts61`, `-lts66`, `-lts612` | ✅ |
| CPU-microarch | `linux-x64v2/v3/v4`, `linux-znver2…5` | ❌ won't boot on the wrong CPU |
| Niche | `linux-cjktty`, `-nitrous`, `-tachyon`, `-vfio(-lts)` | ❌ low demand |

**Files Modified**

- **build-scripts/build-the-iso.sh** — `picker=` var; `KERNEL_CANDIDATES` trimmed to mainstream + `linux-mainline`; dynamic grep widened to CachyOS/XanMod/LTS-pins; picker-aware dispatch.

## 2026-05-27 — kernel selector: gum picker (truecolor Arc Dark) with dialog fallback

`select_kernels()` now prefers **`gum`** for the `kernel="ask"` picker, falling back to **`dialog`** when gum isn't installed. gum renders **truecolor**, so it hits the exact Arc Dark palette the dialog theme could only approximate: blue accent `#5294e2`, text `#d3dae3`, muted header `#8b9bb4`. Refactored into `_select_kernels_gum` (`gum choose --no-limit` + a second `gum choose` for the live-boot kernel) and `_select_kernels_dialog` (existing checklist/radiolist, unchanged); the parent runs `detect_available_kernels` once and dispatches on `command -v gum`. gum is host-only (not in the ISO), which is fine — the selector runs host-side at build time.

**Files Modified**

- **build-scripts/build-the-iso.sh** — split `select_kernels` into gum + dialog backends.

## 2026-05-27 — live ISO: pre-trust the "Install kiro" launcher

Mirror of the `kiro-iso` fix. The live desktop's **Install kiro** launcher triggered XFCE/Thunar's "Untrusted application launcher" prompt before Calamares would start. Added a live-session autostart — **`/usr/local/bin/kiro-trust-desktop-launchers`** via **`~/.config/autostart/trust-desktop-launchers.desktop`** (liveuser only) — that sets `metadata::trusted` + the XFCE `metadata::xfce-exe-checksum` (computed at runtime) on `~/Desktop/*.desktop` at login, so the launcher opens straight into Calamares. Confirmed on the live VM (Thunar 4.20.8). Live-session scope only; installed systems unaffected.

**Files Modified**

- **archiso/airootfs/usr/local/bin/kiro-trust-desktop-launchers** (new)
- **archiso/airootfs/home/liveuser/.config/autostart/trust-desktop-launchers.desktop** (new)

## 2026-05-27 — kernel selector: build any kernel(s) into the ISO

`build-the-iso.sh` no longer hardcodes `linux-lqx`. A new `kernel=` config var (default `linux-lqx`; set to `ask` for an interactive **`dialog`** checklist) lets you build the ISO with **any kernel(s)** the enabled repos offer — single or multiple. This pairs with the new `kiro_kernel` Calamares module (`kiro-calamares-config-next`), which installs whatever kernel(s) the ISO ships; together the whole pipeline (live ISO + installed system) is kernel-agnostic from one selection point.

**How it works.** `select_kernels()` detects available kernels by checking a candidate list (`linux`, `-lts`, `-zen`, `-hardened`, `-rt`, `-rt-lts`, `-lqx`, `-cachyos`) plus **every `linux-cachyos*` flavor dynamically** — CachyOS kernels topped our benchmark study, so all flavors are exposed and discovered at runtime rather than hardcoded. Only kernels with a matching `-headers` are offered (DKMS NVIDIA needs them). When multiple are picked, a second `dialog` chooses which one the **live ISO boots** (the "primary"). `apply_kernel()` then rewrites the **build-tree** copies (not the repo): all selected kernels + `-headers` into `packages.x86_64`, and the primary into the boot entries (`efiboot`/`syslinux`/`grub`) and the live presets (`kiro`, `linux.preset`). The repo keeps `linux-lqx` as its canonical default, mirroring the existing `inject_nvidia_packages()` pattern. The selector runs **host-only** (terminal-native `dialog`, so it works over SSH/tty).

**Files Modified**

- **`build-scripts/build-the-iso.sh`** — `kernel=` config var; `detect_available_kernels()`, `select_kernels()`, `apply_kernel()`; wired into `main()` + `show_overview`.

## 2026-05-26 — cups: airootfs trimmed to socket-only

Mirror of the production `kiro-iso` fix. The live ISO airootfs enabled CUPS three different ways: **`sockets.target.wants/cups.socket`**, **`printer.target.wants/cups.service`**, and **`multi-user.target.wants/cups.path`**. The service and path symlinks were redundant — socket activation alone starts `cupsd` on demand when a client opens the print socket. Removed **`printer.target.wants/cups.service`** and **`multi-user.target.wants/cups.path`** (and the now-empty `printer.target.wants/` directory), leaving only **`cups.socket`**.

**Why this matters.** These airootfs symlinks only affect the *live* session — they are not carried into the installed system, where service enablement is driven by Calamares. Printing was off after a fresh install + reboot. The matching fix lives in **`kiro-calamares-config-next`**, which now explicitly enables **`cups.socket`** (socket activation only) on the installed system. Socket-only everywhere keeps live and installed behaviour consistent.

**Files modified.**
- `archiso/airootfs/etc/systemd/system/printer.target.wants/cups.service` (removed)
- `archiso/airootfs/etc/systemd/system/multi-user.target.wants/cups.path` (removed)

## 2026-05-26 — README: community framing + "development" not "experimental"

Same de-"personal" reword as `kiro-iso`: the overview now leads with Kiro as a **community Arch-based Linux distribution**, this repo as its **development** ISO builder (the `-next` track). Per a new HQ convention, the `-next` track is described as "development", never "experimental". Both rules codified in [Kiro-HQ/ASSISTANT.md](../../Insync/Kiro/Kiro-HQ/ASSISTANT.md). README only — no build artifacts affected, no rebuild needed.

## 2026-05-21 — LIQUORIX.md synced from stable + filename uppercase

**What changed.** `liquorix.md` (lowercase) renamed to [LIQUORIX.md](./LIQUORIX.md) (uppercase) to match the top-level-md UPPERCASE filename convention used across Kiro / EDU / KIRO repos. Content overwritten with the user-facing rewrite from `kiro-iso/LIQUORIX.md` — the two files are now byte-identical.

**Why.** Stable promoted the kernel switch and rewrote the doc to reflect "we shipped this" instead of "should we?" Per the ECOSYSTEM cascade rule, any stable change that should land in both ISO tracks is applied to both — so `-next` gets the same doc to keep `kiro-iso` ↔ `kiro-iso-next` parity for shared assets.

**Files modified.**
- [LIQUORIX.md](./LIQUORIX.md) (renamed from `liquorix.md` + content rewritten)

## 2026-05-18 — TODO housekeeping

Short session. No code changed — this was a pure status-tracking pass after earlier build and boot testing.

**BIOS/syslinux boot path verified.** The syslinux configs had been updated for `linux-lqx` in a previous session but only UEFI (GRUB + systemd-boot in VirtualBox) had been confirmed working. BIOS boot was tested and confirmed good. Moved from Backlog to Done.

**PipeWire status confirmed.** The PipeWire stack was marked "Needs build + audio test" — now confirmed verified working.

**Remaining open item:** NVIDIA `driver=nonfree` boot + DKMS compile against `linux-lqx-headers` on real NVIDIA hardware. Only remaining Backlog item.

**Files Modified:** `TODO.md`

---

## 2026-05-18 — `v26.05.18.01`

### ISO audit: VirtualBox installed-system verification + audit.sh

**Build script fix — `isoLabel` missing `next`.** The checksum phase at the end of `build-the-iso.sh` was constructing `isoLabel="kiro-${kiroVersion}-x86_64.iso"` but `mkarchiso` produces filenames from `iso_name` in `profiledef.sh`, which is `kiro-next`. The mismatch caused sha1sum/sha256sum/md5sum to fail with "No such file or directory" on every build. Fixed to `isoLabel="kiro-next-${kiroVersion}-x86_64.iso"`.

**`audit.sh` — installed system health checker.** A comprehensive `audit.sh` script was written and committed to the repo root (also synced to `edu-system-files/usr/local/bin/`). It SSHes into or runs locally on an installed Kiro system and checks 63+ conditions across: kernel (`linux-lqx`), microcode (correct vendor, wrong one removed), mkinitcpio hooks (no archiso hook, microcode/kms present), audio stack (PipeWire complete, pulseaudio absent), all 4 Calamares module results (`kiro_before`, `kiro_final`, `kiro_remove_nvidia`, `kiro_ucode`), pacman repos, desktop session files, SDDM theme, user groups, systemd services, key file permissions, NVIDIA handling, bootloader, and `pacman -Qk` package integrity. Results are grouped as PASS / WARN / FAIL with a summary count. Designed to be extended month-by-month.

**VirtualBox audit findings (v26.05.18.01, UEFI, Intel VirtualBox):**
- 63 PASS — all core functionality verified working
- 1 WARN — `/etc/calamares/` config dir left on system (explained by the FAIL below)
- 1 FAIL — `kiro-calamares-config-next` still installed; `kiro_final`'s final removal step ran `pacman -R --noconfirm kiro-calamares-config-next` inside a `try/except` that swallows the failure — the package has no dependencies and is manually removable, but the silent failure means it wasn't cleaned up at install time
- Firmware warnings during build (`softing_cs`, `lantiq_gswip`, `adf7242`) are benign — ultra-niche hardware with no firmware in any Arch package; harmless and unfixable without blacklisting modules
- `pacman -Qk` exceptions: `ohmychadwm-git` (makepkg cleans build artifacts), `bind`/`cups`/`nfs-utils` (config files created only when services are first used) — all whitelisted in audit.sh

**Files Modified:** `build-scripts/build-the-iso.sh`, `audit.sh` (new)

---

### edu-chadwm dropped; README accuracy overhaul

**`edu-chadwm` removed going forward.** The package `edu-chadwm-git` was already commented out in `archiso/packages.x86_64`, but references to it persisted in `build-scripts/build-the-iso.sh` (the `desktop` label variable), `CLAUDE.md`, and `README.md`. All forward-facing references have been cleaned up. CHANGELOG historical entries were left intact — they accurately describe what the ISO shipped at the time.

**README rewritten for accuracy.** A full audit revealed several stale or incorrect entries:

- `enable-oomd.sh` and `disable-oomd.sh` were referenced in the project tree and Key Scripts section but do not exist in the repo — removed
- `personal_repo/` was listed as a root-level directory — it does not exist; the relevant comment is in `archiso/pacman.conf` — removed
- `packages.bootstrap` was listed with the wrong name; the actual file is `bootstrap_packages` — corrected
- `setup.sh`, `change-version.sh`, `up.sh`, and `CHANGELOG.md` were missing from the project tree — added
- The Building KIRO section omitted the required first step (`change-version.sh`) and made no mention of the NVIDIA driver selection knob — both added
- "Based on the ArcoLinux project" in the Overview — ArcoLinux branding reference removed
- The stale "Recent Changes" section (listing Calamares migrations from months ago) replaced with a link to `CHANGELOG.md`
- ArcoLinux tutorial link removed from Resources
- `✅` emoji bullets and the `🖖` sign-off removed throughout

**Files Modified:** `build-scripts/build-the-iso.sh`, `CLAUDE.md`, `README.md`

---

### Build script standardization — full template conformance pass

All four build scripts were audited against the project standard template (modelled on `up.sh`) and brought into full conformance. This was a correctness and maintainability pass, not cosmetic cleanup — several of the changes fix real failure modes that were silently swallowed before.

#### `build-scripts/build-the-iso.sh`

The most significant rewrite. The old script had `set -e` only, meaning unset variable references and failed pipe segments would silently continue and corrupt the build in hard-to-diagnose ways. It also had no error trap, so a failing phase gave no indication of *where* it failed.

The new version adds `set -euo pipefail` and the standard `on_error` trap that prints the failing line number and command. Beyond that:

- **`SCRIPT_DIR` / `REPO_DIR`** replace the hand-rolled `installed_dir="$(dirname)/.."` pattern. All file paths are now anchored to the script's location, so the build works correctly regardless of which directory you call it from.
- **`check_not_root()`** hard-aborts if run as root. The old version only printed a warning and continued — a user who missed the message would proceed to build as root, which `mkarchiso` handles poorly.
- **`wget` failure guard** — the old code fetched `.bashrc` from edu-shells with no failure check. If the download failed (network blip, GitHub down), the build would continue with whatever stale content was in skel. Now a failed download aborts with a clear error.
- **Safe skel cleanup** — `rm -rf skel/.*` was replaced with `find -mindepth 1 -delete`. The `.*` glob can expand to include `.` or `..` on some systems, which would be catastrophic.
- **Config block at the top** — `nvidia_driver`, `clean_pacman_cache`, and `remove_build_folder` are now gathered at the top of the file before any functions. Previously these knobs were scattered through 490 lines; now they're the first thing you see when you open the file.
- **Named phase functions** — each build phase is now a function (`prepare_build_tree`, `prepopulate_keyring`, `inject_nvidia_packages`, etc.) called from `main()`. This makes the high-level flow immediately readable and allows individual phases to be tested in isolation.
- **Removed dead code** — `archisoRequiredVersion="archiso 84-1"` was declared but never checked anywhere in the script. Removed.
- **TTY-safe colors** — raw `tput setaf` calls had no `[[ -t 1 ]]` guard. If the script was ever piped or redirected, the escape codes would corrupt the output. The new colors block falls back to empty strings when stdout is not a terminal.
- **Startup `sleep` calls removed** — there were `sleep 2` and `sleep 3` calls at startup that served no purpose. The BTRFS countdown (10 seconds with CTRL+C prompt) was intentionally kept — that one gives the user a real chance to abort.
- **Phase numbering fixed** — the old script had phases 1, 2, 3, 4, 4b, 5, 7, 8, 9 (Phase 6 missing entirely, 4b awkward). Phases are now sequential 1–9.

#### `change-version.sh`

Added `set -euo pipefail`, the standard header, `SCRIPT_DIR`, TTY-safe colors, log functions, and `on_error` trap. Previously, if any `sed` call silently failed (e.g. a regex didn't match because a file format changed), the version bump would partially update some files and leave others stale — and the script would exit 0. Now any failure aborts immediately and reports the line. All paths anchored to `SCRIPT_DIR` so the script works from any working directory. Dead commented-out debug lines removed. Logic wrapped in `bump_version()` inside `main()`.

#### `build-scripts/get-pacman-repos-keys-and-mirrors.sh`

**Critical fix:** the `pacman.conf` copy used `new_conf="pacman.conf"` — a bare filename resolved against `$PWD`. If `build-the-iso.sh` called this script (which it does, via `bash "$SCRIPT_DIR/get-pacman-repos-keys-and-mirrors.sh"`), the working directory at call time is the repo root, not `build-scripts/`. The copy would fail or source the wrong file. Fixed to `"${SCRIPT_DIR}/pacman.conf"`. Also brought into full template conformance with standard header, colors, log functions, and `on_error` trap.

#### `build-scripts/install-yay-or-paru.sh`

The yay and paru install branches were identical except for the package name and URL — a straight copy-paste. Collapsed into a single `install_aur_helper name url` function. Added `/tmp` cleanup after `makepkg` (the original left the tarball and source directory behind). Full template conformance.

#### `archiso/airootfs/etc/dev-rel`

`ISO_CODENAME` was still set to `arconet - kiro` — a leftover ArcoLinux branding reference. Changed to `kiro`.

---

**Files Modified:** `build-scripts/build-the-iso.sh`, `build-scripts/get-pacman-repos-keys-and-mirrors.sh`, `build-scripts/install-yay-or-paru.sh`, `change-version.sh`, `archiso/airootfs/etc/dev-rel`, `TODO.md` (created stub)

---

## 2026-05-01 — `v26.05.01.01`
- **Version bump** + mirrorlist refresh

## 2026-04-30 — `v26.04.30.01`

- **Version bump** + mirrorlist refresh — removed one stale mirror entry to keep the list clean and reduce the chance of hitting a dead server on first boot

## 2026-04-29 — `v26.04.29.01`
- **Version bump** + mirrorlist refresh

---

## 2026-04-28 — `v26.04.28.01`

### `up.sh` — maintenance improvements

Two new lines were added to **`up.sh`**, the daily ISO maintenance helper script. This script is what drives the version bump + mirrorlist cycle that keeps every ISO build fresh and reproducible.

---

## 2026-04-26 — `v26.04.26.01`

### Script renamed: `setup-git-v5.sh` → `setup.sh`

The developer environment bootstrap script was renamed from **`setup-git-v5.sh`** to the simpler **`setup.sh`**. The old name carried an explicit version number in the filename, which is an anti-pattern — the version is already tracked by git. The new name is cleaner, easier to type, and makes it obvious what the script does without implying it is just one in a long series of sequential versions.

### Mirrorlist cleanup

Two mirror entries were removed from the embedded mirrorlist. Stale or unreliable mirrors slow down the first `pacman -Syu` run on a freshly booted live system, so keeping the list curated is worth the small maintenance cost.

---

## 2026-04-25 — `v26.04.25.01`

### Package added: `capitaine-cursors`

**`capitaine-cursors`** is a clean, modern X11 cursor theme inspired by macOS. Adding it to the ISO ensures that every desktop environment — XFCE4, ohmychadwm, and edu-chadwm — ships with a polished, HiDPI-aware cursor out of the box, rather than falling back to the default X11 arrow. This is a small quality-of-life detail that significantly improves the first-impression polish of the live session.

---

## 2026-04-20 — `v26.04.20.01`

### Enabled `systemd-resolved` as a DNS resolver

Four systemd symlinks were added to enable **`systemd-resolved`** at boot:

- `dbus-org.freedesktop.resolve1.service` — exposes the resolver on D-Bus so applications can query it via the standard API
- `systemd-resolved-monitor.socket` — allows runtime monitoring of DNS state
- `systemd-resolved-varlink.socket` — the modern varlink IPC socket used by newer tools
- `systemd-resolved.service` (under `sysinit.target.wants`) — starts the resolver early in boot

**Why this matters:** `systemd-resolved` is the recommended DNS resolver for systemd-based systems. It provides automatic mDNS (Avahi-style local hostname resolution), DNSSEC validation, DNS-over-TLS support, and proper integration with VPNs and per-interface DNS settings. Without it enabled, the live system falls back to basic `/etc/resolv.conf` parsing, which can cause subtle failures on networks with mDNS hostnames or split-horizon DNS. This change pairs with the `nsswitch.conf` update made on 2026-03-22 (which set the host resolution order to `files mymachines mdns_minimal [NOTFOUND=return] resolve dns wins myhostname`) to create a fully modern DNS stack that works reliably on home networks, office environments, and bare-metal servers alike.

---

## 2026-04-19 — `v26.04.19.01`

### New packages: `edu-powermenu-git`, `edu-system-files-git`, `cpuid`

- **`edu-powermenu-git`** — adds the KIRO/edu branded power menu (shutdown, reboot, suspend, lock) that integrates consistently with all three desktop environments. Previously users had to reach into a terminal or use desktop-specific logout dialogs; this gives a single consistent entry point regardless of which WM is active.

- **`edu-system-files-git`** — pulls in the curated set of system configuration files maintained in the edu ecosystem. These cover sensible defaults for things like font rendering, GTK theming, locale settings, and input handling. Shipping them through a package (rather than baking raw config files into the ISO airootfs) means they can be updated independently via `pacman -Syu` without requiring a full ISO rebuild.

- **`cpuid`** — a command-line tool that decodes the CPU identification registers and reports detailed processor information (family, model, features, cache topology). Useful for hardware debugging, virtualization compatibility checks, and verifying that CPU feature flags needed for specific workloads are actually present. Particularly valuable on a live ISO where users may be running it on unfamiliar hardware.

### Desktop label updated

The ISO desktop label in **`build-the-iso.sh`** was updated from `xfce4/chadwm` to `xfce4/edu-chadwm/ohmychadwm`. This accurately reflects the three desktop environments that ship in the ISO and makes it immediately clear to anyone reading the build output (or examining the ISO metadata) what they are getting.

---

## 2026-04-17 — Mirror URL fix in `get-pacman-repos-keys-and-mirrors.sh`

The Chaotic-AUR mirror URL inside the installation script was updated to point to the current active endpoint. Mirror URLs for Chaotic-AUR have changed over the project's lifetime as the infrastructure evolved; using a stale URL causes the Chaotic-AUR key import and repository setup to fail silently or with a confusing error, which blocks the entire installation workflow for users who want AUR packages. Keeping this URL current is maintenance work that directly affects the user's first-boot experience.

---

## 2026-04-16 — OOMD, Shell Debranding, and Documentation Day

This was a dense day of work with multiple distinct themes. Seven commits landed.

### systemd Out-of-Memory Daemon (OOMD) — fully integrated

The live ISO now ships with **`systemd-oomd`** enabled and configured. OOMD is systemd's built-in out-of-memory killer, and unlike the kernel's OOM killer — which is a last resort that can freeze a system for tens of seconds before acting — OOMD monitors memory pressure proactively at the cgroup level and kills the heaviest-consuming processes before the system becomes completely unresponsive.

The following was added:

- **`archiso/airootfs/etc/systemd/oomd.conf`** — global OOMD configuration tuned for desktop workloads: swap usage threshold and memory pressure thresholds set to intervene before the system locks up
- **`system.slice.d/oomd.conf`** — applies OOMD monitoring to all system-level services, so a runaway daemon doesn't take the entire system down
- **`user.slice.d/oomd.conf`** — applies OOMD monitoring to the user session, so a memory-hungry browser or desktop app triggers a clean kill rather than a kernel panic cascade
- **`system.conf.d/memory-accounting.conf`** — enables per-cgroup memory accounting, which is a prerequisite for OOMD to work; without this, OOMD cannot observe per-process memory usage

The service and socket symlinks (`dbus-org.freedesktop.oom1.service`, `systemd-oomd.service`, `systemd-oomd.socket`) were added to ensure OOMD starts automatically on boot.

During the day, both an `enable-oomd.sh` and `disable-oomd.sh` helper script were created and then removed. The initial plan was to provide opt-in scripts for post-install systems, but the right approach turned out to be integrating OOMD directly into the ISO configuration so every boot has it active without any user intervention. The scripts were folded into the static config and deleted.

### `.bashrc` — ArcoLinux debranding and shell hygiene

The default **`/etc/skel/.bashrc`** that every new user inherits received a significant cleanup pass, completing the transition away from the ArcoLinux branding that was present in the original base.

**What was removed:**

- `alias toboot`, `togrub`, `torefind` — these called `arcolinux-toboot`, `arcolinux-togrub`, `arcolinux-torefind`, scripts that do not exist in a KIRO system
- `alias vbm` — called `arcolinux-vbox-share`, an ArcoLinux-specific VirtualBox helper
- `alias rvariety`, `rkmix`, `rconky` — called ArcoLinux removal scripts; replaced `rvariety` with the edu equivalent `edu-remove-variety`
- `alias whichvga` — updated from `arcolinux-which-vga` to `edu-which-vga`
- `alias narcomirrorlist` — replaced with `alias nchaoticmirrorlist` pointing to the Chaotic-AUR mirrorlist, which is actually present on the system
- `alias iso`, `isoo` — these printed ArcoLinux version info; removed entirely since the KIRO version is in `/etc/dev-rel`
- `alias vbm` — ArcoLinux VirtualBox mounting helper, not applicable

**What was added:**

- `alias u="sudo pacman -Syu"` — a short, memorable shortcut for the most common maintenance operation
- `alias neo="neofetch"` — quick system info display
- `alias npicom="$EDITOR ~/.config/arco-chadwm/picom/picom.conf"` — quick editor access to the picom compositor config, useful for chadwm users tuning their compositor
- `alias nchaoticmirrorlist="sudo $EDITOR /etc/pacman.d/chaotic-mirrorlist"` — quick access to edit the Chaotic-AUR mirrorlist
- `### EDU-SHELLS` section header — organizes the file to match the structure used in the edu-shells package

**PATH deduplication fix:**

The old `~/.bashrc` used naive `PATH="$HOME/.bin:$PATH"` assignments to add local directories to `PATH`. If `.bashrc` is sourced more than once (which happens in nested shells, tmux, and some login scenarios), these assignments duplicate the same directory in `PATH` repeatedly. The fix uses the standard `case ":$PATH:" in *":$dir:"*` guard pattern, which is a well-known shell idiom that only appends the directory if it is not already present. This prevents PATH from ballooning with repeated entries and avoids subtle issues where the wrong version of a tool might be picked up due to a duplicated and reordered PATH.

### Documentation — `OVERVIEW.md` added, `README.md` expanded

A new **`OVERVIEW.md`** file was added (214 lines) with a complete structural breakdown of the repository: what each directory contains, how the build system works, which services are enabled by default, and how the three desktop environments relate to each other. This is intended as a quick-start reference for anyone contributing to the project or trying to understand the ISO without having to read every config file individually.

**`README.md`** was nearly tripled in size (from ~90 lines to ~370 lines), adding detailed sections on:

- Prerequisites and build steps
- What each package category includes and why
- How to customize the package list
- Service topology (which services are enabled and what they do)

### Screenshots reorganized into `images/`

The four screenshot images (`kiro-chadwm.jpg`, `kiro-ohmychadwm.jpg`, `kiro-xfce.jpg`, `kiro.jpg`) were moved from the repository root into a dedicated **`images/`** subfolder, and the `README.md` image references were updated accordingly. This is a housekeeping change that keeps the root of the repository clean — a flat root directory with a mix of scripts, configs, and image files makes it hard to quickly find what you are looking for.

---

## 2026-04-15 — PCI Latency, Optimization Config Separation

### System optimization configs moved to `edu-dot-files`

Several systemd drop-in config files that were previously baked directly into the ISO airootfs were removed:

- `systemd/journald.conf.d/volatile-storage.conf`
- `systemd/system.conf.d/10-parallel-services.conf` (at this point)

These configs are now delivered by the **`edu-dot-files-git`** package instead. This is an important architectural decision: configs that live inside the ISO can only be updated by rebuilding and redistributing the ISO, which is a multi-hundred-megabyte operation. Configs delivered by a package can be updated with a simple `pacman -Syu`. Moving non-ISO-critical configuration out of the airootfs and into the dotfiles package reduces the ISO size slightly and means users always get the latest tuning without waiting for a new ISO release.

### PCI Latency optimization — added and removed in the same day

A **`pci-latency`** script was added (`/usr/local/bin/pci-latency`, 56 lines) along with a `pci-latency.service` systemd unit that runs it at boot. The script reads each PCI device's latency timer register and sets it to an optimal value, which can reduce audio crackling under load and improve I/O responsiveness, particularly on older hardware and systems with multiple PCI peripherals competing for bus time.

Later the same day, the script and service were removed from the ISO. The decision was made to keep PCI latency tuning in the external dotfiles (`edu-dot-files`) rather than the ISO configuration, for the same reason as above: it is a user-facing optimization rather than something required for the live session to function. Users who want it can install the dotfiles package. This keeps the ISO lean and focused on boot-critical configuration only.

### `ananicy-cpp.service` enabled

**Ananicy-cpp** (Another Auto NICe daemon, C++ rewrite) is a process scheduler that automatically adjusts process priorities and I/O scheduling classes based on a curated rules database. Enabling it at boot via a symlink means the live session immediately benefits from better CPU scheduling: interactive applications like browsers and terminals get higher priority, build tools and background processes get lower priority, and the system feels more responsive under mixed workloads. This pairs with `cachyos-ananicy-rules-git` (already in the package list) which provides the extensive rules database.

### `profile.d/userbin.sh` — `~/.local/bin` in PATH at login

A small `profile.d` script was added to ensure `~/.local/bin` is present in `PATH` for all login sessions. This is where pip, pipx, cargo, and other language-specific installers place user-owned executables. Without this, tools installed to `~/.local/bin` are invisible to the shell unless the user manually adds the path, which is a common source of confusion on Arch-based systems where the default shell config is minimal.

---

## 2026-04-14 — Power Management Iteration, Nanorc, Boot Config

This day involved several commits that explored and then refined the power management configuration.

### Power management tuning

The power management stack was iterated through several states:

1. **`tlp` removed, `tuned` added** — TLP (Laptop Power Saving) was replaced by `tuned`, a daemon from Red Hat/Fedora that uses profiles to tune system performance vs. power tradeoffs. Unlike TLP, which is focused primarily on laptops and batteries, `tuned` works equally well on desktops and servers, making it a better fit for a general-purpose ISO. `upower` was added at the same time — it provides a D-Bus API for battery and power state that desktop environments use to show charge level and trigger suspend.

2. **CPU governor config added** — `cpupower` was added with a config file (`/etc/default/cpupower`) setting `governor='performance'`, which keeps the CPU at maximum frequency. For a live ISO used for testing and installation, maximum performance is generally preferable over power saving.

3. **`cpupower` and `tuned` removed** — After testing, both were removed from the package list. The conclusion was that for a live ISO session, the kernel's default scheduler and governor behavior is sufficient, and adding power management daemons introduces complexity without clear benefit in a short-lived session. `alsa-utils` was also removed (ALSA is handled via the higher-level PipeWire/PulseAudio stack already present). `ntp` was removed in favor of `systemd-timesyncd` which is already part of systemd.

### `archlinux-tweak-tool` upgraded to GTK4

The **`archlinux-tweak-tool-git`** package was replaced with **`archlinux-tweak-tool-gtk4-git`**. The GTK4 version is the actively maintained branch; the GTK3 version is legacy. This ensures the tweak tool works correctly under modern GTK theme configurations and is compatible with the libadwaita-based theming that newer GTK4 applications use.

### `10-parallel-services.conf` — systemd timeout tuning

A new drop-in config at `system.conf.d/10-parallel-services.conf` was added with two settings:

```ini
[Manager]
DefaultTimeoutStopSec=10s
DefaultTimeoutAbortSec=5s
```

The default `DefaultTimeoutStopSec` is 90 seconds — meaning systemd will wait up to 90 seconds for a service to stop before killing it. On a live ISO, this makes shutdown feel extremely slow if any service hangs. Reducing it to 10 seconds means a clean shutdown completes in well under 30 seconds total even with misbehaving services. `DefaultTimeoutAbortSec=5s` similarly limits the time given to services that are force-killed before systemd gives up entirely.

### `nanorc` — syntax highlighting for the default editor

A comprehensive **`nanorc`** configuration file (349 lines) was added to the ISO's `/etc/nanorc`. The default nano on Arch Linux ships with minimal syntax highlighting; this config enables color-coded syntax for: shell scripts, Python, C/C++, Makefiles, INI files, systemd unit files, pacman config files, and several other formats. Since `nano` is the default editor in the ISO (set in `.bashrc`), this means users editing config files during installation get a readable, color-coded experience rather than plain monochrome text.

---

## 2026-03-27 — systemd-networkd: Type-Based Interface Matching

The network configuration files in `archiso/airootfs/etc/systemd/network/` were updated to use type-based interface matching instead of name-based glob matching:

**Before:**

- `20-ethernet.network`: matched `Name=en*` and `Name=eth*`
- `20-wlan.network`: matched `Name=wl*`
- `20-wwan.network`: matched `Name=ww*`

**After:**

- `20-ethernet.network`: matches `Type=ether` with `Kind=!*` to exclude virtual interfaces
- `20-wlan.network`: matches `Type=wlan`
- `20-wwan.network`: matches `Type=wwan`

**Why this matters:** Predictable Network Interface Names (the `en*`/`wl*` prefix convention) are not guaranteed. On some systems, particularly with USB ethernet adapters, VM guests, or exotic hardware, interface names may not follow the `en`/`wl` convention. By matching on `Type=` instead of `Name=`, the network configuration works correctly on any hardware regardless of what the kernel chose to name the interface. The `Kind=!*` filter on the ethernet rule excludes virtual ethernet interfaces (veth, bridge members, etc.) which should not be managed by the live session's network config — this was an existing issue noted in the previous config via a comment referencing Arch bug #70892.

---

## 2026-03-22 — `nsswitch.conf` — Host Resolution Order Fixed

The Name Service Switch configuration (`/etc/nsswitch.conf`) was updated to change the `hosts:` line:

**Before:** `hosts: mymachines resolve [!UNAVAIL=return] files dns mdns wins myhostname`

**After:** `hosts: files mymachines mdns_minimal [NOTFOUND=return] resolve dns wins myhostname`

**Why this matters:** The original order put `resolve` (systemd-resolved) before `files`, meaning `/etc/hosts` was not consulted first. This caused two problems: (1) local hostname overrides in `/etc/hosts` were ignored, which is unexpected behavior; (2) on systems where `systemd-resolved` is not yet started, host lookups could time out instead of falling back gracefully. The new order matches the recommended Arch Linux configuration: `files` first (so `/etc/hosts` always wins), then `mymachines` (for systemd container hostnames), then `mdns_minimal` with `[NOTFOUND=return]` (mDNS for `.local` hostnames, with early exit to avoid false positives), then `resolve` (systemd-resolved for everything else), then `dns` (direct DNS as a fallback).

---

## 2026-03-14 — Package added: `lxappearance`

**`lxappearance`** is the GTK theme, icon, and font configuration tool from the LXDE project. While KIRO uses XFCE4's settings manager for the primary desktop, `lxappearance` is indispensable for configuring GTK appearance in the tiling window manager environments (ohmychadwm, edu-chadwm) where there is no XFCE settings daemon running. Without it, users of those WMs would have no GUI way to change the GTK theme or cursor, and would need to edit `~/.config/gtk-3.0/settings.ini` by hand.

---

## 2025-12-26 — `up.sh` Major Rewrite

The **`up.sh`** daily maintenance script — used to bump the version, refresh the mirrorlist, and prepare each new ISO build — was rewritten from scratch with significantly better engineering:

**Before:** Basic bash script with `set -eo pipefail`, inline code, and a simple toggle variable for mirrorlist fetching.

**After:**

- `#!/usr/bin/env bash` shebang instead of `#!/bin/bash` — more portable and respects the user's PATH
- `set -Eeuo pipefail` — the `-E` flag ensures ERR traps are inherited by functions and subshells, `-u` treats unset variables as errors (catches typos in variable names), together making the script fail fast and visibly rather than silently producing wrong results
- Dedicated helper functions: `die()`, `info()`, `ensure_paths()`, `write_static_mirrorlist()` — replacing inline code with named functions makes the script readable and testable
- Configurable connection timeouts (`CONNECT_TIMEOUT=5`, `MAX_TIME=20`, `RETRIES=3`) — instead of letting curl hang indefinitely on a slow mirror, these limits ensure the script fails predictably if the network is unavailable
- `trap cleanup EXIT` — a cleanup handler that removes temporary files even if the script exits with an error, preventing stale temp files from accumulating

This rewrite makes the daily build process more reliable, particularly in CI-like environments or when the network is flaky.

---

## 2025-12-21 — Removed `nvidia-dkms` from package list

**`nvidia-dkms`** was removed from `packages.x86_64`. The DKMS version of the NVIDIA driver requires the kernel headers and a build toolchain at install time, and rebuilds the kernel module every time the kernel updates. On a live ISO, this is inappropriate: the ISO cannot know which NVIDIA driver will match the user's card, the build process is slow, and DKMS modules built in the live session do not persist to the installed system. Users with NVIDIA hardware should install the appropriate driver (either `nvidia` or `nvidia-dkms`) after installation via the KIRO hardware detection tooling. The open-source `nouveau` driver (handled via mesa) remains available for basic display output during the live session.

---

## 2026-04-09 — Application Layer Expansion

### New user-facing applications

A substantial set of new packages was added to bring the ISO closer to a complete daily-driver environment:

- **`gcolor3`** — a modern GTK3 color picker with hex, RGB, and HSL output. Useful for design work, theming, and web development. A basic but frequently-needed tool that was absent.
- **`hw-probe`** — uploads hardware probe data to the Linux Hardware Database (`linux-hardware.org`), helping the community track hardware compatibility. Also useful locally as a `lshw`-style diagnostic tool. (Note: an initial typo `hwprobe` was corrected to `hw-probe` in a follow-up commit.)
- **`resources`** — a GNOME-style system monitor with per-process CPU, memory, GPU, and network usage. A modern alternative to the aging `gnome-system-monitor` and more informative than plain `htop` for desktop users.
- **`signal-desktop`** + **`signal-in-tray`** — Signal, the end-to-end encrypted messaging application. Including it in the ISO signals (no pun intended) that privacy is a priority. `signal-in-tray` adds a system tray icon so Signal can run in the background without occupying a taskbar slot.
- **`shortwave`** — an internet radio player with a searchable station database. A lightweight application for background music during work sessions.
- **`spotify`** — the desktop Spotify client for music streaming. While not open-source, it is one of the most commonly requested applications on Linux and including it avoids users having to go through the AUR manually after installation.

### `archlinux-logout` upgraded to GTK4

**`archlinux-logout-git`** was replaced with **`archlinux-logout-gtk4-git`**, the actively maintained GTK4 port of the ArcoLinux logout dialog. The GTK3 version is no longer developed. The GTK4 version is visually identical but built on the modern toolkit, ensuring compatibility with current GNOME and GTK theming systems.

### Build script improvements

Two fixes landed in **`build-the-iso.sh`**:

1. **`set -e` re-enabled** — the `set -e` flag (exit on error) had been commented out with `#set -e`. This meant build failures could be silently swallowed and the script would continue in a broken state, potentially producing a corrupt ISO. Re-enabling it makes the build fail loudly and immediately when something goes wrong.

2. **`installed_dir` path detection fixed** — the previous method used `dirname $(readlink -f $(basename pwd))`, which is unreliable: `basename pwd` just returns the directory name without a path, so `readlink -f` was resolving relative to the current directory in an unpredictable way. The replacement `"$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/.."` is a standard idiom that correctly resolves the script's parent directory regardless of how or from where the script is invoked.

---

## 2026-04-05 — Window Manager Package Additions

Several packages needed by the tiling window manager environments (ohmychadwm, edu-chadwm) were added:

- **`fastcompmgr-git`** — a lightweight X11 compositor forked from `compton`. While `picom` is used for the main chadwm setup, `fastcompmgr` is an alternative that some chadwm configurations prefer for its lower overhead on older hardware.
- **`maim`** — a screenshot tool designed as a modern replacement for `scrot`. It supports region selection, window selection, and piping output to other tools. Used by chadwm keybindings for quick screenshots.
- **`octopi`** — a Qt5 graphical frontend for pacman. Provides a package manager GUI for users who prefer not to use the terminal for package operations. Important for the live ISO where new users may be evaluating the system.
- **`redshift`** — adjusts the screen's color temperature based on time of day (warmer/orange at night, neutral during the day). Reduces eye strain during extended sessions. Unlike `f.lux`, Redshift is fully open-source and integrates cleanly with both X11 and Wayland.
- **`xautolock`** — automatically locks the screen after a configurable idle timeout. The chadwm setups use `xautolock` + `i3lock` (or similar) to implement session locking, since there is no desktop environment managing this automatically.
- **`xclip`** — a command-line clipboard interface. Used heavily in chadwm dotfiles for copy/paste operations in scripts (e.g., copying a color hex code from `gcolor3` into a config file).
- **`autorandr`** was removed — it was present in the chadwm package section but is not used by any of the window manager configurations in the ISO. `autorandr` is a tool for automatically applying monitor layout profiles, a function that is handled by `arandr` (which is already in the list) for interactive use and by `xrandr` scripts for programmatic use.

---

## 2025-06-19 — Personal Repository Support

A local **`personal_repo`** infrastructure was added for hosting custom or private packages that should not go into the public Chaotic-AUR:

- **`pacman.conf`** — a `[personal_repo]` section was added, pointing to a local database file. This allows the ISO build system to install packages from a local repo during the build, without those packages needing to be available on the internet.
- **`updaterepo.sh`** — a helper script that rebuilds the local repository database using `repo-add`. Run after adding a new `.pkg.tar.zst` to the repo directory.
- **`kiro-dummy-git`** — a placeholder package used to test that the personal repo infrastructure is working before real packages are added.
- Initial database and files binaries included to bootstrap the repo structure.

This feature allows KIRO-specific packages (branding assets, configuration packages, proprietary binaries) to be installed without publishing them to a public repository.

---

## 2025-06-17 — Installation Scripts Refactor

The scripts used to set up Chaotic-AUR on a freshly installed system were rewritten and consolidated:

### `get-pacman-repos-keys-and-mirrors.sh` — complete rewrite

The new script replaced both the old `get-the-keys-and-mirrors-chaotic-aur.sh` and `get-the-keys-and-mirrors-arcolinux.sh` with a single, unified script. Key improvements:

- **ANSI color output** — progress steps, warnings, and errors are now color-coded, making it immediately obvious when something goes wrong vs. completing successfully
- **`set -euo pipefail`** — the script now fails fast on any error rather than continuing in a broken state. The `-u` flag catches undefined variable references (typos in variable names), and `pipefail` ensures errors in piped commands are not masked
- **Dynamic Chaotic-AUR package URL fetching** — instead of hardcoding the URL of the Chaotic-AUR keyring and mirrorlist packages, the script now fetches the current package URL dynamically from the Chaotic-AUR CDN. This means the script continues to work even when the Chaotic-AUR team updates their package versioning scheme
- **Error handling** — each major step (key import, mirrorlist installation, pacman.conf editing) now has explicit error handling and user-readable failure messages

### `install-yay-or-paru.sh` added

A new script for bootstrapping an AUR helper (either `yay` or `paru`) was added. AUR helpers are not available in the official Arch repositories, so installing one requires a manual `git clone` + `makepkg` process. This script automates that process, detecting which helper the user prefers and handling the bootstrap from scratch.

### `pacman.conf` added to installation scripts

A template **`pacman.conf`** was added to the `installation-scripts/` directory for use as a reference during post-install setup, pre-configured with the Chaotic-AUR repository block.

---

## 2025-05-29 — ArcoLinux Cleanup and Simplification

A focused cleanup pass removed ArcoLinux-specific infrastructure that was no longer needed:

### Scripts removed

- **`arcolinux-snapper`** — ArcoLinux's BTRFS snapshot helper. KIRO does not mandate BTRFS, so this script was unused and confusing to have present.
- **Installation flag files** (`chaotics-repo`, `no-chaotics-repo`, `personal-repo`) — these were marker files used by the ArcoLinux build system to conditionally include repositories. The KIRO build system handles this differently, and these files served no function.

### `pacman.conf` cleaned up

The ISO's embedded `pacman.conf` had several commented-out sections referencing the ArcoLinux and Kiro package repositories from earlier development iterations. These were removed, leaving only the active repository configuration (Chaotic-AUR + optional `personal_repo`). Commented-out repository blocks are confusing because they imply the repositories exist and could be uncommented, when in reality they are stale references.

### Syslinux boot menu simplified

The `archiso_sys-linux.cfg` (syslinux boot configuration) was stripped down to a single, clean boot entry. The original ArcoLinux config had multiple boot options (safe mode, various kernel parameters), most of which were not relevant to KIRO and added visual noise to the BIOS boot menu. A single, well-labeled default entry is cleaner and reduces the chance of a user accidentally booting with the wrong parameters.

### GRUB simplified; `grub` package added

The GRUB boot menu entries were similarly reduced, and the `grub` package itself was added to `packages.x86_64`. This ensures the installed system has GRUB available for configuration after installation, and that the live session's boot menu is clean and minimal.

### `virtual-machine-check.service` removed

This service — inherited from ArcoLinux — detected whether the system was running inside a VM and applied VM-specific tweaks at boot. Removing it was the right call: the service added boot time, and any VM-specific configuration should be handled by the VM guest additions packages (`open-vm-tools`, `virtualbox-guest-utils`) rather than a custom detection service.

### `build-the-iso.sh` simplified

Three outdated lines in the build script that referenced ArcoLinux-specific paths and logic were removed, simplifying the build flow.

---

## 2025-05-23 — Package List Expansion

The ISO package list received a major expansion, shifting from a minimal configuration to a more complete daily-driver environment:

### Applications added

- **`chromium`** — the open-source Chromium browser, complementing the existing Firefox install. Having both browsers available is useful for web development testing and for users who prefer Chromium's lower memory overhead compared to Chrome.
- **`gimp`** — the GNU Image Manipulation Program. Essential for any image editing work, from quick photo corrections to full compositing.
- **`inkscape`** — vector graphics editor. Pairs with GIMP for a complete open-source graphics workflow.
- **`meld`** — a visual diff and merge tool. Invaluable for comparing config files, reviewing patches, and resolving merge conflicts. Much more approachable than `diff` for users who prefer a GUI.
- **`nitrogen`** — a lightweight wallpaper manager for X11. Used by the chadwm environments to set the desktop background (XFCE4 handles this through its own settings manager).
- **`qbittorrent`** — a clean, Qt-based torrent client. Useful for downloading Arch-based ISO files, large open-source archives, and similar content.
- **`scrot`** — a command-line screenshot tool. Used in various keyboard shortcut bindings in the window manager configs.
- **`vlc`** — the VLC media player. Handles virtually every audio and video format without requiring additional codec packages.
- **`variety`** — a wallpaper changer that can download images from Flickr, NASA APOD, Reddit, and other sources on a schedule. Keeps the desktop visually fresh.
- **`simplescreenrecorder-qt6-git`** — a screen recorder with an intuitive GUI. The Qt6 build is preferred over the older Qt5 version for better HiDPI support and compatibility with modern display systems.

### Utilities added

- **`galculator`** — a scientific calculator with both standard and expression modes, GTK-based
- **`arandr`** — a graphical frontend for `xrandr` for managing monitor arrangements; essential for multi-monitor setups
- **`baobab`** — a disk usage visualizer (GNOME Disk Usage Analyzer). Makes it easy to identify what is consuming storage on a system being evaluated for installation
- **`gnome-screenshot`** — screenshot tool with timed capture and area selection

### Packages removed

- **`arc-gtk-theme`** — removed in favor of the `edu-arc-dawn-git` branded theme (already in the list)
- Several ArcoLinux font packages — these were ArcoLinux-branded font collections that served no purpose in a KIRO system

---

## 2025-04-29 — Versioning and Repository Infrastructure

### `change-version.sh` added

A dedicated script for bumping the ISO version across all files that embed it was added. Without this, version bumping requires manually editing `dev-rel`, `profiledef.sh`, `build-the-iso.sh`, and potentially other files — an error-prone process that inevitably leads to version mismatches. `change-version.sh` updates all of these in a single operation.

### `up.sh` added

The **`up.sh`** script was introduced as the daily maintenance helper. Running it refreshes the mirrorlist and calls `change-version.sh` to bump the date-based version string, preparing the working tree for a new build. This script is the single entry point for the daily rebuild cycle.

### `pacman.conf.kiro` added

An alternate `pacman.conf` variant was added for reference and comparison purposes. This gives a clear record of the intended final-state pacman configuration separate from the working `pacman.conf`, making it easier to diff what changed during troubleshooting.

### `linux-zen.preset` removed

Support for the Zen kernel was dropped. The Zen kernel is a performance-tuned variant, but maintaining a separate initramfs preset for it adds complexity. The CachyOS kernel (via Chaotic-AUR) better serves the performance tuning use case for the KIRO audience and does not require a separate preset in the ISO configuration.

### `pacman.conf` — Chaotic-AUR added

The **Chaotic-AUR** repository was added to the ISO's embedded `pacman.conf`. Chaotic-AUR is a binary repository that mirrors the most popular AUR packages as pre-built binaries, eliminating the need to compile from source. This is critical for the KIRO ISO because many of the tools in the package list (edu-* packages, window manager components, several AUR applications) are only available through Chaotic-AUR. Without it, the package list would be dramatically reduced or build times would become impractical.

---

## 2025-04-27 — Initial Commit

The KIRO ISO project was bootstrapped from an ArcoLinux base. This initial commit established the complete repository structure:

### ISO Configuration (`archiso/`)

The full `airootfs/` overlay was included — 93 files comprising the complete file system overlay that gets merged over the base Arch Linux system during ISO creation. This includes:

- All systemd service enablement symlinks for: SDDM (display manager), NetworkManager, Bluetooth, Avahi (mDNS), and CUPS (printing)
- Base configuration files for the shell, editor, and system
- The initial package list (`packages.x86_64`) — a comprehensive selection covering the full XFCE4 desktop, chadwm/ohmychadwm tiling window managers, development tools, multimedia applications, and system utilities

### Boot Configuration

- **GRUB** — EFI boot entries with standard, NVIDIA-nomodeset, and no-KMS options
- **Syslinux** — BIOS legacy boot configuration
- **systemd-boot** — EFI loader entries for modern UEFI systems

The three-bootloader setup ensures the ISO is bootable on any x86_64 system regardless of firmware type.

### Desktop Environments

Three desktop environments were configured from the start:

- **XFCE4** — the primary, full-featured desktop for users who want a traditional DE experience
- **chadwm** (later renamed `edu-chadwm`) — a customized build of dwm (suckless window manager) with a curated patch set for a practical tiling workflow
- **ohmychadwm** — a more opinionated chadwm configuration with additional visual polish

### Build System

**`installation-scripts/40-build-the-iso.sh`** (465 lines) — the main build automation script that orchestrates the ArchISO build process: validating the environment, installing build dependencies, calling `mkarchiso`, and packaging the output.

**`setup-git-v5.sh`** — the developer environment setup script (later renamed to `setup.sh`) that configures git, SSH keys, and other developer prerequisites.
