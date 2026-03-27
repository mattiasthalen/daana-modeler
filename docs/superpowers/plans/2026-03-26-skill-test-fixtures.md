# Skill Test Fixtures Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create 16 test fixtures with mocked bootstrap context and expected SQL output for the query, USS, and star skills.

**Architecture:** Each fixture is a markdown file that references shared bootstrap/connection context via `@bootstrap-context.md`. The bootstrap context is real `f_focal_read()` output from Adventure Works DDW. Expected SQL is hand-crafted by applying skill patterns to the bootstrap data.

**Tech Stack:** Markdown, PostgreSQL SQL dialect

---

## Entity Selections for Tests

### Query Tests
- **Entities:** PRODUCT (single entity), SALES_ORDER_DETAIL + PRODUCT (multi-entity)
- **Rationale:** PRODUCT has rich attributes (strings, numerics, timestamps). SALES_ORDER_DETAIL has M:1 to PRODUCT for cross-entity tests.

### USS Tests (same entity selection for all 4 modes)
- **Bridge sources:** SALES_ORDER_DETAIL (measures: order_qty, unit_price, unit_price_discount), SALES_ORDER (timestamps: order_date, due_date, ship_date; measures: sub_total, tax_amt, freight)
- **Direct peripherals:** PRODUCT (via SOD→PRODUCT), CUSTOMER (via SO→CUSTOMER), SPECIAL_OFFER (via SOD→SPECIAL_OFFER)
- **Transitive peripherals:** PERSON (via CUSTOMER→PERSON)
- **Entity classification:**

| Entity | Role | Reason |
|--------|------|--------|
| SALES_ORDER_DETAIL_FOCAL | Bridge source | Has transactional measures (order_qty, unit_price, unit_price_discount) |
| SALES_ORDER_FOCAL | Bridge source | Has event timestamps (order_date, ship_date, due_date) and measures (sub_total, tax_amt, freight) |
| PRODUCT_FOCAL | Peripheral | Referenced by SOD on FOCAL02_KEY side |
| CUSTOMER_FOCAL | Peripheral | Referenced by SO on FOCAL02_KEY side |
| SPECIAL_OFFER_FOCAL | Peripheral | Referenced by SOD on FOCAL02_KEY side |
| PERSON_FOCAL | Peripheral | Referenced by CUSTOMER on FOCAL02_KEY side (transitive) |

- **Relationship chains:**

| Relationship Table | TYPE_KEY | FOCAL01 (many) | FOCAL02 (one) | Direction |
|---|---|---|---|---|
| SALES_ORDER_DETAIL_SALES_ORDER_X | 48 | SALES_ORDER_DETAIL_KEY | SALES_ORDER_KEY | SOD → SO |
| SALES_ORDER_DETAIL_PRODUCT_X | 20 | SALES_ORDER_DETAIL_KEY | PRODUCT_KEY | SOD → PRODUCT |
| SALES_ORDER_DETAIL_SPECIAL_OFFER_X | 3 | SALES_ORDER_DETAIL_KEY | SPECIAL_OFFER_KEY | SOD → SPECIAL_OFFER |
| SALES_ORDER_CUSTOMER_X | 7 | SALES_ORDER_KEY | CUSTOMER_KEY | SO → CUSTOMER |
| CUSTOMER_PERSON_X | 65 | CUSTOMER_KEY | PERSON_KEY | CUSTOMER → PERSON |

### Star Tests
- **Dimension tests:** PRODUCT (rich attributes for SCD demonstrations), CUSTOMER + PERSON (for mixed types)
- **Fact tests:** SALES_ORDER_DETAIL as transaction fact with dimensions PRODUCT, CUSTOMER, SPECIAL_OFFER. SALES_ORDER for periodic/accumulating/factless variations.

---

## Task 1: Create shared context files

> **Depends on:** nothing
> **Parallel with:** nothing (must complete first)

**Files:**
- Create: `tests/bootstrap-context.md`
- Create: `tests/connection-context.md`

**Step 1: Create `tests/bootstrap-context.md`**

Write the real `f_focal_read()` output from Adventure Works DDW as a markdown file. Run this command to get the data:

```bash
PGPASSWORD=devpass psql -h localhost -p 5442 -U dev -d customerdb -P pager=off --csv -c "
SELECT
  fr.FOCAL_NAME,
  fr.FOCAL_PHYSICAL_SCHEMA,
  fr.DESCRIPTOR_CONCEPT_NAME,
  fr.ATOMIC_CONTEXT_NAME,
  fr.ATOM_CONTX_KEY,
  fr.ATTRIBUTE_NAME,
  fr.ATR_KEY,
  tcn.VAL_STR as PHYSICAL_COLUMN
FROM DAANA_METADATA.f_focal_read('9999-12-31') fr
LEFT JOIN DAANA_METADATA.LOGICAL_PHYSICAL_X lp
  ON lp.ATR_KEY = fr.ATR_KEY AND lp.ATOM_CONTX_KEY = fr.ATOM_CONTX_KEY AND lp.ROW_ST = 'Y'
LEFT JOIN DAANA_METADATA.TBL_PTRN_COL_NM tcn
  ON lp.TBL_PTRN_COL_KEY = tcn.TBL_PTRN_COL_KEY AND tcn.ROW_ST = 'Y'
WHERE fr.FOCAL_PHYSICAL_SCHEMA = 'DAANA_DW'
ORDER BY fr.FOCAL_NAME, fr.DESCRIPTOR_CONCEPT_NAME, fr.ATOMIC_CONTEXT_NAME
"
```

Convert the CSV output into a markdown file with this structure:

```markdown
# Bootstrap Context — Adventure Works DDW

This file contains the real `f_focal_read('9999-12-31')` output from the Adventure Works DDW database, used as shared context for all skill test fixtures.

## Bootstrap Data

| focal_name | focal_physical_schema | descriptor_concept_name | atomic_context_name | atom_contx_key | attribute_name | atr_key | physical_column |
|---|---|---|---|---|---|---|---|
| ADDRESS_FOCAL | DAANA_DW | ADDRESS_DESC | ADDRESS_ADDRESS_CITY | 97 | ADDRESS_CITY | ... | VAL_STR |
... (all 313 rows)
```

**Step 2: Create `tests/connection-context.md`**

```markdown
# Connection Context — Adventure Works DDW

Mocked connection profile for test fixtures. Do not execute queries — this is for SQL generation testing only.

## Connection Profile

- **Type:** PostgreSQL
- **Host:** localhost
- **Port:** 5442
- **User:** dev
- **Database:** customerdb
- **Source Schema:** daana_dw
- **Target Schema:** (varies per test — specified in each fixture)

## Dialect Rules

- Use PostgreSQL syntax
- No QUALIFY — use subquery with WHERE clause
- Window frames: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
- Use lowercase for all identifiers
```

**Step 3: Commit**

```bash
git add tests/bootstrap-context.md tests/connection-context.md
git commit -m "test: add shared bootstrap and connection context fixtures (#49)"
```

---

## Task 2: Create query test fixtures

> **Depends on:** Task 1
> **Parallel with:** Tasks 3, 4

**Files:**
- Create: `tests/query/latest-snapshot.md`
- Create: `tests/query/history-single-entity.md`
- Create: `tests/query/history-multi-entity.md`
- Create: `tests/query/cutoff-date-latest.md`

**References to read before generating expected SQL:**
- `skills/query/references/ad-hoc-query-agent.md` — the query patterns (Pattern 1, 2, 3)
- `skills/query/SKILL.md` — the skill workflow and rules

### Step 1: Write `tests/query/latest-snapshot.md`

**Scenario:** "Show me all products with their names and list prices"

**Inputs:**
- Question: "Show me all products with their names and list prices"
- Time dimension: Latest
- Cutoff date: Current (no cutoff)

**Entities used from bootstrap:**
- PRODUCT_FOCAL → PRODUCT_DESC
- TYPE_KEY 22 (PRODUCT_PRODUCT_NAME) → PRODUCT_NAME, VAL_STR
- TYPE_KEY 50 (PRODUCT_PRODUCT_LIST_PRICE) → PRODUCT_LIST_PRICE, VAL_NUM

**Expected pattern:** Pattern 1 (Latest Snapshot) from ad-hoc-query-agent.md — RANK window partitioned by PRODUCT_KEY + TYPE_KEY, ordered by EFF_TMSTP DESC, VER_TMSTP DESC. Filter `WHERE rnk = 1 AND ROW_ST = 'Y'`. Pivot via `MAX(CASE WHEN TYPE_KEY = ... THEN ... END)`.

Generate the fixture file following this template:

```markdown
# Test: Latest Snapshot — Product Names and List Prices

Read @bootstrap-context.md and @connection-context.md before proceeding.

## Skill

`/daana-query`

## Inputs

- **Question:** "Show me all products with their names and list prices"
- **Time dimension:** Latest
- **Cutoff date:** Current (no cutoff)

## Expected Output

​```sql
<generated SQL using Pattern 1 from ad-hoc-query-agent.md>
​```
```

### Step 2: Write `tests/query/history-single-entity.md`

**Scenario:** "Show me the history of product list prices"

**Inputs:**
- Question: "Show me the history of product list prices"
- Time dimension: History
- Cutoff date: Current (no cutoff)

**Entities used from bootstrap:**
- PRODUCT_FOCAL → PRODUCT_DESC
- TYPE_KEY 50 (PRODUCT_PRODUCT_LIST_PRICE) → PRODUCT_LIST_PRICE, VAL_NUM

**Expected pattern:** Pattern 2 (History, Single Entity) from ad-hoc-query-agent.md — Build `twine` CTE with UNION ALL of atomic contexts, carry-forward via window MAX, deduplicate, pivot.

### Step 3: Write `tests/query/history-multi-entity.md`

**Scenario:** "Show me sales order details with product names"

**Inputs:**
- Question: "Show me sales order details with their product names"
- Time dimension: History
- Cutoff date: Current (no cutoff)

**Entities used from bootstrap:**
- SALES_ORDER_DETAIL_FOCAL → SALES_ORDER_DETAIL_DESC (TYPE_KEYs: 60 order_qty/VAL_NUM, 129 unit_price/VAL_NUM, 36 unit_price_discount/VAL_NUM)
- SALES_ORDER_DETAIL_PRODUCT_X (TYPE_KEY 20) → SALES_ORDER_DETAIL_KEY (FOCAL01_KEY), PRODUCT_KEY (FOCAL02_KEY)
- PRODUCT_FOCAL → PRODUCT_DESC (TYPE_KEY 22 product_name/VAL_STR)

**Expected pattern:** Pattern 3 (History, Multi-Entity) from ad-hoc-query-agent.md — Resolve latest relationship first, then merged timeline via UNION ALL across entities.

### Step 4: Write `tests/query/cutoff-date-latest.md`

**Scenario:** "Show me all products with their names and list prices as of 2014-01-01"

**Inputs:**
- Question: "Show me all products with their names and list prices"
- Time dimension: Latest
- Cutoff date: 2014-01-01

**Entities used from bootstrap:** Same as Step 1 (PRODUCT_DESC, TYPE_KEYs 22, 50).

**Expected pattern:** Pattern 1 (Latest Snapshot) + cutoff modifier — add `AND EFF_TMSTP <= '2014-01-01'` to inner WHERE clause.

### Step 5: Commit

```bash
git add tests/query/
git commit -m "test: add query skill test fixtures (#49)"
```

---

## Task 3: Create USS test fixtures

> **Depends on:** Task 1
> **Parallel with:** Tasks 2, 4

**Files:**
- Create: `tests/uss/event-grain-snapshot.md`
- Create: `tests/uss/columnar-dates-snapshot.md`
- Create: `tests/uss/event-grain-historical.md`
- Create: `tests/uss/columnar-dates-historical.md`

**References to read before generating expected SQL:**
- `skills/uss/references/uss-patterns.md` — all USS SQL patterns
- `skills/uss/references/uss-examples.md` — worked example with similar entities
- `skills/uss/SKILL.md` — the skill workflow, column naming, and file naming rules

**Shared interview answers across all 4 USS tests:**
- Entity classification: See "Entity Selections for Tests > USS Tests" section above
- Materialization: All views (`CREATE OR REPLACE VIEW`)
- Output folder: `uss/`
- Target schema: `uss`

**Shared file list per test (6 peripherals + 1 bridge + 0-2 synthetics):**
- `product.sql` — peripheral for PRODUCT
- `customer.sql` — peripheral for CUSTOMER
- `person.sql` — peripheral for PERSON
- `special_offer.sql` — peripheral for SPECIAL_OFFER
- `sales_order.sql` — peripheral for SALES_ORDER (yes, bridge sources also get peripherals)
- `sales_order_detail.sql` — peripheral for SALES_ORDER_DETAIL
- `_bridge.sql` — central bridge table
- `_dates.sql` — synthetic date peripheral (event-grain only)
- `_times.sql` — synthetic time peripheral (event-grain only)

**Column naming rules (from uss-patterns.md):**
1. Take `atomic_context_name` (e.g., `PRODUCT_PRODUCT_NAME`)
2. Identify entity prefix from focal_name without `_FOCAL` (e.g., `PRODUCT`)
3. Strip exactly ONE leading `{ENTITY}_` prefix → `PRODUCT_NAME`
4. Lowercase → `product_name`

**Key bootstrap data for SQL generation:**

SALES_ORDER_DETAIL descriptors (SALES_ORDER_DETAIL_DESC):
| TYPE_KEY | Attribute | Column | Derived name |
|----------|-----------|--------|--------------|
| 131 | SALES_ORDER_DETAIL_CARRIER_TRACKING_NUMBER | VAL_STR | carrier_tracking_number |
| 60 | SALES_ORDER_DETAIL_ORDER_QTY | VAL_NUM | order_qty |
| 129 | SALES_ORDER_DETAIL_UNIT_PRICE | VAL_NUM | unit_price |
| 36 | SALES_ORDER_DETAIL_UNIT_PRICE_DISCOUNT | VAL_NUM | unit_price_discount |

SALES_ORDER descriptors (SALES_ORDER_DESC):
| TYPE_KEY | Attribute | Column | Derived name |
|----------|-----------|--------|--------------|
| 95 | SALES_ORDER_ACCOUNT_NUMBER | VAL_STR | account_number |
| 58 | SALES_ORDER_COMMENT | VAL_STR | comment |
| 76 | SALES_ORDER_DUE_DATE | END_TMSTP | due_date |
| 126 | SALES_ORDER_FREIGHT | VAL_NUM | freight |
| 87 | SALES_ORDER_ONLINE_ORDER_FLAG | VAL_STR | online_order_flag |
| 55 | SALES_ORDER_ORDER_DATE | STA_TMSTP | order_date |
| 26 | SALES_ORDER_PURCHASE_ORDER_NUMBER | VAL_STR | purchase_order_number |
| 10 | SALES_ORDER_SHIP_DATE | END_TMSTP | ship_date |
| 71 | SALES_ORDER_STATUS | VAL_STR | status |
| 110 | SALES_ORDER_SUB_TOTAL | VAL_NUM | sub_total |
| 17 | SALES_ORDER_TAX_AMT | VAL_NUM | tax_amt |

PRODUCT descriptors (PRODUCT_DESC — relevant subset):
| TYPE_KEY | Attribute | Column | Derived name |
|----------|-----------|--------|--------------|
| 90 | PRODUCT_CLASS | VAL_STR | class |
| 118 | PRODUCT_COLOR | VAL_STR | color |
| 116 | PRODUCT_DAYS_TO_MANUFACTURE | VAL_NUM | days_to_manufacture |
| 69 | PRODUCT_DISCONTINUED_DATE | END_TMSTP | discontinued_date |
| 28 | PRODUCT_FINISHED_GOODS_FLAG | VAL_STR | finished_goods_flag |
| 18 | PRODUCT_LINE | VAL_STR | line |
| 50 | PRODUCT_LIST_PRICE | VAL_NUM | list_price |
| 120 | PRODUCT_MAKE_FLAG | VAL_STR | make_flag |
| 22 | PRODUCT_NAME | VAL_STR | name |
| 14 | PRODUCT_NUMBER | VAL_STR | number |
| 67 | PRODUCT_REORDER_POINT | VAL_NUM | reorder_point |
| 34 | PRODUCT_SAFETY_STOCK_LEVEL | VAL_NUM | safety_stock_level |
| 64 | PRODUCT_SELL_END_DATE | END_TMSTP | sell_end_date |
| 35 | PRODUCT_SELL_START_DATE | STA_TMSTP | sell_start_date |
| 82 | PRODUCT_SIZE | VAL_STR | size |
| 41 | PRODUCT_STANDARD_COST | VAL_NUM | standard_cost |
| 42 | PRODUCT_STYLE | VAL_STR | style |
| 15 | PRODUCT_WEIGHT | VAL_NUM | weight |

CUSTOMER descriptors (CUSTOMER_DESC):
| TYPE_KEY | Attribute | Column | Derived name |
|----------|-----------|--------|--------------|
| 89 | CUSTOMER_ACCOUNT_NUMBER | VAL_STR | account_number |

PERSON descriptors (PERSON_DESC):
| TYPE_KEY | Attribute | Column | Derived name |
|----------|-----------|--------|--------------|
| 40 | PERSON_EMAIL_PROMOTION | VAL_NUM | email_promotion |
| 57 | PERSON_FIRST_NAME | VAL_STR | first_name |
| 119 | PERSON_LAST_NAME | VAL_STR | last_name |
| 53 | PERSON_MIDDLE_NAME | VAL_STR | middle_name |
| 68 | PERSON_SUFFIX | VAL_STR | suffix |
| 47 | PERSON_TITLE | VAL_STR | title |
| 25 | PERSON_TYPE | VAL_STR | type |

SPECIAL_OFFER descriptors (SPECIAL_OFFER_DESC):
| TYPE_KEY | Attribute | Column | Derived name |
|----------|-----------|--------|--------------|
| 94 | SPECIAL_OFFER_CATEGORY | VAL_STR | category |
| 1 | SPECIAL_OFFER_DESCRIPTION | VAL_STR | description |
| 44 | SPECIAL_OFFER_DISCOUNT_PCT | VAL_NUM | discount_pct |
| 43 | SPECIAL_OFFER_END_DATE | END_TMSTP | end_date |
| 75 | SPECIAL_OFFER_MAX_QTY | VAL_NUM | max_qty |
| 96 | SPECIAL_OFFER_MIN_QTY | VAL_NUM | min_qty |
| 11 | SPECIAL_OFFER_START_DATE | STA_TMSTP | start_date |
| 19 | SPECIAL_OFFER_TYPE | VAL_STR | type |

### Step 1: Write `tests/uss/event-grain-snapshot.md`

**Inputs:**
- Temporal mode: Event-grain unpivot
- Historical mode: Snapshot (latest values)

This is the main USS path. Apply patterns from uss-patterns.md:
- Peripherals: RANK dedup + pivot per entity's descriptor table
- Bridge: Resolve descriptors (measures + timestamps) per bridge source → resolve relationships → join → unpivot timestamps via CROSS JOIN LATERAL → UNION ALL bridge sources → UNION ALL peripheral rows
- Synthetic: `_dates.sql` and `_times.sql` peripherals

Generate 9 files in the Expected Output section: 6 peripherals + bridge + 2 synthetics.

### Step 2: Write `tests/uss/columnar-dates-snapshot.md`

**Inputs:**
- Temporal mode: Columnar dates (timestamps as named columns, no unpivot)
- Historical mode: Snapshot (latest values)

Differences from event-grain:
- No timestamp unpivot — each timestamp stays as a named column (e.g., `order_date`, `ship_date`)
- No `event` or `event_occurred_on` columns
- No `_key__dates` or `_key__times` FK columns
- No `_dates.sql` or `_times.sql` synthetic peripherals

Generate 7 files: 6 peripherals + bridge.

### Step 3: Write `tests/uss/event-grain-historical.md`

**Inputs:**
- Temporal mode: Event-grain unpivot
- Historical mode: Historical (valid_from / valid_to)

Differences from event-grain snapshot:
- Peripherals get `valid_from` and `valid_to` columns (temporal versioning)
- Bridge gets `valid_from` and `valid_to` columns
- Use temporal join patterns instead of simple RANK dedup

Generate 9 files: 6 peripherals + bridge + 2 synthetics.

### Step 4: Write `tests/uss/columnar-dates-historical.md`

**Inputs:**
- Temporal mode: Columnar dates
- Historical mode: Historical (valid_from / valid_to)

Combines columnar (no unpivot) with historical (valid_from/valid_to).

Generate 7 files: 6 peripherals + bridge.

### Step 5: Commit

```bash
git add tests/uss/
git commit -m "test: add USS skill test fixtures (#49)"
```

---

## Task 4: Create star test fixtures

> **Depends on:** Task 1
> **Parallel with:** Tasks 2, 3

**Files:**
- Create: `tests/star/dimension-type-0.md`
- Create: `tests/star/dimension-type-1.md`
- Create: `tests/star/dimension-type-2.md`
- Create: `tests/star/dimension-mixed.md`
- Create: `tests/star/fact-transaction.md`
- Create: `tests/star/fact-periodic-snapshot.md`
- Create: `tests/star/fact-accumulating-snapshot.md`
- Create: `tests/star/fact-factless.md`

**References to read before generating expected SQL:**
- `skills/star/references/dimension-patterns.md` — SCD type patterns (0, 1, 2, mixed)
- `skills/star/references/fact-patterns.md` — fact table patterns (transaction, periodic, accumulating, factless)
- `skills/star/SKILL.md` — the skill workflow

**Shared settings:**
- Materialization: All views (`CREATE OR REPLACE VIEW`)
- Output folder: `star/`
- Target schema: `star`

### Step 1: Write `tests/star/dimension-type-0.md`

**Scenario:** PRODUCT dimension with SCD Type 0 (retain original)

**Inputs:**
- Entity: PRODUCT
- SCD Type: 0 (all attributes)
- Attributes: product_name (TYPE_KEY 22, VAL_STR), product_number (14, VAL_STR), product_color (118, VAL_STR), product_list_price (50, VAL_NUM)

**Expected pattern:** From dimension-patterns.md Type 0 — order by EFF_TMSTP ASC (first recorded value), RANK=1 per entity+TYPE_KEY. Pivot via CASE/MAX.

### Step 2: Write `tests/star/dimension-type-1.md`

**Scenario:** PRODUCT dimension with SCD Type 1 (overwrite/current only)

**Inputs:**
- Entity: PRODUCT
- SCD Type: 1 (all attributes)
- Same attributes as Step 1

**Expected pattern:** From dimension-patterns.md Type 1 — order by EFF_TMSTP DESC (latest value), RANK=1 per entity+TYPE_KEY, filter ROW_ST='Y'. One row per entity.

### Step 3: Write `tests/star/dimension-type-2.md`

**Scenario:** PRODUCT dimension with SCD Type 2 (full history)

**Inputs:**
- Entity: PRODUCT
- SCD Type: 2 (all attributes)
- Same attributes as Step 1

**Expected pattern:** From dimension-patterns.md Type 2 — temporal alignment pattern. Multiple rows per entity with valid_from/valid_to. Build timeline from all EFF_TMSTPs, carry-forward values.

### Step 4: Write `tests/star/dimension-mixed.md`

**Scenario:** PRODUCT dimension with mixed SCD types

**Inputs:**
- Entity: PRODUCT
- Mixed SCD types:
  - product_name (TYPE_KEY 22): Type 0 (retain original name)
  - product_number (TYPE_KEY 14): Type 1 (current number)
  - product_list_price (TYPE_KEY 50): Type 2 (full price history)
  - product_color (TYPE_KEY 118): Type 2 (full color history)

**Expected pattern:** From dimension-patterns.md Mixed Types — combine Type 0 (ASC), Type 1 (DESC current), and Type 2 (temporal) in a single dimension. Type 2 attributes drive the timeline; Type 0/1 are joined in.

### Step 5: Write `tests/star/fact-transaction.md`

**Scenario:** SALES_ORDER_DETAIL as transaction fact

**Inputs:**
- Fact entity: SALES_ORDER_DETAIL
- Grain: One row per sales order detail (finest grain)
- Measures: order_qty (TYPE_KEY 60, VAL_NUM), unit_price (129, VAL_NUM), unit_price_discount (36, VAL_NUM)
- Dimension FKs: PRODUCT (via SOD_PRODUCT_X, TYPE_KEY 20), SALES_ORDER (via SOD_SO_X, TYPE_KEY 48), SPECIAL_OFFER (via SOD_SPECIAL_OFFER_X, TYPE_KEY 3)
- Date attribute: inherited from SALES_ORDER via relationship (order_date, TYPE_KEY 55, STA_TMSTP)

**Expected pattern:** From fact-patterns.md Transaction Fact — resolve measures via RANK+pivot on descriptor, resolve dimension FKs via RANK on each relationship table, join together. Multi-hop for date: resolve SOD→SO relationship first, then get SO's order_date.

### Step 6: Write `tests/star/fact-periodic-snapshot.md`

**Scenario:** SALES_ORDER as periodic snapshot fact (monthly order summary)

**Inputs:**
- Fact entity: SALES_ORDER
- Grain: One row per sales order per snapshot period
- Measures: sub_total (TYPE_KEY 110, VAL_NUM), tax_amt (17, VAL_NUM), freight (126, VAL_NUM)
- Date attribute: order_date (TYPE_KEY 55, STA_TMSTP)
- Dimension FKs: CUSTOMER (via SO_CUSTOMER_X, TYPE_KEY 7)
- Snapshot period: Month (truncate order_date to month)

**Expected pattern:** From fact-patterns.md Periodic Snapshot — aggregate measures per snapshot period. Resolve measures + timestamps via RANK+pivot, resolve relationships, truncate date to period grain.

### Step 7: Write `tests/star/fact-accumulating-snapshot.md`

**Scenario:** SALES_ORDER as accumulating snapshot fact (order lifecycle)

**Inputs:**
- Fact entity: SALES_ORDER
- Grain: One row per sales order
- Milestone timestamps: order_date (TYPE_KEY 55, STA_TMSTP), ship_date (10, END_TMSTP), due_date (76, END_TMSTP)
- Measures: sub_total (TYPE_KEY 110, VAL_NUM), tax_amt (17, VAL_NUM), freight (126, VAL_NUM)
- Dimension FKs: CUSTOMER (via SO_CUSTOMER_X, TYPE_KEY 7)

**Expected pattern:** From fact-patterns.md Accumulating Snapshot — one row per entity, multiple date columns tracking milestones. Resolve all timestamps + measures + relationships via RANK+pivot pattern.

### Step 8: Write `tests/star/fact-factless.md`

**Scenario:** SALES_ORDER_DETAIL as factless fact (no measures, just tracking which products were ordered)

**Inputs:**
- Fact entity: SALES_ORDER_DETAIL
- Grain: One row per sales order detail
- Measures: None
- Dimension FKs: PRODUCT (via SOD_PRODUCT_X, TYPE_KEY 20), SALES_ORDER (via SOD_SO_X, TYPE_KEY 48), SPECIAL_OFFER (via SOD_SPECIAL_OFFER_X, TYPE_KEY 3)

**Expected pattern:** From fact-patterns.md Factless Fact — resolve dimension FKs only (no measures). RANK on each relationship table, join together. Result is just dimension keys.

### Step 9: Commit

```bash
git add tests/star/
git commit -m "test: add star skill test fixtures (#49)"
```

---

## Parallelism Map

```
Task 1 (shared context)
  ├── Task 2 (query fixtures) ──┐
  ├── Task 3 (USS fixtures)   ──┼── all parallel
  └── Task 4 (star fixtures)  ──┘
```
