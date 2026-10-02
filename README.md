# Retail Data Engineering Project

## Overview

End-to-end retail data engineering pipeline built using Databricks, Apache Spark, Python and SQL. The project modernizes an e-commerce data platform by migrating raw CSV data from a legacy relational system into a scalable Lakehouse architecture. It processes **5 dimension tables** (brands, category, customers, date, products) and **1 fact table** (order items — 183,378 rows) through a Bronze → Silver → Gold medallion pipeline, producing BI-ready Delta tables and a denormalised view under Unity Catalog.

The dataset is sourced from the [CodeBasics Databricks Mini Course](https://codebasics.io/resources/databrick-mini-course-end-to-end-project?utm_source=youtube&utm_medium=description&utm_campaign=data_engineering&utm_content=761SQ9Hxbic). See [`data/README.md`](data/README.md) for download instructions.

## Tech Stack

* Databricks
* Apache Spark / PySpark
* Python
* SQL
* Delta Lake
* Medallion Architecture

## Architecture

Bronze → Silver → Gold

```
Raw CSVs → Bronze (StringType) → Silver (Cleaned & Typed) → Gold (Enriched) → Denormalised View → BI
```

| Layer | Purpose | Tables |
| --- | --- | --- |
| Bronze | Raw ingestion with explicit schemas + lineage columns | 6 tables (`brz_*`) |
| Silver | Cleaned, typed, deduplicated, standardized | 6 tables (`slv_*`) |
| Gold | Enriched with joins, derived columns, BI-ready | 4 tables (`gld_*`) + 1 view |

## Data Pipeline

1. **Ingest raw retail data** — Read 6 CSV datasets from a Databricks Managed Volume into Bronze Delta tables using explicit `StringType` schemas. Add `_source_file` and `_ingested_at` metadata columns for lineage.
2. **Store data in Bronze layer** — Preserve raw data as-is with no transformations. All columns kept as strings to prevent ingestion failures and defer type decisions downstream.
3. **Clean and transform data in Silver** — Column-by-column audit: cast strings to date/timestamp/int/float, handle nulls, remove duplicates, strip special characters, fix spelling errors, standardize category codes and channel names.
4. **Join and enrich datasets** — LEFT JOIN products with brands & category for names; map customers to geographic regions across 7 countries; add `date_id`, `is_weekend`, and `month_name` to the date dimension.
5. **Create Gold analytical tables** — Derive monetary columns (`gross_amount`, `discount_amount`, `net_amount`), normalize 7 currencies to INR using an FX rate lookup, add `coupon_flag` and `date_id`, rename columns for BI readability.
6. **Perform data quality validation** — Verify Bronze row counts, Silver duplicate checks on composite keys, Gold row count continuity (183,378 rows preserved end-to-end — zero data loss).
7. **Build analytical queries** — Create a denormalised Gold view (`fact_transactions_denorm`) joining fact + date + product dimensions with a derived `hour_of_day` column for BI dashboards and Databricks Genie.

## Project Structure

```
ecommerce-medallion-databricks/
├── README.md
├── requirements.txt
├── .gitignore
├── architecture/
│   ├── architecture.md            — Detailed architecture description (12 sections)
│   ├── data-flow.md               — Step-by-step data flow with ASCII pipeline diagram
│   ├── data-dictionary.md         — Complete column-level documentation for all 17 tables
│   ├── E-Commerce Data Platform Architecture.png
│   └── E-Commerce Analytics Data Flow Pipeline.png
├── data/
│   └── README.md                  — Dataset source and download instructions
├── docs/
│   └── project_documentation.md   — Full project documentation (15 sections)
├── notebooks/
│   ├── 1_setup/
│   │   └── New Notebook 2026-09-17 12:43:06.ipynb   — Create catalog + schemas
│   ├── 2_medallion_processing_dim/
│   │   ├── 1_dim_bronze.ipynb                        — Ingest 5 dim CSVs to Bronze
│   │   ├── 2_dim_silver.ipynb                        — Clean 5 dim tables to Silver
│   │   └── 3_dim_gold.ipynb                          — Enrich 3 dim tables to Gold
│   ├── 3_medallion_processing_fact/
│   │   ├── 1_fact_bronze.ipynb                       — Ingest order_items CSV to Bronze
│   │   ├── 2_fact_silver.ipynb                       — Clean order_items to Silver
│   │   └── 3_fact_gold.ipynb                         — Enrich order_items to Gold
│   └── sql/
│       └── 1_create_denorm_view_reporting            — Create denormalised BI view
```

## Data Quality

| Checkpoint | Validation Performed |
| --- | --- |
| Bronze ingestion | Explicit schema applied; source-file metadata captured; loaded-row counts verified |
| Silver quality (dimensions) | EDA per column; duplicates removed (category: 2, date: 3); whitespace/special characters handled; nulls addressed; data types cast |
| Silver quality (fact) | Mixed representations cleaned (`'Two'`→`'2'`, `$` stripped, `%` stripped); 119,548 null coupon codes filled with `'NA'`; all columns cast to proper types |
| Fact duplicates | Checked on composite key (`order_id`, `item_seq`) — no duplicates found |
| Gold continuity | Row count validated: 183,378 rows in Gold matches Silver — zero data loss |

### Key Data Quality Issues Found and Resolved

| Dimension | Issue | Resolution |
| --- | --- | --- |
| Brands | `@` characters in brand codes; trailing whitespace | Removed non-alphanumeric chars, trimmed, standardized codes |
| Category | Lowercase codes; 2 duplicate rows | Uppercased codes, removed duplicates |
| Customers | 300 null customer IDs; 30K null phones; `.0` suffix | Dropped null IDs, filled phones, stripped `.0` |
| Date | All strings; negative week values; inconsistent casing; 3 duplicates | Cast types, title-cased days, absolute weeks, removed duplicates |
| Products | 4,126 null colors; `g` suffix on weight; comma decimals; spelling errors; negative ratings | Filled nulls, stripped suffixes, fixed decimals, corrected spelling, applied `abs()` |
| Fact | Mixed types; `$`/`%` symbols; all strings; 65% null coupons | Replaced text values, stripped symbols, cast types, filled nulls |

## Key Transformations

### Dimension Enrichment

* **Products** — LEFT JOIN with `slv_brands` (on `brand_code`) and `slv_category` (on `category_code`) to add `brand_name` and `category_name`
* **Customers** — Added `region` column via state-to-region mapping for 7 countries (India, Australia, UK, US, UAE, Singapore, Canada); 87,150 unmapped rows filled with `'Other'`
* **Date** — Added `date_id` (YYYYMMDD integer join key), `is_weekend` flag (1/0), and `month_name`

### Fact Enrichment

| Derived Column | Formula |
| --- | --- |
| `gross_amount` | `quantity × unit_price` |
| `discount_amount` | `gross_amount × discount_pct / 100` |
| `net_amount` | `gross_amount − discount_amount + tax_amount` |
| `date_id` | `dt` formatted as YYYYMMDD |
| `coupon_flag` | `1 if coupon_code ≠ 'NA', else 0` |
| `inr_rate` | FX lookup by `unit_price_currency` |
| `net_amount_inr` | `net_amount × inr_rate` |

### Column Renames for BI

`dt` → `transaction_date` | `order_ts` → `transaction_ts` | `order_id` → `transaction_id` | `item_seq` → `seq_no` | `discount_pct` → `discount_percent` | `sale_amount` → `net_amount` | `sale_amount_inr` → `net_amount_inr`

### Denormalised View

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

## Results

| Table / View | Layer | Type | Rows | Description |
| --- | --- | --- | --- | --- |
| `brz_brands` | Bronze | Table | 52 | Raw brand data |
| `brz_category` | Bronze | Table | 10 | Raw category data |
| `brz_customers` | Bronze | Table | 300,000 | Raw customer data |
| `brz_date` | Bronze | Table | 95 | Raw date data |
| `brz_products` | Bronze | Table | 50,000 | Raw product data |
| `brz_order_items` | Bronze | Table | 183,378 | Raw order items |
| `slv_brands` | Silver | Table | 52 | Cleaned brands |
| `slv_category` | Silver | Table | 8 | Cleaned categories (2 duplicates removed) |
| `slv_customers` | Silver | Table | 299,700 | Cleaned customers (300 nulls dropped) |
| `slv_date` | Silver | Table | 92 | Cleaned dates (3 duplicates removed) |
| `slv_products` | Silver | Table | 50,000 | Cleaned products |
| `slv_order_items` | Silver | Table | 183,378 | Cleaned order items |
| `gld_dim_products` | Gold | Table | 50,000 | Products with brand & category names |
| `gld_dim_customers` | Gold | Table | 299,700 | Customers with region mapping |
| `gld_dim_date` | Gold | Table | 92 | Dates with `date_id`, `is_weekend`, `month_name` |
| `gld_fact_order_items` | Gold | Table | 183,378 | Order items with monetary calcs & FX conversion |
| `fact_transactions_denorm` | Gold | View | 183,378 | Denormalised fact + date + product dims for BI |

**Zero data loss**: 183,378 fact rows preserved from Bronze through Gold. All 4 Gold tables and 1 view are ready for BI dashboards and Databricks Genie.

## How to Run

### Prerequisites

* Databricks workspace with Unity Catalog enabled
* Serverless compute (auto-selected)
* Source CSV data uploaded to `/Volumes/ecommerce/source_data/raw/` (see [`data/README.md`](data/README.md) for download instructions)

### Steps

1. **Setup** — Run the notebook in `1_setup/` to create the `ecommerce` catalog and `bronze`/`silver`/`gold` schemas
2. **Dimension pipeline** — Run in order:
   * `1_dim_bronze` → `2_dim_silver` → `3_dim_gold`
3. **Fact pipeline** — Run in order (can run in parallel with dimensions):
   * `1_fact_bronze` → `2_fact_silver` → `3_fact_gold`
4. **Reporting view** — Run `1_create_denorm_view_reporting` (after both pipelines complete)
5. **Query** — Use Databricks SQL or Genie:

```sql
SELECT * FROM ecommerce.gold.fact_transactions_denorm LIMIT 10;
```

### Local Development

```bash
pip install -r requirements.txt
```

## Documentation

| File | Content |
| --- | --- |
| [`architecture/architecture.md`](architecture/architecture.md) | Full architecture description — layer designs, data model, decisions, scalability |
| [`architecture/data-flow.md`](architecture/data-flow.md) | Step-by-step data flow with ASCII pipeline diagram |
| [`architecture/data-dictionary.md`](architecture/data-dictionary.md) | Complete column-level documentation for all 17 tables |
| [`docs/project_documentation.md`](docs/project_documentation.md) | Project summary — scope, objectives, processing flow, data quality |
| [`data/README.md`](data/README.md) | Dataset source and download instructions |

## License

This project is for educational and portfolio purposes. Dataset provided by [CodeBasics](https://codebasics.io/resources/databrick-mini-course-end-to-end-project).