# daana-modeler

daana-modeler is a Claude Code plugin for the Daana data platform. It provides a reusable Focal agent and five skills for building, querying, and generating star schemas from DMDL data models.

## Repository Structure

- **`.claude-plugin/`** — Marketplace manifest
  - `marketplace.json` — Marketplace catalog for plugin discovery
- **`plugin/`** — Distributable plugin contents
  - `.claude-plugin/plugin.json` — Plugin metadata (name: `daana`)
  - `agents/focal/AGENT.md` — Reusable Focal agent for connection, bootstrap, and SQL execution
  - `agents/focal/references/` — Focal framework, bootstrap, connections, dialect
  - `skills/model/SKILL.md` — Model interview skill (`/daana-model`)
  - `skills/model/references/` — Model schema, examples, source format references
  - `skills/map/SKILL.md` — Mapping interview skill (`/daana-map`)
  - `skills/map/references/` — Mapping schema, examples, source format references
  - `skills/query/SKILL.md` — Data query skill (`/daana-query`)
  - `skills/query/references/` — Ad-hoc query patterns
  - `skills/uss/SKILL.md` — Unified Star Schema generator (`/daana-uss`)
  - `skills/uss/references/` — USS SQL patterns, worked examples
  - `skills/star/SKILL.md` — Traditional star schema generator (`/daana-star`, skeleton)
  - `skills/star/references/` — Dimension patterns, fact patterns
- **`docs/superpowers/specs/`** — Design specifications
- **`docs/superpowers/plans/`** — Implementation plans

## External References

- `external.lock` pins upstream repos. When exploring project context (e.g., start of brainstorming), fresh-clone each repo and compare HEAD against the pinned commit to detect new upstream changes.
