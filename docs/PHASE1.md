# Phase 1 — Fedora pilot

**Status:** building. The pilot VM exists and `tundra-pilot` carries the artifact tree, the script
corpus and the stock baseline. The desktop design has not been driven through its task checklist,
and nothing has been applied to a clean install yet.
**Repo:** artifacts live in `tundra-linux/tundra-pilot`. These documents live in
`tundra-linux/planning`.
**Reference platform:** Fedora KDE Plasma Desktop 44 — Plasma 6.6.4, KDE Gear 25.12.3, KDE
Frameworks 6.25.0. Read from the running pilot with `rpm -q` on 2026-09-18. Note that
`plasmashell --version` aborts without a display, so the package query is the reliable oracle when
reading this over ssh.
**Pilot host:** a VMware Workstation guest for the desktop work — EFI, 4 vCPU, 8 GB, with 3D
acceleration and an audio device, both of which P1-V08 needs — plus a vSphere guest that exists
only to clear the virtualization gate (P1-D02).

Phase 1 does not produce an operating system. It produces a version-controlled set of
configuration artifacts, validated on Fedora KDE Plasma Desktop 44, that Phase 2 consumes as the
Tundra default profile. The pilot machine is scaffolding; the repo is the deliverable.

The reason to do this on Fedora rather than directly on Alpine is that desktop design and OS
engineering are separate problems, and fighting both at once is how this kind of project stalls.
Fedora gives a working Plasma stack for free, so the only variable is the design itself.

See [PLAN.md](PLAN.md) for the project overview and [PHASE2.md](PHASE2.md) for what happens to
these artifacts. Phase 2's Track 0 spikes run in parallel with this phase, not after it.

---

## Where the phase stops

The dividing line is the init system. Phase 1 owns everything above it and nothing below.

```
  Phase 1              desktop configuration, shortcuts, theming, fonts, MIME defaults
  (this document)      applications as Flatpaks
                       rootless Podman and Distrobox
                       KVM and virt-manager
                       network shares, secrets, printing, Bluetooth
                       the zsh configuration and the script corpus
 ─────────────────────────────────────────────────────────────────────────────────────
  Phase 2              init and service supervision, seat and session management
  (PHASE2.md)          musl, the Alpine base, the package overlay
                       EROFS root, A/B slots, RAUC, the installer
```

Two rules follow from that line, and between them they settle most of the small questions this
document would otherwise have to answer one at a time.

**No effort goes into making Fedora behave like an OpenRC system.** Fedora runs systemd and the
pilot lets it. Plasma's non-systemd startup path is reachable here — `startkderc` carries a
`[General] systemdBoot` key that sends `startplasma` down the `plasma_session` route instead, read
from `startkde/startplasma.cpp` in `plasma-workspace` on 2026-09-14 — and the pilot deliberately
does not use it. Rehearsing the absence of systemd without OpenRC underneath tests a configuration
that will never ship. Phase 2's Spike A covers that ground on a real Alpine VM, which is where the
answers are worth having.

**Every artifact is shaped for its Alpine destination, and the pilot adapts.** Where the two
distributions lay files out differently, the artifact takes Alpine's shape and the apply script
does whatever Fedora needs to load it. The zsh configuration is the case that forced the rule.
Alpine's `zsh` reads drop-ins from `/etc/zsh/zshrc.d/*.zsh` and Fedora's has no equivalent, so the
artifact is a drop-in file and Fedora gets a sourcing hook (P1-D10). Writing it Fedora's way and
translating later is how a pilot produces work that has to be done twice.

## Outcomes

Each is an artifact that exists in the repo when Phase 1 is done.

- **P1-O01** A configuration repo that applies to a clean Fedora KDE 44 install with no manual
  steps and no undocumented prerequisites.
- **P1-O02** A Windows-ergonomics desktop design that has been driven through the task checklist in
  P1-D23 enough times to know it works: panel layout, shortcut set, file manager behaviour, theming.
- **P1-O03** An `org.tundra.desktop` Look-and-Feel package that produces the panel layout on a
  fresh user account.
- **P1-O04** An `/etc/xdg` defaults set, plus a per-key record of which keys Plasma actually honours
  as system defaults and which it silently ignores.
- **P1-O05** A POSIX-clean script corpus and the lint harness that keeps it clean.
- **P1-O06** The Tundra zsh configuration, as a drop-in file plus a user stub.
- **P1-O07** The Fedora 44 package delta, documented as install and remove lists with a reason and
  an Alpine counterpart against each entry, as the input to Phase 2's package selection.
- **P1-O08** A working KVM and `virt-manager` recipe: packages, groups, services, and the
  permissions model.
- **P1-O09** The desktop design written down as a specification independent of the configuration
  that implements it: the panel arrangement, the shortcut table, the file manager behaviours and the
  theming intent, each stated as a rule rather than as a KDE key. Phase 2 consumes the config files;
  [Phase 3](PHASE3.md) discards them and consumes only this, because a Look-and-Feel package and a
  `kdeglobals` are worthless the moment Plasma goes. It costs an afternoon during Phase 1 and saves
  rediscovering the design from a screenshot years later.
- **P1-O10** The default application set as a manifest of Flatpak references, installed system-wide
  by the apply script on a clean machine.
- **P1-O11** The presentation layer as artifacts: default application associations, fontconfig, icon
  and cursor themes, wallpaper and the branding that carries the Tundra name.
- **P1-O12** A container recipe: rootless Podman and Distrobox working for an unprivileged user,
  with the subuid, subgid and storage-location decisions recorded.
- **P1-O13** A network-shares and secrets recipe: SMB browsing from Dolphin, with credentials stored
  and unlocked in a way that does not prompt twice a session.
- **P1-O14** A printing and Bluetooth recipe: packages, services, and the Plasma modules that expose
  them, with the hardware-dependent half explicitly deferred to Phase 2.
- **P1-O15** A translation record mapping every file in the tree to its Alpine destination, which is
  what makes P1-C03 checkable rather than aspirational.

## Constraints

- **P1-C01** Everything intended to survive into Phase 2 must be XDG- or KDE-standard. Fedora
  tooling, RPM-specific paths and distribution-specific helpers do not transfer and must not appear
  in any captured artifact.
- **P1-C02** Phase 1 stops at the init system, as laid out above. It does not own init, service
  supervision, seat or session management, the image, or the update mechanism, and it does not
  simulate their absence.
- **P1-C03** Where Fedora and Alpine disagree on where a file goes, the artifact takes the Alpine
  shape and the pilot adapts. The adaptation lives in `scripts/apply.sh` and never in the artifact.
- **P1-C04** No systemd-specific mechanism appears in a captured artifact. `systemctl --user` units
  are out, and autostart goes through `~/.config/autostart/` per the XDG spec, because that is what
  will exist on Alpine. This is a rule about what gets captured, not about how the pilot boots.
- **P1-C05** `/bin/sh` stays `bash`. Fedora's RPM scriptlets assume bash and packaging policy states
  the assumption; repointing the symlink breaks package transactions, and the `bash` package owns
  the symlink so updates revert it anyway.
- **P1-C06** Every captured artifact records the Plasma version it was captured against, because KDE
  config keys move between releases and a file with no provenance cannot be safely replayed later.

## Decisions

- **P1-D01** Pilot on **Fedora KDE Plasma Desktop 44**. The installed pilot runs Plasma
  6.6.4-1.fc44, read with `rpm -q` on 2026-09-18, which is also the newest f44 build
  `mdapi.fedoraproject.org` reports. Rejected: Fedora Kinoite or a `bootc` image, which would also
  rehearse the image and atomic-update half of Tundra. Rejected because the pilot's job is to
  iterate on desktop configuration quickly, and an immutable base makes that loop slower. The cost
  is that Phase 1 rehearses none of the image machinery, which Phase 2 absorbs through its Track 0
  spikes.
- **P1-D02** The pilot runs in **two guests**: a VMware Workstation guest carrying everything except
  the virtualization gate, and a **vSphere guest carrying that one gate**. The split is forced,
  not chosen. Nested virtualization is unavailable under Workstation on the development host,
  because Windows 11 Enterprise there runs virtualization-based security with Credential Guard and
  HVCI active and the DeviceGuard policy marked `Locked`. That keeps the Hyper-V hypervisor resident,
  which puts Workstation into its ULM monitor mode, and ULM cannot pass AMD-V through to a guest:
  `vhv.enable` is inert and Workstation refuses the *Virtualize AMD-V/RVI* option outright at power
  on. Read from the host's `Win32_DeviceGuard` WMI class and
  `HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard` on 2026-09-16. The lock is a managed-endpoint
  policy rather than a local setting, so treat it as a fixed property of the machine. ESXi has no
  such layer and exposes hardware-assisted virtualization to the guest, which is what makes P1-V06
  reachable there. Three consequences follow. Everything that is not virtualization is verified on
  Workstation, which is the correct host for it regardless of the lock, because Workstation gives
  the guest 3D acceleration and an audio device and P1-V08 requires both. P1-V06 alone moves to
  vSphere, and is the only gate that does. Nothing hardware-shaped is verified on either, because
  both are still virtual machines: suspend and resume, backlight, wifi, discrete graphics, real
  printers and real Bluetooth adapters stay out of reach, and qualifying them remains entirely with
  Phase 2's hardware matrix (P1-R06). Rejected: moving the whole pilot to Hyper-V, which does
  support nested virtualization on AMD at Windows 11 24H2. It gives Linux guests no audio device and
  no 3D acceleration, so it would trade P1-V06 for P1-V08 and degrade the Plasma work this phase
  exists to do. Rejected: pursuing a VBS exemption, which is a request against a corporate security
  control with a long lead time and a likely refusal, to buy one gate.
- **P1-D03** Start on Fedora 44 and **rebase to Fedora 45 when it reaches general availability**,
  currently scheduled for 2026-10-20 with the beta on 2026-09-15, read from the Fedora 45 schedule
  on 2026-09-14. The rebase is not an interruption to absorb. It is the one chance this project gets
  to rehearse the base-rebase discipline Phase 2 commits to every six months under `P2-D15`, on a
  system where a bad outcome costs a VM snapshot. P1-V20 is the gate. Rejected: pinning to 44 for
  the whole pilot, which buys a stable provenance record and rehearses nothing. Rejected: starting
  on the 45 beta, which trades desktop-design time for beta triage.
- **P1-D04** Panel and taskbar: a single bottom panel with the Kickoff application launcher at left,
  an Icons-Only Task Manager in the middle, and the system tray with a digital clock at right. This
  is the Windows arrangement and Plasma does it natively, with no extension layer to maintain.
- **P1-D05** Shortcuts are captured as a **delta, not a set**. Dump the stock `kglobalshortcutsrc`
  from a fresh Fedora 44 account into `baseline/` before changing anything, then record only the
  entries that differ. Plasma already ships several of the intended bindings, and shipping a full
  copy of the file would freeze unrelated defaults and fight every future Plasma release. The target
  bindings are `Super+E` for Dolphin, `Super+R` for KRunner, `Super+D` to show the desktop,
  `Ctrl+Shift+Esc` for `org.kde.plasma-systemmonitor.desktop`, and `Super` plus arrows for quick
  tiling. Which of those are already defaults is determined empirically, not assumed.
- **P1-D06** Dolphin: editable location bar on by default, to match the Explorer path box. Keep the
  `F4` embedded terminal. Do **not** ship a single-click override — Plasma 6 has defaulted to
  double-click since 6.0, so the setting is already correct and an explicit override only adds a key
  to maintain. Note that `SingleClick` lives in `kdeglobals` under `[KDE]`, not in `dolphinrc`, if it
  ever does need setting.
- **P1-D07** Theming is Breeze for Qt and Breeze-GTK for GTK, set explicitly so GTK applications
  follow the Plasma colour scheme. No Kvantum, no xsettingsd, no theme-syncing daemon. Static theme
  selection is one config key; a theme engine is a dependency and a running process.
- **P1-D08** Deliver defaults by the rule *anything Tundra keeps controlling ships in the image*.
  `/etc/skel` carries stubs only. The panel and desktop layout ship as a Look-and-Feel package at
  `/usr/share/plasma/look-and-feel/org.tundra.desktop/`. Remaining KDE config goes to `/etc/xdg/`,
  where KDE reads it through `XDG_CONFIG_DIRS` as defaults that `~/.config` overrides, so a user's
  change wins and a later image update still reaches everyone who has not touched that key. Do not
  use Kiosk immutability markers (`[$i]`): the audience is technically literate and locking settings
  generates support load. This is what makes the defaults updatable at all — `/etc/skel` applies only
  at user creation and reaches nobody who already has an account.
- **P1-D09** The Look-and-Feel package is selected as the system default by `kdeglobals`
  `[KDE] LookAndFeelPackage=org.tundra.desktop`, which `/etc/xdg` supplies like any other cascading
  key. `plasmashell` reads that key and, finding no existing applet configuration for the user, runs
  `contents/layouts/org.kde.plasma.desktop-prelayout.js` followed by
  `contents/layouts/org.kde.plasma.desktop-layout.js` from the package; the default containment comes
  from `contents/defaults` under `[Desktop][org.kde.plasma.desktop]`. Read from
  `shell/shellcorona.cpp` in `plasma-workspace` on 2026-09-14. Field semantics for `metadata.json`
  are upstream API and are not transcribed here; see
  [KDE's Look and Feel documentation](https://community.kde.org/Plasma/lookAndFeelPackage). The one
  thing to get right locally is that the layout script is what supplies the default panel, so nothing
  in `skel/` ever names a Plasma applet (P1-V13).
- **P1-D10** The zsh configuration ships as a **drop-in file**, not as a replacement for the
  distribution's system zshrc. Alpine's `zsh` sources `/etc/zsh/zshrc.d/*.zsh` from its own
  `/etc/zsh/zshrc`, so the artifact is `shell/tundra.zsh` and on Tundra it is a file copy with no
  other change. Fedora has no such directory: its `zsh` is built with `--enable-etcdir=/etc`, giving
  `/etc/zshrc`, `/etc/zshenv` and `/etc/zprofile`, all RPM-owned and marked `%config(noreplace)`, and
  its `/etc/zshrc` sources only `/etc/profile.d/*.sh` and does so under `emulate -L ksh`, which is
  useless for zsh-specific configuration. The pilot therefore installs the same artifact to
  `/etc/zshrc.d/tundra.zsh` and appends one guarded sourcing loop to `/etc/zshrc`. That append is the
  single deliberate modification to a package-owned file in this repo, it is marked as such, and it
  does not exist on Tundra (P1-R07). Read from Fedora's `zsh.spec` and Alpine's `main/zsh` files on
  2026-09-14.
- **P1-D11** The zsh configuration itself is hand-written against zsh's own modules: `compinit` for
  completion and `vcs_info` for git status in the prompt. No Starship, no Powerlevel10k, no Oh My
  Zsh. A prompt is not worth a binary dependency or a framework's update surface.
- **P1-D12** The shared zsh config carries a small set of transition aliases — `cls` for `clear`,
  `ipconfig` for `ip -color a` — to absorb muscle-memory misses without shadowing any real command.
  Keep the list short; this is a courtesy, not a compatibility layer.
- **P1-D13** Shell discipline on the pilot: leave `/bin/sh` alone (P1-C05) and target **BusyBox
  `ash`** directly. Every script is checked with `shellcheck -s sh`, `devscripts-checkbashisms`, and
  parsed with `busybox ash -n`. `scripts/lint.sh` runs all three over every `#!/bin/sh` script in the
  tree and exits nonzero on the first failure; a `.git/hooks/pre-commit` that calls it is what makes
  the rule hold, since a check run by hand is a check that stops being run. The hook is installed by
  `scripts/apply.sh` so a fresh clone gets it without a separate instruction.
  Rejected: carrying `dash` as a second parser. `dash` is stricter than BusyBox `ash` and is the only
  tool in the old chain that rejected bashisms outright, but it is a shell that will never run a
  Tundra script, and the constructs it uniquely catches are almost entirely covered by
  `shellcheck -s sh` and `checkbashisms`. Testing against the shell that ships beats testing against
  a stricter one that does not.
  Fedora's `busybox` is a fair stand-in for Alpine's *as a parser*. The two builds' `ash`
  configuration differs in four options — `ASH_BASH_SOURCE_CURDIR`, `ASH_IDLE_TIMEOUT`, `ASH_MAIL`
  and `ASH_VERSION_VAR`, all of which Alpine enables and Fedora does not — and none of them changes
  how a script parses. The option that matters, `CONFIG_ASH_BASH_COMPAT`, is enabled in both, so
  `busybox ash -n` accepts `[[`, `source` and the other bash-compat extensions without complaint and
  the parser catches syntax errors rather than bashisms (P1-R02). Read from each project's BusyBox
  configuration on 2026-09-14.
- **P1-D14** Scripts use the **BusyBox command vocabulary**, not the GNU one. A script that runs on
  Tundra may use only applets Alpine's BusyBox provides, and only the flags that BusyBox implements.
  This is the half of shell portability that no parser checks: `sed -i` exists in BusyBox and
  `grep -P` does not, `find` has no `-printf`, and `date -d` takes far less than GNU's version.
  Fedora's `busybox` is the wrong oracle for this, which is the one thing that makes the rule need a
  mechanism rather than a habit. Its build enables around ninety applets Alpine's does not, including
  `ar`, `patch`, `xz`, `ed`, `man` and `w`, so a vocabulary check against it passes scripts that
  break on the target. It also omits a dozen that Alpine ships, including `lsblk`, `nsenter`,
  `fallocate` and `uuidgen`, so it fails scripts that are fine there. Both lists read from the two
  BusyBox configurations on 2026-09-14.
  The mechanism is therefore an **Alpine container**, not a local binary. `scripts/lint.sh` runs the
  vocabulary check and any executable check inside a stock `alpine` image through Podman, which
  P1-D19 already installs: that image is BusyBox, musl and `apk` with no GNU coreutils present, so
  the applet list is the target's and a GNU-only flag fails the way it would on Tundra. Only
  execution catches flag-level differences, so the static half is a command-word allowlist generated
  from `busybox --list` inside the container and checked against the scripts, and the executable half
  is whatever the script corpus can actually be run through.
  The four scripts in P1-D24 are pilot-only: they call `dnf` and `rpm`, they cannot run in an
  Alpine container, and they are exempt. The rule's subject is the P1-D27 update mechanism in
  `target/`, which ships on Tundra and runs under BusyBox. `scripts/lint.sh` distinguishes the two
  classes by directory, and the exemption is recorded rather than assumed.
- **P1-D15** The Fedora package delta. Install: `zsh`, `busybox`, `ShellCheck`,
  `devscripts-checkbashisms`, the virtualization stack from P1-D16, and the container stack from
  P1-D19. No `dash`, per P1-D13. Remove: `plasma-discover`, `plasma-discover-notifier`, `PackageKit`,
  `kf6-baloo-file`, and the PIM stack (`akonadi-*`, `kmail`, `korganizer`) where present. Two of
  those removals need their boundaries stated, because both look like they should take Plasma with
  them and neither does.
  `plasma-desktop` links `libpackagekitqt6.so.2`, which comes from `PackageKit-Qt6`, a Qt binding
  library that does not itself require the `PackageKit` daemon, so the daemon goes and the library
  stays. `plasma-desktop` links `libKF6Baloo.so.6`, which comes from `kf6-baloo-libs` rather than
  from `kf6-baloo-file`, so the indexer daemon is the separable part; `baloo-file` is not a Fedora
  package name at all. Dependency data read from `mdapi.fedoraproject.org` for f44 on 2026-09-14.
  Every entry in both lists carries a reason and an Alpine counterpart, because this list is the
  input to Phase 2's selection and an unexplained entry cannot be translated. There is no
  `plasma-desktop-meta` on Fedora — that name belongs to Alpine, which ships both it and
  `plasma-desktop-meta-elogind` — so the Fedora-side equivalent of avoiding it is installing from the
  KDE spin and pulling no extra comps groups.
- **P1-D16** Virtualization: `virt-manager`, `qemu-kvm`, `libvirt-daemon-kvm`,
  `libvirt-daemon-config-network` and `edk2-ovmf`. Enable `libvirtd` and add the user to the
  `libvirt` group. Do **not** add the user to `kvm`; on modern Fedora udev handles `/dev/kvm`
  permissions and the group membership is noise. Fedora's `libvirt-daemon-common` ships
  `/usr/share/polkit-1/rules.d/50-libvirt.rules`, which is what makes `libvirt` group membership grant
  password-less access, so the pilot needs no polkit artifact of its own. Read from `libvirt.spec` on
  2026-09-14. Whether Alpine ships the same rule is a Phase 2 question and is recorded as one in the
  translation record.
- **P1-D17** Applications ship as **Flatpaks**, installed system-wide from Flathub. The set is
  `org.chromium.Chromium`, `com.vscodium.codium`, `org.gimp.GIMP`, `org.libreoffice.LibreOffice`,
  `org.kde.okular` and `org.videolan.VLC`, all six confirmed present on Flathub on 2026-09-18. A
  browser, an editor and an image editor cover the pilot's verification needs; the last three are
  what a Windows migrant's first week actually contains, because a machine that cannot open a PDF,
  a spreadsheet or a video reads as broken rather than as minimal. The set stops there and Flathub
  covers the rest, which is one search away. Flatpaks live on the persistent partition rather than
  in the image, so the cost of each addition is install time and support surface, not image size.
  The manifest is `flatpak/apps.txt`, one reference per line, and the apply
  script adds the Flathub remote and installs from it. Without real Flatpaks on the machine the
  portal and PipeWire checks test nothing, and the Flatpak-versus-host theming problems stay
  invisible until Phase 2, where there is no package manager left to work around them with. No store
  frontend ships. `flatpak-kcm`, which is a permissions settings module rather than a store, is
  allowed.
- **P1-D18** The capture set is at minimum the panel layout, `kglobalshortcutsrc` (as a delta per
  P1-D05), `kdeglobals`, `dolphinrc`, `kwinrc`, `plasmarc`, `mimeapps.list`, the fontconfig drop-in,
  and `gtk-3.0/settings.ini` with `gtk-4.0/settings.ini`. Each capture records the Plasma version per
  P1-C06. P1-D08 decides how each one is delivered; the panel layout in particular never ships as a
  copied `plasma-org.kde.plasma.desktop-appletsrc`.
- **P1-D19** Containers: rootless **Podman** with **Distrobox**, both packaged on Fedora and in
  Alpine `community`. The decisions that transfer are the subuid and subgid ranges, the container
  storage location, and the default Distrobox image. The cgroup delegation that makes rootless work
  is systemd's job on Fedora and OpenRC's problem in Phase 2, so the pilot records what it depends on
  rather than how Fedora provides it.
- **P1-D20** Network shares and secrets: `kio-extras` supplies `smb://` browsing in Dolphin and
  `kdenetwork-filesharing` the outbound half, both present on Fedora and in Alpine `community`.
  Credentials go in KWallet. The decision that matters to a Windows migrant is that connecting to a
  share asks once and not again, so the artifacts are the wallet configuration and the Dolphin Places
  entries, not the samba client defaults.
- **P1-D21** Presentation defaults: default application associations in `mimeapps.list` pointing at
  the P1-D17 Flatpaks, a fontconfig drop-in naming the default sans, serif and monospace families,
  and the Breeze icon and cursor themes set explicitly. Wallpaper and the Tundra branding ship inside
  the Look-and-Feel package, which is the only place they can live and still reach a fresh account.
  Fedora's default font packages and Alpine's are different packages under different names, so the
  artifact names families and never packages.
- **P1-D22** Printing and Bluetooth: install CUPS and BlueZ with the Plasma modules that expose them,
  enable the services, and verify as far as a VM allows. The scheduler runs, a print queue accepts a
  job against a network or PDF printer, and the Bluetooth module loads and reports no adapter
  cleanly. Real device support is hardware qualification and belongs to Phase 2 (P1-R06).
- **P1-D23** The desktop design is validated against a **written task checklist**, not against a
  claim of daily use. The checklist lives at `docs/checklist.md` and lists what a Windows migrant
  does in the first week: find a file by name, extract an archive, connect to a network share, take
  and paste a screenshot, switch between two windows of the same application, change the default
  browser, connect to wifi, mount a USB stick, install an application. Each entry records the steps
  taken and whether the design got in the way. This replaces "used daily long enough to know it
  works" for two reasons. The pilot is a VM on a Windows host and would not honestly get that use. A
  checklist also survives being handed to a second person, and an impression does not.
- **P1-D24** Repo layout, in `tundra-pilot`. The tree mirrors its install destinations so applying it
  is a copy rather than a translation, and so Phase 2 can consume directories wholesale instead of
  rereading this document.

  ```
  tundra-pilot/
    baseline/                    stock config captured before any change, P1-D05. Never edited
    xdg/                       → /etc/xdg          system-wide KDE defaults
      kdeglobals                 theme, colour scheme, LookAndFeelPackage
      kwinrc                     tiling, window behaviour, shortcuts owned by KWin
      kglobalshortcutsrc         the P1-D05 delta only, never the whole file
      dolphinrc                  P1-D06
      plasmarc                   P1-D18
      mimeapps.list              P1-D21
      gtk-3.0/settings.ini       Breeze-GTK
      gtk-4.0/settings.ini       Breeze-GTK
    look-and-feel/
      org.tundra.desktop/      → /usr/share/plasma/look-and-feel/org.tundra.desktop/
        metadata.json            package identity; see the KDE docs for field semantics
        contents/defaults        default containment, theme, icons, cursor
        contents/layouts/org.kde.plasma.desktop-layout.js    the P1-D04 panel
        contents/previews/       screenshots for the theme picker
        contents/wallpapers/     P1-D21 branding
    fontconfig/
      60-tundra.conf           → /etc/fonts/conf.d/                           P1-D21
    shell/
      tundra.zsh               → /etc/zsh/zshrc.d/ on Tundra                  P1-D10, P1-D11
    skel/
      .zshrc                   → /etc/skel/.zshrc            customization stub only
    flatpak/
      apps.txt                   P1-D17, one Flatpak reference per line
    packages/
      install.txt                P1-D15; package, reason, Alpine counterpart
      remove.txt                 P1-D15
    scripts/                     pilot-only. Runs on Fedora, calls dnf and rpm, exempt from P1-D14
      apply.sh                   idempotent; what P1-V11 runs
      capture.sh                 pulls live config back into the tree, records Plasma version
      lint.sh                    P1-D13 and P1-D14, invoked by the pre-commit hook
      translate-check.sh         P1-V21
    target/                      ships on Tundra. BusyBox vocabulary only, P1-D14. The P1-D27
                                 update mechanism
    docs/
      provenance.md              P1-O04: per key, which Plasma version, and whether it takes
                                 from /etc/xdg or needs /etc/skel
      design.md                  P1-O09: the design as rules, not as KDE keys. The only
                                 artifact Phase 3 inherits
      translation.md             P1-O15: every file, its Alpine destination, and any adaptation
      checklist.md               P1-D23: the task checklist and its results
  ```
- **P1-D25** `scripts/apply.sh` is the only supported way to put this repo on a machine. It runs as
  root, takes no arguments in the default path, and changes nothing on a second run. It covers, in
  order: the package delta, `/etc/xdg`, the Look-and-Feel package, fontconfig, the zsh drop-in and
  its Fedora sourcing hook, `/etc/skel`, the Flathub remote and the application manifest, services
  and group membership, and the pre-commit hook. It never writes to any `~/.config`, because a script
  that edits the live user's configuration hides exactly the failures P1-V12 exists to find.
  `--dry-run` prints what it would change and exits. `scripts/capture.sh` is the other direction and
  the only thing that writes into the tree from a running system: it copies the P1-D18 set out of
  `~/.config`, diffs `kglobalshortcutsrc` against `baseline/`, and appends the running Plasma version
  to the provenance record.
- **P1-D26** Privilege escalation is **`doas`**, and `sudo` is not carried on Tundra. Alpine ships
  `doas` in `main`, which gets roughly two years of support; its `sudo` is in `community`, which is
  supported only on the newest stable branch, so choosing `sudo` would put the escalation tool on
  the short support cycle for no gain. The artifact is a single `/etc/doas.conf` permitting the
  `wheel` group with `persist`, identical on both systems: Fedora packages the same program as
  `opendoas` 6.8.2-10.fc44, read from `mdapi.fedoraproject.org` on 2026-09-18. The pilot does not
  remove Fedora's `sudo`, which `dnf` and RPM workflows expect. Rejected: Alpine's
  `doas-sudo-shim`, which provides a `sudo` command that calls `doas`. It would absorb pasted
  commands and muscle memory, at the cost of documentation that has to hedge about which name a
  reader has and an escalation path with two spellings. One name is worth more than the
  convenience.
- **P1-D27** Flatpak applications update through **an unattended periodic job, a login-time
  notification, and a CLI wrapper**, which Phase 2 implements. No single piece
  meets all three criteria. The periodic job is what gets a security fix onto a machine nobody is
  administering, which is the whole reason the question exists. The notification is what lets a
  user see what changed, and it has to be a separate program because the update runs as root and
  cannot reach anyone's session bus; it is an XDG autostart entry, not a user service unit, so it
  works on a system with no systemd. The wrapper is the manual path and is the same script the
  timer runs. The schedule takes the Alpine shape — `/etc/periodic/daily`, run by BusyBox `crond` —
  and the pilot adapts by generating a systemd timer in `scripts/apply.sh`, so nothing
  systemd-shaped enters an artifact (P1-C04). Total mechanism is three short scripts, which is what
  keeps it affordable under `G-C01`. Rejected: reinstating one graphical frontend for Flatpak
  alone, which costs no code but contradicts the no-store-frontend position and drags PackageKit
  back in. Rejected: a CLI wrapper alone, which is the cheapest option and fails the first
  criterion outright.

## Risks

Severity is the cost if the risk lands, not the odds of it landing.

| # | Risk | Severity | Mitigation |
|---|---|---|---|
| P1-R01 | **Fedora 44's Plasma Setup first-run wizard may overwrite seeded defaults.** Plasma Setup is new in this release and its interaction with a seeded `/etc/skel` and `/etc/xdg` is unknown. | Medium | P1-V10 explicitly records whether the wizard ran and what it changed |
| P1-R02 | **No parser in the chain rejects a bashism.** Both distributions build BusyBox with `CONFIG_ASH_BASH_COMPAT=y`, so `busybox ash -n` parses `[[` and `source` happily, and dropping `dash` (P1-D13) removes the one tool that did not. The parse step now catches syntax errors and nothing else. | Medium | `checkbashisms` and `shellcheck -s sh` are the tools doing this work and both run on every script. The exposure is a bashism that passes both and then breaks on a BusyBox built without bash-compat, which is a configuration Tundra controls and can simply not adopt. If Fedora's `busybox` omits the `ash` applet (P1-V03), the container from P1-D14 covers the parse too |
| P1-R03 | **Plasma's `/etc/xdg` defaults coverage is not uniform.** Some KCMs write keys they do not read back as system defaults, so a value in `/etc/xdg` may be silently ignored while the identical value in `~/.config` works. | Medium | P1-V12 establishes empirically which keys take. Anything that does not falls back to `/etc/skel`, recorded in `docs/provenance.md` as reaching new accounts only |
| P1-R04 | **Plasma version skew between the pilot and Alpine.** The pilot carries Plasma 6.6.4 with KDE Gear 25.12.3; Alpine v3.24 carries Plasma 6.6.6 with KDE Gear 26.04.2. The pilot is behind on both, which is the safer direction — a key that works here generally still exists there — but it is not safe in general, because a key can be renamed as easily as added. Pilot read 2026-09-18, Alpine 2026-09-14. | Medium | The provenance discipline in P1-C06 and the translation record in P1-O15. Every captured key gets checked against the Alpine package version before Phase 2 consumes it. The P1-D03 rebase to Fedora 45 widens this gap rather than closing it |
| P1-R05 | **The pilot machine is not the deliverable.** The characteristic failure mode of this phase is making one install pleasant to use rather than making the configuration reproducible. A setting changed by hand in System Settings and never captured is work that has to be done twice, and it is invisible, because the machine looks right. | High | P1-V11 is the gate that catches it, and it is worth running weekly from the start rather than once at the end. `scripts/capture.sh` exists so that capturing is cheaper than not capturing |
| P1-R06 | **A VM proves nothing hardware-shaped.** Suspend and resume, backlight, wifi, discrete graphics, real printers and real Bluetooth adapters are untestable on the pilot, and they are where a desktop distribution usually breaks. | Medium | Accepted deliberately in P1-D02. P1-D22 verifies these subsystems only as far as services and panels, and the hardware half is named as Phase 2 work under `P2-D17` rather than left to be discovered there |
| P1-R07 | **The pilot writes to a package-owned config file.** Fedora's `zsh` owns `/etc/zshrc` and `/etc/skel/.zshrc` as `%config(noreplace)`; the apply script appends to the first and replaces the second. `noreplace` means updates leave the modification alone and drop an `.rpmnew` beside it, so nothing breaks quietly, but `rpm -V zsh` reports both files forever. | Low | P1-V14 asserts the append is present exactly once and that no other package-owned file is modified. The condition does not exist on Tundra, where the artifact is a plain drop-in |
| P1-R08 | **The Fedora 45 rebase moves Plasma underneath the provenance record.** Every key captured against 6.6.4 is re-validated after the rebase or it is a claim about a version the pilot no longer runs. | Medium | P1-V20 makes the rebase a gate with a recorded diff rather than an event that happens to the machine |
| P1-R09 | **The gates are cleared on two hosts that can drift apart.** P1-V06 runs on a vSphere guest and everything else on the Workstation pilot (P1-D02). A package delta applied to one and not the other makes P1-V06 a statement about a system that is not the pilot, and the failure is quiet because both machines pass their own gates. | Low | Both guests are built from the same repo checkout by `scripts/apply.sh`, so the virt stack under test is the P1-D16 one on either. The vSphere guest is disposable and rebuilt rather than maintained, which is cheaper than keeping two machines in step. The provenance record names the host that cleared each gate |

## Verification

Each gate is a command and a pass condition.

- **P1-V01** `readlink -f /bin/sh` returns a path ending in `bash`. This confirms P1-C05 is intact;
  the pilot must *not* have repointed it.
- **P1-V02** `scripts/lint.sh` exits 0, having run `shellcheck -s sh`, `checkbashisms` and
  `busybox ash -n` over every `#!/bin/sh` script in the tree, and having run the P1-D14 vocabulary
  check inside a stock `alpine` container over every script not marked pilot-only. A script using a
  command absent from `busybox --list` in that container fails the gate. Run as a pre-commit hook,
  not by hand.
- **P1-V03** `busybox --list | grep -x ash` prints `ash`.
- **P1-V04** `getent passwd "$USER" | cut -d: -f7` returns the zsh path. Do not use `echo $SHELL`; it
  reflects the login environment and will lie in an already-open terminal.
- **P1-V05** `virsh -c qemu:///system list --all` as an unprivileged user returns a VM list without a
  permission error or a root password prompt. The URI is mandatory: bare `virsh` defaults to
  `qemu:///session` for non-root and returns an empty list even with no system access at all.
- **P1-V06** On the vSphere guest, with *Expose hardware assisted virtualization to the guest OS*
  enabled, `test -c /dev/kvm` succeeds and a guest created in `virt-manager` reaches a login prompt.
  This is the only gate that does not run on the Workstation pilot (P1-D02). It is also what stops
  P1-V05 from standing in for working virtualization: P1-V05 exercises the libvirt socket and group
  wiring and passes unchanged on a host with no hardware virtualization at all.
- **P1-V07** `rpm -q PackageKit plasma-discover kf6-baloo-file` reports all three as not installed,
  `rpm -q PackageKit-Qt6 kf6-baloo-libs` reports both as installed, and a subsequent `dnf upgrade`
  pulls none of the removed three back in. The second half matters: if those two libraries went with
  the removals, `plasma-desktop` went too.
- **P1-V08** A Flatpak application from the P1-D17 manifest, launched from the Kickoff menu, opens a
  file picker (portals work) and plays audio (PipeWire works).
- **P1-V09** Every binding in the P1-D05 delta fires, and the stock bindings the delta deliberately
  leaves alone still fire too.
- **P1-V10** Create a fresh user after seeding the defaults, log in, and confirm the panel layout,
  shortcuts and theme match the reference account. Record whether Plasma Setup ran and what it
  changed (P1-R01).
- **P1-V11** The repo applies to a clean Fedora KDE 44 install from a git checkout by running
  `scripts/apply.sh` once, with no manual steps, and a second run reports no changes. This is the
  gate that defines P1-O01 and the one most worth running repeatedly.
- **P1-V12** Seed the defaults into `/etc/xdg`, create a fresh user, log in, change nothing, and diff
  the resulting `~/.config` against the intended values. Every key that did not take is a P1-R03
  instance and goes into the provenance record.
- **P1-V13** With `org.tundra.desktop` installed and named in `kdeglobals` under
  `[KDE] LookAndFeelPackage`, a fresh user gets the intended panel layout on first login, and
  `grep -ri plasma /etc/skel` finds no file naming a Plasma applet.
- **P1-V14** After `scripts/apply.sh` has run twice, `grep -c tundra /etc/zshrc` returns 1, a fresh
  interactive zsh shows the Tundra prompt and resolves the P1-D12 aliases, and `rpm -Va zsh` reports
  modifications to `/etc/zshrc` and `/etc/skel/.zshrc` and to nothing else (P1-R07).
- **P1-V15** `podman run --rm alpine true` succeeds as an unprivileged user and `podman info` reports
  cgroup version 2 and rootless mode. `distrobox create` followed by `distrobox enter` produces a
  shell in which `ldd --version` names glibc.
- **P1-V16** Dolphin opens an `smb://` share, the credential is stored, and reconnecting after a
  logout and login does not prompt again (P1-D20).
- **P1-V17** For each type in the P1-D21 set, `xdg-mime query default <type>` returns the intended
  desktop ID, and opening a file of that type from Dolphin launches the Flatpak rather than a host
  application.
- **P1-V18** `lpstat -r` prints that the scheduler is running, a queue added against a network or PDF
  printer accepts a job, and the Bluetooth module in System Settings loads and reports the absence of
  an adapter without erroring (P1-D22).
- **P1-V19** On a clean install, `scripts/apply.sh` leaves every reference in `flatpak/apps.txt`
  present in `flatpak list --system --app` and visible in the Kickoff menu.
- **P1-V20** After rebasing the pilot to Fedora 45, `scripts/apply.sh` runs clean and P1-V01 through
  P1-V19 and P1-V22 through P1-V23 pass, P1-V06 on a vSphere guest rebased alongside it and the
  rest on the Workstation pilot (P1-D02). Naming the split matters here: this gate is stated as a
  range, and a range silently asserts that one machine can clear all of it. Every gate that needed
  a change in order to pass is recorded in the provenance record against the new Plasma version
  (P1-D03, P1-R08).
- **P1-V21** `scripts/translate-check.sh` exits 0, having confirmed that every file in the tree
  outside `baseline/` and `docs/` has an entry in `docs/translation.md` naming its Alpine
  destination. This is what makes P1-C03 enforceable.
- **P1-V22** `doas -C /etc/doas.conf` reports the file parses, a member of `wheel` runs
  `doas id -u` and gets `0` after one password prompt, a second `doas` inside the persist window
  does not prompt again, and a user outside `wheel` is refused (P1-D26).
- **P1-V23** With the P1-D27 mechanism installed, running the update by hand reports what it
  changed and writes the state file; a login after it produces exactly one desktop notification
  naming the changed applications; a second login with no intervening update produces none. On the
  pilot, `systemctl list-timers tundra-update.timer` shows it scheduled.

## Definition of done

Phase 1 is complete when all of the following hold:

1. P1-O01 through P1-O15 exist in the repo.
2. P1-V01 through P1-V23 pass on a clean Fedora KDE install performed from the repo: on Fedora 45 if
   it has shipped, on Fedora 44 if it has not. P1-V06 passes on the vSphere guest and the rest on
   the Workstation pilot (P1-D02); the provenance record names which host cleared each.
3. The provenance record from P1-O04 lists every captured key, the Plasma version it was captured
   against, and whether it takes from `/etc/xdg` or needs `/etc/skel`.
4. The translation record from P1-O15 covers every file in the tree.
5. The P1-D27 update mechanism works end to end, because Phase 2 ships it rather than designing it.
6. The task checklist in P1-D23 has been run end to end at least twice, on separate iterations of the
   design, with results recorded both times.

## Work sequence

1. Check the development host for locked virtualization-based security before building anything.
   That single fact decides whether P1-V06 runs on the pilot or needs a second guest (P1-D02), and
   it is far cheaper to learn now than at step 5.
2. Build the pilot VM: Fedora KDE 44 on VMware Workstation, UEFI firmware. Snapshot it clean on
   first boot, before any configuration. Every later step that wants a clean install reverts to that
   snapshot instead of reinstalling, which is what makes P1-V11 cheap enough to run weekly (P1-R05).
3. Capture the stock `kglobalshortcutsrc`, `kdeglobals` and panel state into `baseline/` before
   changing anything. This baseline is what makes P1-D05's delta approach possible, and it cannot be
   recovered later.
4. Apply the package delta (P1-D15, P1-D16, P1-D19) and verify P1-V01, P1-V03, P1-V05, P1-V07.
5. Build the vSphere guest from the same checkout and clear P1-V06 there (P1-D02, P1-R09). Do it
   here rather than at the end. P1-V06 is what proves the P1-D16 recipe actually works, and finding
   out that it does not, after the package lists are frozen, is the expensive order to discover it
   in.
6. Install the Flatpak set (P1-D17) and verify P1-V08. Do this early: it is what makes every later
   check test the system users will actually have.
7. Build the desktop design (P1-D04 through P1-D07) by hand, then run the task checklist against it
   (P1-D23). Do not start capturing until the design has stopped changing.
8. Stand up the repo layout (P1-D24), the apply and capture scripts (P1-D25), and the lint harness
   (P1-D13, P1-V02).
9. Write the zsh configuration (P1-D10 through P1-D12) and verify P1-V14.
10. Do the subsystem work: containers (P1-D19, P1-V15), shares and secrets (P1-D20, P1-V16),
    presentation defaults (P1-D21, P1-V17), printing and Bluetooth (P1-D22, P1-V18).
11. Capture (P1-D18), build the Look-and-Feel package (P1-D09), and split delivery per P1-D08.
12. Run P1-V10 through P1-V13 against a fresh user, then P1-V11 against a clean install. Iterate
    until both pass without hand edits.
13. Write the translation record and make P1-V21 pass.
14. When Fedora 45 ships, rebase and run P1-V20.

## Open questions

None. The three that stood here — the Flatpak update path, the size of the default application
set, and what escalates privilege — are settled in P1-D27, P1-D17 and P1-D26 respectively. All
three were cheap to decide now and expensive to change once documentation and scripts reference
them.
