# Phase 3 — the Tundra desktop

**Status:** direction, not design. Nothing here is decided to the standard of `PHASE1.md` or
`PHASE2.md`.
**Alpine package versions** were read from `pkgs.alpinelinux.org` against v3.24 on 2026-09-14.

Phase 3 drops KDE Plasma and builds a modular desktop dedicated to Tundra: a Wayland session whose
shell is written for this project rather than adapted to it. The motivation is cohesion: a desktop
assembled from one set of decisions instead of configured into shape from a general-purpose one.

This document is deliberately weaker than the other two. It is direction: what the components should
be like and roughly how big, which parts are worth writing, and what has to be true for any of it to
work. It is not a plan, and nothing here is settled to the standard of the other phases. What is
decided is the *character* of the thing; almost every concrete choice is still an open question.
Treat the verification section as a description of what finished looks like rather than as gates
anyone can run today, because there is nothing to run them against.

See [PLAN.md](PLAN.md) for the project overview. Phase 3 replaces the desktop layer that
[PHASE2.md](PHASE2.md) ships. Everything below the session is unaffected: the image, the update
system, the package base.

**Nothing here changes Phase 1 or Phase 2.** Those two ship a Plasma desktop and are not waiting on
any of this. The single line running the other way is [PHASE1.md](PHASE1.md) P1-O09, which asks that
the desktop design be written down as rules rather than only as KDE configuration — worth an
afternoon during Phase 1 whether or not Phase 3 ever happens.

---

## What replacing KDE actually means

The direction as originally sketched named seven components: a session, a window manager, a bar, a
menu, a settings app, a terminal, a file manager and a text editor. That list is the visible half.

Plasma is not a window manager with a panel attached. It is roughly twenty services, most of which
are invisible until they are missing, and the desktop is unusable without them. This table is the
actual scope of "drop KDE", and it is the most useful thing in this document.

| Function | Plasma supplies | Phase 3 |
|---|---|---|
| Compositor and window manager | KWin | **write or adopt** — `P3-Q03` |
| Panel and taskbar | Plasma shell | **write** — carries the identity |
| Application launcher | Kickoff | **write** — carries the identity |
| Settings UI | System Settings | **write** — carries the identity |
| File manager | Dolphin | **write** — the largest single item, `P3-R04` |
| Text editor | Kate, KWrite | write or adopt |
| Terminal | Konsole | adopt — `foot` 1.27.0, community |
| **Desktop portals** | `xdg-desktop-portal-kde` | **unsolved** — `P3-R02`, the hard one |
| Polkit agent | `polkit-kde-agent` | adopt |
| Screen locker | `kscreenlocker` | adopt |
| Display and output config | KScreen | adopt or write |
| Network UI | `plasma-nm` | adopt or write |
| Audio UI | `plasma-pa` | adopt or write |
| Bluetooth | BlueDevil | adopt |
| Notifications | Plasma | adopt — `mako` or similar |
| Clipboard manager | Klipper | adopt |
| Idle and power management | PowerDevil | adopt |
| Secrets and keyring | KWallet | adopt |
| Removable media and automount | Solid, KIO | adopt or write |
| Screenshot and screen recording | Spectacle | adopt |
| Accessibility | Orca integration, KWin a11y | **unsolved** — `P3-R05` |

Four rows carry the reason to do this at all. The rest is plumbing that should be adopted rather
than written, and the discipline in `P3-D04` is what keeps this phase finite.

### What the suckless references mean

The direction names dwm, dwmblocks, dmenu and st. None of their code survives contact with Wayland,
and none of it is meant to: X11 clients talk to a display server that Wayland does not have, and
`PHASE2.md` P2-D10 keeps X11 out of the image anyway. The references are a statement about
**character and size**, and that is the part worth holding onto.

The character: one program doing one job, small enough to read in an afternoon, configured by a flat
file or a recompile rather than a settings framework, composing over pipes and standard protocols
instead of a private IPC bus, and useful on its own outside the desktop that ships it.

The size, as a budget rather than a rule:

| Component | Reference | Scope signal |
|---|---|---|
| Compositor | dwm, ~2000 SLOC | **Wayland changes this.** A compositor must also do what the X server did, so `dwl` sits at ~3200 SLOC against an aspirational 2200 that its maintainers say is unreachable for a complete compositor. Budget at Wayland scale |
| Bar | dwmblocks | Small. Status is text arriving on a pipe; the hard part is the taskbar, not the blocks |
| Launcher | dmenu, ~1000 SLOC | One job: filter a list, print the choice. The Windows-style start menu is a larger thing wearing this shape |
| Terminal | st, ~5000 SLOC | `foot` 1.27.0 already is this, in Wayland, in `community`. Adopt unless there is a reason not to |
| Settings | no analogue | The suckless tradition has no settings app, because it has no settings. This is where the style is genuinely tested against a Windows-expat audience |
| File manager | no analogue | `P3-R04`. Nothing in this tradition is the right size for it |
| Text editor | Notepad++ | Named for behaviour, not for scale |

The two rows with no analogue are the interesting ones. A desktop for people leaving Windows needs a
settings UI and a file manager, and the suckless answer to both is that you should not want them.
Phase 3 has to decide what minimal means for a component whose whole purpose is discoverability.

---

## The compositor landscape

Surveyed 2026-09-14. This moves fast, and packaging moves fastest — `mangowm` was rebuilt
two days before this was written. Re-check before deciding anything on it.

### Libraries, if writing one

| Library | Language | State | Notes |
|---|---|---|---|
| **wlroots** | C | 0.20.2 and 0.19.3, both Alpine v3.24 community | The default. Nearly everything below is built on it |
| **Smithay** | Rust | Powers COSMIC, stable 1.2.0 July 2026 | No longer speculative; ships in Pop!_OS |
| **aquamarine** | C++ | Hyprland's own since 0.42 | A backend abstraction for Hyprland, not a general-purpose wlroots replacement |
| **Louvre** | C++ | Niche | Exists, little adoption |
| **libweston** | C | Reference implementation | Rarely chosen for this |

wlroots breaks its API every release, which is why Alpine ships `wlroots0.19` and `wlroots0.20` as
separate parallel packages rather than one `wlroots`. A compositor written on it is ported forward
roughly every six months, permanently, on top of `P2-D15`'s rebase. Hyprland left over this friction
and built `aquamarine`; that took a team.

### Compositors, if starting from one

| Compositor | Window model | Alpine v3.24 | Notes |
|---|---|---|---|
| **labwc** 0.20.0 | Stacking, Openbox-like | community | The only packaged stacking option. Behavioural match for `P1-D02` |
| **MangoWM** 0.17.0 | dwm-style tags and layouts | **edge** community, built 2026-09-12 | dwl fork, stayed small, builds in seconds. Ships `mangobar`. Animations, blur, shadows via scenefx |
| **niri** 25.11 | Scrollable tiling | community | Rust on Smithay. Wrong window model, useful as proof the Rust path ships |
| **dwl** | dwm-style tiling | **not packaged** | The suckless reference, absent from Alpine entirely |
| **wayfire** | Stacking with effects | not in v3.24 | — |

Nothing packaged today is both stacking *and* suckless-sized. labwc is the right shape and larger
than `P3-D01` argues for; MangoWM is the right size and the wrong shape. That gap is the honest
argument for writing one, and it should be weighed against `G-C01` rather than won on style.

### Rust and wlroots do not combine

`wlroots-rs` is abandoned. Way Cooler's author spent a long time on it, gave up, and
[rewrote the compositor in C](http://way-cooler.org/blog/2019/04/29/rewriting-way-cooler-in-c.html).
Both reasons are structural rather than fixable: safe bindings over wlroots' C design mean either
unsafe everywhere, which forfeits the reason to use Rust, or a wrapper fighting the library; and
custom Wayland protocols, the standard way to extend a compositor, cannot be defined safely through
the bindings. Smithay exists because of this. It is not a Rust binding to wlroots, it is the Rust
answer to the same problem.

So the compositor's library and its language are one decision, not two. `P3-Q03`.

### The language question is two questions, not one

The compositor is a Wayland **server**. The bar, launcher, settings app and file manager are Wayland
**clients**. They sit on opposite sides of the protocol and share almost no code, which means the
constraint above does not propagate outward: a C compositor with Rust clients loses very little,
because there was never much to share between them.

The sharing worth having is *among the clients*: layer-shell and seat boilerplate, config loading,
theming, icon lookup, `.desktop` parsing, D-Bus plumbing. All four need most of it. One language
across those four with a shared internal library is a real win, and it is independent of what the
compositor is written in.

That is why `P3-D05` can defer the compositor decision without blocking anything else.

### Rust on musl is not the problem it looks like

`niri` is Rust, built on Smithay, packaged in Alpine v3.24 community and compiled against musl. The
whole path already ships in the exact configuration Tundra targets.

Two details rather than obstacles. Rust's `x86_64-unknown-linux-musl` target sets
`crt_static_default = true`, so it links musl statically by default; distro packaging wants dynamic
linking against the system musl, and turning that off has sharp edges — it pulls `libgcc_s`, which
Alpine does not install by default. Alpine's packaging already solves this, so it is inherited
rather than solved. And musl's allocator is slower than glibc's under some loads, which for a
compositor is worth measuring rather than assuming.

The build pipeline does not run on the deployed image. `abuild` runs in an Alpine environment, but
that environment is a container on any host. An `alpine:3.24` container on a CI runner is
standard practice, and it is the same shape as `P2-D15`'s edge-watching build with a different tag.
What does not work is skipping the Alpine environment entirely and cross-compiling glibc-to-musl
elsewhere: that is fine for a standalone static binary and wrong for a distro package, because the
package must link against the branch's actual library versions.

The tension Rust does carry is with `P3-D01`. Alpine's Rust packaging uses `options="net"` with
`cargo fetch --locked`, which is mechanically fine, but "small enough to read in an afternoon" does
not survive a large transitive crate tree, and `P2-C05` makes every one of those an implicit
obligation. If Phase 3 goes Rust, the minimalism claim has to mean *the program* is small, stated
plainly, because it will not mean the dependency graph is.

---

## Outcomes

- **P3-O01** Tundra boots to a Wayland session built for Tundra, with no Plasma component in the
  image.
- **P3-O02** The Windows ergonomics from Phase 1 survive the transition intact. A user moving from a
  Phase 2 machine to a Phase 3 machine finds the same panel, the same shortcuts, and the same file
  manager behaviour.
- **P3-O03** Every Flatpak application works exactly as it did under Plasma, file pickers and screen
  sharing included. This is a constraint on the design, not a hope.
- **P3-O04** The written components are independently useful. A bar, a menu and a settings app that
  only work inside Tundra are a maintenance burden; ones that run on any wlroots compositor can
  attract contributors.
- **P3-O05** The session is assembled from parts that can be replaced one at a time, so that no
  single component becoming unmaintained strands the desktop.

## Constraints

- **P3-C01** Everything in `PHASE2.md` still holds. Phase 3 changes the desktop layer and nothing
  beneath it: musl, OpenRC, the read-only root, A/B updates and the six-month rebase are unchanged.
- **P3-C02** Flatpak remains the only application channel (P2-D07), so desktop portals are a
  correctness requirement, not an integration nicety. A session without a working file-chooser
  portal cannot open a file in any application the user has installed.
- **P3-C03** The maintainer budget in `PLAN.md` G-C01 is unchanged and is the binding constraint on
  this phase more than any technical question.
- **P3-C04** Phase 3 cannot begin until Phase 2 has shipped and is being maintained on its cadence.
  Running a six-month rebase and writing a desktop environment at once is not a schedule, it is two
  projects.
- **P3-C05** Whatever compositor Tundra ships must advertise a foreign-toplevel protocol, or the
  taskbar cannot exist. Wayland deliberately does not let a client enumerate windows, so a taskbar
  needs `ext-foreign-toplevel-list-v1` (the standardised read-only one, which Smithay implements as
  `wayland::foreign_toplevel_list`), or `wlr-foreign-toplevel-management-v1` (the wlroots-ecosystem
  one, read/write), or compositor-specific IPC. This is a live failure mode rather than a
  theoretical one: generic window lists show an empty desktop against compositors that never
  advertise the protocol. It is a requirement `P3-D04`'s bar places back onto `P3-Q03`, and it is
  cheap to satisfy when writing a compositor and non-negotiable when adopting one.

## Decisions

Few, and provisional. Most of this phase is open questions.

- **P3-D01** **Components are built in the suckless character**, as described above: one job each,
  readable in an afternoon, flat-file or compile-time configuration, composing over standard
  protocols, and useful outside Tundra. This is the design position the whole phase rests on, and it
  is the part of the original direction that is actually decided. It governs what gets written, not
  what gets adopted: `foot` and `wlroots` are both larger than anything here would be, and that is
  fine.
- **P3-D02** Ship the Tundra session as a *selectable alternative* first, and make it default only
  when it is better than the Plasma session on the same machine. A desktop that must be finished
  before it can be tried never gets tried.
- **P3-D03** Solve portals before writing anything else. See `P3-R02`; this is the item most likely
  to be discovered late and most expensive to retrofit.
- **P3-D04** **Write the shell, adopt the plumbing.** The four components that carry Tundra's
  identity are written: bar, launcher, settings, file manager. Everything else in the table above
  is adopted from an existing project, even where the result is less cohesive than writing it. This
  is the decision that determines whether Phase 3 is finite. A project that writes its own
  notification daemon, clipboard manager and polkit agent has stopped building a desktop and started
  building a career.
- **P3-D05** **The compositor decision is deferred and blocks nothing.** It is the hardest question
  in the phase (`P3-Q03`) and the least urgent, because the compositor is a Wayland server while the
  four written components are Wayland clients: they share almost no code, so the clients can be
  designed, prototyped and even shipped against an existing compositor before the question is
  settled. The only thing the clients need from it is `P3-C05`. Deciding the compositor early buys
  nothing and forecloses the option of building on something like MangoWM or labwc rather than
  maintaining a compositor forever.

## Risks

Severity is the cost if the risk lands, not the odds of it landing.

| # | Risk | Severity | Mitigation |
|---|---|---|---|
| P3-R01 | **Phase 1's artifacts do not transfer.** The Look-and-Feel package, `kdeglobals`, `dolphinrc` and `kglobalshortcutsrc` are KDE-specific and become worthless the moment KDE goes. Phase 1's *design* survives; its *files* do not. | High | PHASE1.md P1-O09 makes the design a written specification independent of the config that implements it. Without that, Phase 3 restarts the Phase 1 design work from a screenshot |
| P3-R02 | **The portal gap breaks every application.** Dropping KDE drops `xdg-desktop-portal-kde`. `xdg-desktop-portal-wlr` covers screencast and screenshot only, not the file chooser, and the documented advice for self-assembled desktops is to fall back to `xdg-desktop-portal-gtk` — which drags in GTK, in a project whose reason for existing is cohesion. | High | P3-D03 makes this the first problem solved, not the last. P3-Q05 records the fork |
| P3-R03 | **Scope.** Twenty components against one maintainer. The visible seven are perhaps a third of the work, and the invisible thirteen are the ones users notice only by their absence. | High | P3-D04 is the only real control. If the adopt list shrinks during the build, the phase is failing and the schedule is the last thing to show it |
| P3-R04 | **A Windows-style file manager is a multi-year project by itself.** Dolphin is two decades of work on mounts, trash, archives, network transparency, thumbnails and previews. A file manager that handles only local directories will be the most-complained-about part of Tundra. | High | Consider adopting for v1 and writing later; P3-Q04 |
| P3-R05 | **Accessibility regresses to nothing.** Plasma has screen-reader integration and keyboard navigation a hand-rolled shell will not have by default, and retrofitting a11y is far harder than designing for it. This is a *toolkit* risk wearing a language costume: GTK and Qt carry mature AT-SPI, while AccessKit's AT-SPI adapter is still described as almost production-ready, so Rust toolkits lag specifically on Linux screen readers. | Medium | `P3-Q04` decides the toolkit with this as an explicit input rather than a discovery. Shipping a desktop a blind user cannot operate is a legitimate choice for a small project, but it should be stated rather than arrived at |
| P3-R06 | **A half-finished desktop is worse than Plasma.** The failure mode is not abandonment before starting; it is switching the default too early and running for years on something less capable than what it replaced. | High | P3-D02. The Plasma session stays shippable until the Tundra session wins on merit |
| P3-R07 | **wlroots breaks its API every release,** which is why Alpine ships `wlroots0.19` and `wlroots0.20` as parallel packages rather than one. A compositor written on it is ported forward roughly every six months, permanently, stacked on `P2-D15`'s rebase. Hyprland left wlroots over this and built its own backend; that took a team. | Medium | Falls entirely on the "write one" branches of `P3-Q03`. Building on labwc or MangoWM moves this cost to someone else, which is most of their appeal |
| P3-R08 | **Losing Plasma means losing its security maintenance.** Every written component becomes Tundra's to patch, on a desktop handling untrusted content. | Medium | Weigh per component in P3-D04. This is a real argument for adopting more and writing less |

## What finished looks like

Acceptance criteria, not runnable gates. There is nothing to run them against yet.

- **P3-V01** A Flatpak application opens a file picker, saves a file, and shares a screen, with no
  Plasma or GNOME component installed.
- **P3-V02** Every shortcut in the Phase 1 specification fires, and the panel matches the Phase 1
  design without reference to any KDE configuration file.
- **P3-V03** The session survives the Phase 2 update cycle: an A/B update replaces the image and the
  desktop comes back with user state intact.
- **P3-V04** A machine runs the Tundra session as default for a month of ordinary use without the
  maintainer reaching for the Plasma session.
- **P3-V05** Each written component builds and runs on a stock wlroots compositor outside Tundra
  (P3-O04).

## Open questions

- **P3-Q01** What does Plasma actually fail at? This is the question the whole phase rests on and it
  is not yet answered. "Cohesion" is a real motivation, but it needs to be stated as something
  observable: a behaviour Plasma cannot be configured into, a resource cost, a maintenance burden
  from P2-D15's rebases, or a licensing or identity requirement. Until it is written down, there is
  no way to tell whether Phase 3 succeeded, and no way to decide the trades in P3-D04.
- **P3-Q02** Does Phase 3 replace Plasma or ship beside it permanently? P3-D02 defers this, but a
  project maintaining two desktop sessions indefinitely has doubled its surface.
- **P3-Q03** **The compositor fork.** Write one or start from one, and in what language. These are
  one question, because Rust means Smithay and C means wlroots, with no supported path between them.
  Deferred by `P3-D05`; the consequences are what make it worth deferring rather than guessing.

  | Path | Gets you | Costs you |
  |---|---|---|
  | **Build on MangoWM** (dwl fork, C, wlroots) | Least work by a distance. Small, actively maintained, `mangobar` already exists as a starting point for the bar. Matches `P3-D01`'s size exactly | dwm-shaped: tags, layouts, tiling — the wrong model for `P1-D02`. Its scenefx blur and shadows pull against the restraint. In `edge`, so it needs v3.25+ or a slot in the `P2-D15` overlay cap |
  | **Build on labwc** (C, wlroots) | The only packaged stacking model, so the behaviour is right without a fight. In v3.24 community today. Someone else absorbs the wlroots API churn | Larger and more configurable than the style argues for. Tundra's identity lives in its config rather than its code |
  | **Write on wlroots** (C) | Exactly the behaviour and the character. Full control of `P3-C05` | The largest component in the phase, plus an API port-forward every six months on top of `P2-D15`. Weigh against `G-C01` |
  | **Write on Smithay** (Rust) | Memory safety in the process handling untrusted client input, where it is worth most. `niri` proves it ships on Alpine musl | Same scale as above with a smaller pool of prior art to copy, and the crate-tree tension with `P3-D01` |

  Both "build on" rows keep the option of writing one later. The reverse is not true, which is the
  argument for starting there even if the end state is a compositor of Tundra's own.

- **P3-Q04** What toolkit for the settings app and file manager? This is a larger decision than the
  language and it is where `P3-R05` actually lands. GTK and Qt have mature AT-SPI support; the Rust
  answer is AccessKit, whose AT-SPI adapter is described as almost ready for production, so Rust
  toolkits test well against Windows screen readers and lag specifically on Linux. Against that,
  GTK drags GTK into the image, which is the same irony as `P3-R02`'s portal fallback, and Qt drags
  back what Phase 3 just removed with Plasma. A Rust-native toolkit avoids both and costs the
  accessibility story. There is no free option, and the choice should be made deliberately rather
  than falling out of the language decision.
- **P3-Q05** Portal backend: adopt `xdg-desktop-portal-gtk` and accept GTK in the image, adopt
  `xdg-desktop-portal-wlr` and write only the missing file chooser, or write a backend outright.
- **P3-Q06** Does this need contributors, and therefore the open-source organisation in `PLAN.md`
  G-D01? A twenty-component desktop is not a solo project on any timeline that matters, and the
  answer changes how the components are licensed, structured and documented from the first commit
  rather than later.

## Sources

Surveyed 2026-09-14. Package versions and repository placement change fastest; re-check those before
acting on them.

- https://pkgs.alpinelinux.org/packages — wlroots 0.20.2/0.19.3, labwc 0.20.0, niri 25.11, foot
  1.27.0 (v3.24 community); mangowm 0.17.0 and mangobar 0.2.1 (edge community); dwl and wayfire
  absent from v3.24
- http://way-cooler.org/blog/2019/04/29/rewriting-way-cooler-in-c.html — why wlroots-rs was
  abandoned
- https://github.com/smithay/smithay
- https://github.com/YaLTeR/niri — Rust on Smithay, the musl existence proof
- https://github.com/mangowm/mango and https://wiki.archlinux.org/title/MangoWM
- https://labwc.github.io/
- https://hypr.land/news/independentHyprland/ — Hyprland leaving wlroots for aquamarine
- https://en.wikipedia.org/wiki/COSMIC_desktop — COSMIC 1.2.0, July 2026
- https://wayland.app/protocols/ext-foreign-toplevel-list-v1 and
  https://wayland.app/protocols/wlr-foreign-toplevel-management-unstable-v1 — P3-C05
- https://accesskit.dev/ — AT-SPI adapter status for Rust toolkits
- https://rust-lang.github.io/rfcs/1721-crt-static.html — the musl static-linking default
- https://wiki.alpinelinux.org/wiki/APKBUILD_examples:Rust — Alpine's Rust packaging convention
