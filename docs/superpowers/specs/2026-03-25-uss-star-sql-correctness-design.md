# USS/Star SQL Correctness — Design Spec

**Date:** 2026-03-25
**Milestone:** USS/Star SQL Correctness
**Issues:** #38, #39, #40, #41, #42, #43
**Approach:** Targeted reference file patches

## Context

The USS and Star skills generate SQL DDL from Focal-based data warehouses. Six bugs produce incorrect SQL: missing peripherals, wrong column names, wrong schema names, and undocumented temporal patterns.

All fixes are to skill reference files (`.md`). No structural changes to the skill architecture.

## Scope

| File | Issues |
|------|--------|
| `skills/uss/references/uss-patterns.md` | #38, #39, #40, #42, #43 |
| `skills/uss/references/uss-examples.md` | #38, #41 |
| `skills/uss/SKILL.md` | #38, #42 |
| `skills/star/references/dimension-patterns.md` | #40, #43 |
| `skills/star/references/fact-patterns.md` | #40, #42 |

## Fix Details

### #38 — All entities contribute bridge rows + recursive peripheral discovery

**Problem:** The USS skill only puts "bridge source" entities into the bridge UNION ALL. Peripherals are FK targets only. But in USS, every entity should act as both dimension AND fact — all peripherals must contribute their own rows to the bridge.

Additionally, transitive M:1 chains are not followed. E.g., SALES_ORDER -> CUSTOMER -> PERSON is not resolved — PERSON is never discovered as a peripheral.

**Fix:**
1. Add "Recursive Peripheral Discovery" algorithm to uss-patterns.md:
   - Start with bridge source entities
   - Collect all M:1 targets (FOCAL02_KEY side) as peripherals
   - For each peripheral, check if it has its own M:1 relationships
   - Follow those chains recursively until no new entities are discovered
   - All discovered entities become peripherals
2. Update bridge UNION ALL pattern: every entity (sources AND peripherals) contributes rows to the bridge
3. Update uss-examples.md worked example to show peripheral entities contributing bridge rows
4. Update SKILL.md entity classification to reference recursive discovery

### #39 — Temporal bridge keys include valid_from

**Problem:** When peripherals are historical (valid_from/valid_to), the bridge FK to the peripheral only includes the entity key. This can match the wrong temporal version.

**Fix:** Update historical bridge pattern in uss-patterns.md. For temporal peripherals, the bridge must carry both the entity key AND the peripheral's valid_from to enable point-in-time joins.

### #40 — FOCAL01/02 pattern names used as physical column names

**Problem:** Subagents use `FOCAL01_KEY`/`FOCAL02_KEY` (bootstrap pattern names) as physical column names in SQL. The actual column names are the `attribute_name` values from the bootstrap.

**Fix:**
1. Add a prominent CRITICAL callout at the top of the relationship resolution sections in uss-patterns.md
2. Add inline SQL comments in every relationship template: `-- Use attribute_name, NOT FOCAL01_KEY/FOCAL02_KEY`
3. Add "Common Mistakes" section to uss-patterns.md listing this as mistake #1
4. Apply same reinforcement in star fact-patterns.md and dimension-patterns.md

### #41 — Document temporal join pattern for consumers

**Problem:** The consumer join example in uss-examples.md uses simple key joins (`ON b._key__product = p.PRODUCT_KEY`). For historical peripherals, this produces duplicates due to multiple valid_from versions.

**Fix:** Add "Historical Mode Consumer Patterns" section to uss-examples.md showing the temporal predicate pattern:
```sql
JOIN uss.product p ON b._key__product = p.PRODUCT_KEY
    AND p.valid_from <= b.event_occurred_on
    AND p.valid_to > b.event_occurred_on
```

### #42 — Wrong schema name for source tables

**Problem:** Subagents use `focal.TABLE` instead of `daana_dw.TABLE`. The correct schema comes from the bootstrap's `FOCAL_PHYSICAL_SCHEMA` column.

**Fix:**
1. Add schema rule to prerequisites in uss-patterns.md: source schema = `FOCAL_PHYSICAL_SCHEMA` from bootstrap
2. Use `{source_schema}` variable in all SQL templates instead of hardcoded `daana_dw`
3. Add explicit schema instruction to SKILL.md Phase 2 (before subagent dispatch)
4. Apply same fix in star SKILL.md and fact-patterns.md

### #43 — Dimension column names too aggressively stripped

**Problem:** Column naming strips entity prefix too aggressively. `PRODUCT_PRODUCT_NAME` becomes `name` instead of `product_name`. Multiple dimensions then have ambiguous `name` columns.

**Fix:** Replace vague stripping rule with explicit algorithm:
1. Take `atomic_context_name` (e.g., `PRODUCT_PRODUCT_NAME`)
2. Identify the entity name (e.g., `PRODUCT`) — this is the focal_name without the `_FOCAL` suffix
3. Strip only the leading `{ENTITY}_` prefix once (e.g., `PRODUCT_` -> `PRODUCT_NAME`)
4. Lowercase the result -> `product_name`

Examples:
- `PRODUCT_PRODUCT_NAME` -> `product_name` (not `name`)
- `STORE_STORE_NAME` -> `store_name` (not `name`)
- `CUSTOMER_CUSTOMER_FIRST_NAME` -> `customer_first_name` (not `first_name`)

Apply in both uss-patterns.md and star dimension-patterns.md.

## Verification

After applying fixes:
1. Invoke `/daana-uss` against adventure-works-ddw and verify:
   - All M:1 chain peripherals discovered (CUSTOMER -> PERSON, etc.)
   - All entities contribute bridge rows
   - No `FOCAL01`/`FOCAL02` in generated SQL
   - No `focal.` schema prefix — only `daana_dw.`
   - Column names like `product_name` not `name`
   - Historical mode shows temporal join examples
2. Invoke `/daana-star` against adventure-works-ddw and verify:
   - No `FOCAL01`/`FOCAL02` in generated SQL
   - No `focal.` schema prefix
   - Dimension columns properly named

## Parallelization

The 6 fixes touch overlapping files but different sections. Group by file for implementation:

- **Group A** (uss-patterns.md): #38 recursive discovery + bridge rows, #39 temporal keys, #40 FOCAL callout, #42 schema variable, #43 column naming — sequential edits to one file
- **Group B** (uss-examples.md): #38 worked example update, #41 temporal join docs — sequential edits to one file
- **Group C** (uss SKILL.md): #38 classification update, #42 schema instruction — sequential edits
- **Group D** (star dimension-patterns.md): #40 FOCAL callout, #43 column naming — sequential edits
- **Group E** (star fact-patterns.md): #40 FOCAL callout, #42 schema variable — sequential edits

Groups A-E can be parallelized (different files). Within each group, edits are sequential.
