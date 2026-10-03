---
title: "Ecommerce Data Engineering Project Documentation"
description: "Databricks Lakehouse, Medallion Architecture, Analytics & Incremental Processing"
---

# Ecommerce Data Engineering Modernization

## 1. Executive Summary

This project demonstrates the modernization of an ecommerce data platform from a traditional relational-data workflow to a cloud-oriented Lakehouse architecture in Databricks. The implementation covers historical backfill into Delta tables, layered transformation using Bronze/Silver/Gold zones, business-ready modeling, analytics through Databricks Genie and a BI dashboard, and the setup pattern for daily incremental processing.

The pipeline processes **5 dimension tables** (brands, category, customers, date, products) and **1 fact table** (order items) across three medallion layers, producing **4 Gold tables** and **1 denormalised BI view** under the `ecommerce` Unity Catalog. The fact table contains 183,378 order line items spanning August–October 2025.

## 2. Business / Technical Problem

The source data describes a traditional system used to maintain and analyze large volumes of ecommerce transaction data. The modernization objective is to move the processing workflow toward a scalable Lakehouse pattern that separates raw ingestion, data quality transformations, and business-ready analytics data.

Key challenges addressed:

* Raw CSV data contains inconsistent formatting, mixed data types, spelling errors, and special characters
* Multiple currencies (GBP, INR, AUD, USD, SGD, AED, CAD) require normalization for cross-region analysis
* Dimension tables need enrichment (region mapping, brand/category name joins, date attributes) before they are analytics-ready
* BI dashboards require a single denormalised view to avoid complex joins at query time

## 3. Objectives

* Backfill historical source data into a Delta Lake / Lakehouse structure
* Organize transformations using a Bronze, Silver, and Gold medallion model
* Improve data quality through explicit schemas, cleaning, type conversion, and validation checks
* Create business-ready dimension and fact tables for analytics use cases
* Provide self-service business analysis through Databricks Genie
* Serve dashboard workloads from a denormalized Gold-layer view
* Automate recurring dimension and fact processing using Databricks Jobs and schedules

## 4. Scope

### In Scope

* **6 source CSV datasets** ingested from a Databricks Managed Volume (`/Volumes/ecommerce/source_data/raw/`)
* **5 dimension tables**: brands (52 rows), category (10 rows), customers (300,000 rows), date (95 rows), products (50,000 rows)
* **1 fact table**: order items (183,378 rows, 13 data columns)
* **Bronze layer**: raw ingestion with explicit `StringType` schemas and metadata lineage columns
* **Silver layer**: column-by-column data quality audit, type casting, null handling, deduplication, standardization
* **Gold layer**: enrichment via joins, derived monetary columns, FX currency conversion, region mapping, date attributes
* **Denormalised view**: BI-ready view joining fact + date + product dimensions
* **Documentation**: architecture description, data flow, and data dictionary

### Out of Scope

* Real-time / streaming ingestion (batch full-refresh only)
* Incremental MERGE/upsert logic (documented as a production consideration but not implemented)
* External API-based FX rate lookup (static dictionary used)
* Row-level security and column masking policies
* Data lineage visualization via Unity Catalog lineage UI

## 5. Target Architecture

The project uses a Databricks Lakehouse architecture with Unity Catalog governance. Raw CSV files are stored in a Databricks Managed Volume under `/Volumes/ecommerce/source_data/raw/` and processed through three medallion layers into managed Delta tables.

```
Databricks Workspace
├── Unity Catalog
│   └── ecommerce (CATALOG)
│       ├── bronze (SCHEMA) — 6 raw ingestion tables
│       ├── silver (SCHEMA) — 6 cleaned & typed tables
│       └── gold   (SCHEMA) — 4 enriched tables + 1 view
└── Volumes
    └── ecommerce/source_data/raw/ — 6 CSV source datasets
```

> **Note:** The project uses a Databricks Managed Volume to represent the raw landing zone. In a production environment, durable external object storage such as S3 or ADLS would be used instead.

## 6. Medallion Data Architecture

| Layer | Purpose | Data State | Write Mode | Table Naming |
| --- | --- | --- | --- | --- |
| Bronze | Raw ingestion — preserve source data as-is | Unstructured, all `StringType` | `overwrite` + `mergeSchema` | `brz_*` |
| Silver | Cleaning — type casting, null handling, deduplication, standardization | Cleansed, properly typed | `overwrite` + `mergeSchema` | `slv_*` |
| Gold | Enrichment — joins, derived columns, BI-ready denormalised views | Enriched, analysis-ready | `overwrite` + `mergeSchema` | `gld_*` / `fact_*` |

### Bronze Layer Design

* **Schema enforcement:** Explicit `StructType` with all fields as `StringType` — no schema inference, prevents silent type mismatches
* **Lineage columns:** Every table gets `_source_file` (from Spark's `_metadata.file_path`) and `_ingested_at` (current timestamp)
* **Design principle:** Preserve raw data exactly as received — no transformations, no type casting, no filtering

### Silver Layer Design

* **Column-by-column audit:** Each column is individually inspected (distinct values, null counts, length consistency, describe statistics) before any transformation
* **Type casting:** String columns cast to appropriate types (date, timestamp, integer, float)
* **Data cleaning:** Whitespace trimming, special character removal, spelling corrections, duplicate removal, null handling

### Gold Layer Design

* **Enrichment:** LEFT JOINs with other tables, derived columns (monetary calculations, flags, date attributes)
* **Column renaming:** Business-friendly names for BI consumption (e.g., `dt` → `transaction_date`, `sale_amount` → `net_amount`)
* **Denormalised view:** Single view joining fact + dimensions for dashboard consumption

## 7. Processing Flow

The implementation is organized into **8 notebooks** across **4 folders**, executed in a sequential dependency order. The dimension and fact pipelines are independent and can run in parallel; the denormalised view depends on both.

```
                    ┌─────────────────────────────┐
                    │     1_setup/                 │
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
          │  1_dim_bronze       │    │  1_fact_bronze         │
          │  2_dim_silver       │    │  2_fact_silver        │
          │  3_dim_gold         │    │  3_fact_gold          │
          └─────────┬──────────┘    └─────────────┬──────────┘
                    │                              │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │   4_denormalised_view/        │
                    │ 1_create_denorm_view_         │
                    │   reporting                  │
                    └─────────────────────────────┘
                                   │
                          BI Dashboard Ready
```
**Dependency rules:**

* `1_setup` must run first (creates catalog and schemas)
* Dimension pipeline and fact pipeline are **independent** of each other and can run in parallel
* `4_denormalised_view` depends on **both** the dimension and fact gold tables being complete
* Within each pipeline, Bronze → Silver → Gold must run in order

| Pipeline | Notebook / folder pattern | Main processing responsibilities |
| --- | --- | --- |
| Setup | `1_setup/` | Create the `ecommerce` catalog and Bronze/Silver/Gold schemas |
| Dimensions - Bronze | `2_medallion_processing_dim/1_dim_bronze` | Read 5 raw CSVs with explicit `StringType` schemas, add metadata columns, write to Bronze Delta tables |
| Dimensions - Silver | `2_medallion_processing_dim/2_dim_silver` | Column-by-column EDA, cleaning, duplicate removal, type casting, null handling, standardization |
| Dimensions - Gold | `2_medallion_processing_dim/3_dim_gold` | LEFT JOIN products with brands & category; add region mapping to customers; add `date_id`, `is_weekend`, `month_name` to date |
| Facts - Bronze | `3_medallion_processing_fact/1_fact_bronze` | Load order items CSV with explicit schema (13 columns), keep all fields as strings, add metadata, validate counts |
| Facts - Silver | `3_medallion_processing_fact/2_fact_silver` | Column-by-column audit: cast date/timestamp/int/float types, fix `quantity` ('Two'→2), strip `$` and `%`, clean channel, fill null coupons |
| Facts - Gold | `3_medallion_processing_fact/3_fact_gold` | Derive gross/discount/net amounts, FX conversion to INR, create `date_id` and `coupon_flag`, rename columns for BI, validate row count |
| Reporting | `4_denormalised_view/1_create_denorm_view_reporting` | Create `fact_transactions_denorm` view: LEFT JOIN fact + date dim + product dim, derive `hour_of_day` |

## 8. Key Data Modeling Logic

### Dimensions

* **Products** — Brand and category attributes are cleaned in Silver. In Gold, `slv_products` is LEFT JOINed with `slv_brands` (on `brand_code`) and `slv_category` (on `category_code`) to add `brand_name` and `category_name` → `gld_dim_products` (50,000 rows).
* **Customers** — Customer data is enriched with a `region` attribute using a country-state-to-region mapping covering 7 countries (India, Australia, UK, US, UAE, Singapore, Canada). 87,150 unmapped states are filled with 'Other' → `gld_dim_customers` (299,700 rows).
* **Date** — Calendar data is extended with `date_id` (YYYYMMDD integer join key), `is_weekend` flag (1/0), and `month_name` → `gld_dim_date` (92 rows).
* Column order and business-facing names are finalized before the Gold Delta tables are saved.

### Fact

* `gross_amount` = `quantity × unit_price` (total before discount)
* `discount_amount` = `gross_amount × discount_pct / 100` (dollar value of discount)
* `sale_amount` = `gross_amount − discount_amount + tax_amount` (final amount owed)
* `coupon_flag` = 1 if `coupon_code ≠ 'NA'`, else 0 (binary indicator)
* `date_id` = `dt` with hyphens removed (YYYYMMDD integer for joining to date dimension)
* `inr_rate` = FX conversion rate looked up by `unit_price_currency` (static dictionary: INR=1.00, AED=24.18, AUD=57.55, CAD=62.93, GBP=117.98, SGD=68.18, USD=88.29)
* `sale_amount_inr` = `sale_amount × inr_rate` (sale amount normalized to INR)

> **Note:** Currency is normalized to INR using project-level hard-coded rates. The source notes identify an API-based rate source as the production approach.

Column renames for BI readability: `dt`→`transaction_date`, `order_ts`→`transaction_ts`, `order_id`→`transaction_id`, `item_seq`→`seq_no`, `discount_pct`→`discount_percent`, `sale_amount`→`net_amount`, `sale_amount_inr`→`net_amount_inr`.

## 9. Analytics & BI Consumption

The project treats the Gold layer as the analytics source. Databricks Genie can be configured on Gold data to surface suggested business questions and generate queries for interactive analysis. A **Databricks AI/BI Dashboard** ("Sales Insights") is built directly on the denormalized Gold view (`fact_transactions_denorm`) that combines fact and dimension attributes and adds an `hour_of_day` field derived from `transaction_ts`.

> **Analytics principle:** Do not point business dashboards or Genie at the raw or Bronze layers; Gold is the analytics-ready layer.

### Sales Insights Dashboard

An interactive AI/BI Dashboard providing business-ready sales analytics with KPIs, trend analysis, and product breakdowns. Built on a local metric view over `fact_transactions_denorm` with 15 dimensions and 11 measures. Materialization is enabled for faster published dashboard loads.

#### Key Metrics (KPIs)

| Metric | Value |
| --- | --- |
| Total Revenue | $1.81B |
| Total Orders | 104,756 |
| Total Quantity Sold | 245,635 |
| Avg Order Value | $17,324 |

#### Dashboard Widgets

| Widget | Type | Description |
| --- | --- | --- |
| Total Revenue | Counter | Net revenue across all transactions |
| Total Orders | Counter | Distinct order count |
| Total Quantity Sold | Counter | Total items sold |
| Avg Order Value | Counter | Revenue per order |
| Monthly Revenue Trend | Line Chart | Revenue over time by month |
| Revenue by Channel | Bar Chart | Revenue split by Website vs Application |
| Top Products by Revenue | Table | Product SKUs ranked by revenue and quantity |
| Revenue by Category | Bar Chart | Revenue by product category |
| Revenue by Day of Week | Bar Chart | Revenue distribution across weekdays |
| Revenue by Hour of Day | Bar Chart | Revenue by hour (0-23) |
| Coupon Usage Distribution | Pie Chart | Orders with vs without coupons (35% coupon usage) |
| Top Brands by Revenue | Table | Brands ranked by revenue and order count |
| Quarterly Revenue by Year | Grouped Bar Chart | Quarterly revenue comparison across years |

#### Global Filters

* **Date Range** — Filter by transaction date
* **Sales Channel** — Filter by Website / Application
* **Product Category** — Filter by product category

#### Dashboard Screenshot

![Sales Insights Dashboard](../dashboard/Sales%20Insights%202026-10-03%2008_27.png)

#### Published Dashboard

[View Live Dashboard](https://dbc-72edc3ff-139d.cloud.databricks.com/dashboardsv3/01f1befd784d169f89bd5867c401359e/published?o=7474657952996939)

### Gold Layer Tables

| Table/View | Type | Rows | Description |
| --- | --- | --- | --- |
| `gld_dim_products` | Table | 50,000 | Products enriched with brand & category names |
| `gld_dim_customers` | Table | 299,700 | Customers enriched with region |
| `gld_dim_date` | Table | 92 | Dates with `date_id`, `is_weekend`, `month_name` |
| `gld_fact_order_items` | Table | 183,378 | Enriched order items with monetary calcs & FX conversion |
| `fact_transactions_denorm` | View | 183,378 | Denormalised fact + date + product dims for BI |

### Join Keys Between Gold Tables

| Fact Column | Dimension Table | Dimension Column |
| --- | --- | --- |
| `date_id` | `gld_dim_date` | `date_id` |
| `product_id` | `gld_dim_products` | `product_id` |
| `customer_id` | `gld_dim_customers` | `customer_id` |

## 10. Data Quality & Validation

| Checkpoint | Validation performed |
| --- | --- |
| Bronze ingestion | Schema applied; source-file metadata captured; loaded-row counts checked |
| Silver quality (dimensions) | EDA performed; duplicates removed (category: 2, date: 3); whitespace/special characters handled; nulls addressed; data types cast |
| Silver quality (fact) | Mixed data representations cleaned (`'Two'`→`'2'`, `$` stripped, `%` stripped); null coupon codes filled (119,548); all columns cast to proper types |
| Fact duplicates | Checked on composite key (`order_id`, `item_seq`) — no duplicates found |
| Gold continuity | Row count validated: 183,378 rows in Gold matches Silver — no data loss |
| Business rules | Domain owner input required for unique-code expectations and additional business columns |

### Key Data Quality Issues Found and Resolved

| Dimension | Issue | Resolution |
| --- | --- | --- |
| Brands | `@` characters in brand codes; trailing whitespace; inconsistent category code formats | Removed non-alphanumeric chars, trimmed, standardized codes (BOOKS→BKS, GROCERY→GRCY, TOYS→TOY) |
| Category | Lowercase codes; 2 duplicate rows | Uppercased codes, removed duplicates |
| Customers | 300 null customer IDs; 30K null phones; `.0` suffix on phones | Dropped null IDs, filled phones with 'Not Available', stripped `.0` |
| Date | All strings; negative week values; inconsistent day name casing; 3 duplicates | Cast types, title-cased days, absolute weeks, removed duplicates |
| Products | 4,126 null colors; `g` suffix on weight; comma decimals; lowercase codes; spelling errors; negative ratings | Filled nulls, stripped suffixes, fixed decimals, uppercased, corrected spelling, applied abs() |
| Fact | Mixed types (`'Two'`); `$` and `%` symbols; all strings; 65% null coupons | Replaced text values, stripped symbols, cast all types, filled nulls with 'NA' |

## 11. Operational Model

The project separates the one-time historical backfill from recurring daily processing. Historical data establishes the initial Delta/Lakehouse state. The recurring process is orchestrated through Databricks Jobs, with tasks pointing to the processing notebooks and triggers/schedules controlling execution. The source notes specify two jobs: one for dimension processing and one for fact processing.

## 12. Production Considerations

* Use durable external object storage (e.g., S3 or ADLS) rather than a Managed Volume for the raw landing zone
* Replace project-level hard-coded currency rates with a reliable API/data source for production calculations
* Define authoritative business rules with domain owners before finalizing unique codes, derived measures, and other Gold columns
* Implement incremental MERGE/upsert logic for daily processing (current model uses full `overwrite` refresh)
* Add data quality monitoring via Delta Live Tables expectations or Unity Catalog data quality rules
* Replace static region mapping with a maintained dimension table for production use

## 13. Repository / Documentation Structure

```
ecommerce-medallion-databricks/
├── architecture/
│   ├── architecture.md          — Detailed architecture description (13 sections)
│   ├── data-flow.md             — Step-by-step data flow with ASCII diagram
│   ├── data-dictionary.md       — Complete column-level documentation for all 17 tables
│   ├── E-Commerce Data Platform Architecture.png
│   └── E-Commerce Analytics Data Flow Pipeline.png
├── dashboard/
│   ├── Sales Insights.lvdash.json  — AI/BI Dashboard definition file
│   └── Sales Insights 2026-10-03 08_27.png — Dashboard screenshot
├── data/
│   └── README.md                — Dataset source and download instructions
├── docs/
│   └── project_documentation.md — This file
├── notebooks/
│   └── (notebook copies for version control)
├── requirements.txt
├── .gitignore
└── README.md
```

### Source Project Structure (Databricks Workspace)

```
project_ecommerce/
├── 1_setup/
│   └── New Notebook 2026-09-17 12:43:06.ipynb   — Create catalog + schemas
├── 2_medallion_processing_dim/
│   ├── 1_dim_bronze.ipynb                        — Ingest 5 dim CSVs to Bronze
│   ├── 2_dim_silver.ipynb                        — Clean 5 dim tables to Silver
│   └── 3_dim_gold.ipynb                          — Enrich 3 dim tables to Gold
├── 3_medallion_processing_fact/
│   ├── 1_fact_bronze.ipynb                       — Ingest order_items CSV to Bronze
│   ├── 2_fact_silver.ipynb                       — Clean order_items to Silver
│   └── 3_fact_gold.ipynb                         — Enrich order_items to Gold
└── 4_denormalised_view/
    └── 1_create_denorm_view_reporting.ipynb     — Create denormalised BI view
```

## 14. Technology Stack

| Component | Technology / pattern |
| --- | --- |
| Compute & processing | Databricks (serverless) / Apache Spark (PySpark) |
| Storage format | Delta Lake (managed tables) |
| Architecture | Medallion: Bronze → Silver → Gold |
| Governance | Unity Catalog (`ecommerce` catalog with bronze/silver/gold schemas) |
| Raw landing | Databricks Managed Volume; S3/ADLS noted as industry equivalents |
| Transformation | PySpark / Spark DataFrame operations |
| Orchestration | Databricks Jobs |
| Business Q&A | Databricks Genie |
| BI | Databricks dashboard over a Gold denormalized view |

## 15. Project Overview

| Document purpose | Shareable project overview for GitHub, technical handover, and colleague/stakeholder reference |
| --- | --- |
| Project type | Data engineering / ETL modernization / analytics platform |
| Primary platform | Databricks with Spark and Delta tables |
| Source pattern | Historical CSV files representing data extracted from a legacy relational system |
| Data architecture | Medallion architecture: Bronze → Silver → Gold |
| Analytics | Databricks Genie and a Databricks BI dashboard on a denormalized Gold view |
| Orchestration | Databricks Jobs for daily dimension and fact processing |

| Area | Included in the project | Notes |
| --- | --- | --- |
| Platform setup | Workspace folder, application catalog, Bronze/Silver/Gold schemas | Catalog setup is notebook-driven |
| Raw ingestion | CSV files loaded into a Databricks Managed Volume | Volume mimics a raw object-storage landing area |
| Dimensions | Bronze, Silver, and Gold processing | 5 tables: brands, category, customers, date, products |
| Fact table | Bronze, Silver, and Gold processing | 1 table: order items (183,378 rows) |
| Analytics | Databricks Genie | Gold layer is the analytics source |
| Dashboard | AI/BI Dashboard on denormalized Gold view | 13 widgets, 3 global filters, 4 KPIs ($1.81B revenue, 104,756 orders) |
| Incremental processing | Databricks Jobs, tasks, schedules, and triggers | Separate dimension and fact jobs |

> **Core outcome:** A repeatable path from raw source files to governed, analytics-ready Delta data, with orchestration for recurring processing.
