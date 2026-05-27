# TODO

## In Progress

## Up Next

## Backlog

- **`isoLabel` mismatch in `build-the-iso.sh`** — line ~99 sets `isoLabel="kiro-next-${kiroVersion}-…"` but `apply_version_bump()` re-derives it as `kiro-${kiroVersion}-…` (drops `next`). Self-corrects when `bump_version=yes`; with `bump_version=no` it can mismatch the produced ISO name and fail the checksum phase. Pre-existing, unrelated to the kernel work. (flagged 2026-05-27)

## Done

- **Fix wrong microcode left installed after Calamares install** — `kiro_ucode` now removes the non-matching ucode package after installing the correct one. Verified working.
- **Fix grub.cfg and loopback.cfg kernel paths for linux-lqx** — all paths updated to `vmlinuz-linux-lqx` / `initramfs-linux-lqx.img`. Verified working.
- **linux.preset cleanup in installed system** — `kiro_final` now removes the archiso-only `linux.preset` artifact from the installed target. Verified working.
- **PipeWire as default audio stack** — replaced `pulseaudio`, `pulseaudio-alsa`, `pulseaudio-bluetooth` with `pipewire`, `pipewire-alsa`, `pipewire-audio`, `pipewire-pulse`, `wireplumber`, `gst-plugin-pipewire`, `pamixer`. Verified working.
- **Test BIOS/syslinux boot path** — syslinux configs updated for linux-lqx. BIOS boot verified working.
- **Test NVIDIA mode on real hardware** — `driver=nonfree` boot + DKMS compile against `linux-lqx-headers` verified working on real NVIDIA GPU.
