# Phase 1 — Fedora pilot

Phase 1 does not produce an operating system. It produces a version-controlled set of
configuration artifacts, validated on Fedora KDE Plasma Desktop 44, that Phase 2 consumes as the
Tundra default profile. The pilot machine is scaffolding; the repo is the deliverable.

The reason to do this on Fedora rather than directly on Alpine is that desktop design and OS
engineering are separate problems, and fighting both at once is how this kind of project stalls.
Fedora gives a working Plasma stack for free, so the only variable is the design itself.

See [PLAN.md](PLAN.md) for the project overview and [PHASE2.md](PHASE2.md) for what happens to
these artifacts. Phase 2's Track 0 spikes run in parallel with this phase, not after it.

---

## Outcomes

Each is an artifact that exists in the repo when Phase 1 is done.

- **P1-O01** A configuration repo that applies to a clean Fedora KDE 44 install with no manual
  steps and no undocumented prerequisites.
- **P1-O02** A Windows-ergonomics desktop design that has been used daily long enough to know it
  works: panel layout, shortcut set, file manager behaviour, theming.
- **P1-O03** An `org.tundra.desktop` Look-and-Feel package that produces the panel layout on a
  fresh user account.
- **P1-O04** An `/etc/xdg` defaults set, plus a per-key record of which keys Plasma actually honours
  as system defaults and which it silently ignores.
- **P1-O05** A POSIX-clean script corpus and the lint harness that keeps it clean.
- **P1-O06** The Tundra zsh configuration, split between a system-wide file and a user stub.
- **P1-O07** The Fedora 44 package delta, documented as install and remove lists with a reason
  against each entry, as the input to Phase 2's Alpine package selection.
- **P1-O08** A working KVM and `virt-manager` recipe: packages, groups, services, and the
  permissions model.

## Constraints

- **P1-C01** Everything intended to survive into Phase 2 must be XDG- or KDE-standard. Fedora
  tooling, RPM-specific paths and distribution-specific helpers do not transfer and must not appear
  in any captured artifact.
- **P1-C02** `/bin/sh` stays `bash`. Fedora's RPM scriptlets assume bash and packaging policy states
  the assumption; repointing the symlink breaks package transactions, and the `bash` package owns
  the symlink so updates revert it anyway.
- **P1-C03** No systemd-specific user-session mechanisms. `systemctl --user` units are out.
  Autostart goes through `~/.config/autostart/` per the XDG spec, because that is what will exist on
  Alpine.
- **P1-C04** Fedora KDE 44 with Plasma 6.6.4 is the reference platform. Every captured artifact
  records the Plasma version it was captured against, because KDE config keys move between releases
  and a file with no provenance cannot be safely replayed later.

## Decisions

- **P1-D01** Pilot on **Fedora KDE Plasma Desktop 44**, currently shipping Plasma 6.6.4. Rejected:
  Fedora Kinoite or a `bootc` image, which would also rehearse the image and atomic-update half of
  Tundra. Rejected because the pilot's job is to iterate on desktop configuration quickly, and an
  immutable base makes that loop slower. The cost is that Phase 1 rehearses none of the image
  machinery, which Phase 2 absorbs through its Track 0 spikes.
- **P1-D02** Panel and taskbar: a single bottom panel with the Kickoff application launcher at
  left, an Icons-Only Task Manager in the middle, and the system tray with a digital clock at right.
  This is the Windows arrangement and Plasma does it natively, with no extension layer to maintain.
- **P1-D03** Shortcuts are captured as a **delta, not a set**. Dump the stock `kglobalshortcutsrc`
  from a fresh Fedora 44 account first, then record only the entries that differ. Plasma already
  ships several of the intended bindings, and shipping a full copy of the file would freeze
  unrelated defaults and fight every future Plasma release. The target bindings are `Super+E` for
  Dolphin, `Super+R` for KRunner, `Super+D` to show the desktop, `Ctrl+Shift+Esc` for
  `org.kde.plasma-systemmonitor.desktop`, and `Super` plus arrows for quick tiling. Which of those
  are already defaults is determined empirically, not assumed.
- **P1-D04** Dolphin: editable location bar on by default, to match the Explorer path box. Keep the
  `F4` embedded terminal. Do **not** ship a single-click override — Plasma 6 has defaulted to
  double-click since 6.0, so the setting is already correct and an explicit override only adds a key
  to maintain. Note that `SingleClick` lives in `kdeglobals` under `[KDE]`, not in `dolphinrc`, if it
  ever does need setting.
- **P1-D05** Theming is Breeze for Qt and Breeze-GTK for GTK, set explicitly so GTK applications
  follow the Plasma colour scheme. No Kvantum, no xsettingsd, no theme-syncing daemon. Static theme
  selection is one config key; a theme engine is a dependency and a running process.
- **P1-D06** Deliver defaults by the rule *anything Tundra keeps controlling ships in the image*.
  `/etc/skel` carries stubs only. The panel and desktop layout ship as a Look-and-Feel package at
  `/usr/share/plasma/look-and-feel/org.tundra.desktop/` with
  `contents/layouts/org.kde.plasma.desktop-layout.js`, the supported mechanism for a distro default
  panel. Remaining KDE config goes to `/etc/xdg/`, where KDE reads it through `XDG_CONFIG_DIRS` as
  defaults that `~/.config` overrides, so a user's change wins and a later image update still reaches
  everyone who has not touched that key. The real zsh config lives in the system-wide zsh file, with
  `/etc/skel/.zshrc` reduced to a customization stub. Do not use Kiosk immutability markers (`[$i]`):
  the audience is technically literate and locking settings generates support load. This is what
  makes the defaults updatable at all — `/etc/skel` applies only at user creation and reaches nobody
  who already has an account.
- **P1-D07** Shell discipline on the pilot: leave `/bin/sh` alone (P1-C02) and enforce POSIX
  correctness through tooling. Every script is checked with `shellcheck -s sh`,
  `devscripts-checkbashisms`, and parsed by both `dash` and `busybox ash`. This catches bashisms
  before they reach Alpine, which was the point of the original dash idea, without putting `dnf` at
  risk.
- **P1-D08** The zsh configuration is hand-written against zsh's own modules: `compinit` for
  completion and `vcs_info` for git status in the prompt. No Starship, no Powerlevel10k, no Oh My
  Zsh. A prompt is not worth a binary dependency or a framework's update surface.
- **P1-D09** The Fedora package delta. Install: `zsh`, `dash`, `busybox`, `ShellCheck`,
  `devscripts-checkbashisms`, and the virtualization stack from P1-D10. Remove: `plasma-discover`,
  `plasma-discover-notifier`, `PackageKit`, `baloo-file` and the PIM stack (`akonadi-*`, `kmail`,
  `korganizer`) where present. Install `plasma-desktop` rather than `plasma-desktop-meta`. Every
  entry carries a reason, because this list is the input to Phase 2's Alpine selection and an
  unexplained entry cannot be translated.
- **P1-D10** Virtualization: `virt-manager`, `qemu-kvm`, `libvirt-daemon-kvm`,
  `libvirt-daemon-config-network` and `edk2-ovmf`. Enable `libvirtd` and add the user to the
  `libvirt` group. Do **not** add the user to `kvm`; on modern Fedora udev handles `/dev/kvm`
  permissions and the group membership is noise.
- **P1-D11** The capture set is at minimum the panel layout, `kglobalshortcutsrc` (as a delta per
  P1-D03), `kdeglobals`, `dolphinrc`, `kwinrc`, and `gtk-3.0/settings.ini` with
  `gtk-4.0/settings.ini`. Each capture records the Plasma version per P1-C04. P1-D06 decides how
  each one is delivered; the panel layout in particular never ships as a copied
  `plasma-org.kde.plasma.desktop-appletsrc`.
- **P1-D12** Repo layout. `skel/` for the `/etc/skel` stubs, `xdg/` mirroring `/etc/xdg`,
  `look-and-feel/org.tundra.desktop/` for the global theme package, `shell/` for the system zsh
  config, `packages/` for the P1-D09 lists, `scripts/` for the apply and capture scripts, and
  `docs/` for the captured-key provenance record from P1-O04. The tree mirrors its install
  destinations so that applying it is a copy rather than a translation.
- **P1-D13** The shared zsh config carries a small set of transition aliases — `cls` for `clear`,
  `ipconfig` for `ip -color a` — to absorb muscle-memory misses without shadowing any real command.
  Keep the list short; this is a courtesy, not a compatibility layer.
- **P1-D14** The pilot installs its applications as Flatpaks, not RPMs. Without that, the portal
  and PipeWire checks test nothing. It also surfaces the Flatpak-versus-host theming problems while
  there is still a package manager available to work around them.

## Risks

- **P1-R01** *Fedora 44's Plasma Setup first-run wizard may overwrite seeded defaults.* Plasma Setup
  is new in this release and its interaction with a seeded `/etc/skel` and `/etc/xdg` is unknown.
  Mitigation: P1-V09 explicitly records whether the wizard ran and what it changed.
- **P1-R02** *`dash` and BusyBox `ash` are not the same shell.* They diverge on `local`, `echo`
  behaviour and several builtins, so validating against dash alone lets real breakage through.
  Mitigation: P1-D07 parses with both. `busybox` 1.37.0 is packaged in Fedora 44; if the build omits
  the `ash` applet, fall back to a container for that check.
- **P1-R03** *Plasma's `/etc/xdg` defaults coverage is not uniform.* Some KCMs write configuration
  keys they do not read back as system defaults, so a value placed in `/etc/xdg` may be silently
  ignored while the identical value in `~/.config` works. Mitigation: P1-V11 establishes empirically
  which keys take. Anything that does not take falls back to `/etc/skel` and is recorded as reaching
  new accounts only.
- **P1-R04** *Plasma version skew between the pilot and Alpine.* Alpine v3.24 currently carries
  `plasma-desktop` 6.6.6 against Fedora 44's 6.6.4, so configs should move cleanly today. That
  parity is incidental and will drift. Mitigation: the provenance discipline in P1-C04, and
  re-verification when skew exceeds one minor release.
- **P1-R05** *The pilot machine is not the deliverable.* The characteristic failure mode of this
  phase is spending the time making one Fedora install pleasant to use rather than making the
  configuration reproducible. A setting changed by hand in System Settings and never captured is
  work that has to be done twice. Mitigation: P1-V10 is the gate that catches it, and it should be
  run early and often rather than once at the end.

## Verification

- **P1-V01** `readlink -f /bin/sh` returns a path ending in `bash`. This confirms P1-C02 is intact;
  the pilot must *not* have repointed it.
- **P1-V02** For every script in the repo, `shellcheck -s sh`, `checkbashisms`, `dash -n` and
  `busybox ash -n` all exit 0. Run as a pre-commit hook, not by hand.
- **P1-V03** `busybox --list | grep -x ash` prints `ash`.
- **P1-V04** `getent passwd "$USER" | cut -d: -f7` returns the zsh path. Do not use `echo $SHELL`;
  it reflects the login environment and will lie in an already-open terminal.
- **P1-V05** `virsh -c qemu:///system list --all` as an unprivileged user returns a VM list without
  a permission error or a root password prompt. The URI is mandatory: bare `virsh` defaults to
  `qemu:///session` for non-root and returns an empty list even with no system access at all.
- **P1-V06** `rpm -q plasma-discover PackageKit baloo-file` reports all three as not installed, and
  a subsequent `dnf upgrade` does not pull them back in.
- **P1-V07** A Flatpak application launched from the Kickoff menu opens a file picker (portals work)
  and plays audio (PipeWire works).
- **P1-V08** Every binding in the P1-D03 delta fires, and the stock bindings the delta deliberately
  leaves alone still fire too.
- **P1-V09** Create a fresh user after seeding the defaults, log in, and confirm the panel layout,
  shortcuts and theme match the reference account. Record whether Plasma Setup ran and what it
  changed (P1-R01).
- **P1-V10** The repo applies to a clean Fedora KDE 44 install from a git checkout with no manual
  steps. This is the gate that defines P1-O01 and the one most worth running repeatedly.
- **P1-V11** Seed the defaults into `/etc/xdg`, create a fresh user, log in, change nothing, and
  diff the resulting `~/.config` against the intended values. Every key that did not take is a
  P1-R03 instance and goes into the provenance record.
- **P1-V12** With `org.tundra.desktop` installed and set as the default global theme, a fresh user
  gets the intended panel layout on first login, with no file in `/etc/skel` naming a Plasma applet.

## Definition of done

Phase 1 is complete when all of the following hold:

1. P1-O01 through P1-O08 exist in the repo.
2. P1-V01 through P1-V12 pass on a clean Fedora KDE 44 install performed from the repo.
3. The provenance record from P1-O04 lists every captured key, the Plasma version it was captured
   against, and whether it takes from `/etc/xdg` or needs `/etc/skel`.
4. P1-Q01 is answered, because Phase 2 implements it.
5. The design has been in daily use long enough that P1-O02 reflects what works rather than what
   sounded right.

## Work sequence

1. Install Fedora KDE 44. Capture the stock `kglobalshortcutsrc`, `kdeglobals` and panel state
   before changing anything — this baseline is what makes P1-D03's delta approach possible, and it
   cannot be recovered later.
2. Apply the package delta (P1-D09, P1-D10) and verify P1-V01, P1-V03, P1-V05, P1-V06.
3. Build the desktop design (P1-D02 through P1-D05) by hand, then use it. Do not start capturing
   until it has stopped changing.
4. Stand up the repo layout (P1-D12) and the lint harness (P1-D07, P1-V02).
5. Write the zsh configuration (P1-D08, P1-D13).
6. Capture (P1-D11), build the Look-and-Feel package, and split delivery per P1-D06.
7. Run P1-V09 through P1-V12 against a fresh user, then P1-V10 against a clean install. Iterate
   until both pass without hand edits.

## Open questions

- **P1-Q01** How do Flatpak applications get updated? Phase 2 removes Discover and every store
  frontend, which leaves no answer for the thing users update most often. Options: a background
  timer plus a notification, a small Tundra CLI wrapper, or reinstating one graphical frontend for
  Flatpak alone. The "no app store" position is a host-minimalism decision rather than a
  user-experience one, and asking people to run a command to get browser security updates does not
  fit the audience. Decide it here, where it can be tried on a real desktop, even though it ships
  in Phase 2.
