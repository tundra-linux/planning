# Tundra

Tundra is an image-based, Alpine-derived desktop aimed at technically literate people leaving
Windows. Applications come from Flathub, the host filesystem is read-only, updates are atomic A/B,
and Podman with Distrobox is the escape valve for anything that needs a glibc userland.

The shell is KDE Plasma through Phases 1 and 2. Phase 3 is the intent to replace it with something
written for this project, and is direction rather than design.

**Status:** design, nothing built.
**Last substantive revision:** 2026-09-14.

## Repositories

```
tundra-linux/
  planning/       this repo; documents, no code
    docs/         PLAN.md, PHASE1.md, PHASE2.md, PHASE3.md
    AGENTS.md     document conventions
  tundra-pilot/   Phase 1 artifacts, the configuration tree in PHASE1.md P1-D12
                  (empty as of 2026-09-14)
```

Phase 2 has no build repository yet. It needs one before Track 0 Spike B produces anything worth
keeping; `PHASE2.md` P2-D01 describes the `aports` overlay layout that goes in it.

## Phases

**[Phase 1 — Fedora pilot](PHASE1.md).** Produces a version-controlled set of configuration
artifacts, validated on Fedora KDE Plasma Desktop 44: the panel layout, shortcut set, file manager
behaviour, theming, zsh configuration, and package delta. It does not produce an operating system.
The point is to settle the desktop design somewhere it does not have to fight a build system.

**[Phase 2 — the Tundra distribution](PHASE2.md).** Builds the OS around those artifacts: `aports`
overlay, OpenRC services, EROFS root, RAUC A/B updates, Flatpak, rootless Podman. Two of its spikes
run during Phase 1 rather than after it, because the session plumbing and the image pipeline are the
parts Phase 1 deliberately does not rehearse.

**[Phase 3 — the Tundra desktop](PHASE3.md).** Drops KDE Plasma for a modular Wayland session
written for this project. Direction rather than design: it carries more open questions than
decisions, and the first of them is what Plasma actually fails at. It changes the desktop layer and
nothing beneath it, and it cannot start until Phase 2 is shipping on its cadence.

## Project-wide

- **G-D01** The distro is named **Tundra**. No Linux distribution uses it: absent from DistroWatch's
  index, and the collisions in the wider ecosystem — a MyAnimeList scrobbler, a C build system, the
  TundraNAT64 tool — are in different namespaces. Toyota holds TUNDRA for vehicles, which does not
  reach software. Register the domain and the GitHub org before announcing anything.
- **G-C01** Single maintainer, no funding, no CI fleet. Every decision that adds a package to the
  overlay adds a rebuild obligation forever, and every target added to the hardware matrix has to be
  requalified on each release. This constraint kills more soft forks than any technical problem in
  either phase document.

## How to read the identifiers

Each phase document numbers its own outcomes, constraints, decisions, risks, verification gates and
open questions with a `P1-`, `P2-` or `P3-` prefix. `G-` is project-wide and lives here.
Cross-document references are written in full, as `P1-O09` or `P2-D15`.

Identifiers are **positional, not durable**: they renumber whenever a document is reorganised, so
they must not be cited from outside this repo. The full rule and the checks that keep references
honest are in [`AGENTS.md`](../AGENTS.md).

---

## Appendix A — audit of the source conversation

This project started as a conversation with Gemini, and the first version of this file was that
transcript. Every factual claim in it was checked. Most of the architecture was sound. The
specifics were unreliable, and the record is kept here because several of the corrections are the
reason particular decisions in the phase documents read the way they do.

### Wrong

| Claim | Reality |
| --- | --- |
| "Track Alpine's official v3.x LTS branches" | Alpine has no LTS release designation. `linux-lts` is a *kernel package* name. Stable branches get ~2 years for `main` and ~6 months for `community`, and the whole desktop stack is in `community`. This is P2-C02 and P2-R01, and the conversation missed it entirely. |
| Use `seatd` over `elogind`, "zero dependencies beyond libc" | Plasma needs the logind API. Alpine's own KDE documentation installs elogind and `polkit-elogind`. The wiki states the two are mutually exclusive. seatd is correct for Sway, not for Plasma. |
| Use `turnstile` over elogind for session supervision | turnstile is 0.1.11 in `edge/testing` only, self-described as work in progress, and is a session/login tracker rather than a logind replacement. Not available on a stable branch at all. |
| `systemd-sysupdate` as an A/B update option | Requires systemd. Irrelevant to an OpenRC distro. RAUC is the viable one and does work without systemd. |
| NVIDIA support needs "careful orchestration within your custom APK overlay" | NVIDIA's proprietary userspace is glibc-only and NVIDIA has declined to provide a musl build. No amount of overlay work fixes it. Nouveau/NVK or nothing. |
| `sudo ln -sf /usr/bin/dash /usr/bin/sh` on Fedora | Fedora packaging policy has RPM scriptlets assume `/bin/sh` is bash. This breaks package transactions, and the `bash` package owns the symlink so updates will revert it. |
| Install package `checkbashisms` | Fedora ships it as `devscripts-checkbashisms`. |
| `Ctrl+Shift+Esc` → `org.kde.ksysguard.desktop` | KSysGuard was replaced in Plasma 6. The desktop ID is `org.kde.plasma-systemmonitor.desktop`. |
| Config file `~/.config/dolphincrc` | The file is `dolphinrc`. |
| `SingleClick=false` belongs in that file | `SingleClick` lives in `kdeglobals` under `[KDE]`; it is a system-wide KDE setting, not a Dolphin one. |
| "Overriding KDE's legacy single-click defaults" | Plasma 6 has shipped double-click as the default since 6.0, for exactly the Windows-migrant reason given. Nothing to override, which is why P1-D04 ships no override. |
| Verification: `virsh list --all` as a normal user should return an empty list | Non-root `virsh` defaults to `qemu:///session`, which returns empty regardless of libvirt group membership. The check as written passes on a broken system. Use `virsh -c qemu:///system list --all`. |
| Verification: `sh --version` should output `/usr/bin/dash` | dash has no `--version` flag. |
| Verification: `echo $SHELL` after `chsh` | `$SHELL` is inherited from the login session and will not change in an open terminal. |
| System zsh config at `/etc/zsh/zprofile` | That is the Debian layout. Alpine does use `/etc/zsh/`; Fedora's layout needs checking on the machine (`rpm -ql zsh \| grep /etc`) rather than assuming either. |
| "Eliminate SDDM" | Fedora 44 already replaced SDDM with Plasma Login Manager across all KDE variants, so the pilot will not be running SDDM to begin with. Tundra runs SDDM because Alpine has no PLM package (P2-D14). |

### Overstated or incomplete

| Claim | Correction |
| --- | --- |
| A script validated under `dash` "will execute identically under BusyBox ash" | Close, not identical. They differ on `local`, `echo` handling and several builtins. Test both. |
| Flatpak "bypasses Alpine's musl hurdles entirely" | True for the applications, which bring their own glibc runtime. The host still needs portals, PipeWire, Mesa and a session bus, all of which are musl-native problems. |
| Podman/Distrobox just needs "cgroups v2, subuid/subgid, storage" | Accurate as a list, understated as effort. OpenRC does not do cgroup delegation the way systemd does, and BusyBox `adduser` does not provision subuid/subgid ranges. |
| "Everything in `/etc/skel` transfers cleanly" | Only at user creation. It reaches nobody who already has an account, which is a real problem for an OS that ships updates. This is why P1-D06 puts controlled defaults in the image instead. |
| `usermod -aG libvirt,kvm $USER` | The `libvirt` group is the one that matters. `kvm` group membership is generally unnecessary on modern Fedora, where udev handles `/dev/kvm` permissions. |
| Adding Flatpak paths to `PATH` in `.zshrc` | Fedora already exports `/var/lib/flatpak/exports/bin` via `/etc/profile.d/flatpak.sh`. Harmless, but it is not the fix it appears to be, and on Tundra it is the system profile that has to do this. |
| "xsettingsd is a dynamic theme-syncing daemon" to avoid | On Plasma, `xsettingsd` is what `kde-gtk-config` uses to apply GTK settings on X11. In a Wayland-only image (P2-D10) it is moot. |

### Confirmed

- The name **Tundra** is clear for a Linux distribution.
- Alpine v3.24 (released 2026-06-09) carries `plasma-desktop` 6.6.6, `flatpak` 1.16.6,
  `distrobox` 1.8.2.5, `virt-manager` 5.1.0 and `erofs-utils` 1.9.1, all in `community`.
- Alpine 3.23 shipped apk-tools 3.0, which keeps the v2 index and package format for now. Relevant
  to P2-D01: overlay repo tooling should assume apk 3 but v2 formats.
- The general Fedora-pilot-then-translate strategy holds. Desktop configuration is KDE- and
  XDG-standard, and it does move. The caveat is P2-R05: the pilot rehearses the desktop and none of
  the image machinery.
- Flatpak on a musl host genuinely does solve the application-compatibility problem, and it is the
  reason the whole approach is viable.
- RAUC is init-system agnostic at its core and runs standalone via `rauc service`.
- `erofs-utils` is packaged, so P2-D02 is buildable on a stock Alpine branch.

Checked later, while settling the branch policy, defaults mechanism, display manager, update
transport and hardware matrix:

- A custom Look-and-Feel package with `contents/layouts/org.kde.plasma.desktop-layout.js` is the
  supported way to ship a distro default panel layout (P1-D06).
- `plasma-login-manager` is absent from Alpine entirely, including edge. SDDM 0.21.0 is in
  `community` (P2-D14).
- RAUC supports verity bundles with HTTP streaming, and adaptive `block-hash-index` updates at 0.8%
  index overhead, distinct from casync's chunk-store approach. Its PKI documentation describes four
  signing models but no key-rotation mechanism (P2-D16, P2-R09).
- RAUC's bootloader backends are barebox, grub, uboot, efi, custom and noop. The `grub` backend
  drives `grub-editenv` with `<bootname>_OK`, `<bootname>_TRY` and `ORDER`; the `efi` backend uses
  the `BootCurrent` EFI variable (P2-D03).
- RAUC itself is in no stable Alpine branch — `rauc` 1.10.1 sits in `edge/testing`, built
  2023-08-08, per `pkgs.alpinelinux.org` on 2026-09-14. Tundra builds its own (P2-D04).
- Alpine's wiki documents immutable root with atomic upgrades using btrfs snapshots and rEFInd, a
  different architecture from image-based A/B. Recorded as a rejected alternative in P2-D02.

### Sources

- https://www.alpinelinux.org/releases/
- https://wiki.alpinelinux.org/wiki/KDE
- https://wiki.alpinelinux.org/wiki/Seat_manager
- https://wiki.alpinelinux.org/wiki/Podman
- https://wiki.alpinelinux.org/wiki/Flatpak
- https://wiki.alpinelinux.org/wiki/NVIDIA
- https://pkgs.alpinelinux.org/packages
- https://alpinelinux.org/posts/Alpine-3.23.0-released.html
- https://github.com/chimera-linux/turnstile
- https://rauc.readthedocs.io/en/latest/integration.html
- https://rauc.readthedocs.io/en/latest/advanced.html
- https://rauc.readthedocs.io/en/latest/reference.html
- https://wiki.alpinelinux.org/wiki/Immutable_root_with_atomic_upgrades
- https://community.kde.org/Plasma/lookAndFeelPackage
- https://userbase.kde.org/Plasma/Create_a_Look_and_Feel_Package
- https://docs.pagure.org/packaging-guidelines/Packaging:Scriptlets.html
- https://pagure.io/packaging-committee/issue/184
- https://packages.fedoraproject.org/
- https://fedoramagazine.org/whats-new-in-fedora-kde-plasma-desktop-44/
- https://fedoraproject.org/wiki/Changes/PlasmaLoginManager
- https://www.phoronix.com/news/KDE-Plasma-6-Double-Click
- https://archlinux.org/packages/extra/x86_64/plasma-systemmonitor/files/
- https://libvirt.org/uri.html
- https://distrowatch.com/
