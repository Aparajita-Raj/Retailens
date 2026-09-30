1. Project Overview

RetailLens is an end-to-end e-commerce analytics pipeline built using Databricks and Snowflake.

The project takes the Olist Brazilian E-Commerce dataset through a layered data-processing workflow:

Olist CSV files
      |
      v
Databricks Volume
      |
      v
Bronze
Raw ingestion + provenance
      |
      v
Silver
Cleaning + validation + business grains
      |
      v
Gold
Business-ready aggregates
      |
      v
CSV export
      |
      v
Snowflake Stage
      |
      v
COPY INTO
      |
      v
Snowflake Analytics

The pipeline is designed around three business questions:

How does late delivery affect customer review scores across product categories?

How much revenue comes from repeat customers, and how does it change quarter by quarter?

Which sellers show higher late-delivery rates, and how does that relate to review scores?

2. Dataset

The project uses the Olist Brazilian E-Commerce public dataset, consisting of nine linked CSV files:

olist_orders_dataset.csv

olist_order_items_dataset.csv

olist_order_payments_dataset.csv

olist_order_reviews_dataset.csv

olist_customers_dataset.csv

olist_products_dataset.csv

olist_sellers_dataset.csv

product_category_name_translation.csv

olist_geolocation_dataset.csv

The files are stored in Databricks under:

/Volumes/workspace/dbsf_2328073/raw_files

3. Databricks Layers

Bronze

Notebook:

01_Bronze_2328073.ipynb

Bronze keeps the source data close to the original form. CSV schema inference is disabled so the raw business fields are not silently converted during ingestion.

The Bronze tables add provenance metadata:

_source_file

_ingested_at

_row_hash

The Unity Catalog-compatible implementation uses:

_metadata.file_path

for source-file provenance.

Main Bronze tables:

bronze_orders_2328073
bronze_order_items_2328073
bronze_order_payments_2328073
bronze_order_reviews_2328073
bronze_customers_2328073
bronze_products_2328073
bronze_sellers_2328073
bronze_category_translation_2328073
bronze_geolocation_2328073

Silver

Notebook:

02_Silver_2328073.ipynb

Silver performs the main cleaning and modeling work:

safe timestamp and numeric conversion using try_cast

order deduplication

orphan-record checks

delivery-duration calculation

late_flag creation

review aggregation

customer, product and seller joins

join-safe analytical grains

Main Silver tables:

silver_orders_2328073
silver_order_items_2328073
silver_order_payments_2328073
silver_order_reviews_2328073
silver_customers_2328073
silver_products_2328073
silver_sellers_2328073
silver_rejects_2328073
silver_order_metrics_2328073
silver_order_category_2328073
silver_seller_order_2328073

Gold

Notebook:

03_Gold_2328073.ipynb

Gold contains three business-ready outputs:

gold_category_delivery_review_2328073
gold_repeat_customer_quarter_2328073
gold_seller_performance_2328073

These are exported to:

/Volumes/workspace/dbsf_2328073/raw_files/gold_export/

4. Snowflake Layer

Database:

RETAILLENS_2328073

Schema:

ANALYTICS

Named stage:

RETAILLENS_STAGE

File format:

RETAILLENS_CSV_FF

Analytics tables:

CATEGORY_DELIVERY_REVIEW
REPEAT_CUSTOMER_QUARTER
SELLER_PERFORMANCE

The Gold outputs are transferred to Snowflake using:

Gold CSV
   |
   v
Snowflake internal stage
   |
   v
COPY INTO
   |
   v
Analytics tables

METADATA$FILENAME was used to identify staged files before loading.

COPY_HISTORY was used to verify the load.

A repeated COPY INTO of an already-loaded file returned:

0 files processed

5. Databricks Job

The pipeline is orchestrated using a Databricks Job with three dependent tasks:

Bronze_2328073
       |
       v
Silver_2328073
       |
       v
Gold_2328073

The successful project run completed the three tasks in sequence.

Observed run time:

Bronze: 1m 58s
Silver: 1m 12s
Gold:   35.3s
Total:  3m 46s

Compute used in the Job:

Serverless

6. Results

Observed project results include:

Measure

Result

Orders processed

99,441

Order items

112,650

Sellers in Silver

3,095

Snowflake category rows

74

Snowflake repeat-quarter rows

10

Snowflake seller rows

2,970

Eligible sellers (delivered_orders >= 20)

804

Seller late-rate/review correlation

-0.51

For category analysis, the Snowflake query uses a minimum of 50 orders per category.

For seller analysis, the Snowflake query uses a minimum of 20 delivered orders.

The seller correlation is descriptive and indicates an observed association between seller late rate and average review score; it should not be treated as proof of causation.

7. How to Run

Databricks

Open the Databricks workspace.

Make sure the source CSV files exist in:

/Volumes/workspace/dbsf_2328073/raw_files

Open:

01_Bronze_2328073
02_Silver_2328073
03_Gold_2328073

Run the notebooks in order:

01_Bronze_2328073
        |
        v
02_Silver_2328073
        |
        v
03_Gold_2328073

Or run the configured Databricks Job:

Bronze_2328073 → Silver_2328073 → Gold_2328073

Snowflake

After the Gold CSV exports are available:

Create/use the database:

CREATE DATABASE IF NOT EXISTS RETAILLENS_2328073;
USE DATABASE RETAILLENS_2328073;

Create/use the schema:

CREATE SCHEMA IF NOT EXISTS ANALYTICS;
USE SCHEMA ANALYTICS;

Create the CSV file format and stage.

Upload the three Gold CSV files to RETAILLENS_STAGE.

Create the three analytics tables.

Run COPY INTO for each file.

Validate row counts.

Query INFORMATION_SCHEMA.COPY_HISTORY.

Run the three business-analysis queries.

8. Business Queries

Question 1 — Delivery delay vs reviews

The category query compares:

average delivery days

late-delivery rate

average review score

late-order review score

on-time review score

review-score gap

Main table:

CATEGORY_DELIVERY_REVIEW

Question 2 — Repeat-customer revenue

The quarterly query compares:

total orders

total revenue

repeat orders

repeat revenue

repeat revenue percentage

Main table:

REPEAT_CUSTOMER_QUARTER

Question 3 — Seller performance

The seller query compares:

delivered orders

late orders

late rate

average review score

revenue

state-level late-rate rank

It uses a CTE and a window function:

RANK() OVER (
    PARTITION BY SELLER_STATE
    ORDER BY LATE_RATE DESC
)

Main table:

SELLER_PERFORMANCE

9. Repository Structure

A suggested GitHub repository layout is:

RetailLens/
│
├── README.md
│
├── notebooks/
│   ├── 01_Bronze_2328073.ipynb
│   ├── 02_Silver_2328073.ipynb
│   └── 03_Gold_2328073.ipynb
│
├── sql/
│   └── 2328073_Capstone.sql
│
├── report/
│   └── RetailLens_Report_2328073.pdf
│
└── screenshots/
    ├── bronze_success.png
    ├── silver_success.png
    ├── gold_output.png
    ├── databricks_job_success.png
    ├── snowflake_q1.png
    ├── snowflake_q2.png
    ├── snowflake_q3.png
    └── copy_history.png

10. Important GitHub Safety Notes

Do not commit:

Snowflake passwords

access tokens

API keys

private keys

secret scopes or secret values

browser credentials

private account information

The notebooks in this project use object names and paths, not passwords or tokens.

For a public repository, replace any environment-specific paths or account-specific settings that are not required for reproduction.

11. Project Evidence

The project includes evidence for:

Bronze raw ingestion

Bronze row-count validation

Silver processing

Gold outputs

Databricks Job success

Snowflake serving-table row counts

repeated COPY INTO

COPY_HISTORY

three business-question outputs
