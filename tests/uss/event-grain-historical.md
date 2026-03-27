# USS Fixture: Event-Grain, Historical (Type 2)

## Interview Answers

- **Bridge sources:** SALES_ORDER_DETAIL_FOCAL, SALES_ORDER_FOCAL
- **Peripherals:** PRODUCT_FOCAL, CUSTOMER_FOCAL, SPECIAL_OFFER_FOCAL, PERSON_FOCAL (transitive via CUSTOMER)
- **Temporal mode:** Event-grain unpivot
- **Peripheral versioning:** Type 2 (full history for all)
- **Materialization:** All views
- **Target schema:** uss
- **Source schema:** daana_dw

## Generated Files (9)

### `product.sql`

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
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY PRODUCT_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.PRODUCT_DESC
),
pivoted AS (
    SELECT
        PRODUCT_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 90 THEN VAL_STR END) AS class,
        MAX(CASE WHEN TYPE_KEY = 118 THEN VAL_STR END) AS color,
        MAX(CASE WHEN TYPE_KEY = 116 THEN VAL_NUM END) AS days_to_manufacture,
        MAX(CASE WHEN TYPE_KEY = 69 THEN END_TMSTP END) AS discontinued_date,
        MAX(CASE WHEN TYPE_KEY = 28 THEN VAL_STR END) AS finished_goods_flag,
        MAX(CASE WHEN TYPE_KEY = 18 THEN VAL_STR END) AS line,
        MAX(CASE WHEN TYPE_KEY = 50 THEN VAL_NUM END) AS list_price,
        MAX(CASE WHEN TYPE_KEY = 120 THEN VAL_STR END) AS make_flag,
        MAX(CASE WHEN TYPE_KEY = 22 THEN VAL_STR END) AS name,
        MAX(CASE WHEN TYPE_KEY = 14 THEN VAL_STR END) AS number,
        MAX(CASE WHEN TYPE_KEY = 67 THEN VAL_NUM END) AS reorder_point,
        MAX(CASE WHEN TYPE_KEY = 34 THEN VAL_NUM END) AS safety_stock_level,
        MAX(CASE WHEN TYPE_KEY = 64 THEN END_TMSTP END) AS sell_end_date,
        MAX(CASE WHEN TYPE_KEY = 35 THEN STA_TMSTP END) AS sell_start_date,
        MAX(CASE WHEN TYPE_KEY = 82 THEN VAL_STR END) AS size,
        MAX(CASE WHEN TYPE_KEY = 41 THEN VAL_NUM END) AS standard_cost,
        MAX(CASE WHEN TYPE_KEY = 42 THEN VAL_STR END) AS style,
        MAX(CASE WHEN TYPE_KEY = 15 THEN VAL_NUM END) AS weight
    FROM ranked
    WHERE rnk = 1
    GROUP BY PRODUCT_KEY, EFF_TMSTP
),
peripheral_final AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY PRODUCT_KEY, EFF_TMSTP) AS _peripheral_key,
        PRODUCT_KEY,
        class,
        color,
        days_to_manufacture,
        discontinued_date,
        finished_goods_flag,
        line,
        list_price,
        make_flag,
        name,
        number,
        reorder_point,
        safety_stock_level,
        sell_end_date,
        sell_start_date,
        size,
        standard_cost,
        style,
        weight,
        EFF_TMSTP AS effective_from,
        COALESCE(
            LEAD(EFF_TMSTP) OVER (
                PARTITION BY PRODUCT_KEY
                ORDER BY EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS effective_to
    FROM pivoted
)
SELECT * FROM peripheral_final

UNION ALL

SELECT
    -1 AS _peripheral_key,
    'UNKNOWN' AS PRODUCT_KEY,
    NULL AS class,
    NULL AS color,
    NULL::numeric AS days_to_manufacture,
    NULL::timestamp AS discontinued_date,
    NULL AS finished_goods_flag,
    NULL AS line,
    NULL::numeric AS list_price,
    NULL AS make_flag,
    NULL AS name,
    NULL AS number,
    NULL::numeric AS reorder_point,
    NULL::numeric AS safety_stock_level,
    NULL::timestamp AS sell_end_date,
    NULL::timestamp AS sell_start_date,
    NULL AS size,
    NULL::numeric AS standard_cost,
    NULL AS style,
    NULL::numeric AS weight,
    '1900-01-01'::timestamp AS effective_from,
    '9999-12-31'::timestamp AS effective_to;
```

### `customer.sql`

```sql
CREATE OR REPLACE VIEW uss.customer AS
WITH ranked AS (
    SELECT
        CUSTOMER_KEY,
        TYPE_KEY,
        VAL_STR,
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY CUSTOMER_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.CUSTOMER_DESC
),
pivoted AS (
    SELECT
        CUSTOMER_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 89 THEN VAL_STR END) AS account_number
    FROM ranked
    WHERE rnk = 1
    GROUP BY CUSTOMER_KEY, EFF_TMSTP
),
peripheral_final AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY CUSTOMER_KEY, EFF_TMSTP) AS _peripheral_key,
        CUSTOMER_KEY,
        account_number,
        EFF_TMSTP AS effective_from,
        COALESCE(
            LEAD(EFF_TMSTP) OVER (
                PARTITION BY CUSTOMER_KEY
                ORDER BY EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS effective_to
    FROM pivoted
)
SELECT * FROM peripheral_final

UNION ALL

SELECT
    -1 AS _peripheral_key,
    'UNKNOWN' AS CUSTOMER_KEY,
    NULL AS account_number,
    '1900-01-01'::timestamp AS effective_from,
    '9999-12-31'::timestamp AS effective_to;
```

### `special_offer.sql`

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
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY SPECIAL_OFFER_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SPECIAL_OFFER_DESC
),
pivoted AS (
    SELECT
        SPECIAL_OFFER_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 94 THEN VAL_STR END) AS category,
        MAX(CASE WHEN TYPE_KEY = 1 THEN VAL_STR END) AS description,
        MAX(CASE WHEN TYPE_KEY = 44 THEN VAL_NUM END) AS discount_pct,
        MAX(CASE WHEN TYPE_KEY = 43 THEN END_TMSTP END) AS end_date,
        MAX(CASE WHEN TYPE_KEY = 75 THEN VAL_NUM END) AS max_qty,
        MAX(CASE WHEN TYPE_KEY = 96 THEN VAL_NUM END) AS min_qty,
        MAX(CASE WHEN TYPE_KEY = 11 THEN STA_TMSTP END) AS start_date,
        MAX(CASE WHEN TYPE_KEY = 19 THEN VAL_STR END) AS type
    FROM ranked
    WHERE rnk = 1
    GROUP BY SPECIAL_OFFER_KEY, EFF_TMSTP
),
peripheral_final AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY SPECIAL_OFFER_KEY, EFF_TMSTP) AS _peripheral_key,
        SPECIAL_OFFER_KEY,
        category,
        description,
        discount_pct,
        end_date,
        max_qty,
        min_qty,
        start_date,
        type,
        EFF_TMSTP AS effective_from,
        COALESCE(
            LEAD(EFF_TMSTP) OVER (
                PARTITION BY SPECIAL_OFFER_KEY
                ORDER BY EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS effective_to
    FROM pivoted
)
SELECT * FROM peripheral_final

UNION ALL

SELECT
    -1 AS _peripheral_key,
    'UNKNOWN' AS SPECIAL_OFFER_KEY,
    NULL AS category,
    NULL AS description,
    NULL::numeric AS discount_pct,
    NULL::timestamp AS end_date,
    NULL::numeric AS max_qty,
    NULL::numeric AS min_qty,
    NULL::timestamp AS start_date,
    NULL AS type,
    '1900-01-01'::timestamp AS effective_from,
    '9999-12-31'::timestamp AS effective_to;
```

### `person.sql`

```sql
CREATE OR REPLACE VIEW uss.person AS
WITH ranked AS (
    SELECT
        PERSON_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY PERSON_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.PERSON_DESC
),
pivoted AS (
    SELECT
        PERSON_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 40 THEN VAL_NUM END) AS email_promotion,
        MAX(CASE WHEN TYPE_KEY = 57 THEN VAL_STR END) AS first_name,
        MAX(CASE WHEN TYPE_KEY = 119 THEN VAL_STR END) AS last_name,
        MAX(CASE WHEN TYPE_KEY = 53 THEN VAL_STR END) AS middle_name,
        MAX(CASE WHEN TYPE_KEY = 68 THEN VAL_STR END) AS suffix,
        MAX(CASE WHEN TYPE_KEY = 47 THEN VAL_STR END) AS title,
        MAX(CASE WHEN TYPE_KEY = 25 THEN VAL_STR END) AS type
    FROM ranked
    WHERE rnk = 1
    GROUP BY PERSON_KEY, EFF_TMSTP
),
peripheral_final AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY PERSON_KEY, EFF_TMSTP) AS _peripheral_key,
        PERSON_KEY,
        email_promotion,
        first_name,
        last_name,
        middle_name,
        suffix,
        title,
        type,
        EFF_TMSTP AS effective_from,
        COALESCE(
            LEAD(EFF_TMSTP) OVER (
                PARTITION BY PERSON_KEY
                ORDER BY EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS effective_to
    FROM pivoted
)
SELECT * FROM peripheral_final

UNION ALL

SELECT
    -1 AS _peripheral_key,
    'UNKNOWN' AS PERSON_KEY,
    NULL::numeric AS email_promotion,
    NULL AS first_name,
    NULL AS last_name,
    NULL AS middle_name,
    NULL AS suffix,
    NULL AS title,
    NULL AS type,
    '1900-01-01'::timestamp AS effective_from,
    '9999-12-31'::timestamp AS effective_to;
```

### `sales_order_detail.sql`

```sql
CREATE OR REPLACE VIEW uss.sales_order_detail AS
WITH ranked AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_DESC
),
pivoted AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 131 THEN VAL_STR END) AS carrier_tracking_number,
        MAX(CASE WHEN TYPE_KEY = 60 THEN VAL_NUM END) AS order_qty,
        MAX(CASE WHEN TYPE_KEY = 129 THEN VAL_NUM END) AS unit_price,
        MAX(CASE WHEN TYPE_KEY = 36 THEN VAL_NUM END) AS unit_price_discount
    FROM ranked
    WHERE rnk = 1
    GROUP BY SALES_ORDER_DETAIL_KEY, EFF_TMSTP
),
peripheral_final AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY SALES_ORDER_DETAIL_KEY, EFF_TMSTP) AS _peripheral_key,
        SALES_ORDER_DETAIL_KEY,
        carrier_tracking_number,
        order_qty,
        unit_price,
        unit_price_discount,
        EFF_TMSTP AS effective_from,
        COALESCE(
            LEAD(EFF_TMSTP) OVER (
                PARTITION BY SALES_ORDER_DETAIL_KEY
                ORDER BY EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS effective_to
    FROM pivoted
)
SELECT * FROM peripheral_final

UNION ALL

SELECT
    -1 AS _peripheral_key,
    'UNKNOWN' AS SALES_ORDER_DETAIL_KEY,
    NULL AS carrier_tracking_number,
    NULL::numeric AS order_qty,
    NULL::numeric AS unit_price,
    NULL::numeric AS unit_price_discount,
    '1900-01-01'::timestamp AS effective_from,
    '9999-12-31'::timestamp AS effective_to;
```

### `sales_order.sql`

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
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY SALES_ORDER_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DESC
),
pivoted AS (
    SELECT
        SALES_ORDER_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 95 THEN VAL_STR END) AS account_number,
        MAX(CASE WHEN TYPE_KEY = 58 THEN VAL_STR END) AS comment,
        MAX(CASE WHEN TYPE_KEY = 76 THEN END_TMSTP END) AS due_date,
        MAX(CASE WHEN TYPE_KEY = 126 THEN VAL_NUM END) AS freight,
        MAX(CASE WHEN TYPE_KEY = 87 THEN VAL_STR END) AS online_order_flag,
        MAX(CASE WHEN TYPE_KEY = 55 THEN STA_TMSTP END) AS order_date,
        MAX(CASE WHEN TYPE_KEY = 26 THEN VAL_STR END) AS purchase_order_number,
        MAX(CASE WHEN TYPE_KEY = 10 THEN END_TMSTP END) AS ship_date,
        MAX(CASE WHEN TYPE_KEY = 71 THEN VAL_STR END) AS status,
        MAX(CASE WHEN TYPE_KEY = 110 THEN VAL_NUM END) AS sub_total,
        MAX(CASE WHEN TYPE_KEY = 17 THEN VAL_NUM END) AS tax_amt
    FROM ranked
    WHERE rnk = 1
    GROUP BY SALES_ORDER_KEY, EFF_TMSTP
),
peripheral_final AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY SALES_ORDER_KEY, EFF_TMSTP) AS _peripheral_key,
        SALES_ORDER_KEY,
        account_number,
        comment,
        due_date,
        freight,
        online_order_flag,
        order_date,
        purchase_order_number,
        ship_date,
        status,
        sub_total,
        tax_amt,
        EFF_TMSTP AS effective_from,
        COALESCE(
            LEAD(EFF_TMSTP) OVER (
                PARTITION BY SALES_ORDER_KEY
                ORDER BY EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS effective_to
    FROM pivoted
)
SELECT * FROM peripheral_final

UNION ALL

SELECT
    -1 AS _peripheral_key,
    'UNKNOWN' AS SALES_ORDER_KEY,
    NULL AS account_number,
    NULL AS comment,
    NULL::timestamp AS due_date,
    NULL::numeric AS freight,
    NULL AS online_order_flag,
    NULL::timestamp AS order_date,
    NULL AS purchase_order_number,
    NULL::timestamp AS ship_date,
    NULL AS status,
    NULL::numeric AS sub_total,
    NULL::numeric AS tax_amt,
    '1900-01-01'::timestamp AS effective_from,
    '9999-12-31'::timestamp AS effective_to;
```

### `_bridge.sql`

```sql
CREATE OR REPLACE VIEW uss._bridge AS
WITH
-- ============================================================
-- SALES_ORDER_DETAIL: Resolve descriptors (measures) — historical
-- ============================================================
ranked_sales_order_detail AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        ROW_ST,
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_DESC
),
sales_order_detail_attrs AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 131 AND ROW_ST = 'Y' THEN VAL_STR END) AS carrier_tracking_number,
        MAX(CASE WHEN TYPE_KEY = 60 AND ROW_ST = 'Y' THEN VAL_NUM END) AS order_qty,
        MAX(CASE WHEN TYPE_KEY = 129 AND ROW_ST = 'Y' THEN VAL_NUM END) AS unit_price,
        MAX(CASE WHEN TYPE_KEY = 36 AND ROW_ST = 'Y' THEN VAL_NUM END) AS unit_price_discount
    FROM ranked_sales_order_detail
    WHERE rnk = 1
    GROUP BY SALES_ORDER_DETAIL_KEY, EFF_TMSTP
),

-- ============================================================
-- SALES_ORDER_DETAIL: Resolve relationships (M:1) — historical
-- ============================================================
ranked_sales_order_detail_sales_order_x AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        SALES_ORDER_KEY,
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_SALES_ORDER_X
    WHERE TYPE_KEY = 48
),
rel_sales_order_detail_sales_order AS (
    SELECT SALES_ORDER_DETAIL_KEY, SALES_ORDER_KEY
    FROM ranked_sales_order_detail_sales_order_x
    WHERE rnk = 1
),
ranked_sales_order_detail_product_x AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        PRODUCT_KEY,
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_PRODUCT_X
    WHERE TYPE_KEY = 20
),
rel_sales_order_detail_product AS (
    SELECT SALES_ORDER_DETAIL_KEY, PRODUCT_KEY
    FROM ranked_sales_order_detail_product_x
    WHERE rnk = 1
),
ranked_sales_order_detail_special_offer_x AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        SPECIAL_OFFER_KEY,
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_SPECIAL_OFFER_X
    WHERE TYPE_KEY = 3
),
rel_sales_order_detail_special_offer AS (
    SELECT SALES_ORDER_DETAIL_KEY, SPECIAL_OFFER_KEY
    FROM ranked_sales_order_detail_special_offer_x
    WHERE rnk = 1
),

-- ============================================================
-- SALES_ORDER: Resolve descriptors (timestamps + measures) — historical
-- ============================================================
ranked_sales_order AS (
    SELECT
        SALES_ORDER_KEY,
        TYPE_KEY,
        VAL_STR,
        VAL_NUM,
        STA_TMSTP,
        END_TMSTP,
        ROW_ST,
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY SALES_ORDER_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DESC
),
sales_order_attrs AS (
    SELECT
        SALES_ORDER_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 95 AND ROW_ST = 'Y' THEN VAL_STR END) AS account_number,
        MAX(CASE WHEN TYPE_KEY = 58 AND ROW_ST = 'Y' THEN VAL_STR END) AS comment,
        MAX(CASE WHEN TYPE_KEY = 76 AND ROW_ST = 'Y' THEN END_TMSTP END) AS due_date,
        MAX(CASE WHEN TYPE_KEY = 126 AND ROW_ST = 'Y' THEN VAL_NUM END) AS freight,
        MAX(CASE WHEN TYPE_KEY = 87 AND ROW_ST = 'Y' THEN VAL_STR END) AS online_order_flag,
        MAX(CASE WHEN TYPE_KEY = 55 AND ROW_ST = 'Y' THEN STA_TMSTP END) AS order_date,
        MAX(CASE WHEN TYPE_KEY = 26 AND ROW_ST = 'Y' THEN VAL_STR END) AS purchase_order_number,
        MAX(CASE WHEN TYPE_KEY = 10 AND ROW_ST = 'Y' THEN END_TMSTP END) AS ship_date,
        MAX(CASE WHEN TYPE_KEY = 71 AND ROW_ST = 'Y' THEN VAL_STR END) AS status,
        MAX(CASE WHEN TYPE_KEY = 110 AND ROW_ST = 'Y' THEN VAL_NUM END) AS sub_total,
        MAX(CASE WHEN TYPE_KEY = 17 AND ROW_ST = 'Y' THEN VAL_NUM END) AS tax_amt
    FROM ranked_sales_order
    WHERE rnk = 1
    GROUP BY SALES_ORDER_KEY, EFF_TMSTP
),

-- ============================================================
-- SALES_ORDER: Resolve relationships (M:1) — historical
-- ============================================================
ranked_sales_order_customer_x AS (
    SELECT
        SALES_ORDER_KEY,
        CUSTOMER_KEY,
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY SALES_ORDER_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_CUSTOMER_X
    WHERE TYPE_KEY = 7
),
rel_sales_order_customer AS (
    SELECT SALES_ORDER_KEY, CUSTOMER_KEY
    FROM ranked_sales_order_customer_x
    WHERE rnk = 1
),

-- ============================================================
-- CUSTOMER: Resolve relationships (M:1) for transitive chain
-- ============================================================
ranked_customer_person_x AS (
    SELECT
        CUSTOMER_KEY,
        PERSON_KEY,
        EFF_TMSTP,
        RANK() OVER (
            PARTITION BY CUSTOMER_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.CUSTOMER_PERSON_X
    WHERE TYPE_KEY = 65
),
rel_customer_person AS (
    SELECT CUSTOMER_KEY, PERSON_KEY
    FROM ranked_customer_person_x
    WHERE rnk = 1
),

-- ============================================================
-- SALES_ORDER: Join descriptors + relationships (point-in-time)
-- ============================================================
sales_order_joined AS (
    SELECT
        soa.SALES_ORDER_KEY,
        soa.EFF_TMSTP,
        soa.account_number,
        soa.comment,
        soa.due_date,
        soa.freight,
        soa.online_order_flag,
        soa.order_date,
        soa.purchase_order_number,
        soa.ship_date,
        soa.status,
        soa.sub_total,
        soa.tax_amt,
        COALESCE(p_customer._peripheral_key, -1) AS _key__customer,
        COALESCE(p_person._peripheral_key, -1) AS _key__person
    FROM sales_order_attrs soa
    LEFT JOIN rel_sales_order_customer r_cust
        ON soa.SALES_ORDER_KEY = r_cust.SALES_ORDER_KEY
    LEFT JOIN rel_customer_person r_pers
        ON r_cust.CUSTOMER_KEY = r_pers.CUSTOMER_KEY
    LEFT JOIN uss.customer p_customer
        ON r_cust.CUSTOMER_KEY = p_customer.CUSTOMER_KEY
        AND soa.EFF_TMSTP >= p_customer.effective_from
        AND soa.EFF_TMSTP < p_customer.effective_to
    LEFT JOIN uss.person p_person
        ON r_pers.PERSON_KEY = p_person.PERSON_KEY
        AND soa.EFF_TMSTP >= p_person.effective_from
        AND soa.EFF_TMSTP < p_person.effective_to
),

-- ============================================================
-- SALES_ORDER: Add valid_from / valid_to
-- ============================================================
sales_order_versioned AS (
    SELECT
        soj.*,
        soj.EFF_TMSTP AS valid_from,
        COALESCE(
            LEAD(soj.EFF_TMSTP) OVER (
                PARTITION BY soj.SALES_ORDER_KEY
                ORDER BY soj.EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS valid_to
    FROM sales_order_joined soj
),

-- ============================================================
-- SALES_ORDER_DETAIL: Join descriptors + relationships + inherit SALES_ORDER data (point-in-time)
-- ============================================================
sales_order_detail_joined AS (
    SELECT
        soda.SALES_ORDER_DETAIL_KEY,
        soda.EFF_TMSTP,
        soda.carrier_tracking_number,
        soda.order_qty,
        soda.unit_price,
        soda.unit_price_discount,
        COALESCE(p_sales_order._peripheral_key, -1) AS _key__sales_order,
        COALESCE(p_product._peripheral_key, -1) AS _key__product,
        COALESCE(p_special_offer._peripheral_key, -1) AS _key__special_offer,
        soj._key__customer,
        soj._key__person,
        soj.order_date,
        soj.due_date,
        soj.ship_date
    FROM sales_order_detail_attrs soda
    LEFT JOIN rel_sales_order_detail_sales_order r_so
        ON soda.SALES_ORDER_DETAIL_KEY = r_so.SALES_ORDER_DETAIL_KEY
    LEFT JOIN rel_sales_order_detail_product r_prd
        ON soda.SALES_ORDER_DETAIL_KEY = r_prd.SALES_ORDER_DETAIL_KEY
    LEFT JOIN rel_sales_order_detail_special_offer r_spo
        ON soda.SALES_ORDER_DETAIL_KEY = r_spo.SALES_ORDER_DETAIL_KEY
    LEFT JOIN sales_order_joined soj
        ON r_so.SALES_ORDER_KEY = soj.SALES_ORDER_KEY
    LEFT JOIN uss.sales_order p_sales_order
        ON r_so.SALES_ORDER_KEY = p_sales_order.SALES_ORDER_KEY
        AND soda.EFF_TMSTP >= p_sales_order.effective_from
        AND soda.EFF_TMSTP < p_sales_order.effective_to
    LEFT JOIN uss.product p_product
        ON r_prd.PRODUCT_KEY = p_product.PRODUCT_KEY
        AND soda.EFF_TMSTP >= p_product.effective_from
        AND soda.EFF_TMSTP < p_product.effective_to
    LEFT JOIN uss.special_offer p_special_offer
        ON r_spo.SPECIAL_OFFER_KEY = p_special_offer.SPECIAL_OFFER_KEY
        AND soda.EFF_TMSTP >= p_special_offer.effective_from
        AND soda.EFF_TMSTP < p_special_offer.effective_to
),

-- ============================================================
-- SALES_ORDER_DETAIL: Add valid_from / valid_to
-- ============================================================
sales_order_detail_versioned AS (
    SELECT
        sodj.*,
        sodj.EFF_TMSTP AS valid_from,
        COALESCE(
            LEAD(sodj.EFF_TMSTP) OVER (
                PARTITION BY sodj.SALES_ORDER_DETAIL_KEY
                ORDER BY sodj.EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS valid_to
    FROM sales_order_detail_joined sodj
),

-- ============================================================
-- SALES_ORDER_DETAIL: Unpivot timestamps to events
-- ============================================================
sales_order_detail_events AS (
    SELECT
        sodv.SALES_ORDER_DETAIL_KEY,
        sodv._key__sales_order,
        sodv._key__product,
        sodv._key__special_offer,
        sodv._key__customer,
        sodv._key__person,
        sodv.order_qty AS _measure__sales_order_detail__order_qty,
        sodv.unit_price AS _measure__sales_order_detail__unit_price,
        sodv.unit_price_discount AS _measure__sales_order_detail__unit_price_discount,
        e.event_name AS event,
        e.event_tmstp AS event_occurred_on,
        e.event_tmstp::date AS _key__dates,
        e.event_tmstp::time AS _key__times,
        sodv.valid_from,
        sodv.valid_to
    FROM sales_order_detail_versioned sodv
    CROSS JOIN LATERAL (
        VALUES
            ('order_date', sodv.order_date),
            ('due_date', sodv.due_date),
            ('ship_date', sodv.ship_date)
    ) AS e(event_name, event_tmstp)
    WHERE e.event_tmstp IS NOT NULL
),

-- ============================================================
-- SALES_ORDER: Unpivot timestamps to events
-- ============================================================
sales_order_events AS (
    SELECT
        sov.SALES_ORDER_KEY,
        sov._key__customer,
        sov._key__person,
        sov.freight AS _measure__sales_order__freight,
        sov.sub_total AS _measure__sales_order__sub_total,
        sov.tax_amt AS _measure__sales_order__tax_amt,
        e.event_name AS event,
        e.event_tmstp AS event_occurred_on,
        e.event_tmstp::date AS _key__dates,
        e.event_tmstp::time AS _key__times,
        sov.valid_from,
        sov.valid_to
    FROM sales_order_versioned sov
    CROSS JOIN LATERAL (
        VALUES
            ('order_date', sov.order_date),
            ('due_date', sov.due_date),
            ('ship_date', sov.ship_date)
    ) AS e(event_name, event_tmstp)
    WHERE e.event_tmstp IS NOT NULL
)

-- ============================================================
-- UNION ALL: Combine all entities into the bridge
-- ============================================================
SELECT
    'sales_order_detail' AS peripheral,
    COALESCE(p_self._peripheral_key, -1) AS _key__sales_order_detail,
    sode._key__sales_order,
    sode._key__product,
    sode._key__special_offer,
    sode._key__customer,
    sode._key__person,
    sode.event,
    sode.event_occurred_on,
    sode._key__dates,
    sode._key__times,
    sode._measure__sales_order_detail__order_qty,
    sode._measure__sales_order_detail__unit_price,
    sode._measure__sales_order_detail__unit_price_discount,
    NULL::numeric AS _measure__sales_order__freight,
    NULL::numeric AS _measure__sales_order__sub_total,
    NULL::numeric AS _measure__sales_order__tax_amt,
    sode.valid_from,
    sode.valid_to
FROM sales_order_detail_events sode
LEFT JOIN uss.sales_order_detail p_self
    ON sode.SALES_ORDER_DETAIL_KEY = p_self.SALES_ORDER_DETAIL_KEY
    AND sode.valid_from >= p_self.effective_from
    AND sode.valid_from < p_self.effective_to

UNION ALL

SELECT
    'sales_order' AS peripheral,
    NULL::bigint AS _key__sales_order_detail,
    COALESCE(p_self._peripheral_key, -1) AS _key__sales_order,
    NULL::bigint AS _key__product,
    NULL::bigint AS _key__special_offer,
    soe._key__customer,
    soe._key__person,
    soe.event,
    soe.event_occurred_on,
    soe._key__dates,
    soe._key__times,
    NULL::numeric AS _measure__sales_order_detail__order_qty,
    NULL::numeric AS _measure__sales_order_detail__unit_price,
    NULL::numeric AS _measure__sales_order_detail__unit_price_discount,
    soe._measure__sales_order__freight,
    soe._measure__sales_order__sub_total,
    soe._measure__sales_order__tax_amt,
    soe.valid_from,
    soe.valid_to
FROM sales_order_events soe
LEFT JOIN uss.sales_order p_self
    ON soe.SALES_ORDER_KEY = p_self.SALES_ORDER_KEY
    AND soe.valid_from >= p_self.effective_from
    AND soe.valid_from < p_self.effective_to

UNION ALL

-- ============================================================
-- PRODUCT: Peripheral bridge rows
-- ============================================================
SELECT
    'product' AS peripheral,
    NULL::bigint AS _key__sales_order_detail,
    NULL::bigint AS _key__sales_order,
    p._peripheral_key AS _key__product,
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
    NULL::numeric AS _measure__sales_order__freight,
    NULL::numeric AS _measure__sales_order__sub_total,
    NULL::numeric AS _measure__sales_order__tax_amt,
    p.effective_from AS valid_from,
    p.effective_to AS valid_to
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
    c._peripheral_key AS _key__customer,
    COALESCE(p_person._peripheral_key, -1) AS _key__person,
    NULL AS event,
    NULL::timestamp AS event_occurred_on,
    NULL::date AS _key__dates,
    NULL::time AS _key__times,
    NULL::numeric AS _measure__sales_order_detail__order_qty,
    NULL::numeric AS _measure__sales_order_detail__unit_price,
    NULL::numeric AS _measure__sales_order_detail__unit_price_discount,
    NULL::numeric AS _measure__sales_order__freight,
    NULL::numeric AS _measure__sales_order__sub_total,
    NULL::numeric AS _measure__sales_order__tax_amt,
    c.effective_from AS valid_from,
    c.effective_to AS valid_to
FROM uss.customer c
LEFT JOIN rel_customer_person r_pers
    ON c.CUSTOMER_KEY = r_pers.CUSTOMER_KEY
LEFT JOIN uss.person p_person
    ON r_pers.PERSON_KEY = p_person.PERSON_KEY
    AND c.effective_from >= p_person.effective_from
    AND c.effective_from < p_person.effective_to

UNION ALL

-- ============================================================
-- SPECIAL_OFFER: Peripheral bridge rows
-- ============================================================
SELECT
    'special_offer' AS peripheral,
    NULL::bigint AS _key__sales_order_detail,
    NULL::bigint AS _key__sales_order,
    NULL::bigint AS _key__product,
    so._peripheral_key AS _key__special_offer,
    NULL::bigint AS _key__customer,
    NULL::bigint AS _key__person,
    NULL AS event,
    NULL::timestamp AS event_occurred_on,
    NULL::date AS _key__dates,
    NULL::time AS _key__times,
    NULL::numeric AS _measure__sales_order_detail__order_qty,
    NULL::numeric AS _measure__sales_order_detail__unit_price,
    NULL::numeric AS _measure__sales_order_detail__unit_price_discount,
    NULL::numeric AS _measure__sales_order__freight,
    NULL::numeric AS _measure__sales_order__sub_total,
    NULL::numeric AS _measure__sales_order__tax_amt,
    so.effective_from AS valid_from,
    so.effective_to AS valid_to
FROM uss.special_offer so

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
    pe._peripheral_key AS _key__person,
    NULL AS event,
    NULL::timestamp AS event_occurred_on,
    NULL::date AS _key__dates,
    NULL::time AS _key__times,
    NULL::numeric AS _measure__sales_order_detail__order_qty,
    NULL::numeric AS _measure__sales_order_detail__unit_price,
    NULL::numeric AS _measure__sales_order_detail__unit_price_discount,
    NULL::numeric AS _measure__sales_order__freight,
    NULL::numeric AS _measure__sales_order__sub_total,
    NULL::numeric AS _measure__sales_order__tax_amt,
    pe.effective_from AS valid_from,
    pe.effective_to AS valid_to
FROM uss.person pe;
```

### `_dates.sql`

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

### `_times.sql`

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
