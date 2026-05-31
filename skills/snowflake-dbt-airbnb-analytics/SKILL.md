---
name: snowflake-dbt-airbnb-analytics
description: Snowflake + dbt + Streamlit analytics engineering project for Inside Airbnb data with staging, marts, tests, and dashboards
triggers:
  - set up airbnb analytics with snowflake and dbt
  - load inside airbnb data to snowflake
  - create dbt staging and mart models for airbnb
  - build analytics dashboard with streamlit
  - configure dbt profiles for snowflake
  - run incremental models in dbt snowflake
  - test data quality with dbt tests
  - generate dbt documentation for airbnb project
---

# Snowflake dbt Airbnb Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete analytics engineering workflow:
- Load Inside Airbnb CSV data into Snowflake raw tables
- Transform with dbt (staging → intermediate → marts)
- Validate data quality with dbt tests
- Visualize in Streamlit dashboard

**Key layers:**
- **Staging**: Clean, cast, standardize (`stg_airbnb__*`)
- **Intermediate**: Joins, enrichment, revenue proxy (`int_airbnb__*`)
- **Marts**: Dimensions (`dim_*`), facts (`fct_*`), aggregates (`agg_*`)

## Installation

```bash
# Clone and create virtual environment
git clone https://github.com/analyticsdurgesh/Snowflake_DBT_Project.git
cd Snowflake_DBT_Project
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

**Requirements.txt includes:**
- dbt-snowflake
- snowflake-connector-python
- streamlit
- pandas

## Configuration

### 1. Snowflake Credentials (dbt)

Create `profiles.yml` from example:

```bash
cp profiles.yml.example profiles.yml
```

Edit `profiles.yml`:

```yaml
snowflake_dbt_project:
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT
      user: YOUR_USER
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: YOUR_ROLE
      database: AIRBNB_DB
      warehouse: COMPUTE_WH
      schema: ANALYTICS
      threads: 4
      client_session_keep_alive: False
  target: dev
```

**Best practice**: Use environment variables for secrets:

```bash
export SNOWFLAKE_PASSWORD='your_password'
```

### 2. Snowflake Credentials (Python Loader)

Create `config/local_credentials.json` from example:

```bash
cp config/local_credentials.example.json config/local_credentials.json
```

Edit `config/local_credentials.json`:

```json
{
  "snowflake": {
    "account": "YOUR_ACCOUNT",
    "user": "YOUR_USER",
    "password": "YOUR_PASSWORD",
    "warehouse": "COMPUTE_WH",
    "database": "AIRBNB_DB",
    "schema": "RAW",
    "role": "YOUR_ROLE"
  }
}
```

**Note**: This file is `.gitignore`'d — never commit credentials.

### 3. Download Inside Airbnb Data

Download NYC data from [Inside Airbnb](https://insideairbnb.com/get-the-data/):

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

## Loading Data to Snowflake

Run the Python loader script:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

**What it does:**
1. Executes `setup/snowflake_setup.sql` (creates database, schemas, stage)
2. Uploads CSV/GZIP files to Snowflake internal stage
3. Creates raw tables with inferred schema
4. Copies data from stage to `RAW` schema tables

**Key code from loader:**

```python
import snowflake.connector
import json

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)['snowflake']

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

# Upload file to stage
cursor = conn.cursor()
cursor.execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE")

# Copy into raw table
cursor.execute("""
    COPY INTO RAW.LISTINGS
    FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
    ON_ERROR = 'CONTINUE'
""")
```

## dbt Commands

### Initial Setup

```bash
# Verify connection
dbt debug --profiles-dir .

# Install dependencies (if any packages)
dbt deps --profiles-dir .
```

### Running Models

```bash
# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select stg_airbnb__listings+ --profiles-dir .

# Run only marts layer
dbt run --select marts.* --profiles-dir .

# Full refresh (rebuild incremental models)
dbt run --full-refresh --profiles-dir .
```

### Testing

```bash
# Run all tests
dbt test --profiles-dir .

# Test specific model
dbt test --select dim_listings --profiles-dir .

# Test sources only
dbt test --select source:* --profiles-dir .
```

### Documentation

```bash
# Generate documentation
dbt docs generate --profiles-dir .

# Serve documentation site
dbt docs serve --profiles-dir . --port 8080
```

## dbt Model Examples

### Staging Model: `stg_airbnb__listings.sql`

```sql
-- models/staging/stg_airbnb__listings.sql
WITH source AS (
    SELECT * FROM {{ source('airbnb_raw', 'listings') }}
),

cleaned AS (
    SELECT
        id::BIGINT AS listing_id,
        name::VARCHAR AS listing_name,
        host_id::BIGINT AS host_id,
        host_name::VARCHAR AS host_name,
        neighbourhood_group::VARCHAR AS neighbourhood_group,
        neighbourhood::VARCHAR AS neighbourhood,
        latitude::FLOAT AS latitude,
        longitude::FLOAT AS longitude,
        room_type::VARCHAR AS room_type,
        price::FLOAT AS price,
        minimum_nights::INTEGER AS minimum_nights,
        number_of_reviews::INTEGER AS number_of_reviews,
        last_review::DATE AS last_review_date,
        reviews_per_month::FLOAT AS reviews_per_month,
        calculated_host_listings_count::INTEGER AS host_listings_count,
        availability_365::INTEGER AS availability_365
    FROM source
)

SELECT * FROM cleaned
```

### Intermediate Model: `int_airbnb__listing_enriched.sql`

```sql
-- models/intermediate/int_airbnb__listing_enriched.sql
WITH listings AS (
    SELECT * FROM {{ ref('stg_airbnb__listings') }}
),

neighbourhoods AS (
    SELECT * FROM {{ ref('stg_airbnb__neighbourhoods') }}
),

enriched AS (
    SELECT
        l.*,
        n.neighbourhood_group AS neighbourhood_group_official,
        CASE
            WHEN l.price > 500 THEN 'Premium'
            WHEN l.price > 200 THEN 'High'
            WHEN l.price > 100 THEN 'Medium'
            ELSE 'Budget'
        END AS price_tier,
        CASE
            WHEN l.availability_365 > 300 THEN 'High'
            WHEN l.availability_365 > 150 THEN 'Medium'
            ELSE 'Low'
        END AS availability_tier
    FROM listings l
    LEFT JOIN neighbourhoods n
        ON l.neighbourhood = n.neighbourhood
)

SELECT * FROM enriched
```

### Incremental Fact Model: `fct_listing_calendar.sql`

```sql
-- models/marts/fct_listing_calendar.sql
{{
    config(
        materialized='incremental',
        unique_key='calendar_key',
        merge_update_columns=['available', 'price', 'adjusted_price', 'estimated_revenue']
    )
}}

WITH calendar AS (
    SELECT * FROM {{ ref('int_airbnb__calendar_enriched') }}
),

final AS (
    SELECT
        {{ dbt_utils.generate_surrogate_key(['listing_id', 'date']) }} AS calendar_key,
        listing_id,
        date,
        available,
        price,
        adjusted_price,
        CASE
            WHEN available = 'f' THEN COALESCE(adjusted_price, price, 0)
            ELSE 0
        END AS estimated_revenue,
        EXTRACT(YEAR FROM date) AS year,
        EXTRACT(MONTH FROM date) AS month,
        EXTRACT(DAY FROM date) AS day,
        DAYNAME(date) AS day_name
    FROM calendar
    {% if is_incremental() %}
    WHERE date > (SELECT MAX(date) FROM {{ this }})
    {% endif %}
)

SELECT * FROM final
```

### Dimension Model: `dim_hosts.sql`

```sql
-- models/marts/dim_hosts.sql
WITH listings AS (
    SELECT * FROM {{ ref('int_airbnb__listing_enriched') }}
),

host_agg AS (
    SELECT
        host_id,
        MAX(host_name) AS host_name,
        COUNT(DISTINCT listing_id) AS total_listings,
        AVG(price) AS avg_listing_price,
        SUM(number_of_reviews) AS total_reviews,
        MIN(last_review_date) AS first_review_date,
        MAX(last_review_date) AS last_review_date,
        LISTAGG(DISTINCT neighbourhood, ', ') WITHIN GROUP (ORDER BY neighbourhood) AS neighbourhoods,
        LISTAGG(DISTINCT room_type, ', ') WITHIN GROUP (ORDER BY room_type) AS room_types
    FROM listings
    GROUP BY host_id
)

SELECT * FROM host_agg
```

### Aggregate Model: `agg_neighbourhood_monthly_performance.sql`

```sql
-- models/marts/agg_neighbourhood_monthly_performance.sql
WITH listing_monthly AS (
    SELECT * FROM {{ ref('agg_listing_monthly_performance') }}
),

listings AS (
    SELECT * FROM {{ ref('dim_listings') }}
),

neighbourhood_monthly AS (
    SELECT
        lm.year,
        lm.month,
        l.neighbourhood,
        l.neighbourhood_group,
        COUNT(DISTINCT lm.listing_id) AS active_listings,
        SUM(lm.total_available_days) AS total_available_days,
        SUM(lm.total_booked_days) AS total_booked_days,
        AVG(lm.avg_price) AS avg_price,
        SUM(lm.total_estimated_revenue) AS total_estimated_revenue,
        AVG(lm.occupancy_rate) AS avg_occupancy_rate
    FROM listing_monthly lm
    INNER JOIN listings l ON lm.listing_id = l.listing_id
    GROUP BY 1, 2, 3, 4
)

SELECT * FROM neighbourhood_monthly
```

## dbt Sources Configuration

**models/staging/sources.yml:**

```yaml
version: 2

sources:
  - name: airbnb_raw
    database: AIRBNB_DB
    schema: RAW
    tables:
      - name: listings
        description: Raw Airbnb listings data
        columns:
          - name: id
            description: Unique listing identifier
            tests:
              - not_null
              - unique
      - name: calendar
        description: Daily availability and pricing
        columns:
          - name: listing_id
            tests:
              - not_null
          - name: date
            tests:
              - not_null
      - name: reviews
        description: Guest reviews
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

### Generic Tests in Schema Files

**models/marts/schema.yml:**

```yaml
version: 2

models:
  - name: dim_listings
    description: Listing dimension table
    columns:
      - name: listing_id
        description: Primary key
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
          - dbt_utils.expression_is_true:
              expression: ">= 0"

  - name: fct_listing_calendar
    description: Daily listing calendar fact table
    columns:
      - name: calendar_key
        tests:
          - not_null
          - unique
      - name: listing_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_listings')
              field: listing_id
```

### Singular Tests

**tests/assert_no_duplicate_listing_dates.sql:**

```sql
-- Ensure no duplicate listing_id + date combinations
SELECT
    listing_id,
    date,
    COUNT(*) AS record_count
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, date
HAVING COUNT(*) > 1
```

## Streamlit Dashboard

**dashboard/streamlit_app.py:**

```python
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
with open('config/local_credentials.json', 'r') as f:
    creds = json.load(f)['snowflake']

# Connect to Snowflake
@st.cache_resource
def get_connection():
    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        password=creds['password'],
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema='ANALYTICS',
        role=creds['role']
    )

conn = get_connection()

st.title("Airbnb Analytics Dashboard")

# Top neighbourhoods by revenue
st.header("Top 10 Neighbourhoods by Estimated Revenue")
query = """
SELECT
    neighbourhood,
    SUM(total_estimated_revenue) AS total_revenue,
    AVG(avg_occupancy_rate) AS avg_occupancy
FROM agg_neighbourhood_monthly_performance
WHERE year = 2024
GROUP BY neighbourhood
ORDER BY total_revenue DESC
LIMIT 10
"""
df = pd.read_sql(query, conn)
st.bar_chart(df.set_index('neighbourhood')['total_revenue'])

# Host analytics
st.header("Top Hosts by Listings")
query_hosts = """
SELECT
    host_name,
    total_listings,
    avg_listing_price,
    total_reviews
FROM dim_hosts
ORDER BY total_listings DESC
LIMIT 10
"""
df_hosts = pd.read_sql(query_hosts, conn)
st.dataframe(df_hosts)

# Monthly trends
st.header("Monthly Revenue Trend")
month_filter = st.slider("Month", 1, 12, (1, 12))
query_monthly = f"""
SELECT
    year || '-' || LPAD(month::VARCHAR, 2, '0') AS year_month,
    SUM(total_estimated_revenue) AS revenue
FROM agg_neighbourhood_monthly_performance
WHERE month BETWEEN {month_filter[0]} AND {month_filter[1]}
GROUP BY year_month
ORDER BY year_month
"""
df_monthly = pd.read_sql(query_monthly, conn)
st.line_chart(df_monthly.set_index('year_month'))
```

**Run dashboard:**

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Workflows

### Fresh Data Load

When you get a new Inside Airbnb snapshot:

```bash
# 1. Load raw data
python scripts/load_inside_airbnb_to_snowflake.py

# 2. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 3. Run all other models
dbt run --exclude fct_listing_calendar --profiles-dir .

# 4. Test everything
dbt test --profiles-dir .
```

### Develop New Model

```bash
# 1. Create model file in models/marts/
# 2. Run just that model
dbt run --select my_new_model --profiles-dir .

# 3. Add tests in schema.yml
# 4. Run tests
dbt test --select my_new_model --profiles-dir .

# 5. Generate docs
dbt docs generate --profiles-dir .
```

### Debug Failed Model

```bash
# Run with verbose logging
dbt run --select failed_model --profiles-dir . --debug

# Compile SQL without running
dbt compile --select failed_model --profiles-dir .

# Check compiled SQL in target/compiled/
cat target/compiled/snowflake_dbt_project/models/marts/failed_model.sql
```

## Macros and Utilities

**macros/generate_schema_name.sql:**

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

**Usage in model:**

```sql
-- models/marts/dim_listings.sql
{{ config(schema='marts') }}
-- This will create table in MARTS schema instead of ANALYTICS_MARTS
```

## Troubleshooting

### dbt Connection Errors

**Error**: `Database 'AIRBNB_DB' does not exist`

**Fix**: Run Snowflake setup first:

```sql
-- In Snowflake UI or SnowSQL
CREATE DATABASE IF NOT EXISTS AIRBNB_DB;
CREATE SCHEMA IF NOT EXISTS AIRBNB_DB.RAW;
CREATE SCHEMA IF NOT EXISTS AIRBNB_DB.ANALYTICS;
```

Or use the loader script which executes `setup/snowflake_setup.sql`.

### Incremental Model Not Updating

**Error**: New data not appearing in `fct_listing_calendar`

**Fix**: Full refresh the incremental model:

```bash
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .
```

### Test Failures on Relationships

**Error**: `relationships` test fails for `listing_id`

**Cause**: Orphan records in fact table (listing deleted from dimension)

**Fix**: Add referential integrity in intermediate model:

```sql
-- Only include listings that exist in dimension
WHERE listing_id IN (SELECT listing_id FROM {{ ref('dim_listings') }})
```

### Streamlit Connection Timeout

**Error**: `snowflake.connector.errors.DatabaseError: 250001`

**Fix**: Increase warehouse size or add `client_session_keep_alive`:

```python
conn = snowflake.connector.connect(
    ...
    client_session_keep_alive=True
)
```

### File Upload Errors

**Error**: `PUT file://data/raw/listings.csv.gz` fails

**Fix**: Check file path is relative to script location:

```python
import os
script_dir = os.path.dirname(__file__)
file_path = os.path.join(script_dir, '../data/raw/listings.csv.gz')
cursor.execute(f"PUT file://{file_path} @INSIDE_AIRBNB_STAGE")
```

## Best Practices

1. **Never commit credentials** — use `profiles.yml` and `local_credentials.json` (both gitignored)
2. **Use environment variables** for CI/CD:
   ```bash
   export DBT_SNOWFLAKE_PASSWORD='xxx'
   dbt run --profiles-dir .
   ```
3. **Tag models** for selective runs:
   ```yaml
   {{ config(tags=['daily', 'critical']) }}
   ```
   ```bash
   dbt run --select tag:daily
   ```
4. **Document all models** in `schema.yml` with descriptions
5. **Full refresh incrementals** when raw data changes significantly
6. **Use `dbt_utils.generate_surrogate_key`** for composite keys
7. **Test referential integrity** between facts and dimensions

## Project Structure Reference

```
models/
├── staging/
│   ├── sources.yml
│   ├── stg_airbnb__listings.sql
│   ├── stg_airbnb__calendar.sql
│   ├── stg_airbnb__reviews.sql
│   └── stg_airbnb__neighbourhoods.sql
├── intermediate/
│   ├── int_airbnb__listing_enriched.sql
│   ├── int_airbnb__calendar_enriched.sql
│   └── int_airbnb__reviews_enriched.sql
└── marts/
    ├── schema.yml
    ├── dim_listings.sql
    ├── dim_hosts.sql
    ├── fct_listing_calendar.sql (incremental)
    ├── fct_reviews.sql
    ├── agg_listing_monthly_performance.sql
    └── agg_neighbourhood_monthly_performance.sql
```

This skill enables AI agents to help developers build, run, test, and troubleshoot this complete Snowflake + dbt + Streamlit analytics project.
