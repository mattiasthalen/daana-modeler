# Plugin Restructure Design

Restructure daana-modeler from a loose collection of skills into a proper Claude Code plugin. Resolves GitHub issues #4, #5, and #6.

## Context

daana-modeler currently has skills under `skills/` at the repo root but lacks a `.claude-plugin/plugin.json` manifest, so Claude Code cannot discover it as a plugin. The skills are named `daana-model`, `daana-mapping`, and `daana-query`, with an orchestrator at `skills/daana/` that routes between them. Two skills have `disable-model-invocation: true`, preventing Claude from invoking them via the Skill tool — which breaks inter-skill handovers.

## Goals

1. Add a plugin manifest so the repo is a proper Claude Code plugin.
2. Rename skills to produce clean slash commands: `/daana:model`, `/daana:map`, `/daana:query`.
3. Remove the orchestrator and replace it with handover chains between skills.
4. Remove `disable-model-invocation: true` so skills can invoke each other.
5. Move shared references to the plugin root level.

## New Directory Structure

```
daana-modeler/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── model/
│   │   └── SKILL.md
│   ├── map/
│   │   └── SKILL.md
│   └── query/
│       └── SKILL.md
├── references/
│   ├── model-schema.md
│   ├── model-examples.md
│   ├── mapping-schema.md
│   ├── mapping-examples.md
│   └── source-schema-formats.md
├── CLAUDE.md
├── README.md
├── LICENSE
└── .devcontainer/
```

### What's removed

- `skills/daana/` — the orchestrator skill. Its routing logic is replaced by handover sections in each skill. Its source schema parsing guidance moves into `model` and `map` skills directly.
- `skills/daana/references/` — moved to `references/` at the plugin root.

### What's renamed

| Old path | New path |
|---|---|
| `skills/daana-model/` | `skills/model/` |
| `skills/daana-mapping/` | `skills/map/` |
| `skills/daana-query/` | `skills/query/` |

## Plugin Manifest

`.claude-plugin/plugin.json`:

```json
{
  "name": "daana",
  "description": "Interview-driven DMDL model and mapping builder for the Daana data platform",
  "version": "1.0.0",
  "author": {
    "name": "Mattias Thalén"
  },
  "repository": "https://github.com/mattiasthalen/daana-modeler",
  "license": "MIT",
  "keywords": ["daana", "dmdl", "data-modeling", "data-warehouse"]
}
```

No custom component paths are needed. The `skills/` directory is the default discovery location and will be auto-discovered. The `references/` directory is not a plugin component — skills read its files directly via file paths.

## Skill Changes

### Frontmatter changes (all skills)

- Remove `disable-model-invocation: true` from `model` and `map` skills. This fixes issues #5 and #6, allowing Claude to invoke these skills via the Skill tool for handovers.

### Reference path updates (model and map skills)

- Update all references from `skills/daana/references/` to `references/` (relative to plugin root).

### Source schema parsing (model and map skills)

- Add guidance to both `model` and `map` skills: "If the user provides a source schema file, read `references/source-schema-formats.md` and parse it." This replaces the orchestrator's pre-parsing role. Each skill uses the parsed data for its own purpose — `model` for entity/attribute hints, `map` for table/column matching.

### Handover sections

Add a wrap-up/handover section at the end of each skill:

**`model/SKILL.md`**: After writing `model.yaml`, offer to hand over to `/daana:map` to create source mappings.

**`map/SKILL.md`**: After writing a mapping file, offer to continue with another unmapped entity or hand over to `/daana:query` to explore the data.

**`query/SKILL.md`**: If unmapped entities are detected, suggest `/daana:map` to set up mappings.

### Skill instructions

The core interview flows, validation rules, and behavioral instructions within each skill remain unchanged. Only frontmatter, reference paths, and the new handover section are modified.

## CLAUDE.md Updates

- Remove mention of the `/daana` orchestrator entrypoint.
- Update skill directory paths to `skills/model/`, `skills/map/`, `skills/query/`.
- Update shared references path to `references/`.
- Update slash command names to `/daana:model`, `/daana:map`, `/daana:query`.

## README.md Updates

- Change subtitle from "A Claude Code skill" to "A Claude Code plugin".
- Update usage section: replace `/daana` with `/daana:model`, `/daana:map`, `/daana:query`.
- Add brief description of each skill's purpose.
- Installation command remains `claude plugin add` (already correct).

## Issues Resolved

| Issue | Title | Resolution |
|---|---|---|
| #4 | Restructure project as a Claude Code plugin | Plugin manifest added, skills renamed, orchestrator removed |
| #5 | Skill daana-mapping cannot be used with Skill tool due to disable-model-invocation | `disable-model-invocation: true` removed from `map/SKILL.md` |
| #6 | Skill daana-model cannot be used with Skill tool due to disable-model-invocation | `disable-model-invocation: true` removed from `model/SKILL.md` |
