# Query Patterns Alignment Design

**Date:** 2026-03-18
**Source:** https://github.com/PatrikLager/teach_claude_focal

## Problem

The `/daana-query` skill generates invalid SQL queries. Specific failures:
- Temporal alignment (history) queries produce incorrect pivots or NULLs
- Relationship queries use `FOCAL01_KEY`/`FOCAL02_KEY` pattern names instead of actual column names
- ROW_ST handling is inconsistent between latest and history patterns
- Time dimension questions are not enforced before query construction

## Approach

**Approach B: SKILL.md + query-patterns.md** — Keep SKILL.md focused on the session flow (connection, bootstrap, query loop, consent gates) and move all detailed query patterns into a new reference file.

## File Changes

### New: `plugin/skills/query/query-patterns.md`

Contains all query construction patterns extracted from teach_claude_focal's `agent_workflow.md`:

1. **ROW_ST filtering rules** — Latest: `ROW_ST = 'Y'` + RANK. History: include all rows, null out values with `CASE WHEN ROW_ST = 'Y'`.
2. **Pattern 1: Latest** — Single attribute, multi-attribute pivot (RANK + MAX CASE), cutoff date variant, complex atomic context handling.
3. **Pattern 2: Temporal Alignment** — Full 5-stage CTE (twine -> in_effect -> filtered_in_effect -> per-attribute CTEs -> final join with carry-forward timestamps). Cutoff date variant.
4. **Relationship queries** — Detection from bootstrap, FOCAL01_KEY/FOCAL02_KEY -> attribute_name mapping, joining X tables to DESC tables.
5. **Bootstrap fallback** — Direct `ATOM_CONTX_NM` search when `f_focal_read` unavailable.
6. **Lineage tracing** — `INST_KEY` -> `PROCINST_DESC` pattern.
7. **End-to-end worked example** — Domain-agnostic ("show me total amount per supplier").

### Updated: `plugin/skills/query/SKILL.md`

- **Add** `Read ${CLAUDE_SKILL_DIR}/query-patterns.md` instruction at Phase 3.
- **Add** hard-gate for time dimension: two sequential AskUserQuestion calls (latest vs history, then cutoff date), each with a "don't ask again" option.
- **Add** critical rules to Scope: never assume physical columns, relationship column mapping, no default LIMIT.
- **Remove** inline pattern summaries (Patterns 1-4, ROW_ST rules, relationship summary) — replaced by reference to query-patterns.md.

### No change

- `plugin/skills/query/focal-framework.md`
- `plugin/skills/query/connections.md`
- `plugin/skills/query/dialect-postgres.md`

## Design Decisions

- **Hard-gate with "don't ask again"** for time dimension questions — balances correctness (never assume latest vs history) with usability (power users can opt out).
- **Separate query-patterns.md** rather than inline — matches the established pattern of colocated references (focal-framework.md, connections.md, dialect-postgres.md).
- **Content adapted from teach_claude_focal** — same SQL templates and patterns, but using the plugin's conventions (lowercase schema refs, placeholder syntax).
- **BigQuery support deferred** — PostgreSQL first, BigQuery dialect as follow-up work.
