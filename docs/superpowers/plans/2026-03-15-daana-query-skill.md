# daana-query Skill Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a `/daana-query` skill that answers natural language questions about data in a Focal-based Daana data warehouse via live SQL queries.

**Architecture:** Single SKILL.md file following the same pattern as existing skills (`daana-model`, `daana-mapping`). The skill connects to a PostgreSQL container via `docker exec`, runs discovery queries on startup, then enters a free-form conversational loop translating natural language to SQL.

**Tech Stack:** Claude Code skill (markdown), PostgreSQL via `docker exec` + `psql`

---

## File Map

- **Create:** `skills/daana-query/SKILL.md` — Complete skill definition (the only deliverable)

**Reference files (read-only, do not modify):**
- `skills/daana-model/SKILL.md` — Pattern reference for frontmatter, persona, and structure
- `skills/daana-mapping/SKILL.md` — Pattern reference for frontmatter, persona, and structure
- `docs/superpowers/specs/2026-03-15-daana-query-skill-design.md` — Design spec (source of truth for all behavior)

---

## Chunk 1: Write the SKILL.md

### Task 1: Create SKILL.md with frontmatter and persona

**Files:**
- Create: `skills/daana-query/SKILL.md`
- Read: `skills/daana-model/SKILL.md` (pattern reference for frontmatter format)
- Read: `docs/superpowers/specs/2026-03-15-daana-query-skill-design.md` (spec)

- [ ] **Step 1: Read the existing skills for frontmatter pattern**

Read `skills/daana-model/SKILL.md` and `skills/daana-mapping/SKILL.md` to confirm the exact frontmatter format (`name`, `description`, `disable-model-invocation`).

- [ ] **Step 2: Read the spec**

Read `docs/superpowers/specs/2026-03-15-daana-query-skill-design.md` for the complete behavior specification.

- [ ] **Step 3: Write the complete SKILL.md**

Use the Write tool to create `skills/daana-query/SKILL.md` with all sections. The file must contain:

**Frontmatter (YAML):**
```yaml
---
name: daana-query
description: Data agent that answers natural language questions about Focal-based Daana data warehouses via live SQL queries.
disable-model-invocation: true
---
```

**Sections to include (in order):**

1. **Title and persona** — "You are a data analyst fluent in the Focal framework..." Describe the agent's identity: thinks in entities/attributes/relationships, translates natural language to SQL, explains results in business terms. Note that this is a free-form conversational agent, not a phased interview like the model/mapping skills.

2. **Scope** — Read-only data access only. Never modify data. Never create/edit DMDL model or mapping files. Never make assumptions about business logic not present in metadata.

3. **Connection Setup** — On startup, ask the user three questions (one at a time):
   - Container name (e.g., `daana-test-customerdb`)
   - Database user (e.g., `dev`)
   - Database name (e.g., `customerdb`)

   Then validate with `SELECT 1`:
   ```bash
   docker exec <container> psql -U <user> -d <database> -P pager=off --csv -c "SELECT 1"
   ```
   If validation fails, report the error and ask user to verify details.

4. **Discovery Phase** — After successful connection, run these queries automatically using `--csv` format. Document exact SQL and the `docker exec` command pattern (no `-it` flags, include `-P pager=off --csv`):

   Query 1 — List schemas:
   ```sql
   SELECT schema_name FROM information_schema.schemata
   WHERE schema_name NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
   ORDER BY schema_name;
   ```

   Query 2 — List views and tables in `daana_dw`:
   ```sql
   SELECT table_name, table_type
   FROM information_schema.tables
   WHERE table_schema = 'daana_dw'
   ORDER BY table_type, table_name;
   ```

   Query 3 — Get column details:
   ```sql
   SELECT table_name, column_name, data_type
   FROM information_schema.columns
   WHERE table_schema = 'daana_dw'
   ORDER BY table_name, ordinal_position;
   ```

   Query 4 — For each `{ENTITY}_DESC` table discovered:
   ```sql
   SELECT DISTINCT type_key FROM daana_dw.{entity}_desc ORDER BY type_key;
   ```

   If any discovery query fails (container issues, permission errors, missing schemas), report the error clearly and suggest troubleshooting steps (e.g., "Is the container running? Try `docker ps` to check." or "The `daana_dw` schema doesn't exist — has `daana-cli install` been run?"). If `daana_dw` schema is specifically not found, ask the user for guidance on which schema to target.

   After discovery, greet with a summary: entity count, attribute counts per entity, relationship count. Example: "Connected to customerdb. I found 3 entities: CUSTOMER (8 attributes), ORDER (5 attributes), PRODUCT (4 attributes), and 2 relationships (CUSTOMER-ORDER, ORDER-PRODUCT). What would you like to know?"

5. **Query Generation Rules** — Document the query target selection table:

   | Question Type | Target |
   |---|---|
   | Current state | `VIEW_{ENTITY}` |
   | Historical | `VIEW_{ENTITY}_HIST` |
   | Relationship-based | `VIEW_{ENTITY}_WITH_REL` |
   | Lineage / audit | Raw `_DESC` tables + `INST_KEY` joins |
   | Metadata exploration | `_DESC` tables + TYPE_KEY introspection |

   Always use fully-qualified schema names (e.g., `daana_dw.view_customer`).

6. **Safety Guardrails** — Document all rules:
   - Only `SELECT` statements (or `WITH`/CTE followed by `SELECT`)
   - Refuse INSERT, UPDATE, DELETE, DROP, ALTER, TRUNCATE, or any DDL/DML
   - Default `LIMIT 100`, hard upper limit `LIMIT 1000`
   - Prefix all queries with `SET statement_timeout = '30s';`
   - Agent generates all SQL — never interpolate user text directly into SQL strings
   - All identifiers must come from discovered schema/table/column names

7. **Execution Mechanics** — Document the `docker exec` pattern:

   For user-facing queries, run twice:
   - CSV run (for agent to parse and summarize):
     ```bash
     docker exec <container> psql -U <user> -d <database> -P pager=off --csv -c "SET statement_timeout = '30s'; <SQL>"
     ```
   - Tabular run (for user to read):
     ```bash
     docker exec <container> psql -U <user> -d <database> -P pager=off -c "SET statement_timeout = '30s'; <SQL>"
     ```

8. **Result Presentation** — Every result includes:
   - Raw tabular `psql` output
   - Natural language summary interpreting the results in business terms

   For empty results: explain what was searched and suggest broadening criteria.

9. **Mode Switching** — Two modes:
   - **Confirm mode** (default): Show SQL, ask "Run this?" before executing
   - **Auto-execute mode**: Generate and run immediately

   Switch via conversational cues: "just run it" / "auto mode" → auto-execute, "show me first" / "confirm mode" → confirm.

10. **Conversation Loop Behavior** — Document what the agent should do:
    - Reference discovered metadata for correct column names/types
    - Prefer views over raw tables unless Focal internals needed
    - Handle ambiguity by asking clarification
    - On query error: read Postgres error, fix SQL, retry once, then ask user
    - Suggest follow-up questions based on results
    - Explain entity/attribute meaning from metadata when asked
    - Compare values across time using historical views
    - Trace data lineage via INST_KEY when asked
    - No explicit wrap-up phase — the session ends naturally when the user is done asking questions (contrast with the phased wrap-up in `daana-model` and `daana-mapping`)

11. **Focal Framework Context** — Include the table taxonomy:

    | Table Type | Pattern | Purpose |
    |---|---|---|
    | FOCAL | `{ENTITY}_FOCAL` | One row per entity instance (surrogate key) |
    | IDFR | `{ENTITY}_IDFR` | Business identifier to surrogate key mapping |
    | DESC | `{ENTITY}_DESC` | Descriptive attributes in key-value format via TYPE_KEY |
    | Relationship | `{ENTITY1}_{ENTITY2}_X` | Temporal many-to-many relationships |
    | Current view | `VIEW_{ENTITY}` | Current state snapshot |
    | Historical view | `VIEW_{ENTITY}_HIST` | Full change history |
    | Related view | `VIEW_{ENTITY}_WITH_REL` | Current state with pre-joined relationships |

    And the three timestamp types:
    - `EFF_TMSTP` — Business time (when this version became valid)
    - `VER_TMSTP` — System time (when warehouse recorded this version)
    - `POPLN_TMSTP` — Load time (when row was physically inserted)

- [ ] **Step 4: Verify the file was created**

Use the Glob tool to confirm `skills/daana-query/SKILL.md` exists.

- [ ] **Step 5: Commit**

```bash
git add skills/daana-query/SKILL.md
git commit -m "feat: add daana-query skill for natural language data queries"
```

---

### Task 2: Manual smoke test

**Files:**
- Read: `skills/daana-query/SKILL.md`

- [ ] **Step 1: Verify skill structure**

Read `skills/daana-query/SKILL.md` and verify:
- Frontmatter has correct fields (`name`, `description`, `disable-model-invocation: true`)
- All 11 sections from the spec are present
- Discovery SQL queries are exact (not prose descriptions)
- `docker exec` commands have no `-it` flags
- `--csv` used for discovery and parsing runs
- `-P pager=off` included in all commands
- `SET statement_timeout = '30s';` prefix documented
- Safety guardrails section covers all rules from spec
- Focal table taxonomy and timestamp types are complete

- [ ] **Step 2: Verify against spec**

Read the spec at `docs/superpowers/specs/2026-03-15-daana-query-skill-design.md` and cross-reference every requirement against the SKILL.md. Confirm nothing was missed.

- [ ] **Step 3: Test database connectivity**

Run a quick connectivity test against the actual container to verify the `docker exec` pattern works:
```bash
docker exec daana-test-customerdb psql -U dev -d customerdb -P pager=off --csv -c "SELECT 1"
```

Expected output: a single line with `1`.

If the container is not running, note this but do not block — the skill itself handles connection failures gracefully.

- [ ] **Step 4: Test discovery queries (if container is running)**

Run each discovery query from the SKILL.md against the actual database to verify they return valid results:
```bash
docker exec daana-test-customerdb psql -U dev -d customerdb -P pager=off --csv -c "SELECT schema_name FROM information_schema.schemata WHERE schema_name NOT IN ('pg_catalog', 'information_schema', 'pg_toast') ORDER BY schema_name;"
```

Verify the output is parseable CSV. If `daana_dw` schema exists, run the remaining discovery queries too.
