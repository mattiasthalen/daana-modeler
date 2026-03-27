# Test: Latest Snapshot — Product Names and List Prices

Read @bootstrap-context.md and @connection-context.md before proceeding.

## Skill

`/daana-query`

## Inputs

- **Question:** "Show me all products with their names and list prices"
- **Time dimension:** Latest
- **Cutoff date:** Current (no cutoff)

## Expected Output

```sql
SELECT
  product_key,
  MAX(CASE WHEN type_key = 22 /* product_product_name */ THEN val_str END) AS product_name,
  MAX(CASE WHEN type_key = 50 /* product_product_list_price */ THEN val_num END) AS product_list_price
FROM (
  SELECT
    product_key,
    type_key,
    row_st,
    RANK() OVER (
      PARTITION BY product_key, type_key
      ORDER BY eff_tmstp DESC, ver_tmstp DESC
    ) AS nbr,
    val_str,
    val_num
  FROM daana_dw.product_desc
  WHERE type_key IN (22, 50)
) a
WHERE nbr = 1 AND row_st = 'Y'
GROUP BY product_key
```
