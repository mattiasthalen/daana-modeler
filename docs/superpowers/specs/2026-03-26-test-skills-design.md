# Test Skills — Design Spec

**Date:** 2026-03-26
**Issue:** #49
**Status:** Brainstorm

## Problem

There are no automated tests for skills. When reference files change (e.g., USS/Star SQL correctness fixes #38-43), we have no way to verify the output is correct without manually invoking each skill against a live database.

The issue proposes mocking the bootstrap & metadata context so query/uss/star skills can be tested without calling the focal skill or connecting to a database.

## Context

### What produces testable output?

| Skill | Output type | Deterministic? | Testable without DB? |
|-------|------------|----------------|---------------------|
| `/daana-focal` | DB connection + bootstrap cache | No — requires live DB | No |
| `/daana-model` | `model.yaml` via interview | No — creative interview | Not meaningfully |
| `/daana-map` | `mappings/*.yaml` via interview | No — creative interview | Not meaningfully |
| `/daana-query` | SQL SELECT statements | Semi — SQL structure is deterministic, aliases/formatting vary | **Yes** — mock bootstrap provides all metadata needed |
| `/daana-uss` | SQL DDL files (views/tables) | Semi — same caveat | **Yes** — mock bootstrap provides all metadata needed |
| `/daana-star` | SQL DDL files (views/tables) | Semi — same caveat (skeleton status) | **Yes** once implemented — mock bootstrap provides all metadata needed |

### What the skills actually consume

Query, USS, and Star skills all follow the same pattern:

1. **Bootstrap data** — the tabular result from `f_focal_read()` with columns: `focal_name`, `focal_physical_schema`, `descriptor_concept_name`, `atomic_context_name`, `atom_contx_key`, `attribute_name`, `table_pattern_column_name`
2. **Connection + dialect details** — host, port, dialect rules
3. **Interview answers** — user choices (entity selection, temporal mode, etc.)
4. **Reference files** — the skill's `references/*.md` files (already in-repo)

Items 1, 2, and 3 can all be mocked. Item 4 is already static. This means we can construct a fully self-contained test context.

### Test domain: Adventure Works

The `adventure-works-ddw` repo (pinned in `external.lock`) is the natural test fixture. It contains:

- **15 entities:** PERSON, ADDRESS, EMPLOYEE, DEPARTMENT, PRODUCT, WORK_ORDER, VENDOR, PURCHASE_ORDER, CUSTOMER, STORE, SALES_PERSON, SALES_TERRITORY, SALES_ORDER, SALES_ORDER_DETAIL, SPECIAL_OFFER
- **19 relationships** forming deep M:1 chains (e.g., SALES_ORDER_DETAIL → SALES_ORDER → CUSTOMER → PERSON)
- A rich `model.yaml` to derive mock bootstrap data from

### Upstream changes

No new changes detected in upstream repos since pinned commits:
- `adventure-works-ddw` — HEAD matches pinned commit `6d41203`
- `teach_claude_focal` — private repo, inaccessible (404)
- `daana-cli` — private repo, inaccessible (404)

## Brainstorm: Approaches

### Approach A: Mock Context Markdown + Pattern Assertions

Create test case markdown files that embed a mocked bootstrap context and specify expected patterns in the output SQL. No live DB, no Claude API calls during testing — a human or Claude session runs the skill against the mock and checks the assertions.

**Structure:**

```
tests/
  fixtures/
    adventure-works-bootstrap.md   # Mock bootstrap as markdown table
    adventure-works-connection.md   # Mock connection + dialect details
  cases/
    query/
      q01-latest-single-entity.md
      q02-latest-multi-entity.md
      q03-history-single-entity.md
      q04-relationship-chain.md
      q05-cutoff-date.md
    uss/
      u01-snapshot-event-grain.md
      u02-snapshot-columnar.md
      u03-historical-event-grain.md
      u04-recursive-peripheral-discovery.md
    star/
      s01-scd-type-2-dimension.md
      s02-transaction-fact.md
```

**Each test case contains:**

```markdown
# Test: Q01 — Latest single-entity query

## Given
<!-- Bootstrap context from fixtures/adventure-works-bootstrap.md -->
<!-- Connection: PostgreSQL, schema: daana_dw -->

## Interview Answers
- Question: "Show me all customers with their account numbers"
- Time dimension: Latest
- Cutoff: None (current)

## Expected SQL Patterns

### MUST contain
- `FROM daana_dw.customer_desc` (correct schema + table)
- `TYPE_KEY = {resolved_key}` (uses bootstrap TYPE_KEY, not hardcoded)
- `RANK() OVER (PARTITION BY CUSTOMER_KEY, TYPE_KEY ORDER BY EFF_TMSTP DESC`
- `ROW_ST = 'Y'`
- `customer_account_number` (correct column naming)

### MUST NOT contain
- `FOCAL01_KEY` or `FOCAL02_KEY` as column names
- `focal.` as schema prefix
- `LIMIT` (no default limit)
- Hardcoded TYPE_KEY values from other installations
```

**Pros:**
- No infrastructure needed — works as documentation and test spec
- Pattern assertions are resilient to formatting/alias changes
- Can be validated by a human reviewer or by Claude in a session
- Serves as regression spec for reference file changes

**Cons:**
- Not automated — requires manual or Claude-assisted verification
- No CI integration without Claude API calls

### Approach B: Test Skill (`/daana-test`)

A dedicated skill that automates Approach A. It reads the fixture files, feeds each test case to the target skill's subagent prompt template (bypassing the interview phase), and validates the output against pattern assertions.

**How it works:**

1. Read `tests/fixtures/adventure-works-bootstrap.md` for mock bootstrap
2. For each test case in `tests/cases/{skill}/`:
   - Parse the "Interview Answers" section
   - Construct the subagent prompt using the target skill's template
   - Dispatch a subagent with the mock context
   - Capture the generated SQL
   - Check MUST contain / MUST NOT contain patterns
   - Report pass/fail

**Pros:**
- Semi-automated — run `/daana-test` and get a pass/fail report
- Reuses existing subagent infrastructure
- Tests the actual skill prompts end-to-end

**Cons:**
- Requires Claude API calls (one subagent per test case)
- Non-deterministic — LLM output varies between runs
- Slower and more expensive than static checks

### Approach C: Golden Output Files + Diff Review

Store "golden" SQL output files alongside test cases. After running a skill, diff the output against the golden file. Accept the diff if it's cosmetic (formatting, alias order), flag it if structural patterns change.

**Structure:**

```
tests/
  golden/
    uss/
      customer.sql
      product.sql
      _bridge.sql
      _dates.sql
    query/
      q01-latest-single-entity.sql
  ...
```

**Pros:**
- Full output validation, not just patterns
- Easy to review diffs in PRs

**Cons:**
- Very brittle — any formatting change breaks the test
- High maintenance burden to keep golden files updated
- Still requires Claude API calls to generate output

### Approach D: Hybrid — Static Spec + Automated Validation Skill

Combine Approaches A and B:

1. **Test case markdown files** (Approach A) serve as the source of truth — human-readable, reviewable, version-controlled
2. **A `/daana-test` skill** (Approach B) can optionally automate validation
3. **Pattern assertions** are the primary check — not exact output match
4. **A verification checklist** (already exists in the USS/Star correctness spec) provides the manual fallback

This gives us:
- **Offline validation** — any developer can read the test cases and manually verify
- **Online validation** — `/daana-test` automates the checks when Claude is available
- **CI-lite** — pattern assertions could eventually be extracted into a simple script (regex checks on SQL output files) without needing Claude

## Recommendation

**Start with Approach D (Hybrid)** but prioritize the static parts first:

### Phase 1: Test fixtures + test case specs (no automation)
1. Create mock bootstrap fixture from adventure-works-ddw model.yaml
2. Write 4-5 test cases per skill (query, USS) with pattern assertions
3. Document the verification checklist per skill

### Phase 2: `/daana-test` skill (optional automation)
4. Build the test skill that reads fixtures and dispatches subagents
5. Add pattern-matching validation logic
6. Report pass/fail per test case

### Phase 3: CI integration (future)
7. Extract pattern assertions into a lightweight script
8. Run as a GitHub Action after reference file changes

## Key Design Decisions Needed

1. **Mock bootstrap format** — Should it be the raw `f_focal_read()` tabular output (markdown table), or a simplified/annotated version? Raw format tests the skill's ability to interpret bootstrap data, which is where bugs hide.

2. **TYPE_KEY values in fixtures** — Use realistic but fictional TYPE_KEY integers (e.g., 100, 101, 102...) so test assertions can reference them. These are installation-specific, so we must never assert on specific TYPE_KEY values in the MUST contain section — only that the skill resolves them from bootstrap.

3. **How many test cases per skill?** Suggested minimum coverage:

   | Skill | Cases | Coverage |
   |-------|-------|----------|
   | Query | 5 | Latest single, latest multi-entity, history single, relationship chain, cutoff date |
   | USS | 4 | Snapshot+event-grain, snapshot+columnar, historical+event-grain, recursive peripheral discovery |
   | Star | 2 | SCD-2 dimension, transaction fact (once star is implemented) |

4. **Assertion granularity** — Pattern checks should test:
   - Schema correctness (`{source_schema}.TABLE` not `focal.TABLE`)
   - Column naming (`product_name` not `name`)
   - Relationship resolution (`attribute_name` not `FOCAL01_KEY`)
   - Structural patterns (RANK dedup, ROW_ST filtering, UNION ALL members)
   - Forbidden patterns (hardcoded TYPE_KEYs, DDL/DML in query skill)

## Adventure Works Bootstrap Fixture Design

The mock bootstrap should be derived from the `model.yaml` in `adventure-works-ddw`. For each entity+attribute, we generate a bootstrap row:

```markdown
| focal_name | focal_physical_schema | descriptor_concept_name | atomic_context_name | atom_contx_key | attribute_name | table_pattern_column_name |
|---|---|---|---|---|---|---|
| CUSTOMER_FOCAL | daana_dw | CUSTOMER_DESC | CUSTOMER_CUSTOMER_ACCOUNT_NUMBER | 100 | CUSTOMER_ACCOUNT_NUMBER | VAL_STR |
| PRODUCT_FOCAL | daana_dw | PRODUCT_DESC | PRODUCT_PRODUCT_NAME | 200 | PRODUCT_NAME | VAL_STR |
| PRODUCT_FOCAL | daana_dw | PRODUCT_DESC | PRODUCT_PRODUCT_LIST_PRICE | 201 | PRODUCT_LIST_PRICE | VAL_NUM |
| ... | | | | | | |
```

Relationship tables get rows with FOCAL01_KEY/FOCAL02_KEY patterns:

```markdown
| focal_name | focal_physical_schema | descriptor_concept_name | atomic_context_name | atom_contx_key | attribute_name | table_pattern_column_name |
|---|---|---|---|---|---|---|
| SALES_ORDER_CUSTOMER_FOCAL | daana_dw | SALES_ORDER_CUSTOMER_X | SALES_ORDER_IS_PLACED_BY_CUSTOMER | 500 | SALES_ORDER_KEY | FOCAL01_KEY |
| SALES_ORDER_CUSTOMER_FOCAL | daana_dw | SALES_ORDER_CUSTOMER_X | SALES_ORDER_IS_PLACED_BY_CUSTOMER | 500 | CUSTOMER_KEY | FOCAL02_KEY |
| ... | | | | | | |
```

This gives skills the exact same data shape they get from a live `f_focal_read()` call, but without needing a database.
