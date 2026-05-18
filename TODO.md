# TODO

## In Progress

## Up Next

## Backlog

- **Fix wrong microcode left installed after Calamares install**
  Both `amd-ucode` and `intel-ucode` are in the squashfs. `kiro_ucode` installs the correct one via bundled `.pkg.tar.zst`, but never removes the wrong one. On an Intel machine, `amd-ucode` remains installed (and vice versa). Root cause: pacman databases on the live environment don't reflect actual installed state, so a conditional remove fails silently. Fix in `kiro_ucode` or `kiro_final` — detect CPU, remove non-matching ucode package from the installed target. Affects `kiro-calamares-config-next` (`kiro_ucode/main.py`).

- **Fix grub.cfg and loopback.cfg kernel paths for linux-lqx**
  `archiso/grub/grub.cfg` and `archiso/grub/loopback.cfg` still reference `vmlinuz-linux` / `initramfs-linux.img`. Used for GRUB loopback boot (multi-ISO USB) and BIOS fallback.

- **Test BIOS/syslinux boot path**
  syslinux configs were updated for linux-lqx but only UEFI was tested in VirtualBox.

- **Test NVIDIA mode on real hardware**
  `driver=nonfree` boot + DKMS compile against `linux-lqx-headers` not yet validated on real NVIDIA GPU.

## Done
