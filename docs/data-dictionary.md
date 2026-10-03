# Data Dictionary — E-Commerce Medallion Data Platform

This document describes every table and column across the Bronze, Silver, and Gold layers of the `ecommerce` Unity Catalog.

**Convention:** All Bronze columns are `StringType`. Type casting happens in Silver. Enrichment and renaming happen in Gold. Two metadata columns are present in every table: `_source_file` (string) and `_ingested_at` (timestamp).

---

## Bronze Layer

### `ecommerce.bronze.brz_brands` (52 rows)

| Column | Type | Description |
| --- | --- | --- |
| `brand_code` | string | Short brand identifier (e.g., ACME, NOVW) — may contain non-alphanumeric chars like `@` |
| `brand_name` | string | Full brand name (e.g., AcmeTech, NovaWave) — may have trailing whitespace |
| `Category_code` | string | Foreign key to category dimension (e.g., CE, APP) — mixed case, some full-form values (BOOKS, GROCERY, TOYS) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

### `ecommerce.bronze.brz_category` (10 rows)

| Column | Type | Description |
| --- | --- | --- |
| `category_code` | string | Short category identifier (e.g., ce, app, hnk) — lowercase |
| `category_name` | string | Full category name (e.g., Electronics, Apparel) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

### `ecommerce.bronze.brz_customers` (300,000 rows)

| Column | Type | Description |
| --- | --- | --- |
| `customer_id` | string | Customer identifier (e.g., CUST000000000001) — 16 chars, 300 nulls |
| `phone` | string | Phone number with `.0` suffix from float storage (e.g., 917280033536.0) — 30,000 nulls |
| `country_code` | string | ISO country code (e.g., IN, AU, GB, US, AE, SG, CA) |
| `country` | string | Full country name (e.g., India, Australia, United Kingdom) |
| `state` | string | State/province code (e.g., MH, VIC, ENG, MA) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

### `ecommerce.bronze.brz_date` (95 rows)

| Column | Type | Description |
| --- | --- | --- |
| `date` | string | Date in DD-MM-YYYY format (e.g., 01-08-2025) |
| `year` | string | Year (e.g., 2025) |
| `day_name` | string | Day of week — inconsistent casing (e.g., friday, SATURDAY, SUNDAY) |
| `quarter` | string | Quarter number as string (e.g., 3) |
| `week_of_year` | string | Week number — negative values (e.g., -31) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

### `ecommerce.bronze.brz_products` (50,000 rows)

| Column | Type | Description |
| --- | --- | --- |
| `product_id` | string | Product identifier (e.g., 2000000028279) — 13 chars |
| `sku` | string | Stock keeping unit (e.g., ABLF-APP-0000L) |
| `category_code` | string | Foreign key to category dimension — lowercase (e.g., app) |
| `brand_code` | string | Foreign key to brand dimension — lowercase (e.g., ablf) |
| `color` | string | Product color — 4,126 nulls (e.g., Beige, Black) |
| `size` | string | Product size (e.g., L, M, S, XL) |
| `material` | string | Material — spelling mistakes present (e.g., Coton, Alumium, Ruber) |
| `weight_grams` | string | Weight with `g` suffix (e.g., 305g) |
| `length_cms` | string | Length with comma decimal separator (e.g., 22,2) |
| `width_cm` | string | Width — note: column name uses `_cm` not `_cms` |
| `height_cms` | string | Height (e.g., 6.3) |
| `rating_count` | string | Rating count — some negative values (e.g., -5) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

### `ecommerce.bronze.brz_order_items` (183,378 rows)

| Column | Type | Description |
| --- | --- | --- |
| `dt` | string | Transaction date (e.g., 2025-08-30) |
| `order_ts` | string | Order timestamp (e.g., 2025-08-30 12:46:50) |
| `customer_id` | string | Foreign key to customers dimension (e.g., CUST000000114495) |
| `order_id` | string | Order identifier (e.g., 676395) — 6 chars |
| `item_seq` | string | Line item sequence within an order (e.g., 1, 2, 3, 4, 5) |
| `product_id` | string | Foreign key to products dimension (e.g., 2000000136295) — 13 chars |
| `quantity` | string | Quantity ordered — mixed types (numeric and 'Two') (e.g., 1, 4, Two, 3) |
| `unit_price_currency` | string | Currency code for unit price (e.g., GBP, AUD, INR, USD, SGD, AED, CAD) |
| `unit_price` | string | Unit price amount — some with `$` prefix (e.g., 13, $6) |
| `discount_pct` | string | Discount percentage with `%` suffix (e.g., 16%, 7%, 0%) |
| `tax_amount` | string | Tax amount (e.g., 1, 3, 158) |
| `channel` | string | Sales channel — abbreviated (e.g., web, app) |
| `coupon_code` | string | Coupon code applied — 119,548 nulls (e.g., NEW10, PRIME5, FEST20, SAVE50) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

---

## Silver Layer

### `ecommerce.silver.slv_brands` (52 rows)

| Column | Type | Description |
| --- | --- | --- |
| `brand_code` | string | Brand identifier — cleaned: non-alphanumeric chars removed, trimmed (e.g., ACME, VOLT) |
| `brand_name` | string | Brand name — trimmed (e.g., AcmeTech, VoltEdge) |
| `Category_code` | string | Category code — standardized to short form (BOOKS→BKS, GROCERY→GRCY, TOYS→TOY). Note: column name retains capital C from Bronze. |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

### `ecommerce.silver.slv_category` (8 rows — 2 duplicates removed)

| Column | Type | Description |
| --- | --- | --- |
| `category_code` | string | Category identifier — uppercased (e.g., CE, APP, HNK, BPC, BKS, GRCY, TOY, SPT) |
| `category_name` | string | Full category name (e.g., Electronics, Apparel, Home & Kitchen) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

### `ecommerce.silver.slv_customers` (299,700 rows — 300 null customer_id rows dropped)

| Column | Type | Description |
| --- | --- | --- |
| `customer_id` | string | Customer identifier — nulls dropped (e.g., CUST000000000001) |
| `phone` | string | Phone number — `.0` suffix removed, nulls filled with 'Not Available' (e.g., 917280033536) |
| `country_code` | string | ISO country code (e.g., IN, AU, GB, US, AE, SG, CA) |
| `country` | string | Full country name (e.g., India, Australia, United Kingdom, United States) |
| `state` | string | State/province code (e.g., MH, VIC, ENG, MA) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

### `ecommerce.silver.slv_date` (92 rows — 3 duplicates removed)

| Column | Type | Description |
| --- | --- | --- |
| `date` | date | Date value — cast from string to date type (e.g., 2025-08-06) |
| `year` | integer | Year — cast from string to int (e.g., 2025) |
| `day_name` | string | Day of week — title-cased for consistency (e.g., Friday, Saturday, Sunday) |
| `quarter` | string | Readable quarter label (e.g., Q3-2025, Q4-2025) |
| `week_of_year` | string | Readable week label (e.g., Week32-2025, Week33-2025) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |
| `quarter_number` | integer | Numeric quarter value extracted from original quarter column (e.g., 3, 4) |
| `week_number` | integer | Numeric week value — absolute value of original week_of_year (e.g., 31, 32) |

### `ecommerce.silver.slv_products` (50,000 rows)

| Column | Type | Description |
| --- | --- | --- |
| `product_id` | string | Product identifier (e.g., 2000000028279) |
| `sku` | string | Stock keeping unit (e.g., ABLF-APP-0000L) |
| `category_code` | string | Category code — uppercased (e.g., APP, CE, HNK) |
| `brand_code` | string | Brand code — uppercased (e.g., ABLF, NOVW, HMNS) |
| `color` | string | Product color — nulls filled with 'NA' (e.g., Beige, Black, NA) |
| `size` | string | Product size (e.g., L, M, S, XL) |
| `material` | string | Material — spelling corrected (e.g., Cotton, Aluminum, Rubber) |
| `weight_grams` | float | Weight in grams — `g` suffix stripped, cast to float (e.g., 305.0) |
| `length_cms` | float | Length in cm — comma replaced with dot, cast to float (e.g., 22.2) |
| `width_cms` | float | Width in cm — cast from `width_cm` to float (e.g., 17.1). Note: original `width_cm` column was dropped. |
| `height_cms` | float | Height in cm — cast to float (e.g., 6.3) |
| `rating_count` | integer | Rating count — absolute value applied, cast to int (e.g., 0, 12, 6208) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

### `ecommerce.silver.slv_order_items` (183,378 rows — no rows dropped)

| Column | Type | Description |
| --- | --- | --- |
| `dt` | date | Transaction date — cast from string to date (e.g., 2025-08-30) |
| `order_ts` | timestamp | Order timestamp — cast from string to timestamp (e.g., 2025-08-30 12:46:50) |
| `customer_id` | string | Foreign key to customers dimension (e.g., CUST000000114495) |
| `order_id` | string | Order identifier (e.g., 676395) — 6 chars, 104,756 distinct |
| `item_seq` | integer | Line item sequence — cast from string to int (e.g., 1, 2, 3, 4, 5) |
| `product_id` | string | Foreign key to products dimension (e.g., 2000000136295) — 13 chars |
| `quantity` | integer | Quantity ordered — 'Two' replaced with '2', cast to int (e.g., 1, 2, 3, 4) |
| `unit_price_currency` | string | Currency code (e.g., GBP, AUD, INR, USD, SGD, AED, CAD) |
| `unit_price` | float | Unit price — `$` prefix stripped, cast to float (e.g., 13.0, 487.0) |
| `discount_pct` | float | Discount percentage — `%` suffix stripped, cast to float (e.g., 16.0, 7.0, 0.0) |
| `tax_amount` | float | Tax amount — cast to float (e.g., 1.0, 3.0, 158.0) |
| `channel` | string | Sales channel — standardized (e.g., Website, Application) |
| `coupon_code` | string | Coupon code — nulls filled with 'NA' (e.g., NEW10, PRIME5, FEST20, SAVE50, NA) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

---

## Gold Layer

### `ecommerce.gold.gld_dim_products` (50,000 rows)

Enriched by LEFT JOIN with `slv_brands` (on `brand_code`) and `slv_category` (on `category_code`).

| Column | Type | Description |
| --- | --- | --- |
| `product_id` | string | Product identifier — primary key (e.g., 2000000028279) |
| `sku` | string | Stock keeping unit (e.g., ABLF-APP-0000L) |
| `category_code` | string | Category code (e.g., APP, CE, HNK) |
| `category_name` | string | **Enriched** — full category name from `slv_category` (e.g., Apparel, Electronics) |
| `brand_code` | string | Brand code (e.g., ABLF, NOVW) |
| `brand_name` | string | **Enriched** — full brand name from `slv_brands` (e.g., AcmeTech, NovaWave) |
| `color` | string | Product color — nulls filled with 'NA' (e.g., Beige, NA) |
| `size` | string | Product size (e.g., L, M, S, XL) |
| `material` | string | Material — spelling corrected (e.g., Cotton, Aluminum, Rubber) |
| `weight_grams` | float | Weight in grams (e.g., 305.0) |
| `length_cms` | float | Length in cm (e.g., 22.2) |
| `width_cms` | float | Width in cm (e.g., 17.1) |
| `height_cms` | float | Height in cm (e.g., 6.3) |
| `rating_count` | integer | Rating count — non-negative (e.g., 0, 12, 6208) |
| `_source_file` | string | Source CSV file path (from products table) |
| `_ingested_at` | timestamp | Ingestion timestamp (from products table) |

### `ecommerce.gold.gld_dim_customers` (299,700 rows)

Enriched with `region` column via state-to-region mapping for 7 countries.

| Column | Type | Description |
| --- | --- | --- |
| `customer_id` | string | Customer identifier — primary key (e.g., CUST000000000001) |
| `phone` | string | Phone number — formatted, nulls as 'Not Available' (e.g., 917280033536) |
| `country_code` | string | ISO country code (e.g., IN, AU, GB, US, AE, SG, CA) |
| `country` | string | Full country name (e.g., India, Australia, United Kingdom) |
| `state` | string | State/province code (e.g., MH, VIC, ENG, MA) |
| `region` | string | **Enriched** — geographic region derived from country+state mapping. Values: West, South, North, SouthEast, East, NorthEast, England, Wales, Scotland, Northern Ireland, Abu Dhabi, Dubai, Sharjah, Singapore, Other (for unmapped states) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

**Region mapping coverage (7 countries):**

| Country | States Mapped | Regions |
| --- | --- | --- |
| India | 11 | West, South, North |
| Australia | 4 | SouthEast, West, East, NorthEast |
| UK | 4 | England, Wales, Scotland, Northern Ireland |
| US | 6 | NorthEast, South, West |
| UAE | 3 | Abu Dhabi, Dubai, Sharjah |
| Singapore | 1 | Singapore |
| Canada | 6 | West, East, Other |

Note: 87,150 rows had unmapped states and were assigned 'Other'. This is due to country name mismatches (e.g., mapping uses 'UK' but data has 'United Kingdom').

### `ecommerce.gold.gld_dim_date` (92 rows)

Enriched with `date_id`, `is_weekend`, and `month_name` derived columns.

| Column | Type | Description |
| --- | --- | --- |
| `date_id` | integer | **Enriched** — date as YYYYMMDD integer, serves as join key to fact table (e.g., 20250806) |
| `date` | date | Date value (e.g., 2025-08-06) |
| `year` | integer | Year (e.g., 2025) |
| `month_name` | string | **Enriched** — full month name (e.g., August, September, October) |
| `day_name` | string | Day of week — title-cased (e.g., Friday, Saturday) |
| `is_weekend` | integer | **Enriched** — 1 if Saturday or Sunday, 0 otherwise |
| `quarter` | string | Readable quarter label (e.g., Q3-2025, Q4-2025) |
| `quarter_number` | integer | Numeric quarter (e.g., 3, 4) |
| `week_of_year` | string | Readable week label (e.g., Week32-2025) |
| `week_number` | integer | Numeric week (e.g., 32, 33) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

### `ecommerce.gold.gld_fact_order_items` (183,378 rows)

Enriched fact table with 7 derived columns and several column renames for BI readability.

| Column | Type | Description |
| --- | --- | --- |
| `date_id` | string | **Enriched** — date as YYYYMMDD string, join key to `gld_dim_date` (e.g., 20250801) |
| `transaction_date` | date | Transaction date — renamed from `dt` (e.g., 2025-08-01) |
| `transaction_ts` | timestamp | Order timestamp — renamed from `order_ts` (e.g., 2025-08-01 22:53:52) |
| `transaction_id` | string | Order identifier — renamed from `order_id` (e.g., 643611) |
| `customer_id` | string | Foreign key to `gld_dim_customers` (e.g., CUST000000241190) |
| `seq_no` | integer | Line item sequence — renamed from `item_seq` (e.g., 1, 2, 3) |
| `product_id` | string | Foreign key to `gld_dim_products` (e.g., 2000000028279) |
| `channel` | string | Sales channel (e.g., Website, Application) |
| `coupon_code` | string | Coupon code — 'NA' if no coupon (e.g., NA, NEW10, PRIME5) |
| `coupon_flag` | integer | **Enriched** — 1 if coupon applied, 0 otherwise |
| `unit_price_currency` | string | Currency code (e.g., GBP, INR, AUD, USD, SGD, AED, CAD) |
| `quantity` | integer | Quantity ordered (e.g., 1, 2, 3, 4) |
| `unit_price` | float | Unit price in original currency (e.g., 11.0, 487.0) |
| `gross_amount` | float | **Enriched** — `quantity × unit_price`, total before discount (e.g., 11.0, 487.0) |
| `discount_percent` | float | Discount percentage — renamed from `discount_pct` (e.g., 10.0, 7.0, 0.0) |
| `discount_amount` | float | **Enriched** — `gross_amount × discount_pct / 100` (e.g., 1.1, 19.48) |
| `tax_amount` | float | Tax amount in original currency (e.g., 2.0, 24.0) |
| `net_amount` | float | **Enriched** — `gross_amount − discount_amount + tax_amount`, final amount owed — renamed from `sale_amount` (e.g., 11.9, 491.52) |
| `inr_rate` | float | **Enriched** — FX conversion rate to INR, looked up by currency (e.g., 117.98 for GBP) |
| `net_amount_inr` | float | **Enriched** — `net_amount × inr_rate`, sale amount in INR — renamed from `sale_amount_inr` (e.g., 1403.96) |
| `_source_file` | string | Source CSV file path |
| `_ingested_at` | timestamp | Ingestion timestamp |

**FX rates used for `inr_rate` lookup:**

| Currency | INR Rate |
| --- | --- |
| INR | 1.00 |
| AED | 24.18 |
| AUD | 57.55 |
| CAD | 62.93 |
| GBP | 117.98 |
| SGD | 68.18 |
| USD | 88.29 |

---

## Gold Denormalised View

### `ecommerce.gold.fact_transactions_denorm` (183,378 rows — VIEW)

Denormalised view joining the fact table with date and product dimensions. All fact columns plus dimension attributes and a derived time-of-day column.

| Column | Type | Source | Description |
| --- | --- | --- | --- |
| `date_id` | string | Fact | Date as YYYYMMDD — join key to date dimension |
| `transaction_date` | date | Fact | Transaction date |
| `transaction_ts` | timestamp | Fact | Order timestamp |
| `transaction_id` | string | Fact | Order identifier |
| `customer_id` | string | Fact | Customer identifier |
| `seq_no` | integer | Fact | Line item sequence |
| `product_id` | string | Fact | Product identifier |
| `channel` | string | Fact | Sales channel (Website/Application) |
| `coupon_code` | string | Fact | Coupon code or 'NA' |
| `coupon_flag` | integer | Fact | 1 if coupon applied, 0 otherwise |
| `unit_price_currency` | string | Fact | Currency code |
| `quantity` | integer | Fact | Quantity ordered |
| `unit_price` | float | Fact | Unit price in original currency |
| `gross_amount` | float | Fact | Total before discount |
| `discount_percent` | float | Fact | Discount percentage |
| `discount_amount` | float | Fact | Discount dollar value |
| `tax_amount` | float | Fact | Tax amount |
| `net_amount` | float | Fact | Final amount owed |
| `inr_rate` | float | Fact | FX rate to INR |
| `net_amount_inr` | float | Fact | Net amount in INR |
| `_source_file` | string | Fact | Source CSV file path |
| `_ingested_at` | timestamp | Fact | Ingestion timestamp |
| `year` | integer | Date dim | Year (e.g., 2025) |
| `month_name` | string | Date dim | Full month name (e.g., August) |
| `day_name` | string | Date dim | Day of week (e.g., Friday) |
| `is_weekend` | integer | Date dim | 1 if weekend, 0 otherwise |
| `quarter` | string | Date dim | Readable quarter label (e.g., Q3-2025) |
| `quarter_number` | integer | Date dim | Numeric quarter (e.g., 3) |
| `week_of_year` | string | Date dim | Readable week label (e.g., Week32-2025) |
| `week_number` | integer | Date dim | Numeric week (e.g., 32) |
| `sku` | string | Product dim | Stock keeping unit |
| `category_code` | string | Product dim | Category code |
| `category_name` | string | Product dim | Full category name |
| `brand_code` | string | Product dim | Brand code |
| `brand_name` | string | Product dim | Full brand name |
| `color` | string | Product dim | Product color or 'NA' |
| `size` | string | Product dim | Product size |
| `rating_count` | integer | Product dim | Product rating count |
| `hour_of_day` | integer | **Derived** | Hour extracted from `transaction_ts` (0–23) |

---

## Table Summary

| Layer | Table | Type | Rows | Columns |
| --- | --- | --- | --- | --- |
| Bronze | `brz_brands` | Table | 52 | 5 |
| Bronze | `brz_category` | Table | 10 | 4 |
| Bronze | `brz_customers` | Table | 300,000 | 7 |
| Bronze | `brz_date` | Table | 95 | 7 |
| Bronze | `brz_products` | Table | 50,000 | 14 |
| Bronze | `brz_order_items` | Table | 183,378 | 15 |
| Silver | `slv_brands` | Table | 52 | 5 |
| Silver | `slv_category` | Table | 8 | 4 |
| Silver | `slv_customers` | Table | 299,700 | 7 |
| Silver | `slv_date` | Table | 92 | 9 |
| Silver | `slv_products` | Table | 50,000 | 14 |
| Silver | `slv_order_items` | Table | 183,378 | 15 |
| Gold | `gld_dim_products` | Table | 50,000 | 16 |
| Gold | `gld_dim_customers` | Table | 299,700 | 8 |
| Gold | `gld_dim_date` | Table | 92 | 12 |
| Gold | `gld_fact_order_items` | Table | 183,378 | 22 |
| Gold | `fact_transactions_denorm` | View | 183,378 | 37 |

---

## Dashboard Consumption Layer

The `fact_transactions_denorm` view is consumed directly by a **Databricks AI/BI Dashboard** ("Sales Insights") with no additional ETL. The dashboard provides 13 widgets and 3 global filters built on a local metric view with 15 dimensions and 11 measures.

### Dashboard KPIs

| Metric | Value |
| --- | --- |
| Total Revenue | $1.81B |
| Total Orders | 104,756 |
| Total Quantity Sold | 245,635 |
| Avg Order Value | $17,324 |

### Dashboard Screenshot

![Sales Insights Dashboard](../dashboard/Sales%20Insights%202026-10-03%2008_27.png)

### Published Dashboard

[View Live Dashboard](https://dbc-72edc3ff-139d.cloud.databricks.com/dashboardsv3/01f1befd784d169f89bd5867c401359e/published?o=7474657952996939)