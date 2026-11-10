# Bronze Layer Validation Procedure

Olist analytics project | Dataset: olist\_bronze | Draft v1

## 1. Purpose

Confirm that the nine bronze tables loaded completely and correctly before any team builds on them. Run the checks once after loading, record the results in section 9, and rerun them whenever bronze is reloaded.

Expected values come from the standard Olist dataset. A mismatch is a prompt to investigate against the source CSV, not proof of an error. Run all queries in the BigQuery console.

## 2. Check 1: row counts

Every table should contain all rows of its source file.

```sql
SELECT table_id, row_count
FROM `olist_bronze.__TABLES__`
ORDER BY table_id;
```

| Table                                | Expected rows |
| ------------------------------------ | ------------- |
| customers                            | 99,441        |
| geolocation                          | 1,000,163     |
| order\_items                         | 112,650       |
| order\_payments                      | 103,886       |
| order\_reviews                       | 99,224        |
| orders                               | 99,441        |
| product\_category\_name\_translation | 71            |
| products                             | 32,951        |
| sellers                              | 3,095         |

If order\_reviews has noticeably fewer rows, the quoted line breaks in review comments were probably not handled. Reload with Quoted newlines ticked.

## 3. Check 2: column types

```sql
SELECT table_name, column_name, data_type
FROM `olist_bronze.INFORMATION_SCHEMA.COLUMNS`
ORDER BY table_name, ordinal_position;
```

Expected: all date and time columns are TIMESTAMP, prices and coordinates are FLOAT64, counts and review scores are INT64, and every zip code prefix is STRING. The console schema editor accepts the names INTEGER and FLOAT, which are stored as INT64 and FLOAT64.

## 4. Check 3: key uniqueness

```sql
SELECT 'orders.order_id' AS check_name, COUNT(*) - COUNT(DISTINCT order_id) AS dups FROM `olist_bronze.orders`
UNION ALL
SELECT 'customers.customer_id', COUNT(*) - COUNT(DISTINCT customer_id) FROM `olist_bronze.customers`
UNION ALL
SELECT 'products.product_id', COUNT(*) - COUNT(DISTINCT product_id) FROM `olist_bronze.products`
UNION ALL
SELECT 'sellers.seller_id', COUNT(*) - COUNT(DISTINCT seller_id) FROM `olist_bronze.sellers`
UNION ALL
SELECT 'order_items (order_id, order_item_id)', COUNT(*) - COUNT(DISTINCT CONCAT(order_id, '-', order_item_id)) FROM `olist_bronze.order_items`
UNION ALL
SELECT 'order_payments (order_id, payment_sequential)', COUNT(*) - COUNT(DISTINCT CONCAT(order_id, '-', payment_sequential)) FROM `olist_bronze.order_payments`
UNION ALL
SELECT 'order_reviews.review_id', COUNT(*) - COUNT(DISTINCT review_id) FROM `olist_bronze.order_reviews`;
```

Expected: 0 for the first six rows. A small number of duplicate review\_id values is a known characteristic of the source data, so record it and handle it in the gold layer instead of treating it as a load error. Confirm against the source file.

## 5. Check 4: relationships between tables

Look for child records whose parent row is missing.

```sql
SELECT 'items without order' AS check_name, COUNT(*) AS n
FROM `olist_bronze.order_items` i
WHERE NOT EXISTS (SELECT 1 FROM `olist_bronze.orders` o WHERE o.order_id = i.order_id)
UNION ALL
SELECT 'items without product', COUNT(*)
FROM `olist_bronze.order_items` i
WHERE NOT EXISTS (SELECT 1 FROM `olist_bronze.products` p WHERE p.product_id = i.product_id)
UNION ALL
SELECT 'items without seller', COUNT(*)
FROM `olist_bronze.order_items` i
WHERE NOT EXISTS (SELECT 1 FROM `olist_bronze.sellers` s WHERE s.seller_id = i.seller_id)
UNION ALL
SELECT 'orders without customer', COUNT(*)
FROM `olist_bronze.orders` o
WHERE NOT EXISTS (SELECT 1 FROM `olist_bronze.customers` c WHERE c.customer_id = o.customer_id)
UNION ALL
SELECT 'payments without order', COUNT(*)
FROM `olist_bronze.order_payments` p
WHERE NOT EXISTS (SELECT 1 FROM `olist_bronze.orders` o WHERE o.order_id = p.order_id)
UNION ALL
SELECT 'reviews without order', COUNT(*)
FROM `olist_bronze.order_reviews` r
WHERE NOT EXISTS (SELECT 1 FROM `olist_bronze.orders` o WHERE o.order_id = r.order_id)
UNION ALL
SELECT 'orders without items', COUNT(*)
FROM `olist_bronze.orders` o
WHERE NOT EXISTS (SELECT 1 FROM `olist_bronze.order_items` i WHERE i.order_id = o.order_id);
```

Expected: 0 for the first six rows. A non-zero value for orders without items is normal. These are mostly cancelled or unavailable orders, typically a few hundred.

## 6. Check 5: value sanity

```sql
SELECT
  (SELECT COUNTIF(order_delivered_customer_date < order_purchase_timestamp) FROM `olist_bronze.orders`) AS delivered_before_purchase,
  (SELECT COUNTIF(price <= 0 OR freight_value < 0) FROM `olist_bronze.order_items`) AS bad_item_amounts,
  (SELECT COUNTIF(review_score NOT BETWEEN 1 AND 5) FROM `olist_bronze.order_reviews`) AS bad_scores,
  (SELECT COUNTIF(geolocation_lat NOT BETWEEN -35 AND 6 OR geolocation_lng NOT BETWEEN -75 AND -30) FROM `olist_bronze.geolocation`) AS coords_outside_brazil;

SELECT
  (SELECT COUNTIF(LENGTH(customer_zip_code_prefix) < 5) FROM `olist_bronze.customers`) AS short_zip,
  (SELECT COUNTIF(LENGTH(review_id) != 32) FROM `olist_bronze.order_reviews`) AS bad_review_ids,
  (SELECT COUNTIF(LENGTH(order_id) != 32) FROM `olist_bronze.orders`) AS bad_order_ids;
```

Expected: 0 for all values. A short zip code means leading zeros were lost. Wrong ID lengths usually mean a quoting or line-break problem shifted the columns. A small number of coordinates outside Brazil is a known source issue and is filtered in the gold layer.

Also confirm the date range: the earliest order\_purchase\_timestamp should be in 2016 and the latest in 2018.

## 7. Check 6: null profile (information only)

```sql
SELECT
  COUNTIF(product_category_name IS NULL) AS no_category,
  COUNTIF(product_weight_g IS NULL) AS no_weight
FROM `olist_bronze.products`;

SELECT
  COUNTIF(order_approved_at IS NULL) AS no_approval,
  COUNTIF(order_delivered_carrier_date IS NULL) AS no_carrier_date,
  COUNTIF(order_delivered_customer_date IS NULL) AS no_customer_date
FROM `olist_bronze.orders`;
```

Some nulls exist in the source data, for example undelivered orders have no delivery date. Empty fields in CSV files load as NULL by default. Record the counts so every team handles nulls the same way in the gold layer.

## 8. Interpreting findings

| Finding                                                                                      | Likely cause                                              | Action                                         |
| -------------------------------------------------------------------------------------------- | --------------------------------------------------------- | ---------------------------------------------- |
| Row count lower than expected                                                                | Header row count wrong, or quoted line breaks not handled | Reload the table with the correct load options |
| Wrong column type                                                                            | Schema entered incorrectly                                | Reload with the explicit schema                |
| Duplicate keys in a table that should be unique                                              | Duplicated load or duplicated source rows                 | Check the source file, then reload or document |
| Orphan child records                                                                         | Partial load of a parent table                            | Reload the parent table and rerun the check    |
| Duplicate review\_id, orders without items, <br> coordinates outside Brazil, null categories | Characteristics of the source data                        | Document and handle in the gold layer          |

Bronze stays read-only. Fixes happen by reloading from the raw CSVs, never by editing bronze tables by hand.

## 9. Results log

| Check             | Date | Run by | Result          | Notes                                                                                                                                                                                                                                                                |
| ----------------- | ---- | ------ | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Row counts     |      |        | Pass.           |                                                                                                                                                                                                                                                                      |
| 2. Column types   |      |        | Pass.           |                                                                                                                                                                                                                                                                      |
| 3. Key uniqueness |      |        | Pass with note. | order_reviews.review_id has 814 duplicates, but (review_id, order_id) is unique (0 duplicates). <br> 547 orders have more than one review. <br> Aggregate by order in gold.                                                                                          |
| 4. Relationships  |      |        | Pass.           |                                                                                                                                                                                                                                                                      |
| 5. Value sanity   |      |        | Pass with note. | 29 geolocation rows have coordinates outside Brazil (Europe, Mexico/Texas, <br> Philippines, Canary Islands, Argentina) despite Brazilian city and state values. <br> Source data error. Filter with the Brazil bounding box in gold before averaging by zip prefix. |
| 6. Null profile   |      |        |                 |                                                                                                                                                                                                                                                                      |
