---
name: snowflake-dbt-airbnb-analytics
description: Snowflake + dbt analytics engineering project for Inside Airbnb data with staging, intermediate, and mart layers, incremental models, tests, and Streamlit dashboard
triggers:
  - set up Airbnb Snowflake dbt project
  - load Inside Airbnb data to Snowflake
  - create dbt models for Airbnb analytics
  - build incremental fact tables in dbt
  - run dbt tests for data quality
  - deploy Streamlit dashboard for Airbnb data
  - configure dbt profiles for Snowflake
  - troubleshoot dbt Snowflake connection
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill teaches AI coding agents how to work with the **Snowflake_DBT_Project**, a complete analytics engineering pipeline that:

- Loads Inside Airbnb open data into Snowflake via Python scripts
- Transforms raw data through dbt staging, intermediate, and mart layers
- Implements incremental fact modeling with Snowflake merge strategy
- Validates data quality with dbt generic and singular tests
- Powers a Streamlit dashboard with analytics-ready marts

## Architecture Overview

```
Inside Airbnb CSVs → Python loader → Snowflake stage → RAW schema → 
dbt staging → dbt intermediate → dbt marts → Streamlit dashboard
```

**Key layers:**
- **RAW**: Text-preserving raw tables (listings, calendar, reviews, neighbourhoods)
- **Staging**: Clean, cast, standardize (`stg_airbnb__*`)
- **Intermediate**: Joins, enrichment, revenue proxy (`int_airbnb__*`)
- **Marts**: Dimensions (`dim_*`), facts (`fct_*`), aggregates (`agg_*`)

## Installation

```bash
# Clone the repository
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**Expected dependencies (from requirements.txt):**
- `dbt-snowflake`
- `snowflake-connector-python`
- `streamlit`
- `pandas`

## Configuration

### 1. Snowflake Credentials (profiles.yml)

Create `profiles.yml` from the example (this file is git-ignored):

```bash
cp profiles.yml.example profiles.yml
```

Edit `profiles.yml`:

```yaml
snowflake_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: ACCOUNTADMIN
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
```

**Environment variables:**
```bash
export SNOWFLAKE_ACCOUNT="your_account.region"
export SNOWFLAKE_USER="your_username"
export SNOWFLAKE_PASSWORD="your_password"
```

### 2. Python Loader Credentials

Create `config/local_credentials.json` from example:

```bash
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `config/local_credentials.json`:

```json
{
  "account": "your_account.region",
  "user": "your_username",
  "password": "your_password",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "schema": "RAW",
  "role": "ACCOUNTADMIN"
}
```

**This file is git-ignored. Never commit credentials.**

### 3. Raw Data Files

Download Inside Airbnb data for New York City from [insideairbnb.com](https://insideairbnb.com/get-the-data/) and place in `data/raw/`:

```
data/raw/
├── listings.csv.gz
├── calendar.csv.gz
├── reviews.csv.gz
└── neighbourhoods.csv
```

## Loading Raw Data to Snowflake

The Python loader script creates Snowflake objects and uploads raw files:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What this script does:**
1. Connects to Snowflake using `config/local_credentials.json`
2. Executes `setup/snowflake_setup.sql` to create database, schemas, stage
3. Uploads local CSV/GZIP files to `INSIDE_AIRBNB_STAGE` internal stage
4. Creates raw tables from CSV headers
5. Copies data into `RAW.LISTINGS`, `RAW.CALENDAR`, `RAW.REVIEWS`, `RAW.NEIGHBOURHOODS`

**Example loader pattern:**

```python
import snowflake.connector
import json

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

# Connect to Snowflake
conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    warehouse=creds['warehouse'],
    database=creds['database'],
    schema=creds['schema'],
    role=creds['role']
)

cursor = conn.cursor()

# Upload file to stage
cursor.execute("""
    PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE/listings/
    AUTO_COMPRESS=FALSE
    OVERWRITE=TRUE
""")

# Create raw table from file
cursor.execute("""
    CREATE OR REPLACE TABLE RAW.LISTINGS
    USING TEMPLATE (
        SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
        FROM TABLE(
            INFER_SCHEMA(
                LOCATION=>'@INSIDE_AIRBNB_STAGE/listings/',
                FILE_FORMAT=>'CSV_FORMAT'
            )
        )
    )
""")

# Copy data into table
cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings/
    FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
    ON_ERROR = CONTINUE
""")

cursor.close()
conn.close()
```

## dbt Commands

### Test Connection

```bash
dbt debug --profiles-dir .
```

### Run All Models

```bash
dbt run --profiles-dir .
```

### Run Specific Model and Downstream

```bash
dbt run --select dim_listings+ --profiles-dir .
```

### Run Tests

```bash
dbt test --profiles-dir .
```

### Full Refresh Incremental Models

```bash
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .
```

### Generate Documentation

```bash
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

## dbt Model Structure

### Staging Models

**Location:** `models/staging/`

**Purpose:** Clean, cast, and standardize raw data.

**Example:** `models/staging/stg_airbnb__listings.sql`

```sql
{{ config(materialized='view') }}

SELECT
    id::BIGINT AS listing_id,
    name::VARCHAR AS listing_name,
    host_id::BIGINT AS host_id,
    host_name::VARCHAR AS host_name,
    neighbourhood_cleansed::VARCHAR AS neighbourhood,
    room_type::VARCHAR AS room_type,
    price::VARCHAR AS price_text,
    minimum_nights::INTEGER AS minimum_nights,
    number_of_reviews::INTEGER AS number_of_reviews,
    last_review::DATE AS last_review_date,
    reviews_per_month::FLOAT AS reviews_per_month,
    calculated_host_listings_count::INTEGER AS host_total_listings_count,
    availability_365::INTEGER AS availability_365,
    CURRENT_TIMESTAMP AS _loaded_at
FROM {{ source('raw_airbnb', 'listings') }}
WHERE id IS NOT NULL
```

**Key patterns:**
- `{{ config(materialized='view') }}` for lightweight staging
- `{{ source('raw_airbnb', 'listings') }}` references `sources.yml`
- Type casting with `::` Snowflake syntax
- Aliasing to standard column names

### Intermediate Models

**Location:** `models/intermediate/`

**Purpose:** Reusable joins, enrichment, business logic.

**Example:** `models/intermediate/int_airbnb__listing_enriched.sql`

```sql
{{ config(materialized='view') }}

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
    REPLACE(REPLACE(l.price_text, '$', ''), ',', '')::FLOAT AS price,
    l.minimum_nights,
    l.number_of_reviews,
    l.last_review_date,
    l.reviews_per_month,
    l.host_total_listings_count,
    l.availability_365,
    l._loaded_at
FROM listings l
LEFT JOIN neighbourhoods n
    ON l.neighbourhood = n.neighbourhood
```

**Key patterns:**
- CTEs for readability
- `{{ ref('stg_airbnb__listings') }}` for model dependencies
- Price cleaning logic (remove $, commas)
- Neighbourhood enrichment join

### Mart Dimension Models

**Location:** `models/marts/dimensions/`

**Example:** `models/marts/dimensions/dim_listings.sql`

```sql
{{ config(
    materialized='table',
    unique_key='listing_id'
) }}

SELECT
    listing_id,
    listing_name,
    host_id,
    host_name,
    neighbourhood,
    neighbourhood_group,
    room_type,
    price,
    minimum_nights,
    host_total_listings_count,
    availability_365,
    _loaded_at
FROM {{ ref('int_airbnb__listing_enriched') }}
```

**Key patterns:**
- `materialized='table'` for performance
- `unique_key` for testing
- Select only dimension attributes (no measures)

### Mart Fact Models (Incremental)

**Location:** `models/marts/facts/`

**Example:** `models/marts/facts/fct_listing_calendar.sql`

```sql
{{ config(
    materialized='incremental',
    unique_key=['listing_id', 'calendar_date'],
    merge_update_columns=['available', 'price', 'minimum_nights', 'maximum_nights']
) }}

SELECT
    listing_id,
    calendar_date,
    available,
    price,
    minimum_nights,
    maximum_nights,
    neighbourhood,
    room_type,
    CASE 
        WHEN available = FALSE THEN price
        ELSE 0
    END AS estimated_revenue,
    CURRENT_TIMESTAMP AS _loaded_at
FROM {{ ref('int_airbnb__calendar_enriched') }}

{% if is_incremental() %}
WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

**Key patterns:**
- `materialized='incremental'` for large fact tables
- Composite `unique_key=['listing_id', 'calendar_date']`
- `merge_update_columns` for Snowflake merge strategy
- `{% if is_incremental() %}` filter for performance
- Revenue proxy: unavailable nights × price

### Aggregate Models

**Location:** `models/marts/aggregates/`

**Example:** `models/marts/aggregates/agg_listing_monthly_performance.sql`

```sql
{{ config(materialized='table') }}

SELECT
    listing_id,
    neighbourhood,
    room_type,
    DATE_TRUNC('MONTH', calendar_date) AS month,
    COUNT(*) AS total_days,
    SUM(CASE WHEN available = FALSE THEN 1 ELSE 0 END) AS unavailable_days,
    AVG(price) AS avg_price,
    SUM(estimated_revenue) AS total_estimated_revenue,
    ROUND(SUM(CASE WHEN available = FALSE THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS unavailability_rate
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, neighbourhood, room_type, DATE_TRUNC('MONTH', calendar_date)
```

**Key patterns:**
- `DATE_TRUNC('MONTH', ...)` for monthly grain
- Aggregation metrics: counts, sums, averages, percentages
- Dashboard-ready output

## dbt Sources Configuration

**File:** `models/staging/sources.yml`

```yaml
version: 2

sources:
  - name: raw_airbnb
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: listings
        description: Raw Inside Airbnb listings data
        columns:
          - name: id
            description: Unique listing identifier
            tests:
              - not_null
              - unique

      - name: calendar
        description: Raw calendar availability and pricing
        columns:
          - name: listing_id
            tests:
              - not_null
          - name: date
            tests:
              - not_null

      - name: reviews
        description: Raw guest reviews
        columns:
          - name: listing_id
            tests:
              - not_null
          - name: id
            tests:
              - not_null
              - unique

      - name: neighbourhoods
        description: Neighbourhood reference data
```

## dbt Tests

### Generic Tests (in schema.yml)

```yaml
version: 2

models:
  - name: dim_listings
    description: Listing dimension table
    columns:
      - name: listing_id
        description: Unique listing identifier
        tests:
          - not_null
          - unique

      - name: room_type
        description: Type of room offered
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']

      - name: price
        description: Nightly price in dollars
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### Singular Tests (custom SQL)

**File:** `tests/fct_listing_calendar_no_duplicates.sql`

```sql
-- Check for duplicate listing-date combinations
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS duplicate_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

**File:** `tests/fct_listing_calendar_valid_listing_refs.sql`

```sql
-- Check all listing_ids exist in dim_listings
SELECT
    fct.listing_id
FROM {{ ref('fct_listing_calendar') }} fct
LEFT JOIN {{ ref('dim_listings') }} dim
    ON fct.listing_id = dim.listing_id
WHERE dim.listing_id IS NULL
```

## Streamlit Dashboard

**File:** `dashboard/streamlit_app.py`

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)

# Connect to Snowflake
@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        role=creds['role']
    )

conn = get_connection()

# Query function
@st.cache_data
def run_query(query):
    with conn.cursor() as cur:
        cur.execute(query)
        df = cur.fetch_pandas_all()
    return df

# Dashboard title
st.title("Inside Airbnb Analytics Dashboard")

# Neighbourhood performance
st.header("Top 10 Neighbourhoods by Estimated Revenue")
query = """
    SELECT
        neighbourhood,
        SUM(total_estimated_revenue) AS total_revenue,
        AVG(avg_price) AS avg_price,
        AVG(unavailability_rate) AS avg_unavailability_rate
    FROM ANALYTICS.agg_neighbourhood_monthly_performance
    GROUP BY neighbourhood
    ORDER BY total_revenue DESC
    LIMIT 10
"""
df = run_query(query)
st.dataframe(df)
st.bar_chart(df.set_index('NEIGHBOURHOOD')['TOTAL_REVENUE'])

# Room type analysis
st.header("Average Price by Room Type")
query = """
    SELECT
        room_type,
        AVG(price) AS avg_price
    FROM ANALYTICS.dim_listings
    GROUP BY room_type
    ORDER BY avg_price DESC
"""
df = run_query(query)
st.bar_chart(df.set_index('ROOM_TYPE')['AVG_PRICE'])

# Top hosts
st.header("Top 10 Hosts by Listing Count")
query = """
    SELECT
        host_name,
        host_total_listings_count
    FROM ANALYTICS.dim_hosts
    ORDER BY host_total_listings_count DESC
    LIMIT 10
"""
df = run_query(query)
st.dataframe(df)
```

**Run dashboard:**

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Workflows

### Initial Setup (Fresh Project)

```bash
# 1. Install dependencies
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 2. Configure credentials
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
# Edit both files with your Snowflake credentials

# 3. Download raw data to data/raw/

# 4. Load raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 5. Run dbt
dbt debug --profiles-dir .
dbt run --profiles-dir .
dbt test --profiles-dir .

# 6. Launch dashboard
streamlit run dashboard/streamlit_app.py
```

### Refresh Data (New Snapshot)

```bash
# 1. Download new Inside Airbnb files to data/raw/

# 2. Reload raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run tests
dbt test --profiles-dir .
```

### Develop New Model

```bash
# 1. Create model SQL file in models/
# Example: models/marts/aggregates/agg_my_metric.sql

# 2. Run just that model
dbt run --select agg_my_metric --profiles-dir .

# 3. Test the output
dbt run --select agg_my_metric --profiles-dir . --store-failures

# 4. Add to lineage
dbt run --select +agg_my_metric --profiles-dir .

# 5. Document in schema.yml and add tests
```

### Add Custom Test

**Singular test:** `tests/my_custom_test.sql`

```sql
-- Fail if any records returned
SELECT
    listing_id,
    COUNT(*) as issue_count
FROM {{ ref('fct_listing_calendar') }}
WHERE price < 0
GROUP BY listing_id
```

**Run:**

```bash
dbt test --select my_custom_test --profiles-dir .
```

## Troubleshooting

### "Connection refused" or "Incorrect username or password"

**Fix:** Verify credentials in `profiles.yml` or environment variables:

```bash
export SNOWFLAKE_ACCOUNT="abc12345.us-east-1"  # Include region
export SNOWFLAKE_USER="YOUR_USERNAME"
export SNOWFLAKE_PASSWORD="YOUR_PASSWORD"

dbt debug --profiles-dir .
```

### "Database 'AIRBNB_DB' does not exist"

**Fix:** Run Snowflake setup SQL first:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

Or manually execute `setup/snowflake_setup.sql` in Snowflake UI.

### "Cannot find module 'snowflake.connector'"

**Fix:** Install dependencies:

```bash
pip install -r requirements.txt
```

### Incremental model not updating

**Fix:** Full refresh:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### "Compilation Error: depends on a node named 'X' which was not found"

**Fix:** Check model references. Ensure `{{ ref('stg_airbnb__listings') }}` matches file name exactly.

```bash
dbt deps --profiles-dir .
dbt compile --profiles-dir .
```

### Tests failing on price format

**Cause:** Inside Airbnb price column has `$` and `,` characters.

**Fix:** Already handled in `int_airbnb__listing_enriched.sql`:

```sql
REPLACE(REPLACE(l.price_text, '$', ''), ',', '')::FLOAT AS price
```

### Dashboard connection error

**Fix:** Verify `config/local_credentials.json` exists and has correct values:

```json
{
  "account": "abc12345.us-east-1",
  "user": "YOUR_USERNAME",
  "password": "YOUR_PASSWORD",
  "warehouse": "COMPUTE_WH",
  "database": "AIRBNB_DB",
  "role": "ACCOUNTADMIN"
}
```

### dbt cannot find profiles.yml

**Fix:** Use `--profiles-dir .` to point to current directory:

```bash
dbt run --profiles-dir .
```

Or move `profiles.yml` to `~/.dbt/profiles.yml` (default location).

## Key Files Reference

| File | Purpose |
|------|---------|
| `dbt_project.yml` | dbt project configuration, model paths, materializations |
| `profiles.yml` | Snowflake connection credentials (git-ignored) |
| `config/local_credentials.json` | Python script + dashboard credentials (git-ignored) |
| `setup/snowflake_setup.sql` | Snowflake database, schema, stage, file format DDL |
| `scripts/load_inside_airbnb_to_snowflake.py` | Python loader script |
| `models/staging/sources.yml` | Raw source definitions and tests |
| `models/staging/schema.yml` | Staging model tests and documentation |
| `models/marts/schema.yml` | Mart model tests and documentation |
| `dashboard/streamlit_app.py` | Streamlit reporting dashboard |

## Security Best Practices

**Never commit:**
- `profiles.yml`
- `config/local_credentials.json`
- `.user.yml`
- `target/` (dbt artifacts)
- `logs/` (dbt logs)
- `.venv/` (Python virtual environment)

**Already in .gitignore:**
```
profiles.yml
config/local_credentials.json
.user.yml
target/
dbt_packages/
logs/
.venv/
```

**Use environment variables in production:**

```yaml
# profiles.yml
account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
user: "{{ env_var('SNOWFLAKE_USER') }}"
password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
```

This skill enables AI coding agents to guide developers through setting up, configuring, running, testing, and troubleshooting the complete Snowflake + dbt + Streamlit analytics pipeline for Inside Airbnb data.
