# Phase 2 — the Tundra distribution

Phase 2 builds the operating system: an Alpine soft fork with a read-only root filesystem, atomic
A/B updates, Flatpak as the only application channel, and rootless Podman as the escape valve for
anything that needs glibc. It consumes the configuration artifacts Phase 1 produces and supplies
everything underneath them.

Phase 2 is not strictly downstream of Phase 1. Two spikes — session plumbing, and the image plus
update machinery — start during Phase 1 and are described under Track 0 below. Everything else
waits on the entry criteria.

See [PLAN.md](PLAN.md) for the project overview and [PHASE1.md](PHASE1.md) for the artifacts this
phase consumes.

---

## Entry criteria

The main body of Phase 2 starts when Phase 1's deliverables exist and its gates pass. Specifically:
the configuration repo applies to a clean install unattended (P1-O01), the desktop design is settled
rather than still moving (P1-O02), the Look-and-Feel package works on a fresh account (P1-O03), the
provenance record says which keys take from `/etc/xdg` and which do not (P1-O04), the script corpus
and its lint harness exist (P1-O05), the zsh configuration is split into system file and stub
(P1-O06), the package delta is documented with reasons (P1-O07), and the KVM recipe is written
(P1-O08). Phase 1's remaining exit conditions apply too: its gates pass on a clean install, the
provenance record is complete, and P1-Q01 is answered, since P2-D06 implements the answer.

Track 0 does not wait for any of this.

## Track 0 — spikes that run during Phase 1

Both of these exist to convert unknowns into known work before the schedule depends on them. They
are throwaway: the output is a written list of what breaks, not a shippable system.

**Spike A, session plumbing.** Stand up a scratch Alpine VM running Plasma under OpenRC with
elogind and polkit. elogind, `XDG_RUNTIME_DIR`, the display manager and Plasma's power management
all interact, and on Fedora systemd hides the whole knot. This is the hardest part of Phase 2 and
Phase 1 rehearses none of it, so it should be the first thing attempted rather than the last.

**Spike B, image and update machinery.** Build an EROFS rootfs, boot it read-only, and install a
RAUC update to the inactive slot on a scratch VM. Same reasoning: Phase 1 deliberately trades away
any rehearsal of the image pipeline, and that debt should be paid down in parallel rather than
discovered at integration time.

## Outcomes

- **P2-O01** Tundra boots to a KDE Plasma Wayland desktop on an Alpine base with a read-only root
  filesystem.
- **P2-O02** System updates are atomic: a signed rootfs image is written to the inactive slot and
  activated on reboot, with automatic rollback on a failed boot.
- **P2-O03** All end-user applications install as Flatpaks. The host image carries no application
  package manager and no graphical app store.
- **P2-O04** Rootless Podman and Distrobox work out of the box, giving users a glibc environment for
  CLI tooling the musl host cannot run.
- **P2-O05** A Windows user who knows what a filesystem is can operate the desktop without a
  tutorial. The design is Phase 1's (P1-O02); this outcome is its delivery on Tundra.
- **P2-O06** The system shell is BusyBox `ash` and the interactive user shell is `zsh`, running the
  configuration from P1-O06.
- **P2-O07** The host image carries no dependency that is not either load-bearing or an industry
  standard with no credible alternative.
- **P2-O08** `virt-manager` with a working KVM stack ships in the base image as a first-class
  feature, not an add-on.

## Constraints

- **P2-C01** The host is musl libc. Nothing that ships only glibc binaries runs on the host. This
  is why Flatpak and Podman are load-bearing rather than convenient.
- **P2-C02** Alpine's `main` repository gets ~2 years of support. `community` is supported only on
  the newest stable branch: when v3.N+1 ships, v3.N's community stops. Plasma, Flatpak, Podman,
  Distrobox, `virt-manager`, SDDM and `erofs-utils` are all in `community`. There is no 2-year LTS
  for any part of Tundra's desktop stack. Community support follows the newest branch rather than
  expiring outright, which is what makes P2-D14 affordable.
- **P2-C03** No systemd. Every service, session and update mechanism has to work under OpenRC.
- **P2-C04** KDE Plasma requires the logind D-Bus API. This constrains the seat and session
  decision more than dependency count does.
- **P2-C05** The maintainer budget in `PLAN.md` G-C01 applies throughout. Every package added to
  the overlay is a permanent rebuild obligation.

## Decisions

- **P2-D01** Base on Alpine as a soft fork: track upstream `aports` and carry a Tundra `aports`
  overlay plus a signed APK repository for branding, kernel configuration, boot scripts, and any
  `community` package that has to be kept alive past its upstream support window. Overlay tooling
  should assume apk-tools 3 with the v2 index and package formats, which is what Alpine 3.23 shipped.
- **P2-D02** Root filesystem is **EROFS**, read-only, with overlayfs for `/etc` and tmpfs for
  `/tmp`. `/home` and `/var` live on a separate persistent partition. `erofs-utils` 1.9.1 is
  packaged in Alpine `community`. SquashFS is the fallback if EROFS support in `linux-lts` needs
  kernel configuration changes not worth carrying.
- **P2-D03** The A/B updater is **RAUC**. It needs GLib, OpenSSL and D-Bus rather than systemd; the
  service runs standalone via `rauc service`, so an OpenRC init script covers it. Rejected:
  `systemd-sysupdate`, which requires systemd and was never viable here.
- **P2-D04** Use **elogind**, not seatd. Plasma wants the logind API, and Alpine's KDE
  documentation is built around elogind plus `polkit-elogind`. seatd is the right answer for Sway
  and the wrong answer here. This knowingly overrides the dependency-minimalism instinct in P2-O07.
- **P2-D05** Keep a display manager. TTY autologin into `dbus-run-session startplasma-wayland`
  drops PAM session registration, which the lock screen and power management depend on. A laptop
  that does not lock is a security bug in a distro aimed at ex-Windows users.
- **P2-D06** Flatpak is the only application channel. Install the `flatpak` CLI plus
  `xdg-desktop-portal` and `xdg-desktop-portal-kde`; no Discover, no store frontend. Bind
  `/var/lib/flatpak` to the persistent partition. Flatpak runtimes carry their own glibc, which is
  what makes a musl desktop host tractable at all. The update path for those applications is
  P1-Q01, answered in Phase 1 and implemented here.
- **P2-D07** Podman rootless plus Distrobox, both packaged in Alpine `community` (Distrobox
  1.8.2.5). Set `rc_cgroup_mode=unified`, provision `/etc/subuid` and `/etc/subgid` at user
  creation, and put container storage on the persistent partition.
- **P2-D08** BusyBox `ash` is `/bin/sh`; `zsh` is the interactive shell for human users. The zsh
  configuration comes from P1-O06 and lives in Alpine's system-wide zsh file, with
  `/etc/skel/.zshrc` as a stub.
- **P2-D09** Pure Wayland. No `xorg-server` in the host image. `xwayland` ships because Flatpak
  applications will need it, but nothing on the host requires it.
- **P2-D10** Alpine package selection follows the Phase 1 delta (P1-O07), translated rather than
  copied. Install `plasma-desktop` rather than `plasma-desktop-meta`; omit Discover, PackageKit,
  Baloo and the PIM stack.
- **P2-D11** Target AMD and Intel graphics. NVIDIA support means Nouveau/NVK only: the proprietary
  driver's userspace is glibc-only and NVIDIA has declined to provide a musl build, so this is a
  hard boundary rather than a packaging problem to solve later.
- **P2-D12** Scope QEMU to `qemu-system-x86_64` and `qemu-img` rather than taking the meta-package
  and its foreign architectures.
- **P2-D13** Use **SDDM** 0.21.0 from `community`. `plasma-login-manager` is not packaged in Alpine
  at all, in stable or edge. The display manager is explicitly not a transferable Phase 1 artifact —
  the pilot runs Plasma Login Manager on Fedora 44 and Tundra runs SDDM — and the skew is harmless
  because a display manager carries almost no user-facing design beyond theming and autologin
  policy. Revisit if Plasma Login Manager reaches Alpine.
- **P2-D14** Ship from the newest Alpine stable branch and rebase on Alpine's cadence, targeting a
  Tundra release within 8 weeks of each Alpine release, so May and November become July and January.
  Because community support follows the newest branch (P2-C02), sitting on it gives continuous
  coverage; the cost is a fixed release cadence rather than a package-maintenance burden. Atomic A/B
  with rollback is what makes a six-month base rebase survivable, which is the same bet Silverblue
  and Kinoite make. Two supporting rules. Run a second CI build against `edge` continuously, not to
  ship from but as early warning, since edge is the pre-release state of the next branch and
  anything that will break the rebase breaks in CI months ahead. And cap the overlay: forked
  `community` packages need written justification, are permanent, and stay under 10. Past 20 the
  strategy has failed and the base needs reconsidering.
  Rejected: tracking `edge`, which trades one scheduled disruption for continuous unscheduled ones.
  Rejected: forking the desktop stack into the overlay, which P2-C05 rules out — Plasma alone is
  100+ packages on top of KDE Frameworks and Qt.
- **P2-D15** Update transport: **verity** bundles, **adaptive** updates, static HTTPS, single-CA
  PKI. Verity adds dm-verity over the payload and enables HTTP streaming, so RAUC installs straight
  from a URL instead of staging a full bundle locally; on an immutable system with a small writable
  partition that is a requirement rather than a nicety. Bandwidth comes from RAUC's adaptive
  `block-hash-index` method, which fetches only the 4KB blocks that differ from what is already on
  the device, at 0.8% index overhead, and works against any prior version or an interrupted attempt.
  Rejected: casync, which achieves similar savings but requires bundle conversion and a server-side
  chunk store to operate forever. Hosting is object storage behind a CDN serving static files: one
  bundle per release plus a small JSON manifest the client polls. Rejected: hawkBit, which is fleet
  management for industrial devices and a service that would have to be run, secured and backed up;
  staged rollouts can be manifest channels long before a deployment server is warranted. Signing
  uses a single CA with an offline root, issuing a release certificate with limited validity and a
  separate development certificate. See P2-R09 on rotation.
- **P2-D16** Qualify on three targets: a **qemu/virtio VM** as the primary CI target, **one Intel
  laptop**, and **one AMD machine**. The VM matters most and is easy to under-weight, because A/B
  update and rollback testing is destructive, has to run on every build, and only a VM makes that
  automatable. The Intel laptop covers the ordinary-hardware case and exercises suspend/resume,
  backlight and wifi, where real breakage lives and a VM says nothing. The AMD machine covers
  discrete graphics and gaming-adjacent use, which drives a large share of Windows departures. Out
  of scope for v1: NVIDIA (P2-D11), Apple Silicon, anything needing out-of-tree or signed vendor
  modules, and ARM. Three is the number because P2-D14 requires requalifying the whole matrix every
  six months. Both physical machines must be ones that can be bricked; neither is a daily driver.

## Risks

- **P2-R01** *The six-month rebase is a hard deadline, not a goal.* P2-D14 keeps continuous
  community support only while Tundra sits on the newest Alpine stable. Every week between an Alpine
  release and the Tundra rebase is a week the desktop stack receives no security fixes, so a slipped
  rebase is a security incident rather than a missed feature. Mitigation: the `edge` CI build
  (P2-V10) removes most of the surprise, and the cadence is the project's one immovable commitment.
  If two consecutive rebases slip, the maintainer budget assumption is wrong and scope has to shrink.
- **P2-R02** *NVIDIA hardware is out of scope by construction.* Anyone leaving Windows with an
  NVIDIA GPU gets Nouveau/NVK performance or nothing. Mitigation: say so on the download page rather
  than absorbing the support load.
- **P2-R03** *Session plumbing is the hardest part of this phase and Phase 1 does not rehearse it.*
  Mitigation: Track 0 Spike A, run during Phase 1 rather than after it.
- **P2-R04** *Rootless Podman needs cgroups v2 with proper delegation,* which OpenRC does not set
  up the way systemd does. Mitigation: `rc_cgroup_mode=unified`, and P2-V05 as an acceptance gate
  rather than a nice-to-have.
- **P2-R05** *Phase 1 rehearses none of the image build, A/B update or rollback machinery.* That is
  the deliberate cost of piloting on a mutable Fedora. Mitigation: Track 0 Spike B.
- **P2-R06** *`turnstile` is version 0.1.11 in Alpine `edge/testing`,* not in any stable branch, and
  self-described as work in progress. It is not a replacement for elogind under Plasma today.
  Mitigation: revisit no earlier than the second Tundra release.
- **P2-R07** *Secure Boot is unscoped.* Signed A/B images imply a decision about shim and about
  whether Tundra enrols its own keys. Nothing in this plan addresses it.
- **P2-R08** *Maintainer bandwidth.* One person maintaining an overlay repo, an image pipeline, an
  update server and a desktop stack on a non-glibc base, requalifying the full hardware matrix every
  six months. This is the failure mode that kills most soft forks.
- **P2-R09** *The signing certificate will expire and RAUC has no key-rotation mechanism.* Rotation
  means issuing a new certificate and pushing an updated keyring to devices, so the keyring update
  path has to work before the first certificate expires. A device that misses the rotation window
  can never be updated again by any means short of reinstallation. Mitigation: ship and test the
  keyring update path in release 1, not release 3 (P2-V12), and set release-certificate validity
  longer than the worst-case interval between a user's updates.
- **P2-R10** *The handoff may arrive incomplete.* Phase 2 assumes a settled desktop design. If
  Phase 1's design is still moving when this phase starts, every change lands as rework in an image
  pipeline where iteration is slow — which is precisely the cost the two-phase split exists to
  avoid. Mitigation: P2-V01 is a gate, not a formality, and a design still in flux is a reason to
  extend Phase 1 rather than to start Phase 2 early.

## Verification

Entry gate.

- **P2-V01** Every Phase 1 deliverable exists and every Phase 1 gate passes on a clean install
  performed from the Phase 1 repo.

Track 0 gates. Both should pass during Phase 1.

- **P2-V02** A scratch Alpine VM reaches a Plasma Wayland session under OpenRC with elogind, and
  the spike produces a written list of what broke and what it cost to fix.
- **P2-V03** A scratch VM boots an EROFS root read-only and RAUC installs a bundle to the inactive
  slot.

Desktop and runtime gates.

- **P2-V04** Plasma reaches a usable Wayland session on Alpine under OpenRC with elogind, including
  a working lock screen and suspend.
- **P2-V05** `podman run --rm alpine true` succeeds as an unprivileged user, and `podman info`
  reports cgroup version 2.
- **P2-V06** `distrobox create` and `distrobox enter` produce a working glibc shell.

Image gates.

- **P2-V07** A RAUC update installs to the inactive slot, the system boots into it, and a
  deliberately corrupted image rolls back automatically without manual intervention.
- **P2-V08** `mount | grep ' / '` shows the root filesystem mounted read-only, and `/home`,
  `/var/lib/flatpak` and container storage are all on the persistent partition.
- **P2-V09** A `/etc` change made by the user survives a full A/B update cycle.

Release engineering gates.

- **P2-V10** A CI build against Alpine `edge` runs on every push and on a schedule, and its failures
  are visible without anyone going to look for them. A build nobody sees does not count as early
  warning.
- **P2-V11** An adaptive update between two consecutive Tundra releases transfers substantially less
  than the full image. Measure the bytes; if it approaches full-image size the block index is not
  doing its job and the release process is probably perturbing block alignment.
- **P2-V12** A device holding only the old keyring accepts a keyring update, then installs a bundle
  signed by the new release certificate. Run this before release 1 ships (P2-R09).
- **P2-V13** A full rebase to the next Alpine stable completes in a dry run, on the `edge` CI
  branch, before that Alpine release is cut.

## Definition of done

Release 1 of Tundra ships when:

1. P2-O01 through P2-O08 hold on all three P2-D16 targets.
2. P2-V01 through P2-V13 pass, with P2-V12 passing before rather than after the release.
3. P2-Q01 is answered and implemented, because there is no release without a way to install it.
4. P2-Q02 is answered, because there is no release without somewhere to serve it from.
5. The overlay package list is under the P2-D14 cap and every entry carries its justification.

## Work sequence

1. Track 0 Spikes A and B, during Phase 1.
2. Stand up the `aports` overlay, the signed APK repository, and the build host (P2-D01).
3. Base system: Alpine branch, kernel, elogind, polkit, seat and session plumbing, informed by
   Spike A (P2-D04, P2-D05).
4. Desktop: Plasma, Wayland, SDDM, PipeWire, portals, and the Phase 1 configuration artifacts
   (P2-D09, P2-D10, P2-D13, P2-O05).
5. Application and container layers: Flatpak, Podman, Distrobox, virtualization (P2-D06, P2-D07,
   P2-O08).
6. Image pipeline: EROFS build, partition layout, persistent-volume binds, informed by Spike B
   (P2-D02).
7. Update system: RAUC integration, bundle format, PKI, keyring update path, hosting (P2-D03,
   P2-D15).
8. Installer (P2-Q01), then hardware qualification across the P2-D16 matrix.

## Open questions

- **P2-Q01** How does a user install Tundra? There is no installer decision anywhere yet. An
  image-based distro needs one, it has to partition for A/B slots plus persistent storage, and it is
  the first thing someone leaving Windows touches. Options are Calamares, something custom, or
  shipping a pre-imaged disk and deferring. This is the largest remaining gap in the project.
- **P2-Q02** Hosting and cost model for P2-D15. Which object storage and CDN, what per-release and
  per-user bandwidth actually costs at 100, 1000 and 10000 installs, and who pays. Adaptive updates
  make the cadence affordable but not free.
- **P2-Q03** What happens to a machine that has been offline longer than one release cycle? P2-D14
  produces a new base every six months, and whether A/B updates chain cleanly across two or three
  releases or require a reinstall is untested and unspecified.
