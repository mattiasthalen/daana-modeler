# Test: Event-Grain Historical — Sales Order Detail USS

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
- **Historical mode:** Historical (valid_from / valid_to)
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
        EFF_TMSTP,
        ROW_ST,
        RANK() OVER (
            PARTITION BY PRODUCT_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.PRODUCT_DESC
),
product_attrs AS (
    SELECT
        PRODUCT_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 90 AND ROW_ST = 'Y' THEN VAL_STR END) AS product_class,
        MAX(CASE WHEN TYPE_KEY = 118 AND ROW_ST = 'Y' THEN VAL_STR END) AS product_color,
        MAX(CASE WHEN TYPE_KEY = 116 AND ROW_ST = 'Y' THEN VAL_NUM END) AS product_days_to_manufacture,
        MAX(CASE WHEN TYPE_KEY = 69 AND ROW_ST = 'Y' THEN END_TMSTP END) AS product_discontinued_date,
        MAX(CASE WHEN TYPE_KEY = 28 AND ROW_ST = 'Y' THEN VAL_STR END) AS product_finished_goods_flag,
        MAX(CASE WHEN TYPE_KEY = 18 AND ROW_ST = 'Y' THEN VAL_STR END) AS product_line,
        MAX(CASE WHEN TYPE_KEY = 50 AND ROW_ST = 'Y' THEN VAL_NUM END) AS product_list_price,
        MAX(CASE WHEN TYPE_KEY = 120 AND ROW_ST = 'Y' THEN VAL_STR END) AS product_make_flag,
        MAX(CASE WHEN TYPE_KEY = 22 AND ROW_ST = 'Y' THEN VAL_STR END) AS product_name,
        MAX(CASE WHEN TYPE_KEY = 14 AND ROW_ST = 'Y' THEN VAL_STR END) AS product_number,
        MAX(CASE WHEN TYPE_KEY = 67 AND ROW_ST = 'Y' THEN VAL_NUM END) AS product_reorder_point,
        MAX(CASE WHEN TYPE_KEY = 34 AND ROW_ST = 'Y' THEN VAL_NUM END) AS product_safety_stock_level,
        MAX(CASE WHEN TYPE_KEY = 64 AND ROW_ST = 'Y' THEN END_TMSTP END) AS product_sell_end_date,
        MAX(CASE WHEN TYPE_KEY = 35 AND ROW_ST = 'Y' THEN STA_TMSTP END) AS product_sell_start_date,
        MAX(CASE WHEN TYPE_KEY = 82 AND ROW_ST = 'Y' THEN VAL_STR END) AS product_size,
        MAX(CASE WHEN TYPE_KEY = 41 AND ROW_ST = 'Y' THEN VAL_NUM END) AS product_standard_cost,
        MAX(CASE WHEN TYPE_KEY = 42 AND ROW_ST = 'Y' THEN VAL_STR END) AS product_style,
        MAX(CASE WHEN TYPE_KEY = 15 AND ROW_ST = 'Y' THEN VAL_NUM END) AS product_weight
    FROM ranked
    WHERE rnk = 1
    GROUP BY PRODUCT_KEY, EFF_TMSTP
),
product_versioned AS (
    SELECT
        p.*,
        p.EFF_TMSTP AS valid_from,
        COALESCE(
            LEAD(p.EFF_TMSTP) OVER (
                PARTITION BY p.PRODUCT_KEY
                ORDER BY p.EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS valid_to
    FROM product_attrs p
)
SELECT
    PRODUCT_KEY,
    product_class,
    product_color,
    product_days_to_manufacture,
    product_discontinued_date,
    product_finished_goods_flag,
    product_line,
    product_list_price,
    product_make_flag,
    product_name,
    product_number,
    product_reorder_point,
    product_safety_stock_level,
    product_sell_end_date,
    product_sell_start_date,
    product_size,
    product_standard_cost,
    product_style,
    product_weight,
    valid_from,
    valid_to
FROM product_versioned;
```

### customer.sql

```sql
CREATE OR REPLACE VIEW uss.customer AS
WITH ranked AS (
    SELECT
        CUSTOMER_KEY,
        TYPE_KEY,
        VAL_STR,
        EFF_TMSTP,
        ROW_ST,
        RANK() OVER (
            PARTITION BY CUSTOMER_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.CUSTOMER_DESC
),
customer_attrs AS (
    SELECT
        CUSTOMER_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 89 AND ROW_ST = 'Y' THEN VAL_STR END) AS customer_account_number
    FROM ranked
    WHERE rnk = 1
    GROUP BY CUSTOMER_KEY, EFF_TMSTP
),
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
rel_person AS (
    SELECT CUSTOMER_KEY, PERSON_KEY, EFF_TMSTP
    FROM ranked_customer_person_x
    WHERE rnk = 1
),
customer_joined AS (
    SELECT
        c.CUSTOMER_KEY,
        c.customer_account_number,
        r.PERSON_KEY,
        c.EFF_TMSTP
    FROM customer_attrs c
    LEFT JOIN rel_person r
        ON c.CUSTOMER_KEY = r.CUSTOMER_KEY
        AND c.EFF_TMSTP = r.EFF_TMSTP
),
customer_versioned AS (
    SELECT
        cj.*,
        cj.EFF_TMSTP AS valid_from,
        COALESCE(
            LEAD(cj.EFF_TMSTP) OVER (
                PARTITION BY cj.CUSTOMER_KEY
                ORDER BY cj.EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS valid_to
    FROM customer_joined cj
)
SELECT
    CUSTOMER_KEY,
    customer_account_number,
    PERSON_KEY,
    valid_from,
    valid_to
FROM customer_versioned;
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
        EFF_TMSTP,
        ROW_ST,
        RANK() OVER (
            PARTITION BY SPECIAL_OFFER_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SPECIAL_OFFER_DESC
),
special_offer_attrs AS (
    SELECT
        SPECIAL_OFFER_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 94 AND ROW_ST = 'Y' THEN VAL_STR END) AS special_offer_category,
        MAX(CASE WHEN TYPE_KEY = 1 AND ROW_ST = 'Y' THEN VAL_STR END) AS special_offer_description,
        MAX(CASE WHEN TYPE_KEY = 44 AND ROW_ST = 'Y' THEN VAL_NUM END) AS special_offer_discount_pct,
        MAX(CASE WHEN TYPE_KEY = 43 AND ROW_ST = 'Y' THEN END_TMSTP END) AS special_offer_end_date,
        MAX(CASE WHEN TYPE_KEY = 75 AND ROW_ST = 'Y' THEN VAL_NUM END) AS special_offer_max_qty,
        MAX(CASE WHEN TYPE_KEY = 96 AND ROW_ST = 'Y' THEN VAL_NUM END) AS special_offer_min_qty,
        MAX(CASE WHEN TYPE_KEY = 11 AND ROW_ST = 'Y' THEN STA_TMSTP END) AS special_offer_start_date,
        MAX(CASE WHEN TYPE_KEY = 19 AND ROW_ST = 'Y' THEN VAL_STR END) AS special_offer_type
    FROM ranked
    WHERE rnk = 1
    GROUP BY SPECIAL_OFFER_KEY, EFF_TMSTP
),
special_offer_versioned AS (
    SELECT
        s.*,
        s.EFF_TMSTP AS valid_from,
        COALESCE(
            LEAD(s.EFF_TMSTP) OVER (
                PARTITION BY s.SPECIAL_OFFER_KEY
                ORDER BY s.EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS valid_to
    FROM special_offer_attrs s
)
SELECT
    SPECIAL_OFFER_KEY,
    special_offer_category,
    special_offer_description,
    special_offer_discount_pct,
    special_offer_end_date,
    special_offer_max_qty,
    special_offer_min_qty,
    special_offer_start_date,
    special_offer_type,
    valid_from,
    valid_to
FROM special_offer_versioned;
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
        EFF_TMSTP,
        ROW_ST,
        RANK() OVER (
            PARTITION BY PERSON_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.PERSON_DESC
),
person_attrs AS (
    SELECT
        PERSON_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 40 AND ROW_ST = 'Y' THEN VAL_NUM END) AS person_email_promotion,
        MAX(CASE WHEN TYPE_KEY = 57 AND ROW_ST = 'Y' THEN VAL_STR END) AS person_first_name,
        MAX(CASE WHEN TYPE_KEY = 119 AND ROW_ST = 'Y' THEN VAL_STR END) AS person_last_name,
        MAX(CASE WHEN TYPE_KEY = 53 AND ROW_ST = 'Y' THEN VAL_STR END) AS person_middle_name,
        MAX(CASE WHEN TYPE_KEY = 68 AND ROW_ST = 'Y' THEN VAL_STR END) AS person_suffix,
        MAX(CASE WHEN TYPE_KEY = 47 AND ROW_ST = 'Y' THEN VAL_STR END) AS person_title,
        MAX(CASE WHEN TYPE_KEY = 25 AND ROW_ST = 'Y' THEN VAL_STR END) AS person_type
    FROM ranked
    WHERE rnk = 1
    GROUP BY PERSON_KEY, EFF_TMSTP
),
person_versioned AS (
    SELECT
        p.*,
        p.EFF_TMSTP AS valid_from,
        COALESCE(
            LEAD(p.EFF_TMSTP) OVER (
                PARTITION BY p.PERSON_KEY
                ORDER BY p.EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS valid_to
    FROM person_attrs p
)
SELECT
    PERSON_KEY,
    person_email_promotion,
    person_first_name,
    person_last_name,
    person_middle_name,
    person_suffix,
    person_title,
    person_type,
    valid_from,
    valid_to
FROM person_versioned;
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
        EFF_TMSTP,
        ROW_ST,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_DESC
),
sod_attrs AS (
    SELECT
        SALES_ORDER_DETAIL_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 131 AND ROW_ST = 'Y' THEN VAL_STR END) AS sales_order_detail_carrier_tracking_number,
        MAX(CASE WHEN TYPE_KEY = 60 AND ROW_ST = 'Y' THEN VAL_NUM END) AS sales_order_detail_order_qty,
        MAX(CASE WHEN TYPE_KEY = 129 AND ROW_ST = 'Y' THEN VAL_NUM END) AS sales_order_detail_unit_price,
        MAX(CASE WHEN TYPE_KEY = 36 AND ROW_ST = 'Y' THEN VAL_NUM END) AS sales_order_detail_unit_price_discount
    FROM ranked
    WHERE rnk = 1
    GROUP BY SALES_ORDER_DETAIL_KEY, EFF_TMSTP
),
ranked_sod_sales_order_x AS (
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
rel_sales_order AS (
    SELECT SALES_ORDER_DETAIL_KEY, SALES_ORDER_KEY, EFF_TMSTP
    FROM ranked_sod_sales_order_x
    WHERE rnk = 1
),
ranked_sod_product_x AS (
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
rel_product AS (
    SELECT SALES_ORDER_DETAIL_KEY, PRODUCT_KEY, EFF_TMSTP
    FROM ranked_sod_product_x
    WHERE rnk = 1
),
ranked_sod_special_offer_x AS (
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
rel_special_offer AS (
    SELECT SALES_ORDER_DETAIL_KEY, SPECIAL_OFFER_KEY, EFF_TMSTP
    FROM ranked_sod_special_offer_x
    WHERE rnk = 1
),
sod_joined AS (
    SELECT
        a.SALES_ORDER_DETAIL_KEY,
        a.sales_order_detail_carrier_tracking_number,
        a.sales_order_detail_order_qty,
        a.sales_order_detail_unit_price,
        a.sales_order_detail_unit_price_discount,
        r_so.SALES_ORDER_KEY,
        r_p.PRODUCT_KEY,
        r_spo.SPECIAL_OFFER_KEY,
        a.EFF_TMSTP
    FROM sod_attrs a
    LEFT JOIN rel_sales_order r_so
        ON a.SALES_ORDER_DETAIL_KEY = r_so.SALES_ORDER_DETAIL_KEY
        AND a.EFF_TMSTP = r_so.EFF_TMSTP
    LEFT JOIN rel_product r_p
        ON a.SALES_ORDER_DETAIL_KEY = r_p.SALES_ORDER_DETAIL_KEY
        AND a.EFF_TMSTP = r_p.EFF_TMSTP
    LEFT JOIN rel_special_offer r_spo
        ON a.SALES_ORDER_DETAIL_KEY = r_spo.SALES_ORDER_DETAIL_KEY
        AND a.EFF_TMSTP = r_spo.EFF_TMSTP
),
sod_versioned AS (
    SELECT
        sj.*,
        sj.EFF_TMSTP AS valid_from,
        COALESCE(
            LEAD(sj.EFF_TMSTP) OVER (
                PARTITION BY sj.SALES_ORDER_DETAIL_KEY
                ORDER BY sj.EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS valid_to
    FROM sod_joined sj
)
SELECT
    SALES_ORDER_DETAIL_KEY,
    sales_order_detail_carrier_tracking_number,
    sales_order_detail_order_qty,
    sales_order_detail_unit_price,
    sales_order_detail_unit_price_discount,
    SALES_ORDER_KEY,
    PRODUCT_KEY,
    SPECIAL_OFFER_KEY,
    valid_from,
    valid_to
FROM sod_versioned;
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
        EFF_TMSTP,
        ROW_ST,
        RANK() OVER (
            PARTITION BY SALES_ORDER_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DESC
),
so_attrs AS (
    SELECT
        SALES_ORDER_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 95 AND ROW_ST = 'Y' THEN VAL_STR END) AS sales_order_account_number,
        MAX(CASE WHEN TYPE_KEY = 58 AND ROW_ST = 'Y' THEN VAL_STR END) AS sales_order_comment,
        MAX(CASE WHEN TYPE_KEY = 76 AND ROW_ST = 'Y' THEN END_TMSTP END) AS sales_order_due_date,
        MAX(CASE WHEN TYPE_KEY = 126 AND ROW_ST = 'Y' THEN VAL_NUM END) AS sales_order_freight,
        MAX(CASE WHEN TYPE_KEY = 87 AND ROW_ST = 'Y' THEN VAL_STR END) AS sales_order_online_order_flag,
        MAX(CASE WHEN TYPE_KEY = 55 AND ROW_ST = 'Y' THEN STA_TMSTP END) AS sales_order_order_date,
        MAX(CASE WHEN TYPE_KEY = 26 AND ROW_ST = 'Y' THEN VAL_STR END) AS sales_order_purchase_order_number,
        MAX(CASE WHEN TYPE_KEY = 10 AND ROW_ST = 'Y' THEN END_TMSTP END) AS sales_order_ship_date,
        MAX(CASE WHEN TYPE_KEY = 71 AND ROW_ST = 'Y' THEN VAL_STR END) AS sales_order_status,
        MAX(CASE WHEN TYPE_KEY = 110 AND ROW_ST = 'Y' THEN VAL_NUM END) AS sales_order_sub_total,
        MAX(CASE WHEN TYPE_KEY = 17 AND ROW_ST = 'Y' THEN VAL_NUM END) AS sales_order_tax_amt
    FROM ranked
    WHERE rnk = 1
    GROUP BY SALES_ORDER_KEY, EFF_TMSTP
),
ranked_so_customer_x AS (
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
rel_customer AS (
    SELECT SALES_ORDER_KEY, CUSTOMER_KEY, EFF_TMSTP
    FROM ranked_so_customer_x
    WHERE rnk = 1
),
so_joined AS (
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
        r.CUSTOMER_KEY,
        s.EFF_TMSTP
    FROM so_attrs s
    LEFT JOIN rel_customer r
        ON s.SALES_ORDER_KEY = r.SALES_ORDER_KEY
        AND s.EFF_TMSTP = r.EFF_TMSTP
),
so_versioned AS (
    SELECT
        sj.*,
        sj.EFF_TMSTP AS valid_from,
        COALESCE(
            LEAD(sj.EFF_TMSTP) OVER (
                PARTITION BY sj.SALES_ORDER_KEY
                ORDER BY sj.EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS valid_to
    FROM so_joined sj
)
SELECT
    SALES_ORDER_KEY,
    sales_order_account_number,
    sales_order_comment,
    sales_order_due_date,
    sales_order_freight,
    sales_order_online_order_flag,
    sales_order_order_date,
    sales_order_purchase_order_number,
    sales_order_ship_date,
    sales_order_status,
    sales_order_sub_total,
    sales_order_tax_amt,
    CUSTOMER_KEY,
    valid_from,
    valid_to
FROM so_versioned;
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
        EFF_TMSTP,
        ROW_ST,
        RANK() OVER (
            PARTITION BY SALES_ORDER_DETAIL_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DETAIL_DESC
),
sod_attrs AS (
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
-- SALES_ORDER_DETAIL: Resolve relationships (M:1)
-- ============================================================
ranked_sod_sales_order_x AS (
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
rel_sod_sales_order AS (
    SELECT SALES_ORDER_DETAIL_KEY, SALES_ORDER_KEY, EFF_TMSTP
    FROM ranked_sod_sales_order_x
    WHERE rnk = 1
),
ranked_sod_product_x AS (
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
rel_sod_product AS (
    SELECT SALES_ORDER_DETAIL_KEY, PRODUCT_KEY, EFF_TMSTP
    FROM ranked_sod_product_x
    WHERE rnk = 1
),
ranked_sod_special_offer_x AS (
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
rel_sod_special_offer AS (
    SELECT SALES_ORDER_DETAIL_KEY, SPECIAL_OFFER_KEY, EFF_TMSTP
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
        r_spo.SPECIAL_OFFER_KEY AS _key__special_offer,
        a.EFF_TMSTP
    FROM sod_attrs a
    LEFT JOIN rel_sod_sales_order r_so
        ON a.SALES_ORDER_DETAIL_KEY = r_so.SALES_ORDER_DETAIL_KEY
        AND a.EFF_TMSTP = r_so.EFF_TMSTP
    LEFT JOIN rel_sod_product r_p
        ON a.SALES_ORDER_DETAIL_KEY = r_p.SALES_ORDER_DETAIL_KEY
        AND a.EFF_TMSTP = r_p.EFF_TMSTP
    LEFT JOIN rel_sod_special_offer r_spo
        ON a.SALES_ORDER_DETAIL_KEY = r_spo.SALES_ORDER_DETAIL_KEY
        AND a.EFF_TMSTP = r_spo.EFF_TMSTP
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
        EFF_TMSTP,
        ROW_ST,
        RANK() OVER (
            PARTITION BY SALES_ORDER_KEY, TYPE_KEY, EFF_TMSTP
            ORDER BY VER_TMSTP DESC
        ) AS rnk
    FROM daana_dw.SALES_ORDER_DESC
),
so_attrs AS (
    SELECT
        SALES_ORDER_KEY,
        EFF_TMSTP,
        MAX(CASE WHEN TYPE_KEY = 55 AND ROW_ST = 'Y' THEN STA_TMSTP END) AS order_date,
        MAX(CASE WHEN TYPE_KEY = 10 AND ROW_ST = 'Y' THEN END_TMSTP END) AS ship_date,
        MAX(CASE WHEN TYPE_KEY = 76 AND ROW_ST = 'Y' THEN END_TMSTP END) AS due_date,
        MAX(CASE WHEN TYPE_KEY = 110 AND ROW_ST = 'Y' THEN VAL_NUM END) AS sub_total,
        MAX(CASE WHEN TYPE_KEY = 17 AND ROW_ST = 'Y' THEN VAL_NUM END) AS tax_amt,
        MAX(CASE WHEN TYPE_KEY = 126 AND ROW_ST = 'Y' THEN VAL_NUM END) AS freight
    FROM ranked_sales_order
    WHERE rnk = 1
    GROUP BY SALES_ORDER_KEY, EFF_TMSTP
),

-- ============================================================
-- SALES_ORDER: Resolve relationships (M:1)
-- ============================================================
ranked_so_customer_x AS (
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
rel_so_customer AS (
    SELECT SALES_ORDER_KEY, CUSTOMER_KEY, EFF_TMSTP
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
        r_cust.CUSTOMER_KEY AS _key__customer,
        o.EFF_TMSTP
    FROM so_attrs o
    LEFT JOIN rel_so_customer r_cust
        ON o.SALES_ORDER_KEY = r_cust.SALES_ORDER_KEY
        AND o.EFF_TMSTP = r_cust.EFF_TMSTP
),

-- ============================================================
-- SALES_ORDER: Add valid_from / valid_to
-- ============================================================
so_versioned AS (
    SELECT
        so.*,
        so.EFF_TMSTP AS valid_from,
        COALESCE(
            LEAD(so.EFF_TMSTP) OVER (
                PARTITION BY so.SALES_ORDER_KEY
                ORDER BY so.EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS valid_to
    FROM so_joined so
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
        so.due_date,
        sod.EFF_TMSTP
    FROM sod_joined sod
    LEFT JOIN so_versioned so
        ON sod._key__sales_order = so.SALES_ORDER_KEY
        AND sod.EFF_TMSTP >= so.valid_from
        AND sod.EFF_TMSTP < so.valid_to
),

-- ============================================================
-- SALES_ORDER_DETAIL: Add valid_from / valid_to
-- ============================================================
sod_versioned AS (
    SELECT
        sod.*,
        sod.EFF_TMSTP AS valid_from,
        COALESCE(
            LEAD(sod.EFF_TMSTP) OVER (
                PARTITION BY sod.SALES_ORDER_DETAIL_KEY
                ORDER BY sod.EFF_TMSTP
            ),
            '9999-12-31'::timestamp
        ) AS valid_to
    FROM sod_with_sales_order sod
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
        e.event_tmstp::time AS _key__times,
        sod.valid_from,
        sod.valid_to
    FROM sod_versioned sod
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
        e.event_tmstp::time AS _key__times,
        so.valid_from,
        so.valid_to
    FROM so_versioned so
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
    NULL::numeric AS _measure__sales_order__freight,
    sode.valid_from,
    sode.valid_to
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
    soe._measure__sales_order__freight,
    soe.valid_from,
    soe.valid_to
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
    NULL::numeric AS _measure__sales_order__freight,
    p.valid_from,
    p.valid_to
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
    NULL::numeric AS _measure__sales_order__freight,
    c.valid_from,
    c.valid_to
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
    NULL::numeric AS _measure__sales_order__freight,
    sp.valid_from,
    sp.valid_to
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
    NULL::numeric AS _measure__sales_order__freight,
    p.valid_from,
    p.valid_to
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
