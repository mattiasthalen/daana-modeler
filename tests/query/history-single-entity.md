# Test: History Single Entity — Product List Prices Over Time

Read @bootstrap-context.md and @connection-context.md before proceeding.

## Skill

`/daana-query`

## Inputs

- **Question:** "Show me the history of product list prices"
- **Time dimension:** History
- **Cutoff date:** Current (no cutoff)

## Expected Output

```sql
WITH twine AS (
  SELECT product_key, type_key, eff_tmstp, ver_tmstp, row_st,
         val_num,
         'product_product_list_price' AS timeline
  FROM daana_dw.product_desc
  WHERE type_key = 50 /* product_product_list_price */
)

, in_effect AS (
  SELECT
    product_key,
    type_key,
    eff_tmstp,
    ver_tmstp,
    row_st,
    val_num,
    MAX(CASE WHEN timeline = 'product_product_list_price' THEN eff_tmstp END)
      OVER (PARTITION BY product_key ORDER BY eff_tmstp
            RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      AS eff_tmstp_product_product_list_price,
    RANK() OVER (
      PARTITION BY product_key, eff_tmstp
      ORDER BY eff_tmstp DESC
    ) AS rn
  FROM twine
)

, filtered_in_effect AS (
  SELECT * FROM in_effect WHERE rn = 1
)

, cte_product_product_list_price AS (
  SELECT
    product_key,
    eff_tmstp,
    CASE WHEN row_st = 'Y' THEN val_num ELSE NULL END AS product_list_price
  FROM filtered_in_effect
  WHERE type_key = 50 /* product_product_list_price */
)

SELECT DISTINCT
  filtered_in_effect.product_key,
  filtered_in_effect.eff_tmstp,
  cte_product_product_list_price.product_list_price
FROM filtered_in_effect
LEFT JOIN cte_product_product_list_price
  ON filtered_in_effect.product_key = cte_product_product_list_price.product_key
  AND filtered_in_effect.eff_tmstp_product_product_list_price = cte_product_product_list_price.eff_tmstp
```
