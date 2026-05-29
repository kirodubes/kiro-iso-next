# IDEAS

## Claude's Ideashop

### CachyOS-comparison improvements (2026-05-28)

Full multi-distro analysis (CachyOS / EndeavourOS / Garuda / Manjaro) of what would meaningfully push Kiro toward "best Arch ISO out there". Ranked by impact × effort. Cross-repo dependencies noted inline.

#### Tier 1 — High impact

**Online install mode (`settings_online.conf` + custom `pacstrap` Calamares module)** — CachyOS ships **both** offline and online installers; user picks at boot. Online install fetches fresh packages via `pacstrap` at install time. Benefits: always-current packages on first boot (no big post-install update wave), smaller ISO possible (strip rarely-used packages from squashfs since they'll be fetched), chwd can pull ANY driver from the network rather than only what's baked in (solves the offline NVIDIA driver-cache deadlock already on TODO). Cost: real work — port CachyOS's `pacstrap` module to `kiro-calamares-config-next` + maintain two `settings.conf` files. Biggest single "feels like a polished distro" upgrade. **Cross-repo:** primarily `kiro-calamares-config-next` (new module + 2nd settings.conf); `kiro-iso-next` adds an "Online install" GRUB live-menu entry.

**`packagechooser` for desktop selection at install time** — CachyOS's online installer presents desktop choices (KDE/GNOME/XFCE/i3/etc.) as radio buttons in Calamares. Kiro currently bakes XFCE+ohmychadwm into one ISO. For a community distro, offer "XFCE / ohmychadwm / minimal" at install time — same ISO, three install paths. The Calamares `packagechooser` module is already upstream — just needs configuration with the right package groups. Phase in: offer "with ohmychadwm" vs "without" first, expand later. **Cross-repo:** `kiro-calamares-config-next` (packagechooser config); `kiro-iso-next` (defines package groups).

**`rate-mirrors` as a Calamares step** — `rate-mirrors` is already in `packages.x86_64`. Add a `shellprocess` Calamares module that runs `rate-mirrors --allow-root arch > /etc/pacman.d/mirrorlist` inside the target chroot during install. Every Kiro install starts with the user's fastest local mirrors, not whatever was in the squashfs from build day. **Cost:** trivial (one shellprocess config). **Cross-repo:** `kiro-calamares-config-next` only. Strong "do this first" candidate.

#### Tier 2 — Quick wins

**Verify cachyos-hello (or equivalent) autostarts on first boot** — `cachyos-hello` is already shipped in `packages.x86_64`. Check whether it actually runs on first login: CachyOS pairs it with an autostart `.desktop` in `/etc/xdg/autostart/`. A first-boot welcome that says "Welcome to Kiro, here's how to update / get support / join the community" is the highest-leverage UX touch for new users. EndeavourOS and Garuda both have similar. **Cost:** tiny — verify autostart, possibly rebrand to a `kiro-hello` if you want it Kiro-themed instead of CachyOS-branded. **Cross-repo:** `kiro-iso-next` (airootfs autostart entry) + possibly a new `kiro-hello` source package in `nemesis_repo`.

**`archlinux-keyring-wkd-sync.timer` enabled by default** — ~~when the Arch keyring rotates, users who haven't updated in months get "signature is unknown" errors. The `archlinux-keyring-wkd-sync` package + its systemd timer auto-rotates keys in the background. CachyOS enables it. Add to `archiso/airootfs/etc/systemd/system/timers.target.wants/` so it's enabled on every Kiro install.~~ **VERIFIED 2026-05-28: ALREADY ACTIVE BY DEFAULT.** Upstream `archlinux-keyring` (in `base`) already ships the timer + a preset symlink in `/usr/lib/systemd/system/timers.target.wants/`. Confirmed `active` on hq with `systemctl is-active archlinux-keyring-wkd-sync.timer`. No action needed — every Kiro install already gets it.

**`chwd-kernel` integration with ATT** — ~~`chwd-kernel` is a companion binary that does mhwd-kernel-style operations (list/install/remove kernels with their `-headers`, all backed by chwd's data). ATT could shell out to `chwd-kernel --install linux-zen` instead of hand-rolling kernel management. Less code, same UX, automatically tracks upstream kernel-package availability. **Cross-repo:** `archlinux-tweak-tool-gtk4` (ATT's kernel page).~~ **APPLIED 2026-05-29.**

#### Tier 3 — Worth knowing, low priority

**`cachy-chroot` rescue utility** — CachyOS ships `cachy-chroot`, a wrapper that mounts a target rootfs and arch-chroots in even with LUKS/LVM/Btrfs subvolumes. Live ISO becomes a recovery tool for broken installs. CachyOS's PKGBUILD is GPL. Rebuild as `kiro-chroot` (or keep upstream name) in `nemesis_repo`. Niche but high-trust signal — "this distro takes recovery seriously." **Cross-repo:** `nemesis_repo` build of `cachy-chroot`; `kiro-iso-next` adds it to `packages.x86_64`.

**Calamares slideshow polish** — `kiro-calamares-config-next` ships `01cal.jpg` through `12cal.jpg` for the install-time slideshow. Check whether these still match the current Kiro branding (post-ArcoLinux de-brand). A polished slideshow showing real Kiro screenshots is the lowest-effort marketing during install. Manjaro's and Garuda's slideshows are where they put serious branding work. **Cross-repo:** `kiro-calamares-config-next` (branding dir).

#### Skip — already decided

Plymouth (rejected), Wayland-anything (X11-only commitment), multiple ISO flavors per DE (XFCE+ohmychadwm is the canonical pairing), Snapper (timeshift is the choice), social media integration (rejected), Garuda's BTRFS+timeshift-auto-snapshot pre-update hook (Kiro already explicitly avoids `timeshift-autosnap`).

#### Evaluated and rejected — `linux-cachyos-nvidia-open` prebuilt NVIDIA module (2026-05-28)

I initially recommended swapping `nvidia-open-dkms` for `linux-cachyos-nvidia-open` (the prebuilt NVIDIA-Open module package CachyOS ships) to skip first-boot DKMS compile time. **Rejected after testing** — the PKGBUILD has `depends=("$pkgbase=$_kernver" ...)`, an exact-version pin to `linux-cachyos`. CachyOS gets away with this because they release both packages from the same PKGBUILD in lockstep. For Kiro pulling from `[chaotic-aur]`, the two packages drift out of sync between releases and pacman fails: `cannot resolve "linux-cachyos=7.0.9-1", a dependency of "linux-cachyos-nvidia-open"`.

Reconsider only if Kiro decides to build `linux-cachyos` + `linux-cachyos-nvidia-open` in nemesis_repo in lockstep on every CachyOS release. That's a large ongoing maintenance commitment for the price of saving ~1 minute of first-boot DKMS compile time — not worth it. The same trap applies to **every** `linux-cachyos-*` companion package (zfs, r8125, etc.) — they all pin to the kernel version.

`nvidia-open-dkms` stays as the correct downstream-friendly choice because it's kernel-version-agnostic by design.

#### One-pick if you only do one before launch

**`rate-mirrors` Calamares step.** Costs almost nothing, every user benefits on every install, ships in the next ISO build.

---

### Monthly audit diff — compare audit runs over time

After each monthly `audit.sh` run, save the output to `~/kiro-audit-YYYY-MM-DD.txt` and diff against the previous month's file. A one-liner wrapper script (`audit-compare.sh`) runs the audit, saves the result, then prints `diff` against the last saved file with color highlighting. Over time this builds a regression history: when a PASS becomes a FAIL you know exactly which ISO build introduced it, without having to remember what changed. Rationale: the audit currently shows current state; the diff shows drift.

### ISO-to-ISO package diff script

After each build, compare the new `pkglist.txt` against the previous one and print three sections: packages added, packages removed, packages with a version change. A 10-line bash script using `comm` on sorted files is all it takes. Rationale: right now there is no quick way to see "what actually changed in this build vs the last one?" — you have to diff two raw pkglist files by hand. A diff summary at the end of `build-the-iso.sh` (or as a standalone `diff-pkglists.sh`) gives an instant audit trail and catches accidental package additions or removals before the ISO is uploaded.

### Build health dashboard — post-build HTML report
After `mkarchiso` completes, generate a simple static HTML file in `~/kiro-Out/` alongside the ISO that lists: build date, kiro version, NVIDIA driver selected, total package count, ISO size, and all three checksums in one place. A single `xdg-open` command opens it in the browser. Rationale: right now the build information is scattered across terminal output, the pkglist file, and three separate checksum files. A single report page makes it easy to screenshot and share when posting a new release, and gives a quick sanity check that the right driver was injected before uploading.
