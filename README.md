# E-Commerce Data Pipeline on Databricks
### An End-to-End Data Engineering Project using the Medallion Architecture

---

## Project Overview

This project implements a production-style data pipeline for an e-commerce business using Databricks and Apache Spark. Raw transactional and dimensional data is ingested, cleaned, and transformed across three quality layers Bronze → Silver → Gold — following the Medallion Architecture pattern. The final Gold layer powers BI dashboards and business reporting.

This project was built as a part of my learning Data Engineering with the help of Dhaval Patel, Hemanandh and Codebasics team.(https://www.youtube.com/watch?v=761SQ9Hxbic)

---

## Architecture

```
Raw CSV Files (Landing Zone)
        │
        ▼
  ┌─────────────┐
  │   BRONZE    │  ← Raw ingestion, schema enforcement, metadata columns
  └─────────────┘
        │
        ▼
  ┌─────────────┐
  │   SILVER    │  ← Data cleaning, type casting, deduplication, anomaly fixes
  └─────────────┘
        │
        ▼
  ┌─────────────┐
  │    GOLD     │  ← BI-ready tables, enriched dimensions, calculated metrics
  └─────────────┘
        │
        ▼
  BI Dashboard / Genie AI
```

All layers are stored as Delta tables inside the `ecommerce` Unity Catalog.

---

## Project Structure

```
 1_dim_bronze.ipynb       # Ingest dimension tables into Bronze layer
 1_fact_bronze.ipynb      # Ingest order_items fact data into Bronze layer
 2_dim_silver.ipynb       # Clean and transform dimension tables → Silver
 2_fact_silver.ipynb      # Clean and transform order_items → Silver
 3_dim_gold.ipynb         # Build BI-ready dimension tables → Gold
 3_fact_gold.ipynb        # Build enriched fact table with metrics → Gold
 README.md
```

---

## Data Model

### Dimension Tables

| Table | Description |
|---|---|
| `brz/slv/gld_dim_products` | Product catalog with brand, category, material, size, weight |
| `brz/slv/gld_dim_customers` | Customer info with country, state, and derived region |
| `brz/slv/gld_dim_date` | Calendar table with date_id, month, quarter, week, is_weekend |
| `brz/slv/slv_brands` | Brand master with brand code and category mapping |
| `brz/slv/slv_category` | Product category reference |

### Fact Table

| Table | Description |
|---|---|
| `gld_fact_order_items` | Order line items with calculated sales metrics |

---

##  Pipeline Details

### Bronze Layer-- Raw Ingestion

Notebooks: `1_dim_bronze.ipynb`, `1_fact_bronze.ipynb`

- Reads raw CSV files from the Databricks Volume landing zone (`/Volumes/ecommerce/source_data/raw/`)
- Applies explicit Spark schemas for all entities (brands, category, products, customers, date, order_items)
- Preserves raw data as-is no transformations applied
- Adds two metadata columns to every table:
  - `_source_file` / `file_name` tracks the originating file path
  - `ingested_at` / `ingest_timestamp` records the ingestion timestamp
- Writes to Delta tables with `mergeSchema = true` for schema evolution support

Bronze Tables Created:
`brz_brands`, `brz_category`, `brz_products`, `brz_customers`, `brz_calendar`, `brz_order_items`

---

###  Silver Layer - Cleaning & Transformation

Notebooks: `2_dim_silver.ipynb`, `2_fact_silver.ipynb`

This layer handles all data quality issues discovered in the raw data.

#### Dimension Cleaning

| Entity | Issues Fixed |
|---|---|
| Brands | Trimmed whitespace, removed special characters from `brand_code`, corrected anomalous `category_code` values (e.g. `GROCERY → GRCY`) |
| Category | Removed duplicates on `category_code`, standardised casing to uppercase |
| Products | Stripped `g` suffix from `weight_grams` and cast to Integer; replaced commas with dots in `length_cm` and cast to Float; normalised `brand_code` and `category_code` to uppercase; fixed spelling errors in `material` (`Coton → Cotton`, `Alumium → Aluminum`, `Ruber → Rubber`); converted negative `rating_count` values to absolute values |
| Customers | Dropped rows with null `customer_id` (~300 rows); filled null `phone` values with `"Not Available"` |
| Calendar | Converted `date` from string (`dd-MM-yyyy`) to proper DateType; removed duplicate dates; normalised `day_name` casing using `initcap`; converted negative `week_of_year` to positive; enriched `quarter` (e.g. `Q1-2025`) and `week` (e.g. `Week1-2025`) with year labels; renamed `week_of_year → week` |

#### Fact Cleaning (`order_items`)

- Dropped duplicates on `(order_id, item_seq)`
- Converted text quantity `"Two"` - integer `2`
- Stripped `$` symbols from `unit_price` and cast to Double
- Stripped `%` from `discount_pct` and cast to Double
- Handled mixed-format timestamps in `order_ts` with fallback parsing
- Cast `dt` to DateType, `item_seq` to Integer, `tax_amount` to Double

---

###  Gold Layer - BI-Ready Tables

Notebooks: `3_dim_gold.ipynb`, `3_fact_gold.ipynb`

#### Dimension Enrichment

- `gld_dim_products` — Joins Products × Brands × Category using SQL CTEs; fills unmatched brands/categories with `"Not Available"` using COALESCE
- `gld_dim_customers`— Enriches customers with a **region** column derived from a manually curated country–state mapping (India, Australia, UK, USA, UAE); unmatched states default to `"Other"`
- `gld_dim_date` — Adds `date_id` (integer surrogate key in `yyyyMMdd` format), `month_name`, and `is_weekend` flag; reorders columns for BI consumption

#### Fact Metrics (`gld_fact_order_items`)

The following calculated columns are added to the order items:

| Column | Formula |
|---|---|
| `gross_amount` | `quantity × unit_price` |
| `discount_amount` | `ceil(gross_amount × discount_pct / 100)` |
| `sale_amount` | `gross_amount − discount_amount + tax_amount` |
| `coupon_flag` | Boolean — whether a coupon code was used |
| `date_id` | Integer surrogate key for joining with `gld_dim_date` |
| `sale_amount_inr` | `sale_amount` converted to INR using fixed FX rates |

FX Rates used (as of 2025-10-15):

| Currency | Rate to INR |
|---|---|
| USD | 88.29 |
| GBP | 117.98 |
| AUD | 57.55 |
| CAD | 62.93 |
| AED | 24.18 |
| SGD | 68.18 |
| INR | 1.00 |

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Databricks (Free Edition)| Unified analytics platform |
| Apache Spark / PySpark | Distributed data processing |
| Delta Lake | ACID-compliant storage format with time travel |
| Unity Catalog | Centralised data governance and access control |
| Databricks Volumes | Raw file storage (landing zone) |
| SQL | Gold layer table creation via Spark SQL |
| Databricks Workflows | Pipeline orchestration |
| Genie AI | Natural language querying of Gold tables |

---

##  How to Run

1. Sign up for [Databricks Free Edition](https://bit.ly/4nK0NTN)
2. Set up your catalog: create `ecommerce` catalog with `bronze`, `silver`, and `gold` schemas
3. Upload raw CSV files to `/Volumes/ecommerce/source_data/raw/`
4. Run notebooks in order:
   ```
   1_dim_bronze  →  1_fact_bronze
   2_dim_silver  →  2_fact_silver
   3_dim_gold    →  3_fact_gold
   ```
5. Connect to BI tool (Databricks SQL or Power BI) to the Gold layer tables

---

## Key Business Insights Enabled

- Total sales revenue by country, channel, and product category
- Discount and coupon usage analysis
- Customer segmentation by region
- Weekly and monthly sales trends
- Top products by revenue and volume

---

##  Credits

This project was built following the Codebasics Databricks Tutorial on YouTube.

- Tutorial: [Databricks Free Edition – End-to-End Data + AI Project](https://www.youtube.com/watch?v=761SQ9Hxbic)
- Instructors:Dhaval Patel & Hemanandh
- Channel:Codebasics (https://www.youtube.com/@codebasics)

