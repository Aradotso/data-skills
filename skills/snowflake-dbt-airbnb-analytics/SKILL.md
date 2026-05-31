---
name: snowflake-dbt-airbnb-analytics
description: Portfolio analytics engineering project using Snowflake, dbt, and Streamlit for Inside Airbnb data transformation and visualization
triggers:
  - set up snowflake dbt analytics project
  - load inside airbnb data to snowflake
  - run dbt models for airbnb analytics
  - build airbnb data warehouse with dbt
  - create streamlit dashboard for airbnb data
  - configure snowflake and dbt for inside airbnb
  - test dbt models for data quality
  - generate dbt documentation for airbnb project
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you work with a complete analytics engineering portfolio project that demonstrates loading Inside Airbnb data into Snowflake, transforming it with dbt across staging, intermediate, and mart layers, and visualizing results in a Streamlit dashboard.

## What This Project Does

The Snowflake_DBT_Project is a classroom-safe analytics engineering reference implementation that:

- Loads Inside Airbnb CSV/GZIP files into Snowflake using internal stages
- Transforms raw data through dbt staging → intermediate → marts layers
- Implements incremental fact modeling with Snowflake merge strategy
- Validates data quality with generic and singular dbt tests
- Generates dbt documentation
- Powers a Streamlit dashboard for business analytics

**Key architectural components:**
- Raw data in Snowflake `RAW` schema (text-preserving)
- dbt staging views for cleaning and standardization
- Intermediate models for joins and enrichment
- Dimension tables (`dim_listings`, `dim_hosts`)
- Fact tables (`fct_listing_calendar`, `fct_reviews`)
- Monthly aggregates for reporting

## Installation

### Prerequisites

- Snowflake account with appropriate privileges
- Python 3.8+
- Inside Airbnb dataset files (recommended: NYC)

### Setup Steps

```bash
# Clone the repository
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project

# Create and activate virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Credential Configuration

This project uses **local-only** credential files that are git-ignored. Never commit real credentials.

```bash
# Create local credential files from examples
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**Edit `profiles.yml`** with your Snowflake connection details:

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USER
      password: YOUR_PASSWORD  # Or use authenticator: externalbrowser
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Edit `config/local_credentials.json`** for Streamlit dashboard:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USER",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

## Dataset Preparation

Download Inside Airbnb data from [insideairbnb.com/get-the-data](http://insideairbnb.com/get-the-data/).

**Expected file structure:**

```
data/raw/
├── listings.csv.gz
├── calendar.csv.gz
├── reviews.csv.gz
└── neighbourhoods.csv
```

**For NYC (recommended):**
- Navigate to New York City, New York, United States
- Download `listings.csv.gz`, `calendar.csv.gz`, `reviews.csv.gz`, `neighbourhoods.csv`
- Place in `data/raw/` directory

## Loading Raw Data to Snowflake

The Python loader script automates the entire raw data ingestion process.

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What the loader does:**

1. Executes `setup/snowflake_setup.sql` to create database, schemas, warehouse, and stage
2. Uploads local files to Snowflake internal stage `INSIDE_AIRBNB_STAGE`
3. Creates raw tables with schema inferred from CSV headers
4. Copies data from stage into `RAW` schema tables

**Key loader code pattern:**

```python
import snowflake.connector
import json

# Load credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)

# Connect to Snowflake
conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    warehouse=creds['warehouse'],
    database=creds['database'],
    role=creds['role']
)

cursor = conn.cursor()

# Create stage
cursor.execute("""
    CREATE OR REPLACE STAGE INSIDE_AIRBNB_STAGE
    FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
""")

# Upload file to stage
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Create raw table and copy data
cursor.execute("""
    CREATE OR REPLACE TABLE RAW.LISTINGS AS
    SELECT * FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
""")

cursor.close()
conn.close()
```

## dbt Commands

All dbt commands use `--profiles-dir .` to reference the local `profiles.yml`.

### Initial Build

```bash
# Verify connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation locally
dbt docs serve --profiles-dir .
```

### Incremental Refresh (New Data Snapshot)

When you load a fresh Inside Airbnb snapshot:

```bash
# Load new raw data
python scripts/load_inside_airbnb_to_snowflake.py

# Full refresh the incremental fact table and downstream models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Test data quality
dbt test --profiles-dir .
```

### Selective Model Execution

```bash
# Run only staging models
dbt run --select staging.* --profiles-dir .

# Run specific model and dependencies
dbt run --select +agg_neighbourhood_monthly_performance --profiles-dir .

# Run specific model and downstream models
dbt run --select int_airbnb__listing_enriched+ --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .
```

## dbt Model Layers

### Staging Models (`models/staging/`)

Clean and standardize raw data with consistent column naming.

**Example: `stg_airbnb__listings.sql`**

```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'listings') }}
)

SELECT
    id::NUMBER AS listing_id,
    TRIM(name) AS listing_name,
    host_id::NUMBER AS host_id,
    TRIM(host_name) AS host_name,
    TRIM(neighbourhood) AS neighbourhood,
    TRIM(room_type) AS room_type,
    price::FLOAT AS price,
    minimum_nights::NUMBER AS minimum_nights,
    number_of_reviews::NUMBER AS number_of_reviews,
    last_review::DATE AS last_review_date,
    reviews_per_month::FLOAT AS reviews_per_month,
    calculated_host_listings_count::NUMBER AS host_listings_count,
    availability_365::NUMBER AS availability_365,
    latitude::FLOAT AS latitude,
    longitude::FLOAT AS longitude
FROM source
WHERE id IS NOT NULL
```

**Source definition (`models/staging/sources.yml`):**

```yaml
version: 2

sources:
  - name: raw
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: listings
        columns:
          - name: id
            tests:
              - not_null
              - unique
      - name: calendar
      - name: reviews
      - name: neighbourhoods
```

### Intermediate Models (`models/intermediate/`)

Reusable joins and business logic.

**Example: `int_airbnb__listing_enriched.sql`**

```sql
WITH listings AS (
    SELECT * FROM {{ ref('stg_airbnb__listings') }}
),

neighbourhoods AS (
    SELECT * FROM {{ ref('stg_airbnb__neighbourhoods') }}
)

SELECT
    l.listing_id,
    l.listing_name,
    l.host_id,
    l.host_name,
    l.neighbourhood,
    n.neighbourhood_group,
    l.room_type,
    l.price,
    l.minimum_nights,
    l.number_of_reviews,
    l.last_review_date,
    l.reviews_per_month,
    l.host_listings_count,
    l.availability_365,
    l.latitude,
    l.longitude,
    CASE
        WHEN l.price > 0 AND l.availability_365 > 0 THEN TRUE
        ELSE FALSE
    END AS is_active_listing
FROM listings l
LEFT JOIN neighbourhoods n
    ON l.neighbourhood = n.neighbourhood
```

### Mart Models (`models/marts/`)

#### Dimension: `dim_listings.sql`

```sql
SELECT
    listing_id,
    listing_name,
    host_id,
    neighbourhood,
    neighbourhood_group,
    room_type,
    minimum_nights,
    latitude,
    longitude,
    CURRENT_TIMESTAMP() AS dbt_loaded_at
FROM {{ ref('int_airbnb__listing_enriched') }}
WHERE is_active_listing = TRUE
```

#### Incremental Fact: `fct_listing_calendar.sql`

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        on_schema_change='fail'
    )
}}

WITH calendar AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
)

SELECT
    listing_id,
    calendar_date,
    available,
    price,
    adjusted_price,
    minimum_nights,
    maximum_nights,
    estimated_revenue,
    CURRENT_TIMESTAMP() AS dbt_loaded_at
FROM calendar

{% if is_incremental() %}
WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**Incremental strategy:** Uses Snowflake merge on composite key `[listing_id, calendar_date]`.

#### Aggregate: `agg_neighbourhood_monthly_performance.sql`

```sql
WITH listing_agg AS (
    SELECT * FROM {{ ref('agg_listing_monthly_performance') }}
),

listings AS (
    SELECT * FROM {{ ref('dim_listings') }}
)

SELECT
    l.neighbourhood,
    l.neighbourhood_group,
    la.year_month,
    COUNT(DISTINCT la.listing_id) AS total_listings,
    SUM(la.total_estimated_revenue) AS total_estimated_revenue,
    AVG(la.avg_price) AS avg_price,
    AVG(la.availability_rate) AS avg_availability_rate,
    SUM(la.unavailable_nights) AS total_unavailable_nights
FROM listing_agg la
JOIN listings l ON la.listing_id = l.listing_id
GROUP BY 1, 2, 3
```

## dbt Tests

### Generic Tests (`models/staging/schema.yml`)

```yaml
version: 2

models:
  - name: stg_airbnb__listings
    columns:
      - name: listing_id
        tests:
          - not_null
          - unique
      - name: room_type
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      - name: price
        tests:
          - not_null
```

### Singular Tests (`tests/`)

**`tests/assert_no_duplicate_calendar_entries.sql`:**

```sql
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS duplicate_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

**`tests/assert_calendar_listings_exist.sql`:**

```sql
SELECT
    c.listing_id
FROM {{ ref('fct_listing_calendar') }} c
LEFT JOIN {{ ref('dim_listings') }} l
    ON c.listing_id = l.listing_id
WHERE l.listing_id IS NULL
```

## Streamlit Dashboard

### Running the Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

The dashboard reads credentials from `config/local_credentials.json`.

### Dashboard Code Pattern

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
@st.cache_resource
def get_connection():
    with open('config/local_credentials.json') as f:
        creds = json.load(f)
    
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

conn = get_connection()

# Query data
@st.cache_data
def load_neighbourhood_data():
    query = """
        SELECT
            neighbourhood_group,
            neighbourhood,
            total_estimated_revenue,
            total_listings,
            avg_price,
            avg_availability_rate
        FROM agg_neighbourhood_monthly_performance
        WHERE year_month = (SELECT MAX(year_month) FROM agg_neighbourhood_monthly_performance)
        ORDER BY total_estimated_revenue DESC
        LIMIT 20
    """
    return pd.read_sql(query, conn)

df = load_neighbourhood_data()

# Display
st.title("Inside Airbnb Analytics")
st.subheader("Top Neighbourhoods by Estimated Revenue")
st.dataframe(df)
st.bar_chart(df.set_index('neighbourhood')['total_estimated_revenue'])
```

## Common Patterns

### 1. Adding a New Staging Model

```sql
-- models/staging/stg_airbnb__new_source.sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'new_table') }}
)

SELECT
    id::NUMBER AS new_id,
    TRIM(name) AS new_name,
    created_at::TIMESTAMP AS created_at
FROM source
WHERE id IS NOT NULL
```

### 2. Creating a Custom Macro

```sql
-- macros/calculate_revenue.sql
{% macro calculate_revenue(price_column, available_column) %}
    CASE
        WHEN {{ available_column }} = 'f' THEN {{ price_column }}
        ELSE 0
    END
{% endmacro %}
```

**Usage in model:**

```sql
SELECT
    listing_id,
    calendar_date,
    {{ calculate_revenue('price', 'available') }} AS estimated_revenue
FROM {{ ref('stg_airbnb__calendar') }}
```

### 3. Testing Referential Integrity

```sql
-- tests/assert_reviews_have_valid_listings.sql
SELECT
    r.review_id
FROM {{ ref('fct_reviews') }} r
LEFT JOIN {{ ref('dim_listings') }} l
    ON r.listing_id = l.listing_id
WHERE l.listing_id IS NULL
```

### 4. Using dbt Variables

```yaml
# dbt_project.yml
vars:
  min_reviews_threshold: 5
```

```sql
-- models/marts/dim_active_listings.sql
SELECT *
FROM {{ ref('dim_listings') }}
WHERE number_of_reviews >= {{ var('min_reviews_threshold') }}
```

### 5. Snapshot Configuration (if needed)

```sql
-- snapshots/listings_snapshot.sql
{% snapshot listings_snapshot %}

{{
    config(
      target_schema='snapshots',
      unique_key='listing_id',
      strategy='timestamp',
      updated_at='dbt_loaded_at'
    )
}}

SELECT * FROM {{ ref('dim_listings') }}

{% endsnapshot %}
```

## Troubleshooting

### Connection Issues

**Problem:** `dbt debug` fails with authentication error.

**Solution:**
- Verify credentials in `profiles.yml` match your Snowflake account
- Check account identifier format: `<account_locator>.<region>` or `<org>-<account>`
- For SSO, use `authenticator: externalbrowser` instead of password

```yaml
# profiles.yml with SSO
dev:
  type: snowflake
  account: YOUR_ACCOUNT
  user: YOUR_USER
  authenticator: externalbrowser
  role: YOUR_ROLE
  # ... rest of config
```

### Python Loader Fails

**Problem:** `PUT` command fails during file upload.

**Solution:**
- Ensure files exist in `data/raw/` directory
- Check file permissions (readable by current user)
- Verify Snowflake stage exists: `SHOW STAGES;`
- Check warehouse is running and sized appropriately

**Debug pattern:**

```python
import os

# Verify file exists before upload
file_path = 'data/raw/listings.csv.gz'
if not os.path.exists(file_path):
    raise FileNotFoundError(f"Missing {file_path}")

# Check file size
file_size = os.path.getsize(file_path)
print(f"File size: {file_size} bytes")
```

### dbt Model Fails

**Problem:** Model fails with `Object does not exist` error.

**Solution:**
- Check upstream dependencies exist: `dbt run --select +your_model`
- Verify source schema matches `sources.yml` configuration
- Ensure database and schema names are correct in `profiles.yml`

**Debug query:**

```sql
-- Run directly in Snowflake to verify source
SELECT * FROM AIRBNB_DB.RAW.LISTINGS LIMIT 10;
```

### Incremental Model Issues

**Problem:** `fct_listing_calendar` doesn't update with new data.

**Solution:**
- Use `--full-refresh` flag to rebuild: `dbt run --full-refresh --select fct_listing_calendar`
- Check `is_incremental()` logic in model
- Verify unique key doesn't have duplicates

**Debug incremental logic:**

```sql
-- Add to model for debugging
SELECT
    MAX(calendar_date) AS max_existing_date,
    MIN(calendar_date) AS min_existing_date,
    COUNT(*) AS row_count
FROM {{ this }}

{% if is_incremental() %}
-- This block only runs on incremental builds
{% endif %}
```

### Test Failures

**Problem:** Generic test `unique` fails on `listing_id`.

**Solution:**
- Investigate duplicates: `SELECT listing_id, COUNT(*) FROM stg_airbnb__listings GROUP BY 1 HAVING COUNT(*) > 1`
- Fix in staging model with `DISTINCT` or `ROW_NUMBER()`
- Check raw data quality in source CSV

**Deduplication pattern:**

```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'listings') }}
),

deduplicated AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY CURRENT_TIMESTAMP()) AS rn
    FROM source
)

SELECT * FROM deduplicated WHERE rn = 1
```

### Dashboard Connection Error

**Problem:** Streamlit cannot connect to Snowflake.

**Solution:**
- Verify `config/local_credentials.json` exists and has correct values
- Check file is valid JSON (use a validator)
- Ensure Snowflake IP whitelisting allows your connection (if applicable)

**Test connection:**

```python
import snowflake.connector
import json

with open('config/local_credentials.json') as f:
    creds = json.load(f)

try:
    conn = snowflake.connector.connect(**creds)
    print("✓ Connection successful")
    cursor = conn.cursor()
    cursor.execute("SELECT CURRENT_VERSION()")
    print(f"Snowflake version: {cursor.fetchone()[0]}")
    cursor.close()
    conn.close()
except Exception as e:
    print(f"✗ Connection failed: {e}")
```

### Performance Issues

**Problem:** dbt models run slowly.

**Solution:**
- Increase warehouse size in Snowflake
- Use clustering keys on large tables
- Partition calendar data by year/month
- Add `persist_docs` only when needed

**Clustering example:**

```sql
{{
    config(
        materialized='table',
        cluster_by=['calendar_date']
    )
}}

SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
```

## Key Files Reference

| File | Purpose |
|------|---------|
| `dbt_project.yml` | dbt project configuration |
| `profiles.yml` | Snowflake connection (local, git-ignored) |
| `config/local_credentials.json` | Streamlit credentials (local, git-ignored) |
| `scripts/load_inside_airbnb_to_snowflake.py` | Raw data loader |
| `setup/snowflake_setup.sql` | Database/schema/warehouse DDL |
| `models/staging/sources.yml` | Source table definitions |
| `models/staging/schema.yml` | Model tests and documentation |
| `dashboard/streamlit_app.py` | Streamlit reporting app |

## Best Practices

1. **Never commit credentials** — Use local ignored files and reference environment variables in production
2. **Test incrementally** — Run `dbt test --select <model>` after building each model
3. **Document models** — Add descriptions in `schema.yml` for generated docs
4. **Use consistent naming** — Follow `stg_`, `int_`, `dim_`, `fct_`, `agg_` prefixes
5. **Version sources** — Pin source data versions in documentation
6. **Full refresh carefully** — Incremental models can be expensive to rebuild
7. **Monitor warehouse usage** — Suspend warehouses when not in use to control costs

This skill covers the complete workflow for using the Snowflake dbt Airbnb Analytics project, from initial setup through production dashboard deployment.
