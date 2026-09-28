# Specification Conventions

Every file in `docs/spec/` follows these rules.

## Authority

When sources conflict, this order applies (highest first):

1. `docs/decisions.md` D1–D20. **D20 amends earlier decisions and overrides the design docs** where they differ. For example:
   - one download can feed several slots;
   - release selection is coverage-aware;
   - MySQL is dropped;
   - announce conditions, autograb rules and delay are in the PCD extension schema;
   - pairing uses a local socket with peer credentials.
2. `docs/design/open-issues.md`:
   - **Group A** resolutions are accepted.
   - **Group B** decisions are in its "Decision" column.
   - **Group C** items are proposed defaults, specified as open items (below).
3. `docs/design/*.md`, the design rationale for each area.
4. `docs/analysis/*.md` and `docs/verification/*.md`, as background and evidence. Verification reports override analysis reports.

Where a design doc says something that a decision or open-issues outcome has since changed, write the **decided** version. Never silently invent behaviour. If something is unspecified and needed, add it as an open item.

## Normative language

- Use **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY** in the sense of RFC 2119, in bold.
- Every normative statement is a numbered requirement: `**PREFIX-NNN** (MUST) Text.` Numbers are three digits, sequential within the file, and never reused.
- Prefixes by file:

| File | Prefix |
|---|---|
| 00-overview.md | OVR |
| 01-architecture.md | ARC |
| 02-domain-model.md | DOM |
| 03-acquisition.md | ACQ |
| 04-evaluation.md | EVL |
| 05-downloads.md | DLD |
| 06-import.md | IMP |
| 07-profiles.md | PRF |
| 08-metadata.md | MET |
| 09-subtitles.md | SUB |
| 10-identity-and-access.md | IAM |
| 11-api-and-clients.md | API |
| 12-platform.md | PLT |
| 13-deployment.md | DEP |
| 14-roadmap.md | RDM |
| 15-open-items.md | OPN |

- Each requirement MAY end with a traceability note in italics: `_Source: design/downloads.md §3.4; D20._`

## Open items

A Group C default, or any other still-unconfirmed choice, is written as a requirement that uses the default. It is tagged with an open-item reference in the form `[OPN-NNN]`, which points to an entry in `15-open-items.md`.

## Structure of each file

1. `# NN. Title`, followed by a one-paragraph scope statement and the phase(s) it applies to (P1/P2/P3).
2. **Concepts**: definitions of the terms used in the file, consistent with the glossary in `00-overview.md`.
3. **Requirements**, grouped by subsection.
4. **Data model**, where relevant: ent-style schema sketches (entities, fields, edges, indexes, uniqueness). Shared entities are *owned* by `02-domain-model.md`; other files reference them and define only their own.
5. **Flows**: numbered steps or text diagrams for the key workflows, including failure paths.
6. **Non-goals**: what is explicitly out of scope, with reasons, such as a dropped reference feature.
7. **Traceability**: a short table of what this file preserves from which reference app, and what it deliberately changes (D19).

## Canonical terms

- Domain entity names follow `docs/design/library.md`: Title, Item, SlotDefinition, TitleSlot, ItemSlot, MediaFile, Intent, Follow, ExternalID, and so on.
- Download terms follow `design/downloads.md`: Download, phase and retention.
- Indexer terms follow `design/acquisition.md`: Indexer, facet, Release.
- If two designs name the same concept differently, use the `library.md` name for domain entities and the owning design's name otherwise. Note the alias once in the Concepts section.

## Style

- Plain, concise technical prose. Tables for enumerations. No marketing language.
- Be concrete: state names, field names, defaults, limits and error codes.
- Cite reference-app behaviour only where it defines an invariant to preserve (for example, "parser results MUST match the ported Sonarr test corpus").
- Don't restate other files; cross-reference them by file and requirement ID.
- The project name is **bonnyread**.
- The license is AGPL-3.0.
- Supported databases are PostgreSQL ≥ 15 and SQLite.
