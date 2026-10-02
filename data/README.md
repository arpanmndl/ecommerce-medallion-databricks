# Data Source

The raw e-commerce dataset used in this project is **not included** in this repository. It is excluded via `.gitignore` to keep the repo lightweight and avoid committing large CSV files.

## Where to Get the Data

The dataset is provided by **CodeBasics** as part of their free Databricks mini-course:

> **CodeBasics — Databricks Mini Course (End-to-End Project)**
> [https://codebasics.io/resources/databrick-mini-course-end-to-end-project](https://codebasics.io/resources/databrick-mini-course-end-to-end-project?utm_source=youtube&utm_medium=description&utm_campaign=data_engineering&utm_content=761SQ9Hxbic)

### Steps to Download

1. Visit the link above
2. Scroll to the bottom of the page
3. Click the **Download Project Assets** button
4. Extract the downloaded ZIP file
5. Inside the extracted folder, locate the `0_data/` directory — this contains all raw CSV files

## Dataset Overview

The `0_data/` folder contains 6 CSV datasets that feed the Bronze layer of the medallion pipeline:

| Dataset | CSV Folder | Rows | Description |
| --- | --- | --- | --- |
| Brands | `brands/` | 52 | Brand codes and names |
| Category | `category/` | 10 | Category codes and names |
| Customers | `customers/` | 300,000 | Customer IDs, phone, country, state |
| Date | `date/` | 95 | Calendar dates with year, day name, quarter, week |
| Products | `products/` | 50,000 | Product IDs, SKU, category/brand codes, dimensions, ratings |
| Order Items | `order_items/landing/` | 183,378 | Transaction line items with pricing, tax, channel, coupons |

## How to Load the Data into Databricks

After downloading, upload the CSV files to a Databricks Managed Volume:

```
/Volumes/ecommerce/source_data/raw/
├── brands/*.csv
├── category/*.csv
├── customers/*.csv
├── date/*.csv
├── products/*.csv
└── order_items/landing/*.csv
```

You can upload via:

* **Databricks UI** — Catalog Explorer > Volumes > Upload
* **Databricks CLI** — `databricks fs cp -r 0_data/ /Volumes/ecommerce/source_data/raw/`
* **Notebook** — Use `dbutils.fs.cp()` to copy files programmatically

Once the data is in place, run the pipeline notebooks starting with `1_setup/` to create the catalog and schemas, then proceed through the Bronze → Silver → Gold layers.

## License

The dataset is provided by CodeBasics for educational purposes. Please refer to their [website](https://codebasics.io/resources/databrick-mini-course-end-to-end-project) for any usage terms and conditions.