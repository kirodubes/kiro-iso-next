# Distro Testing Log

Results of boot and install testing for kiro-iso-next builds. Newest first.

---

## 2026-10-03 (build 22:20) — v26.10.03 kiro-next, bare metal **UEFI**, **live session only**: NVIDIA free-entry fix + VA-API verified on an RTX 3070

Verifies kiro-iso-next `1dba1d8` (udev rule loading nouveau on `driver=free` + `vulkan-nouveau`) and `5ff56a8`
(`libva-nvidia-driver` with the open driver) on **metal-D**: Ryzen 7 3700X desktop, **GeForce RTX 3070 (GA104)**,
three monitors (DP 1080p, HDMI 1440p, DP 1440p@144). Live ISO only, never installed on. `ISO_BUILD` 22:12:05.
Baseline: the production `v26.10.03` 06:23 build on the same box (see kiro-iso DISTRO_TESTING), where entry 1
came up on `simpledrm`, one monitor at 1024x768, llvmpipe.

| Boot entry | Kernel driver | Monitors | OpenGL / Vulkan | VA-API | Result |
|------------|---------------|----------|-----------------|--------|--------|
| 1 `driver=free` | `nouveau` (loaded by `99-kiro-free-nouveau.rules`, no manual step) | 3, native | zink on NVK / NVK; glamor up on first boot | n/a | **PASS** |
| 2 `driver=nonfree` | `nvidia` open 615.71.09, nouveau **not** loaded | 3, native | NVIDIA 4.6 / NVIDIA | `VA-API NVDEC driver`, 19 decode profiles incl. H.264, HEVC Main/10/12/444, VP9, AV1 | **PASS** |

- Entry 1: zero failed units. The only error lines are `Module nvidia is blacklisted`, expected from the entry's
  `module_blacklist=nvidia,...` (Vulkan/VA-API loaders probing for nvidia).
- Entry 2: the new udev rule stays out of the way (`driver=nonfree` doesn't match, and `module_blacklist=nouveau`
  still blocks nouveau). Zero failed units.
- Not exercised: installs (`kiro_final` removing the rule, `kiro_remove_nvidia` removing `libva-nvidia-driver`),
  since metal-D is live-only. Needs a VM install from this ISO on `driver=free`.

---

## 2026-05-18 — v26.05.18.01 — VirtualBox (UEFI, Intel, NAT)

**Environment:** VirtualBox 7.x, UEFI firmware, Intel CPU (amd-ucode correctly absent), NAT networking with SSH port forwarding 2222→22

**Boot:** PASS — UEFI boot via systemd-boot, linux-lqx 7.0.9-lqx1-1-lqx kernel loaded

**Install:** Calamares install completed. Post-install audit via `audit.sh`:

| Check                                              | Result   |
|----------------------------------------------------|----------|
| Kernel (linux-lqx running)                         | PASS     |
| Boot files (vmlinuz-linux-lqx, initramfs)          | PASS     |
| Microcode (intel-ucode, no amd-ucode)              | PASS     |
| mkinitcpio (no archiso hook, has microcode/kms)    | PASS     |
| linux-lqx.preset exists, linux.preset removed      | PASS     |
| PipeWire stack complete, pulseaudio absent         | PASS     |
| calamares + mkinitcpio-archiso removed             | PASS     |
| kiro-calamares-config-next removed                 | **FAIL** |
| Calamares live-only artifacts cleaned up           | PASS     |
| /root permissions 700, sudoers.d 750, polkit 750   | PASS     |
| EDITOR=nano, Bluetooth AutoEnable=true             | PASS     |
| makepkg.conf optimized (MAKEFLAGS, PKGEXT, !debug) | PASS     |
| Pacman repos (nemesis_repo, chaotic-aur, multilib) | PASS     |
| ohmychadwm + XFCE desktop entries                  | PASS     |
| SDDM edu-simplicity theme                          | PASS     |
| User groups (wheel, audio, video, storage…)        | PASS     |
| Services (NetworkManager, sddm, bluetooth)         | PASS     |
| shadow/gshadow 400 permissions                     | PASS     |
| NVIDIA (correctly absent, no GPU)                  | PASS     |
| systemd-boot installed                             | PASS     |
| Package integrity (pacman -Qk)                     | PASS     |

**Score:** 63 PASS, 1 WARN (/etc/calamares dir leftover — caused by FAIL below), 1 FAIL

**Known issue:** `kiro-calamares-config-next` not removed post-install — `kiro_final` removal step fails silently (pacman lock race suspected). Package is manually removable. Does not affect system functionality.

**BIOS/syslinux boot path:** Not tested (VirtualBox uses UEFI). See TODO.md.
