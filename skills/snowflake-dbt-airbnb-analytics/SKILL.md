---
name: snowflake-dbt-airbnb-analytics
description: Analytics engineering project using Snowflake, dbt, and Streamlit for Inside Airbnb data with marts, tests, and incremental models
triggers:
  - how do I set up the Airbnb Snowflake dbt project
  - load Inside Airbnb data into Snowflake
  - run dbt models for Airbnb analytics
  - build incremental fact tables with dbt and Snowflake
  - create Streamlit dashboard with Snowflake connection
  - configure dbt profiles for Snowflake
  - test dbt models for data quality
  - build analytics marts with dbt staging and intermediate layers
---

# snowflake-dbt-airbnb-analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete analytics engineering workflow: loading Inside Airbnb open data into Snowflake, transforming it through dbt staging/intermediate/mart layers with incremental models, validating data quality with tests, and serving results via a Streamlit dashboard.

## What This Project Does

- Loads raw CSV/GZIP files (listings, calendar, reviews, neighbourhoods) into Snowflake
- Transforms data through dbt layers: staging → intermediate → marts (dimensions, facts, aggregates)
- Implements incremental fact modeling with Snowflake merge strategy
- Validates data quality with dbt generic and singular tests
- Powers a Streamlit dashboard connected to analytics marts
- Uses local-only credential files (ignored by git) for public safety

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

### 1. Set Up Local Credentials (Not Committed to Git)

```bash
# Copy example files
cp profiles.yml.example profiles.yml
cp config/local_credentials.example.json config/local_credentials.json
```

### 2. Edit `profiles.yml` with Snowflake Connection

```yaml
airbnb_analytics:
  target: dev
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
      client_session_keep_alive: False
```

### 3. Edit `config/local_credentials.json` for Streamlit

```json
{
  "snowflake": {
    "account": "YOUR_ACCOUNT",
    "user": "YOUR_USER",
    "password": "YOUR_PASSWORD",
    "warehouse": "COMPUTE_WH",
    "database": "AIRBNB_ANALYTICS",
    "schema": "ANALYTICS",
    "role": "YOUR_ROLE"
  }
}
```

**Security**: Both files are `.gitignore`d. Use environment variables in production:

```python
import os
snowflake_user = os.getenv('SNOWFLAKE_USER')
snowflake_password = os.getenv('SNOWFLAKE_PASSWORD')
```

## Data Preparation

Download Inside Airbnb data for New York City and place in `data/raw/`:

```text
data/raw/listings.csv.gz
data/raw/calendar.csv.gz
data/raw/reviews.csv.gz
data/raw/neighbourhoods.csv
```

Download from: https://insideairbnb.com/get-the-data/

## Loading Data into Snowflake

The Python loader script creates Snowflake objects, uploads files to internal stage, and copies into raw tables:

```bash
python scripts/load_inside_airbnb_to_snowflake.py
```

### Understanding the Loader Script

```python
# Key sections from load_inside_airbnb_to_snowflake.py

import snowflake.connector
import json

# Load credentials
with open('config/local_credentials.json') as f:
    creds = json.load(f)['snowflake']

# Connect to Snowflake
conn = snowflake.connector.connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    warehouse=creds['warehouse'],
    database=creds['database'],
    role=creds['role']
)

# Execute setup SQL (creates schemas, stages)
with open('setup/snowflake_setup.sql') as f:
    setup_sql = f.read()
    for statement in setup_sql.split(';'):
        if statement.strip():
            conn.cursor().execute(statement)

# Upload files to Snowflake internal stage
conn.cursor().execute("PUT file://data/raw/listings.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")
conn.cursor().execute("PUT file://data/raw/calendar.csv.gz @INSIDE_AIRBNB_STAGE AUTO_COMPRESS=FALSE")

# Copy into raw tables
conn.cursor().execute("""
COPY INTO RAW.LISTINGS
FROM @INSIDE_AIRBNB_STAGE/listings.csv.gz
FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
ON_ERROR = CONTINUE
""")
```

## dbt Model Layers

### Staging Models

Clean raw data, cast types, standardize names:

```sql
-- models/staging/stg_airbnb__listings.sql
{{ config(materialized='view') }}

SELECT
    id::BIGINT AS listing_id,
    host_id::BIGINT AS host_id,
    host_name,
    neighbourhood,
    neighbourhood_group,
    latitude::FLOAT AS latitude,
    longitude::FLOAT AS longitude,
    room_type,
    price::FLOAT AS price,
    minimum_nights::INT AS minimum_nights,
    number_of_reviews::INT AS number_of_reviews,
    last_review::DATE AS last_review_date,
    reviews_per_month::FLOAT AS reviews_per_month,
    calculated_host_listings_count::INT AS host_listings_count,
    availability_365::INT AS availability_365
FROM {{ source('raw_airbnb', 'listings') }}
WHERE id IS NOT NULL
```

### Intermediate Models

Join and enrich data:

```sql
-- models/intermediate/int_airbnb__listing_enriched.sql
{{ config(materialized='view') }}

SELECT
    l.listing_id,
    l.host_id,
    l.host_name,
    l.neighbourhood,
    n.neighbourhood_group,
    l.latitude,
    l.longitude,
    l.room_type,
    l.price,
    l.minimum_nights,
    l.number_of_reviews,
    l.last_review_date,
    l.reviews_per_month,
    l.host_listings_count,
    l.availability_365
FROM {{ ref('stg_airbnb__listings') }} l
LEFT JOIN {{ ref('stg_airbnb__neighbourhoods') }} n
    ON l.neighbourhood = n.neighbourhood
```

### Mart Models - Incremental Fact Table

```sql
-- models/marts/fct_listing_calendar.sql
{{
  config(
    materialized='incremental',
    unique_key=['listing_id', 'calendar_date'],
    on_schema_change='fail'
  )
}}

WITH calendar_enriched AS (
    SELECT
        listing_id,
        calendar_date,
        available,
        price,
        minimum_nights,
        maximum_nights
    FROM {{ ref('int_airbnb__calendar_enriched') }}
    {% if is_incremental() %}
    WHERE calendar_date > (SELECT MAX(calendar_date) FROM {{ this }})
    {% endif %}
)

SELECT
    listing_id,
    calendar_date,
    available,
    price,
    minimum_nights,
    maximum_nights,
    CASE WHEN available = 'f' THEN price ELSE 0 END AS estimated_revenue
FROM calendar_enriched
```

### Aggregate Models

```sql
-- models/marts/agg_listing_monthly_performance.sql
{{ config(materialized='table') }}

SELECT
    listing_id,
    DATE_TRUNC('month', calendar_date) AS month,
    COUNT(*) AS total_days,
    SUM(CASE WHEN available = 'f' THEN 1 ELSE 0 END) AS unavailable_days,
    SUM(estimated_revenue) AS estimated_monthly_revenue,
    AVG(price) AS avg_daily_price
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, DATE_TRUNC('month', calendar_date)
```

## Running dbt

```bash
# Test connection
dbt debug --profiles-dir .

# Run all models
dbt run --profiles-dir .

# Run specific model and downstream dependencies
dbt run --select fct_listing_calendar+ --profiles-dir .

# Full refresh for incremental models
dbt run --full-refresh --select fct_listing_calendar --profiles-dir .

# Run tests
dbt test --profiles-dir .

# Generate documentation
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .
```

## dbt Tests

### Generic Tests in Schema YAML

```yaml
# models/staging/schema.yml
version: 2

models:
  - name: stg_airbnb__listings
    columns:
      - name: listing_id
        tests:
          - unique
          - not_null
      - name: room_type
        tests:
          - accepted_values:
              values: ['Entire home/apt', 'Private room', 'Shared room', 'Hotel room']
      - name: price
        tests:
          - not_null
          - dbt_utils.expression_is_true:
              expression: ">= 0"
```

### Singular Tests

```sql
-- tests/singular/assert_no_duplicate_listing_dates.sql
SELECT
    listing_id,
    calendar_date,
    COUNT(*) AS occurrences
FROM {{ ref('fct_listing_calendar') }}
GROUP BY listing_id, calendar_date
HAVING COUNT(*) > 1
```

## Streamlit Dashboard

```python
# dashboard/streamlit_app.py
import streamlit as st
import snowflake.connector
import pandas as pd
import json

# Load credentials
with open('config/local_credentials.json') as f:
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
        schema=creds['schema'],
        role=creds['role']
    )

conn = get_connection()

st.title("Airbnb Analytics Dashboard")

# Query aggregated data
query = """
SELECT
    neighbourhood_group,
    SUM(estimated_monthly_revenue) AS total_revenue,
    AVG(avg_daily_price) AS avg_price
FROM agg_neighbourhood_monthly_performance
GROUP BY neighbourhood_group
ORDER BY total_revenue DESC
"""

df = pd.read_sql(query, conn)
st.bar_chart(df.set_index('neighbourhood_group')['total_revenue'])

# Top hosts
hosts_query = """
SELECT
    host_name,
    COUNT(DISTINCT listing_id) AS listing_count
FROM dim_listings
GROUP BY host_name
ORDER BY listing_count DESC
LIMIT 10
"""

hosts_df = pd.read_sql(hosts_query, conn)
st.dataframe(hosts_df)
```

Run the dashboard:

```bash
streamlit run dashboard/streamlit_app.py
```

## Common Patterns

### Pattern 1: Source Configuration

```yaml
# models/staging/sources.yml
version: 2

sources:
  - name: raw_airbnb
    database: AIRBNB_ANALYTICS
    schema: RAW
    tables:
      - name: listings
      - name: calendar
      - name: reviews
      - name: neighbourhoods
```

### Pattern 2: Incremental Model with Merge Strategy

```sql
{{ config(
    materialized='incremental',
    unique_key=['listing_id', 'calendar_date'],
    incremental_strategy='merge',
    on_schema_change='fail'
) }}

SELECT ... FROM {{ ref('source_model') }}
{% if is_incremental() %}
WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

### Pattern 3: Custom Schema Macro

```sql
-- macros/get_custom_schema.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

### Pattern 4: Reusable SQL Functions

```sql
-- macros/calculate_availability_rate.sql
{% macro calculate_availability_rate(available_column, total_column) %}
    ROUND(
        ({{ available_column }}::FLOAT / NULLIF({{ total_column }}, 0)) * 100,
        2
    )
{% endmacro %}

-- Usage in model:
{{ calculate_availability_rate('available_days', 'total_days') }} AS availability_rate
```

## Refreshing Data

For new Inside Airbnb snapshots:

```bash
# 1. Download new data to data/raw/
# 2. Reload Snowflake
python scripts/load_inside_airbnb_to_snowflake.py

# 3. Full refresh incremental models
dbt run --full-refresh --select fct_listing_calendar+ --profiles-dir .

# 4. Run tests
dbt test --profiles-dir .
```

## Troubleshooting

### Error: `Database 'AIRBNB_ANALYTICS' does not exist`

Ensure `setup/snowflake_setup.sql` creates the database:

```sql
CREATE DATABASE IF NOT EXISTS AIRBNB_ANALYTICS;
CREATE SCHEMA IF NOT EXISTS AIRBNB_ANALYTICS.RAW;
CREATE SCHEMA IF NOT EXISTS AIRBNB_ANALYTICS.ANALYTICS;
```

Run the loader script which executes setup SQL.

### Error: `Compilation Error in model ... depends on a node named 'source.raw_airbnb.listings' which was not found`

Check `models/staging/sources.yml` matches your Snowflake database/schema names:

```yaml
sources:
  - name: raw_airbnb
    database: AIRBNB_ANALYTICS  # Must match actual database
    schema: RAW                  # Must match actual schema
```

### Error: `Incremental model ... has schema changes`

Default `on_schema_change='fail'` prevents silent data loss. Options:

```sql
{{ config(
    materialized='incremental',
    on_schema_change='append_new_columns'  # or 'sync_all_columns' or 'ignore'
) }}
```

Or full refresh:

```bash
dbt run --full-refresh --select model_name
```

### Streamlit: `ModuleNotFoundError: No module named 'snowflake'`

Install connector:

```bash
pip install snowflake-connector-python
```

### dbt: `Runtime Error: Could not find profile named 'airbnb_analytics'`

Ensure `dbt_project.yml` profile name matches `profiles.yml`:

```yaml
# dbt_project.yml
profile: 'airbnb_analytics'
```

```yaml
# profiles.yml
airbnb_analytics:  # Must match
  target: dev
```

### Performance: Slow `calendar` table queries

The calendar table is large. Cluster fact tables:

```sql
{{ config(
    materialized='incremental',
    cluster_by=['calendar_date']
) }}
```

Or filter date ranges in queries:

```sql
SELECT * FROM fct_listing_calendar
WHERE calendar_date BETWEEN '2024-01-01' AND '2024-12-31'
```

## Business Questions Examples

```sql
-- Which neighbourhoods have highest revenue?
SELECT
    neighbourhood_group,
    SUM(estimated_monthly_revenue) AS total_revenue
FROM agg_neighbourhood_monthly_performance
GROUP BY neighbourhood_group
ORDER BY total_revenue DESC;

-- Top hosts by listing count
SELECT
    host_name,
    COUNT(DISTINCT listing_id) AS listings
FROM dim_listings
GROUP BY host_name
ORDER BY listings DESC
LIMIT 10;

-- Average price by room type
SELECT
    room_type,
    AVG(price) AS avg_price
FROM dim_listings
GROUP BY room_type;

-- Monthly availability trends
SELECT
    month,
    AVG(unavailable_days::FLOAT / total_days * 100) AS avg_occupancy_rate
FROM agg_listing_monthly_performance
GROUP BY month
ORDER BY month;
```

## Key Files

- `scripts/load_inside_airbnb_to_snowflake.py` — Data loader
- `setup/snowflake_setup.sql` — Database/schema/stage DDL
- `dbt_project.yml` — dbt project config
- `profiles.yml` — Snowflake connection (local, gitignored)
- `models/staging/` — Staging layer views
- `models/intermediate/` — Intermediate joins
- `models/marts/` — Dimensions, facts, aggregates
- `tests/` — Singular SQL tests
- `dashboard/streamlit_app.py` — Streamlit reporting UI
- `config/local_credentials.json` — Streamlit Snowflake config (local, gitignored)

This skill covers the complete workflow from raw data to dashboard, with incremental models, testing, and public-safe credential patterns.
