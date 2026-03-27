# USS Subagent Performance Fix — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Constrain the USS Phase 2 subagent to only Read two reference files and Write SQL files, eliminating unnecessary tool calls.

**Architecture:** Single-file edit to `skills/uss/SKILL.md`. Remove connection details and dialect instructions from the subagent prompt, replace embedded reference content with Read instructions, and add explicit tool constraints.

**Tech Stack:** Markdown (Claude Code skill file)

---

### Task 1: Remove parent-level reference read instruction

**Files:**
- Modify: `skills/uss/SKILL.md:17`

**Step 1: Edit SKILL.md**

Remove line 17:
```
Read @references/uss-patterns.md and @references/uss-examples.md before proceeding.
```

This line tells the parent agent to read USS references it doesn't use — the subagent will read them itself.

**Step 2: Commit**

```bash
git add skills/uss/SKILL.md
git commit -m "refactor: remove parent-level USS reference reads

The parent agent doesn't use USS patterns/examples directly.
The subagent will read them itself in Phase 2."
```

---

### Task 2: Update subagent prompt — remove connection details, dialect, and embedded references

**Files:**
- Modify: `skills/uss/SKILL.md:100-147`

**Step 1: Edit the subagent prompt template**

Replace lines 102-147 (the numbered list) with the updated version below. Changes:
- Remove old item #4 (Connection details)
- Remove old item #5 (Dialect instructions)
- Replace old items #6-#7 (embedded USS patterns/examples) with a Read instruction + tool constraints
- Renumber all items

New content for the numbered list (lines 102-147):

```markdown
The subagent prompt MUST include all of the following — the subagent has no other context:

1. **Role:** "You are a SQL DDL generator creating a Unified Star Schema from Focal metadata."
2. **Scope rules:** Copy the Scope section from this skill verbatim:
   - DDL generation only. You produce SQL files — you do not modify existing data.
   - Never hardcode TYPE_KEYs — always resolve from bootstrap.
   - Only follow M:1 relationship chains (no fan-out). Exclude or flag M:M relationships.
   - Never assume physical columns — always resolve via bootstrap.
   - Use the active dialect from the focal context for all SQL generation. Only PostgreSQL patterns are currently implemented.
3. **Tool constraints:**
   - Read the USS reference files listed below, then generate all SQL files using only the Write tool.
   - Do NOT use the Bash tool. Do NOT run any database queries.
   - Do NOT read any files other than the two reference files specified below.
4. **Reference files:** "Before generating any files, read these two files:
   - `{skill_dir}/references/uss-patterns.md` — SQL patterns for all USS components
   - `{skill_dir}/references/uss-examples.md` — Complete worked example with sample SQL"

   Where `{skill_dir}` is the absolute path to the `skills/uss/` directory (resolve at prompt-build time).
5. **Bootstrap data:** The full cached bootstrap result from the current session context, serialized as a markdown table.
6. **Interview answers:**
   - Entity classification: bridge sources and peripherals
   - Temporal mode: event-grain unpivot or columnar dates
   - Peripheral versioning: latest all (Type 1), full history all (Type 2), or per-peripheral with individual choices
   - Materialization: views, tables, mixed, or custom per-file
   - Output folder path
   - Target schema name
7. **Column naming conventions:**

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

8. **File naming rules:**
    - Peripheral entities: lowercased entity name without `_FOCAL` suffix (e.g., `CUSTOMER_FOCAL` -> `customer.sql`)
    - Synthetic files: prefixed with underscore (`_bridge.sql`, `_dates.sql`, `_times.sql`)

9. **DDL wrapping rules:** Based on the user's materialization choice:
    - **View:** `CREATE OR REPLACE VIEW {schema}.{name} AS ...`
    - **Table:** `CREATE TABLE {schema}.{name} AS ...`

10. **Generation order:** Peripherals first, then bridge, then synthetic date peripheral, then synthetic time peripheral.

11. **Output instructions:** "Generate all SQL files and write them to {output_folder}. Return a list of generated files with brief descriptions."
```

**Step 2: Commit**

```bash
git add skills/uss/SKILL.md
git commit -m "fix: constrain USS subagent to Read+Write only (#53)

- Remove connection details and dialect instructions from subagent prompt
- Replace embedded USS patterns/examples with Read instructions
- Add explicit tool constraints: only Read 2 reference files, only Write SQL files
- Reduces expected tool calls from ~73 to ~19"
```

---

### Task 3: Bump plugin version

**Files:**
- Modify: `.claude-plugin/plugin.json`

**Step 1: Bump patch version**

Read `.claude-plugin/plugin.json`, increment the patch version number.

**Step 2: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "chore: bump version to <new_version>"
```
