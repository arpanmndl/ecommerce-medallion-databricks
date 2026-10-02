# Architecture Description — E-Commerce Medallion Data Platform on Databricks

## 1. Architecture Pattern: Medallion (Multi-Hop)

The project follows the **medallion architecture**, a data design pattern that organizes data into three layers of increasing quality and structure — **Bronze → Silver → Gold**. Each layer serves a distinct purpose and builds upon the previous one:

| Layer | Purpose | Data State | Write Mode | Table Naming |
| --- | --- | --- | --- | --- |
| Bronze | Raw ingestion — preserve source data as-is | Unstructured, all `StringType` | `overwrite` + `mergeSchema` | `brz_*` |
| Silver | Cleaning — type casting, null handling, deduplication, standardization | Cleansed, properly typed | `overwrite` + `mergeSchema` | `slv_*` |
| Gold | Enrichment — joins, derived columns, BI-ready denormalised views | Enriched, analysis-ready | `overwrite` + `mergeSchema` | `gld_*` / `fact_*` |

## 2. Technology Stack

| Component | Technology |
| --- | --- |
| Compute | Databricks (serverless) |
| Processing | Apache Spark (PySpark) |
| Storage Format | Delta Lake (managed tables) |
| Governance | Unity Catalog (`ecommerce` catalog) |
| Source Data | CSV files in Unity Catalog Volumes (`/Volumes/ecommerce/source_data/raw/`) |
| Orchestration | Databricks notebooks (sequential execution) |
| BI | Databricks SQL views + dashboards |

## 3. Unity Catalog Structure

```
ecommerce (CATALOG)
├── bronze (SCHEMA)
│   ├── brz_brands          (52 rows)
│   ├── brz_category        (10 rows)
│   ├── brz_customers       (300,000 rows)
│   ├── brz_date             (95 rows)
│   ├── brz_products     (50,000 rows)
│   └── brz_order_items (183,378 rows)
├── silver (SCHEMA)
│   ├── slv_brands          (52 rows)
│   ├── slv_category          (8 rows)
│   ├── slv_customers   (299,700 rows)
│   ├── slv_date             (92 rows)
│   ├── slv_products    (50,000 rows)
│   └── slv_order_items (183,378 rows)
└── gold (SCHEMA)
    ├── gld_dim_products   (50,000 rows)
    ├── gld_dim_customers (299,700 rows)
    ├── gld_dim_date          (92 rows)
    ├── gld_fact_order_items (183,378 rows)
    └── fact_transactions_denorm (VIEW)
```

## 4. Data Source Architecture

All raw data arrives as CSV files in Unity Catalog Volumes under `/Volumes/ecommerce/source_data/raw/`:

```
/Volumes/ecommerce/source_data/raw/
├── brands/*.csv          → 52 brand records (brand_code, brand_name, category_code)
├── category/*.csv        → 10 category records (category_code, category_name)
├── customers/*.csv       → 300,000 customer records (customer_id, phone, country, state)
├── date/*.csv            → 95 date records (date, year, day_name, quarter, week_of_year)
├── products/*.csv        → 50,000 product records (product_id, sku, category_code, brand_code, dimensions, rating)
└── order_items/landing/*.csv → 183,378 order line items (dt, order_ts, customer_id, order_id, product_id, pricing, tax, channel, coupon)
```

## 5. Pipeline Architecture (Notebook Orchestration)

The pipeline is structured as **8 notebooks** across **4 folders**, executed in a strict sequential dependency order:

```
                    ┌─────────────────────────────┐
                    │     0. 1_setup/             │
                    │  Create catalog + schemas   │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │                              │
          ┌─────────▼──────────┐    ┌─────────────▼──────────┐
          │ DIMENSION PIPELINE  │    │    FACT PIPELINE        │
          │ 2_medallion_        │    │ 3_medallion_            │
          │   processing_dim/   │    │   processing_fact/      │
          ├─────────────────────┤    ├────────────────────────┤
          │                     │    │                        │
          │  1_dim_bronze       │    │  1_fact_bronze         │
          │  (5 CSVs → Bronze)  │    │  (1 CSV → Bronze)      │
          │         │           │    │         │              │
          │  2_dim_silver       │    │  2_fact_silver        │
          │  (Clean → Silver)   │    │  (Clean → Silver)      │
          │         │           │    │         │              │
          │  3_dim_gold         │    │  3_fact_gold          │
          │  (Enrich → Gold)    │    │  (Enrich → Gold)       │
          └─────────┬──────────┘    └─────────────┬──────────┘
                    │                              │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │   4_denormalised_view/       │
                    │ 1_create_denorm_view_        │
                    │   reporting                  │
                    │ (Fact + Dims → Denorm VIEW)  │
                    └─────────────────────────────┘
                                   │
                          BI Dashboard Ready
```

**Dependency rules:**
* `1_setup` must run first (creates catalog/schemas)
* Dimension pipeline and fact pipeline are **independent** of each other and can run in parallel
* `4_denormalised_view` depends on **both** the dimension and fact gold tables being complete
* Within each pipeline, Bronze → Silver → Gold must run in order

## 6. Bronze Layer Design

**Design principle:** Preserve raw data exactly as received. No transformations, no type casting, no filtering.

* **Schema enforcement:** Explicit `StructType` with all fields as `StringType` — no schema inference, which prevents silent type mismatches
* **Lineage columns:** Every table gets `_source_file` (from Spark's `_metadata.file_path`) and `_ingested_at` (current timestamp)
* **Write strategy:** `mode("overwrite")` with `mergeSchema=True` — full refresh on each run, allows schema evolution
* **Volume source:** All CSVs read from Unity Catalog Volumes (not DBFS), enabling governance and access control

## 7. Silver Layer Design

**Design principle:** Clean and standardize without losing data. Column-by-column audit before any transformation.

**Dimension tables (5):**

| Table | Key Cleaning Steps |
| --- | --- |
| `slv_brands` | Remove non-alphanumeric chars from `brand_code` (`@`), trim whitespace, standardize category codes (BOOKS→BKS, GROCERY→GRCY, TOYS→TOY) |
| `slv_category` | Uppercase `category_code` for cross-table consistency, remove 2 duplicate rows |
| `slv_customers` | Drop 300 null `customer_id` rows (0.1%), fill 30K null phones with 'Not Available', strip `.0` suffix from phone numbers |
| `slv_date` | Cast `date`→date, `year`→int, title-case `day_name`, convert negative `week_of_year` to positive, create readable `quarter` (Q3-2025) and `week_of_year` (Week31-2025) with numeric companion columns, remove 3 duplicates |
| `slv_products` | Fill 4,126 null colors with 'NA', strip `g` suffix from weight→float, fix comma decimals→float, uppercase codes, fix material spelling (Coton→Cotton, Alumium→Aluminum, Ruber→Rubber), abs() on negative ratings→int |

**Fact table (1):**

| Table | Key Cleaning Steps |
| --- | --- |
| `slv_order_items` | Cast `dt`→date, `order_ts`→timestamp, `item_seq`→int, `quantity`: replace 'Two'→'2' then cast to int, `unit_price`: strip `$`→float, `discount_pct`: strip `%`→float, `tax_amount`→float, `channel`: web→Website, app→Application, `coupon_code`: fill 119,548 nulls with 'NA' |

**Data quality methodology:** Each column is individually inspected before transformation — distinct values checked, nulls counted, length consistency verified, describe() statistics generated. Assessment results are documented in markdown cells before applying fixes.

## 8. Gold Layer Design

**Design principle:** Enrich with business value. Add derived columns, join dimensions, create BI-ready structures.

### Gold Dimension Tables

| Table | Source | Enrichment | Join Keys |
| --- | --- | --- | --- |
| `gld_dim_products` | `slv_products` + `slv_brands` + `slv_category` | LEFT JOIN adds `brand_name`, `category_name` | `brand_code`, `category_code` |
| `gld_dim_customers` | `slv_customers` + region mapping | Adds `region` via state-to-region mapping for 7 countries; 87,150 unmapped filled with 'Other' | `country` + `state` |
| `gld_dim_date` | `slv_date` | Adds `date_id` (YYYYMMDD int), `is_weekend` (1/0), `month_name` | Derived from `date` |

**Region mapping:** A static Python dictionary maps country→state→region for India (11 states), Australia (4), UK (4), US (6), UAE (3), Singapore (1), Canada (6). Converted to a Spark DataFrame and LEFT JOINed onto the customers table.

### Gold Fact Table

| Table | Source | Derived Columns |
| --- | --- | --- |
| `gld_fact_order_items` | `slv_order_items` | 7 new columns |

| Derived Column | Formula | Purpose |
| --- | --- | --- |
| `gross_amount` | `quantity × unit_price` | Pre-discount total |
| `discount_amount` | `gross_amount × discount_pct / 100` | Discount dollar value |
| `net_amount` | `gross_amount − discount_amount + tax_amount` | Final amount owed |
| `date_id` | `dt` formatted as YYYYMMDD | Integer join key to date dimension |
| `coupon_flag` | `1 if coupon_code ≠ 'NA'` | Binary indicator |
| `inr_rate` | FX lookup by `unit_price_currency` | Currency conversion rate |
| `net_amount_inr` | `net_amount × inr_rate` | Standardized INR amount |

**FX rate architecture:** A static dictionary of 7 currencies (INR=1.00, AED=24.18, AUD=57.55, CAD=62.93, GBP=117.98, SGD=68.18, USD=88.29) is converted to a Spark DataFrame and LEFT JOINed on `unit_price_currency`. This normalizes all monetary amounts to a single currency (INR) for cross-region analysis.

**Column renaming for BI:** `dt`→`transaction_date`, `order_ts`→`transaction_ts`, `order_id`→`transaction_id`, `item_seq`→`seq_no`, `discount_pct`→`discount_percent`, `sale_amount`→`net_amount`, `sale_amount_inr`→`net_amount_inr`

### Gold Denormalised View

```sql
CREATE OR REPLACE VIEW ecommerce.gold.fact_transactions_denorm AS
  SELECT
    o.*,
    -- Date dimension attributes
    d.year, d.month_name, d.day_name, d.is_weekend,
    d.quarter, d.quarter_number, d.week_of_year, d.week_number,
    -- Product dimension attributes
    p.sku, p.category_code, p.category_name,
    p.brand_code, p.brand_name, p.color, p.size, p.rating_count,
    -- Derived time-of-day
    extract(HOUR FROM transaction_ts) AS hour_of_day
  FROM ecommerce.gold.gld_fact_order_items o
  LEFT JOIN ecommerce.gold.gld_dim_date     d ON o.date_id    = d.date_id
  LEFT JOIN ecommerce.gold.gld_dim_products p ON p.product_id = o.product_id
```

**Design rationale:** BI dashboards consume this view directly — no joins needed by report authors. `LEFT JOIN` preserves all fact rows even if dimension records are missing. `hour_of_day` is derived at query time from `transaction_ts`.

## 9. Data Model — Entity Relationships

```
                         ┌─────────────────────┐
                         │   gld_dim_date       │
                         │─────────────────────│
                         │ date_id (PK)        │
                         │ date, year, month    │
                         │ day_name, is_weekend │
                         │ quarter, week_number │
                         └──────────┬──────────┘
                                    │
                              date_id│
                                    │
┌──────────────────┐    ┌───────────┴──────────┐    ┌──────────────────────┐
│ gld_dim_products │    │ gld_fact_order_items  │    │  gld_dim_customers    │
│──────────────────│    │──────────────────────│    │──────────────────────│
│ product_id (PK)  │◄───┤ product_id (FK)       │    │ customer_id (PK)      │
│ sku, category    │    │ transaction_id        ├───►│ phone, country       │
│ brand_name       │    │ customer_id (FK)      │    │ state, region         │
│ color, size      │    │ date_id (FK)          │    └──────────────────────┘
│ material, weight │    │ quantity, unit_price  │
│ rating_count     │    │ gross/discount/net    │
└──────────────────┘    │ coupon_code/flag      │
                        │ inr_rate, net_inr     │
                        └──────────────────────┘
                                    │
                            fact_transactions_denorm
                            (VIEW — all 3 joined)
```

## 10. Key Architectural Decisions

| Decision | Rationale |
| --- | --- |
| All columns as `StringType` in Bronze | Preserves raw data; prevents ingestion failures from type mismatches; defers all type decisions to Silver where data can be inspected |
| Explicit `StructType` (no schema inference) | Ensures consistent column names/types across runs; rejects unexpected columns |
| `_source_file` + `_ingested_at` metadata columns | Provides full data lineage — every row knows which file it came from and when it was loaded |
| `mode("overwrite")` for all layers | Full-refresh model — simple, idempotent, no incremental state management needed. Suitable for this dataset size (183K rows) |
| LEFT JOINs for all gold enrichments | Preserves all fact/dimension rows even when dimension records are missing — no silent data loss |
| Static FX rate dictionary (not a lookup table) | Rates are fixed for analysis purposes; avoids dependency on an external FX API or table. Could be replaced with a dim table for dynamic rates |
| Region mapping as Python dict → Spark DataFrame | Handles 7 countries' state-to-region mapping in code; 87,150 unmapped rows filled with 'Other' |
| Denormalised VIEW (not a materialised table) | Always reflects the latest gold tables; no storage cost; computed on query. Suitable for 183K rows |
| Column renaming in Gold fact table | Gold layer is consumer-facing — names like `transaction_date` and `net_amount` are clearer for BI users than `dt` and `sale_amount` |
| `date_id` as integer (YYYYMMDD) | Compact, sortable, and serves as a clean join key between fact and date dimension |

## 11. Scalability Considerations

The current architecture is designed for **batch full-refresh** on a moderate dataset (~183K fact rows). Potential scaling paths:

| Current | Scaling Path |
| --- | --- |
| `mode("overwrite")` full refresh | Switch to `MERGE` / Delta upsert for incremental loads |
| Static FX dictionary | External FX rate dimension table updated daily |
| Sequential notebook execution | Orchestrate via Databricks Jobs with task dependencies |
| Manual run | Schedule via Databricks Jobs (cron-based or file-arrival trigger) |
| Single denormalised view | Add materialised views or pre-aggregated gold tables for heavy dashboard workloads |
| No data quality monitoring | Add Delta Live Tables expectations or Unity Catalog data quality rules |

## 12. Catalog/Schema/Governance Architecture

```
Databricks Workspace
└── Unity Catalog
    └── ecommerce (CATALOG)
        ├── bronze (SCHEMA)  — raw ingestion tables
        ├── silver (SCHEMA)  — cleaned & typed tables
        └── gold   (SCHEMA)  — BI-ready tables & views
```

All tables are **managed Delta tables** — Databricks handles storage, metadata, and optimization. The `ecommerce` catalog provides centralized governance, access control, and data discovery through Unity Catalog.