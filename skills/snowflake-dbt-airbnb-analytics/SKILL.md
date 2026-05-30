---
name: snowflake-dbt-airbnb-analytics
description: Build a complete Snowflake + dbt analytics pipeline with Inside Airbnb data, incremental models, tests, and Streamlit dashboard
triggers:
  - set up snowflake dbt airbnb project
  - create dbt models for inside airbnb data
  - load airbnb data into snowflake with dbt
  - build analytics pipeline for airbnb listings
  - configure dbt staging and mart layers
  - create incremental dbt models in snowflake
  - build streamlit dashboard for airbnb analytics
  - test dbt models for data quality
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build a complete analytics engineering pipeline using Snowflake, dbt, and Inside Airbnb data. The project demonstrates staging, intermediate, and mart model layers, incremental fact tables, data quality tests, and a Streamlit dashboard.

## What This Project Does

The `Snowflake_DBT_Project` loads Inside Airbnb open data (listings, calendar, reviews, neighbourhoods) into Snowflake, transforms it through dbt staging/intermediate/mart layers, validates data quality with tests, and powers a Streamlit reporting dashboard. It demonstrates production-grade analytics engineering patterns including incremental models with merge strategy, dimensional modeling, and revenue proxy calculations.

## Project Architecture

**Data Flow:**
1. Inside Airbnb CSV/GZIP files → Python loader
2. Snowflake internal stage → RAW schema tables
3. dbt staging views (clean/cast)
4. dbt intermediate layer (joins/enrichment)
5. dbt marts (dimensions/facts/aggregates)
6. Streamlit dashboard (visualization)

**dbt Layers:**
- **Sources**: `RAW.LISTINGS`, `RAW.CALENDAR`, `RAW.REVIEWS`, `RAW.NEIGHBOURHOODS`
- **Staging**: `stg_airbnb__*` (cleaning, type casting)
- **Intermediate**: `int_airbnb__*` (reusable joins, revenue logic)
- **Marts**: `dim_listings`, `dim_hosts`, `fct_listing_calendar` (incremental), `fct_reviews`
- **Aggregates**: `agg_listing_monthly_performance`, `agg_neighbourhood_monthly_performance`

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

## Configuration

### 1. Set Up Credentials (Local-Only, Not Committed)

```bash
# Copy credential templates
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

**Edit `profiles.yml`:**
```yaml
snowflake_dbt_airbnb:
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: "{{ env_var('SNOWFLAKE_ROLE', 'AIRBNB_ANALYTICS_ROLE') }}"
      warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE', 'AIRBNB_ANALYTICS_WH') }}"
      database: "{{ env_var('SNOWFLAKE_DATABASE', 'AIRBNB_ANALYTICS') }}"
      schema: ANALYTICS
      threads: 4
  target: dev
```

**Edit `config/local_credentials.json`:**
```json
{
  "account": "your_account",
  "user": "your_user",
  "password": "your_password",
  "warehouse": "AIRBNB_ANALYTICS_WH",
  "database": "AIRBNB_ANALYTICS",
  "schema": "ANALYTICS",
  "role": "AIRBNB_ANALYTICS_ROLE"
}
```

**Alternative: Use environment variables:**
```bash
export SNOWFLAKE_ACCOUNT=your_account
export SNOWFLAKE_USER=your_user
export SNOWFLAKE_PASSWORD=your_password
export SNOWFLAKE_ROLE=AIRBNB_ANALYTICS_ROLE
export SNOWFLAKE_WAREHOUSE=AIRBNB_ANALYTICS_WH
export SNOWFLAKE_DATABASE=AIRBNB_ANALYTICS
```

### 2. Download Inside Airbnb Data

Download from [Inside Airbnb](https://insideairbnb.com/get-the-data/) (recommended: New York City):

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Key Commands

### Load Raw Data into Snowflake

```bash
# Run the Python loader (creates stage, uploads files, creates raw tables)
python scripts/load_inside_airbnb_to_snowflake.py
```

**What this does:**
- Executes `setup/snowflake_setup.sql` to create database/schema/stage
- Uploads CSV/GZIP files to Snowflake internal stage
- Creates raw tables by inferring schema from CSV headers
- Copies data from stage into raw tables

### dbt Commands

```bash
# Verify dbt connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Run only marts layer
dbt run --select marts --profiles-dir .

# Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Run tests for specific model
dbt test --select stg_airbnb__listings --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation locally
dbt docs serve --profiles-dir .
```

### Run Streamlit Dashboard

```bash
streamlit run dashboard/streamlit_app.py
```

## Code Examples

### Example 1: Create a Staging Model

**File: `models/staging/stg_airbnb__listings.sql`**

```sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'listings') }}
),

cleaned AS (
    SELECT
        TRY_CAST(id AS INTEGER) AS listing_id,
        TRIM(name) AS listing_name,
        TRY_CAST(host_id AS INTEGER) AS host_id,
        TRIM(host_name) AS host_name,
        TRIM(neighbourhood) AS neighbourhood,
        TRIM(neighbourhood_group) AS neighbourhood_group,
        UPPER(TRIM(room_type)) AS room_type,
        TRY_CAST(REPLACE(price, '$', '') AS DECIMAL(10,2)) AS price,
        TRY_CAST(minimum_nights AS INTEGER) AS minimum_nights,
        TRY_CAST(number_of_reviews AS INTEGER) AS number_of_reviews,
        TRY_TO_DATE(last_review) AS last_review_date,
        TRY_CAST(reviews_per_month AS DECIMAL(10,2)) AS reviews_per_month,
        TRY_CAST(calculated_host_listings_count AS INTEGER) AS host_listings_count,
        TRY_CAST(availability_365 AS INTEGER) AS availability_365,
        TRY_CAST(latitude AS DECIMAL(10,8)) AS latitude,
        TRY_CAST(longitude AS DECIMAL(11,8)) AS longitude,
        CURRENT_TIMESTAMP() AS dbt_loaded_at
    FROM source
)

SELECT * FROM cleaned
WHERE listing_id IS NOT NULL
```

**Config block at top of file:**
```sql
{{
    config(
        materialized='view',
        schema='staging'
    )
}}
```

### Example 2: Create an Incremental Fact Table

**File: `models/marts/fct_listing_calendar.sql`**

```sql
{{
    config(
        materialized='incremental',
        unique_key=['listing_id', 'calendar_date'],
        merge_update_columns=['available', 'price', 'minimum_nights', 'maximum_nights']
    )
}}

WITH calendar_enriched AS (
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
    listing_name,
    host_id,
    neighbourhood,
    room_type,
    dbt_loaded_at
FROM calendar_enriched

{% if is_incremental() %}
    WHERE calendar_date >= (SELECT MAX(calendar_date) FROM {{ this }})
{% endif %}
```

### Example 3: Create an Intermediate Model with Revenue Logic

**File: `models/intermediate/int_airbnb__calendar_enriched.sql`**

```sql
WITH calendar AS (
    SELECT * FROM {{ ref('stg_airbnb__calendar') }}
),

listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
),

calendar_with_listing AS (
    SELECT
        c.listing_id,
        c.calendar_date,
        c.available,
        c.price,
        c.adjusted_price,
        c.minimum_nights,
        c.maximum_nights,
        l.listing_name,
        l.host_id,
        l.host_name,
        l.neighbourhood,
        l.neighbourhood_group,
        l.room_type,
        -- Revenue proxy: if not available, assume booked
        CASE
            WHEN c.available = FALSE THEN COALESCE(c.adjusted_price, c.price, 0)
            ELSE 0
        END AS estimated_revenue,
        c.dbt_loaded_at
    FROM calendar c
    INNER JOIN listings l
        ON c.listing_id = l.listing_id
)

SELECT * FROM calendar_with_listing
```

### Example 4: Create Monthly Aggregate

**File: `models/marts/agg_listing_monthly_performance.sql`**

```sql
WITH calendar_facts AS (
    SELECT * FROM {{ ref('fct_listing_calendar') }}
),

monthly_metrics AS (
    SELECT
        listing_id,
        listing_name,
        host_id,
        neighbourhood,
        neighbourhood_group,
        room_type,
        DATE_TRUNC('MONTH', calendar_date) AS month_start,
        COUNT(*) AS days_in_month,
        SUM(CASE WHEN available = FALSE THEN 1 ELSE 0 END) AS booked_days,
        SUM(CASE WHEN available = TRUE THEN 1 ELSE 0 END) AS available_days,
        AVG(price) AS avg_price,
        SUM(estimated_revenue) AS total_estimated_revenue,
        ROUND(100.0 * SUM(CASE WHEN available = FALSE THEN 1 ELSE 0 END) / COUNT(*), 2) AS occupancy_rate
    FROM calendar_facts
    GROUP BY 1, 2, 3, 4, 5, 6, 7
)

SELECT * FROM monthly_metrics
ORDER BY month_start DESC, total_estimated_revenue DESC
```

### Example 5: Add Data Quality Tests

**File: `models/staging/schema.yml`**

```yaml
version: 2

models:
  - name: stg_airbnb__listings
    description: Cleaned and standardized Airbnb listings
    columns:
      - name: listing_id
        description: Primary key for listings
        tests:
          - unique
          - not_null
      - name: host_id
        description: Host identifier
        tests:
          - not_null
      - name: room_type
        description: Type of room offered
        tests:
          - accepted_values:
              values: ['ENTIRE HOME/APT', 'PRIVATE ROOM', 'SHARED ROOM', 'HOTEL ROOM']
      - name: price
        description: Listing price per night
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### Example 6: Custom Singular Test

**File: `tests/assert_no_duplicate_listing_dates.sql`**

```sql
-- Check for duplicate listing-date combinations in calendar fact
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS occurrence_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

### Example 7: Streamlit Dashboard Integration

**File: `dashboard/streamlit_app.py`**

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials from local file (not committed)
with open('config/local_credentials.json') as f:
    creds = json.load(f)

@st.cache_resource
def get_snowflake_connection():
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
    conn = get_snowflake_connection()
    return pd.read_sql(query, conn)

st.title("Airbnb Analytics Dashboard")

# Top neighbourhoods by revenue
st.header("Top 10 Neighbourhoods by Estimated Revenue")
query = """
SELECT
    neighbourhood,
    SUM(total_estimated_revenue) AS total_revenue,
    AVG(occupancy_rate) AS avg_occupancy_rate
FROM ANALYTICS.agg_neighbourhood_monthly_performance
GROUP BY neighbourhood
ORDER BY total_revenue DESC
LIMIT 10
"""
df = run_query(query)
st.bar_chart(df.set_index('neighbourhood')['total_revenue'])

# Room type distribution
st.header("Average Price by Room Type")
query = """
SELECT
    room_type,
    AVG(avg_price) AS avg_price,
    COUNT(DISTINCT listing_id) AS listing_count
FROM ANALYTICS.agg_listing_monthly_performance
GROUP BY room_type
ORDER BY avg_price DESC
"""
df = run_query(query)
st.dataframe(df)
```

## Common Patterns

### Full Refresh Workflow for New Data Snapshot

```bash
# 1. Download new Inside Airbnb data to data/raw/
# 2. Reload raw tables
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Rebuild aggregates downstream
dbt run --select agg_listing_monthly_performance+ --profiles-dir .

# 5. Run tests
dbt test --profiles-dir .
```

### Selective Model Runs

```bash
# Run only staging models
dbt run --select staging --profiles-dir .

# Run specific model and all downstream
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Run specific model and all upstream
dbt run --select +fct_listing_calendar --profiles-dir .

# Run models matching tag
dbt run --select tag:daily --profiles-dir .
```

### Testing Strategies

```bash
# Test only sources
dbt test --select source:* --profiles-dir .

# Test specific layer
dbt test --select marts --profiles-dir .

# Run tests but skip data tests (only schema tests)
dbt test --exclude test_type:data --profiles-dir .
```

## Troubleshooting

### Issue: `Database does not exist or not authorized`

**Solution:** Run the Snowflake setup script first:

```sql
-- Execute in Snowflake manually or via loader script
CREATE DATABASE IF NOT EXISTS AIRBNB_ANALYTICS;
CREATE SCHEMA IF NOT EXISTS AIRBNB_ANALYTICS.RAW;
CREATE WAREHOUSE IF NOT EXISTS AIRBNB_ANALYTICS_WH WITH WAREHOUSE_SIZE = 'XSMALL';
```

The Python loader runs `setup/snowflake_setup.sql` automatically.

### Issue: `dbt debug` fails with connection error

**Solution:** Verify credentials and use explicit profile directory:

```bash
# Check profiles.yml exists
ls profiles.yml

# Verify environment variables
echo $SNOWFLAKE_ACCOUNT

# Run debug with explicit profile
dbt debug --profiles-dir . --profile snowflake_dbt_airbnb
```

### Issue: Incremental model not updating

**Solution:** Force full refresh:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Issue: `stg_airbnb__listings` returns empty results

**Solution:** Check that raw data was loaded:

```sql
-- In Snowflake
SELECT COUNT(*) FROM AIRBNB_ANALYTICS.RAW.LISTINGS;
```

If zero, re-run the loader:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

### Issue: Calendar fact table has duplicates

**Solution:** Check the singular test and fix upstream:

```bash
# Run duplicate test
dbt test --select assert_no_duplicate_listing_dates --profiles-dir .

# If duplicates exist, investigate source data
dbt run --full-refresh --select stg_airbnb__calendar+ --profiles-dir .
```

### Issue: Streamlit dashboard won't connect

**Solution:** Verify `config/local_credentials.json` exists and has correct values:

```bash
# Check file exists
ls config/local_credentials.json

# Test connection manually
python -c "import snowflake.connector; import json; creds = json.load(open('config/local_credentials.json')); conn = snowflake.connector.connect(**creds); print('Connected:', conn.is_closed())"
```

## Advanced Usage

### Add Custom dbt Macro for Revenue Calculation

**File: `macros/calculate_revenue_proxy.sql`**

```sql
{% macro calculate_revenue_proxy(available_col, price_col, adjusted_price_col) %}
    CASE
        WHEN {{ available_col }} = FALSE
        THEN COALESCE({{ adjusted_price_col }}, {{ price_col }}, 0)
        ELSE 0
    END
{% endmacro %}
```

**Usage in model:**

```sql
SELECT
    listing_id,
    calendar_date,
    {{ calculate_revenue_proxy('available', 'price', 'adjusted_price') }} AS estimated_revenue
FROM {{ ref('stg_airbnb__calendar') }}
```

### Add dbt Test for Price Range

**File: `tests/generic/test_price_in_range.sql`**

```sql
{% test price_in_range(model, column_name, min_value=0, max_value=10000) %}

SELECT *
FROM {{ model }}
WHERE {{ column_name }} < {{ min_value }}
   OR {{ column_name }} > {{ max_value }}

{% endtest %}
```

**Apply in schema.yml:**

```yaml
models:
  - name: stg_airbnb__listings
    columns:
      - name: price
        tests:
          - price_in_range:
              min_value: 0
              max_value: 10000
```

### Add dbt Documentation

**File: `models/marts/schema.yml`**

```yaml
version: 2

models:
  - name: fct_listing_calendar
    description: |
      Incremental fact table tracking daily calendar availability and pricing
      for each Airbnb listing. Includes estimated revenue proxy based on
      unavailable nights.
    columns:
      - name: listing_id
        description: Foreign key to dim_listings
      - name: calendar_date
        description: Calendar date for availability snapshot
      - name: estimated_revenue
        description: Proxy revenue if night is booked (unavailable)
```

Generate and view:

```bash
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir . --port 8080
```

## Project-Specific Notes

- **No Real Transactions**: Inside Airbnb doesn't provide booking data. The project uses calendar unavailability as a revenue proxy.
- **Incremental Strategy**: `fct_listing_calendar` uses Snowflake's `merge` strategy with unique key `(listing_id, calendar_date)`.
- **Credential Files**: `profiles.yml` and `config/local_credentials.json` are gitignored. Always use the `.example` templates.
- **Data Freshness**: Inside Airbnb snapshots are periodic. For production, schedule `load_inside_airbnb_to_snowflake.py` and `dbt run` via Airflow or dbt Cloud.

This skill covers the complete workflow from raw data ingestion to analytics-ready marts and visualization with Streamlit, following dbt and Snowflake best practices.
