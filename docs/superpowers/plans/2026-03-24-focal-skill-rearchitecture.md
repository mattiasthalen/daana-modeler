# Focal Skill Rearchitecture Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Rearchitect focal from agent to skill invocation pattern — create focal skill, refactor query, create USS skill, create star skeleton.

**Architecture:** `daana:focal` is a standalone skill invoked as a prerequisite by consumer skills (`query`, `uss`, `star`). It loads connection + bootstrap context into the main session with an early-exit gate if already bootstrapped. USS generates Unified Star Schema DDL files from live Focal metadata.

**Tech Stack:** Claude Code plugin skills (Markdown-driven), SQL (PostgreSQL dialect)

**Design spec:** `docs/superpowers/specs/2026-03-24-focal-skill-rearchitecture-design.md`

---

## Parallel Execution Groups

```
Group A (worktree 1):          Group B (worktree 2):          Group C (worktree 3):
  Task 1: Create focal skill     Task 4: Create USS SKILL.md     Task 6: Create star skeleton
  Task 2: Refactor query skill   Task 5: Create USS references   Task 7: Move dimension/fact refs
  Task 3: Update ad-hoc-query

                    Final (after all groups merge):
                      Task 8: Update CLAUDE.md + plugin.json + version bump
```

Tasks within each group are sequential. Groups A, B, and C run in parallel.

---

### Task 1: Create focal SKILL.md

**Files:**
- Create: `skills/focal/SKILL.md`
- Create: `skills/focal/references/` (directory)

**Step 1: Create the focal SKILL.md**

Create `skills/focal/SKILL.md` with the following content. This is the core skill — connection interview, bootstrap, early-exit gate, and dialect awareness.

```markdown
---
name: daana-focal
description: Shared Focal foundation — connects to a Focal-based Daana data warehouse and bootstraps metadata into the session context. Invoke as a prerequisite from consumer skills.
---

# Daana Focal

You are the shared foundation for all Focal-aware Daana skills. You connect to a Focal-based Daana data warehouse and bootstrap the metadata into the session context so that consumer skills can work with it directly.

Read `${CLAUDE_SKILL_DIR}/references/focal-framework.md` before proceeding.

## Scope

- **Connection and bootstrap only.** You establish context — you never query business data, generate DDL, or modify anything.
- Never generate or execute INSERT, UPDATE, DELETE, DROP, ALTER, TRUNCATE, or any other DDL/DML.
- Never hardcode TYPE_KEYs — they differ between installations.

## Early-Exit Gate

<HARD-GATE>
**Before running any phase, check if the bootstrap result (the metadata entity/attribute listing from `f_focal_read()`) is already present in the conversation context.** If it is — announce "Focal context already active, skipping bootstrap." and exit immediately. Do NOT re-run the bootstrap.
</HARD-GATE>

If the bootstrap result is NOT present, proceed to Phase 1.

## Phase 1: Connection

Read `${CLAUDE_SKILL_DIR}/references/connections.md` for the connection profile schema.

### Step 1 — Look for connections.yaml

**You MUST search for the connections file before asking any connection questions.**

Use the Glob tool to search for the file:

```
pattern: "**/connections.yaml"
```

If found, read the first match and parse the YAML profiles.

<HARD-GATE>
**You MUST ask the user to confirm the profile before using it. Do NOT skip this step, even for a single profile.**
</HARD-GATE>

- **Single profile:** Call the `AskUserQuestion` tool (do NOT print the question as text):
  - Question: "I found one connection profile: **dev** (postgresql). Use this profile?"
  - Options: "Yes" / "No, connect manually"

- **Multiple profiles:** Call the `AskUserQuestion` tool (do NOT print the question as text):
  - Question: "Which connection profile would you like to use?"
  - Options: one per profile, labeled with name and type (e.g., "dev (postgresql)")

**STOP and wait for the user's answer before proceeding. Do NOT extract connection details or proceed to any other step until the user confirms.**

- **If the file does not exist:** proceed to Step 3 (manual fallback).

### Step 2 — Extract connection details

From the chosen profile, extract `host`, `port`, `user`, `database`, and `password`. Environment variable references (`${VAR_NAME}`) are passed through as-is — the shell resolves them at execution time.

### Step 3 — No connections.yaml fallback

If `connections.yaml` is not found:
> "No connections.yaml found. Let's connect manually."

Then ask **one at a time:**

1. **Database user** — "Database user?" (e.g., `dev`)
2. **Database name** — "Database name?" (e.g., `customerdb`)

### Step 4 — Dialect resolution

After determining the connection type (from the profile, or ask the user if connecting manually):

- Try to read `${CLAUDE_SKILL_DIR}/references/dialect-<type>.md` (e.g., `dialect-postgres.md`)
- If found — use it for all connection and bootstrap mechanics.
- If not found — call the `AskUserQuestion` tool (do NOT print the question as text):
  - Question: "No native support for [type] yet. I can try translating from PostgreSQL patterns, but results may need tweaking. Want me to try?"
  - Options: "Yes, try transpiling" / "No, cancel"

  If transpiling — read `${CLAUDE_SKILL_DIR}/references/dialect-postgres.md` as reference.

### Step 5 — Validate connectivity

Run the connectivity check command from the dialect file. If validation fails, report the error and ask the user to verify the details.

## Phase 2: Bootstrap

Read `${CLAUDE_SKILL_DIR}/references/bootstrap.md` before proceeding.

### Step 6 — Bootstrap consent

<HARD-GATE>
**You MUST ask the user for permission before running the bootstrap query. Do NOT skip this step.**
</HARD-GATE>

After a successful connection, you MUST call the `AskUserQuestion` tool (do NOT print the question as text):

- Question: "Connected! Want me to bootstrap the Focal metadata? I'll run one query to discover all entities, attributes, and relationships."
- Options: "Yes, bootstrap metadata" / "No, skip bootstrap"

**STOP and wait for the user's answer. Do NOT proceed until the user responds to the AskUserQuestion.**

- **If the user says yes:** proceed to Step 7.
- **If the user says no:** announce that bootstrap was skipped and exit. Consumer skills will need to handle the lack of metadata.

### Step 7 — Run bootstrap query

Run the bootstrap query from `${CLAUDE_SKILL_DIR}/references/bootstrap.md`. Cache the entire result in memory for the session.

### Bootstrap interpretation

Each row maps the full chain from entity to physical column:

| Column | What it tells you |
|--------|-------------------|
| `focal_name` | The entity (e.g., `CUSTOMER_FOCAL`, `ORDER_FOCAL`) |
| `descriptor_concept_name` | The physical table name (e.g., `CUSTOMER_DESC`, `ORDER_PRODUCT_X`) |
| `atomic_context_name` | The TYPE_KEY meaning (e.g., `CUSTOMER_CUSTOMER_EMAIL_ADDRESS`) |
| `atom_contx_key` | The actual TYPE_KEY value to use in queries |
| `attribute_name` | The logical attribute name within the atomic context |
| `table_pattern_column_name` | The generic column where the value is stored (e.g., `VAL_STR`, `VAL_NUM`, `EFF_TMSTP`) |

**Relationship table detection:** When `table_pattern_column_name` is `FOCAL01_KEY` or `FOCAL02_KEY`, this is a relationship table. Use `attribute_name` as the physical column name instead.

### Bootstrap failure

If the bootstrap query fails:
- Function not found: "The `f_focal_read` function doesn't exist — has `daana-cli install` been run?"
- No results: "No entities found in DAANA_DW. Has the model been deployed?"

### Post-Bootstrap Summary

After bootstrap completes, summarize what was found:

> "Bootstrapped from DAANA_METADATA. Found N entities: ENTITY_1 (X atomic contexts), ENTITY_2 (Y atomic contexts), ... and N relationships. Active dialect: [dialect]. Focal context is now active."

The consumer skill resumes from here with full metadata in context.
```

**Step 2: Commit**

```bash
git add skills/focal/SKILL.md
git commit -m "feat: create focal skill with connection and bootstrap workflow"
```

---

### Task 2: Move reference files from query to focal

**Files:**
- Move: `skills/query/references/focal-framework.md` → `skills/focal/references/focal-framework.md`
- Move: `skills/query/references/bootstrap.md` → `skills/focal/references/bootstrap.md`
- Move: `skills/query/references/connections.md` → `skills/focal/references/connections.md`
- Move: `skills/query/references/dialect-postgres.md` → `skills/focal/references/dialect-postgres.md`

**Step 1: Move the four reference files**

```bash
mkdir -p skills/focal/references
git mv skills/query/references/focal-framework.md skills/focal/references/focal-framework.md
git mv skills/query/references/bootstrap.md skills/focal/references/bootstrap.md
git mv skills/query/references/connections.md skills/focal/references/connections.md
git mv skills/query/references/dialect-postgres.md skills/focal/references/dialect-postgres.md
```

**Step 2: Commit**

```bash
git add -A
git commit -m "refactor: move focal reference files from query to focal skill"
```

---

### Task 3: Refactor query SKILL.md to use focal prerequisite

**Files:**
- Modify: `skills/query/SKILL.md`

**Step 1: Rewrite query SKILL.md**

Replace the entire file. The key changes:
- Add `**REQUIRED SUB-SKILL:** Use daana:focal` at the top
- Remove Phase 1 (Connection) and Phase 2 (Bootstrap) entirely
- Renumber Phase 3 → Phase 1, Phase 3B → Phase 1B, Phase 4 → Phase 2
- Update all `${CLAUDE_SKILL_DIR}/references/` paths — remove references to moved files
- In the parallel subagent dispatch section, update the prompt construction to note that connection details and bootstrap data come from the focal skill's context (already in the session), not from focal's reference files

The new SKILL.md should contain:

```markdown
---
name: daana-query
description: Data agent that answers natural language questions about Focal-based Daana data warehouses via live SQL queries.
---

# Daana Query

**REQUIRED SUB-SKILL:** Use daana:focal

You are a data analyst fluent in the Focal framework. You think in entities, attributes, and relationships, translate natural language questions into SQL, and explain results in business terms.

The focal skill establishes the database connection and bootstraps metadata. Once focal completes, the session flows through two phases: Query Loop and Handover.
```

Then copy the rest of the existing SKILL.md starting from `## Scope`, but with these changes:

1. Remove Phase 1 (Connection) — lines 34-95 of the current file
2. Remove Phase 2 (Bootstrap) — lines 96-146 of the current file
3. Renumber remaining phases:
   - "## Multi-Question Detection" stays as-is (it's not a phase)
   - "## Phase 3B: Multi-Query Flow" → "## Phase 1B: Multi-Query Flow"
   - "## Phase 3: Query Loop" → "## Phase 1: Query Loop"
   - "## Phase 4: Handover" → "## Phase 2: Handover"
4. In Phase 1 (Query Loop), update the reference path:
   - `${CLAUDE_SKILL_DIR}/references/ad-hoc-query-agent.md` — stays (this file didn't move)
   - Remove the `${CLAUDE_SKILL_DIR}/references/focal-framework.md` reference in the lineage tracing section — the focal framework knowledge is already in context from the focal skill
5. In Phase 1B (Multi-Query Flow), update the subagent prompt construction:
   - Items 3 (Bootstrap data) and 4 (Connection details) come from the current session context (loaded by focal skill), not from reference files
   - Item 5 (Dialect instructions) — note that the dialect file was loaded by focal and is in context
6. Update all phase references within the text (e.g., "Phase 3" → "Phase 1", "Phase 4" → "Phase 2", "Phase 3B" → "Phase 1B")

**Step 2: Verify the remaining references directory**

After editing, `skills/query/references/` should contain only:
- `ad-hoc-query-agent.md`

Run: `ls skills/query/references/` to verify.

**Step 3: Commit**

```bash
git add skills/query/SKILL.md
git commit -m "refactor: remove connection/bootstrap from query, use focal prerequisite"
```

---

### Task 4: Create USS SKILL.md

**Files:**
- Create: `skills/uss/SKILL.md`

**Step 1: Create the USS SKILL.md**

Create `skills/uss/SKILL.md` based on PR #29's content, adapted for the skill-based focal pattern. Key changes from PR #29:

1. Replace "Dispatch the focal agent" (Phase 1) with `**REQUIRED SUB-SKILL:** Use daana:focal` at the top
2. Remove the entire Phase 1 (Bootstrap) section — focal handles this
3. Renumber phases: Interview → Phase 1, Generate → Phase 2, Handover → Phase 3
4. In Phase 3 (Handover), replace "dispatch the focal agent to execute" with direct SQL execution using the connection details already in context from focal

```markdown
---
name: daana-uss
description: Generate a Unified Star Schema (Francesco Puppini) as DDL from a Focal-based Daana data warehouse — a single bridge table connecting all peripherals through resolved M:1 chains.
---

# Daana Unified Star Schema Generator

**REQUIRED SUB-SKILL:** Use daana:focal

You generate a Unified Star Schema (USS) as a folder of SQL DDL files from a Focal-based Daana data warehouse. The USS eliminates fan traps and chasm traps by creating a single bridge table that all peripherals (complete entity views) join to through resolved M:1 relationship chains.

The focal skill establishes the database connection and bootstraps metadata. Once focal completes, the session flows through three phases: Interview, Generate, and Handover.

Read `${CLAUDE_SKILL_DIR}/references/uss-patterns.md` and `${CLAUDE_SKILL_DIR}/references/uss-examples.md` before proceeding.

## Key Concepts

- **Bridge** (`_bridge.sql`) — Central table. UNION ALL of fact rows from all participating entities. Contains resolved FK keys to peripherals, measures, and (if event-grain) unpivoted event timestamps. A `peripheral` column identifies the source entity.
- **Peripheral** (`{entity}.sql`) — Complete entity view with ALL attributes regardless of type. Joins to bridge via surrogate key.
- **Synthetic Peripherals** — Auto-generated `_dates.sql` and `_times.sql` that join to the bridge via the event timestamp.

## Scope

- **DDL generation only.** You produce SQL files — you do not modify existing data.
- Never hardcode TYPE_KEYs — always resolve from bootstrap.
- Only follow M:1 relationship chains (no fan-out). Exclude or flag M:M relationships.
- Never assume physical columns — always resolve via bootstrap.
- Use the active dialect from the focal context for all SQL generation. Only PostgreSQL patterns are currently implemented.

## Phase 1: Interview

Ask the user one question at a time using the `AskUserQuestion` tool. Do NOT print questions as text.

### Question 1 — Entity Selection

Auto-classify entities from the bootstrap:
- **Bridge candidates:** Entities with at least one timestamp attribute (STA_TMSTP or END_TMSTP) and/or numeric attributes (VAL_NUM)
- **Peripheral candidates:** Entities referenced via M:1 relationships (on the FOCAL02_KEY side)

Present the classification and ask the user to confirm:
- Question: "Based on the metadata, here's my proposed USS layout:\n\n**Bridge sources:** ENTITY_A, ENTITY_B\n**Peripherals:** ENTITY_C, ENTITY_D\n\nDoes this look right?"
- Options: "Yes" / "No, let me adjust"

If the user adjusts, re-classify based on their input.

### Question 2 — Temporal Mode

- Question: "How should timestamps be handled in the bridge?"
- Options:
  - "Event-grain unpivot (Recommended)" — Unpivots all timestamps into `event` + `event_occurred_on` rows. Enables canonical `_dates` and `_times` peripherals.
  - "Columnar dates" — Each timestamp stays as a separate column (e.g., `order_date`, `ship_date`). No synthetic date/time peripherals.

### Question 3 — Historical Mode

- Question: "Should the USS capture the latest snapshot or preserve temporal history?"
- Options:
  - "Snapshot (latest values)" — RANK pattern for dedup. One row per fact instance.
  - "Historical (valid_from / valid_to)" — Preserve effective timestamps. Adds `valid_from` and `valid_to` columns to bridge and peripherals.

### Question 4 — Materialization

- Question: "How should the USS be materialized?"
- Options:
  - "All views" — Every file is a CREATE VIEW statement
  - "All tables" — Every file is a CREATE TABLE AS statement
  - "Bridge as table, peripherals as views" — Bridge materialized, peripherals are views
  - "Custom" — I'll specify per file

If "Custom", ask for each file type (bridge, peripherals, synthetics) separately.

### Question 5 — Output Folder

- Question: "Where should I write the SQL files?"
- Options: Let the user type a path (provide a suggested default like `uss/`)

## Phase 2: Generate

Build and write SQL files to the output folder. Follow the patterns in `${CLAUDE_SKILL_DIR}/references/uss-patterns.md` exactly.

### Generation Order

1. **Peripherals first** — One `.sql` per peripheral entity (lowercased: `customer.sql`, `product.sql`)
2. **Bridge** — `_bridge.sql` (depends on knowing all peripheral keys)
3. **Synthetic date peripheral** — `_dates.sql` (depends on bridge for min/max year)
4. **Synthetic time peripheral** — `_times.sql` (independent)

### Column Naming Conventions

| Pattern | Example | Description |
|---------|---------|-------------|
| `peripheral` | `peripheral` | Source entity name (no prefix) |
| `event` | `event` | Event name (no prefix) |
| `event_occurred_on` | `event_occurred_on` | Full timestamp (no prefix) |
| `_key__{entity}` | `_key__customer` | FK to peripheral |
| `_key__dates` | `_key__dates` | FK to synthetic date peripheral |
| `_key__times` | `_key__times` | FK to synthetic time peripheral |
| `_measure__{entity}__{attr}` | `_measure__order_line__unit_price` | Measure value |
| `valid_from` | `valid_from` | Historical mode only |
| `valid_to` | `valid_to` | Historical mode only |

### File Naming

- Peripheral entities: lowercased entity name without `_FOCAL` suffix (e.g., `CUSTOMER_FOCAL` → `customer.sql`)
- Synthetic files: prefixed with underscore (`_bridge.sql`, `_dates.sql`, `_times.sql`)

### Wrap each file with DDL

Based on the user's materialization choice:
- **View:** `CREATE OR REPLACE VIEW {schema}.{name} AS ...`
- **Table:** `CREATE TABLE {schema}.{name} AS ...`

Ask the user for the target schema name if not obvious from the connection profile.

## Phase 3: Handover

After generating all files:

1. List the generated files with a brief description of each.
2. Call the `AskUserQuestion` tool (do NOT print the question as text):
   - Question: "Want me to execute these DDL statements against the database?"
   - Options: "Yes, execute all" / "No, I'll run them manually"
   - If yes: execute each file in order (peripherals → bridge → synthetics) using the connection details from the focal context.
   - If no: "Files are ready in `{output_folder}/`. You can run them manually."
3. Suggest: "You can now use `/daana-query` to query the unified star schema."
```

**Step 2: Commit**

```bash
git add skills/uss/SKILL.md
git commit -m "feat: create USS skill with interview-driven DDL generation"
```

---

### Task 5: Create USS reference files

**Files:**
- Create: `skills/uss/references/uss-patterns.md`
- Create: `skills/uss/references/uss-examples.md`

**Step 1: Create uss-patterns.md**

Copy from PR #29 branch. The content is available at:
```bash
git show origin/feat/focal-agent-and-uss-skill:skills/uss/references/uss-patterns.md
```

If the path doesn't resolve, try `plugin/skills/uss/references/uss-patterns.md` prefix instead.

Write to `skills/uss/references/uss-patterns.md`. No modifications needed — the patterns are dialect-agnostic in structure (concrete SQL uses Postgres syntax, which is correct for this deliverable).

**Step 2: Create uss-examples.md**

Copy from PR #29 branch:
```bash
git show origin/feat/focal-agent-and-uss-skill:skills/uss/references/uss-examples.md
```

If the path doesn't resolve, try `plugin/skills/uss/references/uss-examples.md` prefix instead.

Write to `skills/uss/references/uss-examples.md`. No modifications needed.

**Step 3: Commit**

```bash
git add skills/uss/references/
git commit -m "feat: add USS SQL patterns and worked example references"
```

---

### Task 6: Create star skill skeleton

**Files:**
- Create: `skills/star/SKILL.md`

**Step 1: Create star SKILL.md**

```markdown
---
name: daana-star
description: Generate traditional star schema SQL (fact tables + dimension tables) from a Focal-based Daana data warehouse.
---

# Daana Star Schema Generator

**REQUIRED SUB-SKILL:** Use daana:focal

> **Status:** Skeleton — full implementation deferred to a future design spec.

This skill generates traditional star schema DDL (fact tables and dimension tables) from a Focal-based Daana data warehouse.

The focal skill establishes the database connection and bootstraps metadata. Once focal completes, the session flows through the phases below.

## Phases

1. **Interview** — Classify entities as facts or dimensions, select SCD types, choose materialization.
2. **Generate** — Produce SQL files for fact tables and dimension tables.
3. **Handover** — Offer to execute DDL or suggest `/daana-query`.

## References

- `${CLAUDE_SKILL_DIR}/references/dimension-patterns.md` — SCD types 0-6, mixed types, design considerations.
- `${CLAUDE_SKILL_DIR}/references/fact-patterns.md` — Transaction, periodic snapshot, accumulating snapshot, factless facts.
```

**Step 2: Commit**

```bash
git add skills/star/SKILL.md
git commit -m "feat: create star skill skeleton"
```

---

### Task 7: Move dimension/fact references from query to star

**Files:**
- Move: `skills/query/references/dimension-patterns.md` → `skills/star/references/dimension-patterns.md`
- Move: `skills/query/references/fact-patterns.md` → `skills/star/references/fact-patterns.md`

**Step 1: Move the reference files**

```bash
mkdir -p skills/star/references
git mv skills/query/references/dimension-patterns.md skills/star/references/dimension-patterns.md
git mv skills/query/references/fact-patterns.md skills/star/references/fact-patterns.md
```

**Step 2: Verify query references directory**

After this move, `skills/query/references/` should contain only:
- `ad-hoc-query-agent.md`

Run: `ls skills/query/references/` to verify.

**Step 3: Commit**

```bash
git add -A
git commit -m "refactor: move dimension/fact pattern references from query to star skill"
```

---

### Task 8: Update CLAUDE.md, plugin.json, and bump version

**Files:**
- Modify: `CLAUDE.md`
- Modify: `.claude-plugin/plugin.json`

**Step 1: Update CLAUDE.md**

Replace the repository structure section to reflect all six skills:

```markdown
# daana-modeler

daana-modeler is a Claude Code plugin for the Daana data platform. It provides six skills for building, querying, and transforming DMDL data models.

## Repository Structure

- **`.claude-plugin/`** — Plugin manifests
  - `plugin.json` — Plugin metadata (name: `daana`)
  - `marketplace.json` — Marketplace catalog for plugin discovery
- **`skills/focal/SKILL.md`** — Shared Focal foundation skill (`/daana-focal`)
  - `skills/focal/references/` — Focal framework, bootstrap, connections, dialect references
- **`skills/model/SKILL.md`** — Model interview skill (`/daana-model`)
  - `skills/model/references/` — Model schema, examples, source format references
- **`skills/map/SKILL.md`** — Mapping interview skill (`/daana-map`)
  - `skills/map/references/` — Mapping schema, examples, source format references
- **`skills/query/SKILL.md`** — Data query skill (`/daana-query`)
  - `skills/query/references/` — Ad-hoc query patterns
- **`skills/uss/SKILL.md`** — Unified Star Schema generator (`/daana-uss`)
  - `skills/uss/references/` — USS SQL patterns, worked examples
- **`skills/star/SKILL.md`** — Star schema generator skeleton (`/daana-star`)
  - `skills/star/references/` — Dimension patterns, fact patterns
- **`docs/superpowers/specs/`** — Design specifications
- **`docs/superpowers/plans/`** — Implementation plans

## External References

- `external.lock` pins upstream repos. When exploring project context (e.g., start of brainstorming), fresh-clone each repo and compare HEAD against the pinned commit to detect new upstream changes.
```

**Step 2: Bump plugin.json version**

Change `"version": "1.9.0"` to `"version": "1.10.0"`.

**Step 3: Commit**

```bash
git add CLAUDE.md .claude-plugin/plugin.json
git commit -m "feat: bump version to 1.10.0, update CLAUDE.md for new skills"
```
