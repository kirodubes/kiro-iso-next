# TODO

## In Progress

## Up Next

## Backlog

## Done

- **Fix `isoLabel` mismatch in `build-the-iso.sh`** — `apply_version_bump()` re-derived `isoLabel="kiro-${kiroVersion}-…"` (dropping `next`), mismatching `iso_name="kiro-next"`. With the default `bump_version=yes` this made `create_checksums` checksum the wrong ISO (a stale `kiro-…` leftover) and leave the real `kiro-next-…` unchecksummed; without a leftover it would fail at `sha1sum`. Fixed both derivations to `kiro-next-…`. (flagged + fixed 2026-05-27)
- **Fix wrong microcode left installed after Calamares install** — `kiro_ucode` now removes the non-matching ucode package after installing the correct one. Verified working.
- **Fix grub.cfg and loopback.cfg kernel paths for linux-lqx** — all paths updated to `vmlinuz-linux-lqx` / `initramfs-linux-lqx.img`. Verified working.
- **linux.preset cleanup in installed system** — `kiro_final` now removes the archiso-only `linux.preset` artifact from the installed target. Verified working.
- **PipeWire as default audio stack** — replaced `pulseaudio`, `pulseaudio-alsa`, `pulseaudio-bluetooth` with `pipewire`, `pipewire-alsa`, `pipewire-audio`, `pipewire-pulse`, `wireplumber`, `gst-plugin-pipewire`, `pamixer`. Verified working.
- **Test BIOS/syslinux boot path** — syslinux configs updated for linux-lqx. BIOS boot verified working.
- **Test NVIDIA mode on real hardware** — `driver=nonfree` boot + DKMS compile against `linux-lqx-headers` verified working on real NVIDIA GPU.
