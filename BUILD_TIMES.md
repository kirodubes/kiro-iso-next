# Build & Install Times

Tracks wall-clock for ISO builds (auto-appended by [`build-scripts/build-the-iso.sh`](build-scripts/build-the-iso.sh)) and for Calamares installs (extracted from `/var/log/Calamares.log` on the target). Newest entries at the top of each table.

Useful for spotting cost regressions when changing squashfs compression, kernel set, package list, or Calamares modules.

> **Test machines** are recorded under generic labels — `metal-A` / `metal-B` (bare metal, UEFI / systemd-boot), `metal-C` (bare metal, BIOS / grub, legacy NVIDIA), `kvm-vm` and `<vm-host>` (virtual machines). Real hostnames and network addresses are deliberately not recorded in this repo; use the same labels for new entries.

## ISO Builds

| When             | Version    | Kernel(s)                  | Squashfs       | Duration | ISO size | Notes                                    |
|------------------|------------|----------------------------|----------------|----------|----------|------------------------------------------|
| 2026-07-19 07:00 | v26.07.19 | linux-cachyos linux-zen | zstd L3 -b 1M | 8m4s | 6.1G | |
| 2026-06-14 05:10 | v26.06.14 | linux-cachyos linux-zen | zstd L3 -b 1M | 8m26s | 6.1G | |
| 2026-06-13 21:38 | v26.06.13 | linux-cachyos linux-zen | zstd L3 -b 1M | 7m27s | 6.1G | |
| 2026-06-13 19:47 | v26.06.13 | linux-cachyos linux-zen | zstd L3 -b 1M | 7m14s | 6.1G | |
| 2026-06-13 19:20 | v26.06.13 | linux-cachyos linux-zen | zstd L3 -b 1M | 7m55s | 6.1G | |
| 2026-06-13 18:03 | v26.06.13 | linux-cachyos linux-zen | zstd L3 -b 1M | 7m19s | 6.1G | |
| 2026-06-10 23:09 | v26.06.10 | linux-cachyos linux-zen | zstd L3 -b 1M | 7m51s | 6.4G | |
| 2026-06-10 22:06 | v26.06.10 | linux-cachyos linux-zen | zstd L3 -b 1M | 8m8s | 6.4G | |
| 2026-06-09 18:58 | v26.06.09 | linux-cachyos linux-zen | zstd L3 -b 1M | 10m39s | 6.4G | |
| 2026-06-09 10:53 | v26.06.09 | linux-cachyos linux-zen | zstd L3 -b 1M | 9m30s | 6.4G | |
| 2026-06-09 09:40 | v26.06.09 | linux-cachyos linux-zen | zstd L3 -b 1M | 7m27s | 6.1G | |

## Calamares Installs

| When             | ISO        | Target              | Duration | mkinitcpio passes | Notes                                          |
|------------------|------------|---------------------|----------|-------------------|------------------------------------------------|
| 2026-06-08 13:25 | v26.06.08  | kvm-vm | 2m35s    | 2                 | Part B validated: spice-vdagent kept on kvm    |
