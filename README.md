# planning — Tundra

The planning home for **Tundra**, an image-based Alpine-derived KDE Plasma desktop aimed at
technically literate people leaving Windows. Nothing here ships: this repo holds documents and no
code.

What Tundra is and where it is going: [`PLAN.md`](docs/PLAN.md). Rules for working on these documents:
[`AGENTS.md`](AGENTS.md).

## Repo map

| Path | What |
|---|---|
| [`PLAN.md`](docs/PLAN.md) | Index. What Tundra is, the repository map, project-wide decisions, and the audit of the conversation the project started as |
| [`PHASE1.md`](docs/PHASE1.md) | The Fedora pilot, scoped as an artifact set. Produces the desktop design; produces no operating system |
| [`PHASE2.md`](docs/PHASE2.md) | The Tundra distribution. Opens with how the system works, then consumes Phase 1's artifacts |
| [`PHASE3.md`](docs/PHASE3.md) | The Tundra desktop: dropping KDE for a modular Wayland session. Direction, not design |
| [`AGENTS.md`](AGENTS.md) | Document conventions: the identifier scheme, the rewrite-and-renumber rule, and the checks to run before committing |

## Sibling repositories

```
tundra-linux/
  planning/       this repo — docs/ holds the plan and the three phase documents
  tundra-pilot/   Phase 1 artifacts — the configuration tree in PHASE1.md P1-D24
```

Phase 2 has no build repository yet. `PHASE2.md` P2-D01 describes the `aports` overlay layout that
goes in one when it exists.

## Status

Phase 1 is building, and further along than the other two by a distance. `tundra-pilot` carries the
artifact tree and the script corpus, and all of it has been applied to the pilot VM: the package
delta, the defaults, the Flatpak set, the shell, `doas` and the update mechanism. A freshly created
account gets the intended panel, and every gate reachable without someone sitting at the machine
passes.

What is left needs a person or a second machine. The task checklist has never been run, so nobody
has judged the design by using it. The virtualization gate needs a vSphere guest, because the
development host runs locked virtualization-based security. And no decision anywhere here has
survived contact with real hardware. Phases 2 and 3 are still design.

The phases are not strictly sequential: `PHASE2.md` carries two Track 0 spikes — Plasma on
Alpine under OpenRC, and the EROFS plus RAUC image pipeline — that are meant to run *during* Phase 1,
because they cover the ground the Fedora pilot deliberately does not.

Phase 1 has no open questions left. Three of Phase 2's gate a release: there is no installer
decision (`P2-Q01`), no answer yet on whether the running root is verified or Secure Boot supported
(`P2-Q04`), and nothing yet opens an image or a plain text file on a host with no host applications
(`P2-Q05`).
