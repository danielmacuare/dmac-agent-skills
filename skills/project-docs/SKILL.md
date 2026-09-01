---
name: project-docs
description: Regenerate the Deutsche Bank SoW, discovery, gap, risk, and MoSCoW summaries for the Config Management and SWIM projects from tracked source files, skipping anything unchanged since the last run.
disable-model-invocation: true
---

This skill runs a **cascade** of five stages — sow → discovery → gaps → risks → moscow — where each stage's output feeds specific later stages (see each stage's Read line for exactly which). Run selected stages in numeric order; do not skip ahead within that selection.

## Configuration

Before touching the File registry, read `config.toml` (next to this file) for two machine-specific roots: `shared_root` and `outputs_root`. If `config.toml` doesn't exist, copy `config.example.toml` to `config.toml`, ask the user to confirm or correct the two paths for their machine, and update the file before continuing.

## Selecting stages

By default, running this skill means all five stages. The user may instead name one or more stages to run — `sow`, `discovery`, `gaps`, `risks`, `moscow` — and only those run.

A named stage still needs its prerequisites' output on disk: gaps needs sow's and discovery's output; moscow needs sow's, discovery's, and gaps' output. Risks has no such prerequisite — it only reads raw input files. If a named stage's prerequisite output is missing, add that prerequisite stage to the run and tell the user why, e.g.: "Also running Stage 2 (Discovery): Stage 3 (Gaps) needs its output and it isn't on disk yet." Never add a prerequisite whose output already exists, even if stale — that's the cache's job, not the selection's.

## File registry

Paths below are relative to the two roots resolved in Configuration: `{shared_root}` and `{outputs_root}`.

| ID | Item | Path |
|---|---|---|
| IN-01 | Config Management SoW | `{shared_root}/program/config-management/sow/World_Wide_Technologies - Statement of Work _GNS_Network_Automation_Programme_Configuration_Management 7_13_2026 1.docx` |
| IN-02 | SWIM SoW | `{shared_root}/program/swim-expansion/sow/World Wide Technology R-01017057 - Statment of Work GNS Automation Programme Expansion 7_15_2026 v1.2 Clean 2 1.docx` |
| IN-03 | Config Management meeting notes (folder) | `{shared_root}/program/config-management/meeting-notes` |
| IN-04 | SWIM meeting notes (folder) | `{shared_root}/program/swim-expansion/meeting-notes` |
| IN-05 | Shared meeting notes (folder, both projects) | `{shared_root}/program/shared/meeting-notes` |
| IN-06 | Open Design Register | `{shared_root}/program/shared/working-docs/DB - Open Design Register 2026-08-21.md` |
| OUT-01 | Discovery folder | `{outputs_root}/discovery` |
| OUT-02 | Gaps folder | `{outputs_root}/gaps` |
| OUT-03 | SoW folder | `{outputs_root}/sow` |
| OUT-04 | MoSCoW folder | `{outputs_root}/moscow` |
| OUT-05 | Cache file | `{outputs_root}/cache.json` |
| OUT-06 | Risks folder | `{outputs_root}/risks` |

A folder ID (IN-03, IN-04, IN-05) means every file inside it, not the folder itself — each file inside gets its own cache entry, keyed by its own path.

## Cache

`OUT-05` holds one JSON object keyed by absolute path, covering **both** input and output files:

```json
{
  "<absolute path>": {
    "id": "IN-01",
    "hash": "sha256:<hex>",
    "last_seen": "<ISO 8601 timestamp>"
  }
}
```

Before touching any file in the registry (input or output):

1. Resolve its path. If the path from the cache no longer resolves (file moved or deleted), record it under **moved/deleted** for the end-of-run summary and skip it — do not fail the run.
2. Hash the file's contents (sha256).
3. If the path has no cache entry, or the hash differs from the cached one: read it fully, do the stage's work with it, and write/update its cache entry with the new hash and `last_seen`. Record it under **read** for the summary.
4. If the hash matches the cached entry exactly: treat the file as unchanged. Skip re-reading its content for this stage's work, but still record it under **unchanged** for the summary.

Tell the user immediately, inline, the first time a path appears with no prior cache entry: "New file read into context and cached: `<path>`."

## Output file header

Every file this skill generates — `summary-*.md` and `moscow-*.md` alike — starts with this block, filled in before any other content:

```markdown
> **Generated with AI.** Source files: <id list, e.g. IN-01, IN-05>.
> Last updated: <ISO 8601 timestamp of this run>.
```

## Stage 1 — SoW

Read IN-01 and IN-02 (per the cache rule above). For each project, summarize its SoW: scope, deliverables, timeline, commercial terms — whatever the source actually states, not a fixed template.

Write:
- `OUT-03/summary-config_mgmt-sow.md` (from IN-01)
- `OUT-03/summary-swim-sow.md` (from IN-02)

**Done when:** both files exist with the header block, and both IN-01 and IN-02 are accounted for as read or unchanged.

## Stage 2 — Discovery

Read the config management project's notes (IN-03), the SWIM project's notes (IN-04), and the shared notes (IN-05) — every file in each folder, per the cache rule. Summarize what was actually discussed and requested in each project's discovery sessions, folding in anything from IN-05 relevant to that project.

Write:
- `OUT-01/summary-config_mgmt-discovery.md` (from IN-03 + IN-05)
- `OUT-01/summary-swim-discovery.md` (from IN-04 + IN-05)

**Done when:** both files exist with the header block, and every file under IN-03, IN-04, and IN-05 is accounted for as read or unchanged.

## Stage 3 — Gaps

For each project, read that project's stage-1 SoW summary and stage-2 discovery summary (the files just written, not the raw originals) and compare them. A gap is anything discovery sessions requested that the SoW doesn't cover, or anything the SoW commits to that discovery never surfaced.

Write:
- `OUT-02/summary-config_mgmt-gaps.md` (from `summary-config_mgmt-sow.md` + `summary-config_mgmt-discovery.md`)
- `OUT-02/summary-swim-gaps.md` (from `summary-swim-sow.md` + `summary-swim-discovery.md`)

**Done when:** both files exist with the header block, and each lists every mismatch found, or states explicitly that none were found.

## Stage 4 — Risks

Read IN-06, plus the same project/shared meeting notes as stage 2 (IN-03+IN-05 for config management, IN-04+IN-05 for SWIM), per the cache rule. Identify every risk raised in the Open Design Register or discussed in the notes.

Write:
- `OUT-06/summary-config_mgmt-risks.md` (from IN-06 + IN-03 + IN-05)
- `OUT-06/summary-swim-risks.md` (from IN-06 + IN-04 + IN-05)

Each file's body is a single risk table, one row per risk:

| Column | Content |
|---|---|
| Risk ID | `R001`, `R002`, … sequential, unique within that file |
| Risk Description | What the risk is |
| Category | One of: Schedule, Process, Technical, Environment, Scope, Planning, Decision |
| Impact | What happens if it materializes |
| Risk Score | High / Medium / Low |
| Mitigation / Action | What reduces or resolves it |
| Owner | Who's accountable |
| Status | Open / Mitigated / Closed / etc. |
| Comments | Anything else worth noting |

**Done when:** both files exist with the header block and a fully populated risk table, and IN-06 plus every relevant meeting-notes file is accounted for as read or unchanged.

## Stage 5 — MoSCoW

For each project, read its stage-1 SoW summary, stage-2 discovery summary, and stage-3 gap summary (the already-generated files — not the raw originals, and not the risk files, which this stage doesn't draw on). Sort every item across those three files into Must/Should/Could/Won't Have.

A **Must Have** clears a double bar: written into the SoW's scope or deliverables, *and* confirmed in discovery with no open item, contradiction, or reversal recorded against it anywhere in the discovery or gap record. Should/Could/Won't Have don't need that bar, but every item still cites the source file(s) and date it comes from.

Tag every Should/Could/Won't Have item with no basis anywhere in the SoW — sourced only from discovery, the gap analysis, or an optional-extension appendix — with **(NPOIS)** right after its bold lead-in. Must Have items never carry the tag: the double bar above already requires a SoW basis for all of them.

Match this structure exactly:
- `# MoSCoW Prioritization — GNS <Config Management|SWIM> Workstream`
- `**Source documents cross-checked:**` — bullet list naming the three source files read above.
- `## Methodology` — state the Must-Have double bar, that Should/Could/Won't come from the same record without it, and what an **(NPOIS)** tag means: no basis anywhere in the SoW.
- `## Must Have` — numbered; each item bold, with a paragraph citing its SoW basis and discovery confirmation date.
- `## Should Have` — bulleted; each item bold, with the open caveat or dependency keeping it out of Must Have.
- `## Could Have` — bulleted; optional-extension or later-phase scope.
- `## Won't Have` — bulleted; explicitly out of scope, or blocked on a Change Order.

Write:
- `OUT-04/moscow-config_mgmt.md`
- `OUT-04/moscow-swim.md`

**Done when:** both files exist with the header block and all four MoSCoW sections populated (or a section explicitly states it's empty), every item traces to at least one of the three source files, and every Should/Could/Won't Have item without a SoW basis carries **(NPOIS)**.

## End-of-run summary

Report:
- Stages run, and any prerequisite stage added automatically to satisfy a selection.
- Files read and newly cached this run.
- Files updated since the last run (hash changed).
- Files moved or deleted (present in the cache but not found at their recorded path).
