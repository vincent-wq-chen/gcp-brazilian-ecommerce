# Bronze Layer Load Procedure

Olist analytics project | Target dataset: olist\_bronze | Draft v1

## 1. Purpose

This document records how the nine raw Olist CSV files are loaded into the shared bronze dataset in BigQuery, so that anyone can reproduce or rebuild it. The raw files in Cloud Storage are the bronze layer and are never modified. Bronze is loaded once, validated (see the Bronze Layer Validation Procedure) and then treated as read-only.

## 2. Source and target

| Item                     | Value                               |
| ------------------------ | ----------------------------------- |
| Source bucket and folder | gs://olist-raw-data-2026/raw\_csv/  |
| Target project           | project-e6cc16fd-3698-47ad-880      |
| Target dataset           | olist\_bronze                       |
| Dataset location         | US (must match the bucket location) |
| Table type               | Native tables                       |

File-to-table mapping (confirm the exact file names in the bucket before running):

| Source file                              | Target table                         |
| ---------------------------------------- | ------------------------------------ |
| olist\_customers\_dataset.csv            | customers                            |
| olist\_geolocation\_dataset.csv          | geolocation                          |
| olist\_order\_items\_dataset.csv         | order\_items                         |
| olist\_order\_payments\_dataset.csv      | order\_payments                      |
| olist\_order\_reviews\_dataset.csv       | order\_reviews                       |
| olist\_orders\_dataset.csv               | orders                               |
| olist\_products\_dataset.csv             | products                             |
| olist\_sellers\_dataset.csv              | sellers                              |
| product\_category\_name\_translation.csv | product\_category\_name\_translation |

## 3. Load settings

These settings were used for every table, whether loaded in the console or with the script in section 5.

| Setting             | Value                                          | Reason                                                     |
| ------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| Table type          | Native                                         | Faster queries, stable copy, supports streaming later      |
| Schema              | Explicit, Auto detect off                      | Types must not depend on guessing                          |
| Header rows to skip | 1                                              | Each file has a header row                                 |
| Errors allowed      | 0                                              | A bad row stops the load instead of being dropped silently |
| Write preference    | Write if empty (console) or overwrite (script) | First load vs rebuild                                      |
| Null handling       | Default (empty field becomes NULL)             | The files use blank fields for missing values              |

**Extra option for order_reviews only:** when loading this table in the console, also tick Quoted newlines under Advanced options. The script equivalent is `allow_quoted_newlines = true`. Without it the load can fail with parsing errors or lose rows, because the comment text has embedded line breaks. Every other table uses the settings above unchanged.

## 4. Special handling

- **Zip code prefixes are STRING.** Auto detect would read them as integers and drop leading zeros, which breaks joins.
- **order\_reviews needs quoted newlines enabled.** Review comments contain line breaks inside quoted text.
- **order_reviews needs quoted newlines enabled.** Review comments contain line breaks inside quoted text. In the console, tick Quoted newlines under Advanced options for this table; in the script, use `allow_quoted_newlines = true`.
- **products keeps the original misspelled column names** (product\_name\_lenght, product\_description\_lenght). Rename them in the gold layer, not in bronze.
- **Type names differ by interface.** The console schema text editor uses INTEGER and FLOAT. SQL uses INT64 and FLOAT64. Both give the same stored types.
- **No constraints are enforced.** Keys and relationships are checked by the validation queries, not by BigQuery.
- **Bronze is not edited by hand.** Fixes are made by reloading from the raw CSV files.

## 5. Load script

Save each statement as its own file in the repository folder data\_ingestion/ (for example data\_ingestion/01\_customers.sql). Run them in the BigQuery console query editor. OVERWRITE replaces the table if it already exists, so the same script works for the first load and for a rebuild. Check the option names against the current BigQuery LOAD DATA documentation if a statement is rejected.

**customers**

```sql
LOAD DATA OVERWRITE olist_bronze.customers (
  customer_id STRING,
  customer_unique_id STRING,
  customer_zip_code_prefix STRING,
  customer_city STRING,
  customer_state STRING
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://olist-raw-data-2026/raw_csv/olist_customers_dataset.csv'],
  skip_leading_rows = 1
);
```

**geolocation**

```sql
LOAD DATA OVERWRITE olist_bronze.geolocation (
  geolocation_zip_code_prefix STRING,
  geolocation_lat FLOAT64,
  geolocation_lng FLOAT64,
  geolocation_city STRING,
  geolocation_state STRING
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://olist-raw-data-2026/raw_csv/olist_geolocation_dataset.csv'],
  skip_leading_rows = 1
);
```

**order\_items**

```sql
LOAD DATA OVERWRITE olist_bronze.order_items (
  order_id STRING,
  order_item_id INT64,
  product_id STRING,
  seller_id STRING,
  shipping_limit_date TIMESTAMP,
  price FLOAT64,
  freight_value FLOAT64
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://olist-raw-data-2026/raw_csv/olist_order_items_dataset.csv'],
  skip_leading_rows = 1
);
```

**order\_payments**

```sql
LOAD DATA OVERWRITE olist_bronze.order_payments (
  order_id STRING,
  payment_sequential INT64,
  payment_type STRING,
  payment_installments INT64,
  payment_value FLOAT64
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://olist-raw-data-2026/raw_csv/olist_order_payments_dataset.csv'],
  skip_leading_rows = 1
);
```

**order\_reviews** (quoted newlines enabled)

```sql
LOAD DATA OVERWRITE olist_bronze.order_reviews (
  review_id STRING,
  order_id STRING,
  review_score INT64,
  review_comment_title STRING,
  review_comment_message STRING,
  review_creation_date TIMESTAMP,
  review_answer_timestamp TIMESTAMP
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://olist-raw-data-2026/raw_csv/olist_order_reviews_dataset.csv'],
  skip_leading_rows = 1,
  allow_quoted_newlines = true
);
```

**orders**

```sql
LOAD DATA OVERWRITE olist_bronze.orders (
  order_id STRING,
  customer_id STRING,
  order_status STRING,
  order_purchase_timestamp TIMESTAMP,
  order_approved_at TIMESTAMP,
  order_delivered_carrier_date TIMESTAMP,
  order_delivered_customer_date TIMESTAMP,
  order_estimated_delivery_date TIMESTAMP
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://olist-raw-data-2026/raw_csv/olist_orders_dataset.csv'],
  skip_leading_rows = 1
);
```

**products**

```sql
LOAD DATA OVERWRITE olist_bronze.products (
  product_id STRING,
  product_category_name STRING,
  product_name_lenght INT64,
  product_description_lenght INT64,
  product_photos_qty INT64,
  product_weight_g INT64,
  product_length_cm INT64,
  product_height_cm INT64,
  product_width_cm INT64
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://olist-raw-data-2026/raw_csv/olist_products_dataset.csv'],
  skip_leading_rows = 1
);
```

**sellers**

```sql
LOAD DATA OVERWRITE olist_bronze.sellers (
  seller_id STRING,
  seller_zip_code_prefix STRING,
  seller_city STRING,
  seller_state STRING
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://olist-raw-data-2026/raw_csv/olist_sellers_dataset.csv'],
  skip_leading_rows = 1
);
```

**product\_category\_name\_translation**

```sql
LOAD DATA OVERWRITE olist_bronze.product_category_name_translation (
  product_category_name STRING,
  product_category_name_english STRING
)
FROM FILES (
  format = 'CSV',
  uris = ['gs://olist-raw-data-2026/raw_csv/product_category_name_translation.csv'],
  skip_leading_rows = 1
);
```

## 6. How to run and rebuild

1. Confirm the target dataset exists in the same location as the bucket (US).
2. Confirm your account can read the bucket and create tables in olist\_bronze.
3. Run the statements from data\_ingestion/ one at a time in the BigQuery query editor.
4. If a load fails, read the error message. It names the row and column. The most common causes are a wrong header row count, a type that does not fit the data, or quoted line breaks in order\_reviews.
5. Never fix data by editing bronze tables. Correct the schema or options and rerun the statement.
6. After all nine loads, run the validation procedure and record the results.

## 7. Expected row counts

These come from the standard Olist dataset. Confirm against the source files if a number differs.

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

## 8. Load log

| Table                                | Loaded on | Loaded by | Method (console or script) | Row count matches |
| ------------------------------------ | --------- | --------- | -------------------------- | ----------------- |
| customers                            |           |           | console                    | yes               |
| geolocation                          |           |           | console                    | yes               |
| order\_items                         |           |           | console                    | yes               |
| order\_payments                      |           |           | console                    | yes               |
| order\_reviews                       |           |           | console                    | yes               |
| orders                               |           |           | console                    | yes               |
| products                             |           |           | console                    | yes               |
| sellers                              |           |           | console                    | yes               |
| product\_category\_name\_translation |           |           | console                    | yes               |
