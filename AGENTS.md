# AGENTS.md

Rules for working on the Tundra planning documents. This repo holds documents and no code.

## Documents

| File | Holds |
|---|---|
| [`PLAN.md`](docs/PLAN.md) | Index: what Tundra is, the repo map, project-wide items, and the audit of the source conversation the project began as |
| [`PHASE1.md`](docs/PHASE1.md) | The Fedora pilot, scoped as an artifact set |
| [`PHASE2.md`](docs/PHASE2.md) | The Alpine distribution, which consumes those artifacts |
| [`PHASE3.md`](docs/PHASE3.md) | The Tundra desktop, replacing KDE. Direction rather than design — hold it to a lower bar than the other two, and do not add verification gates that cannot be run |

Phase 1 artifacts are built in the sibling `tundra-pilot` repo. Phase 2 has no repo yet.

The phase boundary is by *time*, not by kind: each document carries its own what, how and when.
That is why `PHASE2.md` opens with a narrative section — nothing else supplies the "how it works"
layer, and the ledgers alone do not add up to a description of the system.

## Identifiers

Outcomes, constraints, decisions, risks, verification gates and open questions are numbered with a
phase prefix: `P1-D07`, `P2-V12`, `P3-Q01`. `G-` is project-wide and lives in `PLAN.md`. A new phase
means adding its prefix to the check regexes below, or every identifier it defines reads as dangling.

**Identifiers are positional, not durable.** They renumber whenever a document is reorganised. Two
consequences:

- Do not cite them from outside this repo. An identifier in a commit message, an issue title or a
  chat message rots silently the next time something is inserted.
- Inside these documents, cite them freely. A renumber rewrites every reference at once, and the
  check below catches anything missed.

If a durable reference is ever needed, use a slug rather than a number.

## Editing

**Keep the current state clean.** Rewrite, renumber and remove rather than accumulating struck-through
rows and revision logs. A document should read as though it had always said what it now says.

This works because history lives in git, which means it only works if changes are committed. Commit
per meaningful change with a message saying what changed and why. `git log -p docs/PHASE2.md` is the
record of superseded decisions; there is no other one.

**Cite what was verified.** A version number, a package repository or an upstream behaviour gets an
inline note of where it came from and when it was read. A later reader needs to know what to
re-check, not to trust a bare number. Package versions in these documents were read from
`pkgs.alpinelinux.org` on 2026-09-14.

**Do not transcribe upstream formats.** Link to them. Config file field semantics belong to their
projects and drift; what belongs here is the decision about how Tundra uses them.

## Structure

Ledger what gets referenced, prose what gets understood.

- Outcomes, constraints, decisions and verification gates are bullets. Each is a single assertion.
- Risks are tables with a severity column, because severity buried in a paragraph cannot be scanned.
- Architecture is prose with figures. It is understood once and shapes everything downstream;
  numbering it into assertions destroys the thing it is for.

A verification gate is a command with a stated pass condition, not a description of a desirable
state.

## Checks

Before committing a change that touches identifiers:

An identifier is *defined* in one of two forms: bolded at the head of a bullet (`**P1-D07**`), or as
the first cell of a risk-table row (`| P1-R01 |`). Both must be matched — a check that only knows
the bold form reports every risk as dangling. Run from the repo root:

```sh
DEF='\*\*\(P1\|P2\|P3\|G\)-[OCDRVQ][0-9]\+\*\*\|^| \(P1\|P2\|P3\)-[OCDRVQ][0-9]\+ |'
ID='\(P1\|P2\|P3\|G\)-[OCDRVQ][0-9]\+'

# no duplicate definitions, and each prefix sequence contiguous from 01
grep -oh "$DEF" docs/*.md | grep -o "$ID" | sort | uniq -d     # must be empty

# every referenced identifier is defined somewhere
grep -oh "$DEF" docs/*.md | grep -o "$ID" | sort -u > /tmp/defined
grep -ohr "$ID" docs/ *.md | sort -u > /tmp/used
comm -13 /tmp/defined /tmp/used                                # must be empty
```

The second check spans the root files too, because `README.md` and this file cite identifiers and
will rot silently when one is renumbered.

Relative links must resolve. The three documents sit together in `docs/` and link to each other by
bare filename; the root files reach them as `docs/<name>.md`.

## Out of scope

This repo does not adopt `quickwire/planning`'s `ENGINEERING.md`. That document scopes itself to
TypeScript stacks and its document rules differ from these deliberately.
