# E-Commerce Medallion Architecture — Data Flow Description

## Overview

The project implements a **medallion architecture** (Bronze → Silver → Gold) on Databricks using PySpark and Unity Catalog. Raw e-commerce CSV data is ingested, cleaned, enriched, and exposed as a BI-ready denormalised view. The project spans **8 notebooks** across 4 folders.

---

## Step 0: Environment Setup (`1_setup/`)

Creates the Unity Catalog `ecommerce` and three schemas — `bronze`, `silver`, and `gold` — to hold tables at each medallion layer.

---

## Step 1: Dimension Ingestion — Bronze (`2_medallion_processing_dim/1_dim_bronze`)

Reads 5 raw dimension CSV files from `/Volumes/ecommerce/source_data/raw/` into Bronze Delta tables. Each CSV is read with an explicit `StructType` (all `StringType`) to preserve raw data as-is. Two metadata columns are added for lineage:

* `_source_file` — from Spark's built-in `_metadata.file_path`
* `_ingested_at` — current timestamp at ingestion time

| Dimension | Source CSV | Bronze Table | Rows |
| --- | --- | --- | --- |
| Brands | `raw/brands/*.csv` | `bronze.brz_brands` | 52 |
| Category | `raw/category/*.csv` | `bronze.brz_category` | 10 |
| Customers | `raw/customers/*.csv` | `bronze.brz_customers` | 300,000 |
| Date | `raw/date/*.csv` | `bronze.brz_date` | 95 |
| Products | `raw/products/*.csv` | `bronze.brz_products` | 50,000 |

---

## Step 2: Dimension Cleaning — Silver (`2_medallion_processing_dim/2_dim_silver`)

Each bronze dimension table is read and cleaned column-by-column, following the pattern: check nulls → inspect distinct values → clean/transform → check duplicates → write to Silver.

| Dimension | Bronze Table | Silver Table | Final Rows | Key Cleaning |
| --- | --- | --- | --- | --- |
| Brands | `brz_brands` | `slv_brands` | 52 | Removed `@` from brand codes, trimmed whitespace, standardized category codes (BOOKS→BKS, GROCERY→GRCY, TOYS→TOY) |
| Category | `brz_category` | `slv_category` | 8 | Uppercased category codes, removed 2 duplicate rows |
| Customers | `brz_customers` | `slv_customers` | 299,700 | Dropped 300 null customer IDs, filled 30K null phones with 'Not Available', removed `.0` suffix from phone numbers |
| Date | `brz_date` | `slv_date` | 92 | Cast `date`→date type, `year`→int, title-cased `day_name`, created readable `quarter` (Q3-2025) and `week_of_year` (Week31-2025) with numeric companion columns, removed 3 duplicates |
| Products | `brz_products` | `slv_products` | 50,000 | Filled 4,126 null colors with 'NA', stripped `g` from weight_grams→float, replaced comma decimals in length_cms→float, uppercased category/brand codes, cast width/height to float, fixed material spelling (Coton→Cotton, Alumium→Aluminum, Ruber→Rubber), applied abs to negative rating_count→int |

---

## Step 3: Dimension Enrichment — Gold (`2_medallion_processing_dim/3_dim_gold`)

Enriches 3 of the 5 silver dimension tables to create the final gold dimension tables:

| Gold Table | Source | Rows | Enrichment |
| --- | --- | --- | --- |
| `gold.gld_dim_products` | `slv_products` + `slv_brands` + `slv_category` | 50,000 | LEFT JOIN on `brand_code` and `category_code` to add `brand_name` and `category_name` |
| `gold.gld_dim_customers` | `slv_customers` + region mapping | 299,700 | Added `region` column via state-to-region mapping for 7 countries (India, Australia, UK, US, UAE, Singapore, Canada); 87,150 unmapped rows filled with 'Other' |
| `gold.gld_dim_date` | `slv_date` | 92 | Added `date_id` (YYYYMMDD integer join key), `is_weekend` flag (1/0), and `month_name` (e.g., August) |

---

## Step 4: Fact Ingestion — Bronze (`3_medallion_processing_fact/1_fact_bronze`)

Reads raw order items CSV from `/Volumes/ecommerce/source_data/raw/order_items/landing/*.csv` into Bronze with an explicit `StringType` schema (13 data columns) plus 2 metadata columns.

| Bronze Table | Rows | Columns |
| --- | --- | --- |
| `bronze.brz_order_items` | 183,378 | 15 (13 data + 2 metadata) |

**Data columns:** `dt`, `order_ts`, `customer_id`, `order_id`, `item_seq`, `product_id`, `quantity`, `unit_price_currency`, `unit_price`, `discount_pct`, `tax_amount`, `channel`, `coupon_code`

---

## Step 5: Fact Cleaning — Silver (`3_medallion_processing_fact/2_fact_silver`)

Performs a comprehensive column-by-column data quality audit on the 183,378-row fact table, then cleans each column:

| Column | Issue Found | Resolution |
| --- | --- | --- |
| `dt` | Stored as string | Cast to `date` |
| `order_ts` | Stored as string | Cast to `timestamp` |
| `item_seq` | Stored as string | Cast to `integer` |
| `quantity` | Value `'Two'` instead of `2` | Replaced `'Two'`→`'2'`, cast to `integer` |
| `unit_price` | `$` prefix on some values | Stripped `$`, cast to `float` |
| `discount_pct` | `%` suffix on all values | Stripped `%`, cast to `float` |
| `tax_amount` | Stored as string | Cast to `float` |
| `channel` | Abbreviated names (`web`, `app`) | Replaced with `Website` / `Application` |
| `coupon_code` | 119,548 nulls (65%) | Filled with `'NA'` |

**Validated with no changes:** `customer_id` (88,495 distinct, 16 chars), `order_id` (104,756 distinct, 6 chars), `product_id` (48,702 distinct, 13 chars), `unit_price_currency` (7 valid codes). No duplicates on composite key (`order_id`, `item_seq`).

| Silver Table | Rows |
| --- | --- |
| `silver.slv_order_items` | 183,378 (unchanged — no rows dropped) |

---

## Step 6: Fact Enrichment — Gold (`3_medallion_processing_fact/3_fact_gold`)

Enriches the silver fact table with 7 derived columns, renames several columns for BI readability, and writes to Gold:

**Derived columns added:**

| Column | Formula | Description |
| --- | --- | --- |
| `gross_amount` | `quantity × unit_price` | Total before discount |
| `discount_amount` | `gross_amount × discount_pct / 100` | Dollar value of discount |
| `sale_amount` | `gross_amount − discount_amount + tax_amount` | Final amount owed |
| `date_id` | `dt` with hyphens removed (YYYYMMDD) | Integer join key for date dimension |
| `coupon_flag` | `1 if coupon_code ≠ 'NA', else 0` | Binary coupon usage indicator |
| `inr_rate` | FX lookup by `unit_price_currency` | Conversion rate to INR |
| `sale_amount_inr` | `sale_amount × inr_rate` | Sale amount converted to INR |

**FX rates used (static lookup):** INR=1.00, AED=24.18, AUD=57.55, CAD=62.93, GBP=117.98, SGD=68.18, USD=88.29

**Column renames:** `dt`→`transaction_date`, `order_ts`→`transaction_ts`, `order_id`→`transaction_id`, `item_seq`→`seq_no`, `discount_pct`→`discount_percent`, `sale_amount`→`net_amount`, `sale_amount_inr`→`net_amount_inr`

| Gold Table | Rows | Validated |
| --- | --- | --- |
| `gold.gld_fact_order_items` | 183,378 | Row count matches silver — no data loss |

---

## Step 7: BI-Ready Denormalised View (`4_denormalised_view/1_create_denorm_view_reporting`)

Creates a single denormalised view that joins the gold fact table with two gold dimension tables, eliminating the need for BI consumers to perform joins:

```sql
CREATE OR REPLACE VIEW ecommerce.gold.fact_transactions_denorm AS
  SELECT
    o.*,
    d.year, d.month_name, d.day_name, d.is_weekend,
    d.quarter, d.quarter_number, d.week_of_year, d.week_number,
    p.sku, p.category_code, p.category_name,
    p.brand_code, p.brand_name, p.color, p.size, p.rating_count,
    extract(HOUR FROM transaction_ts) AS hour_of_day
  FROM ecommerce.gold.gld_fact_order_items o
  LEFT JOIN ecommerce.gold.gld_dim_date     d ON o.date_id    = d.date_id
  LEFT JOIN ecommerce.gold.gld_dim_products p ON p.product_id = o.product_id
```

| View | Rows | Joins |
| --- | --- | --- |
| `gold.fact_transactions_denorm` | 183,378 | Fact → Date dim (on `date_id`), Fact → Product dim (on `product_id`) |

Also derives `hour_of_day` from `transaction_ts` for time-of-day analysis.

---

## Complete Data Flow Diagram

```
                        /Volumes/ecommerce/source_data/raw/
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                                   │
              5 Dim CSVs                         1 Fact CSV
                    │                                   │
         ┌──── 1_dim_bronze ──┐               ┌── 1_fact_bronze ──┐
         │   (explicit schema, │               │  (explicit schema, │
         │    StringType,      │               │   StringType,     │
         │    metadata cols)   │               │   metadata cols)  │
         └────────┬───────────┘               └────────┬─────────┘
                  │                                     │
         ┌──── 2_dim_silver ──┐               ┌── 2_fact_silver ──┐
         │  (null checks,     │               │ (column-by-column  │
         │   type casting,    │               │  audit: type cast, │
         │   dedup, standard- │               │  fix formats,      │
         │   ize values)      │               │  handle nulls)     │
         └────────┬───────────┘               └────────┬─────────┘
                  │                                     │
         ┌──── 3_dim_gold ────┐               ┌── 3_fact_gold ────┐
         │  Products: +brand  │               │ +gross_amount     │
         │    +category names  │               │ +discount_amount  │
         │  Customers: +region│               │ +sale_amount      │
         │  Date: +date_id,   │               │ +date_id          │
         │    +is_weekend,    │               │ +coupon_flag      │
         │    +month_name    │               │ +inr_rate (FX)    │
         └────────┬───────────┘               │ +sale_amount_inr  │
                  │                           │ +column renames   │
                  │                           └────────┬─────────┘
                  │                                     │
                  │              ┌───────────────────────┘
                  │              │
                  │     1_create_denorm_view_reporting
                  │     (LEFT JOIN fact + date dim + product dim)
                  │              │
                  └──────────► gold.fact_transactions_denorm (VIEW)
                                 183,378 rows
                              Ready for BI Dashboards
```

---

## Final Gold Layer Tables

| Table/View | Type | Rows | Description |
| --- | --- | --- | --- |
| `gold.gld_dim_products` | Table | 50,000 | Products enriched with brand & category names |
| `gold.gld_dim_customers` | Table | 299,700 | Customers enriched with region |
| `gold.gld_dim_date` | Table | 92 | Dates with `date_id`, `is_weekend`, `month_name` |
| `gold.gld_fact_order_items` | Table | 183,378 | Enriched order items with monetary calcs & FX conversion |
| `gold.fact_transactions_denorm` | View | 183,378 | Denormalised fact + date + product dims for BI |

## Join Keys Between Gold Tables

| Fact Column | Dimension Table | Dimension Column |
| --- | --- | --- |
| `date_id` | `gld_dim_date` | `date_id` |
| `product_id` | `gld_dim_products` | `product_id` |
| `customer_id` | `gld_dim_customers` | `customer_id` |