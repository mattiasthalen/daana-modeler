# Test: Event-Grain Snapshot — Sales Order Detail USS

Read @bootstrap-context.md and @connection-context.md before proceeding.

## Skill

`/daana-uss`

## Inputs

- **Entity classification:**

| Entity | Role | Reason |
|--------|------|--------|
| SALES_ORDER_DETAIL_FOCAL | Bridge source | Has transactional measures (order_qty, unit_price, unit_price_discount) |
| SALES_ORDER_FOCAL | Bridge source | Has event timestamps (order_date, ship_date, due_date) and measures (sub_total, tax_amt, freight) |
| PRODUCT_FOCAL | Peripheral | Referenced by SOD on FOCAL02_KEY side |
| CUSTOMER_FOCAL | Peripheral | Referenced by SO on FOCAL02_KEY side |
| SPECIAL_OFFER_FOCAL | Peripheral | Referenced by SOD on FOCAL02_KEY side |
| PERSON_FOCAL | Peripheral | Referenced by CUSTOMER on FOCAL02_KEY side (transitive) |

- **Temporal mode:** Event-grain unpivot
- **Historical mode:** Snapshot
- **Materialization:** All views
- **Output folder:** uss/
- **Target schema:** uss

## Expected Output

### product.sql

```sql
CREATE OR REPLACE VIEW uss.product AS
WITH ranked AS (
    SELECT
        PRODUCT_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        STA_TMSTP,
        END_TMSTP,
        RANK() OVER (
            PARTITION BY PRODUCT_KEY, TYPE_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.PRODUCT_DESC
    WHERE ROW_ST = 'Y'
)
SELECT
    PRODUCT_KEY,
    MAX(CASE WHEN TYPE_KEY = 90 THEN VAL_STR END) AS product_class,
    MAX(CASE WHEN TYPE_KEY = 118 THEN VAL_STR END) AS product_color,
    MAX(CASE WHEN TYPE_KEY = 116 THEN VAL_NUM END) AS product_days_to_manufacture,
    MAX(CASE WHEN TYPE_KEY = 69 THEN END_TMSTP END) AS product_discontinued_date,
    MAX(CASE WHEN TYPE_KEY = 28 THEN VAL_STR END) AS product_finished_goods_flag,
    MAX(CASE WHEN TYPE_KEY = 18 THEN VAL_STR END) AS product_line,
    MAX(CASE WHEN TYPE_KEY = 50 THEN VAL_NUM END) AS product_list_price,
    MAX(CASE WHEN TYPE_KEY = 120 THEN VAL_STR END) AS product_make_flag,
    MAX(CASE WHEN TYPE_KEY = 22 THEN VAL_STR END) AS product_name,
    MAX(CASE WHEN TYPE_KEY = 14 THEN VAL_STR END) AS product_number,
    MAX(CASE WHEN TYPE_KEY = 67 THEN VAL_NUM END) AS product_reorder_point,
    MAX(CASE WHEN TYPE_KEY = 34 THEN VAL_NUM END) AS product_safety_stock_level,
    MAX(CASE WHEN TYPE_KEY = 64 THEN END_TMSTP END) AS product_sell_end_date,
    MAX(CASE WHEN TYPE_KEY = 35 THEN STA_TMSTP END) AS product_sell_start_date,
    MAX(CASE WHEN TYPE_KEY = 82 THEN VAL_STR END) AS product_size,
    MAX(CASE WHEN TYPE_KEY = 41 THEN VAL_NUM END) AS product_standard_cost,
    MAX(CASE WHEN TYPE_KEY = 42 THEN VAL_STR END) AS product_style,
    MAX(CASE WHEN TYPE_KEY = 15 THEN VAL_NUM END) AS product_weight
FROM ranked
WHERE rnk = 1
GROUP BY PRODUCT_KEY;
```

### customer.sql

```sql
CREATE OR REPLACE VIEW uss.customer AS
WITH ranked AS (
    SELECT
        CUSTOMER_KEY,
        TYPE_KEY,
        VAL_STR,
        RANK() OVER (
            PARTITION BY CUSTOMER_KEY, TYPE_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.CUSTOMER_DESC
    WHERE ROW_ST = 'Y'
),
customer_attrs AS (
    SELECT
        CUSTOMER_KEY,
        MAX(CASE WHEN TYPE_KEY = 89 THEN VAL_STR END) AS customer_account_number
    FROM ranked
    WHERE rnk = 1
    GROUP BY CUSTOMER_KEY
),
ranked_customer_person_x AS (
    SELECT
        CUSTOMER_KEY,
        PERSON_KEY,
        RANK() OVER (
            PARTITION BY CUSTOMER_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.CUSTOMER_PERSON_X
    WHERE ROW_ST = 'Y'
      AND TYPE_KEY = 65
),
rel_person AS (
    SELECT CUSTOMER_KEY, PERSON_KEY
    FROM ranked_customer_person_x
    WHERE rnk = 1
)
SELECT
    c.CUSTOMER_KEY,
    c.customer_account_number,
    r.PERSON_KEY
FROM customer_attrs c
LEFT JOIN rel_person r
    ON c.CUSTOMER_KEY = r.CUSTOMER_KEY;
```

### special_offer.sql

```sql
CREATE OR REPLACE VIEW uss.special_offer AS
WITH ranked AS (
    SELECT
        SPECIAL_OFFER_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        STA_TMSTP,
        END_TMSTP,
        RANK() OVER (
            PARTITION BY SPECIAL_OFFER_KEY, TYPE_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SPECIAL_OFFER_DESC
    WHERE ROW_ST = 'Y'
)
SELECT
    SPECIAL_OFFER_KEY,
    MAX(CASE WHEN TYPE_KEY = 94 THEN VAL_STR END) AS special_offer_category,
    MAX(CASE WHEN TYPE_KEY = 1 THEN VAL_STR END) AS special_offer_description,
    MAX(CASE WHEN TYPE_KEY = 44 THEN VAL_NUM END) AS special_offer_discount_pct,
    MAX(CASE WHEN TYPE_KEY = 43 THEN END_TMSTP END) AS special_offer_end_date,
    MAX(CASE WHEN TYPE_KEY = 75 THEN VAL_NUM END) AS special_offer_max_qty,
    MAX(CASE WHEN TYPE_KEY = 96 THEN VAL_NUM END) AS special_offer_min_qty,
    MAX(CASE WHEN TYPE_KEY = 11 THEN STA_TMSTP END) AS special_offer_start_date,
    MAX(CASE WHEN TYPE_KEY = 19 THEN VAL_STR END) AS special_offer_type
FROM ranked
WHERE rnk = 1
GROUP BY SPECIAL_OFFER_KEY;
```

### person.sql

```sql
CREATE OR REPLACE VIEW uss.person AS
WITH ranked AS (
    SELECT
        PERSON_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        RANK() OVER (
            PARTITION BY PERSON_KEY, TYPE_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.PERSON_DESC
    WHERE ROW_ST = 'Y'
)
SELECT
    PERSON_KEY,
    MAX(CASE WHEN TYPE_KEY = 40 THEN VAL_NUM END) AS person_email_promotion,
    MAX(CASE WHEN TYPE_KEY = 57 THEN VAL_STR END) AS person_first_name,
    MAX(CASE WHEN TYPE_KEY = 119 THEN VAL_STR END) AS person_last_name,
    MAX(CASE WHEN TYPE_KEY = 53 THEN VAL_STR END) AS person_middle_name,
    MAX(CASE WHEN TYPE_KEY = 68 THEN VAL_STR END) AS person_suffix,
    MAX(CASE WHEN TYPE_KEY = 47 THEN VAL_STR END) AS person_title,
    MAX(CASE WHEN TYPE_KEY = 25 THEN VAL_STR END) AS person_type
FROM ranked
WHERE rnk = 1
GROUP BY PERSON_KEY;
```

### sales_order_detail.sql

```sql
CREATE OR REPLACE VIEW uss.sales_order_detail AS
WITH ranked AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY, TYPE_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_DESC
    WHERE ROW_ST = 'Y'
),
sod_attrs AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        MAX(CASE WHEN TYPE_KEY = 131 THEN VAL_STR END) AS sales_order_detail_carrier_tracking_number,
        MAX(CASE WHEN TYPE_KEY = 60 THEN VAL_NUM END) AS sales_order_detail_order_qty,
        MAX(CASE WHEN TYPE_KEY = 129 THEN VAL_NUM END) AS sales_order_detail_unit_price,
        MAX(CASE WHEN TYPE_KEY = 36 THEN VAL_NUM END) AS sales_order_detail_unit_price_discount
    FROM ranked
    WHERE rnk = 1
    GROUP BY SALES_ORDER_DETAIL_KEY
),
ranked_sod_sales_order_x AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        SALES_ORDER_KEY,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_SALES_ORDER_X
    WHERE ROW_ST = 'Y'
      AND TYPE_KEY = 48
),
rel_sales_order AS (
    SELECT SALES_ORDER_DETAIL_KEY, SALES_ORDER_KEY
    FROM ranked_sod_sales_order_x
    WHERE rnk = 1
),
ranked_sod_product_x AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        PRODUCT_KEY,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_PRODUCT_X
    WHERE ROW_ST = 'Y'
      AND TYPE_KEY = 20
),
rel_product AS (
    SELECT SALES_ORDER_DETAIL_KEY, PRODUCT_KEY
    FROM ranked_sod_product_x
    WHERE rnk = 1
),
ranked_sod_special_offer_x AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        SPECIAL_OFFER_KEY,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_SPECIAL_OFFER_X
    WHERE ROW_ST = 'Y'
      AND TYPE_KEY = 3
),
rel_special_offer AS (
    SELECT SALES_ORDER_DETAIL_KEY, SPECIAL_OFFER_KEY
    FROM ranked_sod_special_offer_x
    WHERE rnk = 1
)
SELECT
    a.SALES_ORDER_DETAIL_KEY,
    a.sales_order_detail_carrier_tracking_number,
    a.sales_order_detail_order_qty,
    a.sales_order_detail_unit_price,
    a.sales_order_detail_unit_price_discount,
    r_so.SALES_ORDER_KEY,
    r_p.PRODUCT_KEY,
    r_spo.SPECIAL_OFFER_KEY
FROM sod_attrs a
LEFT JOIN rel_sales_order r_so
    ON a.SALES_ORDER_DETAIL_KEY = r_so.SALES_ORDER_DETAIL_KEY
LEFT JOIN rel_product r_p
    ON a.SALES_ORDER_DETAIL_KEY = r_p.SALES_ORDER_DETAIL_KEY
LEFT JOIN rel_special_offer r_spo
    ON a.SALES_ORDER_DETAIL_KEY = r_spo.SALES_ORDER_DETAIL_KEY;
```

### sales_order.sql

```sql
CREATE OR REPLACE VIEW uss.sales_order AS
WITH ranked AS (
    SELECT
        SALES_ORDER_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        STA_TMSTP,
        END_TMSTP,
        RANK() OVER (
            PARTITION BY SALES_ORDER_KEY, TYPE_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DESC
    WHERE ROW_ST = 'Y'
),
so_attrs AS (
    SELECT
        SALES_ORDER_KEY,
        MAX(CASE WHEN TYPE_KEY = 95 THEN VAL_STR END) AS sales_order_account_number,
        MAX(CASE WHEN TYPE_KEY = 58 THEN VAL_STR END) AS sales_order_comment,
        MAX(CASE WHEN TYPE_KEY = 76 THEN END_TMSTP END) AS sales_order_due_date,
        MAX(CASE WHEN TYPE_KEY = 126 THEN VAL_NUM END) AS sales_order_freight,
        MAX(CASE WHEN TYPE_KEY = 87 THEN VAL_STR END) AS sales_order_online_order_flag,
        MAX(CASE WHEN TYPE_KEY = 55 THEN STA_TMSTP END) AS sales_order_order_date,
        MAX(CASE WHEN TYPE_KEY = 26 THEN VAL_STR END) AS sales_order_purchase_order_number,
        MAX(CASE WHEN TYPE_KEY = 10 THEN END_TMSTP END) AS sales_order_ship_date,
        MAX(CASE WHEN TYPE_KEY = 71 THEN VAL_STR END) AS sales_order_status,
        MAX(CASE WHEN TYPE_KEY = 110 THEN VAL_NUM END) AS sales_order_sub_total,
        MAX(CASE WHEN TYPE_KEY = 17 THEN VAL_NUM END) AS sales_order_tax_amt
    FROM ranked
    WHERE rnk = 1
    GROUP BY SALES_ORDER_KEY
),
ranked_so_customer_x AS (
    SELECT
        SALES_ORDER_KEY,
        CUSTOMER_KEY,
        RANK() OVER (
            PARTITION BY SALES_ORDER_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_CUSTOMER_X
    WHERE ROW_ST = 'Y'
      AND TYPE_KEY = 7
),
rel_customer AS (
    SELECT SALES_ORDER_KEY, CUSTOMER_KEY
    FROM ranked_so_customer_x
    WHERE rnk = 1
)
SELECT
    s.SALES_ORDER_KEY,
    s.sales_order_account_number,
    s.sales_order_comment,
    s.sales_order_due_date,
    s.sales_order_freight,
    s.sales_order_online_order_flag,
    s.sales_order_order_date,
    s.sales_order_purchase_order_number,
    s.sales_order_ship_date,
    s.sales_order_status,
    s.sales_order_sub_total,
    s.sales_order_tax_amt,
    r.CUSTOMER_KEY
FROM so_attrs s
LEFT JOIN rel_customer r
    ON s.SALES_ORDER_KEY = r.SALES_ORDER_KEY;
```

### _bridge.sql

```sql
CREATE OR REPLACE VIEW uss._bridge AS
WITH
-- ============================================================
-- SALES_ORDER_DETAIL: Resolve descriptors (measures)
-- ============================================================
ranked_sales_order_detail AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY, TYPE_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_DESC
    WHERE ROW_ST = 'Y'
),
sod_attrs AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        MAX(CASE WHEN TYPE_KEY = 131 THEN VAL_STR END) AS carrier_tracking_number,
        MAX(CASE WHEN TYPE_KEY = 60 THEN VAL_NUM END) AS order_qty,
        MAX(CASE WHEN TYPE_KEY = 129 THEN VAL_NUM END) AS unit_price,
        MAX(CASE WHEN TYPE_KEY = 36 THEN VAL_NUM END) AS unit_price_discount
    FROM ranked_sales_order_detail
    WHERE rnk = 1
    GROUP BY SALES_ORDER_DETAIL_KEY
),

-- ============================================================
-- SALES_ORDER_DETAIL: Resolve relationships (M:1)
-- ============================================================
ranked_sod_sales_order_x AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        SALES_ORDER_KEY,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_SALES_ORDER_X
    WHERE ROW_ST = 'Y'
      AND TYPE_KEY = 48
),
rel_sod_sales_order AS (
    SELECT SALES_ORDER_DETAIL_KEY, SALES_ORDER_KEY
    FROM ranked_sod_sales_order_x
    WHERE rnk = 1
),
ranked_sod_product_x AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        PRODUCT_KEY,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_PRODUCT_X
    WHERE ROW_ST = 'Y'
      AND TYPE_KEY = 20
),
rel_sod_product AS (
    SELECT SALES_ORDER_DETAIL_KEY, PRODUCT_KEY
    FROM ranked_sod_product_x
    WHERE rnk = 1
),
ranked_sod_special_offer_x AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        SPECIAL_OFFER_KEY,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_SPECIAL_OFFER_X
    WHERE ROW_ST = 'Y'
      AND TYPE_KEY = 3
),
rel_sod_special_offer AS (
    SELECT SALES_ORDER_DETAIL_KEY, SPECIAL_OFFER_KEY
    FROM ranked_sod_special_offer_x
    WHERE rnk = 1
),

-- ============================================================
-- SALES_ORDER_DETAIL: Join descriptors + relationships
-- ============================================================
sod_joined AS (
    SELECT
        a.SALES_ORDER_DETAIL_KEY,
        a.carrier_tracking_number,
        a.order_qty,
        a.unit_price,
        a.unit_price_discount,
        r_so.SALES_ORDER_KEY AS _key__sales_order,
        r_p.PRODUCT_KEY AS _key__product,
        r_spo.SPECIAL_OFFER_KEY AS _key__special_offer
    FROM sod_attrs a
    LEFT JOIN rel_sod_sales_order r_so
        ON a.SALES_ORDER_DETAIL_KEY = r_so.SALES_ORDER_DETAIL_KEY
    LEFT JOIN rel_sod_product r_p
        ON a.SALES_ORDER_DETAIL_KEY = r_p.SALES_ORDER_DETAIL_KEY
    LEFT JOIN rel_sod_special_offer r_spo
        ON a.SALES_ORDER_DETAIL_KEY = r_spo.SALES_ORDER_DETAIL_KEY
),

-- ============================================================
-- SALES_ORDER: Resolve descriptors (timestamps + measures)
-- ============================================================
ranked_sales_order AS (
    SELECT
        SALES_ORDER_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        STA_TMSTP,
        END_TMSTP,
        RANK() OVER (
            PARTITION BY SALES_ORDER_KEY, TYPE_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DESC
    WHERE ROW_ST = 'Y'
),
so_attrs AS (
    SELECT
        SALES_ORDER_KEY,
        MAX(CASE WHEN TYPE_KEY = 55 THEN STA_TMSTP END) AS order_date,
        MAX(CASE WHEN TYPE_KEY = 10 THEN END_TMSTP END) AS ship_date,
        MAX(CASE WHEN TYPE_KEY = 76 THEN END_TMSTP END) AS due_date,
        MAX(CASE WHEN TYPE_KEY = 110 THEN VAL_NUM END) AS sub_total,
        MAX(CASE WHEN TYPE_KEY = 17 THEN VAL_NUM END) AS tax_amt,
        MAX(CASE WHEN TYPE_KEY = 126 THEN VAL_NUM END) AS freight
    FROM ranked_sales_order
    WHERE rnk = 1
    GROUP BY SALES_ORDER_KEY
),

-- ============================================================
-- SALES_ORDER: Resolve relationships (M:1)
-- ============================================================
ranked_so_customer_x AS (
    SELECT
        SALES_ORDER_KEY,
        CUSTOMER_KEY,
        RANK() OVER (
            PARTITION BY SALES_ORDER_KEY
            ORDER BY EFF_TMSTP DESC, VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_CUSTOMER_X
    WHERE ROW_ST = 'Y'
      AND TYPE_KEY = 7
),
rel_so_customer AS (
    SELECT SALES_ORDER_KEY, CUSTOMER_KEY
    FROM ranked_so_customer_x
    WHERE rnk = 1
),

-- ============================================================
-- SALES_ORDER: Join descriptors + relationships
-- ============================================================
so_joined AS (
    SELECT
        o.SALES_ORDER_KEY,
        o.order_date,
        o.ship_date,
        o.due_date,
        o.sub_total,
        o.tax_amt,
        o.freight,
        r_cust.CUSTOMER_KEY AS _key__customer
    FROM so_attrs o
    LEFT JOIN rel_so_customer r_cust
        ON o.SALES_ORDER_KEY = r_cust.SALES_ORDER_KEY
),

-- ============================================================
-- SALES_ORDER_DETAIL: Inherit SALES_ORDER data (multi-hop)
-- ============================================================
sod_with_sales_order AS (
    SELECT
        sod.SALES_ORDER_DETAIL_KEY,
        sod._key__sales_order,
        sod._key__product,
        sod._key__special_offer,
        sod.carrier_tracking_number,
        sod.order_qty,
        sod.unit_price,
        sod.unit_price_discount,
        so._key__customer,
        so.order_date,
        so.ship_date,
        so.due_date
    FROM sod_joined sod
    LEFT JOIN so_joined so
        ON sod._key__sales_order = so.SALES_ORDER_KEY
),

-- ============================================================
-- SALES_ORDER_DETAIL: Unpivot timestamps to events
-- ============================================================
sod_events AS (
    SELECT
        sod.SALES_ORDER_DETAIL_KEY,
        sod._key__sales_order,
        sod._key__product,
        sod._key__special_offer,
        sod._key__customer,
        sod.order_qty AS _measure__sales_order_detail__order_qty,
        sod.unit_price AS _measure__sales_order_detail__unit_price,
        sod.unit_price_discount AS _measure__sales_order_detail__unit_price_discount,
        e.event_name AS event,
        e.event_tmstp AS event_occurred_on,
        e.event_tmstp::date AS _key__dates,
        e.event_tmstp::time AS _key__times
    FROM sod_with_sales_order sod
    CROSS JOIN LATERAL (
        VALUES
            ('order_date', sod.order_date),
            ('ship_date', sod.ship_date),
            ('due_date', sod.due_date)
    ) AS e(event_name, event_tmstp)
    WHERE e.event_tmstp IS NOT NULL
),

-- ============================================================
-- SALES_ORDER: Unpivot timestamps to events
-- ============================================================
so_events AS (
    SELECT
        so.SALES_ORDER_KEY,
        so._key__customer,
        so.sub_total AS _measure__sales_order__sub_total,
        so.tax_amt AS _measure__sales_order__tax_amt,
        so.freight AS _measure__sales_order__freight,
        e.event_name AS event,
        e.event_tmstp AS event_occurred_on,
        e.event_tmstp::date AS _key__dates,
        e.event_tmstp::time AS _key__times
    FROM so_joined so
    CROSS JOIN LATERAL (
        VALUES
            ('order_date', so.order_date),
            ('ship_date', so.ship_date),
            ('due_date', so.due_date)
    ) AS e(event_name, event_tmstp)
    WHERE e.event_tmstp IS NOT NULL
)

-- ============================================================
-- UNION ALL: Combine all entities into the bridge
-- ============================================================
SELECT
    'sales_order_detail' AS peripheral,
    sode.SALES_ORDER_DETAIL_KEY AS _key__sales_order_detail,
    sode._key__sales_order,
    sode._key__product,
    sode._key__special_offer,
    sode._key__customer,
    NULL::bigint AS _key__person,
    sode.event,
    sode.event_occurred_on,
    sode._key__dates,
    sode._key__times,
    sode._measure__sales_order_detail__order_qty,
    sode._measure__sales_order_detail__unit_price,
    sode._measure__sales_order_detail__unit_price_discount,
    NULL::numeric AS _measure__sales_order__sub_total,
    NULL::numeric AS _measure__sales_order__tax_amt,
    NULL::numeric AS _measure__sales_order__freight
FROM sod_events sode

UNION ALL

SELECT
    'sales_order' AS peripheral,
    NULL::bigint AS _key__sales_order_detail,
    soe.SALES_ORDER_KEY AS _key__sales_order,
    NULL::bigint AS _key__product,
    NULL::bigint AS _key__special_offer,
    soe._key__customer,
    NULL::bigint AS _key__person,
    soe.event,
    soe.event_occurred_on,
    soe._key__dates,
    soe._key__times,
    NULL::numeric AS _measure__sales_order_detail__order_qty,
    NULL::numeric AS _measure__sales_order_detail__unit_price,
    NULL::numeric AS _measure__sales_order_detail__unit_price_discount,
    soe._measure__sales_order__sub_total,
    soe._measure__sales_order__tax_amt,
    soe._measure__sales_order__freight
FROM so_events soe

UNION ALL

-- ============================================================
-- PRODUCT: Peripheral bridge rows
-- ============================================================
SELECT
    'product' AS peripheral,
    NULL::bigint AS _key__sales_order_detail,
    NULL::bigint AS _key__sales_order,
    p.PRODUCT_KEY AS _key__product,
    NULL::bigint AS _key__special_offer,
    NULL::bigint AS _key__customer,
    NULL::bigint AS _key__person,
    NULL AS event,
    NULL::timestamp AS event_occurred_on,
    NULL::date AS _key__dates,
    NULL::time AS _key__times,
    NULL::numeric AS _measure__sales_order_detail__order_qty,
    NULL::numeric AS _measure__sales_order_detail__unit_price,
    NULL::numeric AS _measure__sales_order_detail__unit_price_discount,
    NULL::numeric AS _measure__sales_order__sub_total,
    NULL::numeric AS _measure__sales_order__tax_amt,
    NULL::numeric AS _measure__sales_order__freight
FROM uss.product p

UNION ALL

-- ============================================================
-- CUSTOMER: Peripheral bridge rows
-- ============================================================
SELECT
    'customer' AS peripheral,
    NULL::bigint AS _key__sales_order_detail,
    NULL::bigint AS _key__sales_order,
    NULL::bigint AS _key__product,
    NULL::bigint AS _key__special_offer,
    c.CUSTOMER_KEY AS _key__customer,
    c.PERSON_KEY AS _key__person,
    NULL AS event,
    NULL::timestamp AS event_occurred_on,
    NULL::date AS _key__dates,
    NULL::time AS _key__times,
    NULL::numeric AS _measure__sales_order_detail__order_qty,
    NULL::numeric AS _measure__sales_order_detail__unit_price,
    NULL::numeric AS _measure__sales_order_detail__unit_price_discount,
    NULL::numeric AS _measure__sales_order__sub_total,
    NULL::numeric AS _measure__sales_order__tax_amt,
    NULL::numeric AS _measure__sales_order__freight
FROM uss.customer c

UNION ALL

-- ============================================================
-- SPECIAL_OFFER: Peripheral bridge rows
-- ============================================================
SELECT
    'special_offer' AS peripheral,
    NULL::bigint AS _key__sales_order_detail,
    NULL::bigint AS _key__sales_order,
    NULL::bigint AS _key__product,
    sp.SPECIAL_OFFER_KEY AS _key__special_offer,
    NULL::bigint AS _key__customer,
    NULL::bigint AS _key__person,
    NULL AS event,
    NULL::timestamp AS event_occurred_on,
    NULL::date AS _key__dates,
    NULL::time AS _key__times,
    NULL::numeric AS _measure__sales_order_detail__order_qty,
    NULL::numeric AS _measure__sales_order_detail__unit_price,
    NULL::numeric AS _measure__sales_order_detail__unit_price_discount,
    NULL::numeric AS _measure__sales_order__sub_total,
    NULL::numeric AS _measure__sales_order__tax_amt,
    NULL::numeric AS _measure__sales_order__freight
FROM uss.special_offer sp

UNION ALL

-- ============================================================
-- PERSON: Peripheral bridge rows
-- ============================================================
SELECT
    'person' AS peripheral,
    NULL::bigint AS _key__sales_order_detail,
    NULL::bigint AS _key__sales_order,
    NULL::bigint AS _key__product,
    NULL::bigint AS _key__special_offer,
    NULL::bigint AS _key__customer,
    p.PERSON_KEY AS _key__person,
    NULL AS event,
    NULL::timestamp AS event_occurred_on,
    NULL::date AS _key__dates,
    NULL::time AS _key__times,
    NULL::numeric AS _measure__sales_order_detail__order_qty,
    NULL::numeric AS _measure__sales_order_detail__unit_price,
    NULL::numeric AS _measure__sales_order_detail__unit_price_discount,
    NULL::numeric AS _measure__sales_order__sub_total,
    NULL::numeric AS _measure__sales_order__tax_amt,
    NULL::numeric AS _measure__sales_order__freight
FROM uss.person p;
```

### _dates.sql

```sql
CREATE OR REPLACE VIEW uss._dates AS
WITH date_range AS (
    SELECT
        DATE_TRUNC('year', MIN(event_occurred_on))::date AS start_date,
        (DATE_TRUNC('year', MAX(event_occurred_on)) + INTERVAL '1 year' - INTERVAL '1 day')::date AS end_date
    FROM uss._bridge
),
date_spine AS (
    SELECT
        d::date AS date_key
    FROM date_range,
         GENERATE_SERIES(date_range.start_date, date_range.end_date, '1 day'::interval) AS d
)
SELECT
    date_key AS _key__dates,
    EXTRACT(YEAR FROM date_key)::int AS year,
    EXTRACT(QUARTER FROM date_key)::int AS quarter,
    EXTRACT(MONTH FROM date_key)::int AS month,
    TO_CHAR(date_key, 'Month') AS month_name,
    EXTRACT(DAY FROM date_key)::int AS day_of_month,
    EXTRACT(ISODOW FROM date_key)::int AS day_of_week,
    TO_CHAR(date_key, 'Day') AS day_name,
    EXTRACT(DOY FROM date_key)::int AS day_of_year,
    EXTRACT(WEEK FROM date_key)::int AS iso_week,
    CASE
        WHEN EXTRACT(ISODOW FROM date_key) IN (6, 7) THEN FALSE
        ELSE TRUE
    END AS is_weekday
FROM date_spine
ORDER BY date_key;
```

### _times.sql

```sql
CREATE OR REPLACE VIEW uss._times AS
WITH time_spine AS (
    SELECT
        (INTERVAL '0 seconds' + (s || ' seconds')::interval)::time AS time_key
    FROM GENERATE_SERIES(0, 86399) AS s
)
SELECT
    time_key AS _key__times,
    EXTRACT(HOUR FROM time_key)::int AS hour,
    EXTRACT(MINUTE FROM time_key)::int AS minute,
    EXTRACT(SECOND FROM time_key)::int AS second,
    CASE
        WHEN EXTRACT(HOUR FROM time_key) < 6 THEN 'Night'
        WHEN EXTRACT(HOUR FROM time_key) < 12 THEN 'Morning'
        WHEN EXTRACT(HOUR FROM time_key) < 18 THEN 'Afternoon'
        ELSE 'Evening'
    END AS day_part
FROM time_spine
ORDER BY time_key;
```
