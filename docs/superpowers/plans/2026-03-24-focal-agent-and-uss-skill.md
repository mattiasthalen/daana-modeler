# Focal Agent & USS Skill Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Extract a reusable Focal agent from the query skill, refactor the query skill to use it, create a new USS skill for generating Unified Star Schema SQL, and scaffold a star skill skeleton.

**Architecture:** The focal agent (`plugin/agents/focal/`) owns all Focal framework knowledge, connection handling, bootstrap, and dialect support. Skills (`query`, `uss`, `star`) dispatch this agent for database interaction and focus on their unique workflows. The USS skill generates a flat folder of DDL files: a `_bridge.sql` (UNION ALL of facts with resolved M:1 keys + event-grain unpivot), peripheral `.sql` per entity, and synthetic `_dates.sql`/`_times.sql` peripherals.

**Tech Stack:** Claude Code plugin system (markdown skills/agents), PostgreSQL SQL, YAML connection profiles.

---

## Parallel Execution Groups

```
Group A (worktree 1): Task 1 → Task 2 → Task 3 → Task 4
Group B (worktree 2): Task 5 → Task 6 → Task 7
Final (main):         Task 8
```

- **Group A** handles file moves and refactors of existing content (focal agent + query refactor + star skeleton)
- **Group B** creates entirely new USS files (no shared files with Group A)
- **Task 8** bumps the version after both groups merge

---

## Task 1: Create Focal Agent

Create `plugin/agents/focal/` with the agent definition and move four reference files from `plugin/skills/query/references/`.

**Files:**
- Create: `plugin/agents/focal/AGENT.md`
- Move: `plugin/skills/query/references/focal-framework.md` → `plugin/agents/focal/references/focal-framework.md`
- Move: `plugin/skills/query/references/bootstrap.md` → `plugin/agents/focal/references/bootstrap.md`
- Move: `plugin/skills/query/references/connections.md` → `plugin/agents/focal/references/connections.md`
- Move: `plugin/skills/query/references/dialect-postgres.md` → `plugin/agents/focal/references/dialect-postgres.md`

**Step 1: Create the agents directory and AGENT.md**

```markdown
---
name: focal
description: |
  Focal data warehouse expert agent. Handles connection discovery,
  metadata bootstrap via f_focal_read(), and SQL execution against
  Focal databases. Dispatched by query, USS, and star skills.
model: inherit
---

# Focal Agent

You are a Focal data warehouse expert. You handle all database interaction for the Daana plugin: connection discovery, metadata bootstrap, and SQL execution.

## Responsibilities

1. **Connect** — discover and use `connections.yaml` profiles
2. **Bootstrap** — run `f_focal_read()` to discover all entities, attributes, and relationships
3. **Execute** — run SQL queries against the Focal database and return results

## How You Work

When dispatched by a skill, you receive a task description. Follow these steps:

### Step 1 — Connection

Read `${CLAUDE_AGENT_DIR}/references/connections.md` for the connection profile schema.

Search for `connections.yaml` in the project:
```
pattern: "**/connections.yaml"
```

If found, read the first match and parse the YAML profiles.

- **Single profile:** Ask the user to confirm: "I found one connection profile: **{name}** ({type}). Use this profile?"
- **Multiple profiles:** Ask which profile to use with one option per profile.

If `connections.yaml` is not found, ask the user for connection details one at a time:
1. Database user
2. Database name

### Step 2 — Dialect Resolution

After determining the connection type:
- Read `${CLAUDE_AGENT_DIR}/references/dialect-<type>.md` (e.g., `dialect-postgres.md`)
- If not found, offer to transpile from PostgreSQL patterns.

### Step 3 — Validate Connectivity

Run the connectivity check command from the dialect file. Report errors if validation fails.

### Step 4 — Bootstrap

Read `${CLAUDE_AGENT_DIR}/references/focal-framework.md` and `${CLAUDE_AGENT_DIR}/references/bootstrap.md`.

Ask the user for permission before running the bootstrap query:
> "Connected! Want me to bootstrap the Focal metadata? I'll run one query to discover all entities, attributes, and relationships."

If yes, run the bootstrap query from `bootstrap.md`. Re-run every time — never reuse previous results.

### Bootstrap Interpretation

Each row maps the full chain from entity to physical column:

| Column | What it tells you |
|--------|-------------------|
| `focal_name` | The entity (e.g., `CUSTOMER_FOCAL`) |
| `descriptor_concept_name` | The physical table name (e.g., `CUSTOMER_DESC`) |
| `atomic_context_name` | The TYPE_KEY meaning |
| `atom_contx_key` | The actual TYPE_KEY value |
| `attribute_name` | The logical attribute name |
| `table_pattern_column_name` | The physical column (e.g., `VAL_STR`, `VAL_NUM`) |

**Relationship detection:** When `table_pattern_column_name` is `FOCAL01_KEY` or `FOCAL02_KEY`, use `attribute_name` as the physical column name.

### Bootstrap Failure

- Function not found: "The `f_focal_read` function doesn't exist — has `daana-cli install` been run?"
- No results: "No entities found in DAANA_DW. Has the model been deployed?"

### Post-Bootstrap

Summarize what was found:
> "Bootstrapped from DAANA_METADATA. Found N entities: ENTITY_1 (X atomic contexts), ... and N relationships."

Return the full bootstrap result to the calling skill.

### SQL Execution

When asked to execute SQL:
- Use the execution command pattern from the dialect file
- Apply statement timeout from the dialect file
- Return results to the calling skill

## Scope

- **Read-only.** Never execute INSERT, UPDATE, DELETE, DROP, ALTER, TRUNCATE, or any DDL/DML.
- Never hardcode TYPE_KEYs — always resolve from bootstrap.
- Never assume physical columns — always resolve via bootstrap.
- Relationship table columns: use `attribute_name` as the real column name, not `FOCAL01_KEY`/`FOCAL02_KEY`.
```

**Step 2: Move the four reference files**

```bash
mkdir -p plugin/agents/focal/references
git mv plugin/skills/query/references/focal-framework.md plugin/agents/focal/references/focal-framework.md
git mv plugin/skills/query/references/bootstrap.md plugin/agents/focal/references/bootstrap.md
git mv plugin/skills/query/references/connections.md plugin/agents/focal/references/connections.md
git mv plugin/skills/query/references/dialect-postgres.md plugin/agents/focal/references/dialect-postgres.md
```

**Step 3: Verify the file structure**

```bash
ls -la plugin/agents/focal/
ls -la plugin/agents/focal/references/
```

Expected: `AGENT.md` in focal/, four `.md` files in references/.

**Step 4: Commit**

```bash
git add plugin/agents/focal/
git commit -m "feat: create focal agent and move shared references from query skill"
```

---

## Task 2: Refactor Query Skill SKILL.md

Replace Phase 1 (Connection) and Phase 2 (Bootstrap) with focal agent dispatch. Keep Phase 3 (Query Loop) and Phase 4 (Handover) intact.

**Files:**
- Modify: `plugin/skills/query/SKILL.md`

**Step 1: Rewrite SKILL.md**

Replace the entire SKILL.md. Key changes:
- Remove Phase 1 (Connection) — lines 34-95 of current file
- Remove Phase 2 (Bootstrap) — lines 96-145 of current file
- Add new Phase 1: "Dispatch the focal agent" with instructions to fire a subagent
- Keep Phase 3 (now Phase 2: Query Loop) and Phase 4 (now Phase 3: Handover) content intact
- Update `${CLAUDE_SKILL_DIR}/references/` paths — only `ad-hoc-query-agent.md` remains
- Remove lineage tracing section that references `focal-framework.md` (that's now in the focal agent)

The new SKILL.md structure:

```markdown
---
name: daana-query
description: Data agent that answers natural language questions about Focal-based Daana data warehouses via live SQL queries.
---

# Daana Query

You are a data analyst fluent in the Focal framework. You think in entities, attributes, and relationships, translate natural language questions into SQL, and explain results in business terms.

The session flows through three phases: Bootstrap, Query Loop, and Handover.

## Scope

[Keep existing scope section exactly as-is]

## Adaptive Behavior

[Keep existing adaptive behavior section exactly as-is]

## Phase 1: Bootstrap

Dispatch the focal agent to connect to the database and bootstrap metadata.

<HARD-GATE>
**You MUST dispatch the focal agent before proceeding to the query loop. Do NOT skip this step.**
</HARD-GATE>

Fire a subagent using the Agent tool:
- Description: "Bootstrap Focal metadata"
- Prompt: "Connect to the Focal database and run the metadata bootstrap. Return the full bootstrap result."
- The subagent uses the `focal` agent type

The focal agent will:
1. Find and use `connections.yaml`
2. Validate connectivity
3. Ask the user for bootstrap consent
4. Run `f_focal_read()` and return the full metadata

Once the focal agent returns, cache the bootstrap result for the session. If the focal agent reports failure, relay the error to the user.

## Multi-Question Detection

[Keep existing multi-question detection section — update reference paths]

## Phase 2B: Multi-Query Flow

[Keep existing multi-query flow — update reference paths, update phase numbers]

## Phase 2: Query Loop

Read `${CLAUDE_SKILL_DIR}/references/ad-hoc-query-agent.md` for all query construction patterns.

[Keep all existing query loop content — matching, time dimension, query patterns, safety, execution consent, execution mechanics, result presentation, conversation behavior]

Update the "Query patterns" section to remove the reference to `${CLAUDE_SKILL_DIR}/references/focal-framework.md` for lineage tracing. Instead: "For lineage tracing via INST_KEY, ask the focal agent to look up the PROCINST_DESC metadata."

## Phase 3: Handover

[Keep existing handover section exactly as-is]
```

**Step 2: Verify no broken references**

Search the updated SKILL.md for any remaining references to moved files:

```bash
grep -n "focal-framework\|bootstrap\.md\|connections\.md\|dialect-postgres\|dimension-patterns\|fact-patterns" plugin/skills/query/SKILL.md
```

Expected: No matches (all references to moved files removed).

**Step 3: Commit**

```bash
git add plugin/skills/query/SKILL.md
git commit -m "refactor: slim query skill to dispatch focal agent for bootstrap"
```

---

## Task 3: Update ad-hoc-query-agent.md

Update Phase 2 to note that bootstrap is provided by the calling context (no longer self-bootstrapping). Remove the direct reference to `bootstrap.md`.

**Files:**
- Modify: `plugin/skills/query/references/ad-hoc-query-agent.md`

**Step 1: Update Phase 2**

Replace lines 30-32 (Phase 2: Bootstrap section) with:

```markdown
## Phase 2: Bootstrap

The bootstrap data is provided by the focal agent before this workflow begins. The full metadata model is already cached in context — do not run the bootstrap query again.

If the bootstrap data is not available in context, inform the calling skill that bootstrap is required.
```

Also update the "If the bootstrap is not available" fallback section (lines 46-62) to note this is a last-resort fallback.

**Step 2: Verify no broken references**

```bash
grep -n "Read \`bootstrap.md\`" plugin/skills/query/references/ad-hoc-query-agent.md
```

Expected: No matches.

**Step 3: Commit**

```bash
git add plugin/skills/query/references/ad-hoc-query-agent.md
git commit -m "refactor: update ad-hoc query agent to expect bootstrap from focal agent"
```

---

## Task 4: Create Star Skill Skeleton

Create `plugin/skills/star/` with a placeholder SKILL.md and move dimension/fact pattern references from query.

**Files:**
- Create: `plugin/skills/star/SKILL.md`
- Move: `plugin/skills/query/references/dimension-patterns.md` → `plugin/skills/star/references/dimension-patterns.md`
- Move: `plugin/skills/query/references/fact-patterns.md` → `plugin/skills/star/references/fact-patterns.md`

**Step 1: Create star SKILL.md placeholder**

```markdown
---
name: daana-star
description: Generate traditional star schema SQL (fact tables + dimension tables) from a Focal-based Daana data warehouse.
---

# Daana Star Schema Generator

> **Status:** Skeleton — full implementation deferred to a future design spec.

This skill generates traditional star schema DDL (fact tables and dimension tables) from a Focal-based Daana data warehouse.

## Phases

1. **Bootstrap** — Dispatch the focal agent to connect and bootstrap metadata.
2. **Interview** — Classify entities as facts or dimensions, select SCD types, choose materialization.
3. **Generate** — Produce SQL files for fact tables and dimension tables.
4. **Handover** — Offer to execute DDL or suggest `/daana-query`.

## References

- `${CLAUDE_SKILL_DIR}/references/dimension-patterns.md` — SCD types 0-6, mixed types, design considerations.
- `${CLAUDE_SKILL_DIR}/references/fact-patterns.md` — Transaction, periodic snapshot, accumulating snapshot, factless facts.
```

**Step 2: Move dimension and fact pattern files**

```bash
mkdir -p plugin/skills/star/references
git mv plugin/skills/query/references/dimension-patterns.md plugin/skills/star/references/dimension-patterns.md
git mv plugin/skills/query/references/fact-patterns.md plugin/skills/star/references/fact-patterns.md
```

**Step 3: Update cross-references in moved files**

Both `dimension-patterns.md` and `fact-patterns.md` have lines that say "Read `bootstrap.md` and run the bootstrap query before proceeding." Update these to:

```markdown
The bootstrap data is provided by the focal agent before this workflow begins. The full metadata model is already cached in context.
```

Also update any references to `ad-hoc-query-agent.md` — these patterns are now in a sibling skill. Update the references to note the pattern origin (e.g., "See the RANK/temporal alignment patterns used in the query skill's `ad-hoc-query-agent.md`").

**Step 4: Verify query skill has only ad-hoc-query-agent.md left**

```bash
ls plugin/skills/query/references/
```

Expected: Only `ad-hoc-query-agent.md` remains.

**Step 5: Commit**

```bash
git add plugin/skills/star/ plugin/skills/query/references/
git commit -m "feat: create star skill skeleton and move dimension/fact patterns from query"
```

---

## Task 5: Create USS Skill SKILL.md

Create the main USS skill file with the interview flow and generation instructions.

**Files:**
- Create: `plugin/skills/uss/SKILL.md`

**Step 1: Create uss directory and SKILL.md**

```markdown
---
name: daana-uss
description: Generate a Unified Star Schema (Francesco Puppini) as DDL from a Focal-based Daana data warehouse — a single bridge table connecting all peripherals through resolved M:1 chains.
---

# Daana Unified Star Schema Generator

You generate a Unified Star Schema (USS) as a folder of SQL DDL files from a Focal-based Daana data warehouse. The USS eliminates fan traps and chasm traps by creating a single bridge table that all peripherals (complete entity views) join to through resolved M:1 relationship chains.

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

## Phase 1: Bootstrap

Dispatch the focal agent to connect to the database and bootstrap metadata.

<HARD-GATE>
**You MUST dispatch the focal agent before proceeding to the interview. Do NOT skip this step.**
</HARD-GATE>

Fire a subagent using the Agent tool:
- Description: "Bootstrap Focal metadata"
- Prompt: "Connect to the Focal database and run the metadata bootstrap. Return the full bootstrap result."
- The subagent uses the `focal` agent type

Once the focal agent returns, cache the bootstrap result for the session.

## Phase 2: Interview

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

## Phase 3: Generate

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

## Phase 4: Handover

After generating all files:

1. List the generated files with a brief description of each.
2. Ask: "Want me to execute these DDL statements against the database?"
   - If yes: dispatch the focal agent to execute each file in order (peripherals → bridge → synthetics).
   - If no: "Files are ready in `{output_folder}/`. You can run them manually."
3. Suggest: "You can now use `/daana-query` to query the unified star schema."
```

**Step 2: Verify SKILL.md reads correctly**

```bash
wc -l plugin/skills/uss/SKILL.md
```

Expected: File exists and has substantial content.

**Step 3: Commit**

```bash
git add plugin/skills/uss/SKILL.md
git commit -m "feat: create USS skill with interview flow and generation phases"
```

---

## Task 6: Create uss-patterns.md

Create the USS SQL pattern reference with bridge, peripheral, and synthetic peripheral templates.

**Files:**
- Create: `plugin/skills/uss/references/uss-patterns.md`

**Step 1: Create uss-patterns.md**

This file documents the SQL generation patterns for each USS component. It must include:

1. **Peripheral pattern** — CTE-per-descriptor-table with RANK dedup, pivot TYPE_KEY to named columns, join all CTEs on FOCAL key.
2. **Bridge pattern (event-grain, snapshot)** — For each bridge source entity: resolve descriptors (RANK + pivot), resolve relationships (RANK for M:1 FKs), UNPIVOT timestamps to event rows, derive `_key__dates`/`_key__times`, add measures with `_measure__` prefix. UNION ALL across entities with NULL for non-shared measures.
3. **Bridge pattern (event-grain, historical)** — Same as snapshot but preserves `valid_from`/`valid_to` from EFF_TMSTP/VER_TMSTP.
4. **Bridge pattern (columnar, snapshot)** — Same as event-grain but timestamps stay as columns, no `_key__dates`/`_key__times`.
5. **Bridge pattern (columnar, historical)** — Columnar with valid_from/valid_to.
6. **Synthetic date peripheral** — Date spine from bridge min/max year with attributes (year, quarter, month, month_name, day_of_month, day_of_week, day_name).
7. **Synthetic time peripheral** — Time spine at second grain (86,400 rows) with attributes (hour, minute, second).
8. **Fan-out prevention** — How to detect M:1 vs M:M from bootstrap data.
9. **DDL wrapping** — CREATE VIEW vs CREATE TABLE AS templates.

Use concrete SQL examples with placeholder entity/attribute names that clearly map to bootstrap columns. Follow PostgreSQL dialect.

Reference `ad-hoc-query-agent.md` Pattern 1 (RANK) as the base dedup pattern — note this is in the query skill, not locally. The patterns here should be self-contained with full SQL examples.

**Step 2: Commit**

```bash
git add plugin/skills/uss/references/uss-patterns.md
git commit -m "feat: add USS SQL pattern reference for bridge, peripherals, and synthetics"
```

---

## Task 7: Create uss-examples.md

Create a worked example showing how bootstrap data translates to generated USS SQL files.

**Files:**
- Create: `plugin/skills/uss/references/uss-examples.md`

**Step 1: Create uss-examples.md**

This file provides a complete worked example:

1. **Example bootstrap data** — A sample bootstrap result with 3-4 entities (e.g., ORDER, ORDER_LINE, CUSTOMER, PRODUCT) showing focal_name, descriptor_concept_name, atomic_context_name, atom_contx_key, attribute_name, table_pattern_column_name.

2. **Entity classification** — Show how the bootstrap maps to:
   - Bridge sources: ORDER_LINE (has timestamps + numeric measures), ORDER (has timestamps)
   - Peripherals: CUSTOMER (mostly strings, referenced by ORDER), PRODUCT (mostly strings, referenced by ORDER_LINE)

3. **Relationship analysis** — Show how M:1 chains are detected:
   - ORDER_LINE → ORDER (M:1 via ORDER_LINE_ORDER_X)
   - ORDER → CUSTOMER (M:1 via ORDER_CUSTOMER_X)
   - ORDER_LINE → PRODUCT (M:1 via ORDER_LINE_PRODUCT_X)

4. **Generated SQL files** — Complete SQL for each file:
   - `customer.sql` — Peripheral with all CUSTOMER attributes
   - `product.sql` — Peripheral with all PRODUCT attributes
   - `order.sql` — Peripheral with all ORDER attributes
   - `_bridge.sql` — Bridge with event-grain unpivot, snapshot mode
   - `_dates.sql` — Date spine
   - `_times.sql` — Time spine

5. **Variant: columnar mode** — Show how `_bridge.sql` differs in columnar mode (timestamps as columns instead of event rows).

6. **Variant: historical mode** — Show how `valid_from`/`valid_to` columns appear.

**Step 2: Commit**

```bash
git add plugin/skills/uss/references/uss-examples.md
git commit -m "feat: add USS worked example reference"
```

---

## Task 8: Version Bump and Plugin Metadata

Update `plugin.json` version and description to reflect the new skills.

**Files:**
- Modify: `plugin/.claude-plugin/plugin.json`

**Step 1: Bump version**

Update version from `1.8.0` to `1.9.0`. Update description to mention USS:

```json
{
  "name": "daana",
  "description": "Interview-driven DMDL model, mapping, query, and unified star schema builder for the Daana data platform",
  "version": "1.9.0",
  "author": {
    "name": "Mattias Thalén"
  },
  "repository": "https://github.com/mattiasthalen/daana-modeler",
  "license": "MIT",
  "keywords": ["daana", "dmdl", "data-modeling", "data-warehouse", "unified-star-schema"]
}
```

**Step 2: Verify no broken skill references**

```bash
# Check that all referenced files exist
ls plugin/agents/focal/AGENT.md
ls plugin/agents/focal/references/focal-framework.md
ls plugin/agents/focal/references/bootstrap.md
ls plugin/agents/focal/references/connections.md
ls plugin/agents/focal/references/dialect-postgres.md
ls plugin/skills/query/SKILL.md
ls plugin/skills/query/references/ad-hoc-query-agent.md
ls plugin/skills/uss/SKILL.md
ls plugin/skills/uss/references/uss-patterns.md
ls plugin/skills/uss/references/uss-examples.md
ls plugin/skills/star/SKILL.md
ls plugin/skills/star/references/dimension-patterns.md
ls plugin/skills/star/references/fact-patterns.md
```

Expected: All files exist.

```bash
# Check that query/references has only ad-hoc-query-agent.md
ls plugin/skills/query/references/
```

Expected: Only `ad-hoc-query-agent.md`.

**Step 3: Commit**

```bash
git add plugin/.claude-plugin/plugin.json
git commit -m "feat: bump version to 1.9.0 and update plugin metadata for USS skill"
```
