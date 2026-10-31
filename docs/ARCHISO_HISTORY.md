# archiso sync history

Every upstream `archiso` version Kiro was compared against, and what was decided for each upstream change.
`/kiro-archiso` reads this file first. A change marked **Skipped** is only proposed again if upstream changes it again.
The upstream copies used for each diff are kept in `/usr/share/archiso-<version>` on the build host.
The current baseline is in [ARCHISO_BASELINE](ARCHISO_BASELINE).

Decision classes: **Taken** (applied to Kiro), **Skipped** (Erik declined, reason given), **Kiro-own** (Kiro deliberately
differs), **N/A** (Kiro doesn't have the file or feature), **Already** (Kiro already had it).

## archiso 92-1 (compared against 91-1, 2026-10-31)

Upstream changelog: memtest86+-efi 9.00 renamed its EFI binaries (`memtest.efi` → `memtest86<arch>.efi`), new
memtest entries for the other UEFI architectures, `linux-atm` removed, `kms` hook removed.

| Change | Decision | Reason |
|--------|----------|--------|
| Remove `linux-atm` from `packages.x86_64` | Taken | ADSL ATM userspace; upstream says ATM support was dropped with linux 7.2 |
| Remove `kms` from the mkinitcpio `HOOKS` | Skipped | Kiro's `plymouth` splash relies on early KMS; upstream gave no reason |
| GRUB memtest entry: `memtest86${uefi_machine_type_lower}.efi`, `if/elif` | Skipped | Erik's choice; GRUB isn't one of Kiro's bootmodes (syslinux + systemd-boot), so it is drift only |
| systemd-boot memtest x64 entry (`memtest86x64.efi`) | Skipped | Erik's choice; Kiro has never had a UEFI memtest entry. Taking it later means renaming it to `05-…` (sort-key 03 is Kiro's nomodeset) and raising `check_archiso_version` to 92-1 |
| systemd-boot memtest IA32 / AA64 / LOONGARCH64 entries | N/A | Kiro is x86_64 only and targets newer machines |
| mkarchiso copies `memtest86<arch>.efi` | Already | Builder-side; works with Kiro's profile unchanged, nothing breaks |
| `packages.aarch64`: `linux-atm` removed | N/A | Kiro has no aarch64 list |

BIOS memtest is unchanged: `memtest86+` still ships `memtest.bin`, which mkarchiso installs as `memtest` for syslinux.

## archiso 91-1 (compared against 90-1, 2026-10)

From the CHANGELOG entry "archiso profile synced with upstream archiso 91" (commit `b60cf13`).

| Change | Decision | Reason |
|--------|----------|--------|
| `profiledef.sh`: drop `buildmodes=('iso')` | Taken | `iso` is mkarchiso's default |
| Remove `airootfs/etc/modprobe.d/broadcom-wl.conf` | Taken | Empty override for a package Kiro no longer ships |
| GRUB UEFI Shell entries collapsed into `uefi_machine_type_lower` | Taken | Same behaviour, upstream patches apply cleanly |
| `%KERNEL_PARAMS%` at the end of every boot entry | Taken | Later used for `kernel_params_x86_64`; `check_archiso_version` requires 91-1 |
| Upstream deleted `airootfs/etc/mkinitcpio.d/linux.preset` | Kiro-own | Kiro builds one image per kernel and the build script retargets the preset |
| mkarchiso warns when run as root via sudo | Kiro-own | Kiro's build still runs `sudo mkarchiso`; warning only |
| aarch64 package list, `kernel_params_aarch64`, stubble DTB images, speech entries | N/A | x86_64 only |

## archiso 90-1 and 89-1

No sync decisions were recorded. The upstream copies are still in `/usr/share/archiso-90-1` and `/usr/share/archiso-89-1`.
