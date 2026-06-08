# Build & Install Times

Tracks wall-clock for ISO builds (auto-appended by [`build-scripts/build-the-iso.sh`](build-scripts/build-the-iso.sh)) and for Calamares installs (extracted from `/var/log/Calamares.log` on the target). Newest entries at the top of each table.

Useful for spotting cost regressions when changing squashfs compression, kernel set, package list, or Calamares modules.

## ISO Builds

| When             | Version    | Kernel(s)                  | Squashfs       | Duration | ISO size | Notes                                    |
|------------------|------------|----------------------------|----------------|----------|----------|------------------------------------------|

## Calamares Installs

| When             | ISO        | Target              | Duration | mkinitcpio passes | Notes                                          |
|------------------|------------|---------------------|----------|-------------------|------------------------------------------------|
| 2026-06-08 13:25 | v26.06.08  | erik@192.168.122.78 | 2m35s    | 2                 | Part B validated: spice-vdagent kept on kvm    |
