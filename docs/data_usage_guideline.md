# Olist E-commerce Dataset: Data Usage Guidelines

This document outlines the known data anomalies, structural quirks, and recommended handling procedures for the entire Olist E-commerce dataset in the Bronze layer (`olist_bronze`). Analysts, Analytics Engineers, and Data Scientists must review these guidelines before building downstream models (Silver/Gold layers) or calculating KPIs directly from the raw data.

## 1. Customer Identification (`olist_bronze.customers`)

**Issue Description & Business Logic:**
The `customers` table contains two distinct ID columns that are frequently confused by new analysts: `customer_id` and `customer_unique_id`.

* `customer_id`: This is a **transactional key**. Every time a user places an order, a *new* `customer_id` is generated. It is a 1:1 mapping with `order_id`.
* `customer_unique_id`: This is the **actual user key** (representing a unique person/account).

**Downstream Handling Recommendations:**

* **Table Joins:** Always use `customer_id` when joining the `customers` table to the `orders` table.
* **User Metrics:** When calculating Retention Rate, Lifetime Value (LTV), or Daily Active Users (DAU), you **MUST** `COUNT(DISTINCT customer_unique_id)`. Using `customer_id` will completely mask repeat purchases, making it look like every order is from a new customer.

## 2. Order Payments (`olist_bronze.order_payments`)

**Issue Description & Business Logic:**
A single `order_id` can have multiple rows in the payments table. This happens when a user splits the payment (e.g., using a voucher + a credit card) or pays in multiple sequential methods.

**Downstream Handling Recommendations:**

* **Calculating Order Revenue:** Never assume one row equals one order's revenue. To get the total revenue of an order, you must aggregate: `SUM(payment_value) GROUP BY order_id`.
* **Installments vs. Sequential:** `payment_installments` represents credit card installments (e.g., paying in 3 months). It does *not* multiply the `payment_value`. The `payment_value` is already the total amount the customer paid. `payment_sequential` simply orders the payment methods used in a single checkout.

## 3. Order Items & Freight Fan-Out (`olist_bronze.order_items`)

**Issue Description & Business Logic:**
The `order_items` table is at the **Item Level**, not the Order Level.

* If a customer buys 3 identical mugs in one order, there will be 3 separate rows with `order_item_id` = 1, 2, and 3.
* The `price` and `freight_value` in this table are strictly **per item**, not per order.

**Downstream Handling Recommendations:**

* **Avoid Fan-Out Errors:** If you join `orders` directly to `order_items` and `SUM(freight_value)`, it is correct. But if you join `order_payments` (which is at the order/payment level) to `order_items`, you will create a Cartesian product and artificially inflate revenue. Always aggregate `order_items` to the `order_id` grain *before* joining with payments.

## 4. Product Catalog & Translations (`olist_bronze.products` & `olist_bronze.category_name_translation`)

**Issue Description:**

* **Missing Attributes:** ~610 products are missing `product_category_name` and 2 are missing `product_weight_g`.
* **Missing Translations:** The translation table does NOT contain translations for every single Portuguese category in the products table (e.g., `portateis_cozinha_e_preparadores_de_alimentos` and `pc_gamer` are missing).

**Downstream Handling Recommendations:**

* **Translation Joins:** Always use a `LEFT JOIN` from `products` to the `translation` table. If you use an `INNER JOIN`, you will silently drop products that lack an English translation.
* **Handling Nulls:** When building Silver models, use `COALESCE(translation.product_category_name_english, products.product_category_name, 'Unknown')` to fallback to the Portuguese name or 'Unknown' if the translation is missing.

## 5. Order Lifecycle Timestamps (`olist_bronze.orders`)

**Issue Description:**
The `orders` table contains a significant volume of `NULL` values in timestamp columns (`order_approved_at`, `order_delivered_carrier_date`, `order_delivered_customer_date`).

**Root Cause:**
These `NULL` values **do not represent missing data**. They represent the natural business lifecycle of an e-commerce order (e.g., abandoned checkouts, canceled orders, orders currently in transit).

**Downstream Handling Recommendations:**

* **Do not impute or fill these dates.**
* **SLA & Time-Diff Metrics:** When calculating average fulfillment duration or shipping SLA compliance, you **must** apply `WHERE order_status = 'delivered'` before calculating timestamp differences to exclude unfinished orders.

## 6. Reviews Deduplication (`olist_bronze.order_reviews`)

**Issue Description:**
A single `order_id` can occasionally have multiple reviews (multiple rows). This typically occurs if a customer submits a review, is contacted by customer service, and then submits a second revised review.

**Downstream Handling Recommendations:**

* **Deduplication:** Before joining `order_reviews` to your main sales view, deduplicate the table. Group by `order_id` and keep the row with the most recent `review_creation_date` or `review_answer_timestamp` (e.g., using `QUALIFY ROW_NUMBER() OVER(PARTITION BY order_id ORDER BY review_answer_timestamp DESC) = 1`).

## 7. Geographic Data Anomalies (`olist_bronze.geolocation`)

**Issue Description:**
The raw geographic dataset contains 29 coordinate records (`geolocation_lat`, `geolocation_lng`) that fall completely outside Brazilian national borders (e.g., mapping to Europe or North America). Furthermore, a single `geolocation_zip_code_prefix` has multiple varying lat/lng points.

**Downstream Handling Recommendations:**

* **Spatial Filtering:** When performing geospatial mapping, filter out invalid coordinates using spatial boundaries or by excluding known bad points during the Silver layer transformation.
* **Joining Geo-Data:** **Never** join customer or seller tables directly to the raw geolocation table on `zip_code_prefix` — it will cause a massive fan-out since the relationship is 1-to-many. You must aggregate the geolocation table first (e.g., `SELECT zip_code_prefix, AVG(lat), AVG(lng) GROUP BY zip_code_prefix`) before joining.