---
name: snowflake-dbt-airbnb-analytics
description: Snowflake + dbt analytics engineering project for Inside Airbnb data with staging, intermediate, and mart layers, incremental models, tests, and Streamlit dashboard
triggers:
  - set up Airbnb analytics project with dbt and Snowflake
  - load Inside Airbnb data into Snowflake
  - build dbt marts for Airbnb listings and calendar data
  - create incremental fact table in dbt with Snowflake merge
  - run dbt tests for Airbnb data quality
  - build Streamlit dashboard on top of dbt marts
  - configure dbt profiles for Snowflake analytics project
  - implement analytics engineering layers with dbt
---

# Snowflake dbt Airbnb Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete analytics engineering workflow: loading Inside Airbnb open data into Snowflake, transforming it through dbt staging/intermediate/mart layers, validating data quality with tests, and visualizing results in a Streamlit dashboard.

## What This Project Does

- **Raw ingestion**: Python script loads CSV/GZIP files into Snowflake internal stages
- **dbt transformations**: Staging → intermediate → marts (dimensions, facts, aggregates)
- **Incremental modeling**: `fct_listing_calendar` uses Snowflake merge strategy
- **Data quality**: Generic and singular tests for primary keys, relationships, accepted values
- **Reporting**: Streamlit dashboard queries mart tables for neighbourhood/host/listing analytics

## Dataset

Uses [Inside Airbnb](https://insideairbnb.com/get-the-data/) open data. Recommended: New York City dataset.

Expected files in `data/raw/`:
- `listings.csv.gz`
- `calendar.csv.gz`
- `reviews.csv.gz`
- `neighbourhoods.csv`

## Installation

```bash
# Clone and set up virtual environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Create local credential files (ignored by git)
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `profiles.yml` with your Snowflake credentials:

```yaml
snowflake_dbt_airbnb:
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USER
      password: YOUR_PASSWORD
      role: YOUR_ROLE
      database: AIRBNB_ANALYTICS
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
  target: dev
```

Edit `config/local_credentials.json`:

```json
{
  "account": "YOUR_ACCOUNT",
  "user": "YOUR_USER",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_ANALYTICS",
  "schema": "ANALYTICS",
  "role": "YOUR_ROLE"
}
```

**Security**: Never commit these files. Use environment variables in production:

```python
import os
config = {
    "account": os.getenv("SNOWFLAKE_ACCOUNT"),
    "user": os.getenv("SNOWFLAKE_USER"),
    "password": os.getenv("SNOWFLAKE_PASSWORD"),
    # ...
}
```

## Load Raw Data

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

This script:
1. Runs `setup/snowflake_setup.sql` to create database/schemas/stage
2. Uploads local CSV files to Snowflake internal stage
3. Creates raw tables with text columns (preserves data exactly as-is)
4. Copies data from stage into `RAW` schema tables

Key functions from `load_inside_airbnb_to_snowflake.py`:

```python
import snowflake.connector
import json

def get_snowflake_connection():
    with open('config/local_credentials.json') as f:
        creds = json.load(f)
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        role=creds['role']
    )

def upload_file_to_stage(cursor, local_path, stage_name):
    """Upload local CSV/GZIP to Snowflake internal stage"""
    cursor.execute(f"PUT file://{local_path} @{stage_name} AUTO_COMPRESS=FALSE OVERWRITE=TRUE")

def create_raw_table_from_stage(cursor, table_name, stage_file):
    """Infer schema from CSV and create raw table with all VARCHAR columns"""
    cursor.execute(f"""
        CREATE OR REPLACE TABLE RAW.{table_name} 
        USING TEMPLATE (
            SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
            FROM TABLE(INFER_SCHEMA(
                LOCATION=>'@INSIDE_AIRBNB_STAGE/{stage_file}',
                FILE_FORMAT=>'CSV_FORMAT'
            ))
        )
    """)
```

## dbt Model Layers

### Staging (`models/staging/`)

Clean, cast, and standardize raw data. Views only.

```sql
-- models/staging/stg_airbnb__listings.sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'listings') }}
)
SELECT
    TRY_CAST(id AS INTEGER) AS listing_id,
    TRIM(name) AS listing_name,
    TRY_CAST(host_id AS INTEGER) AS host_id,
    TRIM(host_name) AS host_name,
    TRIM(neighbourhood) AS neighbourhood,
    TRIM(room_type) AS room_type,
    TRY_CAST(price AS DECIMAL(10,2)) AS price,
    TRY_CAST(minimum_nights AS INTEGER) AS minimum_nights,
    TRY_CAST(number_of_reviews AS INTEGER) AS number_of_reviews,
    TRY_TO_DATE(last_review) AS last_review_date,
    TRY_CAST(reviews_per_month AS DECIMAL(10,2)) AS reviews_per_month,
    TRY_CAST(availability_365 AS INTEGER) AS availability_365
FROM source
```

### Intermediate (`models/intermediate/`)

Join and enrich staging models. Add business logic.

```sql
-- models/intermediate/int_airbnb__calendar_enriched.sql
WITH calendar AS (
    SELECT * FROM {{ ref('stg_airbnb__calendar') }}
),
listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
)
SELECT
    calendar.listing_id,
    calendar.calendar_date,
    calendar.is_available,
    calendar.price AS calendar_price,
    
    -- Use calendar price, fall back to listing price
    COALESCE(calendar.price, listings.price) AS effective_price,
    
    -- Revenue proxy: if unavailable, assume booked
    CASE 
        WHEN calendar.is_available = FALSE 
        THEN COALESCE(calendar.price, listings.price, 0)
        ELSE 0 
    END AS estimated_revenue,
    
    -- Enrich with listing attributes
    listings.listing_name,
    listings.host_id,
    listings.neighbourhood,
    listings.room_type,
    listings.minimum_nights
FROM calendar
LEFT JOIN listings 
    ON calendar.listing_id = listings.listing_id
```

### Marts (`models/marts/`)

Analytics-ready dimensions and facts.

**Incremental fact table with Snowflake merge:**

```sql
-- models/marts/fct_listing_calendar.sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['is_available', 'effective_price', 'estimated_revenue']
    )
}}

WITH calendar_enriched AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
)
SELECT
    listing_id,
    calendar_date,
    is_available,
    effective_price,
    estimated_revenue,
    neighbourhood,
    room_type
FROM calendar_enriched

{% if is_incremental() %}
    WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**Monthly aggregate mart:**

```sql
-- models/marts/agg_listing_monthly_performance.sql
WITH daily_calendar AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
)
SELECT
    listing_id,
    DATE_TRUNC('month', calendar_date) AS month_date,
    COUNT(*) AS total_days,
    SUM(CASE WHEN is_available THEN 1 ELSE 0 END) AS available_days,
    SUM(CASE WHEN NOT is_available THEN 1 ELSE 0 END) AS unavailable_days,
    ROUND(AVG(effective_price), 2) AS avg_daily_price,
    ROUND(SUM(estimated_revenue), 2) AS total_estimated_revenue,
    ROUND(100.0 * SUM(CASE WHEN is_available THEN 1 ELSE 0 END) / COUNT(*), 1) AS availability_pct
FROM daily_calendar
GROUP BY listing_id, DATE_TRUNC('month', calendar_date)
```

## dbt Commands

```bash
# Run all models
dbt run --profiles-dir .

# Run only marts
dbt run --select marts --profiles-dir .

# Full refresh incremental model
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select int_airbnb__calendar_enriched+ --profiles-dir .

# Generate and serve documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .

# Debug connection
dbt debug --profiles-dir .
```

## dbt Tests

**Generic tests in schema.yml:**

```yaml
# models/staging/schema.yml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_ANALYTICS
    schema: RAW
    tables:
      - name: listings
        columns:
          - name: id
            tests:
              - not_null
              - unique

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

**Singular test:**

```sql
-- tests/assert_no_duplicate_calendar_dates.sql
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS occurrences
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

Run with `dbt test --profiles-dir .`

## Streamlit Dashboard

```python
# dashboard/streamlit_app.py
import streamlit as st
import snowflake.connector
import pandas as pd
import json

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

def run_query(query):
    conn = get_connection()
    return pd.read_sql(query, conn)

st.title("Airbnb Analytics Dashboard")

# Neighbourhood revenue
st.subheader("Top Neighbourhoods by Estimated Revenue")
neighbourhood_query = """
    SELECT
        neighbourhood,
        SUM(total_estimated_revenue) AS total_revenue,
        AVG(availability_pct) AS avg_availability_pct
    FROM ANALYTICS.agg_neighbourhood_monthly_performance
    GROUP BY neighbourhood
    ORDER BY total_revenue DESC
    LIMIT 10
"""
df_neighbourhood = run_query(neighbourhood_query)
st.bar_chart(df_neighbourhood.set_index('neighbourhood')['total_revenue'])

# Host performance
st.subheader("Top Hosts by Listing Count")
host_query = """
    SELECT
        host_id,
        host_name,
        total_listings,
        avg_price
    FROM ANALYTICS.dim_hosts
    ORDER BY total_listings DESC
    LIMIT 10
"""
df_hosts = run_query(host_query)
st.dataframe(df_hosts)
```

Run with:

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Full refresh after new data load

```bash
python scripts/load_inside_airbnb_to_snowflake.py
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .
dbt test --profiles-dir .
```

### Build only specific layer

```bash
# Just staging
dbt run --select staging --profiles-dir .

# Intermediate and downstream
dbt run --select intermediate+ --profiles-dir .
```

### Custom macro for revenue proxy

```sql
-- macros/calculate_revenue.sql
{% macro calculate_revenue(is_available_column, price_column) %}
    CASE 
        WHEN {{ is_available_column }} = FALSE 
        THEN {{ price_column }}
        ELSE 0 
    END
{% endmacro %}

-- Use in model:
{{ calculate_revenue('is_available', 'effective_price') }} AS estimated_revenue
```

### dbt source freshness check

```yaml
# models/staging/schema.yml
sources:
  - name: airbnb_raw
    freshness:
      warn_after: {count: 24, period: hour}
      error_after: {count: 48, period: hour}
    loaded_at_field: _loaded_at
    tables:
      - name: listings
```

Run: `dbt source freshness --profiles-dir .`

## Troubleshooting

**Connection issues:**

```bash
dbt debug --profiles-dir .
```

Check:
- `profiles.yml` exists in project root (not `~/.dbt/`)
- Snowflake account format: `abc12345.us-east-1` (not URL)
- Warehouse is running
- Role has access to database/schema

**Incremental model not updating:**

Force full refresh:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

**Stage file upload fails:**

Check file paths in loader script:

```python
# Paths must be absolute or relative to script location
local_path = os.path.abspath('data/raw/listings.csv.gz')
```

**Tests failing on accepted_values:**

Check actual values:

```sql
SELECT DISTINCT room_type FROM RAW.listings;
```

Update test in `schema.yml` with actual values.

**Streamlit credential errors:**

Verify `config/local_credentials.json` exists and matches `profiles.yml` values.

## Key Files

- `dbt_project.yml`: Project config, model paths, materializations
- `models/staging/schema.yml`: Source and model tests, freshness
- `models/marts/schema.yml`: Mart model documentation
- `scripts/load_inside_airbnb_to_snowflake.py`: Raw data loader
- `setup/snowflake_setup.sql`: Database/schema/stage DDL
- `dashboard/streamlit_app.py`: Dashboard app

## Project Structure

```
models/
├── staging/          # stg_airbnb__* (views)
├── intermediate/     # int_airbnb__* (views with joins)
└── marts/           # dim_*, fct_*, agg_* (tables/incremental)
```

Naming convention:
- Staging: `stg_<source>__<entity>`
- Intermediate: `int_<source>__<entity>_<suffix>`
- Marts: `dim_<entity>`, `fct_<entity>`, `agg_<entity>_<grain>`
