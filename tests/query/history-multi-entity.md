# Test: History Multi-Entity — Sales Order Details with Product Names

Read @bootstrap-context.md and @connection-context.md before proceeding.

## Skill

`/daana-query`

## Inputs

- **Question:** "Show me sales order details with their product names"
- **Time dimension:** History
- **Cutoff date:** Current (no cutoff)

## Expected Output

```sql
WITH twine AS (
  -- Anchor descriptor: order_qty
  SELECT sales_order_detail_key, type_key, eff_tmstp, ver_tmstp, row_st,
         val_num, CAST(NULL AS VARCHAR) AS product_key,
         'sales_order_detail_sales_order_detail_order_qty' AS timeline
  FROM daana_dw.sales_order_detail_desc
  WHERE type_key = 60 /* sales_order_detail_sales_order_detail_order_qty */
  UNION ALL
  -- Anchor descriptor: unit_price
  SELECT sales_order_detail_key, type_key, eff_tmstp, ver_tmstp, row_st,
         val_num, NULL,
         'sales_order_detail_sales_order_detail_unit_price' AS timeline
  FROM daana_dw.sales_order_detail_desc
  WHERE type_key = 129 /* sales_order_detail_sales_order_detail_unit_price */
  UNION ALL
  -- Anchor descriptor: unit_price_discount
  SELECT sales_order_detail_key, type_key, eff_tmstp, ver_tmstp, row_st,
         val_num, NULL,
         'sales_order_detail_sales_order_detail_unit_price_discount' AS timeline
  FROM daana_dw.sales_order_detail_desc
  WHERE type_key = 36 /* sales_order_detail_sales_order_detail_unit_price_discount */
  UNION ALL
  -- Relationship: sales_order_detail -> product
  SELECT sales_order_detail_key, type_key, eff_tmstp, ver_tmstp, row_st,
         CAST(NULL AS NUMERIC) AS val_num, product_key,
         'sales_order_detail_sales_order_detail_refers_to_product_product' AS timeline
  FROM daana_dw.sales_order_detail_product_x
  WHERE type_key = 20 /* sales_order_detail_sales_order_detail_refers_to_product_product */
)

, in_effect AS (
  SELECT
    sales_order_detail_key, type_key, eff_tmstp, ver_tmstp, row_st,
    val_num, product_key,
    MAX(CASE WHEN timeline = 'sales_order_detail_sales_order_detail_order_qty' THEN eff_tmstp END)
      OVER (PARTITION BY sales_order_detail_key ORDER BY eff_tmstp
            RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      AS eff_tmstp_sales_order_detail_sales_order_detail_order_qty,
    MAX(CASE WHEN timeline = 'sales_order_detail_sales_order_detail_unit_price' THEN eff_tmstp END)
      OVER (PARTITION BY sales_order_detail_key ORDER BY eff_tmstp
            RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      AS eff_tmstp_sales_order_detail_sales_order_detail_unit_price,
    MAX(CASE WHEN timeline = 'sales_order_detail_sales_order_detail_unit_price_discount' THEN eff_tmstp END)
      OVER (PARTITION BY sales_order_detail_key ORDER BY eff_tmstp
            RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      AS eff_tmstp_sales_order_detail_sales_order_detail_unit_price_discount,
    MAX(CASE WHEN timeline = 'sales_order_detail_sales_order_detail_refers_to_product_product' THEN eff_tmstp END)
      OVER (PARTITION BY sales_order_detail_key ORDER BY eff_tmstp
            RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      AS eff_tmstp_sales_order_detail_sales_order_detail_refers_to_product_product,
    RANK() OVER (
      PARTITION BY sales_order_detail_key, eff_tmstp
      ORDER BY eff_tmstp DESC
    ) AS rn
  FROM twine
)

, filtered_in_effect AS (
  SELECT * FROM in_effect WHERE rn = 1
)

-- Anchor descriptor CTEs
, cte_sales_order_detail_sales_order_detail_order_qty AS (
  SELECT sales_order_detail_key, eff_tmstp,
    CASE WHEN row_st = 'Y' THEN val_num ELSE NULL END AS sales_order_detail_order_qty
  FROM filtered_in_effect
  WHERE type_key = 60 /* sales_order_detail_sales_order_detail_order_qty */
)

, cte_sales_order_detail_sales_order_detail_unit_price AS (
  SELECT sales_order_detail_key, eff_tmstp,
    CASE WHEN row_st = 'Y' THEN val_num ELSE NULL END AS sales_order_detail_unit_price
  FROM filtered_in_effect
  WHERE type_key = 129 /* sales_order_detail_sales_order_detail_unit_price */
)

, cte_sales_order_detail_sales_order_detail_unit_price_discount AS (
  SELECT sales_order_detail_key, eff_tmstp,
    CASE WHEN row_st = 'Y' THEN val_num ELSE NULL END AS sales_order_detail_unit_price_discount
  FROM filtered_in_effect
  WHERE type_key = 36 /* sales_order_detail_sales_order_detail_unit_price_discount */
)

-- Relationship CTE
, cte_sales_order_detail_sales_order_detail_refers_to_product_product AS (
  SELECT sales_order_detail_key, eff_tmstp,
    CASE WHEN row_st = 'Y' THEN product_key ELSE NULL END AS product_key
  FROM filtered_in_effect
  WHERE type_key = 20 /* sales_order_detail_sales_order_detail_refers_to_product_product */
)

-- Related entity: product history
, related_twine AS (
  SELECT product_key, type_key, eff_tmstp, ver_tmstp, row_st,
         val_str,
         'product_product_name' AS timeline
  FROM daana_dw.product_desc
  WHERE type_key = 22 /* product_product_name */
)

, related_in_effect AS (
  SELECT
    product_key, type_key, eff_tmstp, ver_tmstp, row_st, val_str,
    RANK() OVER (PARTITION BY product_key, eff_tmstp ORDER BY eff_tmstp DESC) AS rn
  FROM related_twine
)

, related_filtered AS (
  SELECT * FROM related_in_effect WHERE rn = 1
)

, cte_product_product_name AS (
  SELECT product_key, eff_tmstp,
    CASE WHEN row_st = 'Y' THEN val_str ELSE NULL END AS product_name
  FROM related_filtered
  WHERE type_key = 22 /* product_product_name */
)

-- Final join
SELECT DISTINCT
  fie.sales_order_detail_key,
  fie.eff_tmstp,
  cte_oq.sales_order_detail_order_qty,
  cte_up.sales_order_detail_unit_price,
  cte_upd.sales_order_detail_unit_price_discount,
  cte_rel.product_key,
  related_pn.product_name
FROM filtered_in_effect fie
LEFT JOIN cte_sales_order_detail_sales_order_detail_order_qty cte_oq
  ON fie.sales_order_detail_key = cte_oq.sales_order_detail_key
  AND fie.eff_tmstp_sales_order_detail_sales_order_detail_order_qty = cte_oq.eff_tmstp
LEFT JOIN cte_sales_order_detail_sales_order_detail_unit_price cte_up
  ON fie.sales_order_detail_key = cte_up.sales_order_detail_key
  AND fie.eff_tmstp_sales_order_detail_sales_order_detail_unit_price = cte_up.eff_tmstp
LEFT JOIN cte_sales_order_detail_sales_order_detail_unit_price_discount cte_upd
  ON fie.sales_order_detail_key = cte_upd.sales_order_detail_key
  AND fie.eff_tmstp_sales_order_detail_sales_order_detail_unit_price_discount = cte_upd.eff_tmstp
LEFT JOIN cte_sales_order_detail_sales_order_detail_refers_to_product_product cte_rel
  ON fie.sales_order_detail_key = cte_rel.sales_order_detail_key
  AND fie.eff_tmstp_sales_order_detail_sales_order_detail_refers_to_product_product = cte_rel.eff_tmstp
LEFT JOIN LATERAL (
  SELECT product_name
  FROM cte_product_product_name
  WHERE product_key = cte_rel.product_key
    AND eff_tmstp <= fie.eff_tmstp
  ORDER BY eff_tmstp DESC
  LIMIT 1
) related_pn ON TRUE
```
